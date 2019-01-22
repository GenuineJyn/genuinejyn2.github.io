首先，看一下Transport的数据结构

```
type Transport struct {
    idleMu     sync.Mutex
    wantIdle   bool                                // user has requested to close all idle conns 注：在CloseIdleConnections()会被置为true，关闭所有idleConn
    idleConn   map[connectMethodKey][]*persistConn // most recently used at end
    idleConnCh map[connectMethodKey]chan *persistConn
    idleLRU    connLRU

    reqMu       sync.Mutex
    reqCanceler map[*Request]func(error)
```

#获取链接
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
对于Block 3和Block 4比较好理解，request被取消或者超时，返回相关错误。
这里需要特别说明一下handlePendingDial，在select中的5个case中除了第一个外，其他的4个都需要调用它，`handlePendingDial`是一个Closure, 其中创建一个goroutine跟Block 1一样等待接收刚创建的连接，因为创建过程也在goroutine中完成，如果select多个case条件同时满足，会随机选取一个执行，例如走到Block 2搞到一个早前使用过的连接，如果有新的连接创建，不能浪费，也发送到idleConnCh中，供别人使用，逻辑实现在`t.putOrCloseIdleConn(v.pc)`中，接下来重点来讲解一下, 代码如下：

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
putOrCloseIdleConn首先判断是不是persistConn是不是中断状态，如果是则调用close关闭，否则根据cacheKey(connectMethodKey)获取t.idleConnCh[key], 把自己发送到idleConnCh，参见之前讲到的接收的Block 2地方，并且根据key把自己append到t.idleConn[key]slice尾部，t.idleLRU做调整，对于当前连接需要调整定时器超时时间

```
	 pconn.idleTimer.Reset(t.IdleConnTimeout)
```
如果首次则创建Timer

```
pconn.idleTimer = time.AfterFunc(t.IdleConnTimeout, pconn.closeConnIfStillIdle)
```
如果超时时间已到没有被更新，则连接会被关闭。


### 完成发送和接收通信过程 
resp, err = pconn.roundTrip(treq)

### 重试逻辑
