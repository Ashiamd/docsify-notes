# MIT6.824网课学习笔记-02

> [【MIT 6.824】学习笔记 1： MapReduce (qq.com)](https://mp.weixin.qq.com/s/I0PBo_O8sl18O5cgMvQPYA) <= 2021版网络大佬的笔记
>
> [简介 - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/) <= 知乎大佬的gitbook完整笔记(2020版，我这里看的网课是2021版)，推荐也阅读一遍2020版课程的大佬笔记
>
> [6.824](https://pdos.csail.mit.edu/6.824/schedule.html) <= 课程表

# Lecture16 BigData-Spark

> [MIT 6.824：spark - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/516534599) <= 网络博主的笔记

## 16.1 Spark-简述

​	某种程度上，Spark是Hadoop的继任者（毕竟都是批处理框架），通常用于科学计算。Spark很适合需要执行多轮mapreduce的场景，因为Spark将中间结果保存在内存中。

​	这篇论文中定义的RDD已经过时了，RDD被**数据帧(dataframes)**取代。但是<u>数据帧(dataframes)可认为是特定列的RDD(RDD with explicit columns)</u>，RDD的所有好的设计思想同样适用于数据帧(dataframes)。这节课只讨论RDD，并将其等同于数据帧。

+ In-memory computaion：Spark针对内存进行计算

## 16.2 Spark-RDD执行模型(execution model)

> [Spark中的Transformations和Actions介绍_sisi.li8的博客-CSDN博客_spark actions transformations](https://blog.csdn.net/qq_35885488/article/details/102745211) <= 有详细的算子清单描述
>
> 1. 所有的`transformation`都是采用的懒策略，如果只是将`transformation`提交是不会执行计算的，计算只有在`action`被提交的时候才被触发。
> 2. `action`操作：action是得到一个值，或者一个结果（直接将RDD cache到内存中）
>
> [Spark：Stage介绍_简单随风的博客-CSDN博客_spark stage](https://blog.csdn.net/lt326030434/article/details/120073802) <= stage划分讲解，图文并茂
>
> [spark——宽/窄依赖 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/350746998) <= 图文并茂，推荐阅读
>
> spark中， 某些操作会导致RDD中的数据在分区之间进行重新分区。新的分区被创建出来，同时有些分区被分解或合并。所有基于重分区操作而需要移动的数据被成为shuffle。在编写spark作业时，由于此时的计算并不是在同一个executor的内存中完成，而是通过网络在多个executor之间交换数据，因此shuffle操作可能导致严重的性能滞后。shuffle越多，作业执行过程中的stage也越多，这样会影响性能。之后的计划中将会学习spark的调优，所以shuffle过程将会在后面的学习中进行补充。
>
> spark driver会基于两方面的因素来确定stage。这是通过定义RDD的两种依赖类型来完成的，**窄依赖和宽依赖。**
>
> [spark持久化操作 persist(),cache()_donger__chen的博客-CSDN博客_spark persist](https://blog.csdn.net/donger__chen/article/details/86366339)
>
> spark对同一个RDD执行多次算法的默认原理：每次对一个RDD执行一个算子操作时，都会重新从源头处计算一遍。如果某一部分的数据在程序中需要反复使用，这样会增加时间的消耗。
>
> 为了改善这个问题，spark提供了一个数据持久化的操作，我们可通过persist()或cache()将需要反复使用的数据加载在内存或硬盘当中，以备后用。

​	了解RDD编程模型的最好方式就是先了解一些简单用例：

```scala
lines = sparks.textFile("hdfs://...") // 1. 仅创建一个RDD: lines，这里表示RDD存储在HDFS中
errors = lines.filter(_.startsWith("ERROR")) // 2. 创建第二个RDD: errors，lines只读，对其执行filter过滤以"ERROR"开头的行，生成新的RDD
errors.persist() // 3. Spark在内存中保存原始的RDD(errors)，用于后续计算。

// Count errors mentioning MySQL:
errors.filter(_.contains("MySQL")).count() // 4. count指令为action操作，会实际导致计算

// Return the time fields of errors mentioning
// HDFS as an array (assuming time is field
// number 3 in a tab-separated format):
erorrs.filter(_.contains("HDFS")) // 5. 
			.map(_.split('\t')(3)) // 6. 
			.collect() // 7. 
```

1. 这里创建一个RDD对象lines，该RDD存储在HDFS中，例如第100万条记录在HDFS分区1上，第200万条数据在分区2。这个RDD lines代表了这组分区。运行这行时，什么都不会发生，即论文所说的**惰性计算(lazy computations)**，实际计算在后续才进行。
2. 创建第二个RDD对象errors，接收对只读对象lines过滤仅保留开头"ERROR"的数据行。这里同样没有进行计算，只是预先创一个一个**数据流(dataflow)**，或者论文中所说的**计算迭代图(iterative graph of the computation)**。<u>后续如果完整到某个action操作时，整个逻辑流水线式执行，即先创建lines，后过滤lines的数据获取errors，再进行后续的其他RDD操作</u>。
3. 指定RDD对象errors在内存中保存，以便后续计算使用
4. 过滤errors中的数据，只保留带有"MySQL"字眼的记录，使用`count()`这个action操作统计数量（实际导致计算）
5. 过滤errors中的数据，只保留带有"HDFS"字眼的记录
6. 用`\t`切割字符串后保留第3个匹配的切割后字符串部分
7. 最后`collect()`该action操作，汇总结果（实际导致计算）

​	**可以看出来这里和Mapreduce有很大不同。在mapreduce中，每次运算完后数据回到文件中，下一轮计算需要重新从文件中读取数据；而这里Spark使用persist方法避免从磁盘重新读取数据**。

​	**RDD的API支持两类方法：Transformations和Actions，Actions才是实际导致计算发生的操作。**

+ Transformations：负责将RDD转换成别的RDD，**每个RDD都是只读(read-only)或者不可变的(immutable)**。所以你不能修改RDD，只能从现有的RDD生成新的RDD。
+ Actions：实际执行计算。只有执行了Actions后，前面的Transformations涉及的文件操作等逻辑才会实际被执行。

​	假设是使用命令行用Scala编写Spark，那么在4和7执行的时候，用户交互的命令行界面才会返回，因为执行的`count()`和`collect()`都是action操作，会实际触发Spark进行计算。action操作对应的时间点，Spark会collect许多worker，向它们发送jobs，或者通知调度器(scheduler)需要执行job。	

---

**问题：当错误文件从P1（HDFS分区1）中提取出来时，然后另一个错误文件从P2中提取出来，我理解上这是并行发生的？**

**回答：是的，就像mapreduce中有很多worker，worker在每个分区上工作，调度者scheduler会发送job给每个worker，而job是和数据分区关联的task。worker获取一个task后运行。所以你可以在分区之间获得并行性。你也会得到流水线中各个阶段之间的并行性。**

问题：lineage和事务日志(the log of transactions)有什么不同，是否只是操作粒度不同(the granularity of the operations)？

回答：目前我们看到的日志是严格线性的(strictly linear)，lineage也是线性的，但稍后会看到的使用fork的例子，其中一个阶段依赖于多个不同的RDD，这在日志中不具有代表性。它们有一些相似之处，比如你从开始状态开始，所有的操作都是具有确定性的，最后你会得到某种确定性的结束状态，就这点上它们是相似的。

问题：这个filter的例子中，它只需在每个分区上应用filter，但有时，比如我看到transformation也包含join或sort。

回答：是的，join和sort更复杂，后序介绍。

问题：第三行persist是不是我们开始进行计算的时候？

回答：不，这里还没有任何东西会被计算出来，前三行仍然只是在描述Spark程序需要执行的操作。

**问题：前面的代码中，如果不调用`errors.persist`，会发生什么？**

**回答：如果不调用`errors.persist`，那么后续发生的第二次计算，即（序号7. action，collect），Spark需要先重新计算errors（我理解上，指的是需要重新构造lines对象RDD，然后过滤得到errors。即如果没有保存RDD到内存，那么当前RDD进行完一次action计算后，下一次如果代码中用同一个RDD想进行下一个action计算，就需要重新从源头构造当前的RDD）**

问题：对于不调用persist的分区，在mapreduce的情况中，我们把它们存储在中间文件中，但我们仍然将它们存储在本地文件系统中。Spark这里处理数据时，是会产生中间文件存储在磁盘中吗，还是将整个数据流处理保存在内存中？

回答：默认情况下，整个流都在内存中，但有一个例外，稍后会详细讨论。代码序号3.这里`persist`使用另一个flag，应该是reliable。然而集合存储在HDFS，这个称为checkpoint（代码序号2.）。

问题：对于分区，这里代码序号2.对应的HDFS，说有多个分区，这个是Spark临时为每个worker进行的分区划分HDFS数据？

回答：代码序号1.对应的lines这个RDD产生于HDFS。这里说的分区，是HDFS文件直接直接定义的（也就是说原始的HDFS中的文件本来就分区好了，不是后面SPark临时分区的）。但是你也可以执行repetition重新分区的transformation，一般是stage这样做，后续会提到。比如使用hash partition技巧，你还可以定义自己的分区程序，提供一个分区程序对象或抽象。

追问：所以这里代码里HDFS文件分区，之前就已经由HDFS处理了，但如果你后续想Spark再自定义patition流程，也可以再来一次。

回答：是的。

## 16.3 Spark-worker窄依赖容错

​	和mapreduce处理容错的方式基本一致，Spark中的某个worker崩溃时，需要重新执行stage。

​	<u>这里只讨论Spark的worker崩溃的情况。Spark的worker崩溃，意味着丢失内存中的数据，或者说丢失RDD的partition数据，而后续的RDD计算部分可能依赖于丢失的partition。所以，这里需要重新读取(reread)或重新计算(re-compute)这个partition分区</u>。**调度器(scheduler)某时刻发现没有得到worker的应答，然后重新运行对应partition的stage**。

​	<u>和mapreduce函数式编程一样，视频中列举的transformations算子，都是**函数式的**。它们将RDD作为输入，产生另一个RDD作为输出，是完全确定性的(completely deterministic)</u>。所以这里如果发现一个worker崩溃了，scheduler重新安排某个worker生成同样的partition即可。

---

问题：所以就是因为函数式API实现，所以Spark能保证partion重计算后不变？

回答：是的。

## 16.4 Spark-worker宽依赖容错

​	如"16.3 Spark-worker窄依赖容错"所述，Spark的窄依赖容错，和mapreduce的容错处理基本一致。相对更为棘手的是宽依赖的容错处理。	

​	假设有个Spark在指令流中，child-RDD-1的数据来源自parent-RDD-1～parent-RDD-3的结果，假设是由于进行了join等操作，而child-RDD-1往下执行产生child-RDD-2。此时如果产生chid-RDD-2的worker崩溃，为了恢复这个RDD，就需要往源头追溯，重新计算，最后会追溯到需要parent-RDD-1～parent-RDD-3并行进行重新计算。显然，如果还是按照窄依赖的方式进行容错恢复，性能损耗严重。

​	**对于宽依赖容错恢复，解决方案是设置检查点或持久化RDD**。

​	比如在parent-RDD-1～parent-RDD-3取checkpoint检查点存储的partition快照RDD，而不是重新进行并发计算。之后只需要重新计算child-RDD-1～child-RDD-2即可。

---

**问题：如果使用了一般的persist函数，但是论文也提到了一个RELIABLE flag。想知道，使用persist函数和使用RELIABLE flag有什么区别？**

**回答：persist函数，意味着你将RDD保存到内存，后续action计算可以重复使用（默认是一个action触发后，后续其他action使用到同样的RDD用于触发计算时，需要重新计算一遍这个被依赖的RDD，而persist会保存这个RDD在内存中免去重新计算）；checkpoint或者RELIABLE flag意味着，你将整个RDD的副本写入HDFS，而HDFS是持久或稳定的文件存储系统。**

问题：有没有一种方法可以告诉Spark不再持久化某些东西，因为如果你持久化了RDD，而且你做了大量的计算，但是后面的计算不在使用那个RDD，你可能会把它永远留在内存中。

回答：Spark使用了一个通用的策略，如果真的没有空间了，它们可能会将一些RDD放到HDFS或者删除它们，论文对具体的计划有些含糊。当然，当计算结束，用户退出或停止driver，那我想那些RDD肯定从内存中消失了。

## 16.5 Spark-迭代计算PageRank

> [google pagerank_百度百科 (baidu.com)](https://baike.baidu.com/item/google pagerank/2465380?fromtitle=pagerank&fromid=111004&fr=aladdin)
>
> PageRank，网页排名，又称网页级别、Google左侧排名或佩奇排名，是一种由根据[网页](https://baike.baidu.com/item/网页?fromModule=lemma_inlink)之间相互的[超链接](https://baike.baidu.com/item/超链接?fromModule=lemma_inlink)计算的技术，而作为网页排名的要素之一，以[Google](https://baike.baidu.com/item/Google?fromModule=lemma_inlink)公司创办人[拉里·佩奇](https://baike.baidu.com/item/拉里·佩奇?fromModule=lemma_inlink)（Larry Page）之姓来命名。
>
> PageRank通过网络浩瀚的超链接关系来确定一个页面的等级。Google把从A页面到B页面的链接解释为A页面给B页面投票，Google根据投票来源（甚至来源的来源，即链接到A页面的页面）和投票目标的等级来决定新的等级。简单的说，一个高等级的页面可以使其他低等级页面的等级提升。
>
> [PageRank算法详解 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/137561088) <= 推荐阅读
>
> 在实际应用中许多数据都以图（graph）的形式存在，比如，互联网、社交网络都可以看作是一个图。图数据上的机器学习具有理论与应用上的重要意义。 PageRank 算 法是图的链接分析（link analysis）的代表性算法，属于图数据上的无监督学习方法。
>
> PageRank算法最初作为互联网网页重要度的计算方法，1996 年由Page和Brin提出，并用于谷歌搜索引擎的网页排序。事实上，PageRank 可以定义在任意有向图上，后来被应用到社会影响力分析、文本摘要等多个问题。

​	PageRank是一个对网页赋予权重或重要性的算法，（权重）取决于指向网页的超链接的数量。

​	这里代码用例通过Spark实现PageRank算法，如果最后补充`ranks.collect()`，那么计算就会实际在机器集群上进行：

```scala
val links = spark.textFile(...).map(...).persist()
var ranks = // RDD of (URL, rank) pairs
for (i <- 1 to ITERATIONS) {
  // Build an RDD of (targetURL, float) pairs
  // with the contributions sent by each page
  val contribs = links.join(ranks).flatMap {
    (url, (links, rank)) =>
    links.map(dest => (dest, rank/links.size))
  }
  // Sum contributions by URL and get new ranks
  ranks = contribs.reduceByKey((x,y) => x+y)
  .mapValues(sum => a/N + (1-a)*sum)
}
```

​	这里有2个RDD，其中links表示图的连接/边(the connection of the graphs)，即URL之间的指向关系；而ranks表示每个url的排名。（这里省略代码讲解的记录，想仔细分析的可以看原视频）

​	这里links通过`persist()`常驻内存，所以下面ranks在for循环中迭代使用links，不需要重新计算links，直接从内存中读同一份RDD对象links即可。

![img](https://pic3.zhimg.com/80/v2-2afc25a81e3c651a9a02c50801562412_1440w.webp)

​	随着ranks中迭代次数的不断增加，每一次循环(图中的contribs)都是一个stage，当调度器计算新的stage时会将transformation的一部分附加到lineage graph中。代码中，links被保存到内存，下面ranks中每一次循环都会重复使用links。<u>图中links和ranks在每次循环中做join，构成一个stage，这里涉及网络分区通信，论文中提到一些优化手段，可以指定分区RDD，使用hash分区</u>，意味着links和ranks文件，这两个RDD将以相同的方式分区，它们是按key或hash key进行partition的。**论文中提到这里可以通过hash key分区，使得具有相同hash key的数据partition落到同一个机器上，这样即使join直观上是宽依赖，但是可以像窄依赖一样执行**。比如对于U1，对应分区P1，你只需要查看links的P1和ranks的P1，而hash key以相同的方式散列到同一台机器上。<u>scheduler或程序员可以指定这些散列分区，调度器看到join使用相同的散列分区，因此就不需要做宽依赖，不需要像mapreduce那样做一个完整的barrier，而是优化成窄依赖执行</u>。

​	这里如果一个worker崩溃，你可能需要重新执行许多循环或一次迭代，所以这里可以设置比如每10次迭代生成一个checkpoint，避免重新整个完整的计算。

---

问题：所以我们不是每次都需要重新计算links或其他东西，比如我们不持久化的话？

回答：这里唯一持久化的只有links，这里常驻内存，而ranks1、ranks2等都是新的RDD，但你偶尔可能像持久化它们，保存到HDFS中。这样如果你遇到故障，就不需要循环迭代从0开始计算所有东西。

问题：不同的contribs，可以并行计算吗？

回答：在不同的分区上，所以是可以并行计算的。

## 16.6 Spark-小结

+ RDD by functional transformation：RDD是函数式转换创建的
+ group togethr in sort of lineage graph：他们以谱系图的形式聚集在一起
  + reuse：允许重复使用
  + clever optimization：允许调度器进行一些优化组织
+ more expressive than MR：表现力比mapreduce更强

+ in memory：大多数数据在内存中，读写性能好

---

问题：我想论文中提到了自动检查点(automatic checkpoints)，使用关于每次计算所需时间的数据，我不太理解是什么意思。但是，他们将针对什么进行优化？

回答：可以创建一个完整的检查点，这是一种优化。创建检查点是昂贵的，需要时间。但在重新执行时，如果机器出现故障，也要花很多时间。比如，如果你从来不做检查点，那么你要从头开始计算。但如果你定期创建检查点，你不需要重新计算。但如果你频繁创建检查点，你可能把所有时间都花费在了创建检查点上。所以这里有一个优化问题。

问题：所以可能计算检查点，需要非常大的计算？

回答：是的，或者在PageRank的情况中，也许每10次迭代做一次checkpoint。当然如果检查点很小，你可以创建checkpoint稍微频繁点。但在PageRank中checkpoint很大， 每个网页都有一行或一条记录。

问题：关于driver的问题，driver是不是在客户端？

回答：是的。

问题：如果driver崩溃，我们会丢弃整个图，因为这是个应用程序？

回答：是的，但我也不知道具体会发生什么，因为调度器有容错。也许你可以重新连接，但我不太清楚。

问题：关于为什么dependency optimization的问题。你提到他们会进行hash patitioning，这是怎么回事？

回答：这个和Spark无关，是数据库数据常见的分区概念。即假设对dataset1和dataset2使用hash partion，之后相同hash key的links和ranks数据会分布到同一个机器上，而此时对links和ranks进行join，就不需要机器进行网络通信，直接在机器本地进行处理即可。即多个机器处理多个分区，但是每个分区内links和ranks的join只需要本地进行，不需要和其他机器网络通信。

问题：什么时候是宽依赖，什么时候是窄依赖？

回答：明确的父RDD和子RDD是一对一那就是窄依赖；如果是1父RDD对N个子RDD，就是宽依赖。

**问题：worker和stage？**

**回答：每个worker在parition上运行一个stage。所有stage在不同的worker上并行运行。pipeline中每个stage是批处理(batch)。**

# Lecture17 Memcached缓存系统

> [MIT6.824 Facebook的Memcache - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/471267649) <= 网络博主的笔记

​	这是一篇2013年来自Facebook的论文，memcached在网站开发中很常用。这篇论文不目的不是介绍构建系统的新思想/新概念/新方式，更多的是构建系统的实践经验的总结。

​	在这种情况下，可以支持10亿/s的请求量。从论文中可以学到3个lesson：

+ 高性能：只使用市面上现有的组件构建系统，但是整体性能很高。

+ 性能和一致性之间的取舍：设计主要受性能驱动，但同时想要得到某种程度的一致性。这里Facebook的应用不需要真正的线性一致性，比如新闻的消息延迟了一会后才更新并不影响业务。
+ 经验法则：cautionaru tables，论文中列举了一些警示项。

## 17.1 网站演变

1. 单服务端，单DB
2. 多服务端，单DB
3. 多服务端，多DB（分片或其他手段，使DB处理环节也有并行性。这里可能涉及分布式事务，需要2PL+2PC）
4. 多服务端，多DB，添加缓存

​	到了4. 添加缓存时，系统新增了缓存层(cache layer)，每个单独的缓存服务器被称为memcached daemon，整个缓存集群称为memcache。

​	这里的主要挑战包括：

1. 如何保持数据库和缓存之间的一致性

2. 如何保证数据库不会过载

## 17.2 最终一致性(eventual consistency)

​	Facebook这里不追求强一致性/线性一致性，他们追求的是：

+ 顺序写入(write ordering)：写入按照某种一致的整体顺序应用，你不会在时间问题上感到奇怪，这些由数据库完成，对于memcache层不是大问题
+ 可接受读旧数据(reads are behind is ok)：读取短暂落后也没关系。
  + 例外就是客户端进行写后读，即写后要求读到刚更新的数据。(clients read their own writes)

​	*zookeeper也是不同session不保证读到其他session的最新写入，但是同一个session内写后读能读到最新数据。*

## 17.3 缓存失效方案(cache invalidation)

​	Facebook的基本方案即，**缓存失效方案(cache invalidation plan)**。

![img](https://pic2.zhimg.com/80/v2-95c3f42178d2995e6dfbafc36feb538d_1440w.webp)

​	FE前端将数据写入MySQL，此时squeal监听MySQL的事务日志，发现有一个key被修改，向缓存发送失效消息，缓存将会删除指定key的value缓存。此时另一个FE发起read，缓存不命中，然后从MySQL读数据，之后将数据加入缓存。

---

**问题：为什么不在删除缓存后，立即更新缓存呢？**

**回答：那就是所谓的更新方案(update scheme)，原则上也是可行的，但实现会更麻烦点，这里缓存、数据库、客户端之间会需要一些cooperation。比如client1和client2发请求，原本client1发送的x=1应该更早生效，client2发送的x=2应该后生效作为新值，但由于网络问题，导致client2请求先被处理，如果用更新方案，缓存x=2，后续client1请求被处理后，缓存x=1，这里缓存中的x会一直都是旧值。当然，你可以通过修改MySQL内部一些流程，比如要求按序处理client1和client2的请求等等。但是这里系统希望能直接用现成的组件搭建，而不是再进行复杂的修改，所以论文中选择的是失效方案。**

## 17.4 系统优化

​	Facebook为了容灾，在两个地区部署了数据中心：

+ 只有primary数据中心接收数据的写入，backup数据中心只读

+ **就算是backup数据中心发生了写入请求，也会请求primary数据中心**

+ primary数据中心的数据变化，通过squeal进程监听并同步到backup数据中心

  这里primary和backup之间存在一定延迟，不过系统不追求强一致性，所以可以接受。

​	系统获取高性能的手段（按序逐渐增加优化手段）：

1. 分区或分片 (partition or shard data)

   + 大容量 (capacity)：一般对数据分区后，每个分区可以容纳大量数据

   + 并行度 (parallism)：分区后，不同分区数据读取可以并行

2. **复制数据 (replicate data)**

   + <u>热键友好 (hot key)：可以使相同键扩展到不同的memcached服务器（两个数据中心），缓解单点热键问题</u>

   + 需扩容 (capacity)：复制数据本身需要另外的硬件资源，但本身没有扩大存储的数据的容量

3. **构造集群(cluster)**

   + <u>热键友好 (hot key)：同一个数据中心内，再将单缓存服务扩展成缓存集群服务，热键由单实例存储复制到集群内多实例</u>
   + 减少连接 (reduce connection)：避免了incast congestion问题，集群内实例均摊了前端的缓存数据请求TCP连接
   + 网络压力 (reduce network pressure)：很难建立具有双向带宽的网络，这里流量均摊到集群内的实例，整体网络压力承载量大

​	如果系统中有冷键，其将被存储在多个regions，基本上什么都不做。所以Facebook还有一个额外的pool，称为**region pool，应用程序可以决定保存不是很受欢迎的键到regional pool，它们不会在时间上跨所有集群复制，你可以考虑region pool在多个集群之间共享，用于不太受欢迎的键存储。**

---

问题：这里如果backup发起写请求，会写到primary，但是这里backup缓存失效后，因为同步需要时间，backup可能还是会缓存到旧数据？

回答：是的，后面会说怎么处理这个问题。

## 17.5 保护数据库(protecting db)

> [什么是惊群，如何有效避免惊群? - 知乎 (zhihu.com)](https://www.zhihu.com/question/22756773)
>
> 惊群效应（thundering herd）是指多进程（多线程）在同时阻塞等待同一个事件的时候（休眠状态），如果等待的这个事件发生，那么他就会唤醒等待的所有进程（或者线程），但是最终却只能有一个进程（线程）获得这个时间的“控制权”，对该事件进行处理，而其他进程（线程）获取“控制权”失败，只能重新进入休眠状态，这种现象和性能浪费就叫做惊群效应。

​	假设所有的memcache都失效了，db就算分片了也不能够扛下10亿/s的查询请求。

+  新建集群 (new cluster)

  假设新建了一个cluster，预想分担系统50%流量，但是new cluster的memcache还没有数据，如果50%读请求都进入new cluster，会导致流量都直接击中底层db，导致db崩溃。所以这里创建new cluster时，会填充new cluster的memcache，待预热完缓存数据后，再实际切50%流量到new cluster。

+ **惊群问题 (thundering herd problem)**

  某个热键被更新后，缓存失效，多个前端请求同一个热键，得到null后都请求db查询数据，增大了数据库瞬时压力。**这里Facebook采用租约(lease)解决这个问题，即热键失效时，某个前端获得lease，负责更新该缓存，而其他前端则retry请求缓存，而不是直接查询db。**（这里lease用于解决性能问题，后续会提到同样也解决一致性问题）

+ 缓存崩溃 (memcache server failed)

  如果不做处理，缓存崩溃后，会有大量请求直接发送到db，增加db压力。这里系统设置了gutter pool，用于临时处理memcache崩溃的场景。当memcache崩溃后，改成查询gutter pool，若还是没有，就查询数据库然后将数据放到gutter pool中（同样起到缓存作用，但gutter pool只是容错应急用的，非常态使用）。

## 17.6 缓存读写竞争(race)

1. 避免缓存旧值(avoid stale set problem)

   Client1执行`get(key)`，Client2执行`delete(key)`，Client1执行`put(key)`。由于有lease机制，这里C1执行get时获取lease，而C2执行delete时会使得C1的lease失效（因为更新了db，所以C2需要让key缓存失效），后面C1的put会失效(被拒绝)。**利用lease机制，避免了"17.5 保护数据库(protecting db)"提到的惊群问题，也避免了这里旧值填充的问题**。

2. 冷集群预热时缓存其他集群旧值(cold cluster)

   C1更新db值，执行`delete(key)`，C2从cold cluster读数据未命中，从另一个cluster缓存取数据，并执行`put(key)`。这里C2设置的缓存还是旧值。**这里Facebook通过two-second hold-off，推迟两秒机制解决该问题，其规定从cold cluser删除key值后，2s内不能对那个key做任何put操作**。所以这里举例的C2执行的put操作会被拒绝。<u>这种问题只会出现在集群预热阶段，即集群新建后暂时没有数据，会从其他运行中的集群取数据（可能由于primary和backup之间同步延迟问题，导致取到旧缓存值）</u>。因为这问题只出现在预热阶段，所以他们认为2s够用了，这也足够让其他写入传播/同步到cold database。

3. backup写后读不到值(primary/backup)

   C1在backup执行写操作，数据会写到primary的DB，随后backup的缓存失效(执行`delete(key)`)，此时C1直接`get(key)`取到空值，因为primary的数据修改通过squeal同步到backup的缓存需要时间。**这里Facebook通过remote marker解决该问题。当C1执行`delete(key)`后，标记key为"remote"，后续马上执行`get(key)`时，发现key的"remote"标记，于是从远程的primary的缓存读取值，而不是从本地读取。完成`get(key)`后，再移除key的"remote"标记。**

---

问题：即使没有lease invalidation机制，我们仍然会遵从弱一致性，你会获得在过去某个时间发生的顺序写入。但至少这能确保观察到自己的写入，对吗？

回答：lease还能确保你不会回到过去（读旧数据）。

## 17.7 小结

+ 缓存的重要性(caching is vital)：正确的使用缓存，使得论文中的系统能够承载10亿/s的请求
+ 分区和分片(partition and shard)：分区和分片，提高整体存储量和并行度
+ 复制(replicaiton)：均摊热键访问成本
+ 缓存和DB的一致性处理：利于lease、two-second hold off、remote marker等特殊技术手段，保证系统的弱一致性，主要是防止出现长期缓存旧值的问题。

---

问题：远程标记(remote marker)，他们怎么知道这是一个相关的data race。或者他们怎么决定什么时候需要使用远程标记？

回答：论文中没有明确说明，但猜测是为了保证最终一致性(eventual consistency)中的写后读能读取到刚才的最新写入。

问题：他们对get请求使用UDP，对其他请求使用TCP，这是一种业界普遍的做法吗？

回答：一般人们更喜欢用TCP，但TCP开销更大。有些人喜欢在UDP上使用他们自己的类似可靠传输协议的技术，比如QUIC等。

# Lecture18 Fork Consistency SUNDR
