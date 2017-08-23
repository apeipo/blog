---
title: Python2和3编码整理
date: 2017-07-06 14:42:22
tags: [Python]
category: 后端
---
## 参考文章
[Python编码](https://funhacks.net/explore-python/Basic/character_encoding.html)
[Python2 与 Python3 的编码对比](http://kuanghy.github.io/2016/10/15/encoding-python2-vs-python3)
## UnicodeEncodeError& UnicodeDecodeError
用 Python2 编写程序的时候经常会遇到 UnicodeEncodeError 和 UnicodeDecodeError，它们出现的根源就是如果代码里面混合使用了 str 类型和 unicode 类型的字符串，**Python 会默认使用 ascii 编码尝试对 unicode 类型的字符串编码 (encode)，或对 str 类型的字符串解码 (decode)**。
例如：**s = u'中文' + '中国'**, python2会隐式地调用 `'中国'.decode('ascii')`，发生decodeError。
同样的，如果某些方法要求str类型，而调用时传入了unicode类型，python会使用encode('ascii')对传入值进行转换，如果穿入值存在非ascii字符，就会发送EncodeError
## Python2和3的字符串
### 字符串类型
**Python2 中字符的类型**：
str： 某种编码（UTF-8，GBK等）类型的字节序列
unicode： Unicode类型的字符串
**Python3 中字符的类型**：
str： Unicode类型的字符串
bytes： 某种编码（UTF-8，GBK等）类型的字节序列

不管是Python2还是3，我们可以认为字符串有两种状态：
1. 未编码的状态（unicode字节序列），如中文对应：u'\u4e2d\u6587'
2. 编码后的状态（按指定编码转换字节序列），如中文按utf-8编码后是'\xe4\xb8\xad\xe6\x96\x87'

Python2 和 Python3 中的两种字符类型都分别对应这两种状态，然后相互之间进行编解码转化。
实际上，Python3中的str可以认为是Python2中的unicode类型，而bytes相当于Python2中的str类型。
### encode和decode
编码(encode)就是将字符串转换成指定编码的字节码；解码(decode)就是按指定的编码解析字节序列，将比特位显示成字符。
Python2和3中两个方法的对比：

|  | Python2 | Python3 |
| --- | --- | --- |
| encode | unicode 按指定编码转为 str（即字节码），默认ascii | str(unicode) 编码为 bytes,默认使用utf-8 **只有str类型有该方法** |
| decode | str 按指定编码解码为 unicode，默认ascii) | bytes 解码为 str ，默认使用utf-8 **只有bytes类型有该方法** |

在 Python2 中，str 和 unicode 都有 encode 和 decode 方法。但是不建议对 str 使用 encode(会内部隐式地先decode解码)，对 unicode 使用 decode(会内部隐式地先encode编码), 这是 Python2 设计上的缺陷。
Python3 则进行了优化，**str 只有一个 encode 方法将字符串转化为一个字节码，而且 bytes 也只有一个 decode 方法将字节码转化为一个文本字符串**。
### 字符串处理的前提
对于Python文件中字符串的解析，有一些前提：

1. Python 文件开始已经声明对应的编码
2. Python 文件本身的确是使用该编码保存的
3. 两者的编码类型要一样（比如都是 UTF-8 或者都是 GBK 等）

**满足这些前提，Python解析器才能正确的把文本解析为对应的unicode**。
同样的，在Python2中，如果用u'中文'这种方式定义字符串也得满足上诉前提，否则Python无法将其转为unicode字符（定义普通的str不存在该问题，因为Python2中的str就是字节序列，Python不会转为unicode）。
### 其他区别

1. Python2 的 str 和 unicode 都是 basestring 的子类，所以两者可以直接进行拼接操作。而 Python3 中的 bytes 和 str 是两个独立的类型，两者不能进行拼接。
2. Python2 中，普通字符串（如：s = "中文"）的编码类型，对应着你的 Python 文件本身保存为何种编码有关，最常见的 Windows 平台中，默认用的是 GBK。Python3 中，定义的字符串就已经是 Unicode 类型的 str了（Python解释器根据文件编码转换而来）。

总体来说，Python3的编码表示更直接，字符串总是unicode，二进制制是bytes，不像Python2中将两者混在一起造成处理上的混淆。


