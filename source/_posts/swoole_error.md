---
title: swoole使用中踩的坑
tags: [PHP, Swoole]
category: [后端]
date: 2017-04-10 21:05:14
---
从去年开始使用swoole开发内部的一些小的服务，中间遇到一些坑，在此记录
## 多进程不能共用连接
[官方文档](https://wiki.swoole.com/wiki/page/325.html)
其实这个是所有多进程、多线程程序都需要注意的问题，当进程or线程共享资源的时候，一定要考虑资源冲突，否则会出现各种诡异的问题（死锁、数据返回异常、连接被关闭等等等）。
如下代码，在swoole中，在server启动时创建了一个redis连接，在onRequest中使用。
代码看上去没什么问题，但是实际使用时，如果压力很大，就会出现多进程抢占连接导致的问题。原因是因为创建的redis连接实际上是一个全局对象，每个work进程都在使用同一个连接。

```php
class Server 
{
	public $redis;
	public $server;
	
  //初始化函数
  public function init(){
		//initServer
		$server = new swoole_http_server( $this->productConfig["server"]["host"], 
		                                  $this->productConfig["server"]["port"]);
		$server->on('request', array($this, 'onRequest'));
		$server->on('workerstart', array($this, 'onWorkerStart'));
		$this->server = $server;
		//initRedis
		$this->redis = new Redis();
		$this->redis->connect($this->productConfig["redis"]["host"], 
		$this->productConfig["redis"]["port"]);  
		$server->start();
	}
	
	//请求处理函数
	public function onRequest($request, $response) {
		//do some thing with redis
		$this->redis->get("somekey");
		$response->end(json_encode(["errno" => 0]));
	}
	
	//worker进程创建函数
	public function onWorkerStart($serv, $workerId) {
	}
}

$server = new Server();
$init   = $server->init();
if ($init["errno"] !== 0) {
    die(json_encode($init));
} 
$server->run();
```

### 解决方法
1. 不在server中创建，而在onRequest中，接收到请求时创建
简单粗暴的解决方案，每次收到请求时独立创建局部的redis连接，请求结束后释放。这种方式缺点很明显，连接没有复用，影响性能。

2. 在onWorkerStart中创建连接，并按workerId索引每个worker进程的redis连接
代码如下（主体代码参考上面），在Server中增加一个redisPool，worker启动时创建连接后注册到pool中。这样能保证每次请求时，使用的都是各进程独立的redis连接。

```php
	//在Server类中增加$redisPool变量，初始化为空数组
	public $redisPool = [];
	//worker进程创建函数
	public function onWorkerStart($serv, $workerId) {
		$redis   = new \Redis();
		$tmpRes  = $redis->connect($this->productConfig["redis"]["host"], 
		                           $this->productConfig["redis"]["port"]);  
		if ($tmpRes === false) {
		    $this->logger->error("Redis Init failed workerId:$workerId");
		    return false;
		}
		$redis->setOption(Redis::OPT_READ_TIMEOUT, 
		                  $this->productConfig["redis"]["read_timeout"]);
		$this->redisPool[$workerId] = $redis;//将连接注册到pool中
	}
	//请求处理函数
	public function onRequest($request, $response) {
		//do some thing with redis
		$redis = $this->redisPool[$this->server->worker_id];
		$response->end(json_encode(["errno" => 0]));
	}
   
```

## tcp协议包完整性
[官方文档](https://wiki.swoole.com/wiki/page/50.html)
[Swoole的自定义协议功能的使用](http://blog.csdn.net/ldy3243942/article/details/40920743?utm_source=tuicool&utm_medium=referral)
在默认情况下，使用swoole-server时（TCP协议），swoole不对包的进行完整性校验，在onReceive中接收到的包可能是不完整的，也有可能是多份数据。这是由于TCP协议的原理所造成的：
>TCP是一个流式协议。客户端向服务器发送的一段数据，可能并不会被服务器一次就完整的收到;客户端向服务器发送的多段数据，可能服务器一次就收到了全部的数据

如下代码，在onReceive中接收到数据后，转给task进程进行处理，task进程处理结束后返回。

```php
//请求处理（接收到客户端发送的数据）
public function onReceive(swoole_server $serv, int $fd, int $from_id, string $data) {
		//
		$this->logger->debug("ReceiveMSG: " . $data, ["fd" => $fd]);
		$params = [
			"fd" 	=> $fd,
			"data" 	=> $data
		];
		$conInfo = $serv->connection_info($fd);
		$this->logger->debug("conninfo fd:$fd ", ["info" => $conInfo]);
		$this->server->task($params);
}
// 实际的task处理
public function onTask($serv, $taskId, $srcWorkerId, $params) {
	$fd 	= $params["fd"];
	$sendRes = $serv->send($fd, "hehe\t0\t[]\n");
	$this->logger->debug("conninfo-InTask fd:$fd ", ["sendRes" => $sendRes]);
	$serv->close($fd);
}
//与客户端的连接被关闭
public function onClose(swoole_server $server, int $fd, int $reactorId) {
		$this->logger->debug("fd:$fd is Close");
}
```
执行时（发送的数据比较大），有可能出现这样的日志。从日志里可以看到

1. 同一个server\_fd(53)接收到了两份数据，两份数据来自同一个client\_fd(22)
2. 第一个task向客户端发送数据成功了，但是第二个发送失败（sendRes：false）
![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-04-10-14918263781334.jpg)
实际打印出数据（在onReceive中），会发现客户端发送的数据被拆成了两份，因此触发了两次onReceive。
第二个task中sendRes失败，是因为在处理第一份数据时，task中已经把客户端连接给关闭了。

### 解决办法
#### open_eof_check
使用swoole提供的open_eof_check，保证数据包的完整性。
>此选项将检测客户端连接发来的数据，当数据包结尾是指定的字符串时才会投递给Worker进程。否则会一直拼接数据包，直到超过缓存区或者超时才会中止

EOF即为数据的结束标记，具体由客户端使用的发送方式而定，比如Memcache协议以"\r\n"结尾，Java中BuffWriter.newLine()发送的数据在有可能是`"\n"`结尾，也有可能是`"\r\n"`。
注意：**swoole的EOF检测不会从数据中间查找eof字符串，所以Worker进程可能会同时收到多个数据包，需要在应用层代码中自行explode("\n", $data) 来拆分**，1.7.15版本增加了open_eof_split，支持从数据中查找EOF，并切分数据。

```php
'open_eof_check' => true, //打开EOF检测
'package_eof' 	 => "\n", //设置EOF
```

#### 手动拼接请求数据
默认情况下，同一个客户端fd会被分配到同一个worker中处理，所以数据可以拼接起来，当发现结尾是EOF字符时才进行处理。
例如可以在全局数据中保存一个数组buff，接收到数据后进行拼接和判断。

```php
$buff[$fd] .= $data;
if (substr($buff[$fd], -1) == "\n") {//\n也可以是其他的EOF字符
	//数据完整，执行具体的逻辑
} else {
	//数据不完整，返回等待下次接收
}
```


