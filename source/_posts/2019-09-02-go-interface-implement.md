title: Go Interface 源码解析
date: 2019-09-02 10:18:00
tags: [Go]
---

> 本文基于`go1.12.4`源码

## 源码
### 类型定义:
`runtime/runtime2.go`

#### 不含`method`的`interface`
```go
type eface struct {
    _type *_type
    data  unsafe.Pointer
}
```

#### 包含`method`的`interface`
```go
type iface struct {
    tab  *itab
    data unsafe.Pointer
}
```
## `eface` 分析
首先写一个`eface`的具体`case`:
```go
package main

import (
        "fmt"
        "unsafe"
)

func main() {
    num := 3
    var inter interface{} = num
    e := *(*eface)(unsafe.Pointer(&inter))
    fmt.Printf("%+v\n", e)
    fmt.Printf("%+v\n", *(*int)(e.data))
    fmt.Printf("%+v\n", &num)
}

type eface struct {
    _type *_type
    data  unsafe.Pointer
}

type _type struct {
    size       uintptr
    ptrdata    uintptr // size of memory prefix holding all pointers
    hash       uint32
    tflag      tflag
    align      uint8
    fieldalign uint8
    kind       uint8
    alg        *typeAlg
    // gcdata stores the GC type data for the garbage collector.
    // If the KindGCProg bit is set in kind, gcdata is a GC program.
    // Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
    gcdata    *byte
    str       nameOff
    ptrToThis typeOff
}

type tflag uint8

type typeAlg struct {
    // function for hashing objects of this type
    // (ptr to object, seed) -> hash
    hash func(unsafe.Pointer, uintptr) uintptr
    // function for comparing objects of this type
    // (ptr to object A, ptr to object B) -> ==?
    equal func(unsafe.Pointer, unsafe.Pointer) bool
}

type nameOff int32
type typeOff int32
```

对应汇编:
```x86asm
"".main STEXT size=782 args=0x0 locals=0x128
    0x0000 00000 (main.go:8)    TEXT    "".main(SB), ABIInternal, $296-0
    ...
    0x0032 00050 (main.go:9)    LEAQ    type.int(SB), AX ;  通过 type.int  把 int 类型 转换为 *_type  类型，并把地址放到寄存器 AX
    0x0039 00057 (main.go:9)    MOVQ    AX, (SP); 把 AX 内容放到栈底, 这里是下一个函数 newobject 调用的参数
    0x003d 00061 (main.go:9)    CALL    runtime.newobject(SB) ; 函数调用
    0x0042 00066 (main.go:9)    MOVQ    8(SP), AX; newobject 返回值放到8(SP), 然后再放到 AX
    0x0047 00071 (main.go:9)    MOVQ    AX, "".&num+128(SP); 返回值从 AX 放到 &num 变量位置, 表示 num 的地址
    0x004f 00079 (main.go:9)    MOVQ    $3, (AX); (AX)表示 AX 的地址对应的值, 赋值为 3
    0x0056 00086 (main.go:10)   MOVQ    "".&num+128(SP), AX ; num 地址的值赋值个 AX 
    0x005e 00094 (main.go:10)   MOVQ    (AX), AX ; AX 值是地址，其对应的值赋值给 AX, 也就是常量 3
    0x0061 00097 (main.go:10)   MOVQ    AX, ""..autotmp_7+64(SP); 把这个值赋值给一个临时变量 autotmp_7
    0x0066 00102 (main.go:10)   MOVQ    AX, (SP) ; 把值赋值给SP，栈底，是下一个函数 convT64 的参数
    0x006a 00106 (main.go:10)   CALL    runtime.convT64(SB); 调用 runtime.convT64, 参数为 uint64, 返回值为 unsafe.Pointer
    0x006f 00111 (main.go:10)   MOVQ    8(SP), AX; convT64 返回值放到8(SP), 并且赋值到 AX
    0x0074 00116 (main.go:10)   MOVQ    AX, ""..autotmp_8+80(SP); AX 的值放到临时变量 `autotmp_8`
    0x0079 00121 (main.go:10)   LEAQ    type.int(SB), CX ; 通过 type.int  把 int 类型 转换为 *_type  类型，并把地址放到寄存器 CX
    0x0080 00128 (main.go:10)   MOVQ    CX, "".inter+136(SP); 将 CX 中代表*_type 地址的值放到 inter 变量eface类型的 _type 变量中
    0x0088 00136 (main.go:10)   MOVQ    AX, "".inter+144(SP); 将 AX 中代表 convT64返回值 3 的 unsafe.Pointer 类型 放到 inter 变量 eface 类型的 data 中 
    ...
```
 这里只关注`main.go:9`和`main.go:10`的处理, 对应代码为:
    ```go
    8 num := 3
    ```
