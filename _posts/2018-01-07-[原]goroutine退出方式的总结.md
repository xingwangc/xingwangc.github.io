---
layout: post
title:  "[原]goroutine退出方式的总结"
date:   2018-01-07
categories: Go Golang goroutine
---

## goroutine的退出机制

大家都知道goroutine是Go语言并发的利器，通过goroutine我们可以很容易的编写高并发的程序。但是goroutine设计的退出机制是由goroutine自己退出，不能在外部强制结束一个正在执行的goroutine(只有一种情况正在运行的goroutine会因为其他goroutine的结束被终止，就是main函数退出或程序停止执行)。关于goroutine为什么要设计成这样的退出机制，改天再po两篇译文上来（别人已经写得很清楚了，我想我应该不需要做额外的总结了）。

但是最近遇到一个坑，就是我有很多可并发的一次性事务。对每一个事务我都起一个goroutine来执行。正常情况下事务执行完毕, goroutine就退出。没有循环，没有复杂的逻辑控制，顺序执行就完事儿了。但是坑就坑在顺序执行上了，如果顺序执行过程中因为某个原因block了，比如读IO，获取连接，或者就是一个很耗时的计算，我想为这个goroutine设置一个超时退出，或者异常退出，却因为goroutine的这种机制我没法为这种类型的goroutine根据超时、异常执行强制退出的操作。

所以乘机整理了一下几种能够让一个goroutine退出的机制。

## main 退出

这个没有什么好具体说的，main是Go程序的主入口，main函数退出基本意味着你代码执行的结束。进程都退出了，所有它占有的资源都会还给操作系统，所以还结束的goroutines也没什么好玩儿的了。

## 通过channel通知退出

这个最主要的goroutine退出方式。goroutine虽然不能强制结束另外一个goroutine，但是它是它可以通过channel通知另外一个goroutine你的表演该结束了。常用的方法到处都可以看到，这里也不详细说明了，直接上一个示例：

下面的示例中起了一个goroutine执行`cancelByChannel`，但是在起它之前还通过`time.After`返回了一个`time.Time`类型的channel，该channel上在定时超时时会发送一个当前时间数据。`cancelByChannel每隔1s会检查这个channel上是否有数据接收，如果有数据则退出goroutine，如果没有信号接收就在连接上发送一条数据。所以下面这段代码在运行10s发送10条消息后将退出。

程序起起来后，另开一个终端执行`nc localhost:8000`（Linux上）或`nc localhost 8000`（mac 上）可以看到程序执行情况。

```
package main

import (
        "context"
        "fmt"
        "io"
        "net"
        "sync"
        "time"
)

func cancelByChannel(c net.Conn, quit <-chan time.Time, wg *sync.WaitGroup) {
        defer c.Close()
        defer wg.Done()

        for {

                select {
                case <-quit:
                        fmt.Println("cancel goroutine by channel!")
                        return
                default:
                        _, err := io.WriteString(c, "hello cancelByChannel")
                        if err != nil {
                                return
                        }
                        time.Sleep(1 * time.Second)
                }
        }
}

func main() {
        listener, err := net.Listen("tcp", "localhost:8000")
        if err != nil {
                fmt.Println(err)
                return
        }

        conn, err := listener.Accept()
        if err != nil {
                fmt.Println(err)
                return
        }

        wg := sync.WaitGroup{}

        wg.Add(1)
        quit := time.After(time.Second * 10)
        go cancelByChannel(conn, quit, &wg)
	wg.Wait()
}
```

### 通过context通知goroutine退出

通过channel通知goroutine退出还有一个更好的方法就是使用context。关于Context的详细信息可以参考前面的文章[Go并发模式: Context](https://xingwangc.github.io/go/%E5%B9%B6%E5%8F%91/context/2018/01/02/%E8%AF%91-Go%E5%B9%B6%E5%8F%91%E6%A8%A1%E5%BC%8F-context.html)。它本质还是接收一个channel数据，只是是通过ctx.Done()获取。将上面的示例稍作修改就可以用起来了。

```
func cancelByContext(ctx context.Context, c net.Conn, wg *sync.WaitGroup) {
        defer c.Close()
        defer wg.Done()
        for {
                select {
                case <-ctx.Done():
                        fmt.Println("cancel goroutine by context:", ctx.Err())
                        return
                default:
                        _, err := io.WriteString(c, "hello cancelByContext")
                        if err != nil {
                                return
                        }
                        time.Sleep(1 * time.Second)
                }
        }
}
```

main函数中将这两行代码：

```
	quit := time.After(time.Second * 10)
        go cancelByChannel(conn, quit, &wg)
```

使用下面几行替换：

```
	ctx, cancel := context.WithTimeout(context.Background(), time.Second*10)
        defer cancel()
        go cancelByContext(ctx, conn, &wg)
