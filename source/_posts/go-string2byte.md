---
title: 优化string和byte转换
date: 2019-10-25 17:15:48
tags: [Golang]
category: Golang
---
源自fastHttp，利用StringHeader和SliceHeader的结构只差一个cap来进行转换，减少转换开销。
需要注意的是，这种优化方式，用`s2b`转换出来的byte数组不能有修改操作（因为string底层的内存不允许修改）。

```go
//bs.go
package bs

import (
	"unsafe"
	"reflect"
)

// type StringHeader struct {
// 	Data uintptr
// 	Len  int
// }
//
// type SliceHeader struct {
// 	Data uintptr
// 	Len  int
// 	Cap  int
// }

func b2s(b []byte) string {
   //byte数组和string的结构只差一个Cap，直接转换截断
	return *(*string)(unsafe.Pointer(&b))
}

// !!!caution: 转出来的byte不能有修改操作
func s2b(s string) []byte {
	ps := (*reflect.StringHeader)(unsafe.Pointer(&s))

	b := reflect.SliceHeader{
		Data: ps.Data,
		Len:  ps.Len,
		Cap:  ps.Len, //增加数组的Cap字段
	}
	
	//sliceHeader转为实际类型的数组
	return *(*[]byte)(unsafe.Pointer(&b))
}

func normal_b2s(b []byte) string {
	return string(b)
}

func normal_s2b(s string) []byte {
	return []byte(s)
}

```

## benchmark
两种转换bench的结果对比, 由于减少了拷贝，性能比常用的直接转换提高了20倍
![-w558](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/2019/10/25/15719950374880.jpg)


原始的转换方式，pprof的结果，可以看出来大部分的时间消耗在内存拷贝上
![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/2019/10/25/15719950735681.jpg)

附benchmark程序：
```go

var testBytes = []byte("测试字符串bytes")
var testString = "测试字符串bytes"

//  go test -v -run="none" -bench=".*B2S"
func BenchmarkB2S(b *testing.B) {
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		b2s(testBytes)
	}
}

func BenchmarkNormalB2S(b *testing.B) {
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		normal_b2s(testBytes)
	}
}

func BenchmarkS2B(b *testing.B) {
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		s2b(testString)
	}
}

func BenchmarkNormalS2B(b *testing.B) {
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		normal_s2b(testString)
	}
}
```


