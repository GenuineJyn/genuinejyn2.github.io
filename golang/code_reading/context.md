Golang标准库源码阅读 - Context

Context由Google官方在1.7版本中引入，在Golang并发编程中被广泛使用，在处理请求时更加方便地在goroutines中传递request-scoped values, cancelation signals and deadlines，在golang使用过程中接触大量的context的使用，也踩过坑，从代码量上看去掉注释实现代码寥寥不多，似乎没有彻底明白context的设计精髓，广泛涉猎Context相关内容及源码阅读后加以整理以求能够合理和正确地使用context。

本文主要涉及如下几个方面内容:

1. Context源码实现；
2. 如何正确的使用Context；

# 1.Context源码实现

首先来看一下Context接口的定义，相关内容如下：

```
type Context interface {
	// Deadline returns the time when work done on behalf of this context
   	// should be canceled. Deadline returns ok==false when no deadline is
   	// set. Successive calls to Deadline return the same results.
	Deadline() (deadline time.Time, ok bool)
	
	// Done returns a channel that's closed when work done on behalf of this
    // context should be canceled. Done may return nil if this context can
    // never be canceled. Successive calls to Done return the same value.
    Done() <-chan struct{}
    
    Err() error
    Value(key interface{}) interface{}
}
```

简单说明一下所有方法：

