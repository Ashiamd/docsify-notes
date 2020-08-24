

# 《Netty、Redis、ZooKeeper高并发实战》学习笔记

> 对亏了系统的学习和各种社区的帖子帮助，总算完成了手机App和服务器的IM通讯了

## 第1章 高并发时代的必备技能

### 1.2.2 Redis成为缓存事实标准的原因

1. 速度快
2. 丰富的数据结构
3. 单线程
4. 可持久化：支持RDB与AOF两种方式，将内存的数据写入外部的物理存储设备
5. 支持发布/订阅
6. 支持Lua脚本
7. **支持分布式锁**
8. **支持原子操作和事务**
9. 支持主-从(Master-Slave)复制与高可用(Redis Sentinel)集群
10. 支持管道

> 关于RDB和AOF的介绍，我找了两片网文;
>
> 笼统地讲就是AOF实时备份比RDB快，恢复比RDB慢；通常通过主从库的策略，主库为了性能可以不开启备份，从库开启AOF或者RDB。
>
> [AOF持久化](https://blog.csdn.net/qq_35433716/article/details/82195106)
>
> [Redis详解（六）------ RDB 持久化](https://www.cnblogs.com/ysocean/p/9114268.html)

## 第2章 高并发IO的底层原理

### 2.1 IO读写的基础原理

1. read系统调用与write系统调用

   两者都不是直接将数据从物理设备到内存或者相反。上层应用调用操作系统的read和write都会涉及缓冲区。

   + 调用操作系统的read

     把数据从**内核缓冲区**复制到**进程缓冲区**

   + 调用操作系统的write

     把数据从**进程缓冲区**复制到**内核缓冲区**

   read&write两大系统调用，都不负责数据在内核缓冲区和物理设备(如磁盘)之间的交换，这项底层的读写交换，是由操作系统内核(Kernel)完成的。

   *在用户程序中，无论是Socket的IO、还是文件的IO操作，都属于上层应用的开发，它们的输入(Input)和输出(Output)的处理，在编程的流程上，都是一致的。*

   > 关于read和write系统调度暂时就找了几篇网文
   >
   > [linux中的read和write系统调用](https://blog.csdn.net/u013592097/article/details/52057663)
   >
   > [read和write函数](https://www.e-learn.cn/content/qita/968658)

#### 2.1.1 内核缓冲区与进程缓冲区

​	如果频繁执行物理设备的实际IO操作，无疑影响系统性能。有了缓冲区，上层应用使用read系统调用时，仅仅把数据从内核缓冲区复制到上层应用的缓冲区(进程缓冲区)；write系统调用则把数据从进程缓冲区复制到内核缓冲区中。*底层操作会对内核缓冲区进行监控，等待缓冲区达到一定数量的时候，再进行IO设备的中断操作，集中执行物理设备的实际IO操作*，这种机制提高了系统的性能。<u>至于什么时候中断(读中断、写中断)，由操作系统的内核来决定，用户程序则不需要关心</u>。

​	从数量上看，**在Linux系统中，操作系统内核只有一个内核缓冲区。而每个用户程序(进程)，有自己独立的缓冲区，叫作进程缓冲区**。所以，用户程序的IO读写程序，在大多数情况下，并没有进行实际的IO操作，而是在<u>进程缓冲区和内核缓冲区</u>之间直接进行数据的交换。

#### 2.1.2 详解典型的系统调用流程

​	客户端请求->Linux内核通过系统调度read&write读取网卡内数据并存到内核缓冲区->用户程序通过read系统调度读取内核缓冲区的数据到进程缓冲区->用户程序处理加工数据->用户程序调用write系统调度将数据存入内核缓冲区->Linux内核通过网络IO将内核缓冲区数据写入网卡->网卡通过底层的通信协议将数据发送给目标客户端

### 2.2 四种主要的IO模型

常见的四种IO模型：

1. 同步阻塞IO (Blocking IO)

   + 阻塞与非阻塞

     阻塞IO，指的是需要内核IO操作彻底完成后，才返回到用户空间执行用户的操作。**阻塞指的是用户空间程序的执行状态**。

   + 同步与异步

     同步IO，是一种**用户空间与内核空间的IO发起方式**。同步IO是指用户空间的线程是主动发起IO请求的一方，内核空间是被动接受方。异步IO则反过来，是指系统内核是主动发起IO请求的一方，用户空间的线程是被动接受方。

2. 同步非阻塞IO (Non-blocking IO)

   ​	非阻塞IO，即用户空间的程序不需要等待内核IO彻底完成，可以立即返回用户空间执行用户的操作，即处于非阻塞的状态，与此同时内核会立即返回给用户一个状态值。

   ​	简单说，*非阻塞是用户空间(调用线程)拿到内核返回的状态值就返回自己的空间*，IO操作可以干就干，不可以干就干别的事情。

   ​	非阻塞IO要求socket被设置为NONBLOCK。

   ​	**这里的NIO (同步非阻塞IO)模型，并非Java的NIO(New IO)库**。

3. IO多路复用 (IO Multiplexing)

   ​	即经典的**Reactor反应器设计模式**，有时也被称为*异步阻塞IO*，Java中的Selector选择器和Linux中的epoll都是这种模型。

4. 异步IO (Asynchronous IO)

   ​	异步IO，指的是用户空间与内核空间的调用方式反过来。用户空间的线程变成被动接受者，而内核空间变成了主动调用者。*这有点类似Java中比较典型的回调模式，用户空间的线程向内核空间注册了各种IO事件的回调函数，而内核去主动调用*。

#### 2.2.1 同步阻塞IO (Blocking IO)

​	阻塞IO优点：应用的程序开发简单，在阻塞等待数据期间，用户线程挂起。在阻塞期间，用户线程基本不会占用CPU资源。

​	阻塞IO缺点：一般情况，会为每个连接配备一个独立的线程；反过来就是**一个线程维护一个连接的IO操作**。高并发场景下，需要使用大量的线程来维护大量的网路连接，内存、线程切换开销会非常巨大。

#### 2.2.2 同步非阻塞NIO (None Blocking IO)

​	内核IO操作分为：“**等待内核缓冲区数据**”，“**复制到用户缓冲区**”两部分。那么阻塞IO需要等待两个步骤都结束才能停止阻塞。而异步IO在非阻塞的socket发起read读操作的系统调用，流程如下：

1. 在内核数据没有准备好的阶段，用户线程发起IO请求时，立即返回。所以，为了读取到最终的数据，用户线程需要不断地发起IO系统调用。
2. 内核数据到达后，用户线程发起系统调用，用户线程阻塞。内核开始复制数据，它会将数据从内核缓冲区复制到用户缓冲区(用户空间的内存)，然后内核返回结果(例如返回复制到的用户缓冲区的字节数)。
3. 用户线程读到数据后，才会解除阻塞状态，重新运行起来。也就是说，用户进程需要经过多次的尝试，才能保证最终真正读到数据，而后继续执行。

​	同步非阻塞IO的特点：应用程序的线程需要不断地进行IO系统调用，轮询数据是否已经准备好，如果没有准备好，就继续轮询，直到完成IO系统调用为止。

+ 优点：每次发起IO系统调用，在内核等待数据过程中可以立即返回。用户线程不会阻塞，实用性较好。
+ 缺点：不断地轮询内核，会占用大量的CPU时间，效率低下。

​	*同步非阻塞IO，可以简称NIO，但不是Java中的NIO*。<u>Java的NIO(New IO)，对应的不是四种基础IO模型的NIO(None Blocking IO)模型，而是另外的一种模型，叫作IO多路复用模型(IO Multiplexing)</u>。

#### 2.2.3 IO多路复用模型 (IO Multiplexing)

> [epoll原理详解及epoll反应堆模型](https://blog.csdn.net/daaikuaichuan/article/details/83862311#font_size5epollfont_20)

IO多路复用模型用于避免同步非阻塞IO模型中轮询等待的问题。

​	在IO多路复用模型中，引入了一种新的系统调用，查询IO的就绪状态。在Linux系统中，对应的系统调用为select/epoll系统调用。通过该系统调用，一个进程可以监视多个文件描述符，一旦某个描述符就绪(一般是内核缓冲区可读/可写)，内核就能够将就绪的状态返回给应用程序，随后，应用程序根据就绪的状态，进行相应的IO系统调用。

​	目前支持IO多路复用的系统调用，有select、epoll等等。select系统调用，几乎在所有的操作系统上都有支持，具有良好的跨平台特性。epoll是在Linux2.6内核中提出的，是select系统调用的Linux增强版本。

​	IO多路复用模型的特点：涉及两种系统调用System Call（select/epoll就绪查询和IO操作）。<u>IO多路复用模型建立在操作系统的基础设施之上，即操作系统的内核必须能够提供多路分离的系统调用select/epoll</u>。

​	和NIO模型相似，**多路复用IO也需要轮询**。负责select/epoll状态查询调用的线程，需要不断地进行select/epoll轮询，查找出达到IO操作就绪地socket连接。

​	<u>IO多路复用模型与同步非阻塞IO模型有密切关系。对于注册在选择器上的每一个可以查询的socket连接，一般都设置成为**同步非阻塞**模型</u>。（对于用户程序而言是无感知的）

​	**IO多路复用模型的优点：与一个线程维护一个连接的阻塞IO模式相比，使用select/epoll的最大优势在于，一个选择器查询线程可以同时处理成千上万个连接(Connection)**。

​	Java语言的NIO（New IO）技术，使用的就是IO多路复用模型。在Linux系统上，使用的是epoll系统调用。

​	IO多路复用模型的缺点：**本质上，select/epoll系统调用是阻塞式的，属于同步IO**。

​	如何彻底解除线程的阻塞，就必须使用异步IO模型。

#### 2.2.4 异步IO模型 (Asynchronous IO)

​	AIO基本流程：用户线程通过系统调用，向内核注册某个IO操作。内核在整个IO操作(包括数据准备、数据复制)完成后，通知用户程序，用户执行后续的业务操作。

​	**在异步IO模型中，在整个内核的数据处理过程中，包括内核将数据从网络物理设备(网卡)读取到内核缓冲区、将内核缓冲区的数据复制到用户缓冲区，用户程序都不需要阻塞**。

​	异步IO模型的特点：在内核等待数据和复制数据的两个阶段，用户线程都不是阻塞的。用户线程需要接纳内核的IO操作完成的时间，或者用户线程需要注册一个IO操作完成的回调函数。正因为如此，<u>异步IO有时候也被称为信号驱动IO</u>。

​	缺点：应用程序仅需要进行事件的注册与接收，其余的工作都留给了操作系统，也就是说，**需要底层内核提供支持**。

​	目前，Windows系统通过IOCP实现了真正的异步IO。而Linux系统下2.6版本才引入，还不完善，底层仍用epoll实现，与IO多路复用相同，性能上没有明显的优势。

​	**Netty使用的是IO多路复用模型，而不是异步IO模型**。

### 2.3 通过合理配置来支持百万级并发连接

即时采用了最先进的模型，如果不进行合理的配置，也没有办法支撑百万级的网络连接并发。

​	这里所涉及的配置，就是Linux操作系统中文件句柄数的限制。

​	在生产环境Linux系统中，基本都需要解除文件句柄数的限制。原因是，Linux的系统默认指为1024，即一个进程最多可以可以接收1024个socket连接。这是远远不够的。

​	文件句柄，也叫文件描述符。在Linux系统中，文件可分为：普通文件、目录文件、链接文件和设备文件。文件描述符(File Descriptor)是内核为了高效管理已被打开的文件所创建的索引，它是一个非负整数(通常是小整数)，用于指代被打开的文件。所有的IO系统调用，包括socket的读写调用，都是通过文件描述符完成的。

​	Linux通过ulimit命令，可以看到单个进程能够打开的最大文件句柄数量。

```shell
ulimit -n
```

​	如果要永久地调整这个系统参数，如下操作

```shell
ulimit -SHn 1000000
```

​	普通用户通过ulimit命令，可以将软极限改到硬极限的最大设置值。如果要修改硬极限，必须拥有root用户权限。

​	终极解除Linux系统的最大文件打开数量的限制，可以通过编辑Linux的极限配置文件/etc/security/limits.conf来解决，修改此文件，加入如下内容。

```shell
	soft nofile 1000000
	hard nofile 1000000
```

​	soft nofile表示软性极限，hard nofile表示硬式极限。

​	在使用和安装目前非常火的分布式搜索引擎——ElasticSearch，就必须去修改这个文件，增加最大的文件句柄数的极限值。

​	在服务器运行Netty时，也需要去解除文件句柄数量的限制，修改/etc/security/limits.conf文件即可。

## 第3章 Java NIO通信基础详解

​	现在主流的技术框架或中间件服务器，都使用了Java NIO技术，譬如Tomcat、Jetty、Netty。

### 3.1 Java NIO简介

​	Java1.4版本开始，引进新的异步IO库，称之为Java New IO库，简称JAVA NIO。New IO类库的目标，就是要让Java支持非阻塞IO，基于这个原因，更多人喜欢称Java NIO为非阻塞IO(Non-Block IO)，称“老的”阻塞式的Java IO为OIO (Old IO)。

​	Java NIO由以下三个核心组件组成：

+ Channel（通道）
+ Buffer（缓冲区）
+ Selector（选择器）

​	Java NIO属于多路复用模型，提供了统一的API，为大家屏蔽了底层的不同操作系统的差异。

#### 3.1.1 NIO和OIO的对比

1. OIO面向流(Stream Oriented)，NIO面向缓冲区(Buffer Oriented)。

   OIO不能随便改变流Stream的读取指针，而NIO读写通过Channel和Buffer，可以任意操作指针位置。

2. OIO的操作是阻塞的，而NIO的操作是非阻塞的。

   OIO调用read，必须等read系统调用的数据准备和数据复制都结束后才解除阻塞；而NIO在数据准备阶段会直接返回不阻塞，只有在(内核缓冲区)有数据(进行数据复制或已经完成复制后)会阻塞，且NIO使用多路复用模式，一个Selector(一个线程)可以轮询成千上万个连接Connection，不断进行select/epoll轮询，查找出达到IO操作就绪的socket连接。

3. OIO没有选择器(Selector)概念，而NIO有选择器的概念。

   NIO的实现，基于底层的选择器的系统调用。**NIO的选择器，需要底层操作系统提供支持**。而OIO不需要用到选择器。

#### 3.1.2 通道(Channel)

​	在OIO中，同一个网络连接会关联到两个流：一个输入流(Input Stream)，另一个输出流(Output Stream)。通过这两个流，不断地进行输入和输出的操作。

​	在NIO中，同一个网络连接使用一个通道表示，所有的NIO的IO操作都是从通道开始的。一个通道类似于OIO的两个流的结合体，既可以从通道读取，也可以向通道写入。

#### 3.1.3 Selector选择器

​	IO多路复用：一个进程/线程可以同时监听多个文件描述符(<u>一个网络连接，操作系统底层使用一个文件描述符来表示</u>)，**一旦其中的一个或者多个文件描述符可读或者可写，系统内核就通知进程/线程**。在Java应用层面，使用Selector选择器来实现对多个文件描述符的监听。

​	实现多路复用，具体开发层面，首先把通道注册到选择器，然后通过选择器内部的机制，可以查询(select)这些注册的通道是否有已经就绪的IO事件(例如可读、可写、网络连接完成)。

​	一个选择器只需要一个线程来监控。与OIO相比，使用选择器的最大优势：系统开销小，系统不必为每一个网络连接(文件描述符)创建进程/线程，从而大大减小了系统的开销。

#### 3.1.4 缓冲区(Buffer)

​	通道的读取将数据从通道读取到缓冲区；通道的写入将数据从缓冲区写入到通道中。

### 3.2 详解NIO Buffer类及其属性

​	NIO的Buffer(缓冲区)本质上是一个内存块，既可以写入数据，也可以从中读取数据。NIO的Buffer类是一个抽象类，位于java.nio包，内部是一个内存块(数组)。

​	NIO的Buffer对象比其普通内存块(Java数组)，提供了更有效的方法来进行写入和读取的交替访问。

​	强调，**Buffer类是一个非线程安全类**。

#### 3.2.1 Buffer类

​	Buffer类是一个抽象类，对应于Java的主要数据类型，在NIO中有8种缓冲区类，分别如下：ByteBuffer、CharBuffer、DoubleBuffer、FloatBuffer、IntBuffer、LongBuffer、ShortBuffer、MappedByteBuffer。

​	前7种Buffer覆盖了能在IO中传输的所有Java基本数据类型。<u>第8种类型MappedByteBuffer是专门用于内存映射的一种ByteBuffer类</u>。

​	**实际应用最多的还是ByteBuffer二进制字节缓冲区类型**。

#### 3.2.2 Buffer类的重要属性

​	Buffer类内部有一个byte[]数组内存块，作为内存缓冲区。其中，三个重要的成员属性capacity(容量)、position(读写位置)、limit(读写的限制)。此外，标记属性mark(标记)，可以将当前的position临时存入mark中，需要的时候可以再从mark标记恢复到position位置。

1. capacity属性

   ​	表示内部容量大小，Buffer类的对象初始化会按照其大小分配内部的内存，不能再改变。一旦写入的对象的数量超过capacity容量，缓冲区就满了，不能再写入。

   ​	**强调，capacity容量不是指内存块byte[]数组的字节的数量，指的是写入的对象的数量**。

   ​	Buffer是抽象类，使用子类实例化，例如DoubleBuffer写入的数据是double类型，如果capacity是100，那么最多可以写入100个double数据。

2. position属性

   ​	position属性与缓冲区的读写模式有关。

   ​	写入模式下，position的值变化规则如下：(1)刚进入写模式时，position值为0，表示当前的写入位置为从头开始。(2)每当一个数据写入缓冲区之后，position会向后移动到下一个可写的位置。(3)初始的position值为0，最大的可写值为limit-1。当position达到时，缓冲区就无空间可写了。

   ​	在读模式下，(1)缓冲区刚进入到读模式时，position被重置为0。(2)从缓冲区读取时，从position位置开始读，读取数据后，position移动到下一个可读的位置。(3)position最大值为最大可读上限limit，当position达到limit时，缓冲区无数据可读。

   ​	**调用flip翻转方法，可以转换缓冲区的读写模式**。

   ​	flip反转过程中，position由原本的写入位置变成新的可读位置，也就是0，表示可以从头开始读。flip翻转的另一半工作，就是调整limit属性。

3. limit属性

   ​	表示读写的最大上限，与缓冲区的读写模式有关。

   ​	写模式，表示写入数据最大上限。刚进入写模式时，limit的值会被设置成缓冲区的capacity容量值，表示可以一直将缓冲区的容量写满。

   ​	读模式，表示最多能从缓冲区读取到多少数据。

   ​	调用flip翻转方法，将写模式下的position值设置成读模式下的limit值，即将之前的写入的最大数量作为可以读取的上限值。

#### 3.2.4 4个属性的小结

| 属性     | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| capacity | 容量，可以容纳的最大数据量；在缓冲区创建时设置并不能再改变   |
| limit    | 上限，缓冲区中当前的数据量                                   |
| position | 位置，缓冲区下一个要被读写的元素的索引                       |
| mark     | 标记，调用mark()方法来设置mark=position，再调用reset()可以让position恢复到mark标记的位置即postion=mark |

### 3.3 详解NIO Buffer类的重要方法

#### 3.3.1 allocate()创建缓冲区

​	在使用Buffer之前，需要先获取Buffer子类的实例对象，并分配内存空间。

​	为获取Buffer实例对象，不是new，而是调用子类的allocate()方法。

#### 3.3.2 put()写入缓冲区

​	<u>調用allocate方法分配内存、返回了实例对象后，缓冲区实例对象处于写模式</u>，可以写入对象。

ps:源码中Buffer子类有成员变量Boolean isReadOnly,由于java默认赋值false，即写模式。

#### 3.3.3 flip()翻转

​	向缓冲区写入数据后，需要先把缓冲区切换从写模式切换道读模式，才能够从缓冲区读取数据。

​	flip()方法的从写到读转换的规则，详细介绍如下：

​	首先，设置可读的长度上线limit。将写模式下的缓冲区中内容的最后写入位置position值，作为读模式下的limit上限值。

​	其次，把读的起始位置position设置为0，表示从头开始读。

​	最后，**清除之前的mark标记**，因为mark保存的是写模式下的临时位置。在读模时下，如果继续使用旧的mark标记，会造成位置混乱。

ps:我自己试了下，如果没有先mark()，调用reset()会报错。源码上mark初始值为-1。

```java
public final Buffer flip() {
    limit = position;
    position = 0;
    mark = -1;
    return this;
}
```

​	要将缓冲区再一次切换到写入模式，可以调用Buffer.clear()清空或者Buffer.compact()压缩方法，将缓冲区转换为写模式。

+ 写模式->读模式：flip()
+ 读模式->写模式：Buffer.clear()或Buffer.compact()

```java
public final Buffer clear() {
    position = 0;
    limit = capacity;
    mark = -1;
    return this;
}
// public abstract IntBuffer compact();
```

#### 3.3.4 get()从缓冲区读取

​	读模式下，get方法每次从position位置读取一个位置，并进行相应的缓冲区属性调整。

​	当limit与position相等，若继续get()读，会抛出BufferUnderflowException异常。

​	缓冲区可以重复读。

#### 3.3.5 rewind()倒带

​	已经读完的数据，如果需要再读一遍，可以调用rewind()方法。

+ position重置为0
+ limit不变
+ **mark标记被清除**，赋值-1

```java
public final Buffer rewind() {
    position = 0;
    mark = -1;
    return this;
}
```

​	rewind()和flip()相似，区别在于：rewind()不会影响limit属性；而flip()会重设limit属性。

#### 3.3.6 mark()和reset()

​	Buffer.mark()和Buffer.reset()配套使用。mark()保存当前postion值到mark属性，reset()将mark值恢复到position中。

#### 3.3.7 clear()清空缓存区

+ position重置为0
+ limit设置为容量上限
+ mark清除，赋值为-1

```java
public final Buffer clear() {
    position = 0;
    limit = capacity;
    mark = -1;
    return this;
}
```

#### 3.3.8 使用Buffer类的基本步骤

​	总体来说，使用Java NIO Buffer类的基本步骤如下：

1. 使用创建子类实例对象的allocate()方法，创建一个Buffer类的实例对象。
2. 调用put()方法，将数据写入到缓冲区中。
3. 写入完成后，在开始读取数据前，调用Buffer.flip()方法，将缓冲区转换为读模式。
4. 调用get()方法，从缓冲区中读取数据。
5. 读取完成后，调用Buffer.clear()或Buffer.compact()方法，将缓冲区转换为写模式。

> Buffer.compact()
>
> 将剩余未读数据赋值到数组开头，limit=limit-position,即limit大小为剩余数据的数量，position置0。
>
> 比如for循环put 写0~4这5个数字，然后调用flip()再for循环get读0~2这3个数字，调用compact(),此时hb成员变量的内容为3 4 2 3 4；position为0，limit为2，capacity不受影响。这里hb即申请的heap buffers，假设使用IntBuffer，那么hb即final int[] hb;

### 3.4 详解NIO Channel(通道)类

​	NIO 中一个连接就是用一个Channel来表示。一个通道可以表示一个底层的文件描述符，例如硬盘设备、文件、网络连接等。Java NIO的通道可以更细化，对应不同的网络传输协议类型，Java中都有不同的NIO Channel实现。

#### 3.4.1 Channel(通道)的主要类型

​	这里只介绍最为重要的四种Channel(通道)实现：FileChannel、SocketChannel、ServerSocketChannel、DatagramChannel。

​	对于以上四种通道，说明如下：

1. FileChannel文件通道，用于文件的数据读写。
2. SocketChannel套接字通道，用于Socket套接字TCP连接的数据读写。
3. ServerSocketChannel服务器套接字通道(或服务器监听通道)，允许我们监听TCP连接请求，为每个监听到的请求，创建一个SocketChannel套接字通道。
4. DatagramChannel数据包通道，用于UDP协议的数据读写。

​	这四种通道，涵盖了文件IO、TCP网络、UDP IO基础IO。下面从Channel的获取、读取、写入、关闭四个重要的操作，来对四种通道进行简单的介绍。

#### 3.4.2 FileChannel文件通道

​	FileChannel是专门操作文件的通道。**FileChannel为阻塞模式，不能设置为非阻塞模式**。

1. 获取FileChannel通道

   + 通过文件的输入流、输出流获取FileChannel文件通道。

     ```java
     FileInputStream fileInputStream = new FileInputStream(path);
     FileChannel channel = fileInputStream.getChannel();
     ```

   + 通过RandomAccessFile文件随机访问类，获取FileChannel文件通道。

     同样调用RandomAccessFile实例对象的getChannel()方法。

2. 读取FileChannel通道

   ​	大部分应用场景，从通道读取数据都会调用通道的int read(ByteBuffer dst)方法，它从通道读取到数据写入到ByteBuffer缓冲区，并且返回读取到的数据量。

   ```java
   while( (length = inChannel.read(buf)) != -1 ){ //处理读取到的buf中的数据 }
   ```

   ​	这里对于通道Channel是读取数据，对于Buffer缓冲区是写入数据(处于写模式)。

3. 写入FileChannel通道

   ​	大部分应用场景，调用通道的int write(ByteBuffer src)方法。

   ```java
   while( ( outlength = outchannel.write(buf) ) != 0){ //... }
   ```

   ​	这里对于通道是写入数据，对于Buffer缓冲区是读取数据(必须处于读模式)

4. 关闭通道

   ```java
   channel.close();//使用完通道记得关闭
   ```

5. 强制刷新到磁盘

   在将缓冲区写入通道时，处于性能原因，操作系统不可能每次都实时将数据写入磁盘。如果需要保证写入通道的缓冲数据，最终都能真正地写入磁盘，可以调用FileChannel的force()方法。

   ```java
   channel.force(true);
   ```

   ps:看了下方法介绍，说如果是本地设备(文件系统)，能保证写入，但是如果不是本地的，不保证一定写入了。这里我估计是只写入远程的设备or网络传输协议等。

#### 3.4.3 使用FileChannel完成文件复制的实践案例

​	需要注意的是Buffer的读写模式切换。调用flip()写模式切换为读模式，调用clear()或compact()读模式切换为写模式。

​	比起使用channel的write和read方法，可以考虑使用效率更高的channel.transferFrom方法完成文件的复制。

```java
while (pos < size) {
    //每次复制最多1024个字节，没有就复制剩余的
    count = size - pos > 1024 ? 1024 : size - pos;
    //复制内存,偏移量pos + count长度
    pos += outChannel.transferFrom(inChannel, pos, count);
}
```

#### 3.4.4 SocketChannel套接字通道

​	在NIO中，涉及网络连接的通道主要有两个，一个是SocketChannel负责连接传输，另一个ServerSocketChannel负责连接的监听。

​	NIO中的SocketChannel传输通道，与OIO中的Socket类对应。

​	NIO中的ServerSocketChannel监听通道，对应于OIO中的ServerSocket类。

​	<u>ServerSocketChannel应用于服务器端，而SocketChannel同时处于服务器端和客户端。换句话说，对应于一个连接，两端都有一个负责传输的SocketChannel传输通道</u>。

​	SocketChannel和erverSocketChannel都支持阻塞式和非阻塞式两种模式。调用configureBlocking方法调整：

+ socketChannel.configureBlocking(false)设置为非阻塞模式。

+ socketChannel.configureBlocking(true)设置为阻塞模式。

​	阻塞模式下SocketChannel通道的connect、read、write同步阻塞，效率与Java旧的OIO面向流的阻塞式读写操作相同。下面讲解非阻塞模式的操作。

1. 获取SocketChannel传输通道

   ​	在客户端，先通过SocketChannel静态方法open()获得一个套接字传输通道；然后，将socket套接字设置为非阻塞模式；最后，通过connect()实例方法，对服务器的IP端口发起连接。

   ```java
   SocketChannel channel = SocketChannel.open();
   channel.configureBlocking(false);
   channel.connect(new InetSocketAddress("127.0.0.1") , 80);
   ```

   ​	非阻塞情况下，与服务器的连接可能还没有真正建立，socketChannel.connect方法就返回了，因此需要不断地自旋，检查当前是否连接到了主机。

   ```java
   while(! socketChannel.finishConnect() ){ //... }
   ```

   ​	当新连接事件到来时，在服务器端的ServerSocketChannel能成功地查询出一个新连接事件，并且通过调用服务器端ServerSocketChannel监听套接字的accept()方法，来获取新连接的套接字通道。

   ```java
   ServerSocketChannel server = (ServerSocketChannel) key.channel();
   SocketChannel socketChannel = server.accpet();
   socketChannel.configureBlocking(false);
   ```

   ​	强调，NIO套接字通道，主要用于非阻塞应用场景。所以，需要调用configureBlocking(false)，从阻塞模式设置为非阻塞模式。

2. 读取SocketChannel传输通道

   ​	当SocketChannel通道可读时，可以从SocketChannel读取数据，具体方法与前面的文件通道相同。

   ​	读取是异步的，需要检查read返回值判断是否读到了数据，除非读到对方的技术标记返回-1，否则返回读取的字节数。非阻塞模式下，需NIO的Selector通道选择器来轮询查找可读的通道。

3. 写入到SocketChannel传输通道

   ​	与写入到FileChannel一样，大部分应用场景都会调用通道的int write(ByteBuffer src)方法。

4. 关闭SocketChannel传输通道

   ​	调用channel的close方法之前，最好先调用channel的shutdownOutput()终止对此通道的写连接，此时若再尝试写入会报错ClosedChannelException。

#### 3.4.5 使用SocketChannel发送文件的实践案例

​	需要注意的点就是，可以设计成发送<u>文件名、文件大小、文件本身</u>。

#### 3.4.6 DatagramChannel数据报通道

​	和Socket套接字的TCP传输协议不同，UDP协议不是面向连接的协议。使用UDP协议时，只要知道服务器的IP和端口，就可以直接向对方发送数据。在Java中使用UDP协议传输数据，比TCP协议更加简单。在Java NIO中，使用DatagramChannel数据报通道来处理UDP协议的数据传输。

1. 获取DatagramChannel数据报通道

   ​	调用DatagramChannel类的open静态方法获取数据报通道。然后调用configureBlocking(false)方法，设置成非阻塞模式。

   ​	如果需要接收数据，还需要调用bind方法绑定一个数据报的监听端口。

   ```java
   channel.socket().bind(new InetSocketAddress(18080));
   ```

2. 读取DatagramChannel数据报通道

   ​	当DatagramChannel通道可读时，可以从DatagramChannel读取数据。读取方法为receive(ByteBuffer dst)，read()方法用于建立连接Channel，然而UDP无连接。

   ```java
   ByteBuffer buf = ByteBuffer.allocate(1024);
   SocketAddress client = datagramChannel.receive(buffer);
   ```

3. 写入DatagramChannel数据报通道

   ​	向DatagramChannel发送数据，调用的是send方法，不是write();

   ```java
   buffer.flip();//缓冲区切换到读模式
   dChannel.send(buffer, new InetSocketAddress(IP,PORT));
   buffer.clear();//缓冲区切换到写模式
   ```

   **由于UDP是面向非连接的协议，因此，在调用send方法发送数据的时候，需要指定接收方的地址(IP和端口)**

4. 关闭DatagramChannel数据报通道

   ```java
   dChannel.close();
   ```

#### 3.4.7 使用DatagramChannel数据报通道发送数据的实践案例

​	步骤基本就是获取数据报通道实例对象，往buffer里写数据，然后send到服务器端(指定IP和PORT)。

​	服务器端则是通过DatagramChannel数据报通道绑定一个服务器地址(IP+PORT)，接收客户端发送过来的UDP数据报，即从datagramChannel数据报通道接收数据，写入ByteBuffer缓冲区中。

### 3.5 详解NIO Selector选择器

#### 3.5.1 选择器以及注册

​	选择器的使命就是完后IO的多路复用。<u>一个通道代表一条连接通路，通过选择器可以同时监控多个通道的IO(输入输出)状况</u>。选择器和通道的关系，是监视和被监视的关系。

​	**一般来说，一个单线程处理一个选择器，一个选择器可以监控很多通道**。

​	通道和选择器之间的关系，通过register(注册)的方式完成。调用通道的Channel.register(Selector sel,int ops)方法，可以将通道实例注册到一个选择器中。register方法有两个参数：第一个参数，指定通道注册到的选择器实例；第二个参数，指定选择器要监控的**IO事件类型**。

​	可供选择器监控的通道IO事件类型，包括以下四种：

+ 可读：SelectionKey.OP_READ = 1 << 0
+ 可写：SelectionKey.OP_WRITE = 1 << 2
+ 连接：SelectionKey.OP_CONNECT = 1 << 3
+ 接收：SelectionKey.OP_ACCEPT = 1 << 4

​	如果选择器要监控通道的多种事件，可以用"按位或"运算符实现。

```java
int key = SelectionKey.OP_READ | SelectionKey.OP_WRITE ; // 监控读和写
```

​	**这里的IO事件不是对通道的IO操作，而是通道的某个IO操作的一种就绪状态，表示通道具备完成某个IO操作的条件**。

​	比方说，某个SocketChannel通道，完成了和对端的握手连接，则处于“连接就绪”(OP_CONNECT)状态。

​	再比方说，某个ServerSocketChannel服务器通道，监听到一个新连接的到来，则处于"接收就绪"(OP_ACCEPT)状态。

​	还比方说，一个有数据可读的SocketChannel通道，处于"读就绪"(OP_READ)状态；一个等待写入数据的，处于"写就绪"(OP_WRITE)状态。

#### 3.5.2 SelectableChannel可选择通道

​	并不是所有的通道都是可以被选择器监控或选择的。比方说，FileChannel文件通道就不能被选择器复用。判断一个通道能否被选择器监控或选择，有一个前提：判断它是否继承了抽象类SelectableChannel(可选择通道)。<u>如果继承了SelectableChannel，则可以被选择， 否则不能</u>。

​	SelectableChannel提供了实现通道的可选择性所需要的公共方法。**Java NIO中所有网络连接Socket套接字通道，都继承了SelectableChannel类，都是可选择的**。FileChannel没有继承它，不是可选择通道。

#### 3.5.3 SelectionKey选择键

​	通道和选择器的监控关系注册成功后，就可以选择就绪事件。<u>通过调用Selector的select()方法，选择器可以不断地选择通道中所发生操作的就绪状态，返回注册过的感兴趣的那些IO事件</u>。

​	SelectionKey选择键，即那些被选择器选中的IO事件。<u>一个IO事件发生(就绪状态达成)后，如果之前在选择器中注册过，就会被选择器选中，并放入SelectionKey选择键集合中</u>；如果没有注册过，即时发生了IO事件，也不会被选择器选中。可以简单理解为：选择键就是被选中了的IO事件。

​	在编程时，选择键的功能强大。**通过SelectionKey选择键，不仅仅可以获得通道的IO事件类型，比方说Selection.OP_READ；还可以获得IO事件发生的所在通道；另外也可以获得选出选择键的选择器实例**。

#### 3.5.4 选择器使用流程

​	(1)获取选择器实例；(2)将通道注册到选择器；(3)轮询感兴趣的IO就绪事件(选择键集合)

第一步：获取选择器实例

​	通过调用静态工厂方法open()获取选择器实例

```java
Selector selector = Selector.open();
```

​	open()内部向选择器SPI（SelectorProvider）发出请求，通过默认的SelecterProvider(选择器提供者)对象，获取一个新的选择器实例。SPI(Service Provider Interface，服务提供接口)，是JDK的一种可以扩展的服务提供和发现机制。

​	<u>Java通过SPI方式，提供选择器的默认实现版本。其他服务提供商可以通过SPI方式提供定制化版本的选择器的动态替换或者扩展</u>。

```java
public static Selector open() throws IOException {
    return SelectorProvider.provider().openSelector();
}
```

第二步：将通道注册到选择器实例

​	通过调用通道的register()方法，将ServerSocketChannel通道注册到选择器上。

```java
serverSocketChannel.register(selector,SelectionKey.OP_ACCEPT);
```

+ 注册到选择器的通道，必须处于非阻塞模式，否则抛出IllegalBlockingModeException异常。（*FileChannel文件通道只有阻塞模式，不能与选择器一起用；而Socket套接字相关的所有通道都可以*）

+ 一个通道，并不一定要支持所有的四种IO事件。（例如服务器监听通道ServerSocketChannel仅支持Accept接收到新连接的IO事件；而SocketChannel传输通道不支持Accept此IO事件）

**可以在注册之前，通过通道的validOps()方法来获取该通道所有支持的IO事件集合**。

第三步：选出感兴趣的IO就绪事件(选择键集合)

​	通过Selector选择器的select()方法，选出已经注册的、已经就绪的IO事件，保存到SelectionKey选择键集合中。SelectionKey集合保存在选择器实例内部，是一个元素为SelectionKey类型的集合(Set)。调用选择器的selectedKeys()方法，可以获取选择键集合。

​	接下来，需要迭代集合的每一个选择键，根据具体IO事件类型，执行对应的业务操作。大致处理流程如下

```java
while( selector.select() > 0 ){
    Set selectedKeys = selector.selectedKeys();
    Iterator keyIterator = selectedKeys.iterator();
    while(keyIterator.hasNext()){
        SelectionKey key = keyIterator.next();
        if(key.isAcceptable()){
        	// IO事件：ServerSocketChannel服务器监听通道有新连接
        } else if(key.isConnectable()){
            // IO事件：传输通道连接成功
        } else if(key.isReadable()){
            // IO事件：传输通道可读
        } else if(key.isWritable()){
            // IO事件：传输通道可写
        }
        // 处理完成后，移除选择键。
        keyIterator.remove();
    }
}
```

​	处理完成后，需要将选择键从SelectionKey集合中移除，防止下一次循环的时候，被重复处理。SelectionKey集合不能添加元素，如果试图向选择键集合添加元素，则抛出java.lang.UnsupportedOperationException异常。

​	用于选择就绪的IO事件的select()方法，有多个重载的实现版本：

​	(1) select()：阻塞调用，直到至少有一个通道发生了注册的IO事件。

​	(2) select(long timeout)：和前者一样，但最长阻塞时间为timeout指定的毫秒数。

​	(3) selectNow()：非阻塞，不管有没有IO时间，都会立刻返回。

​	select()方法返回的整数值(int 整数类型)，表示发生了IO事件的通道数量（从上一次select到这次select之间）。强调一下，**select()方法返回的数量与IO事件数无关，是指发生了选择器感兴趣的IO事件的通道数**。

#### 3.5.5 使用NIO实现Discard服务器的实践案例

​	主要需要留意的就两步

```java
// ...
// 13、若选择键的IO事件是"可读"事件， 读取数据
SocketChannel socketChannel = (SocketChannel) selectedKey.channel();
// ...
// 15、移除选择键
selectedKeys.remove();
```

​	程序涉及两次选择器注册：一次是注册serverChannel服务器通道；另一次，注册接收到的socketChannel客户端传输通道值。serverChannel服务器通道注册的，是新连接的IO事件Selection.OP_ACCEPT；客户端socketChannel传输通道注册的，是可读IO事件SelectionKey.OP_READ。

​	DiscardServer在对选择键进行处理时，通过对类型进行判断，然后进行相应的处理。

1. 如果是Selection.OP_ACCEPT新连接事件类型，代表serverChannel服务器通道发生了新连接事件，则通过服务器通道的accept方法，获取新的socketChannel传输通道，并且注册到选择器。
2. 如果是SelectionKey.OP_READ可读事件类型，代表某个客户端通过通道有数据可读，则读取选择键socketChannel传输通道的数据，然后丢弃。

#### 3.5.6 使用SocketChannel在服务器端接收文件的实践案例

客户端大致流程：

+  获取文件通道FileChannel
+ 创建与服务器连接的SocketChannel
+ 发送文件名、文件大小
+ 发送文件内容，每次先从FileChannel读取到buffer，再从buffer写入到socketChannel

服务器端大致流程：

+ 获取Selector选择器
+ 获取ServerSocketChannel通道，设置为非阻塞
+ 为通道绑定连接
+ 把ServerSocketChannel通道注册到选择器上，且注册的IO事件为"接收新连接"
+ 轮询有感兴趣的IO就绪事件(选择键集合)的通道
+ 获取选择键集合，遍历单个选择键并处理
  + 如果IO事件是"接收新连接"，就获取客户端新连接
  + 客户端新连接切换为非阻塞模式
  + 将客户端新连接注册到selector选择器上
+ 对余下业务处理
+ 移除已选择的选择键

​	其中，对于文件接收的处理，由于客户端分3段发送文件名、文件大小、文件内容，所以服务器端简单的用if-else按顺序判断存储文件的包装类对应属性是否为空，为空则读取并存到相对应成员变量中。

> ​	对应每一个客户端socketChannel，创建一个Client客户端对象，用于保存客户端状态，分别保存文件名、文件大小和写入的目标文件通道outChannel。
>
> ​	socketChannel和Client对象之间是一对一的对应关系：建立连接的时候，以socketChannel作为键Key，Client对象作为值Value，将Client保存在map中。当socketChannel传输通道有数据可读时，通过选择键key.channel()方法，取出IO事件所在socketChannel通道。然后通过socketChannel通道，从map中取到对应的Client对象。
>
> ​	**接收到数据时，如果文件名为空，先处理文件名称，并把文件名保存到Client对象，同时创建服务器上的目标文件；接下来再读到数据，说明收到了文件大小，把文件大小保存到Client对象；接下来再收到数据，说明是文件内容了，则写入Client对象的outChannel文件通道中，直到数据读取完毕。**

### 3.6 本章小结

​	与JavaOIO相比，Java NIO编程大致的特点如下：

1. 在NIO中，服务器接收新连接的工作，是异步进行的。不像Java的OIO那样，服务器监听连接，是同步、阻塞的。NIO可以通过选择器(多路复用器)，后续不断地轮询选择器的选择键集合，选择新到来的连接。
2. 在NIO中，SocketChannel传输通道的读写是异步的。如果没有可读写的数据，负责IO通信的线程不会同步等待。这样，线程就可以处理其他连接的通道；不需要像OIO那样，线程一直阻塞，等待所负责的连接可用为止。
3. NIO中，一个选择器线程可以同时处理成千上万个客户端连接，性能不会随着客户端的增加而下降。

​	*总之，有了**Linux底层的epoll支持**，有了Java NIO Selector选择器这样的应用层IO复用技术，Java车程序从而可以实现IO通信的高TPS、高并发，使服务器具备并发数十万、数百万的连接能力。Java的NIO技术非常适合用于高性能、高负载的网络服务器。鼎鼎大名的通信服务器中间件Netty，就是基于Java的NIO技术实现的。*

​	Java NIO技术仅仅是基础，如果要实现通信的高性能和高并发，还离不开高效的设计模式——Reactor反应器模式。

## 第4章 鼎鼎大名的Reactor反应器模式

### 4.1 Reactor反应器模式为何如此重要

​	Web服务器Nginx、高性能缓存服务器Redis、高性能通信中间件Netty皆基于反应器模式。

#### 4.1.1 为什么首先学习Reactor反应器模式

​	越是高水平的Java代码，抽象的层次越高。如果先了解代码的设计模式，再去看代码，阅读就很轻松。

#### 4.1.2 Reactor反应器模式简介

​	Dong Lea，Java中Concurrent并发包的重要作者之一，在文章《Scalable IO in Java》中对反应器模式的定义，具体如下：

​	反应器模式由**Reactor反应器线程、Handlers处理器**两大角色组成：

​	（1）Reactor反应器线程的职责：负责响应IO事件，并且分发到Handlers处理器。

​	（2）Handlers处理器的职责：非阻塞的执行业务处理逻辑。

#### 4.1.3 多线程的OIO的致命缺陷

​	Java的OIO编程，用一个while循环，不断判断是否有新连接

```java
while(true){
    socket = accept();	// 阻塞，接收连接
    handle(socket);		// 读取数据、业务处理、写入结果
}
```

​	为了解决此方法的严重连接阻塞问题，出现一个极为经典模式Connection Per Thread(一个线程处理一个连接)模式。

​	Conneciton Per Thread模式的优点：解决了前面的新连接被严重阻塞的问题，在一定程度上，极大地提高了服务器的吞吐量。缺点：对于大量连接，需要消耗大量的线程资源。

### 4.2 单线程Reactor反应器模式

​	在反应器模式中，有Reactor反应器和Handler处理器两个重要的组件。

​	(1) Reactor反应器：负责查询IO事件，当检测到一个IO事件，将其发送给相应的Handler处理器去处理。这里的IO事件，就是NIO中选择器监控的通道IO事件。

​	(2) Handler处理器：与IO事件（或者选择键）绑定，负责IO事件的处理。完成真正的连接建立、通道的读取、处理业务逻辑、负责将结果写出到通道等。

#### 4.2.1 什么是单线程Reactor反应器

​	简单说，Reactor反应器和Handlers处理器处于一个线程中执行。

​	基于Java NIO，实现简单的单线程版本的反应器模式，需要用到SelectionKey选择键的几个重要的成员方法：

+ void attach(Object o)

  ​	此方法可以将任何的Java POJO对象，作为附件添加到SelectionKey实例，相当于附件属性的setter方法。

  ​	这方法非常重要，因为在单线程版本的反应器模式中，*需要将Handler处理器实例，作为附件添加到SelectionKey实例*。

+ Object attachment()

  ​	取出之前通过attach(Object o)添加到SelectionKey选择键实例的附件，相当于附件属性的getter方法，与attach(Object o)配套使用。

  ​	该方法同样非常重要，当IO事件发生，选择键被select方法选到，可以直接将事件的附件取出，也就是之前绑定的Handler处理器实例，通过该Handler，完成相应的处理。

 	反应器模式中，需要进行attach和attachment结合使用：选择键注册完成后，调用attach方法，将Handler处理器绑定到选择键；当事件发生时，调用attachment方法，可以从选择键中取出Handler处理器，将事件分发到Handler处理器中，完成业务处理。

#### 4.2.2 单线程Reactor反应器的参考代码

#### 4.2.3 一个Reactor反应器版本的EchoServer实践案例

​	EchoServer回显服务器：读取客户端的输入，回显到客户端。基于Reactor反应器模式来实现，设计3个重要的类：

​	（1）设计一个反应器类：EchoServerReactor类

​	（2）设计两个处理器类：AcceptorHandler新连接处理器、EchoHandler回显处理器。

#### 4.2.4 单线程Reactor反应器模式的缺点

​	单线程Reactor反应器模式，是基于Java的NIO实现的。相对于传统的多线程OIO，反应器模式不再需要启动成千上万条线程，效率自然大大提升。

​	然而， Reactor和Handler都执行在同一线程上，当其中某个Handler阻塞时，会导致其他所有的Handler都得不到执行。如果被阻塞的Handler不仅仅负责输入和输出处理的业务，还包括负责连接监听的AcceptorHandler处理器，当AcceptorHandler被阻塞，会导致整个服务不能接收新的连接。

​	目前服务器都是多核的，单线程反应器模式不能充分利用多和资源。

### 4.3 多线程的Reactor反应器模式

#### 4.3.1 多线程池Reactor反应器演进

+ Handler采用多线程
+ Reactor引入多个Selector选择器

​	(1)将负责输入输出处理的IOHandler处理器的执行，放入独立的线程池中。业务处理线程与负责服务监听和IO事件查询的反应器线程相隔离

​	(2)如果服务器为多核的CPU，可以将反应器线程拆分成多个子反应器(SubReactor)线程；同时，引入多个选择器，每一个SubReactor子线程负责一个选择器。这样，充分利用系统资源，也提高了反应器管理大量连接、选择大量通道的能力。

#### 4.3.2 多线程Reactor反应器的实践案例

​	在前面"回显服务器"(EchoServer)的基础上，完成多线程Reactor反应器的升级。

​	（1）引入多个选择器

​	（2）设计一个新的子反应器(SubReactor)类，一个子反应器负责查询一个选择器。

​	（3）开启多个反应器的处理线程，一个线程负责执行一个子反应器(SubReactor)。

​	**为了提高效率， 建议SubReactor的数量和选择器的数量一致，避免多个线程负责一个选择器，导致需要进行线程同步，引起的效率降低**。

#### 4.3.3 多线程Handler处理器的实践案例

### 4.4 Reactor反应器模式小结

1. 反应器模式和生产者消费者模式对比

   ​	相似之处：生产者消费者模式中，一个或多个生产者将事件加入到一个队列中，一个或多个消费者主动从这个队列中提取事件来处理。

   ​	不同之处：反应器模式基于查询，没有专门的队列去缓冲存储IO事件，在查询到IO事件后，反应器根据不同IO选择键(事件)将其分发给对应的Handler处理器来处理。

2. 反应器模式和观察者模式(Observer Pattern)对比

   ​	相似之处：反应器模式中，当查询到IO事件后，服务处理程序使用单路/多路分发(Dispatch)策略，同步地分发这些IO事件。观察者模式(发布/订阅模式)定义依赖关系，让多个观察者同时监听某一个主题(Topic)。这些主题发生变化时，会通知所有观察者，它们能够执行相应的处理。

   ​	不同之处：反应器模式中，Handler处理器实例和IO事件（选择键）的订阅关系，基本上是一个事件绑定到一个Handler处理器；而观察者模式中，同一个时刻，同一个主题可以被订阅过的多个观察者处理。

​	反应器模式优缺点：

1. 优点：

+ 响应快，虽然同一反应器线程本身是同步的，但不会被单个连接的同步IO所阻塞；
+ 变成先后对简单，最大程度避免了复杂的多线程同步，也避免了多线程的各个进程之间的开销。
+ 可扩展，可以方便地通过增加反应器线程个数来充分利用CPU资源。

2. 缺点：

+ 增加了一定复杂性，有一定门槛，不易于调试。
+ 需要操作系统底层的IO多路复用的支持，如Linux中的epoll。如果操作系统的底层不支持IO多路复用，反应器模式不会有那么高效。
+ 同一个Handler业务线程中，如果出现一个长时间的数据读写，会影响这个反应器中其他通道的IO处理。例如在大文件传输时，IO操作就会影响其他客户端的相应，因而对于这种操作，还需要进一步对反应器模式进行改进。

### 4.5 本章小结

## 第5章 并发基础中的Future异步回调模式

### 5.1 从泡茶的案例说起

​	为了异步执行整个泡茶流程，分别设计三条线程：主线程、清理线程、烧水线程。

1. 主线程（MainThread）:启动清理线程、启动烧水线程，等清洗、烧水的工作完成后，泡茶喝。
2. 清理线程（WashThread）：洗茶壶、洗茶杯。
3. 烧水线程（HotWaterThread）：洗好水壶，灌上凉水，放在火上，一直等水烧开。

下面分别使用阻塞模式、异步回调模式来实现泡茶喝的案例。

### 5.2 join异步阻塞

​	多线程join合并，join原理：阻塞当前的线程，直到准备合并的目标线程执行完成。

#### 5.2.1 线程的join合并流程

​	在Java中，线程Thread的合并流程：假如线程A调用了线程B的B.join方法，合并B线程。那么A线程进入阻塞状态，直到B线程执行完成。

#### 5.2.2 使用join实现异步泡茶喝的实践案例

#### 5.2.3 详解join合并方法

​	join方法的应用场景：A线程调用B线程的join方法，等待B线程执行完成；在B线程没有完成前，A线程阻塞。

+ join是实例方法，不是静态方法
+ join调用时，不是线程所指向的目前线程阻塞，而是当前线程阻塞。
+ 只有等到当前线程所指向的线程执行完成、或者超时，当前线程才能重新恢复执行。

​	join有一个问题，被合并的线程没有返回值。如果要获得异步线程的执行结果，可以使用Java的FutureTask系列类。

### 5.3 FutureTask异步回调之重武器

​	为了获取异步线程的返回结果，Java在1.5版本之后提供了一种新的多线程的创建方式——FutureTask方式。FutureTask方式包含了一系列的Java相关的类，在java.util.concurrent包中，最为重要的是FutureTask类和Callable接口。

#### 5.3.1 Callable接口

​	Callable有返回值

```java
package java.util.concurrent;
@FunctionalInterface
public interface Callable<V> {
    // call方法有返回值
    V call() throws Exception;
}
```

​	Callable不能作为Thread线程实例的target使用，为此Java提供了在Callable实例和Thread的target成员之间的一个搭桥的类——FutureTask类。

#### 5.3.2 初探FutureTask类

​	FutureTask表示一个未来执行的任务，表示新线程所执行的操作，同样java.util.concurrent包。FutureTask类的构造函数的参数为Callable类型，是对Callable的二次分装，可以执行Callable的call方法。FutureTask类间接继承了Runnable接口，从而可以作为Thread实例的target执行目标。

​	FutureTask类的构造函数的源代码：

```java
public FutureTask(Callable<V> callable) {
    if(callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;	//	ensure visibility of callable
}
```

​	FutureTask将Callable的call执行结果返回值存储起来，通过FutureTask的实例方法获取返回值。

​	在Java语言中，将FutureTask类的一系列操作，抽象出来作为一个重要的接口——Future接口。FutureTask当然也实现了该接口。

#### 5.3.3 Future接口

​	主要对并发任务的执行及获取其结果的一些操作。主要提供了3大功能：

1. 判断并发任务是否执行完成
2. 获取并发的任务完成后的结果
3. 取消并发执行中的任务

​	Future接口的源代码如下:

```java
package java.util.concurrent;
public interface Future<V> {
    boolean cancel(boolean mayInterruptRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)throws InterruptedException, ExecutionException, TimeoutException;
}
```

​	**其中，get方法是阻塞性的方法，如果并发任务没有执行完成or没有超时，调用该方法的线程会一致阻塞直到并发任务执行完成**。

#### 5.3.4 再探FutureTask类

​	FutureTask类实现了Future接口，提供了外部操作异步任务的能力。

​	FutureTask内部有一个Callable类型的成员，代表异步执行的逻辑

```java
private Callable<V> callable;
```

​	callable实例属性必须要在FutureTask类的实例构造时进行初始化。

​	FutureTask内部的run方法会执行其callable成员的call方法，执行完成后的结果保存到成员——outcome属性。

```java
private Object outcome;
```

#### 5.3.5 使用FutureTask类实现异步泡茶喝的实践案例

​	**FutureTask和Callable都是泛型类，泛型参数表示返回结果的类型。所以，在使用的时候，它们两个实例的泛型参数一定需要保持一致的。**

​	最后，通过FutureTask类的实例，取得异步线程的执行结果。

​	通过FutrueTask类的get方法获取异步结果，主线程也会被阻塞。这一点，**FutureTask和join也是一样的，它们两都是异步阻塞模式**。

​	异步阻塞的效率往往是比较低的，被阻塞的主线程不能干任何事情，唯一能干的，就是傻傻地等待。**原生Java API，除了阻塞模式的获取结果外，并没有实现非阻塞的异步结果获取方法**。如果需要用到获取异步的结果，则需要引入一些额外的框架，这里首先介绍谷歌公司的Guava框架。

### 5.4 Guava的异步回调

​	Guava是谷歌公司提供的Java扩展包，提供了一种异步回调的解决方案。例如，Guava的异步任务接口ListenableFuture，扩展了Java的Future接口，实现非阻塞获取异步结果的功能。

​	总体来说，Guava的主要手段是增强而不是另起炉灶。为了实现非阻塞获取异步线程的结果，Guava对Java的异步回调机制，做了以下的增强：

1. 引入了一个新的接口ListenableFuture，继承了Java的Future接口，使得Java的Future异步任务，在Guava中能被监控和获得非阻塞异步执行的结果。
2. 引入了一个新的接口FutureCallback，这是一个独立的新接口。该接口的目的，是在异步任务执行完成后，根据异步结果，完成不同的回调处理，并且可以处理异步结果。

#### 5.4.1 详解FutureCallback

​	FutureCallback是一个新增的接口，用来填写异步任务执行完后的监听逻辑。FutureCallback拥有两个回调方法：

1. onSuccess方法，在异步任务执行成功后被回调；调用时，异步任务的执行结果，作为onSuccess方法的参数被传入。
2. onFailure方法，在异步方法执行过程中，抛出异常时被回调；调用时，异步任务所抛出的异常，作为onFailure方法的参数被传入。

​	ps：感觉和Javascript的ES6语法的Promise类似。

​	FutureCallback的源代码如下：

```java
package com.google.common.util.concurrent;
public interface FutureCallback<V> {
    void onSuccess(@Nullable V var1);
    void onFailure(Throwable var1);
}
```

​	注意，Guava的FutureCallback与Java的Callable，名字相近，但实质不同，存在本质的区别：

1. Java的Callable接口，代表的是异步执行的逻辑。
2. Guava的FutureCallback接口，代表的是Callable异步逻辑执行完成之后，根据成功或者异常两种情况，所需要执行的善后工作。

​	Guava是对Java Future异步回调的增强，使用Guava异步回调，也需要用到Java的Callable接口。简答地说，<u>只有在Java的Callable任务执行的结果出来之后，才可能执行Guava中的FutureCallback结果回调</u>。

​	Guava引入一个新接口ListenableFuture，继承了Java的Future接口，增强了监控的能力，完成异步任务Callable和FutureCallback结果回调之间的监控关系。

#### 5.4.2 详解ListenableFuture

Guava的ListenableFuture是对Java的Future接口的扩展，可以理解为异步任务的实例。

```java
package com.google.common.util.concurrent;
import java.util.concurrent.Executor;
import java.util.concurrent.Future;
public interface ListenableFuture<V> extends Future<V> {
	// 此方法由Guava内部调用
    void addListener(Runnable r, Executor e);
}
```

仅增加了一个方法——addListener方法。作用即将前一小节的FutureCallback善后回调工作封装成一个内部的Runnable异步回调任务，在Callable异步任务完成后，回调FutureCallback进行善后处理。

​	addListener方法只在Guava内部调用，实际变成不用主动调用。

​	实际编程中，使用Guava的Futures工具类的addCallback静态方法，把FutureCallback的回调实例绑定到ListenableFuture异步任务。

```java
Futures.addCallback(listenableFuture, new FutureCallback<Boolean>(){
   public void onSuccess(Boolean r)
   {
       // listenableFuture内部的Callable成功时回调此方法
   } 
   public void onFailure(Throwable t)
   {
       // listenableFuture内部的Callable异常时回调此方法
   }
});
```

#### 5.4.3 ListenableFuture异步任务

​	如果要获取Guava的ListenableFuture异步任务实例，主要是通过向线程池ThreadPool提交Callable任务的方式来获取。这里的线程池指的是Guava自己定制的Guava线程池。

​	Guava线程池，是对Java线程池的一种装饰。

```java
// java线程池
ExecutorService jPool = Executors.newFixedThreadPool(10);
// Guava线程池
ListeningExecutorService gPool = MoreExecutors.listeningDecorator(jPool);
```

​	首先创建Java线程池，然后以它作为Guava线程池的参数，在构造一个Guava线程池。有了Guava的线程池之后，就可以通过submit方法来提交任务了；任务提交之后的返回结果，就是我们所要的ListenableFuture异步任务实例了。

​	获取异步任务实例的方式，即通过向线程池提交Callable业务逻辑来实现。

```java
// 调用submit方法来提交任务，返回异步任务实例
ListenableFuture<Boolean> hFuture = gPool.submit(hJob);
// 绑定回调实例
Futures.addCallback(listenableFuture, new FutureCallback<Booelean>(){
    // 实现回调方法，onSuccess和onFailure
});
```

​	获取了ListenableFuture实例之后，通过Futures.addCallback方法，将FutureCallback回调逻辑的实例绑定到ListenableFuture异步任务实例，实现异步执行完成后的回调。

​	Guava异步回调的流程如下：

1. 实现Java的Callable接口，创建异步执行逻辑。还有一种情况，如果不需要返回值，异步执行逻辑也可以实现Java的Runnable接口
2. 创建Guava线程池
3. 将第1步创建的Callable/Runnable异步执行逻辑的实例，通过submit提交到Guava线程池，从而获取到ListenableFuture异步任务实例。
4. 创建FutureCallback回调实例，通过Futures.addCallback将回调实例绑定到ListenableFuture异步任务上。

​	完成以上四步，当Callable/Runnable异步执行逻辑完成后，就会回调异步回调实例FutureCallback的回调方法onSuccess/onFailure。

#### 5.4.4 使用Guava实现泡茶喝的实践案例

​	Guava异步回调和Java的FutureTask异步回调， 本质的不同在于：

+ Guava是非阻塞的异步回调，调用线程是不阻塞的，可以继续执行自己的业务逻辑。
+ **FutureTask是阻塞的异步回调，调用线程是阻塞的，在获取异步结果的过程中，一直阻塞，等待异步线程返回结果。**

### 5.5 Netty的异步回调模式

​	**Netty官方文档指出Netty的网络操作都是异步的**。在Netty源代码中，大量使用了异步回调处理模式。在Netty的业务开发层面，Netty应用的Handler处理器中的业务处理代码，也都是异步执行的。所以，了解Netty的异步回调， 无论是Netty应用级的开发还是源代码的开发，都是十分重要的。

​	Netty和Guava一样，实现了自己的异步回调体系：Netty继承和扩展了JDK Future系列异步回调的API，定义了自身Future系列接口和类，实现了异步任务的监控、异步执行结果的获取。

​	总体来说，Netty对Java Future异步任务的扩展如下：

1. 继承Java的Future接口，得到一个新的属于Netty自己的Future异步任务接口；该接口对原有的接口进行了增强，使得Netty异步任务能够以非阻塞的方式处理回调的结果。
2. 引入了一个新接口——GenericFutureListener，用于表示异步执行完成的监听器。这个接口和Guava的FutureCallback的回调接口不同。Netty使用了监听器的模式，异步任务的执行结果完成后的回调逻辑抽象成了Listener监听器接口。可以将Netty的GenericFutureListener监听器接口加入Netty异步任务Futrue中，实现对异步任务执行状态的事件监听。

+ Netty的Future接口，可以对应到Guava的ListenableFuture接口
+ Netty的GenericFutureListener接口，可以对应到Guava的FutureCallback接口

#### 5.5.1 详解GenericFutureListener接口

​	前面提到，和Guava的FutureCallback一样，Netty新增了一个接口来封装异步非阻塞回调的逻辑——GenericFutureListener接口

```java
package io.netty.util.concurrent;
import java.util.EventListener;
public interface GenericFutureListener<F extends Future<?>> extends EventListener {
    // 监听器的回调方法
    void operationComplete(F var1) throws Exception;
}
```

​	该回调方法表示异步任务操作完成。在Future异步任务执行完成后，将回调此方法。在大多数情况下，Netty的异步回调代码编写在GenericFutureListener接口的实现类中的operationComplete方法中。

​	GenericFutureListener的父接口EventListener是一个空接口，没有任何的抽象方法，是一个仅仅具有标识作用的接口。

#### 5.5.2 详解Netty的Future接口

​	Netty对Java的Future进行扩展，位于io.netty.util.concurrent包中。

​	和Guava的ListenableFuture一样，Netty的Future接口，扩展一系列的方法，对执行的过程进行监控，对异步回调完成事件进行监听Listen。

```java
public interface Future<V> extends java.util.concurrent.Future<V> {
	boolean isSuccess();	//判断异步执行是否成功
    boolean isCancellable();	//判断异步执行是否取消
    Throwable cause();	//获取异步任务异常的原因
	
    //增加异步任务执行完成与否的监听器Listener
    Future<V> addListener(GenericFutureListener<? extends Future<? super V>> listener);
    //移除异步任务执行完成与否的监听器Listener
    Future<V> removeListener(GenericFutureListener<? extends Future<? super V>> listener);
    // ...
}
```

​	<u>Netty的Future接口一般不会直接使用，而是会使用子接口</u>。Netty有一系列的子接口，代表不同类型的异步任务，如ChannelFuture接口。

​	ChannelFuture子接口表示通道IO操作的异步任务；如果在通道的异步IO操作完成后，需要执行回调操作，就需要使用到ChannelFuture接口。

#### 5.5.3 ChannelFuture的使用

​	**在Netty的网络编程中，网络连接通道的输入和输出处理都是异步进行的**，都会返回一个ChannelFuture接口的实例。通过返回的异步任务实例，可以为它增加异步回调的监听器。在异步任务真正完成后，回调才会执行。

```java
// connect是异步的，仅提交异步任务
ChannelFuture future = bootstap.connect(new InetSocketAddress("www.manning.com",80));

//connect的异步任务真正执行完成后，future回调监听器才会执行
future.addListener(new ChannelFutureListener() {
   @Override
    public void operationComplete(ChannelFuture channelFuture) throws Exception {
        if(channelFuture.isSuccess()){
            System.out.println("Connection established");
        }else {
            System.out.println("Connection attempt failed");
            channelFuture.cause().printStackTrace();
        }
    }
});
```

​	GenericFutureListener接口在Netty中是一个基础类型接口。在网络编程的异步回调中，一般使用Netty中提供的某个子接口，如ChannelFutureListener接口。

#### 5.5.4 Netty的出站和入站异步回调

​	**Netty的出战和入站操作都是异步的**。

​	以最为经典的NIO出站操作——write为例，说明一下ChannelFuture的使用。

​	在调用write操作后，Netty并没完成对Java NIO底层连接的写入操作，因为是异步执行的。

```java
// write输出方法，返回的是一个异步任务
ChannelFuture future = ctx.channel().write(msg);
// 为异步任务，加上监听器
future.addListener(new ChannelFutureListener(){
    @Override
    public void operationComplete(ChannelFuture future){
        // write操作完成后的回调代码
    }
});
```

​	在调用write操作后，是<u>立即返回</u>，返回的是一个ChannelFuture接口的实例。通过这个实例，可以绑定异步回调监听器，这里的异步回调逻辑需要我们编写。

### 5.6 本章小结

​	Guava和Netty的异步回调都是非阻塞的，而Java的join、FutureTask都是阻塞的。

## 第6章 Netty原理与基础

​	Netty是一个JavaNIO客户端/服务器框架。基于Netty，可以快速轻松地开发网络服务器和客户端的应用程序。与直接使用Java NIO相比，Netty能更快速轻松地开发网络服务器和客户端的应用程序。Netty极大地简化了TCP、UDP套接字、HTTPWeb服务程序开发。

​	Netty目标之一，开发做到“快速和轻松”，除了做到“快速和轻松”的开发TCP/UDP等自定义协议的通讯程序之外，Netty经过精心设计，还可以做到“快速和轻松”地开发应用层协议的程序，如FTP、SMTP、HTTP以及其他的传统应用层协议。

​	Netty目标值二，高性能、高可扩展性。基于Java的NIO，Netty设计了一套优秀的Reactor反应器模式。在基于Netty的反应器模式实现中的Channel（通道）、Handler（处理器）等基类，能快速扩展以覆盖不同协议、完成不同业务处理的大量应用类。

### 6.1 第一个Netty的实践案例DiscardServer

#### 6.1.1 创建第一个Netty项目

​	建议使用Netty4.0以上的版本。

#### 6.1.2 第一个Netty服务器端程序

 ```java
public class NettyDiscardServer {
    private final int serverPort;
    ServerBootstrap b = new ServerBootstrap();

    public NettyDiscardServer(int port) {
        this.serverPort = port;
    }

    public void runServer() {
        //创建reactor 线程组
        EventLoopGroup bossLoopGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerLoopGroup = new NioEventLoopGroup();

        try {
            //1 设置reactor 线程组
            b.group(bossLoopGroup, workerLoopGroup);
            //2 设置nio类型的channel
            b.channel(NioServerSocketChannel.class);
            //3 设置监听端口
            b.localAddress(serverPort);
            //4 设置通道的参数
            b.option(ChannelOption.SO_KEEPALIVE, true);
            b.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
            b.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);

            //5 装配子通道流水线
            b.childHandler(new ChannelInitializer<SocketChannel>() {
                //有连接到达时会创建一个channel
                protected void initChannel(SocketChannel ch) throws Exception {
                    // pipeline管理子通道channel中的Handler
                    // 向子channel流水线添加一个handler处理器
                    ch.pipeline().addLast(new NettyDiscardHandler());
                }
            });
            // 6 开始绑定server
            // 通过调用sync同步方法阻塞直到绑定成功
            ChannelFuture channelFuture = b.bind().sync();
            Logger.info(" 服务器启动成功，监听端口: " +
                    channelFuture.channel().localAddress());

            // 7 等待通道关闭的异步任务结束
            // 服务监听通道会一直等待通道关闭的异步任务结束
            ChannelFuture closeFuture = channelFuture.channel().closeFuture();
            closeFuture.sync();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 8 优雅关闭EventLoopGroup，
            // 释放掉所有资源包括创建的线程
            workerLoopGroup.shutdownGracefully();
            bossLoopGroup.shutdownGracefully();
        }

    }

    public static void main(String[] args) throws InterruptedException {
        int port = NettyDemoConfig.SOCKET_SERVER_PORT;
        new NettyDiscardServer(port).runServer();
    }
}
 ```

​	Netty是基于反应器模式实现的。还好，大家已经非常深入地了解反应器模式，现在大家顺藤摸瓜学习Netty的结构就相对简单了。

​	首先要讲的是Reactor反应器。反应器的作用是进行一个IO事件的select查询和dispatch分发。Netty中对应的反应器组件有多种，应用场景不同，用到的反应器也各不相同。一般来说，对应多线程的Java NIO通信的应用场景，**Netty的反应器类型为：NioEventLoopGroup**。

​	在上面的例子中，使用了两个NioEventLoopGroup实例。第一个通常被称为”包工头“，负责服务器通道新连接的IO事件的监听。第二个通常被称为”工人“，主要负责传输通道的IO事件的处理。

​	其次要说的是Handler处理器（也称之为处理程序）。**Handler处理器的作用是对应到IO事件，实现IO事件的业务处理**。Handler处理器需要专门开发。

​	再次，上面的例子中，还用到了**Netty的服务启动类ServerBootstrap，它的职责是一个组装和集成器**，将不同的Netty组件组装到一起。另外，ServerBootstrap能够按照应用场景的需要，为组件设置好对应的参数，最后实现Netty服务器的监听和启动。

#### 6.1.3 业务处理器NettyDiscardHandler

​	**在反应器Reactor模式中，所有业务处理都在Handler处理器中完成**。这里编写一个新类：NettyDiscardHandler。其业务处理即把所有接收到的内容直接丢弃。

```java
public class NettyDiscardHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {

        ByteBuf in = (ByteBuf) msg;
        try {
            Logger.info("收到消息,丢弃如下:");
            while (in.isReadable()) {
                System.out.print((char) in.readByte());
            }
            System.out.println();
        } finally {
            ReferenceCountUtil.release(msg);
        }
    }
}
```

​	首先说明一下，这里将引入一个新的概念：入站和出战。入站指的是输入，出站指的是输出。

​	Netty的Handler处理器需要处理多种IO事件（如可读、可写），对应于不同的IO事件，Netty提供了一些基础的方法。这些方法都已经提前封装好，后面直接继承或者实现即可。比如说，对u有处理入站的IO事件的方法，对应的接口为ChannelInboundHandler入站处理接口，而ChannelInboundHandlerAdapter则是Netty提供的入站处理的默认实现。

​	也就是说，如果要实现自己的入站处理器Handler，只要继承ChannelInboundHandlerAdapter入站处理器，再写入自己的入站处理的业务逻辑。如果要读取入站的数据，只要写在了入站处理方法channelRead中即可。

​	上面例子中的channelRead方法，读取了Netty的输入数据缓冲区ByteBuf。Netty的ByteBuf，可以对应到前面介绍的NIO数据缓冲区。它们在功能上是类似的，不过相对而言，Netty的版本性能更好，使用也更加方便。

#### 6.1.4 运行NettyDiscardServer

​	上面的例子中，出现了Netty的各种组件：**服务器启动器、缓冲区、反应器、Handler业务处理器、Future异步任务监听、数据传输通道等**。这些Netty组件都是需要掌握的。

​	虽然客户端EhoClient客户端是使用Java NIO编写的，而NettyDiscardServer服务端是使用Netty编写的，但是不影响它们之间的相互通信。因为NettyDiscardServer的底层也是使用Java NIO。

### 6.2 解密Netty中的Reactor反应器模式

​	前面的章节已经反复说明：<u>设计模式是Java代码或者程序的重要组织方式，如果不了解设计模式，学习Java程序往往找不到头绪，上下求索而不得其法</u>。故而，在学习Netty组件之前，必须了解Netty中的反应器模式是如何实现的。

​	现在回顾一下JavaNIO中IO事件的处理流程和反应器模式的基础内容。

#### 6.2.1 回顾Reactor反应器模式中IO事件的处理流程

​	channel --> selector --> Reactor --> handler

​	整个流程大致分为4步。具体如下：

​	**第1步：通道注册。IO源于通道（Channel）。IO是和通道（对应于底层连接而言）强相关的。一个IO事件，一定属于某个通道。但是，如果要查询通道的事件，首先要将通道注册到选择器。只需通道提前注册到Selector选择器即可，IO事件会被选择器查询到。**

​	**第2步：查询选择。在反应器模式中，一个反应器（或者SubReactor子反应器）会负责一个线程；不断地轮询，查询选择器中的IO事件（选择键）。**

​	**第3步：事件分发。如果查询到IO事件，则分发给与IO事件有绑定关系的Hanlder业务处理器。**

​	**第4步：完成真正的IO操作和业务处理，这一步由Handler业务处理器负责。**

其中，第1步和第2步，其实是Java NIO的功能，反应器模式仅仅是利用了Java NIO的优势而已。

#### 6.2.2 Netty中的Channel通道组件

​	Channel通道组件是Netty中非常重要的组件。反应器模式和通道紧密相关，反应器的查询和分发的IO事件都来自于Channel通道组件。

​	Netty中不直接使用Java NIO的Channel通道组件，对Channel通道组件进行了自己的封装。在Netty中，由一系列的Channel通道组件，为了支持多种通道协议，对于每一种通道连接协议，Netty都实现了自己的通道。

​	另外一点就是，**除了Java的NIO，Netty还能处理Java的面向流的OIO（Old-IO，即传统的阻塞式IO）**。

​	总结起来，**Netty中的每一种协议的通道，都有NIO（异步IO）和OIO（阻塞式IO）两个版本**。

​	对应不同的协议，Netty中常见的通道类型如下：

+ NioSocketChannel：异步非阻塞TCP Socket传输通道。

+ NioServerSocketChannel：异步非阻塞TCP Socket服务器端监听通道。
+ NioDatagramChannel：异步非阻塞的UDP传输通道。
+ NioSctpChannel：异步非阻塞Stcp传输通道。
+ NioSctpServerChannel：异步非阻塞Stcp服务器端监听通道。
+ OioSocketChannel：同步阻塞式TCP Socket传输通道。
+ OioServerSocketChannel：同步阻塞式TVP Socket服务器端监听通道。
+ OioDatagramChannel：同步阻塞式UDP传输通道。
+ OioSctpChannel：同步阻塞式Sctp传输通道。
+ OioSctpServerChannel：同步阻塞式Sctp服务器端监听通道。

> 自己查的一些资料网文
>
> [SCTP协议详解](https://blog.csdn.net/wuxing26jiayou/article/details/79743683)
>
> [为什么SCTP没有被大量使用/知道？](https://cloud.tencent.com/developer/ask/28242)

​	一般来说，服务器端编程用到最多的通信协议还是TCP协议。对应的传输通道类型为NioSocketChannel类，服务器监听类为NioSerevrSocketChannel。在主要使用的方法上，其他的通道类型和这个NioSocketChannel类在原理上基本是相通的。本书很多案例以NioSocketChannel通道为主。

​	在Netty的NioSocketChannel内部封装了一个SelectableChannel成员。<u>通道这个内部的Java NIO通道，Netty的NioSocketChannel通道上的IO操作，最终会落到Java NIO的SelectableChannel底层通道</u>。

#### 6.2.3 Netty中的Reactor反应器

​	在反应器模式中，一个反应器（或者SubReactor子反应器）会负责一个事件处理线程，不断地轮询。通过Selector选择器不断查询注册过的IO事件（选择键）。如果查询到IO事件，则分发给Handler业务处理器。

​	**Netty中的反应器有多个实现类，与Channel通道类有关系**。对应于NioSocketChannel通道，Netty的反应器为：NioEventLoop。

​	NioEventLoop类绑定了两个重要的Java成员属性：一个是Thread线程类的成员，一个是Java NIO选择器的成员属性。

​	一个NioEventLoop拥有一个Thread线程，负责一个Java NIO Selector选择器的IO事件轮询。

​	在Netty中，EventLoop反应器和NettyChannel通道是一对多关系：一个反应器可以注册成千上万的通道。

#### 6.2.4 Netty中的Handler处理器

​	可供选择器监控的IO事件类型包括以下4种：

+ 可读：SelectionKey.OP_READ
+ 可写：SelectionKey.OP_WRITE
+ 连接：SelectionKey.OP_CONNECT
+ 接收：SelectionKey.OP_ACCEPT

​	在Netty中，EventLoop反应器内部有一个Java NIO选择器成员执行以上的事件的查询，然后进行对应的事件分发。事件分发（Dispatch）的目标就是Netty自己的Handler处理器。

​	Netty的Handler处理器分为两大类：第一类是ChannelInboundHandler通道入站处理器；第二类是ChannelOutboundHandler通道出站处理器。二者都继承了ChannelHandler处理器接口。

​	Netty中的入站处理，不仅仅是OP_READ输入事件的处理，还是从通道底层出触发，由Netty通过层层传递，调用ChannelInboundHandler通道入站处理器进行的某个处理。以底层的Java NIO中的OP_READ输入事件为例：在通道中发生了OP_READ事件后，会被EventLoop查询到，然后分发给ChannelInboundHandler通道入站处理器，调用它的入站处理的方法read。在ChannelInboundHandler通道入站处理器内部的read方法可以从通道中读取数据。

​	**OP_WRITE可写事件是Java NIO的底层概念，它和Netty的出站处理的概念不是一个维度，Netty的出站处理时应用层维度的**。Netty的出站处理，指的是从ChannelOutboundHandler通道出站处理器到通道的某次IO操作，例如，在应用程序完成业务处理后，可以通过ChannelOutboundHandler通道处理器将处理的结果写入底层通道。它的最常用的一个方法就是write()方法，把数据写入到通道。

​	这两个业务处理器接口都有各自的默认实现：xxxxAdapter（通道入站/出战处理适配器）。

#### 6.2.5 Netty的流水线（Pipeline）

​	梳理Netty的反应器模式中各个组件之间的关系：

1. 反应器（或者SubReactor子反应器）和通道之间是一对多关系：一个反应器可以查询很多个通道的IO事件。
2. 通道和Handler处理器实例，是多对多关系：一个通道的IO事件被多个的Handler实例处理；一个Handler处理器实例也能绑定到很多的通道，处理多个通道的IO事件。

​	为了处理通道和Handler处理器实例之间的绑定关系，Netty设计了一个特殊的组件，叫作ChannelPipeline（通道流水线），它像一条管道，将绑定到一个通道的多个Handler实例，串在一起，形成一条流水线。**ChannelPipeline（通道流水线）的默认实现，实际上被设计成一个双向链表。所有的Handler处理器实例被包装成了双向链表的结点，加入到ChannelPipeline（通道流水线）中**。

​	重点申明：**一个Netty通道拥有一条Handler处理器流水线，成员的名称叫作pipeline**。

​	以入站处理为例。每个来自通道的IO事件，都会进入一次ChannelPipeline通道流水线。在进入第一个Handler处理器后，这个IO事件将按照**既定**的从前往后次序，在流水线上不断地向后流动，流向下一个Handler处理器。

​	在向后流动的过程中，会出现3种情况：

1. 如果后面还有其他Handler入站处理器，那么IO事件可以交给下一个Handler处理器，向后流动。
2. 如果后面没有其他的入站处理器，这就意味着这个IO事件在此流水线中的处理结束了。
3. 如果在流水线中间需要终止流动，可以选择不将IO事件交给下一个Handler处理器，流水线的执行也被终止了。

​	为什么说Handler处理是按照既定的次序，而不是从前往后的次序？**Netty规定：入站处理器Handler的执行次序，是从前往后；出站处理器Handler的执行次序，是从后到前**。

​	入站的IO操作只能从Inbound入站处理器类型的Handler流过；出站的IO操作只能从Outbound出战处理器类型的Handler流过。

​	为了开发方便，Netty提供了一个类把三个组件（EventLoop反应器、通道、Handler处理器）快速组装起来。这个系列的类叫作BootStrap启动器。严格来说，不止一个类名字为BootStrap，例如在服务器端的启动类叫作ServerBootstrap类。

### 6.3 详解BootStrap启动器类

​	Bootstrap类是Netty提供的一个遍历的工厂类，可以通过它来完成Netty的客户端或服务器端的Netty组件的组装，以及Netty程序的初始化。Netty官方解释是，完全可以不使用Bootstrap启动器。不过一点点手动创建通道、完成各种设置和启动、并且注册到EventLoop，这个过程会非常麻烦。通常情况下，还是使用这个便利的Bootstap工具类会效率更高。

​	Netty中，有两个启动类，分别表示在服务器端和客户端。（Bootstrap是client专用，ServerBootstrap是server专用）。它们的配置和使用都是相同的。

​	在介绍ServerBootstrap的服务器启动流程之前，首先介绍一下涉及到的两个基础概念：父子通道、EventLoopGroup线程组（事件循环线程组）。

#### 6.3.1 父子通道

​	在Netty中，每一个NioSocketChannel通道所封装的是Java NIO通道，再往下就对应到了操作系统底层的socket描述符。**理论上讲，操作系统底层的socket描述符分为两类**：

1. **连接监听类型**。连接监听类型的socket描述符，放在服务器端，它负责接收客户端的套接字连接；在服务器端，**一个“连接监听类型”的socket描述符可以接受（Accept）成千上万的传输类的socket描述符**。
2. **传输数据类型**。数据传输类的socket描述符负责传输数据。同一条TCP的Socket传输链路，在服务器端和客户端，都分别会有一个与之对应的数据传输类型的socket描述符。

​	在Netty中，异步非阻塞的服务器端监听通道NioServerSocketChannel，封装在Linux底层的描述符，是“连接监听类型”socket描述符；而NioSocketChannel异步非阻塞TCP Socket传输通道，封装在底层Linux的描述符，是“数据传输类型”的socket描述符。

​	在Netty中，将有接收关系的NioServerSocketChannel和NioSocketChannell，叫作父子通道。其中，**NioServerSocketChannel负责服务器连接监听和接收，也叫做父通道（Parent Channel）。对应于每一个接收到的NioSocketChannel传输类通道，而叫做子通道（Child Channel）**。

#### 6.3.2 EventLoopGroup线程组

​	Netty中的Reactor反应器模式，肯定不是单线程版本的反应器模式，而是多线程版本的反应器模式。

​	在Netty中，一个EventLoop相当于一个子反应器（SubReactor）。一个NioEventLoop子反应器拥有了一个线程，同时拥有一个Java NIO选择器。**Netty使用EventLoopGroup线程组组织外层的反应器，多个EventLoop线程组成一个EventLoopGroup线程组**。

​	反过来说，Netty的EventLoopGroup线程组就是一个多线程版本的反应器。而其中的单个EventLoop线程对应一个子反应器（SubReactor）。

​	Netty程序开发往往不会直接用单个EventLoop线程，而是使用EventLoopGroup线程组。EventLoopGroup的构造函数有一个参数，用于指定内部的线程数。在构造器初始化时，会按照传入的线程数量，在内部构造多个Thread线程和多个EventLoop子反应器（一个线程对应一个EventLoop子反应器），进行多线程的IO事件查询和分发。

​	如果使用EventLoopGroup的无参数的构造函数，没有传入线程数或者传入的线程数为0，那么EventLoopGroup内部的线程数是多少？**默认的EventLoopGroup内部线程数为最大可用CPU处理器数量的2倍**。假设电脑使用的是4核的CPU，那么在内部会启动8个EventLoop线程，相当于8个子反应器（SubReactor）实例。

​	从前文可知，为了及时接受（Accept）到新连接，**在服务器端，一般会有两个独立的反应器，一个反应器负责新连接的监听和接受，另一个反应器负责IO事件处理**。对应到Netty服务器程序中，则是设置两个EventLoopGroup线程组，一个EventLoopGroup负责新连接的监听和接受，一个EventLoopGroup负责IO事件处理。

​	负责新连接的监听和接受的EventLoopGroup线程组，查询父通道的IO事件，有点像负责招工的包工头，因此可以形象地称为“包工头”(Boss)线程组。另一个EventLoopGroup线程组负责查询所有子通道的IO事件，并且执行Handler处理器中的业务处理——例如数据的输入和输出（有点像搬砖），这个线程组可以形象地称为“工人”（Worker）线程组。

​	至此，已经介绍完两个基础概念：父子通道、EventLoopGroup线程组。

#### 6.3.3 Bootstrap的启动流程

​	<u>Bootstrap的启动流程，也就是Netty组件的组装、配置，以及Netty服务器或者客户端的启动流程</u>。在本节中对启动流程进行了梳理，大致分成了8步骤。本书仅仅演示的是服务器端启动器的使用，用到的启动类为ServerBootstrap。正式使用之前，首先创建一个服务器端的启动器实例。

```	java
// 创建一个服务器端的启动器
ServerBootstrap b = new ServerBootstrap();
```

​	接下来，结合前面的NettyDiscardServer服务器的代码，给大家详细介绍一下Bootstrap启动流程中精彩的8个步骤。

​	第1步：创建反应器线程组，并赋值给ServerBootstrap启动器实例

```java
// 创建反应器线程组
// boss线程组
EventLoopGroup bossLoopGroup = new NioEventLoopGroup(1);
// worker线程组
EventLoopGroup workerLoopGroup = new NioEventLoopGroup();
// ...
// 1 设置反应器线程组
b.group(bossLoopGroup, workerLoopGroup);
```

​	第2步：设置通道的IO类型

​	Netty不止支持Java NIO，也支持阻塞式的OIO（也叫BIO，Block-IO，即阻塞式IO）。下面配置的是Java NIO类型的通道类型，方法如下：

```java
// 2 设置nio类型的通道
b.channel(NioServerSocketChannel.class);
```

​	如果确实需要指定Bootstrap的IO模型为BIO，那么这里配置上Netty的OioServerSocketChannel.class类即可。由于NIO的优势巨大，通常不会在Netty中使用BIO。

​	第3步：设置监听端口

```java
// 3 设置监听端口
b.localAddress(new InetSocketAddress(port));
```

​	这是最为简单的一步，主要是设置服务器的监听地址。

​	第4步：设置传输通道的配置选项

```java
// 4 设置通道的参数
b.option(ChannelOption.SO_KEEPALIVE, true);
b.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
```

​	这里用到了Bootstrap的option()选项设置方法。对于服务器的Bootstrap而言，这个方法的作用是：<u>给父通道（Parent Channel）接收连接通道设置一些选项</u>。

​	<u>如果要给子通道（Child Channel）设置一些通道选项，则需要用另一个childOption()设置方法</u>。

​	可以设置哪些通道选项（ChannelOption）呢？在上面的代码中，设置了一些底层TCP相关的选项ChannelOption.SO_KEEPALIVE。该选项表示：是否开启TCP底层心跳机制，true为开启，false为关闭。

​	第5步：装配子通道的Pipeline流水线

​	上一节介绍到，每一个通道的子通道，都用一条ChannelPipeline流水线。它的内部有一个双向的链表。装配流水线的方式是：将业务处理器ChannelHandler实例加入双向链表中。

​	装配子通道的Handler流水线调用childHandler()方法，传递一个ChannelInitializer通道初始化类的实例。**在父通道成功接收一个连接，并创建成功一个子通道后，就会初始化子通道，这里配置的ChannelInitializer实例就会被调用**。

​	<u>在ChannelInitializer通道初始化类的实例中，有一个initChannel初始化方法，在子通道创建后会被执行到，向子通道流水线增加业务处理器</u>。

```java
// 5 装配子通道流水线
b.childHandler(new ChannelInitializer<SocketChannel>() {
    // 有连接到达时会创建一个通道的子通道，并初始化
    protected void initChannel(SocketChannel ch) throws Exception {
        // 流水线管理子通道中的Handler业务处理器
        // 向子通道流水线添加一个Handler业务处理器
        ch.pipeline().addLast(new NettyDiscardHandler());
    }
})
```

​	为什么仅装配子通道的流水线，而不需要装配父通道的流水线呢？原因是：父通道也就是NioServerSocketChannel连接接受通道， 它的内部业务处理是固定的：**接受新连接后，创建子通道，然后初始化子通道**，所以不需要特别的配置。<u>如果需要完成特殊的业务处理，可以使用ServerBootstrap的handler（ChannelHandler handler）方法，为父通道设置ChannelInitializer初始化器</u>。

​	<u>说明一下，ChannelInitializer处理器有一个泛型参数SocketChannel，它代表需要初始化的通道类型，这个类型需要和前面的启动器中设置的通道类型，一一对应起来</u>。

​	第6步：开始绑定服务器新连接的监听端口

```java
// 6 开始绑定端口，通过调用sync同步阻塞直到绑定成功
ChannelFuture channelFuture = b.bind().sync();
Logger.info(" 服务器启动成功，监听端口： " + channelFuture.channel().localAddress());
```

​	这个也很简单。b.bind()方法的功能：返回一个端口绑定Netty的异步任务channelFuture。在这里，并没有给channelFuture异步任务增加回调监听器，而是阻塞channelFuture异步任务，直到端口绑定任务执行完成。

​	**在Netty中，所有的IO操作都是异步执行的**，这就意味着任何一个IO操作会立刻返回，在返回的时候，异步任务还没有真正执行。什么时候执行完成呢？**Netty中的IO操作，都会返回异步任务实例（如ChannelFuture实例），通过自我阻塞一直到ChannelFuture异步任务执行完成，或者为ChannelFuture增加事件监听器的两种方式，以获得Netty中的IO操作的真正结果**。上面使用了第一种。

​	至此，服务器正式启动。

​	第7步：自我阻塞，直到通道关闭

```java
// 7 等待通道关闭
// 自我阻塞，直到通道关闭的异步任务结束
ChannelFuture closeFuture = channelFuture.channel().closeFuture();
closeFuture.sync();
```

​	如果要阻塞当前线程直到通道关闭，可以使用通常的closeFuture()方法，以获得通道关闭的异步任务。当通道被关闭时，closeFuture实例的sync()方法会返回。

​	第8步：关闭EventLoopGroup

```java
// 8 关闭EventLoopGroup
// 释放掉所有资源，包括创建的反应器线程
workerLoopGroup.shutdownGracefully();
bossLoopGroup.shutdownGracefully();
```

​	<u>关闭Reactor反应器线程组，同时会关闭内部的subReactor子反应器线程，也会关闭内部的Selector选择器、内部的轮询线程以及负责查询的所有的子通道。在子通道关闭后，会释放掉底层的资源，如TCP Socket文件描述符等</u>。

#### 6.3.4 ChannelOption通道选项

​	无论对对于NioServerSocketChannel父通道类型，还是对于NioSocketChannel子通道类型，都可以设置一系列的ChannelOption选项。在ChannelOption类中，定义了一大票选项，下面介绍一些常见的选项。

1. SO_RCVBUF，SO_SNDBUF

   ​	此为TCP参数。**每个TCP socket（套接字）在内核中都有一个发送缓冲区和一个接收缓冲区**，这个两个选项就是用来设置TCP连接的这两个缓冲区大小的。TCP的全双工的工作模式以及<u>TCP的滑动窗口</u>便是依赖于这两个两个独立的缓冲区及其填充的状态。

   > [解析TCP之滑动窗口(动画演示)](https://blog.csdn.net/yao5hed/article/details/81046945)
   >
   > [TCP的滑动窗口与拥塞窗口](https://blog.csdn.net/ligupeng7929/article/details/79597423)

2. TCP_NODELAY

   ​	此为TCP参数。表示立即发送数据，默认值为True（Netty默认为True，而操作系统默认为False）。该值用于设置**Nagle算法的启用，该算法将小的碎片数据连接成更大的报文（或数据包）来最小化所发送报文的数量，如果需要发送一些较小的报文，则需要禁用该算法**。<u>Netty默认禁用该算法，从而最小化报文传输的延时</u>。

   ​	说明一下：这个参数的值，与是否开启Nagle算法是相反的，设置为true表示关闭，设置为false表示开启。通俗地讲，如果要求高实时性，有数据发送时就立刻发送，就设置为true，如果需要减少发送次数和减少网络交互次数，就设置为false

   > [结合RPC框架通信谈 netty如何解决TCP粘包问题(即使关闭nagle算法，粘包依旧存在)](https://www.cnblogs.com/yuanjiangw/p/9954079.html)

3. SO_KEEPALIVE

   ​	此为TCP参数。表示**底层TCP协议的心跳机制**。true为连接保持心跳，默认值为false。启用该功能时，TCP会主动探测空闲连接的有效性。可以将此功能视为TCP的心跳机制，需要注意的是：<u>默认的心跳间隔是7200s即2小时。Netty默认关闭该功能</u>。

4. SO_REUSEADDR

   ​	此为TCP参数。设置为true时表示地址复用，默认值为false。有四种情况需要用到这个参数设置：

   + 当有一个有相同本地地址和端口的socket1处于TIME_WAIT状态时，而我们希望启动的程序的socket2要占用该地址和端口。例如在重启服务且保持先前的端口时。
   + 有多块网卡或用IP Alias技术的机器在同一端口启动多个进程，但每个进程绑定的本地IP地址不能相同。
   + 单个进程绑定相同的端口到多个socket（套接字）上，但每个socket绑定的IP地址不同。
   + 完全相同的地址和端口的重复绑定。但这只用于UDP的多播，不用于TCP。

5. SO_LINGER

   ​	此为TCP参数。表示关闭socket的延时事件，默认值为-1，表示禁用该功能。-1表示socket.close()方法立即返回，但操作系统底层会将发送缓冲区全部发送到对端。 0表示socket.close()方法立即返回，操作系统放弃发送缓冲区的数据，直接向对端发送**RST包**，对端收到复位错误。非0整数值表示调用socket.close()方法的线程被阻塞，直到延迟时间到来、发送缓冲区中的数据发送完毕，若超时，则对端会收到复位错误。

   > [TCP连接异常终止（RST包）](https://blog.csdn.net/hik_zxw/article/details/50167703)

6. SO_BACKLOG

   ​	此为TCP参数。<u>表示服务器接收连接的队列长度，如果队列已满，客户端连接池将被拒绝</u>。默认值，在Windows中为200，其他操作系统为128。

   ​	**如果连接建立频繁，服务器处理新连接较慢，可以适当调大这个参数**。

7. SO_BROADCAST

   ​	此为TCP参数。表示设置广播模式

### 6.4 详解Channel通道

​	先介绍下，在使用Channel通道的过程中所设涉及的主要成员和方法。然后，为大家介绍一下Netty所提供了一个专门的单元测试通道——EmbededChannel（嵌入式通道）。

#### 6.4.1 Channel通道的主要成员和方法

​	在Netty中，通道是其中的核心概念之一，代表着网络连接。通道是通信的主题，由它负责同对端进行网络通信，可以写入数据到对端，也可以从对端读取数据。

​	通道的抽象类AbstractChannel的构造函数如下：

```java
protected AbstractChannel(Channel parent) {
    this.parent = parent; // 父通道
    id = newId();
    unsafe = new Unsafe();// 底层的NIO通道，完成实际的IO操作
    pipeline = new ChannelPipeline(); // 一条通道，拥有一条流水线
}
```

​	AbstractChannel内部有一个pipeline属性，表示处理器的流水线。Netty在对通道进行初始化的时候，将pipeline属性初始化为DefaultChannelPipeline的实例。这段代码也表明，每个通道拥有一条ChannelPipeline处理器流水线。

​	AbstractChannel内部有一个parent属性，表示通道的父通道。对于连接监听通道（如NioServerSocketChannel实例）来说，其父亲通道为null；而对于每一条传输通道（如NioSocketChannel实例），其parent属性的值为接收到该连接的服务器连接监听通道。

​	**几乎所有的通道实现类都继承了AbstractChannel抽象类，都拥有上面的parent和pipeline两个属性成员**。

​	再来看一下，在通道接口中定义的几个重要方法：

+ 方法1：ChannelFuture connect(SocketAddress address)

  ​	此方法的作用为：连接远程服务器。方法的参数为远程服务器的地址<u>，调用后会立即返回</u>，返回值为负责连接操作的异步任务ChannelFuture。此方法在客户端的传输通道使用。

+ 方法2：ChannelFuture bind(SocketAddress address)

  ​	此方法的作用为：绑定监听地址，开始监听新的客户端连接。此方法在服务器的新连接监听和接收通道使用。

+ 方法3：ChannelFuture close()

  ​	此方法的作用为：关闭通道连接，返回连接关闭的ChannelFuture异步任务。**如果需要在连接正式关闭后执行其他操作，则需要为异步任务设置回调方法；或者调用ChannelFuture异步任务的sync()方法来阻塞当前线程，一直等到通道关闭的异步任务执行完毕**。

+ 方法4：ChannelFuture read()

  ​	此方法的作用为：读取通道数据，并且启动入站处理。具体来说，从内部的Java NIO Channel通道读取数据，然后启动内部的Pipeline流水线，开始数据读取的入站处理。此方法的返回通道自身用于链式调用。

+ 方法5：ChannelFuture write(Object o)

  ​	此方法的作用为：启动出站流水处理，把处理后的最终数据写到底层Java NIO通道。此方法的返回值为出站处理的异步处理任务。

+ 方法6：Channel flush()

  ​	此方法的作用为：将缓冲区中的数据立即写出到对端。并不是每一次write操作都是将数据直接写出到对端，**write操作的作用在大部分情况下仅仅是写入到操作系统的缓冲区，操作系统会将根据缓冲区的情况，决定什么时候把数据写到对端**。而**执行flush()方法立即将缓冲区的数据写到对端**。

​	上面6种方法，仅仅是比较常见的方法。在Channel接口以及各种通道的实现类中，还定义了大量的通道操作方法。在一般的日常的开发中，如果需要用到，最好直接查阅Netty API文档或者Netty源代码。

#### 6.4.2 EmbeddedChannel嵌入式通道

​	在Netty的实际开发中，通信的基础工作，Netty已经替大家完成。<u>实际上，大量的工作是设计和开发ChannelHandler通道业务处理器，而不是开发Outbound出战处理器，换句话说就是开发Inbound入站处理器</u>。开发完成后，需要投入单元测试。单元测试的大致流程：需要将Handler业务处理器加入到通道的Pipeline流水线中，接下来先后启动Netty服务器、客户端程序，相互发送消息，测试业务处理器的效果。如果每开发一个业务处理器，都进行服务器和客户端的重复启动，这整个过程是非常的烦琐和浪费时间的。

​	Netty提供了一个专用的通道——EmbeddedChannel（嵌入式通道），来解决这种徒劳、低效的重复工作。

​	**EmbeddedChannel仅仅是模拟入站和出站的操作，底层不进行实际的传输，不需要启动Netty服务器和客户端。除了不进行传输之外，EmbeddedChannel的其他的事件机制和处理流程和真正的传输通道是一模一样的**。因此，使用它，开发人员可以在开发的过程中方便、快速地进行ChannelHnadler业务处理器的测试。

​	为了模拟数据的发送和接收，EmbeddedChannel提供了一组专门的方法

| 名称                  | 说明                                                         |
| --------------------- | ------------------------------------------------------------ |
| **writeInbound**(...) | 向通道写入inbound入站数据，模拟通道收到数据。也就是说，这些写入的数据会被流水线上的入站处理器处理 |
| readInbound(...)      | 从EmbeddedChannel中读取入站数据，返回经过流水线最后一个入站处理器处理完成之后的入站数据。如果没有数据，则返回null |
| writeOutbound(...)    | 向通道写入outbound出站数据，模拟通道发送数据。也就是说，这些写入的数据会被流水线上的出站处理器处理 |
| **readOutbound**(...) | 从EmbeddedChannel中读取出站数据，返回经过流水线最后一个出站处理器处理之后的出站数据。如果没有数据，则返回null |
| finish()              | 结束EmbeddedChannel，它会调用通道close方法                   |

​	最为重要的两个方法为：writeInbound和readOutbound方法。

+ 方法1：writeInbound入站数据写到通道

  ​	它的使用场景是：测试入站处理器。在测试入站处理器时（例如一个解码器），需要读取Inbound（入站）数据。可以调用writeInbound方法，向EmbeddedChannel写入一个入站而二进制ByteBuf数据包，模拟底层的入站包。

+ 方法2：readOutbound读取通道的出站数据

  ​	它的使用场景是：测试出站处理器。在测试出站处理器时（例如测试一个编码器），需要查看处理过的结果数据。可以调用readOutbound方法，读取通道的最终出站结果，它是经过流水线一系列的出站处理后，最终的出站数据包。重复一遍，通过readOutbound，可以读取完成EmbeddedChannel最后一个出站处理器，处理后的ByteBuf二进制出站包。

​	总之，EmbeddedChannel类，即具备通道的通用接口和方法，又增加了一些单元测试的辅助方法，在开发时非常实用。

> [Netty 使用 EmbeddedChannel 进行单元测试](https://blog.csdn.net/al_assad/article/details/79472347)
> [netty使用EmbeddedChannel对channel的出入站进行单元测试](https://www.jianshu.com/p/1573e81e655d)

### 6.5 详解Handler业务处理器

​	在Reactor反应器经典模式中，反应器查询到IO事件后，分发到Handler业务处理器，由Handler完成IO操作和业务处理。

​	**整个IO处理操作环节包括：从通道读取数据包、数据包解码、业务处理、目标数据编码、把数据包写到通道，然后由通道发送到对端**。

​	前后两个环节，**<u>从通道读数据包</u>和<u>由通道发送到对端</u>，由Netty的底层负责完成，不需要用户程序负责**。

​	用户程序主要在Handler业务处理器中，Handler涉及的环节为：**数据包解码、业务处理、目标数据编码、把数据包写到通道中**。

​	前面已经介绍过，从应用程序开发人员的角度来看，有入站和出站两种类型操作。

+ 入站处理，触发的方向为：自底向上，Netty的内部（如通道）到ChannelInboundHandler入站处理器。
+ 出站处理，触发的方向为：自顶向下，从ChannelOutboundHandler出站处理器到Netty的内部（如通道）。

​	按照这种方向来分，前面数据包解码、业务处理两个环节——属于入站处理器的工作；后面目标数据编码、把数据包写到通道中两个环节——属于出站处理器的工作。

#### 6.5.1 ChannelInboundHandler通道入站处理器

​	当数据或者信息入站到Netty通道时，Netty将触发入站管理器ChannelInboundHandler所对应的入站API，进行入站操作处理。

​	ChannelInboundHandler主要操作：

1. channelRegistered

   ​	当通道注册完成后，Netty会调用fireChannelRegistered，触发通道注册事件。通道会启动该入站操作的流水线处理，在通道注册过的入站处理器Handler的channelRegitstered方法，会被调用到。

2. channelActive

   ​	当通道激活完成后，Netty会调用fireChannelActive，触发通道激活事件。通常会启动该入站操作的流水线处理，在通道注册过的入站处理器Handler的channelActive方法，会被调用到。

3. channelRead

   ​	当通道缓冲区可读，Netty会调用fireChannelRead，触发通道可读事件。通道会启动该入站操作的流水线处理，在通道注册过的入站处理器Handler的channelRead方法，会被调用到。

4. channelReadComplete

   ​	当通道缓冲区读完，Netty会调用fireChannelReadComplete，触发通道读完事件。通道会启动该入站操作的流水线处理，在通道注册过的入站处理器Handler的channelReadComplete方法，会被调用到。

5. channelInactive

   ​	当连接被断开或者不可用，Netty会调用fireChannelInactive，触发连接不可用事件。通道会启动对应的流水线处理，在通道注册过的入站处理器Handler的channelInactive方法，会被调用到。

6. exceptionCaught

   ​	当通道处理过程发生异常时，Netty会调用fireExceptionCaught，触发异常捕获事件。通道会启动异常捕获的流水线处理，在通道注册过的处理器Handler的exceptionCaught方法，会被调用到。注意，<u>这个方法是在通道处理器中ChannelHandler定义的方法，入站处理器、出战处理器接口都继承到了该方法</u>。

​	上面仅介绍了ChannelInboundHandler的其中几个比较重要的方法。在Netty中，它的默认实现为ChannelInboundHandlerAdapter，在实际开发中，只需要继承这个ChannelInboundHandlerAdapter默认实现，重写自己需要的部分即可。

#### 6.5.2 ChannelOutboundHandler通道出站处理器

​	当业务处理完成后，需要操作Java NIO底层通道时，通过一系列的ChannelOutboundHandler通道出站处理器，完成Netty通道到底层通道的操作。比方说建立底层连接、断开底层连接、写入底层Java NIO 通道等。ChannelOutboundHandler接口定义了大部分的出站操作。

​	再强调一下，**出站处理的方向：是通过上层Netty通道，去操作底层Java IO通道**。

​	主要出站（Outbound）的操作如下：

1. bind

   ​	监听地址（IP+端口）绑定：完成底层Java IO通道的IP地址绑定。如果使用TCP传输协议，这个方法用于服务器端。

2. connect

   ​	连接服务器：完成底层 Java IO通道的服务器端的连接操作。如果使用TCP传输协议，这个方法用于客户端。

3. write

   ​	写数据到底层：完成Netty通道向底层Java IO通道的数据写入操作。<u>此方法仅仅是触发一下操作而已，并不是完成实际的数据写入操作</u>。

4. flush

   ​	腾空缓冲区中的数据，把这些数据写到对端：将底层缓冲区的数据腾空，立即写出到对端。

5. read

   ​	从底层读数据：完成Nettu通道从java IO通道的数据读取。

6. disConnect

   ​	断开服务器连接：断开底层Java IO通道的服务器端连接。如果使用TCP传输协议，此方法主要用于客户端。

7. close

   ​	主动关闭通道：关闭底层的通道，例如服务器端的新连接监听通道。

​	上面仅介绍了ChannelOutboundHandler的其中几个比较重要的方法。在Netty中，它的默认实现为ChannelOutboundHandlerAdapter，在实际开发中，只需要继承这个ChannelOutboundHandlerAdapter默认实现，重写自己需要的部分即可。

#### 6.5.3 ChannelInitializer通道初始化处理器

​	通道和Handler业务处理器的关系：一条Netty的通道拥有一条Handler业务处理器流水线，负责装配自己的Handler业务处理器。装配Handler的工作，发生在通道开始工作之前。现在的问题是：如果向流水线中装配业务处理器呢？这就得借助通道的初始化类——ChannelInitializer。

​	首先回顾NettyDiscardServer丢弃服务端的代码，在给接收到的新连接搭配Handler业务处理器时，使用childHandler设置了一个ChannelInitializer实例：

```java
// 5 装配子通道流水线
b.childHandler(new ChannelInitializer<SocketChannel>() {
    // 有连接到达时会创建一个通道
    protected void initChannel(SocketChannel ch)throws Exception {
        // 流水线管理子通道中的Handler业务处理器
        // 向子通道流水线添加一个Handler业务处理器
        ch.pipeline().addLast(new NettyDiscardHandler());
    }
});
```

​	上面的ChannelInitializer也是通道初始化器，属于入站处理器的类型。在示例代码中，使用了ChannelInitializer的initChannel()方法。

​	initChannel()方法是ChannelInitailizer定义的一个抽象方法，这个抽象方法需要开发人员自己实现。<u>在父通道调用initChannel()方法时，会将新接收的通道作为参数，传递给initChannel()方法</u>。**initChannel()方法内部法制的业务代码是：拿到新连接通道作为实际参数，往它的流水线中装配Handler业务处理器**。

#### 6.5.4 ChannelInboundHandler的生命周期的实践案例

​	为了弄清Handler业务处理器的各个方法的执行顺序和生命周期，这里定义一个简单的入站Handler处理器——InHandlerDemo。

```java
public class InHandlerDemo extends ChannelInboundHandlerAdapter {
    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        Logger.info("被调用：handlerAdded()");
        super.handlerAdded(ctx);
    }

    @Override
    public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
        Logger.info("被调用：channelRegistered()");
        super.channelRegistered(ctx);
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        Logger.info("被调用：channelActive()");
        super.channelActive(ctx);
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        Logger.info("被调用：channelRead()");
        super.channelRead(ctx, msg);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        Logger.info("被调用：channelReadComplete()");
        super.channelReadComplete(ctx);
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        Logger.info("被调用：channelInactive()");
        super.channelInactive(ctx);
    }

    @Override
    public void channelUnregistered(ChannelHandlerContext ctx) throws Exception {
        Logger.info("被调用: channelUnregistered()");
        super.channelUnregistered(ctx);
    }

    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
        Logger.info("被调用：handlerRemoved()");
        super.handlerRemoved(ctx);
    }
}
```

​	为了演示这个入站处理器，需要编写一个单元测试代码：将上面的Inhandler入站处理器加入到一个EmbeddedChannel嵌入式通道的流水线中。接着，通过writeInbound方法写入ByteBuf数据包。InHandlerDemo作为一个入站处理器，会处理从通道到流水线的入站报文——ByteBuf数据包。单元测试代码如下：

```java
public class InHandlerDemoTester {
    @Test
    public void testInHandlerLifeCircle() {
        final InHandlerDemo inHandler = new InHandlerDemo();
        //初始化处理器
        ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
            protected void initChannel(EmbeddedChannel ch) {
                ch.pipeline().addLast(inHandler);
            }
        };
        //创建嵌入式通道
        EmbeddedChannel channel = new EmbeddedChannel(i);
        ByteBuf buf = Unpooled.buffer();
        buf.writeInt(1);
        //模拟入站，写一个入站包
        channel.writeInbound(buf);
        channel.flush();
        //模拟入站，再写一个入站包
        channel.writeInbound(buf);
        channel.flush();
        //通道关闭
        channel.close();
        try {
            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

​	运行上面的测试用例，输出的结果具体如下：

```java
被调用：handlerAdded()
被调用：channelRegistered()
被调用：channelActive()
被调用：channelRead()
被调用：channelReadComplete()
被调用：channelRead()
被调用：channelReadComplete()
被调用：channelInactive()
被调用: channelUnregistered()
被调用：handlerRemoved()
```

​	其中，读数据的入站回调为：channelRead()->channelReadComplete()；入站方法会多次调用，每一次有ByteBuf数据包入站都会调用到。

​	除两个入站回调方法外，其余6个方法都和ChannelHandler生命周期有关。

1. handlerAdded()：当业务处理器被加入到流水线后，此方法被回调。也就是在完成ch.pipeline().addLast(handler) 语句之后，会回调handlerAdded()。

2. channelRegistered()：当通道成功绑定一个NioEventLoop线程后，会通过流水线回调所有业务处理器的channelRegistered()方法。
3. channelActive()：当通道激活成功后，会通过流水线会调用所有业务处理器的channelActive()方法。通道激活成功指的是，所有的业务处理器添加、注册的异步任务完成，并且NioEventLoop线程绑定的异步任务完成。
4. channelInactive()：当通道的底层连接已经不是ESTABLISH状态，或者底层连接已经关闭时，会首先回调所有业务处理器的channelInactive()。
5. channelUnregistered()：通道和NioEventLoop线程接触绑定，移除对这条通道的事件处理之后，回调所有业务处理器的channelInactive()方法。
6. handlerRemoved()：最后，Netty会移除掉通道上所有的业务处理器，并且回调所有的的业务处理器的handlerRemoved()方法。

​	上面列取的6个生命周期方法中，前3个在通道创建的时候被现后调用，后面3个在通道关闭时现后被回调。

​	除了生命周期的回调， 就是入站和出站处理的回调。 对于Inhandler入站处理器，有两种很重要的回调方法为：

（1）channelRead()：有数据包入站，通道可读。流水线会启动入站处理流程，从前向后，入站处理器的channelRead()方法会被依次回调到。

（2）channelReadComplete()：流水线完成入站处理后，会从前向后，依次回调每个入站处理器的channelReadComplete()方法，表示数据读取完毕。

​	对于出战处理器ChannelOutboundHandler的生命周期以及回调的顺序，与入站处理器大致相同。

```java
public class OutHandlerDemo extends ChannelOutboundHandlerAdapter {
    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        System.out.println("被调用：handlerAdded()");
        super.handlerAdded(ctx);
    }


    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
        System.out.println("被调用： handlerRemoved()");
        super.handlerRemoved(ctx);
    }

    @Override
    public void bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) throws Exception {
        System.out.println("被调用： bind()");
        super.bind(ctx, localAddress, promise);
    }

    @Override
    public void connect(ChannelHandlerContext ctx, SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise) throws Exception {
        System.out.println("被调用： connect()");
        super.connect(ctx, remoteAddress, localAddress, promise);
    }

    @Override
    public void disconnect(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
        System.out.println("被调用： disconnect()");
        super.disconnect(ctx, promise);
    }

    @Override
    public void read(ChannelHandlerContext ctx) throws Exception {
        System.out.println("被调用： read()");
        super.read(ctx);
    }

    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        System.out.println("被调用： write()");
        super.write(ctx, msg, promise);
    }

    @Override
    public void flush(ChannelHandlerContext ctx) throws Exception {
        System.out.println("被调用： flush()");
        super.flush(ctx);
    }
}
```

```java
public class OutHandlerDemoTester {
    @Test
    public void testlifeCircle() {
        final OutHandlerDemo handler = new OutHandlerDemo();
        ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
            protected void initChannel(EmbeddedChannel ch) {
                ch.pipeline().addLast(handler);
            }
        };

        final EmbeddedChannel channel = new EmbeddedChannel(i);

//        channel.pipeline().addLast(handler);

        //测试出站写入

        ByteBuf buf = Unpooled.buffer();
        buf.writeInt(1);
       
        ChannelFuture f = channel.pipeline().writeAndFlush(buf);
        
        buf.writeInt(2);
        
        f = channel.pipeline().writeAndFlush(buf);
        
        f.addListener(new GenericFutureListener<Future<? super Void>>() {
            @Override
            public void operationComplete(Future<? super Void> future) throws Exception {
                if (future.isSuccess()) {
                    System.out.println("write is finished");
                }
                channel.close();
            }
        });
        try {
            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }
}
```

```java
被调用：handlerAdded()
被调用： read()
被调用： write()
被调用： flush()
被调用： write()
被调用： flush()
write is finished
被调用： handlerRemoved()
```

### 6.6 详解Pipeline流水线

​	前面讲到，一条Netty通道需要很多的Handler业务处理器来处理业务。每条通道内部都有一条流水线（Pipeline）将Handler装配起来。Netty的业务处理器流水线ChannelPipeline是基于责任链设计模式（Chain of Responsibility）来设计的，内部是一个双向链表结构，能够支持动态地添加和删除Handler业务处理器。

#### 6.6.1 Pipeline入站处理流程

​	为了完整地演示Pipeline入站处理流程，将新建三个极为简单的入站处理器，在ChannelInitializer通道初始化处理器的initChannel方法中把它们加入到流水线中。三个入站处理器分别为：SimpleInHandlerA、SimpleInHandlerB、SimpleHandlerC，添加顺序为A->B->C

```java
public class InPipeline {
    static class SimpleInHandlerA extends ChannelInboundHandlerAdapter {
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            Logger.info("入站处理器 A: 被回调 ");
            super.channelRead(ctx, msg);
        }
    }
    static class SimpleInHandlerB extends ChannelInboundHandlerAdapter {
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            Logger.info("入站处理器 B: 被回调 ");
            super.channelRead(ctx, msg);
        }
    }
    static class SimpleInHandlerC extends ChannelInboundHandlerAdapter {
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            Logger.info("入站处理器 C: 被回调 ");
            super.channelRead(ctx, msg);
        }
    }

    @Test
    public void testPipelineInBound() {
        ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
            protected void initChannel(EmbeddedChannel ch) {
                ch.pipeline().addLast(new SimpleInHandlerA());
                ch.pipeline().addLast(new SimpleInHandlerB());
                ch.pipeline().addLast(new SimpleInHandlerC());

            }
        };
        EmbeddedChannel channel = new EmbeddedChannel(i);
        ByteBuf buf = Unpooled.buffer();
        buf.writeInt(1);
        //向通道写一个入站报文
        channel.writeInbound(buf);
        try {
            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    static class SimpleInHandlerB2 extends ChannelInboundHandlerAdapter {
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            Logger.info("入站处理器 B: 被回调 ");
            //不调用基类的channelRead, 终止流水线的执行
//            super.channelRead(ctx, msg);
        }
    }

    //测试流水线的截断
    @Test
    public void testPipelineCutting() {
        ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
            protected void initChannel(EmbeddedChannel ch) {
                ch.pipeline().addLast(new SimpleInHandlerA());
                ch.pipeline().addLast(new SimpleInHandlerB2());
                ch.pipeline().addLast(new SimpleInHandlerC());
            }
        };
        EmbeddedChannel channel = new EmbeddedChannel(i);
        ByteBuf buf = Unpooled.buffer();
        buf.writeInt(1);
        //向通道写一个入站报文
        channel.writeInbound(buf);
        try {
            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

​	输出结果

```java
入站处理器 A: 被回调 
入站处理器 B: 被回调 
```

​	*查看了一下fireChannelRead()的源码(该方法6.5.1 有介绍，即通道缓冲区可读时，Netty会调用它，触发通道可读事件)*

```java
// super.channelRead(ctx, msg);
// public class ChannelInboundHandlerAdapter extends ChannelHandlerAdapter implements ChannelInboundHandler
@Skip
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    ctx.fireChannelRead(msg);
}

// public class DefaultChannelPipeline implements ChannelPipeline
public final ChannelPipeline fireChannelRead(Object msg) {
    AbstractChannelHandlerContext.invokeChannelRead(this.head, msg);
    return this;
}

// abstract class AbstractChannelHandlerContext implements ChannelHandlerContext, ResourceLeakHint
static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
        final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            next.invokeChannelRead(m);
        } else {
            executor.execute(new Runnable() {
                public void run() {
                    next.invokeChannelRead(m);
                }
            });
        }

    }
```

​	在channelRead()方法中，我们打印当前Handler业务处理器的信息，然后调用父类的channelRead()方法，而父类的channelRead()方法会自动调用下一个inBoundHandler的channelRead()方法，**并且会把当前inBoundHandler入站处理器中处理完毕的对象传递到下一个inBoundHandler入站处理器，我们在示例程序中传递的对象都是同一个信息(msg)**。

​	在channelRead()方法中，如果不调用父类的channelRead()方法，结果就是不会继续执行流水线的下一个Handler处理器。

​	我们可以看到，**入站处理器的流动顺序是：从前到后。加在前面的，执行也在前面**。

#### 6.6.2 Pipeline出站处理流程

​	为了完整地延时Pipeline出站处理流程，将新建三个极为简答地出站处理器，在ChannelInitializer通道初始化处理器的initChannel方法中，把它们加入到流水线中。三个出站处理器分别为：SimpleOutHandlerA、SimpleOutHandlerB、SimpleOutHandlerC，添加的顺序为A->B->C。

```java
public class OutPipeline {
    public class SimpleOutHandlerA extends ChannelOutboundHandlerAdapter {
        @Override
        public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
            System.out.println("出站处理器 A: 被回调" );
            super.write(ctx, msg, promise);
        }
    }
    public class SimpleOutHandlerB extends ChannelOutboundHandlerAdapter {
        @Override
        public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
            System.out.println("出站处理器 B: 被回调" );
            super.write(ctx, msg, promise);
        }
    }
    public class SimpleOutHandlerC extends ChannelOutboundHandlerAdapter {
        @Override
        public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
            System.out.println("出站处理器 C: 被回调" );
            super.write(ctx, msg, promise);
        }
    }
    @Test
    public void testPipelineOutBound() {
        ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
            protected void initChannel(EmbeddedChannel ch) {
                ch.pipeline().addLast(new SimpleOutHandlerA());
                ch.pipeline().addLast(new SimpleOutHandlerB());
                ch.pipeline().addLast(new SimpleOutHandlerC());
            }
        };
        EmbeddedChannel channel = new EmbeddedChannel(i);
        ByteBuf buf = Unpooled.buffer();
        buf.writeInt(1);
        //向通道写一个出站报文
        channel.writeOutbound(buf);
        try {
            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public class SimpleOutHandlerB2 extends ChannelOutboundHandlerAdapter {
        @Override
        public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
            System.out.println("出站处理器 B: 被回调" );
            //不调用基类的channelRead, 终止流水线的执行
//            super.write(ctx, msg, promise);
        }
    }

    @Test
    public void testPipelineOutBoundCutting() {
        ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
            protected void initChannel(EmbeddedChannel ch) {
                ch.pipeline().addLast(new SimpleOutHandlerA());
                ch.pipeline().addLast(new SimpleOutHandlerB2());
                ch.pipeline().addLast(new SimpleOutHandlerC());
            }
        };
        EmbeddedChannel channel = new EmbeddedChannel(i);
        ByteBuf buf = Unpooled.buffer();
        buf.writeInt(1);
        //向通道写一个出站报文
        channel.writeOutbound(buf);
        try {
            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }
}
```

​	执行结果：

```java
出站处理器 C: 被回调
出站处理器 B: 被回调
```

​	在write()方法中，打印当前Handler业务处理器的信息，然后调用父类的write()方法，而这里父类的write()方法会自动调用下一个outBoundHandler出站处理器的write()方法。这里有个小问题：在OutBoundHandler出站处理器的write()方法中，如果不调用父类的write()方法，结果就是流水线Pipeline的下一个出站处理器Handler不被调用。

​	在代码中，通过pipeline.addLast()方法添加OutBoundHandler出站处理器的顺序为A->B->C。从结果可以看出，出站流水处理次序为从后向前：C->B->A。**最后加入的出站处理器，反而执行在最前面。这一点和Inbound入站处理次序是相反的**。

#### 6.6.3 ChannelHandlerContext上下文

​	<u>不管我们定义的是**哪种类型**的Handler业务处理器，最终它们都是以双向链表的方式保存在流水线中</u>。这里的流水线的节点类型，并不是前面的Handler业务处理器基类，而是一个新的Netty类型：ChannelHandlerContext通道处理器上下文类。

​	**在Handler业务处理器被添加到流水线中时，会创建一个通达处理器上下文ChannelHandlerContext，它代表了ChannelHandler通道处理器和ChannelPipeline通道流水线之间的关联**。

​	ChannelHandlerContext中包含了许多方法，主要可以分为**两类**：**第一类是获取上下文关联的Netty组件实例**，如所关联的通道、所关联的流水线、上下文内部Handler业务处理器实例等；**第二类是入站和出站的处理方法**。

​	在Channel、ChannelPipeline、ChannelHandlerContext三个类中，会有同样的出站和入站处理方法，同一个操作出现在不同的类中，功能有何不同？**如果通过Channel或ChannelPipeline的实例来调用这些方法，它们就会在整条流水线中传播。然而，如果是通过ChannelHandlerContext通道处理上下文进行调用，就只会从当前的节点开始执行Handler业务处理器，并传播到<u>同类型处理器</u>的下一站（节点）**。

​	Channel、Handler、ChannelHandlerContext三者的关系为：**Channel通道拥有一条ChannelPipeline通道流水线，每一个流水线节点为一个ChannelHandlerContext通道处理器上下文对象，每一个上下文包裹了一个ChannelHandler通道处理器**。在ChannelHandler通道处理器的入站/出站处理方法中，Netty都会传递一个Context上下文实例作为实际参数。<u>通过Context实例的参数，在业务处理中，可以获取ChannelPipeline通道流水线的实例或者Channel通道的实例</u>。

#### 6.6.4 截断流水线的处理

​	在入站/出站的过程中，由于业务条件不满足，需要截断流水线的处理，不让处理进入下一站，怎么办呢？

​	首先以channelRead通道读方法的流程为例，看看如何截断入站处理流程。这里的办法是：在channelRead方法中，不再调用父类的channelRead入站方法，它的代码如下：

```java
public class Inpipeline {
    // ... 省略SimpleInHandlerA、SimpleInHandlerC
    
    // 定义SimpleInHandlerB2
        static class SimpleInHandlerB2 extends ChannelInboundHandlerAdapter {
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            Logger.info("入站处理器 B: 被回调 ");
            //不调用基类的channelRead, 终止流水线的执行
//            super.channelRead(ctx, msg);
        }
    }

    //测试流水线的截断
    @Test
    public void testPipelineCutting() {
        ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
            protected void initChannel(EmbeddedChannel ch) {
                ch.pipeline().addLast(new SimpleInHandlerA());
                ch.pipeline().addLast(new SimpleInHandlerB2());
                ch.pipeline().addLast(new SimpleInHandlerC());

            }
        };
        EmbeddedChannel channel = new EmbeddedChannel(i);
        ByteBuf buf = Unpooled.buffer();
        buf.writeInt(1);
        //向通道写一个入站报文
        channel.writeInbound(buf);
        try {
            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

​	执行结果：

```java
入站处理器 A: 被回调 
入站处理器 B: 被回调 
```

​	从运行的结果看来，入站处理器C没有执行到，说明通过没调用父类的super.channelRead方法，处理流水线被成功地截断了。

​	在channelRead方法中，<u>入站处理传入下一站还有一种方法：调用Context上下文的ctx.fireChannelRead(msg)方法</u>。如果要截断流水线的处理，很显然，就不能调用ctx.fireChannelRead(msg)方法。

​	上面读操作流程的截断仅仅是一个示例。如果要截断其他的入站处理的流水线操作（使用Xxx指代），也可以同样处理：

1. 不调用supper.channelXxx（ChannelHandlerContext...）

2. 也不调用ctx.fireChannelXxx()

​	如何截断出站处理流程呢？结论是：**<u>出站处理流程只要开始执行，就不能被截断</u>**。强制截断的话，Netty会抛出异常。如果业务条件不满足，可以不启动出站处理。

ps:我自己试了一下，出站截断没报错，比较迷？

#### 6.6.5 Handler业务处理器的热拔插

​	<u>Netty中的处理器流水线式一个双向链表</u>。在程序执行过程中，可以动态进行业务处理器的**热拔插：动态地增加、删除流水线上的业务处理器Handler**。主要的Handler热拔插方法声明在ChannelPipeline接口中，如下：

```java
package io.netty.channel;

public interface ChannelPipeline extends Iterable<Entry<String, ChannelHandler>>
{
    //...
    //在头部增加一个业务处理器，名字由name指定
    ChannelPipeline addFirst(String name, ChannelHandler handler);
    //在尾部增加一个业务处理器，名字由name指定
    ChannelPipeline addLast(String name, ChannelHandler handler);
    //在baseName处理器的前面增加一个业务处理器，名字由name指定
    ChannelPipeline addBefore(String baseName, String name, ChannelHandler handler);
    //在baseName处理器的后面增加一个业务处理器，名字由name指定
    ChannelPipeline addAfter(String baseName, String name, ChannelHandler handler);
    //删除一个业务处理器实例
    ChannelPipeline remove(ChannelHandler handler);
    //删除一个处理器实例
    ChannelHandler remove(String handler);
    //删除第一个业务处理器
    ChannelHandler removeFirst();
    //删除最后一个业务处理器
    ChannelHandler removeLast();
}
```

​	下面是一个简单的示例：调用流水线实例的remove(ChannelHandler)方法，从流水线动态地删除一个Handler实例。代码如下：

```java
public class PipelineHotOperateTester {

    static class SimpleInHandlerA extends ChannelInboundHandlerAdapter {

        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            System.out.println("入站处理器 A: 被回调 ");
            super.channelRead(ctx, msg);
            //从流水线删除当前Handler
            ctx.pipeline().remove(this);
        }

    }

    static class SimpleInHandlerB extends ChannelInboundHandlerAdapter {
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            System.out.println("入站处理器 B: 被回调 ");
            super.channelRead(ctx, msg);
        }
    }

    static class SimpleInHandlerC extends ChannelInboundHandlerAdapter {
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            System.out.println("入站处理器 C: 被回调 ");
            super.channelRead(ctx, msg);
        }
    }

    //测试处理器的热拔插
    @Test
    public void testPipelineHotOperating() {
        ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
            protected void initChannel(EmbeddedChannel ch) {
                ch.pipeline().addLast(new SimpleInHandlerA());
                ch.pipeline().addLast(new SimpleInHandlerB());
                ch.pipeline().addLast(new SimpleInHandlerC());

            }
        };
        EmbeddedChannel channel = new EmbeddedChannel(i);
        ByteBuf buf = Unpooled.buffer();
        buf.writeInt(1);
        //第一次向通道写入站报文
        channel.writeInbound(buf);
        
        //第二次向通道写入站报文
        channel.writeInbound(buf);
        //第三次向通道写入站报文
        channel.writeInbound(buf);
        try {
            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

​	运行结果：

```java
入站处理器 A: 被回调 
入站处理器 B: 被回调 
入站处理器 C: 被回调 
入站处理器 B: 被回调 
入站处理器 C: 被回调 
入站处理器 B: 被回调 
入站处理器 C: 被回调 
```

​	从运行结果可以看出，在SimpleInHandlerA从流水线删除后，在后面的入站流水处理中，SimpleInHandlerA已经不再被调用了。

​	Netty的通道初始化处理器——ChannelInitializer，在它的注册回调channelRegistered方法中，就使用了ctx.pipeline().remove(this)，将自己从流水线中删除。

```java
// public abstract class ChannelInitializer<C extends Channel> extends ChannelInboundHandlerAdapter 

protected abstract void initChannel(C var1) throws Exception;

public final void channelRegistered(ChannelHandlerContext ctx) throws Exception {
    if (this.initChannel(ctx)) {
        ctx.pipeline().fireChannelRegistered();
        this.removeState(ctx);
    } else {
        ctx.fireChannelRegistered();
    }

}
```

​	<u>ChannelInitializer在完成通道的初始化之后，为什么要将自己从流水线中删除呢？原因很简单，就是一条通道只需要做一次初始化的工作</u>。

### 6.7 详解ByteBuf缓冲区

​	Netty提供了ByteBuf来代替Java NIO的ByteBuffer缓冲区，以操纵内存缓冲区。

#### 6.7.1 ByteBuf的优势

​	与Java NIO 的ByteBuffer相比，ByteBuf的优势如下：

+ **Pooling（池化，这点减少了内存复制和GC，提升了效率）**
+ 复合缓冲区类型，支持零复制
+ 不需要调用flip()方法去切换读/写模式
+ 扩展性好，例如StringBuffer
+ 可以自定义缓冲区类型
+ **读取和写入索引分开**
+ 方法的链式调用
+ 可以进行引用计数，方便重复使用

#### 6.7.2 ByteBuf的逻辑部分

​	ByteBuf是一个字节容器，<u>内部是一个字节数组</u>。从逻辑上来分，字节容器内部可以分为四个部分。

| 第一部分 | 第二部分 | 第三部分 | 第四部分 |
| -------- | -------- | -------- | -------- |
| 废弃     | 可读     | 可写     | 可扩容   |

​	第一部分是已用字节，表示已经使用完的废弃的无效字节；第二部分是可读字节，这部分数据是ByteBuf保存的有效数据，从ByteBuf中读取的数据都来自这一部分；第三部分是可写字节，写入到ByteBuf的数据都会写到这一部分中；第四部分是可扩容字节，表示的是该ByteBuf最多还能扩容的大小。

#### 6.7.3 ByteBuf的重要属性

​	ByteBuf通过三个整性的属性有效地区分可读和可写数据，使得读写之间相互没有冲突。这三个属性定义在AbstractByteBuf抽象类中，分别是：

+ readerIndex（读指针）
+ writerIndex（写指针）
+ maxCapacity（最大容量）

​	ByteBuf的这三个重要属性，如下图

```none
[] 表示涵盖的范围，()表示第XX部分，<>表示指针
[							ByteBuf 内部数组							]
[					capacity					]
(废弃) <readerIndex> (可读) <writerIndex>  (可写)    (可扩容) <maxCapacity>
```

​	这三个属性的详细介绍如下：

+ readerIndex（读指针）：表示读取的起始位置。每读取一个字节，readerIndex自动增加1，一旦readerIndex与writerIndex相同，则表示ByteBuf不可读了。
+ writerIndex（写指针）：指示写入的初始位置。每写一个字节，writerIndex自动增加1，一旦增加到writerIndex与capacity()容量相同，则表示ByteBuf已经不可以写了。capacity()是一个成员方法，不是一个成员属性，它表示ByteBuf中可以写入的容量。注意，它不是最大容量maxCapacity。
+ maxCapacity（最大容量）：表示ByteBuf可以扩容的最大容量。当向ByteBuf写数据的时候，如果容量不足，可以进行扩容。扩容的最大限度由maxCapacity的值来设定，超过maxCapacity就会报错。

#### 6.7.4 ByteBuf的三组方法

​	ByteBuf的方法大致可以分为三组。

第一组：容量系列

+ capacity():表示ByteBuf的容量，它的值是以下三部分之和：废弃的字节数、可读字节数和可写字节数。
+ maxCapacity()：表示ByteBuf最大能够容纳的字节数。当向ByteBuf中写数据的时候，如果发现容量不足，则进行扩容，直到扩容到maxCapacity设定的上限。

第二组：写入系列

+ isWritable()：表示ByteBuf是否可写。如果capacity()容量大于writerIndex 指针的位置，则表示可写，否则为不可写。注意：如果isWritable()返回false，并不代表不能再往ByteBuff中写数据了。如果Netty发现往ByteBuf中写数据写不进去的话，会自动扩容ByteBuf。
+ writableBytes()：取得可写入的字节数，它的值等于容量capacity()减去writerIndex。
+ maxWritableBytes()：取得最大的可写字节数，它的值等于最大容量maxCapacity减去writerIndex。
+ **writeBytes(byte[] src)**：把src字节数组中的数据全部写到ByteBuf。这就是最为常用的一个方法。
+ writeTYPE(TYPE value)：写入基础数据类型的数据。TYPE表示基础数据类型，包含了8大基础数据类型。具体如下：writeByte()、writeBoolean()、writeChar()、writeShort()、writeInt()、writeLong()、writeFloat()、writeDouble()。

+ setTYPE(TYPE value)：基础数据类型的设置，不改变writerIndex指针值，包含了8大基础数据类型的设置。具体如下：setByte()、setBoolean()、setChar()、setShort()、setInt()、setLong()、setFloat()、setDouble()。<u>setTYPE系列与writeTYPE系列的不同：setType系列不改变读写指针writerIndex的值；writeTYPE系列会改变writerIndex 的值</u>。

+ markWriterIndex()与resetWriterIndex()：这两个方法一起介绍。前一个方法表示把当前的写指针writerIndex属性的值保存在markedWriterIndex属性中；后一个方法表示把之前保存的markedWriterIndex的值恢复到写指针writerIndex属性中。markedWriterIndex属性相当于一个暂存属性，也定义在AbstractByteBuf抽象基类中。

第三组：读取系列

+ isReadable()：返回ByteBuf是否可读。如果writerIndex指针的值大于readerIndex指针的值，则表示可读，否则为不可读。
+ readableBytes()：返回表示ByteBuf当前可读取的字节数，它的值等于writerIndex减去readerIndex。
+ **readBytes(byte[] dst)**：读取ByteBuf中的数据。将数据从ByteBuf读取到dst字节数组中，这里dst字节数组的大小，通常等于readableBytes()。这个方法也是最常用的一个方法之一。
+ readType()：读取基础数据类型，可以读取8大基础数据类型。具体如下：readByte()、readBoolean()、readChar()、readShort()、readInt()、readLong()、readFloat()、readDouble()。

+ getTYPE(TYPE value)：读取基础数据类型，并且不改变指针值。具体如下：getByte()、getBoolean()、getChar()、getShort()、getInt()、getLong()、getFloat()、getDouble()。<u>getTYPE系列与readTYPE系列的不同：getTYPE系列不会改变指针readerIndex的值；readTYPE系列会改变读指针readerIndex的值</u>。
+ markReaderIndex()与resetReaderIndex()：这两个方法一起介绍。前一个方法表示把当前的读指针ReaderIndex属性的值保存在markedReaderIndex属性中；后一个方法表示把之前保存的markedReaderIndex的值恢复到读指针ReaderIndex属性中。markedReaderIndex属性相当于一个暂存属性，也定义在AbstractByteBuf抽象基类中。

#### 6.7.5 ByteBuf基本使用的实践案例

​	ByteBuf的基本使分为三部分：

1. 分配一个ByteBuf实例；
2. 向ByteBuf写数据；
3. 从ByteBuf读数据。

​	这里使用了默认的分配器，分配了一个初始容量为9，最大限制为100个自己的缓冲区。关于ByteBuf实例的分配，稍后具体详细介绍。

```java
public class WriteReadTest {

    @Test
    public void testWriteRead() {
        ByteBuf buffer = ByteBufAllocator.DEFAULT.buffer(9,100);
        System.out.println("动作：分配 ByteBuf(9, 100)" + buffer);
        buffer.writeBytes(new byte[]{1, 2, 3, 4});
        System.out.println("动作：写入4个字节 (1,2,3,4)" + buffer);
        System.out.println("start==========:get==========");
        getByteBuf(buffer);
        System.out.println("动作：取数据 ByteBuf" + buffer);
        System.out.println("start==========:read==========");
        readByteBuf(buffer);
        System.out.println("动作：读完 ByteBuf" + buffer);
    }

    //读取一个字节
    private void readByteBuf(ByteBuf buffer) {
        while (buffer.isReadable()) {
            System.out.println("读取一个字节:" + buffer.readByte());
        }
    }
    
    //读取一个字节，不改变指针
    private void getByteBuf(ByteBuf buffer) {
        for (int i = 0; i < buffer.readableBytes(); i++) {
            System.out.println("读取一个字节:" + buffer.getByte(i));
        }
    }
}
```

​	运行结果：

```java
动作：分配 ByteBuf(9, 100)PooledUnsafeDirectByteBuf(ridx: 0, widx: 0, cap: 9/100)
动作：写入4个字节 (1,2,3,4)PooledUnsafeDirectByteBuf(ridx: 0, widx: 4, cap: 9/100)
start==========:get==========
读取一个字节:1
读取一个字节:2
读取一个字节:3
读取一个字节:4
动作：取数据 ByteBufPooledUnsafeDirectByteBuf(ridx: 0, widx: 4, cap: 9/100)
start==========:read==========
读取一个字节:1
读取一个字节:2
读取一个字节:3
读取一个字节:4
动作：读完 ByteBufPooledUnsafeDirectByteBuf(ridx: 4, widx: 4, cap: 9/100)
```

​	可以看到，使用get取数据是不会影响到ByteBuf的指针属性值的。

#### 6.7.6 ByteBuf的引用计数

​	**Netty的ByteBuf的内存回收工作是通过<u>引用计数</u>的方式管理的**。JVM中使用”计数器“（一种GC算法）来标记对象是否”不可达“进而收回（注：GC是Garbage Collection的缩写，即Java中的垃圾回收机制），Netty也使用了这种手段来对ByteBuf的引用进行计数。Netty采用”计数器“来追踪ByteBuf的生命周期，一是对Pooled ByteBuf的支持，二是能够尽快地”发现“那些可以回收的ByteBuf（非Pooled），以便提升ByteBuf的分配和销毁的效率。

​	插个题外话：什么是Pooled(池化)的ByteBuf缓冲区呢？在通信程序的执行过程中，Buffer缓冲区实例会被频繁创建、使用、释放。大家都知道，频繁创建对象、内存分配、释放内存，系统的开销大、性能低，如何提升性能、提高Buffer实例的使用率呢？**从Netty4版本开始，新增了对象池化的机制。即创建一个Buffer对象池，将没有被引用的Buffer对象，放入对象缓冲池中；当需要时，则重新从对象池中取出，则不需要重新创建**。

​	<u>引用计数的大致规则如下：默认情况下，当创建完一个ByteBuf时，它的引用为1；每次调用retain()方法，它的引用就加1；每次调用release()方法，就是将引用计数减1；如果引用为0，再次访问这个ByteBuf对象，就会抛出异常；如果引用为0，表示这个ByteBuf没有哪个进程引用它，它占用的内存需要回收</u>。

```java
public class ReferenceTest {
    @Test
    public  void testRef()
    {
        ByteBuf buffer  = ByteBufAllocator.DEFAULT.buffer();
        System.out.println("after create:"+buffer.refCnt());
        buffer.retain();
        System.out.println("after retain:"+buffer.refCnt());
        buffer.release();
        System.out.println("after release:"+buffer.refCnt());
        buffer.release();
        System.out.println("after release:"+buffer.refCnt());
        //错误:refCnt: 0,不能再retain
        buffer.retain();
        System.out.println("after retain:"+buffer.refCnt());
    }
}
```

​	输出如下：

```java
after create:1
after retain:2
after release:1
after release:0
断开与目标 VM 的连接，地址：'127.0.0.1:58248', transport: 'socket'

io.netty.util.IllegalReferenceCountException: refCnt: 0, increment: 1
	at io.netty.util.internal.ReferenceCountUpdater.retain0(ReferenceCountUpdater.java:123)
	at io.netty.util.internal.ReferenceCountUpdater.retain(ReferenceCountUpdater.java:110)
	at io.netty.buffer.AbstractReferenceCountedByteBuf.retain(AbstractReferenceCountedByteBuf.java:80)
    ...报错信息

```

​	最后一次retain方法抛出了IllegalReferenceCountException异常。原因是：在此之前，缓冲区buffer的引用计数已经为0，不能再retain了。也就是说：**在Netty中，引用计数为0的缓冲区不能再继续使用**。

​	**为了确保引用计数不会混乱，在Netty的业务处理器开发过程中，应该坚持一个原则：retain和release方法成对使用**。简单地说，在一个方法中，调用了retain，就应该调用一次release。

```java
public void handlMethodA(ByteBuf byteBuf) {
    byteBuf.retain();
    try {
        handlMethodB(byteBuf);
    } finally {
        byteBuf.release();
    }
}
```

​	如果retain和release这两个方法，一次都不调用呢？则在缓冲区使用完后，调用一次release，就是释放一次。例如在Netty流水线上，中间所有的Handler业务处理器处理完ByteBuf之后直接传递给下一个，由最后一个Handler负责调用release来释放缓冲区的内存空间。

​	**当引用计数已经为0，Netty会进行ByteBuf的回收**。分为两种情况：（1）Pooled池化的ByteBuf内存，回收的方法是：放入可以重新分配的ByteBuf池子，等待下一次分配。（2）Unpooled未池化的ByteBuf缓冲区，回收分为两种情况：如果是堆(Heap)结构缓冲，会被JVM的垃圾回收机制回收；如果是Direct类型，调用本地方法释放外部内存(unsafe.freeMemory)。

> [Netty Unpooled 内存分配](https://www.jianshu.com/p/566d162e89c8)
>
> [自顶向下深入分析Netty（九）--UnpooledByteBuf源码分析](https://www.jianshu.com/p/ae8010b06ac2)

#### 6.7.7 ByteBuf的Allocator分配器

​	**Netty通过ByteBufAllocator分配器来创建缓冲区和分配内存空间**。Netty提供了ByteBufAllocator的两种实现：PoolByteBufAllocator和UnpooledByteAllocator。

​	PoolByteBufAllocator（池化ByteBuf分配器）将ByteBuf实例放入池中，提高了性能，将内存碎片减少到最小**；这个池化分配器采用了jemalloc高效内存分配的策略，该策略被好几种现代操作系统所使用**。

​	<u>UnpooledByteBufAllocator是普通的未池化ByteBuf分配器，它没有把ByteBuf放入池中，每次被调用时，返回一个新的ByteBuf实例：通过Java的垃圾回收机制回收。</u>

​	为了验证**两者的性能**，大家可以做一下对比实验：

（1）使用UnpooledByteBufAllocator的方式分配ByteBuf缓冲区，开启10000个长连接，每秒所有的连接发送一条信息，再看看服务器的内存使用量的情况。

​	实验的参考结果：在短时间内，可以看到占到10GB多的内存空间，但随着系统的运行，内存空间不断增长，直到整个系统内存被占满而导致内存溢出，最终系统宕机。

（2）把UnplooedByteBufAllocator换成PooledByteBufAllocator，再进行试验，看看服务器的内存使用量的情况。

​	实验的参考结果：内存使用量基本能维持在一个连接占用1MB左右的内存空间，内存使用量保持在10GB左右，经过长时间的运行测试，我们会发现内存使用量基本都能维持在这个数量附近。系统不会因为内存被消耗尽而崩溃。

​	在Netty中，默认的分配器为ByteBufAllocator.DEFAULT，可以通过Java系统参数（System.Property）的选项io.netty.allocator.type进行配置，配置时使用字符串值：“unpooled”，“pooled”。

​	不同的Netty版本，对于分配器的默认使用策略是不一样的。在Netty4.0版本中，默认的分配器为UnpooledByteBufAllocator。而在Netty4.1版本中，默认的分配器为PooledByteBufAllocator。现在PooledByteBufAllocator已经广泛使用了一段时间，并且有了**增强的缓冲区泄露追踪机制**。因此，可以在Netty程序中设置启动器Bootstrap的时候，将PooledByteBufAllocator设置为默认的分配器。

```java
SerevrBootstrap b = new ServerBootstrap();
// ... 
// 4 设置通道的参数
b.option(ChannelOption.SO_KEEPALIVE, true);
b.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
b.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
```

​	内存管理的策略可以灵活调整，这是使用Netty所带来的又一个好处。只需一行简单的配置，就能获得到池化缓冲区带来的好处。在底层，Netty为我们干了所有“脏活、累活”！这主要是因为Netty用到了Java的Jemalloc内存管理库。

​	使用分配器分配ByteBuf的方法有多种。下面列出主要的几种：

```java
public class AllocatorTest {
    @Test
    public void showAlloc() {
        ByteBuf buffer = null;
        //方法一：默认分配器，分配初始容量为9，最大容量100的缓冲
        buffer = ByteBufAllocator.DEFAULT.buffer(9, 100);
        //方法二：默认分配器，分配初始为256，最大容量Integer.MAX_VALUE 的缓冲
        buffer = ByteBufAllocator.DEFAULT.buffer();
        //方法三：非池化分配器，分配基于Java的堆内存缓冲区
        buffer = UnpooledByteBufAllocator.DEFAULT.heapBuffer();
        //方法四：池化分配器，分配基于操作系统的管理的直接内存缓冲区
        buffer = PooledByteBufAllocator.DEFAULT.directBuffer();
        //…其他方法
    }
}
```

​	如果没特别的要求，使用第一种或者第二种分配方法分配缓冲区即可。

#### 6.7.8 ByteBuf缓冲区的类型

​	缓冲区的类型，根据内存的管理方不同，分为堆缓冲区和直接缓冲区，也就是Heap ByteBuf和Direct ByteBuf。另外，为了方便缓冲区进行组合，提供了一种组合缓冲区。

| 类型            | 说明                                                         | 优点                                                         | 不足                                                         |
| --------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Heap ByteBuf    | 内存数据为一个**Java数组**，存储在JVM的堆空间中，通过hasArray来判断是不是堆缓冲区 | 未使用池化的情况下，能提供快速的分配和释放                   | 写入底层传输通道之前，都会复制到<u>直接缓冲区</u>            |
| Direct ByteBuf  | 内部数据存储在**操作系统的物理内存**中                       | 能获取超过JVM堆限制大小的内存空间；<u>写入传输通道比堆缓冲区更快</u> | <u>释放和分配空间昂贵（使用系统的方法）</u>；在Java中操作时需要复制一份到堆上 |
| CompositeBuffer | 多个缓冲区的组合表示                                         | 方便一次操作多个缓冲区实例                                   |                                                              |

​	上面三种缓冲区的类型，无论哪一种，都可以通过池化（Pooled）、非池化（Unpooled）两种分配器来创建和分配内存空间。

​	下面对Direct Memory（直接内存）进行一些特别的介绍：

+ **Direct Memory不属于Java堆内存，所分配的内存其实是调用操作系统malloc()函数来获得的**；由Netty的本地内存堆Native堆进行管理。
+ Direct Memory容量可通过-XX:MaxDirectMemorySize来指定，如果不指定，则默认与Java堆的最大值(-Mmx指定)一样。注意：并不是强制要求，有的JVM默认Direct Memory与-Mmx无直接关系
+ **Direct Memory的使用避免了Java堆和Native堆之间来回复制数据**。在某些应用场景提高了性能。
+ 在需要频繁创建缓冲区的场合，由于<u>创建和销毁Direct Buffer（直接缓冲区）的代价比较高昂</u>，因此不宜使用Direct Buffer。也就是说，**Direct Buffer尽量在池化分配器中分配和回收**。如果能将Direct Buffer进行复用，在读写频繁的情况下，就可以大幅度改善性能。
+ **对Direct Buffer的读写比Heap Buffer快，但是它的创建和销毁比普通Heap Buffer慢**。
+ **在Java的垃圾回收机制回收Java堆时，Netty框架也会释放不再使用的Direct Buffer缓冲区**，<u>因为它的内存为**堆外内存**，所以清理的工作不会为Java虚拟机(JVM)带来压力</u>。注意一下垃圾回收的应用场景：（1）垃圾回收仅在Java堆被填满，以至于无法为新的堆分配请求提供服务时发生；（2）在Java应用程序中调用System.gc()函数释放内存。

> [Java的堆,栈,方法区](https://blog.csdn.net/danny_idea/article/details/81137306)

#### 6.7.9 三类ByteBuf使用的实践案例

​	首先对比介绍一下，Heap ByteBuf和Direct ByteBuf两类缓冲区的使用。它们有以下几点不同：

+ 创建的方法不同：Heap ByteBuf通过调用分配器的buffer()方法来创建；而Direct ByteBuf的创建，是通过调用分配器的directBuffer()方法。
+ Heap ByteBuf缓冲区可以直接通过array()方法读取内部数组；而Direct ByteBuf缓冲区不能读取内部数组。
+ 可以调用hasArray()方法来判断是否为Heap ByteBuf类型的缓冲区；如果hasArray()返回值为true，则表示是Heap堆缓冲，否则就不是。
+ **Direct ByteBuf要读取缓冲数据进行业务处理，相对比较麻烦，需要通过getBytes/readBytes等方法先将数据复制到Java堆内存，然后进行其他的计算**。

​	Heap ByteBuf和Direct ByteBuf这两类缓冲区的使用对比，实践案例的代码如下：

```java
public class BufferTypeTest {
   final static Charset UTF_8 = Charset.forName("UTF-8");

    //堆缓冲区
    @Test
    public  void testHeapBuffer() {
        //取得堆内存
        ByteBuf heapBuf =  ByteBufAllocator.DEFAULT.buffer();
        heapBuf.writeBytes("疯狂创客圈:高性能学习社群".getBytes(UTF_8));
        if (heapBuf.hasArray()) {
            //取得内部数组
            byte[] array = heapBuf.array();
            int offset = heapBuf.arrayOffset() + heapBuf.readerIndex();
            int length = heapBuf.readableBytes();
            System.out.println(new String(array,offset,length, UTF_8));
        }
        heapBuf.release();

    }

    //直接缓冲区
    @Test
    public  void testDirectBuffer() {
        ByteBuf directBuf =  ByteBufAllocator.DEFAULT.directBuffer();
        directBuf.writeBytes("疯狂创客圈:高性能学习社群".getBytes(UTF_8));
        if (!directBuf.hasArray()) {
            int length = directBuf.readableBytes();
            byte[] array = new byte[length];
            //读取数据到堆内存
            directBuf.getBytes(directBuf.readerIndex(), array);
            System.out.println(new String(array, UTF_8));
        }
        directBuf.release();
    }
}
```

​	注意，<u>如果hasArray()返回false，不一定代表缓冲区一定就是Direct ByteBuf直接缓冲区，也可能是CompositeByteBuf缓冲区</u>。

​	**在很多通信编程场景下，需要多个ByteBuf组成一个完整的消息**：例如HTTP协议传输时消息总是由Header（消息头）和Body（消息体）组成的。<u>如果传输的内容很长，就会分成多个消息包进行发送，消息中的Header就需重用，而不是每次发送都创建新的Header</u>。

​	下面演示一下通过CompositeByteBuf来复用Header，代码如下：

```java
public class CompositeBufferTest {
    static Charset utf8 = Charset.forName("UTF-8");

    @Test
    public void byteBufComposite() {
        CompositeByteBuf cbuf = ByteBufAllocator.DEFAULT.compositeBuffer();
        //消息头
        ByteBuf headerBuf = Unpooled.copiedBuffer("疯狂创客圈:", utf8);
        //消息体1
        ByteBuf bodyBuf = Unpooled.copiedBuffer("高性能 Netty", utf8);
        cbuf.addComponents(headerBuf, bodyBuf);
        sendMsg(cbuf);
        headerBuf.retain();
        cbuf.release();

        cbuf = ByteBufAllocator.DEFAULT.compositeBuffer();
        //消息体2
        bodyBuf = Unpooled.copiedBuffer("高性能学习社群", utf8);
        cbuf.addComponents(headerBuf, bodyBuf);
        sendMsg(cbuf);
        cbuf.release();
    }

    private void sendMsg(CompositeByteBuf cbuf) {
        //处理整个消息
        for (ByteBuf b : cbuf) {
            int length = b.readableBytes();
            byte[] array = new byte[length];
            //将CompositeByteBuf中的数据复制到数组中
            b.getBytes(b.readerIndex(), array);
            //处理一下数组中的数据
            System.out.print(new String(array, utf8));
        }
        System.out.println();
    }
}
```

​	上面程序中，向CompositeByteBuf对象增加ByteBuf对象实例，这里调用了addComponents方法。Heap ByteBuf和Direct ByteBuf两种类型都可以增加。**如果内部只存在一个实例，则CompositeByteBuf中的hasArray()方法，将返回这个唯一实例的hasArray()方法的值；如果有多个实例，CompositeByteBuf中的hasArray()方法返回false**。

​	**调用nioBuffer()方法可以将CompositeByteBuf实例合并成一个新的<u>Java NIO ByteBuffer缓冲区</u>**（注意：不是ByteBuf）

```java
public class CompositeBufferTest {
    static Charset utf8 = Charset.forName("UTF-8");

    @Test
    public void intCompositeBufComposite() {
        CompositeByteBuf cbuf = Unpooled.compositeBuffer(3);
        cbuf.addComponent(Unpooled.wrappedBuffer(new byte[]{1, 2, 3}));
        cbuf.addComponent(Unpooled.wrappedBuffer(new byte[]{4}));
        cbuf.addComponent(Unpooled.wrappedBuffer(new byte[]{5, 6}));
        //合并成一个单独的缓冲区
        ByteBuffer nioBuffer = cbuf.nioBuffer(0, 6);
        byte[] bytes = nioBuffer.array();
        System.out.print("bytes = ");
        for (byte b : bytes) {
            System.out.print(b);
        }
        cbuf.release();
    }
// ...
}
```

​	在以上代码中，使用到了Netty中一个非常方便的类——**Unpooled帮助类，用它来创建和使用非池化的缓冲区**。另外，<u>还可以在Netty程序之外独立使用Unpooled帮助类</u>。

#### 6.7.10 ByteBuf的自动释放

​	在入站处理时，Netty是何时自动创建入站的ByteBuf的呢？

​	查看Netty源代码，我们可以看到，Netty的Reactor反应器线程会在底层的Java NIO通道读数据时，也就是AbstractNioByteChannel.NioByteUnsafe.read()处，调用ByteBufAllocator方法，创建ByteBuf实例，从**操作系统缓冲区**把数据读取到ByteBuf实例中，然后调用pipeline.fireChannelRead(byteBuf)方法将读取到的数据包送入到入站处理流水线中。

​	再看看入站处理时，入站的ByteBuf是如何自动释放的。

方式一：TailHandler自动释放

​	**Netty默认会在ChannelPipeline通道流水线的最后添加一个TailHandle末尾处理器**，它实现了默认的处理方法，在这些方法中会帮助完成ByteBuf内存释放的工作。

​	在默认情况下，如果每个InboundHandler入站处理器，把最初的ByteBuf数据包一路往下转，那么TailHandler末尾处理器会自动释放掉入站的ByteBuf实例。

​	如何让ByteBuf数据包通过流水线一路向后传递呢？

​	如果自定义的InboundHandler入站处理器继承自ChannelInboundHandlerAdapter适配器，那么可以在InboundHandler的入站处理方法中调用基类的入站处理方法。演示代码如下：

```java
public class DemoHandler extends ChannelInboundHandlerAdapter {
    /**
      * 出站处理方法
      * @param ctx 上下文
      * @param msg 入站数据包
      * @throws Exception 可能抛出的异常
      */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf byteBuf = (ByteBuf)	msg;
        // ...省略ByteBuf的业务处理
        // 自动释放ByteBuf的方法：调用父类的入站方法，将msg向后传递
        // super.channelRead(ctx,msg);
    }
}
```

​	总体来说，如果自定义的InboundHandler入站处理器继承自ChannelInboundHandlerAdapter适配器，那么可以调用以下两种方法来释放ByteBuf内存；

（1）手动释放ByteBuf。具体的方式为调用byteBuf.release()。

（2）调用父类的入站方法将msg向后传递，以来后面的处理器释放ByteBuf。具体的方式为调用基类的入站处理方法super.channelRead(ctx,msg)。

```java
public class DemoHandler extends ChannelInboundHandlerAdapter {
    /**
      * 出站处理方法
      * @param ctx 上下文
      * @param msg 入站数据包
      * @throws Exception 可能抛出的异常
      */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf byteBuf = (ByteBuf)	msg;
        // ...省略ByteBuf的业务处理
        // 释放ByteBuf的两种方法
        // 方法一：手动释放ByteBuf
        byteBuf.release();
        
        // 方法二：调用父类的入站方法，将msg向后传递
        // super.channelRead(ctx,msg);
    }
}
```

方式二：SimpleChannelInboundHandler自动释放

​	**如果Handler业务处理器需要截断流水线的处理流程，不将ByteBuf数据包送入后边的InboundHandler入站处理器，这时，流水线末端的TailHandler末尾处理器自动释放缓冲区的工作自然就失效了**。

​	在这种场景下，Handler业务处理器有两种选择：

+ 手动释放ByteBuf实例。
+ **继承SimpleChannelInboundHandler，利用它的自动释放功能**。

​	这里，我们聚焦的是第二种选择：看看SimpleChannelInboundHandler是如何自动释放的。以入站读数据为例，Handler业务处理器必须继承自SimpleChannelInboundHandler基类。并且，<u>业务处理器的代码必须移动到重写的channelRead0(ctx，msg)方法中</u>。SimpleChannelInboundHandler类的channelRead等入站处理方法，会在调用完实际的channelRead0后，帮忙释放ByteBuf实例。

​	如果大家好奇，想看看SimpleChannelInboundHandler是如何释放ByteBuf的，那么就一起看看Netty源代码。

​	截取部分代码如下所示：

```java
// public abstract class SimpleChannelInboundHandler<I> extends ChannelInboundHandlerAdapter
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        boolean release = true;
        try {
            if (this.acceptInboundMessage(msg)) {
                // 调用实际的业务代码，必须由子类继承，并且提供实现
                this.channelRead0(ctx, msg);
            } else {
                release = false;
                ctx.fireChannelRead(msg);
            }
        } finally {
            if (this.autoRelease && release) {
                ReferenceCountUtil.release(msg);
            }

        }

    }

protected abstract void channelRead0(ChannelHandlerContext var1, I var2) throws Exception;
```

​	在Netty的SimpleChannelInboundHandler类的源代码中，执行完由子类继承的channelRead0()业务处理后，在<u>finally语句代码段中，ByteBuf被释放了一次，如果ByteBuf计数器为零，将被彻底释放掉</u>。

​	在看看出站处理时，Netty是何时释放出站的ByteBuf的呢？

​	出站缓冲区的自动释放方式：HeadHandler自动释放。**在出站处理流程中，申请分配到的ByteBuf主要是通过HeadHandler完成自动释放的**。

​	出站处理用到的ByteBuf缓冲区，一般是要发送的消息，通常由Handler业务处理器所申请而分配的。例如，在write出站写入通道时，通过调用ctx.writeAndFlush(Bytebufmsg)，ByteBuf缓冲区进入出站处理的流水线。在每一个出站Handler业务处理器中的处理完成后，最后数据包（或消息）会来到出站处理的最后一棒HeadHandler，在数据输出完成后，ByteBuf会释放一次，如果计数器为零，将被彻底释放掉。

​	**在Netty开发中，必须密切关注ByteBuf缓冲区的释放，如果释放不及时，会造成Netty的内存泄漏（Memory Leak），最终导致内存耗尽**。

### 6.8 ByteBuf浅层复制的高级使用方式

​	首先要说明下，浅层复制是一种非常重要的操作。可以很大程度地避免内存复制。这一点对于大规模消息通信来说是非常重要的。

​	ByteBuf的浅层复制分为两种，有**切片（slice）浅层复制**和**整体（duplicate）浅层复制**

#### 6.8.1 slice切片浅层复制

​	ByteBuf的slice方法可以获取到一个ByteBuf的一个切片。一个ByteBuf可以进行多次的切片浅层复制；多次切片后的ByteBuf对象可以共享一个存储区域。

​	slice方法有两个重载版本：

（1）public ByteBuf slice()

（2）public ByteBuf slice(int index, int length)

​	第一个是不带参数的slice方法，在内部是调用了第二个带参数的slice方法，调用大致方式为：buf.slice(buf.readerIndex(), buf.readableBytes())。也就是说，<u>第一个无参数slice方法的返回值是ByteBuf实例中可读部分的切片</u>。

​	第二个带参数的slice(int index, int length)方法，可以通过灵活地设置不同起始位置和长度，来获取到ByteBuf**不同区域**的切片。

​	一个简单的slice的使用示例代码如下：

```java
public class SliceTest {
    @Test
    public  void testSlice() {
        ByteBuf buffer = ByteBufAllocator.DEFAULT.buffer(9, 100);
        System.out.println("动作：分配 ByteBuf(9, 100)" + buffer);
        buffer.writeBytes(new byte[]{1, 2, 3, 4});
        System.out.println("动作：写入4个字节 (1,2,3,4)" + buffer);
        ByteBuf slice = buffer.slice();
        System.out.println("动作：切片 slice" + slice);
    }
}
```

​	输出：

```java
动作：分配 ByteBuf(9, 100)PooledUnsafeDirectByteBuf(ridx: 0, widx: 0, cap: 9/100)
动作：写入4个字节 (1,2,3,4)PooledUnsafeDirectByteBuf(ridx: 0, widx: 4, cap: 9/100)
动作：切片 sliceUnpooledSlicedByteBuf(ridx: 0, widx: 4, cap: 4/4, unwrapped: PooledUnsafeDirectByteBuf(ridx: 0, widx: 4, cap: 9/100))
```

​	如果输出isReadable()、readerIndex()等，可得下表结果

|                          | 原ByeteBuf | slice切片   |
| ------------------------ | ---------- | ----------- |
| isReadable()             | true       | true        |
| readerIndex()            | 0          | 0           |
| readableBytes()          | 4          | 4           |
| ***isWritable()***       | ***true*** | ***false*** |
| writerIndex()            | 4          | 4           |
| ***writableBytes()***    | ***5***    | ***0***     |
| ***capacity()***         | ***9***    | ***4***     |
| ***maxCapacity()***      | ***100***  | ***4***     |
| ***maxWritableBytes()*** | ***96***   | ***0***     |

​	调用slice()方法后，返回的切片是一个新的ByteBuf对象，该对象的几个重要属性值，大致如下：

+ readerIndex（读指针）的值为0。
+ writerIndex（写指针）的值为源ByteBuf的readableBytes()可读字节数。
+ maxCapacity（最大容量）的值为源ByteBuf的readableBytes()可读字节数。

​	切片后的新ByteBuf有两个特点：

+ 切片不可以写入，原因是：maxCapacity与writerIndex值相同。
+ 切片和源ByteBuf的可读字节数相同，原因是：切片后的可读字节书为自己的属性writerIndex-readerIndex，也就是源ByteBuf的readableBytes()-0；

​	**切片后的新ByteBuf和源ByteBuf的关联性：**

+ **切片不会复制源ByteBuf的底层数据，底层数组和源ByteBuf的底层数组是同一个。**
+ **切片不会改变源ByteBuf的引用计数**。

​	<u>从根本上说，slice()无参数方法所生成的切片就是**源ByteBuf可读部分的浅层复制**</u>。

#### 6.8.2 duplicate整体浅层复制

​	**和slice切片不同，duplicate()返回的是源ByteBuf的整个对象的一个浅层复制**，包括如下内容：

+ duplicate的读写指针、最大容量值，与源ByteBuf的读写指针相同。
+ **duplicate()不会改变源ByteBuf的引用计数。**
+ **duplicate()不会复制源ByteBuf的底层数据。**

​	duplicate()和slice()方法都是浅层复制。不同的是，slice()方法是切取一段(只有可读部分)的浅层复制，而duplicate()是整体的浅层复制。

#### 6.8.3 浅层复制的问题

​	浅层复制方法不会实际去复制数据，也不会改变ByteBuf的引用计数，这就会导致一个问题：在源ByteBuf调用release()之后，一旦引用计数为零，就变得不能访问了；在这种场景下，源ByteBuf的所有浅层复制实例也不能进行读写了；如果强行对浅层复制实例进行读写，则会报错。

​	<u>因此，在调用浅层复制实例时，可以通过调用一次retain()方法来增加引用，表示它们对应的底层内存多了一次引用，引用计数为2。在浅层复制实例用完后，需要调用两次release()方法，将引用计数减一，这样就不影响源ByteBuf的内存释放</u>。

### 6.9 EchoServer回显服务器的实践案例

#### 6.9.1 NettyEchoServer回显服务器的服务器端

​	前面实现过Java NIO版本的EchoServer回显服务器，在学习了Netty后，这里为大家设计和实现一个Netty版本的EchoServer回显服务器。功能很简单：从服务器读取客户端输入的数据，然后将数据直接回显到Console控制台。

​	首先是服务器端的实践案例，目标为掌握以下知识：

+ 服务器端ServerBootstrap的装配和使用。
+ 服务器端NettyEchoServerHandler入站处理器的channelRead入站处理方法的编写。
+ Netty的ByteBuf缓冲区的读取、写入，以及ByteBuf的引用计数的查看。

​	服务器端的ServerBootstrap装配和启动过程，代码如下：

```java
public class NettyEchoServer {
    // ...
	 public void runServer() {
        //创建reactor 线程组
        EventLoopGroup bossLoopGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerLoopGroup = new NioEventLoopGroup();
     	// ... 省略设置：1 反应器线程组 2 通道类型 3 设置监听接口 4 通道选项等
       	// 5 装配子通道流水线
        b.childHandler(new ChannelInitializer<SocketChannel>() {
            // 有连接到达时会创建一个通道
            protected void initChannel(SocketChannel ch) throws Exception {
                // 流水线管理子通道中的Handler业务处理器
                // 向子通道流水线添加一个Handler业务处理器
                ch.pipeline().addLast(NettyEchoServerHandler.INSTANCE);
            }
        });
        // ... 省略启动、等待、从容关闭（或称为优雅关闭）等
     }
    // ... 省略main方法
}
```

#### 6.9.2 共享NettyEchoServerHandler处理器

​	Netty版本的EchoServerHandler回显服务器处理器，继承自ChannelInboundHandlerAdapter，然后覆盖了channelRead方法，这个方法在可读IO事件到来时，被流水线回调。

​	这个回显服务器处理器的逻辑分为两步：

​	第一步，从channelRead方法的msg参数。

​	第二步，调用ctx.channel().writeAndFlush()把数据写回客户端。

​	先看第一步，读取从对端输入的数据。channelRead方法的msg参数的形参类型不是ByteBuf，而是Object，为什么呢？实际上，**msg的参数类型是由流水线的上一站决定的**。大家知道，入站处理的流程是：Netty读取底层的二进制数据，填充到msg时，msg是ByteBuf类型，然后经过流水线，传入到第一个入站处理器；每一个节点处理完后，将自己的处理结果（类型不一定是ByteBuf）作为msg参数，不断向后传递。因此，msg参数的形参类型，必须是Object类型。不过，可以肯定的是，**第一个入站处理器的channelRead方法的msg实参类型，绝对是ByteBuf类型**，因为它是Netty读取到的ByteBuf数据包。在本实例中，NettyEchoServerHandler就是第一个业务处理器，虽然msg实参类型是Object，但是实际类型就是ByteBuf，所以可以强制转换成ByteBuf类型。

​	另外，**从Netty4.1开始，ByteBuf的默认类型是Direct ByteBuf直接内存**(6.7.8)。大家知道，Java不能直接访问Direct ByteBuf内部的数据，必须先通过getBytes、readBytes等方法，将数据读入Java数组中，然后才能继续在数组中进行处理。

​	第二步将数据写回客户端。这一步很简单，直接复用前面的msg实例即可。不过要注意，如果上一步使用的readBytes，那么这一步就不能直接将msg写回了，因为数据已经被readBytes读完了。幸好，**上一步调用的读数据方法是getBytes，他不影响ByteBuf的数据指针，因此可以继续使用**(6.7.4)。这一步调用了ctx.writeAndFlush，把msg数据写回客户端。也可调用ctx.channel().writeAndFlush()方法。这两个方法在这里的效果是一样的。因为这个流水线上没有任何的出站处理器。

*ps：如果有其他出站处理器，如果调用ctx.writeAndFlush是走整个出站流水线，而ctx.channel().writeAndFlush()是从当前出站处理器开始接下去依次走出站流水线（出站和入站流水线的走向相反，出站是从后往前类似栈，入站流水线是从前往后类似队列，两者实际上的数据结构本质都是双向链表）***（6.6.3 有介绍Channel、ChannelPipeline、ChannelHandlerContext执行入站、出站方法的效果区别）**

​	服务器端的入站处理器NettyEchoServerHandler的代码如下：

```java
@ChannelHandler.Sharable
public class NettyEchoServerHandler extends ChannelInboundHandlerAdapter {
    public static final NettyEchoServerHandler INSTANCE = new NettyEchoServerHandler();

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf in = (ByteBuf) msg;
        Logger.info("msg type: " + (in.hasArray()?"堆内存":"直接内存"));

        int len = in.readableBytes();
        byte[] arr = new byte[len];
        in.getBytes(0, arr);
        Logger.info("server received: " + new String(arr, "UTF-8"));

        //写回数据，异步任务
        Logger.info("写回前，msg.refCnt:" + ((ByteBuf) msg).refCnt());
        ChannelFuture f = ctx.writeAndFlush(msg);
        f.addListener((ChannelFuture futureListener) -> {
            Logger.info("写回后，msg.refCnt:" + ((ByteBuf) msg).refCnt());
        });
    }
}
```

​	这里的NettyEchoServerHandler在前面加了一个特殊的Netty注解：**@ChannelHandler.Sharable**。**这个注解的作用是标注一个Handler实例可以被多个通道安全地共享**。什么叫作Handler共享呢？就是多个通道的流水线可以加入同一个Handler业务处理器实例。而这种操作，**Netty默认是不允许的**。

​	但是，很多应用场景需要Handler业务处理器实例能共享。例如，一个服务器处理十万以上的通道，如果一个通道新建很多重复的Handler实例，就需要上十万以上重复的Handler实例，这就会浪费很多宝贵的空间，降低了服务器的性能。所以，**如果在Handler实例中，没有与特定通道强相关的数据或者状态，建议设计成共享的模式**：在前面加了一个Netty注解：@ChannelHandler.Sharable。

​	<u>反过来，如果没有加@ChannelHandler.Sharable注解，视图将同一个Handler实例添加到多个ChannelPipeline通道流水线时，Netty将会抛出异常</u>。

​	还有一个隐藏比较深的重点：**同一个通道上的所有业务处理器，只能被同一个线程处理**。所以，不是@Sharable共享类型的业务处理器，在线程层面是安全的，不需要进行线程的同步控制。而不同的通道，可能绑定到多个不同的EventLoop反应器线程。因此，加上了@ChannelHandler.Sharable注解后的共享业务处理器的实例，可能别多个线程并发执行。这样，就会导致一个结果：@Sharable共享实例不是线程层面安全的。显而易见，**@Sharable共享的业务处理器，如果需要操作的数据不仅仅是<u>局部变量</u>，则需要进行线程的同步控制，以保证操作是线程层面安全的**。

​	如何判断一个Hnadler是否为@Sharable共享呢？ChannelHandlerAdapter提供了实用方法——isSharable()。如果其对应的实现加上了@Sharable注解，那么这个方法将返回true，表示它可以被添加到多个ChannelPipeline通道流水线中。

​	NettyEchoServerHandler回显服务器处理器没有保存与任何通道连接相关的数据，也没有内部的其他数据需要保存。所以，它不光是可以用来共享，而且不需要做任何的同步控制。在这里，为它加上了@Sharable注解表示可以共享，更进一步，这里还设计了一个通用的INSTANCE静态实例，所有的通道直接使用这个INSTANCE实例即可。

​	最后，揭示一个比较奇怪的问题。

​	运行程序，大家会看到在写入客户端的工作完成后，ByteBuf的引用计数的值变为0.在上面的代码中，既没有自动释放的代码，也没有手动释放的带啊吗，为什么，引用计数就没有了呢？

​	这个问题，比较有意思，留给大家自行思考。答案，就藏在上文之中。

​	*ps:6.7.10 ByteBuf自动释放有提到入站有两种自动释放方式（法1：Netty默认在ChannelPipeline通道流水线最后添加一个TailHandler末尾处理器；法2：Handler业务处理器继承SimpleChannelInboundHandler基类）；出站缓冲区的自动释放则是Netty自动添加的HeadHandler释放。*

#### 6.9.3 NettyEchoClient客户端代码

​	其次是客户端的实践案例，目标为掌握以下知识：

+ 客户端Bootstrap的装配和使用
+ 客户端NettyEchoClientHandler入站处理器中，接收回写的数据，并且释放内存。
+ 有多种方式用于释放ByteBuf，包括：自动释放、手动释放。

​	客户端Bootstrap的装配和使用，代码如下：

```java
public class NettyEchoClient {

    private int serverPort;
    private String serverIp;
    Bootstrap b = new Bootstrap();

    public NettyEchoClient(String ip, int port) {
        this.serverPort = port;
        this.serverIp = ip;
    }

    public void runClient() {
        //创建reactor 线程组
        EventLoopGroup workerLoopGroup = new NioEventLoopGroup();

        try {
            //1 设置reactor 线程组
            b.group(workerLoopGroup);
            //2 设置nio类型的channel
            b.channel(NioSocketChannel.class);
            //3 设置监听端口
            b.remoteAddress(serverIp, serverPort);
            //4 设置通道的参数
            b.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);

            //5 装配子通道流水线
            b.handler(new ChannelInitializer<SocketChannel>() {
                //有连接到达时会创建一个channel
                protected void initChannel(SocketChannel ch) throws Exception {
                    // pipeline管理子通道channel中的Handler
                    // 向子channel流水线添加一个handler处理器
                    ch.pipeline().addLast(NettyEchoClientHandler.INSTANCE);
                }
            });
            ChannelFuture f = b.connect();
            f.addListener((ChannelFuture futureListener) ->
            {
                if (futureListener.isSuccess()) {
                    Logger.info("EchoClient客户端连接成功!");

                } else {
                    Logger.info("EchoClient客户端连接失败!");
                }

            });

            // 阻塞,直到连接完成
            f.sync();
            Channel channel = f.channel();

            Scanner scanner = new Scanner(System.in);
            Print.tcfo("请输入发送内容:");

            while (scanner.hasNext()) {
                //获取输入的内容
                String next = scanner.next();
                byte[] bytes = (Dateutil.getNow() + " >>" + next).getBytes("UTF-8");
                //发送ByteBuf
                ByteBuf buffer = channel.alloc().buffer();
                buffer.writeBytes(bytes);
                channel.writeAndFlush(buffer);
                Print.tcfo("请输入发送内容:");

            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 优雅关闭EventLoopGroup，
            // 释放掉所有资源包括创建的线程
            workerLoopGroup.shutdownGracefully();
        }

    }
    public static void main(String[] args) throws InterruptedException {
        int port = NettyDemoConfig.SOCKET_SERVER_PORT;
        String ip = NettyDemoConfig.SOCKET_SERVER_IP;
        new NettyEchoClient(ip, port).runClient();
    }
}
```

​	在上面的代码中，客户端在连接到服务器端成功后不断循环，获取控制台的输入，通过服务器端的通道发送到服务器。

#### 6.9.4 NettyEchoClientHandler处理器

​	客户端的流水线不是空的，还需要装配一个回显处理器，功能很简单，就是接收服务器写过来的数据包，显示在Console控制台上。代码如下：

```java
@ChannelHandler.Sharable
    /**
     * 出站处理方法
     * 
     * @param ctx 上下文
     * @param msg 入站数据包
     * @throws Exception 可能抛出的异常
     */
public class NettyEchoClientHandler extends ChannelInboundHandlerAdapter {
    public static final NettyEchoClientHandler INSTANCE = new NettyEchoClientHandler();

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf in = (ByteBuf) msg;
        int len = in.readableBytes();
        byte[] arr = new byte[len];
        in.getBytes(0, arr);
        Logger.info("client received: " + new String(arr, "UTF-8"));
        
        // 释放ByteBuf的两种方法
        // 方法一：手动释放ByteBuf
        in.release();
        
        // 方法二：调用父类的入站方法，将msg向后传递
        // super.channelRead(ctx,msg);
    }
}
```

​	通过代码可以看到，从服务器端发送过来的ByteBuf，被手动方式强制释放掉了。当然，也可以使用前面的介绍的自动释放方式来释放ByteBuf。

​	*ps:这里不是继承SimpleChannelInboundHandler实现的inboundHandler，所以如果要调用自动释放的ByteBuf.release()就必须调用父类的channelRead()，这样Netty默认最后加上TailHandler自动释放ByteBuf这一操作才会被执行（因为被截断就没意义了）。*

### 6.10 本章小结

​	本章详细介绍了Netty的基本原理：Reactor反应器模式在Netty中的应用，Netty中Reactor反应器、Handler业务处理器、Channel通道以及它们三者之间的相互关系。另外，Netty为了有效地管理通道和Handler业务处理器之间的关系，还引入了一个重要组件——Pipeline流水线。

​	如果第4章的Reactor反应器模式，大家理解得比较清晰了，那么掌握Netty的基本原理，其实就是一件非常简单的事件。

​	本章还介绍了Netty的ByteBuf缓冲区的使用，这也是使用Netty需要掌握的一项非常基础的知识。ByteBuf的入门不难，真正用好的话，还是有蛮多学问的，需要不断地积累经验。

​	防止内存泄漏（Memory Leak），不仅仅是Java的难题，也是Netty的难题，它不仅仅是技术问题，也是经验问题。针对这个问题，疯狂创客圈社群通过博客或者视频的方式，总结了Netty的内存使用以及Netty内存泄漏的排查方法。

