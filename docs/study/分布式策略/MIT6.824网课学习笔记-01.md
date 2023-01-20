# MIT6.824网课学习笔记-01

> [【MIT 6.824】学习笔记 1： MapReduce (qq.com)](https://mp.weixin.qq.com/s/I0PBo_O8sl18O5cgMvQPYA) <= 2021版网络大佬的笔记
>
> [简介 - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/) <= 知乎大佬的gitbook完整笔记(2020版，我这里看的网课是2021版)，推荐也阅读一遍2020版课程的大佬笔记
>
> [6.824](https://pdos.csail.mit.edu/6.824/schedule.html) <= 课程表

# Lecture1 概述(Introduction)

## 1.1 分布式系统-简述

​	非正式的定义，这里认为几个只能通过网络发送/接收数据包进行交互的系统，即构成分布式系统。

​	使用分布式系统的场景/原因：

+ 机器资源共享 (sharing)
+ 通过并行提高性能 (increase capacity though parallelism)
+ **提高服务容错性 (tolerate faults)**

+ 利用物理隔离的手段提高整体服务安全性 (achieve security via isolation)

## 1.2 分布式系统-历史发展

+ 局域网分布式系统的服务 (1980s)
+ 互联网规模的分布式系统，比如DNS（域名服务系统）、email

+ 数据中心 (Data center)，伴随大型网站而生 (1990s)
  + 常见的有网页搜索（爬虫实现，然后需要建立大量倒排索引）
  + 商城购物系统（海量商品、订单、用户信息等数据）
+ 云计算 (Cloud computing) (2000s)
  + 本地运算/本地运行应用，转移到云服务上运算/运行应用
+ 如今也是研究和发展的热点话题

## 1.3 分布式系统-难点

> 网络分区：
>
> [Network Partition_百度百科 (baidu.com)](https://baike.baidu.com/item/Network Partition/8720737?fr=aladdin)
>
> [尝试正确理解网络分区 - 简书 (jianshu.com)](https://www.jianshu.com/p/275c645f453c)
>
> 那么对于一个n个节点组成的网络来说，如果n 个节点可以被分为k个不相交且覆盖的group, 每个group内所有节点全是两两正常连接，而任意两个group之间的任何节点无连接。当k=1 时，网络正常，当k > 1 时，我们称之为network partition。
>
> [《RabbitMQ实战指南》整理（七）网络分区 - Jscroop - 博客园 (cnblogs.com)](https://www.cnblogs.com/Jscroop/p/14382790.html)
>
> RabbitMQ集群内两两节点之间会有信息交互，如果某节点出现网络故障或是端口不同，则会使与此节点的交互出现中断，经过超时判定后，会判定网络分区。网络分区的判断与net_ticktime参数息息相关，其默认值为60。集群内部每个节点间会每隔四分之一net_ticktime记一次应答，如果任何数据被写入节点中，则此节点会被认为已经被应答了，如果连续4此没有应答，则会判定此节点已下线，其余节点可以将此节点剥离出当前分区。
>
> 对于未配置镜像的集群，网络分区发生之后，队列也会随着宿主节点而分散在各自的分区中。对于消息发送方而言，可以成功发送消息，但是会有路由失败的情况，需要配合mandatory等机制保障消息的可靠性，**对于消息消费方而言，可能会出现不可预知的现象，比如已消费消息ack会失效**。网络分区发生后，客户端与某分区重新建立通信链路，其分区中如果没有相应的队列进程，则会有异常报出。**如果从网络分区中恢复，数据不会丢失，但是客户端会重复消费**。
>
> 对于已镜像的集群，**网络分区的发生会引起消息的丢失**，解决办法为消息发送端需要具备Basic.Return的能力，其次在检测到网络分区之后，需要迅速地挂起所有生产者进程，之后连接分区中的每个节点消费分区中所有的队列数据，在消费完之后再处理网络分区，最后从网络分区中恢复之后再恢复生产者进程。整个过程可以最大程度上保证网络分区之后的消息的可靠性。**需要注意的是整个过程中会有大量的消息重复，消费客户端需要做好相应的幂等性处理**。也可以将所有旧集群资源迁移到新集群来解决这个问题。
>
> ---

+ 许多并发场景 (many concurrent part)

+ **需要处理故障/宕机问题 (must deal with partial failure)**

  **尤其是发生网络分区问题 (network partition)**

+ 体现性能优势 (realize the performance benefits)

  通常任务并非真正所有步骤都并行执行，需要良好的实现才能达到增加机器数就提高吞吐量的效果。

## 1.4 课程介绍

实验内容：

+ Lab1：实现mapreduce

+ Lab2：在存在故障和网络分区的情况下，使用raft协议完成复制
+ Lab3：通过实验2，复制一个Key-Value存储的服务
+ Lab4：构造分片(sharded)的key-value存储服务

## 1.5 支持分布式系统的底层基础架构

​	课程不会花大量时间在分布式的应用程序上，而是着重介绍支撑这些分布式应用程序开发的底层基础架构。常见的有：

+ Storage，存储基础架构，比如键值服务器、文件系统等
+ Computation，计算框架，用来编排或构件分布式应用程序，比如经典的mapreduce
+ Comminication，分布式系统逃不开网络通信问题，比如后续会讨论的RPC(远程过程调用)

​	需要关注的是RPC提供的语义（同时也是分布式系统和通信关联的部分）：

+ **utmost once：最多一次**
+ **exactly once：恰好一次**
+ **at least once：至少一次**

​	**对于分布式系统的底层基础架构，我们通常抽象的目标是做到让使用者觉得和单机操作无意，这是非常难实现的**。（也就是隐藏分布式中各类难题的具体实现，对外暴露时争取和普通本地串行函数别无二致。）

## 1.6 分布式系统的3个重要特性

> [尾部延迟是什么？如何避免尾部延迟？ | 程序员技术之旅 (zhangbj.com)](https://www.zhangbj.com/p/769.html) <= 简述概念和产生原因
>
> 有`1%`的请求耗时高于`99%`的请求耗时，影响用户体验，甚至拖垮服务。

+ fault tolerance：容错性
  +  availability：可用性，一般用p999等指标衡量
    + 主要依赖replication复制技术
  + recoverability：可恢复性，当机器崩溃或故障时，在重启时恢复正常工作状态
    + 主要依赖logging or transaction日志或事务一类的技术
    + durable storage，需要将数据写入持久化的存储器，便于后续恢复工作

+ consistency：一致性，即多个服务应该提供一致的服务，客户端请求哪个服务端获取的响应需要一致

  关于一致性，有很多不同实现方式：

  + 强一致性：多个机器执行的行为，就类似串行执行
  + 最终一致性：较为宽松，多个行为执行后，只需要保证最后一个行为的结果在多台服务器都得到相同的表现

+ performance：分布式系统往往希望能比单机系统要具备更高的性能，但是提高性能本身和提供容错性、一致性是冲突的。

  性能一般涉及两个指标：

  + throughput 吞吐量：目标是吞吐量与部署的机器数成正比
  + <u>latency 低延迟：其中一台机器执行慢就会导致整个请求响应慢，这称为**尾部延迟(tail latency)**</u>

---

​	为了达到强一致性，需要不同机器之间的通信，这可能降低性能

​	为了实现容错，需要从不同机器上复制数据，还需要将数据写入持久化存储器这一昂贵操作

​	通常要兼顾以上3个特性很难做到，常见的实现要么牺牲一点一致性换取更高的性能；要么牺牲一点容错性换取更好的性能，不同的实现方式有不同的权衡。

## 1.7 mapreduce-简述

> [MapReduce_百度百科 (baidu.com)](https://baike.baidu.com/link?url=A72Qje56iqpXJP6T5k4W8wUkld3Su6xG6jUUNzuUIiATyQNmYaE6PrnSrYV-7wofMpwF7n5VV0a0aV0NzSm1XRs9ggFXCcqEw-RwbjatMEa)
>
> MapReduce是一种编程模型，用于大规模数据集（大于1TB）的并行运算。概念"Map（映射）"和"Reduce（归约）"，是它们的主要思想，都是从函数式编程语言里借来的，还有从矢量编程语言里借来的特性。它极大地方便了编程人员在不会分布式并行编程的情况下，将自己的程序运行在[分布式系统](https://baike.baidu.com/item/分布式系统/4905336?fromModule=lemma_inlink)上。 当前的软件实现是指定一个Map（映射）函数，用来把一组键值对映射成一组新的键值对，指定并发的Reduce（归约）函数，用来保证所有映射的键值对中的每一个共享相同的键组。
>
> [MapReduce(一)：MapReduce简述 - 简书 (jianshu.com)](https://www.jianshu.com/p/748359a7385f)
>
> [什么是MapReduce(入门篇)_大数据梦想家的博客-CSDN博客_mapreduce](https://blog.csdn.net/weixin_44318830/article/details/103041840)

​	论文的背景是google的两位数据工程师需要处理爬虫数据，建立倒排索引，用于网页搜索。需要几个小时处理TB级别的数据。传统的代码实现，并发执行任务，如果中间一个线程/线程出错，可能整个任务都需要重新执行，所以还需要考虑容错性设计。

​	程序员通过编写函数式或无状态的map函数和reduce函数的实现，实现分布式数据的处理。mapreduce内部通过其他机制保证执行过程的容错性、分布式通信等问题，对程序员隐藏这些细节。

​	map-reduce经典举例即统计字母出现的次数，多个进程各自通过map函数统计获取到的数据片段的字母的出现次数；后续再通过reduce函数，汇总聚合map阶段下每个进程对各自负责的数据片段统计的字母出现次数。一旦执行了shuffle，多个reduce函数可以各自只聚合一种字母的出现总次数，彼此之间不干扰。

​	开销昂贵的部分即shuffle，map的结果经过shuffle按照一定的顺序整理/排序，然后才分发给不同的reduce处理。这里shuffle的操作理论比map、reduce昂贵。

---

提问：排序操作是否可以通过map-reduce完成

回答：可以，排序在mapreduce中是讨论最多的应用之一，可以通过mapreduce实现。你可以将输入拆分成不同部分，mapper对这些部分进行排序，输出拆分成r个桶，每个reduce对这r个桶进行排序，最后输出完整的文件。

## 1.8 mapreduce-执行流程

> [【MIT 6.824】学习笔记 1： MapReduce (qq.com)](https://mp.weixin.qq.com/s/I0PBo_O8sl18O5cgMvQPYA) <= 图片出处

![Image](https://mmbiz.qpic.cn/mmbiz_png/hBL5R2neMA1ynq0HZhJ5kup6vibWOUCRs4ZIhdia6NzgB6g5Hem4Zy9pHqPEzHUwLgB96M2kmQ7elmTQtnqkE3AA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

+ Master进程，被称为coordinator协调器，负责orchestrate编排wokers，把map jobs分配给它们
+ reduce、map被称为task任务

1. coordinator协调器将文件分配给特定的workers，worker对分配到的文件调用map函数

2. worker将执行map函数产生的中间结果存储到本地磁盘

3. worker的map函数执行完毕后并告知master中间结果存储的位置
4. 所有worker的map执行完毕后，coordinator协调器分配worker执行reduce函数
5. worker记录分配到的map中间结果，获取数据，按key键sort排序，在每个key、values集合上调用reduce函数
6. 每个reduce函数执行时产生结果数据，你可以聚合输出文件获取最终结果

​	输入文件在全局文件系统中，被称为GFS。Google现在使用的是不同的global file system，但该论文中使用的是GFS。

​	上面流程最后reduce输出结果会被保存到GFS，而map产生的中间文件不会被保存到GFS中（而是保存到worker运行的本地机器上）。

---

问题：在远程读取进程中，文件是否会传输到reducer？

回答：是的。map函数产生的中间结果存放在执行map函数的worker机器的磁盘上，而之后解调器分配文件给reducer执行reduce函数时，中间结果数据需要通过网络传输到reducer机器上。这里其实很少有网络通信，因为一个worker在一台机器上，而每台机器同时运行着worker进程和GFS进程。worker运行map产生中间结果存储在本地，而之后协调器给worker分配文件以执行reduce函数时，才需要通过网络获取中间结果数据，最后reduce处理完在写入GFS，写入GFS的动作也往往需要通络传输。

**问题：协调器是否负责对数据进行分区，并将数据分发到每个worker或机器上？**

**回答：不是的。mapreduce运行用户程序，这些输入数据在GFS中。（也就是说协调器告知worker从GFS取哪些数据进行map，后续协调器又告知worker从哪些worker机器上获取中间结果数据进行reduce，最后又统一写入到GFS中）**

问题：这里涉及的排序是如何工作的？比如谁负责排序，如何排序？

回答：在中间结果数据传递到reduce函数之前，mapreduce库进行一些排序。比如所有的中间结果键a、b、c到一个worker。比如`(a,1) (b,1) (c,1) (a,1)` 数据，被排序成`(a,1) (a,1) (b,1) (c,1) `后才传递给reduce函数。

问题：很多函数式编程是否可以归结为mapreduce问题？

回答：是的。因为map、reduce函数的概念，在函数式编程语言中非常常见，或者说函数式编程真是map、reduce的灵感来源。

## 1.9 mapreduce-容错性

​	当coordinator协调器发现worker失败了，会restart重启对应的task任务，并要求worker重新执行原本分配的map或reduce函数。

​	如果coordinator协调器分配任务后，其管理的worker机器在一定时间内没有响应，协调器就会假设管理的worker机器崩溃了。这意味着如果存在另一个worker正好空闲，它将被分配崩溃的worker原本该执行的任务。这就是fault tolerance容错的基本方案。

​	换句话说，如果coordinator没有收到worker反馈task任务完成，那么会coordinator重新分配worker要求执行task（可能分配到同一个worker，重点是task会被重新执行）。**这里涉及一个问题，map是否会被执行两次？**

​	或许没反馈task执行done完成的worker是遇到网络分区等问题，并没有宕机，或者协调者不能与worker达成网络通信，但实际上worker仍然在运行map任务，它正在产生中间结果。**这里的答案是，同一个map可以被运行两次**。

​	**被执行两次是能够接受的（幂等性问题），正是map和reduce属于函数式(functional)的原因之一**。<u>如果map/reduce是一个funcitonal program，那么使用相同输入运行时，产生的输出会是相同的（也就是保证幂等）</u>。

​	**类似的，reduce能够运行两次吗？**<u>是的，和map出于相同的原因，从容错的角度上看，执行reduce函数和map函数并没有太大区别</u>。**需要注意的是，这时候可能有两个reducer同时有相同的输出文件需要写入GFS，它们首先在全局文件系统GFS中产生一个中间文件，然后进行atomic rename原子重命名，将文件重命名为实际的最终名称**。<u>因为在GFS中执行的重命名是原子操作，最后哪个reducer胜出并不重要，因为reduce是函数式的，它们最终输出的数据都是一样的</u>。

---

**问题：一台机器应该可以执行多个map任务，如果它分配10个map任务，而在执行第7个map任务时失败了，master得知后，会安排将这7个已完成的map任务分布式地重新执行，可能分散到不同的map机器上，对吗？**

**回答：是的。虽然我认为通常一台机器只运行一个map函数或一个reduce函数，而不是多个。**

**问题：在worker完成map任务后，它是否会直接将文件写入其他机器可见的位置，或者只是将文件保存到自己的文件系统中？**

**回答：map函数总是在本地磁盘产生结果，所以中间结果文件只会在本地文件系统中。**

问题：即使一次只做一个map任务，但是如果执行了多次map任务后，如果机器突然崩溃，那么会丢失之前负责的所有map任务所产生的中间结果文件，对吗？

回答：不，中间结果文件放在文件系统中。所以当机器恢复时，中间结果文件还在那里，因为文件数据是被持久化保存的，而不是只会存在于内存中（换句话说，这里依赖了操作系统的文件系统本身的容错性）。并且map或reduce会直接访问包含intermediate results中间结果的机器。

## 1.10 mapreduce-其他异常场景

+ 协调器会失败吗（Coordinator fail）？

  不会。协调器不允许失败。如果协调器真的失败了，整个job（包含具体的多个map、reduce步骤task）需要重新运行。在这篇论文中，没有谈论到协调器失败后他们的应对方式。

  （协调器不允许失败）这使得容错性很难做得更高，因为它维护一些工作状态（每个map、reduce函数执行的状态），在论文的库中，协调器不能失败。

  后面会谈论一些技术手段，可以实现协调器容错，他们可以这么做却不打算，原因是他们认为比起协调器，运行map函数的上千个机器中崩溃一台的概率更高（也就是收益和成本不成正比，所以暂时没有实现协调器容错的打算）。

+ 执行缓慢的worker（Slow workers）？

  比如GFS也在同一台机器上运行占用大量的机器周期或带宽，或硬件本身问题，导致worker执行map/reduce很慢。**慢的worker被称为straggler，当剩下几个map/reduce任务没有执行时，协调者会另外分配相同的map/reduce任务到其他闲置worker上运行，达到backup task的效果**（因为函数式，map/reduce以相同输入执行最后会产生相同输出，所以执行多少次都不会有问题）。

  **通过backup task，性能不会受限于最慢的几个worker，因为有更快的worker会领先它们完成task（map或reduce）。<u>这是应对straggler的普遍做法，通过replicate tasks复制任务，获取更快完成task的输出结果，处理了tail latency尾部延迟问题</u>。**

# Lecture2 远程过程调用和多线程(RPC and Threads)

## 2.1 选用go语言的原因

​	这里主要讲为什么这个课程选用Go语言进行编程。

+ good support for threads/RPC：对线程和RPC的支持度高
+ gc：自带GC，无需考虑垃圾回收问题
+ type safe：类型安全
+ simple：简单易上手
+ compiled：编译型语言，运行时开销更低

## 2.2 线程Thread of exeution

> [goroutine官网简介](https://golang.google.cn/doc/effective_go#goroutines)

​	在Go中，线程成为goroutine，也有人称其为线程的线程(a thread of thread)。每个线程都有自己的**PC程序计数器、函数调用栈Stack、一套寄存器Registers**。

 	相关的primitive原语：

+ start/go：启动/运行一个线程
+ exit：线程退出，一般从某个函数退出/结束执行后，会自动隐式退出
+ stop：停止一个线程，比如向一个没有读者的channel写数据，那么channel阻塞，go可能会运行时暂时停止这个线程
+ resume：恢复原本停止的线程重新执行，需要恢复程序计数器(program counter)、栈指针(stack pointer)、寄存器(register)状态，让处理器继续运行该线程。

## 2.3 为什么需要多线程

+ express concurrency：依靠多线程达到并发效果
  + I/O concurrency：I/O并发
  + multi-core parallelism：多核并行，提高整体吞吐量
  + convinience：方便，经常有需要异步执行or定时执行的任务，可以通过线程完成

## 2.4 多线程的挑战

+ race conditions：多线程会引入竞态条件的场景
  + avoid sharing：避免共享内存以防止竞态条件场景的产生（Go有一个竞态检测器race detector，能够辅助识别代码中的一些竞态条件场景）
  + use locks：让一系列指令变成原子操作

+ coordination：同步协调问题，比如一个线程的执行依赖另一个线程的执行结果等
  + channels：通道允许同时通信和协调
  + condition variables：配合互斥锁使用
+ deadlock：死锁问题，比如在go中简单的死锁场景，一个写线程往channel写数据，但是永远没有读线程从channel读数据，那么写线程被永久阻塞，即死锁，go会抓住这种场景，抛出运行时错误runtime error。

## 2.5 go如何应对多线程挑战

 	go常见的处理多线程问题的方式有两种：

+ channels：通道
  + no-sharing场景：如果线程间不需要共享内存（变量等），一般偏向于使用channels完成线程间的通信
+ locks + condition variables：锁和条件变量配套使用
  + shared-memory：如果线程间需要共享内存，则采用锁+条件变量的方案。比如键值对key-value服务，需要共享key-value table。

## 2.6 go多线程代码示例

> 逃逸分析：
>
> [先聊聊Go的「堆栈」，再聊聊Go的「逃逸分析」。 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/586249256) <= 简单明了，推荐阅读。还有具体的代码举例分析
>
> 相比于把内存分配到堆中，分配到栈中优势更明显。Go语言也是这么做的：Go编译器会尽可能将变量分配到到栈上。但是，**当编译器无法证明函数返回的变量有没有被引用时，编译器就必须在堆上分配该变量，以此避免悬挂指针（dangling pointer）的问题。另外，如果局部变量占用内存非常大，也会将其分配在堆上**。
>
> 编译器通过**逃逸分析**技术去选择堆或者栈，逃逸分析的基本思想如下：
>
> + **检查变量的生命周期是否是完全可知的，如果通过检查，则在栈上分配**
> + **否则，就是所谓的逃逸，必须在堆上进行分配**
>
> Go语言虽然没有明确说明逃逸分析原则，但是有以下几点准则，是可以参考的。
>
> - 不同于JAVA JVM的运行时逃逸分析，Go的逃逸分析是在**编译期**完成的：编译期无法确定的参数类型**必定**放到堆中；
> - 如果变量在函数外部存在引用，则**必定**放在堆中；
> - 如果变量占用内存较大时，则**优先**放到堆中；
> - 如果变量在函数外部没有引用，则**优先**放到栈中；
>
> [tcmalloc_百度百科 (baidu.com)](https://baike.baidu.com/item/tcmalloc/2832982?fr=aladdin)
>
> tcmalloc全称Thread-Caching Malloc，即线程缓存的malloc，实现了高效的多线程内存管理，用于替代系统的内存分配相关的函数（malloc、free，new，new[]等）。
>
> [Java逃逸分析 - JaxYoun - 博客园 (cnblogs.com) ](https://www.cnblogs.com/JaxYoun/p/16424286.html)<= 推荐阅读
>
> 在计算机语言编译器优化原理中，逃逸分析是指分析指针动态范围的方法，它同编译器优化原理的指针分析和外形分析相关联。当变量（或者对象）在方法中分配后，其指针有可能被返回或者被全局引用，这样就会被其他方法或者线程所引用，这种现象称作指针（或者引用）的逃逸(Escape)。通俗点讲，如果一个对象的指针被多个方法或者线程引用时，那么我们就称这个对象的指针（或对象）的逃逸（Escape）。
>
> 1. **方法逃逸**：在一个方法体内，定义一个局部变量，而它可能被外部方法引用，比如作为调用参数传递给方法，或作为对象直接返回。或者，可以理解成对象跳出了方法。
>
> 2. **线程逃逸**：这个对象被其他线程访问到，比如赋值给了实例变量，并被其他线程访问到了。对象逃出了当前线程。
>
> 如果一个对象不会在方法体内，或线程内发生逃逸（或者说是通过逃逸分析后，认定其未能发生逃逸），就可以做如下优化：
>
> 1. **栈上分配**：
>    一般情况下，不会逃逸的对象所占空间比较大，如果能使用栈上的空间，那么大量的对象将随方法的结束而销毁，减轻了GC压力
> 2. **同步消除**：
>    如果你定义的类的方法上有同步锁，但在运行时，却只有一个线程在访问，此时逃逸分析后的机器码，会去掉同步锁运行。
> 3. **标量替换**：
>    <u>Java虚拟机中的原始数据类型（int，long等数值类型以及reference类型等）都不能再进一步分解，它们可以称为标量。相对的，如果一个数据可以继续分解，那它称为聚合量，Java中最典型的聚合量是对象</u>。如果逃逸分析证明一个对象不会被外部访问，并且这个对象是可分解的，那程序真正执行的时候将可能不创建这个对象，而改为直接创建它的若干个被这个方法使用到的成员变量来代替。拆散后的变量便可以被单独分析与优化，属性被扁平化后可以不用再通过引用指针来建立关系，可以连续紧凑的存储，对各种存储都更友好，执行期间能省去大量由于数据搬运造成性能损耗。同时还可以各自分别在栈帧或寄存器上分配空间，原本的对象就无需整体分配空间了。
>
> [深入理解JVM逃逸分析_wh柒八九的博客-CSDN博客](https://blog.csdn.net/qq_31960623/article/details/120178489) <= 推荐阅读，带有代码示例。
>
> ---
>
> go语言的channel：
>
> [Go 语言中的带有缓冲 Channel（Let‘s Go 三十一）_带缓冲的channel_甄齐才的博客-CSDN博客](https://blog.csdn.net/coco2d_x2014/article/details/127621765)
>
> 无缓冲信道 Channel 是无法保存任何值的，该类型信道要求 发送 goroutine 和 接受 goroutine 两者同时准备好，这样才能完成发送与接受的操作。假使 两者 goroutine 未能同时准备好，信道便会先执行 发送 和 接受 的操作， goroutine 会阻塞等待。这种对信道进行发送和接收的交互行为本身就是同步的。其中任意一个操作都无法离开另一个操作单独存在。
>
> 带缓冲信道在很多特性上和无缓冲信道是类似的。无缓冲信道可以看作是长度永远为 0 的带缓冲信道。因此根据这个特性，带缓冲信道在下面列举的情况下依然会发生阻塞：
>
> - 带缓冲信道被填满时，尝试再次发送数据时发生阻塞。
> - 带缓冲信道为空时，尝试接收数据时发生阻塞。

​	首先是一个模拟投票选举的go程序。

```go
//  vote-count-1.go
package main

import "time"
import "math/rand"

func main() {
  rand.Seed(time.Now().UnixNano())

  count := 0
  finished := 0

  for i := 0; i < 10; i ++ {
    // 匿名函数，创建共 10 个线程
    go func() {
      vote := requestVote() // 一个内部sleep随机时间，最后返回true的函数，模拟投票
      if vote {
        count++
      }
      finished++
    } () 
  }
  for count < 5 && finished != 10 {
    // wait
  }
  if count >= 5 {
    println("received 5+ votes!")
  } else {
    println("lost")
  }
}

// ....
```

​	通过执行`go run -race vote-count-1.go`用go自带的工具检测出竞态条件场景。这里很明显新建的多个goroutine共享了变量count、finished，构成竞态条件场景。

​	一种解决方法，**仅使用锁**。涉及修改的代码片段如下：

```go
//  vote-count-2.go
package main

import "sync"
import "time"
import "math/rand"

func main() {
  rand.Seed(time.Now().UnixNano())

  count := 0
  finished := 0
  var mu sync.Mutex

  for i := 0; i < 10; i ++ {
    // 匿名函数，创建共 10 个线程
    go func() {
      vote := requestVote() // 一个内部sleep随机时间，最后返回true的函数，模拟投票
      // 临界区(critical section)加锁
      mu.Lock()
      // 推迟到基本block结束后执行，这里即函数执行结束后 自动执行解锁操作。利用defer语言，一般在声明加锁后，立即defer声明推迟解锁 
      defer mu.Unlock()
      if vote {
        count++
      }
      finished++
    } () 
  }
  // 这里的循环仅仅是为了等待其他线程的结果，CPU空转，用条件变量实现则可以更好地提前放弃占用CPU
  for {
    mu.Lock()
    if count >= 5 || finished == 10 {
     	break 
    }
    mu.Unlock()
  }
  if count >= 5 {
    println("received 5+ votes!")
  } else {
    println("lost")
  }
  mu.Unlock()
}

// ....
```

​	对于下面主线程CPU空转等待其他线程结果的逻辑，可以增加sleep等待，但是坏处是这里该休眠多久并不好把控。

```go

  // 这里的循环仅仅是为了等待其他线程的结果，CPU空转，用条件变量实现则可以更好地提前放弃占用CPU
  for {
    mu.Lock()
    if count >= 5 || finished == 10 {
     	break 
    }
    mu.Unlock()
    time.Sleep(50 * time.Millsecond)

// ....
```

​	另一种解决竞态条件的方法，**使用锁和条件变量**，代码修改如下：

```go
//  vote-count-3.go
package main

import "sync"
import "time"
import "math/rand"

func main() {
  rand.Seed(time.Now().UnixNano())

  count := 0
  finished := 0
  var mu sync.Mutex
  // 条件变量 和 指定的 lock 绑定
  cond := sync.NewCond(&mu)

  for i := 0; i < 10; i ++ {
    // 匿名函数，创建共 10 个线程
    go func() {
      vote := requestVote() // 一个内部sleep随机时间，最后返回true的函数，模拟投票
      // 临界区(critical section)加锁
      mu.Lock()
      // 推迟到基本block结束后执行，这里即函数执行结束后 自动执行解锁操作。利用defer语言，一般在声明加锁后，立即defer声明推迟解锁 
      defer mu.Unlock()
      if vote {
        count++
      }
      finished++
      // Broadcast 唤醒所有等待条件变量 c 的 goroutine，无需锁保护；Signal 只唤醒任意 1 个等待条件变量 c 的 goroutine，无需锁保护。
      // 这里只有一个waiter，所以用Signal或者Broadcast都可以
      cond.Broadcast()
    } () 
  }

  mu.Lock()
  for count < 5 && finished != 10 {
    // 如果条件不满足，则在制定的条件变量上wait。内部原子地进入sleep状态，并释放与条件变量关联的锁。当条件变量得到满足时，这里内部重新获取到条件变量关联的锁，函数返回。
    cond.Wait() 
  }
  if count >= 5 {
    println("received 5+ votes!")
  } else {
    println("lost")
  }
  mu.Unlock()
}

// ....
```

​	当然，还可以用通道channel实现：

```go
//  vote-count-4.go
package main

import "time"
import "math/rand"

func main() {
  rand.Seed(time.Now().UnixNano())
	// 下面统一只有主线程更新这两个变量了
  count := 0
  finished := 0
  ch := make(chan bool)
  for i := 0; i < 10; i ++ {
    // 匿名函数，创建共 10 个线程
    go func() {
      // 将 requestVote 写入 通道channel
      ch <- requestVote() // 一个内部sleep随机时间，最后返回true的函数，模拟投票
    } () 
  }
  // 这里实现并不完美，如果count >= 5了，主线程不会再监听channel，导致其他还在运行的子线程会阻塞在往channel写数据的步骤。但是这里主线程退出后子线程们也会被销毁，影响不大。但如果是在一个长期运行的大型工程中，这里就存在泄露线程leaking threads
  for count < 5 && finished < 10 {
    // 主线程在这里阻塞，直到从channel中读取到数据
    v := <- ch
    if v {
      count += 1
    }
    finished += 1
  }
  if count >= 5 {
    println("received 5+ votes!")
  } else {
    println("lost")
  }
}

// ....
```



---

问题：看起来上面匿名函数可以访问函数外定义的变量，那么作用域是如何工作的？

回答：如果匿名函数使用的不是函数内声明的变量，那么会指向外部作用于的变量，这里即静态作用域。

追问：上面的go执行的匿名函数启动的线程中，如果把外层for循环的i传递进匿名函数中使用，会发生什么？

回答：如果直接在匿名函数中使用变量i，会在执行到匿名函数中读取i的那一行代码时，读取i的当时值，这很可能不是我们想要的。一般我们希望的是获取创建线程时，for循环中i当时对应的值。所以需要将i作为参数传入匿名函数，如下修改代码：

```go
for i := 0; i < 10; i++ {
  go func(i int) {
    vote := requestVote()
    mu.Lock()
    defer mu.Unlock()
    if vote {
      count++
    }
    finished++
    // 如果i没有作为参数传递进来，那么读取到的是执行这一行代码时i的值（执行新线程代码时，外层for中的i的值同时也在时刻改变中
    // 这里i作为匿名函数的参数传递进来，所以会是符合预期的0～9的值
    count = i
  } (i)
}
```

**问题：看上去count和finished是主线程的本地变量，它们在主函数退出后应该会被销毁，那此时如果主线程推出后，新建的go线程还在使用这些变量，会怎么样？**

**回答：这里根据go语言的逃逸分析可知，被共享的变量实际上go会在堆中进行分配，而不是在主线程的栈中，所以可以安全的在多个线程之间共享。（原本讲师回答其实不对，这里修正了下）**

问题：channel代码示例中，channel是自带了buffer吗？

回答：是的，通常你往channel写入数据时，<u>如果没有人从channel读取数据，那么channel写者会阻塞。这里用的不是缓存channel</u>。<u>无缓存/缓冲区的channel，需要写者和读者都同时就绪才能通信</u>。讲师表示在实验中用过带缓存/缓冲区的channel，表示不好用，用了很后悔，一般尽量不用，哈哈。

问题：有无办法在不退出主线程的情况下kill其他子线程？

回答：你可以往channel发送一个变量，示意子线程exit。但你必须自己去实现这个流程。

## 2.7 go爬虫(Crawler)

实现目标：

+ I/O concurrency：IO并发性
+ fetch once：保证每个url只爬一次
+ exploit parallelism：利用并发行特性

这里代码不是重点，可以自己找视频或代码，这里不贴了。

## 2.8 远程过程调用(RPC)

> [Go RPC ](https://www.jianshu.com/p/5ade587dbc58)<= 该小节简单讲解了RPC，这里推荐这篇文章。或者可以去了解下gRPC。

RPC(Remote Produce Call)远程过程调用。

RPC系统的目标：

+ **使得RPC在编程使用方面和栈上本地过程调用无差异**。比如gRPC等实现，看上去Client和Server都像是在调用本地方法，对开发者隐藏了底层的网络通信环节流程

---

RPC semantics under failures (RPC失败时的语义) ：

+ at-least-once：至少执行一次
+ at-most-once：至多执行一次，即重复请求不再处理
+ Exactly-once：正好执行一次，很难做到

# Lecture3 谷歌文件系统GFS

> [GFS(Google文件系统) - 百度百科](https://baike.baidu.com/item/GFS/1813072?fr=aladdin)
>
> GFS是一个可扩展的[分布式文件系统](https://baike.baidu.com/item/分布式文件系统/1250388?fromModule=lemma_inlink)，用于大型的、分布式的、对大量数据进行访问的应用。它运行于廉价的普通硬件上，并提供容错功能。它可以给大量的用户提供总体性能较高的服务。

本章节主要讨论点：

+ storage：存储

+ GFS：GFS的设计

+ consistency：一致性

## 3.1 存储Storage设计难点

构建(fault tolerance)可容错的系统，需要可靠的storage存储实现。

通常应用开发保持无状态stateless，而底层存储负责永久保存数据。

+ 高性能(high perference)
  + shard：需要分片（并发读取）
  + data across server：跨多个服务机器存储数据（单机的网卡、CPU等硬件吞吐量限制）
+ 多实例/多机器(many servers)
  + constant faults：单机故障率低，但是足够多的机器则出现部分故障的概率高，在GFS论文上千台机器规模下，每天总会有3个机器出现失败/故障。因此需要有容错设计
+ 容错(fault tolerance)
  + replication：通常通过复制数据保证容错性，即当前磁盘数据异常/缺失等情况，尝试从另一个磁盘获取数据
+ 复制(replication)
  + potential inconsistencies：潜在的数据不一致问题需要考虑

+ 强一致性(strong consistency)：
  + lower performance：一般需要通过一些消息机制保证一致性，这会稍微带来一些性能影响，但一般底层为了保证数据一致性而额外进行的网络通信等操作在整体性能的开销中占比并不会很高。其中可能涉及需要将通信的一些结果写入存储中，这是相对昂贵的操作。

## 3.2 一致性(consistency)

​	理想的一致性，即整个分布式系统对外表现时像单个机器一样运作。

​	**并发性(concurrency)**和**故障/失败(failures)**是两个实现一致性时需要考虑的难点。

---

+ 并发性问题举例：
  W1写1，W2写2；R1和R2准备读取数据。W1和W2并发写，在不关心谁先谁后的情况下，考虑一致性，则我们希望R1和R2都读取到1或者都读取到2，R1和R2读取的值应该一致。（可通过分布式锁等机制解决）

+ 故障/失败问题举例：

  一般为了容错性，会通过复制的方式解决。而不成熟的复制操作，会导致读者在不做修改的情况下读取到两次不同的数据。比如，我们要求所有写者写数据时，需要往S1和S2都写一份。此时W1和W2并发地分别写1和2到S1、S2，而R1和R2即使在W1和W2都完成写数操作后，再从S1或S2读数时结果可能是1也可能是2（因为没有明确的协议指出这里W1和W2的数据在S1、S2上以什么方式存储，可能1被2覆盖，反之亦然）。

## 3.3 GFS-简述

> [谷歌Colossus文件系统的设计经验 - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1971701) <= 可供拓展阅读

​	GFS旨在保持高性能，且有复制、容错机制，但很难保持一致性。google确实曾使用GFS，虽然后继被新的文件系统Colossus取代。

​	在论文中可以看到mapper从GFS系统(上千个磁盘)能够以超过10000MB/s的速度读取数据。论文发表时，当时单个磁盘的读取速度大概是30MB/s，一般在几十MB/s左右。

​	GFS的几个主要特征：

+ Big：large data set，巨大的数据集
+ Fast：automatic sharding，自动分片到多个磁盘
+ Gloal：all apps see same files，所有应用程序从GFS读取数据时看到相同的文件（一致性）

+ Fault tolerance：automic，尽可能自动地采取一些容错恢复操作

## 3.4 GFS-数据读取流程简述

> [GFS文件系统剖析（中）一致性模型及读写流程介绍-社区博客-网易数帆 (163.com)](https://sq.sf.163.com/blog/article/172834757163061248)

![img](https://nos.netease.com/cloud-website-bucket/201807041332473b21bf58-526d-4648-a79e-d4ee2d5aaf98.png)

​	GFS通过Master管理文件系统的元数据等信息，其他Client只能往GFS写入或读取数据。当应用通过GFS Client读取数据时，大致流程如下：

1. Client向Master发起读数据请求
2. Master查询需要读取的数据对应的目录等信息，汇总文件块访问句柄、这些文件块所在的服务器节点信息给Client（大文件通常被拆分成多个块Chunk存放到不同服务器上，单个Chunk很大， 这里是64MB）
3. Client得知需要读取的Chunk的信息后，直接和拥有这些Chunk的服务器网络通信传输Chunks

## 3.5 GFS-Master简述

这里大致介绍Master负责的工作：

+ 维护文件名到块句柄数组的映射(file name => chunk handles)

  这些信息大多数存放在内存中，所以Master可以快速响应客户端Client

+ 维护每个块句柄(chunk handle)的版本(version)

+ 维护块存储服务器列表(list of chunk servers)

  + 主服务器(primary)
    + Master还需维护每一个主服务器(primary)的租赁时间(lease time)
  + 次要服务器(secondaries)

  典型配置即将chunk存储到3台服务器上

+ log+check point：通过日志和检查点机制维护文件系统。所有变更操作会先在log中记录，后续才响应Client。这样即使Master崩溃/故障，重启时也能通过log恢复状态。master会定期创建自己状态的检查点，落到持久性存储上，重启/恢复状态时只需重放log中最后一个check point检查点之后的所有操作，所以恢复也很快。

​	这里需要思考的是，哪些数据需要放到稳定的存储中(比如磁盘)？

+ 比如file name => chunk hanles的映射，平时已经在内存中存储了，还有必要存在稳定的存储中吗？

  需要，否则崩溃后恢复时，内存数据丢失，master无法索引某个具体的文件，相当于丢失了文件。

+ chunk handle 到 存放chunk的服务器列表，这一层映射关系，master需要稳定存储吗？

  不需要，master重启时会要求其他存储chunk数据的服务器说明自己维护的chunk handles数据。这里master只需要内存中维护即可。同样的，主服务器(primary)、次要服务器(secondaries)、主服务器(primary)的租赁时间(lease time)也都只需要在内存中即可。

+ chunk handle的version版本号信息呢，master需要稳定存储吗？

  需要。否则master崩溃重启时，master无法区分哪些chunk server存储的chunk最新的。比如可能有服务器存储的chunk version是14，由于网络问题，该服务器还没有拿到最新version 15的数据，master必须能够区分哪些server有最新version的chunk。

---

问题：Master崩溃重启后，会连接所有的chunk server，找到最大的version？

回答：Master会尝试和所有chunk server通信，尽量获取最新version。当然有可能拥有最新version的chunk server由于网络等原因正好联系不上，此时能联系上的存活最久的chunk server的version会比master存储的version小。

## 3.6 GFS-文件读取

1. Client向Master发请求，要求读取X文件的Y偏移量的数据
2. Master回复Client，X文件Y偏移量相关的块句柄、块服务器列表、版本号(chunk handle, list of chunk servers, version)

3. Client 缓存cache块服务器列表(list of chunk servers)
4. Client从最近的服务器请求chunk数据(reads from closest servers)
5. 被Client访问的chunk server检查version，version正确则返回数据

---

+ **为什么这里Client要缓存list of chunk server信息呢？**

  **因为在这里的设计中，Master只有一台服务器，我们希望尽量减少Client和Server之间的通信次数，客户端缓存可以大大减少Master机器的负载**。

+ **为什么Client尽量访问最近的服务器来获取数据(reads from closest servers)？**

  **因为这样在宛如拓扑结构的网络中可以最大限度地减少网络流量(mininize network traffic)，提高整体系统的吞吐量。**

+ **为什么在Client访问chunk server时，chunk server需要检查verison？**

  为了尽量避免客户端读到过时数据的情况。

## 3.7 GFS-文件写入

> [谷歌技术"三宝"之GFS](http://t.zoukankan.com/yorkyang-p-9889525.html)

​	这里主要关注文件写入中的**append操作**，因为把记录追加到文件中这个在他们的业务中很常见。**在mapreduce中，reducer将处理后的记录数据(计算结果)很快地追加(append)到file中**。

![img](https://img2018.cnblogs.com/blog/1038618/201811/1038618-20181101152404626-1502756600.png)

1. Client向Master发出请求，查询应该往哪里写入filename对应的文件。

2. Master查询filename到chunk handle映射关系的表，找到需要修改的chunk handle后，再查询chunk handle到chunk server数组映射关系的表，以list of chunk servers(primary、secondaries、version信息)作为Client请求的响应结果

   接下去有两种情况，已有primary和没有primary(假设这是系统刚启动后不久，还没有primary)

   + 有primary

     继续后续流程

   + 无primary

     + **master在chunk servers中选出一个作为primary，其余的chunk server作为secondaries**。(暂时不考虑选出的细节和步骤)
       + master会增加version（每次有新的primary时，都需要考虑时进入了一个new epoch，所以需要维护新的version），然后向primary和secondaries发送新的version，并且会发给primary有效期限的租约lease。这里primary和secondaries需要将version存储到磁盘，否则重启后内存数据丢失，无法让master信服自己拥有最新version的数据(同理Master也是将version存储在磁盘中)。

3. Client发送数据到想写入的chunk servers(primary和secondaries)，有趣的是，**这里Client只需访问最近的secondary，而这个被访问的secondary会将数据也转发到列表中的下一个chunk server**，<u>**此时数据还不会真正被chunk severs存储**</u>。（即上图中间黑色粗箭头，secondary收到数据后，马上将数据推送到其他<u>本次需要写</u>的chunk server）

   **这么做提高了Client的吞吐量，避免Client本身需要消耗大量网络接口资源往primary和多个secondaries都发送数据**。

4. 数据传递完毕后，Client向primary发送一个message，表明本次为append操作。

   primary此时需要做几件事：

   1. primary此时会检查version，如果version不匹配，那么Client的操作会被拒绝
   2. **primary检查lease是否还有效，如果自己的lease无效了，则不再接受任何mutation operations（因为租约无效时，外部可能已经存在一个新的primary了）**
   3. 如果version、lease都有效，那么primary会选择一个offset用于写入
   4. primary将前面接收到的数据写入稳定存储中

5. primary发送消息到secondaries，表示需要将之前接收的数据写入指定的offset

6. secondaries写入数据到primary指定的offset中，并回应primary已完成数据写入

7. primary回应Client，你想append追加的数据已完成写入

   <u>当然，存在一些情况导致数据append失败，此时primary本身写入成功，但是后续存在某些/某个secondaries写入失败，此时会向Client返回错误error</u>。**Client遇到这种错误后，通常会retry整个流程直到数据成功append，这也就是所谓的最少一次语义(do at-least-once)**

---

+ **需要注意的是，假设append失败，Client再次重试，此时流程中primary指定写入的offset和上一次会是一样的吗？**

  不，primary会指定一个新的offset。假设primary+2台secondaries，可能上一次p和s1都写成功，仅s2失败。此时retry需要用新的offset，或许p、s1、s2就都写入成功了。这里可以看出来**副本记录是可以重复的(replicates records can be duplicated)**，这和我们常见的操作系统中标准的文件系统不一样。

  好在应用程序不需要直接和这种特殊的文件系统交互，而是通过库操作，库的内部实现隐藏了这些细节，用户不会看到曾经失败的副本记录数据。如果你append数据，库会给数据绑定一个id，如果库读取到相同id的数据，会跳过前面的一个。同时库内通过checksums检测数据的变化，同时保证数据不会被篡改。

---

问题：比起重写每一个副本(replica)，直接记住某个副本(replica)失败了不是更好吗？

回答：是的，有很多不同的实现方式。上面流程中假设遇到短暂的网络问题等，也不一定会直接判定会写入失败，可能过一会chunk server会继续写入，并且最后判定为写入成功。

问题：所有的这些服务器都是可信的吗？（换言之，流程中好像没说明一些权限问题）

回答：是的，这很重要，这里不像Linux文件系统，有权限和访问控制写入(permissions and access control writes)一类的东西，服务器完全可信（因为这完全是一个内部系统）。

**问题：写入流程中提到需要校验version，是否存在场景就是Client要读取旧version的数据？比如Client自己存储的version就是旧版本，所以读取数据时，拥有旧version的chunk server反而能够响应数据。**

**回答：假设这里有P(primary)，S1(secondary 1)，S2(secondary 2)和C(Client)，S2由于网络等问题和P、S1断开联系，此时重新选出Primary(假设还是原本的P)，version变为11，而S2断联所以还是version10的数据。一个本地cache了version10和list of chunk sevrers的Client正好直接从S2请求读取数据（因为S2最近），而S2的version和Client一致，所以直接读取到S2的数据。**

问题：上面这个举例中，为什么version11这个信息不会返回到Client呢？

回答：原因可能是Client设置的cache时间比较长，并且协议中没有要求更新version。

**问题：上面举例中，version更新后，不会推送到S2吗？即通知S2应该记录最新的version为11**

**回答：version版本递增是在master中维护的，version只会在选择新的primary的时候更新。**论文中还提到serial number序列号的概念，不过这个和version无关，这个用于表示写入顺序。

问题：primary完成写入前，怎么知道要检查哪一些secondaries（是否完成了写入）？

回答：master告知的。master告诉primary需要更新secondaries，这形成了new replica group for that chunk。

## 3.8 GFS-一致性(Consistency)

> [spanner(谷歌公司研发的数据库) - 百度百科](https://baike.baidu.com/item/spanner/6763355?fr=aladdin)

​	这里一致性问题，可以简单地归结于，当你完成一个append操作后，进行read操作会读取到什么数据？

​	这里假设一个场景，我们讨论一下可能产生的一致性问题，这里有一个M（maseter），P（primary），S（Secondary）：

+ **某时刻起，M得不到和P之间的ping-pong通信的响应，什么时候Master会指向一个新的P？**

  1. M比如等待P的lease到期，否则会出现两个P
  2. 此时可能有些Client还在和这个P交互

  **假设我们此时选出新P，有两个P同时存在，这种场景被称为脑裂(split-brain)，此时会出现很多意料外的情况，比如数据写入顺序混乱等问题，严重的脑裂问题可能会导致系统最后出现两个Master**。

  <u>这里M知道P的lease期限，在P的lease期限结束之前，不会选出新P，即使M无法和P通信，但其他Client可能还能正常和P通信</u>。

----

这里扩展一下问题，**如何达到更强的一致性(How to get stronger consistency)？**

比如这里GFS有时候写入会失败，导致一部分S有数据，一部分S没数据（虽然对外不可见这种失败的数据）。我们或许可以做到要么所有S都成功写入，要么所有S都不写入。

事实上Google还有很多其他的文件系统，有些具有更强的一致性，比如Spanner有更强的一致性，且支持事务，其应用场景和这里不同。我们知道这里GFS是为了运行mapreduce而设计的。

# Lecture4 主/备复制(Primary/Backup Replication)

> [（VM-FT论文解读）容错虚拟机分布式系统的设计](https://blog.csdn.net/weixin_40910614/article/details/117995014) <= 推荐阅读

+ failures：复制失败场景和对策
+ challenge：实现难点
+ 2 appliction：2个典型应用场景的探讨
  + 状态转移复制(state transfer replication)
  + 复制状态机(replicated state machines)

+ case study
  + VM FT：VMware fault tolerance

## 4.1 失败场景(Failures)

​	常见的复制失败场景分类：

+ fail-stop failure：基础设备或计算机组件问题，导致系统暂停工作(stop compute)，一般指计算机<u>原本工作正常</u>，由于一些突发的因素暂时工作失败了，比如网线被切断等。
+ logic bugs, configuration errors：本身复制的逻辑有问题，或者复制相关的主从配置有异常，这类failure导致的失败，不能靠系统自身自动修复。
+ malicious errors：这里我们的设计假设的内部系统中每一部分都是可信的，所以我们无法处理试图伪造协议的恶意攻击者。
+ handling stop failure：比如主集群突发地震无法正常服务，我们希望系统能自动切换到使用backup备份集群提供服务。当然如果主从集群都在一个数据中心，那么一旦出现机房被毁问题，大概率整个系统就无法提供服务了。

<u>在后续课程介绍中，我们的讲解/设计，都基于一个前提假设：系统中软件工作正常、没有逻辑/配置错误、没有恶意攻击者</u>。

我们只关注**处理停止失败(handling stop failures)**。

## 4.2 实现难点(Challenge)

> [故障转移 - 百度百科](https://baike.baidu.com/item/%E6%95%85%E9%9A%9C%E8%BD%AC%E7%A7%BB/14768924?fr=aladdin)
>
> 在计算机术语中，**故障转移**（英语：failover），即当活动的服务或应用意外终止时，快速启用冗余或备用的服务器、系统、硬件或者网络接替它们工作。 故障转移 (failover)与交换转移操作基本相同，只是故障转移通常是自动完成的，没有警告提醒手动完成，而交换转移需要手动进行。

+ 如何判断primary失败？(has primary failed?)

  + **你无法直接区分发生了网络分区问题(network partition)还是实质的机器故障问题(machine failed)**

    或许只是部分机器访问不到primary，但是客户端还是正常和primary交互中。你必须有一些机制保证不会让系统同时出现两个primary(假设机制中只允许正常情况下有且只有一个primary工作)。

  + **(Split-brain system)脑裂场景**

    假设机制不完善，可能导致两个网络分区下各自有一个primary，客户端们和不同的primary交互，最后导致整个系统内部状态产生严重的分歧（比如存储的数据、数据的版本等差异巨大）。此时如果重启整个系统，我们就不得不手动处理这些复杂的分歧状态问题（就好似手动处理git merge冲突似的）。

+ **如何让主备保持同步(how to keep the primary/backup in sync)**

  **我们的目标是primary失败时，backup能接手primary的工作，并且从primary停止的地方继续工作。这要求backup总是能拿到primary最新写入的数据，保持最新版本**。我们不希望直接向客户端返回错误或者无法响应请求，因为从客户端角度来看，primary和backup无区别，backup就是为容错而生，理应也能正常为自己提供服务。

  + 需要**保证应用中的所有变更，按照正确顺序被处理(apply changes in order)**
  + 必须**避免/解决非决定论(avoid non-determinism)**。<u>即相同的变更在primary和backup上应该有一致的表现</u>。

+ **故障转移(failover)**

  primary出现问题时，我们希望切换到backup提供服务。但是切换之前，我们需要保证primary已经完成了所有正在执行的工作。即我们不希望在primary仍然在响应client时突然切换backup（如果遇到网络分区等问题，会使得故障转移难上加难）。

----

问题：什么时候需要进行故障转移(failover)？

回答：当primary失败时，我们希望切换到backup提供服务

## 4.3 主备复制

1. **状态转移(state transfer)**：primary正常和client交互，每次响应client之前，先生成记录checkpoint检查点，将checkpoint同步到备份backup，待backup都同步完状态后，primary再响应client。

2. **复制状态机(replicated state machine，RSM)**：与状态转移类似，只是这里primary和backup之间同步的不是状态，而是操作。即primary正常和client交互，每次响应client之前，先生成操作记录operations，将operations同步到备份backup，待backup都执行完相同的操作后，primary再响应client。

​	这两种方案都有被应用，共同的**要点在于，primary响应client之前，首先确保和backup同步到相同的状态，然后再响应client。这样当primary故障时，任意backup接管都能有和原primary对齐的状态**。

​	**状态转移(state transfer)的缺点在于一个操作可能产生很多状态state，此时同步状态就变得昂贵且复杂。所以市面上的应用一般更乐意采用第二种方案，即复制状态机(replicated state machine，RSM)。**

​	<u>前面介绍的GFS追加append流程("3.7 GFS-文件写入")，实际上也是primary将append操作通知到其他secondaries，即采用复制状态机(replicated state machine，RSM)</u>，而不是将append后的结果状态同步到其他secondaries。

----

问题：为什么client不需要发送数据到backup备机？

回答：因为这里client发送的请求是具有**确定性的操作**，只需向primary请求就够了。主备复制机制保证primary能够将具有确定性的操作正确同步到其他backup，即系统内部自动保证了primary和backup之间的一致性，不需要client额外干预。**接下来的问题即，怎么确定一个操作是否具有确定性？在复制状态机(replicated state machine，RSM)方案中，即要求所有的操作都是具有确定性的，不允许存在非确定性的操作**。

**问题：是不是存在着混合的机制，即混用状态转移(state transfer)和复制状态机(replicated state machine，RSM)？**

**回答：是的。比如有的混合机制在默认情况下以复制状态机(replicated state machine，RSM)方案工作，而当集群内primary或backup故障，为此创建一个新的replica时则采用状态转移(state transfer)转移/复制现有副本的状态**。

## 4.4 复制状态机RSM-复制什么级别的操作

​	使用复制状态机时，我们需要考虑什么级别的操作需要被复制。有以下几种可能性：

+ 应用程序级别的操作(application-level operations)

  比如GFS的文件append或write。如果你在应用程序级别的操作上使用复制状态机，那也意味着你的复制状态机实现内部需要密切关注应用程序的操作细节，比如GFS的append、write操作发生时，复制状态机应该如何处理这些操作。<u>一般而言你需要修改应用程序本身，以执行或作为复制状态机方法的一部分</u>。

+ 机器层面的操作(machine-level operaitons)，或者说processor level / coputer level

  这里对应的状态state是寄存器的状态、内存的状态，操作operation则是传统的计算机指令。**这种级别下，复制状态机无感知应用程序和操作系统，只感知最底层的机器指令**。

  有一种传统的进行机器级别复制的方式，比如你可以额外购买机器/处理器，这些硬件本身支持复制/备份，但是这么做很昂贵。

  这里讨论的论文(VM-FM论文)**通过虚拟机(virtual machine, VM)实现**。

## 4.5 通过虚拟化实现复制(VM-FT: exploit virtualization)

![3](https://img-blog.csdnimg.cn/20210618165533813.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDkxMDYxNA==,size_16,color_FFFFFF,t_70)

​	虚拟化复制对应用程序透明，且能够提供很强的一致性。早期的VMware就是以此实现的，尽管现在新版可能有所不同。缺陷就是这篇论文只支持单核，不支持多核(multi-core)。或许后来的FT支持了，但应该不是纯粹通过复制状态机实现的，或许是通过状体转移实现的，这些都只是猜测，毕竟VMware没有透露后续产品的技术细节。

​	我们先简单概览一下这个系统实现。

​	首先有一个虚拟机控制器(virtual machine monitor)，或者有时也被称为hypervisor，在这论文中，对应的hypervisor即VM-FT。

​	当发生中断（比如定时器中断）时，作为hypervisor的VM-FT会接收到中断信号，此时它会做两件事：

1. **通过日志通道将中断信号发送给备份计算机（sends it over a logging channel to a backup computer）**
2. 将中断信号传递到实际的虚拟机，比如运行在guest space的Linux。

​	同理，当client向primary发送网络数据包packet时，primary所在的硬件产生中断，而VM-FT将同样的中断通过logging channel发送给backup computer（也就是另一个VM-FT），然后将中断发送到当前VM-FT上的虚拟机(比如Linux)。另一台backup的VM-FT上运行着和priamry相同的Linux虚拟机，其也会同样收到来自backup的VM-FT的中断信号。primary虚拟机Linux之后往虚拟网卡写数据，产生中断，同样VM-FT也会将中断往backup的VM-FT发送一份。**最后就是primary上的Linux虚拟机和backup上的Linux虚拟机都往各自的虚拟网卡发送了数据，但是backup的VM-FT知道自己是backup备机，所以从虚拟网卡接收数据后什么也不会做，只有primary的VM-FT会真正往物理网卡写数据，响应client的请求**。

​	论文中提到，在primary和backup两个VM-FT以外，假设还通过网络和外部一个storage存储保持通讯。外部storage通过一个flag记录primary和backup状态，记录谁是primary等信息。

​	当primary和backup之间发生网络分区问题，而primary、backup仍可以与这个外部storage通信时，primary和backup互相会认为对方宕机了，都想把自己当作新的primary为外界的client提供服务。**此时，原primary和原backup都试图通过test-and-set原子操作在外部storage修改flag记录（比如由0改成1之类的），谁先完成修改修改，谁就被外部storage认定为新的primary；而后来者test-and-set操作会返回1(test-and-set会返回旧值，这里返回1而不是0，表示已经有人领先自己把0改成1了)，其得知自己是后来者，会主动放弃称为primary的机会，在论文中提到会选择终结自己(terminate itself)。**

​	看上去这个方案能够避免同时出现两个primary的场景，并且把容错部分转移到了存储服务器中，这些看似都很合理，但事实并非如此。

---

问题：上面primary和backup的VM-FT争抢地希望通过test-and-set将flag的值从0设置成1，但这个flag是什么时候被设置成0的？

回答： 首先启动一台primary时，我们希望有备份，所以启动了另一个backup。之后我们希望有一个repair计划，保证primary失败后，backup能够接替，避免之后backup也失败后就没有机器能够运行了。而在VMFT中，修复方案由人工执行。当有人接收到报警通知，就马上根据第一个VM image虚拟机镜像创建一个replica，确保它们同步。根据协议，当backup启动完成备份时，flag就被重置为0。（当1个VM-FT向VMotion请求被复制时，正在被复制的VM-FT仍可以正常地向Client提供服务，而新的replica被复制出来时，会将storage中的flag标志设置为0。）

问题：primary和backup之间的logging channel中断会发生什么，又或者client访问server的通道中断了，会发生什么？

回答：如果server被访问的通道都断开了，那么即使primary、backup本身工作正常，可server也不会再对client提供服务，需要等待网络恢复正常，这是一个灾难性的例子。

## 4.6 差异来源(Divergence sources)

​	如果前面primary执行的指令都是确定性的，那么primary和backup无疑可以保证拥有相同的状态。但是不可避免的是可能出现一些非确定性的事件，我们需要考虑如何处理。我们的**目标即，将每一条非确定性的指令(non-deterministic instruction)变成确定性指令(deterministic instruction)**。

+ 非确定性的指令(non-deterministic instruction)

  <u>比如获取时间的指令，我们不可能保证primary和backup都在同一时间执行，返回值一般来说会不同</u>。

+ 网络包接收/时钟中断(input packets / timer interrupters)

  比如网络包输入时导致中断，primary和backup在原本CPU执行流中插入中断处理的位置可能不同。比如primary在第1～2条指令执行的位置插入网络包的中断处理，而backup在第2～3条指令执行的位置插入中断处理，这也有可能导致后续primary和backup的状态不一致。所以我们希望这里数据包产生的中断（或者时钟中断），在primary和backup相同的CPU指令流位置插入执行中断处理，确保不会产生不一致的状态。

+ 多核(multi-core)

  并发也是一个导致差异（primary和backup不一致）的潜在因素。而这篇论文的解决方案即不允许多核。比如，在primary上多核的两个线程竞争lock，我们需要保证backup上最后获取到lock的线程和primary的保持一致，这需要很多额外的复杂机制去实现这一点，该论文显然不想处理这个问题。这里讲师也不明确VMware在后来的产品是怎么保证多核机器的复制的，所以后续不讨论multi-core的场景。

​	论文假设大多数操作指令都是确定性的，只有出现这里谈论的一些非确定性的操作发生时，才需要考虑如何处理primary和backup之间的同步问题。

## 4.7 VM-FT的中断处理

​	根据前面的讨论，可以知道中断是一个非确定性的差异来源，我们需要有机制保证primary和backup处理中断后仍保持状态一致。

​	这里VM-FT是这样处理的，**当接受到中断时，VM-FT能知道CPU已经执行了多少指令（比如执行了100条指令），并且计算一个位置（比如100），告知backup之后在指令执行到第100条的时候，执行中断处理程序。大多数处理器（比如x86）支持在执行到第X条指令后停止，然后将控制权返还给操作系统（这里即虚拟机监视器）**。

​	通过上面的流程，VM-FT能保证primary和backup按照相同的指令流顺序执行。当然，**这里backup会落后一条message（因为primary总是领先backup执行完需要在logging channel上传递的消息）**。

---

问题：确定性操作，就不需要通过日志通道(logging channel)同步吗？

回答：是的，不需要。因为它们都有一份所有指令的复制，只有非确定性的指令才需要痛殴logging channel同步。

## 4.8 VM-FT的非确定性指令(non-deterministic instruction)处理

​	对于非确定性的指令，VM-FT基本上是这样处理的：

​	**在启动Guest space中的Linux之前，先扫描Linux中所有的非确定性指令，确保把它们转为无效指令(invalid instruction)。当Guest space中的Linux执行这些非确定性的指令时，它将控制权通过trap交给hypervisor，此时hypervisor通过导致trap的原因能知道guest在执行非确定的指令，它会模拟这条指令的执行效果，然后记录指令模拟执行后的结果，比如记录到寄存器a0中，值为221。而backup备机上的Linux在后续某个时间点也会执行这条非确定性指令，然后通过trap进入backup的hypervisor，通常backup的hypervisor会等待，直到primary将这条指令模拟执行后的结果同步给自己(backup)，然后backup就能和primary在这条非确定性指令执行上拥有一致的结果**。

---

问题：扫描guest space的操作系统拥有的不确定性指令，发生在创建虚拟机时吗？

回答：是的，通常认为是在启动引导虚拟机时(when it boot to the VM)。

问题：如果这里有一条完全确定的指令，那么backup有可能执行比primary快吗？

回答：论文中有一个完整的结论，这里只需要primary和bakcup能有大致相同的执行速度（谁快一点谁慢一点影响不大），通常让primary和backup具有相同的硬件环境来实现这一点。

## 4.9 VM-FT失败场景处理

​	这里举例primary故障的场景。

​	比如primary本来维护一个计数器为10，client请求将其自增到11，但是primary内部自增了计数器到11，但是响应client前正好故障了。如果backup此时接手，其执行完logging channel里要求同步的指令，但是自增到11这个并没有反映到bakcup上。如果client再次请求自增计数器，其会获取到11而不是12。

​	为了避免出现上诉这种场景，VM-FT指定了一套**输出规则(Output rule)**。 实际上上诉场景并不会发生。**primary在响应client之前，会通过logging channel发送消息给backup，当backup确定接受到消息且之后能执行相同的操作后，会发送ack确认消息给primary。primary收到ack后，确认backup之后能和自己拥有相同的状态（就算backup有延迟慢了一点也没事，反正backup可以通过稳定存储等手段记录这条操作，之后确保执行后未来能和primary达到相同的状态），然后才响应client**。

​	在任何复制系统(replication system)，都能看到类似输出规则(output rule)的机制（比如在raft或者zookeeper论文中）。

## 4.10 VM-FT性能问题

​	因为VM-FT的操作都基于机器指令或中断的级别上，所以需要牺牲一定的性能。

​	论文中统计在primary/backup模式下运行时，性能和平时单机差异不会太大，保持在0.94~0.98的水平。而当网络输入输出流量很高时，性能下降很明显，下降将近30%。这里导致性能下降的原因可能是，primary处理大量数据包时，需要等待backup也处理完毕。

​	**正因为在指令层面(instruction level)使用复制状态机性能下降比较明显，所以现在人们更倾向于在应用层级(application level)使用复制状态机。**<u>但是在应用层级使用复制状态机，通常需要像GFS一样修改应用程序</u>。

----

问题：前面举例client发请求希望primary中维护的计数器自增的操作，看似确定性操作，为什么还需要通过logging channel通知bakcup？

回答：因为client请求是一个网络请求，所以需要通知backup。并且这里primary会确定backup接受到相同的指令操作回复自己ack后，才响应client。

问题：VM-FT系统的目标，是为了帮助提高服务器的性能吗？上诉这些机制看着并不简单且看着会有性能损耗。或者说它是为了帮助分发虚拟机本身？

回答：这一套方案，仅仅是为了让运行在服务器上的应用拥有更强的容错性。虽然在机器指令或中断级别上做复制比起应用级别上性能损耗高，但是这对于应用程序透明，不要求修改应用程序代码。

问题：关于输出规则(Output rule)，客户端是否可能看到相同的响应两次？

回答：可能，客户端完全有可能收到两次回复。但是论文中认为这种情况是可容忍的，因为网络本身就可能产生重复的消息，而底层TCP协议可以处理这些重复消息。

问题：如果primary宕机了几分钟，backup重新创建一个replica并通过test-and-set将storage的flag从0设置为1，自己成为新的primary。然后原primary又恢复了，这时候会怎么样？

回答：原primary会被clean，terminate自己。

问题：上面的storage除了flag以外，只需要存储其他东西吗？比如现在有多个backup的话。

回答：论文这里的方案只针对一个backup，如果你有多个backup，那么你还会遇到其他的问题，并且需要更复杂的协议和机制。

问题：处理大量网络数据包时，primary只会等待backup确认第一个数据包吗？

回答：不是，primary每处理一个数据包，都会通过logging channel同步到backup，并且等待backup返回ack。满足了输出规则(output rule)之后，primary才会发出响应。这里它们有一些方法让这个过程尽量快。

问题：关于logging channel，我看论文中提到用UDP。那如果出现故障，某个packet没有被确认，primary是不是直接认为backup失败，然后不会有任何重播？

回答：不是。因为有定时器中断，定时器中断大概每10ms左右触发一次。如果有包接受失败，primary会在心跳中断处理时尝试重发几次给backup，如果等待了可能几秒了还是有问题，那么可能直接stop停止工作。
