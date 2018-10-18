---
title: Golang内存模型
tags: [Golang，内存]
category: [后端]
date: 2018-09-12 12:12:14
---
[参考链接-Go语言内存模型](http://hugozhu.myalert.info/2013/04/20/31-golang-memory-model.html)
本文章主要是结合个人理解对官方文章[The Go Memory Model](https://golang.org/ref/mem)的翻译
## 背景
内存模型的定义：
[参考链接-内存模型和Java内存模型](https://mp.weixin.qq.com/s/ME_rVwhstQ7FGLPVcfpugQ)
[参考链接-CPU缓存一致性入门](http://www.infoq.com/cn/articles/cache-coherency-prime)
简单来说，内存模型定义了并发环境下操作变量的规范，以保证共享数据的可见性、原子性。
不同的语言有各自的内存模型，如Java通过volite、sync等一些列关键字来保证。
Golang的内存模型规定了保证变量在不同gorotune之间可见性（一个G中对变量地更改能被其他读取该变量的G观测到）的条件。
## Happens Before
在单个gorotune中，读写和代码中的顺序一致，即编译器和CPU对程序的优化必须以不改变语言规范定义的行为为前提。如下代码，在编译执行期间，编译器和CPU都可能优化指令，比如先执行`b=2`在执行`a=1`。但是`c=a+2`一定是在`a=1`之后执行的。
```go
package main
import (
    "log"
)
var a, b, c int
func main() {
    a = 1
    b = 2
    c = a + 2
    log.Println(a, b, c)
}
```
由于指令重排和优化的存在，一个gorotune中对变量的操作顺序和在另一个gorotune中观测到更改的顺序可能是不一样的。如下顺序，可能输出`000 122 023（先观测到b的更改）`
```go
package main
import (
    "log"
)
var a, b, c int
func main() {
    go func() {
        a = 1
        b = 2
    }()
    go func() {
        c = a + 2
    }()
    log.Println(a, b, c)
}
```
### 定义
`happens-before`用来描述go程序中局部的内存操作顺序。如果一个内存操作`e1 happens-before e2`，则`e2 happens-after e1`。如果`e1 not happens-before e2 并且e1 not happens-after e2`则e1和e2是并发的（顺序无法判断）。
1. 在单个gorotune内，happends-before就是程序代码的顺序。
2. 两个事件之间存在三种关系：`happens-before` `concurrent` `happens-after`，concurrent表示两个事件的顺序是不确定的。（这儿也可以看出来，`not happens-before != happens-after`，因为还可能有`coucurrent`）
3. happens-before是可推导的，`A happens-before B happens-before C => A happens-before C`

#### 1.允许可见
当满足以下两个条件时，对变量v的读操作**允许**观测到对变量w的写操作（有点绕口，或者说v的写操作对v的读操作是`允许`可见的）：
1. r not happens-before w（**r在w之后或者并发**）
2. w和r之间没有其他写操作 (其他w可以和w并发)

#### 2.保证可见（garuantee）
如果要保证w对r可见，就需要确保w是r唯一允许可见的写操作。当满足以下两个条件时，对变量v地写操作对读操作是`保证`可见的：
1. w happens-before r
2. 所有其他队v的写操作，只happens-before w或 happens-afer r

**2.保证可见的要求比1.允许可见严格，它要求没有其他的写和wr并发**。
在单个gorotune中，由于没有并发，1和2是等价的：对一个变量地读能读到最近一次对变量地写。
在多个gororune并发时，如果想让写对某个期望的读可见（读到期望的值），则必须使用同步事件来确定读写的顺序。

>从本质上来讲，happens-before规则确定了CPU缓冲和主存的同步时间点（通过内存屏障等指令），从而使得对变量的读写顺序可被确定–也就是我们通常说的“同步”

go中可以通过锁和channel通讯来进行同步，同时有以下一系列happens-before规则可以用来确定事件的顺序。
## 同步规则
### 包初始化
1. 如果package-p引用了package-q，则q的init方法happens-before p的任意方法执行
2. main.main方法开始执行 happens-after 所有init方法执行完成。

### Gorotune创建
gorotune创建语句happens-before 该gorotune的执行语句开始执行
如下代码，可以保证能输出`hello`，因为语句`@1 happens-before @2` `@2 happens-before @3`可以推导出`@1 happens-before @3`.
```go
var a string

func f() {
	print(a)           //@3
}

func hello() {
	a = "hello, world"   //@1
	go f()               //@2
	time.Sleep(1 * time.Second)
}
```
### Gorotune销毁
gorotune的退出不保证happens-before任意事件。如下代码，@1和@2之间没有同步事件，不能保证@1 happens-before @2，因此输出是不确定的，可能是空也可能是hello。
在极端情况下，编译器可能优化删除整个gorotune声明。
```go
var a string
func hello() {
    go func() { 
        a = "hello"  //@1
    }()  
    print(a) //@2
}
```
### Channel通讯
channel通讯是go中的主要同步方法，每个channel的发送操作都会有一个与之匹配的receive操作，receive大多数都是在其他gorotune中，因此定义了以下规则来确定happens顺序：
* channel发送操作happens-before对应的receive操作完成

根据该规则，如下代码能保证输出helloworld.(@2 happens-before @3 happens-before @1==@4)
```go
var c = make(chan int, 10)
var a string

func f() {
	a = "hello, world" //@2
	c <- 0            //@3
}

func main() {
	go f()
	<-c               //@1
	print(a)          //@4
}
```

* channel的关闭操作happens-before接受操作接收到最后的0值 

根据该规则，同样是上面代码，将@3语句修改为`close(c)`同样能保证输出hello-world

* 无缓冲channel的receive happens-before对应的send操作完成

根据该规则，如果上面代码中将c修改为无缓冲的channel，则需要调换@3和@1语句才能保证输出hello-world
```go
var c = make(chan int)
var a string

func f() {
	a = "hello, world" //@2
	<-c           //@3
}

func main() {
	go f()
	c<-0               //@1
	print(a)          //@4
}
```

* 对于容量为C的缓冲通道，第n次receive happens-before第n+k次send.

这条规则保证了任意时刻缓冲通道中的元素不会超过其容量。当C=0时，这条规则等同于无缓冲通道的happens-before规则。
这个特性常用于限制并发资源的数量，如下代码，限制同一时刻，最多三个gorotune在运行`w()`
```go
var limit = make(chan int, 3)

func main() {
	for _, w := range work {
		go func(w func()) {
			limit <- 1
			w()
			<-limit
		}(w)
	}
	select{}
}
```

### Locks
sync包提供了两种锁的数据结构`sync.Mutex`和`sync.RWMutex`，锁的happens-before规则：
* 对于任意锁变量l，定义`n<m`，第n次l.Unlock() happens-before 第m次l.Lock()

这个定义看着有点绕，简单点说就是：`mutex.Unlock happens-before 下一次（或者多次）Lock`
如下代码，能保证输出hello-world，第一次unlock happens-before 第二次lock
```go
var l sync.Mutex
var a string

func f() {
	a = "hello, world"
	l.Unlock()
}

func main() {
	l.Lock()
	go f()
	l.Lock()
	print(a)
}
```

* For any call to l.RLock on a sync.RWMutex variable l, there is an n such that the l.RLock happens (returns) after call n to l.Unlock and the matching l.RUnlock happens before call n+1 to l.Lock.

这段描述了读写锁的happens规则，实在找不到好的翻译。读写锁有4个操作`Lock,UnLock,RLock,RUnlock`，第n次RLock happens-after 第n次UnLock，与该RLock对应的RUnlock happens-before 第n+1次Lock
### Once
Once.Do实现了单例模式，在并发下，Do调用的函数只会被执行一次。
* 对于`Once.Do(f)`，对于函数f的调用f() happens-before 任意Once.Do返回。

## 错误的同步方法
在并发环境下，如果不关注gorotune之间的同步，会引发很多可见性造成的问题（不稳定）
### case1
如下代码，可能输出2和0，因为在主gorotune中观测到的变量a，b的写顺序是无法保证的。
```go
var a, b int

func f() {
	a = 1
	b = 2
}

func g() {
	print(b)
	print(a)
}

func main() {
	go f()
	g()
}
```
### case2
有时为了保证变量在多个gorotune中只被初始化一次，会用共享变量来做状态检查.
如下代码是一个常见的错误，由于可见性的问题，在某个gorotune中，`done=true`不能保证在`a="hello,world"`被读取到，因此这个可能打印空字符串。
另外，这个例子中once.Do只保证setup被执行一次，但是无法保证可见性。
```go
var a string
var done bool
var once sync.Once

func setup() {
	a = "hello, world"
	done = true
}

func doprint() {
	if !done {
		once.Do(setup)
	}
	print(a)
}

func twoprint() {
	go doprint()
	go doprint()
}
```
### case3
有时我们为了程序等待某个事件结束后退出，会用共享变量来同步处理状态。
如下代码，存在两个问题:
1. done赋值和a变量赋值的顺序在主gorotune中不保证，因此可能输出空字符串
2. done的更改对主gorotune不保证可见（没有同步事件保证），因此程序可能陷入死循环

```go
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func main() {
	go setup()
	for !done {
	}
	print(a)
}
```
### case4
另外一个例子，问题同case3，即使主程序观测到了g==nil的变化退出循环，也无法保证读取到的msg是hello-world

```go
type T struct {
	msg string
}

var g *T

func setup() {
	t := new(T)
	t.msg = "hello, world"
	g = t
}

func main() {
	go setup()
	for g == nil {
	}
	print(g.msg)
}
```

对于上诉这些case，解决的办法都是一样的，在多个gorotune之间加入同步事件。
