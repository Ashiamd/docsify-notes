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
>
> [Netty 源码分析之 三 我就是大名鼎鼎的 EventLoop(一)](https://segmentfault.com/a/1190000007403873)
>
> [深入理解 NioEventLoopGroup初始化](https://www.cnblogs.com/ZhuChangwu/p/11192219.html)

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
> [Linux NAT 类型详解](https://blog.csdn.net/sdc20102010/article/details/80112580)
>
> [路由器上的NAT工作在哪一层](https://zhidao.baidu.com/question/95619581.html)
>
> [UDP通信在NAT中session保持时间测试](https://www.jianshu.com/p/34ef5099904c)
>
> [NAT与NAPT德区别。](https://zhidao.baidu.com/question/1429862685193512779.html)

#### UDP打洞和为何打洞、为什么TCP"不适合"P2P，以及NAT介绍(下面讲的是NAT的NAPT)

下面介绍流程如下

1.   A、NAT中的4种NAPT
2.   B、NAT的概念
   1. 为什么会出现NAT
   2. NAT是啥，能干啥
3.   C、TCP是否需要NAT，以及谈谈为什么UDP需要打洞(什么是NAT穿透)
   1. TCP当然也是需要走上面的NAT流程的
   2. TCP是否能实现P2P
   3. TCP有链接和UDP无连接
   4. 为什么需要UDP打洞(什么是UDP打洞/NAT穿透)
4.   D、如何实现UDP打洞（NAT穿透），实现流程。



A、先简单介绍NAT中的4种NAPT类型

​	对称NAT进行打洞/穿透比较随缘（**家用路由据说一般都不是对称NAT，即是锥形的**）。

+  全锥型(Full Cone) 

+ 受限锥型(Restricted Cone) **限制IP**

+ 端口受限锥型(Port Restricted Cone) **限制IP和PORT**

+ 对称型(Symmetric) **限制IP和PORT，并且不同外网和内部使用NAT的不同PORT**

  ==对称和锥型最大的区别就是是否NAT使用同一个端口了。==

B、谈论打洞问题之前，必须先了解NAT概念。

1. 为什么会出现NAT。

   学过计网的都知道ACBDE类IP地址吧，IPv4地址总共就32位，地球上上网的那么多，每个人都分一个的话肯定不够(IPv6就不说了，毕竟还没有做到全普及)。**为了解决这个ipv4地址不够所有人用的问题，就诞生了NAT**。

2. NAT是啥，能干啥。

   ​	**NAT是一种网络地址转换方法**，用一个port标识内网和外网的联系。

   ​	前面提到IPv4的地址是有限的，**生活中拥有IPv4公网地址的往往是网络提供商**(联通、电信、移动之类的)。我们为了能"上网"（连接互联网的意思），那么肯定需要一个IPv4公网地址，这样别人才能在网络上找到我，我也才能找到别人(指拥有IPv4公网地址的人)。网络提供商肯定不能直接让你用这个IPv4标识你自己的电脑，为什么呢？因为要是你这么干，别人也会这么干。假如2个人甲和乙都能共用这个ipv4(1.1.1.1)。那么甲(1.1.1.1:80)和另一个网友老王(ipv4:2.2.2.2:80)聊天，结果老王回复你的时候，乙(ipv4:1.1.1.1:80)收到了这个消息，那不是乱套了。

   ​	网络提供商(拥有ipv4：1.1.1.1的机器)开辟了一个WAN广域网，甲和乙都去找它办宽带了，甲宽带分配到ip(162.100.100.1)，乙分配到ip(168.240.162.5)，这两个ip是联系ipv4:1.1.1.1用的，**不能直接和互联网上的ipv4交流**。

   ​	有了各自的宽带后，甲就能够和老王聊天了。但是甲的ip162.100.100.1不是ipv4，不处于互联网，不能和老王直接交流。这时候NAT就出来帮忙了。甲(162.100.100.1:80)--NAT（网络提供商ipv4:1.1.1.1:**6751**）-->老王(ipv4:2.2.2.2:80)。

   ​	NAT使用一个port标识 甲的ip:port(162.100.100.1:80)和另一个外网ipv4拥有者老王(ipv4:2.2.2.2:80)的联系。

   ​	老王收到甲的消息时，看到的ip和port只能是公网ipv4的，也就是网络提供商ipv4:1.1.1.1:**6751**。老王给甲回消息，也是传给ipv4:1.1.1.1:**6751**。这里老王根本不知道甲处在的WAN中的ip和port到底是多少。

   ​	网络提供商的路由器查找NAT表，得知6751端口标识的是自己创建其中一个WAN中的甲和老王的联系。于是把ipv4:1.1.1.1:**6751**收到的数据发送给甲（162.100.100.1：80）。

   ​	这样一来甲和老王就成功联系在一起了。甲----甲处在的网络的NAT---> 老王；老王----甲处在的网络的NAT---->甲。

   ​	这里需要说明几个重要的知识点，为了下面提到UDP打洞做铺垫。

   + 一，那就是**NAT映射必须由该WAN内的用户建立**。换句话说就是，在甲主动联系老王之前，老王是不可能知道怎么联系甲的，就算甲把自己的ip和端口(162.100.100.1：80)告诉了老王也没有用，因为甲的这个ip只有在甲所处的WAN网络内才是有效的。只有甲主动联系老王后，NAT表上才会多一个端口port记录这条"通道"，老王也才能通过这个"通道"访问到甲。

   + 二，NAT4种模式对于NAT的映射处理不同，基本是出于安全的考量。

   + 三，NAT标记内网和外网交流的端口port不是永久分配的，会有回收时间的。

     ​	这个很好理解，不然你要是程序都不运行了，还占用网络提供商路由器的端口6751，你一个人还好，要是一堆人都这么干，明明不上网了却还占用NAT表的port，久而久之NAT表爆满了，每个port都被用掉了，那么大家都别想用到这个ipv4(1.1.1.1)了，都不用上网了。



**C、在大致了解了NAT后，聊聊TCP是否需要NAT，以及谈谈为什么UDP需要打洞(什么是NAT穿透)**

1. TCP当然也是需要走上面的NAT流程的！

   ​	只要甲想要和老王聊天，那就必须用ipv4，而甲只能通过NAT来获取ipv4和老王聊天。这里甲和老王聊天如果使用到的是TCP，那么往往是 甲--甲to老王的聊天信息--> 聊天TCP服务器 --甲to老王的聊天信息-->老王。

   ​	聊天软件作为中介一般的存在，和甲、老王分别连接TCP连接，然后之后分别使用channel通道和甲、老王交流数据(<u>这里channel实际上不是同一个，但是逻辑上就像是同一个</u>，下面直接用一个标识)交流。

   ​	甲--channel+A--聊天服务器--channel+B--老王(他其实也是在另一个WAN里面，用的也是网络提供商的ipv4)

   ​	建立连接后，为了保持这个channel的有效性，甲会每隔一段时间发心跳包给聊天服务器，老王同理。这样子甲和老王各自所处的WAN的NAT表就会一直保持甲、老王各自和聊天服务器的联系，使用port标记。

   ​	对于TCP开发来说，如果长时间没有发送心跳包，tcp连接就会断开(其实就是NAT上面的port失效了，联系外网失败了，据说NAT标记port时，对于TCP的保留会比UDP久，我自己没实际测试过具体差异)。

2. **UDP为什么需要打洞。讲之前先讲讲TCP是否能实现P2P**。

   ​	UDP通信的开发不像TCP有连接通信，TCP这种有连接的模式，往往是A-B-C三者之间。比如上面的甲--聊天服务器--老王。聊天服务器为了能让甲和老王聊天，必须一直记录甲和老王各自的NAT拥有的ipv4和记录port。UDP主要还可以用在P2P聊天模式（直接 甲---老王，不过首先还是得至少经过一次甲--聊天服务器--老王）。

   ​	这里先讨论下TCP能不能实现P2P聊天。**P2P也就是甲和老王交流完全不通过第三方，甲信息直接第一手交给老王，老王同理**。不是所有情况TCP都可以实现P2P（这和TCP需要建立连接有关）。

   ​	**==只有在甲和老王都是直接拥有ipv4的情况下，可以通过直接TCP实现P2P==**。

   + 甲和老王都是直接拥有ipv4的用户。

     ​	这种情况没啥好说的，甲和老王能够直接通过ipv4建立TCP连接了，不需要通过任何转换和中介，也就是P2P了。

   + 甲或者老王仅有其中一方拥有ipv4（假设老王有ipv4）

     ​	这种情况下，甲要和老王交流必须通过NAT。老王向甲发话，实际上是向甲所在的NAT(1.1.1.1:6751)发话。由于前面提到NAT必须由内网用户主动发起，外网才能发现这个WAN的NAT的ip和port，所以老王是不可能直接向甲发话的，必须甲先主动和老王发话，老王才能找到甲。

     ​	**这种情况TCP如果要老王和甲建立连接且P2P交流，那么必须老王能够知道甲所在的WAN对应的NAT的ip和port**。但是老王是不可能知道甲所在的NAT的port的，这个必须甲先发出请求后，老王才能获取到port，而且老王并无法确定这个port一定是对应甲的。因为甲所在的WAN中不止甲能够使用NAT和老王交流，完全可以冒出来一个乙也故意发相同内容的数据包给老王。老王收到2个数据包，ip都是(1.1.1.1)，但是一个port是(6571甲)，另一个是(6682乙)，由于数据包的内容完全一模一样，请问老王要记住哪个port来建立TCP连接？正是由于这种原因，TCP就算这种情况能实现P2P，也不安全。

   + 甲和老王都没有ipv4，都是通过各自的NAT来使用网络提供商的ipv4进行互联网交流

     ​	这种情况甲和老王甚至完全**没办法直接交流**(指不通过第三方公网ipv4)。原因如下，甲不知道老王的网络提供商ipv4是多少，老王也不知道甲的网络提供商ipv4是多少。**就算甲和老王各自上网查了自己的网络提供商的ipv4分别是(1.1.1.1)和(2.2.2.2)，甲和老王还是不能互相交流**。

     ​	原因很简单，前面提过了NAT必须由一方先发起。这里甲和老王谁发起都一样。假设甲知道老王的WAN的ipv4是2.2.2.2，于是他通过162.100.100.1:80打算通过自己的WAN的1.1.1.1向2.2.2.2发包，然后他意识到根本不知道向2.2.2.2的哪个端口发包。**因为NAT的port必须内网的那方先向外网发送后才会分配**。

     ​	所以这种情况直接P2P是百分百不可能的。必须有像聊天服务器这类拥有ipv4的存在作为中介存储甲和老王各自的NAT的ipv4和port，甲和老王才能够交流。

   上面对TCP是否能够建立P2P进行了一定的探讨。其实最主要的问题就是NAT影响了甲和老王之间是否能够直接联系。

   ​	下面回到UDP打洞问题之前，**再讲讲TCP有链接和UDP无连接**。

   ​	UDP不像TCP需要建立连接(3次握手和4次挥手)。UDP只要知道发送的地址ip:port和自己的ip:port就ok了。**TCP建立连接的好处就是建立连接后不需要关注网络状态变化**。因为只要网络状态变化了，TCP就可以将其视为连接断开，那么下次A和B要交流时就重新记录ip和port。所以建立TCP连接的代价就是每次变换网络都需要重新建立连接(这个操作耗时，也会比UDP效率低)，但是建立连接后的所有操作，都不需要考虑ip和port这些网络层的东西了。

   ​	UDP相对TCP灵活，因为UDP不需要建立连接。换言之，**UDP每次发送都要得知对方的ip和port**。如果甲和老王都直接拥有ipv4，这时候使用UDP交流可以省去TCP的3次建立连接的操作。但是现实中，哪有那么理想的情况，往往是甲和老王的ip、port会一直变动。

   ​	现在假设我们聊天服务器用UDP实现，不用TCP。那么必然会面临一个问题：

   甲和老王的ip和port时常变动，**甲必须通过某种形式了解到老王最新的ip和port，才能和其进行p2p交流**。

   前面多次强调如果用到NAT，那么不通过第三方是不可能实现P2P的。这里也是一样的。我们的解决手段通常如下：

   （1）甲---udp---> 拥有ipv4的服务器(记录甲所在的WAN的NAT的ip和port，这个port标记甲和服务器的联系，假设为1.1.1.1:6751)

   （2）老王---udp--->  拥有ipv4的服务器(记录甲所在的WAN的NAT的ip和port，这个port标记老王和服务器的联系，假设为2.2.2.2:7891)

   （3）甲想直接和老王进行p2p交流。于是服务器把2.2.2.2:7891发给甲，和甲说你发给这个地址就可以和老王进行交流了（对老王也进行相同操作），"以后别找我做中介了"**(实际是不可能真的再也不找他做中介的，后面会说原因)**。

   （4）甲直接把数据包发到2.2.2.2:7891，结果被老王所在的WAN的NAT直接拒绝了。这个过程由于甲访问的目的地址不再是服务器，而是老王所在的WAN，所以甲的WAN的NAT对于这次操作的记录port**不一定**是原本的6751。（具体port会是多少，首先和NAT的类型有关，其次和服务提供商的NAT分配port的策略有关）

   （5）老王不知道甲的数据包被拒绝这件事，在这之后的较短时间内也发了数据给甲的1.1.1.6751。(太长的话甲的WAN的NAT，1.1.1.1:6751就会去掉这个甲试图访问老王的这条记录，这里假设甲和老王之间的这个联系用的还是甲WAN的NAT的6751端口，同时假设老王访问甲的这次操作用的还是2.2.2.2:7891)

   （6）"奇迹发生了"，甲获取到了老王的聊天信息，并且之后再发送一次给老王上次的信息，老王也收到了。

   这里不知道大家还记不记得**前面提到的NAT，必须由内网一方主动联系外网，NAT才会标记一个port标识这个联系**。（这里出于==两方都是全锥型(Full Cone) NAT的情况==）

   + 正是由于如此，甲第一次访问老王，由于老王没有主动联系过甲，该数据包被老王的NAT直接丢弃；

   + 但是虽然甲的数据包被老王的NAT丢弃了，但是甲这个操作使得甲的NAT用port标记了这次行为；

   + 老王在甲的NAT的port过期之前发送数据给甲，由于甲有事先尝试联系过老王，甲的NAT-port有记录，所以老王的数据包没有被丢失，被甲WAN的NAT转发给了老王；

   + 之后同理，由于老王访问过甲，甲再次访问老王也就能访问得通了。

   上面这种甲能够访问老王的操作，这是**UDP打洞(NAT穿透)**。

3. **D、如何实现UDP打洞（NAT穿透）**

   ​	大致了解TCP和UDP后。**接下来就是UDP打洞来实现P2P了。但==UDP实现的P2P，在用到NAT的情况下，其实还是不能完全的P2P的，中间还是得借助第三方==**。

   ​	通过前面大篇幅的介绍，得知甲如果想和老王通过UDP通信，需要时刻知道对方的最新ip和port。这需要有第三方记录两者的ip和port的更新（聊天服务器记录）。

   ​	下面**假设甲和老王都是端口受限锥形NAT的前提**下进行UDP打洞。因为如果两者都是对称NAT或者一个对称，一个端口受限锥形的话，虽然可以通过巧妙的算法实现P2P，但是消耗还是挺大的，复杂度也高。（<u>直接说这2种情况就不能实现打洞也不是不行，因为实在复杂，依赖的条件还涉及网络提供商的NAT-port分配策略，条件算苛刻的</u>）

   **打洞流程**大致如下：

   （1）甲---UDP--->聊天服务器(这个过程很频繁，假设50s一次，这个UDP包可以什么数据都没有，指记录甲WAN的NAT对应的ip和port)

   （2）老王---UDP--->聊天服务器（同理50s一次，时间设置短一方面是防止NAT的port过期，一方面是能尽可能更上用户的ip、port变换速度。当然不是越短越好，没必要1s一次UDP，那用户流量不炸裂）

   （3）当甲**单方面**和老王想要P2P私聊的时候，甲和服务器反应，服务器把甲想和老王私聊的事情告诉老王，并把服务器关于甲的地址记录(1.1.1.1:6751)发送给老王，同时也把老王的告诉甲，老王发送空数据的UDP给甲(当然被甲WAN的NAT丢弃了)。

   ​	老王---UDP--->甲。由于老王NAT是**端口受限锥形NAT**，老王访问甲的这个操作会使得老王WAN为其分配新的(port2.2.2.2:7777)，之前的(port2.2.2.2:7891)是对应老王和服务器之间的联系。

   （4）甲虽然没收到老王的包，但是因为甲**单方面**想和老王P2P私聊，所以是肯定会向老王(2.2.2.7891，服务器之前通知甲的是这个)发送聊天信息。同理甲WAN的NAT会新建port标识这一个操作(1.1.1.1.6666)。

   ​	**聪明的你是不是发现问题了，那就是这样下去甲和乙都会陷入死循环，互相发包，阁制WAN的NAT一直重新分配port标识甲->老王或者老王->甲**。（这里我没实际操作过，但从理论上推测，只要甲和老王WAN的NAT各自会舍弃对方的包且重新分配port，那么只要没有运气好某一方访问对面的时候port正好撞上了，那么就死循环）。

   ​	==虽然网上说**端口受限锥形NAT**和**端口受限锥形NAT**两方也是可以UDP打洞的，但是我暂时从上面的推测，感觉也是行不通的==。理由也很简单，因为**NAT对内网用户来说是透明**的，甲本身不知道自己对应的NAT的ip和port是多少，没法直接把变化后的甲WAN的NAT-ip-port告诉老王，同理也不可能先告诉服务器再转告给老王。**（NAT可以静态配置，但是NAPT我不清楚有没有，如果也可以自己指定NAT的ip和port，那么就不会死循环）**

   ​	当甲或者老王其中一方不是端口受限锥形NAT（限制IP和port），而是更为宽松的受限锥型NAT（只限制IP）时，再考虑上面打洞的（4）时，同理发现打洞没法进行。假设甲的NAT只限制IP，由于老王发送的数据包，甲的NAT是第一次看见这个IP和port，所以丢弃，进而甲永远没法知道和老王通信的ip和port。如果老王是只限制IP的NAT的那一方，那同理，甲的NAT还是舍弃了老王的包。和前面的双方都是**端口受限锥形NAT**一样，**由于自身没法获取自己的NAT，所以没法和对方交流**。

   ​	接下来考虑甲是**全锥型NAT**(不限IP和port，甲的该应用162.100.100.1:80和1.1.1.1交互永远使用的是port7891)。这种情况下，打洞的（4）就可以进行了。

   （4）甲能收到老王的包，因为甲该应用对外使用的NAT-port一直都是7891。甲获取到老王WAN的NAT新分配给老王的2.2.2.2:7777，并且直接通过该ip和port向老王发送消息。由于老王事先主动像甲发送过数据，老王WAN的NAT记录了老王->甲的port为7777，所以老王能收到甲的数据包。这时候就能够P2P联系了。当然，如果其中一方ip或者port改变了，需要重新走一遍打洞的(1)~(4)流程。



-------



​	==UDP打洞一般也就P2P的应用场景需要涉及。如果所有交互都是P1->Server->P2的形式，那么用不到UDP打洞。==

​	~~这里讲讲NAT，如果不想看上面的推荐文章的话，这里简述一下。~~

​	~~首先提一下TCP和UDP。**TCP建立连接**，所以客户端和服务端交互很丝滑，两方都通过channel就ok了；**UDP没有建立连接**，所以服务端必须经常获取用户最新的IP和PORT，才能服务端主动发送消息给客户端。~~

​	~~上面简单的说建立连接和没有建立连接，感觉看上去很抽象。其实这里主要是忽略了**NAT映射**这一个重要概念，才显得抽象。~~

​	~~不管是TCP还是UDP，在client和server交互的过程难免需要经过NAT映射。这里假设client没有自己的公网IP，而server有公网IP（通常我们的家用网络就是没有自己的公网IP，你查的IP是你家办的宽带对应的宽带提供商的公网IP，这个公网IP是很多户人家共用的。而服务器一般都是网上租的，大都有自己的公网IP）。~~

1. ~~首先，你客户端client**主动发送**UDP---> 服务器Server。这一个过程会经过NAT映射。~~

   ~~你家宽带IP算是网络提供商网络下的一个私网IP，这个是移动/联通/电信之类的给你的宽带分配的。正常情况下，你私网IP是不能和外网交互的（就好比你自己电脑不联网，能玩单机游戏，但是不能玩网游似的），但是网络提供商提供NAT服务，能够把你和外网某个公网IP：PORT的联系用一个port标识。~~

   ~~下面IP字段我就随便打打了，不考虑真实性。~~

   ~~你Client（192.168.1.17:8888）--试图访问外网-->  某公网IP服务器(172.162.12.10:8080)~~  

   ~~上面这个过程本来是不可行的，因为你在一个局域网环境(联通/移动/电信家办宽带)，用的是私网IP，但是网络提供商有公网IP，它通过NAT帮你搞定访问问题（这里讲的不是很好，NAT是用来解决IP资源不足的问题，但是我相信懂的人应该懂我意思）~~

   ~~你Client（192.168.1.17:8888）--**网络提供商NAT(假设ip：178.47.16.53 ，port 2045)-**->  某公网IP服务器(172.162.12.10:8080)~~  

   ~~上面多了一个NAT过程，**用port标识 （192.168.1.17:8888）和(172.162.12.10:8080) 的联系**。这个port端口2045是随便写的，具体它给你多少和不同的网络提供商的分配策略有关，可能随机也可能按照某种顺序。~~

   ~~这里强调NAT的这个**port只能唯一标识**你的当前这个IP:PORT和外网哪个IP:PORT的联系，这种NAT可能是`端口受限锥型(Port Restricted Cone)`也可能是`对称型(Symmetric)`。这个不细说，要知道详细的可以看看上面推荐的关于NAT的解释。~~

2. ~~有了这个NAT映射后，服务器感知到有个client发送数据给自己。注意，这里服务器抓包获取到的IP和PORT不是你Client的，而是网络提供商公网IP和NAT的端口port。~~

   ~~NAT--->Server（接受到来自**178.47.16.53:2045**的UDP包，也就是网络提供商IP:NAT端口）。~~

   ~~服务器并不知道你Client在这个宽带网络下到底是什么私网IP和端口（即192.168.1.17:8888，这个它是不知道的）。~~

3. ~~Server--NAT(你家网络提供商NAT)->你Client。~~

   ~~这个过程，服务器是把数据发给了**178.47.16.53:2045,网络提供商IP:NAT端口**，那么既然不是发给你的，你怎么会收到UDP数据包？？答案就是，NAT通过查找自己的NAT映射表得知port 2045标识的是**（192.168.1.17:8888）和(172.162.12.10:8080) 的联系**，那么通过**178.47.16.53:2045**的包就会发送到其内网的**（192.168.1.17:8888）也就是你家的私网IP和应用端口**。~~

~~那么UDP为什么会需要打洞呢？~~



最上面推荐的文章对打洞和NAT都解释很清楚了。下面讲讲我自己过去遇到的问题和解决。

​	以前有次找了个UDP代码，那时候对UDP、TCP那些还不太熟，所以先找网上代码跑跑看能看效果再学习。这时候遇到一个问题，UDPclient和UDPserver都在我本地网络下是能正常交互的，这里代码逻辑就简单的Echo，client发一句话给server，server重新发回来。后面我把UDPserver放到服务器上，结果**服务器Linux只用tcpdump抓包能抓到UDP包，程序却收不到UDP包**。当时对linux网卡等了解不多，所以没能解决。

​	最近又回想起来这个事情，于是重跑一遍代码，tcpdump抓包，发现能抓到UCP包，但是程序还是收不到数据。这次我对Linux网卡有过学习了。所以查看ifconfig发现抓包的UDP目的地址是eth0的ip，而服务跑在docker网桥上，所以没能收到UDP包。于是<u>我docker-compose指定了network_mode: host，使得docker容器使用和宿主机相同的网络环境</u>，进而能获取到eth0的包，问题就解决了。

​	期间顺便给服务端代码加了个HashMap记录client，然后把client的数据发送给所有和服务器交互过的client。我这里4个client每3秒发送UDP包给Server，然后server不管收到谁的UDP包，记录或更新其在HashMap中的IP和port记录，然后把该UDP包的内容发送给HashMap中记录的所有client。（主要最近要做一个群聊的功能，以前是想用UDP，但是那时候出现获取不到数据的问题，后面就用TCP了。最近重新试试，因为对计网\Linux了解多了不少，问题倒是解决挺快的了）

## 6. Netty中的零拷贝（Zero-Copy）机制

> [理解Netty中的零拷贝（Zero-Copy）机制](https://my.oschina.net/plucury/blog/192577?p=1)
>
> [netty深入理解系列-Netty零拷贝的实现原理](https://www.cnblogs.com/200911/articles/10432551.html)
>
> [对于 Netty ByteBuf 的零拷贝(Zero Copy) 的理解](https://segmentfault.com/a/1190000007560884)
>
> [netty四种BUFFER的内存测试](https://blog.csdn.net/phil_code/article/details/51259405)



### 6.1 Netty零拷贝--一起看源码呗
&nbsp;&nbsp;&nbsp;&nbsp;先推荐一下贴出的文章。本篇文章也是参考下述文章后，再对部分类进行源码查看的。
> [理解Netty中的零拷贝（Zero-Copy）机制](https://my.oschina.net/plucury/blog/192577?p=1)
> 
> [netty深入理解系列-Netty零拷贝的实现原理](https://www.cnblogs.com/200911/articles/10432551.html)
> 
> [对于 Netty ByteBuf 的零拷贝(Zero Copy) 的理解](https://segmentfault.com/a/1190000007560884)
> 
> [netty四种BUFFER的内存测试](https://blog.csdn.net/phil_code/article/details/51259405)
> 
> [Netty学习之ByteBuf](https://www.jianshu.com/p/b95a82ab7462)
>
> [netty与内存分配(2)-PooledByteBufAllocator](https://www.jianshu.com/p/e2a37d48b8ab)

​	**Netty的零拷贝，基本就是对ByteBuf的内存空间的重复利用。而原理的话，没什么高深的，就是Netty帮你管理了ByteBuf的指针。**
​	所以可以做到不同的ByteBuf使用相同区域的内存（只复制指针、不new新的内部数组空间）。也可以把逻辑上的N个ByteBuf“合并”成一个新的大ByteBuf，但是内部没有new出N个ByteBuf内存之和的空间来存储整个大ByteBuf，而是使用原本的内存空间，只不过Netty封装该N个ByteBuf到List中，帮你管理了指针。反之，一个ByteBuf拆成多个ByteBuf，而共用相同的内存空间也是Netty能做到的。
（*想必，对于学过C语言的各位来说，应该不难理解。就是Netty这种实现，听上去简单，但是要我们自己手动实现和管理的话，还是比较复杂的。*）

***

​	下面，就和大家一起看看Netty中使用到`零拷贝`思想的类。
（*下面的类，主要根据文章开头提及的文章去展开，上面的那些文章我觉得都挺好的。*）

#### 0. PooledByteBufAllocator（池化ByteBuf分配器）和UnpooledByteBufAllocator

​	这两个的源代码就不走了。感兴趣的可以自己看看[netty与内存分配(2)-PooledByteBufAllocator](https://www.jianshu.com/p/e2a37d48b8ab)这一片文章关于PooledByteBufAllocator的解读。

​	池化ByteBuf的技术，和线程池的目的都是为了减少资源开销，个人认为和`零拷贝`的思想也是相符的。

> 下面PooledByteBufAllocator和UnpooledByteBufAllocator介绍摘至《Netty-Redis-ZooKeeper高并发实战》

​	**Netty通过ByteBufAllocator分配器来创建缓冲区和分配内存空间**。Netty提供了ByteBufAllocator的两种实现：`PooledByteBufAllocator`和`UnpooledByteAllocator`。

​	`PooledByteBufAllocator`（池化ByteBuf分配器）将ByteBuf实例放入池中，提高了性能，将内存碎片减少到最小**；这个池化分配器采用了jemalloc高效内存分配的策略，该策略被好几种现代操作系统所使用**。

​	<u>UnpooledByteBufAllocator是普通的未池化ByteBuf分配器，它没有把ByteBuf放入池中，每次被调用时，返回一个新的ByteBuf实例：通过Java的垃圾回收机制回收。</u>

​	使用Netty官方的DiscardServer代码进行DEBUG，也可以看到，Server服务启动的bind绑定端口时，会进行一些初始化的操作，其中就有根据系统来使用不同的`ByteBufAllocator`。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200502184337441.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FzaGlhbWQ=,size_16,color_FFFFFF,t_70)

​	`ByteBufUtil`，如果当前系统是Android，就使用`UnPooledByteBufAllocator`,否则使用`PooledByteBufAllocator`。一般服务器的服务端代码都是部署在Linux的，也就是使用`PooledByteBufAllocator`。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200502183907778.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FzaGlhbWQ=,size_16,color_FFFFFF,t_70)

#### 1. 通过 CompositeByteBuf 实现零拷贝

​	**该类可以将多个ByteBuf“组合”成一个新的ByteBuf，这里不是直接复制N个ByteBuf的内容到一个新的ByteBuf对象并返回，而是用一个数组将多个ByteBuf组成一个返回。实际使用通常通过Unpooled.wrappedBuffer(.....)，一般不会直接使用到**`CompositeByteBuf` 。（即新返回的ByteBuf还是使用的原本的内存空间）。

​	下面贴出部分源代码，可以根据序号（1）（2）....（4）顺序查看。

```java
/**
 * A virtual buffer which shows multiple buffers as a single merged buffer.  It is recommended to use
 * {@link ByteBufAllocator#compositeBuffer()} or {@link Unpooled#wrappedBuffer(ByteBuf...)} instead of calling the
 * constructor explicitly.
 */
public class CompositeByteBuf extends AbstractReferenceCountedByteBuf implements Iterable<ByteBuf> {
    
    // (1)(1)(1)(1)(1)(1)(1)(1)(1)(1)(1)(1)(1)(1)(1)(1)(1)
    // 存储组装的ByteBuf，比如N个ByteBuf组成一个，内部就是把N个ByteBuf存这个数组里了
    // Component含有ByteBuf类型的成员对象。
    private Component[] components; // resized when needed
    
   
    // (2)(2)(2)(2)(2)(2)(2)(2)(2)(2)(2)(2)(2)(2)(2)(2)(2)
    // Component内部类，里面含有ByteBuf成员对象
    // 可以看到下面完全没有 新建ByteBuf的操作，用的都是ByteBuf对象的引用。
    private static final class Component {
        final ByteBuf srcBuf; // the originally added buffer
        final ByteBuf buf; // srcBuf unwrapped zero or more times
        int srcAdjustment; // index of the start of this CompositeByteBuf relative to srcBuf
        int adjustment; // index of the start of this CompositeByteBuf relative to buf
        int offset; // offset of this component within this CompositeByteBuf
        int endOffset; // end offset of this component within this CompositeByteBuf
       
        private ByteBuf slice; // cached slice, may be null

        Component(ByteBuf srcBuf, int srcOffset, ByteBuf buf, int bufOffset,
                int offset, int len, ByteBuf slice) {
            this.srcBuf = srcBuf;
            this.srcAdjustment = srcOffset - offset;
            this.buf = buf;
            this.adjustment = bufOffset - offset;
            this.offset = offset;
            this.endOffset = offset + len;
            this.slice = slice;
        }

        int srcIdx(int index) {
            return index + srcAdjustment;
        }

        int idx(int index) {
            return index + adjustment;
        }

        int length() {
            return endOffset - offset;
        }

        void reposition(int newOffset) {
            int move = newOffset - offset;
            endOffset += move;
            srcAdjustment -= move;
            adjustment -= move;
            offset = newOffset;
        }

        // copy then release
        void transferTo(ByteBuf dst) {
            dst.writeBytes(buf, idx(offset), length());
            free();
        }

        ByteBuf slice() {
            ByteBuf s = slice;
            if (s == null) {
                slice = s = srcBuf.slice(srcIdx(offset), length());
            }
            return s;
        }

        ByteBuf duplicate() {
            return srcBuf.duplicate();
        }

        ByteBuffer internalNioBuffer(int index, int length) {
            // Some buffers override this so we must use srcBuf
            return srcBuf.internalNioBuffer(srcIdx(index), length);
        }

        void free() {
            slice = null;
            // Release the original buffer since it may have a different
            // refcount to the unwrapped buf (e.g. if PooledSlicedByteBuf)
            srcBuf.release();
        }
    }
    
    //...
    
    // (3)(3)(3)(3)(3)(3)(3)(3)(3)(3)(3)(3)(3)(3)(3)(3)(3)
    // 组合ByteBuf的操作，最后都会执行这个方法。
    // N个ByteBuf被组装成1个大ByteBuf返回给用户使用，本质是通过数组存储了ByteBuf的引用
    // 学过C/C++的应该对指针这个词很熟悉了吧。这里相当于把对象的指针都存储起来了，
    // 大ByteBuf用的内存还是原本那些指针所指向的对象所占用的内存。
    // 不需要在Java中new一个和N个ByteBuf所占用的内存空间相同大小的ByteBuf对象来组装它们。
    private CompositeByteBuf addComponents0(boolean increaseWriterIndex,
                                            final int cIndex, ByteBuf[] buffers, int arrOffset) {
        final int len = buffers.length, count = len - arrOffset;
        // only set ci after we've shifted so that finally block logic is always correct
        int ci = Integer.MAX_VALUE;
        try {
            checkComponentIndex(cIndex);
            shiftComps(cIndex, count); // will increase componentCount
            int nextOffset = cIndex > 0 ? components[cIndex - 1].endOffset : 0;
            for (ci = cIndex; arrOffset < len; arrOffset++, ci++) {
                ByteBuf b = buffers[arrOffset];
                if (b == null) {
                    break;
                }
                // 这个方法里面也没有new ByteBuf,往下翻，看newComponent方法的实现就知道了
                Component c = newComponent(ensureAccessible(b), nextOffset);
                components[ci] = c;
                nextOffset = c.endOffset;
            }
            return this;
        } finally {
           // ...
        }
    }
    
    // ... 
    
    // (4)(4)(4)(4)(4)(4)(4)(4)(4)(4)(4)(4)(4)(4)(4)(4)(4)
    // (3)代码片段中addComponents0(...)使用到该方法，这里同样没有新建ByteBuf对象。
    @SuppressWarnings("deprecation")
    private Component newComponent(final ByteBuf buf, final int offset) {
        final int srcIndex = buf.readerIndex();
        final int len = buf.readableBytes();

        // unpeel any intermediate outer layers (UnreleasableByteBuf, LeakAwareByteBufs, SwappedByteBuf)
        ByteBuf unwrapped = buf;
        int unwrappedIndex = srcIndex;
        while (unwrapped instanceof WrappedByteBuf || unwrapped instanceof SwappedByteBuf) {
            unwrapped = unwrapped.unwrap();
        }

        // unwrap if already sliced
        if (unwrapped instanceof AbstractUnpooledSlicedByteBuf) {
            unwrappedIndex += ((AbstractUnpooledSlicedByteBuf) unwrapped).idx(0);
            unwrapped = unwrapped.unwrap();
        } else if (unwrapped instanceof PooledSlicedByteBuf) {
            unwrappedIndex += ((PooledSlicedByteBuf) unwrapped).adjustment;
            unwrapped = unwrapped.unwrap();
        } else if (unwrapped instanceof DuplicatedByteBuf || unwrapped instanceof PooledDuplicatedByteBuf) {
            unwrapped = unwrapped.unwrap();
        }

        // We don't need to slice later to expose the internal component if the readable range
        // is already the entire buffer
        final ByteBuf slice = buf.capacity() == len ? buf : null;

        return new Component(buf.order(ByteOrder.BIG_ENDIAN), srcIndex,
                unwrapped.order(ByteOrder.BIG_ENDIAN), unwrappedIndex, offset, len, slice);
    }
    
    // ... 
    @Override
    public ByteBuf unwrap() {
        return null;
    }
    
}
```

​	上面删掉了一些源代码。注意点就是`CompositeByteBuf`有个内部类数组成员对象`private Component[] components;`，而内部类`Component`中封装了`ByteBuf`，可以看到在`CompositeByteBuf`的`addComponents0(...)`方法中，用`components`这个Component数组存储“预组合在一起”的N个ByteBuf，并没有真正新建和N个Bytebuf占用内存相同大小的新Java对象ByteBuf。

​	而实际中，我们不需要直接对`CompositeByteBuf`操作，往往用`Unpooled.wrappedBuffer(...)`来完成ByteBuf的“组装”。

```java
public class ByteBufTest {

    private EventLoopGroup bossGroup = new NioEventLoopGroup(1);
    private EventLoopGroup workGroup = new NioEventLoopGroup();

    @Test
    public void BootStrapTest() throws InterruptedException {
        UnpooledByteBufAllocator allocator = new UnpooledByteBufAllocator(true);
        ByteBuf byteBuf1 = Unpooled.wrappedBuffer(new byte[]{1,2,3,4,5,6});
        ByteBuf byteBuf2 = Unpooled.wrappedBuffer(new byte[]{7,8,9,10,11,12});
        Unpooled.wrappedBuffer(byteBuf1,byteBuf2); // 这里打断点
    }
}

```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200502195708846.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FzaGlhbWQ=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200502195838409.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FzaGlhbWQ=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200502200005900.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FzaGlhbWQ=,size_16,color_FFFFFF,t_70)

#### 2. ByteBuf对象.slice()

​	**Netty的ByteBuf的slice()方法，将1个ByteBuf切分成N个ByteBuf，也可以重用原本的ByteBuf内存空间，而不是另外new出N个ByteBuf导致多占用1倍的原内存空间**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200502201427535.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FzaGlhbWQ=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200502201741697.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FzaGlhbWQ=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200502202433970.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FzaGlhbWQ=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200502202228961.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FzaGlhbWQ=,size_16,color_FFFFFF,t_70)

​	从上面的过程可以看出，ByteBuf的slice（）切分出来的小ByteBuf对应的数据来源和原本的大ByteBuf是一致的。也就是slice（）能复用原本的内存空间，而不需要copy一份数据，new新对象。

#### 3. 通过 FileRegion 实现零拷贝

​	这个不再展开了。额，可以去看看上面提到的文章[对于 Netty ByteBuf 的零拷贝(Zero Copy) 的理解](https://segmentfault.com/a/1190000007560884)。





## 7. Netty踩坑

> [netty踩坑初体验](https://www.jianshu.com/p/187d2d95a493)

## 8. ByteBuf

> [Netty学习之ByteBuf](https://www.jianshu.com/p/b95a82ab7462)

## 9. select、poll、epoll

> [select、poll、epoll之间的区别(搜狗面试)](https://www.cnblogs.com/aspirant/p/9166944.html)
>
> [Linux IO模式及 select、poll、epoll详解](https://segmentfault.com/a/1190000003063859)

## 10. protobuf

> [Protobuf原理分析小结](https://www.jianshu.com/p/522f13206ba1)
>
> [深入 ProtoBuf - 简介](https://www.jianshu.com/p/a24c88c0526a)
>
> [深入 ProtoBuf - 编码](https://www.jianshu.com/p/73c9ed3a4877)

