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

## 3.1 存储Storage

## 3.2

## 3.3

0:55



