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
