title: go context
date: 2019-11-25 18:42:54
tags: [Go]
---

# Contex 的作用
在 Go 服务器中, 每个请求都是由一个独立的 goroutine 进行处理的。请求处理程序
往往会启动其它的 goroutine 来访问后端，比如数据库和 RPC 服务。 处理请求的 
goroutine 通常需要访问特定的值，比如用户身份的标识，token, 请求的超时时间。
当一个请求被取消或者超时时，处理改请求的所有 goroutine 都应该迅速退出，这样
系统就可以回收他们正在使用的资源。

对于 Go 语言，由于是单进程的模式 goroutine 之间内存是共享的，那么 goroutine 是
如何获取自己的上下文数据的呢？对于一些多线程模式运行的语言中，比如 Java  可以
通过 ThreadLocal 来传递线程间的上下文，但是 Go 语言并不提倡这种模式，Go 语言中
你甚至无法知道 goroutine 的编号，一切都是 Go 自己帮你管理的。为了解决这个问题
Go 使用的就是传递 Context 参数。

这种方式是 Go 语言比较特殊的地方，也是很多人诟病的地方，如果你要传递上下文正规
的方式就是这种，Go 语言甚至规定了它的具体用法：
1. 不要把它放到一个结构体中, 而是在需要的地方直接传递它
2. 放到函数的第一个参数中，并且命名为 ctx

对于第一个限制，后面会详细讲解其原因。

# Context 结构
Context 本质上是为了传递上下文，这个上下文不止是一些变量，还包括传递事件。
下面我们给一个使用的例子：

```go
package main

import (
        "context"
)

func main() {
        ctx1 := context.Background()

        ctx2, _ := context.WithCancel(ctx1)
        ctx3, _ := context.WithCancel(ctx1)

        ctx4, _ := context.WithCancel(ctx2)
        ctx5, _ := context.WithCancel(ctx2)

        ctx6, _ := context.WithCancel(ctx3)
        ctx7, _ := context.WithCancel(ctx3)

        ctx8, _ := context.WithCancel(ctx4)
        ctx9, _ := context.WithCancel(ctx4)

        ctx10, _ := context.WithCancel(ctx5)
        ctx11, _ := context.WithCancel(ctx5)

        ctx12, _ := context.WithCancel(ctx6)
        ctx13, _ := context.WithCancel(ctx6)

        ctx14, _ := context.WithCancel(ctx7)
        ctx15, _ := context.WithCancel(ctx7)

        println(ctx8)
        println(ctx9)
        println(ctx10)
        println(ctx11)
        println(ctx12)
        println(ctx13)
        println(ctx14)
        println(ctx15)
}
```
对于前面这个例子，最终会形成一个 Context 的树形结构，结构如下：

![](/assets/img/go/context/context_tree.svg)

## 根节点

前面的结构中 `ctx1` 是根节点, 根节点通过 `context.Background()`  函数创建的，
函数源码如下: 

```go
var (
        background = new(emptyCtx)
        todo       = new(emptyCtx)
)

func Background() Context {
        return background
}
```
可以看到根节点是一个 `emptyCtx` 类型的数据,  这个结构实现了 `Context` 接口:

```go
type Context interface {
 Deadline() (deadline time.Time, ok bool)
 Done() <-chan struct{}
 Err() error
 Value(key interface{}) interface{}
}

type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
        return
}

func (*emptyCtx) Done() <-chan struct{} {
        return nil
}

func (*emptyCtx) Err() error {
        return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
        return nil
}
```
 可以看到 `emptyCtx` 实现很简单，基本都是返回 `nil`。对于根节点来说它并不能够
真正的传递一些信息和事件，其它的 context 则是依赖这个作为根节点来实现的。

## cancelCtx 

