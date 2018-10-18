---
title: Go-Module实践
tags: [Golang]
category: [后端]
date: 2018-10-18 12:00:14
---
go1.11版本的一大变化是新增了go-module支持，简化go项目的依赖管理。目前go-module的资料较少，只能通过`go help mod` 和 `go help modules`看一些简单的介绍。

本篇文章通过创建和发布一个lib并在项目中使用它来实践go-modules的版本管理有什么不同
## 准备
先简单看下要发布的lib代码，比较简单，获取任意map类型的key并以数组（用interface包装）的形式返回
git仓库为：`github.com/apeipo/go-mapkeys`
```go
package maputil

import (
	"reflect"
)

// return map keys as interface
// panic if m is not map type
func MapKeys(m interface{}) interface{} {
	mtyp := reflect.TypeOf(m)
	if mtyp.Kind() != reflect.Map {
	   panic("not a map type")
	}
	v := reflect.ValueOf(m)
	keys := v.MapKeys() // panic if not map type
	rkeys := reflect.MakeSlice(reflect.SliceOf(mtyp.Key()), 0, v.Len())
	for _, key := range keys {
		rkeys = reflect.Append(rkeys, key)
	}
	return rkeys.Interface()
}
```
## 创建go-module
```go
git clone github.com/apeipo/go-mapkeys
cd go-mapkeys
go mod init github.com/apeipo/go-mapkeys
```
执行以上代码就完成了一个go-module的初始化，会在目录下创建`go.mod`文件。文件内容如下：

```sh
module github.com/apeipo/go-mapkeys
```

