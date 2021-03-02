# 分布式大数据框架Hadoop-学习笔记

> [分布式大数据框架Hadoop](https://www.ipieuvre.com/course/311)	<=	下面是根据该课程学习的笔记
>
> 启动后访问本地的[yarn web](http://localhost:8088)和[hdfs web](localhost:9870)

# 1. 大数据的发展

## 1.1 信息浪潮

+ 第一次浪潮：信息处理
+ 第二次浪潮：信息传输
+ 第三次浪潮：信息爆炸

举例：技术升级=>图片像素不断提高=>存储容量需求增大

## 1.2 大数据3V

单体数据 X 用户数 = 大数据

+ 数据量大Volume
+ 数据种类繁多Variety
+ 处理速度快Velocity

大数据不仅仅是数据的"大量化"，而是包含"快速化"、"多样化"和"价值化"等多重属性

大数据三个特点：3V（大、杂、快）

# 2. Hadoop理论概述

## 2.1 版本演变

+ 1.0：HDFS、MapReduce
+ 2.0：HDFS、**YARN**、MapReduce、Others

YARN（cluster resource management）

## 2.2 简介

+ Hadoop是 Apache软件基金会旗下的一个开源分布式计算平台,为用户提供了系统底层细节透明的分布式基础架构
+ 使用Java开发，具有跨平台性，可以部署在廉价的计算机集群中
+ 分布式环境下提供了海量数据的处理能力
+ 生态圈广

## 2.3 组成

+ 分布式文件系统HDFS（Hadoop Distributed File System）
+ 分布式计算MapReduce

## 2.4 生态系统架构

> [HIVE与PIG对比](https://blog.csdn.net/eversliver/article/details/81107160)
>
> [日志收集组件flume和logstash对比](https://www.jianshu.com/p/f0b25ce6dd17)
>
> [HADOOP生态圈介绍](https://www.cnblogs.com/hanzhi/articles/8969109.html)	<=	详细介绍的文章，下面图的出处。

+ HDFS（分布式存储系统）
+ MapReduce（分布式计算框架）
+ Zookeeper（分布式协调服务）
+ Hbase（分布式数据库）
+ Flume（日志收集）
+ Sqoop（数据库TEL工具）
+ Hive（数据仓库）
+ Pig（工作流引擎）
+ Mathout（数据挖掘库）
+ Oozie（作业流调度系统）
+ Ambari（安装部署工具）

![image](https://images2015.cnblogs.com/blog/204677/201601/204677-20160105160808168-1696883282.png)

# 3. Hadoop伪分布式安装

1. 本地模式：适用于开发MapReduce程序
2. 伪分布模式：Hadoop只有一个节点，HDFS块复制限制为单个副本，环境部署上单机，但是多进程执行，程序执行逻辑类似完全分布模式
3. 完全分布模式：N台主机组成一个Hadoop集群，主从节点分离

# 4. Hadoop伪分布式安装-实操

> 具体实验步骤看章鱼大数据平台的步骤就好了

## 4.1 相关知识

​	Hadoop由Apache基金会开发的分布式系统基础架构，是利用集群对大量数据进行分布式处理和存储的软件框架。用户可以轻松地在Hadoop集群上开发和运行处理海量数据的应用程序。Hadoop有高可靠，高扩展，高效性，高容错等优点。Hadoop 框架最核心的设计就是HDFS和MapReduce。HDFS为海量的数据提供了存储，MapReduce为海量的数据提供了计算。此外，Hadoop还包括了Hive，Hbase，ZooKeeper，Pig，Avro，Sqoop，Flume，Mahout等项目。

​	Hadoop的运行模式分为3种：本地运行模式，伪分布运行模式，完全分布运行模式。

1. 本地模式（local mode）

   这种运行模式在一台单机上运行，没有HDFS分布式文件系统，而是直接读写本地操作系统中的文件系统。在本地运行模式（local mode）中不存在守护进程，所有进程都运行在一个JVM上。单机模式适用于开发阶段运行MapReduce程序，这也是最少使用的一个模式。

2. 伪分布模式

   这种运行模式是在单台服务器上模拟Hadoop的完全分布模式，单机上的分布式并不是真正的分布式，而是使用线程模拟的分布式。在这个模式中，所有守护进程(NameNode，DataNode，ResourceManager，NodeManager，SecondaryNameNode)都在同一台机器上运行。因为伪分布运行模式的Hadoop集群只有一个节点，所以HDFS中的块复制将限制为单个副本，其secondary-master和slave也都将运行于本地主机。此种模式除了并非真正意义的分布式之外，其程序执行逻辑完全类似于完全分布式，因此，常用于开发人员测试程序的执行。本次实验就是在一台服务器上进行伪分布运行模式的搭建。

3. 完全分布模式

   这种模式通常被用于生产环境，使用N台主机组成一个Hadoop集群，Hadoop守护进程运行在每台主机之上。这里会存在Namenode运行的主机，Datanode运行的主机，以及SecondaryNameNode运行的主机。在完全分布式环境下，主节点和从节点会分开。

# 5. Hadoop开发插件安装-实操

> 具体实验步骤看章鱼大数据平台的步骤就好了

# 6. Hadoop常用命令

## 6.1 格式

+ HDFS提供了Shell的操作接口

+ 文件操作命令与Linux相似

+ 格式为：hadoop fs -<命令> <目标>

  ​		如：hadoop fs -ls /user

## 6.2 hdfs常用指令

1. 查看hdfs系统版本

   ```shell
   hdfs version
   ```

2. 查看hdfs系统状态

   ```shell
   hdfs dfsadmin -report
   ```

3. 查看目录及文件

   ```shell
   hadoop fs -ls /
   ```

4. 创建及删除目录

   ```shell
   hadoop fs -mkdir /input
   hadoop fs -rm -r /input
   ```

5. 创建文件<small>(注意是touchz)</small>

   ```shell
   hadoop fs -touchz test.txt
   ```

6. 上传及下载文件

   ```shell
   hadoop fs -put test.txt /input
   hadoop fs -get /input/test.txt /data
   ```

6. 查看文件内容

   ```shell
   hadoop fs -cat /input/test.txt
   ```

7. 当在Hadoop中设置了回收站功能时，删除的文件会保留在回收站中，可以使用expunge方法清空回收站

   ```shell
   hadoop fs -expunge
   ```

8. 进入/退出Hadoop安全模式

   + ```shell
     hdfs dfsadmin -safemode enter
     ```

   + ```shell
     hdfs dfsadmin -safemode leave
     ```

9. 启动/关闭hadoop

   + ```shell
     hadoop安装目录/sbin/start-all.sh
     ```

   + ```shell
     hadoop安装目录/sbin/stop-all.sh
     ```

10. 

# 7. Hadoop Shell基本操作-实操

## 7.1 相关知识

+ 调用文件系统(FS)Shell命令应使用`hadoop fs <args>`的形式。 

+ 所有的的FS shell命令使用URI路径作为参数。

+ URI格式是scheme://authority/path。
+ 对HDFS文件系统，scheme是hdfs，对本地文件系统，scheme是file。其中scheme和authority参数都是可选的，如果未加指定，就会使用配置中指定的默认scheme。
+ 一个HDFS文件或目录比如/parent/child可以表示成hdfs://namenode:namenodeport/parent/child，或者更简单的/parent/child（假设你配置文件中的默认值是namenode:namenodeport）。
+ 大多数FS Shell命令的行为和对应的Unix Shell命令类似，出错信息会输出到stderr，其他信息输出到stdout。

## 7.2 细节知识

+ 在分布式文件系统启动的时候，开始的时候会有安全模式，当分布式文件系统处于安全模式的情况下，文件系统中的内容不允许修改也不允许删除，直到安全模式结束。
+ 安全模式主要是为了系统启动的时候检查各个DataNode上数据块的有效性，同时根据策略必要的复制或者删除部分数据块。
+ 运行期通过命令也可以进入安全模式。在实践过程中，系统启动的时候去修改和删除文件也会有安全模式不允许修改的出错提示，只需要等待一会儿即可。

# 8. Hadoop基础学习--习题

# 9. HDFS开发

## 9.1 HDFS理论讲解

### 9.1.1 Hadoop基础概念

**Hadoop核心**

1. HDFS分布式存储系统
2. MapReduce分布式运算框架

**集群和分布式概念**

+ 集群：集群就是逻辑上处理同一任务的机器集合，可以属于同一机房，也可以属于不同的机房
+ 分布式：分布式文件系统把文件分布存储到多个计算机节点上，成千上万的额计算机节点构成计算机集群

**分布式文件系统的结构**

​	分布式文件系统在物理结构上是由计算机集群中的多个节点构成的，这些节点分为两类，一类叫"主节点"（Master Node）或者也被称为"名称节点"（NameNode），另一类叫"从节点"（Slave Node）或者也被称为"数据节点"（DataNode）

### 9.1.2 HDFS基本概念

+ 块（Block）
+ 名称节点（NameNode）
+ 数据节点（DataNode）

#### 1. 块

HDFS的文件被分成块进行存储，块是文件存储处理的逻辑单元

+ 支持大规模文件存储：大规模文件可拆分若干块，不同文件块可分发到不同的节点上
+ 简化系统设计：简化了存储管理、方便元数据的管理
+ **适合数据备份**：每个文件块都可以冗余存储到多个节点上，大大提高了系统的容错性和可用性

#### 2. 节点

+ NameNode
  + 存储元数据
  + 元数据保存在**内存**中
  + **保存文件，block，datanode之间的映射关系**
+ DataNode
  + 存储文件内容
  + 文件内容保存在**磁盘**
  + **维护了block id到datanode本地文件的映射关系**

##### NameNode

​	NameNode：负责管理分布式文件系统的命名空间（NameSpace），保存了两个核心的数据结构，即FsImage和EditLog。

+ **FsImage用于维护文件系统树以及文件树中所有的文件和文件夹的元数据**
+ **EditLog操作日志文件记录了所有针对文件的创建、删除、重命名等操作**

​	**名称节点记录了每个文件中各个块所在的数据节点的位置信息**。

---

​	名称节点运行期间EditLog不断变大的问题：

​	运行时，所有对HDFS的更新操作，都会记录懂啊EditLog，导致EditLog不断变大。当重启HDFS时，FsImage的所有内容会首先加载到内存中，之后再执行EditLog。由于EditLog十分庞大，会导致整个重启过程十分缓慢。

​	解决方案：SecondaryNameNode第二名称节点

##### SecondaryNameNode

> [Hadoop学习之SecondaryNameNode](https://www.cnblogs.com/zlingh/p/3986786.html)

​	SecondaryNameNode第二名称节点是HDFS架构中的一个组成部分，用来**保存名称节点对HDFS元数据信息的备份，并减少名称节点重启的时间**。

​	SecondaryNameNode一般单独运行在一台机器上。

工作流程如下：

- SecondaryNameNode节点通知NameNode节点生成新的日志文件，以后的日志都写到新的日志文件中。
- SecondaryNameNode节点用http get从NameNode节点获得fsimage文件及旧的日志文件。
- SecondaryNameNode节点将fsimage文件加载到内存中，并执行日志文件中的操作，然后生成新的fsimage文件。
- SecondaryNameNode节点将新的fsimage文件用http post传回NameNode节点上。
- NameNode节点可以将旧的fsimage文件及旧的日志文件，换为新的fsimage文件和新的日志文件(第一步生成的)，然后更新fstime文件，写入此次checkpoint的时间。
- 这样NameNode节点中的fsimage文件保存了最新的checkpoint的元数据信息，日志文件也重新开始，不会变的很大了。

流程图如下所示：

![wKiom1OSgRaDTnQyAAHtiGS9Pvg733.jpg](http://s3.51cto.com/wyfs02/M00/2D/0F/wKiom1OSgRaDTnQyAAHtiGS9Pvg733.jpg)

##### DataNode

DataNode是HDFS的工作节点，存放数据块

数据节点（DataNode）

+ 数据节点是分布式文件系统HDFS的工作节点，负责数据的存储和读取，会根据客户端或者是名称节点的调度来进行**数据的存储和检索**，并且**向名称节点定期发送自己所存储的块的列表**
+ 每个数据节点中的数据会被保存在各自节点的本地Linux文件系统中

### 9.1.2 HDFS体系结构

> [Hadoop分布式文件系统：架构和设计](https://hadoop.apache.org/docs/r1.0.4/cn/hdfs_design.html)

HDFS采用master/slave架构。一个HDFS集群是由一个Namenode和一定数目的Datanodes组成。Namenode是一个中心服务器，负责管理文件系统的名字空间(namespace)以及客户端对文件的访问。集群中的Datanode一般是一个节点一个，负责管理它所在节点上的存储。

HDFS暴露了文件系统的名字空间，用户能够以文件的形式在上面存储数据。从内部看，一个文件其实被分成一个或多个数据块，这些块存储在一组Datanode上。

+ Namenode执行文件系统的名字空间操作，比如打开、关闭、重命名文件或目录。它也负责确定数据块到具体Datanode节点的映射。

+ Datanode负责处理文件系统客户端的读写请求。在Namenode的统一调度下进行数据块的创建、删除和复制。

![HDFS 架构](https://hadoop.apache.org/docs/r1.0.4/cn/images/hdfsarchitecture.gif)

## 9.2 HDFS数据处理原理

**HDFS要实现的目标**

+ 兼容廉价的硬件设备
+ 流数据读写
+ 大数据集
+ 简单的文件模型
+ 强大的跨平台兼容性

**HDFS局限性**

+ 不适合低延迟数据访问
+ 无法高效存储大量小文件
+ 不支持多用户写入及任意修改文件

### 9.2.1 HDFS数据处理

#### 1. HDFS存储原理：冗余数据保存

> [Hadoop分布式文件系统：架构和设计](https://hadoop.apache.org/docs/r1.0.4/cn/hdfs_design.html)

​	作为一个分布式文件系统，为了保证系统的容错性和可用性，HDFS采用了多副本方式对数据进行冗余存储，**通常一个数据块的多个副本会被分布到不同的数据节点上**。

+ 加快数据传输速度
+ 容易检查数据错误
+ **保证数据可靠性**

​	Namenode全权管理数据块的复制，它周期性地从集群中的每个Datanode接收心跳信号和块状态报告(Blockreport)。接收到心跳信号意味着该Datanode节点工作正常。块状态报告包含了一个该Datanode上所有数据块的列表。

![HDFS Datanodes](https://hadoop.apache.org/docs/r1.0.4/cn/images/hdfsdatanodes.gif)

#### 2. HDFS数据存放规则

+ 第一个副本：放置在**上传文件的数据节点**；如果是集群外提交，则随机挑选一台磁盘不太满、CPU不太忙的节点
+ 第二个副本：放置在与第一个副本**不同的机架的节点**上
+ 第三个副本：与第一个副本**相同机架的其他节点**上
+ 更多副本：随机节点

> [Hadoop分布式文件系统：架构和设计](https://hadoop.apache.org/docs/r1.0.4/cn/hdfs_design.html)
>
> 大型HDFS实例一般运行在跨越多个机架的计算机组成的集群上，不同机架上的两台机器之间的通讯需要经过交换机。在大多数情况下，同一个机架内的两台机器间的带宽会比不同机架的两台机器间的带宽大。
>
> 通过一个[机架感知](https://hadoop.apache.org/docs/r1.0.4/cn/cluster_setup.html#Hadoop的机架感知)的过程，Namenode可以确定每个Datanode所属的机架id。一个简单但没有优化的策略就是将副本存放在不同的机架上。这样可以有效防止当整个机架失效时数据的丢失，并且允许读数据的时候充分利用多个机架的带宽。这种策略设置可以将副本均匀分布在集群中，有利于当组件失效情况下的负载均衡。但是，因为这种策略的一个写操作需要传输数据块到多个机架，这增加了写的代价。
>
> 在大多数情况下，副本系数是3，HDFS的存放策略是将一个副本存放在**本地机架的节点**上，一个副本放在**同一机架的另一个节点**上，最后一个副本放在**不同机架的节点**上。这种策略减少了机架间的数据传输，这就提高了写操作的效率。
>
> 机架的错误远远比节点的错误少，所以这个策略不会影响到数据的可靠性和可用性。于此同时，因为数据块只放在两个（不是三个）不同的机架上，所以此策略减少了读取数据时需要的网络传输总带宽。
>
> 在这种策略下，副本并不是均匀分布在不同的机架上。三分之一的副本在一个节点上，三分之二的副本在一个机架上，其他副本均匀分布在剩下的机架中，这一策略在不损害数据可靠性和读取性能的情况下改进了写的性能。

#### 3. HDFS数据读取

1. 客户端请求向NameNode读取元数据
2. NameNode查看不同数据块不同副本的存放位置列表（列表中包含副本所在节点）
3. NameNode调用API来确定客户端和DataNode所属机架ID，返回离客户端最近的数据块元数据
4. 客户端根据元数据从指定DataNode读取数据

> [Hadoop分布式文件系统：架构和设计](https://hadoop.apache.org/docs/r1.0.4/cn/hdfs_design.html)
>
> 为了降低整体的带宽消耗和读取延时，HDFS会尽量让读取程序读取离它最近的副本。如果在读取程序的同一个机架上有一个副本，那么就读取该副本。如果一个HDFS集群跨越多个数据中心，那么客户端也将首先读本地数据中心的副本。

#### 4. 数据错误与恢复

​	HDFS具有较高的容错性，可以兼容廉价的硬件，它把硬件出错看作一种常态，而不是异常，并设计了检测数据错误和进行自动恢复的机制。主要包括以下几种情形：名称节点出错、数据节点出错和数据出错。

+ 名称节点出错

  NameNode出错，还有SecondaryNameNode提供的备份。

+ 数据节点出错

  每个DataNode会定期向名称节点发送"心跳"信息，向NameNode报告自己的状态。

  当某个DataNode出错，NameNode不会再给它们发送任何I/O请求。

  由于有DataNode出错，意味着某些数据的副本数量将小于冗余因子（默认是3），就会启动数据冗余复制，为缺失副本的数据生成新的副本。

  > **HDFS和其他分布式文件系统的最大区别就是可以调整冗余数据的位置**

+ 数据出错

  + 网络传输和磁盘错误等因素，都会造成数据错误
  + 客户端在读取到数据后，会采用md5和sha1对数据进行校验，以确定读取到正确的数据

  1. 在文件被创建时，客户端就会对每一个文件块进行信息摘录，并把这些信息写入到同一个路径的隐藏文件里面
  2. 请求到一个数据节点读取该数据块，并且向名称节点报告这个文件块有错误，名称节点会定期检查并且重新复制这个块

### 9.2.2 小结

1. HDFS的架构和相关概念
   + 块
   + NameNode
   + SecondaryNameNode
2. 数据存储和备份原理
   + 冗余备份
   + 名称节点、数据节点、数据恢复

## 9.3 HDFS的Java-API

+ HDFS文件操作
+ HDFS查看文件信息

+ HDFS压缩和解压缩文件
+ ...

## 9.4 HDFS数据读写过程

+ FileSystem

  是一个通用文件系统的抽象基类，可以被分布式文件系统继承，所有可能使用Hadoop文件系统的代码，都要使用这个类，Hadoop为FileSystem这个抽象类提供了多种具体实现

+ DistributedFileSystem

  是FileSystem在HDFS文件系统中的具体实现

+ FileSystem的open()方法

  + 返回的是一个输入流FSDataInputStream对象，在HDFS文件系统中，具体的输入流就是DFSInputStream

+ FileSystem中的create()方法

  返回的是一个输出流FSDataOutputStream对象，在HDFS文件系统中，具体的输出流就是DFSOutputStream

### 数据读写过程基本代码

```java
Configuration conf = new Configuration();
FileSystem fs = FileSystem.get(conf);
FSDataInputStream in = fs.open(new Path(uri));
FSDataOutputStream out = fs.create(new Path(uri));
```

​	备注：创建一个Configuration对象时，其构造方法会默认加载工程项目下两个配置文件，分别是`hdfs-site.xml`以及`core-site.xml`，这两个文件中会有访问HDFS所需的参数值，主要是`fs.defaultFS`，指定了HDFS的地址（比如`hdfs://localhost:9000`），有了这个地址客户端就可以通过这个地址访问HDFS了

### HDFS读写数据

> [HDFS读写数据流程](https://www.cnblogs.com/Java-Script/p/11090379.html)	<=	下面内容出自该博客

#### 读数据

　1. 与NameNode通信查询元数据，找到文件块所在的DataNode服务器
 　2. 挑选一台DataNode（网络拓扑上的就近原则，如果都一样，则随机挑选一台DataNode）服务器，请求建立socket流
 　3. DataNode开始发送数据(从磁盘里面读取数据放入流，以packet（一个packet为64kb）为单位来做校验)
 　4. 客户端以packet为单位接收，先在本地缓存，然后写入目标文件

![img](https://segmentfault.com/img/remote/1460000013767517?w=999&h=709)

#### 写数据

1. 跟NameNode通信请求上传文件，NameNode检查目标文件是否已经存在，父目录是否已经存在

2. NameNode返回是否可以上传

3. Client先对文件进行切分，请求第一个block该传输到哪些DataNode服务器上

4. NameNode返回3个DataNode服务器DataNode 1，DataNode 2，DataNode 3

5. Client请求3台中的一台DataNode 1(网络拓扑上的就近原则，如果都一样，则随机挑选一台DataNode)上传数据（本质上是一个RPC调用，建立pipeline）,DataNode 1收到请求会继续调用DataNode 2,然后DataNode 2调用DataNode 3，将整个pipeline建立完成，然后逐级返回客户端

6. Client开始往DataNode 1上传第一个block（先从磁盘读取数据放到一个本地内存缓存），以packet为单位。写入的时候DataNode会进行数据校验，它并不是通过一个packet进行一次校验而是以chunk为单位进行校验（512byte）。DataNode 1收到一个packet就会传给DataNode 2，DataNode 2传给DataNode 3，DataNode 1每传一个packet会放入一个应答队列等待应答

7. 当一个block传输完成之后，Client再次请求NameNode上传第二个block的服务器.

![img](https://img2018.cnblogs.com/blog/699090/201906/699090-20190626155745864-1227676006.png)

# 10. HDFS JAVA API-实操

​	Hadoop中关于文件操作类基本上全部是在"org.apache.hadoop.fs"包中，Hadoop类库中最终面向用户提供的接口类是FileSystem，该类封装了几乎所有的文件操作，例如CopyToLocalFile、CopyFromLocalFile、mkdir及delete等。

# 11. HDFS学习--习题

# 12. MapReduce理论概述

1. 分布式并行编程
2. 并行编程之MapReduce
3. Map和Reduce函数

## 12.1 MapReduce核心思想

> [一张图看懂MapReduce 架构是如何工作的？ ](https://www.sohu.com/a/131500649_628522)

![img](https://img.mp.itc.cn/upload/20170401/b147dd26ac6e4d5e80e493ca848677b5_th.jpg)

核心思想：**分而治之**。

​	一个存储在分布式文件系统HDFS中的大规模数据集，会被切分成许多独立的分片（split）即：

​	**一个大任务分成多个小的子任务（map），由多个节点进行并行执行，并行执行后，合并结果（reduce）**

<table>
  <tr>
  	<th>函数</th>
    <th>输入</th>
    <th>输出</th>
    <th>说明</th>
  </tr>
  <tr>
    <td>Map</td>
    <td>
    	<p>
        &lt;k1,v1&gt;<br/>
        如：<br/>
        &lt;行号，“a b c”&gt;
      </p>
    </td>
    <td>
    	<p>
        List(&lt;k2,v2&gt;)<br/>
        如：<br/>
        &lt;"a"，1&gt;<br/>
        &lt;"b"，1&gt;<br/>
        &lt;"c"，1&gt;
      </p>
    </td>
    <td>
    	<p>
        1. 将小数据集进一步解析成一批&lt;key，value&gt;对，输入Map函数中进行处理<br/>
        2. 每一个输入的&lt;k1,v1&gt;会输出一批&lt;k2,v2&gt;。&lt;k2,v2&gt;是计算的中间结果
      </p>
    </td>
  </tr>
  <tr>
    <td>Reduce</td>
    <td>
    	<p>
        &lt;k2,List(v2)&gt;<br/>
        如：<br/>
        &lt;"a"，&lt;1,1,1&gt;&gt;
      </p>
    </td>
    <td>
    	<p>
        &lt;k3,v3&gt;<br/>
        &lt;"a",3&gt;
      </p>
    </td>
    <td>
    	<p>
        输入的中间结果&lt;k2,List(v2)&gt;中的List(v2)表示是一批属于同一个k2的value
      </p>
    </td>
  </tr>
</table>

+ 分布式编程架构
+ 以数据为中心，更看重吞吐量
+ 分而治之的思想
+ Map将一个任务分解成多个子任务
+ Reduce将多个子任务的计算结果汇总

# 13. Mapreduce体系结构

+ Job
  + 一个任务，一个作业
  + 一个Job会被分成多个Task
+ Task
  + task里面，又分为Maptask和Reducetask，也就是一个Map任务和Reduce任务

+ JobTracker & TaskTracker
  + MapReduce框架采用了Master/Slave架构，包括一个Master和若干个Slave
  + Master上运行JobTracker，负责作业的调度、处理和失败后的恢复
  + Slave上运行的TaskTracker，负责接收JobTracker发给它的作业指令

> Hadoop框架是用Java实现的，但是，MapReduce应用程序不一定要用Java来写，也可以用python

JobTracker：

1. 作业调度
2. 分配任务、监控任务执行进度
3. 监控TaskTracker的状态

TaskTracker的角色：

1. 执行任务
2. 汇报任务状态

![img](https://images2015.cnblogs.com/blog/1166438/201706/1166438-20170626223359321-476907339.png)

MapReduce体系结构主要由四个部分组成，分别是：Client、JobTracker、TaskTracker以及Task

Client：客户端，用于提交作业

JobTracker：作业跟踪器，负责作业调度，作业执行，作业失败后恢复

TaskScheduler：任务调度器，负责任务调度

TaskTracker：任务跟踪器，负责任务管理(启动任务，杀死任务等)

1. Client-提交作业，查看作业状态
   **提交作业：**用户编写的MapReduce程序通过Client提交到JobTracker端
   **查看作业状态：**用户可通过Client提供的一些接口查看作业运行状态

2. JobTracker-资源监控、作业调度
   **JobTracker负责资源监控和作业调度**
   **资源监控：**JobTracker 监控所有TaskTracker与Job的健康状况，一旦发现节点失效(通信失败或节点故障)，就将相应的任务转移到其他节点
   **作业调度：**JobTracker 会跟踪任务的执行进度、资源使用量等信息，并将这些信息告诉任务调度器（TaskScheduler），而任务调度器会选择合适的(比较空闲)节点资源来执行任务

3. TaskScheduler-任务调度器 

4. TaskTracker-任务管理
   TaskTracker 会周期性地通过“心跳”将本节点上资源的使用情况和任务的运行进度汇报给JobTracker，同时接收JobTracker 发送过来的命令并执行相应的操作（如启动新任务、杀死任务等）
   TaskTracker 使用“slot”等量划分本节点上的资源量（CPU、内存等）。**一个Task 获取到一个slot 后才有机会运行**，而<u>Hadoop调度器(TaskScheduler)的作用就是将各个TaskTracker上的空闲slot分配给Task使用</u>。

   slot 分为Map slot 和Reduce slot 两种，分别供MapTask 和Reduce Task 使用

5. Task
   Task 分为Map Task 和Reduce Task 两种，均由TaskTracker 启动

> [MapReduce的体系结构](https://www.cnblogs.com/ostin/articles/7082822.html)

# 14. MapReduce学习--习题

# 15. Mapreduce实例:WordCount

分片=>Map=>中间磁盘=>Reduce=>输出

# 16. Mapreduce实例：WordCount-实操

## 16.1 相关知识

​	MapReduce采用的是“分而治之”的思想，把对大规模数据集的操作，分发给一个主节点管理下的各个从节点共同完成，然后通过整合各个节点的中间结果，得到最终结果。简单来说，MapReduce就是”任务的分解与结果的汇总“。

1. MapReduce的工作原理

在分布式计算中，MapReduce框架负责处理了并行编程里分布式存储、工作调度，负载均衡、容错处理以及网络通信等复杂问题，现在我们把处理过程高度抽象为Map与Reduce两个部分来进行阐述，其中Map部分负责把任务分解成多个子任务，Reduce部分负责把分解后多个子任务的处理结果汇总起来，具体设计思路如下。

（1）Map过程需要继承org.apache.hadoop.mapreduce包中Mapper类，并重写其map方法。通过在map方法中添加两句把key值和value值输出到控制台的代码，可以发现map方法中输入的value值存储的是文本文件中的一行（以回车符为行结束标记），而输入的key值存储的是该行的首字母相对于文本文件的首地址的偏移量。然后用StringTokenizer类将每一行拆分成为一个个的字段，把截取出需要的字段（本实验为买家id字段）设置为key，并将其作为map方法的结果输出。

（2）Reduce过程需要继承org.apache.hadoop.mapreduce包中Reducer类，并重写其reduce方法。Map过程输出的<key,value>键值对先经过shuffle过程把key值相同的所有value值聚集起来形成values，此时values是对应key字段的计数值所组成的列表，然后将<key,values>输入到reduce方法中，reduce方法只要遍历values并求和，即可得到某个单词的总次数。

在main()主函数中新建一个Job对象，由Job对象负责管理和运行MapReduce的一个计算任务，并通过Job的一些方法对任务的参数进行相关的设置。本实验是设置使用将继承Mapper的doMapper类完成Map过程中的处理和使用doReducer类完成Reduce过程中的处理。还设置了Map过程和Reduce过程的输出类型：key的类型为Text，value的类型为IntWritable。任务的输出和输入路径则由字符串指定，并由FileInputFormat和FileOutputFormat分别设定。完成相应任务的参数设定后，即可调用job.waitForCompletion()方法执行任务，其余的工作都交由MapReduce框架处理。

2. MapReduce框架的作业运行流程

[![img](https://www.ipieuvre.com/doc/exper/1e146ce6-91ad-11e9-beeb-00215ec892f4/img/01.png)](https://www.ipieuvre.com/doc/exper/1e146ce6-91ad-11e9-beeb-00215ec892f4/img/01.png)

（1）ResourceManager：是YARN资源控制框架的中心模块，负责集群中所有资源的统一管理和分配。它接收来自NM(NodeManager)的汇报，建立AM，并将资源派送给AM(ApplicationMaster)。

（2）NodeManager：简称NM，NodeManager是ResourceManager在每台机器上的代理，负责容器管理，并监控他们的资源使用情况（cpu、内存、磁盘及网络等），以及向ResourceManager提供这些资源使用报告。

（3）ApplicationMaster：以下简称AM。YARN中每个应用都会启动一个AM，负责向RM申请资源，请求NM启动Container，并告诉Container做什么事情。

（4）Container：资源容器。YARN中所有的应用都是在Container之上运行的。AM也是在Container上运行的，不过AM的Container是RM申请的。Container是YARN中资源的抽象，它封装了某个节点上一定量的资源（CPU和内存两类资源）。Container由ApplicationMaster向ResourceManager申请的，由ResouceManager中的资源调度器异步分配给ApplicationMaster。Container的运行是由ApplicationMaster向资源所在的NodeManager发起的，Container运行时需提供内部执行的任务命令（可以是任何命令，比如java、Python、C++进程启动命令均可）以及该命令执行所需的环境变量和外部资源（比如词典文件、可执行文件、jar包等）。

另外，一个应用程序所需的Container分为两大类，如下：

①运行ApplicationMaster的Container：这是由ResourceManager（向内部的资源调度器）申请和启动的，用户提交应用程序时，可指定唯一的ApplicationMaster所需的资源。

②运行各类任务的Container：这是由ApplicationMaster向ResourceManager申请的，并为了ApplicationMaster与NodeManager通信以启动的。

以上两类Container可能在任意节点上，它们的位置通常而言是随机的，即ApplicationMaster可能与它管理的任务运行在一个节点上。

## 16.2 编写思路

下图描述了该mapreduce的执行过程

[![img](https://www.ipieuvre.com/doc/exper/1e146ce6-91ad-11e9-beeb-00215ec892f4/img/12.png)](https://www.ipieuvre.com/doc/exper/1e146ce6-91ad-11e9-beeb-00215ec892f4/img/12.png)

​	大致思路是将hdfs上的文本作为输入，MapReduce通过InputFormat会将文本进行切片处理，并将每行的首字母相对于文本文件的首地址的偏移量作为输入键值对的key，文本内容作为输入键值对的value，经过在map函数处理，输出中间结果<word,1>的形式，并在reduce函数中完成对每个单词的词频统计。整个程序代码主要包括两部分：Mapper部分和Reducer部分。

+ Mapper代码

  ```java
    public static class doMapper extends Mapper<Object, Text, Text, IntWritable>{  
      //第一个Object表示输入key的类型；第二个Text表示输入value的类型；第三个Text表示输出键的类型；第四个IntWritable表示输出值的类型  
      public static final IntWritable one = new IntWritable(1);  
      public static Text word = new Text();  
      @Override  
      protected void map(Object key, Text value, Context context)  
        throws IOException, InterruptedException  
        //抛出异常  
      {  
        StringTokenizer tokenizer = new StringTokenizer(value.toString(),"\t");  
        //StringTokenizer是Java工具包中的一个类，用于将字符串进行拆分  
  
        word.set(tokenizer.nextToken());  
        //返回当前位置到下一个分隔符之间的字符串  
        context.write(word, one);  
        //将word存到容器中，记一个数  
      }
    }
  ```

​	在map函数里有三个参数，前面两个Object key,Text value就是输入的key和value，第三个参数Context context是可以记录输入的key和value。例如context.write(word,one)；此外context还会记录map运算的状态。map阶段采用Hadoop的默认的作业输入方式，把输入的value用StringTokenizer()方法截取出的买家id字段设置为key，设置value为1，然后直接输出<key,value>。

+ Reducer代码

  ```java
  public static class doReducer extends Reducer<Text, IntWritable, Text, IntWritable>{  
    //参数同Map一样，依次表示是输入键类型，输入值类型，输出键类型，输出值类型  
    private IntWritable result = new IntWritable();  
    @Override  
    protected void reduce(Text key, Iterable<IntWritable> values, Context context)  
      throws IOException, InterruptedException {  
      int sum = 0;  
      for (IntWritable value : values) {  
        sum += value.get();  
      }  
      //for循环遍历，将得到的values值累加  
      result.set(sum);  
      context.write(key, result);  
    }  
  }
  ```

​	**map输出的<key,value>先要经过shuffle过程把相同key值的所有value聚集起来形成<key,values>后交给reduce端**。reduce端接收到<key,values>之后，将输入的key直接复制给输出的key,用for循环遍历values并求和，求和结果就是key值代表的单词出现的总次，将其设置为value，直接输出<key,value>。

---

完整代码：

```java
package mapreduce;  
import java.io.IOException;  
import java.util.StringTokenizer;  
import org.apache.hadoop.fs.Path;  
import org.apache.hadoop.io.IntWritable;  
import org.apache.hadoop.io.Text;  
import org.apache.hadoop.mapreduce.Job;  
import org.apache.hadoop.mapreduce.Mapper;  
import org.apache.hadoop.mapreduce.Reducer;  
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;  
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;  
public class WordCount {  
  public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {  
    Job job = Job.getInstance();  
    job.setJobName("WordCount");  
    job.setJarByClass(WordCount.class);  
    job.setMapperClass(doMapper.class);  
    job.setReducerClass(doReducer.class);  
    job.setOutputKeyClass(Text.class);  
    job.setOutputValueClass(IntWritable.class);  
    Path in = new Path("hdfs://localhost:9000/mymapreduce1/in/buyer_favorite1");  
    Path out = new Path("hdfs://localhost:9000/mymapreduce1/out");  
    FileInputFormat.addInputPath(job, in);  
    FileOutputFormat.setOutputPath(job, out);  
    System.exit(job.waitForCompletion(true) ? 0 : 1);  
  }  
  public static class doMapper extends Mapper<Object, Text, Text, IntWritable>{  
    public static final IntWritable one = new IntWritable(1);  
    public static Text word = new Text();  
    @Override  
    protected void map(Object key, Text value, Context context)  
      throws IOException, InterruptedException {  
      StringTokenizer tokenizer = new StringTokenizer(value.toString(), "\t");  
      word.set(tokenizer.nextToken());  
      context.write(word, one);  
    }  
  }  
  public static class doReducer extends Reducer<Text, IntWritable, Text, IntWritable>{  
    private IntWritable result = new IntWritable();  
    @Override  
    protected void reduce(Text key, Iterable<IntWritable> values, Context context)  
      throws IOException, InterruptedException {  
      int sum = 0;  
      for (IntWritable value : values) {  
        sum += value.get();  
      }  
      result.set(sum);  
      context.write(key, result);  
    }  
  }  
} 
```

# 17. MapReduce统计—求平均值

​	求平均值是MapReduce比较常见的算法，求平均数的算法思路：

​	Map端读取数据，在数据输入到Reduce之前先经过shuffle，将map函数输出的key值相同的所有value值形成一个集合value-list，然后将输入到Reduce端，Reduce端汇总并且统计记录数，然后作商即可。

# 18. MapReduce统计—求平均值-实操

## 18.1 相关知识

求平均数是MapReduce比较常见的算法，求平均数的算法也比较简单，一种思路是Map端读取数据，在数据输入到Reduce之前先经过shuffle，将map函数输出的key值相同的所有的value值形成一个集合value-list，然后将输入到Reduce端，Reduce端汇总并且统计记录数，然后作商即可。具体原理如下图所示：

[![img](https://www.ipieuvre.com/doc/exper/1e3fa46d-91ad-11e9-beeb-00215ec892f4/img/01.png)](https://www.ipieuvre.com/doc/exper/1e3fa46d-91ad-11e9-beeb-00215ec892f4/img/01.png)

## 18.2 编写思路

+ Mapper代码

  ```java
  public static class Map extends Mapper<Object , Text , Text , IntWritable>{  
    private static Text newKey=new Text();  
    //实现map函数  
    public void map(Object key,Text value,Context context) throws IOException, InterruptedException{  
      // 将输入的纯文本文件的数据转化成String  
      String line=value.toString();  
      System.out.println(line);  
      String arr[]=line.split("\t");  
      newKey.set(arr[0]);  
      int click=Integer.parseInt(arr[1]);  
      context.write(newKey, new IntWritable(click));  
    }  
  }
  ```

  map端在采用Hadoop的默认输入方式之后，将输入的value值通过split()方法截取出来，我们把截取的商品点击次数字段转化为IntWritable类型并将其设置为value，把商品分类字段设置为key,然后直接输出key/value的值。

+ Reducer代码

  ```java
  public static class Reduce extends Reducer<Text, IntWritable, Text, IntWritable>{  
    //实现reduce函数  
    public void reduce(Text key,Iterable<IntWritable> values,Context context) throws IOException, InterruptedException{  
      int num=0;  
      int count=0;  
      for(IntWritable val:values){  
        num+=val.get(); //每个元素求和num  
        count++;        //统计元素的次数count  
      }  
      int avg=num/count;  //计算平均数  
  
      context.write(key,new IntWritable(avg));  
    }  
  }  
  ```

  map的输出<key,value>经过shuffle过程集成<key,values>键值对，然后将<key,values>键值对交给reduce。reduce端接收到values之后，将输入的key直接复制给输出的key，将values通过for循环把里面的每个元素求和num并统计元素的次数count，然后用num除以count 得到平均值avg，将avg设置为value，最后直接输出<key,value>就可以了。

+ 完整代码

  ```java
  package mapreduce;  
  import java.io.IOException;  
  import org.apache.hadoop.conf.Configuration;  
  import org.apache.hadoop.fs.Path;  
  import org.apache.hadoop.io.IntWritable;  
  import org.apache.hadoop.io.NullWritable;  
  import org.apache.hadoop.io.Text;  
  import org.apache.hadoop.mapreduce.Job;  
  import org.apache.hadoop.mapreduce.Mapper;  
  import org.apache.hadoop.mapreduce.Reducer;  
  import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;  
  import org.apache.hadoop.mapreduce.lib.input.TextInputFormat;  
  import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;  
  import org.apache.hadoop.mapreduce.lib.output.TextOutputFormat;  
  public class MyAverage{  
    public static class Map extends Mapper<Object , Text , Text , IntWritable>{  
      private static Text newKey=new Text();  
      public void map(Object key,Text value,Context context) throws IOException, InterruptedException{  
        String line=value.toString();  
        System.out.println(line);  
        String arr[]=line.split("\t");  
        newKey.set(arr[0]);  
        int click=Integer.parseInt(arr[1]);  
        context.write(newKey, new IntWritable(click));  
      }  
    }  
    public static class Reduce extends Reducer<Text, IntWritable, Text, IntWritable>{  
      public void reduce(Text key,Iterable<IntWritable> values,Context context) throws IOException, InterruptedException{  
        int num=0;  
        int count=0;  
        for(IntWritable val:values){  
          num+=val.get();  
          count++;  
        }  
        int avg=num/count;  
        context.write(key,new IntWritable(avg));  
      }  
    }  
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException{  
      Configuration conf=new Configuration();  
      System.out.println("start");  
      Job job =new Job(conf,"MyAverage");  
      job.setJarByClass(MyAverage.class);  
      job.setMapperClass(Map.class);  
      job.setReducerClass(Reduce.class);  
      job.setOutputKeyClass(Text.class);  
      job.setOutputValueClass(IntWritable.class);  
      job.setInputFormatClass(TextInputFormat.class);  
      job.setOutputFormatClass(TextOutputFormat.class);  
      Path in=new Path("hdfs://localhost:9000/mymapreduce4/in/goods_click");  
      Path out=new Path("hdfs://localhost:9000/mymapreduce4/out");  
      FileInputFormat.addInputPath(job,in);  
      FileOutputFormat.setOutputPath(job,out);  
      System.exit(job.waitForCompletion(true) ? 0 : 1);  
  
    }  
  }
  ```

# 19. MapReduce统计—去重

数据去重：主要是对数据进行有意义的筛选。统计大数据集上的数据种类个数，从网站日志中计算访问地等这些看似庞杂的任务都会涉及数据去重。最终目标是让原始数据中出现次数超过一次的数据在输出文件中只出现一次。

# 20. MapReduce统计—去重-实操

## 20.1 相关知识

​	“数据去重”主要是为了掌握和利用并行化思想来对数据进行有意义的筛选。统计大数据集上的数据种类个数、从网站日志中计算访问地等这些看似庞杂的任务都会涉及数据去重。

​	数据去重的最终目标是让原始数据中出现次数超过一次的数据在输出文件中只出现一次。在MapReduce流程中，map的输出<key,value>经过shuffle过程聚集成<key,value-list>后交给reduce。我们自然而然会想到将同一个数据的所有记录都交给一台reduce机器，无论这个数据出现多少次，只要在最终结果中输出一次就可以了。具体就是reduce的输入应该以数据作为key，而对value-list则没有要求（可以设置为空）。当reduce接收到一个<key,value-list>时就直接将输入的key复制到输出的key中，并将value设置成空值，然后输出<key,value>。

MaprReduce去重流程如下图所示：

[![img](https://www.ipieuvre.com/doc/exper/1e213b2a-91ad-11e9-beeb-00215ec892f4/img/01.png)](https://www.ipieuvre.com/doc/exper/1e213b2a-91ad-11e9-beeb-00215ec892f4/img/01.png)

## 20.2 编写思路

​	数据去重的目的是让原始数据中出现次数超过一次的数据在输出文件中只出现一次。我们自然想到将相同key值的所有value记录交到一台reduce机器，让其无论这个数据出现多少次，最终结果只输出一次。具体就是reduce的输出应该以数据作为key,而对value-list没有要求，当reduce接收到一个时，就直接将key复制到输出的key中，将value设置为空。

+ Map代码

  ```java
  public static class Map extends Mapper<Object , Text , Text , NullWritable>  
    //map将输入中的value复制到输出数据的key上，并直接输出  
  {  
    private static Text newKey=new Text();      //从输入中得到的每行的数据的类型  
    public void map(Object key,Text value,Context context) throws IOException, InterruptedException  
      //实现map函数  
    {             //获取并输出每一次的处理过程  
      String line=value.toString();  
      System.out.println(line);  
      String arr[]=line.split("\t");  
      newKey.set(arr[1]);  
      context.write(newKey, NullWritable.get());  
      System.out.println(newKey);  
    }  
  } 
  ```

  map阶段采用Hadoop的默认的作业输入方式，把输入的value用split()方法截取，截取出的商品id字段设置为key,设置value为空，然后直接输出<key,value>。

+ Reduce代码

  ```java
  public static class Reduce extends Reducer<Text, NullWritable, Text, NullWritable>{  
    public void reduce(Text key,Iterable<NullWritable> values,Context context) throws IOException, InterruptedException  
      //实现reduce函数  
    {  
      context.write(key,NullWritable.get());   //获取并输出每一次的处理过程  
    }  
  }  
  ```

  map输出的<key,value>键值对经过shuffle过程，聚成<key,value-list>后，会交给reduce函数。reduce函数,不管每个key 有多少个value，它直接将输入的值赋值给输出的key，将输出的value设置为空，然后输出<key,value>就可以了。

+ 完整代码

  ```java
  package mapreduce;  
  import java.io.IOException;  
  import org.apache.hadoop.conf.Configuration;  
  import org.apache.hadoop.fs.Path;  
  import org.apache.hadoop.io.IntWritable;  
  import org.apache.hadoop.io.NullWritable;  
  import org.apache.hadoop.io.Text;  
  import org.apache.hadoop.mapreduce.Job;  
  import org.apache.hadoop.mapreduce.Mapper;  
  import org.apache.hadoop.mapreduce.Reducer;  
  import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;  
  import org.apache.hadoop.mapreduce.lib.input.TextInputFormat;  
  import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;  
  import org.apache.hadoop.mapreduce.lib.output.TextOutputFormat;  
  public class Filter{  
    public static class Map extends Mapper<Object , Text , Text , NullWritable>{  
      private static Text newKey=new Text();  
      public void map(Object key,Text value,Context context) throws IOException, InterruptedException{  
        String line=value.toString();  
        System.out.println(line);  
        String arr[]=line.split("\t");  
        newKey.set(arr[1]);  
        context.write(newKey, NullWritable.get());  
        System.out.println(newKey);  
      }  
    }  
    public static class Reduce extends Reducer<Text, NullWritable, Text, NullWritable>{  
      public void reduce(Text key,Iterable<NullWritable> values,Context context) throws IOException, InterruptedException{  
        context.write(key,NullWritable.get());  
      }  
    }  
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException{  
      Configuration conf=new Configuration();  
      System.out.println("start");  
      Job job =new Job(conf,"filter");  
      job.setJarByClass(Filter.class);  
      job.setMapperClass(Map.class);  
      job.setReducerClass(Reduce.class);  
      job.setOutputKeyClass(Text.class);  
      job.setOutputValueClass(NullWritable.class);  
      job.setInputFormatClass(TextInputFormat.class);  
      job.setOutputFormatClass(TextOutputFormat.class);  
      Path in=new Path("hdfs://localhost:9000/mymapreduce2/in/buyer_favorite1");  
      Path out=new Path("hdfs://localhost:9000/mymapreduce2/out");  
      FileInputFormat.addInputPath(job,in);  
      FileOutputFormat.setOutputPath(job,out);  
      System.exit(job.waitForCompletion(true) ? 0 : 1);  
    }  
  }
  ```

# 21. MapReduce排序—自然排序

​	在MapReduce过程中默认就有对数据的排序。它是按照key值进行排序的

+ 如果key为封装int的IntWriable类型，那么MapReduce会按照数字大小对key排序
+ 如果key为封装String的Text类型，那么MapReduce将按照数据字典顺序对字符排序

MapReduce框架会确保每一个Reducer的输入都是按Key进行排序的。

一般，将排序以及Map的输出传输到Reduce的过程称为混洗（shuffle）。

Shuffle阶段的排序可以理解成两部分：

1. 一个是对spill进行分区时，由于一个分区包含多个key值，所以要对分区内的<key,value>按照key进行排序，即key值相同的一串<key,value>存放在一起，这样一个partition内按照key值整体有序了
2. 第二部分并不是排序，而是进行merge，merge有两次，一次是map端将多个spill按照分区和分区内的key进行merge，形成一个大的文件。第二次merge是在reduce端，进入同一个reduce的多个map的输出merge在一起

## 21.1 MapReduce的Shuffle

> [Hadoop学习之路（二十三）MapReduce中的shuffle详解](https://www.cnblogs.com/qingyunzong/p/8615024.html)	<=	以下内容出自该博客

​	从Map输出到Reduce输入的整个过程可以广义地称为Shuffle。Shuffle横跨Map端和Reduce端，在Map端包括Spill过程，在Reduce端包括copy和sort过程，如图所示：

![img](https://images2018.cnblogs.com/blog/1228818/201803/1228818-20180321083504916-1942630366.png)

**Spill过程**

Spill过程包括输出、排序、溢写、合并等步骤，如图所示：

![img](https://images2018.cnblogs.com/blog/1228818/201803/1228818-20180321083605120-770489646.png)

**Collect**

每个Map任务不断地以对的形式把数据输出到在内存中构造的一个环形数据结构中。使用环形数据结构是为了更有效地使用内存空间，在内存中放置尽可能多的数据。

这个数据结构其实就是个字节数组，叫Kvbuffer，名如其义，但是这里面不光放置了数据，还放置了一些索引数据，给放置索引数据的区域起了一个Kvmeta的别名，在Kvbuffer的一块区域上穿了一个IntBuffer（字节序采用的是平台自身的字节序）的马甲。数据区域和索引数据区域在Kvbuffer中是相邻不重叠的两个区域，用一个分界点来划分两者，分界点不是亘古不变的，而是每次Spill之后都会更新一次。初始的分界点是0，数据的存储方向是向上增长，索引数据的存储方向是向下增长，如图所示：

![img](https://images2018.cnblogs.com/blog/1228818/201803/1228818-20180321083814970-49385953.png)

Kvbuffer的存放指针bufindex是一直闷着头地向上增长，比如bufindex初始值为0，一个Int型的key写完之后，bufindex增长为4，一个Int型的value写完之后，bufindex增长为8。

索引是对在kvbuffer中的索引，是个四元组，包括：value的起始位置、key的起始位置、partition值、value的长度，占用四个Int长度，Kvmeta的存放指针Kvindex每次都是向下跳四个“格子”，然后再向上一个格子一个格子地填充四元组的数据。比如Kvindex初始位置是-4，当第一个写完之后，(Kvindex+0)的位置存放value的起始位置、(Kvindex+1)的位置存放key的起始位置、(Kvindex+2)的位置存放partition的值、(Kvindex+3)的位置存放value的长度，然后Kvindex跳到-8位置，等第二个和索引写完之后，Kvindex跳到-32位置。

Kvbuffer的大小虽然可以通过参数设置，但是总共就那么大，和索引不断地增加，加着加着，Kvbuffer总有不够用的那天，那怎么办？把数据从内存刷到磁盘上再接着往内存写数据，把Kvbuffer中的数据刷到磁盘上的过程就叫Spill，多么明了的叫法，内存中的数据满了就自动地spill到具有更大空间的磁盘。

关于Spill触发的条件，也就是Kvbuffer用到什么程度开始Spill，还是要讲究一下的。如果把Kvbuffer用得死死得，一点缝都不剩的时候再开始Spill，那Map任务就需要等Spill完成腾出空间之后才能继续写数据；如果Kvbuffer只是满到一定程度，比如80%的时候就开始Spill，那在Spill的同时，Map任务还能继续写数据，如果Spill够快，Map可能都不需要为空闲空间而发愁。两利相衡取其大，一般选择后者。

Spill这个重要的过程是由Spill线程承担，Spill线程从Map任务接到“命令”之后就开始正式干活，干的活叫SortAndSpill，原来不仅仅是Spill，在Spill之前还有个颇具争议性的Sort。

**Sort**

先把Kvbuffer中的数据按照partition值和key两个关键字升序排序，移动的只是索引数据，排序结果是Kvmeta中数据按照partition为单位聚集在一起，同一partition内的按照key有序。

**Spill**

Spill线程为这次Spill过程创建一个磁盘文件：从所有的本地目录中轮训查找能存储这么大空间的目录，找到之后在其中创建一个类似于“spill12.out”的文件。Spill线程根据排过序的Kvmeta挨个partition的把数据吐到这个文件中，一个partition对应的数据吐完之后顺序地吐下个partition，直到把所有的partition遍历完。一个partition在文件中对应的数据也叫段(segment)。

所有的partition对应的数据都放在这个文件里，虽然是顺序存放的，但是怎么直接知道某个partition在这个文件中存放的起始位置呢？强大的索引又出场了。有一个三元组记录某个partition对应的数据在这个文件中的索引：起始位置、原始数据长度、压缩之后的数据长度，一个partition对应一个三元组。然后把这些索引信息存放在内存中，如果内存中放不下了，后续的索引信息就需要写到磁盘文件中了：从所有的本地目录中轮训查找能存储这么大空间的目录，找到之后在其中创建一个类似于“spill12.out.index”的文件，文件中不光存储了索引数据，还存储了crc32的校验数据。(spill12.out.index不一定在磁盘上创建，如果内存（默认1M空间）中能放得下就放在内存中，即使在磁盘上创建了，和spill12.out文件也不一定在同一个目录下。)

每一次Spill过程就会最少生成一个out文件，有时还会生成index文件，Spill的次数也烙印在文件名中。索引文件和数据文件的对应关系如下图所示：

![img](https://images2018.cnblogs.com/blog/1228818/201803/1228818-20180321083927742-1030906351.png)

在Spill线程如火如荼的进行SortAndSpill工作的同时，Map任务不会因此而停歇，而是一无既往地进行着数据输出。Map还是把数据写到kvbuffer中，那问题就来了：只顾着闷头按照bufindex指针向上增长，kvmeta只顾着按照Kvindex向下增长，是保持指针起始位置不变继续跑呢，还是另谋它路？如果保持指针起始位置不变，很快bufindex和Kvindex就碰头了，碰头之后再重新开始或者移动内存都比较麻烦，不可取。Map取kvbuffer中剩余空间的中间位置，用这个位置设置为新的分界点，bufindex指针移动到这个分界点，Kvindex移动到这个分界点的-16位置，然后两者就可以和谐地按照自己既定的轨迹放置数据了，当Spill完成，空间腾出之后，不需要做任何改动继续前进。分界点的转换如下图所示：

![img](https://images2018.cnblogs.com/blog/1228818/201803/1228818-20180321083949570-945173243.png)

Map任务总要把输出的数据写到磁盘上，即使输出数据量很小在内存中全部能装得下，在最后也会把数据刷到磁盘上。

**Merge**

Map任务如果输出数据量很大，可能会进行好几次Spill，out文件和Index文件会产生很多，分布在不同的磁盘上。最后把这些文件进行合并的merge过程闪亮登场。

Merge过程怎么知道产生的Spill文件都在哪了呢？从所有的本地目录上扫描得到产生的Spill文件，然后把路径存储在一个数组里。Merge过程又怎么知道Spill的索引信息呢？没错，也是从所有的本地目录上扫描得到Index文件，然后把索引信息存储在一个列表里。到这里，又遇到了一个值得纳闷的地方。在之前Spill过程中的时候为什么不直接把这些信息存储在内存中呢，何必又多了这步扫描的操作？特别是Spill的索引数据，之前当内存超限之后就把数据写到磁盘，现在又要从磁盘把这些数据读出来，还是需要装到更多的内存中。之所以多此一举，是因为这时kvbuffer这个内存大户已经不再使用可以回收，有内存空间来装这些数据了。（对于内存空间较大的土豪来说，用内存来省却这两个io步骤还是值得考虑的。）

然后为merge过程创建一个叫file.out的文件和一个叫file.out.Index的文件用来存储最终的输出和索引。

一个partition一个partition的进行合并输出。对于某个partition来说，从索引列表中查询这个partition对应的所有索引信息，每个对应一个段插入到段列表中。也就是这个partition对应一个段列表，记录所有的Spill文件中对应的这个partition那段数据的文件名、起始位置、长度等等。

然后对这个partition对应的所有的segment进行合并，目标是合并成一个segment。当这个partition对应很多个segment时，会分批地进行合并：先从segment列表中把第一批取出来，以key为关键字放置成最小堆，然后从最小堆中每次取出最小的输出到一个临时文件中，这样就把这一批段合并成一个临时的段，把它加回到segment列表中；再从segment列表中把第二批取出来合并输出到一个临时segment，把其加入到列表中；这样往复执行，直到剩下的段是一批，输出到最终的文件中。

最终的索引数据仍然输出到Index文件中。

![img](https://images2018.cnblogs.com/blog/1228818/201803/1228818-20180321084015633-2083307488.png)

Map端的Shuffle过程到此结束。

**Copy**

Reduce任务通过HTTP向各个Map任务拖取它所需要的数据。每个节点都会启动一个常驻的HTTP server，其中一项服务就是响应Reduce拖取Map数据。当有MapOutput的HTTP请求过来的时候，HTTP server就读取相应的Map输出文件中对应这个Reduce部分的数据通过网络流输出给Reduce。

Reduce任务拖取某个Map对应的数据，如果在内存中能放得下这次数据的话就直接把数据写到内存中。Reduce要向每个Map去拖取数据，在内存中每个Map对应一块数据，当内存中存储的Map数据占用空间达到一定程度的时候，开始启动内存中merge，把内存中的数据merge输出到磁盘上一个文件中。

如果在内存中不能放得下这个Map的数据的话，直接把Map数据写到磁盘上，在本地目录创建一个文件，从HTTP流中读取数据然后写到磁盘，使用的缓存区大小是64K。拖一个Map数据过来就会创建一个文件，当文件数量达到一定阈值时，开始启动磁盘文件merge，把这些文件合并输出到一个文件。

有些Map的数据较小是可以放在内存中的，有些Map的数据较大需要放在磁盘上，这样最后Reduce任务拖过来的数据有些放在内存中了有些放在磁盘上，最后会对这些来一个全局合并。

**Merge Sort**

这里使用的Merge和Map端使用的Merge过程一样。Map的输出数据已经是有序的，Merge进行一次合并排序，所谓Reduce端的sort过程就是这个合并的过程。一般Reduce是一边copy一边sort，即copy和sort两个阶段是重叠而不是完全分开的。

Reduce端的Shuffle过程至此结束。

# 22. MapReduce排序—二次排序

## 22.1 二次排序原理

+ 二次排序：在mapreduce中，所有key是需要被比较和排序的，并且是二次，先根据partitioner，再根据大小。

+ 二次排序原理：先按照第一字段排序，然后在第一字段相同时按照第二字段排序。根据这一点，我们可以构造一个复合类IntPair，他有两个字段，先利用分区对第一字段排序，再利用分区内的比较对第二字段排序。Java代码主要分为四部分：自定义key，自定义分区函数类，map部分，reduce部分。

## 22.2 小结

注意：输出应该符合自定义Map中定义的输出<IntPair,IntWritable>。最终是生成一个List<IntPair,IntWritable>。

**在map阶段的最后，会先调用job.setPartitionerClass对这个List进行分区，每个分区映射到一个reducer**。

**每个分区内又调用job.setSortComparatorClass设置的key比较函数类排序**。这本身就是一个二次排序。

# 23. Mapreduce实例——排序-实操

## 23.1 相关知识

Map、Reduce任务中Shuffle和排序的过程图如下：

[![img](https://www.ipieuvre.com/doc/exper/1e30f996-91ad-11e9-beeb-00215ec892f4/img/01.png)](https://www.ipieuvre.com/doc/exper/1e30f996-91ad-11e9-beeb-00215ec892f4/img/01.png)

流程分析：

1.Map端：

（1）每个输入分片会让一个map任务来处理，**默认情况下，以HDFS的一个块的大小（默认为64M）为一个分片**，当然我们也可以设置块的大小。map输出的结果会暂且放在一个环形内存缓冲区中（该缓冲区的大小默认为100M，由io.sort.mb属性控制），**当该缓冲区快要溢出时（默认为缓冲区大小的80%，由io.sort.spill.percent属性控制），会在本地文件系统中创建一个溢出文件，将该缓冲区中的数据写入这个文件**。

（2）**在写入磁盘之前，线程首先根据reduce任务的数目将数据划分为相同数目的分区，也就是一个reduce任务对应一个分区的数据**。这样做是为了避免有些reduce任务分配到大量数据，而有些reduce任务却分到很少数据，甚至没有分到数据的尴尬局面。其实分区就是对数据进行hash的过程。然后**对每个分区中的数据进行排序**，**如果此时设置了Combiner，将排序后的结果进行Combine操作，这样做的目的是让尽可能少的数据写入到磁盘**。

（3）**当map任务输出最后一个记录时，可能会有很多的溢出文件，这时需要将这些文件合并**。合并的过程中会不断地进行排序和combine操作，目的有两个：①尽量减少每次写入磁盘的数据量。②尽量减少下一复制阶段网络传输的数据量。最后合并成了一个已分区且已排序的文件。为了减少网络传输的数据量，这里可以将数据压缩，只要将mapred.compress.map.out设置为true就可以了。

（4）将分区中的数据拷贝给相对应的reduce任务。有人可能会问：分区中的数据怎么知道它对应的reduce是哪个呢？其实**map任务一直和其父TaskTracker保持联系，而TaskTracker又一直和JobTracker保持心跳。所以JobTracker中保存了整个集群中的宏观信息。只要reduce任务向JobTracker获取对应的map输出位置就ok了哦**。

​	到这里，map端就分析完了。那到底什么是Shuffle呢？Shuffle的中文意思是“洗牌”，如果我们这样看：一个map产生的数据，结果通过hash过程分区却分配给了不同的reduce任务，是不是一个对数据洗牌的过程呢？

2.Reduce端：

（1）Reduce会接收到不同map任务传来的数据，并且每个map传来的数据都是有序的。**如果reduce端接受的数据量相当小，则直接存储在内存中**（缓冲区大小由mapred.job.shuffle.input.buffer.percent属性控制，表示用作此用途的堆空间的百分比），**如果数据量超过了该缓冲区大小的一定比例（由mapred.job.shuffle.merge.percent决定），则对数据合并后溢写到磁盘中**。

（2）随着溢写文件的增多，后台线程会将它们合并成一个更大的有序的文件，这样做是为了给后面的合并节省时间。**其实不管在map端还是reduce端，MapReduce都是反复地执行排序，合并操作，现在终于明白了有些人为什么会说：排序是hadoop的灵魂**。

（3）合并的过程中会产生许多的中间文件（写入磁盘了），但MapReduce会让写入磁盘的数据尽可能地少，并且**最后一次合并的结果并没有写入磁盘，而是直接输入到reduce函数**。

​	熟悉MapReduce的人都知道：**排序是MapReduce的天然特性！在数据达到reducer之前，MapReduce框架已经对这些数据按键排序了。但是在使用之前，首先需要了解它的默认排序规则。它是按照key值进行排序的，如果key为封装的int为IntWritable类型，那么MapReduce按照数字大小对key排序，如果Key为封装String的Text类型，那么MapReduce将按照数据字典顺序对字符排序。**

​	了解了这个细节，我们就知道应该使用封装int的Intwritable型数据结构了，也就是在map这里，将读入的数据中要排序的字段转化为Intwritable型，然后作为key值输出（不排序的字段作为value）。reduce阶段拿到<key，value-list>之后，将输入的key作为输出的key，并根据value-list中的元素的个数决定输出的次数。

## 23.2 编写思路

​	在MapReduce过程中默认就有对数据的排序。它是按照key值进行排序的，如果key为封装int的IntWritable类型，那么MapReduce会按照数字大小对key排序，如果Key为封装String的Text类型，那么MapReduce将按照数据字典顺序对字符排序。在本例中我们用到第一种，key设置为IntWritable类型，其中MapReduce程序主要分为Map部分和Reduce部分。

+ Map部分代码

  ```java
  public static class Map extends Mapper<Object,Text,IntWritable,Text>{  
    private static Text goods=new Text();  
    private static IntWritable num=new IntWritable();  
    public void map(Object key,Text value,Context context) throws IOException, InterruptedException{  
      String line=value.toString();  
      String arr[]=line.split("\t");  
      num.set(Integer.parseInt(arr[1]));  
      goods.set(arr[0]);  
      context.write(num,goods);  
    }  
  }  
  ```

  在map端采用Hadoop默认的输入方式之后，将输入的value值用split()方法截取，把要排序的点击次数字段转化为IntWritable类型并设置为key，商品id字段设置为value，然后直接输出<key,value>。map输出的<key,value>先要经过shuffle过程把相同key值的所有value聚集起来形成<key,value-list>后交给reduce端。

+ Reduce部分代码

  ```java
  public static class Reduce extends Reducer<IntWritable,Text,IntWritable,Text>{  
    private static IntWritable result= new IntWritable();  
    //声明对象result  
    public void reduce(IntWritable key,Iterable<Text> values,Context context) throws IOException, InterruptedException{  
      for(Text val:values){  
        context.write(key,val);  
      }  
    }  
  }  
  ```

  reduce端接收到<key,value-list>之后，将输入的key直接复制给输出的key,用for循环遍历value-list并将里面的元素设置为输出的value，然后将<key,value>逐一输出，根据value-list中元素的个数决定输出的次数。

+ 完整代码

  ```java
  package mapreduce;  
  import java.io.IOException;  
  import org.apache.hadoop.conf.Configuration;  
  import org.apache.hadoop.fs.Path;  
  import org.apache.hadoop.io.IntWritable;  
  import org.apache.hadoop.io.Text;  
  import org.apache.hadoop.mapreduce.Job;  
  import org.apache.hadoop.mapreduce.Mapper;  
  import org.apache.hadoop.mapreduce.Reducer;  
  import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;  
  import org.apache.hadoop.mapreduce.lib.input.TextInputFormat;  
  import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;  
  import org.apache.hadoop.mapreduce.lib.output.TextOutputFormat;  
  public class OneSort {  
    public static class Map extends Mapper<Object , Text , IntWritable,Text >{  
      private static Text goods=new Text();  
      private static IntWritable num=new IntWritable();  
      public void map(Object key,Text value,Context context) throws IOException, InterruptedException{  
        String line=value.toString();  
        String arr[]=line.split("\t");  
        num.set(Integer.parseInt(arr[1]));  
        goods.set(arr[0]);  
        context.write(num,goods);  
      }  
    }  
    public static class Reduce extends Reducer< IntWritable, Text, IntWritable, Text>{  
      private static IntWritable result= new IntWritable();  
      public void reduce(IntWritable key,Iterable<Text> values,Context context) throws IOException, InterruptedException{  
        for(Text val:values){  
          context.write(key,val);  
        }  
      }  
    }  
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException{  
      Configuration conf=new Configuration();  
      Job job =new Job(conf,"OneSort");  
      job.setJarByClass(OneSort.class);  
      job.setMapperClass(Map.class);  
      job.setReducerClass(Reduce.class);  
      job.setOutputKeyClass(IntWritable.class);  
      job.setOutputValueClass(Text.class);  
      job.setInputFormatClass(TextInputFormat.class);  
      job.setOutputFormatClass(TextOutputFormat.class);  
      Path in=new Path("hdfs://localhost:9000/mymapreduce3/in/goods_visit1");  
      Path out=new Path("hdfs://localhost:9000/mymapreduce3/out");  
      FileInputFormat.addInputPath(job,in);  
      FileOutputFormat.setOutputPath(job,out);  
      System.exit(job.waitForCompletion(true) ? 0 : 1);  
  
    }  
  }
  ```

# 24. Mapreduce实例——二次排序-实操

## 24.1 相关知识

​	在Map阶段，使用job.setInputFormatClass定义的InputFormat将输入的数据集分割成小数据块splites，同时InputFormat提供一个RecordReader的实现。本实验中使用的是TextInputFormat，他提供的RecordReader会将文本的字节偏移量作为key，这一行的文本作为value。这就是自定义Map的输入是<LongWritable, Text>的原因。然后调用自定义Map的map方法，将一个个<LongWritable, Text>键值对输入给Map的map方法。注意输出应该符合自定义Map中定义的输出<IntPair, IntWritable>。最终是生成一个List<IntPair, IntWritable>。**在map阶段的最后，会先调用job.setPartitionerClass对这个List进行分区，每个分区映射到一个reducer**。**每个分区内又调用job.setSortComparatorClass设置的key比较函数类排序**。可以看到，这本身就是一个二次排序。 **如果没有通过job.setSortComparatorClass设置key比较函数类，则可以使用key实现的compareTo方法进行排序**。 在本实验中，就使用了IntPair实现的compareTo方法。

​	在Reduce阶段，**reducer接收到所有映射到这个reducer的map输出后，也是会调用job.setSortComparatorClass设置的key比较函数类对所有数据对排序**。然后开始构造一个key对应的value迭代器。这时就要用到**分组，使用job.setGroupingComparatorClass设置的分组函数类**。**只要这个比较器比较的两个key相同，他们就属于同一个组，它们的value放在一个value迭代器，而这个迭代器的key使用属于同一个组的所有key的第一个key**。最后就是进入Reducer的reduce方法，reduce方法的输入是所有的（key和它的value迭代器）。同样注意输入与输出的类型必须与自定义的Reducer中声明的一致。

## 24.2 编写思路

​	二次排序：在mapreduce中，所有的key是需要被比较和排序的，并且是二次，先根据partitioner，再根据大小。而本例中也是要比较两次。先按照第一字段排序，然后在第一字段相同时按照第二字段排序。根据这一点，我们可以构造一个复合类IntPair，他有两个字段，先利用分区对第一字段排序，再利用分区内的比较对第二字段排序。Java代码主要分为四部分：自定义key，自定义分区函数类，map部分，reduce部分。

+ 自定义key的代码：

  ```java
  public static class IntPair implements WritableComparable<IntPair>  
  {  
    int first;  //第一个成员变量  
    int second;  //第二个成员变量  
  
    public void set(int left, int right)  
    {  
      first = left;  
      second = right;  
    }  
    public int getFirst()  
    {  
      return first;  
    }  
    public int getSecond()  
    {  
      return second;  
    }  
    @Override  
    //反序列化，从流中的二进制转换成IntPair  
    public void readFields(DataInput in) throws IOException  
    {  
      // TODO Auto-generated method stub  
      first = in.readInt();  
      second = in.readInt();  
    }  
    @Override  
    //序列化，将IntPair转化成使用流传送的二进制  
    public void write(DataOutput out) throws IOException  
    {  
      // TODO Auto-generated method stub  
      out.writeInt(first);  
      out.writeInt(second);  
    }  
    @Override  
    //key的比较  
    public int compareTo(IntPair o)  
    {  
      // TODO Auto-generated method stub  
      if (first != o.first)  
      {  
        return first < o.first ? 1 : -1;  
      }  
      else if (second != o.second)  
      {  
        return second < o.second ? -1 : 1;  
      }  
      else  
      {  
        return 0;  
      }  
    }  
    @Override  
    public int hashCode()  
    {  
      return first * 157 + second;  
    }  
    @Override  
    public boolean equals(Object right)  
    {  
      if (right == null)  
        return false;  
      if (this == right)  
        return true;  
      if (right instanceof IntPair)  
      {  
        IntPair r = (IntPair) right;  
        return r.first == first && r.second == second;  
      }  
      else  
      {  
        return false;  
      }  
    }  
  }
  ```

  所有自定义的key应该实现接口WritableComparable，因为是可序列的并且可比较的，并重载方法。该类中包含以下几种方法：1.反序列化，从流中的二进制转换成IntPair 方法为public void readFields(DataInput in) throws IOException 2.序列化，将IntPair转化成使用流传送的二进制 方法为public void write(DataOutput out)3. key的比较 public int compareTo(IntPair o) 另外**新定义的类应该重写的两个方法 public int hashCode() 和public boolean equals(Object right)** 。

+ 分区函数类代码

  ```java
  public static class FirstPartitioner extends Partitioner<IntPair, IntWritable>  
  {  
    @Override  
    public int getPartition(IntPair key, IntWritable value,int numPartitions)  
    {  
      return Math.abs(key.getFirst() * 127) % numPartitions;  
    }  
  }  
  ```

  对key进行分区，根据自定义key中first乘以127取绝对值在对numPartions取余来进行分区。这主要是为**实现第一次排序**。

+ 分组函数类代码

  ```java
  public static class GroupingComparator extends WritableComparator  
  {  
    protected GroupingComparator()  
    {  
      super(IntPair.class, true);  
    }  
    @Override  
    //Compare two WritableComparables.  
    public int compare(WritableComparable w1, WritableComparable w2)  
    {  
      IntPair ip1 = (IntPair) w1;  
      IntPair ip2 = (IntPair) w2;  
      int l = ip1.getFirst();  
      int r = ip2.getFirst();  
      return l == r ? 0 : (l < r ? -1 : 1);  
    }  
  }
  ```

  分组函数类。**在reduce阶段，构造一个key对应的value迭代器的时候，只要first相同就属于同一个组，放在一个value迭代器。这是一个比较器，需要继承WritableComparator**。

+ Map代码：

  ```java
  public static class Map extends Mapper<LongWritable, Text, IntPair, IntWritable>  
  {  
    //自定义map  
    private final IntPair intkey = new IntPair();  
    private final IntWritable intvalue = new IntWritable();  
    public void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException  
    {  
      String line = value.toString();  
      StringTokenizer tokenizer = new StringTokenizer(line);  
      int left = 0;  
      int right = 0;  
      if (tokenizer.hasMoreTokens())  
      {  
        left = Integer.parseInt(tokenizer.nextToken());  
        if (tokenizer.hasMoreTokens())  
          right = Integer.parseInt(tokenizer.nextToken());  
        intkey.set(right, left);  
        intvalue.set(left);  
        context.write(intkey, intvalue);  
      }  
    }  
  }
  ```

  在map阶段，使用job.setInputFormatClass定义的InputFormat将输入的数据集分割成小数据块splites，同时InputFormat提供一个RecordReader的实现。**本例子中使用的是TextInputFormat，他提供的RecordReader会将文本的一行的行号作为key，这一行的文本作为value**。这就是自定义Map的输入是<LongWritable, Text>的原因。然后调用自定义Map的map方法，将一个个<LongWritable, Text>键值对输入给Map的map方法。注意输出应该符合自定义Map中定义的输出<IntPair, IntWritable>。最终是生成一个List<IntPair, IntWritable>。**在map阶段的最后，会先调用job.setPartitionerClass对这个List进行分区，每个分区映射到一个reducer**。**每个分区内又调用job.setSortComparatorClass设置的key比较函数类排序**。可以看到，这本身就是一个二次排序。**如果没有通过job.setSortComparatorClass设置key比较函数类，则使用key实现compareTo方法**。在本例子中，使用了IntPair实现compareTo方法

+ Reduce代码：

  ```java
  public static class Reduce extends Reducer<IntPair, IntWritable, Text, IntWritable>  
  {  
    private final Text left = new Text();  
    private static final Text SEPARATOR = new Text("------------------------------------------------");  
  
    public void reduce(IntPair key, Iterable<IntWritable> values,Context context) throws IOException, InterruptedException  
    {  
      context.write(SEPARATOR, null);  
      left.set(Integer.toString(key.getFirst()));  
      System.out.println(left);  
      for (IntWritable val : values)  
      {  
        context.write(left, val);  
        //System.out.println(val);  
      }  
    }  
  }
  ```

  在reduce阶段，**reducer接收到所有映射到这个reducer的map输出后，也是会调用job.setSortComparatorClass设置的key比较函数类对所有数据对排序**。**然后开始构造一个key对应的value迭代器**。这时就要用到**分组，使用job.setGroupingComparatorClass设置的分组函数类**。**只要这个比较器比较的两个key相同，他们就属于同一个组，它们的value放在一个value迭代器，而这个迭代器的key使用属于同一个组的所有key的第一个key**。最后就是进入Reducer的reduce方法，reduce方法的输入是所有的key和它的value迭代器。同样注意输入与输出的类型必须与自定义的Reducer中声明的一致。

+ 完整代码：

  ```java
  package mapreduce;  
  import java.io.DataInput;  
  import java.io.DataOutput;  
  import java.io.IOException;  
  import java.util.StringTokenizer;  
  import org.apache.hadoop.conf.Configuration;  
  import org.apache.hadoop.fs.Path;  
  import org.apache.hadoop.io.IntWritable;  
  import org.apache.hadoop.io.LongWritable;  
  import org.apache.hadoop.io.Text;  
  import org.apache.hadoop.io.WritableComparable;  
  import org.apache.hadoop.io.WritableComparator;  
  import org.apache.hadoop.mapreduce.Job;  
  import org.apache.hadoop.mapreduce.Mapper;  
  import org.apache.hadoop.mapreduce.Partitioner;  
  import org.apache.hadoop.mapreduce.Reducer;  
  import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;  
  import org.apache.hadoop.mapreduce.lib.input.TextInputFormat;  
  import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;  
  import org.apache.hadoop.mapreduce.lib.output.TextOutputFormat;  
  public class SecondarySort  
  {  
  
    public static class IntPair implements WritableComparable<IntPair>  
    {  
      int first;  
      int second;  
  
      public void set(int left, int right)  
      {  
        first = left;  
        second = right;  
      }  
      public int getFirst()  
      {  
        return first;  
      }  
      public int getSecond()  
      {  
        return second;  
      }  
      @Override  
  
      public void readFields(DataInput in) throws IOException  
      {  
        // TODO Auto-generated method stub  
        first = in.readInt();  
        second = in.readInt();  
      }  
      @Override  
  
      public void write(DataOutput out) throws IOException  
      {  
        // TODO Auto-generated method stub  
        out.writeInt(first);  
        out.writeInt(second);  
      }  
      @Override  
  
      public int compareTo(IntPair o)  
      {  
        // TODO Auto-generated method stub  
        if (first != o.first)  
        {  
          return first < o.first ? 1 : -1;  
        }  
        else if (second != o.second)  
        {  
          return second < o.second ? -1 : 1;  
        }  
        else  
        {  
          return 0;  
        }  
      }  
      @Override  
      public int hashCode()  
      {  
        return first * 157 + second;  
      }  
      @Override  
      public boolean equals(Object right)  
      {  
        if (right == null)  
          return false;  
        if (this == right)  
          return true;  
        if (right instanceof IntPair)  
        {  
          IntPair r = (IntPair) right;  
          return r.first == first && r.second == second;  
        }  
        else  
        {  
          return false;  
        }  
      }  
    }  
  
    public static class FirstPartitioner extends Partitioner<IntPair, IntWritable>  
    {  
      @Override  
      public int getPartition(IntPair key, IntWritable value,int numPartitions)  
      {  
        return Math.abs(key.getFirst() * 127) % numPartitions;  
      }  
    }  
    public static class GroupingComparator extends WritableComparator  
    {  
      protected GroupingComparator()  
      {  
        super(IntPair.class, true);  
      }  
      @Override  
      //Compare two WritableComparables.  
      public int compare(WritableComparable w1, WritableComparable w2)  
      {  
        IntPair ip1 = (IntPair) w1;  
        IntPair ip2 = (IntPair) w2;  
        int l = ip1.getFirst();  
        int r = ip2.getFirst();  
        return l == r ? 0 : (l < r ? -1 : 1);  
      }  
    }  
    public static class Map extends Mapper<LongWritable, Text, IntPair, IntWritable>  
    {  
      private final IntPair intkey = new IntPair();  
      private final IntWritable intvalue = new IntWritable();  
      public void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException  
      {  
        String line = value.toString();  
        StringTokenizer tokenizer = new StringTokenizer(line);  
        int left = 0;  
        int right = 0;  
        if (tokenizer.hasMoreTokens())  
        {  
          left = Integer.parseInt(tokenizer.nextToken());  
          if (tokenizer.hasMoreTokens())  
            right = Integer.parseInt(tokenizer.nextToken());  
          intkey.set(right, left);  
          intvalue.set(left);  
          context.write(intkey, intvalue);  
        }  
      }  
    }  
  
    public static class Reduce extends Reducer<IntPair, IntWritable, Text, IntWritable>  
    {  
      private final Text left = new Text();  
      private static final Text SEPARATOR = new Text("------------------------------------------------");  
  
      public void reduce(IntPair key, Iterable<IntWritable> values,Context context) throws IOException, InterruptedException  
      {  
        context.write(SEPARATOR, null);  
        left.set(Integer.toString(key.getFirst()));  
        System.out.println(left);  
        for (IntWritable val : values)  
        {  
          context.write(left, val);  
          //System.out.println(val);  
        }  
      }  
    }  
    public static void main(String[] args) throws IOException, InterruptedException, ClassNotFoundException  
    {  
  
      Configuration conf = new Configuration();  
      Job job = new Job(conf, "secondarysort");  
      job.setJarByClass(SecondarySort.class);  
      job.setMapperClass(Map.class);  
      job.setReducerClass(Reduce.class);  
      job.setPartitionerClass(FirstPartitioner.class);  
  
      job.setGroupingComparatorClass(GroupingComparator.class);  
      job.setMapOutputKeyClass(IntPair.class);  
  
      job.setMapOutputValueClass(IntWritable.class);  
  
      job.setOutputKeyClass(Text.class);  
  
      job.setOutputValueClass(IntWritable.class);  
  
      job.setInputFormatClass(TextInputFormat.class);  
  
      job.setOutputFormatClass(TextOutputFormat.class);  
      String[] otherArgs=new String[2];  
      otherArgs[0]="hdfs://localhost:9000/mymapreduce8/in/goods_visit2";  
      otherArgs[1]="hdfs://localhost:9000/mymapreduce8/out";  
  
      FileInputFormat.setInputPaths(job, new Path(otherArgs[0]));  
  
      FileOutputFormat.setOutputPath(job, new Path(otherArgs[1]));  
  
      System.exit(job.waitForCompletion(true) ? 0 : 1);  
    }  
  }
  ```

# 25. MapReduce排序—倒排序索引

## 25.1 倒排索引-原理

+ "倒排索引"是文档检索系统中最常用的数据结构，被广泛地应用于全文搜索引擎。
  + 它主要是用来存储某个单词（或词组）在一个文档或一组文档中的存储位置的映射，即提供了一种根据"内容来查找文档"的方式。由于不是根据"文档来确定文档所包含"的内容，而是进行相反的操作，因而被称为倒排索引（Inverted Index）
+ 实现"倒排索引"主要关注的信息为：单词、文档URL及词频

## 25.2 小结

倒排索引：它主要是用来存储某个单词（或词组）在一个文档或一组文档中的存储位置的映射，即提供了一种根据"内容来查找文档"的方式。

# 26. Mapreduce实例——倒排索引-实操

## 26.1 相关知识

​	"倒排索引"是文档检索系统中最常用的数据结构，被广泛地应用于全文搜索引擎。它主要是用来存储某个单词（或词组）在一个文档或一组文档中的存储位置的映射，即提供了一种根据内容来查找文档的方式。由于不是根据文档来确定文档所包含的内容，而是进行相反的操作，因而称为倒排索引（Inverted Index）。

​	**实现"倒排索引"主要关注的信息为：单词、文档URL及词频**。

​	下面以本实验goods3、goods_visit3、order_items3三张表的数据为例，根据MapReduce的处理过程给出倒排索引的设计思路：

（1）Map过程

首先使用默认的TextInputFormat类对输入文件进行处理，得到文本中每行的偏移量及其内容。显然，Map过程首先必须分析输入的<key,value>对，得到倒排索引中需要的三个信息：单词、文档URL和词频，接着我们对读入的数据利用Map操作进行预处理，如下图所示：

[![img](https://www.ipieuvre.com/doc/exper/1e88d94b-91ad-11e9-beeb-00215ec892f4/img/01.png)](https://www.ipieuvre.com/doc/exper/1e88d94b-91ad-11e9-beeb-00215ec892f4/img/01.png)

这里存在两个问题：**第一，<key,value>对只能有两个值，在不使用Hadoop自定义数据类型的情况下，需要根据情况将其中两个值合并成一个值，作为key或value值。第二，通过一个Reduce过程无法同时完成词频统计和生成文档列表，所以必须增加一个Combine过程完成词频统计**。

这里将商品ID和URL组成key值（如"1024600：goods3"），将词频（商品ID出现次数）作为value，这样做的好处是可以利用MapReduce框架自带的Map端排序，将同一文档的相同单词的词频组成列表，传递给Combine过程，实现类似于WordCount的功能。

（2）Combine过程

经过map方法处理后，Combine过程将key值相同的value值累加，得到一个单词在文档中的词频，如下图所示。如果直接将下图所示的输出作为Reduce过程的输入，在Shuffle过程时将面临一个问题：所有具有相同单词的记录（由单词、URL和词频组成）应该交由同一个Reducer处理，但当前的key值无法保证这一点，所以必须修改key值和value值。这次将单词（商品ID）作为key值，URL和词频组成value值（如"goods3：1"）。这样做的好处是可以利用MapReduce框架默认的HashPartitioner类完成Shuffle过程，将相同单词的所有记录发送给同一个Reducer进行处理。

[![img](https://www.ipieuvre.com/doc/exper/1e88d94b-91ad-11e9-beeb-00215ec892f4/img/01-1.png)](https://www.ipieuvre.com/doc/exper/1e88d94b-91ad-11e9-beeb-00215ec892f4/img/01-1.png)

（3）Reduce过程

经过上述两个过程后，Reduce过程只需将相同key值的所有value值组合成倒排索引文件所需的格式即可，剩下的事情就可以直接交给MapReduce框架进行处理了。如下图所示

[![img](https://www.ipieuvre.com/doc/exper/1e88d94b-91ad-11e9-beeb-00215ec892f4/img/01-2.png)](https://www.ipieuvre.com/doc/exper/1e88d94b-91ad-11e9-beeb-00215ec892f4/img/01-2.png)

## 26.2 编写思路

+ Map代码

  首先使用默认的TextInputFormat类对输入文件进行处理，得到文本中每行的偏移量及其内容。显然，Map过程首先必须分析输入的<key,value>对，得到倒排索引中需要的三个信息：单词、文档URL和词频，这里存在两个问题：第一，<key,value>对只能有两个值，在不使用Hadoop自定义数据类型的情况下，需要根据情况将其中两个值合并成一个值，作为key或value值。第二，通过一个Reduce过程无法同时完成词频统计和生成文档列表，所以必须增加一个Combine过程完成词频统计。

  ```java
  public static class doMapper extends Mapper<Object, Text, Text, Text>{  
    public static Text myKey = new Text();   // 存储单词和URL组合  
    public static Text myValue = new Text();  // 存储词频  
    //private FileSplit filePath;     // 存储Split对象  
  
    @Override   // 实现map函数  
    protected void map(Object key, Text value, Context context)  
      throws IOException, InterruptedException {  
      String filePath=((FileSplit)context.getInputSplit()).getPath().toString();  
      if(filePath.contains("goods")){  
        String val[]=value.toString().split("\t");  
        int splitIndex =filePath.indexOf("goods");  
        myKey.set(val[0] + ":" + filePath.substring(splitIndex));  
      }else if(filePath.contains("order")){  
        String val[]=value.toString().split("\t");  
        int splitIndex =filePath.indexOf("order");  
        myKey.set(val[2] + ":" + filePath.substring(splitIndex));  
      }  
      myValue.set("1");  
      context.write(myKey, myValue);  
    }  
  }
  ```

+ Combiner代码

  **经过map方法处理后，Combine过程将key值相同的value值累加，得到一个单词在文档中的词频**。如果直接将输出作为Reduce过程的输入，在Shuffle过程时将面临一个问题：所有具有相同单词的记录（由单词、URL和词频组成）应该交由同一个Reducer处理，但当前的key值无法保证这一点，所以必须修改key值和value值。这次将单词作为key值，URL和词频组成value值。这样做的好处是可以利用MapReduce框架默认的HashPartitioner类完成Shuffle过程，将相同单词的所有记录发送给同一个Reducer进行处理。

  ```java
  public static class doCombiner extends Reducer<Text, Text, Text, Text>{  
    public static Text myK = new Text();  
    public static Text myV = new Text();  
  
    @Override //实现reduce函数  
    protected void reduce(Text key, Iterable<Text> values, Context context)  
      throws IOException, InterruptedException {  
      // 统计词频  
      int sum = 0 ;  
      for (Text value : values) {  
        sum += Integer.parseInt(value.toString());  
      }  
      int mysplit = key.toString().indexOf(":");  
      // 重新设置value值由URL和词频组成  
      myK.set(key.toString().substring(0, mysplit));  
      myV.set(key.toString().substring(mysplit + 1) + ":" + sum);  
      context.write(myK, myV);  
    }  
  }  
  ```

+ Reduce代码

  经过上述两个过程后，Reduce过程只需将相同key值的value值组合成倒排索引文件所需的格式即可，剩下的事情就可以直接交给MapReduce框架进行处理了。

  ```java
  public static class doReducer extends Reducer<Text, Text, Text, Text>{  
  
    public static Text myK = new Text();  
    public static Text myV = new Text();  
  
    @Override     // 实现reduce函数  
    protected void reduce(Text key, Iterable<Text> values, Context context)  
      throws IOException, InterruptedException {  
      // 生成文档列表  
      String myList = new String();  
  
      for (Text value : values) {  
        myList += value.toString() + ";";  
      }  
      myK.set(key);  
      myV.set(myList);  
      context.write(myK, myV);  
    }  
  }
  ```

+ 完整代码

  ```java
  package mapreduce;  
  import java.io.IOException;  
  import org.apache.hadoop.fs.Path;  
  import org.apache.hadoop.io.Text;  
  import org.apache.hadoop.mapreduce.Job;  
  import org.apache.hadoop.mapreduce.Mapper;  
  import org.apache.hadoop.mapreduce.Reducer;  
  import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;  
  import org.apache.hadoop.mapreduce.lib.input.FileSplit;  
  import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;  
  public class MyIndex {  
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {  
      Job job = Job.getInstance();  
      job.setJobName("InversedIndexTest");  
      job.setJarByClass(MyIndex.class);  
  
      job.setMapperClass(doMapper.class);  
      job.setCombinerClass(doCombiner.class);  
      job.setReducerClass(doReducer.class);  
  
      job.setOutputKeyClass(Text.class);  
      job.setOutputValueClass(Text.class);  
  
      Path in1 = new Path("hdfs://localhost:9000/mymapreduce9/in/goods3");  
      Path in2 = new Path("hdfs://localhost:9000/mymapreduce9/in/goods_visit3");  
      Path in3 = new Path("hdfs://localhost:9000/mymapreduce9/in/order_items3");  
      Path out = new Path("hdfs://localhost:9000/mymapreduce9/out");  
  
      FileInputFormat.addInputPath(job, in1);  
      FileInputFormat.addInputPath(job, in2);  
      FileInputFormat.addInputPath(job, in3);  
      FileOutputFormat.setOutputPath(job, out);  
  
      System.exit(job.waitForCompletion(true) ? 0 : 1);  
    }  
  
    public static class doMapper extends Mapper<Object, Text, Text, Text>{  
      public static Text myKey = new Text();  
      public static Text myValue = new Text();  
      //private FileSplit filePath;  
  
      @Override  
      protected void map(Object key, Text value, Context context)  
        throws IOException, InterruptedException {  
        String filePath=((FileSplit)context.getInputSplit()).getPath().toString();  
        if(filePath.contains("goods")){  
          String val[]=value.toString().split("\t");  
          int splitIndex =filePath.indexOf("goods");  
          myKey.set(val[0] + ":" + filePath.substring(splitIndex));  
        }else if(filePath.contains("order")){  
          String val[]=value.toString().split("\t");  
          int splitIndex =filePath.indexOf("order");  
          myKey.set(val[2] + ":" + filePath.substring(splitIndex));  
        }  
        myValue.set("1");  
        context.write(myKey, myValue);  
      }  
    }  
    public static class doCombiner extends Reducer<Text, Text, Text, Text>{  
      public static Text myK = new Text();  
      public static Text myV = new Text();  
  
      @Override  
      protected void reduce(Text key, Iterable<Text> values, Context context)  
        throws IOException, InterruptedException {  
        int sum = 0 ;  
        for (Text value : values) {  
          sum += Integer.parseInt(value.toString());  
        }  
        int mysplit = key.toString().indexOf(":");  
        myK.set(key.toString().substring(0, mysplit));  
        myV.set(key.toString().substring(mysplit + 1) + ":" + sum);  
        context.write(myK, myV);  
      }  
    }  
  
    public static class doReducer extends Reducer<Text, Text, Text, Text>{  
  
      public static Text myK = new Text();  
      public static Text myV = new Text();  
  
      @Override  
      protected void reduce(Text key, Iterable<Text> values, Context context)  
        throws IOException, InterruptedException {  
  
        String myList = new String();  
  
        for (Text value : values) {  
          myList += value.toString() + ";";  
        }  
        myK.set(key);  
        myV.set(myList);  
        context.write(myK, myV);  
      }  
    }  
  }  
  ```

# 27. Mapreduce实例——单表join-实操

## 27.1 相关知识

> [区分笛卡儿积，自然连接，等值连接，内连接，外连接](https://baijiahao.baidu.com/s?id=1655935519271290347&wfr=spider&for=pc)	<=	回顾下数据库基础知识

​	以本实验的buyer1(buyer_id,friends_id)表为例来阐述单表连接的实验原理。单表连接，连接的是左表的buyer_id列和右表的friends_id列，且左表和右表是同一个表。

​	因此，在map阶段将读入数据分割成buyer_id和friends_id之后，会将buyer_id设置成key，friends_id设置成value，直接输出并将其作为左表；再将同一对buyer_id和friends_id中的friends_id设置成key，buyer_id设置成value进行输出，作为右表。

​	为了区分输出中的左右表，需要在输出的value中再加上左右表的信息，比如在value的String最开始处加上字符1表示左表，加上字符2表示右表。这样在map的结果中就形成了左表和右表，然后**在shuffle过程中完成连接**。

​	reduce接收到连接的结果，其中每个key的value-list就包含了"buyer_idfriends_id--friends_idbuyer_id"关系。取出每个key的value-list进行解析，将左表中的buyer_id放入一个数组，右表中的friends_id放入一个数组，然后**对两个数组求笛卡尔积**就是最后的结果了。

[![img](https://www.ipieuvre.com/doc/exper/1e693f7d-91ad-11e9-beeb-00215ec892f4/img/01.png)](https://www.ipieuvre.com/doc/exper/1e693f7d-91ad-11e9-beeb-00215ec892f4/img/01.png)

## 27.2 编写思路

+ Map代码

  ```java
  public static class Map extends Mapper<Object,Text,Text,Text>{  
    //实现map函数  
    public void map(Object key,Text value,Context context)  
      throws IOException,InterruptedException{  
      String line = value.toString();  
      String[] arr = line.split("\t");   //按行截取  
      String mapkey=arr[0];  
      String mapvalue=arr[1];  
      String relationtype=new String();  //左右表标识  
      relationtype="1";  //输出左表  
      context.write(new Text(mapkey),new Text(relationtype+"+"+mapvalue));  
      //System.out.println(relationtype+"+"+mapvalue);  
      relationtype="2";  //输出右表  
      context.write(new Text(mapvalue),new Text(relationtype+"+"+mapkey));  
      //System.out.println(relationtype+"+"+mapvalue);  
  
    }  
  } 
  ```

  Map处理的是一个纯文本文件，Mapper处理的数据是由InputFormat将数据集切分成小的数据集InputSplit，并用RecordReader解析成<key/value>对提供给map函数使用。map函数中用split("\t")方法把每行数据进行截取，并把数据存入到数组arr[]，把arr[0]赋值给mapkey，arr[1]赋值给mapvalue。用两个context的write()方法把数据输出两份，再通过标识符relationtype为1或2对两份输出数据的value打标记。

+ Reduce代码

  ```java
  public static class Reduce extends Reducer<Text, Text, Text, Text>{  
    //实现reduce函数  
    public void reduce(Text key,Iterable<Text> values,Context context)  
      throws IOException,InterruptedException{  
      int buyernum=0;  
      String[] buyer=new String[20];  
      int friendsnum=0;  
      String[] friends=new String[20];  
      Iterator ite=values.iterator();  
      while(ite.hasNext()){  
        String record=ite.next().toString();  
        int len=record.length();  
        int i=2;  
        if(0==len){  
          continue;  
        }  
        //取得左右表标识  
        char relationtype=record.charAt(0);  
        //取出record，放入buyer  
        if('1'==relationtype){  
          buyer [buyernum]=record.substring(i);  
          buyernum++;  
        }  
        //取出record，放入friends  
        if('2'==relationtype){  
          friends[friendsnum]=record.substring(i);  
          friendsnum++;  
        }  
      }  
      //buyernum和friendsnum数组求笛卡尔积  
      if(0!=buyernum&&0!=friendsnum){  
        for(int m=0;m<buyernum;m++){  
          for(int n=0;n<friendsnum;n++){  
            if(buyer[m]!=friends[n]){  
              //输出结果  
              context.write(new Text(buyer[m]),new Text(friends[n]));  
            }  
          }  
        }  
      }  
    }  
  }
  ```

  ​	reduce端在接收map端传来的数据时已经把相同key的所有value都放到一个Iterator容器中values。reduce函数中，首先新建两数组buyer[]和friends[]用来存放map端的两份输出数据。然后Iterator迭代中hasNext()和Next()方法加while循环遍历输出values的值并赋值给record，用charAt(0)方法获取record第一个字符赋值给relationtype，用if判断如果relationtype为1则把用substring(2)方法从下标为2开始截取record将其存放到buyer[]中，如果relationtype为2时将截取的数据放到frindes[]数组中。然后用三个for循环嵌套遍历输出<key,value>，其中key=buyer[m]，value=friends[n]。

+ 完整代码

  ```java
  package mapreduce;  
  import java.io.IOException;  
  import java.util.Iterator;  
  import org.apache.hadoop.conf.Configuration;  
  import org.apache.hadoop.fs.Path;  
  import org.apache.hadoop.io.Text;  
  import org.apache.hadoop.mapreduce.Job;  
  import org.apache.hadoop.mapreduce.Mapper;  
  import org.apache.hadoop.mapreduce.Reducer;  
  import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;  
  import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;  
  public class DanJoin {  
    public static class Map extends Mapper<Object,Text,Text,Text>{  
      public void map(Object key,Text value,Context context)  
        throws IOException,InterruptedException{  
        String line = value.toString();  
        String[] arr = line.split("\t");  
        String mapkey=arr[0];  
        String mapvalue=arr[1];  
        String relationtype=new String();  
        relationtype="1";  
        context.write(new Text(mapkey),new Text(relationtype+"+"+mapvalue));  
        //System.out.println(relationtype+"+"+mapvalue);  
        relationtype="2";  
        context.write(new Text(mapvalue),new Text(relationtype+"+"+mapkey));  
        //System.out.println(relationtype+"+"+mapvalue);  
      }  
    }  
    public static class Reduce extends Reducer<Text, Text, Text, Text>{  
      public void reduce(Text key,Iterable<Text> values,Context context)  
        throws IOException,InterruptedException{  
        int buyernum=0;  
        String[] buyer=new String[20];  
        int friendsnum=0;  
        String[] friends=new String[20];  
        Iterator ite=values.iterator();  
        while(ite.hasNext()){  
          String record=ite.next().toString();  
          int len=record.length();  
          int i=2;  
          if(0==len){  
            continue;  
          }  
          char relationtype=record.charAt(0);  
          if('1'==relationtype){  
            buyer [buyernum]=record.substring(i);  
            buyernum++;  
          }  
          if('2'==relationtype){  
            friends[friendsnum]=record.substring(i);  
            friendsnum++;  
          }  
        }  
        if(0!=buyernum&&0!=friendsnum){  
          for(int m=0;m<buyernum;m++){  
            for(int n=0;n<friendsnum;n++){  
              if(buyer[m]!=friends[n]){  
                context.write(new Text(buyer[m]),new Text(friends[n]));  
              }  
            }  
          }  
        }  
      }  
    }  
    public static void main(String[] args) throws Exception{  
  
      Configuration conf=new Configuration();  
      String[] otherArgs=new String[2];  
      otherArgs[0]="hdfs://localhost:9000/mymapreduce7/in/buyer1";  
      otherArgs[1]="hdfs://localhost:9000/mymapreduce7/out";  
      Job job=new Job(conf," Table join");  
      job.setJarByClass(DanJoin.class);  
      job.setMapperClass(Map.class);  
      job.setReducerClass(Reduce.class);  
      job.setOutputKeyClass(Text.class);  
      job.setOutputValueClass(Text.class);  
      FileInputFormat.addInputPath(job, new Path(otherArgs[0]));  
      FileOutputFormat.setOutputPath(job, new Path(otherArgs[1]));  
      System.exit(job.waitForCompletion(true)?0:1);  
  
    }  
  }  
  ```

# 28. MapReduce多表关联—Map端join

## 28.1 Map端join

​	MapReduce提供了表连接操作，其中包括：

+ Map端join
+ Reduce端join
+ 单表连接

​	现在讨论的是Map端join，Map端join是指数据达到map处理函数之前进行合并的，效率要远远高于Reduce端join，因为Reduce端join是把所有的数据都经过Shuffle，非常消耗资源

## 28.2 使用场景和原理

+ Map端join的使用场景：
  + 一张表数据十分小、一张表数据很大
+ 原理：
  + Map端join是针对以上场景进行的优化：将小表中的数据全部加在到内存，按**关键字**建立索引。大表中的数据作为map的输入，对map()函数每一对<key,value>输入，就能够方便地和已加载到内存的小数据进行连接。把连接结果按key输出，经过shuffile阶段，reduce端的到的就是已经按key分组并且连接好了的数据。

## 28.3 执行流程

1. 首先在提交作业的时候将小表文件放到该作业的DistributedCache中，然后从DistributedCache中取出该小表进行join连接的<key,value>键值对，将其解释分割放到内存中（可以放到HashMap等等容器中）
2. 要重写MyMapper类下面的setup()方法，因为这个方法是先于map方法执行的，将较小表先读入到一个HashMap中。
3. 重写map函数，一行行读入大表的内容，逐一与HashMap中的内容进行比较，若Key相同，则对数据进行格式化处理，然后直接输出。
4. map函数输出的<key,value>键值对首先经过一个shuffle把key值相同的所有value放到一个迭代器中形成values，然后将<key,values>键值对传递给reduce函数，reduce函数输入的key直接复制给输出的key，输入的values通过增强版for循环遍历逐一输出，循环的次数决定了<key,value>输出的次数

## 28.4 小结

​	Map端join：适用于一张大表和一张小表之间的连接。先将**小表的内容加载到缓存**中，在setup中读取内容，然后在map函数中处理连接，reduce中遍历输出

（注意：两张表中有公共字段）

# 29. Mapreduce实例——Map端join-实操

## 29.1 相关知识

​	**MapReduce提供了表连接操作其中包括Map端join、Reduce端join还有单表连接**。

​	现在我们要讨论的是Map端join，Map端join是指数据到达map处理函数之前进行合并的，效率要远远高于Reduce端join，因为Reduce端join是把所有的数据都经过Shuffle，非常消耗资源。

1. Map端join的使用场景：一张表数据十分小、一张表数据很大。

   ​	Map端join是针对以上场景进行的优化：**将小表中的数据全部加载到内存，按关键字建立索引**。大表中的数据作为map的输入，对map()函数每一对<key,value>输入，都能够方便地和已加载到内存的小数据进行连接。把连接结果按key输出，经过shuffle阶段，reduce端得到的就是已经按key分组并且连接好了的数据。

为了支持文件的复制，Hadoop提供了一个类DistributedCache，使用该类的方法如下：

（1）用户使用静态方法DistributedCache.addCacheFile()指定要复制的文件，它的参数是文件的URI（如果是HDFS上的文件，可以这样：hdfs://namenode:9000/home/XXX/file，其中9000是自己配置的NameNode端口号）。JobTracker在作业启动之前会获取这个URI列表，并将相应的文件拷贝到各个TaskTracker的本地磁盘上。

（2）用户使用DistributedCache.getLocalCacheFiles()方法获取文件目录，并使用标准的文件读写API读取相应的文件。

2.本实验Map端Join的执行流程

（1）首先在提交作业的时候先将小表文件放到该作业的DistributedCache中，然后从DistributeCache中取出该小表进行join连接的 <key ,value>键值对，将其解释分割放到内存中（可以放到HashMap等等容器中）。

（2）**要重写MyMapper类下面的setup()方法，因为这个方法是先于map方法执行的，将较小表先读入到一个HashMap中**。

（3）重写map函数，一行行读入大表的内容，逐一的与HashMap中的内容进行比较，若Key相同，则对数据进行格式化处理，然后直接输出。

（4）map函数输出的<key,value >键值对首先经过一个shuffle把key值相同的所有value放到一个迭代器中形成values，然后将<key,values>键值对传递给reduce函数，reduce函数输入的key直接复制给输出的key，输入的values通过增强版for循环遍历逐一输出，循环的次数决定了<key,value>输出的次数。

## 29.2 编写思路

​	Map端join适用于一个表记录数很少（100条），另一表记录数很多（像几亿条）的情况，我们把小表数据加载到内存中，然后扫描大表，看大表中记录的每条join key/value是否能在内存中找到相同的join key记录，如果有则输出结果。这样避免了一种数据倾斜问题。Mapreduce的Java代码分为两个部分：Mapper部分，Reduce部分。

+ Mapper代码

  ```java
  public static class MyMapper extends Mapper<Object, Text, Text, Text>{  
    private Map<String, String> dict = new HashMap<>();  
  
    @Override  
    protected void setup(Context context) throws IOException,  
    InterruptedException {  
      String fileName = context.getLocalCacheFiles()[0].getName();  
      System.out.println(fileName);  
      BufferedReader reader = new BufferedReader(new FileReader(fileName));  
      String codeandname = null;  
      while (null != ( codeandname = reader.readLine() ) ) {  
        String str[]=codeandname.split("\t");  
        dict.put(str[0], str[2]+"\t"+str[3]);  
      }  
      reader.close();  
    }  
    @Override  
    protected void map(Object key, Text value, Context context)  
      throws IOException, InterruptedException {  
      String[] kv = value.toString().split("\t");  
      if (dict.containsKey(kv[1])) {  
        context.write(new Text(kv[1]), new Text(dict.get(kv[1])+"\t"+kv[2]));  
      }  
    }  
  }
  ```

  该部分分为setup方法与map方法。在setup方法中首先用getName()获取当前文件名为orders1的文件并赋值给fileName，然后用bufferedReader读取内存中缓存文件。在读文件时用readLine()方法读取每行记录，把该记录用split("\t")方法截取，与order_items文件中相同的字段str[0]作为key值放到map集合dict中，选取所要展现的字段作为value。map函数接收order_items文件数据，并用split("\t")截取数据存放到数组kv[]中（其中kv[1]与str[0]代表的字段相同），用if判断，如果内存中dict集合的key值包含kv[1],则用context的write()方法输出key2/value2值，其中kv[1]作为key2,其他dict.get(kv[1])+"\t"+kv[2]作为value2。

+ Reduce代码

  ```java
  public static class MyReducer extends Reducer<Text, Text, Text, Text>{  
    @Override  
    protected void reduce(Text key, Iterable<Text> values, Context context)  
      throws IOException, InterruptedException {  
      for (Text text : values) {  
        context.write(key, text);  
      }  
    }  
  }  
  ```

  map函数输出的<key,value >键值对首先经过一个suffle把key值相同的所有value放到一个迭代器中形成values，然后将<key,values>键值对传递给reduce函数，reduce函数输入的key直接复制给输出的key，输入的values通过增强版for循环遍历逐一输出。

+ 完整代码

  ```java
  package mapreduce;  
  import java.io.BufferedReader;  
  import java.io.FileReader;  
  import java.io.IOException;  
  import java.net.URI;  
  import java.net.URISyntaxException;  
  import java.util.HashMap;  
  import java.util.Map;  
  import org.apache.hadoop.fs.Path;  
  import org.apache.hadoop.io.Text;  
  import org.apache.hadoop.mapreduce.Job;  
  import org.apache.hadoop.mapreduce.Mapper;  
  import org.apache.hadoop.mapreduce.Reducer;  
  import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;  
  import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;  
  public class MapJoin {  
  
    public static class MyMapper extends Mapper<Object, Text, Text, Text>{  
      private Map<String, String> dict = new HashMap<>();  
  
      @Override  
      protected void setup(Context context) throws IOException,  
      InterruptedException {  
        String fileName = context.getLocalCacheFiles()[0].getName();  
        //System.out.println(fileName);  
        BufferedReader reader = new BufferedReader(new FileReader(fileName));  
        String codeandname = null;  
        while (null != ( codeandname = reader.readLine() ) ) {  
          String str[]=codeandname.split("\t");  
          dict.put(str[0], str[2]+"\t"+str[3]);  
        }  
        reader.close();  
      }  
      @Override  
      protected void map(Object key, Text value, Context context)  
        throws IOException, InterruptedException {  
        String[] kv = value.toString().split("\t");  
        if (dict.containsKey(kv[1])) {  
          context.write(new Text(kv[1]), new Text(dict.get(kv[1])+"\t"+kv[2]));  
        }  
      }  
    }  
    public static class MyReducer extends Reducer<Text, Text, Text, Text>{  
      @Override  
      protected void reduce(Text key, Iterable<Text> values, Context context)  
        throws IOException, InterruptedException {  
        for (Text text : values) {  
          context.write(key, text);  
        }  
      }  
    }  
  
    public static void main(String[] args) throws ClassNotFoundException, IOException, InterruptedException, URISyntaxException {  
      Job job = Job.getInstance();  
      job.setJobName("mapjoin");  
      job.setJarByClass(MapJoin.class);  
  
      job.setMapperClass(MyMapper.class);  
      job.setReducerClass(MyReducer.class);  
  
      job.setOutputKeyClass(Text.class);  
      job.setOutputValueClass(Text.class);  
  
      Path in = new Path("hdfs://localhost:9000/mymapreduce5/in/order_items1");  
      Path out = new Path("hdfs://localhost:9000/mymapreduce5/out");  
      FileInputFormat.addInputPath(job, in);  
      FileOutputFormat.setOutputPath(job, out);  
  
      URI uri = new URI("hdfs://localhost:9000/mymapreduce5/in/orders1");  
      job.addCacheFile(uri);  
  
      System.exit(job.waitForCompletion(true) ? 0 : 1);  
    }  
  }
  ```

# 30. MapReduce多表关联—Reduce端join

## 30.1 Reduce端join-原理

+ 在Reduce端进行join连接是MapReduce框架进行表之间join操作最为常见的模式。
+ Reduce端join实现原理
  + Map端的主要工作，为来自不同表（文件）的key/value**对打标签以区别不同来源**的记录。然后用连接字段作为key，其余部分和新加的标志作为value，最后进行输出。
  + Reduce端的主要工作，在Reduce端义连接字段作为key的分组已经完成，我们只需要在每一个分组当中将那些来源于不同文件的记录（在map阶段已经打标志）分开，最后进行笛卡尔积就ok了。
+ Reduce端join的使用场景
  + Reduce端连接比Map端连接更为普遍，因为在map阶段不能获取所有需要的join字段，即：同一个key对应的字段可能位于不同map中，但是Reduce端连接效率比较低，因为所有数据都必须经过Shuffle过程。

## 30.2 小结

Reduce端join：

​	Map端读取所有的文件，并在输出的内容里加上标识，代表数据是从哪个文件里来的。Reduce处理函数中，按照标识对数据进行处理。然后将相同的key值进行join连接操作，求出结果并直接输出。

# 31. Mapreduce实例——Reduce端join-实操

## 31.1 相关知识

在Reudce端进行Join连接是MapReduce框架进行表之间Join操作最为常见的模式。

1.Reduce端Join实现原理

（1）Map端的主要工作，为来自不同表（文件）的key/value对打标签以区别不同来源的记录。然后用连接字段作为key，其余部分和新加的标志作为value，最后进行输出。

（2）Reduce端的主要工作，在Reduce端以连接字段作为key的分组已经完成，我们只需要在每一个分组当中将那些来源于不同文件的记录（在map阶段已经打标志）分开，最后进行笛卡尔只就ok了。

2.Reduce端Join的使用场景

​	**Reduce端连接比Map端连接更为普遍，因为在map阶段不能获取所有需要的join字段，即：同一个key对应的字段可能位于不同map中，但是Reduce端连接效率比较低，因为所有数据都必须经过Shuffle过程。**

3.本实验的Reduce端Join代码执行流程：

（1）Map端读取所有的文件，并在输出的内容里加上标识，代表数据是从哪个文件里来的。

（2）在Reduce处理函数中，按照标识对数据进行处理。

（3）然后将相同的key值进行Join连接操作，求出结果并直接输出。

## 31.2 编写思路

> [SQL之in和exit区别篇](https://blog.csdn.net/qq_36561697/article/details/80713824)	<=	回顾一下
>
> [in与exists的取舍](https://blog.csdn.net/dreamwbt/article/details/53363497)	<=	回顾一下
>
> 一般而言，外循环的数量级小的，速度更快，因为外层复杂度N，但是内层走索引的话就能缩小到logM
>
> A join B也是笛卡尔积，最后保留指定字段相同的结果而已（A内循环，B外循环）
>
> A in B，先计算B，然后笛卡尔积，（A内循环，B外循环）
>
> A exist B，先计算A，然后笛卡尔积（B内循环，A外循环）
>
> not in内外表都不会用到索引，而not exists能用到索引，所以后者任何情况都比前者好

(1)Map端读取所有的文件，并在输出的内容里加上标识，代表数据是从哪个文件里来的。

(2)在reduce处理函数中，按照标识对数据进行处理。

(3)然后将相同key值进行join连接操作，求出结果并直接输出。

Mapreduce中join连接分为Map端Join与Reduce端Join，这里是一个Reduce端Join连接。程序主要包括两部分：Map部分和Reduce部分。

+ Map代码

  ```java
  public static class mymapper extends Mapper<Object, Text, Text, Text>{  
    @Override  
    protected void map(Object key, Text value, Context context)  
      throws IOException, InterruptedException {  
      String filePath = ((FileSplit)context.getInputSplit()).getPath().toString();  
      if (filePath.contains("orders1")) {  
        //获取行文本内容  
        String line = value.toString();  
        //对行文本内容进行切分  
        String[] arr = line.split("\t");  
        ///把结果写出去  
        context.write(new Text(arr[0]), new Text( "1+" + arr[2]+"\t"+arr[3]));  
        System.out.println(arr[0] + "_1+" + arr[2]+"\t"+arr[3]);  
      }else if(filePath.contains("order_items1")) {  
        String line = value.toString();  
        String[] arr = line.split("\t");  
        context.write(new Text(arr[1]), new Text("2+" + arr[2]));  
        System.out.println(arr[1] + "_2+" + arr[2]);  
      }  
    }  
  }  
  ```

  Map处理的是一个纯文本文件，Mapper处理的数据是由InputFormat将数据集切分成小的数据集InputSplit，并用RecordReader解析成<key,value>对提供给map函数使用。在map函数中，首先用getPath()方法获取分片InputSplit的路径并赋值给filePath，if判断filePath中如果包含goods.txt文件名，则将map函数输入的value值通过Split("\t")方法进行切分，与goods_visit文件里相同的商品id字段作为key，其他字段前加"1+"作为value。如果if判断filePath包含goods_visit.txt文件名，步骤与上面相同，只是把其他字段前加"2+"作为value。最后把<key,value>通过Context的write方法输出。

+ Reduce代码

  ```java
  public static class myreducer extends Reducer<Text, Text, Text, Text>{  
    @Override  
    protected void reduce(Text key, Iterable<Text> values, Context context)  
      throws IOException, InterruptedException {  
      Vector<String> left  = new Vector<String>();  //用来存放左表的数据  
      Vector<String> right = new Vector<String>();  //用来存放右表的数据  
      //迭代集合数据  
      for (Text val : values) {  
        String str = val.toString();  
        //将集合中的数据添加到对应的left和right中  
        if (str.startsWith("1+")) {  
          left.add(str.substring(2));  
        }  
        else if (str.startsWith("2+")) {  
          right.add(str.substring(2));  
        }  
      }  
      //获取left和right集合的长度  
      int sizeL = left.size();  
      int sizeR = right.size();  
      //System.out.println(key + "left:"+left);  
      //System.out.println(key + "right:"+right);  
      //遍历两个向量将结果写进去  
      for (int i = 0; i < sizeL; i++) {  
        for (int j = 0; j < sizeR; j++) {  
          context.write( key, new Text(  left.get(i) + "\t" + right.get(j) ) );  
          //System.out.println(key + " \t" + left.get(i) + "\t" + right.get(j));  
        }  
      }  
    }  
  } 
  ```

  map函数输出的<key,value>经过shuffle将key相同的所有value放到一个迭代器中形成values，然后将<key,values>键值对传递给reduce函数。reduce函数中，首先新建两个Vector集合，用于存放输入的values中以"1+"开头和"2+"开头的数据。然后用增强版for循环遍历并嵌套if判断，若判断values里的元素以1+开头，则通过substring(2)方法切分元素，结果存放到left集合中，若values里元素以2+开头，则仍利用substring(2)方法切分元素，结果存放到right集合中。最后再用两个嵌套for循环，遍历输出<key,value>，其中输入的key直接赋值给输出的key，输出的value为left +"\t"+right。

+ 完整代码

  ```java
  package mapreduce;  
  import java.io.IOException;  
  import java.util.Iterator;  
  import java.util.Vector;  
  import org.apache.hadoop.fs.Path;  
  import org.apache.hadoop.io.Text;  
  import org.apache.hadoop.mapreduce.Job;  
  import org.apache.hadoop.mapreduce.Mapper;  
  import org.apache.hadoop.mapreduce.Reducer;  
  import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;  
  import org.apache.hadoop.mapreduce.lib.input.FileSplit;  
  import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;  
  public class ReduceJoin {  
    public static class mymapper extends Mapper<Object, Text, Text, Text>{  
      @Override  
      protected void map(Object key, Text value, Context context)  
        throws IOException, InterruptedException {  
        String filePath = ((FileSplit)context.getInputSplit()).getPath().toString();  
        if (filePath.contains("orders1")) {  
          String line = value.toString();  
          String[] arr = line.split("\t");  
          context.write(new Text(arr[0]), new Text( "1+" + arr[2]+"\t"+arr[3]));  
          //System.out.println(arr[0] + "_1+" + arr[2]+"\t"+arr[3]);  
        }else if(filePath.contains("order_items1")) {  
          String line = value.toString();  
          String[] arr = line.split("\t");  
          context.write(new Text(arr[1]), new Text("2+" + arr[2]));  
          //System.out.println(arr[1] + "_2+" + arr[2]);  
        }  
      }  
    }  
  
    public static class myreducer extends Reducer<Text, Text, Text, Text>{  
      @Override  
      protected void reduce(Text key, Iterable<Text> values, Context context)  
        throws IOException, InterruptedException {  
        Vector<String> left  = new Vector<String>();  
        Vector<String> right = new Vector<String>();  
        for (Text val : values) {  
          String str = val.toString();  
          if (str.startsWith("1+")) {  
            left.add(str.substring(2));  
          }  
          else if (str.startsWith("2+")) {  
            right.add(str.substring(2));  
          }  
        }  
  
        int sizeL = left.size();  
        int sizeR = right.size();  
        //System.out.println(key + "left:"+left);  
        //System.out.println(key + "right:"+right);  
        for (int i = 0; i < sizeL; i++) {  
          for (int j = 0; j < sizeR; j++) {  
            context.write( key, new Text(  left.get(i) + "\t" + right.get(j) ) );  
            //System.out.println(key + " \t" + left.get(i) + "\t" + right.get(j));  
          }  
        }  
      }  
    }  
  
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {  
      Job job = Job.getInstance();  
      job.setJobName("reducejoin");  
      job.setJarByClass(ReduceJoin.class);  
  
      job.setMapperClass(mymapper.class);  
      job.setReducerClass(myreducer.class);  
  
      job.setOutputKeyClass(Text.class);  
      job.setOutputValueClass(Text.class);  
  
      Path left = new Path("hdfs://localhost:9000/mymapreduce6/in/orders1");  
      Path right = new Path("hdfs://localhost:9000/mymapreduce6/in/order_items1");  
      Path out = new Path("hdfs://localhost:9000/mymapreduce6/out");  
  
      FileInputFormat.addInputPath(job, left);  
      FileInputFormat.addInputPath(job, right);  
      FileOutputFormat.setOutputPath(job, out);  
  
      System.exit(job.waitForCompletion(true) ? 0 : 1);  
    }  
  } 
  ```

# 32. Mapreduce实例——ChainMapReduce-实操

## 32.1 相关知识

​	一些复杂的任务难以用一次MapReduce处理完成，需要多次MapReduce才能完成任务。

​	**Hadoop2.0开始MapReduce作业支持链式处理**，类似于工厂的生产线，每一个阶段都有特定的任务要处理，比如提供原配件——>组装——>打印出厂日期，等等。通过这样进一步的分工，从而提高了生成效率，我们Hadoop中的链式MapReduce也是如此，这些Mapper可以像水流一样，一级一级向后处理，有点类似于Linux的管道。前一个Mapper的输出结果直接可以作为下一个Mapper的输入，形成一个流水线。

​	**链式MapReduce的执行规则：<big>整个Job中只能有一个Reducer</big>，在Reducer前面可以有一个或者多个Mapper，在Reducer的后面可以有0个或者多个Mapper。**

​	Hadoop2.0支持的链式处理MapReduce作业有以下三种：

（1）**顺序链接MapReduce作业**

​	类似于Unix中的管道：mapreduce-1 | mapreduce-2 | mapreduce-3 ......，每一个阶段创建一个job，并将当前输入路径设为前一个的输出。**在最后阶段删除链上生成的中间数据**。

（2）**具有复杂依赖的MapReduce链接**

​	若mapreduce-1处理一个数据集， mapreduce-2 处理另一个数据集，而mapreduce-3对前两个做内部链接。这种情况通过Job和JobControl类管理非线性作业间的依赖。如x.addDependingJob(y)意味着x在y完成前不会启动。

（3）**预处理和后处理的链接**

​	**一般将预处理和后处理写为Mapper任务**。可以自己进行链接或使用ChainMapper和ChainReducer类，生成作业表达式类似于：

​	MAP+ | REDUCE | MAP*

​	如以下作业： Map1 | Map2 | Reduce | Map3 | Map4，把Map2和Reduce视为MapReduce作业核心。Map1作为前处理，Map3， Map4作为后处理。

​	**ChainMapper使用模式：预处理作业，ChainReducer使用模式：设置Reducer并添加后处理Mapper**

​	本实验中用到的就是第三种作业模式：预处理和后处理的链接，生成作业表达式类似于 Map1 | Map2 | Reduce | Map3

## 32.2 编写思路

​	mapreduce执行的大体流程如下图所示：

[![img](https://www.ipieuvre.com/doc/exper/1e95e4b9-91ad-11e9-beeb-00215ec892f4/img/12.png)](https://www.ipieuvre.com/doc/exper/1e95e4b9-91ad-11e9-beeb-00215ec892f4/img/12.png)

​	由上图可知，ChainMapReduce的执行流程为：

​	①首先将文本文件中的数据通过InputFormat实例切割成多个小数据集InputSplit，然后通过RecordReader实例将小数据集InputSplit解析为<key,value>的键值对并提交给Mapper1；

​	②Mapper1里的map函数将输入的value进行切割，把商品名字段作为key值，点击数量字段作为value值，筛选出value值小于等于600的<key,value>，将<key,value>输出给Mapper2；

​	③Mapper2里的map函数再筛选出value值小于100的<key,value>，并将<key,value>输出；

​	④Mapper2输出的<key,value>键值对先经过shuffle，将key值相同的所有value放到一个集合，形成<key,value-list>，然后将所有的<key,value-list>输入给Reducer；

​	⑤Reducer里的reduce函数将value-list集合中的元素进行累加求和作为新的value，并将<key,value>输出给Mapper3；

​	⑥Mapper3里的map函数筛选出key值小于3个字符的<key,value>，并将<key,value>以文本的格式输出到hdfs上。该ChainMapReduce的Java代码主要分为四个部分，分别为：FilterMapper1，FilterMapper2，SumReducer，FilterMapper3。

+ FilterMapper1代码

  ```java
  public static class FilterMapper1 extends Mapper<LongWritable, Text, Text, DoubleWritable> {  
    private Text outKey = new Text();    //声明对象outKey  
    private DoubleWritable outValue = new DoubleWritable();    //声明对象outValue  
    @Override  
    protected void map(LongWritable key, Text value, Mapper<LongWritable, Text, Text, DoubleWritable>.Context context)  
      throws IOException,InterruptedException {  
      String line = value.toString();  
      if (line.length() > 0) {  
        String[] splits = line.split("\t");  //按行对内容进行切分  
        double visit = Double.parseDouble(splits[1].trim());  
        if (visit <= 600) {    //if循环，判断visit是否小于等于600  
          outKey.set(splits[0]);  
          outValue.set(visit);  
          context.write(outKey, outValue);  //调用context的write方法  
        }  
      }  
    }  
  } 
  ```

  ​	首先定义输出的key和value的类型，然后在map方法中获取文本行内容，用Split("\t")对行内容进行切分，把包含点击量的字段转换成double类型并赋值给visit，用if判断，如果visit小于等于600，则设置商品名称字段作为key,设置该visit作为value，用context的write方法输出<key,value>。

+ FilterMapper2代码

  ```java
  
  public static class FilterMapper2 extends Mapper<Text, DoubleWritable, Text, DoubleWritable> {  
    @Override  
    protected void map(Text key, DoubleWritable value, Mapper<Text, DoubleWritable, Text, DoubleWritable>.Context context)  
      throws IOException,InterruptedException {  
      if (value.get() < 100) {  
        context.write(key, value);  
      }  
    }  
  } 
  ```

  ​	接收mapper1传来的数据，通过value.get()获取输入的value值，再用if判断如果输入的value值小于100，则直接将输入的key赋值给输出的key，输入的value赋值给输出的value，输出<key,value>。

+ SumReducer代码

  ```java
  public  static class SumReducer extends Reducer<Text, DoubleWritable, Text, DoubleWritable> {  
    private DoubleWritable outValue = new DoubleWritable();  
    @Override  
    protected void reduce(Text key, Iterable<DoubleWritable> values, Reducer<Text, DoubleWritable, Text, DoubleWritable>.Context context)  
      throws IOException, InterruptedException {  
      double sum = 0;  
      for (DoubleWritable val : values) {  
        sum += val.get();  
      }  
      outValue.set(sum);  
      context.write(key, outValue);  
    }  
  }  
  ```

  ​	FilterMapper2输出的<key,value>键值对先经过shuffle，将key值相同的所有value放到一个集合，形成<key,value-list>，然后将所有的<key,value-list>输入给SumReducer。在reduce函数中，用增强版for循环遍历value-list中元素，将其数值进行累加并赋值给sum，然后用outValue.set(sum)方法把sum的类型转变为DoubleWritable类型并将sum设置为输出的value，将输入的key赋值给输出的key，最后用context的write()方法输出<key,value>。

+ FilterMapper3代码

  ```java
  public  static class FilterMapper3 extends Mapper<Text, DoubleWritable, Text, DoubleWritable> {  
    @Override  
    protected void map(Text key, DoubleWritable value, Mapper<Text, DoubleWritable, Text, DoubleWritable>.Context context)  
      throws IOException, InterruptedException {  
      if (key.toString().length() < 3) {  //for循环，判断key值是否大于3  
        System.out.println("写出去的内容为：" + key.toString() +"++++"+ value.toString());  
        context.write(key, value);  
      }  
    }  
  }  
  ```

  ​	接收reduce传来的数据，通过key.toString().length()获取key值的字符长度，再用if判断如果key值的字符长度小于3，则直接将输入的key赋值给输出的key，输入的value赋值给输出的value，输出<key，value>。

+ 完整代码

  ```java
  package mapreduce;  
  import java.io.IOException;  
  import java.net.URI;  
  import org.apache.hadoop.conf.Configuration;  
  import org.apache.hadoop.fs.Path;  
  import org.apache.hadoop.io.LongWritable;  
  import org.apache.hadoop.io.Text;  
  import org.apache.hadoop.mapreduce.Job;  
  import org.apache.hadoop.mapreduce.Mapper;  
  import org.apache.hadoop.mapreduce.Reducer;  
  import org.apache.hadoop.mapreduce.lib.chain.ChainMapper;  
  import org.apache.hadoop.mapreduce.lib.chain.ChainReducer;  
  import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;  
  import org.apache.hadoop.mapreduce.lib.input.TextInputFormat;  
  import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;  
  import org.apache.hadoop.mapreduce.lib.output.TextOutputFormat;  
  import org.apache.hadoop.mapreduce.lib.partition.HashPartitioner;  
  import org.apache.hadoop.fs.FileSystem;  
  import org.apache.hadoop.io.DoubleWritable;  
  public class ChainMapReduce {  
    private static final String INPUTPATH = "hdfs://localhost:9000/mymapreduce10/in/goods_0";  
    private static final String OUTPUTPATH = "hdfs://localhost:9000/mymapreduce10/out";  
    public static void main(String[] args) {  
      try {  
        Configuration conf = new Configuration();  
        FileSystem fileSystem = FileSystem.get(new URI(OUTPUTPATH), conf);  
        if (fileSystem.exists(new Path(OUTPUTPATH))) {  
          fileSystem.delete(new Path(OUTPUTPATH), true);  
        }  
        Job job = new Job(conf, ChainMapReduce.class.getSimpleName());  
        FileInputFormat.addInputPath(job, new Path(INPUTPATH));  
        job.setInputFormatClass(TextInputFormat.class);  
        ChainMapper.addMapper(job, FilterMapper1.class, LongWritable.class, Text.class, Text.class, DoubleWritable.class, conf);  
        ChainMapper.addMapper(job, FilterMapper2.class, Text.class, DoubleWritable.class, Text.class, DoubleWritable.class, conf);  
        ChainReducer.setReducer(job, SumReducer.class, Text.class, DoubleWritable.class, Text.class, DoubleWritable.class, conf);  
        ChainReducer.addMapper(job, FilterMapper3.class, Text.class, DoubleWritable.class, Text.class, DoubleWritable.class, conf);  
        job.setMapOutputKeyClass(Text.class);  
        job.setMapOutputValueClass(DoubleWritable.class);  
        job.setPartitionerClass(HashPartitioner.class);  
        job.setNumReduceTasks(1);  
        job.setOutputKeyClass(Text.class);  
        job.setOutputValueClass(DoubleWritable.class);  
        FileOutputFormat.setOutputPath(job, new Path(OUTPUTPATH));  
        job.setOutputFormatClass(TextOutputFormat.class);  
        System.exit(job.waitForCompletion(true) ? 0 : 1);  
      } catch (Exception e) {  
        e.printStackTrace();  
      }  
    }  
    public static class FilterMapper1 extends Mapper<LongWritable, Text, Text, DoubleWritable> {  
      private Text outKey = new Text();  
      private DoubleWritable outValue = new DoubleWritable();  
      @Override  
      protected void map(LongWritable key, Text value, Mapper<LongWritable, Text, Text, DoubleWritable>.Context context)  
        throws IOException,InterruptedException {  
        String line = value.toString();  
        if (line.length() > 0) {  
          String[] splits = line.split("\t");  
          double visit = Double.parseDouble(splits[1].trim());  
          if (visit <= 600) {  
            outKey.set(splits[0]);  
            outValue.set(visit);  
            context.write(outKey, outValue);  
          }  
        }  
      }  
    }  
    public static class FilterMapper2 extends Mapper<Text, DoubleWritable, Text, DoubleWritable> {  
      @Override  
      protected void map(Text key, DoubleWritable value, Mapper<Text, DoubleWritable, Text, DoubleWritable>.Context context)  
        throws IOException,InterruptedException {  
        if (value.get() < 100) {  
          context.write(key, value);  
        }  
      }  
    }  
    public  static class SumReducer extends Reducer<Text, DoubleWritable, Text, DoubleWritable> {  
      private DoubleWritable outValue = new DoubleWritable();  
      @Override  
      protected void reduce(Text key, Iterable<DoubleWritable> values, Reducer<Text, DoubleWritable, Text, DoubleWritable>.Context context)  
        throws IOException, InterruptedException {  
        double sum = 0;  
        for (DoubleWritable val : values) {  
          sum += val.get();  
        }  
        outValue.set(sum);  
        context.write(key, outValue);  
      }  
    }  
    public  static class FilterMapper3 extends Mapper<Text, DoubleWritable, Text, DoubleWritable> {  
      @Override  
      protected void map(Text key, DoubleWritable value, Mapper<Text, DoubleWritable, Text, DoubleWritable>.Context context)  
        throws IOException, InterruptedException {  
        if (key.toString().length() < 3) {  
          System.out.println("写出去的内容为：" + key.toString() +"++++"+ value.toString());  
          context.write(key, value);  
        }  
      }  
  
    }  
  
  }
  ```

# 33. MapReduce实战PageRank算法-实操

## 33.1 相关知识

​	PageRank：网页排名，右脚网页级别。是以Google 公司创始人Larry Page 之姓来命名。PageRank 计算每一个网页的PageRank值，并根据PageRank值的大小对网页的重要性进行排序。

PageRank的基本思想：

**1.如果一个网页被很多其他网页链接到的话说明这个网页比较重要，也就是PageRank值会相对较高**

**2.如果一个PageRank值很高的网页链接到一个其他的网页，那么被链接到的网页的PageRank值会相应地因此而提高。**

PageRank的算法原理：

PageRank算法总的来说就是预先给每个网页一个PR值，由于PR值物理意义上为一个网页被访问概率，所以一般是1/N，其中N为网页总数。另外，一般情况下，所有网页的PR值的总和为1。如果不为1的话也不是不行，最后算出来的不同网页之间PR值的大小关系仍然是正确的，只是不能直接地反映概率了。

[![img](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/01.png)](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/01.png)

如图，假设现在有四张网页，对于页面A来说，它链接到页面B，C，D，即A有3个出链，则它跳转到每个出链B，C，D的概率均为1/3.。如果A有k个出链，跳转到每个出链的概率为1/k。同理B到A，C，D的概率为1/2，0，1/2。C到A，B，D的概率为1，0，0。D到A，B，C的概率为0，1/2，1/2。

转化为矩阵为：

[![img](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/02.jpg)](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/02.jpg)

​	在上图中，第一列为页面A对各个页面转移的概率，第一行为各个页面对页面A转移的概率。

​	**初始时，每一个页面的PageRank值都是均等的，为1/N，这里也即是1/4**。

​	然后对于页面A来说，根据每一个页面的PageRank值和每个页面对页面A的转移概率，可以算出新一轮页面A的PageRank值。这里，只有页面B转移了自己的1/2给A。页面C转移了自己的全部给A，所以新一轮A的PageRank值为1/4\*1/2+1/4*1=9/24。

​	为了计算方便，我们设置各页面初始的PageRank值为一个列向量V0。然后再基于转移矩阵，我们可以直接求出新一轮各个页面的PageRank值。即 V1 = MV0

[![img](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/03.png)](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/03.png)

​	现在得到了各页面新的PageRank值V1, 继续用M 去乘以V1 ,就会得到更新的PageRank值。一直迭代这个过程，可以证明出V最终会收敛。此时停止迭代。这时的V就是各个页面的PageRank值。

[![img](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/04.png)](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/04.png)

[![img](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/05.png)](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/05.png)

​	处理Dead Ends(终止点)：

​	上面的PageRank计算方法要求整个Web是强联通的。而实际上真实的Web并不是强联通的，**有一类页面，它们不存在任何外链，对其他网页没有PR值的贡献，称之为Dead Ends(终止点)**。如下图：

[![img](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/06.png)](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/06.png)

​	这里页面C即是一个终止点。而上面的算法之所以能够成功收敛，很大因素上基于转移矩阵每一列的和为1（每一个页面都至少有一个出链）。当页面C没有出链时，转移矩阵M如下所示：

[![img](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/07.png)](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/07.png)

​	基于这个转移矩阵和初始的PageRank列向量，每一次迭代过的PageRank列向量如下：

[![img](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/08.png)](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/08.png)

​	解决该问题的一种方法是：迭代拿掉图中的Dead Ends点以及相关的边，之所以是迭代拿掉，是因为当拿掉最初的Dead Ends之后，又可能产生新的Dead Ends点。直到图中没有Dead Ends点为止。然后对剩余的所有节点，计算它们的PageRank ，然后以拿掉Dead Ends的逆序反推各个Dead Ends的PageRank值。

​	比如在上图中，首先拿掉页面C，发现没有产生新的Dead Ends。然后对A，B，D 计算他们的PageRank，他们初始PageRank值均为1/3，且A有两个出链，B有两个出链，D有一个出链，那么由上面的方法可以算出各页面最终的PageRank值。假设算出A的PageRank 为x，B的PageRank 为y，D的PageRank 为z，那么C的PageRank值为1/3\*x + 1/2\*z 。

​	处理Spider Traps（蜘蛛陷阱）：

​	真实的Web链接关系若是转换成转移矩阵，那必将是一个稀疏的矩阵。而稀疏的矩阵迭代相乘会使得中间产生的PageRank向量变得不平滑（一小部分值很大，大部分值很小或接近于0）。而一种Spider Traps节点会加剧这个不平滑的效果，也即是蜘蛛陷阱。它是指某一些页面虽然有外链，但是它只链向自己。如下图所示：

[![img](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/09.png)](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/09.png)

​	如果对这个图按照上面的方法进行迭代计算PageRank ， 计算后会发现所有页面的PageRank值都会逐步转移到页面C上来，而其他页面都趋近于零。

[![img](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/10.png)](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/10.png)

​	为了解决这个问题，我们需要对PageRank 计算方法进行一个平滑处理–加入teleporting(跳转因子)。也就是说，用户在访问Web页面时，除了按照Web页面的链接关系进行选择以外，他也可能直接在地址栏上输入一个地址进行访问。这样就避免了用户只在一个页面只能进行自身访问，或者进入一个页面无法出来的情况。

加入跳转因子之后，PageRank向量的计算公式修正为：

[![img](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/11.png)](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/11.png)

其中，β 通常设置为一个很小的数（0.2或者0.15），e为单位向量，N是所有页面的个数，乘以1/N是因为随机跳转到一个页面的概率是1/N。这样，每次计算PageRank值，既依赖于转移矩阵，同时依赖于小概率的随机跳转。

以上图为例，改进后的PageRank值计算如下：

[![img](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/12.png)](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/12.png)

按照这个计算公式迭代下去，会发现spider traps 效应被抑制了，使得各个页面得到一个合理的PageRank值。

## 33.2 任务内容

基于MapReduce 的PageRank 设计思路：

假设目前需要排名Dev网页有如下四个：

[![img](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/13.png)](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/13.png)

其中每一行中第一列为网页，第二列为该网页的pagerank值，之后的列均为该网页链接的其他网页。

Baidu存在三个外链接。

Google存在两个外链接。

Sina存在一个外链接。

Hao123存在两个外链接。

由数据可以看出：指向Baidu的链接有一个，指向Google的链接有两个，指向Sina的链接有三个，指向Hao123的链接有两个，所以Sina的PR应该最高，其次是Google和Hao123相等，最后是Baidu。

因为我们要迭代的计算PageRank值，那么每次MapReduce 的输出要和输入的格式是一样的，这样才能使得Mapreduce 的输出用来作为下一轮MapReduce 的输入。

我们每次得到的输出如下：

[![img](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/14.png)](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/14.png)

Map过程的设计：

1.对每一行文本进行解析，获得当前网页、当前网页的PageRank值、当前网页要链接到的其他网页

2.计算出要链接到的其他网页的个数，然后求出当前网页对其他网页的贡献值。

输出设计时，要输出两种：

第一种输出的< key ,value>中的key 表示其他网页，value 表示当前网页对其他网页的贡献值。

第二种输出的< key ,value>中的key 表示当前网页，value 表示所有其他网页。

为了区别这两种输出，第一种输出的value里加入“@”，第二种输出的value里加入“&”

经过Map后的结果为：

[![img](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/15.png)](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/15.png)

Map结果输出之后，经过Shuffle过程排序并合并，结果如下：

[![img](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/16.png)](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/16.png)

由shuffle结果可知，shuffle过程的输出key表示一个网页，输出value表示一个列表，里面有两类：一类是从其他网页获得的贡献值，一类是该网页的所有出链网页

Reduce过程的设计：

1.shuffle的输出也即是reduce的输入。

2.reduce输入的key直接作为输出的key

对reduce输入的value进行解析，它是一个列表：

若列表里的值里包含“@”，就把该值“@”后面的字符串转化成float型加起来

若列表里的值里包含“&”，就把该值“&”后面的字符串提取出来

把所有贡献值的加和，和提取的字符串进行连接，作为reduce的输出value

最终输出如下：

[![img](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/17.png)](https://www.ipieuvre.com/doc/exper/28fcb358-91ad-11e9-beeb-00215ec892f4/img/17.png)

## 33.3 完整代码

​	下面编写代码，实现功能为：使用MR实现PageRank，计算出Baidu、Google、Sina、Hao123四个网站的PR值排名。

```java
package mr_pagerank;  
import java.io.IOException;  
import java.util.StringTokenizer;  
import org.apache.hadoop.conf.Configuration;  
import org.apache.hadoop.fs.Path;  
import org.apache.hadoop.io.Text;  
import org.apache.hadoop.mapreduce.Job;  
import org.apache.hadoop.mapreduce.Mapper;  
import org.apache.hadoop.mapreduce.Reducer;  
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;  
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;  
public class PageRank {  
  /*map过程*/  
  public static class mapper extends Mapper<Object,Text,Text,Text>{  
    private String id;  
    private float pr;  
    private int count;  
    private float average_pr;  
    public void map(Object key,Text value,Context context)  
      throws IOException,InterruptedException{  
      StringTokenizer str = new StringTokenizer(value.toString());//对value进行解析  
      id =str.nextToken();//id为解析的第一个词，代表当前网页  
      pr = Float.parseFloat(str.nextToken());//pr为解析的第二个词，转换为float类型，代表PageRank值  
      count = str.countTokens();//count为剩余词的个数，代表当前网页的出链网页个数  
      average_pr = pr/count;//求出当前网页对出链网页的贡献值  
      String linkids ="&";//下面是输出的两类，分别有'@'和'&'区分  
      while(str.hasMoreTokens()){  
        String linkid = str.nextToken();  
        context.write(new Text(linkid),new Text("@"+average_pr));//输出的是<出链网页，获得的贡献值>  
        linkids +=" "+ linkid;  
      }  
      context.write(new Text(id), new Text(linkids));//输出的是<当前网页，所有出链网页>  
    }  
  }  
  /*reduce过程*/  
  public static class reduce extends Reducer<Text,Text,Text,Text>{  
    public void reduce(Text key,Iterable<Text> values,Context context)  
      throws IOException,InterruptedException{  
      String link = "";  
      float pr = 0;  
      /*对values中的每一个value进行分析，通过其第一个字符是'@'还是'&'进行判断 
    通过这个循环，可以求出当前网页获得的贡献值之和，也即是新的PageRank值；同时求出当前 
    网页的所有出链网页 */  
      for(Text val:values){  
        if(val.toString().substring(0,1).equals("@")){  
          pr += Float.parseFloat(val.toString().substring(1));  
        }  
        else if(val.toString().substring(0,1).equals("&")){  
          link += val.toString().substring(1);  
        }  
      }  
      pr = 0.8f*pr + 0.2f*0.25f;//加入跳转因子，进行平滑处理  
      String result = pr+link;  
      context.write(key, new Text(result));  
    }  
  }  


  public static void main(String[] args) throws Exception{  
    Configuration conf = new Configuration();  
    conf.set("mapred.job.tracker", "hdfs://127.0.0.1:9000");  
    //设置数据输入路径  
    String pathIn ="hdfs://127.0.0.1:9000/pagerank/input";  
    //设置数据输出路径  
    String pathOut="hdfs://127.0.0.1:9000/pagerank/output/pr";  
    for(int i=1;i<100;i++){      //加入for循环，最大循环100次  
      Job job = new Job(conf,"page rank");  
      job.setJarByClass(PageRank.class);  
      job.setMapperClass(mapper.class);  
      job.setReducerClass(reduce.class);  
      job.setOutputKeyClass(Text.class);  
      job.setOutputValueClass(Text.class);  
      FileInputFormat.addInputPath(job, new Path(pathIn));  
      FileOutputFormat.setOutputPath(job, new Path(pathOut));  
      pathIn = pathOut;//把输出的地址改成下一次迭代的输入地址  
      pathOut = pathOut+'-'+i;//把下一次的输出设置成一个新地址。  
      System.out.println("正在执行第"+i+"次");  
      job.waitForCompletion(true);//把System.exit()去掉  
      //由于PageRank通常迭代30~40次，就可以收敛，这里我们设置循环35次  
      if(i == 35){  
        System.out.println("总共执行了"+i+"次之后收敛");  
        break;  
      }  
    }  
  }  

}
```



