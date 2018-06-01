title: golang http client
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
上面说到连接池，每个`client`的连接池结构是这样的:`idleConn   map[connectMethodKey][]*persistConn`。
