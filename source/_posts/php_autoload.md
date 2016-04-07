---
title: php中的autoload
date: 2016-04-07 18:07:03
tags: [PHP]
category: PHP
---
# php中的autoload
PHP中当一个类依赖于另一个类时，原始的方法是require该文件，当项目or框架中存在大量的类文件时，需要require每个需要的PHP文件，而autoload可以让PHP在执行时引入需要的类文件。
## __autoload
php5提供了`__autoload`方法来实现类的自动加载,当遇到未加载的类时，会调用`__autoload`方法进行require。
使用`__autoload`的一个例子：

```php
//该方法必须在全局(global)内能进行访问
function __autoload($class) {
    $file = dirname(__FILE__) . DIRECTORY_SEPARATOR . "/$class.class.php";
    if (file_exists($file)) {
        require_once($file);
    }
}

$hehe = new Heher();
$hehe->sendHehe();

/*
 *Heher.class.php
 */
class Heher {
    public function sendHehe() {
        echo "I'm from auto load , Hehe!";
    }
}
```
## spl_autoload_register
PHP 在 5.1.2 以後提供了SPL，SPL 全名為 Standard PHP Library，为一系列基础的工具类以及方法，`spl_autoload_register`提供了比`__autoload`更为强大的功能。
`spl_autoload_register`的使用：

```php
function _hehe_autoload($class) {
    $file = dirname(__FILE__) . DIRECTORY_SEPARATOR . "/$class.class.php";
    if (file_exists($file)) {
        require_once($file);
    }
}

spl_autoload_register('_hehe_autoload');

//5.3之后可以使用匿名函数
spl_autoload_register(function ($class) {
    $file = dirname(__FILE__) . DIRECTORY_SEPARATOR . "/$class.class.php";
    if (file_exists($file)) {
        require_once($file);
    }
});


$hehe = new Heher();
$hehe->sendHehe();
```
可以注册类的静态方法进行load：

```php
class LoadClass {
    public static function _hehe_autoload($class) {
        $file = dirname(__FILE__) . DIRECTORY_SEPARATOR . "/$class.class.php";
        if (file_exists($file)) {
            require_once($file);
        }
    }
}

spl_autoload_register(array('LoadClass', '_hehe_autoload'));

```
## 区别
1.`spl_autoload_register`能注册多个自动加载函数，按注册的顺序依次执行，而`__autoload`方法只允许定义一次。

>If there must be multiple autoload functions, spl_autoload_register() allows for this. It effectively creates a queue of autoload functions, and runs through each of them in the order they are defined. By contrast, __autoload() may only be defined once.

2.spl注册的函数能进行其他操作，用`spl_autoload_unregister`可以注销函数，用`spl_autoload_functions`可以查看已经注册的函数，在这些机器上可以对自动加载做更灵活的操作。



