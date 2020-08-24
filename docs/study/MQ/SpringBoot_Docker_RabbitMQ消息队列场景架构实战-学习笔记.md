# SpringBoot+Docker+RabbitMQ消息队列场景架构实战-学习笔记

> [视频链接](https://www.bilibili.com/video/av86571508) https://www.bilibili.com/video/av86571508

##  0. 课程大纲

1. SpringBoot整合RabbitMQ
2. RabbitMQ的常见开发模式 Direct Fanout Topic Headers
3. 如何保证消息的可靠生产和可靠投递 CAP
4. 什么是死信队列和延迟队列

5. RabbitMQ高可用

## 1. 为什么要使用RabbitMQ

### 目标

掌握和了解什么是RabbitMQ

### 分析

​	RabbitMQ是一个开源的消息队列服务器，用来通过普通协议在完全不同的应用之间共享数据，RabbitMQ是使用Erlang语言(==数据传递==)来编写的,并且RabbitMQ是==基于AMQP协议==(协议是一种规范和约束)的。开源做到跨平台机制。

​	消息队列中间件是分布式系统中重要的组件，主要解决应用==解耦,异步消息,流量削锋等问题,实现高性能,高可用,可伸缩和最终一致性架构。目前使用较多的消息队列有：ActiveMQ, RabbitMQ, ZeroMQ, Kafka,MetaMQ, RocketMQ==。

### 在什么情况下使用RabbitMQ

+ **读多写少用缓存(Redis)；写多读少用队列(RabbitMQ, ActiveMQ)**
+ 解耦,系统A在代码中直接调用系统B和系统C的代码，如果将来D系统接入，系统A还需要修改代码,过于麻烦
+ **异步**，将消息写入消息队列，非必要的业务逻辑以异步的方式运行，加快响应速度（并行和串行）
+ **削峰**，并发量大的时候，所有的请求直接怼到数据库，造成数据库连接异常
  + 达到数据的"最终一致"

### 使用的企业

+ 滴滴,美团,头条,去哪儿等
+ 开源,性能优秀,稳定性好
+ 提供可靠的消息投递模式(confirm)返回模式(return)
+ 与SpringAMQP完美的整合，API丰富
+ 集群模式丰富(mirror rabbitmq),表达式配置,HA模式,镜像队列模型
+ 保证数据不丢失的前提做到高可用性,可用性

过去的单峰架构开发，相同的war包复制到数个服务器tomcat运行，再通过服务器的Nginx进行请求的负载均衡。由于数据库常常是影响服务响应时长的一个焦点，优化使用数据库，首先是索引化，进一步是数据库横向（集群-主从库）、竖向（业务逻辑分块）扩展、读写分离；由于数据库操作往往读多于写，所以又引入缓存；而写多于读的场景，则考虑使用消息队列。

> [深入理解AMQP协议](https://blog.csdn.net/weixin_37641832/article/details/83270778)
>
> [XMPP详解](https://www.jianshu.com/p/84d15683b61e)
>
> [Mybatis的一级缓存和二级缓存的理解和区别](https://blog.csdn.net/llziseweiqiu/article/details/79413130)
>
> [看完这篇文章,我奶奶都懂了https的原理](https://www.jianshu.com/p/b0303de5f638)

## 2-1. 什么是AMQP高级消息队列协议

## 目标

掌握和了解什么是AMQP协议

### 概述

AMQP全称：Advanced Message Queuing Protocol（高级消息队列协议）

是具有现代特性的二进制协议，是一个提供统一消息服务的应用层标准髙级消息队列协议，是应用协议的一个开发标准，为面向消息的中间件设计。

### 核心概念

+ **Server**：又称Broker，接受客户端的连接，实现AMQP实体服务。安装rabbitmq-server
+ **Connection**：连接,应用程序与Broker的网络连接TCP/IP(安全性 操作系统会对tcp有限制65535)Rabbitmq长连接的技术，我只开一次
+ **Channel**：网络信道，几乎所有的操作都在Channel中进行，Channel是进行消息读写的通道，客户端可以建立对各Channel，每个Channel代表一个会话任务。
+ **Message**：消息，服务与应用程序之间传送的数据，由Properties和body组成，Properties可是对消息进行修饰，比如消息的优先级，延迟等高级特性，Body则就是消息体的内容。
+ **Virtual Host**：虚拟地址，用于进行逻辑隔离，最上层的消息路由，一个虚拟主机可以有若干个 Exchange和Queue，同一个虚拟主机里面不能有相同名字的Exchange或Queue
+ **Exchange**：交换机，接受消息，根据路由键发送消息到绑定的队列。direct activemq kafka 
+ **Bindings**：Exchange和Queue之间的虚拟连接，binding中可以保护多个routing key
+ **Routing key**：是一个路由规则，虚拟机可以用它来确定如何路由一个特定消息
+ **Queue**：队列，也称为Message Queue消息队列，保存消息并将它们转发给消费者。

## 2-2. 为什么要使用RabbitMQ

> [还不理解“分布式事务”？这篇给你讲清楚！](https://www.cnblogs.com/zjfjava/p/10425335.html)
>
> [HTTP请求方法及幂等性探究](https://blog.csdn.net/qq_15037231/article/details/78051806)
>
> [http 请求包含哪几个部分（请求行、请求头、请求体）](https://www.cnblogs.com/qiang07/p/9304771.html)

## 3-1. 虚拟化容器技术-使用Docker-集成安装RabbitMQ

## 3-2.  Rabbitmq的服务介绍

## 4. 使用RabbitMQ场景

## 5-1. SpringBoot-整合RabbitMQ发布订阅机制-Fanout-01

> [RabbitMQ官网](https://www.rabbitmq.com/)

## 5-2. RabbitMQ发布订阅机制-Fanout-02

## 6. SpringBoot-整合RabbitMQ发布订阅机制-Direct

## 7. SpringBoot-整合RabbitMQ主题匹配Topic-副本

## 8. SpringBoot-RabbitMQ消息处理-持久化问题-副本

> [RabbitMQ 使用QOS（服务质量）+Ack机制解决内存崩溃的情况](https://www.cnblogs.com/yxlblogs/p/10253609.html)
>
> [RabbitMQ基础概念详细介绍](https://www.cnblogs.com/williamjie/p/9481774.html)
>
> [RabbitMQ教程](https://blog.csdn.net/hellozpc/article/details/81436980)
>
> [基于Netty与RabbitMQ的消息服务](https://www.cnblogs.com/luxiaoxun/p/4257105.html)

了解一些点：

1. 可靠生产
2. 可靠消费
3. 消息转移
4. 什么是消息冗余
5. 什么是死信队列，重定向队列（场景是什么）
6. 什么是延迟队列（场景是什么）
7. rabbitmq高可用，集群模式有哪些：mirror queue + keepalive

## 9. 个人补充

> [docker安装与使用](https://www.cnblogs.com/glh-ty/articles/9968252.html)
>
> [docker快速安装rabbitmq](https://www.cnblogs.com/angelyan/p/11218260.html)
>
> [docker 安装rabbitMQ](https://www.cnblogs.com/yufeng218/p/9452621.html)
>
> [阿里云-docker安装rabbitmq及无法访问主页](https://www.cnblogs.com/hellohero55/p/11953882.html)
>
> [Windows 下安装RabbitMQ服务器及基本配置](https://www.cnblogs.com/vaiyanzi/p/9531607.html)
>
> [Springboot 整合RabbitMq ，用心看完这一篇就够了](https://blog.csdn.net/qq_35387940/article/details/100514134)
>
> [SPRINGBOOT + JAVAMAIL + RABBITMQ实现异步邮件发送功能](http://www.freesion.com/article/4814171772/)
>
> [springboot2.x集成RabbitMQ实现消息发送确认 与 消息接收确认（ACK）](https://blog.csdn.net/qq_36850813/article/details/103296210)
>
> [RabbitMQ的死信队列详解](https://www.jianshu.com/p/986ee5eb78bc)
>
> [RabbitMQ基本概念（三）：后台管理界面](https://baijiahao.baidu.com/s?id=1608453370506467252&wfr=spider&for=pc)
>
> [利用Docker-compose一键搭建rabbitmq服务器](https://www.jianshu.com/p/1127ad6ee546)
>
> [使用docker-compose安装RabbitMQ](https://blog.csdn.net/chuishuwu3807/article/details/100893787)