### 树的建立
cancelCtx 是一个可以传递 cancel 事件的 context， 通过 `WithCancel` 函数可以获取
一个 cancelCtx 类型的结构，并且还会返回它对应的 cancel 函数，当我们调用这个函数
时就会把事件传递到这个结构及他的所有子节点。源码实现如下：

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
        c := newCancelCtx(parent) // 新建一个 cancelCtx
        propagateCancel(parent, &c) // 把当前新节点放到 parent 的子节点中
        return &c, func() { c.cancel(true, Canceled) } // 返回 cancel 函数
}

// newCancelCtx returns an initialized cancelCtx.
func newCancelCtx(parent Context) cancelCtx {
        return cancelCtx{Context: parent}
}
```
通过 `newCancelCtx` 函数我们把当前节点的 `Context` 字段指向了 `parent`, 也就是父节点。

通过调用 `propagateCancel` 函数我们可以把当前新建的节点放到对应的 context 树中，

```go

// propagateCancel arranges for child to be canceled when parent is.
func propagateCancel(parent Context, child canceler) {
        if parent.Done() == nil { // 对于 parent.Done() == nil 直接返回，
                                  // 因为这个节点是空节点, 没有 children 字段
                return // parent is never canceled
        }
        if p, ok := parentCancelCtx(parent); ok {
                p.mu.Lock()
                if p.err != nil {
                        // parent has already been canceled
                        child.cancel(false, p.err) // 如果父节点存在错误信息，
                                                   // 证明父节点已经被取消，那么子节点也应该取消
                } else {
                        if p.children == nil {
                                p.children = make(map[canceler]struct{})
                        }
                        p.children[child] = struct{}{} // 放入子节点
                }
                p.mu.Unlock()
        } else {
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

对于根节点，执行到下面这里就会返回:

```go
        if parent.Done() == nil {
                return // parent is never canceled
        }
```

也就是根节点是没有 `children` 字段的, 无法通过根节点查找子节点, 但是子节点
可以通过 `Context` 字段找到父节点。

我们再看另一种情况 `ctx4` 是如何挂载到 `ctx2`。前面的操作基本都一致，但是会走到
下面这段逻辑:

```go
        if p, ok := parentCancelCtx(parent); ok {
                p.mu.Lock()
                if p.err != nil {
                        // parent has already been canceled
                        child.cancel(false, p.err)
                } else {
                        if p.children == nil {
                                p.children = make(map[canceler]struct{})
                        }
                        p.children[child] = struct{}{}
                }
                p.mu.Unlock()
        } else {
            ...
        }
```
首先调用 `parentCancelCtx` 函数判断父节点的类型:

```go
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

对于 `cancelCtx`, `timerCtx` 这些类型会返回 true, 然后判断父节点是否已经有错误信息，
如果有错误信息表示父节点已经调用了 cancel, 那么为了传播这个事件，子节点也应该调用
cancel, 对于没有 cancel 的父节点则把当前节点放到父节点的 children 结构中。

如果 `parentCancelCtx` 返回 false 呢？也就是不属于前面几种类型。这个跟前面解释的 context
的使用原则: "1. 不要把它放到一个结构体中, 而是在需要的地方直接传递它" 相关的，也就是当我们
把 context 放到结构体中进行传递则会满足这个条件，走到下面的逻辑: 

```go
                go func() {
                        select {
                        case <-parent.Done():
                                child.cancel(false, parent.Err())
                        case <-child.Done():
                        }
                }()
```

会开起一个新的 goroutine 来监听这个节点，而不会放到树中。 可以看出其实是监听了父节点和
本身节点, 因为如果父节点 cancel 了，子节点也需要 cancel ，因为父节点的事件要传播到子节点;
本身节点也是需要监听，调用 cancel 后也要结束这个 goroutine，如果不监听则需要依赖父节点，
如果父节点不接受这个节点即使调用了 cancel 也无法结束，所以两者缺一不可。

### 事件传递

前面说 `WithCancel` 函数会返回一个 `cancel` 函数，如果我们调用的话会传递这个消息到所有的
子节点中。 我们修改一下前面的代码:

```go
...
ctx4, cancel4 := context.WithCancel(ctx2)
cancel4()
...
```
当我调用 `cancel4()`, 其实调用的是 `cancel(true, err)`, 第一个参数传 true 表示需要从树中
删除其子节点，第二个参数传取消的错误信息。 具体实现如下：

```go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
        if err == nil {
                panic("context: internal error: missing cancel error")
        }
        c.mu.Lock()
        if c.err != nil {
                c.mu.Unlock()
                return // already canceled
        }
        c.err = err
        // 通过 chan 传递给所有监听的程序
        if c.done == nil {
                c.done = closedchan
        } else {
                close(c.done)
        }
        // 递归传递消息给所有子节点
        for child := range c.children {
                // NOTE: acquiring the child's lock while holding parent's lock.
                child.cancel(false, err)
        }
        c.children = nil
        c.mu.Unlock()

        // 从树中删除子节点
        if removeFromParent {
                removeChild(c.Context, c)
        }
}
```
`cancel` 函数主要是有三个作用:
1.  通过 close(chan) 传递给所有的监听的程序这个消息
    监听的程序如下：

    ```go
    func (c *cancelCtx) Done() <-chan struct{} {
            c.mu.Lock()
            if c.done == nil {
                    c.done = make(chan struct{})
            }
            d := c.done
            c.mu.Unlock()
            return d
    }
    ```
2. 消息传递个所有子节点，所有监听子节点的程序也收到消息
3. 把当前子节点机器及其下面的所有节点从树中删除

前面可以看出消息传递给子节点的时候调用了 `cancel` 函数，但是第一个参数传递的是 `false`,
为什么呢? 显然后面把当前节点的子节点已经删除了,  没有必要在对其所有下面的节点执行
删除操作了, 否则就是重复删除。

下面用图来表示, 首先是消息的传播，红色表示收到了消息的节点:

![](/assets/img/go/context/context_tree_cancel.svg)

然后把节点从树中删除:

![](/assets/img/go/context/context_tree_remove.svg)

## timerCtx

timerCtx  是跟时间相关的 Contex, 可以通过这个设置过期时间，并且传播消息

```go
type timerCtx struct {
        cancelCtx
        timer *time.Timer // Under cancelCtx.mu.

        deadline time.Time
}
```

要想新建一个 timerCtx 需要通过 `WithDeadline` 函数(也可以通过 `WithTimeout`， 但这个
函数其实是 `WithDeadline` 的包装调用，无需详细讲解), 第一个参数是父节点，第二个参数是
过期时间。 源码如下：

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
       // 由于父节点的过期时间比子节点的早，不需要单独设置过期,可以直接返回
        if cur, ok := parent.Deadline(); ok && cur.Before(d) {
                // The current deadline is already sooner than the new one.
                return WithCancel(parent) 
        }
        c := &timerCtx{
                cancelCtx: newCancelCtx(parent),
                deadline:  d,
        }
        // 构建context 树
        propagateCancel(parent, c)
        dur := time.Until(d)
       // 已经过了过期时间，直接返回, 不需要设置过期时间
        if dur <= 0 {
            
                c.cancel(true, DeadlineExceeded) // deadline has already passed
                return c, func() { c.cancel(false, Canceled) }
        }
        c.mu.Lock()
        defer c.mu.Unlock()
        // 设置过期时间，在过期时间会调用 cancel 函数
        if c.err == nil {
                c.timer = time.AfterFunc(dur, func() {
                        c.cancel(true, DeadlineExceeded)
                })
        }
        // 返回取消函数
        return c, func() { c.cancel(true, Canceled) }
}
```
这个函数的实现由很多优化的地方，首先对于设置的过期时间会和父节点进行比较，如果父节点过期
时间比当前节点的过期时间早，则直接返回一个 cancelCtx, 不需要设置过期时间，因为父节点肯定
比子节点过期的早，会触发消息的传递，然后传递个子节点，子节点没有机会执行自己的消息传递。
其次，计算完时间后，如果发现已经过期了，直接调用子节点的 cancel 函数，这时已经出发了消息
传递。 上面两个条件都不满足，则会调用 `time.AfterFunc` 函数设置一个时间，到这个时间后会
主动调用 cancel 函数进行消息的传播。

函数也会返回对应的 cancel 函数，我们也可以主动调用, 这个函数实现如下:

```go
func (c *timerCtx) cancel(removeFromParent bool, err error) {
        c.cancelCtx.cancel(false, err)
        if removeFromParent {
                // Remove this timerCtx from its parent cancelCtx's children.
                removeChild(c.cancelCtx.Context, c)
        }
        c.mu.Lock()
        if c.timer != nil {
                c.timer.Stop()
                c.timer = nil
        }
        c.mu.Unlock()
}
```
这里实际调用了 `cancelCtx`的 cancel 函数, 还有一点要注意这里调用了`c.timer.Stop()`,
这里是如果主动调用了 cancel 函数，则其对应的计时器就没有作用了，应该提前停止，这样
可以主动释放资源。

## valueCtx

前面讲的都是如何利用 context 传递消息，这里讲的是如何通过 context 传递数据。
context 的数据传递是通过 `valueCtx` 来完成的，他的定义如下：

```go
type valueCtx struct {
        Context
        key, val interface{}
}
```
主要是包含了 一对 key, val

valueCtx 的生成是通过 `WithValue` 来实现的：

```go
func WithValue(parent Context, key, val interface{}) Context {
        if key == nil {
                panic("nil key")
        }
        // 判断类型是否可以使用 ==  比较
        if !reflectlite.TypeOf(key).Comparable() {
                panic("key is not comparable")
        }
        return &valueCtx{parent, key, val}
}

```
首先判断当前的 key 是否可以使用 == 判断相等(这个定义在 `runtime/alg.go`中
可以看到, 这里不是重点就不介绍了)。 然后返回一个 `valueCtx` 结构。`valueCtx`
的 `Context` 字段指向的是父节点。

`valueCtx` 实现的也是一个树结构, 但是跟前面的 `cancelCtx` 不同，这里的 `valueCtx`
没有指向子节点的指针，只有指向父节点的指针，也就是说只能子节点访问父节点，父节点
无法方位子节点。

通过 `WithValue` 可以给一个 `valueCtx` 设置 key 和 value, 这样就能携带一些信息。
构建的树如下：

{% plantuml %}
@startuml
Background <-- valueCtx1
Background <-- valueCtx2
valueCtx1 <-- valueCtx3
valueCtx1 <-- valueCtx4
valueCtx2 <-- valueCtx5
valueCtx2 <-- valueCtx6

object Background{
}
object valueCtx1 {
Context
key
value
}
object valueCtx2 {
Context
key
value
}
object valueCtx3 {
Context
key
value
}
object valueCtx4 {
Context
key
value
}
object valueCtx5 {
Context
key
value
}
object valueCtx6 {
Context
key
value
}
@enduml
{% endplantuml %}

对于 valueCtx, 我们通过 `Value` 函数来获取这个信息：

```go
func (c *valueCtx) Value(key interface{}) interface{} {
        if c.key == key {
                return c.val
        }
        return c.Context.Value(key)
}
```
可以看到如何当前的 key 匹配到了,则返回对应的值，如果没有找打则会寻找父节点，这样递归的往上找，
直到不是 `valueCtx` 的节点, 返回 nil。 可见 value 的查找是非常低效的。最重要的是当你使用 context
传递数据时，可能会滥用，比如在过渡依赖 context, 在各个地方都会设置值:
1. 查找的时候不一定会从哪个节点开始，如果从父节点查找，而值存在子节点你是查找不到的
2. 如果 key 一致可能会无意中覆盖原来的值
3. 如果多个几点都有查找的 key, 那么查找的结果不一定会是哪一个

对于 key 的限制，golint 有一条规则 : `should not use basic type %s as key in context.WithValue`
哪些是基本类型呢？golint 中定义如下：

```go
var basicTypeKinds = map[types.BasicKind]string{
    types.UntypedBool:    "bool",
    types.UntypedInt:     "int",
    types.UntypedRune:    "rune",
    types.UntypedFloat:   "float64",
    types.UntypedComplex: "complex128",
    types.UntypedString:  "string",
}
```
就是因为对于基本类型而言，复制会出现覆盖，查找出现不确定的情况。一般情况下建议
使用一些自定义类型作为 key, 避免与其他的key冲突。

# 数据结构之间的关系：

前面讲了 context 中好几种数据结构及其实现，其实他们之间是有这非常紧密的联系的，
为了更加直观的看出的他们的关系，这里用一张图来表示：

{% plantuml %}
interface Context
interface canceler


Context : Dealline()
Context : Done()
Context : Err()
Context : Value()

canceler : cancel()
canceler : Done()

emptyCtx : String()
emptyCtx : Dealline()
emptyCtx : Done()
emptyCtx : Err()
emptyCtx : Value()

timerCtx : Deadline()
timerCtx : String()
timerCtx : cancelCtx
timerCtx : cancel()

cancelCtx : Done()
cancelCtx : Err()
cancelCtx : String()
cancelCtx : Context
cancelCtx : children
cancelCtx : cancel()

valueCtx : String()
valueCtx : Value()
valueCtx : Context



Context <|.. emptyCtx
timerCtx *-- *cancelCtx
canceler <|.. cancelCtx
Context <|.. cancelCtx
Context <|.. valueCtx
canceler <|.. timerCtx

{% endplantuml %}

这些结构体基本上都实现了 Context 接口，但是一般每个结构的侧重不一样，对于一些
接口的函数都是默认的实现。 比如 `cancelCtx` 并没有定义 `Value` 函数, `valueCtx`
也没有具体实现 `Done`, 这些函数是什么都不做的。

# context 使用举例
 对于 context 的使用 context 包里有说明：
- 不要将 Context 塞到结构体里。直接将 Context 类型作为函数的第一参数，而且一般都命名为 ctx。
- 不要向函数传入一个 nil 的 context，如果你实在不知道传什么，标准库给你准备好了一个 context：todo。
- 不要把本应该作为函数参数的类型塞到 context 中，context 存储的应该是一些共同的数据。
  例如：登陆的 session、cookie 等。
- 同一个 context 可能会被传递到多个 goroutine，别担心，context 是并发安全的。

context 的使用常见主要有以下几个。下面分别做一下介绍。

## 传递数据
在 web 开发中，我们为了串联整个请求的路径，会在日志中记录每条请求的唯一 id, 并且在访问下游服务
的时候把这个 id 传递下去。通过这个 id, 我们就能够对本次请求的路径进行了解，并且在遇到问题的时候
很好的定位在哪一步出现了问题。下面我们使用 context 来传递这个数据:

```go
package main

import (
        "context"
        "fmt"
        "math/rand"
)

type traceType string

func main() {
        ctx := context.Background()
        ctx = context.WithValue(ctx, traceType("traceId"), rand.Int())
        process(ctx)
}

func process(ctx context.Context) {
        traceID, ok := ctx.Value(traceType("traceId")).(int)
        if ok {
                fmt.Printf("traceType traceID=%d\n", traceID)
        } else {
                fmt.Println("no traceType tranceID")
        }

        traceID, ok = ctx.Value("traceId").(int)
        if ok {
                fmt.Printf("string type traceID=%d\n", traceID)
        } else {
                fmt.Println("no string type tranceID")
        }
}
```

这里注意一点，`WithValue` 的 key 使用的是自定义的类型 `traceType` 而不是基本类型 `string`,
避免了查找冲突和覆盖的问题。所以输出结果为：
```
traceType traceID=5577006791947779410
no string type tranceID
```
在实际的开发中我们要需要在 server 端给每个请求都加上这个 ID, 这个数据优先是从 HEADER 里传过来。
所以一般实际业务中我们这么写：
```go
import (
        "context"
        "fmt"
        "net/http"
)

type requestType string

var traceID = requestType("traceID")

func main() {

        h := hand{}
        http.HandleFunc("/hi", hi)
        http.ListenAndServe(":8000", h)
}

type hand struct{}

func (h hand) ServeHTTP(rw http.ResponseWriter, req *http.Request) {
        v := req.Header.Get("X-TRACE-ID")
        ctx := context.WithValue(req.Context(), traceID, v)
        reqCtx := req.WithContext(ctx)

        http.DefaultServeMux.ServeHTTP(rw, reqCtx)
}

func hi(rw http.ResponseWriter, req *http.Request) {
        v := req.Context().Value(traceID).(string)
        resp := fmt.Sprintf("traceID = %s\n", v)
        fmt.Fprintf(rw, resp)
}
```
## 防止 goroutine 泄露
参考文献中的例子:
有一个 goroutine 往 chan 发送信息:
```go
// gen is a broken generator that will leak a goroutine.
func gen() <-chan int {
    ch := make(chan int)
    go func() {
        var n int
        for {
            ch <- n
            n++
        }
    }()
    return ch
}
```

调用这个函数，当信息发送次数等于 5 就停止运行:

```go
// The call site of gen doesn't have a 
for n := range gen() {
    fmt.Println(n)
    if n == 5 {
        break
    }
}
```

停止运行后有一个问题，就是 gen 函数里的 goroutine 会一直存在，不会退出。
这样就照成了 goroutine 泄露，下面我们利用 context 改进一下这个程序：

```go
// gen is a generator that can be cancellable by cancelling the ctx.
func gen(ctx context.Context) <-chan int {
    ch := make(chan int)
    go func() {
        var n int
        for {
            select {
            case <-ctx.Done():
                return // avoid leaking of this goroutine when ctx is done.
            case ch <- n:
                n++
            }
        }
    }()
    return ch
}
```
加入了 context 参数，for 循环利用 select 监听取消的消息。调用的程序也改进了。
当 接收5次消息后会调用  cancel 函数发送消息，这样前面的 gen 就能够及时退出了。

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel() // make sure all paths cancel the context to avoid context leak

for n := range gen(ctx) {
    fmt.Println(n)
    if n == 5 {
        cancel()
        break
    }
}

// ...
```

## 超时控制
超时控制也是用的比较多的场景。在实际的工作场景中，我们对外提供服务要保证服务的可用性，
可用性的一个指标是响应时间。 一般上游访问我们都会有一个超时时间，当过了这个超时时间
上游就会结束访问，认为这次请求失败了，这时如果我们的服务还在处理响应的请求已经没有必要
了，所以我们应该及时退出，尽快回收资源，提高程序的性能。

```go
import (
        "context"
        "fmt"
        "net/http"
        "time"
)

func main() {

        http.HandleFunc("/hi", hi)
        http.ListenAndServe(":8000", nil)
}

func hi(rw http.ResponseWriter, req *http.Request) {
        ctx, cancel := context.WithTimeout(req.Context(), time.Millisecond*100)
        defer cancel()
        reqCtx := req.WithContext(ctx)

        for {
                select {
                case <-reqCtx.Context().Done():
                        return
                case <-time.After(time.Second):
                        // do something
                }
        }
        //...
}
```
这里要注意的是，如果已经进入了业务的处理内部，无法再回到 select 的阶段是无法取消这个
goroutine 的，也就是只有提前检查，或者周期性的检测才能使用。 



# 参考
[Go Concurrency Patterns: Context](https://blog.golang.org/context)
[深度解密Go语言之context](https://mp.weixin.qq.com/s/GpVy1eB5Cz_t-dhVC6BJNw)
[Using contexts to avoid leaking goroutines](https://rakyll.org/leakingctx/)

