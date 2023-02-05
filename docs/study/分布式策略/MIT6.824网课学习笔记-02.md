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

# Lecture17 缓存一致性-Memcached缓存系统

> [MIT6.824 Facebook的Memcache - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/471267649) <= 网络博主的笔记
>
> [redis探索之缓存一致性_青铜大神的博客-CSDN博客](https://blog.csdn.net/qq_22156459/article/details/125496995)
>
>  一致性是指同一时刻的请求，在缓存中的数据是否与数据库中的数据相同的。
>
> + **强一致性**：数据库更新操作与缓存更新操作是原子性的，缓存与数据库的数据在任何时刻都是一致的，这是最难实现的一致性。
>
> + **弱一致性**：当数据更新后，缓存中的数据可能是更新前的值，也可能是更新后的值，因为这种更新是异步的。
>
> + **最终一致性**：一种特殊的弱一致性，在一定时间后，数据会达到一致的状态。最终一致性是弱一致性的理想状态，也是分布式系统的数据一致性解决方案上比较推崇的。

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

# Lecture18 fork一致性-SUNDR

## 18.1 SUNDR简述

> [拜占庭将军问题_百度百科 (baidu.com)](https://baike.baidu.com/item/拜占庭将军问题/265656?fr=aladdin)
>
> 拜占庭将军问题（Byzantine failures），是由莱斯利·兰伯特提出的点对点通信中的基本问题。含义是在存在消息丢失的不可靠信道上试图通过消息传递的方式达到一致性是不可能的。
>
> [分布式系统入门 | 拜占庭将军问题 - Alan-Yin - 博客园 (cnblogs.com)](https://www.cnblogs.com/alan-yin/p/14901121.html) <= 详细的文章，推荐阅读
>
> **拜占庭故障**（也叫做**交互一致性**，**源一致性**，**错误雪崩**，**拜占庭协议问题**，**拜占庭将领问题**和**拜占庭故障**）是一个计算机系统的状态，特别是**「分布式计算」**系统中，其中的组件可能会故障，并且在组件故障时可能发生故障信息。
>
> 项目开始时，并不清楚总共需要多少台计算机才能保证 n 台故障计算机的阴谋不会“阻挠”正确运行的计算机达成共识。Shostak 表明至少需要 `3 n+ 1`，并设计了一个两轮 `3 n+ 1` 消息传递协议，该协议适用于 n = 1。他的同事 Marshall Pease 将算法推广到任何 n > 0，证明 `3 n+ 1` 是充分必要的。这些结果，连同[Leslie Lamport](https://en.wikipedia.org/wiki/Leslie_Lamport)后来使用数字签名证明 3n 的充分性，发表在开创性的论文—— 《[Reaching Agreement in the Presence of Faults](https://en.wikipedia.org/wiki/Byzantine_fault#cite_note-9)》(在存在故障的情况下达成共识)。

​	该章节主要讨论的主题为**去中心化的系统(decentralized system)**。

​	去中心化，即没有单一的权威机构(single authority)来控制这个系统。前面介绍的系统，或多或少所有的机器和服务在某种程度上合作(cooperate)，且在单一机构处于或任何单一autority的控制下。

​	去中心化的系统更难建立，因为你可能不得不解释**拜占庭失败(byzantine failures)**。在过去的课程中，介绍的分布式协议，都是默认认为每个参与者都遵守协议规则，但在拜占庭问题和拜占庭参与者之间，情况就不一样了，可能有参与者恶意捏造虚假数据进行传输。这类话题，位于分布式系统(distributed system)和安全性(security)的交汇点(intersection)上。后面即将介绍的论文，主要就和密码学或安全理念有关，比如签名(signing)和哈希(hashing)。

​	<u>可能有人好奇，SUNDR有被普遍使用吗？实际上没有任何系统直接实现SUNDR，或者直接基于SUNDR</u>。但是，**SUNDR提出了一些强大的技术或设计思想，特别是signed log，尽管是概念设计。同样的想法出现在很多其他去中心化的系统中**。从git到比特币(Bitcoin)，或其他类型的**加密帐本(cryptographic ledger)**，都能看到类似的设计思想。有一个系统直接收到SUNDR的影响，即Keybase，其被Zoom收购。

## 18.2 SUNDR论文背景

​	论文动机，是为了设置网络文件系统(network file system)。回想之前读过的Frangipani论文，其主题也是实现一致的网络文件系统。不同的是，SUNDR这里认为网络文件系统可以是拜占庭式的，攻击者可能通过某些手段得以操控网络文件系统。

​	**SUNDR实际上不是在文件服务器上维护整个文件系统，文件系统尽可能简单，类似Frangipani论文中的Pedal，它几乎就像一个块设备，而客户端真正实现了文件系统**。所以在SUNDR中，不是client向serevr发送、创建文件，而是从block服务器发送数据块和读取数据块。

​	SUNDR就整体文件操作流程，和Frangipani论文类似，文件系统在client，server负责硬件磁盘块(blocks)资源管理。**两者最大的在于，Frangipani的文件系统Pedal和所有client之间是完全信任的，而SUNDR中client不受信任，文件系统本身也不可信**。

​	这篇论文主要集中在一些安全属性的讨论，重点是**完整性属性(integrity properties)**。与保密性(confidentiality)相反，保密性关于保护数据避免泄漏；而完整性只是为了确保系统结构是正确的，检测数据非法修改行为，无论数据本身是否公开。

​	比如，2003年Debian Linux遭受过恶意开发者在代码嵌入trapdoor的问题，而Debian Linux被部署到众多服务器上使用。当时Debian Linux为了处理该问题，开发冻结了几天，试图找出源代码库中哪些部分仍然是正确的，以及哪些部分被攻击者修改了。这类攻击会周期性地发生，Ubuntu在2018年或2019年也有出现类似的问题。

## 18.3 安全场景举例

> [树根|主体_比特币技术原理----区块链的本质 (cha138.com)](https://it.cha138.com/javascript/show-84427.html)

​	zoobar是一个虚拟银行类型的应用程序，系统的注册用户可以互相转移zoobar（充当货币的作用）。

+ A：某有权限修改`auth.py`的程序员，主要负责权限校验开发
+ B：某有权限修改`bank.py`的程序员，主要负责银行交易等行为的开发
+ C：某拷贝git项目，部署zoobar代码，后续由于代码有后门等原因被攻击的大冤种

​	这里明显存在安全隐患：

1. 某个程序员S偷偷修改`auth.py`，导致C部署项目后，项目内缺失身份认证的一些关键步骤，存在安全漏洞。
2. 某程序员S针对C修改了`bank.py`，删除了里面关于身份校验的逻辑，导致银行业务内不在进行身份认证等校验。因为银行业务流程本身没有改动，所以相对难察觉。

---

​	接下来谈谈一些常见的处理手段，来尝试避免上面的安全隐患。

​	修改文件后，对文件使用非对称加密，即public key加密。这样程序员S无法获取到程序员A的私钥，无法完成程序代码`auth.py`的修改，因为S没有密钥，签名自己的代码修改后上传会被代码托管服务器拒绝。但这里S仍有方法攻击，比如发送程序员A以前修改过的`auth.py`文件，这个文件以前由A签名，所以能成功上传，也许旧代码存在安全漏洞。另外，S可以声明删除`auth.py`文件，C无法知道这是否正常，这可能需要对系统所有文件做一个捆绑，以某种方式决定文件系统最新版本是什么，来避免这个问题。

## 18.4 SUNDR设计思想

​	从前面的`18.3 安全场景举例`，可想到网络文件系统存在很多攻击的方式，并且对应的修复方法并不容易或很难只通过某一种手段完全规避攻击。而SUNDR论文就是为了试图解决这类问题。

​	SUNDR论文提到了一个设计思想，即**对操作的日志签名(signed log of operations)**，尽管论文本身没有直接实现这个大的想法。<u>可以理解为操作日志的增强版本，在日志条目有签名的情况下，涵盖了条目及之前的条目</u>。

​	我们在前面的所有**分布式系统(distributed systems)**和**故障恢复协议(recovery protocols)**中看到，日志是一个非常强大的设计思想，经常用于保证系统的正确性，同样也适用于拜占庭背景下的系统正确性保障。

​	这里假象一个log结构如下，假设写操作为mod，读操作为fetch：

| mod<br/>auth.py<br/>signed A | mod<br/>bank.py<br/>signed B | fetch<br/>auth.py<br/>signed C | Fetch<br/>bank.py<br/>signed C |
| ---------------------------- | ---------------------------- | ------------------------------ | ------------------------------ |

​	**这里每次读或写操作都进行log记录，并且每次操作完后，需要对当前操作记录log以及之前的log进行签名**。即`signed B`这里签名的log包括当前B对`bank.py`的mod操作，也包含之前A对`auth.py`的mod操作。这样当client C接收到B的log时，就不可能丢失A的日志条目，因为一旦丢失，签名就对应不上。通过这种机制，服务器现在更难有选择地删除日志条目。

​	此时如果有一个用户下载项目代码，其参与维护，经过以下流程：

1. 校验签名(check signatures)：项目中所有的操作log都有签名，只要对不上，就知道项目有其他恶意开发者篡改代码。
2. **检查最后一个日志条目(check its own last entry)**：**这是为了保证客户端不会被服务器回滚，所以服务器总是检查最后一个条目，确保最新的log修改条目没有被恶意攻击者回滚**。比如A新增了一个mod操作log，但S通过某些手段回滚log记录，使A的修改在log中不存在。

3. 构造文件系统(construct FS)：它知道文件系统系统没有被回滚到旧版本，所以可以应用所有现存的修改，基本上就是在client上构建文件系统树
4. 新增操作log条目并签名(add entry log + sign)：对项目进行修改，在操作log中新增条目，并对其进行签名（和前面说的一样，需要对当前log条目以及之前的log条目进行签名）
5. 上传操作log到文件系统服务器(upload log to FS)：本地记录log后，向服务器上传log。

​	这个协议在现实中是未完成的，但提供了一些设计概念，让我们思考最终怎么成功在拜占庭服务器的上下文中实现安全。

## 18.5 SUNDR为什么记录fetch

​	可能有人会有疑问，修改操作记录log中很正常，为什么fetch读取操作也要记录到log中？

​	假设fetch操作不需要记录到log中，那么前面"18.4 SUNDR设计思想"的log可以看作如下：

| mod<br/>auth.py<br/>signed A | mod<br/>bank.py<br/>signed B |      |      |
| ---------------------------- | ---------------------------- | ---- | ---- |

​	C一开始从服务器获取前面的log（比如到A的操作log为止），校验无问题后，安装`auth.py`；后面B的操作log也发送过来，C发现B对`bank.py`进行了修改，校验log后无异常，于是安装`bank.py`。<u>问题在于这里A和B之间的修改可能对应的系统版本不一致</u>。比如`auth.py`对应系统版本1，`bank.py`对应系统版本2，两者同时运行会有一些异常。

​	而如果log需要记录fetch，那么故事变更如下：

1. C fetch `auth.py`，服务器发送过来prefix，即A和B的操作记录
2. C add fetch  to log，upload log，C获取log后，添加fetch操作记录，上传log到服务器
3. C fetch `bank.py`，服务器发送过来prefix，包括A和B操作，以及C刚才的fetch操作记录。此时如果log中没有刚才新增的fetch操作，C就会拒绝该日志

​	*（这里稍微有点懵，感觉大意就是如果没有把fetch加入log的话，那么当前client只会一味地接受新log，新log产生的影响可能对上次读log得到的数据有影响；而如果把fetch加入log，那么client就能感知到新log插入的位置在自己fetch之前还是之后，避免新log对已读出的数据有影响却无感知的问题。）*

---

问题：当前例子中是什么阻止服务器将fetch放在日志的正确位置？

回答：每个日志条目都包含之前的所有条目（指签名），服务器不能在fetch A和B之前的prefix之后修改切片。

问题：假设它只想发送A的修改，它知道A的修改和之前所有内容的hash，然后它可以把fetch C插入到那里，因为它知道，这是一个hash。

回答：是的，然后他们就不能发送B的修改。因为B的修改直接在A之后，所以它也不能切分A和B。

## 18.6 fork一致性

​	目前看到的是，服务器不能真正操作日志，它只能发送prefixes或hide parts，他可以将前缀发送给client，但它自身不能修改log。所以log被时刻修改时，不同客户端可能看到不同版本的log。这就是fork一致性的含义。

​	fork一致性场景举例，下面描述A、B、S三方的log记录：

| server S | a    |      |      |
| -------- | ---- | ---- | ---- |
| client A | a    | b    | c    |
| client B | a    | d    | e    |

​	<u>A和B从S fetch log后，自己在本地修改，后续并发地upload log到S。这里可以看出来A和B都拥有同样的a日志前缀记录，但是后面各自的本地log不一样，S接收到A和B的log后，作为服务端不能擅自修改log，所以根据fork一致性，会同时保留A和B的两条log修改链路，而不是直接合并到原有log上</u>。

## 18.7 如何检测fork

​	这篇论文提到了两个检测fork的方案：

1. 带外通信(out-of-band communication)：如果A和B互相访问，询问对方最后一条记录是什么，得到不同的答案，他们就意识到双方造成了log分叉。如果没有分叉，一方的log应该是另一方的前缀。所以<u>客户端可以定期交换最新的日志条目，确保不产生log分叉</u>。
2. **三方受信时间戳机器(timestamp box)**：引入某种受信任的机器，比如时间戳机器。它将时间戳添加到log中，每个客户端知道它是一个文件。在文件系统中，包含当前时间，时间戳机器每隔几秒钟就更新一次文件，客户端读取该文件。他们知道每隔几秒钟会有一次新修改。这里timestamp box相当于就是fork

​	第二种方案也就是SUNDR论文中提出的。Bitcoin比特币就采用类似方案二的方式，以某种方式达成读取共识，决定哪个fork能继续执行。

## 18.8 用户粒度文件系统快照(i-handle)和版本向量(version vector)

​	为处理fork一致性问题（即让用户在不同fork上做取舍，决定使用哪个fork），SUNDR采用用户文件系统快照和版本向量（版本向量包含快照i-handle和用户修改计数）。

​	服务器维护log，而其他人维护snapshot快照。**实际上SUNDR所做的，不是字面上创建快照，而是维护着文件系统的快照视图，并针对每个用户。文件系统按照用户进行分片，每个用户都有属于自己的视图快照。这里有一些协议确保不同的快照和不同的用户是一致的**。

​	**SUNDR中，有一种叫做用户i-handle的东西，用户i-handle唯一标识文件系统中的快照**。<u>i-handle对应一个i-table的加密hash表，其记录系统中所有inode的hash值。当某个文件被修改时，对应的inode节点hash值会更新，往上追溯，i-table、i-handle都会更新。这提供给用户一个完整的文件系统快照</u>。

​	而为了处理用户之间的一致性，SUNDR提出了**版本向量(version vector)**的概念。<u>每个版本向量对应一个用户的i-handle，比如A在修改`auth.py`后有一个i-handle，同时版本向量维护所有用户的当前i-handle的修改计数。比如对于A的i-handle有记录`{A:1; B:0; C:0}`表示只有A修改了A自身的i-handle一次，其他人没进行过修改</u>。

​	**用户对于整个文件系统的快照i-handle，以及每个用户(包括当前用户)对当前i-handle的修改次数，<u>作为一个版本向量(version vector)整体进行签名</u>**。

```pseudocode
// 1. A修改auth.py后生成一个信的i-handle,假设叫 i-handle-A，对应的修改计数只有A进行过1次，所以A的版本向量如下
VS-A: i-handle-A , {A:1; B:0; C:0} // 之后对VS-A sign签名

// 2. B修改之前获取所有的变更，然后新建VS-B版本变量，其修改了文件bank.py，对应版本向量如下
VS-B: i-handle-B , {A:1; B:1; C:0} // 之后对VS-B sign签名
// 这里A修改计数为1，因为B获取了所有的log记录后，才进行修改，而之前A对文件系统中的auth.py进行过1次修改

// 3. C下载所有用户的版本向量，取版本最新的1个，这里最新的是VS-B。因为VS-B包含A的所有操作，以及后续B的所有操作
此时C通过对比VS-A、VS-B，发现VS-B最新，从VS-B版本向量对应的文件系统快照，获取auth.py和bank.py
```

## 18.9 小结

> [梅克尔树_百度百科 (baidu.com)](https://baike.baidu.com/item/梅克尔树/22456281?fr=aladdin)
>
> 梅克尔树（Merkle trees）是[区块链](https://baike.baidu.com/item/区块链/13465666?fromModule=lemma_inlink)的基本组成部分。虽说从理论上来讲，没有梅克尔树的区块链当然也是可能的，只需创建直接包含每一笔交易的巨大区块头（block header）就可以实现，但这样做无疑会带来可扩展性方面的挑战，从长远发展来看，可能最后将只有那些最强大的计算机，才可以运行这些无需受信的区块链。 正是因为有了梅克尔树，以太坊节点才可以建立运行在所有的计算机、笔记本、[智能手机](https://baike.baidu.com/item/智能手机/94396?fromModule=lemma_inlink)，甚至是那些由Slock.it生产的物联网设备之上。

+ 拜占庭参与者问题(Byzantine participant)：拜占庭参与者问题，需要在去中心化系统中处理。因为没有单一的机构作为信任的来源/担保。

+ 签名式日志(signed log)：签名式日志，是对付恶意服务器的一个强大工具，后续的课程会讨论这类日志如何在比特币中使用。特别是fork一致性，fork是如何被创建的，在比特币案例中是如何解决的。

​	这些就是去中心化系统主要讨论的知识点/技术点。

----

问题：SUNDR使用B+树还是什么数据结构？Merkle tree和这些数据结构有什么区别

回答：SUNDR使用Merkle，因为提出这个结构的人就是这个名字，所以称这种树结构为Merkle tree。

问题：验证签名时，如果log有100个条目，需要计算100次hash吗？

回答：通常只计算一次，也就是最后一个新增的条目。虽然理论上每个log entry的签名需要重新计算，但实际应用中，一般用户本地也会保存之前的log的hash值，只需要计算后面新增的hash值进行比对就好了。（提问者吐槽这样效率低，但是就是这么做）

问题：所以hash就像一条Merkle chain？

回答：是的。同样的想法。

**问题：如果改变文件系统中某几个block，是所有block的hash都需要重新计算吗？然后再追溯到inode、i-table、i-handle？**

**回答：只需要计算被修改的几个block的hash，然后往前追溯相关的inode（block对应的inode）、i-table（对应的几个inode）、i-handle（对应的i-table）需要重新计算。论文有提到一种优化可以让这一过程计算更快，但hash通常很快，而签名才是成本更高的操作。**

问题：我们使用版本向量来确保系统不会返回/回滚到旧状态，那为什么系统不直接返回旧状态和旧版本向量，如果它保留第二份复制的话？

回答：版本向量只有fork一致性，SUNDR fork一致性，没有保证更多其他别的。

**问题：fork一致性，需要用到时间戳吗？**

**回答：fork一致性，我的意思是，服务器可以在任何时间点fork日志，为他们可以将日志合并回一起提供一致的视图。**（如何合并决策权还是在client端，sevrer端只是负责存储各种fork情况的log发展分支）

问题：所以这里我们能做到的最好的情况就是保持fork一致性，它允许fork，但我们可以检查出fork。

回答：是的。

追问：这里我们能检测到fork，那么能得到比fork一致性更强的东西吗？

回答：我们可以选择一个fork并继续。

追问：但是SUNDR没有办法做到这一点。

回答：是的。（我理解上就是SUNDR的server不直接决定哪个fork能继续，而是交给client自己决定用哪个fork继续，sevrer只负责存储多个用户版本向量，换言之server保存多个fork，但不对外声明哪个fork是所谓的正统/主干）

**问题：SUNDR现在提出了一些(检测fork的)方法吗？**

**回答：检测fork的方法，提出时间戳机器来检测fork。**

问题：时间戳机器只是一个附加条目的服务器？

回答：是的，并且是可信的，不在对手的控制下。

# Lecture19 p2p比特币(peer-to-peer bitcoin)

1:24
