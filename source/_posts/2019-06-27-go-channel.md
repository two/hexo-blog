title: go channel 原理
date: 2019-06-27 17:27:24
tags:
---
> 本篇主要介绍`chan`的内部实现原理(基于go1.12), 通过源码和图形的方式展示`chan`的内部结构及对`chan`进行操作的过程。
## make chan
在进入源码分析之前，我们假设自己并不知道去哪里看其源码，我们先简单的创建一个`chan`
```go
package main

func main() {
    _ = make(chan int, 3)
}
```
为了分析其内部实现，我们可以通过`compile`工具对其编译生成伪汇编代码:
```
go tool compile -S chan.go
```
生成的汇编代码重点的内容入下:
```x86asm
"".main STEXT size=71 args=0x0 locals=0x20
    0x0000 00000 (chan1.go:3)   TEXT    "".main(SB), ABIInternal, $32-0
    ...
    0x0031 00049 (chan1.go:4)   CALL    runtime.makechan(SB)
    ...
    0x0045 00069 (chan1.go:3)   JMP 0
```
可以看到执行`make`其实最终执行的是`runtime.makechan`这个函数，这个函数的实现在`runtime/chan.go`文件中:
```go
func makechan(t *chantype, size int) *hchan {
    elem := t.elem
    ...
    mem, overflow := math.MulUintptr(elem.size, uintptr(size))
    ...
    var c *hchan
    switch {
    case mem == 0:
        // Queue or element size is zero.
        c = (*hchan)(mallocgc(hchanSize, nil, true))
        // Race detector uses this location for synchronization.
        c.buf = c.raceaddr()
    case elem.kind&kindNoPointers != 0:
        // Elements do not contain pointers.
        // Allocate hchan and buf in one call.
        c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
        c.buf = add(unsafe.Pointer(c), hchanSize)
    default:
        // Elements contain pointers.
        c = new(hchan)
        c.buf = mallocgc(mem, elem, true)
    }

    c.elemsize = uint16(elem.size)
    c.elemtype = elem
    c.dataqsiz = uint(size)
    ...
    return c
```
可以看到最终会返回一个`*hchan`类型，这个就是`chan`的结构体:

```go
type hchan struct {
    qcount   uint           // 队列中有数据的个数
    dataqsiz uint           // 循环队列的大小z
    buf      unsafe.Pointer // 指向循环队列的地址
    elemsize uint16         
    closed   uint32 // chan的关闭状态
    elemtype *_type // element type
    sendx    uint   // 队列中下一个要发送的数据的下标
    recvx    uint   // 队列中下一个要接收的数据的下标
    recvq    waitq  // 等待接受的G队列
    sendq    waitq  // 等待发送的G队列
    lock     mutex  // 操作chan是需要加锁
}
```
执行完上面的`make`后，生成的`chan`如下:
![](/assets/img/go/channel/makechan.png)
## send chan
为了了解我们往`chan`发送的时候都做了什么我可能先写一个demo:
```go
package main

func main() {
    c := make(chan int, 3)
    c <- 3
}
```
查看其汇编代码:
```x86asm
"".main STEXT size=97 args=0x0 locals=0x20
    0x0000 00000 (chan2.go:3)   TEXT    "".main(SB), ABIInternal, $32-0
    ...
    0x0031 00049 (chan2.go:4)   CALL    runtime.makechan(SB)
    ...
    0x004b 00075 (chan2.go:5)   CALL    runtime.chansend1(SB)
    ...
    0x005f 00095 (chan2.go:3)   JMP 0
```
可以看出我们往`chan`发送数据其实执行的是`runtime.chansend1`函数，这个函数很简简单，只是调用了`runtime.chansend`函数,我们主要看一下`runtime.chansend`函数的实现:
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    if c == nil {
        if !block {
            return false
        }
        gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }
    ...
    if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
        (c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
        return false
    }
    ...
    lock(&c.lock)
    // 往已经 closed 的 chan 发送数据会直接 panic
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("send on closed channel"))
    }
    ...
    // 如果有接收队列，则进入send函数
    if sg := c.recvq.dequeue(); sg != nil {
        // Found a waiting receiver. We pass the value we want to send
        // directly to the receiver, bypassing the channel buffer (if any).
        send(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true
    }
    ...
    // 没有接收队列，buf还没有满，则直接往里放数据
    if c.qcount < c.dataqsiz {
        // Space is available in the channel buffer. Enqueue the element to send.
        qp := chanbuf(c, c.sendx)
        if raceenabled {
            raceacquire(qp)
            racerelease(qp)
        }
        typedmemmove(c.elemtype, qp, ep)
        c.sendx++
        if c.sendx == c.dataqsiz { //如果sendx == dataqsize, 证明buf满了，
            c.sendx = 0 // c.sendx=0保证了又从头开始，形成了一个循环队列
        }
        c.qcount++
        unlock(&c.lock)
        return true
    }

    if !block {
        unlock(&c.lock)
        return false
    } 

    //获取一个sudog结构, 把当前发送数据所在的g和要发送的数据都放到这里
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
        mysg.releasetime = -1
    }
    // No stack splits between assigning elem and enqueuing mysg
    // on gp.waiting where copystack can find it.
    mysg.elem = ep
    mysg.waitlink = nil
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.waiting = mysg
    gp.param = nil
    c.sendq.enqueue(mysg) // 把这个sudog结构体放到发送对队列中
    goparkunlock(&c.lock, waitReasonChanSend, traceEvGoBlockSend, 3) //阻塞当前g,直到由于可以发送数据而被唤醒
    // Ensure the value being sent is kept alive until the
    // receiver copies it out. The sudog has a pointer to the
    // stack object, but sudogs aren't considered as roots of the
    // stack tracer.
    KeepAlive(ep)

    // someone woke us up.
    if mysg != gp.waiting {
        throw("G waiting list is corrupted")
    }
    gp.waiting = nil
    if gp.param == nil {
        if c.closed == 0 {
            throw("chansend: spurious wakeup")
        }
        panic(plainError("send on closed channel"))
    }
    gp.param = nil
    if mysg.releasetime > 0 {
        blockevent(mysg.releasetime-t0, 2)
    }
    mysg.c = nil
    releaseSudog(mysg)
    return true
}
```

下面我们有一个图来表示其过程，图中主要分为下面几个步骤:
1. 往上面初始化好的`hchan`结构体发送第 1 个数据: 数据放到`buf[0]`的位置
2. 往`hchan`结构体发送第 2 个数据: 数据放到`buf[1]`的位置
3. 往`hchan`结构体发送第 3 个数据: 数据放到`buf[2]`的位置, 这时`buf`**满了**
4. 往`buf`满了的`hchan`结构体发送第 4 个数据: `g1`会放到`sudog`结构体中，并放到`sendq`队列中，等待被唤醒
5. 往`buf`满了的`hchan`结构体发送第 5 个数据: `g2`会放到`sudog`结构体中，并放到`sendq`队列中，等待被唤醒

![](/assets/img/go/channel/send.gif)

## recv chan
同上面一样，我们先写一个`demo`看看`recv`调用的是哪个函数:

```go
package main

func main() {
    c := make(chan int, 3)
    <-c
}
```
```x86asm
"".main STEXT size=94 args=0x0 locals=0x20
        ...
        0x0031 00049 (chan3.go:4)       CALL    runtime.makechan(SB)
        ...
        0x0048 00072 (chan3.go:5)       CALL    runtime.chanrecv1(SB)
        ...
        0x005c 00092 (chan3.go:3)       JMP     0

