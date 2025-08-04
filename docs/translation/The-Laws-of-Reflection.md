---
draft: true
date: 
  created: 2025-08-03
categories:
  - Reading
tags:
  - Reflection
  - Golang
authors:
  - zhazi
---

# 翻译：The Laws of Reflection

!!! info "文献"

    - [The Laws of Reflection](https://go.dev/blog/laws-of-reflection)

## Introduction

在计算中，反射是指程序检查自身结构的能力，尤其是通过类型来实现(的反射)；它是元编程的一种形式，也是许多困惑的根源。

本文试图通过讲解 Go 语言中的反射机制来厘清这些概念。不同语言的反射模型各不相同（许多语言甚至根本不支持反射），但本文讨论的是 Go，因此下文提到“反射”时，**均特指 Go 语言中的反射。**

> 2022 年 1 月补充：这篇博文写于 2011 年，早于 Go 引入泛型（即参数化多态）。虽然该语言特性并未使文章中的核心内容失效，但我们仍对几处做了微调，以避免让熟悉现代 Go 的读者产生误解。

## Types and interfaces

由于反射建立在类型系统之上，首先让我们回顾一下 Go 中的类型。

Go 是静态类型语言。每个变量都有一个静态类型，也就是说，变量在编译时就确定了唯一的类型：int、float32、\*MyType、\[]byte 等等。如果我们声明

```Go 
type MyInt int

var i int
var j MyInt
```
那么，变量 `i` 的类型是 `int`，而变量 `j` 的类型是 `MyInt`。这两个变量的静态类型不同；尽管它们的底层类型相同，但如果没有显式转换，它们不能互相赋值。

一种重要的类型类别是接口类型，它表示一组固定的方法。（在讨论反射时，我们可以忽略接口定义作为多态代码中的约束使用。）一个接口变量可以存储任何具体的（非接口）值，只要该值实现了接口的方法。一个著名的例子是 `io.Reader` 和 `io.Writer`，即来自 `io` 包的 `Reader` 和 `Writer` 类型：

```go
// Reader is the interface that wraps the basic Read method.
type Reader interface {
    Read(p []byte) (n int, err error)
}

// Writer is the interface that wraps the basic Write method.
type Writer interface {
    Write(p []byte) (n int, err error)
}
```
任何实现了具有此签名的 `Read`（或 `Write`）方法的类型，都可以说实现了 `io.Reader`（或 `io.Writer`）。在本讨论中，这意味着 `io.Reader` 类型的变量可以存储任何具有 `Read` 方法的类型的值：

```go
var r io.Reader
r = os.Stdin
r = bufio.NewReader(r)
r = new(bytes.Buffer)
// and so on
```

需要明确的是，无论具体值 `r` 存储的是什么，`r` 的类型始终是 `io.Reader`：Go 是静态类型语言，`r` 的静态类型是 `io.Reader`。

一个非常重要的接口类型是空接口：

```go
interface{}
```
或其等效的别名，

```go
any
```

它表示一个空的方法集合，并且任何值都可以满足它，因为每个值都有零个或多个方法。

有些人说 Go 的接口是动态类型的，但这是误导性的。它们是静态类型的：接口类型的变量始终具有相同的静态类型，即使在运行时，存储在接口变量中的值可能会改变类型，但该值始终会满足接口。

我们需要对这些内容非常精确，因为反射和接口是密切相关的。

## The representation of an interface

Russ Cox 写了一篇关于 [Go 中接口值表示细节](https://research.swtch.com/2009/12/go-data-structures-interfaces.html)的博客文章。本文无需重复其全部内容，但有必要做一个简化的总结。

接口类型的变量实际上存储一对数据：分配给变量的具体值，以及该值的类型描述符。更准确地说，值是实现了该接口的底层具体数据项，类型则描述了该数据项的完整类型。例如，在

```go
var r io.Reader
tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)
if err != nil {
    return nil, err
}
r = tty
```

之后，形式上可以认为 `r` 内部保存的是一个 `(值, 类型)` 二元组：`(tty, *os.File)`。注意，`*os.File` 除了实现 `Read` 方法外，还实现了其他方法；尽管接口值 `r` 只允许访问 `Read` 方法，但其内部携带的值仍包含该值的完整类型信息。正因如此，我们才能写出如下代码：

```go
var w io.Writer
w = r.(io.Writer)
```
这个赋值表达式是一个类型断言；它断言的是 `r` 中的项也实现了 `io.Writer`，因此我们可以将其赋值给 `w`。赋值后，`w` 将包含一对 `(tty, *os.File)`，这与 `r` 中持有的那一对相同。接口的静态类型决定了可以通过接口变量调用哪些方法，即使内部的具体值可能具有更多的方法。

接下来，我们可以这样做：

```go
var empty interface{}
empty = w
```
我们的空接口值 `empty` 将再次包含相同的那一对 `(tty, *os.File)`。这很方便：一个空接口可以容纳任何值，并且包含我们可能需要的关于该值的所有信息。

（在这里我们不需要类型断言，因为静态类型已经知道 `w` 满足空接口。在将一个值从 `Reader` 转移到 `Writer` 的例子中，我们需要显式地使用类型断言，因为 `Writer` 的方法不是 `Reader` 方法的子集。）

一个重要的细节是，接口变量内部的那对数据始终呈现为 `(值, 具体类型)` 的形式，而不能是 `(值, 接口类型)` 的形式。接口不能持有接口值。

现在我们可以进行反射了。

## The first law of reflection
### 1. Reflection goes from interface value to reflection object.

从最基础的层面来说，反射只是一种机制，用于检查存储在接口变量内部的“类型-值”二元组。要开始使用反射，我们首先需要了解 `reflect` 包中的两个类型：`Type` 和 `Value`。这两个类型提供了访问接口变量内容的能力；而两个简单的函数——`reflect.TypeOf` 和 `reflect.ValueOf`——则可以从一个接口值中提取出对应的 `reflect.Type` 与 `reflect.Value`。（此外，从 `reflect.Value` 也很容易得到对应的 `reflect.Type`，不过目前我们先把 `Value` 和 `Type` 这两个概念分开来看。）

我们先从 `TypeOf` 开始：

```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    var x float64 = 3.4
    fmt.Println("type:", reflect.TypeOf(x))
}
```
此程序将打印

```console
type: float64
```
你可能会想知道这里的接口在哪里，因为程序看起来是在将 `float64` 类型的变量 `x` 传递给 `reflect.TypeOf`，而不是一个接口值。但接口确实存在；正如 godoc 所报告的，`reflect.TypeOf` 的签名中包含了一个空接口：

```go
// TypeOf returns the reflection Type of the value in the interface{}.
func TypeOf(i interface{}) Type
```
当我们调用 `reflect.TypeOf(x)` 时，`x` 会先被存入一个空接口，再作为实参传入；`reflect.TypeOf` 则对这个空接口拆包，以还原出类型信息。

`reflect.ValueOf` 函数当然就是用来还原值的（下文我们将省略样板代码，只保留可执行部分）：

```go
var x float64 = 3.4
fmt.Println("value:", reflect.ValueOf(x).String())
```

打印结果：

```go
value: <float64 Value>
```

（我们显式调用 `String` 方法，因为默认情况下 `fmt` 包会“钻进” `reflect.Value` 内部，把其中存放的具体值打印出来；而 `String` 方法不会这么做。）

`reflect.Type` 和 `reflect.Value` 都有许多方法，可以让我们检查和操作它们。一个重要的例子是，`Value` 有一个 `Type` 方法，返回一个 `reflect.Value` 的 `Type`。另一个例子是，`Type` 和 `Value` 都有一个 `Kind` 方法，返回一个常量，表示存储的项的类型：如 `Uint`、`Float64`、`Slice` 等。另外，`Value` 上还具有一些方法，如 `Int` 和 `Float`，可以让我们获取存储在其中的值（分别作为 `int64` 和 `float64`）：

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("type:", v.Type())
fmt.Println("kind is float64:", v.Kind() == reflect.Float64)
fmt.Println("value:", v.Float())
```

打印结果：

```go
type: float64
kind is float64: true
value: 3.4
```
还有像 `SetInt` 和 `SetFloat` 这样的 methods，但要使用它们，我们需要理解“可设置性”，这是反射定律的第三条法则，下面会讨论。

反射库有几个值得注意的特性。首先，为了保持 API 简洁，`Value` 的“getter”和“setter”方法操作的是能够容纳该值的最大类型：例如，对于所有有符号整数，使用的是 `int64`。也就是说，`Value` 的 `Int` 方法返回的是 `int64`，而 `SetInt` 方法接收的是 `int64`；可能需要将其转换为实际涉及的类型：

```go
var x uint8 = 'x'
v := reflect.ValueOf(x)
fmt.Println("type:", v.Type())                            // uint8.
fmt.Println("kind is uint8: ", v.Kind() == reflect.Uint8) // true.
x = uint8(v.Uint())                                       // v.Uint returns a uint64.
```

第二个性质是：反射对象的 `Kind` 描述的是“底层类型”，而非静态类型。如果反射对象里保存的是一个用户自定义的整型值，比如：

```go
type MyInt int
var x MyInt = 7
v := reflect.ValueOf(x)
```

即使 `x` 的静态类型是 `MyInt` 而不是 `int`，`v` 的 `Kind` 仍然是 `reflect.Int`。换句话说，尽管 `Type` 可以区分 `int` 和 `MyInt`，但 `Kind` 不能。

## The second law of reflection
### 2. Reflection goes from reflection object to interface value.

像物理反射一样，Go 中的反射也会生成它自己的逆操作。

给定一个 `reflect.Value`，我们可以使用 `Interface` 方法恢复一个接口值；实际上，这个方法将类型和值信息重新封装回接口表示，并返回结果：

```go
// Interface returns v's value as an interface{}.
func (v Value) Interface() interface{}
```
因此，我们可以这样写：

```go
y := v.Interface().(float64) // y will have type float64.
fmt.Println(y)
```

从而打印出由反射对象 `v` 表示的那个 `float64` 值。

不过，我们可以做得更好。`fmt.Println`、`fmt.Printf` 等函数的参数都是作为空接口值传递的，然后 `fmt` 包内部会像我们在之前的例子中一样将其解包。因此，要正确打印 `reflect.Value` 的内容，只需要将 `Interface` 方法的结果传递给格式化打印函数：

```go
fmt.Println(v.Interface())
```

（本文最初写成后，fmt 包曾做过一次改动：如今它会自动对 reflect.Value 做这种拆包。因此现在我们只需写成

 ```go
 fmt.Println(v)
 ```

就能得到同样的输出。不过为了清晰，本文仍保留显式的 .Interface() 调用。）

由于我们的值是 float64，还可以用浮点格式打印：

```go
fmt.Printf("value is %7.1e\n", v.Interface())
```

此时输出为

```console
value is 3.4e+00
```
同样地，也无需对 `v.Interface()` 的结果再做 `float64` 的类型断言；空接口内部已经携带了具体值的类型信息，`Printf` 会自动恢复它。

简而言之，`Interface` 方法是 `ValueOf` 函数的逆操作，只是其返回值的静态类型始终为 `interface{}`。

再次强调：反射的过程就是把接口值拆解成反射对象，再重新组装回接口值。

## The third law of reflection
### 3. To modify a reflection object, the value must be settable.

第三条定律最微妙、最容易让人迷惑，但只要从最基础的原则出发，其实也很好理解。

下面这段代码无法运行，但很值得研究：

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
v.SetFloat(7.1) // Error: will panic.
```
运行这段代码时，程序会带着一条晦涩的错误信息 panic：

```console
panic: reflect.Value.SetFloat using unaddressable value
```

问题不在于 7.1 这个值“不可寻址”，而在于 `v` 本身是“不可设置”的。可设置性（settability） 是反射 `Value` 的一种属性，并非所有反射值都具备。

`Value` 的 `CanSet` 方法报告该 `Value` 的可设置性；在我们的例子中：

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("settability of v:", v.CanSet())
```

输出为：

```
settability of v: false
```

在一个不可设置的 `Value` 上调用 `Set` 方法会导致错误。那么，什么是可设置性呢？

可设置性有点像“可寻址”，但条件更严格：它指的是反射对象能否修改创建该反射对象时所使用的**原始存储**。  
可设置性取决于反射对象是否**持有原始项**。当我们写

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
```

时，`reflect.ValueOf` 接收的是 `x` 的**副本**，因此 `v` 内部保存的是这份副本的反射值，而非 `x` 本身。既然 `v` 不持有原始 `x`，它就无法修改 `x` 的存储，于是 `v.CanSet()` 返回 `false`。

于是，如果允许

```go
v.SetFloat(7.1)
```

这条语句成功执行，它并不会修改 `x`，尽管 `v` 看起来是从 `x` 创建的。相反，它只会更新反射值内部的那份 `x` 副本，而 `x` 本身则毫发无损。这种行为既令人困惑又毫无用处，因此被禁止；而“可设置性”正是用来避免这一问题的机制。

如果你觉得这很奇怪，其实并非如此——它只是换了一件外衣的熟悉场景。想想把 `x` 传给一个函数：

```go
f(x)
```

我们并不期望 `f` 能够修改 `x`，因为传进去的是 `x` 的值的副本，而不是 `x` 本身。如果我们希望 `f` 能直接修改 `x`，就必须把 `x` 的地址（即指向 `x` 的指针）传进去：

```go
f(&x)
```

这一点既直接又熟悉，而反射遵循同样的规则：如果想通过反射修改 `x`，就必须把指向该值的指针交给反射库。

让我们这么做。首先照常初始化 `x`，然后创建一个指向它的反射值，记为 `p`：

```go
var x float64 = 3.4
p := reflect.ValueOf(&x) // 注意：取 x 的地址
fmt.Println("type of p:", p.Type())
fmt.Println("settability of p:", p.CanSet())
```

目前的输出是：

```
type of p: *float64
settability of p: false
```

反射对象 `p` 本身不可设置，但我们想设置的不是 `p`，而是 `*p`（即 `p` 指向的值）。为了获取 `p` 指向的内容，我们调用 `Value` 的 `Elem` 方法，它会对指针进行解引用，并将结果保存在另一个反射值 `v` 中：

```go
v := p.Elem()
fmt.Println("settability of v:", v.CanSet())
```

此时 `v` 就是一个可设置的反射对象，输出也证实了这一点：

```
settability of v: true
```

由于 `v` 代表的是 `x`，我们终于可以用 `v.SetFloat` 来修改 `x` 的值：

```go
v.SetFloat(7.1)
fmt.Println(v.Interface())
fmt.Println(x)
```

输出正如预期：

```
7.1
7.1
```

反射可能难以理解，但它所做的与语言本身的机制完全一致，只不过是通过反射的 `Type` 和 `Value` 把过程包装了起来。只要记住：要想通过反射值修改它代表的内容，就必须传入“某样东西的地址”。

## Structs

在前面的例子里，`v` 本身并不是指针，它只是从一个指针推导而来的。这种情况在使用反射修改结构体字段时非常常见：只要我们拥有结构体的地址，就能修改它的字段。

下面是一个简单示例，分析一个结构体值 `t`。我们用结构体的地址创建反射对象，因为稍后需要修改它。接着把 `typeOfT` 设为它的类型，并通过直接的方法调用来遍历字段（细节参见 `reflect` 包文档）。注意，我们从结构体类型中提取字段名称，而字段本身则是普通的 `reflect.Value` 对象。

```go
type T struct {
    A int
    B string
}
t := T{23, "skidoo"}
s := reflect.ValueOf(&t).Elem()
typeOfT := s.Type()
for i := 0; i < s.NumField(); i++ {
    f := s.Field(i)
    fmt.Printf("%d: %s %s = %v\n", i,
        typeOfT.Field(i).Name, f.Type(), f.Interface())
}
```
该程序的输出为

```console
0: A int = 23
1: B string = skidoo
```
关于可设置性，这里顺带再提一点：结构体 `T` 的字段名必须以大写字母开头（即导出），因为只有导出字段才是可设置的。

由于 `s` 持有一个可设置的反射对象，我们可以修改结构体的字段：

```go
s.Field(0).SetInt(77)
s.Field(1).SetString("Sunset Strip")
fmt.Println("t is now", t)
```

运行结果如下：

```
t is now {77 Sunset Strip}
```

如果把程序改成让 `s` 从 `t` 而不是 `&t` 创建，那么对 `SetInt` 和 `SetString` 的调用都会失败，因为 `t` 的字段将不可设置。

## Conclusion

下面再次回顾一下反射的三条定律：

1. Reflection goes from interface value to reflection object.（反射将接口值转换为反射对象。)  
2. Reflection goes from reflection object to interface value.（反射将反射对象转换回接口值。）  
3. To modify a reflection object, the value must be settable.（若要修改反射对象，其值必须是可设置的。）

一旦理解了这三条定律，使用 Go 的反射就会变得容易许多，尽管它依旧微妙。反射是一把强大的工具，应当谨慎使用，除非确有需求，否则应避免滥用。

关于反射，还有许多内容本文尚未涉及——例如通道的发送与接收、内存分配、切片与映射的使用、方法及函数的调用等——但篇幅已够长了。我们将在后续文章中讨论其中的部分主题。
