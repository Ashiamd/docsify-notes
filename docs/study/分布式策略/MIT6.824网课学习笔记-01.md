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

# Lecture5 容错-Raft(Fault Tolerance-Raft) -1

> [Raft 协议原理详解，10 分钟带你掌握！ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/488916891)	<= 图片很多，推荐阅读
>
> [Raft 协议 - 简书 (jianshu.com)](https://www.jianshu.com/p/c9024d05887f) <= 有动图，还不错

​	这章节主要介绍Raft协议，它是分布式复制协议(distributed replication protocol)示例的核心组件之一。

## 5.1 单点故障(single point of failure)

​	前面介绍过的复制系统，都存在单点故障问题(single point of failure)。

+ mapreduce中的cordinator
+ GFS的master
+ VM-FT的test-and-set存储服务器storage

​	而上诉的方案中，采用单机管理而不是采用多实例/多机器的原因，是为了避免**脑裂(split-brain)**问题。

​	不过大多数情况下，单点故障是可以接受的，因为单机故障率显著比多机出现一台故障的概率低，并且重启单机以恢复工作的成本也相对较低，只需要容忍一小段时间的重启恢复工作。

## 5.2 脑裂问题(split-brain)

​	Raft是避免单点故障问题的一种方案，在介绍Raft之前，我们**先了解下为什么单机管理可以避免严重的脑裂问题，尽管单机管理会有单点故障问题**。

​	以VM-FT的test-and-set存储服务器storage举例。假设我们复制storage服务器，使其有两个实例机器S1、S2。（即打破单机管理的场景，看看简单的多机管理下有什么问题）

​	此时C1想要争取称为Primary，于是向S1和S2都发起test-and-set请求。假设因为某种原因，S2没有响应，S1响应，成功将0改成1，此时C1可能直接认为自己成为Primary。

​	这里S2没有响应可以先简单分析两种可能：

1. S2失败/宕机了，对所有请求方都无法提供服务

   如果是这种情况，那么C1成为Primary不会有任何问题，S2就如同从来不曾存在，任何其他C同样只能访问到S1，他们都会知道自己无法成为Primary，并且整个系统只会存在C1作为Primary。

2. S2和C1之间产生了网络分区(network partition)，仅C1无法访问到S2

   这时候如果存在C2也向S1和S2发出请求，此时S1虽然将0改成1会失败，但是S2会成功。如果我们的协议不够严谨，这里C2会认为自己成为了Primary，导致整个系统存在两个Primary。这也就是严重的脑裂问题。

​	**产生这种问题的原因在于，对于请求方，无法简单地判断上诉两种情况，因为对他们来说两种情况的表现都是一样的**。

​	**为避免出现脑裂，我们需要先解决网络分区问题**。	

## 5.3 大多数原则(majority rule)

​	<u>诸如Raft一类的协议用于解决**单点故障**问题，同时也用于解决**网络分区**</u>问题。这类解决方案的基本思想即：**大多数原则(majority rule)**，**简单理解就是少数服从多数**。

​	<u>这里的majority指的是整个系统中无论机器的状态如何，只要获取到大于一半的赞成，则认为获取到majority。比如系统中有5台机器，2台宕机了，5台的一半为2.5，但2.5不为有效值，即获取3台机器赞成，认为获取到majority；如果5台中宕机3台，那么没有人能获取到至少3台的赞成，即无法得到majority</u>。

​	同样以VM-FT的test-and-set存储服务器storage举例，这次storage不是复制出2个实例，而是总共复制出3个实例，S1、S2、S3。

​	此时C1同时向S1、S2、S3请求test-and-set，其中S1和S2成功将0改成1，S3因为其他问题没有响应，但是我们不关系为什么。这里按照majority rule，只要3个S中有2个给出成功响应，我们就认为C1能够成为Primary。此时就算同时有C2向S1、S2、S3发起请求，就算S3成功了，C2根据majorty rule，3台只成功1台，不能成为Primary。

​	<u>为什么根据majority rule，这里就不会出现脑裂，因为能确保多个C之间的请求有**重叠部分(overlap)**，这样根据majority rule，多个C最后也能达成一致，**因为只有1个C有可能成为多数的一方，其他C只能成为少数的一方**</u>。

​	在之后准备介绍的Raft，和这里描述的工作流程基本一致。

+ 在majority rule下，尽管发生网络分区，只会一个拥有多数的分区，不会有其他分区具有多数，只有拥有多数的分区能继续工作（比如这里3台被拆成1台、2台的分区，只有和后者成功通信的能继续工作）。
+ **而如果极端情况下所有分区都不占多数（ 比如这里3台被拆成1台、1台、1台的分区），那么整个系统都不能运行**。

​	上诉3台的场景，只能容忍1台宕机，如果宕机2台，那么任何人都无法达到majorty的情况。**这里通过2f+1拓展可容忍宕机的机器数，f表示可容忍宕机的机器数量**。**2f+1，即使宕机了f台，剩下的f+1>f，仍然可以组成majority完成服务**。

​	例如，当f=2时，表示系统内最多可容忍2台机器处于宕机状态，那么至少需要部署5台服务器（2x2+1=5）。

---

问题：这里大多数是否包括参与者服务器本身，假设是raft，服务器会考虑自身吗？

回答：是的，通常我们看到的raft，candidate会直接vote自己。而leader处理工作时，也会vote记录自己。

## 5.4 Raft历史发展

​	在1980s～1990s，基本不存在诸如majority的协议，所以一直存在单点故障的问题。

​	而在1990s出现了两种被广泛讨论协议，但是由于当时的应用没有自动化容错的需求，所以基本没有被应用。但近15年来(2000s~2020s)大量商用产品使用到这些协议：

+ Paxos
+ View-Stamped replication (也被称为VR)

​	我们将要讨论的是Raft，大概在2014左右有相关的论文，它应用广泛，你可以用它来实现一个完全复制状态机(complete replicated state machine)。

## 5.5 使用Raft构造复制状态机RSM(replicated state machine)

​	这里raft就像一个library应用包。假设我们通过raft协议构造了一个由3台机器组成的K/V存储系统。

系统正常工作时，大致流程如下：

+ Client向3台机器中作为leader的机器发查询请求
+ leader机器将接收到的请求记录到底层raft的顺序log中
+ 当前leader的raft将顺序log中尾部新增的log记录通过网络同步到其他2台机器
+ 其他两台K/V机器的raft成功追加log记录到自己的顺序log中后，回应leader一个ACK
+ leader的raft得知其他机器成功将log存储到各自的storage后，将log操作反映给自己的K/V应用
+ K/V应用实际进行K/V查询，并且将结果响应给Client

系统出现异常时，发生如下事件：

+ Client向leader请求
+ leader向其他2台机器同步log并且获得ACK
+ leader准备响应时突然宕机，无法响应Client
+ <u>其他2台机器重新选举出其中1台作为新的leader</u>
+ Client请求超时或失败，重新发起请求，**系统内部failover故障转移**，所以这次Client请求到的是新leader
+ 新leader同样记录log并且同步log到另一台机器获取到ACK
+ 新leader响应Client

​	<u>这里可以想到的是，剩下存活的两台机器的log中会有重复请求，而我们需要能够检测(detect)出这些重复请求</u>。

​	上面就是用Raft实现复制状态机的一种普遍风格/做法。

---

问题：访问leader的client数量通常是多少？

回答：我想你的疑问是系统只有1个leader的话，那能承受多少请求量。<u>实际上，具体的系统设计还会采用shard将数据分片到多个raft实例上，每个shard可以再有各自的leader，这样就可以平均请求的负载到其他机器上了</u>。

问题：旧leader宕机后，client怎么知道要和新leader通信？

回答：client中有系统中所有服务器的访问列表，这里举例中有3个服务器。当其中一台请求失败时，client会重新随机请求3台中的1台，直到请求成功。

## 5.6 Raft概述

​	这里我们重新概述一下Raft流程。首先这里我们用3台Raft机器，其中一台为leader，另外两台为follower。leader和follower都在storage维护着log，所有K/V的put/get操作都会被记录到log中。

+ Client请求leader，假设是get请求
+ leader将get操作记录到自己的log尾部
+ leader将新的log条目发送给其他2个follower
+ 此时follower1成功将log条目追加到自己本地的log中，回应leader一个ack
+ 此时leader和follower1共2台机器成功追加log，达到majority，于是leader可以进行commit，将操作移交给上层的kv服务。（这里即使宕机了一台，之后重新选举，包含最后操作的服务器将当选成为新的leader，比如原leader或follower1将当选，所以服务能继续正常提供）
+ follower2由于网络问题，此时才回应leader一个ack。从follower角度其并不知道leader已经commit了操作，它只知道leader向自己同步了log并且需要回应ack。
+ 由于leader进行了commit，leader的上层kv会响应Client对应get请求的查询结果。

​	如果上面进行的是一个append操作，那么leader的raft记录log到本地后，leader同步log到其他follower后，如果leader进行了commit，follower也需要将log记录反馈给各自上层的k/v服务，使得append这个写操作生效。

---

问题：如果log从leader同步到其他follower时，leader宕机了，会怎么样？

回答：会重新发生选举，而拥有最新操作log的机器成为新leader后会将追加的log条目传递给其他follower，这样就保证这些机器都拥有最新的log了。

## 5.7 Raft的log用途

​	log的原因/用途：

+ 重传(retranmission)：leader向follower同步消息时，消息可能传递失败，所以需要log记录，方便重传
+ **顺序(order)：主要原因，我们需要被同步的操作，以相同的顺序出现在所有的replica上**
+ 持久化(persistence)：持久化log数据，才能支持失败重传，重启后日志恢复等机制
+ **试探性操作(space tentative)**：比如follower接收到来自leader的log后，并不知道哪些操作被commit了，可能等待一会直到明确操作被commit了才进行后续操作。我们需要一些空间来做这些试探性操作(tentative operations)，而log很合适。

​	**尽管中间有些时间点，可能有些机器的log是落后的。但是当一段时间没有新log产生时，最终这些机器的log会同步到完全相同的状态(logs identical on all servers)**。并且因为这些log是有顺序的，意味着上层的kv服务最终也会达到相同的状态。

## 5.8 Raft的log格式

​	在log中有很多个log entry。每个log entry被log index、leader term唯一标识。

​	每个log entry含有以下信息：

+ command（接受到的指令、操作）
+ leader term（当前系统中leader的任期期限）

​	从上面log entry的格式，可以引出两点信息：

+ elect leader：我们需要选举出特定任期的leader
+ ensue logs become indentical：确保最后所有机器的log能达到一致的状态

----

问题：没有提交(uncommit)的日志条目(log entries)会被覆盖吗？

回答：是的，可能被覆盖，详情后续会谈。

## 5.9 Raft选举(election)

​	假设这里有3台机器，1台leader，2台follower，它们都处于term10。

+ 某一时刻，leader和其他2follower产生了网络分区

+ 2个follower重新进行选举，原因是它们错过了leader的**心跳通信(heartbeats)**，而follower维护的election timer超时了，表明需要重新选举。

  当leader没有收到来自client的新消息时，leader也仍会定期发送heartbeat到其他所有followers，告知自己仍是leader。这也算一种append log，但是不会被实际记录到log中。在heartbeat中，leader会携带一些自己log状态的信息，比如当前log应该多长，最后一条log entry应该是什么样的，帮助其他followers同步到最新状态。

+ 假设follower1的election timer超时更早，其将term自增，从term10变成term11，并且发出选举，先自己投自己一票，然后向原leader和follower2请求选票

+ follower2响应follwer1，投出赞成票，原leader由于网络分区问题没有响应

+ follower1成为新的leader

+ client将请求failover故障转移到新的leader上，即follower1，后续请求发送至follower1

​	**也许有人会问，如果leader在follower1成为新leader时，仍然有client请求到leader上，并且leader又重新恢复了和原follower1、原follower2的网络连接了，会不会导致系统出现两个leader，即脑裂问题？**

​	**实际上不会出现脑裂(split-brain)问题**。如下分析：

+ 原leader接收到client的请求，尝试发log给原follwer1和原follower2
+ 原follower1（新leader）收到log后，会拒绝追加log到本地log中，并回复原leader新的term11
+ 原leader接收到term11后，对比自己的term10会意识到自己不再是leader，接下去要么直接成为follower，要么重新发起选举，但不会继续作为leader提供服务，不导致脑裂

+ client此时可能得到失败或拒绝服务等响应，然后重新向新leader(原follower1)发起请求

## 5.10 Raft-分裂选举问题(split vote)

​	在Raft中规定每个term只能投票一次，而前面的原leader网络分区后，follower1和follower2可能产生分裂选举(split vote)的问题：

+ 由于巧合，follower1和follower2的election timer几乎同一时刻到期，它们都将term由10改成11
+ follower1和follower2在term11都vote自己，然后向对方发起获取投票的请求
+ follower1和follower2因为都在term11投票过自己了，所以都会拒绝对方的拉票请求，这导致双方都成为不了majority，没有人成为新leader
+ 也许有个选举超时时间节点，follower1和follower又继续将term11增加到12，然后重复上诉流程。**如果此时没有人为介入操作，上面重复的term增加、vote自己、没人成为leader，超时后又进行新一轮election的过程可能一直重复下去**

​	**<u>为了避免陷入上面的选举死循环，通常election超时时间是随机的(election timeout is randomized)</u>**。

​	论文中提到，它们设置election timer时，选择150ms到300ms之间的随机值。每次这些follower重制自身的election timeout时，会选择150ms～300ms之间的一个随机数，只有当计时器到点时，才会发起选举eleciton。<u>这样上诉流程中总有一刻follower1或者follower2因为timer领先到点，最终成功vote自己且拉票让对方vote自己，达到majority条件，优胜成为leader</u>。

## 5.11 Raft-选举超时(election timeout)

​	选举超时的时间，应该设置成大概多少才合适？

+ **略大于心跳时间(>= few heartbeats)**

  如果选举超时比心跳还短，那么系统将频繁发起选举，而选举期间系统对外呈现的是阻塞请求，不能正常响应client。因为election时很可能丢失同步的log，一直频繁地更新term，不接受旧leader的log（旧leader的term低于新term，同步log消息会被拒绝）

+ **加入一些随机数(random value)**

  加入适当范围的随机数，能够避免无限循环下去的分裂选举(split vote)问题。random value越大，越能够减少进行split vote的次数，但random value越大，也意味着对于client来说，整个系统停止提供对外服务的时间越长（对外表现和宕机差不多，反正选举期间无法正常响应client的请求）

+ **尽量短(short enough that down time is short)**

  因为选举期间，对外client表现上如同宕机一般，无法正常响应请求，所以我们希望eleciton timeout能够尽量短

​	Raft论文进行了大量实验，以得到250ms～300ms这个在它们系统中的合理值作为eleciton timeout。

## 5.12 Raft-vote需记录到稳定storage

​	这里提一个选举中的细节问题。假设还是leader宕机，follower1和follower2中的follower1发起选举。follower1会先vote自己，然后发起拉票请求希望follower2投票自己。

​	**这里follower1应该用一个稳定的storage记录自己的vote历史记录，有人知道为什么吗？原因是避免重复vote**。假设follower1在vote自己后宕机一小段时间后恢复，我们需要避免follower1又vote自己一次，不然follower1由于vote过自己两次，直接就可以无视其他follower的投票认为自己成为了leader。

​	**所以，为了保证每个term，每个机器只会进行一次vote行为，以保证最后只会产生一个leader，每个参选者都需要用稳定的storage记录自己的vote行为**。

---

问题：这里需要记录vote之前当前机器自身是follower、leader或者candidate吗？

回答：假设机器宕机后恢复，且仍然在选举，它可以在选举结束后根据vote的结果知道自己是leader还是follower。（注意，vote情况记录在稳定的storage中，所以宕机恢复后也能继续选举流程。大不了就是选举作废，term增加又进行新一轮选举）

## 5.13 Raft-日志分歧(log diverge)

> [7.2 选举约束（Election Restriction） - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-07-raft2/7.2-xuan-ju-yue-shu-election-restriction)
>
> 如果你去看处理RequestVote的代码和Raft论文的图2，当某个节点为候选人投票时，节点应该将候选人的任期号记录在持久化存储中。（换言之，就算当前server的term记录落后于其他server，也可以通过通信知道下一次选举term值应该是多少，比如S1的term为5，但是S2的term为7，S1下次选举时也知道要从term8开始，而不是term6）

​	这里讨论一下不同raft server中log出现分歧的问题。

​	一个简单容易想到的场景即S1～S3都在log index 10～11有一致的log记录，而S1作为leader率先在log index12的位置写入log后宕机，之后S2、S3拿不到这个log记录。这个场景比较简单，我们暂不讨论。

​	一个复杂的场景如下：

+ S1在log index10位置有term3的记录，log index11～12暂无记录
+ S2在log index10位置有term3的记录，log index11有term3记录，log index12有term4记录
+ S3在log index10位置有term3的记录，log index11有term3记录，log index12有term5记录

​	**这里首先需要讨论的是，在raft协议下，有可能出现S2和S3在相同log index12下出现不同term记录的情况吗？**

​	答案是有可能。比如可以有如下推理

+ S2作为leader向S1和S3同步term3的log记录，于是S1～S3都在index10有term3记录

+ S1某一时刻宕机了，于是没有拿到后续log index11～12的记录
+ S2发起选举，term从3改成4，并且成功成为term4的leader
+ S2接收到新的请求，记录到log中，所以log index12出现term4记录
+ S2同步term4 log到S3之前，宕机
+ S3重新发起选举，此时也许S2恢复了，S3随后当选成为term5的leader（因为S2恢复，S3获取2票达到majority条件）
+ client向S3请求，S3记录term5的log，于是有log index12出现term5记录

---

![img](https://pic3.zhimg.com/80/v2-473c0c978279aadd20c01654426aad8a_1440w.webp)

这里进行一个复杂场景的讨论，如果原本系统中有某个server宕机，剩余server中谁会成为leader？

<u>下面用（X~Z，Y）表示log index X~Z的位置有term Y的记录</u>

原leader（已宕机，这里不考虑能恢复的情况），原leader仅比a多一条log记录（10，6）

+ a: (1~3, 1); (4~5, 4); (6~7, 5); (8~9, 6)
+ b: (1~3, 1); (4, 4);
+ c: (1~3, 1); (4~5, 4); (6~7, 5); (8~11, 6)
+ d: (1~3, 1); (4~5, 4); (6~7, 5); (8~10, 6); (11~12, 7)
+ e: (1~3, 1); (4~7, 4);
+ f:  (1~3, 1); (4~6, 2); (7~11, 3);

​	这里排出b、e、f，因为a～f都不宕机的情况下，b、e、f拥有的term都很小，绝对不可能获取到majority条件的选票数。<u>因为term较大的server会拒绝其他term小的server的拉票请求</u>。

​	**这里有可能作为新leader的candidate只有a、c、d**。

​	*下面这些内容，属于我自己的猜想，和这个网课无关*

​	这里a、c、d都可能当选，因为最新的term记录在每个server都会有记录，所以下一次不管谁发起选举，都会从term8开始发起选举（或者说就算一开始不知道，只要和d通信过，其他server就会知道下一次选举的term应该是7之后的8，因为通信完一圈，只有d有目前最大的term值）。只要谁先获取到足够多的选票，谁就有可能成为下一个leader。

​	~~**但是只有d最后会当选成leader，因为a、c、d除了投票自己后，还会向其他服务器拉票，一旦a、c发现d拥有最大的term之后，都会放弃成为leader，退让成为follower**。（因为a、c向d拉票的时候，会被d拒绝，并且d会告知a、c自己拥有更大的term8。这里是term8而不是term7，因为d会先将term改成8，之后vote自己，然后再向其他server拉票，所以a和c向自己拉票的时候会收到d的拒绝和被告知d已经达到term8了。）**且这里有个注意点，这里前提是原本leader宕机，而且原本leader记录其实宕机前的部分和d是对齐的。原本leader的所有log都是commited的**。**d当选leader后，会要求其他的server的log和自己对齐，不一致/有分歧的部分会按照d的log覆盖，保证之后系统中所有的server的log对齐。**~~

---

问题：上面a～f争抢leader的场景中，为什么d有term7记录，它原本是leader吗？

回答：d至少是term7的leader，否则它不会有term7的log记录。

问题：前面小节提到leader同步log给其他follower后，会进行commit，这里会不会出现leader自己先commit了，结果崩溃了，其他follower还没有commit？

回答：不会。因为leader会等待确认了其他follower能commit了，再执行commit，然后才会告知上层的KV服务对应的client请求消息。

**追问：这里有个问题，leader不发送其他消息的话，仅发送同步log消息给其他followers，那又怎么知道其他follower能够commit了？因为leader必须等大多数follower能够commit后才会执行自己的commit。**

**回答：我们有说过可以执行一些试探性的操作，这里可以有tentative的log传输，来确认其他follower是否能够commit。（我个人猜测，log同步传输后，follower接收到log并写入storage回应ACK。之后leader准备commit之前，又发送tentative试探性消息，而大多数follower收到消息，先将tentative记录到log中表示自己随时可以commit，然后响应ACK。此时大多数follower回应leader可以commit了，尽管follower可能还没有完成commit操作，但是leader可以进行commit了。因为此时尽管followers中出现宕机，其恢复时通过log也会知道自己在commit状态中，需要把收到的log进行commit。）**

问题：如果leader追加了很多log，然后崩溃了，那么后续选举出来的新leader，会把旧leader同步过来的log进行commit吗？

回答：可能会，也可能不会。这里场景比较复杂，就算新leader没有做commit，即不响应client也没事。因为KV服务的client会认为服务失败（毕竟election阶段无法对外提供服务），client会重新发起请求，所以新leader会插入一条新的log entry，之后再执行log同步、commit的完整流程。

问题：raft和其他课前提到的算法，有什么可以优化的点吗？比如使用批处理之类的，看起来raft很适合使用批处理，因为leader可以一次再log中放入不止一个log entry。或者说raft在性能角度有什么劣势吗？

回答：首先，raft并没有采用批处理来一次插入积攒的多个log entry，也许是因为这会让协议变得更复杂。也许像你所说，在插入log entry的地方增加批处理能提高性能，但是Raft没这么做，仅此而已。可以认为Raft有很多可以提高性能的潜在手段，但是Raft没有实现它们。（因为这个提问者表示自己在实验中尝试通过批处理的形式插入段时间内积攒的log entry，并且表示暂时没遇到什么问题）。

问题：看上去如果出现log丢失，可能client就无法得到响应，所以Raft是不是只适用于能够允许log丢失的场景或服务？

回答：是的，<u>一般来说需要client能够支持重试机制。并且如果有重复的请求，就如前面提到的，需要Raft的上层服务维护**重复检测表(duplicate detection table)**来保证重复请求不会导致错误的业务结果</u>。

# Lecture6 Lab1 Q&A

# Lecture7 容错-Raft(Fault Tolerance-Raft) -2

+ Log divergence
+ Log catch up
+ Persistence
+ Snapshots
+ Linearization

## 7.1 Raft-leader选举规则

> [7.2 选举约束（Election Restriction） - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-07-raft2/7.2-xuan-ju-yue-shu-election-restriction)
>
> 在Raft论文的5.4.1，Raft有一个稍微复杂的选举限制（Election Restriction）。这个限制要求，在处理别节点发来的RequestVote RPC时，需要做一些检查才能投出赞成票。这里的限制是，节点只能向满足下面条件之一的候选人投出赞成票：
>
> 1. 候选人最后一条Log条目的任期号**大于**本地最后一条Log条目的任期号；
> 2. 或者，候选人最后一条Log条目的任期号**等于**本地最后一条Log条目的任期号，且候选人的Log记录长度**大于等于**本地Log记录的长度

Leader election rule：

+ **majority：大多数原则**，即至少获取整个系统内大于全部机器数量一半的选票（包括自己，且每人只能投一次票，宕机的机器也算在系统机器总数内。如果剩余机器数压根凑不到刚好大于一半的机器数，则没有人能够成功获选）
+ **at-least-up-to-date：能当选的机器一定是具有最新term的机器**

## 7.2 Raft-日志覆写同步(未优化版本)Log catch up(unoptimized)

+ **nextIndex**：所有raft节点都维护nextIndex乐观的变量用于<u>记录下一个需要填充log entry的log index</u>。这里说乐观，因为当leader当选时，leader会初始化nextIndex值为当前log index值+1，表示认为leader自身的log一定是最新的
+ **matchIndex**：leader为所有raft节点(包括leader自己)维护一个悲观的matchIndex用于记录leader和其他follower从0开始往后最长能匹配上的log index的位置+1，<u>表示leader和某个follower在matchIndex之前的所有log entry都是对齐的</u>。这里说悲观，因为leader当选时，leader会初始化matchIndex值为0，表示认为自身log中没有一条记录和其他follower能匹配上。随着leader和其他follower同步消息时，matchIndex会慢慢增加。**leader为每个自己的follower维护matchIndex，因为平时根据majority规则，需要保证log已经同步到足够多的followers上**。

​	假设这里有S1～S3三台服务器组成Raft集群，每个Server的log记录如下，(X, Y)表示在log index X有log entry term=Y的log记录：

+ S1：(10, 3)
+ S2：(10, 3); (11, 3); (12, 5)
+ S3：(10, 3); (11, 3); (12, 4)

​	这里可以看出来S2是term5的leader。

​	这里S2通过heartbeats流程顺带发起log catch up，即想要和其他followers同步log entry的整体记录情况，按照majority原则，只需要向除了自身外的一台服务器发送消息即可，这里假设向S3发请求。

+ S2向S3，发送heartbeat，携带信息（**当前nextIndex指向的term，nextIndex-1的term值，nextIndex-1值**），即(空，5，12)
+ S3收到后，检查自己的log发现自己log index12为term4，回复S2一条否定消息no，表明自己还存活，但是不能同意S2要求的append操作，因为S3自己发现自己的log落后了。
+ S2看到S3的否定回应后，认为S3落后于自己，于是将自己的nextIndex从13改成12
+ S2重新发一条请求到S3，这次nextIndex是12，所以携带信息(5, 3, 11)
+ S3接收到后，检查自己log index11的位置为3，发现和S2说的一样，于是按照S2的log记录，在自己log index12的位置将term4改成term5，然后回复S2一条确定消息ok

+ S2收到来自S3的ok后，认为S3这次通信后log和自己对齐是最新的了

+ S2将自己维护的对应S3的matchIndex更新为13，表示log index13之前的log entry，作为leader的S2和作为follower的S3是对齐的

​	<u>到这里为止，S2能够知道log index12的log entry term5至少在2个server上得到复制(S2和S3)，已经满足了majority原则了，所以S2能将消息传递到上层应用了。不幸的是，这不完全是对的</u>。下面会讨论为什么。

​	**这里未优化的版本有个很大的问题，那就是如果Raft集群中出现log落后很多的server，leader需要进行很多次请求才能将其log与自己对齐**。

## 7.3 Raft-日志擦除(Erasing log entries)

> [聊聊RAFT的一个实现(4)–NOPCommand – 萌叔 (vearne.cc)](https://vearne.cc/archives/1851)

​	commit after the leader has committed one entry in its own term.

| log index | 1    | 2    | 3    |
| --------- | ---- | ---- | ---- |
| S1        | 1    | 2    | 4    |
| S2        | 1    | 2    |      |
| S3        | 1    | 2    |      |
| S4        | 1    |      |      |
| S5        | 1    | 3    |      |

​	这里网课描述一个场景，大意就是5台server中，S5在log index2有term3，而某时刻S1在log index3有term4，其他server都还没有log index3的记录。

+ 如果S5率先完成log index1～2的提交，那么S5可能先提起log catch up，导致S1～S4的log都被强制和自己对齐。即S1～S5的log变得完全一致（这里S1多出来的log index3记录会被清除），都变成(1, 3)，因为log index1的位置S1～S5一致，所以log index2的位置都被覆盖成3即可
+ 如果S1筛选完成log index1～3的提交，那么S1～S3的服务器会被覆盖记录成(1,2,4)，而S4～S5不变，因为只有S2和S3在log index1～2和S1是对齐的。

## 7.3 Raft-日志快速覆写同步(Log catch up quikly)

|      | 1    | 2    | 3    | 4    | 5    |
| ---- | ---- | ---- | ---- | ---- | ---- |
| S1   | 4    | 5    | 5    | 5    | 5    |
| S2   | 4    | 6    | 6    | 6    | 6    |

​	如果通过前面"7.2 Log catch up(unoptimized)"流程，可知道假设S2要向S1同步历史log的记录，那么需要从log index5（nextIndex=6）开始请求，一直请求到log index1（nextIndex=2）的位置后，才能找到S2和S1对齐的第一个位置log index1，然后又以1log index为单位，一直同步到nextIndex=6为止。这显然很浪费网络资源。

​	这里Log catch up quickly在论文中没有很详细的描述，但是大致流程如下：

+ S2假设在term7当选leader，于是nextIndex=6，如之前一样，向S1发送heartbeat时携带log同步信息，(空，6，5)，对应（当前nextIndex指向的term，nextIndex-1的term，nextIndex-1值）
+ S1收到后，对比自己logIndex5位置为term5。此时S1不再是简单返回no，还顺带回复自己的log信息（即请求中logIndex位置的term值，这个term值最早出现的logIndex位置），这里S1回复（5，2），表示S2heartbeat中说的logIndex5位置自己是term5不对齐，并且term5的值在自己log可追溯到logIndex2
+ S2收到回应后，可以直接将nextIndex改成2，并且下次heartbeat携带的信息变成（[6,6,6,6], 4, 1），表示nextIndex即往后的数据为[6,6,6,6]
+ S1收到heartnbeat后，发现logIndex1是term4是对齐的，于是按照S2说的，将logIndex2开始往后的共4个位置替换成[6,6,6,6]。

## 7.4 Raft-持久化(Persistence)

> [7.4 持久化（Persistence） - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-07-raft2/7.4-chi-jiu-hua-persistent)
>
> 持久化currentTerm的原因要更微妙一些，但是实际上还是为了实现一个任期内最多只有一个Leader，我们之前实际上介绍过这里的内容。如果（重启之后）我们不知道任期号是什么，很难确保一个任期内只有一个Leader。 
>
> 在这里例子中，S1关机了，S2和S3会尝试选举一个新的Leader。它们需要证据证明，正确的任期号是8，而不是6。如果仅仅是S2和S3为彼此投票，它们不知道当前的任期号，它们只能查看自己的Log，它们或许会认为下一个任期是6（因为Log里的上一个任期是5）。如果它们这么做了，那么它们会从任期6开始添加Log。但是接下来，就会有问题了，因为我们有了两个不同的任期6（另一个在S1中）。这就是<u>为什么currentTerm需要被持久化存储的原因，因为它需要用来保存已经被使用过的任期号</u>。

​	我们这里主要考虑Reboot重启时发生/需要做的事情。

+ 策略1：一个Raft节点崩溃重启后，必须重新加入Raft集群。即对于整个Raft集群来说，重启和新加入Raft节点没有太大区别
  + 重新加入(re-join)，重新加入Raft集群
  + 重放日志(replay the log)，需要重新执行本地存储的log（我理解上只有未commit的需要重放，当然如果不能区分哪些commit的话，那就是所有现存log需要重放）

+ 策略2：快速重启(start from your persistence state)，从上一次存储的持久化状态(快照)的位置开始工作，后续可以通过log catch up的机制，赶上leader的log状态

​	人们更偏向于策略2，快速重启。这就需要搞清楚，需要持久化哪些状态。

---

​	Raft持久化以下状态state：

+ vote for：投票情况，因为需要保证每轮term每个server只能投票一次
+ log：崩溃前的log记录，**因为我们需要保证(promise)已发生的(commit)不会被回退**。否则崩溃重启后，可能发生一些奇怪的事情，比如client先前的请求又重新生效一次，导致某个K/V被覆盖成旧值之类的。
+ current term：崩溃前的当前term值。因为选举(election)需要用到，用于投票和拉票流程，并且**需要保证单调递增**(monotonic increasing)

---

问题：什么时候，server决定进行持久化的动作呢？

回答：每当上面提到的需要持久化的变量state发生变化时，都应该进行持久化，写入稳定存储(磁盘)，即使这可能是很昂贵的操作。你必须保证在回复client或者leader的请求之前，先将需要持久化的数据写入稳定存储，然后再回复。否则如果先回复，但是持久化之前崩溃了，你相当于丢失了一些无法找回的记录。

## 7.5 Raft-服务恢复(Service recovery)

​	类似的，服务重启恢复时有两种策略：

1. 日志重放(replay log)：理论上将log中的记录全部重放一遍，能得到和之前一致的工作状态。这一般来说是很昂贵的策略，特别是工作数年的服务，从头开始执行一遍log，耗时难以估量。所以一般人们不会考虑策略1。
2. **周期性快照(periodic snapshots)**：假设在i的位置创建了快照，那么可以裁剪log，只保留i往后的log。此时重启后可以通过snapshot快照先快速恢复到某个时刻的状态，然后后续可以再通过log catch up或其他手段，将log同步到最新状态。（一般来说周期性的快照不会落后最新版本太多，所以恢复工作要少得多）

​	这里可以扩展考虑一些场景，比如Raft集群中加入新的follower时，可以让leader将自己的snapshot传递给follower，帮助follower快速同步到近期的状态，尽管可能还是有些落后最新版本，但是根据后续log catch up等机制可以帮助follower随后快速跟进到最新版本log。

​	使用快照时，需要注意几点：

+ 需要拒绝旧版本的快照：有可能收到的snapshot比当前服务状态还老
+ 需要保持快照后的log数据：在加载快照时，如果有新log产生，需要保证加载快照后这些新产生的log能够能到保留

---

问题：看上去好像这破坏了抽象的原则，现在上层应用需要感知Raft的动作？

回答：是的，这里需要上层应用辅助一些Raft的工作，比如告知Raft我已经拥有i之前的快照了，可以对应的删除Raft在i状态之前的log了之类的。

问题：如果follower收到snapshot，这是它日志的前缀，由快照覆盖的log记录将被删除，而其余的会被保留。这种情况下，状态机是否会被覆盖？

回答：前面没有什么问题，这里主要需要关注快照如何和状态机通信。在课程实验中，状态机通过apply channel获取快照，然后做后续需要做的事情（比如根据快照，修改状态机中的变量之类的）。

## 7.6 使用Raft

​	重新回顾一下服务使用Raft的大致流程

1. 应用程序中集成Raft相关的library包

2. 应用程序接收Client请求

3. 应用程序调用Raft的start函数/方法

4. 下层Raft进行log同步等流程

5. Raft通过apply channel向上层应用反应执行完成

6. 应用程序响应Client

+ 并且前面提过，可能作为leader的Raft所在服务器宕机，所以Client必须维护server列表来切换请求的目标server为新的leader服务器。

+ 同时，有时候请求会失败，或者Raft底层失败，导致重复请求，而我们需要有手段辨别重复的请求。通常可以在get、put请求上加上请求id或其他标识来区分每个请求。一般维护这些请求id的服务，被称为clerk。提供服务的应用程序通过clerk维护每个请求对应的id，以及一些集群信息。

## 7.7 线性一致性/强一致性(Linearizability/strong consistency)

> [线性一致性_百度百科 (baidu.com)](https://baike.baidu.com/item/线性一致性/22395305?fr=aladdin)
>
> **线性一致性**(Linearizability)，或称**原子一致性**或**严格一致性，**指的是程序在执行的历史中在存在可线性化点P的执行模型，这意味着一个操作将在程序的调用和返回之间的某个点P起作用。这里“起作用”的意思是被系统中并发运行的所有其他线程所感知。
>
> [线性一致性：什么是线性一致性？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/42239873)

​	在论文中对整个系统提供的服务的正确性称为**线性一致性(Linearizability)**，线性一致性需要保证满足一下三个条件：

1. **整体操作顺序一致(total order of operations)**

   即使操作实际上并发进行，你仍然可以按照整体顺序对它们进行排序。（即后续可以根据读写操作的返回值，对所有读写操作整理出一个符合逻辑的整体执行顺序）

2. **实时匹配(match real-time)**

   顺序和真实时间匹配，如果第一个操作在第二个操作开始前就完成，那么在整体顺序中，第一个操作必须排在第二个操作之前(换言之如果这么排序后，整体的执行结果不符合逻辑，那么就不符合"实时匹配")。

3. **读操作总是返回最后一次写操作的结果(read return results of last write)**

---

**问题：这里说的线性一致性，是不是就是人们说的强一致性？**

**回答：是的。一般直觉就是表现上像单机，而技术文献中准确定义称为线性一致性。**

问题：人们为什么决定定义这个property？（指，线性一致性这个概念为啥会被定义出来）

回答：比如你希望多机系统对外表现如同单机一样，线性一致性就是非常直观的定义。数据库世界中有类似的术语，叫做**可串行化(serializability)**。基本上线性一致性和可串行化的唯一区别是，**可串行化不需要实时匹配(match real-time)**。当然，人们对强一致性有不同定义，而我们这里认为线性一致性就是一种强一致性。

问题：可以稍微详细一点介绍clerk吗？

回答：clerk是一个RPC库，它可以帮助记录请求的RPC服务器列表。比如它认为server1是leader，于是Client发请求时，通过clerk会发送到server1，如果server1宕机了，也许clerk根据维护的server列表，会尝试将Client的请求发送到server2，猜测server2是leader。并且clerk会标记每次请求(get、put等)，生成请求id，可以帮助server服务检测重复的请求。

问题：论文12页中提到follower擦除旧log，但是不能回滚状态机，是吗？

回答：正如前面日志擦除所说，Raft**可以擦除未提交(uncommitted)的log**。

**问题：server加载snapshot的时候，怎么保证后续还可以接收新的log**

**回答：可以在加载snapshot之前，先通过COW写时复制的fork创建子进程，子进程加载snapshot，而父进程继续提供服务，例如获取新的log之类的。因为子进程和父进程共享同样的物理内存，所以后续总有办法使得加载完snapshot的子进程获取父进程这段时间内新增的log。**

问题：当生成snapshot且因此压缩/删除旧log后，sever维护的log index是从0开始，还是在原本的位置继续往后？

回答：从原本的位置继续往后，不会回退log index索引。

# Lecture8 Lab2A_2B Q&A

# Lecture9 Zookeeper

高性能(high perference)：

+ 异步(asynchronous)
+ 提供一致性，但非强一致性(consistency)

通用的协调服务(Coordination service)

## 9.1 Zookeeper复制状态机(replicated state machine)

​	首先，Zookeeper是一个复制状态机(zookeeper is replicated state machine)。

​	类似Raft流程，Zookeeper对外服务时：

1. Client访问Zookeeper，create一个Znode
2. Zookeeper调用ZAB库(类似Raft库)，生成操作log，ZAB负责和Zookeeper集群的其他机器完成类似Raft的工作，日志同步、心跳保活等(整体思想和Raft类似，但实现方式不同)
   + ZAB保证日志顺序
   + 能避免网络分区、脑裂问题
   + 同样要求操作都是具有确定性的(换言之不会有非确定性的操作在ZAB之间传递)

3. ZAB完成日志同步等工作后，回应Zookeeper
4. Zookeeper处理逻辑，响应Client的create请求

​	在Zookeeper中，维护着由znode组成的tree树结构。

​	由于ZAB整体思路和Raft类似，这章节不会再重点讨论ZAB，我们主要围绕Zookeeper展开话题

## 9.2 Zookeeper吞吐量

​	视频展示一张论文中的图片，大意就是：

+ 写请求占比高时，Zookeeper实例机器越多，吞吐量越低（3台21K/s，13台9K/s）
+ 读请求占比高时，Zookeeper实例机器越多，吞吐量越高

​	为什么能有如此高的吞吐，大致原因如下，Zookeeper有以下几种策略/设计：

+ **异步(asynchronous)**

  操作都是异步的，Client可以一次向Zookeeper提交多个操作。比如Client一次提交多个put(批量操作)，但是实际上leader只向持久存储写入一次(而不是写入多次)，即Zookeeper批量整合多次写操作，只进行一次磁盘写入操作。

+ **读请求可以由任意server处理(read by any server)**

  + 这里不需要通过leader完成read操作，任何zookeeper实例节点都可以完成read操作
  + **并且其他zookeeper完成read操作时，不需要和leader有任何网络通信**

​	由于"读请求可以由任意server处理(read by any server)"，所以**在Zookeeper中，写后读可能读取到旧值**。

​	思考如下场景，有一个3实例组成的Zookeeper集群：

1. Client1向Zookeeper的leader进行put写请求（将X从0写成1）
2. Zookeeper根据majority原则，只需要自己和另一个Zookeeper实例将log写入稳定存储即可
3. 假设leader为Z1，则Z1、Z2完成log写入后，Z1响应Client
4. 过了很长一段时间后，另一个Client2再通过get请求向Zookeeper读取X，读取结果为什么？
   + 读取X为0，Client2正好读取到Z3，而Z3没有被写入log，所以X还是0
   + 读取X为1，Client2正好读取到Z1或Z2，所以X是1

---

​	**从这里可以看出来，Zookeeper的读写，不是线性一致性(Linearizability)的**：

+ 因为违背了"7.7 线性一致性/强一致性(Linearizability/strong consistency)"的规则2——**实时匹配(match real-time)**。这里Client向Zookeeper写入数据后的很长一段时间再进行读操作，很明显第一个写操作在第二个读操作开始前就完成，那么读操作需要返回X=1，才能满足"实时匹配(match real-time)"。
+ 同时，也违背了规则3——**读操作总是返回最后一次写操作的结果(read return results of last write)**。这里最后一次写操作导致X=1，那么很长时间后进行的读操作，应该读取X=1而不是X=0。

​	**这里如果想保证read能维持线性一致性，简单的做法就是每次read都请求leader**。当然你可以用其他方式来保证。

## 9.3 Zookeeper的一致性规则

> [8.5 一致保证（Consistency Guarantees） - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-08-zookeeper/8.5) <= 网络博主的笔记，这里摘抄部分，补充说明
>
> Zookeeper的一致性规则：
>
> 1. **写请求是线性一致性的(linearizable writes)**
>
>    对于所有的客户端发起的写请求，整体是线性一致性的。
>
> 2. **同一个客户端的请求按照FIFO顺序被执行(FIFO client order)**
>
>    这里的意思是，如果一个特定的客户端发送了一个写请求之后是一个读请求或者任意请求，那么首先，所有的写请求会以这个客户端发送的相对顺序，加入到所有客户端的写请求中（满足保证1）。所以，如果一个客户端说，先完成这个写操作，再完成另一个写操作，之后是第三个写操作，那么在最终整体的写请求的序列中，可以看到这个客户端的写请求以相同顺序出现（虽然可能不是相邻的）。所以，对于写请求，最终会以客户端确定的顺序执行。
>
>    对于读请求，这里会更加复杂一些。我之前说过，在Zookeeper中读请求不需要经过Leader，只有写请求经过Leader，读请求只会到达某个副本。所以，读请求只能看到那个副本的Log对应的状态。对于读请求，我们应该这么考虑FIFO客户端序列，客户端会以某种顺序读某个数据，之后读第二个数据，之后是第三个数据，对于那个副本上的Log来说，每一个读请求必然要在Log的某个特定的点执行，或者说每个读请求都可以在Log一个特定的点观察到对应的状态。<u>然后，后续的读请求，必须要在不早于当前读请求对应的Log点执行。也就是一个客户端发起了两个读请求，如果第一个读请求在Log中的一个位置执行，那么第二个读请求只允许在第一个读请求对应的位置或者更后的位置执行</u>。**第二个读请求不允许看到之前的状态，第二个读请求至少要看到第一个读请求的状态。这是一个极其重要的事实，我们会用它来实现正确的Zookeeper应用程序**。
>
>    这里特别有意思的是，如果一个客户端正在与一个副本交互，客户端发送了一些读请求给这个副本，之后这个副本故障了，客户端需要将读请求发送给另一个副本。这时，<u>尽管客户端切换到了一个新的副本，FIFO客户端序列仍然有效</u>。所以这意味着，如果你知道在故障前，客户端在一个副本执行了一个读请求并看到了对应于Log中这个点的状态，当客户端切换到了一个新的副本并且发起了另一个读请求，<u>假设之前的读请求在这里执行，那么尽管客户端切换到了一个新的副本，客户端的在新的副本的读请求，必须在Log这个点或者之后的点执行</u>。
>
>    **这里工作的原理是，每个Log条目都会被Leader打上zxid的标签，这些标签就是Log对应的条目号**。<u>任何时候一个副本回复一个客户端的读请求，首先这个读请求是在Log的某个特定点执行的，其次回复里面会带上zxid，对应的就是Log中执行点的前一条Log条目号。客户端会记住最高的zxid，当客户端发出一个请求到一个相同或者不同的副本时，**它会在它的请求中带上这个最高的zxid**</u>。这样，其他的副本就知道，应该至少在Log中这个点或者之后执行这个读请求。这里有个有趣的场景，**如果第二个副本并没有最新的Log，当它从客户端收到一个请求，客户端说，上一次我的读请求在其他副本Log的这个位置执行，那么在获取到对应这个位置的Log之前，这个副本不能响应客户端请求**。
>
>    **更进一步，FIFO客户端请求序列是对同一个客户端的所有读请求，写请求生效**。<u>所以，如果我发送一个写请求给Leader，在Leader commit这个请求之前需要消耗一些时间，所以我现在给Leader发了一个写请求，而Leader还没有处理完它，或者commit它。之后，我发送了一个读请求给某个副本。这个读请求需要暂缓一下，以确保FIFO客户端请求序列。读请求需要暂缓，直到这个副本发现之前的写请求已经执行了。这是FIFO客户端请求序列的必然结果，（对于某个特定的客户端）读写请求是线性一致的</u>。最明显的理解这种行为的方式是，如果一个客户端写了一份数据，例如向Leader发送了一个写请求，之后立即读同一份数据，并将读请求发送给了某一个副本，那么客户端需要看到自己刚刚写入的值。如果我写了某个变量为17，那么我之后读这个变量，返回的不是17，这会很奇怪，这表明系统并没有执行我的请求。因为如果执行了的话，写请求应该在读请求之前执行。所以，副本必然有一些有意思的行为来暂缓客户端，比如当客户端发送一个读请求说，我上一次发送给Leader的写请求对应了zxid是多少，这个副本必须等到自己看到对应zxid的写请求再执行读请求。

​	**再次声明，Zookeeper并不提供读的线性一致性**("9.2 Zookeeper吞吐量"有进行过场景分析了)。

​	**Zookeeper并不是遵循线性一致性实现的，它有自己的一致性定义和实现**：

+ **线性化写入(Linearize write)**

  所有的写入操作，必须经过Zookeeper的leader。底层会将生成的log以相同的顺序同步到所有zookeeper节点的log记录中。(即对于写入操作，还是应用复制状态机的方案进行majority的写同步)

+ **先进先出队列维护Client请求(FIFO client order)**

  尽管请求是异步的，所有的client请求，在zookeeper实例中以FIFO的形式维护。<u>即如果client1先于client2请求，那么client2的处理一定在client1之后，client2的请求能观测到client1的操作结果</u>。

  + **按client请求进行写操作(write in client order)**

    这是FIFO client order的附带产物，很好理解。因为client请求以FIFO顺序被zookeeper处理，所以写操作也是按照client请求顺序生效。

  + **读操作能观测到同一个client的最后一次写操作(read obverse last write from same client)**

    **如果<u>相同的client</u>执行完写操作后，立即进行读操作，那么至少可以看到自己的写入结果。换言之，如果是其他client在前一个client写后进行读，zookeeper并不保证能读取到前一个client的写结果**。

  + **读操作会观察到日志的某些前缀(read will observe some prefix of the log)**

    这意味着你可以看到旧数据，可能某个follower有指定前缀的log，但是log里面没有最新的值记录，但是follower仍然可以返回读取的结果。因为zookeeper只保证读取可以观察到log的前缀。按照这个特定，读取可以不按照顺序。

  + **不能读取旧前缀的数据(no read from from the past)**

    <u>这意味着如果你第一次看到了前缀prefix 1，然后你发出读取前缀prefix 1。然后你执行第二次读取，第二次读取必须至少看到prefix 1或者更大的前缀数据(比如prefix 2)，但是绝不能看到比precix 1更小的前缀</u>。

## 9.4 Zookeeper的一致性实现

> [8.6 同步操作（sync） - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-08-zookeeper/8.6-tong-bu-cao-zuo-sync)
>
> Zookeeper有一个操作类型是sync，它本质上就是一个写请求。假设我知道你最近写了一些数据，并且我想读出你写入的数据，所以现在的场景是，我想读出Zookeeper中最新的数据。这个时候，我可以发送**一个sync请求，它的效果相当于一个写请求，所以它最终会出现在所有副本的Log中**，尽管我只关心与我交互的副本，因为我需要从那个副本读出数据。<u>接下来，在发送读请求时，我（客户端）告诉副本，在看到我上一次sync请求之前，不要返回我的读请求</u>。
>
> **如果这里把sync看成是一个写请求，这里实际上符合了FIFO客户端请求序列，因为读请求必须至少要看到同一个客户端前一个写请求对应的状态。所以，如果我发送了一个sync请求之后，又发送了一个读请求。Zookeeper必须要向我返回至少是我发送的sync请求对应的状态**。
>
> <u>不管怎么样，如果我需要读最新的数据，我需要发送一个sync请求，之后再发送读请求。这个读请求可以保证看到sync对应的状态，所以可以合理的认为是最新的。但是同时也要认识到，这是一个代价很高的操作，因为我们现在将一个廉价的读操作转换成了一个耗费Leader时间的sync操作。所以，如果不是必须的，那还是不要这么做</u>。

​	论文中没有特别详细的说明Zookeeper怎么实现"9.3 Zookeeper的一致性规则"中制定的规则，但是我们可以大致猜测其实现。

1. **当客户端想要连接zookeeper服务时，会创建一个session**，它使用session信息连接到zookeeper集群，并在整个session期间维护状态state。
2. Client创建session后和Zookeeper的leader建立连接，随后发起一个write写请求
3. 类似Raft流程，leader生成一个log插入到log记录中，其索引称为zxid。
4. leader类似Raft进行log日志同步，将log同步到majority数量的节点(包括自己)
5. **leader响应Client，同时返回zxid给客户端，客户端会在session中存储这次写入的zxid信息**
6. **Client向zookeeper发起read请求，这个read使用上次写入返回的zxid标记**
7. **假设被请求的follower正好没有这个zxid对应的log记录，于是它会等待直到收到leader的这条zxid同步log后，才会响应Client**

8. 此时假设Client又发起write请求，并且最后在leader和follower1产生zxid=1
9. 随后Client发起read请求，但携带的zxid=0(假设是最早一次写返回的)，并且读取的是follower2
10. **follower2正好有zxid=0的log记录，尽管它还没有zxid=1的记录，但是它仍会响应client的读zxid=0的请求**。<u>**这意味着Zookeeper存在读取旧值的情况(但不允许主动读取旧值)**，这里说不定zxid=1的记录是某个变量的新值，而zxid=0则是旧值（注意，这里如果follower2已经有了zxid=1的记录后，会返回zxid=1的值，而不是返回zxid=0的值，因为不允许返回旧log前缀的数据）</u>。

---

问题：据我所知，好像一般session具有粘性，即上面读写应该会尽可能发生在同一个zookeeper节点，所以上面流程是可能发生的吗？

回答：这里可以假设曾经发生过一个短暂的网络分区或类似的事情，所以上面的流程是可能发生的。另外zookeeper做了一些负载均衡，但是无论如何，上面流程是可能发生的。

**问题：上面写入时说zookeeper会返回zxid，是不是需要已经commit了才会返回zxid？**

**回答：是的，类似Raft流程，这里需要log被commit后，才会返回zxid。**

**问题：这里后续的read携带zxid=0，但是系统中已经有节点存在zxid=1的数据了，看上去似乎违背了zookeeper一致性原则中的"<u>不能读取旧前缀的数据(no read from from the past)</u>"的规则？因为看上去client好像在故意请求日志中特定的前缀数据，而且还是读取旧的数据。**

**回答：实际上，这里zxid含义上表示阻止时间倒流的计数器，作为follower必须至少含有zxid值截止的日志前缀，才能够响应client，而如果你有更高的zxid并不会有问题。这阻止了故意读取旧数据的可能性。**

学生提问：也就是说，从Zookeeper读到的数据不能保证是最新的？

Robert教授：完全正确。我认为你说的是，从一个副本读取的或许不是最新的数据，所以Leader或许已经向过半服务器发送了C，并commit了，过半服务器也执行了这个请求。但是这个副本并不在Leader的过半服务器中，所以或许这个副本没有最新的数据。这就是Zookeeper的工作方式，它并不保证我们可以看到最新的数据。Zookeeper可以保证读写有序，但是只针对一个客户端来说。所以，<u>如果我发送了一个写请求，之后我读取相同的数据，Zookeeper系统可以保证读请求可以读到我之前写入的数据。但是，如果你发送了一个写请求，之后我读取相同的数据，并没有保证说我可以看到你写入的数据</u>（这里同一个人表示同一个client）。这就是Zookeeper可以根据副本的数量加速读请求的基础。

学生提问：那么Zookeeper究竟是不是线性一致呢？

Robert教授：我认为Zookeeper不是线性一致的，但是又不是完全的非线性一致。首先，所有客户端发送的请求以一个特定的序列执行，所以，某种意义上来说，**所有的写请求是线性一致的**。同时，每一个客户端的所有请求或许也可以认为是线性一致的。尽管我不是很确定，Zookeeper的一致性保证的第二条可以理解为，**单个客户端的请求是线性一致的**。

学生提问：zxid必须要等到写请求执行完成才返回吗？

Robert教授：实际上，我不知道它具体怎么工作，但是这是个合理的假设。当我发送了异步的写请求，系统并没有执行这些请求，但是系统会回复我说，好的，我收到了你的写请求，如果它最后commit了，这将会是对应的zxid。所以这里是一个合理的假设，我实际上不知道这里怎么工作。之后如果客户端执行读请求，就可以告诉一个副本说，这个zxid是我之前发送的一个写请求。

## 9.5 Zookeeper中watch通知

> [Zookeeper——Watch机制原理_庄小焱的博客-CSDN博客_zookeeper watch](https://blog.csdn.net/weixin_41605937/article/details/122095904) <= 推荐阅读
>
> 大体上讲 ZooKeeper 实现的方式是通过客服端和服务端分别创建有观察者的信息列表。客户端调用 getData、exist 等接口时，首先将对应的 Watch 事件放到本地的 ZKWatchManager 中进行管理。服务端在接收到客户端的请求后根据请求类型判断是否含有 Watch 事件，并将对应事件放到 WatchManager 中进行管理。
>
> 在事件触发的时候服务端通过节点的路径信息查询相应的 Watch 事件通知给客户端，客户端在接收到通知后，首先查询本地的 ZKWatchManager 获得对应的 Watch 信息处理回调操作。这种设计不但实现了一个分布式环境下的观察者模式，而且通过将客户端和服务端各自处理 Watch 事件所需要的额外信息分别保存在两端，减少彼此通信的内容。大大提升了服务的处理性能。
>
> **需要注意的是客户端的 Watcher 机制是一次性的，触发后就会被删除。**
>
> [zookeeper的watch机制原理解析_java_脚本之家 (jb51.net)](https://www.jb51.net/article/252975.htm) <= 推荐阅读，具体的指令操作示例，很好理解
>
> ----
>
> [8.7 就绪文件（Ready file/znode） - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-08-zookeeper/8.7-jiu-xu-wen-jian-ready-fileznode) <= 2021版这里讲得不太清楚，可以看看网络上2020版对于watch的讲解

​	这里举了一些例子，大概就是在讨论怎么让read读取到最后的write。然后提到zookeeper提供的watch监听指定写事件的变化，**zookeeper保证被监听的写事件发生后，会在下一次写操作发生前完成事件通知**。

## 9.6 ZNode API

> [9.1 Zookeeper API - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-09-more-replication-craq/9.1-zookeeper-api) <= 可以参考这个笔记，里面更详细
>
> 回忆一下Zookeeper的特点：
>
> - Zookeeper基于（类似于）Raft框架，所以我们可以认为它是，当然它的确是容错的，它在发生网络分区的时候，也能有正确的行为。
> - 当我们在分析各种Zookeeper的应用时，我们也需要记住Zookeeper有一些性能增强，使得读请求可以在任何副本被处理，因此，可能会返回旧数据。
> - 另一方面，Zookeeper可以确保一次只处理一个写请求，并且所有的副本都能看到一致的写请求顺序。这样，所有副本的状态才能保证是一致的（写请求会改变状态，一致的写请求顺序可以保证状态一致）。
> - 由一个客户端发出的所有读写请求会按照客户端发出的顺序执行。
> - 一个特定客户端的连续请求，后来的请求总是能看到相比较于前一个请求相同或者更晚的状态（详见8.5 FIFO客户端序列）。

​	通常来说会给应用app1创建一个znode节点，然后znode下再挂在一些子节点，上面记录app1的一些属性，比如IP等。

​	ZNode有三种类型：

+ regular：常规节点，它的容错复制了所有的东西
+ ephemeral：临时节点，节点会自动消失。比如session消失或Znode有段时间没有传递heartbeat，则Zookeeper认为这个Znode到期，随后自动删除这个Znode节点
+ sequential：顺序节点，它的名字和它的version有关，是在特定的znode下创建的，这些子节点在名字中带有序列号，且节点们按照序列号排序（序号递增）。

---

+ create接口，参数为path、data、flags。`create(path, data, flags)`，这里flags对应上面ZNode的3种类型

+ delete接口，参数为path、version。`delete(path, version)`
+ exists接口，参数为path、watch。`exists(path, watch)`
+ getData原语，参数为path、version。`getData(path, version)`
+ setData原语，参数为path、data、version。`setData(path, data, version)`
+ getChildren接口，参数为path、watch。`getChildren(path, watch)`，可以获取特定znode的子节点

## 9.7 Zookeeper-小结

+ 较弱的一致性(weaker consistency)
+ **适合用于当作配置服务(configuration)**

+ 高性能(high perference)

# Lecture10 Patterns and Hints for Concurrency in Go

> 这个建议直接看[视频](https://www.bilibili.com/video/BV16f4y1z7kn/?p=10)，讲师主要围绕一些业务场景举例go代码实现和优化技巧。go虽然学过，但并不是我主要用于项目编程的语言，所以这里不打算做太多笔记。
>
> ----
>
> [【熟肉】100秒介绍LLVM_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1RF411K7F5/?spm_id_from=333.337.search-card.all.click&vd_source=ba4d176271299cb334816d3c4cbc885f)

## 10.1 一些优化go代码的思路

1. 在能简化程序理解的前提下，同一个程序的代码实现，可以考虑将**数据状态**转变成**代码状态**(Convert data state into code state when it make programs clearer)

   视频中举例的是一个正则表达式的匹配`/"([^"\\]|\\.)*"/`，可以用一个看着很复杂的swith-case程序实现（数据状态），但是通过一系列优化后，最后变成只有两个if和一个for语句，并且更容易读懂和理解。

   优化前的原始代码：

   ```go
   state := 0
   for {
     c := readChar()
     switch state {
       case 0:
       if c != '"' {
         return false
       }
       state = 1
       case 1:
       if c == '"' {
         return true
       }
       if c == '\\' {
         state = 2
       } else {
         state = 1
       }
       case 2:
       state = 1
     }
   }
   ```

   优化后的代码（正巧执行也更快，但不是这里讲解的重点，重点是易读性）：

   ```go
   func readString(readChar func() rune) bool {
     if readChar() != '"' {
       return false
     }
     
     var c rune
     for c != '"' {
       c := readChar()
       if c == '\\' {
         readChar()
       }
     }
     return true
   }
   ```

2. 可通过启动新的goroutine来存储code state代码状态(Use additional goroutines to hold addional code state)

   这让我们可以移动程序计数器，将我们不能在原本stack上完成的逻辑，转移到goroutine的栈上去完成。当然创建goroutine需要保证其能够被清除，避免内存泄漏问题。

3. 可通过启动新的goroutine来完成mutex条件变量的工作，如果能简化代码的话(Convert mutexes into goroutines when it makes progrmas clearer)

   这里视频通过发布订阅服务器的代码实现举例，大意就是原本的发布订阅相关的方法中，需要用mutex保护临界区map的读写，会有频繁的加锁、解锁流程。而改写代码后，通过新启动一个goroutine维护这个map，原本的线程通过channel与这个goroutine交互，不再需要加锁、解锁流程（这里其实相当于利用channel代替锁的功能，channel本身会阻塞直到读写方都准备好，其实内部也是有锁，只是不需要开发者再显示声明锁）。

## 10.2 设计模式1: 发布订阅服务器(Publish/subscribe server)

## 10.3 设计模式2: 工作调度器(Worker scheduler)

## 10.4 设计模式3: 复制服务的客户端(Replicated service client)

## 10.5 设计模式4: 协议选择器(protocol multiplexer)

# Lecture11 链式复制(Chain Replication)

## 11.1 Zookeeper锁

​	一开始提到一种粗糙的zookeeper锁实现方式，利用zookeeper的write具有线性一致性的规则，可以靠Zookeeper提供的原语实现Lock和Unlock。

​	粗糙的Lock实现中，大致逻辑即：

1. 尝试创建一个临时znode节点(ephemeral)，如果存在则返回（表示获取到锁）
2. 如果znode节点已存在exist，则阻塞wait等待(这里通过watch监听znode变更事件)，直到znode消失/被删除后，重新发起create一个znode请求（尝试获取锁）
3. 获取锁后，解锁只需要delete之前创建的临时znode即可。

​	<u>这里就算client请求Lock后突然崩溃，Zookeeper也会检查请求Lock的client是否存活，如果client崩溃，则zookeeper服务端会删除这个client的记录，取消Lock动作</u>。

​	看上去这个Lock没什么问题，能保证多个client请求Lock时也只有一个能获取Lock。问题是会造成羊群效应，即1000个client请求，只有1个成功，剩下999个等待，下次lock释放后，又只有1个成功，998个失败后等待，尽管每次只有1个client能获取锁，却伴随大量的Lock请求，是个灾难。

----

​	一个更优的Lock实现伪代码（论文中的伪代码）如下：

```pseudocode
Lock
1 n = create(l + "/lock-", EPHEMERAL | SEQUENTIAL)
2 C = getChildren(l, false)
3 if n is lowest znode in C, exit
4 p = znode in C ordered just before n
5 if exists(p, true) wait for watch event
6 goto 2

Unlock
1 delete(n)
```

​	**可以看到这里没有retry重试加锁的请求操作（前面的粗糙实现中，所有未成功获取锁的客户端都将重试获取），相反的，所有的客户端排成一条线，按序获取锁。**

+ 1行的`SEQUENTIAL`意味着锁文件被创建时，第一个将是"lock-0"，下一个是"lock-1"，以此类推...。这里n获取到对应的序列号，比如第一个锁文件返回n=0

  按照前面举例1000个client最后会创建"lock-0"～"lock-999"这1000个锁文件。

+ 3行，如果当前n已经是创建的znode中锁文件序号最小的，则返回，表示获取锁成功
+ 4行，表示当前n不是当前最小的锁序号，于是获取n的前一个序号，比如n=10，则p=9

+ 5行，这里对p序号对应的锁文件加上watch，表示前一个client通过Unlock删除锁文件后，下一个也就是当前自己这个client就会监听到事件，然后重新goto代码2行走一遍逻辑，此时n会是最小的锁序号，所以当前client获取到lock，然后返回

​	这种锁也被称为**标签锁(ticket locks)**。

​	需要注意的是，这里zookeeper的锁(下面简称为zlock)，和go里面的lock不一样。

​	一旦zookeeper确认zlock的持有者failed了（比如通过session的心跳机制等），我们会看到一些中间状态(intermediate state)。**持有zlock的client崩溃后，zookeeper会撤销client原本持有的zlock锁**。而由于锁被撤销了，会导致临界区的访问出现一些中间状态(因为不再受锁保护了)，而log可以保证临界区访问的正确性。

​	**zlock一般应用场景**：

+ **选举(leader election)**：从一堆client中选举一个leader。并且如果有必要，可以让leader清理中间状态(临界区的一些状态值，比如把state清0之类的)。
+ **软锁("soft" lock)**：通常执行map只需要一个mapper，可以通过软锁保证mapreduce的worker中一次只有一个mapper接受某个特定的map任务。当然，这里mapeer如果failed，锁就会被释放，然后其他mapper可以尝试获取锁，重新执行这个mapper任务。而对于mapreduce程序来说，重复执行(执行两次)是可接受的，因为函数式编程，相同输入下运行几次都应该有相同的输出结果。

---

**问题：前面粗糙版本的Zookeeper锁实现中，zookeeper是怎么确定client已经failed，从而释放ephemerial锁？比如通常只是出现短暂的网络分区。**

回答：**因为client和zookeeper交互时会创建session，zookeeper需要和client在session中互相发送心跳，如果zookeeper一段时间没有收到client的心跳，那么它认为client掉线，并关闭session**。session关闭后，即使client想在原session上发送消息，也行不通。**而这个session期间被创建的ephermeral文件在session关闭后会被删除**。假设client因为网络分区联系不上，如果超过zookeeper容忍的心跳检测阈值，那么zookeeper会关闭session，client必须重新创建一个session。

问题：在zlock中，如果持有锁的server宕机，那么zlock会被撤销。但是不是没宕机，也一样会被撤销，因为zlock有一个标记是"EPHEMERAL"。

回答：是的。只有EPHEMERAL文件才会发生这种情况（锁被撤销，因为client不管什么原因导致的session被关闭后，session周期内存活的EPHEMERAL锁文件自然会被删除）。

**追问：那么我们能不能通过不传递EPHEMERAL，来模拟go锁？**

**回答：你可以这么做，但是一旦宕机后就没有其他cleint能够释放这个锁文件了，会导致死锁。因为能释放这个lock文件的client宕机了，并且去掉EPHEMERAL后，锁文件永久存在，其他client排队永远无法获取锁，而且也没法释放不属于自己的锁文件来解围（这时就需要人工介入删除指定的lock文件了）**。这就是为什么zlock需要EPHEMERAL，保证只在session内存活，避免死锁问题。

追问：client宕机后，其他client真的不能删除这个lock文件吗？理论上只要想的话，其他client可以通过自己编写的delete来删除lock文件吧？

回答：这是一个共识问题（consensus problem），你当然可以通过delete任意删除别人的lock。但是这么做的话，lock本身就没有太大意义了，因为lock本身就是为了保护临界区访问，但是你破坏了规则，任何人可以随意删除其他人的锁。

问题：可以再解释一下"soft" lock软锁吗？

回答：软锁意味着一个操作可以发生两次。正常情况下，一个map被mapper施行成功后，会释放zlock，一切正常。但是mapper中途失败后，由于session断开，zookeeper撤销这个zlock，其他mapper会获取锁，接替原本的mapper执行map任务，也就是说map会被执行两次。但是这里是可接受的，因为mapreduce是函数式编程，相同输入运行几次都得有相同的输出。

问题：在leader选举的zlock使用举例中，这里可以暴露的中间状态(intermediate state)是指什么？

回答：纯粹的leader election不会有中间状态，但是通常leader会创建配置文件，就像zookeeper那样，使用ready trick。（提问者补充表示懂了，所以leader把文件整个创建好，然后原子地进行convert并且rename。讲师表示没错）。

问题：能解释一下zookeeper的ready trick指的是什么吗？

回答：讲师表示事件不多，课后问再回答，并表示上次说过了。（这里ready file应该指的是watch机制讲解的时候提的用例，即client监听ready file的变更事件，然后作出一些动作）可以参考更详细的关于ready file举例的网络笔记 => [8.7 就绪文件（Ready file/znode） - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-08-zookeeper/8.7-jiu-xu-wen-jian-ready-fileznode)

## 11.2 两种构造RSM(复制状态机)的方案

​	这里简单提一下两种构造复制状态机(RSM, replicated state machine)的方案

1. Run all options through Raft/Paxos：通过Raft/Paxos或其他共识算法，完成所有的操作

   事实证明，这种通过Raft/Paxos这类共识算法运行所有操作的方式，在实际应用中并不常见。这并不完全是一种标准的方法（指用于实现复制状态机）。后续会介绍一些其他的设计来实现这一点，比如Spanner。

2. **Configuration service + P/B replication(primary backup replication)**：**更常见的是通过一个配置服务器**（比如zookeeper服务），配置服务器内部可能使用Paxos、Raft、ZAB或其他共识算法框架，配置服务器扮演者coordinator或master的角色（比如GFS的master）。**这里配置服务器除了采用共识算法实现外，还可以运行主备复制(P/B replicaiton)**。

   + 比如GFS虽然只有一个master，但是chunk服务器却有primary和一堆backups，它们有一个用于主备复制的协议。
   + 而VM-FT中，配置服务器是test-and-set的storage服务器，其决定谁是primary，另一方则是backup，primary和back之间通过channel或其他形式同步log等数据，尽管略有延迟。

​	相比第一个方案，第二种方案更常见和通用。

​	**采用配置服务器+主备复制的特点**：

+ **复制服务在状态state维护的成本较低，一般需要维护的数据量很少**
+ **主备复制则主要负责大量的数据复制工作**

​	而直接用共识算法实现复制状态机，往往意味着需要直接在提供服务的server之间来回进行大量的数据复制，检查点的state数据同步等工作，一般实现会更复杂。

---

问题：方案1比起方案2有什么好处吗？

回答：你不需要同时使用它们两个，只需要选择一种。方案1中，你每运行操作，Raft就为你进行同步，所有的东西都集成在单一的组件中（包括主备同步等操作）；而在方案2中，我们拆分出2个组件，一个配置服务中可以包括Raft，而同时我们有主备方案，这分工更加清晰。

问题：那方案2有什么优势吗？如果通过leader达成共识，leader永远不会失败，对吧。

回答：方案2的优势，在随后准备介绍的链式复制中会看到。有一个单独的进程负责配置部分的工作，你不必担心你的主备复制方案。对比的话，GFS的master指定某几个server组成一个特定的replica group。而这里主备协议不用考虑这个问题。

## 11.3 链式复制-概述

> [MIT 6.824 - Chain Replication for Supporting High Throughput and Availability - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/519927237)

​	<u>链式复制，就是上面提到的方案2的主备复制方案(P/B replication)</u>。

​	在论文中，**链式复制(Chian replicaiton)假设系统中存在一个配置服务(Configuration service)作为master，然后链式复制本身有一些属性(properties)**：

1. 读操作/查询操作(read operation / query operation)只涉及一个服务器(reads ops involve 1 server)

2. 具有简单的恢复方案(simple recovery plan)
3. 线性一致性(linearizability)

​	整体来说，这是一个很成熟的设计，已经应用到诸多系统中。

![img](https://pic1.zhimg.com/80/v2-4f029c98e6d963ab6233557dc83fba2c_1440w.webp)

​	在链式复制系统中，首先有一个配置服务(configuration service)记录链式连接的节点信息。

1. client向系统发起write请求，**write写请求总是会发送到链中的head节点服务器**
2. head节点生成log等，通过storage稳定存储更新相关state状态，然后传递操作到下一个节点
3. 下一个节点同样更新自己storage中的state状态，然后向下一个节点传递操作
4. 直到最后一个tail尾节点，修改storage中的状态后，向client回应确认信息。

​	如果你需要更高的可用性（容错性），你可以增加链中的节点数量。

​	**这里tail尾节点就是提交点(commit point)，因为后续读取总是从tail尾节点获取**。任何client发起读请求，都会请求tail节点，而tail节点会立即响应。

+ **写请求总是发送到head节点**

  head修改storage中的state后，向下传递操作，直到tail节点。中间的节点总是更新state，然后向下传递操作。最后由tail节点响应client。

+ **读请求总是发送到tail节点**

  tail节点直接响应client读请求。

​	可以看到的是，这里读写工作负载至少分布在两台服务器上。对于更多的读操作，只需要一台服务器支持，且能够快速回应，这使得我们有很多优化空间（后续介绍）。

​	**而这里称tail尾节点为提交点(commit point)，因为写入发生在尾部的节点，只有在那里写操作可以被读者看见，这也提供了线性一致性**（因为所有写操作都需要经过head到最后tail节点才会生效，且实际应答读和写请求的只有tail节点，只要系统没有崩溃的情况下，client进行写后读操作，一定会观察到最后一次或最近一次写入的结果，但不会读到旧数据）。

​	*假设写请求不需要等到tail回应，而是head直接回应，那会破坏线性一致性吗？假设head收到写请求后，同样向链表往后的节点同步操作信息，但是又马上响应Client，那么之后Client读tail节点可能获取不到刚写入的数据，很明显违背了线性一致性*。

## 11.4 链式复制-崩溃(Crash)

​	有趣的是，链式复制的失败场景是有限的，共3种crash场景：

+ 头节点故障(head fail)
+ 中间节点故障(one of the intermediate server fail)
+ 尾节点故障(tail fail)

---

​	这里假设有S1、S2、S3构成链式复制的3台服务器，其中S1作为head，S3作为tail。简单分析一下系统如何应对3种crash：

1. 头节点故障（处理最简单）

   假设client请求head时，head崩溃了。那么这里S1未向下同步的log记录可以丢弃，因为还没有真正commit。此时，配置服务器得知S1宕机后，会通知S2、S3，宣布S2称为新的head，而S1被抛弃。之后client请求失败后会重试，请求新的head，S2节点。

2. 中间节点故障（处理相对复杂）

   类似的，S2故障后，某一时刻配置服务器通知S1和S3需要组成新的链。并且由于S3可能还没有和S1同步到最新状态（因为有些同步log可能还在S2没有下发，或者本身S2也没有完全同步到S1的l信息），所以还需要额外进行同步流程，将S3同步到和S1一致的状态。

3. 尾节点故障（处理相对简单）

   S3故障后，配置服务器通知S1、S2组成新链，其中S2成为新的tail。client可以从配置服务器知道S2是新tail。其他的流程基本没变动。

​	可以看出来**头节点故障(head fail)**和**尾节点故障(tail fail)**处理相对简单，而**中间节点故障(one of the intermediate server fail)**相对复杂一点。

​	**重要的是，链式复制的失败处理比起Raft要明显简单得多**。简单的原因有二：

1. 整个系统以简单的链表维护
2. 配置相关的工作交给了专门的配置服务

## 11.5 链式复制-新增副本(add replica)

​	正如论文所述，在tail添加新replica最简单，大致流程如下：

1. 假设原tail节点S2下新增S3节点，此时client依旧和原tail节点S2交互
2. S2会持续将同步信息传递给S3，并且记录哪些log已经同步到S3
3. 某时刻S3和S2同步完成，S3通过配置服务告知S2，自己可以成为新的tail
4. 配置服务设置S3为新tail
5. 后续client改成请求tail节点获取读写响应（client可以通过配置服务知道谁是head、tail）

## 11.6 链式复制和Raft在主备复制的比较

​	光从主备复制的角度出发，比较一下链式复制CR(Chain Replication)和Raft。

CR比起Raft的优势：

+ CR拆分请求RPC负载到head和tail

  前面说过CR的head负责接收write请求，而tail节点负责响应wrtie请求和接收并响应read请求。不必像Raft还需要通过leader完成write、read请求

+ head头节点仅发送update一次

  不同于Raft的leader需要向其他所有followers都发送update，CR的head节点只需要向链中的下一个节点发送update。

+ 读操作只涉及tail节点(read ops involve only tail)

  尽管Raft做了read优化，follower就算要处理read请求，还是得向其他leader、followers发送同步log，确定自己是否能够响应read（因为follower可能正好没有最新的数据）。而CR中，只需要tail负责read请求，并且tail一定能同步到最新的write数据（因为write的commit point就是tail节点）。

+ 简单的崩溃恢复机制(simple crash recovery)

  与Raft相比，CR的崩溃处理更简单。

CR比起Raft的劣势：

+ 一个节点故障就需要重新配置(one failure requires reconfiguration)

  因为写入操作需要同步到整个链，写入操作无法确认链中每台机器是否都执行，所以一有fail，就需要重新配置，这意味着中间会有一小段时间的停机。而Raft中只需要majority满足write持久化，就可以继续工作。

## 11.7 链式复制-并行读优化拓展(Extension for read parallelism)

​	由于链式复制中，只需要tail响应read请求，这里可以做一些优化的工作，进一步提高read吞吐量。

​	**基本思路是进行拆分(split)对象，论文中称之为volume，将对象拆分到多个链中(splits object across many chain)**。

​	例如，现在有3个节点S1～S3，我们可以构造3个Chain链：

+ Chain1：S1、S2、S3 (tail)
+ Chain2：S2、S3、S1 (tail)
+ Chian3：S3、S1、S2 (tail)

​	**这里可以通过配置服务进行一些数据分片shard操作。如果数据被均匀write到Chain1～Chain3，那么读操作可以并行地命中不同的shards，均匀分布下，读吞吐量(read throughput)会线性增加，理想情况下这里能得到3倍的吞吐量**。

​	拆分(split)的好处：

1. **得到扩展(scale)，read吞吐量提高(shard越多，chain越多，read吞吐量理论上越高)**

2. **保持线性一致性(linearizability)**

---

问题：在拆分split的场景下，client怎么决定从哪个chain中读取数据？

回答：论文中没有详细说明，或许client可以通过配置服务器得知read请求对应哪个shard，再找出shard对应的chain，然后告知client该请求哪个chain的tail节点。

## 11.8 链式复制-小结

​	前面提到构造复制状态机，有两种方式：

1. 传统的只通过Raft/Paxos/ZAB等共识算法实现

2. 通过配置服务(通过Raft/Paxos/ZAB等共识算法实现)+主备复制实现(这里我们使用链式复制chain replicaiton)

​	可以看到这里方案2更吸引人：

+ 你能够获取可拓展的读性能（链式复制配合配置服务split几个chain，可以通过shard均匀读请求，提高读吞吐）
+ 由于配置服务和主备系统独立，所以如果你有更优秀的共识算法实现的同步机制(synchronization)，你更新配置服务的成本更小

​	因此，人们更偏向于在实际应用中使用方案2。当然也不排除有使用方案1实现复制状态机的，比如你只需要简单的put/get等操作的话。后续会介绍的Spanner就是采用Paxos完成（方案1）。

----

问题：你前面提到Raft使用时，所有的read需要通过majority的同意，我不懂为什么？因为leader不是有所有的committed entries吗？

回答：你问的是另一种策略实现。最原始的，如果你的read都只经过leader处理，并且保证leader总有最新数据，那么leader可以直接处理read。而另一种策略，你希望follower能处理read，那它必须通过majority来确认自己是否拥有最新的数据，否则应该等待同步到最新的数据后，才响应client的read请求，以确保线性一致性。

**问题：我想知道前面"11.5 链式复制-新增副本(add replica)"增加tail时，S3什么时候能成为新的tail，有没有可能S2一直是tail，S3一直没同步完原tail(S2)的数据？**

**回答：论文中有提到一些切换tail的细节，大概可以这么说，S2可以用snapshot快照之类的手段先将目前所有的同步信息都传递给S3，也许S2同步snapshot给S3时，log到100，当S3同步完时，S2又接收了101～110。此时S3可以告诉S2，剩下的101～110你同步给我后，不要当tail了。之后S3通过配置服务告知client自己是新tail，同时告知client自己还有101～110要处理，会等处理完同步信息后再响应client（所以这里会有短暂延迟/阻塞）。**

问题：有没有可能，这里扩展可以采用别的数据结构，例如tree，而不是链表？

回答：读取tree叶子节点，听着可能破坏线性一致性。因为client请求的叶子节点，很可能还没有同步到最新write后的数据。也许你想的方案比我的想象要复杂，树的深度或者链的深度，是由失败时间决定的。如果你通常运行3～5台服务器就满足高可用性的要求了，那你可以比如只选用4台，可能崩溃恢复和日常write会引入一些延迟（这里我没怎么懂后面回答的啥东西，感觉大意就是如果构成系统的机器数本身很少，那链表的深度和树的深度差距不会太大，换言之就是tree结构的性能基于深度可能比链表小，但是这里机器数量本身不多的话，树和链表的读写性能差别也不大）。

**问题：我好奇split拆分多个链后怎么保持强一致性？这里S1、S2、S3都可以作为tail被进行读取。**

**回答：你可以获取不同的分片shard或对象object以获取强一致性，这是分配给chain的。因为分片了，所以你写/读某个对象，它一定是对应具体某一个chain，所以能保证chain的强一致性。**

**追问：拆分chain后，在所有的对象上，我们都有强一致性吗？**

**回答：不，我认为这可能需要更多的机制。比如client读取不同shard的对象1和对象2，你需要额外的机制保证整体顺序和线性一致性。（虽然单个shard对应的chain读写能够保证线性一致性，但整体的需要额外机制。否则比如对象2的写或读在业务上依赖对象1的写或读操作，这里不同chain之间并不能严格保证执行的顺序。）**

问题：当CR(Chian Replication)发生网络分区而不是崩溃，那会发生什么？比如S2可能还存活，但是和配置服务器或者其他什么的存在网络分区。

回答：假设所有配置都有编号，此时如果编号不匹配，那么S3不会接受来自S2的命令。

**问题：有没有可能S3崩溃后，存在Client以为S3还是tail，仍然读取S3？**

**回答：所以论文中提到Client要使用代理Proxy。（即Client可以先通过Proxy访问配置服务，得知最新的tail是什么，然后再请求具体的tial。就算tail崩溃了，下次重试时，Proxy也知道该让Client请求新的tail）**

**问题：重复上面全局所有对象的线性一致性的问题，是不是需要诸如分布式事务一类的机制保证？因为我们希望的是数据a和数据b两个不同shard的数据读写具有线性一致性。（毕竟涉及多个不同对象的操作，我也感觉就是需要分布式事务机制）**

**回答：是的，就是需要分布式事务，这个大话题后面会说。**

**问题：前面zlock提到和go锁不一样，zlock在fail的时候会被看到中间状态是什么意思？**

**回答：因为fail后，zookeeper因为session断开了，会删除EPHEMERAL锁文件，这会导致临界区数据可能被处理/加工到一半就不再被保护（这些就是中间状态），因为zookeeper操作的都是被持久化的文件，所以这些中间状态是可见的。但是你可以通过leader选举后，增加清理这些中间状态的步骤（因为这里说zlock通常可以用于leader election）。而如果获取go锁的goroutine发生fail(崩溃)，表示整个go进程都崩溃了，所以当然没有办法读到中间状态。**

# Lecture12 缓存一致性(Cache Consistency)-Frangipani

