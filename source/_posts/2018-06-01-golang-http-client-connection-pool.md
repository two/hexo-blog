title: golang http client 连接池
date: 2018-06-01 10:44:03
tags: golang
---
>golang标准库`net/http`做为`client`时有哪些细节需要注意呢，这里做一个详细的分析。

## net/http client工作流程

首先分析一下`client`的工作流程。 下面是一般我们进行一个请求时的代码事例:

```go
func DoRequest(req *http.Request) (MyResponse, error) {
    client := &http.Client{}
    resp, err := client.Do(req)
    if resp != nil {
        defer resp.Body.Close()
    }
    if err != nil {
        return nil, err
    }
    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        return nil, err
    }

    response := MyResponse{}
    response.Header = resp.Header
    ...
    response.Body = body

    return response, nil
}
```
代码中我们首先创建一个`http.Client`, 所有的值都是默认值，然后调用`client.Do`发请求，`req`是我们请求的结构体。这里我们也可以用`client.Get`, `client.Post`等函数来调用，从他们的源码来看都是调用的`client.Do`。
`client.Do`的实现在`net/http`包的`go/src/net/http/client.go`源文件中。可以看到函数内部主要是实现了一些参数检查，默认值设置，以及对于多跳请求的处理，最为核心的就是:
```go
...
if resp, didTimeout, err = c.send(req, deadline); err != nil {
            // c.send() always closes req.Body
            reqBodyClosed = true
            if !deadline.IsZero() && didTimeout() {
                err = &httpError{
                    err:     err.Error() + " (Client.Timeout exceeded while awaiting headers)",
                    timeout: true,
                }
            }
            return nil, uerr(err)
        }

var shouldRedirect bool
redirectMethod, shouldRedirect, includeBody = redirectBehavior(req.Method, resp, reqs[0])
if !shouldRedirect {
    return resp, nil
}
...

```

这里真正发请求的函数就是`c.send`, 这个函数的实现也比较简单, 主要是调用了`send`函数，这个函数的实现主要如下:

```go
// didTimeout is non-nil only if err != nil.
func (c *Client) send(req *Request, deadline time.Time) (resp *Response, didTimeout func() bool, err error) {
    if c.Jar != nil {
        for _, cookie := range c.Jar.Cookies(req.URL) {
            req.AddCookie(cookie)
        }
    }
    resp, didTimeout, err = send(req, c.transport(), deadline)
    if err != nil {
        return nil, didTimeout, err
    }
    if c.Jar != nil {
        if rc := resp.Cookies(); len(rc) > 0 {
            c.Jar.SetCookies(req.URL, rc)
        }
    }
    return resp, nil, nil
}
```