* Done()返回一个receive-only的channel，扮演了cancelation signal，当channel被close的时候通知所有使用的functions或goroutines放弃当前工作并返回，需要特别注意：不要重复的调用Done(), channel被关闭后，每次调用Done(）都不会阻塞；
* Err()说明Context被cancel的原因；
* Deadline()返回deadline的时间，时间超时后Context会被cancel()；
* Value()可以让Context携带Request作用域内的数据以共享相关数据；

## 1.1 几类Context：emptyCtx/cancelCtx/timerCtx/valueCtx
### 1.1.1 emptyCtx
```
167 // An emptyCtx is never canceled, has no values, and has no deadline. It is not
168 // struct{}, since vars of this type must have distinct addresses.
169 type emptyCtx int
170 var (
171    background = new(emptyCtx)
172    todo       = new(emptyCtx)
173)
```
emptyCtx不携带任何values，没有dealine永远不会被cancel，在源码中创建了两个全局变量，其中context.Background() return background以及context.TODO() return todo
如果不确定使用那种Context，那就使用context.TODO(), 而context.Background()作为top-level Context for incoming
requests，被广泛用于main，初始化以及tests，由于我们的cancelCtx是一个树的结构，context.Background()是任何Context树的根节点，下面会详细描述。

### 1.1.2 cancelCtx
首先看一下cancelCtx的定义：

```
314 // A cancelCtx can be canceled. When canceled, it also cancels any children
315 // that implement canceler.
316 type cancelCtx struct {
317     Context
318
319     mu       sync.Mutex            // protects following fields
320     done     chan struct{}         // created lazily, closed by first cancel call
321     children map[canceler]struct{} // set to nil by the first cancel call
322     err      error                 // set to non-nil by the first cancel call
323 }
```
cancelCtx包含了Context用于指向父类，cancelCtx只实现了Done()和Err(), 其他Context接口的方法复用父类的，因此cancelCtx依然是一个Context，cancelCtx同时维持了自己所有canceler粒度的子cancelCtx映射，通过cancelCtx维持了Context的树结构，WithCancel的时候再详细介绍，canceler interface定义如下：

```
300 // A canceler is a context type that can be canceled directly. The
301 // implementations are *cancelCtx and *timerCtx.
302 type canceler interface {
303     cancel(removeFromParent bool, err error)
304     Done() <-chan struct{}
305 }
```
cancelCtx实现了canceler, cancel()的时候关闭done，同时调用了children维护所有的子canceler的cancel(),把children置为空，最后把自己开始的子树摘除（从父节点中删除）并把err设定为具体的值。

```
357     if c.done == nil {
358         c.done = closedchan
359     } else {
360         close(c.done)
361     }
```

***`需要注意:`***如果不调用cancelCtx::Done()并不对done做任何实际初始化【注释点1】:

```
325 func (c *cancelCtx) Done() <-chan struct{} {
326     c.mu.Lock()
327     if c.done == nil {
328         c.done = make(chan struct{})      <----  created lazy, initialization here ！！【注释点1】
329     }
330     d := c.done
331     c.mu.Unlock()
332     return d
333 }
```
在执行cancel()使用的是closedchan：

```
// closedchan is a reusable closed channel.
var closedchan = make(chan struct{})

func init() {
    close(closedchan)
}
```


### 1.1.3 timerCtx
timerCtx组合了cancelCtx，同时增加了一种特定的cancel的方式：添加了一个定时器用于超时后cancel。

```
type timerCtx struct {
    cancelCtx
    timer *time.Timer // Under cancelCtx.mu.

    deadline time.Time
}
```

### 1.1.4 valueCtx
valueCtx组合了Context，用于指向父类，同时维护了一个key-value pair。

```
type valueCtx struct {
	Context
	key, value interface{}
}
```

总结一下：
cancelCtx,timerCtx,valueCtx关系图如下：

![context](https://github.com/GenuineJyn/genuinejyn.github.io/blob/master/pictures/context.png)

cancelCtx组合了Context，用于指向父类，valueCtx组合了Context， timerCtx组合了cancelCtx，三者本质上都是Context，虽然自身只实现了部分Context接口的部分方法，cancelCtx和timerCtx实现了canceler。

顺便提一句：emptyCtx/cancelCtx/timerCtx/valueCtx均实现了fmt package中的Stringer接口：
```
type Stringer interface {
    String() string
}
```

## 1.2 WithCancel/WithDeadline/WithTimeour/WithValue

### 1.2.1 WithCancel

```
224 // WithCancel returns a copy of parent with a new Done channel. The returned
225 // context's Done channel is closed when the returned cancel function is called
226 // or when the parent context's Done channel is closed, whichever happens first.
227 //
228 // Canceling this context releases resources associated with it, so code should
229 // call cancel as soon as the operations running in this Context complete.
230 func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
231     c := newCancelCtx(parent)		<---- 创建cancelCtx，Context指向parent Context
232     propagateCancel(parent, &c)		<---- 更新父节点中canceler的children，canceler粒度的树结构更新完成
233     return &c, func() { c.cancel(true, Canceled) }  
234 }
```

#### propagateCancel(parent Context, child canceler)

propagateCancel在parent及往上（向根节点）更新canceler中的children，完成canceler维度的树结构。
```
func propagateCancel(parent Context, child canceler) {
    if parent.Done() == nil {
        return // parent is never canceled
    }
    if p, ok := parentCancelCtx(parent); ok {  <--- 从当前节点往上找，直到找到第一个cancelCtx
        p.mu.Lock()
        if p.err != nil {
            // parent has already been canceled
            child.cancel(false, p.err)
        } else {
            if p.children == nil {
                p.children = make(map[canceler]struct{})
            }
            p.children[child] = struct{}{}     <--- 确定CancelCtx树的父子关系
        }
        p.mu.Unlock()
    } else {								   <--- parent并不是cancelCtx或未组合cancelCtx的情况
        go func() {
            select {
            case <-parent.Done():
                child.cancel(false, parent.Err())
            case <-child.Done():
            }
        }()
    }
}
```
cancelCtx::cancel()会关闭done channel，并且调用canceler中的所有children(子树)的cancel(),并把自己从parent canceler中摘除。

#### parentCancelCtx()
parentCancelCtx从parent Context链中找直到找到一个cancelCtx，如果没有找到则返回(nil, false)

```
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
    for {
        switch c := parent.(type) {
        case *cancelCtx:
            return c, true
        case *timerCtx:
            return &c.cancelCtx, true
        case *valueCtx:
            parent = c.Context
        default:
            return nil, false
        }
    }
}
```
从WithCancel实现上看：cancelCtx维持了一颗canceler的树，通过parent Context向上找到canceler的父节点，并更新父节点的children，完成树的更新。


### 1.2.2 WithDeadline/WithTimeout
```
374 // WithDeadline returns a copy of the parent context with the deadline adjusted
375 // to be no later than d. If the parent's deadline is already earlier than d,
376 // WithDeadline(parent, d) is semantically equivalent to parent.
......
383 func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) 
```
WithDeadline与timerCtx是息息相关的，timerCtx组合了cancelCtx添加了time.Timer用于超时操作，timerCtx的cancel()相应的也增加了timer.Stop()的逻辑，WithDeadline的主要逻辑如下：

* 1.指定的超时时间不能晚于parent Context的Deadline(), 晚于parent Context等同于parent Context;
* 2.timerCtx组合了cancelCtx，类似WithCancel，加入了cancel树链
* 3.添加了定时器逻辑，超时后会调用c.cancel(true, DeadlineExceeded)

因此返回的context的Done()channel如下任何一个条件成立都会被关闭：
* 1）deadline超时;
* 2）返回的CancelFunc被调用;
* 3）parent Context的Done channel被关闭(会递归调用所有子canceler的cancel());

WithTimeout复用了WithDeadline:

```
450 func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
451     return WithDeadline(parent, time.Now().Add(timeout))
452 }
```

### 1.2.3 WithValue
```
454 // WithValue returns a copy of parent in which the value associated with key is
455 // val.
......
func WithValue(parent Context, key, val interface{}) Context
```
valueCtx组合了Context用于指向父类，增加了<key, value>, 因此WithValue会返回一个valueCtx，完成了对这两部分的初始化，要求key必须是可比较的([comparable](https://golang.org/ref/spec#Comparison_operators))，在调用valueCtx::Value(key interface{})的时候，判断key与内部的key是否相等，相当返回对应的value值，如果未匹配递归回溯到父类c.Context.Value(key), 即parent Context的Value(key)

### 1.2.4 举例说明
到这里Context的实现代码大致梳理了一下，举个简单的例子加以说明，代码如下。

```
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    root := context.Background()

    ctx1 := context.WithValue(root, "k1", 1111)
    ctx2, cancel1 := context.WithCancel(ctx1)
    defer cancel1()
    ctx3, cancel2 := context.WithTimeout(ctx2, 2*time.Second)
    defer cancel2()
    ctx4 := context.WithValue(ctx2, "k2", "22222")
    ctx5 := context.WithValue(ctx4, "k3", "33333")
    ctx6 := context.WithValue(ctx3, "k4", "4444")
    ctx7, cancel3 := context.WithDeadline(ctx4, time.Now().Add(4*time.Second))
    defer cancel3()
    ctx8, cancel4 := context.WithCancel(ctx6)
    defer cancel4()

    fmt.Println(ctx7.Value("k1"))
    fmt.Println(ctx6.Value("k4"))
    fmt.Println(ctx5.Value("k2"))

    fmt.Println(ctx7)

    go func() {
        select {
        case <-ctx7.Done():
            fmt.Println("ctx7 canceled")
        }
    }()

    go func() {
        select {
        // ctx8 canceled before ctx7, because ctx3 canceled
        case <-ctx8.Done():
            fmt.Println("ctx8 canceled")
        }
    }()

    time.Sleep(5 * time.Second)
}
```
输出结果：

```
1111
4444
22222
context.Background.WithValue("k1", 1111).WithCancel.WithValue("k2", "22222").WithDeadline(2019-04-22 18:43:02.812461612 +0800 CST m=+4.000247146 [3.999961683s])
ctx8 canceled
ctx7 canceled
```
举例大致维系了树的关系图如下，ctx3超时后cancel会把子树中的ctx8也cancel掉，ctx7等会往父类回溯。
![context_example](https://github.com/GenuineJyn/genuinejyn.github.io/blob/master/pictures/context_example.png)


# 2.如何正确的使用Context
从上面的源码可以看到Context的代码确实不多，那如何合理&正确的使用Context呢，在Context的doc中有部分忠告，同时也有Golang UK Conference 2017的相关Topic。[视频链接](https://www.youtube.com/watch?v=-_B5uQ4UGi0) [中文翻译](https://blog.lab99.org/post/golang-2017-10-27-video-how-to-correctly-use-package-context.html)

## 2.1 使用建议
总结如下：

* 不要在一个结构体中存储Context，把Context想象为一条河流流过你的程序，应该沿着调用栈或channel传递，理想情况下Context伴随request创建而创建，结束而消亡。也有例外情况，仅当做通知消息在channel中进行传递, 例如将其作为可选方式用在net/http.Request中。[issues/22602](https://github.com/golang/go/issues/22602) [issues/14660](https://github.com/golang/go/issues/14660)

```
func (r *Request) WithContext(ctx context.Context) *Request
```

```
// A message processes parameter and returns the result on responseChan.
// ctx is places in a struct, but this is ok to do.
type message struct {
	responseChan chan<- int
	parameter    string
	ctx          context.Context
}

func ProcessMessages(work <-chan message) {
	for job := range work {
		select {
		// If the context is finished, don't bother processing the
		// message.
		case <-job.ctx.Done():
			continue
		default:
		}
		// Assume this takes a long time to calculate
		hardToCalculate := len(job.parameter)
		select {
		case <-job.ctx.Done():
		case job.responseChan <- hardToCalculate:
		}
	}
}
```

* 如果有Context，Context应该作为第一个形参参数，通常命名为ctx，不要传递一个nil Context，不确定使用哪个Context，传递context.TODO。

* 任何阻塞或者耗时长的操作都应该具备被cancel而中断执行的能力，任何函数可能被阻塞或者需要很长时间来完成的，都应该有个 context.Context。另外，每一个RPC调用都应该有超时退出的能力，不仅仅是超时，还需要有能力去结束那些不再需要操作的行为,这是比较合理的API设计。

* 要养成关闭Context的习惯，建立之后立即defer cancel()。特别是对timerCtx（通过context.WithTimeout()/WithDeadline()创建的，内部使用time.AfterFunc，从而会导致context在计时器超时前都不会被垃圾回收，如果一个context被 GC而不是cancel了，那一般是你做错了。

* 为了防止树形结构中出现重复的键(向parent context回溯)，建议约束key的空间，比如使用私有类型，然后用GetXxx()和WithXxx()来操作私有实体，这里使用WithXxx而不是SetXxx，也是因为context.Context从设计上就是按照immutable模式设计的，所以不是修改Context里某个值，而是产生子Context（即valueCtx）。

```
type privateCtxType string

var (
  reqID = privateCtxType("req-id")
)

func GetRequestID(ctx context.Context) (int, bool) {
  id, exists := ctx.Value(reqID).(int)
  return id, exists
}

func WithRequestID(ctx context.Context, reqid int) context.Context {
  return context.WithValue(ctx, reqID, reqid)
}
```

* Context.Value也是immutable的，不要试图在Context.Value里存某个可变更的值，然后改变，期望别的Context可以看到这个改变；
更别指望着在Context.Value里存可变的值，最后多个goroutine并发访问没竞争冒险啥的，因为自始至终，Context就是按照不可变来设计的
比如设置了超时，就别以为可以改变这个设置的超时值，在使用Context.Value的时候，一定要记住这一点。

## 2.2 Context.Value


Context.Value存在很大的争议性，易被滥用，从官方注释来看：Use context Values only for request-scoped data，限定仅仅使用与request范围内的data，也就是说一个request-scoped value伴随request而存在的，视频及博客中也用大量篇幅介绍了Context.Value，大致如下几点：

* 1. Context.Value降低了程序的可读性，隐藏了期望的输入输出参数让接口定义更加模糊，如果你发现你的函数在某些Context.Value下无法正确工作，那就说明这个Context.Value里的信息不应该放在里面，而应该放在接口上。
* 2. Context.Value应该是告知性质而不是控制性质的东西（Inform, not control），数据库连接、认证服务等属于控制性质的内容，不要放到Context中，logger本身不是request-scoped也不要放在Context中。

```
func trace(req *http.Request, c *http.Client) {
  trace := &httptrace.ClientTrace{
    GotConn: func(connInfo httptrace.GotConnInfo) {
      fmt.Println("Got Conn")
    },
    ConnectStart: func(network, addr string) {
      fmt.Println("Dial Start")
    },
    ConnectDone: func(network, addr string, err error) {
      fmt.Println("Dial done")
    },
  }
  req = req.WithContext(httptrace.WithClientTrace(req.Context(), trace))
  c.Do(req)
}
```
http不会因为httptrace存在与否执行不同的逻辑，这里仅仅是告知性质的内容，用户记录日志或调试告知API用户，因此放在Context中。

* 3. Context.Value尽量不要使用。

# 3.曾经遇到的问题（To Be Continued）
## 3.1 不合理的cancel()导致的问题
### 3.1.1 代码抽象

```
func (cli *HttpClient) doRequest(ctx context.Context, req *http.Request) (*http.Response, error) {
	if _, ok := ctx.Deadline(); !ok && cli.timeout > 0 {
		var cancel context.CancelFunc
		ctx, cancel = context.WithTimeout(ctx, cli.timeout)  【标注点1】
		defer cancel()
	}
	req = req.WithContext(ctx)
	......
	
	resp, err := cli.Do(req, opts...)
	return resp, err
}

func (cli *HttpClient) HandyDo(ctx context.Context, req *http.Request, v interface{}) error {
	.....
	rsp, err := cli.doRequest(ctx, req)
	if err != nil {
		return err
	}

	b, err := ioutil.ReadAll(rsp.Body)
	
	......
	
	return json.Unmarshal(b, v)
```
### 3.1.2 现象描述
调用HandyDo()发送了一个请求，结果rsp.Body很大，然后发现err为context.Canceled，即errors.New("context canceled")。

### 3.1.3 原因解释
HandyDo()调用doRequest完成请求，然后对返回结果进行处理，首先发送请求是在doRequest中完成，随后defer cancel()，而HandyDo需要对rsp.Body()进行读取操作，这里存在并发的情况，http.Client底层实现是通过http.Transport实现的，其中[go pconn.readLoop()](https://github.com/golang/go/blob/master/src/net/http/transport.go#L1365)等待读取完成，[整体阻塞在这里](https://github.com/golang/go/blob/master/src/net/http/transport.go#L1817)，以下所有代码均出现在net/http/transport.go中。

```
		select {
		case bodyEOF := <-waitForBodyRead:    //****   【标注点1】正常读取结束
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
		case <-rc.req.Context().Done():	   //****  【标注点2】Request被cancel了 
			alive = false
			pc.t.cancelRequest(rc.req, rc.req.Context().Err())
		case <-pc.closech:
			alive = false
		}
```


resp.Body的实际类型是[\*bodyEOFSignal](https://github.com/GenuineJyn/genuinejyn.github.io/blob/master/golang/code_reading/http.transport.md#21-nethttpresponseclose), 而bodyEOFSignal中body的wrap了一个*http.body, 读取操作和Cancel可能同时完成，参考上述代码的【标注点1】和【标注点2】，到这里原因其实很明显了，等待读取完成的过程中，【标注点2】到来了，整个通信过程被cancel了，下面对cancel的整个过程加以详细说明，以加深对这个问题的理解。
看一下bodyEOFSignal的Read()

```
2176 func (es *bodyEOFSignal) Read(p []byte) (n int, err error) {
2177     es.mu.Lock()
2178     closed, rerr := es.closed, es.rerr
2179     es.mu.Unlock()
2180     if closed {
2181         return 0, errReadOnClosedResBody
2182     }
2183     if rerr != nil {
2184         return 0, rerr
2185     }
2186
2187     n, err = es.body.Read(p)
2188     if err != nil {
2189         es.mu.Lock()
2190         defer es.mu.Unlock()
2191         if es.rerr == nil {
2192             es.rerr = err
2193         }
2194         err = es.condfn(err)    //****  【标注点3】
2195     }
2196     return
2197 }
```
标注点3：

```
2213 // caller must hold es.mu.
2214 func (es *bodyEOFSignal) condfn(err error) error {
2215     if es.fn == nil {
2216         return err
2217     }
2218     err = es.fn(err) //**** 【标注点4】
2219     es.fn = nil
2220     return err
2221 }
```

标注点4：

```
1685             fn: func(err error) error {
1686                 isEOF := err == io.EOF
1687                 waitForBodyRead <- isEOF
1688                 if isEOF {
1689                     <-eofc // see comment above eofc declaration
1690                 } else if err != nil {
1691                     if cerr := pc.canceled(); cerr != nil {   //****  【标注点5】
1692                         return cerr
1693                     }
1694                 }
1695                 return err
1696             },
```

标注点5：

```
1469 // canceled returns non-nil if the connection was closed due to
1470 // CancelRequest or due to context cancelation.
1471 func (pc *persistConn) canceled() error {
1472     pc.mu.Lock()
1473     defer pc.mu.Unlock()
1474     return pc.canceledErr  //****【标注点6】
1475 }
```
到这里我们只需要看一下pc.canceledErr什么时候被赋值即可。
标注点6：

```
1497 func (pc *persistConn) cancelRequest(err error) {
1498     pc.mu.Lock()
1499     defer pc.mu.Unlock()
1500     pc.canceledErr = err
1501     pc.closeLocked(errRequestCanceled)
1502 }
```

```
1952 func (pc *persistConn) roundTrip(req *transportRequest) (resp *Response, err error) {
1953     testHookEnterRoundTrip()
1954     if !pc.t.replaceReqCanceler(req.Request, pc.cancelRequest) {  //**** 【标注点7】
1955         pc.t.putOrCloseIdleConn(pc)
1956         return nil, errRequestCanceled
1957     }
```

【标注点2】处pc.t.cancelRequest()实现如下，`t.reqCanceler `初始化在上述代码的【标注点7】

```
 562 // Cancel an in-flight request, recording the error value.
 563 func (t *Transport) cancelRequest(req *Request, err error) {
 564     t.reqMu.Lock()
 565     cancel := t.reqCanceler[req]
 566     delete(t.reqCanceler, req)
 567     t.reqMu.Unlock()
 568     if cancel != nil {
 569         cancel(err)
 570     }
 571 }
```
 
到此把整个cancel过程梳理清楚了，知道为什么`b, err := ioutil.ReadAll(rsp.Body)` err为“context canceled”的原因了，当然整个时候读取的数据b没办法保证读取完成了。

# References
[1] [https://blog.golang.org/context](https://blog.golang.org/context)

[2] [How to correctly use context.Context in Go 1.7](https://medium.com/@cep21/how-to-correctly-use-context-context-in-go-1-7-8f2c0fafdf39)

[3] [视频笔记：如何正确使用 Context - Jack Lindamood](https://blog.lab99.org/post/golang-2017-10-27-video-how-to-correctly-use-package-context.html#shi-pin-xin-xi)

[4] [Golang标准库源码阅读 - net/http.Transport](https://github.com/GenuineJyn/genuinejyn.github.io/blob/master/golang/code_reading/http.transport.md)

[5] [net/http/transport.go](https://github.com/golang/go/blob/master/src/net/http/transport.go)














