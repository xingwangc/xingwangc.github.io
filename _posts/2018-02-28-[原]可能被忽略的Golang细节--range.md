---
layout: post
title:  "[原]可能被忽略的Golang细节——range"
date:   2018-02-28
categories: Go Golang
---

*range*关键字是Go语言中一个非常有用的迭代array，slice，map, string, channel中元素的内置关键字。

## range的使用

*range*的使用非常简单，对于遍历array，*array，string它返回两个值分别是数据的索引和值，遍历map时返回的两个值分别是key和value，遍历channel时，则只有一个返回数据。各种类型的返回值参考下表：

| range expression | 1st Value | 2nd Value(optional)| notes |
|------------------|-----------|--------------------|-------|
| array`[n]E,*[n]E`| index`int`| value `E[i]`       ||
| slice `[]E`      | index`int`| value `E[i]`       ||
| string `abcd`    | index`int`| rune `int`         |*对于string，range迭代的是Unicode而不是字节，所以返回的值是rune*|
| map`map[k]v`     | key`k`    | value `v`          ||
| channel          | element   | none               ||

使用方式非常简单，下面直接贴一段*gobyexample*中的示例代码作为参考，执行结果请点击链接跳转到[gobyexample](https://gobyexample.com/range)。

```
package main
import "fmt"
func main() {

    nums := []int{2, 3, 4}
    sum := 0
    for _, num := range nums {
        sum += num
    }
    fmt.Println("sum:", sum)

    for i, num := range nums {
        if num == 3 {
            fmt.Println("index:", i)
        }
    }

    kvs := map[string]string{"a": "apple", "b": "banana"}
    for k, v := range kvs {
        fmt.Printf("%s -> %s\n", k, v)
    }

    for k := range kvs {
        fmt.Println("key:", k)
    }

    for i, c := range "go" {
        fmt.Println(i, c)
    }
}
```

range的详细说明也请跳转官方文档的[for_statement](https://golang.org/ref/spec#For_statements)章节自行阅读。本文主要介绍一些平时可（yi）能（bu）被（xiao）忽（xin）略（jiu）的（diao）细（keng）节（li）。

## range表达式构建

先来看看下面这段代码的输出是什么？这段代码会无限循环的执行下去吗？

```
func modifySlice() {
    v := []int{1, 2, 3, 4}
    for i := range v {
        v = append(v, i)
        fmt.Printf("Modify Slice: value:%v\n", v)
    }
}
```

答案肯定不会无限循环的，这么低级的错误，Go的开发者肯定是不会范的。那结果会是什么呢？执行这段代码会打印下面的内容：

```
Modify Slice: value:[1 2 3 4 0]
Modify Slice: value:[1 2 3 4 0 1]
Modify Slice: value:[1 2 3 4 0 1 2]
Modify Slice: value:[1 2 3 4 0 1 2 3]
```

可以看到每次循环在map中插入新的内容后，map的长度确实发生了变化，但是循环只执行了三次，正好是执行range前map的长度。说明range在执行之初就构建好了range表达式的内容了，虽然后面map的内容增加了，但是并不会影响初始构建的结果。官方文档对于range表达式的构建描述是这样的：

> The range expression x is evaluated once before beginning the loop, with one exception: if at most one iteration variable is present and len(x) is constant, the range expression is not evaluated.
> 
> 就是说range表达式在开始执行循环之前就已经构建了，但是有一个例外就是：如果最多只有一个迭代变量，且len(x)表达的是值是一个**常量**的时候range表达式不会构建。
> 
> 那什么时候len(x)是一个常量呢？按照通常的理解，len(string), len(slice), len(array)...的返回应该都是常量啊？事实上不是这样的，这其实是由数据结构的特性决定的。因为相比较于其他语言，对于这一类容器数据结构，在Go语言中不仅有长度属性可以通过内建函数len()获取，还有一个可以通过内建函数cap()获取的容量属性，尤其是slice，map这一类可变长的数据结构。所以对于*常量*的定义，Go官方文档[Length and capacity](https://golang.org/ref/spec#Length_and_capacity)有详细的说明。

如果到这里，你以为你已经理解了*range*的构建，并且一眼就能看出一个for-range循环的迭代方式和执行情况。前面可能就已经有一个大坑为你准备好了。这时候官方文档里面下面这样一段话可能就被你忽略了，让我先贴出来：

> The iteration order over maps is not specified and is not guaranteed to be the same from one iteration to the next. If a map entry that has not yet been reached is removed during iteration, the corresponding iteration value will not be produced. If a map entry is created during iteration, that entry may be produced during the iteration or may be skipped. The choice may vary for each entry created and from one iteration to the next. If the map is nil, the number of iterations is 0.
> 
> 用中国话解释下，首先对于map的迭代来说，map中内容的迭代顺序没有指定，也不保证，简单的说就是迭代map的时候将以随机的顺序返回里面的内容。这个好理解，也就是说如果你想按顺序迭代一些东西，map就不要指望了，换其他数据结构吧。
> 
> **然后就进入高潮部分了，如果一个map的key在还没有被迭代到的时候就被delete掉了，那么它所对应的迭代值就不会被产生了。然后对于在迭代过程中新增加的key，则它可能会被迭代到也可能不会。如何选择会根据key增加的时机以及从上一次到下一次的迭代而不同。**你可能会想，What？你TM在逗我么，说好的提前构建的呢...但是很不幸，事实就是这样，将前面的示例代码改成使用map我执行了几次结果都不一样。所以这种坑还是绕过为好。至于为什么会这样，容我有空再研究研究，下次重开一篇文章介绍。

```
func modifyMap() {
    data := map[string]string{"a": "A", "b": "B", "c": "C"}
    for k, v := range data {
        data[v] = k
        fmt.Println("modify Mapping", data)
    }
}
```

结果1，迭代了6次

```
modify Mapping map[a:A b:B c:C A:a]
modify Mapping map[b:B c:C A:a B:b a:A]
modify Mapping map[c:C A:a B:b C:c a:A b:B]
modify Mapping map[a:A b:B c:C A:a B:b C:c]
modify Mapping map[A:a B:b C:c a:A b:B c:C]
modify Mapping map[a:A b:B c:C A:a B:b C:c]
```

结果2，迭代了4次：

```
modify Mapping map[a:A b:B c:C A:a]
modify Mapping map[b:B c:C A:a B:b a:A]
modify Mapping map[a:A b:B c:C A:a B:b C:c]
modify Mapping map[B:b C:c a:A b:B c:C A:a]
```

## range string

使用range迭代字符串时，需要主要的是range迭代的是Unicode而不是字节。返回的两个值，第一个是被迭代的字符的UTF-8编码的第一个字节在字符串中的索引，第二个值的为对应的字符且类型为[rune](https://golang.org/ref/spec#Rune_literals)(实际就是表示unicode值的整形数据）。

总结下来就是使用range迭代string时，需要注意下面两点：

* 迭代的是Unicode不是字节，第一个返回值是UTF-8编码第一个字节的索引，所以索引值可能并不是连续的。
* 第二个返回值的类型为rune，不是string类型的，如果要使用该值需要格式化。

看下面代码：

```
//string
func rangeString() {
    datas := "aAbB"

    for k, d := range datas {
        fmt.Printf("k_addr:%p, k_value:%v\nd_addr:%p, d_value:%v\n----\n", &k, k, &d, d)
    }
}
```

这段代码的输出如下，这里使用的string是只占用一个字节的字符的字符串，所以返回的第一个索引是连续的，可以看到第二个值都是整型数字。

```
k_addr:0xc420014148, k_value:0
d_addr:0xc420014150, d_value:97
k_addr:0xc420014148, k_value:1
d_addr:0xc420014150, d_value:65
k_addr:0xc420014148, k_value:2
d_addr:0xc420014150, d_value:98
k_addr:0xc420014148, k_value:3
d_addr:0xc420014150, d_value:66
```

### range可以对string做更多的事情

前面说到range是对Unicode进行迭代来迭代字符串的，所以range还能够在迭代过程中发现字符串中非Unicode的字符，并使用`U+FFFD`字符替换改无效字符。

看下面代码的执行:

```
func rangeStringMore() {
    for pos, char := range "中\x80文" { // \x80 is an illegal UTF-8 encoding
        fmt.Printf("character %#U starts at byte position %d\n", char, pos)
    }
}
```

上面这段代码使用range迭代字符串`"中\x80文"`，字符串中`\x80`是一个无效的的Unicode字符，所以range在迭代时会使用`U+FFFD`将其替换。另外UTF-8使用变长方式编码，第一个汉字`中`占用了3个字节，所以遍历第二个字符的时候，其索引已经是3了，但是其只占一个字节，所以第三个字符`文`的索引4.

```
character U+4E2D '中' starts at byte position 0
character U+FFFD '�' starts at byte position 3
character U+6587 '文' starts at byte position 4
```

## range表达式是指针

前面说过range可以迭代的数据类型包括array，slice，map，string和channel。在这些数据类型里面只有array类型能以指针的形式出现在range表达式中。

具体参考下面的代码：

```
func rangePointer() {
    //compile error: cannot range over datas (type *string)
    //d := "aAbBcCdD"
    d := [5]int{1, 2, 3, 4, 5} //range successfully
    //d := []int{1, 2, 3, 4, 5} //compile error: cannot range over datas (type *[]int)
    //d := make(map[string]int) //compile error: cannot range over datas (type *map[string]int)
    datas := &d

    for k, d := range datas {
        fmt.Printf("k_addr:%p, k_value:%v\nd_addr:%p, d_value:%v\n----\n", &k, k, &d, d)
    }
}
```

## 参考链接：

[range](https://github.com/golang/go/wiki/Range)  
[For statements](https://golang.org/ref/spec#For_statements)  
[Length and capacity](https://golang.org/ref/spec#Length_and_capacity)  
[Rune literals](https://golang.org/ref/spec#Rune_literals)  
[Go by Example: Range](https://gobyexample.com/range)
