layout: title
title: golang的webserver是如何工作的
date: 2017-07-01 09:29:38
tags: golang 
---
> 我们知道golang实现一个webserver非常简单，但是其内部是如何工作的呢，我们深入探究一下其原理。

## 实现一个webserver服务
下面我们就用golang内置的服务实现一个简单的webserver:
```golang
package main

import (
        "fmt"
        "log"
        "net/http"
        "strings"
)

func sayhelloName(w http.ResponseWriter, r *http.Request) {
        r.ParseForm()       //解析参数，默认是不会解析的
        fmt.Println(r.Form) //这些信息是输出到服务器端的打印信息
        fmt.Println("path", r.URL.Path)
        fmt.Println("scheme", r.URL.Scheme)
        fmt.Println(r.Form["url_long"])
        for k, v := range r.Form {
                fmt.Println("key:", k)
                fmt.Println("val:", strings.Join(v, ""))
        }
        fmt.Fprintf(w, "Hello astaxie!") //这个写入到w的是输出到客户端的
}

func main() {
        http.HandleFunc("/", sayhelloName)       //设置访问的路由
        http.HandleFunc("/hello", sayhelloName)       //设置访问的路由
        err := http.ListenAndServe(":8090", nil) //设置监听的端口
        if err != nil {
                log.Fatal("ListenAndServe: ", err)
        }
}
```
我们可以通过`go run main.go`来开启Server服务, 当我们访问`http://localhost:8090/`或`http://localhost:8090/hello`都会得到`Hello astaxie!`, 也就是都执行了`sayhelloName`函数。
下面让我们来分析一下服务的代码:
首先我们从`main`函数入口进入程序执行，首先执行了`http.HandleFunc("/", sayhelloName)`和`http.HandleFunc("/hello", sayhelloName)`两个方法，这两个方法其实就是设置路由及其对应的处理函数。
然后执行`http.ListenAndServe(":8090", nil)`这个函数开始监听8090端口并把用户的请求根据之前设置的路由规则交给特定的函数进行处理。
下面我将针对这两个函数进行深入的分析。

##  http.HandleFunc
这个函数是`net/http`包中定义的, 第一个参数`pattern`是`string`类型，表示匹配的URL, 第二个参数`handler`这是个函数类型，表示一个处理函数。其定义在`net/http/server.go`中，第一如下:
```golang
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    DefaultServeMux.HandleFunc(pattern, handler)
}
```
这个函数调用了下面这个函数:
```golang
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    mux.Handle(pattern, HandlerFunc(handler))
}
```
`HandelrFunc`定义如下, 声明为一个函数类型, `HandlerFunc(handler)`就是把`handler`强制类型转化为`HandlerFunc`类型
```golang
type HandlerFunc func(ResponseWriter, *Request)
```
`mux.Handle`的定义如下: 
```golang
func (mux *ServeMux) Handle(pattern string, handler Handler) {
    mux.mu.Lock()
    defer mux.mu.Unlock()

    if pattern == "" {
        panic("http: invalid pattern " + pattern)
    }
    if handler == nil {
        panic("http: nil handler")
    }
    if mux.m[pattern].explicit {
        panic("http: multiple registrations for " + pattern)
    }

    if mux.m == nil {
        mux.m = make(map[string]muxEntry)
    }
    mux.m[pattern] = muxEntry{explicit: true, h: handler, pattern: pattern}

    if pattern[0] != '/' {
        mux.hosts = true
    }

    // Helpful behavior:
    // If pattern is /tree/, insert an implicit permanent redirect for /tree.
    // It can be overridden by an explicit registration.
    n := len(pattern)
    if n > 0 && pattern[n-1] == '/' && !mux.m[pattern[0:n-1]].explicit {
        // If pattern contains a host name, strip it and use remaining
        // path for redirect.
        path := pattern
        if pattern[0] != '/' {
            // In pattern, at least the last character is a '/', so
            // strings.Index can't be -1.
            path = pattern[strings.Index(pattern, "/"):]
        }
        url := &url.URL{Path: path}
        mux.m[pattern[0:n-1]] = muxEntry{h: RedirectHandler(url.String(), StatusMovedPermanently), pattern: pattern}
    }
}
```
可以看出这个函数会把`pattern`和`handler`的对应关系读存储到`mux.m`这个map里了，`mux`类型是`ServeMux`,其定义如下:
```golang
type ServeMux struct {
    mu    sync.RWMutex
    m     map[string]muxEntry
    hosts bool // whether any patterns contain hostnames
}
```
经过上面的处理后通过`http.HandleFunc`设置的`pattern`与`handler`的对应关系都被存储到了`DefaultServeMux`这个对象的`m`中。

##  http.ListenAndServe

