# Golang标准库源码阅读 - net/http.Transport #

最近实现网关的时候采用了http.Transport来实现了http协议反向代理，踩了一些坑，浪费了一些时间解决问题，最后下定决心要把源码好好看一下，并把部分遇涉及RoundTrip问题及解决办法加以记录。
在使用net/http库发送http请求，包括net/http/client.go中提供的各种方法最后都会调用Transport的RoundTrip方法，Transport实现了RoundTripper接口，```RoundTripper is an interface representing the ability to execute a single HTTP transaction, obtaining the Response for a given Request.``` RoundTrip实现了一个http事务，返回指定请求的响应。

```
type RoundTripper interface {
    RoundTrip(*Request) (*Response, error)
}
```

接下来对Transport::RoundTrip做较详细的理解说明。

## 0. Prerequisite ##
看一下Transport的数据结构

```go
type Transport struct {
    idleMu     sync.Mutex
    //注：在CloseIdleConnections()时，wantIdle会被置为true，关闭所有idleConn
    wantIdle   bool                                // user has requested to close all idle conns 
    idleConn   map[connectMethodKey][]*persistConn // most recently used at end
    idleConnCh map[connectMethodKey]chan *persistConn
    idleLRU    connLRU

    reqMu       sync.Mutex
    reqCanceler map[*Request]func(error)
    
    // DisableKeepAlives, if true, prevents re-use of TCP connections
    // between different HTTP requests.
    DisableKeepAlives bool
    ......
}
```
0. go version go1.10.3 linux/amd64
1. DisableKeepAlives 默认是false，使用长连接，注释中说明了设置为true，不会重用TCP链接
2. Proxy的内部这里也不做过多介绍(socks5/http/https)
3. RegisterProtocol可以注册新的协议scheme，相当于http协议上的一层协议，请求会转给注册scheme的RoundTripper而不走http的RoundTrip，具体使用可以参考标准库net/http/filetransport.go, net/http/h2_bundle.go有http/2的实现，https的内容涉及DialTLS的调用将会建立一个TLS的链接，这里不对https和http/2做过多介绍。

## 1. 获取链接
`getConn` 根据connectMethod获取或者创建一个\*persistConn, 连接池结构是这样的:idleConn map[connectMethodKey][]\*persistConn。其中connectMethodKey根据connectMethod生成的：`type connectMethodKey struct {proxy, scheme, addr string}`, map的值是一个*persistConn类型的slice结构，这里就是存放连接的地方，slice的length由MaxIdleConnsPerHost指定的，默认值为：`const DefaultMaxIdleConnsPerHost = 2`

```
func (t *Transport) getConn(treq *transportRequest, cm connectMethod) (*persistConn, error) { 
    ......
    if pc, idleSince := t.getIdleConn(cm); pc != nil {
        ......
        t.setReqCanceler(req, func(error) {})
        return pc, nil
    }
```

在getConn刚开始会首先调用`getIdleConn()`获取已经建立并且闲置的persistConn

```
func (t *Transport) getIdleConn(cm connectMethod) (pconn *persistConn, idleSince time.Time) {
    key := cm.key()
    t.idleMu.Lock()
    defer t.idleMu.Unlock()
    for {
        pconns, ok := t.idleConn[key]
        if !ok {
            return nil, time.Time{}
        }
        if len(pconns) == 1 {
            pconn = pconns[0]
            delete(t.idleConn, key)
        } else {
            // 2 or more cached connections; use the most
            // recently used one at the end.
            pconn = pconns[len(pconns)-1]
            t.idleConn[key] = pconns[:len(pconns)-1]
        }
        t.idleLRU.remove(pconn)
        if pconn.isBroken() {
            // There is a tiny window where this is
            // possible, between the connecting dying and
            // the persistConn readLoop calling
            // Transport.removeIdleConn. Just skip it and
            // carry on.
            continue
        }
        if pconn.idleTimer != nil && !pconn.idleTimer.Stop() {
            // We picked this conn at the ~same time it
            // was expiring and it's trying to close
            // itself in another goroutine. Don't use it.
            continue
        }
        return pconn, pconn.idleAt
    }
}
```

