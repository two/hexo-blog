title: Go 调度器抢占方式
date: 2019-05-28 10:35:56
tags: [Go]
---

## OS 调度
## Go 调度
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
这个地方的意思是如果发现`wait`不为0， 证明还有P正在运行，这时进入循环，循环内会每`100us`再进行一次抢占，直到所有的P都停止运行了才会退出这个循环(todo: 这个地方如何实现的？notetsleep为何总是返回false? 它与 wait之间的关系是怎么联系起来的?) 
由于主`goroutine`是一个循环，及时给这个循环加上了抢占的标记，但是由于没有`morestack`的调用就不会用这个抢占标记，所以主`goroutine`就无法被枪占。
而主`goroutine`中新启的`goroutine`，由于调用了runtime.GC(), 所以本身这个goroutine会被抢占，所以这个`goroutine`会停止执行，等待`GC`。
```go
go func() {
    for i := 0; i < 100; i++ {
        ch <- 1
            if i == 88 {
                runtime.GC()
            }
    }
}()
```
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