这个函数也是在`net/http/server.go`中定义的，其定义如下:
```golang
func ListenAndServe(addr string, handler Handler) error {
    server := &Server{Addr: addr, Handler: handler}
    return server.ListenAndServe()
}
```
上面函数最终对调用到下面这个函数:
```golang
func (srv *Server) Serve(l net.Listener) error {
    defer l.Close()
    if fn := testHookServerServe; fn != nil {
        fn(srv, l)
    }
    var tempDelay time.Duration // how long to sleep on accept failure

    if err := srv.setupHTTP2_Serve(); err != nil {
        return err
    }

    srv.trackListener(l, true)
    defer srv.trackListener(l, false)

    baseCtx := context.Background() // base is always background, per Issue 16220
    ctx := context.WithValue(baseCtx, ServerContextKey, srv)
    ctx = context.WithValue(ctx, LocalAddrContextKey, l.Addr())
    for {
        rw, e := l.Accept()
        if e != nil {
            select {
            case <-srv.getDoneChan():
                return ErrServerClosed
            default:
            }
            if ne, ok := e.(net.Error); ok && ne.Temporary() {
                if tempDelay == 0 {
                    tempDelay = 5 * time.Millisecond
                } else {
                    tempDelay *= 2
                }
                if max := 1 * time.Second; tempDelay > max {
                    tempDelay = max
                }
                srv.logf("http: Accept error: %v; retrying in %v", e, tempDelay)
                time.Sleep(tempDelay)
                continue
            }
            return e
        }
        tempDelay = 0
        c := srv.newConn(rw)
        c.setState(c.rwc, StateNew) // before Serve can return
        go c.serve(ctx)
    }
}
```
这里通过一个for循环不停的接收请求`l.Accept()`来得到接收的请求，然后再通过`go c.serve(ctx)`进行请求的处理。这里用到了协程，也就是每个请求其实是由单独的协程进行处理的，这也是golang作为webserver高效的原因所在。`c.serve`函数中有一个`for`循环，会不断读取同一个请求的数据，直到出现问题或者正确读取完毕。读取完请求后会调用`serverHandler{c.server}.ServeHTTP(w, w.req)`这个函数来处理请求。这个函数定义如下:
```golang
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
    handler := sh.srv.Handler
    if handler == nil {
        handler = DefaultServeMux
    }
    if req.RequestURI == "*" && req.Method == "OPTIONS" {
        handler = globalOptionsHandler{}
    }
    handler.ServeHTTP(rw, req)
}
```
### 当 handler 为 nil:

可以看到当我们不在`ListenAndServe`中传递`handler`时，也就是`sh.srv.Handler = nil`时`hanlder=DefaultServeMux`，这个 `DefaultServeMux`正式我们前面通过`http.HandleFunc`来设置的。 下面调用了`hanlder.ServeHTTP`，这里也就是调用了`DefaultServeMux.ServeHTTP`, 这个函数定义如下:
```golang
// ServeHTTP dispatches the request to the handler whose
// pattern most closely matches the request URL.
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
    if r.RequestURI == "*" {
        if r.ProtoAtLeast(1, 1) {
            w.Header().Set("Connection", "close")
        }
        w.WriteHeader(StatusBadRequest)
        return
    }
    h, _ := mux.Handler(r)
    h.ServeHTTP(w, r)
}
```
这个函数中的`mux.Handler`从请求`r`中找到请求的URL然后在去`mux.m`的map结构中找到对应的映射关系从而得出`h`这个处理函数名。
由于上面说过`h`是转换为类型`HandlerFunc`, 这个类型定义的`ServeHTTP`函数如下:
```golang
// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
```
所以调用`h.ServeHTTP(w,r)`就等于调用`h(w,r)`，也就是我们调用我们自己的写的处理函数。
这些都完成后会执行收尾工作，并把得到的结构返回给请求用户。

### 当 handler 不为 nil:

这时调用`h.ServerHTTP(w,r)`其实就是调用自己传入的`handler`的`ServerHTTP`函数，例如web框架`revel`的源码`github.com/revel/cmd/harness/harness.go`中执行`revel run app`是就会执行下面的函数:
```golang
// Run the harness, which listens for requests and proxies them to the app
// server, which it runs and rebuilds as necessary.
func (h *Harness) Run() {
    var paths []string
    if revel.Config.BoolDefault("watch.gopath", false) {
        gopaths := filepath.SplitList(build.Default.GOPATH)
        paths = append(paths, gopaths...)
    }
    paths = append(paths, revel.CodePaths...)
    watcher = revel.NewWatcher()
    watcher.Listen(h, paths...)

    go func() {
        addr := fmt.Sprintf("%s:%d", revel.HTTPAddr, revel.HTTPPort)
        revel.INFO.Printf("Listening on %s", addr)

        var err error
        if revel.HTTPSsl {
            err = http.ListenAndServeTLS(
                addr,
                revel.HTTPSslCert,
                revel.HTTPSslKey,
                h)
        } else {
            err = http.ListenAndServe(addr, h)
        }
        if err != nil {
            revel.ERROR.Fatalln("Failed to start reverse proxy:", err)
        }
    }()

    // Kill the app on signal.
    ch := make(chan os.Signal)
    signal.Notify(ch, os.Interrupt, os.Kill)
    <-ch
    if h.app != nil {
        h.app.Kill()
    }
    os.Exit(1)
}
``` 
这里也调用了`http.ListenAndServe`但是第二个参数`hanlder`传入了`h`，所以最终会调用`h.ServerHTTP`函数, 这个函数`revel`中是这么实现的:

```golang
// ServeHTTP handles all requests.
// It checks for changes to app, rebuilds if necessary, and forwards the request.
func (h *Harness) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    ...
    // Reverse proxy the request.
    // (Need special code for websockets, courtesy of bradfitz)
    if strings.EqualFold(r.Header.Get("Upgrade"), "websocket") {
        proxyWebsocket(w, r, h.serverHost)
    } else {
        h.proxy.ServeHTTP(w, r)
    }
}
```


