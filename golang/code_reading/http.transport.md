# Golang标准库源码阅读 - http.Transport #

最近实现网关的时候采用了http.Transport来实现了http协议反向代理，踩了一些坑，浪费了一些时间解决问题，最后下定决心要把源码好好看一下，并把遇到的问题及解决办法都加以记录一下。
在使用net/http库发送http请求，无论是Client::Do等最后都会调用Transport的RoundTrip方法，Transport实现了RoundTripper接口，本文档会对Transport::RoundTrip做较详细的理解说明。

```
type RoundTripper interface {
    RoundTrip(*Request) (*Response, error)
}
```

首先，看一下Transport的数据结构

```
type Transport struct {
    idleMu     sync.Mutex
    //注：在CloseIdleConnections()时，wantIdle会被置为true，关闭所有idleConn
    wantIdle   bool                                // user has requested to close all idle conns 
    idleConn   map[connectMethodKey][]*persistConn // most recently used at end
    idleConnCh map[connectMethodKey]chan *persistConn
    idleLRU    connLRU

    reqMu       sync.Mutex
    reqCanceler map[*Request]func(error)
```

## 获取链接
`getConn` 根据connectMethod获取或者创建一个*persistConn

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

getIdleConn函数for死循环内根据connectMethod生成key获取t.idleConn满足条件的[]*persistConn，只有一条空闲链接的时候返回这一条链接，超过2条及以上，选取最近使用的一条（most recently used at end）
选取之后并且返回使用之前还要检查这个链接是否已经中断了以及这条空闲链接中的定时器是否已经超时，如中断或者超时，continue重新在空闲连接池内再找一条。
如果getIdleConn获取到一个空闲链接，那么getConn函数将返回这个链接以供外面使用。如果没有获取到合适的空闲链接，那么接下来就需要创建一条新的链接。

###启动一个goroutine创建一个新的连接

```
    go func() {
        pc, err := t.dialConn(ctx, cm)
        dialc <- dialRes{pc, err}
    }()
```
t.dialConn中调用Transport::dial()，网络连接通过Dialer.DialContext创建，如果scheme为https，还会有TLS的一些初始化，如果是代理的形式还会有相关的操作，这里不做过多介绍。在dialConn最后有如下代码

```
    pconn.br = bufio.NewReader(pconn)
    pconn.bw = bufio.NewWriter(persistConnWriter{pconn})
    go pconn.readLoop()
    go pconn.writeLoop()
```
readLoop主要是读取从server返回的数据,writeLoop主要发送请求到server, 后续再详细介绍。
创建后的连接发送到channel dialc中，等待接收。

根据connectMethod获取对应的idleConnCh，如果不存在则创建一个空的idleConnCh，需要特别强调：**`DisableKeepAlives被置为true，idleConnCh逻辑都不生效`**。

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
这里需要特别说明一下handlePendingDial，在select中的5个case中除了第一个外，其他的4个都需要调用它，`handlePendingDial`是一个Closure, 其中创建一个goroutine跟Block 1一样等待接收刚创建的连接，因为创建过程也在goroutine中完成，如果select多个case条件同时满足，会随机选取一个执行，例如走到Block 2搞到一个早前使用过的连接，如果有新的连接创建，不能浪费，也发送到idleConnCh中或者放到连接池，供别人使用，逻辑实现在`t.putOrCloseIdleConn(v.pc)`中，接下来重点来讲解一下, 代码如下：

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
putOrCloseIdleConn首先判断是不是persistConn是不是中断状态，如果是则调用close关闭，否则根据cacheKey(connectMethodKey)获取t.idleConnCh[key], 把自己发送到idleConnCh，参见之前讲到的接收的Block 2地方，如果idleConnCh被占用，则根据key把自己append到t.idleConn[key]slice尾部，t.idleLRU做调整，对于当前连接需要调整定时器超时时间，

```
pconn.idleTimer.Reset(t.IdleConnTimeout)
```
如果首次则创建Timer，

```
pconn.idleTimer = time.AfterFunc(t.IdleConnTimeout, pconn.closeConnIfStillIdle)
```
如果超时时间已到没有被更新，则连接会被关闭。

