---
title: js中的__proto__和prototype
date: 2015-12-30 20:01:22
tags: [Javascript]
category: 前端
---
## 参考链接
[JS的prototype和__proto__](http://www.cnblogs.com/yangjinjin/archive/2013/02/01/2889103.html)
[原型链](http://rockyuse.iteye.com/blog/1426510)
[**jS核心指南](http://www.cnblogs.com/ifishing/archive/2010/12/08/1900594.html)
[**js对象模型](http://www.cnblogs.com/RicCC/archive/2008/02/15/JavaScript-Object-Model-Execution-Model.html)
[原型误区](http://www.laruence.com/2010/05/13/1462.html)
## prototype和`__proto__`的概念
`prototype`是函数的一个属性（每个**函数**都有一个prototype属性），这个属性是一个指针，指向一个对象。它是显示修改对象的原型的属性。
`__proto__`是一个**对象**拥有的内置属性（请注意：prototype是函数的内置属性，__proto__是对象的内置属性），是JS内部使用寻找原型链的属性。
`__proto__`**指向的是对象构造函数的prototype**

> 每个对象都会在其内部初始化一个属性，就是`__proto__`，当我们访问一个对象的属性时，如果这个对象内部不存在这个属性，那么他就会去`__proto__`里找这个属性，这个`__proto__`又会有自己的`__proto__`，于是就这样一直找下去，也就是我们平时所说的原型链的概念。

使用obj.propName访问一个对象的属性时，按照下面的步骤进行处理(假设obj的内部[[Prototype]]属性名为`__proto__`):

(1)如果obj存在propName属性，返回属性的值，否则
(2)如果`obj.__proto__`为null，返回undefined，否则
(3)返回`obj.__proto__.propName`
调用对象的方法跟访问属性搜索过程一样，因为方法的函数对象就是对象的一个属性值。
提示: 上面步骤中隐含了一个递归过程，步骤3中`obj.__proto__`是另外一个对象，同样将采用1, 2, 3这样的步骤来搜索propName属性。

```js
function Foo() {};
var foo = new Foo();
Foo.prototype.label = "laruence";
alert(foo.label); //output: laruence
alert(Foo.label);//output: undefined
```

```js
//Foo是函数，函数的构造函数都是Function
function Foo() {};
alert(Foo.__proto__ === Foo.prototype); //output: false
alert(Foo.__proto__ === Function.prototype); //output: true
```

![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2016-03-03-14525824622530.jpg)

## 对象创建过程
JS中只有函数对象具备类的概念，因此要创建一个对象，必须使用函数对象。
函数对象内部有[[Construct]]方法和[[Call]]方法，[[Construct]]用于构造对象，[[Call]]用于函数调用，只有使用new操作符时才触发[[Construct]]逻辑。
var obj=new Object(); 是使用**内置的Object这个函数对象**创建实例化对象obj。
var obj={};和var obj=[];这种代码将由JS引擎触发Object和Array的构造过程。function fn(){}; var myObj=new fn();是使用用户定义的类型创建实例化对象。

```
var Person = function(){};
var p = new Person();
```
new的过程拆分成以下三步：
(1) var p={}; 也就是说，初始化一个对象p
(2) p.__proto__ = Person.prototype;
(3) Person.call(p); 也就是说构造p，也可以称之为初始化p


