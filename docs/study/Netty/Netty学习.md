# Netty学习
> 结合B站网课以及自己上网的查询资料学习(正好在做一个前后端分离的Android聊天室，后端要用到UDP，本来尝试了一下DatagramSocket，中间有些计网知识搅浑了。在前辈推荐下，打算先学习一下Netty，再继续尝试聊天室开发)。
> B站视频链接：[【2019年最新Netty深入浅出全集】B站最透彻的的Netty入门到进阶教程](https://www.bilibili.com/video/av77025537)
> 之前就稍微看了一点+网上查阅NIO等资料，发现其实思想和我一开始自己捣鼓出来的用DatagramSocket写的差不多，不过居然是流行的NIO+网络通讯框架，那必然值得学习。
>
> 当然，也推荐阅读[官方的用户手册](https://netty.io/wiki/user-guide-for-4.x.html)来学习,下面的笔记记录可能涉及到视频、网文、官方文档

## 1. 了解BIO,NIO,AIO的区别

> 推荐几篇文章(也可以先从一些成熟的java图书了解)：
> [JAVA的io流和nio有什么区别？](https://www.zhihu.com/question/337609338/answer/778674573) 
>
> [真正理解NIO](https://www.jianshu.com/p/362b365e1bcc)
>
> [IO,NIO,AIO理解](https://www.cnblogs.com/study-makes-me-happy/p/9603290.html)

BIO就是传统的同步阻塞IO（需要实时得知IO的执行、需要等到IO执行结束才能向下继续执行）；

NIO是同步非阻塞的IO(可以理解是能够使1个线程处理IO的能力提升，及原本BIO一线程一IO<因为被阻塞了，多个IO也不起作用>，而NIO一线程多IO<通过selector轮询可用的channel>)。常见的NIO使用就是网络通讯编程了，比如，比较简易的聊天室(UDP)

AIO是异步非阻塞的IO（NIO2），在IO处理中多了更多的信息获取，能够把IO请求交给操作系统处理后再通知程序进行处理。也就是说，AIO多了异步的回调信息，而不再是单纯的自己调用IO时实时监听变化，变化的异常、结果转由异步回调的消息来通知。

NIO提供了Channel、Buffer、Charset、Selector

## 2. 传统TCP/IP流处理问题

> 这里推荐几篇文章
> [TCP粘包，拆包及解决方法](https://www.cnblogs.com/panchanggui/p/9518735.html)

官方文档有提到一个问题，那就是传统的TCP/IP接收数据包时，需要服务端和客户端自行整理数据包的顺序，即如果同时接收到多个数据包，服务端or客户端需要能够正确地读取这几个包。之所以有读取的问题，是因为数据传输是字节流，所以读取时需要自己判断那部分字节信息属于哪个数据包（如果数据包传输时以块的形式，就不会有这种问题）

在网课里面，这个被称作粘包、拆包问题。

## 3. EventLoopGroup

> [NioEventLoopGroup源码分析与线程设定](https://www.cnblogs.com/linlf03/p/11373834.html)
>
> [netty实战之百万级流量NioEventLoopGroup线程数配置](https://blog.csdn.net/linsongbin1/article/details/77698479)
>
> [[netty源码分析]--EventLoopGroup与EventLoop 分析netty的线程模型](https://blog.csdn.net/u010853261/article/details/62043709)
>
> [Netty源码阅读——handler()和childHandler()有什么区别](Netty源码阅读——handler()和childHandler()有什么区别)
>
> [netty5 客户端 NioEventLoopGroup的问题。为什么全都是单线程处理？](https://www.oschina.net/question/3659437_2271832?p=1)
>
> [Netty 系列之 Netty 线程模型](https://www.infoq.cn/article/netty-threading-model/)

​	看了一些网文，想起自己之前不知道在哪里看到的Netty代码，对boss线程设置为CPU内核数，而work线程设置为CPU内核数*10，客户端子channel的线程数为handler数量。感觉有点迷惑行为。除非把耗时操作全交给EventLoop了，不然想不到为什么需要那么多线程。

## 4. 心跳

> [netty 实现长连接，心跳机制，以及重连](https://blog.csdn.net/weixin_41558061/article/details/80582996)

## 5. TCP还是UDP

> [游戏服务器：到底使用UDP还是TCP](https://www.cnblogs.com/crazytomato/p/7987332.html)

### 1. UDP打洞，内网NAT映射问题

> [UDP通讯，外网向内网发消息，内网无法收到 [问题点数：20分，结帖人pylmcy150]](https://bbs.csdn.net/topics/300093018)
>
> [udp外网无法返回数据到内网 [问题点数：40分，结帖人qianshangding]](https://bbs.csdn.net/topics/360137409)
>
> [请教UDP 打洞是个什么过程，有成功过的请进。 [问题点数：100分，结帖人myth_2002]](https://bbs.csdn.net/topics/80304372)
>
> [求助技术贴：对称型NAT 怎么穿透](https://zhidao.baidu.com/question/1694861333912260508.html)
>
> [udp外网无法返回数据到内网 [问题点数：40分，结帖人qianshangding]](https://bbs.csdn.net/topics/360137409)
>
> [NAT的四种类型](https://blog.csdn.net/mycloudpeak/article/details/53550405)
>
> [对称NAT穿透的一种新方法](https://blog.csdn.net/bd_zengxinxin/article/details/80991689)
>
> [nat 类型及打洞原理](https://www.cnblogs.com/gsgs/p/9263679.html)
>
> [UDP通信在NAT中session保持时间测试](https://www.jianshu.com/p/34ef5099904c)

​	对称NAT进行打洞/穿透比较随缘（**家用路由据说一般都不是对称NAT，即是锥形的**）。

+  全锥型(Full Cone) 

+ 受限锥型(Restricted Cone) **限制IP**

+ 端口受限锥型(Port Restricted Cone) **限制IP和PORT**

+ 对称型(Symmetric) **限制IP和PORT，并且不同外网和内部使用NAT的不同PORT**

  ==对称和锥型最大的区别就是是否NAT使用同一个端口了。==

上面的文章对打洞和NAT都解释很清楚了。下面讲讲我自己遇到的问题和解决。

​	以前有次找了个UDP代码，那时候对UDP、TCP那些还不太熟，所以先找网上代码跑跑看能看效果再学习。这时候遇到一个问题，UDPclient和UDPserver都在我本地网络下是能正常交互的，这里代码逻辑就简单的Echo，client发一句话给server，server重新发回来。后面我把UDPserver放到服务器上，结果**服务器Linux只用tcpdump抓包能抓到UDP包，程序却收不到UDP包**。当时对linux网卡等了解不多，所以没能解决。

​	最近又回想起来这个事情，于是重跑一遍代码，tcpdump抓包，发现能抓到UCP包，但是程序还是收不到数据。这次我对Linux网卡有过学习了。所以查看ifconfig发现抓包的UDP目的地址是eth0的ip，而服务跑在docker网桥上，所以没能收到UDP包。于是<u>我docker-compose指定了network_mode: host，使得docker容器使用和宿主机相同的网络环境</u>，进而能获取到eth0的包，问题就解决了。

​	期间顺便给服务端代码加了个HashMap记录client，然后把client的数据发送给所有和服务器交互过的client。我这里4个client每3秒发送UDP包给Server，然后server不管收到谁的UDP包，记录或更新其在HashMap中的IP和port记录，然后把该UDP包的内容发送给HashMap中记录的所有client。（主要最近要做一个群聊的功能，以前是想用UDP，但是那时候出现获取不到数据的问题，后面就用TCP了。最近重新试试，因为对计网\Linux了解多了不少，问题倒是解决挺快的了）