整个获取链接的过程大致如下图所示。



## 完成发送和接收通信过程 
发送请求，接收响应在获取的连接上完成, 如下。在前面讲过创建每一个连接的时候同时启动了两个goroutine：readLoop和writeLoop，需要跟roundTrip通过管道配合完成。

```
resp, err = pconn.roundTrip(treq)
```
接下来看一下roundTrip内部的关键实现。

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
	 err = pc.bw.Flush()
	 ......
	 pc.writeErrCh <- err 【标注点 3】
	 wr.ch <- err
```

连接通过bufio.Writer.Flush()将请求写到连接发送给服务端，`wr.ch <- err`将错误或nil反馈给roundTrip, `pc.writeErrCh <- err`哪里接收后面会讲到，这里可以看到roundTrip->writeLoop()->remote_server完成全双工的写的过程, 跟随wc.ch回到roundTrip中继续看, 在RoundTrip最后是一个for死循环套着一个select语句，select中接收各种channel的信息，并作出相应的响应，对于wc.ch对应的是select中的`case err := <-writeErrCh`,如果err != nil关闭连接返回适当的错误信息。

继续, readLoop()主要是从服务端读回response，主要逻辑如下：

```
for alive {
	...... 
	// 试探性的读一个字节
	_, err := pc.br.Peek(1)
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
### 为什么要关闭resp.Close()

[stackoverflow: What could happen if I don't close response.Body in golang?
](https://stackoverflow.com/questions/33238518/what-could-happen-if-i-dont-close-response-body-in-golang)

stackoverflow上的这个问题有比较概括性的回答, 但是也没有说明白，这里做一点代码级别的说明，回到上边讲到的【标注点 1】具体来看看，代码如下：

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
可以看出resp.Body的实际类型是bodyEOFSignal指针, 而bodyEOFSignal中body的wrap了一个http.body的指针，在bodyEOFSignal有两个Closure：earlyCloseFn/fn, 其中都会发送消息到waitForBodyRead之后都会阻塞在eofc，而eofc是defer close(eofc)只有等退出readLoop()才会被执行到，或者在下面的case中被发送，再看看【标注点 2】代码

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
bodyEOFSignal::Read中会调用到bodyEOFSignal::condfn()调用到fn, 读完会发送eofc保证fn退出，之前说到的【标注点 3】也会在pc.wroteRequest()中接收，如果之前写有error，会返回false，也就是读结果的操作没有比较进行, tryPutIdleConn也是一个closure，会调用Transport:: tryPutIdleConn(), 前面提到的putOrCloseIdleConn也是代用这个方法，尝试把链接放到连接池，也就是这表明读完响应数据后把连接释放供后续使用。

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
每次正常读取数据，最后err == io.EOF都会调用fn, 如果正常的话，会走到【标注点 4】，也会把连接放回连接池等待使用，alive也是true，readLoop继续执行，这时候不关闭Close实际并没有什么影响，可以去net/http/transfer.go中看看body::Close()实现。如果读取错误(!= io.EOF)

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
condfn也调用fn返回错误，waitForBodyRead为false，这时候连接不会被重用了，连接中的定时器超时后会关闭底层连接(net.Conn)

综上，只要发生了读取动作，没有执行resp.Body.Close()，没有什么影响。那就来看看Close的实现代码：

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
        return es.earlyCloseFn()
    }
    err := es.body.Close()
    return es.condfn(err)
}
```
如果没有读取Body的行为，这里会走到es.earlyCloseFn()，readLoop会跳出，连接不会被re-use，如果读取过Body，也执行了Close(), es.body.Close()会返回的是nil，而不是io.EOF，condfn不会调用fn，读取Body的过程中已经调用了fn。在net/http/client.go中注释中有这样一句话，感觉是不对的：

```
470 // Body which the user is expected to close. If the Body is not
471 // closed, the Client's underlying RoundTripper (typically Transport)
472 // may not be able to re-use a persistent TCP connection to the server
473 // for a subsequent "keep-alive" request.
```
## 重试逻辑
