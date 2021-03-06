---
title: 从配置Chares抓包理解HTTPS
tags: [Web]
category: [Web]
date: 2017-03-02 22:04:19
---
这两天看了一些https的文章，想起来之前捣鼓chares抓https包时一知半解，借这个问题总结一下Https。
## 问题
在Chares里通过Proxy->SSL ProxySettings可以打开https代理，没有经过任何配置的话，抓包中看到的https链接是这样的。
![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-03-02-14884377678270.jpg)
要解决这个问题很简单，按照官方的说法，只需要安装Chares的证书并在系统设置中信任该证书即可。
这中间经历了什么过程，为什么安装证书后就可以看到https包内容，这就需要我们对https的原理有一定的了解。
## 加密算法
在了解https之前，先简单说下加密算法。
### 对称加密
顾名思义，对称加密算法加密和解密用相同的密钥(Key)，密钥越大加密性越强，越不容易被破解。
常见的对称加密算法有DES、3DES、RC5等。
对称加密算法的优点是加密效率高，密钥足够大时安全性很强。缺点也很明显，发送者和接受者使用相同的密钥，发送者在发送消息时需要将密钥要发送给接受者，这样相当于没加密，密钥安全性无法保障。
### 非对称加密
对比对称加密，非对称加密有两个密钥，公钥（Public Key）和私钥（Private Key）。公钥加密的数据只能由私钥解密，私钥加密的数据只能由公钥解密。私钥由发送者保存，公钥对外发布。
使用最广泛的非对称加密算法是RSA算法。
非对称加密算法的有点是在私钥不泄露情况下，能保证发送的消息不被篡改，收到的消息无法被破解，缺点是算法复杂度高，性能不如对称加密算法。
可以看出，**在实际使用的安全性上**，非对称加密算法要比对称加密算法高。这里强调的是使用安全性，因为就算法安全性来说，只要密钥够长，破解难度都是很高的。
## HTTPS
了解了加密算法，开始说说Https，思路主要是根据自己对Https的理解过程。
### 裸奔
对于http请求，在没有加密的时代，所有的请求参数都是对外暴露的。
对于攻击者来说，只要想办法拦截到用户的请求，很容易就能获取到交互内容和进行伪造。
![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-03-02-14884423572656.jpg)
### 开始加密
要保证请求的内容即使被拦截也无法破解，最简单的方法就是对请求内容进行加密，但是用哪种加密算法呢？
从使用安全性上考虑，当然是非对称加密算法。但是对于网络请求来说，性能就是生命，性能是第一要考虑的，因此性能更好的对称加密算法是首选。
从对加密算法的介绍中了解到，对称加密最大的问题在于，对于http请求来说，加密解密的密钥需要在Client和Server之间传输。如何保证密钥不被泄露？

```sh	
##################10s分割线##################
```
容易想到的是一点：Server一般都是接收多个Client的请求，因此为了保护密钥，这一点是肯定的：**不同的会话需要使用不同的密钥**。因此，在Client和Server通讯前必须有一个阶段，**商量用什么密钥**，就是密钥生成密钥的过程。
这样问题来了，这个商量的过程怎么保证不被泄露呢？ 再对这个商量过程进行对称加密？那你的这个加密过程又怎么保证呢？...无线循环了。
这时候，非对称加密就派上用场了。
### 加密我的加密
我们可以用非对称加密对密钥的生成过程进行加密。
![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-03-02-14884564685368.jpg)
如上图是一个的加密传输过程，使用非对称加密进行密钥生成，使用对称加密进行信息加密和传输。
1. 客户端请求服务端得到公钥。
2. 客户端生成MasterKey并用公钥加密后发给服务端。
3. 服务端接收到加密的信息并用私钥解密得到MasterKey。
4. 服务端使用MasterKey加密Message发给客户端
5. 客户端收到消息后用MasterKey解密出消息
至此，消息传输完成，使用非对称加密保证了MasterKey只有客户端和服务端知晓，从而保证了对称加密交互过程的信息安全。一个简化版本的Https交互完成。

