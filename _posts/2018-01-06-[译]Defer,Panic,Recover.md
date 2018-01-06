---
layout: post
title:  "[译]Defer, Panic, and Recover"
date:   2018-01-06
categories: Go Golang
---

原文地址：[Defer, Panic, and Recover](https://blog.golang.org/defer-panic-and-recover)

Go有普通的流程控制机制：`if`, `for`, `switch`, `goto`等。它还有Go特有的语句来在单独的goroutine中执行程序。在这里我想讨论一些不太常见的特性：`defer`, `panic`和`recover`。

## defer

defer语句将一个函数调用压入到一个列表中。列表中存储的函数调用，将在执行`defer`语句的外围函数返回后被执行。defer通常用于执行各种清理操作的函数。

例如，让我来看下面这个函数，它打开两个文件，并将一个文件的内容拷贝到另一个函数。

```
func CopyFile(dstName, srcName string) (written int64, err error) {
    src, err := os.Open(srcName)
    if err != nil {
        return
    }

    dst, err := os.Create(dstName)
    if err != nil {
        return
    }

    written, err = io.Copy(dst, src)
    dst.Close()
    src.Close()
    return
}
```

这段代码能够工作，但是它有一个bug。bug就是在`os.Create(dstName)`失败时，函数将直接返回，而不会关闭已经打开的源文件。通过在`err != nil`成立时的`return`前调用`src.Close()`可以很容易的解决这个问题，但是如果函数更复杂有很多返回点的话，这就不会那么容易被发现和解决了。但是引入了`defer`语句后，我们就可以确保文件总是会被关闭了。

```
func CopyFile(dstName, srcName string) (written int64, err error) {
    src, err := os.Open(srcName)
    if err != nil {
        return
    }
    defer src.Close()

    dst, err := os.Create(dstName)
    if err != nil {
        return
    }
    defer dst.Close()

    return io.Copy(dst, src)
}
```

defer语句允许我们在打开文件后就立即考虑关闭文件的事情，保证不管函数有多少返回点文件都将被正确关闭。

defer语句的行为是直接和可预测的。有三条简单的规则：

1. defer函数的参数在评估函数的时候被评估（通俗的讲就是，被defer函数的实参值就是执行defer语句时该变量的值）。

	在下面的实例中，在执行defer语句将函数`fmt.Println(i)`压入defer list时，变量i的值为0，所以被defer的函数实际执行时，它的实参为0。函数最终打印的是0

	```
	func a() {
		i := 0
		defer fmt.Println(i)
		i++
		return
	}
	```

2. 在执行defer的外围函数返回后，被defer的函数会按后入先出（LIFO)的顺序执行。下面的函数将打印"3210"。

	```
	func b() {
		for i := 0; i < 4; i++ {
			defer fmt.Print(i)
		}
	}
	```

3. 被defer的函数能够读取和**修改**外围函数**命名的**的返回值。下面的示例中，被defer的函数对外围函数的返回值`i`执行了一次自增操作。所以函数的返回值是2.

	```
	func c() (i int) {
		defer func() { i++ }()
		return 1
	}
	```

	这个特性对修改函数返回的错误值很有用，稍后我们就能看到一个示例。

## panic

panic是一个內建函数，用于停止执行常规控制流程，并开始安全退出。当一个函数`F`调用了panic后，F将停止执行，任何F中的被defer开始被正常执行，然后F将控制权返回给它的调用者。对于F的调用者来说，这时F的行为就像是一次panic调用一样。这个过程将沿着栈继续往上，直到当前goroutine中的所有函数都返回，此时程序将崩溃。Panics可以通过直接调用`panic`函数发起。它们也可能由运行时错误引起，比如数组访问越界等。

## recover

recover也是一个內建函数，它用于重新获取一个*panicking goroutine*的控制权。recover只有在被defer的函数中才有用。在正常执行期间，调用recover将会返回nil，并且不会有任何影响。如果当前goroutine正在清场，则一次recover调用将能够捕获到panic的值，并恢复正常执行。

下面这个示例说明了panic和defer的机制。

```
package main

import "fmt"

func main() {
    f()
    fmt.Println("Returned normally from f.")
}

func f() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered in f", r)
        }
    }()
    fmt.Println("Calling g.")
    g(0)
    fmt.Println("Returned normally from g.")
}

func g(i int) {
    if i > 3 {
        fmt.Println("Panicking!")
        panic(fmt.Sprintf("%v", i))
    }
    defer fmt.Println("Defer in g", i)
    fmt.Println("Printing in g", i)
    g(i + 1)
}
```

函数`g`接收一个整型的传参`i`，并且如果实参`i`大于3就会panic，如果不大于3则使用参数`i+1`递归调用自己。函数`f` defer了一个函数，该函数调用recover来捕获panic值，如果捕获到的值不是nil则打印捕捉到的数据。尝试描绘一下这段代码的输出是什么样的。

下面就是这段代码的输出，跟你想的一样吗？

```
Calling g.
Printing in g 0
Printing in g 1
Printing in g 2
Printing in g 3
Panicking!
Defer in g 3
Defer in g 2
Defer in g 1
Defer in g 0
Recovered in f 4
Returned normally from f.
```

如果我们拿掉函数`f`中的defer函数，则panic将不会被恢复，清场将执行到goroutine调用栈的顶部，然后终止程序。修改后的程序输出是下面这样的：

```
Calling g.
Printing in g 0
Printing in g 1
Printing in g 2
Printing in g 3
Panicking!
Defer in g 3
Defer in g 2
Defer in g 1
Defer in g 0
panic: 4

panic PC=0x2a9cd8
[stack trace omitted]
```

对于一个真实的**panic**和**recover**的示例，可以参考Go标准库的[json package](http://golang.org/pkg/encoding/json/)。它使用一组递归函数解码JSON编码的数据。当遇到格式错误的JSON时，解析器将调用panic来展开到顶层函数的调用栈，顶层函数将使用recover从panic中恢复控制权并返回一个合适的错误（参考[decode.go](http://golang.org/src/pkg/encoding/json/decode.go)中`decodeState`的`error`和`unmarshal`方法。

Go语言库的一个惯例就是，即使一个包在内部使用了panic，它对外暴露的API仍然会呈现明确的错误返回值。

除了开篇示例的关闭文件外，**defer**还常被用于：释放互斥锁。

```
mu.Lock()
defer mu.Unlock()
```

打印页脚，等等很多。

```
printHeader()
defer printFooter()
```

## summary

**defer**语句（不管是否与**panic**和**recover**一起）为控制流提供了一个不寻常的且强大的机制。它可以用来模拟很多其他语言专门设计的数据结构实现的功能。试一下。
