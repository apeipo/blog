---
title: swool2.0 协程
date: 2017-02-09 20:42:22
tags: [PHP, Swoole]
category: PHP
---

## 2.0更新说明
[官方链接](https://wiki.swoole.com/wiki/page/672.html)
>Swoole 2.0正式版发布了。2.0版本最大的更新是增加了对协程（Coroutine）的支持。正式版已同时支持PHP5和PHP7。基于Swoole2.0协程PHP开发者可以已同步的方式编写代码，底层自动进行协程调度，转变为异步IO。**解决了传统异步编程嵌套回调的问题。**
>与Node.js（ES6+）、Python等语言使用yield/generator、async/await的实现方式相比，Swoole协程无需修改代码添加额外的关键词。
>与Go语言的goroutine相比，Swoole协程是内置式的，应用层代码无需添加go关键词启动协程，只需要使用封装好的协程客户端即可，使用更简单。另外Swoole协程的IO组件在底层内置了超时机制，不需要使用复杂的select/chan/timer实现客户端超时。
>目前Swoole底层内置的协程客户端组件包括：udpclient、tcpclient、httpclient、redisclient、mysqlclient，基本涵盖了开发者常用的几种通信协议。协程组件只能在服务器的onConnect、onRequest、onReceive、onMessage 回调函数中使用。

## 关于协程
### 异步IO-Reactor
#### 同步IO
当一个线程在进行io操作时，传统的处理方法线程会阻塞等待IO完成，等待过程无法接收其他请求，程序的并发能力存在很大的瓶颈。
可以通过多线程or多进程的方式提升程序的并发处理能力，但是没有本质上的提升。原因在于系统能创建的进程数量是有限的，并且阻塞IO这种等待自身就是对cpu资源的浪费。
#### 异步IO
相对于同步IO，异步IO过程中，线程在IO开始时会注册系统回调，IO过程中线程可以进行其他工作，IO完成后执行回调进行后续操作。
可以看到，异步IO一个本质的区别是：**同一个线程，能同时处理多个IO请求**。这种情况下程序的并发能力有很大的提升。
例如某个IO请求的耗时为1s，在单线程同步IO方式下，理论上最大只能达到1qps，异步IO方式的qps远高于这个数。
#### 阻塞和非阻塞
>一个IO操作其实分成了两个步骤：**发起IO请求和实际的IO操作**
>**阻塞IO和非阻塞IO的区别在于第一步**：发起IO请求是否会被阻塞，如果阻塞直到完成那么就是传统的阻塞IO;如果不阻塞，那么就是非阻塞IO
>**同步IO和异步IO的区别就在于第二个步骤是否阻塞**，如果实际的IO读写阻塞请求进程，那么就是同步IO，因此阻塞IO、非阻塞IO、IO复用、信号驱动IO都是同步IO;如果不阻塞，而是操作系统帮你做完IO操作再将结果返回给你，那么就是异步IO

#### Reactor模式
参考文章： [高性能IO](http://www.cnblogs.com/fanzhidongyzby/p/4098546.html)
异步非阻塞的实现是应用了Reactor模式。用户线程向Reactor注册事件处理函数，Reactor负责管理EventHandler（注册、调用、删除）、轮询系统事件（通过epool或者select系统调用）。线程注册了EventHandler之后可以继续执行其他操作。
注意：
1. **Reactor只是一个事件管理器，实际的IO操作是在EventHandler(用户实现)中完成的**。
2. **异步非阻塞IO，实际上并不是真正的异步，因为程序接收的事件是：可读 or 可写，实际的读写操作都是需要应用自己完成（从socket读取）**。
3. **完成的异步IO实现是Proactor模式，程序接收的事件是： 读取完成 or 写入完成了，读写操作由系统完成，应用从缓冲区读取**。

Reactor结构图：![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-02-10-14867149196922.jpg)

Reactor时序图：![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-02-10-14867149572894.jpg)

#### Reactor和观察者模式的区别
观察者模式是一种一对多的**发布-订阅**的关系，更多应用于消息的分发。
Reactor更强调的是**注册-回调**，应用于高效率IO。

### 协程
>协程从底层技术角度看实际上还是异步IO Reactor模型，应用层自行实现了任务调度，借助Reactor切换各个当前执行的用户态线程，但用户代码中完全感知不到Reactor的存在。
个人理解，协程是应用层对异步IO的一种实现，封装了异步IO的回调、上下文切换等过程。

### swoole的协程处理
在swoole中使用自带协程的client，在进行io此操作时（如connect，query），swoole会保存当前的上下文信息保存在swoole开辟的栈内，然后将协程挂起等待返回。
io完成后，触发epool事件，协程切换，恢复上下文，继续执行php代码。
从处理流程可以看出，协程的实现是语言层面的实现，在操作系统中并没有这个概念，而进程和线程是操作系统级别的，其调度和上下文切换是由操作系统对外提供的api实现的。

### swoole2.0协程使用和性能测试
#### 开启协程：

```sh
phpize
./configure --with-php-config={path-to-php-config}  --enable-coroutine
make && make install
```
注意：**目前只支持在onRequet, onReceive, onConnect事件回调函数中使用协程。**
#### 测试代码：

```php
$server = new Swoole\Http\Server('127.0.0.1', 9501);
$dbConn = mysqli_connect("127.0.0.1" , "root", "secret", "homestead");
$swooleDb = new swoole_mysql;
$server->set([
     'worker_num' => 1,
]);
/*触发on request事件时，SWOOLE会开辟一个协程栈，对协程栈进行初始化*/
$server->on('Request', function ($request, $response) use ($dbConn, $swooleDb){
    $style = $request->get["style"];
    //传统方式，同步阻塞IO
    if ($style == "sync") {
        $res = mysqli_query($dbConn, "select sleep(1)");
        $ret = mysqli_fetch_assoc($res);
        echo !empty($ret) ? "true" : "false";
        $response->end("mysql sync is ok \n");
    }
    //异步IO，swoole 1.8.5
    if ($style == "async") {
		$swooleDb->connect(
			["host" => "127.0.0.1", 
			 "user" => "root", 
			 "password" => "secret", 
			 "database" => "test"], 
			 function ($swooleDb, $resp) {
			    $sql = 'select sleep(1)';
			    $swooleDb->query($sql, function(swoole_mysql $swooleDb, $res) {
			    	$response->end("mysql async is ok \n");
			    });
			});
    }
    //协程方式, swoole 2.0
    if ($style == "coroutine") {
        $swoole_mysql = new Swoole\Coroutine\MySQL();
      /**
        client在调用connect函数后，SWOOLE会将PHP上下文信息保存到当前栈内
        然后将协程挂起，待确认连接成功后，触发epoll事件，然后协程切换
        恢复PHP上下文信息，返回结果，继续执行PHP代码
      */
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
./bin/siege -c 10 -r 20 "http://127.0.0.1:9501/?style=coroutine"

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
swoole2之前提供的异步客户端和协程本质上没有区别，都是异步IO Reactor模型，所以没有进行性能测试。
相比于老的异步客户端的各种回调，从代码上可以看出来，协程的方式大大降低了代码的复杂度。
注意：**协程组件只能在服务器的onConnect、onRequest、onReceive、onMessage 回调函数中使用。因为只有这些函数中swoole才会创建协程。**

### 其他客户端的使用
[2.0.5使用示例](https://wiki.swoole.com/wiki/page/672.html)

## 参考文章
关于reactor，个人知识有限，了解的比较片面，详细的可以阅读下面这几篇大神文章。参考文章：
[PHP并发IO编程之路](http://rango.swoole.com/archives/508)
[高性能IO模型](http://www.cnblogs.com/fanzhidongyzby/p/4098546.html)
[Reactor模型](http://www.cnblogs.com/ivaneye/p/5731432.html)
[epool](http://blog.csdn.net/mango_song/article/details/42643971)


