title: go converte between string and byte slice
date: 2019-07-12 13:17:10
tags:
---

## String
Go第一版代码`c`实现, 在`runtime/runtime.h`里:
```c
typedef struct  String      String;

struct String
{
    byte*   str;
    intgo   len;
};

extern  String  runtime·emptystring;
```

可以看到`Go`中的`string`类型其实就是`String`这个类型。
之后`Go`实现了自举，从`runtime/string.go`中可以看到之前的影子:
```go
type stringStruct struct {
    str unsafe.Pointer
    len int
}
```
### todo: 如何通过编译过程查找对应的类型定义


## Byte 
`byte`的类型定义在 `builtin/builtin.go`中:
```go
// uint8 is the set of all unsigned 8-bit integers.
// Range: 0 through 255.
type uint8 uint8

// byte is an alias for uint8 and is equivalent to uint8 in all ways. It is
// used, by convention, to distinguish byte values from 8-bit unsigned
// integer values.
type byte = uint8
```
可以看到其实`byte`是`uint8`的类型别名
## Slice
```go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```
## string to byte slice
写一个`string`强制类型转换为`[]byte`的`demo`:
```go
package main

import "fmt"

func main() {
    var s = "strings"
    var b = []byte(s)
    fmt.Printf("%v\n", b)
}
```
通过命令:
```sh
 go tool compile -S -N -l main.go
```
编译出汇编指令:
```x86asm
"".main STEXT size=317 args=0x0 locals=0xa8
    0x0000 00000 (main.go:5)    TEXT    "".main(SB), ABIInternal, $168-0
    ...
    0x002f 00047 (main.go:6)    LEAQ    go.string."strings"(SB), AX
    0x0036 00054 (main.go:6)    MOVQ    AX, "".s+80(SP) # 把string内容放到这个位置
    0x003b 00059 (main.go:6)    MOVQ    $7, "".s+88(SP) # 把string长度放到这个位置
    ...
    0x005a 00090 (main.go:7)    CALL    runtime.stringtoslicebyte(SB)
    ...
    0x008e 00142 (main.go:8)    CALL    runtime.convTslice(SB)
    ...
```
上面可以看出当定义一个`string`时，其实会存储`string`的内容和长度, 对应前讲的`string`的结构:
```
struct String
{
    byte*   str;
    intgo   len;
};
```
然后又调用了`runtime.stringtoslicebyte(SB)`, 在`runtime/string.go`中:
```go
func stringtoslicebyte(buf *tmpBuf, s string) []byte {
    var b []byte
    if buf != nil && len(s) <= len(buf) { // 如果字符串的长度小于buf长度，直接使用buf
        *buf = tmpBuf{}
        b = buf[:len(s)]
    } else {
        b = rawbyteslice(len(s)) // 否则调用这个进行内存申请
    }
    copy(b, s) // 内存 copy
    return b
}
```
`buf`默认值是`32`:
```go
const tmpStringBufSize = 32

type tmpBuf [tmpStringBufSize]byte
```
如果不满足长度，申请的内存大小为`len(s)`:
```go
// rawbyteslice allocates a new byte slice. The byte slice is not zeroed.
func rawbyteslice(size int) (b []byte) {
    cap := roundupsize(uintptr(size))
    p := mallocgc(cap, nil, false)
    if cap != uintptr(size) {
        memclrNoHeapPointers(add(p, uintptr(size)), cap-uintptr(size))
    }

    // 下面是类型的转换，把申请的内存变成一个slice结构，赋值给b的地址
    *(*slice)(unsafe.Pointer(&b)) = slice{p, size, int(cap)}
    return
}
```
上面的过程重点有三个:
1. 当长度小于`32`时，直接使用临时内存地址
2. 当长度大于`32`时，需要申请新的长度为`len(s)`的内存地址
3. 需要进行内存的`copy`

## byte slice to string
下面返回来，把一个`[]byte`转换为`string`:
```go
package main

import "fmt"

func main() {
    var b = []byte{101, 102, 103}
    var s = string(b)
    fmt.Printf("%v\n", s)
}
```
生成的汇编代码是:
```x86asm
"".main STEXT size=371 args=0x0 locals=0xb8
    0x0000 00000 (main.go:5)    TEXT    "".main(SB), ABIInternal, $184-0
    ...
    0x005b 00091 (main.go:6)    MOVQ    AX, "".b+128(SP) # 把slice内容放到这个位置
    0x0063 00099 (main.go:6)    MOVQ    $3, "".b+136(SP) # 把slice len 放到这个位置
    0x006f 00111 (main.go:6)    MOVQ    $3, "".b+144(SP) # 把slice cap 放到这个位置
    ...
    0x00a2 00162 (main.go:7)    CALL    runtime.slicebytetostring(SB) #调用这个函数进行转换
    ...
    0x00c4 00196 (main.go:8)    CALL    runtime.convTstring(SB)
    ...
```
上面可以看出当定义一个`slice`时，其实会存储`slice`的内容和长度和容量, 对应之前讲的`slice`的结构:
```go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```
然后调用`runtime.slicebytetostring`函数, 在`runtime/string.go`中:
```go
// Buf is a fixed-size buffer for the result,
// it is not nil if the result does not escape.
func slicebytetostring(buf *tmpBuf, b []byte) (str string) {
    l := len(b)
    if l == 0 {
        // Turns out to be a relatively common case.
        // Consider that you want to parse out data between parens in "foo()bar",
        // you find the indices and convert the subslice to string.
        // 长度为0，直接返回空字符串
        return ""
    }

    ...

    // 长度为1，直接返回staticbytes[b[0]]这个提前设定好的地址内容
    if l == 1 {
        // stringStruct结构的str字段指向对应的值得地址
        stringStructOf(&str).str = unsafe.Pointer(&staticbytes[b[0]])
        // stringStruct结构的len字段设置为1
        stringStructOf(&str).len = 1
        return
    }

    var p unsafe.Pointer
    if buf != nil && len(b) <= len(buf) {
        p = unsafe.Pointer(buf)
    } else {
        p = mallocgc(uintptr(len(b)), nil, false)
    }
    stringStructOf(&str).str = p
    stringStructOf(&str).len = len(b)
    memmove(p, (*(*slice)(unsafe.Pointer(&b))).array, uintptr(len(b)))
    return
}

```
