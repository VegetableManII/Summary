# HTTP包

# http.Server()

## 数据结构

```go
// 路由函数结构，是一个接口
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)  // 路由实现器
}
// HandleFunc类型为函数类型该类型实现ServeHTTP接口
type HandlerFunc func(ResponseWriter, *Request)
// http 路由器数据结构
type ServeMux struct {
    mu sync.RWMutex   //锁，由于请求涉及到并发处理，因此这里需要一个锁机制
    m  map[string]muxEntry  // 路由规则，一个string对应一个mux实体，这里的string就是注册的路由表达式
    hosts bool // 是否在任意的规则中带有host信息
}
type muxEntry struct {
    explicit bool   // 是否精确匹配
    h        Handler // 这个路由表达式对应哪个handler
    pattern  string  //匹配字符串
}

```

## HTTP服务启动逻辑流程

- 调用Http.HadnleFunc
  1. 调用DefaultServeMux的HandleFunc
  2. 调用DefaultServeMux的Handle
  3. 往DefaultServeMux的map[string]muxEntry中添加对应的路由实现
- 调用http.ListenAndServe()
  1. 实例化Server
  2. 调用Server的ListenAndServe()
  3. 调用net.Listen()监听端口
  4. 启动for循环读取Accept请求
  5. 每个请求实例化一个Conn，开启一个goroutine为其服务
  6. 读取每个请求的内容
  7. 判断handler是否为空，如果没有设置handler则默认为DefaultServeMux
  8. 调用handler的ServeHttp



# http.Client

## net/http.Transport

## Transport数据结构

- `idleMu`并发控制锁
- `connectMethodKey`=>`Conn` 根据不同方法对应不同的http连接，默认使用长连接不进行tcp连接复用

```go
type Transport struct {
	idleMu     sync.Mutex
	idleConn   map[connectMethodKey][]*persistConn // most recently used at end，保存从connectMethodKey(代表着不同的协议，不同的host，也就是不同的请求)到persistConn的映射
  ......
}
```

## 方法

### getConn获取/新建连接=>从连接池获取连接调用getIdleConn()

#### persistConn表示一个连接对象

```go
type persistConn struct {
    t        *Transport
    cacheKey connectMethodKey
    conn     net.Conn
    tlsState *tls.ConnectionState
    br       *bufio.Reader       // 从tcp输出流里面读
    sawEOF   bool                // whether we've seen EOF from conn; owned by readLoop
    bw       *bufio.Writer       // 写到tcp输入流
    reqch    chan requestAndChan // 主goroutine 往channnel里面写，读循环从     
                                 // channnel里面接受
    writech  chan writeRequest   // 主goroutine 往channnel里面写                                      
                                 // 写循环从channel里面接受
    closech  chan struct{}       // 通知关闭tcp连接的channel 
    writeErrCh chan error
  	lk                   sync.Mutex // guards following fields
		numExpectedResponses int
		closed               bool // whether conn has been closed
		broken               bool // an error has happened on this connection; marked broken so it's not reused.
		canceled             bool // whether this conn was broken due a CancelRequest
		// mutateHeaderFunc is an optional func to modify extra
		// headers on each outbound request before it's written. (the
		// original Request given to RoundTrip is not modified)
		mutateHeaderFunc func(Header)
}
```

#### getConn()