getIdleConn函数for死循环内根据connectMethod生成key获取t.idleConn(连接池)满足条件的[]*persistConn，只有一条空闲链接的时候返回这一条链接，超过2条及以上，选取最近使用的一条（most recently used at end），选取之后并且返回使用之前还要检查这个链接是否已经中断了以及这条空闲链接中的定时器是否已经超时，如中断或者超时，continue重新在空闲连接池内再找一条，上述代码大致的过程就是这样。

如果getIdleConn获取到一个空闲链接，那么getConn函数将返回这个链接以供外面使用。如果没有获取到合适的空闲链接，那么接下来就需要创建一条新的链接。

###1.1 启动一个goroutine创建一个新的连接

```
    go func() {
        pc, err := t.dialConn(ctx, cm)
        dialc <- dialRes{pc, err}
    }()
```
t.dialConn中调用Transport::dial()，网络连接通过Dialer.DialContext(注：Transport的注释中说明Dialer.Dial:Deprecated: Use DialContext instead)创建，如果scheme为https，还会有TLS的一些初始化，如果是代理的形式还会有相关的操作，这里不做过多介绍。在dialConn最后有如下代码:

```
    pconn.br = bufio.NewReader(pconn)
    pconn.bw = bufio.NewWriter(persistConnWriter{pconn})
    go pconn.readLoop()
    go pconn.writeLoop()
```
这里br和bw包装了一下获得的连接pconn，然后启动了读和写循环，readLoop主要是读取从server返回的数据,writeLoop主要发送请求到server, 后续再详细介绍。
创建后的连接发送到channel dialc中，等待接收。

根据connectMethod获取对应的idleConnCh，如果不存在则创建一个空的idleConnCh，需要再一次强调：**`DisableKeepAlives被置为true，idleConnCh逻辑都不生效`**。

接下利用select来处理建立链接过程当中的各种情况，如下5个case都是阻塞的channel在等待接受到来的值，整体阻塞在这里。

```
    select {
    case v := <-dialc:
    	 ...... // Block 1
    case pc := <-idleConnCh:
        handlePendingDial()
    	 ...... // Block 2
    case <-req.Cancel:
    	 handlePendingDial()
    	 ...... // Block 3
    case <-req.Context().Done():
    	 handlePendingDial()
    	 ...... // Block 4
    case err := <-cancelc:
    	 handlePendingDial()
    	 ...... // Block 5
    }
```
进入Block 1会做类似Block 3-5的判断；
对于Block 3-5比较好理解，request被取消或者超时，返回相关错误。
这里需要特别说明一下handlePendingDial，在select中的5个case中除了第一个外，其他的4个都需要调用它，`handlePendingDial`是一个Closure, 其中创建一个goroutine跟Block 1一样等待接收刚创建的连接，因为创建过程也在goroutine中完成，如果select多个case条件同时满足，会随机选取一个执行，例如走到Block 2搞到一个早前使用过的连接，如果有新的连接创建，不能浪费，也发送到idleConnCh中或者放到连接池，供后续使用，逻辑实现在`t.putOrCloseIdleConn(v.pc)`中，接下来重点来讲解一下, 代码如下：

```
    handlePendingDial := func() {
        testHookPrePendingDial()
        go func() {
            if v := <-dialc; v.err == nil {
                t.putOrCloseIdleConn(v.pc)
            }
            testHookPostPendingDial()
        }()
    }
```
putOrCloseIdleConn首先判断是不是persistConn是不是中断状态，如果是则调用close关闭，否则根据cacheKey(connectMethodKey)获取t.idleConnCh[key], 把自己发送到idleConnCh，参见之前讲到的接收的Block 2地方，如果idleConnCh被占用，则根据key把自己append到t.idleConn[key]slice尾部（放到连接池），t.idleLRU做调整，对于当前连接需要调整定时器超时时间，

```
pconn.idleTimer.Reset(t.IdleConnTimeout)
```
如果首次则创建Timer，

```
pconn.idleTimer = time.AfterFunc(t.IdleConnTimeout, pconn.closeConnIfStillIdle)
```
如果超时时间已到没有被更新，则连接会被关闭, 超时时间由IdleConnTimeout设定。

```
    // IdleConnTimeout is the maximum amount of time an idle (keep-alive) connection will remain idle before closing itself.
    // Zero means no limit.
    IdleConnTimeout time.Duration
```
###1.2 获取链接过程总结###

