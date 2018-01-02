---
layout: post
title:  "[译]更多Go Interface细节"
date:   2018-01-01
categories: Go Golang Interface
---

原文: [More on interface](http://go-book.appspot.com/more-interfaces.html)

我们在前一章发现了Interface的魅力，如此简单的定义一种类型行为的概念提供了无限的遐想空间。

如果你还没有读过前一章，你也不应该读这一篇文章，因为这一篇文章的大部分内容都是基于学习了前一篇文章对Interface有一个良好的认识的（译者注：作者这里假设的是读者在不了解interface的情况下直接读这篇文章，所以建议先读前一篇。如果读者已经对Interface有了一个基本的认识，其实还是可以略过前一篇，直接上这一篇的。毕竟前一篇相当的基础。）

## 了解Interface变量中存储的是什么

我们知道一个interface变量可以存储任何实现了这个interface的类型。那是一个很好的开始，但是如果接下来我们想要将一个interface变量中存储的数据恢复并存储到它自己类型的变量中。我们怎么知道到底是什么类型的变量被包裹在Interface中呢？

让我们通过示例来阐明实际的问题：

``` Go
type Element interface{}
type List [] Element

//...

func main() {
    //...
    var number int
    element := list[index]
    // The question is how do I convert 'element' to int, in order to assign
    // it to number and is the value boxed in 'element' actually an int?
    //...
}
```

所以摆在我们前面的问题就是：我们怎么知道存储在interface中的数据是什么类型的？

## Comma-ok式的断言
Go提供了一个方便的语法判断一个interface数据是否能够被转换到一个指定类型，该语法格式为：`value, ok = element.(T)`，其中Value是类型T的一个变量，ok是一个bool类型变量，element是一个interface变量。如果可以将element转换为T，则ok为true，Value是转换结果。否则ok为false，Value将被设置为类型T的零值。

来看下示例：

``` Go 
package main

import (
    "fmt"
    "strconv" //for conversions to and from string
)

type Element interface{}
type List [] Element

type Person struct {
    name string
    age int
}

//For printng. See previous chapter.
func (p Person) String() string {
    return "(name: " + p.name + " - age: "+strconv.Itoa(p.age)+ " years)"
}

func main() {
    list := make(List, 3)
    list[0] = 1 // an int
    list[1] = "Hello" // a string
    list[2] = Person{"Dennis", 70}

    for index, element := range list {
        if value, ok := element.(int); ok {
            fmt.Printf("list[%d] is an int and its value is %d\n", index, value)
        } else if value, ok := element.(string); ok {
            fmt.Printf("list[%d] is a string and its value is %s\n", index, value)
        } else if value, ok := element.(Person); ok {
            fmt.Printf("list[%d] is a Person and its value is %s\n", index, value)
        } else {
            fmt.Printf("list[%d] is of a different type\n", index)
        }
    }
}
```

输出：

	list[0] is an int and its value is 1
	list[1] is a string and its value is Hello
	list[2] is a Person and its value is (name: Dennis - age: 70 years)
	
如此简单！

你是否注意到我们使用了if语法呢？我希望你还记得我们可以在if语句中初始化条件判断的变量。你是否真的记得呢？

是的，我也知道，如果有更多的类型需要测试，则我需要更多if-else语句，使得代码变得越来越难于阅读。这正是为啥我们接下来使用switch来判断类型的原因。

## 使用switch判断类型

示例是最好的呈现，对哇？让我们来重写前面的示例：

``` Go
package main

import (
    "fmt"
    "strconv" //for conversions to and from string
)

type Element interface{}
type List [] Element

type Person struct {
    name string
    age int
}

//For printng. See previous chapter.
func (p Person) String() string {
    return "(name: " + p.name + " - age: "+strconv.Itoa(p.age)+ " years)"
}

func main() {
    list := make(List, 3)
    list[0] = 1 //an int
    list[1] = "Hello" //a string
    list[2] = Person{"Dennis", 70}

    for index, element := range list{
        switch value := element.(type) {
            case int:
                fmt.Printf("list[%d] is an int and its value is %d\n", index, value)
            case string:
                fmt.Printf("list[%d] is a string and its value is %s\n", index, value)
            case Person:
                fmt.Printf("list[%d] is a Person and its value is %s\n", index, value)
            default:
                fmt.Println("list[%d] is of a different type", index)
        }
    }
}
```

输出：

	list[0] is an int and its value is 1
	list[1] is a string and its value is Hello
	list[2] is a Person and its value is (name: Dennis - age: 70 years)
	
来，左手握拳，举到你的太阳穴，跟我宣誓：

*“The element.(type) construct SHOULD NOT be used outside of a switch statement! – Can you use it elsewhere? – NO, YOU CAN NOT!“（`element.(type)`结构绝不能被用于switch条件外的任何地方！你能在其他地方使用它吗？不，你不能！*

如果你只想做一个测试，使用comma-ok就可以了，不要在switch外使用`element.(type)`

## 嵌入式interfaces

Go的语法逻辑是Go真正美妙的地方。当我们学习struct匿名域的时候，我们发现它是如此自然，不是吗？现在应用相同的逻辑，如果能将一个interface1嵌入到另一个interface2中，则interface2能够继承interface1的方法，是不是一件很美妙的事情呢？

我会说：“符合逻辑”。因为毕竟：interface是方法的集合，就像struct是各种类型域的集合一样。然后，确实是这样，在Go里面，你可以嵌入一个interface到另一个interface里面。

例如：假设你有一个被索引的元素集合，你想在不改变集合元素顺序的情况下获取集合的最大、最小值。

一个比较笨，但是能说明问题的方式是使用前面一章中讲到的`sort.Interface`。但是，函数`sort.Sort`事实上会改变输入的集合。

所以我们增加两个方法：`Get`和`Copy`:

``` Go
package main

import (
    "fmt"
    "strconv"
    "sort"
)

type Person struct {
    name string
    age int
    phone string
}

type MinMax interface {
    sort.Interface
    Copy() MinMax
    Get(i int) interface{}
}

func (h Person) String() string {
    return "(name: " + h.name + " - age: "+strconv.Itoa(h.age)+ " years)"
}

type People []Person // People is a type of slices that contain Persons

func (g People) Len() int {
    return len(g)
}

func (g People) Less(i, j int) bool {
    if g[i].age < g[j].age {
        return true
    }
    return false
}

func (g People) Swap(i, j int) {
    g[i], g[j] = g[j], g[i]
}

func (g People) Get(i int) interface{} {return g[i]}

func (g People) Copy() MinMax {
    c := make(People, len(g))
    copy(c, g)
    return c
}

func GetMinMax(C MinMax) (min, max interface{}) {
    K := C.Copy()
    sort.Sort(K)
    min, max =  K.Get(0), K.Get(K.Len()-1)
    return
}

func main() {
    group := People {
        Person{name:"Bart", age:24},
        Person{name:"Bob", age:23},
        Person{name:"Gertrude", age:104},
        Person{name:"Paul", age:44},
        Person{name:"Sam", age:34},
        Person{name:"Jack", age:54},
        Person{name:"Martha", age:74},
        Person{name:"Leo", age:4},
    }

    //Let's print this group as it is
    fmt.Println("The unsorted group is:")
    for _, value := range group {
        fmt.Println(value)
    }

    //Now let's get the older and the younger
    younger, older := GetMinMax(group)
    fmt.Println("\n➞ Younger is", younger)
    fmt.Println("➞ Older is ", older)

    //Let's print this group again
    fmt.Println("\nThe original group is still:")
    for _, value := range group {
        fmt.Println(value)
    }
}
```

输出：

	The unsorted group is:
	(name: Bart - age: 24 years)
	(name: Bob - age: 23 years)
	(name: Gertrude - age: 104 years)
	(name: Paul - age: 44 years)
	(name: Sam - age: 34 years)
	(name: Jack - age: 54 years)
	(name: Martha - age: 74 years)
	(name: Leo - age: 4 years)

	➞ Younger is (name: Leo - age: 4 years)
	➞ Older is (name: Gertrude - age: 104 years)

	The original group is still:
	(name: Bart - age: 24 years)
	(name: Bob - age: 23 years)
	(name: Gertrude - age: 104 years)
	(name: Paul - age: 44 years)
	(name: Sam - age: 34 years)
	(name: Jack - age: 54 years)
	(name: Martha - age: 74 years)
	(name: Leo - age: 4 years)

这个示例也有点白痴的，但是它确实能满足需求。你要知道，嵌入Interface非常的有用，在很多go package中你都可以找到它的身影。比如提供堆操作的`container/heap`包就实现了这类接口：

``` Go
//heap.Interface
type Interface interface {
    sort.Interface //embeds sort.Interface
    Push(x interface{}) //a Push method to push elements into the heap
    Pop() interface{} //a Pop elements that pops elements from the heap
}
```

另一个例子是`io.ReaderWriter` interface，它是两个interface的组合，那两个interface分别是`io.Reader`和`io.Writer`。 它们都是在`io`package中定义的：

``` Go
// io.ReadWriter
type ReadWriter interface {
    Reader
    Writer
}
```
所以实现了`io.ReaderWriter`的类型既能够读又能够写。

**需要我再提醒你一遍`element.(type)`不要在switch以外的地方使用么？**
