title: Go Interface 使用
date: 2019-09-04 16:04:29
tags: [Go]
---
> 本文基于`go1.12.4`源码

## Duck Typing
## 面相对象
### 实现多个接口
 下面举一个例子:
```go
package main

type Mather interface {
    Sub(a, b int64) int64
    Add(a, b int32) int32
}

type Caller interface {
    Name() string
}

type Adder struct{ id int32 }

func main() {
    adder := Adder{id: 6754}
    CallAdd(adder)
    CallSub(adder)
    CallName(adder)
}

//go:noinline
func (adder Adder) Add(a, b int32) int32 { return a + b }

//go:noinline
func (adder Adder) Sub(a, b int64) int64 { return a - b }

//go:noinline
func (adder Adder) Name() string { return "Adder" }

func CallAdd(m Mather) {
    m.Add(12, 2)
}

func CallSub(m Mather) {
    m.Sub(19, 4)
}

func CallName(c Caller) {
    c.Name()
}
```
`Adder`实现了两个接口`Mather`和`Caller`, 定义`CallAdd`和`CallSub`调用`Mather`类型，可以看到:
```x86asm
    0x0031 00049 (main.go:16)   MOVL    $6754, (SP)
    0x0038 00056 (main.go:16)   CALL    runtime.convT32(SB)
    0x003d 00061 (main.go:16)   MOVQ    8(SP), AX; 返回值, data 字段
    0x0042 00066 (main.go:16)   MOVQ    AX, ""..autotmp_1+40(SP)
    0x0047 00071 (main.go:16)   LEAQ    go.itab."".Adder,"".Mather(SB), CX ; Mather 类型
    0x004e 00078 (main.go:16)   MOVQ    CX, (SP) ; 类型从 CX 放到栈底
    0x0052 00082 (main.go:16)   MOVQ    AX, 8(SP);  值 data 从 AX 放到 8(SP)位置
    0x0057 00087 (main.go:16)   CALL    "".CallAdd(SB) ; 前面 (SP)和8(SP)加起来就是一个 adder 实现的 Mather 类型，作为这个函数调用的参数
    0x005c 00092 (main.go:17)   MOVL    "".adder+20(SP), AX
    0x0060 00096 (main.go:17)   MOVL    AX, (SP)
    0x0063 00099 (main.go:17)   CALL    runtime.convT32(SB)
    0x0068 00104 (main.go:17)   MOVQ    8(SP), AX
    0x006d 00109 (main.go:17)   MOVQ    AX, ""..autotmp_2+32(SP)
    0x0072 00114 (main.go:17)   LEAQ    go.itab."".Adder,"".Mather(SB), CX
    0x0079 00121 (main.go:17)   MOVQ    CX, (SP)
    0x007d 00125 (main.go:17)   MOVQ    AX, 8(SP)
    0x0082 00130 (main.go:17)   CALL    "".CallSub(SB); 处理同上 CallAdd
```
接着调用`CallName`, 需要的是一个`Caller`类型的参数:

```x86asm
    0x0087 00135 (main.go:18)   MOVL    "".adder+20(SP), AX; adder 的值放到 AX
    0x008b 00139 (main.go:18)   MOVL    AX, (SP); 放到栈底
    0x008e 00142 (main.go:18)   CALL    runtime.convT32(SB)
    0x0093 00147 (main.go:18)   MOVQ    8(SP), AX; 返回处理后的值 unsafe.Pointer
    0x0098 00152 (main.go:18)   MOVQ    AX, ""..autotmp_3+24(SP)
    0x009d 00157 (main.go:18)   LEAQ    go.itab."".Adder,"".Caller(SB), CX ; Caller 类型地址放到 CX
    0x00a4 00164 (main.go:18)   MOVQ    CX, (SP); 类型地址从 CX 赋值到栈底
    0x00a8 00168 (main.go:18)   MOVQ    AX, 8(SP); 值 data 从 AX 赋值到 8(SP)
    0x00ad 00173 (main.go:18)   CALL    "".CallName(SB): 前面两行把 adder 转换为Caller 类型，并做为参数本函数
```

通过上面可以看出，一个实例实现了多个接口，在具体调用的地方会根据接口的类型转换为不同的接口