```

## panic 退出

这种方法有点走邪门歪道了，这个需要依赖一些标准库和第三方库的设计机制，没有处理好可能会出现一些你完全意料不到的结果，慎用。这种方法涉及到的知识点包括,`defer, panic, recover`，详细可参见前面的译文[defer, panic, recover](https://xingwangc.github.io/go/golang/2018/01/06/%E8%AF%91-Defer,Panic,Recover.html)。

这种方式能解决部分我在本文开头提到的问题。比如如果因为某次IO block了（比如一次数据库inert事务）等。这一类事务比如数据库操作，文件操作的步骤通常是：建立连接（或打开文件），然后执行读写操作，读写完成后关闭连接（或文件）。假如因为某种原因我们在读写操作那一步block了。我们又不希望这个goroutine一直block在那儿，或者我们需要这个goroutine在指定时间内完成。这时我们期望goroutine自己结束并退出可能就有点不现实了。

**这个时候加入我们关闭文件或链接会怎么样？**我也不知道，这样看你网络操作，文件操作所使用的标准库或者第三方库的实现了。但是，通常情况下链接或文件关闭后，你的读写操作要么会立即抛出一个panic，要么就是立即返回一个错误了。**注意，这里说的是通常情况，不是所有情况，还有具体是抛出panic还是error，这些都需要根据你自己的实际情况具体分析。**

所以这里主要针对的是panic和error的退出方式，看下面的模拟示例。(作者遇到的坑是block在一次mongo的写操作上，由于问题不太好表现，这里没有使用该示例，而是基于前面的示例做了一些修改。）这里由于断开net.Conn的连接，通过io.Writer往连接上只是返回了一个error，并不能成功模拟recover panic的方式。所以示例只能作为这种实现方式的参考，另外也说明了这种方式的不确定性。** 所以再强调一遍，一定要慎用。**

```
func cancelByPanic(c net.Conn, wg *sync.WaitGroup) {
        defer func() {
                if err := recover(); err != nil {
                        fmt.Println("cancel goroutine by context:", err)
                }
        }()

        defer wg.Done()

        for {
                _, err := io.WriteString(c, "hello cancelByPanic")
                if err != nil {
			fmt.Println(err)
                        return
                }
                time.Sleep(1 * time.Second)
        }
}
```

上面函数中defer函数中使用了recover来捕获panic error并从panic中拿回控制权，确保程序不会再panic展开到goroutine调用栈顶部后崩溃。

main函数也要做相应的更改，还需要起一个额外的goroutine来根据相应的退出机制关闭连接。示例中设置的是超时。超时后连接关闭，`io.WriteString()`将返回一个错误，然后退出goroutine.

```
	go func(ctx context.Context, conn net.Conn, wg *sync.WaitGroup) {
		defer wg.Done()
                for {
                        select {
                        case <-ctx.Done():
                                fmt.Println("---close the connection outside!")
                                conn.Close()
                                return
                        default:
                        }
                }
        }(ctx, conn, &wg)

        wg.Add(1)
        go cancelByPanic(conn, &wg)
```

## 等它自己退出:)

最后，还有一种情况也可能是大家经常遇到的，就是本文开头提到的你的goroutine可能只是执行一个计算，但是这个计算执行的时间有点长。对于这种方式，貌似如果你不打算改你的设计换一种方式执行程序的话，就只有等它自己结束了。

下面也是一个示例，这个示例只是根据一个初始值计算进行累减数求和。本例中使用简单的递归求和的方式，随着初始值的变大，计算过程会越来越慢。

```
func slowCal(fac int) int {
        if fac < 2 {
                return fac
        }

        return slowCal(fac-1) + slowCal(fac-2)
}

func cancelByWait(wg *sync.WaitGroup) {
        defer wg.Done()

        start := time.Now()

        result := slowCal(50)

        dur := time.Since(start)

        fmt.Println("slow goroutine done:", result, dur)
}
```

main 函数中直接执行`go cancelByWait`即可。

### 这种方式的改进

这个示例还有很大的改进空间，这里也不深入展开了。只简单的提两点，读者可以自己下去尝试下。当然这个例子也很简单，也不用花时间去写代码，想想应该就可以了:)。

1. 可以通过优化算法，以及修改并发方式提高计算速度。

2. 这个示例也是可以引入context或channel来通知计算超时退出的，如果你不想要计算结果的话。

## 总结

由于Goroutine被设计为只能自己退出，而不能强制退出。在实际使用中，我们可能会因为某些原因被block在Goroutines里面，或由于设计缺陷导致一些Goroutines执行很长的时间。只是基于一些其他语言的经验，我们可能会期望有一种外部机制能够强制结束一个Goroutines。但是这就是Go和Goroutine，它的目的就是要提供一种轻量的，简单的并发方式。保证它这个特性的基础也决定了我们不能用外部方式强制关闭一个Goroutines（额外post译文或博文说明这个问题，此文不深入展开)。所以当你遇到这种情况的时候，你可能需要考虑你的设计是不是足够的Go style，或者你对一些外部依赖是否足够了解了。

