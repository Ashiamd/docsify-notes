# 《Netty、Redis、ZooKeeper高并发实战》学习笔记

> 最近一段时间学习IM编程，但是之前急于需要成品，所以学得不是很懂，最后也没能弄好，所以打算系统地学习一下，然后再继续IM编程。

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

   两者都不是直接将数据从物理设备到内存或者相反。上层应用调用操作系统的read和write都会设计缓冲区。

   + 调用操作系统的read

     把数据从**内存缓冲区**复制到**进程缓冲区**

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

### 6.5 详解Handler业务处理器

​	