```go
// send issues an HTTP request.
// Caller should close resp.Body when done reading from it.
func send(ireq *Request, rt RoundTripper, deadline time.Time) (resp *Response, didTimeout func() bool, err error) {
    ...
        stopTimer, didTimeout := setRequestCancel(req, rt, deadline)
    ...
        resp, err = rt.RoundTrip(req)
    ...
        return resp, nil, nil
}
```
这里真正进行网络交互的定位到的函数是`rt.RoundTrip`,这个函数的定义是一个`interface`，从其注释也可以看出他的主要作用是:
```
// RoundTrip executes a single HTTP transaction, returning
// a Response for the provided Request.`
```
由于这个函数是一个`interface`我们需要知道是谁实现了这个函数，看一下`send`的参数就可以找到，实现这个函数的是`c.transport()`的返回值，这个函数的实现如下:
```go
func (c *Client) transport() RoundTripper {
    if c.Transport != nil {
        return c.Transport
    }
    return DefaultTransport
}
```
这里可以看到，返回的对象是`c.Transport`或者`DefaultTransport`, 由于我们创建`client`的时候没有设置`c.Transport`参数，所以这里返回的应该是`DefaultTransport`对象, 这个对象对`RoundTripper`函数的实现大概如下:
```go
// RoundTrip implements the RoundTripper interface.
//
// For higher-level HTTP client support (such as handling of cookies
// and redirects), see Get, Post, and the Client type.
func (t *Transport) RoundTrip(req *Request) (*Response, error) {
    ...
        for {
            ...
                pconn, err := t.getConn(treq, cm)
                ...
                if pconn.alt != nil {
                    // HTTP/2 path.
                    t.setReqCanceler(req, nil) // not cancelable with CancelRequest
                        resp, err = pconn.alt.RoundTrip(req)
                } else {
                    resp, err = pconn.roundTrip(treq)
                }
        }
    ...
}
```
里面具体的细节我们先不关系，对于`HTTP/2`的处理我们也先不关心。这里需要重点关注的是`t.getConn`这个函数。`t.getConn`的作用是获取一个链接，这个链接该怎么获取，是一个值得深究的问题。下面看一下这个函数的关键实现细节:
```go
// getConn dials and creates a new persistConn to the target as
// specified in the connectMethod. This includes doing a proxy CONNECT
// and/or setting up TLS.  If this doesn't return an error, the persistConn
// is ready to write requests to.
func (t *Transport) getConn(treq *transportRequest, cm connectMethod) (*persistConn, error) {
req := treq.Request
         trace := treq.trace
         ctx := req.Context()
         if trace != nil && trace.GetConn != nil {
             trace.GetConn(cm.addr())
         }
     if pc, idleSince := t.getIdleConn(cm); pc != nil {
         if trace != nil && trace.GotConn != nil {
             trace.GotConn(pc.gotIdleConnTrace(idleSince))
         }
         // set request canceler to some non-nil function so we
         // can detect whether it was cleared between now and when
         // we enter roundTrip
         t.setReqCanceler(req, func(error) {})
             return pc, nil
     }
     ...
         handlePendingDial := func() {
             testHookPrePendingDial()
                 go func() {
                     if v := <-dialc; v.err == nil {
                         t.putOrCloseIdleConn(v.pc)
                     }
                     testHookPostPendingDial()
                 }()
         }

cancelc := make(chan error, 1)
             t.setReqCanceler(req, func(err error) { cancelc <- err })

             go func() {
                 pc, err := t.dialConn(ctx, cm)
                     dialc <- dialRes{pc, err}
             }()
idleConnCh := t.getIdleConnCh(cm)
                select {
                    case v := <-dialc:
                            // Our dial finished.
                            if v.pc != nil {
                                if trace != nil && trace.GotConn != nil && v.pc.alt == nil {
                                    trace.GotConn(httptrace.GotConnInfo{Conn: v.pc.conn})
                                }
                                return v.pc, nil
                            }
                            // Our dial failed. See why to return a nicer error
                            // value.
                            select {
                                case <-req.Cancel:
                                    // It was an error due to cancelation, so prioritize that
                                    // error value. (Issue 16049)
                                    return nil, errRequestCanceledConn
                                case <-req.Context().Done():
                                        return nil, req.Context().Err()
                                case err := <-cancelc:
                                          if err == errRequestCanceled {
                                              err = errRequestCanceledConn
                                          }
                                          return nil, err
                                default:
                                              // It wasn't an error due to cancelation, so
                                              // return the original error message:
                                              return nil, v.err
                            }
                    case pc := <-idleConnCh:
                             // Another request finished first and its net.Conn
                             // became available before our dial. Or somebody
                             // else's dial that they didn't use.
                             // But our dial is still going, so give it away
                             // when it finishes:
                             handlePendingDial()
                                 if trace != nil && trace.GotConn != nil {
                                     trace.GotConn(httptrace.GotConnInfo{Conn: pc.conn, Reused: pc.isReused()})
                                 }
                             return pc, nil
                    case <-req.Cancel:
                                 handlePendingDial()
                                     return nil, errRequestCanceledConn
                    case <-req.Context().Done():
                                     handlePendingDial()
                                         return nil, req.Context().Err()
                    case err := <-cancelc:
                              handlePendingDial()
                                  if err == errRequestCanceled {
                                      err = errRequestCanceledConn
                                  }
                              return nil, err
                }
}
```
下面是这个过程的流程图:
{% plantuml %}
start
if (从连接池获取链接) then (yes)
    :返回链接;
else (no)
  fork
	:创建新连接;
    :把链接放到管道dialc中;
    :从dialc中获取连接;
  fork again
	:从管道idleConnCh中获取连接刚刚释放的链接;
    :把新创建的连接放到连接池或者close;
  fork again
    :其他错误处理;
  end fork
  : 返回链接;
endif
stop
{% endplantuml %}
从上面可以看到，获取链接会优先从连接池中获取，如果连接池中没有可用的连接，则会创建一个连接或者从刚刚释放的连接中获取一个，这两个过程时同时进行的，谁先获取到连接就用谁的。
当新创建一个连接, 创建连接的函数定义如下:
```
func (t *Transport) dialConn(ctx context.Context, cm connectMethod) (*persistConn, error)
```
最后这个函数会通过goroutine调用两个函数:
```
go pconn.readLoop()
go pconn.writeLoop()
```
其中`readLoop`主要是读取从server返回的数据,`writeLoop`主要发送请求到server,在`readLoop`函数中有这么一段代码:
```
// Put the idle conn back into the pool before we send the response
// so if they process it quickly and make another request, they'll
// get this same conn. But we use the unbuffered channel 'rc'
// to guarantee that persistConn.roundTrip got out of its select
// potentially waiting for this persistConn to close.
// but after
alive = alive &&
!pc.sawEOF &&
    pc.wroteRequest() &&