初始化变量`num`为`int`类型，并赋值为`3`, 具体的过程已经在前面汇编代码中通过注释的方式标出，下面来看一些细节:
1. 新建 `int` 类型变量需要申请内存, 通过`runtime.newobject`来申请
2. `type.int`  可以获取 `int`类型的 `_type` 结构的地址 `*_type` (具体实现方式被编译优化了，需要再进一步深究)
3. `MOVQ AX, (SP)`是为了把前面放到`AX`的`*_type` 放到栈底, 这个位置下面调用函数`runtime.newobject`的参数, 具体函数实现:
    ```go
    func newobject(typ *_type) unsafe.Pointer {
        return mallocgc(typ.size, typ, true)
    }
    ```
4.  执行完函数调用用会把返回值`unsafe.Pointer`放到`8(SP)`, 然后在放入 `AX`
5. `MOV $3,(AX)` 向表示寄存器`AX`包含的地址对应的值设置为常量`3`
6. `autotmp`是一个临时变量，是为了在程序内复用全局临时变量, 防止变量被修改:
    https://github.com/golang/go/issues/21557, 
    https://github.com/golang/go/issues/29547
    **具体需要再深入研究**

```go
9    var inter interface{} = num
```
 初始化变量`inter`,类型为`interface{}`, 并且指向`num`, 具体过程参考上面的注释部分，一些细节:
1. 这里会调用`runtime.convT64`函数，定义如下: ( 在`go 1.8`版本调用的是`runtime.convT2E`, 在`go1.10`调用的是`runtime.convT2E64`):
```go
func convT64(val uint64) (x unsafe.Pointer) {
    if val == 0 {
        x = unsafe.Pointer(&zeroVal[0])
    } else {
        x = mallocgc(8, uint64Type, false)
        *(*uint64)(x) = val
    }
    return
}
```
 这里的入参就是 `num` 的值 `3`, 返回值是转换为`uint64`的值，并且申请一个地址，值为`3`, **注意: 这里发生了值的`copy`**
2. 通过最后两行赋值`inter`, `inter`类型为`eface`, 定义前面提过，最终给`eface._type` 和`eface.data`赋值
3. `eface.type`的查看这里还没有找到好的方法来查看，`dlv`无法深入到内置的实现。

为了查看`_type` 类型，定义了一个跟 `runtime` 内部实现一样的数据结构，并且通过`unsafe`强制进行数据类型转换，可以得到`_type`的值。
借助`dlv`对上面的程序进行`debug`:
1. 在` e := *(*eface)(unsafe.Pointer(&inter))`处打断点
2. 执行到上面这行后打印出`e`的值:
```
(dlv) p e
main.eface {
        _type: *main._type {
                size: 8,
                ptrdata: 0,
                hash: 4149441018,
                tflag: 7,
                align: 8,
                fieldalign: 8,
                kind: 130,
                alg: *(*main.typeAlg)(0x57adf0),
                gcdata: *1,
                str: 1059,
                ptrToThis: 47520,},
        data: unsafe.Pointer(0xc000080018),}

```
 下面是`*_type`源码的定义:
```go
// 所有类型信息结构体的公共部分
// src/rumtime/runtime2.go
type _type struct {
    size       uintptr  // 类型的大小
    ptrdata    uintptr  // size of memory prefix holding all pointers
    hash       uint32   // 类型的Hash值
    tflag      tflag    // 类型的Tags 
    align      uint8    // 结构体内对齐
    fieldalign uint8    // 结构体作为field时的对齐
    kind       uint8    // 类型编号 定义于runtime/typekind.go
    alg        *typeAlg // 类型元方法 存储hash和equal两个操作。map key便使用key的_type.alg.hash(k)获取hash值
    gcdata    *byte     // GC相关信息
    str       nameOff   // 类型名字的偏移    
    ptrToThis typeOff    
}
```
为了查看`eface.data`的值，可以通过
```go
 fmt.Printf("%+v\n", *(*int)(e.data))
```
 输出，可以看到运行结果为`3`, 正是`num`的值。

