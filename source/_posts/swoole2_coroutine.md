---
title: swool2.0 协程
date: 2017-02-09 20:42:22
tags: [PHP]
category: PHP
---

## 2.0更新说明
[官方链接](https://wiki.swoole.com/wiki/page/672.html)
>Swoole 2.0正式版发布了。2.0版本最大的更新是增加了对协程（Coroutine）的支持。正式版已同时支持PHP5和PHP7。基于Swoole2.0协程PHP开发者可以已同步的方式编写代码，底层自动进行协程调度，转变为异步IO。解决了传统异步编程嵌套回调的问题。
>与Node.js（ES6+）、Python等语言使用yield/generator、async/await的实现方式相比，Swoole协程无需修改代码添加额外的关键词。
>与Go语言的goroutine相比，Swoole协程是内置式的，应用层代码无需添加go关键词启动协程，只需要使用封装好的协程客户端即可，使用更简单。另外Swoole协程的IO组件在底层内置了超时机制，不需要使用复杂的select/chan/timer实现客户端超时。
>目前Swoole底层内置的协程客户端组件包括：udpclient、tcpclient、httpclient、redisclient、mysqlclient，基本涵盖了开发者常用的几种通信协议。协程组件只能在服务器的onConnect、onRequest、onReceive、onMessage 回调函数中使用。

## 关于协程
个人理解，协程对于线程，有点类似线程对于进程的关系。
当一个线程在进行io操作时，传统的处理方法会导致该线程阻塞，无法接收其他请求。
在使用协程的情况下，在进行io时，swoole会保存当前的上下文信息（保存在swoole开辟的栈内），然后将协程挂起等待返回。
io完成后，触发epool事件，协程切换，恢复上下文，继续执行php代码。
所以，协程相当于内部讲程序进行了异步化，单线程处理并发的能力大大提高。

个人系统的知识了解比较少，协程底层以及epool还不是很理解，后续再补充..
参考文章：
[PHP并发IO编程之路](http://rango.swoole.com/archives/508)

## 使用和性能测试
测试代码：

```php
$server = new Swoole\Http\Server('127.0.0.1', 9501);
$dbConn = mysqli_connect("127.0.0.1" , "root", "secret", "homestead");
$server->set([
     'worker_num' => 1,
]);
$server->on('Request', function ($request, $response) use ($dbConn){
    $style = $request->get["style"];
    if ($style == "async") {
        $swoole_mysql = new Swoole\Coroutine\MySQL();
        $swoole_mysql->connect([
            'host' => '127.0.0.1',
            'user' => 'root',
            'password' => 'secret',
            'database' => 'homestead'
        ]);
        $res = $swoole_mysql->query('select sleep(1)');
        echo !empty($res) ? "true" : "false";
        $response->end("mysql async is ok \n");
    }
    if ($style == "sync") {
        $res = mysqli_query($dbConn, "select sleep(1)");
        $ret = mysqli_fetch_assoc($res);
        echo !empty($ret) ? "true" : "false";
        $response->end("mysql sync is ok \n");
    }
});
```

### 测试结果：
#### 传统方式：
./bin/siege -c 10 -r 20 "http://127.0.0.1:9501/?style=sync"

```sh
Transactions:                 200 hits
Availability:              100.00 %
Elapsed time:              200.33 secs
Data transferred:            0.00 MB
Response time:                9.26 secs
Transaction rate:            1.00 trans/sec
Throughput:                0.00 MB/sec
Concurrency:                9.25
Successful transactions:         200
Failed transactions:               0
Longest transaction:           10.03
Shortest transaction:            1.00
```
#### 协程方式：
./bin/siege -c 10 -r 20 "http://127.0.0.1:9501/?style=async"

```sh
Transactions:                 200 hits
Availability:              100.00 %
Elapsed time:               34.18 secs
Data transferred:            0.00 MB
Response time:                1.01 secs
Transaction rate:            5.85 trans/sec
Throughput:                0.00 MB/sec
Concurrency:                5.89
Successful transactions:         200
Failed transactions:               0
Longest transaction:            1.09
Shortest transaction:            1.00
```
**可以看到，单个线程处理耗时为1s的程序，传统处理方式的只能达到1qps，而协程方式能达到6qps**

## 注意事项
**协程组件只能在服务器的onConnect、onRequest、onReceive、onMessage 回调函数中使用。**

## 其他客户端的使用
[2.0.5使用示例](https://wiki.swoole.com/wiki/page/672.html)