### 未实现接口
如果我们调用一个接口的方法，而对应的实例没有实现这个接口会出现什么问题呢？
```go
package main

type Adder struct{ id int32 }

type Empty interface {
    F()
}

func main() {
    adder := Adder{id: 6754}
    CallF(adder)
}

func CallF(e Empty) {
    e.F()
}
```
对上面的代码进行编译，得到:
```
# command-line-arguments
./main2.go:11:7: cannot use adder (type Adder) as type Empty in argument to CallF:
        Adder does not implement Empty (missing F method)
```
可见编译器会在编译阶段对 AST 数据结构进行检查，如果发现没有实现对应的函数，就会报错。具体代码在`cmd/compile/internal/gc/subr.go`:
```go
        if why != nil {
            if isptrto(src, TINTER) {
                *why = fmt.Sprintf(":\n\t%v is pointer to interface, not interface", src)
            } else if have != nil && have.Sym == missing.Sym && have.Nointerface() {
                *why = fmt.Sprintf(":\n\t%v does not implement %v (%v method is marked 'nointerface')", src, dst, missing.Sym)
            } else if have != nil && have.Sym == missing.Sym {
                *why = fmt.Sprintf(":\n\t%v does not implement %v (wrong type for %v method)\n"+
                    "\t\thave %v%0S\n\t\twant %v%0S", src, dst, missing.Sym, have.Sym, have.Type, missing.Sym, missing.Type)
            } else if ptr != 0 {
                *why = fmt.Sprintf(":\n\t%v does not implement %v (%v method has pointer receiver)", src, dst, missing.Sym)
            } else if have != nil {
                *why = fmt.Sprintf(":\n\t%v does not implement %v (missing %v method)\n"+
                    "\t\thave %v%0S\n\t\twant %v%0S", src, dst, missing.Sym, have.Sym, have.Type, missing.Sym, missing.Type)
            } else {
                *why = fmt.Sprintf(":\n\t%v does not implement %v (missing %v method)", src, dst, missing.Sym)
            }
        }
```
### 值接收与指针接收
实现接口方法的时候可以使用指针接收也可以使用值接收，他们有什么区别？不通的接收方式存在什么问题呢？针对实现和调用的方式，我们可以有四种组合，分别是:
1. 值接收，值调用
2. 值接收，指针调用
3. 指针接收，指针调用
4. 指针接收，值调用

#### 值接收，值调用
```go
package main

import "fmt"

type notifer interface {
    notify()
}

type user struct {
    id int32
}

func (u user) notify() {
    fmt.Printf("Sending user email to %d\n", u.id)
}

func main() {
    u := user{9527}
    sendNotification(u)
}

func sendNotification(n notifer) {
    n.notify()
}
```
这种方式是最常见的，可以编译和调用，
```x86asm
    0x001d 00029 (main3.go:18)  MOVL    $0, "".u+20(SP) ; 初始化 u 为空值
    0x0025 00037 (main3.go:18)  MOVL    $9527, "".u+20(SP) ; 给 u 赋值
    0x002d 00045 (main3.go:19)  MOVL    $9527, (SP) ; 放到栈底，作为下面函数调用的参数
    0x0034 00052 (main3.go:19)  CALL    runtime.convT32(SB) ; 返回新申请的堆上的数据，并且返回
    0x0039 00057 (main3.go:19)  MOVQ    8(SP), AX; 函数的返回值放到 AX 中
    0x003e 00062 (main3.go:19)  MOVQ    AX, ""..autotmp_1+24(SP) ;放到临时变量 autotmp_1中
    0x0043 00067 (main3.go:19)  LEAQ    go.itab."".user,"".notifer(SB), CX ; 把 user 转换为 notifer 类型，并把 itab 地址放到 CX
    0x004a 00074 (main3.go:19)  MOVQ    CX, (SP) ; _type 地址 赋值到栈底
    0x004e 00078 (main3.go:19)  MOVQ    AX, 8(SP); data 赋值到 8(SP)
    0x0053 00083 (main3.go:19)  CALL    "".sendNotification(SB) ; 前面两行组成的 interface 作为参数进行函数调用
```
这里详细分析一下下面这行代码:
```x86asm
   LEAQ    go.itab."".user,"".notifer(SB), CX
```

详细解释一下这里的含义: `go tool compile`生成的是一个间接目标文件，还没有经过 链接器的链接, 符号没有把 package 名字填充上，如果填充上的话应该是这样:
`go.itab.main.user,main.notifer(SB)`(package是main), 这个代码可以看出其作用是为了把`user`和`notifer`关联起来，并且取出`itab`的地址。在汇编代码中还可以找出这样一段:
```x86asm
    go.itab."".user,"".notifer SRODATA dupok size=32
        0x0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0010 56 e9 47 80 00 00 00 00 00 00 00 00 00 00 00 00  V.G.............
        rel 0+8 t=1 type."".notifer+0
        rel 8+8 t=1 type."".user+0
        rel 24+8 t=1 "".(*user).notify+0
    go.itablink."".user,"".notifer SRODATA dupok size=8
        0x0000 00 00 00 00 00 00 00 00                          ........
        rel 0+8 t=1 go.itab."".user,"".notifer+0
```
对上面的代码我们一句一句来分析，首先第一句是声明和符号和他的属性: `go.itab."".user,"".notifer SRODATA dupok size=32` 
我们这里得到的是一个 32 字节的全局对象的符号，该符号将被存到二进制文件的 .rodata 段中