## `iface` 分析

```go
package main

import (
        "fmt"
        "unsafe"
)

type Mather interface {
        Add(a, b int32) int32
        Sub(a, b int64) int64
}

type Adder struct{ id int32 }

//go:noinline
func (adder Adder) Add(a, b int32) int32 { return a + b }

//go:noinline
func (adder Adder) Sub(a, b int64) int64 { return a - b }

func main() {
        adder := Adder{id: 6754}
        m := Mather(adder)
		m.Add(12, 2)
		m.Sub(19, 4)
        i := *(*iface)(unsafe.Pointer(&m))
        fmt.Printf("%+v\n", i)
		fmt.Printf("%+v\n", *(*Adder)(i.data))
}

type iface struct {
        tab  *itab
        data unsafe.Pointer
}

type itab struct {
        inter *interfacetype
        _type *_type
        hash  uint32 // copy of _type.hash. Used for type switches.
        _     [4]byte
        fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter
}

type interfacetype struct {
        typ     _type
        pkgpath name
        mhdr    []imethod
}

// See reflect/type.go for details.
type name struct {
        bytes *byte
}
type imethod struct {
        name nameOff
        ityp typeOff
}

type _type struct {
        size       uintptr
        ptrdata    uintptr // size of memory prefix holding all pointers
        hash       uint32
        tflag      tflag
        align      uint8
        fieldalign uint8
        kind       uint8
        alg        *typeAlg
        // gcdata stores the GC type data for the garbage collector.
        // If the KindGCProg bit is set in kind, gcdata is a GC program.
        // Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
        gcdata    *byte
        str       nameOff
        ptrToThis typeOff
}

type tflag uint8

type typeAlg struct {
        // function for hashing objects of this type
        // (ptr to object, seed) -> hash
        hash func(unsafe.Pointer, uintptr) uintptr
        // function for comparing objects of this type
        // (ptr to object A, ptr to object B) -> ==?
        equal func(unsafe.Pointer, unsafe.Pointer) bool
}

type nameOff int32
type typeOff int32
```
对应的汇编代码:
```x86asm
"".main STEXT size=377 args=0x0 locals=0xc8
    0x002f 00047 (main.go:22)   MOVL    $0, "".adder+68(SP) ; 初始化一个addr, 默认值都是空的
    0x0037 00055 (main.go:22)   MOVL    $6754, "".adder+68(SP) ; 对 Addr.id 字段进行赋值
    0x003f 00063 (main.go:23)   MOVL    $6754, (SP) ; 将id的值放到栈底，作为 convT32的参数
    0x0046 00070 (main.go:23)   CALL    runtime.convT32(SB) ; 入参是 id, 输出的 值是转换后的unsafe.Pointer
    0x004b 00075 (main.go:23)   MOVQ    8(SP), AX ; 返回值从 8(SP) 的位置，复制到寄存器 AX
    0x0050 00080 (main.go:23)   MOVQ    AX, ""..autotmp_5+80(SP) ; 将返回值从 AX 复制到临时变量autotmp_5 的位置
    0x0055 00085 (main.go:23)   LEAQ    go.itab."".Adder,"".Mather(SB), CX ; 将 Adder 的 itab转换为 Mather 类型，并将地址放到 CX
    0x005c 00092 (main.go:23)   MOVQ    CX, "".m+104(SP) ; 将 CX 寄存器中的 itab 地址赋值给 m.tab
    0x0061 00097 (main.go:23)   MOVQ    AX, "".m+112(SP) ; 将 AX 寄存器中的 unsafe.Pointer 的 表示的值，赋值给 m.data
```

重点关注`main.go:22` 和`main.go:23`, 首先看一下`main.go:22`:
```go
adder := Adder{id: 6754}
```
需要注意的细节:
1. `struct` 的 赋值是先对其赋值为空，然后再一个字段一个字段赋值，字段在栈中的排列跟定义的顺序有关系，是紧密排列的，并且存在对齐的问题

