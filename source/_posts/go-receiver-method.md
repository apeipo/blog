---
title: Go中指针和非指针Receiver的方法集
date: 2019-10-24 15:15:48
tags: [Golang]
category: Golang
---

在Go中，一个类型实现了某个接口，如果receiver是指针，虽然该类型的实例可以调用接口的方法，但是在进行接口类型判断时无法会被判定为该实例没有实现该接口。但是如果receiver不是指针，是没有问题的。
参考代码如下：

```go
package main

import "fmt"

type SomeIf interface {
	Print()
}

type SomeStruct struct {}
func (s *SomeStruct) Print() {}

func main() {
	var s SomeIf
	ins := SomeStruct{}
	ins.Print() // ok
	s = ins   //compile error: cannot use ins (type SomeStruct) as type SomeIf
}

```

解释这个问题，我们用`T`和`*T`分别代表一个类型的实例和指针。
在进行接口判定时，类型`T`的方法集只包含了receiver为T类型的方法，但是`*T`类型的方法集同时包含了receiver为`T`和`*T`的所有方法。
造成这个差异的主要原因是，将一个`*T`类型传递给一个接口时，任何`*T`都可以解引用得到`T`。
但是相反的，如果一个接口接收了一个T类型数据
1. 不是所有T都是addressable的。(addressable的定义见[go-addressable](http://longlog.me/2019/10/23/go-addressable/))
2. 对于addresable的值，如果进行自动的取址，会带来语言层面的不一致

如下case，buf实现了io.Writer接口，在buf的Write方法中会对buf的内容进行更改，而buf的receiver是指针，Write方法是直接更改调用者。
如果Go自动对`io.Copy(buf, os.Stdin)`中的buf进行自动取地址（假设这里不发生编译错误），会发生什么？

**重要**：因为在函数调用过程会发生变量的复制，即io.Copy中的buf是原始buf变量的复制，从而Write方法中修改的只是buf的复制。从而会导致在Write方法中修改了buf，但是并没有修改原始的buf变量，
很明显，这样带来了不一致，因为我们都知道如果receiver是指针，那么方法中的修改应该是直接改调用者。

```go
var buf bytes.Buffer
io.Copy(buf, os.Stdin) //compile error, buf not implument writer interface

func (buf *bytes.Buffer) Write(...){
    //modify the buf content
}
```
## 另一种理解
另外一种理解，参考这篇文章[method_set](https://go101.org/article/unofficial-faq.html#unaddressable-values)中的解释。
可以这么理解，对于每个定义在`T`类型上的方法，会自动生成一个`*T`类型的方法，在该方法中，自动对`*T`进行解引用再调用。

```go
func (t T) someMethod(...)

//auto gen
func (t *T) someMethod(...) {
    (*t).someMethod(...)
}
```

相反，如果对于每个`*T`上的方法，自动生成`T`的方法，如下代码，**会导致什么问题？**
```go
func (t *T) someMethod(...) {
    t.Name = "hehe"
}

//auto gen
func (rt T) someMethod(...) {
    (&rt).someMethod(...)
}

a := T{}
a.someMethod()
```
同样会产生上诉语言层面的不一致问题，因为rt是a的拷贝，在someMethod中的更改实际上只改了rt，没有更改a。
但是为什么现在在Go中，可以直接通过`T`调用`*T`上的方法？不会发生上诉的不一致问题？，因为这是go加的一个语法糖，在执行`a.someMethod`时，会自动对T进行取地址再调用`(&a).someMethod`，如果a无法取地址(un_addresable)，会产生编译错误（如下代码）。（注意和上面代码的区别，是直接取地址，而不是一个方法间接调用）

```go
type us struct {
	Name string
}

func (u *us) pName()  {
	fmt.Println(u.Name)
}

us{"hehe"}.pName() //runtime error, 因为us{"hehe"}是个字面量，是不可寻址的

# command-line-arguments
gpl/ref/addres.go:20:12: cannot call pointer method on us literal
gpl/ref/addres.go:20:12: cannot take the address of us literal

```
## 参考链接
[different_method_sets](https://golang.org/doc/faq#different_method_sets)
[method_set](https://go101.org/article/unofficial-faq.html#unaddressable-values)