```
同样`runtime.chanrecv1`也是简单调用了`runtime.chanrecv`函数，具体代码如下:
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    ...
    if c == nil {
        if !block {
            return
        }
        gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }
    if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
        c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
        atomic.Load(&c.closed) == 0 {
        return
    }

    lock(&c.lock)
    // 如果chan已经被关闭，并且qcount==0, 则返回默认零值+false(如x, ok := <- c, x是零值，ok=false)
    if c.closed != 0 && c.qcount == 0 {
        if raceenabled {
            raceacquire(c.raceaddr())
        }
        unlock(&c.lock)
        if ep != nil {
            typedmemclr(c.elemtype, ep)
        }
        return true, false
    }
    //如果在接收的时候有发送队列存在，则执行recv函数
    if sg := c.sendq.dequeue(); sg != nil {
        // Found a waiting sender. If buffer is size 0, receive value
        // directly from sender. Otherwise, receive from head of queue
        // and add sender's value to the tail of the queue (both map to
        // the same buffer slot because the queue is full).
        recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true, true
    }
    // 如果存在buf, 存在数据
    if c.qcount > 0 {
        // Receive directly from queue
        qp := chanbuf(c, c.recvx) //获取recvx位置的地址
        if raceenabled {
            raceacquire(qp)
            racerelease(qp)
        }
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp) // 把recvx位置的数据copy到接收的变量中
        }
        typedmemclr(c.elemtype, qp) // 清空原来recvx位置的数据
        c.recvx++
        if c.recvx == c.dataqsiz { // 如果recvx == dataqsiz 证明已经到达最后一个，需要从头开始
            c.recvx = 0 //从头开始，形成一个循环队列
        }
        c.qcount--
        unlock(&c.lock)
        return true, true
    }

    if !block {
        unlock(&c.lock)
        return false, false
    }
    gp := getg()
    mysg := acquireSudog() // 获取一个sudog结构，把对应的g和接收数据的变量地址放到sudog中
    mysg.releasetime = 0
    if t0 != 0 {
        mysg.releasetime = -1
    }
    // No stack splits between assigning elem and enqueuing mysg
    // on gp.waiting where copystack can find it.
    mysg.elem = ep
    mysg.waitlink = nil
    gp.waiting = mysg
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.param = nil
    c.recvq.enqueue(mysg) // 把sudog放入接收队列中
    goparkunlock(&c.lock, waitReasonChanReceive, traceEvGoBlockRecv, 3) //阻塞当前g，直到被唤醒

    // someone woke us up
    if mysg != gp.waiting {
        throw("G waiting list is corrupted")
    }
    gp.waiting = nil
    if mysg.releasetime > 0 {
        blockevent(mysg.releasetime-t0, 2)
    }
    closed := gp.param == nil
    gp.param = nil
    mysg.c = nil
    releaseSudog(mysg)
    return true, !closed
}
```

上面说到如果存在发送队列就会执行`recv`函数，下面看一下这个函数的实现:
```go
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    //对于nobuf的chan, 直接copy数据
    if c.dataqsiz == 0 {
        if raceenabled {
            racesync(c, sg)
        }
        if ep != nil {
            // copy data from sender
            recvDirect(c.elemtype, sg, ep)
        }
    } else {
        // Queue is full. Take the item at the
        // head of the queue. Make the sender enqueue
        // its item at the tail of the queue. Since the
        // queue is full, those are both the same slot.
        qp := chanbuf(c, c.recvx) // 获取接收数据的位置
        if raceenabled {
            raceacquire(qp)
            racerelease(qp)
            raceacquireg(sg.g, qp)
            racereleaseg(sg.g, qp)
        }
        // copy data from queue to receiver
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp) //把recvx位置的数据copy到接收的变量中
        }
        // copy data from sender to queue
        typedmemmove(c.elemtype, qp, sg.elem) // 把发送队列的数据copy到当前recvx的位置
        c.recvx++
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        // 因为上面把发送队列的数据copy到了recvx, 为了保证下一个位置属按照顺序的，需要sendx = recvx
        // 这几步保证了chan是一个FIFO的过程
        c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz 
    }
    sg.elem = nil
    gp := sg.g
    unlockf()
    gp.param = unsafe.Pointer(sg)
    if sg.releasetime != 0 {
        sg.releasetime = cputicks()
    }
    goready(gp, skip+1) // 把出队的g放到ready中，下次调度就可以运行了，不再阻塞
}
```
下面我们有一个图来表示接收数据的过程，图中主要分为下面几个步骤:
1. 初始的`hchan`是上面`send`之后的结构
2. `g3`执行接收操作，首先会把发送队列中的第 1 个`g1`出队，然后把`buf[0]`的数据赋值到`g3`中，再把`g1`的数据赋值到`buf[0]`中
3. `g3`执行接收操作，首先会把发送队列中的第 2 个`g2`出队，然后把`buf[1]`的数据赋值到`g3`中，再把`g2`的数据赋值到`buf[1]`中
4. 这个时候没有发送队列了，所以可以直接把`buf[2]`中的书赋值到`g3`中
5. 把下一个数据`buf[0]`中的书赋值到`g3`中
6. 把最后一个数据`buf[1]`中的书赋值到`g3`中
7. 已经没有数据可以赋值给`g3`了，所以`g3`被放入`sudog`结构体中，入队到了接收队列, 进入阻塞状态

![](/assets/img/go/channel/recv.gif)

## send chan again