然后看如何把`Adder`类型转换为`Mather`接口类型的:
```go
m := Mather(adder)
```
具体赋值的 步骤前面汇编部分的注释已经说明了， 这里需要注意的几个细节:
1. `iface`类型与`eface`不通，有`tab`和`data` 两个字段，分别是`*itab`和`unsafe.Pointer`两个类型
2. `go.itab."".Adder,"".Mather(SB), CX` 将 Adder 的 itab转换为 Mather 类型，并将地址放到 CX, **具体如何实现的，还没有找到查看方法，需要继续研究**

`itab`类型也无法直接查看，这里通过`unsafe`进行转换，在通过`dlv`进行查看:
1. 转换代码:
```go
i := *(*iface)(unsafe.Pointer(&m))
```
2. 通过 dlv 在此处打断点查看:
```
(dlv) p i
main.iface {
        tab: *main.itab {
                inter: *(*main.interfacetype)(0x4c0360),
                _type: *(*main._type)(0x4c63e0),
                hash: 1633631626,
                _: [4]uint8 [0,0,0,0],
                fun: [1]uintptr [4867136],},
        data: unsafe.Pointer(0xc000080010),}

(dlv) p i.tab
*main.itab {
        inter: *main.interfacetype {
                typ: (*main._type)(0x4c0360),
                pkgpath: (*main.name)(0x4c0390),
                mhdr: []main.imethod len: 2, cap: 2, [
                        (*main.imethod)(0x4c03c0),
                        (*main.imethod)(0x4c03c8),
                ],},
        _type: *main._type {
                size: 4,
                ptrdata: 0,
                hash: 1633631626,
                tflag: 7,
                align: 4,
                fieldalign: 4,
                kind: 153,
                alg: *(*main.typeAlg)(0x57bde0),
                gcdata: *1,
                str: 14386,
                ptrToThis: 105952,},
        hash: 1633631626,
        _: [4]uint8 [0,0,0,0],
        fun: [1]uintptr [4867136],}
```
关于`itab`类型是包含接口的静态类型信息、数据的动态类型信息、函数表的结构, 在源码中的定义如下:
```
type itab struct {
    inter *interfacetype //  本实例所实现的接口的类型信息,  用于定位到具体的 interface, 这个是在编译时确定的
    _type *_type //  本实例的具体数据的类型信息, 参考前面 _type 类型的定义 
    hash  uint32 // copy of _type.hash. Used for type switches.
    _     [4]byte
    // fun 表示的 interface 里面的 method 的具体实现
    // 这里放置和接口方法对应的具体数据类型的方法地址
    // 实现接口调用方法的动态分派，一般在每次给接口赋值发生转换时
    // 会更新此表，或者直接拿缓存的itab
    fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}

// interfacetype 只是对于 _type 的一种包装，在其顶部空间还包装了额外的 interface 相关的元信息
type interfacetype struct {
    typ     _type // 所实现的接口的类型
    pkgpath name  // 所实现的接口的定义路径
    mhdr    []imethod //   所实现的接口在定义时的函数声明列表
}

//这里的 method 只是一种函数声明的抽象，比如  func Print() error
type imethod struct {
    name nameOff
    ityp typeOff
}
```

需要注意的点:
1. `func`表示的 interface 里面的 method 的具体实现, 比如这里的两个方法`Sub`和`Add`， 但是`func`的长度为1， 该如何表示多个方法呢?
 看一下函数调用:
```go
    m.Add(12, 2)
    m.Sub(19, 4)
```
对应的汇编:

```x86asm
    0x0066 00102 (main.go:24)   TESTB   AL, (CX); 求与 AL & (CX), 检查 CX 是否为 nil, AL 是 AX 的低8位, AH 是 AX 的高8位
    0x0068 00104 (main.go:24)   MOVQ    go.itab."".Adder,"".Mather+24(SB), CX ; Add函数的入口地址放到 CX
    0x006f 00111 (main.go:24)   MOVQ    AX, (SP)
    0x0073 00115 (main.go:24)   MOVQ    $8589934604, AX ; 将 12, 2两个参数赋值到 AX 中
    0x007d 00125 (main.go:24)   MOVQ    AX, 8(SP); 将 AX 中的参数赋值到 8(SP) 位置
    0x0082 00130 (main.go:24)   CALL    CX ; 调用 m.Add
    0x0084 00132 (main.go:25)   MOVQ    "".m+104(SP), AX ; 前面看到这个位置是 m.tab 的值
    0x0089 00137 (main.go:25)   TESTB   AL, (AX); 检查 AX 是否为 nil
    0x008b 00139 (main.go:25)   MOVQ    32(AX), AX ; (AX)地址 + 32 偏移，指向 Sub
    0x008f 00143 (main.go:25)   MOVQ    "".m+112(SP), CX ; 这个位置是 m.data 的值
    0x0094 00148 (main.go:25)   MOVQ    CX, (SP)
    0x0098 00152 (main.go:25)   MOVQ    $19, 8(SP) ; 把参数 19 放到 8(SP) 位置
    0x00a1 00161 (main.go:25)   MOVQ    $4, 16(SP) ; 把参数 4 放到 16(SP) 位置
    0x00aa 00170 (main.go:25)   CALL    AX ; 调用 m.Sub
```

 需要注意的细节:
1. `TESTB AL, (CX)`是把 `AL & (CX)` 位与的值放到 (CX) 中, 参考: https://github.com/golang/go/issues/10432 & https://github.com/golang/go/issues/27180, 这个步骤其实是为了检查 CX 是否为 nil， 如果是 nil 就没法调用这个函数了

2. `MOVQ go.itab."".Adder,"".Mather+24(SB), CX` 这个为什么是取到了`Add`的地址?
看一下`itab`的定义:
```go
type itab struct {
    inter *interfacetype
    _type *_type
    hash  uint32 
    _     [4]byte
    fun   [1]uintptr
}
```
	当前机器是64位的，所以可以看出`func` 相对于`itab`起始地址的偏移量为: 
 `8(*interfacetype) + 8(*_type) + 4(uint32) + 4(byte=uint8) = 24`
 所以 `MOVQ go.itab."".Adder,"".Mather+24(SB)` 其实就是`func`的第一个函数`Add`的地址

3. 函数`Sub`的地址为什么是`32(AX)`?
可以从前面一句`MOVQ    "".m+104(SP), AX` 得出: `AX`目前指向的是`m.tab`, 也就是`itab`类型的起始地址，`32(AX)`就是相对`AX`有32位的偏移，前面说了相对`itab` `24`位的偏移其实时`Add`函数，然后对于64位系统，函数地址占`8`位，所以`32(AX)`就是下一个函数`Sub`的地址。 

4. 前面只有两个函数，我们如果调换一下接口定义中两个函数的位置，发现生成的汇编是一样的，也就是:**函数顺序与定义的顺序无关**, 如果增加几个函数就可以看出来，其实:** 函数在`func` 中的顺序是按照函数名的字典顺序排列的**

5. ` MOVQ    $8589934604, AX` 为什么是参数赋值?
 这个其实我们可以对常量`8589934604` 进行分析，首先把它转化为二进制:
```
echo 'obase=2;8589934604' | bc
1000000000000000000000000000001100
```
 得到的数据其实是:
```
  +------------------------------------------+
  | 0000001000000000000000000000000000001100 |
  +------------------------------------------+
    \______/\______________________________/  
     +---+             +----+                 
     | 2 |             | 12 |                 
     +---+             +----+                 
```
 其实就是`2`和`12`两个`8`字节的数据组合在一起放到了`AX`寄存器中, 正是`Add(12,2)`的两个参数。

## 断言

`interface{}` 是一个抽象的类型，如果需要转换为具体的类型，则需要类型断言, 类型断言其实有两个:
1. 类型判断: 判断类型是否一致
2. 类型转换: 类型一致取出具体的数据