tryPutIdleConn(trace)
```
这里可以看出，在处理完请求后，会立即把当前连接放到连接池中。

上面说到连接池，每个`client`的连接池结构是这样的:`idleConn   map[connectMethodKey][]*persistConn`。其中`connectMethodKey`的值就是`client`连接的server的`host`值, map的值是一个`*persistConn`类型的`slice`结构，这里就是存放连接的地方，`slice`的长度由`MaxIdleConnsPerHost`这个值指定的，当我们不设置这个值的时候就取默认的设置:`const DefaultMaxIdleConnsPerHost = 2`。

另外这里我们插一个知识点，对于HTTP协议，有一个header值"Connections", 这个值的作用就是`client`向`server`端发请求的时候，告诉`server`是否要保持连接。具体的可以参考[rfc2616](https://www.w3.org/Protocols/rfc2616/rfc2616-sec8.html)。 这个协议头的值有两种可能(参考[MDN文档](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Connection)):
```
Connection: keep-alive
Connection: close
```
当值为`keep-alive`时，`server`端会保持连接，一直到连接超时。当值为`close`时,`server`端会在传输完`response`后主动断掉`TCP`连接。在`HTTP/1.1`之前，这个值默认是`close`, 之后是默认`keep-alive`, 而`net/http`默认的协议是`HTTP/1.1`也就是默认`keep-alive`, 这个值可以通过`DisableKeepAlives`来设置。

从上面的介绍我们可以看出，`net/http`默认是连接复用的，对于每个server会默认的连接池大小是2。
接下来我们看一下连接是如何放进连接池的:
```go
func (t *Transport) putOrCloseIdleConn(pconn *persistConn) {
    if err := t.tryPutIdleConn(pconn); err != nil {
        pconn.close(err)
    }
}


