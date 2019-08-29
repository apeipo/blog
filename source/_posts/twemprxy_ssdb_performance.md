---
title: Twemproxy + SSDB 性能测试
tags: [SSDB,Twemproxy]
category: [后端]
date: 2017-04-6 12:18:14
---
## 背景
1. SSDB是一个高性能的NoSql数据库，底层使用LevelDB，兼容Redis协议。主要用于解决Redis无法存储海量数据的问题。[官网链接](http://ssdb.io/zh_cn/)
2. Twemproxy 是twitter开源的高性能、轻量级redis集群代理

### 问题
如下图，线上ssdb集群的耗时随着时间的增长耗时不断增长，最高时达到6.5s
![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-04-06-14914576091140.jpg)

### 场景
1. 线上部署采用的是twemproxy + ssdb的方式。2台proxy下挂4台ssdb 
2. 服务请求方为4台机器，每台机器50个线程，每个线程单次读取1000条数据（hget请求）
3. twemproxy和ssdb均采用官方推荐配置，ssdb开启缓存为20G
4. **客户端连接proxy使用的是jedis客户端，使用pipline方式进行批量请求**。

可以推算出：
最高QPS：        `4 * 50 * 1000 = 20w ` 
SSDB单机最高QPS： 5w 
PROXY单机最高QPS：10w
上图中6.5s耗时为单个线程请求三份数据的总耗时（串行），每份数据为1000条，因此可以推算出单**次请求1000条数据的耗时在2s上下**，无法满足要求。

### 问题推测
 1. 除了ssdb耗时高以外， 服务请求方机器内存、CPU、IO、网络均正常，并且重启客户端耗时无变化，可以排除是客户端的问题。
 2. SSDB机器和Proxy机器的内存、CPU、IO、网络均正常，可以排除是系统层面的问题
 3. 客户端请求读取的都是半小时内刚写入的数据，可以排除是读取请求都落在磁盘上导致的性能下降

因此，问题原因基本可以判定是出现在Proxy或者SSDB自身上，测试也主要针对这两个部分分别进行。
## Proxy 性能测试
### 环境
1. 测试环境使用单台proxy下挂2台ssdb（相当于线上一半的配置）
2. SSDB数据量分别为107G和117G。
3. 客户端连接使用jedis的pipeline批量请求

### Proxy测试场景和结果
![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-04-06-14914492594626.jpg)
### 分析
1. 提高server_connection对性能的影响明显，在线下环境下，设置为15**提升约1倍的性能**（对比场景11和场景9）
2. 修改启动的-m选项，**提升性能约10%**（对比场景13和12）

关于server_connection和mbuf的官方说明：[twemproxy文档](https://github.com/twitter/twemproxy/blob/master/notes/recommendation.md)
#### server_connection
1. Twemproxy与SSDB客户端之间的保持的连接数。**默认为1**，即Proxy和SSDB实例之间只使用一个连接。
2. Twemproxy对该连接的使用是基于epool事件循环，因此该连接能支持很高的并发。

提高server_connection对性能影响很大，猜测是当前的并发已经达到了单个连接的上限。不过继续当达到一定数量后提高server_connection对性能的不大（对比场景14和13），甚至有可能下降，这个原因没有细查，猜测可能是连接数大了后Proxy的维护和切换成本会提高。

#### mbuf
Twemproxy存储request数据使用的最小单位，默认为16M。当并发高时，官方建议调小该值，否则很可能造成很高的内存占用。

## 效果
在线上修改了server_connection和mbuf参数后，耗时下降明显，接近ssdb的极限性能。

![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-04-06-14914517106043.jpg)

## 其他测试
### Redis替换SSDB能否提升性能?
可以看出，redis对ssdb的性能提升不明显。原因是测试中读取的都是热数据，ssdb的读取都是内存操作，和redis读取的差别不大。
![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-04-06-14914518753097.jpg)
### 使用get/set能否提升性能?
将存储数据进行转换，测试脚本中的hget操作修改为get操作，get的性能相比hget提升约1/3。
但是改为get后，实际的请求量会增长（如果hash结构中有4个key，则改为get后请求增长4倍），因此修改为get请求对整体并没有帮助。
![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-04-06-14914521918853.jpg)

### Proxy的数量多少合适？
从结果数据来看：

1. proxy和ssdb数量达到2：1后，再增加proxy对性能已经基本没有提升（此时实际上已经到了ssdb的性能上限）。
2. proxy和ssdb数量2：1相对于1：1的性能提升约为15%

所以，2：1的话能使性能最大化，但是比较浪费资源，1：1是合理的方案。
实际线上部署时，一台机器可以部署2~4个proxy，线下在单台机器部署3个proxy时，在100并发每次2000数据的情况下，影响CPU-IDLE约5，网卡占用约15%。
![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-04-06-14914520378195.jpg)