下面看一个例子:
```go
package main

var j uint32
var r int32
var ok bool
var eface interface{}

func assertion() {
    i := uint64(42)
    eface = i
    j = eface.(uint32)
    r, ok = eface.(int32)
}
```
对应的汇编语言如下:
```x86asm
    0x0066 00102 (eface.go:11)  MOVL    $0, ""..autotmp_1+36(SP) ; 初始化0值
    0x006e 00110 (eface.go:11)  MOVQ    "".eface+8(SB), AX ; 把 data 放到 AX 寄存器
    0x0075 00117 (eface.go:11)  MOVQ    "".eface(SB), CX ; 把 _type 放到 CX 寄存器
    0x007c 00124 (eface.go:11)  LEAQ    type.uint32(SB), DX ; 把 uint32的_type 值放到 DX 寄存器
    0x0083 00131 (eface.go:11)  CMPQ    CX, DX; 比较 eface._type == uint32 ?
    0x0086 00134 (eface.go:11)  JEQ 138 ; JEQ = jump if equal, 如果类型相等就跳转到 138行
    0x0088 00136 (eface.go:11)  JMP 246 ;  类型不匹配, 跳转到 246 行, 出现 panic
    0x008a 00138 (eface.go:11)  MOVL    (AX), AX ; JEQ 跳转的行 138, 把(AX)地址对应的值放到 AX 寄存器，也就是 eface.data
    0x008c 00140 (eface.go:11)  MOVL    AX, ""..autotmp_1+36(SP) ; 临时变量赋值
    0x0090 00144 (eface.go:11)  MOVL    AX, "".j(SB) ; 赋值给变量 j
    0x0096 00150 (eface.go:12)  MOVQ    "".eface+8(SB), AX ;  把 data 放到 AX 寄存器
    0x009d 00157 (eface.go:12)  LEAQ    type.int32(SB), CX ; 把 int32 的_type 值放到 DX 寄存器
    0x00a4 00164 (eface.go:12)  CMPQ    "".eface(SB), CX ; 比较 eface._type == int32 ?
    0x00ab 00171 (eface.go:12)  JEQ 175 ; 如果类型相等就跳转到 175 行
    0x00ad 00173 (eface.go:12)  JMP 223 ; 跳转到 223 行，输出 panic
    0x00af 00175 (eface.go:12)  MOVL    (AX), AX; 类型相等就把 data 放到 AX
    0x00b1 00177 (eface.go:12)  MOVL    $1, CX 把常量 1 放到 CX
    0x00b6 00182 (eface.go:12)  JMP 184 ; 调到 184 行
    0x00b8 00184 (eface.go:12)  MOVL    AX, ""..autotmp_2+32(SP)
    0x00bc 00188 (eface.go:12)  MOVB    CL, ""..autotmp_3+31(SP) ; CL 是 CX 的低 8 位
    0x00c0 00192 (eface.go:12)  MOVL    ""..autotmp_2+32(SP), AX
    0x00c4 00196 (eface.go:12)  MOVL    AX, "".r(SB) ; AX 是 data 的值， 放到 r 变量中
    0x00ca 00202 (eface.go:12)  MOVBLZX ""..autotmp_3+31(SP), AX; MOVBLZX 用 0 扩展，放到 autotmp_3 变量
    0x00cf 00207 (eface.go:12)  MOVB    AL, "".ok(SB); AL 低8位赋值给 ok ，因为ok 是 bool 类型的， 根据字节对齐，占 8 位
    0x00d5 00213 (eface.go:13)  MOVQ    56(SP), BP
    0x00da 00218 (eface.go:13)  ADDQ    $64, SP
    0x00de 00222 (eface.go:13)  RET
    0x00df 00223 (eface.go:13)  XORL    AX, AX ; eface.type != int32 情况下，执行本行, XOR是异或，所以 AX^AX , 结果为 0
    0x00e1 00225 (eface.go:13)  XORL    CX, CX; 同上， CX 结果为 0
    0x00e3 00227 (eface.go:12)  JMP 184 ; 跳转到 184 行执行，这里要注意的是 AX, CX 寄存器已经为0， 所有后面 ok 的值也位0了
    0x00e5 00229 (eface.go:10)  LEAQ    "".eface+8(SB), DI
    0x00ec 00236 (eface.go:10)  CALL    runtime.gcWriteBarrier(SB)
    0x00f1 00241 (eface.go:10)  JMP 102
    0x00f6 00246 (eface.go:11)  MOVQ    CX, (SP) ; 一个返回值表达式类型不匹配时，执行到这里, CX 类型值放到(SP)作为第一个参数
    0x00fa 00250 (eface.go:11)  MOVQ    DX, 8(SP); 想要的类型从 DX 放到 8(SP) 作为第二个参数
    0x00ff 00255 (eface.go:11)  LEAQ    type.interface {}(SB), AX ; interface 类型的地址放到 AX
    0x0106 00262 (eface.go:11)  MOVQ    AX, 16(SP); AX 值放到 16(SP) 作为第三个参数
    0x010b 00267 (eface.go:11)  CALL    runtime.panicdottypeE(SB); 执行函数调用，使用前面的三个参数，返回 panic
```
 需要注意的点:
