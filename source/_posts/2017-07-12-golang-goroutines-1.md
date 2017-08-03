layout: title
title: golang channels 的串联,扇入扇出及控制
date: 2017-07-12 10:06:03
tags: [golang,goroutines]
---

> 如果说goroutine是Go语言程序的并发体的话，那么channels则是它们之间的通信机制。 一个channel是一个通信机制，它可以让一个goroutine通过它给另一个goroutine发送值信息。 channel之间可以进行串联，并联等组合，组成我们想要的运行方式。 不同goroutine之间需要同步，也需要控制，具体该如何处理这些情况，下面分别进行介绍。

## channel基础
使用内置的make函数，我们可以创建一个channel:
```golang
ch := make(chan int) // ch has type 'chan int'
```
当我们复制一个channel或用于函数参数传递时，我们只是拷贝了一个channel引用，因此调用者和被调用者将引用同一个channel对象。和其它的引用类型一样，channel的零值也是nil。
两个相同类型的channel可以使用==运算符比较。如果两个channel引用的是相同的对象，那么比较的结果为真。一个channel也可以和nil进行比较。
一个channel有发送和接受两个主要操作，都是通信行为。
```golang
ch <- x  // a send statement
x = <-ch // a receive expression in an assignment statement
<-ch     // a receive statement; result is discarded
```
Channel还支持close操作，用于关闭channel，随后对基于该channel的任何发送操作都将导致panic异常。对一个已经被close过的channel进行接收操作依然可以接受到之前已经成功发送的数据；如果channel中已经没有数据的话将产生一个零值的数据。
```
close(ch)
```
以最简单方式调用make函数创建的是一个无缓存的channel，但是我们也可以指定第二个整型参数，对应channel的容量。如果channel的容量大于零，那么该channel就是带缓存的channel。
```golang
ch = make(chan int)    // unbuffered channel
ch = make(chan int, 0) // unbuffered channel
ch = make(chan int, 3) // buffered channel with capacity 3
```
### 不带缓存的Channels
一个基于无缓存Channels的发送操作将导致发送者goroutine阻塞，直到另一个goroutine在相同的Channels上执行接收操作，当发送的值通过Channels成功传输之后，两个goroutine可以继续执行后面的语句。反之，如果接收操作先发生，那么接收者goroutine也将阻塞，直到有另一个goroutine在相同的Channels上执行发送操作。
基于无缓存Channels的发送和接收操作将导致两个goroutine做一次同步操作。因为这个原因，无缓存Channels有时候也被称为同步Channels。当通过一个无缓存Channels发送数据时，接收者收到数据发生在唤醒发送者goroutine之前。

对于不带缓存的Channels，我们使用的是有必须放到goroutine,因为如果直接调用`chanx <- 1`时，会报错`fatal error: all goroutines are asleep - deadlock!`
```golang
package main

func main() {
    chanx := make(chan int)
    chanx <- 1 //fatal error: all goroutines are asleep - deadlock!
    <-chanx
```
由于主goroutine调用了 `chanx <-1`, 但是由于是顺序往下执行，执行时还不存在监听`chanx`的方法存在，所以数据放入`chanx`后无法唤醒接收的方法，只能等待下去，所以就产生了deadlock。
可以修改为下面的形式，把`chanx <- 1`放入到一个goroutine里，然后主goroutine监听了这个`chanx`，当往`chanx`放数据的时候就会有接收的方法被调用。
```golang
package main

func main() {
    chanx := make(chan int)
    go func() {chanx <- 1}() //right
    <-chanx
```
**当使用`range`遍历`chan`时别忘了close**, 下面当没有使用close时:
```golang
package main
import "fmt"
func main() {
    chanx := make(chan int)
    go func() {
        for i := 0; i < 3; i++ {
            chanx <- i
        }
    }()
    for v := range chanx {
        fmt.Println(v)
    }
}
```
output:
```golang
0
1
2
fatal error: all goroutines are asleep - deadlock!
goroutine 1 [chan receive]:

```
`range`会从`channel`中接收数据直到`channel`被`close`为止，正常情况下`close`并不是必须的，只有在接收者需要知道没有更多的数据进入的时候才需要，而`range`正是需要知道这个信息的。所以代码改成下面这样就没问题了:
```golang
package main
import "fmt"
func main() {
    chanx := make(chan int)
    go func() {
        for i := 0; i < 3; i++ {
            chanx <- i
        }
        close(chanx)
    }()
    for v := range chanx {
        fmt.Println(v)
    }
}

```

### 带缓存的Channels
带缓存的Channel内部持有一个元素队列。队列的最大容量是在调用make函数创建channel时通过第二个参数指定的。下面的语句创建了一个可以持有三个字符串元素的带缓存Channel。
```golang
ch = make(chan string, 3)
```
向缓存Channel的发送操作就是向内部缓存队列的尾部插入元素，接收操作则是从队列的头部删除元素。如果内部缓存队列是满的，那么发送操作将阻塞直到因另一个goroutine执行接收操作而释放了新的队列空间。相反，如果channel是空的，接收操作将阻塞直到有另一个goroutine执行发送操作而向队列插入元素。

**队列元素为1的带缓存Channels与不带缓存的Channels是不同的**，下面的例子可以看出:
```golang
package main

func main() {
    chan_nobuffer := make(chan int)
    chan_nobuffer <- 1 //error 必须放到goroutine中
    <-chan_nobuffer

    chan_buffer := make(chan int, 1)
    chan_buffer <- 1 //right
    <-chan_buffer
}
```

