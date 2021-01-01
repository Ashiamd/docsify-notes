# 分布式大数据框架Hadoop-学习笔记

> [分布式大数据框架Hadoop](https://www.ipieuvre.com/course/311)

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