1. 当使用返回值为一个的表达式时，如果出现类型不匹配，会触发`panic`
2. 当使用两个返回值的表达式时,  `r`, `ok`的值随着`AX`, `CX`的值 改变:
分为两种情况: 
当类型相等时: `AX` 值为`eface.data`, `CX` 的值为`1`
赋值的过程如下:
```x86asm
    0x00b8 00184 (eface.go:12)  MOVL    AX, ""..autotmp_2+32(SP)
    0x00bc 00188 (eface.go:12)  MOVB    CL, ""..autotmp_3+31(SP) ; CL 是 CX 的低 8 位, CX 是 1, 二进制是: 0000000000000001; CL 就是: 00000001
    0x00c0 00192 (eface.go:12)  MOVL    ""..autotmp_2+32(SP), AX
    0x00c4 00196 (eface.go:12)  MOVL    AX, "".r(SB) ; AX 是 data 的值， 放到 r 变量中
    0x00ca 00202 (eface.go:12)  MOVBLZX ""..autotmp_3+31(SP), AX; MOVBLZX 用 0 扩展，放到 autotmp_3 变量, autotmp_3 是 00000001, 扩展后是: 0000000000000001
    0x00cf 00207 (eface.go:12)  MOVB    AL, "".ok(SB); AL 低8位赋值给 ok ，因为ok 是 bool 类型的， 根据字节对齐，占 8 位, ok 值为: 00000001
```
 当类型不相等时: `AX`和`CX`的值都初始化位空
```x86asm
    0x00df 00223 (eface.go:13)  XORL    AX, AX ; eface.type != int32 情况下，执行本行, XOR是异或，所以 AX^AX , 结果为 0
    0x00e1 00225 (eface.go:13)  XORL    CX, CX; 同上， CX 结果为 0
    0x00e3 00227 (eface.go:12)  JMP 184 ; 跳转到 184 行执行，这里要注意的是 AX, CX 寄存器已经为0， 所有后面 ok 的值也位0了
```
 赋值过程如下:
```x86asm
    0x00b8 00184 (eface.go:12)  MOVL    AX, ""..autotmp_2+32(SP)
    0x00bc 00188 (eface.go:12)  MOVB    CL, ""..autotmp_3+31(SP) ; CL 是 CX 的低 8 位, CX 是 0, 二进制是: 0000000000000000; CL 就是: 00000000
    0x00c0 00192 (eface.go:12)  MOVL    ""..autotmp_2+32(SP), AX
    0x00c4 00196 (eface.go:12)  MOVL    AX, "".r(SB) ; AX 是 data 的值， 放到 r 变量中, AX 是空值，所以 r == nil
    0x00ca 00202 (eface.go:12)  MOVBLZX ""..autotmp_3+31(SP), AX; MOVBLZX 用 0 扩展，放到 autotmp_3 变量, autotmp_3 是 00000000, 扩展后是: 0000000000000000
    0x00cf 00207 (eface.go:12)  MOVB    AL, "".ok(SB); AL 低8位赋值给 ok ，因为ok 是 bool 类型的， 根据字节对齐，占 8 位, ok 值为: 00000000
```

## 参考文献,
[理解Go语言模型(1)：interface底层详解](https://yougg.github.io/2017/03/27/%E7%90%86%E8%A7%A3go%E8%AF%AD%E8%A8%80%E6%A8%A1%E5%9E%8B1interface%E5%BA%95%E5%B1%82%E8%AF%A6%E8%A7%A3/)
[Go Data Structures: Interfaces](https://research.swtch.com/interfaces)
[Interface Semantics](https://www.ardanlabs.com/blog/2017/07/interface-semantics.html)
[go-internals chapter2 interfacs](https://github.com/two/go-internals/blob/master/chapter2_interfaces/README.md)
