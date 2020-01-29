---
title: Go中的CanAddr和addresable
date: 2019-10-23 20:15:48
tags: [Golang]
category: Golang
---

在Go中，通过反射来修改变量的值（即调用reflect.Value对象的SetXXX方法），必须保证reflect.Value是**可寻址(addresable)**的，reflect.Value的CanAddr判断一个Value是否可取地址。
>CanAddr reports whether the value's address can be obtained with Addr. Such values are called addressable. A value is addressable if it is **an element of a slice**, **an element of an addressable array**, **a field of an addressable struct**, or **the result of dereferencing a pointer.** If CanAddr returns false, calling Addr will panic.

当Value是可取址时，可以通过`value.SetXXX来修改value指向的值`。或者通过`value.Addr().Interface()`获取到interface再转换成实际的指针进行修改（本质上这两种方式是一样的，都是对Value中的ptr进行转换再修改）
```go
x := 2
d := reflect.ValueOf(&x).Elem()   // d refers to the variable x

//使用set       
//d.SetInt(4)

//或者使用指针
px := d.Addr().Interface().(*int) // px := &x
*px = 3                           // x = 3
fmt.Println(x)

```
在了解CanAddr之前需要了解一下addressable的概念
## addressable
[go addressable 详解](https://colobu.com/2018/02/27/go-addressable/)
[unaddressable-values](https://go101.org/article/unofficial-faq.html#unaddressable-values)
在Go中，对一个对象进行取址操作(&x)，要求x必须是可取址(addressable)的，只有以下几种值是可寻址的：
1. 变量：`&x`
2. **可寻址**struct的字段:`&s.Name`
3. slice(不管slice是否可寻址)元素：`&arr[0]`
4. **可寻址**数组的元素：`&a[0]`
5. 指针解引用操作获取的值：`x := &somevar  px:= &*x`

### 为什么slice的元素一定是可寻址的
go中任意slice的底层结构都是一个reflect.SliceHeader的结构体，其data是一个指向其底层数组首元素的指针。从结构体可以看出来，**任何slice的底层数组一定是可寻址的，否则无法得到data指针**，因此任意slice的元素都是可寻址的。

```go
SliceHeader struct {
    data unsafe.Pointer
    len int
    cap int
}
```
### unaddressable values
在这篇文章：[不可寻址的values](https://go101.org/article/unofficial-faq.html#unaddressable-values) 中详细列了不可寻址的值类型。主要包括：
1. 字符串中的字节(如`s := "abc"  ps:=&s[0]`)
2. map元素
3. 通过`type assertions`获取的接口对象的动态值
4. 常数
5. 字面量
6. 包级别的函数
7. 用作函数值的方法
8. 中间值
    * 函数调用
    * 显示类型转换
    * 各种类型操作的中间值（除了指针的取值操作）
        * channel接收（如`&<-somechane`）
        * 子字符串操作
        * 子slice操作
        * 加减乘除运算

**注意第五条**：
实际测试可以发现，字面量的结构体的寻址操作(`&struct{Name string}{"nn"}`)是可以的，但是这并不意味着字面量结构体是可寻址的，因为这在go中这是一个语法糖(`&struct{Name string}{"nn"}`)等价于`tmp := struct{Name string}{"nn"}; (&tmp)`。
### 为什么字符串的字节是不可寻址的
因为字符串是不可变的，如果其字节是可寻址的，就可以通过指针修改值，破坏了定义。
同理，常量不可更改也是这个原因。
### 为什么map元素是不可寻址的
两个原因：
1. map["key"]取元素的操作，不管map为nil或者不包含该key，都会默认返回一个类型的零值。为了安全考虑，类型的该零值是不可寻址的（另外一点，即使零值是可寻址的，也需要在运行时判断一个值是否是类型零值）。
2. map元素的地址可能发生变化，如果允许寻址，可能导致获取的地址在之后无效。

### 不可寻址的struct和array
一些字面量的struct和array是不可寻址的，其元素也是不可寻址的。注意slice和array的区别，**slice任意情况下都可以寻址**
```go
   //error
	a := &struct {
		Name string
	}{"x"}.Name
	
	//error
	b := &[3]int{1,2,3}[0]

   //correct
	c := &[]int{1,2,3}[0]
```
### CanAddr
`CanAddr`只有以下几种类型的`reflect.Value`才返回true：
1. slice元素
2. 可寻址数组的元素
3. 可寻址struct的field
4. 指针解引用的结果

**对比addressable的定义，少了变量**。变量是addressable的但是它的reflect.Value不是`CanAddr`的，主要因为栈上的变量在函数调用结束时会被释放，如果是CanAddr的可能带来安全问题（例如对函数内的变量执行Addr后返回）。对比对变量进行寻址不会有安全问题，因为如果变量的地址在函数外部保存了，逃逸分析会将该变量改为在堆上开辟空间。

注意第四条：**只有指针解引用的结果的Value是CanAddr的，但是指针本身的Value是不能Addr的**（如`reflect.ValueOf(&x)`不是CanAddr的，但是`reflect.ValueOf(&x).Elem()`可以）

```go
type refStruct struct {
	Name string
}

//error
s := refStruct{}
svN := reflect.ValueOf(s).FieldByName("Name")
//panic: reflect: reflect.Value.SetString using unaddressable value
svN.SetString("hehe")

//correct
s := refStruct{}
svN := reflect.ValueOf(&s).Elem().FieldByName("Name")
svN.SetString("hehe")

```
### CanAddr和CanSet
`CanAddr`和`CanSet`的区别在于`CanSet`还会判断struct的filed是否是可导出的，只有可导出的字段才能修改。
>CanAddr reports whether the value's address can be obtained with Addr. Such values are called addressable. A value is addressable if it is an element of a slice, an element of an addressable array, a field of an addressable struct, or the result of dereferencing a pointer. If CanAddr returns false, calling Addr will panic.
#### Addr和Pointer
Pointer返回一个指针指向的地址(uintptr)，只有chanel、slice、map、func、指针类型可以调用
Addr对value进行取址操作，并返回一个原始value指针的Value，原始value.CanAddr为true时才能调用
## 参考链接
[Go-addresable详解](https://colobu.com/2018/02/27/go-addressable/)