```go
func (t *Transport) getConn(req *Request, cm connectMethod) (*persistConn, error) {
    if pc := t.getIdleConn(cm); pc != nil {
        // set request canceler to some non-nil function so we
        // can detect whether it was cleared between now and when
        // we enter roundTrip
        t.setReqCanceler(req, func() {})
        return pc, nil
    }

		type dialRes struct {
    		pc  *persistConn
    		err error
		}
		dialc := make(chan dialRes)
		//定义了一个发送 persistConn的channel

		prePendingDial := prePendingDial
		postPendingDial := postPendingDial

		handlePendingDial := func() {
    		if prePendingDial != nil {
        		prePendingDial()
    		}
    go func() {
        if v := <-dialc; v.err == nil {
            t.putIdleConn(v.pc)
        }
        if postPendingDial != nil {
            postPendingDial()
        }
    		}()
		}

		cancelc := make(chan struct{})
		t.setReqCanceler(req, func() { close(cancelc) })

		// 启动了一个goroutine, 这个goroutine 获取里面调用dialConn搞到
		// persistConn, 然后发送到上面建立的channel  dialc里面，    
		go func() {
    		pc, err := t.dialConn(cm)
    		dialc <- dialRes{pc, err}
		}()

idleConnCh := t.getIdleConnCh(cm)
select {
		case v := <-dialc:
  		  // dialc 我们的 dial 方法先搞到通过 dialc通道发过来了
    		return v.pc, v.err
		case pc := <-idleConnCh:
    		// 这里代表其他的http请求用完了归还的persistConn通过idleConnCh这个    
    		// channel发送来的
    		handlePendingDial()
    		return pc, nil
		case <-req.Cancel:
    		handlePendingDial()
    		return nil, errors.New("net/http: request canceled while waiting for connection")
		case <-cancelc:
    		handlePendingDial()
    		return nil, errors.New("net/http: request canceled while waiting for connection")
		}
}
```

#### getIdleConn()

- `getIdleConn()`获取已经建立并且空闲的连接，只有一条空连接时直接返回，2条及以上时返回最近使用的连接，如果该连接的定时器超时或者该连接已经中断则从新查找空闲的连接

```go
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

- 如果找不到空闲连接则启动一个goroutine创建一个新的连接，在使用时使用`DialContext`来创建。新建立的连接的输入流和输出流都会为它启动读写循环两个协程。

### 核心方法routerRip

- 主goroutine=>`requestAndChan`=>读循环协程
- 主goroutine=>`writeRequest`=>写循环协程
- 主goroutine监听所有所有channel，请求取消

```go
func (pc *persistConn) roundTrip(req *transportRequest) (resp *Response, err error) {
    ... 忽略
    // Write the request concurrently with waiting for a response,
    // in case the server decides to reply before reading our full
    // request body.
    writeErrCh := make(chan error, 1)
    pc.writech <- writeRequest{req, writeErrCh}
    //把request发送给写循环
    resc := make(chan responseAndError, 1)
    pc.reqch <- requestAndChan{req.Request, resc, requestedGzip}
    //发送给读循环
    var re responseAndError
    var respHeaderTimer <-chan time.Time
    cancelChan := req.Request.Cancel
WaitResponse:
    for {
        select {
        case err := <-writeErrCh:
            if isNetWriteError(err) {
                //写循环通过这个channel报告错误
                select {
                case re = <-resc:
                    pc.close()
                    break WaitResponse
                case <-time.After(50 * time.Millisecond):
                    // Fall through.
                }
            }
            if err != nil {
                re = responseAndError{nil, err}
                pc.close()
                break WaitResponse
            }
            if d := pc.t.ResponseHeaderTimeout; d > 0 {
                timer := time.NewTimer(d)
                defer timer.Stop() // prevent leaks
                respHeaderTimer = timer.C
            }
        case <-pc.closech:
            // 如果长连接挂了， 这里的channel有数据， 进入这个case, 进行处理
            

            select {
            case re = <-resc:
                if fn := testHookPersistConnClosedGotRes; fn != nil {
                    fn()
                }
            default:
                re = responseAndError{err: errClosed}
                if pc.isCanceled() {
                    re = responseAndError{err: errRequestCanceled}
                }
            }
            break WaitResponse
        case <-respHeaderTimer:
            pc.close()
            re = responseAndError{err: errTimeout}
            break WaitResponse
            // 如果timeout，这里的channel有数据， break掉for循环
        case re = <-resc:
            break WaitResponse
           // 获取到读循环的response, break掉 for循环
        case <-cancelChan:
            pc.t.CancelRequest(req.Request)
            cancelChan = nil
        }
    }
    
    if re.err != nil {
        pc.t.setReqCanceler(req.Request, nil)
    }
    return re.res, re.err

}
```

