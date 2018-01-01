---
layout: post
title:  "棒棒哒的Go Interface"
date:   2018-01-01
categories: Go Golang Interface
---

原文：[Interface: the awesomesause of Go](http://go-book.appspot.com/interfaces.html)

让我们稍微重温下。我们知道基本的数据类型，使用它们，并且学会创建复合数据类型。我们还知道一些基本的流程控制结构。所以我们能基于这些编程实现功能。接下来我们发现函数实际上也是一种数据，它们是它们自己的值并且也有类型。我们继续学习方法。我们使用方法写函数来操作数据，然后继续实现功能性的数据类型。这一章我们将进入Go的下一个境界。我们将学习对象的一些天生的洪荒之力来做一些实际事情。

废话少说，直奔主题！

## Interface是什么？
简而言之，Interface就是一组方法的集合。我们使用Interface指定一个给定对象的行为。例如前面章节提到的，`Student`和`Employee`都有`SayHi`，但是是不同的实现。这都不是事儿，它们都知道怎么说"Hi".

加点推理：`Student`和`Employee`实现了另外的方法`Sing`，同时`Employee`实现了方法`SpendSalary`, `Student`实现了方法`BorrowMoney`。总结下就是：`Student`实现了方法：`SayHi`, `Sing`, `BorrowMoney`；`Employee`实现了方法`SayHi`, `Sing`和`SpendSalary`。

这些方法的集合就是`Student`和`Employee`满足的Interface.例如：`Student`和`Employee`都符合包含方法`SayHi`和`Sing`的Interface。但是`Employee`不符合由方法`Sing`,`SayHi`和`BorrowMoney`组成的Interface，因为`Employee`没有实现方法`BorrowMoney`。

## Interface类型
Interface类型指定了一组由其他对象实现的方法。使用如下语法：

```Go
type Human struct {
    name string
    age int
    phone string
}

type Student struct {
    Human //an anonymous field of type Human
    school string
    loan float32
}

type Employee struct {
    Human //an anonymous field of type Human
    company string
    money float32
}

// A human likes to stay... err... *say* hi
func (h *Human) SayHi() {
    fmt.Printf("Hi, I am %s you can call me on %s\n", h.name, h.phone)
}

// A human can sing a song, preferrably to a familiar tune!
func (h *Human) Sing(lyrics string) {
    fmt.Println("La la, la la la, la la la la la...", lyrics)
}

// A Human man likes to guzzle his beer!
func (h *Human) Guzzle(beerStein string) {
    fmt.Println("Guzzle Guzzle Guzzle...", beerStein)
}

// Employee's method for saying hi overrides a normal Human's one
func (e *Employee) SayHi() {
    fmt.Printf("Hi, I am %s, I work at %s. Call me on %s\n", e.name,
        e.company, e.phone) //Yes you can split into 2 lines here.
}

// A Student borrows some money
func (s *Student) BorrowMoney(amount float32) {
    loan += amount // (again and again and...)
}

// An Employee spends some of his salary
func (e *Employee) SpendSalary(amount float32) {
    e.money -= amount // More vodka please!!! Get me through the day!
}

// INTERFACES
type Men interface {
    SayHi()
    Sing(lyrics string)
    Guzzle(beerStein string)
}

type YoungChap interface {
    SayHi()
    Sing(song string)
    BorrowMoney(amount float32)
}

type ElderlyGent interface {
    SayHi()
    Sing(song string)
    SpendSalary(amount float32)
}
```

正如你所看到的，一个Interface可以有任意的对象实现。上面示例中，`Student`和`Employee`都实现了`Men` Interface。

反之，一个对象也可以实现任意的Interface。上面示例中，`Student`就实现了`Men`和`YoungChap` Interface。同时`Employee`实现了`Men`和`ElderlyGent`。

最后，所有类型都实现了empty Interface，你应该可以猜到，就是没有方法的Interface，使用`interface{}`表示。

这个时候你可能会说*“牛逼，但是定义Interface的关键是什么呢？难道仅仅是描述类型的行为，他们的共同点是什么呢？”*

稍等！这里还有点东西——当然这里肯定还有更多的货。

另外，我刚才有提到empty interface还没有方法吗？

## Interface的值
由于Interface是它自身的类型，你可能还想知道一个interface类型的值到底是啥。

Tata！这里有个惊天好消息：如果你声明了interface 变量，它可以存储任何实现了这个interface的数据类型。

就是说，如果你定义了一个`Men`类型的interface `m`，它可以存储`Student`，`Employee`或... ...。这是因为它们都实现了`Men` interface指定的方法。

同我一起总结下：如果`m`可以存储任何不同类型的值，我们可以很轻松的声明一个`Men`类型的Slice存储各种不同的值。这就不再是传统的经典Slice类型了。

来一个示例梳理一下刚刚我介绍的东西：

```Go
package main
import "fmt"

type Human struct {
    name string
    age int
    phone string
}

type Student struct {
    Human //an anonymous field of type Human
    school string
    loan float32
}

type Employee struct {
    Human //an anonymous field of type Human
    company string
    money float32
}

//A human method to say hi
func (h Human) SayHi() {
    fmt.Printf("Hi, I am %s you can call me on %s\n", h.name, h.phone)
}

//A human can sing a song
func (h Human) Sing(lyrics string) {
    fmt.Println("La la la la...", lyrics)
}

//Employee's method overrides Human's one
func (e Employee) SayHi() {
    fmt.Printf("Hi, I am %s, I work at %s. Call me on %s\n", e.name,
        e.company, e.phone) //Yes you can split into 2 lines here.
}

// Interface Men is implemented by Human, Student and Employee
// because it contains methods implemented by them.
type Men interface {
    SayHi()
    Sing(lyrics string)
}

func main() {
    mike := Student{Human{"Mike", 25, "222-222-XXX"}, "MIT", 0.00}
    paul := Student{Human{"Paul", 26, "111-222-XXX"}, "Harvard", 100}
    sam := Employee{Human{"Sam", 36, "444-222-XXX"}, "Golang Inc.", 1000}
    Tom := Employee{Human{"Sam", 36, "444-222-XXX"}, "Things Ltd.", 5000}

    //a variable of the interface type Men
    var i Men

    //i can store a Student
    i = mike
    fmt.Println("This is Mike, a Student:")
    i.SayHi()
    i.Sing("November rain")

    //i can store an Employee too
    i = Tom
    fmt.Println("This is Tom, an Employee:")
    i.SayHi()
    i.Sing("Born to be wild")

    //a slice of Men
    fmt.Println("Let's use a slice of Men and see what happens")
    x := make([]Men, 3)
    //These elements are of different types that satisfy the Men interface
    x[0], x[1], x[2] = paul, sam, mike

    for _, value := range x{
        value.SayHi()
    }
}
```

输出：

	This is Mike, a Student:
	Hi, I am Mike you can call me on 222-222-XXX
	La la la la... November rain
	This is Tom, an Employee:
	Hi, I am Sam, I work at Things Ltd.. Call me on 444-222-XXX
	La la la la... Born to be wild
	Let’s use a slice of Men and see what happens
	Hi, I am Paul you can call me on 111-222-XXX
	Hi, I am Sam, I work at Golang Inc.. Call me on 444-222-XXX
	Hi, I am Mike you can call me on 222-222-XXX
	
你可能已经注意到了：Interface类型是抽象的，它们自身并不实现指定的、精确的数据结构或方法。它们只是简单的描述：”如果有对象能够做到这些，它们就可以在这里被使用”。

**注：这些类型都没有提及任何的interface，它们的实现不会明确的提到某个具体的interface。**类似的，一个interface也不会指定或在乎哪个类型需要实现它。看看`Men`是怎么做到的，没有提到`Student`和`Employee`类型。关于Interface的事实就是，如果一个类型实现了它所声明的接口，则它可以被用于应用该类型的值。

## Empty Interface
Empty Interface `interface{}`一个方法也不包含，所以任何类型都实现了它所有的0个方法。

Empty Interface对于描述一个行为并没有什么用（显然，它是一个只有很少字符的实体）。但是当我们想要存储任何类型的值时，它就显得非常有用了（因为所有类型都实现了empty interface）。

``` Go
// a is an empty interface variable
var a interface{}
var i int = 5
s := "Hello world"
// These are legal statements
a = i
a = s
```

如果一个函数的传参是`interface{}`类型，则它可以接受任何类型的参数。同理，如果一个函数返回一个`interface{}`，我们希望它能返回任何类型的数据。

这个到底多有用？继续往下读，你就会发现。

## 使用Interface传参的函数
前面的示例显示interface变量能够存储实现它的任何类型的任何值，这给了我们一些如果针对不同的数据（不同的类型）构建容器。顺着这个思路，我们可以考虑接受Interface传参的函数或方法以便该函数或方法能够处理实现接口的任何类型。

例如：现在你已经知道`fmt.Print`是一个可变参数函数，对吧？它可以接受任何数量的实参。但是你注意到了，有时候我们使用的是`string`，有时候是`int`，有时候又是`float`等类型的参数了吗？事实上，如果你看过`fmt package`的源码、文档，你将发现有一个叫`Stringer`的Interface被定义了。

``` Go
//The Stringer interface found in fmt package
type Stringer interface {
     String() string
}
```

任何一个实现了`Stringer`接口的类型都可以传递给函数`fmt.Print`， `fmt.Print`将打印该类型的`String`方法返回的string。

让我们来试一下：

``` Go
package main
import (
    "fmt"
    "strconv" //for conversions to and from string
)

type Human struct {
    name string
    age int
    phone string
}

//Returns a nice string representing a Human
//With this method, Human implements fmt.Stringer
func (h Human) String() string {
    //We called strconv.Itoa on h.age to make a string out of it.
    //Also, thank you, UNICODE!
    return "❰"+h.name+" - "+strconv.Itoa(h.age)+" years -  ✆ " +h.phone+"❱"
}

func main() {
    Bob := Human{"Bob", 39, "000-7777-XXX"}
    fmt.Println("This Human is : ", Bob)
}
```
输出：

	This Human is : ❰Bob - 39 years - ✆ 000-7777-XXX❱
	
撸一遍刚才我们是怎么使用`fmt.Print`的，我们给了它一个`Human`类型的`Bob`作为实参，然后它非常完美的打印出来了。我们所有要做的就是为`Humen`类型实现一个简单的`String`方法返回一个string。这样做就意味着`Humen`类型满足了`fmt.Stringer`Interface类型，因此`fmt.Print`能够接受`Humen`类型作为传参。

还能想起[colored boxex example](http://go-book.appspot.com/methods.html#colored-boxes-example)吗？我们有一个`Color`类型，实现了一个`String`方法，让我们回到那个示例，并替代`fmt.Println`直接使用这里的方法打印color，你将看到它如你期望的一样工作。

``` Go
//These two lines do the same thing
fmt.Println("The biggest one is", boxes.BiggestsColor().String())
fmt.Println("The biggest one is", boxes.BiggestsColor())
```

很酷，是不是？

另一个让你深爱interface的示例是`sort package`。`sort package`提供函数对int、float、string以及更多类型的slice进行排序。

先看一个基本的示例，然后我们将让你看到这个包的魔力。

``` Go
package main
import(
    "fmt"
    "sort"
)

func main() {
    // list is a slice of ints. It is unsorted as you can see
    list := []int {1, 23, 65, 11, 0, 3, 233, 88, 99}
    fmt.Println("The list is: ", list)

    // let's use Ints function that comes in sort
    // Ints([]int) sorts its parameter in ibcreasing order. Go read its doc.
    sort.Ints(list)
    fmt.Println("The sorted list is: ", list)
}
```

输出：

	The list is: [1 23 65 11 0 3 233 88 99]
	The sorted list is: [0 1 3 11 23 65 88 99 233]
	
用起来就是这么简单。但是我想给你看的远不止如此。事实上，sort package定义了一个接口，该接口甚至就简单的叫`Interface`，它包含三个方法：

``` Go
type Interface interface {
    // Len is the number of elements in the collection.
    Len() int
    // Less returns whether the element with index i should sort
    // before the element with index j.
    Less(i, j int) bool
    // Swap swaps the elements with indexes i and j.
    Swap(i, j int)
}
```

截取部分`sort package`文档：

*一个满足排序的类型，典型的是一个集合。接口可以通过这个包中的程序进行排序。这些方法要求集合中的元素可以被整型的索引枚举*

所以如果我们想排序任何类型数据的slice，只需要为该类型实现这三个方法。我们继续在`Humen`上做实验，我们期望基于年龄对`Humen`类型的slice排序。

``` Go
package main
import (
    "fmt"
    "strconv"
    "sort"
)

type Human struct {
    name string
    age int
    phone string
}

func (h Human) String() string {
    return "(name: " + h.name + " - age: "+strconv.Itoa(h.age)+ " years)"
}

type HumanGroup []Human //HumanGroup is a type of slices that contain Humans

func (g HumanGroup) Len() int {
    return len(g)
}

func (g HumanGroup) Less(i, j int) bool {
    if g[i].age < g[j].age {
        return true
    }
    return false
}

func (g HumanGroup) Swap(i, j int){
    g[i], g[j] = g[j], g[i]
}

func main(){
    group := HumanGroup{
        Human{name:"Bart", age:24},
        Human{name:"Bob", age:23},
        Human{name:"Gertrude", age:104},
        Human{name:"Paul", age:44},
        Human{name:"Sam", age:34},
        Human{name:"Jack", age:54},
        Human{name:"Martha", age:74},
        Human{name:"Leo", age:4},
    }

    //Let's print this group as it is
    fmt.Println("The unsorted group is:")
    for _, v := range group{
        fmt.Println(v)
    }

    //Now let's sort it using the sort.Sort function
    sort.Sort(group)

    //Print the sorted group
    fmt.Println("\nThe sorted group is:")
    for _, v := range group{
        fmt.Println(v)
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

	The sorted group is:
	(name: Leo - age: 4 years)
	(name: Bob - age: 23 years)
	(name: Bart - age: 24 years)
	(name: Sam - age: 34 years)
	(name: Paul - age: 44 years)
	(name: Jack - age: 54 years)
	(name: Martha - age: 74 years)
	(name: Gertrude - age: 104 years)
	
成功！就像文档中所描述的一样工作。我们并没有专门针对HumenGroup进行排序，只是实现了几个sort的方法（len， less，swap）。`Sort`函数需要那个接口为我们实现slice的排序。

我知道你肯定很好奇，想知道这个到底是怎么工作的。其实很简单，sort package中Sort函数的签名是这样的：`func Sort(data Interface)`。它接受任何实现了`sort.Interface`接口的类型，然后深入调用你为你的类型定义的len, less, swap方法。

去看看`sort package`的源码吧，你将很容易的看到这些。

我们已经看了函数使用interface传参的不同示例，这些interface的传参提供对不同类型的抽象。下面让我们自己来实践一下，让我们脑洞大开自己来写一个接收interface的函数吧。

## 我们自己的示例

你是否还记得我们前面看过了`Max(s []int) int`函数呢？还有`Older(s []Person) Person`。他们都有相同的功能。事实上检索int、float slice或一组人中的older，其实都是在干相同的事情：循环和比较大小。

开工！

``` Go
package main
import (
    "fmt"
    "strconv"
)

//A basic Person struct
type Person struct {
    name string
    age int
}

//Some slices of ints, floats and Persons
type IntSlice []int
type Float32Slice []float32
type PersonSlice []Person

type MaxInterface interface {
    // Len is the number of elements in the collection.
    Len() int
    //Get returns the element with index i in the collection
    Get(i int) interface{}
    //Bigger returns whether the element at index i is bigger that the j one
    Bigger(i, j int) bool
}

//Len implementation for our three types
func (x IntSlice) Len() int {return len(x)}
func (x Float32Slice) Len() int {return len(x)}
func (x PersonSlice) Len() int {return len(x)}

//Get implementation for our three types
func(x IntSlice) Get(i int) interface{} {return x[i]}
func(x Float32Slice) Get(i int) interface{} {return x[i]}
func(x PersonSlice) Get(i int) interface{} {return x[i]}

//Bigger implementation for our three types
func (x IntSlice) Bigger(i, j int) bool {
    if x[i] > x[j] { //comparing two int
        return true
    }
    return false
}

func (x Float32Slice) Bigger(i, j int) bool {
    if x[i] > x[j] { //comparing two float32
        return true
    }
    return false
}

func (x PersonSlice) Bigger(i, j int) bool {
    if x[i].age > x[j].age { //comparing two Person ages
        return true
    }
    return false
}

//Person implements fmt.Stringer interface
func (p Person) String() string {
    return "(name: " + p.name + " - age: "+strconv.Itoa(p.age)+ " years)"
}

/*
 Returns a bool and a value
 - The bool is set to true if there is a MAX in the collection
 - The value is set to the MAX value or nil, if the bool is false
*/
func Max(data MaxInterface) (ok bool, max interface{}) {
    if data.Len() == 0{
        return false, nil //no elements in the collection, no Max value
    }
    if data.Len() == 1{ //Only one element, return it alongside with true
        return true, data.Get(1)
    }
    max = data.Get(0)//the first element is the max for now
    m := 0
    for i:=1; i<data.Len(); i++ {
        if data.Bigger(i, m){ //we found a bigger value in our slice
            max = data.Get(i)
            m = i
        }
    }
    return true, max
}

func main() {
    islice := IntSlice {1, 2, 44, 6, 44, 222}
    fslice := Float32Slice{1.99, 3.14, 24.8}
    group := PersonSlice{
        Person{name:"Bart", age:24},
        Person{name:"Bob", age:23},
        Person{name:"Gertrude", age:104},
        Person{name:"Paul", age:44},
        Person{name:"Sam", age:34},
        Person{name:"Jack", age:54},
        Person{name:"Martha", age:74},
        Person{name:"Leo", age:4},
    }

    //Use Max function with these different collections
    _, m := Max(islice)
    fmt.Println("The biggest integer in islice is :", m)
    _, m = Max(fslice)
    fmt.Println("The biggest float in fslice is :", m)
    _, m = Max(group)
    fmt.Println("The oldest person in the group is:", m)
}
```

输出：

	The biggest integer in islice is : 222
	The biggest float in fslice is : 24.8
	The oldest person in the group is: (name: Gertrude - age: 104 years)

`MaxInterface` interface包含三个必须实现的接口：

* `Len() int`：必须返回集合的长度
* `Get(int i) interface{}`：返回集合中索引为i的元素。注意，我们设计它返回的结果为interface{}类型。这个集合可以是任何类型的集合，并能够将集合中的值通过空interface返回。
* `Bigger(i, j int) bool`：如果集合中索引为i的值大于索引为j的值则返回true，否则返回false。

为什么是这3个方法？

* `Len() int`：因为没有任何假设说集合必须是slice。想象一个复杂的数据结构可能有它自己的数据长度定义。
* `Get(int i) interface{}`：同样，没有人说这是在处理slice集合。复杂的数据结构可能以更加复杂的方式存储和检索它们的元素。
* `Bigger(i, j int) bool`：比较数字类型是显而易见的，我们知道那一个更大。而且编程语言都有数值比较的操作符：<，>, =等。但是更大的值这个概念比较微妙。Person A比B更老（大的另一种表达方式），如果A的年纪比B大。

我们的各种类型对这几个方法的实现也是相当的简单粗暴。

这段程序的核心是函数`Max(data MaxInterface)(ok bool, max interface{})`, 它接收实现MaxInterface接口的data，并返回两个值得结果组合。一个返回时bool类型, true表示有最大值，否则表示collection是空的），另一个空接口用于返回集合的最大值。

注意`Max`函数是怎么实现的：没有提到任何的集合类型，它所用到的所用方法都是源于接口定义。这种抽象正是Max函数能够对任何实现了MaxInterface接口的类型有效的保证。每次我们针对给定类型的数据调用Max函数，它都调用该类型所实现的方法。相当的有魅力。如果我们需要找出任何集合的最大值，我们只需要为那个类型实现MaxInterface接口。它就能够像`fmt.Print`一样工作。

这一章就讲到这里，稍作休息然后开始美好的一天。下一次我们将讨论一些interface的细节。不用担心，我保证比这章更简单。

C’mon, admit it! You’ve got a NERDON after this chapter!!!
