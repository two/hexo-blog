title: Go 调度器抢占方式
date: 2019-05-28 10:35:56
tags: [Go]
---

## OS 调度
## Go 调度

被抢占后把 g 状态从 `_Grunning` 改为 `_Grunnable`。
```go
func goschedImpl(gp *g) {
    status := readgstatus(gp)
    if status&^_Gscan != _Grunning {
        dumpgstatus(gp)
        throw("bad g status")
    }
    casgstatus(gp, _Grunning, _Grunnable)
    dropg()
    lock(&sched.lock)
    globrunqput(gp)
    unlock(&sched.lock)

    schedule()
}
```
## Go 调度的问题

### deadloop
Go的抢占需要依赖函数的调用，只有在函数调用(准确的说是函数调用产生morestack调用的时候)的时候才会进行真正的强占，那么对于下面的这个方式:
```go
go func() {
    for {
    }
}
```
这是一个死循环，而且里面没有任何函数调用，也不会进行栈的扩张，所以这个goroutine永远不会被抢占。
参考[Goroutine调度实例简要分析](https://tonybai.com/2017/11/23/the-simple-analysis-of-goroutine-schedule-examples/?utm_source=rss&utm_medium=rss&utm_campaign=the-simple-analysis-of-goroutine-schedule-examples) 这篇文档的说明，我们看一下具体的问题及解决方案。
// todo: 继续完善上篇文章中的例子

### deadloop & GC

还有这样一个`case`:
```go
package main

import "runtime"

func main() {
    var ch = make(chan int, 100)
    go func() {
        for i := 0; i < 100; i++ {
            ch <- 1
            if i == 88 {
                runtime.GC()
            }
        }
    }()

    for {
        // the wrong part
        if len(ch) == 100 {
            sum := 0
            itemNum := len(ch)
            for i := 0; i < itemNum; i++ {
                sum += <-ch
            }
            if sum == itemNum {
                return
            }
        }
    }

}
```
上面这个程序也会hang死。

下面这段代码在主goroutine中运行
```go
for {
    // the wrong part
    if len(ch) == 100 {
        sum := 0
        itemNum := len(ch)
        for i := 0; i < itemNum; i++ {
            sum += <-ch
        }
        if sum == itemNum {
            return
        }
    }
}
```
上面这个程序由于没有函数的调用和`Goshced()`的主动调用所以会通过`阻塞监控`的方式被动弃权。

#### runtime.GC
当执行 `runtime.GC()`的时候都发生了什么？我们来看一下 
通过[dlv](https://github.com/go-delve/delve)这个工具我们可以对这个程序进行断点调试:
```
dlv debug go run gc.go
```
函数会执行到 `stopTheWorldWithSema` 这个函数，这个函数主要作用是停止所有的P，然后进行垃圾回收，我们通过一步一步调试发现, 这个函数会下面这个循环中无法出来:
```go
// wait for remaining P's to stop voluntarily
if wait {
    for {
        // wait for 100us, then try to re-preempt in case of any races
        if notetsleep(&sched.stopnote, 100*1000) {
            noteclear(&sched.stopnote)
            break
        }
        preemptall()
    }
}
```
为什么会在这个地方无法出来？下面分析一下具体原因。

GC种一个步骤是要把所有的 p 都设置为`_Pgcstop` 状态后才能继续进行。 下面看看这个步骤是否能够完成。

`stopTheWorldWithSema`函数更加详细的执行过程如下:
```go
func stopTheWorldWithSema() {
    _g_ := getg()

    // If we hold a lock, then we won't be able to stop another M
    // that is blocked trying to acquire the lock.
    if _g_.m.locks > 0 {
        throw("stopTheWorld: holding locks")
    }

    lock(&sched.lock)
    sched.stopwait = gomaxprocs // 设置stopwait的初始值为最大的 p 的个数
    atomic.Store(&sched.gcwaiting, 1) // 设置 gcwaiting = 1, 表示正在进入GC状态
    preemptall() // 给所有的 p 发送抢占信号，如果成功，则对应的 p 进入 idle 状态
    // stop current P
    _g_.m.p.ptr().status = _Pgcstop // Pgcstop is only diagnostic.
    sched.stopwait-- // 给他当前的设置状态后，stopwait个数减一 
    // try to retake all P's in Psyscall status
    // 遍历所有的 p 如果满足条件(p的状态为 _Psyscall)则释放这个 p , 并且把 p 的状态都设置成 _Pgcstop ; 然后stopwait--
    for _, p := range allp {
        s := p.status
        if s == _Psyscall && atomic.Cas(&p.status, s, _Pgcstop) {
            if trace.enabled {
                traceGoSysBlock(p)
                traceProcStop(p)
            }
            p.syscalltick++
            sched.stopwait--
        }
    }
    // stop idle P's
    for {
        p := pidleget() //获取idle 状态的 p, 从 _Pidle list 获取
        if p == nil {
           break
        }
        p.status = _Pgcstop // 把 p 状态设置为 _Pgcstop
        sched.stopwait-- // 计数 stopwait --
    }
    wait := sched.stopwait > 0
    unlock(&sched.lock)

    // wait for remaining P's to stop voluntarily
    if wait {
        for {
            // wait for 100us, then try to re-preempt in case of any races
            if notetsleep(&sched.stopnote, 100*1000) {
                noteclear(&sched.stopnote)
                break
            }
            // 再次给所有的 p 发送 抢占信号
            preemptall()
        }
    }
...
}
```

上面函数把所有非`_Prunning`状态的 p 都设置为了 `_Pgcstop` 状态，对于 `_Prunning` 状态的 p 如何设置其为 `_Pgcstop` 状态呢? 主要是通过 `preemptall()`函数给每个 p 发送抢占信号
`preemptall()` 其实时调用了 `preemptone()` 前面我们已经讲了具体的原理。被抢占后 p 重新进入调度阶段

```go
func schedule() {
    _g_ := getg()

    if _g_.m.locks != 0 {
        throw("schedule: holding locks")
    }

    if _g_.m.lockedg != 0 {
        stoplockedm()
        execute(_g_.m.lockedg.ptr(), false) // Never returns.
    }

    // 我们不应该调度一个正在执行 cgo 调用的 g
    // 因为 cgo 在使用当前 m 的 g0 栈
    // We should not schedule away from a g that is executing a cgo call,
    // since the cgo call is using the m's g0 stack.
    if _g_.m.incgo {
        throw("schedule: in cgo")
    }

top:
    if sched.gcwaiting != 0 {
        // 如果还在等待 gc，则
        gcstopm()
        goto top // 循环执行
    }
...
```

上面说调度器会会把 `gcwaiting`设置为`1`, 所以这里会进入 `gcstopm()`, 直到所有的 m 都被`stop`
```go
func gcstopm() {
    _g_ := getg()

    if sched.gcwaiting == 0 {
        throw("gcstopm: not waiting for gc")
    }
    if _g_.m.spinning {
        _g_.m.spinning = false
        // OK to just drop nmspinning here,
        // startTheWorld will unpark threads as necessary.
        if int32(atomic.Xadd(&sched.nmspinning, -1)) < 0 {
            throw("gcstopm: negative nmspinning")
        }
    }
    _p_ := releasep()
    lock(&sched.lock)
    _p_.status = _Pgcstop //设置 p 状态为 _Pgcstop
    sched.stopwait--
    if sched.stopwait == 0 {
        notewakeup(&sched.stopnote)
    }
    unlock(&sched.lock)
    stopm()
}
```
在 `gcstopm()` 会把 p 的状态置为 `_Pgcstop`。

**但是死循环的 g 不会被抢占，所以其 p 状态会一直是  Prunning 无法被设置为 Pgcstop**

再回到前面进入死循环的地方:
```go
// wait for remaining P's to stop voluntarily
if wait {
    for {
        // wait for 100us, then try to re-preempt in case of any races
        if notetsleep(&sched.stopnote, 100*1000) {
            noteclear(&sched.stopnote)
            break
        }
        preemptall()
    }
}
```
这里进入死循环的原因是条件
```go
notetsleep(&sched.stopnote, 100*1000) == true
```
不满足
`notetsleep`函数内部每隔一段时间就会返回:
```go
return atomic.Load(key32(&n.key)) != 0 // n.key 为参数 &shced.stopnote.key的值
```
这个函数意思是`&sched.stopnote.key != 0`
如果要想让返回值为 `true` 就需要满足上面的条件。 `stopnote.key`的值有两个函数可以控制:
* `notewakeup` 把 `stopnote` 设置为 1
* `noteclear 把 `stopnote` 设置为 0
所以我们需要调用`notewakeup`才行。而这个函数我们可以看到是在 `gcstopm()`种有调用:
```go
sched.stopwait--
if sched.stopwait == 0 {
    notewakeup(&sched.stopnote)
}
```
由于存在 g 无法被抢占，所以其对应的 p 不会释放, `stopwait`也就不能为`0`, 所以也就无法执行`notewakeup`,最终导致上面的循环无法出来。

死锁状态的发生:
* GC: 要想进行`GC`就需要所有的P都转为空闲状态，而主`goroutine`无法被抢占，对应的`P`也无法进入空闲。所以`GC`会一直阻塞。
* 新启动的`goroutine`: 由于新启动的`goroutine`也进入了空闲状态
* 主`goroutine`: 由于新启动的`goroutine`进入了空闲状态,无法再给`chan`发信号，所以主`goroutine`也无法退出。
由于上面三个都进入了阻塞状态，导致了整个程序进入了死锁状态。


## 参考
[scheduling-in-go-part1](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)
[scheduling-in-go-part2](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html)
[scheduling-in-go-part3](https://www.ardanlabs.com/blog/2018/12/scheduling-in-go-part3.html)
[go-under-the-hood](https://github.com/two/go-under-the-hood/blob/master/book/part2runtime/ch06sched/preemptive.md)
[non-cooperative-preemption](https://github.com/golang/proposal/blob/master/design/24543-non-cooperative-preemption.md)
[如何定位 golang 进程 hang 死的 bug](https://gocn.vip/article/441)
[Goroutine调度实例简要分析](https://tonybai.com/2017/11/23/the-simple-analysis-of-goroutine-schedule-examples/?utm_source=rss&utm_medium=rss&utm_campaign=the-simple-analysis-of-goroutine-schedule-examples)
