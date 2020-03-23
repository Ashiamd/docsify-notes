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

​	看了一些网文，想起自己之前不知道在哪里看到的Netty代码，对boss线程设置为CPU内核数，而work线程设置为CPU内核数*10，客户端子channel的线程数为handler数量。感觉有点迷惑行为。除非把耗时操作全交给EventLoop了，不然想不到为什么需要那么多线程。

## 4. 心跳

> [netty 实现长连接，心跳机制，以及重连](https://blog.csdn.net/weixin_41558061/article/details/80582996)