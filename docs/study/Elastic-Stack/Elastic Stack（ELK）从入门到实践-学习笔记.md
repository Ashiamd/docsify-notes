# Elastic Stack（ELK）从入门到实践 - 学习笔记

> [Elastic Stack（ELK）从入门到实践](https://www.bilibili.com/video/BV1iJ411c7Az) <== 视频链接
>
> [Elasticsearch 的使用Demo](https://blog.csdn.net/u011580290/article/details/88226164)	<=	推荐文章
>
> [gavin5033的博客--ELK专栏](https://blog.csdn.net/gavin5033/category_8070372.html)	<=	推荐ELK学习文章
>
> [Elasticsearch 索引的映射配置详解 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/41929500)	<=	很清晰的概述文章

# 1. Elastic Stack简介

## 1.1 ELK

+ Elastic
+ Logstash
+ Kibana

整体的作用就是日志分析

## 1.2 Elastic Stack

+ **Elasticsearch**

  ​	Elasticsearch **基于java**，是个开源分布式搜索引擎，它的特点有：分布式，零配置，自动发现，索引自动分片，索引副本机制，restful风格接口，多数据源，自动搜索负载等。

+ **Logstash**

  ​	Logstash **基于java**，是一个开源的用于收集,**分析**和存储日志的工具。

+ **Kibana**

  ​	Kibana **基于nodejs**，也是一个开源和免费的工具，Kibana可以为 Logstash 和 ElasticSearch 提供的日志分析友好的Web 界面，可以汇总、分析和搜索重要数据日志。

+ **Beats**

  ​	Beats是elastic公司开源的一款**采集系统监控数据**的代理agent，是在被监控服务器上以客户端形式运行的数据收集器的统称，可以直接把数据发送给Elasticsearch或者通过Logstash发送给Elasticsearch，然后进行后续的数据分析活动。

  Beats由如下组成：

  + Packetbeat：是一个网络数据包分析器，用于监控、收集网络流量信息，Packetbeat嗅探服务器之间的流量，解析应用层协议，并关联到消息的处理，其支持ICMP (v4 and v6)、DNS、HTTP、Mysql、PostgreSQL、Redis、MongoDB、Memcache等协议；
  + **Filebeat：用于监控、收集服务器日志文件，其已取代 logstash forwarder；**
  + Metricbeat：可定期获取外部系统的监控指标信息，其可以监控、收集 Apache、HAProxy、MongoDB、MySQL、Nginx、PostgreSQL、Redis、System、Zookeeper等服务；
  + Winlogbeat：用于监控、收集Windows系统的日志信息；

+ X-Pack

+ Elastic Cloud

# 2. Elasticsearch

## 2.1 简介

​	ElasticSearch是一个基于[Lucene](https://baike.baidu.com/item/Lucene/6753302?fr=aladdin)的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTFUL web接口。ElasticSearch用Java开发，并作为Apache许可条款下的开放源码发布，是当前流行的企业级搜索引擎。设计用于云计算中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。

​	<small>零配置、多用户、云方案、JSON数据 => ElasticSearch</small>

## 2.2 基本概念

### 2.2.1 索引

+ 索引（index）是Elasticsearch对逻辑数据的逻辑存储，所以它可以分为更小的部分。

+ 可以把索引看成关系型数据库的表，索引的结构是为快速有效的全文索引准备的，特别是它不存储原始值。

+ Elasticsearch可以把索引存放在一台机器或者分散在多台服务器上，每个索引有一或多个分片（shard），每个分片可以有多个副本（replica）。

### 2.2.2 文档

+ 存储在Elasticsearch中的主要实体叫文档（document）。用关系型数据库来类比的话，一个文档相当于数据库表中的一行记录。

+ **Elasticsearch和MongoDB中的文档类似，都可以有不同的结构，但Elasticsearch的文档中，相同字段必须有相同类型。**
+ 文档由多个字段组成，每个字段可能多次出现在一个文档里，这样的字段叫多值字段（multivalued）。
+ 每个字段的类型，可以是文本、数值、日期等。字段类型也可以是复杂类型，一个字段包含其他子文档或者数组。

### 2.2.3 映射

+ 所有文档写进索引之前都会先进行分析，如何将输入的文本分割为词条、哪些词条又会被过滤，这种行为叫做映射（mapping）。一般由用户自己定义规则。

### 2.2.4 文档类型

+ 在Elasticsearch中，一个索引对象可以存储很多不同用途的对象。例如，一个博客应用程序可以保存文章和评论。
+ 每个文档可以有不同的结构。
+ **不同的文档类型不能为相同的属性设置不同的类型**。例如，在同一索引中的所有文档类型中，一个叫title的字段必须具有相同的类型。

## 2.3 RESTful API

​	在Elasticsearch中，提供了功能丰富的RESTful API的操作，包括基本的CRUD、创建索引、删除索引等操作。

### 2.3.1创建非结构化索引

​	在Lucene中，创建索引是需要定义字段名称以及字段的类型的，在Elasticsearch中提供了非结构化的索引，就是不需要创建索引结构，即可写入数据到索引中，实际上在Elasticsearch底层会进行结构化操作，此操作对用户是透明的。

​	创建空索引：

