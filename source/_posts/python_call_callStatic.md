---
title: Python中实现PHP的魔术方法call和callStatic
date: 2017-07-24 12:41:22
tags: [Python,PHP]
category: 后端
---
## call和callStatic
有PHP开发经验的同学应该对call和callStatic不陌生，
**__call**方法： 当调用类的方法时，方法不存在或权限不足，会自动调用`__call` 方法。
**__callStatic**方法：  当调用类的静态方法时，方法不存在或权限不足，会自动调用`__callStatic`方法。
如下一个简单的例子，基于`__callStatic`实现一个Redis的简单代理，可以在请求redis之前和之后做一些其他的操作，如耗时、日志。

```php
class Redis {
	public static $redisIns;
	public static function getInstance() {
		//redis初始化
		//...
		return static::$redisIns;
	}
	public static function __callStatic($func, $args) {
		//在请求执行做一些操作
		//...
		//请求具体的redis方法
		$res = '';
		if(static::$redisIns) {
			$res =  static::$redisIns->$func(...$args)
			//return call_user_func_array([static::$redisIns, $func], $args)
		}
		//请求完成后执行一些操作
		//...
		return $res
	}
}
```
## 在Python中实现call
Python中当访问不存在的属性或者方法时，会调用类的`__getattr__`方法，基于这个方法我们可以实现类似`__call`的功能。（注意：Python中的类有一个`__call__`方法，不过该方法是在类被作为方法执行是调用的。）
如下代码，当访问hello这个不存在的属性时，会调用`MagicCall`这个类的`__getattr__`方法。注意：`__getattr__`方法**只负责返回属性，而不负责执行方法**，执行方法是在`ins.hello('xx', 'pp')`这一行执行`__getattr__`的返回结果，因此，`__getattr__`中需要返回一个闭包。
这里没有直接把`callMethod`方法直接返回，而是再套了一层，是为了**保留调用的method_name**。

```python
class MagicCall():
    def __getattr__(self, method_name):
        def func(*args, **kwargs):
            self._callMethod(method_name, *args, **kwargs)
        return func

    def _callMethod(self, method_name, *args, **kwargs):
        pass

class MyClass(MagicCall):
    def _callMethod(self, method_name, *args, **kwargs):
        print(method_name, args, kwargs)

if __name__ == '__main__' :
    ins = MyClass()
    ins.hello('xx', 'pp')
```

当然，这种实现有个问题：**如果没有调用不存在的方法，而是访问不存在的属性，会获取到一个闭包**.. 这个问题要解决的话会比较复杂(可能得根据当前的执行栈进行判断)，这篇文章里不讨论这个。
## 实现callStatic
上一章的实现，当调用类实例不存在的方法时，会正确重定向到`callMethod`。
但是，如果是静态方法调用，比如调用一个不存在的静态方法`MyClass::hello("hehe")`时，就没法调用到`callMethod`方法了。因为，本质上来说，当调用`MyClass::hello()`时，访问的并不是`MyClass`的实例(上文代码中的ins才是MyClass的一个实例)。

那`MyClass`这个类是谁的实例？答案是它的**元类**，即type。当我们用class定义一个类时，实际上是用type创建了一个'类'并实例化了它。
关于元类的分析网上文章有很多，这里不在赘述，我们可以通过类的`__metaclass__`属性修改类的元类。

改进后的代码如下，修改了`MagicCall`的元类为`MagicMeta`，当调用`MyClass.hello()`时，因为`MyClass`这个类是`MagicMeta`的一个实例，所以会执行`MagicMeta`的`__getattr__`方法。

在`MagicMeta->__getattr__`中，self指向的是`MyClass`（self指向类的实例，MyClass是MagicMeta的实例），所以可以根据`self`调用到`MyClass`的`callStaticMethod`

```python

class MagicMeta(type):
    """
    调用MagicCall不存在的方法时，会调用其元类的getattr方法
    """
    def __getattr__(self, method_name):
        def func(*args, **kwargs):
            getattr(self, '_callStaticMethod')(method_name, *args, **kwargs)
        return func

class MagicCall():
    "重定义类的元类"
    __metaclass__ = MagicMeta

    def __getattr__(self, method_name):
        def func(*args, **kwargs):
            self._callMethod(method_name, *args, **kwargs)
        return func

    @staticmethod
    def _callStaticMethod(method_name, *args, **kwargs):
        """
        调用不存在的静态方法时执行的函数
        :param method_name: 方法名
        :param args:   方法参数
        :param kwargs: 方法参数
        :return:
        """
        pass

    def _callMethod(self, method_name, *args, **kwargs):
        """
        调用不存在的方法时执行的函数
        :param method_name: 方法名
        :param args:   方法参数
        :param kwargs: 方法参数
        :return:
        """
        pass

class MyClass(MagicCall):
    @staticmethod
    def _callStaticMethod(method_name, *args, **kwargs):
        print(method_name, args, kwargs)

    def _callMethod(self, method_name, *args, **kwargs):
        print(method_name, args, kwargs)

if __name__ == '__main__' :
    ins = MyClass()
    ins.hello('xx', 'pp')
    MyClass.hello('xxx', 'ppp', xname='linxianlong')
    MethodProxy.hello()
```