```sh	
##################10s分割线，想想该过程有什么问题？##################
```
#### 认证机构(CA)
上图的方案，解决了**密钥的保护问题**，但是会有另外一个问题，就是**身份认证**。
在第一步中，客户端请求服务端得到公钥，**怎么确定这个公钥就是服务端的呢？**。如果这一步的请求被第三方拦截了，返回给客户端第三方自己的公钥，整个交互过程又全暴露了。
具体的过程如下，容我偷懒盗个图。[原图链接](https://showme.codes/2017-02-20/understand-https/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)
![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-03-02-14884574921787.png)
可见，在非对称加密过程中，**需要有公钥的身份验证，确保公钥不被篡改**。
这时候，认证机构(Certification Authority)出现了，由认证机构负责对各Server的公钥进行管理。Server向认证机构申请了资格后，认证机构用自己的私钥给Server的公钥加密，生成一个"证明"。下次客户端请求时，服务端将这个"证明"返回给客户端。客户端用认证机构的公钥可以从证明中获取服务端的公钥。
那认证机构的公钥哪里来呢？总不能每次请求都访问一下认证机构吧。实际上，每台电脑上都保存着认证机构的信息，认证机构的公钥可以从客户端本地查询。
#### 数字证书
上文所说的这个"证明"，就是Https中的数字证书。数字证书由服务端向第三方机构申请，用于Https握手阶段的公钥身份验证。证书由第三方机构用私钥对申请信息(公钥)进行加密生成。
**注意，这里说的证书还只是第一版，并非最终的Https证书**
#### 数字签名
有了数字证书之后，我们来看下现在获取服务端公钥的过程。
![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-03-02-14884592962322.jpg)
可以看到，在引入CA后，解决了公钥被伪造的问题。因为如果攻击者劫持请求并伪造了证书，证书的内容是没法被CA的公钥解密的。

```sh	
##################10s分割线，想想该过程有什么问题？##################
```
我们聪明的攻击者又来了！，上面这个过程难道还有漏洞？
是的，刚刚说到，上面的证书没法被伪造，但是。。。如果攻击者补伪造证书，自己向第三方机构申请证书，并发给客户端呢？！！
大家在想一下这个过程，客户端收到攻击者自己的证书（该证书是合法申请的），用第三方机构公钥解密得到了**客户端认为的服务端公钥**，然后用这个实际上是攻击者的公钥进行密钥生成....信息又全暴露了。
所以，我们还需要**对证书进行认证**，数字签名的引入，就是为了解决该问题。
带数字签名的证书生成过程：
![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-03-02-14884612467318.jpg)
如上图，CA在生成证书时，对证书的内容进行单向加密(这里以Md5为例，可以是其他算法如SHA)生成**摘要**，对摘要用个人私钥进行加密生成了数字签名。
客户端获取到证书后，用CA的公钥对签名进行解密得到摘要，再用Md5对证书内容生成本地的摘要，只要对比两个摘要是否相同就能确认该证书是不是由机构签发的。这里的证书就是真正意义上的Https证书。
至此，Https的密钥生成和消息传递过程的安全问题解决了，**@1不过大家可以再想想上面这个过程还有没有问题，什么情况下会被破解？**。
## 总结
以上的Https过程只说了最主要的部分，实际上Https的握手阶段还要更复杂一些。容我再偷张图：
[原图链接](https://blog.cloudflare.com/keyless-ssl-the-nitty-gritty-technical-details/) ![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-03-02-14884617468013.jpg)
1. 客户端发出协议版本号、一个客户端生成的随机数（**RandomA**），以及客户端支持的加密方法。
2. 服务端确认双方使用的加密方法，并返回数字证书和一个服务端生成的随机数（**RandomB**）。
3. 客户端确认数字证书有效（有效期、签名），然后生成一个新的随机数（**RandomC**），并使用数字证书中的公钥，加密这个随机数，发给服务端。
4. 服务端使用私钥，解密得到发来的RandomC。
5. 客户端和服务端根据约定的加密方法，使用RandomA、B、C生成对话密钥（**Session Key**），后续的对话由该Key进行加密解密。
过程中的三个随机数，可以保证之前所说的，对不同的客户端，使用不同的密钥。
## 回归问题
回到之前的Chares配置的问题。现在想想，如果你是Chares的开发者，用户已经设置了Chares作为代理，要想查看Https的包内容，应该怎么办？

```sh	
##################10s分割线##################
```
根据上文理解的Https的过程，可以看到，第三方如果想查看和伪造https的内容，必须满足两点：
1. 自己当CA，给自己颁发个证书。
2. 让客户端电脑信任自己(CA)。
这两个步骤实际上就是Chares配置https代理过程所做的事，配置完成后，可以查看https的包内容：
![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-03-02-14884622405300.jpg)

从这里也可以回答上面@1的问题，**当系统层面被被攻破后，https就不是安全的**，这也是越狱的iPhone，Root的安卓、以前用安装盘安装的XP不安全的其中一个原因。

## 参考资料
[理解HTTPS为什么安全前，先看看这些东西](http://mp.weixin.qq.com/s/7ImZolr7m3tUuyOgMJeFYg)
[也许，这样理解HTTPS更容易](https://showme.codes/2017-02-20/understand-https/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)
[图解SSL](http://www.ruanyifeng.com/blog/2014/09/illustration-ssl.html)





