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