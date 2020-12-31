# 分布式大数据框架Hadoop-学习笔记

> [分布式大数据框架Hadoop--章鱼大数据](https://www.ipieuvre.com/course/311)

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