注意：**go-mapkeys不能放在GOPATH路径下**，否则会报错。这一点官方的说法是**未来GOPATH可能会取消**，不希望Go的开发者总是理解不了为什么模块需要放到GOPATH下。
### 版本
创建好module后，需要给module加上版本以便其他模块使用。
go-module的版本遵循语义化版本标准[semver](https://semver.org/)，并且通过github的tag对版本进行控制，给mapkeys模块创建一个版本：
```sh
git tag v1.0.0
git push --tags 
```

至此，一个简单的go-module已经创建并且发布了版本`1.0.0`，接下来看看怎么在其他模块中使用
## 使用go-module
先找个目录创建一段测试代码（同样的，**该目录也不能在GOPATH路径下**）
```go
package main

import (
	"github.com/apeipo/go-mapkeys"
	"fmt"
)

func main() {
	m := map[string]int{
		"A" : 20,
		"B" : 30,
	}
	keys := maputil.MapKeys(m)
	fmt.Println(keys)
}
```
创建代码后执行：
```go
go mod init testmod
go build
//find github.com/apeipo/go-mapkeysa
//download github.com/apeipo/go-mapkeys
```
执行过程go会自动扫描代码的仓库依赖并拉取依赖的module，执行完成后go.mod文件中会更新相关的依赖及版本
```go
//go.mod
module test
require (
	github.com/apeipo/go-mapkeys v1.0.0
)
```
同时，目录下回生成一个go.sum文件，文件中记录了依赖包的hash，保证所下载版本的正确性
```sh
//go.sum
github.com/apeipo/go-mapkeys v1.0.0 h1:bPfcNKs0sKWi/cZRkhjv6eNmIeRuymjIBM98ihvlzPc=
github.com/apeipo/go-mapkeys v1.0.0/go.mod h1:bPDRb2IUmwPjag8rByI+zdL3d4/+RkIPggHCioYUhdA=
```
## module更新
完成go-module的创建和使用后，接下来再来看下go-module的更新。
### 小版本(Patch/Minor)更新
第一个版本的改动很小，由于`reflect.MapKeys`调用中已经有了map类型的校验，因此`MapKeys`方法就不需要再多校验一次。我们将代码中的类型校验去掉，并发布版本1.0.1
```go
// return map keys as interface
// panic if m is not map type
func MapKeys(m interface{}) interface{} {
	mtyp := reflect.TypeOf(m)
	v := reflect.ValueOf(m)
	keys := v.MapKeys() // panic if not map type
	rkeys := reflect.MakeSlice(reflect.SliceOf(mtyp.Key()), 0, v.Len())
	for _, key := range keys {
		rkeys = reflect.Append(rkeys, key)
	}
	return rkeys.Interface()
}
```
发布版本
```go
cd go-mapkeys
git commit -m "remove type check" -a
git tag v1.0.1
git push --tags
```
#### 使用方更新
适用方可以通过以下三种方式进行依赖的小版本更新，假设初始版本位`1.0.0`，三种方式的更新策略如下：
```sh
$ go get -u       //更新为最近的minor或者patch版本，如1.0.0会被更新为1.0.1或者1.1.0（存在的话）
$ go get -u=patch //更新为最近的patch版本，1.0.0会更新为1.0.1但是不会更新为1.1.0
$ go get github.com/apeipo/go-mapkeys@v1.0.1 //指定版本
```

进入testmod目录执行`go get/build`，会发现go.mod文件中依赖的版本自动更新为了1.0.1
### 大版本(Major)更新
第二个版本我们给mapkeys增加一个接口，由于MapKeys有可能产生panic，因此我们提供了一个安全版本的接口，对panic进行捕获并以错误的形式返回。
```go
//map_keys.go
func SafeMapKeys(m interface{}) (r interface{}, err error) {
	defer func() {
		if p := recover(); p != nil {
			r = nil
			err = fmt.Errorf("%s", p)
		}
	}()
	mtyp := reflect.TypeOf(m)
	v := reflect.ValueOf(m)
	keys := v.MapKeys() // panic if not map type
	rkeys := reflect.MakeSlice(reflect.SliceOf(mtyp.Key()), 0, v.Len())
	for _, key := range keys {
		rkeys = reflect.Append(rkeys, key)
	}
	return rkeys.Interface(), nil
}

```
该版本新增了功能，因此我们发布一个2.0版本。
先需要修改go.mod文件：
```
module github.com/apeipo/go-mapkeys/v2
```
git发布2.0版本，此步骤与之前相同
```go
cd go-mapkeys
git commit -m "add safe mapkeys" -a
git tag v2.0
git push --tags
```
#### *使用方更新
`go get/build`只能更新依赖的patch和minor版本，不能更新major版本。这里涉及到go-module与其他依赖管理不同的地方，因为根据语义化版本的规范，major版本可以不向前兼容，所以`go-module认为同一个库的不同大版本是两个完全独立的模块`。
我们先在测试代码中引入v2版本（`github.com/apeipo/go-mapkeys/v2`不影响代码中的使用名称）：
```go
//test.go
package main

import (
	"github.com/apeipo/go-mapkeys/v2"
	"fmt"
)

func main() {
	m := map[string]int{
		"A" : 20,
		"B" : 30,
	}
	keys := maputil.MapKeys(m)
	fmt.Println(keys)
	keys, err := maputil.SafeMapKeys("")
	fmt.Println(keys, err)
}

```
再执行`go get`更新依赖，会发现go.mod文件中同时require了两个版本

```sh
//go.mod
module test
require (
	github.com/apeipo/go-mapkeys v1.1.0
	github.com/apeipo/go-mapkeys/v2 v2.0.1
)
```

由于go-module中v1和v2是两个完全独立的模块，因此我们可以在代码中同时引入同一个依赖的不同版本：
```go
package main

import (
	"github.com/apeipo/go-mapkeys"
	maputilv2 "github.com/apeipo/go-mapkeys/v2"
	"fmt"
)

func main() {
	m := map[string]int{
		"A" : 20,
		"B" : 30,
	}
	//use v1.0
	keys := maputil.MapKeys(m)
	fmt.Println(keys)
   //use v2.0
	keys, err := maputilv2.SafeMapKeys("")
	fmt.Println(keys, err)
}

```
## 其他
### 下载路径
模块使用`gomod`初始化后，执行`go get`时，依赖会被保存到`$GOPATH/pkg/mod/$gitpath`路径下，同时以版本号的形式区分。如`mapkeys`这个模块的下载路径时`$GOPATH/pkg/mod/github.com/apeipo/go-mapkeys@v1.1.0`
### vendor
go-mod默认不使用`vendor`目录，依赖不保存在`vendor`目录下，在执行build时会忽略`vendor`目录。

如果要从`vendor`中加载依赖（一些不联通外部网络的环境下回用到），先要使用`go mod vendor`命令将依赖的源码全部拷贝到vendor目录下。再使用`go build -mod vendor`进行编译
### 项目迁移
执行`go mod init modulename`后执行`go get`，后面的事情交给go就行，会自动扫描和下载依赖。
实际测试发现，如果原来的项目是glide，go-mod会直接从`glide.yml`中解析依赖生成go.mod文件。
### 翻墙
`golang.org`下的库无法直接访问，可以再`go.mod`文件中使用replace进行路径替换
```sh
replace (
	golang.org/x/text v0.3.0 => github.com/golang/text v0.3.0
)
```
需要注意的是，replace只对go.mod所在模块生效。例如A模块依赖B模块，B模块中使用了replace，对A模块的下载时不生效的。
## 总结
1. go-mod的使用极其简单，不管是对于服务提供者还是调用方来说，基本上`go mod init`+`go get`就搞定了一切。
2. 和其他依赖管理最大的区别在于不同Major版本认为是不同的模块。（这一点上不知道实践中反响如何，从上面的实践来看，major版本无法直接升级，需要修改代码中的import路径）
3. 当前相关的工具链和文档还不够完善，因为使用了版本相关的路径，大部分IDE中还无法识别gomod导入的模块。

## 参考文章
1. [Introduction to Go Modules](https://roberto.selbach.ca/intro-to-go-modules/)
2. [跳出go-module的泥潭](https://colobu.com/2018/08/27/learn-go-module/)