* `dupok`表示: 该变量对应的标识符可能有多个， 链接时 只选择其中一个即可，一般用于合并相同的常量字符串，减少重复数据占用的空间
* `RODATA`表示: 将变量定义在只读内存段，因此后续任何对此变量的修改操作将导致异常(recover()也无法捕获) )

后面的 两行表示的是这32个字节存储的数据内容,  也就是`itab`被序列化之后的表示方法。 我们再来回顾一下`itab`类型的定义:
```go
type itab struct {       // 32 bytes on a 64bit arch
    inter *interfacetype // offset 0x00 ($00)
    _type *_type         // offset 0x08 ($08)
    hash  uint32         // offset 0x10 ($16)
    _     [4]byte        // offset 0x14 ($20)
    fun   [1]uintptr     // offset 0x18 ($24)
}
```
可以看出前面 32 字节中有内容的部分对应的就是`itab.hash`的四个字节
再往下:
* `rel 0+8 t=1 type."".notifer+0`   : 告诉链接器需要将内容的前8个字节填充为全局符号 `type."".notifer` 的地址 , 也就是 `itab.inter` 字段
* `rel 8+8 t=1 type."".user+0`      : 告诉链接器需要将内容的 8-16 字节填充为全局符号 `type."".user` 的地址 , 也就是 `itab._type`字段
* `rel 24+8 t=1 "".(*user).notify+0`: 这里对应的是`itab.func`的值, 填充的是 user.notify 函数的地址

总结一下`LEAQ    go.itab."".user,"".notifer(SB), CX`的含义就是:
1.  使用 接口 `notifer` 和  类型 `user` 组合成一个 `itab`类型
2.  取 `itab` 地址加载到 `CX` 编译器


#### 值接收，指针调用
```go
package main

import "fmt"

type notifer interface {
    notify()
}

type user struct {
    id int32
}

func (u user) notify() {
    fmt.Printf("Sending user email to %d\n", u.id)
}

func main() {
    u := &user{9527}
    sendNotification(u)
}

func sendNotification(n notifer) {
    n.notify()
}
```
```x86asm
    0x001d 00029 (main3.go:18)  LEAQ    type."".user(SB), AX ; 获取 user 的类型_type 地址放到 AX
    0x0024 00036 (main3.go:18)  MOVQ    AX, (SP) ; _type 地址放到栈底，作为参数
    0x0028 00040 (main3.go:18)  CALL    runtime.newobject(SB) ; 调用 runtime.newobject 会从堆上申请内存 用来存放数据
    0x002d 00045 (main3.go:18)  MOVQ    8(SP), AX ; 返回值放到 AX
    0x0032 00050 (main3.go:18)  MOVQ    AX, ""..autotmp_2+24(SP) ; 返回值赋值给 autotmp_2
    0x0037 00055 (main3.go:18)  MOVL    $9527, (AX); 9527 赋值给 AX 所指向的地址的值
    0x003d 00061 (main3.go:18)  MOVQ    ""..autotmp_2+24(SP), AX 
    0x0042 00066 (main3.go:18)  MOVQ    AX, "".u+16(SP) ; AX 赋值给变量 u, u 的地址是指向 runtime.newobject 新申请的地址，值为 9527
    0x0047 00071 (main3.go:19)  MOVQ    AX, ""..autotmp_1+32(SP) ; AX 赋值给临时变量 autotmp_1
    0x004c 00076 (main3.go:19)  LEAQ    go.itab.*"".user,"".notifer(SB), CX ;  获取 user 实现 notifer 接口类型的地址，放到 CX
    0x0053 00083 (main3.go:19)  MOVQ    CX, (SP) ; itab 地址放到栈底
    0x0057 00087 (main3.go:19)  MOVQ    AX, 8(SP); data 放到 8(SP)
    0x005c 00092 (main3.go:19)  CALL    "".sendNotification(SB) ; 前面两行做给一个 interface 参数，调用此函数
```
跟**值接收，值调用**不通的点有:
1. 变量`u`是`user`类型的变量的地址，需要通过`runtime.newobject`申请新的地址
2. 获取`itab`类型的地址方式不一样: 在`go.itab` 和`user`之间多了一个`*`号
```x86asm
    LEAQ    go.itab.*"".user,"".notifer(SB), CX
```
 关于这个符号的具体细节如下:
```x86asm
go.itab.*"".user,"".notifer SRODATA dupok size=32
    0x0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
    0x0010 c9 1b ab 4c 00 00 00 00 00 00 00 00 00 00 00 00  ...L............
    rel 0+8 t=1 type."".notifer+0
    rel 8+8 t=1 type.*"".user+0
    rel 24+8 t=1 "".(*user).notify+0
go.itablink.*"".user,"".notifer SRODATA dupok size=8
    0x0000 00 00 00 00 00 00 00 00                          ........
    rel 0+8 t=1 go.itab.*"".user,"".notifer+0
```
 可以看到唯一不一样的地方就是:
`rel 8+8 t=1 type.*"".user+0`: 把 user地址类型放到`_type`字段的位置。

从上面的代码可以看出，不一样的地方就是 `_type`这个字段，函数的调用都是一样的，所以**值接收** 也可以用指针类型调用。
#### 指针接收，指针调用
```go
package main

import "fmt"

type notifer interface {
    notify()
}

type user struct {
    id int32
}

func (u *user) notify() {
    fmt.Printf("Sending user email to %d\n", u.id)
}

func main() {
    u := &user{9527}
    sendNotification(u)
}

func sendNotification(n notifer) {
    n.notify()
}
```
这种方式其实跟**值接收, 指针调用** 基本上是一样的，`interface.itab` 和`interface.data`都是一样的。

#### 指针接收，值调用
```go
package main

import "fmt"

type notifer interface {
    notify()
}

type user struct {
    id int32
}

func (u *user) notify() {
    fmt.Printf("Sending user email to %d\n", u.id)
}

func main() {
    u := user{9527}
    sendNotification(u)
}

func sendNotification(n notifer) {
    n.notify()
}
```
这种方式编译时无法通过的，报错如下:
```
./main3.go:22:18: cannot use u (type user) as type notifer in argument to sendNotification:
        user does not implement notifer (notify method has pointer receiver)
```
编译器认为`user`并没有实现`notifer`接口, 为什么呢？为了一探究竟，我们改一下上面的代码:
```go
package main

import "fmt"

type notifer interface {
    notify()
}

type user struct {
    id int32
}

func (u user) notify() {
    fmt.Printf("Sending user email to %d\n", u.id)
}

func (u *user) ptrnotify() {
    fmt.Printf("Sending user email to %d\n", u.id)
}

func main() {
    u1 := user{9527}
    u2 := &user{9527}

    sendNotification(u1)
    sendNotification(u2)
}

func sendNotification(n notifer) {
    n.notify()
}
```
`notify` 函数是值接收者，`ptrnotify`是指针接收者, 观察一下生成的汇编:
```x86asm

"".notifer.notify STEXT dupok size=92 args=0x10 locals=0x10
...
"".(*user).notify STEXT dupok size=108 args=0x8 locals=0x18
...
"".user.notify STEXT size=206 args=0x8 locals=0x80
...
"".(*user).ptrnotify STEXT size=229 args=0x8 locals=0x88
"".main STEXT size=185 args=0x0 locals=0x40
...
"".sendNotification STEXT size=68 args=0x10 locals=0x10
...
"".init STEXT size=100 args=0x0 locals=0x8
...
```
发现生成的函数中`notify`既实现了**值接收**类型的函数，又实现了**指针接收**类型的函数, 所以`notify`对于`user`和`*user`类型都可以调用 
而`ptrnoitfy`函数只实现了**指针接收** 类型的函数，没有实现**值接收**类型的函数，所以无法通过`user`类型调用这个函数
那么为什么会有这个限制呢？ 因为**编辑器不是总能自动获取一个值得地址。**, 看一下下面的代码:
```go
package main

import "fmt"

type duration int

func (d *duration) pretty() string {
    return fmt.Sprintf("Duration: %d", *d)
}

func main() {
    duration(42).pretty()
}
```
 运行时报错:
```
# command-line-arguments
./main2.go:12:14: cannot call pointer method on duration(42)
./main2.go:12:14: cannot take the address of duration(42)
```
 如果改一下:
```go
package main

import "fmt"

type duration int

func (d *duration) pretty() string {
    return fmt.Sprintf("Duration: %d", *d)
}

func main() {
    d := duration(42)
    d.pretty()
}
```
 则可以正常运行，证明第一种方式没有中间变量, 所以`duration(42)`是一个常量，常量无法取地址。

[Go 和 interface 探究](http://xargin.com/go-and-interface/)
[go addressable 详解](https://colobu.com/2018/02/27/go-addressable/)
《go in action 中文版》p98-p103
