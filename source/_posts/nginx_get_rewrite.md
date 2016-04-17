---
title: nginx根据get参数rewrite
tags: [Nginx]
category: [Nginx]
date: 2016-04-17 22:18:14
---
先说一种错误的写法

```nginx
location ~ \.php$ {
  root           html;
  fastcgi_pass   127.0.0.1:9000;
  fastcgi_index  index.php;
  fastcgi_param  SCRIPT_FILENAME  /$document_root$fastcgi_script_name;
  include        fastcgi_params;
  rewrite /info.php?name=(.*) /result.php?newname=$1;
}
```
以上配置，当请求`/info.php?name=hehe`时，会发现请求没经过rewrite。**这是因为nginx的rewrite指令第一个参数匹配的是请求的URI，是不会匹配`QUERY_STRING`的**，如上配置会用`/info.php?name=(.*)`这个正则去匹配`/info.php`,当然匹配无法成功。
正确的方式：

```nginx
location ~ \.php$ {
  root           html;
  fastcgi_pass   127.0.0.1:9000;
  fastcgi_index  index.php;
  fastcgi_param  SCRIPT_FILENAME  /$document_root$fastcgi_script_name;
  include        fastcgi_params;
  if ($query_string ~* "name=(.*)") {
      set $name $1;
      rewrite /info.php /result.php?newname=$name break;
  }
}
```
这里注意一个问题：`set $name $1;` 这行不能省略，如果把上边if里的逻辑修改成这种方式，会发现rewrite后get参数李的newname为空，**这是因为此时rewrite语句中的`$1`并不是if指令里匹配的内容，而是rewrite指令第一个正则匹配的内容**，显然第一个正则(`/info.php`)没有分组，`$1`为空。

```nginx
  if ($query_string ~* "name=(.*)") {
      rewrite /info.php /result.php?newname=$1 break;
  }
```
另外，nginx中，可以通过`$arg_变量名`的方式获得get参数中的字段值，所以也可以写成如下两种形式：

```nginx
  if ($query_string ~* "name=(.*)") {
      rewrite /info.php /result.php?newname=$arg_name break;
  }
  #or
  if ($arg_name != "") {
      rewrite /info.php /result.php?newname=$arg_name break;
  } 
```
## 去除原有参数
按如上方式进行配置后，请求`/info.php?name=hehe`时，会`rewrite`到`/result.php?newname=$name`。
此时，如果在`result.php`中打印`$_GET`，`$_SERVER`的内容：

```php
array(2) { ["newname"]=> string(4) "hehe" ["name"]=> string(4) "hehe" }
#...
"REQUEST_URI": "/info.php?name=hehe",
"DOCUMENT_URI": "/result.php",
"QUERY_STRING": "newname=hehe&name=hehe",
#...
```
可以看到，nginx将原始的`query_string`追加rewrite后的`query_string`中了，如果要去除这个参数，可以将`$args`参数(保存了`query_string`)置为空：

```nginx
  if ($query_string ~* "name=(.*)") {
         set $args ""; 
      rewrite /info.php /result.php?newname=$arg_name break;
  }
```
或者可以在rewrite指令第二个参数最后加一个**?**号：

```nginx
  if ($query_string ~* "name=(.*)") {
      rewrite /info.php /result.php?newname=$arg_name? break;
  }
```
此时再打印GET参数会发现`QUERY_String`中不带着name参数了，nginx官网的描述：
![](http://7xrhmq.com1.z0.glb.clouddn.com/2016-04-15-14607165697950.jpg)



