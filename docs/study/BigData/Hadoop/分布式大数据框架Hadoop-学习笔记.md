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

#### 9.1.2.1 块

HDFS的文件被分成块进行存储，块是文件存储处理的逻辑单元

+ 支持大规模文件存储：大规模文件可拆分若干块，不同文件块可分发到不同的节点上
+ 简化系统设计：简化了存储管理、方便元数据的管理
+ **适合数据备份**：每个文件块都可以冗余存储到多个节点上，大大提高了系统的容错性和可用性

#### 9.1.2.2 节点

+ NameNode
  + 存储元数据
  + 元数据保存在**内存**中
  + **保存文件，block，datanode之间的映射关系**
+ DataNode
  + 存储文件内容
  + 文件内容保存在**磁盘**
  + **维护了block id到datanode本地文件的映射关系**

---

​	NameNode：负责管理分布式文件系统的命名空间（NameSpace），保存了两个核心的数据结构，即FsImage和EditLog。

+ FsImage用于维护文件系统树以及文件树中所有的文件和文件夹的元数据
+ EditLog操作日志文件记录了所有针对文件的创建、删除、重命名等操作

​	名称节点记录了每个文件中各个块所在的数据节点的位置信息