上面介绍`send`说到如果发送数据的时候有`recvq`队列就会调用`send`函数，这个函数的具体实现如下:
```go
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    if raceenabled {
        if c.dataqsiz == 0 {
            racesync(c, sg) // no buf 直接同步
        } else {
            // Pretend we go through the buffer, even though
            // we copy directly. Note that we need to increment
            // the head/tail locations only when raceenabled.
            qp := chanbuf(c, c.recvx) // 获取recvx位置
            raceacquire(qp)
            racerelease(qp)
            raceacquireg(sg.g, qp)
            racereleaseg(sg.g, qp)
            c.recvx++
            if c.recvx == c.dataqsiz {
                c.recvx = 0
            }
            c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
        }
    }
    if sg.elem != nil {
        sendDirect(c.elemtype, sg, ep) //直接把要发送的数据 copy 到 recvq 队列出队的 g 中
        sg.elem = nil
    }
    gp := sg.g
    unlockf()
    gp.param = unsafe.Pointer(sg)
    if sg.releasetime != 0 {
        sg.releasetime = cputicks()
    }
    goready(gp, skip+1) // 把g放到ready队列中，下次有机会被调度，不再阻塞
}

```
![](/assets/img/go/channel/send-recv.gif)
## close
当我们`close`掉一个`chan`都发生了什么呢? 下面写一个`close`的`demo`:
```go
package main

func main() {
    c := make(chan int, 3)
    close(c)
}
```

```x86asm
"".main STEXT size=85 args=0x0 locals=0x20
        ...
        0x0031 00049 (chan4.go:4)       CALL    runtime.makechan(SB)
        ...
        0x003f 00063 (chan4.go:5)       CALL    runtime.closechan(SB)
        0x0053 00083 (chan4.go:3)       JMP     0

```
可以调用了`runtime.closechan`函数，对应的代码为:
```go
func closechan(c *hchan) {
    if c == nil {
        panic(plainError("close of nil channel"))
    }

    lock(&c.lock)
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("close of closed channel")) // 已经关闭的 chan 不能再关闭
    }

    if raceenabled {
        callerpc := getcallerpc()
        racewritepc(c.raceaddr(), callerpc, funcPC(closechan))
        racerelease(c.raceaddr())
    }

    c.closed = 1 // 关闭状态设置为 1

    var glist gList
    // release all readers
    // 遍历所有recvq 队列, 从队列中去掉，并清空其内容，把所有g都放到glist结构中
    for {
        sg := c.recvq.dequeue()
        if sg == nil {
            break
        }
        if sg.elem != nil {
            typedmemclr(c.elemtype, sg.elem)
            sg.elem = nil
        }
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }
        gp := sg.g
        gp.param = nil
        if raceenabled {
            raceacquireg(gp, c.raceaddr())
        }
        glist.push(gp)
    }

    // 遍历所有 sendq 队列, 从队列中去掉，把所有g都放到glist结构中
    // release all writers (they will panic)
    for {
        sg := c.sendq.dequeue()
        if sg == nil {
            break
        }
        sg.elem = nil
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }
        gp := sg.g
        gp.param = nil
        if raceenabled {
            raceacquireg(gp, c.raceaddr())
        }
        glist.push(gp)
    }
    unlock(&c.lock)

    // Ready all Gs now that we've dropped the channel lock.
    // 把刚才所有放到 glist 中的 g 都改为ready 状态，使其不再阻塞
    for !glist.empty() {
        gp := glist.pop()
        gp.schedlink = 0
        goready(gp, 3)
    }
}

```

下面我们分别看一下:
1. 当存在`recvq`队列时:
![](/assets/img/go/channel/close-recv.gif)

2. 当存在`sendq`队列时:

![](/assets/img/go/channel/close-send.gif)

## no buffer chan
前面讲的都是带`buffer`的`chan`, 还有一种是经常使用的不带`buffer`的`chan`，其实处理起来更简单，前面源码部分已经有涉及了，下面看一下操作过程:
1. `make`一个不带`buffer`的`chan`
2. `g1`向这个`chan`发送数据, 由于没有接收者而被阻塞，放到`sendq`中
3. `g2`继续想这个`chan`发送数据，继续放到`sendq`中
4. 来一个接收者`g3`, 这时把`g1`从`sendq`中出队，并把`elem`的值赋值给`g3`的`x`
5. `g3`继续接收,把`g2`从`sendq`中出队，并把`elem`的值赋值给`g3`的`x`
6. 没有发送队列存在，`g3`也进入了阻塞状态，放到了`recvq`队列中

下面是其图形化展示:
![](/assets/img/go/channel/no-buf-chan.gif)