// tryPutIdleConn adds pconn to the list of idle persistent connections awaiting
// a new request.
// If pconn is no longer needed or not in a good state, tryPutIdleConn returns
// an error explaining why it wasn't registered.
// tryPutIdleConn does not close pconn. Use putOrCloseIdleConn instead for that.
func (t *Transport) tryPutIdleConn(pconn *persistConn) error {
    if t.DisableKeepAlives || t.MaxIdleConnsPerHost < 0 {
        return errKeepAlivesDisabled
    }
    if pconn.isBroken() {
        return errConnBroken
    }
    if pconn.alt != nil {
        return errNotCachingH2Conn
    }
    pconn.markReused()
    key := pconn.cacheKey

    t.idleMu.Lock()
    defer t.idleMu.Unlock()
    waitingDialer := t.idleConnCh[key]
    select {
    case waitingDialer <- pconn:
        // We're done with this pconn and somebody else is
        // currently waiting for a conn of this type (they're
        // actively dialing, but this conn is ready
        // first). Chrome calls this socket late binding. See
        // https://insouciant.org/tech/connection-management-in-chromium/
        return nil
    default:
        if waitingDialer != nil {
            // They had populated this, but their dial won
            // first, so we can clean up this map entry.
            delete(t.idleConnCh, key)
        }
    }
    if t.wantIdle {
        return errWantIdle
    }
    if t.idleConn == nil {
        t.idleConn = make(map[connectMethodKey][]*persistConn)
    }
    idles := t.idleConn[key]
    if len(idles) >= t.maxIdleConnsPerHost() {
        return errTooManyIdleHost
    }
    for _, exist := range idles {
        if exist == pconn {
            log.Fatalf("dup idle pconn %p in freelist", pconn)
        }
    }
    t.idleConn[key] = append(idles, pconn)
    t.idleLRU.add(pconn)
    if t.MaxIdleConns != 0 && t.idleLRU.len() > t.MaxIdleConns {
        oldest := t.idleLRU.removeOldest()
        oldest.close(errTooManyIdle)
        t.removeIdleConnLocked(oldest)
    }
    if t.IdleConnTimeout > 0 {
        if pconn.idleTimer != nil {
            pconn.idleTimer.Reset(t.IdleConnTimeout)
        } else {
            pconn.idleTimer = time.AfterFunc(t.IdleConnTimeout, pconn.closeConnIfStillIdle)
        }
    }
    pconn.idleAt = time.Now()
    return nil
}
```
首先会尝试把连接放入到连接池中，如果不成功则`关闭连接`,大致流程如下:
{% plantuml %}
start
if (DisableKeepAlives == true || MaxIdleConnsPerHost < 0) then (yes)
    :关闭连接;
else (no)
    :创建一个管道waitingDialer用于接收立即释放的资源; 
    fork
        :优先把连接放入到waitingDialer中;
        :关闭连接;
        end
    fork again
        :default删除waitingDialer这个管道，防止阻塞;
    fork again
        :其它错误，关闭连接;
        end
    endfork
    if (连接池已满) then (yes)
    :关闭连接;
    end
    else (no)
    :放入连接池;
    endif
endif
end
{% endplantuml %}
如果`DisableKeepAlives`为`true`表示不使用连接复用，所以请求完后会把连接关掉，但是这里需要注意的是，同时发请求的时候我们会设置`Connections: close`, 所以`server`端发送完数据后就会自动断开，所以这种情况的连接其实是`server`端发起的。

## 长连接与短连接
前面我们已经讲过`net/http`默认使用`HTTP/1.1`协议，也就是默认发送`Connections: keep-alive`的头，让服务端保持连接，就是所谓的长连接。
再看`DefaultTransport`的值:
```
// DefaultTransport is the default implementation of Transport and is
// used by DefaultClient. It establishes network connections as needed
// and caches them for reuse by subsequent calls. It uses HTTP proxies
// as directed by the $HTTP_PROXY and $NO_PROXY (or $http_proxy and
// $no_proxy) environment variables.
var DefaultTransport RoundTripper = &Transport{
    Proxy: ProxyFromEnvironment, //代理使用
    DialContext: (&net.Dialer{
        Timeout:   30 * time.Second, //连接超时时间
        KeepAlive: 30 * time.Second, //连接保持超时时间
        DualStack: true, //
    }).DialContext,
    MaxIdleConns:          100, //client对与所有host最大空闲连接数总和
    IdleConnTimeout:       90 * time.Second, //空闲连接在连接池中的超时时间
    TLSHandshakeTimeout:   10 * time.Second, //TLS安全连接握手超时时间
    ExpectContinueTimeout: 1 * time.Second, //发送完请求到接收到响应头的超时时间
}
```
当我们使用`DefaultTransport`时，就是默认使用的长连接。但是默认的连接池`MaxIdleConns`为100， `MaxIdleConnsPerHost`为2，当超出这个范围时，客户端会主动关闭到连接。
如果我们想设置为短连接，有几种方法:
1. 设置`DisableKeepAlives = true`: 这时就会发送`Connections:close`给server端，在server端响应后就会主动关闭连接。
2. 设置`MaxIdleConnsPerHost < 0`: 当`MaxIdleConnsPerHost < 0`时，连接池是无法放置空闲连接的，所以无法复用,连接直接会在`client`端被关闭。

## Server端出现大量的`TIME_WAIT`

当我们在实际使用时，会发现`Server`端出现了大量的`TIME_WAIT`,要想深入了解其原因，我们首先先回顾一下`TCP`三次握手和四次分手的过程:
![](/assets/img/golang/tcp_3_handshake.png)
![](/assets/img/golang/tcp_4_handshake.png)
图中可以看出，`TIME_WAIT`只会出现在主动关闭连接的一方,也就是`server`端出现了大量的主动关闭行为。
默认我们是使用长连接的，只有在超时的情况下`server`端才会主动关闭连接。前面也讲到，如果超出连接池的部分就会在`client`端主动关闭连接，连接池的连接会复用，看着似乎没有什么问题。问题出在我们每次请求都会`new`一个新的`client`,这样每个`client`的连接池里的连接并没有得到复用，而且这时`client`也不会主动关闭这个连接，所以`server`端出现了大量的`keep-alive`但是没有请求的连接，就会主动发起关闭。

todo:补充tcpdump的分析结果

要解决这个问题以下几个方案:
1. `client`复用，也就是我们尽量复用`client`，来保证`client`连接池里面的连接得到复用，而减少出现超时关闭的情况。
2. 设置`MaxIdleConnsPerHost < 0`：这样每次请求后都会由`client`发起主动关闭连接的请求，`server`端就不会出现大量的`TIME_WAIT`
3. 修改`server`内核参数: 当出现大量的`TIME_WAIT`时危害就是导致`fd`不够用,无法处理新的请求。我们可以通过设置`/etc/sysctl.conf`文件中的
    ```
    #表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭  
    net.ipv4.tcp_tw_reuse = 1  
    #表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭  
    net.ipv4.tcp_tw_recycle = 1  
    ```
    达到快速回收和重用的效果，不影响其对新连接的处理。

另外需要注意的是，虽然`DisableKeepAlives = true`也能满足连接池中不放空闲连接，但是这时候会发送`Connections: close`，这时`server`端还是会主动关闭连接，导致大量的`TIME_WAIT`出现，所以这种方法行不通。

以上就是我总结的关于`net/http`中连接池相关的知识。