### 单方向的Channel
channel还有两种语法:`<-chan int`和`chan<- int`，其意思是单方向的channel, 当定义为`out chan<- int`表示`out`只能被往里面放数据，不允许从out拿数据,否则程序会报错`receive from send-only type chan<- int`,如果定义为`in <-chan int`则`in`只能往外输出数据，不允许往`in`里面放数据，否则报错`send to receive-only type <-chan int`

## channel串联
Channels也可以用于将多个goroutine连接在一起，一个Channel的输出作为下一个Channel的输入。 这种串联的Channels就是所谓的管道（pipeline）。下图就是一个串联的channel示意:
![串联channel](/assets/img/golang/goroutines_0712_1.png)
第一个goroutine Counter负责生成一个0,1,2,3,...形式的整数序列,然后把整数序列输入到一个channel中，通过这个channel传递个下一个goroutine Squarer, 负责将从channel接收到的数求平方，然后再把得出的结果通过channel传递给goroutine Printer, Printer负责将从channel接收的数据打印出来。
其程序实现如下:
```golang
package main

import (
    "fmt"
)

func main() {
    chan1 := make(chan int)
    chan2 := make(chan int)
    go Counter(chan1)
    go Squarer(chan2, chan1)
    Printer(chan2)

}

func Counter(out chan<- int) {
    for i := 1; i < 10; i++ {
        out <- i
    }
    close(out)
}

func Squarer(out chan<- int, in <-chan int){
    for v := range in {
        out <- v * v
    }
    close(out)
}

func Printer(in <-chan int) {
    for v := range in {
        fmt.Println(v)
    }
}
```
上面代码中我们创建了两个chan, 然后调用了`Counter`和`Squarer`, 由于上面说:**当我们复制一个channel或用于函数参数传递时，我们只是拷贝了一个channel引用，因此调用者和被调用者将引用同一个channel对象。**所以我们对chan1和chan2的修改都是全局的。
`Counter`往chan1中陆续放入了`0,1,2,3,...`等数列，然后同步的`Squarer`接收到数据对其平方并放入`chan2`,最后`Printer`从`chan2`中输出这些数据。
对于串联的Channel还有另外一种实现方法:
```golang
package main

import (
    "fmt"
)

func main() {
    c := gen(2,3)
    out := sq(c)

    for v := range out {
        fmt.Println(v)
    }
}

func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

func sq(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n*n
        }
        close(out)
    }()
    return out
}
```
上面的`gen`函数用到了golang的**可变参数**这个特性，跟上面的`Counter`不一样的是，这个`gen`会把`chan`当做返回值返回，而不是作为参数传入。`sq`函数也跟`Squarer`函数不一样了:把上一个函数的chan最为参数，下一个输出的chan作为返回值。


## channel扇入扇出
**扇出**：同一个 channel 可以被多个函数读取数据，直到channel关闭。 这种机制允许将工作负载分发到一组worker，以便更好地并行使用 CPU 和 I/O。

```golang
func main() {
    c := gen(2,3)
    c1 := sq(c)
    c2 := sq(c)

    for v := range c1 {
        fmt.Println(v)
    }
    fmt.Println("-------------")

    for v := range c2 {
        fmt.Println(v)
    }

}
```
下面是几种输出样式，可以知道当调用两次`sq`时，其实是对chan的扇出操作，既一个channel被多个函数读取了。每次读取的顺序和个数都不能保证。
```golang
#1 
4
------------------
9
#2
4
9
------------------
#3
9
4
------------------
...
```

**扇入**：多个 channel 的数据可以被同一个函数读取和处理，然后合并到一个 channel，直到所有 channel都关闭。
```golang
func merge(cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    // Start an output goroutine for each input channel in cs.  output
    // copies values from c to out until c is closed, then calls wg.Done.
    output := func(c <-chan int) {
        for n := range c {
            out <- n //对于每个chan其中的元素都放到out中 
        }
        wg.Done() //减少一个goroutine
    }
    wg.Add(len(cs)) //要执行的goroutine个数
    for _, c := range cs {
        go output(c) //对传入的多个channel执行output
    }

    // Start a goroutine to close out once all the output goroutines are
    // done.  This must start after the wg.Add call.
    go func() {
        wg.Wait() //等待，直到所有goroutine都完成后
        close(out) //所有的都放到out后关闭
    }()
    return out
}
```
`merge`函数的参数也是变长的，类型是`chan`, 这个函数还用到了`sync`这个包，这里主要的作用就是对一组goroutines进行同步。首先把传入的cs都通过`output`调用放入`out`中，每处理完一个`c`就调用`wg.Done()`更新剩余的次数, `wg.Wait()`等到所有的channels把数据放到`out`中，然后关闭`out`。
```golang
func main() {
    c := gen(2, 3, 4, 5, 6, 7, 8)
    out2 := sq(c)
    out1 := sq(c)
    for v := range merge(out1, out2) {
        fmt.Println(v)
    }
}
```
下图就展示了扇入扇出的过程:
![串联channel](/assets/img/golang/goroutines_0712_2.png)

## goroutines控制

## 参考
* [The Go Blog - pipelines](https://blog.golang.org/pipelines)
* [Go语言并发模型：像Unix Pipe那样使用channel](https://segmentfault.com/a/1190000006261218)
* [Go语言圣经-channels](https://docs.hacknode.org/gopl-zh/ch8/ch8-04.html)
* [Go语言圣经-并发的循环](https://docs.hacknode.org/gopl-zh/ch8/ch8-05.html)
* [Go语言圣经-可变参数](https://docs.hacknode.org/gopl-zh/ch5/ch5-07.html)
* [快速掌握 Golang context 包](https://deepzz.com/post/golang-context-package-notes.html)
* [A Tour of Go - Range and Close](https://tour.golang.org/concurrency/4)