整个获取链接的过程大致如下图所示。

![getConn](https://github.com/GenuineJyn/genuinejyn.github.io/blob/master/pictures/getConn.png)


## 2 完成发送和接收通信过程 
获取可用的连接后，发送请求，接收响应就在获取的连接上完成, 如下。

```
resp, err = pconn.roundTrip(treq)
```
在前面讲过创建每一个连接的时候同时启动了两个goroutine：readLoop和writeLoop，需要跟roundTrip通过channel配合完成，这个配合过程巧妙的使用大量的channel。接下来看一下roundTrip内部的关键实现。

```
    writeErrCh := make(chan error, 1)
    pc.writech <- writeRequest{req, writeErrCh, continueCh}

    resc := make(chan responseAndError)
    pc.reqch <- requestAndChan{
        req:        req.Request,
        ch:         resc,
        addedGzip:  requestedGzip,
        continueCh: continueCh,
        callerGone: gone,
    }
```   

首先看pc.writech channel, roundTrip发送数据，在writeLoop接收数据，writeLoop接收到数据后主要做了三件事情：

```
	 err := wr.req.Request.write(pc.bw, pc.isProxy, wr.req.extra, pc.waitForContinue(wr.conti     nueCh))
	 if err == nil {
	 	err = pc.bw.Flush()
	 }
	 ......
	 pc.writeErrCh <- err 【标注点 3】
	 wr.ch <- err
```

将请求写到连接发送给服务端，`wr.ch <- err`将错误或nil反馈给roundTrip, `pc.writeErrCh <- err`, readLoop会接收，这里可以看到roundTrip->writeLoop()->remote_server完成全双工的写的过程, 跟随wc.ch回到roundTrip中继续看, 在RoundTrip最后是一个for死循环套着一个select语句，select中接收各种channel的信息，并作出相应的响应，对于wc.ch对应的是select中的`case err := <-writeErrCh`,如果err != nil关闭连接返回适当的错误信息。

继续, readLoop()主要是从服务端读回response，主要逻辑如下：

```
for alive {
	...... 
	// 试探性的读一个字节
	_, err := pc.br.Peek(1) 【标注点 6】
	......
	
	// 接收来自roundTrip的信息
	rc := <-pc.reqch
	......
	resp, err = pc.readResponse(rc, trace)
	
	body := &bodyEOFSignal{......} //【标注点 1】后面再详细介绍 
	resp.Body = body
	rc.ch <- responseAndError{res: resp}
	
	select {
		...... // 【标注点 2】后面再详细介绍
	}

}
```
### 2.1 net/http::Response.Close()

[stackoverflow: What could happen if I don't close response.Body in golang?
](https://stackoverflow.com/questions/33238518/what-could-happen-if-i-dont-close-response-body-in-golang)

stackoverflow上的这个问题有比较概括性的回答, 但是也没有说明白，很多其他forum也基本都是类似情况，只能看看具体内容，并得出了“诡异”的结论，回到上边讲到的【标注点 1】具体来看看，代码如下：

```
        body := &bodyEOFSignal{
            body: resp.Body,
            earlyCloseFn: func() error {
                waitForBodyRead <- false
                <-eofc // will be closed by deferred call at the end of the function
                return nil

            },
            fn: func(err error) error {
                isEOF := err == io.EOF
                waitForBodyRead <- isEOF
                if isEOF {
                    <-eofc // see comment above eofc declaration
                } else if err != nil {
                    if cerr := pc.canceled(); cerr != nil {
                        return cerr
                    }
                }
                return err
            },
        }
        
        resp.Body = body
```
可以看出resp.Body的实际类型是`*bodyEOFSignal`, 而bodyEOFSignal中body的wrap了一个`*http.body`，在bodyEOFSignal有两个Closure：earlyCloseFn/fn, 其中都会发送消息到waitForBodyRead之后都会阻塞在eofc，而eofc是defer close(eofc)只有等退出readLoop()才会被执行到，或者在下面的case中被发送，再看看**【标注点 2】**代码, 代码如下。

```
select {
        case bodyEOF := <-waitForBodyRead: 【标注点 4】
            pc.t.setReqCanceler(rc.req, nil) // before pc might return to idle pool
            alive = alive &&
                bodyEOF &&
                !pc.sawEOF &&
                pc.wroteRequest() &&
                tryPutIdleConn(trace)
            if bodyEOF {
                eofc <- struct{}{}
            }
        case <-rc.req.Cancel:
            alive = false
            pc.t.CancelRequest(rc.req)
        case <-rc.req.Context().Done():
            alive = false
            pc.t.cancelRequest(rc.req, rc.req.Context().Err())
        case <-pc.closech:
            alive = false
        }
```
代码是阻塞在这里的，等待其中一个得到满足，而waitForBodyRead是通过earlyCloseFn/fn来发送的
bodyEOFSignal::Read中会调用到bodyEOFSignal::condfn()调用到fn, 读完会发送eofc保证fn退出，之前说到的【标注点 3】会在pc.wroteRequest()中接收，如果之前写有error，会返回false，也就是读结果的操作没有必要进行, tryPutIdleConn也是一个closure，会调用Transport:: tryPutIdleConn(), 前面提到的putOrCloseIdleConn也是调用这个方法，尝试把链接放到连接池或idleConnCh，也就是这表明读完响应数据后把连接供后续使用。

我们来看看读取数据的过程:

```
func (es *bodyEOFSignal) Read(p []byte) (n int, err error) {
	 n, err = es.body.Read(p)
    
    if err != nil {
		 ......
        err = es.condfn(err)   //condfn会调用fn
    }
}
```
每次正常读取数据，最后err == io.EOF都会调用fn, 如果正常的话，会走到【标注点 4】，也会把连接放回连接池等待使用，alive也是true，readLoop继续执行，这时候不执行resp.Body.Close()实际并没有什么影响，可以去net/http/transfer.go中看看body::Close()实现。如果读取错误(!= io.EOF)

```
			  fn: func(err error) error {
                isEOF := err == io.EOF
                waitForBodyRead <- isEOF
                if isEOF {
                    <-eofc // see comment above eofc declaration
                } else if err != nil {
                		//只有调用cancelRequest或者context cancel了，否则canceled返回nil
                    if cerr := pc.canceled(); cerr != nil {
                        return cerr
                    }
                }
                return err
            },
```
condfn也调用fn返回错误，waitForBodyRead为false，这时候连接不会被重用了，连接中的定时器超时后会关闭底层连接(net.Conn), 这种情况下执行了resp.Body.Close()会走到下面代码【标注点 5】这时候执行es.earlyCloseFn()也没有任何意义了，毕竟readLoop()已经退出了。

综上，只要发生了读取动作，没有执行resp.Body.Close()，没有什么影响。Close的实现代码：

```
func (es *bodyEOFSignal) Close() error {
    es.mu.Lock()
    defer es.mu.Unlock()
    if es.closed {
        fmt.Println("es.closed")
        return nil
    }
    es.closed = true

    if es.earlyCloseFn != nil && es.rerr != io.EOF {
        return es.earlyCloseFn() 【标注点 5】
    }
    err := es.body.Close()
    return es.condfn(err)
}
```
**继续分析**：如果没有读取Body的行为，这里会走到es.earlyCloseFn()，readLoop会跳出，连接不会被re-use，如果读取过Body，也执行了Close(),正常读取（io.EOF)会走到es.body.Close()会返回的是nil，而不是io.EOF，condfn不会调用fn，读取Body的过程中已经调用了fn。在net/http/client.go中注释中有这样一句话，感觉是不对的：

```
470 // Body which the user is expected to close. If the Body is not
471 // closed, the Client's underlying RoundTripper (typically Transport)
472 // may not be able to re-use a persistent TCP connection to the server
473 // for a subsequent "keep-alive" request.
```

resp.Body.Close()的最后读取数据并discard body剩余的数据（顺便提一句：需要判断resp.Body是否是nil，有可能会panic）

```go
// in file: net/http/transfer.go
946		_, err = io.Copy(ioutil.Discard, bodyLocked{b})
```

resp.Body.Close()与是否re-use连接没有什么直接联系，欢迎argue，mail: genuinejyn@gmail.com。

### 2.2 roundTrip总结

![roundTrip总结](https://github.com/GenuineJyn/genuinejyn.github.io/blob/master/pictures/roundtrip.png)



## 3 重试逻辑
Transport.RoundTrip()最后提供了重试机制, 最后把这里说明一下，在实现网关的反向代理过程中，在RoundTrip外层加了重试逻辑，遇到了问题，顺便加以记录一下，在RoundTrip最后代码如下：

```
        if !pconn.shouldRetryRequest(req, err) {
            // Issue 16465: return underlying net.Conn.Read error from peek,
            // as we've historically done.
            if e, ok := err.(transportReadFromServerError); ok {
                err = e.err
            }
            return nil, err
        }
        testHookRoundTripRetried()

        // Rewind the body if we're able to.  (HTTP/2 does this itself so we only
        // need to do it for HTTP/1.1 connections.)
        if req.GetBody != nil && pconn.alt == nil {
            newReq := *req
            var err error
            newReq.Body, err = req.GetBody()
            if err != nil {
                return nil, err
            }
            req = &newReq
        }
```

大致过程：shuldRetryRequest判断是否要重试，调用GetBody重新构造请求，返回循环重新执行。
shuldRetryRequest中判断是否要对Request进行重试有很多分支，下面重点介绍几个，其他可以自己分析。

```
func (pc *persistConn) shouldRetryRequest(req *Request, err error) bool {
	 ......
	 if !pc.isReused() {	//链接被reuse， return false
	 	 return false
	 }

	......
	
	if !req.isReplayable() {
        // Don't retry non-idempotent requests.
        return false
    }
    if _, ok := err.(transportReadFromServerError); ok {
        // We got some non-EOF net.Conn.Read failure reading
        // the 1st response byte from the server.
        return true
    }
    ......
}
```
transportReadFromServerError错误是在【标注点 6】Peek操作可能产生的，Peek并不改变读取的位置(相当于文件句柄并没有偏移，操作的仅仅是reader，而不是操作系统的文件句柄），这种情况下会重试。
这里重点来看一下，isReplayable()这里有判断使用的判断, 在[net/http/httputil: ReverseProxy](https://github.com/golang/go/issues/16036)中介绍了利用反向代理向服务端发送请求的情况（Body != nil && ContentLength == 0), 因此在实现反向代理的时候需要注意这一点。

```
// in file: net/http/request.go
1309 func (r *Request) isReplayable() bool {
1310     if r.Body == nil || r.Body == NoBody || r.GetBody != nil {
1311         switch valueOrDefault(r.Method, "GET") {
1312         case "GET", "HEAD", "OPTIONS", "TRACE":
1313             return true
1314         }
1315     }
1316     return false
1317 }
```
当然走到这里并不是万事大吉了，还有个GetBody存在，在RoundTrip的最后代码：

```
		// Rewind the body if we're able to.  (HTTP/2 does this itself so we only
		// need to do it for HTTP/1.1 connections.)
        if req.GetBody != nil && pconn.alt == nil {
            newReq := *req
            var err error
            newReq.Body, err = req.GetBody()
            if err != nil {
                return nil, err
            }
            req = &newReq
        }
```
从代码上看要使用GetBody()重新构造请求，为什么要重新构造请求呢？
个人在实现网关的http反向代理过程中，在RoundTrip()实现重试，在第一次请求失败过程，第二次重新发送请求的时候，会返回这个错误：

```go
//definition in file: net/http/transfer.go
var ErrBodyReadAfterClose = errors.New("http: invalid Read on closed Body")
```
在Request定义中, Body是一个io.ReadCloser interface，Reqeust相关代码如下：

```
type Reqeust struct {
	 ......
    // Body is the request's body.
    //
    // For client requests a nil body means the request has no
    // body, such as a GET request. The HTTP Client's Transport
    // is responsible for calling the Close method.
    //
    // For server requests the Request Body is always non-nil
    // but will return EOF immediately when no body is present.
    // The Server will close the request body. The ServeHTTP
    // Handler does not need to.
    Body io.ReadCloser

    // GetBody defines an optional func to return a new copy of
    // Body. It is used for client requests when a redirect requires
    // reading the body more than once. Use of GetBody still
    // requires setting Body.
    //
    // For server requests it is unused.
    GetBody func() (io.ReadCloser, error)
    ......
}
```
**在构造并发送请求后，Body会被自动关闭，且只能被读取一次**。在RoundTrip中调用链如下(RoundTrip还有其他异常Close的地方，调用的是http.Request.closeBody()->Request.Body.Close(), 都是一致的代码)：

net/http.(\*persistConn).writeLoop() -> net/http.(\*Request).write() -> net/http.(*transferWriter).WriteBody()

```go
// in file: net/http/transfer.go
func (t *transferWriter) WriteBody(w io.Writer) error {

		 } else if t.ContentLength == -1 {
            ncopy, err = io.Copy(w, body)
        }
	......
	if t.BodyCloser != nil {
        if err := t.BodyCloser.Close(); err != nil {
            return err
        }
    }
    【标注点 8】
    if !t.ResponseToHEAD && t.ContentLength != -1 && t.ContentLength != ncopy {
        return fmt.Errorf("http: ContentLength=%d with Body length %d",
            t.ContentLength, ncopy)
    }
	 ......
}
```
读过之后会关闭，当然这里是多态的形式，Body的具体类型可能是http.body, http.noBody(GET等请求)或者使用ioutil.NopCloser包装的类型等，所以这里得到并且标准库中也明确说明：***`client的Request.Body会被关闭`***。
接下来看一下http.body的Read(), 毕竟是一个ReadCloser，也是Reader。

```
 764 func (b *body) Read(p []byte) (n int, err error) {
 765     b.mu.Lock()
 766     defer b.mu.Unlock()
 767     if b.closed {
 768         return 0, ErrBodyReadAfterClose 【标注点 7】
 769     }
 770     return b.readLocked(p)	【标注点 9】
 771 }
 
 774 func (b *body) readLocked(p []byte) (n int, err error) {
 775     if b.sawEOF {
 776         return 0, io.EOF
 777     }
 778     n, err = b.src.Read(p)
 779
 780     if err == io.EOF {
 781         b.sawEOF = true	// 不能重复读取
     		......
     }
 
```
所以，关闭之后再次读取就会走到【标注点 7】，这就说明了ErrBodyReadAfterClose的产生整个过程。

第一次看到这个错误，没有去深刻理解，参考[router: fix request retries with bodies](https://github.com/flynn/flynn/pull/875)把Request给wrap一下，Close()不做任何操作，自己手动调用RealClose()进行关闭，进而有出现如下的错误：

```
http: ContentLength=238 with Body length 0
```
产生错误代码参见【标注点 8】，虽然没有真正的关闭，跳过【标注点 7】，但是不能第二次读取了，参见【标注点 9】

另外仔细看了一下构建Request的代码如下，这里并没有全部深拷贝，Body是个interface，浅拷贝：

```
// in file: net/http/request.go
// WithContext returns a shallow copy of r with its context changed
// to ctx. The provided ctx must be non-nil.
func (r *Request) WithContext(ctx context.Context) *Request {
    if ctx == nil {
        panic("nil context")
    }
    r2 := new(Request)
    *r2 = *r
    r2.ctx = ctx

    // Deep copy the URL because it isn't
    // a map and the URL is mutable by users
    // of WithContext.
    if r.URL != nil {
        r2URL := new(url.URL)
        *r2URL = *r.URL
        r2.URL = r2URL
    }

    return r2
}
```

所以，要正确的实现重试：**RoundTrip的重试，正确的实现GetBody()；外围重试要正确构建Request**。
[router: fix request retries with bodies](https://github.com/flynn/flynn/pull/875)实现的内容在标准库有类似的：[ioutil.NopCloser](https://github.com/golang/go/blob/master/src/io/ioutil/ioutil.go#L118)正确的思路大致如下：

```
//bodyToBufer can be read repeatedly
bodyToBuffer := Read(http.Request) or buffer_repeatedly_readable

getBody := func() (io.ReadCloser, error) {
	return ioutil.NopCloser(strings.NewReader(body)), nil
}

bodyReader, _ := getBody()

/* 
req, err := http.NewRequest(xxxHTTPMethod, xxxUrl, bodyReader)
if err != nil {		
	return nil, errors.Wrap(err, "creating request")
}
Or other NewRequest Method
*/

req.GetBody = getBody		
```

代码看下来，思路比较清晰单细节很多，代码中有很多值得我们学习的地方：连接池，多goroutine协作，That's all。
