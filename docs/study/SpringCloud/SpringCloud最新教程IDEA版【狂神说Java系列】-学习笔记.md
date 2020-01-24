# kuangaji SpringCloud最新教程IDEA版【狂神说Java系列】

> 视频链接 https://www.bilibili.com/video/av76020761?p=1

## 1. 这个阶段该如何学习

### 回顾之前的知识

+ JavaSE
+ 数据库
+ 前端
+ Servlet
+ Http
+ Mybatis
+ Spring
+ SpringMVC
+ SpringBoot
+ Dubbo、Zookeeper、分布式基础
+ Maven、Git
+ Ajax、Json

### 串一下自己会的东西

+ 数据库
+ Mybatis
+ Spring
+ SpringMVC
+ SpringBoot
+ Dubbo、Zookeeper、分布式基础
+ Maven、Git
+ Ajax、Json

### 这个阶段如何学习

```none
技术选型演变、发展

三层架构 + MVC

框架：
	Spring IOC AOP
	
	SpringBoot，新一代的JavaEE开发标准，自动装配
	
	服务/功能拆分、模块化~ 过去是all in one
	
	模块的开发===all in one,代码没变化~
	
微服务架构4个核心问题？
	1. 服务很多，客户端该怎么访问（负载均衡、热备）
	2. 这么多服务？服务之间如何通信？（HTTP或RPC）
	3. 这么多的服务？如何治理？（服务注册与发现机制，即注册中心）
	4. 服务挂了怎么办？(容灾)
	
解决方案：
	Spring Cloud 生态
	
	1. Spring Cloud NetFlix 一站式解决方案
		客户端和服务器之间需要经过API Gateway，api网关，zuul组件
		Feign ---HttpClient ---- Http通信方式，同步，阻塞
		服务注册与发现：Eureka
		熔断机制：Hystrix
	
	2. Apache Dubbo ZooKeeper 半自动，需要整合别人的
		API:没有，找第三方组件，或者自己实现（自己实现不难）
		Dubbo：java的RPC框架
		ZooKeeper
		熔断机制：节主Hystrix
		
		Dubbo这个方案并不完善~毕竟本身只是为了Java的PRC框架
	
	3. Spring Cloud Alibaba 一站式解决方案！更简单
		
		
新概念：服务网站~ Server Mesh
	istio
	

万变不离其宗
	1. API
	2. HTTP,RPC
	3. 注册和发现
	4. 熔断机制
	
网络不可靠！（本质原因）
```

> [轻松理解MYSQL MVCC 实现机制](https://blog.csdn.net/whoamiyang/article/details/51901888)
>
> [负载均衡与双机热备](https://blog.csdn.net/qq_35642036/article/details/84191697)
>
> [SpringCloud（一）：SpringCloud和Dubbo的对比](https://blog.csdn.net/qq_35620501/article/details/92693398)
>
> [Dubbo和Nginx的区别](https://blog.csdn.net/qq_35001776/article/details/78498430)
>
> [zookeeper 都有哪些使用场景？](https://www.jianshu.com/p/baf966931c32)
>
> [Vert.x与Netty的区别](https://blog.csdn.net/neosmith/article/details/92790681)
>
> [微服务中的熔断机制](https://blog.csdn.net/fsy9595887/article/details/84521681)
>
> [谈谈我对服务熔断、服务降级的理解](https://blog.csdn.net/guwei9111986/article/details/51649240)
>
> [Spring Cloud Alibaba到底坑不坑？](https://www.cnblogs.com/didispace/p/10675601.html)
>
> [Spring Cloud Alibaba实战(一) - 概述](https://developer.aliyun.com/article/718349)
>
> [ServiceMesh简介](https://www.cnblogs.com/stay-foolisher/p/10422971.html)

### 常见面试题

1. 什么是微服务？
2. 微服务之间是如何独立通讯的？
3. SpringCloug和Dubbo有哪些区别？
4. SpringBoot和SpringCloud，请你谈谈对他们的理解
5. 什么是服务熔断？什么是服务降级
6. 微服务的优缺点分别是什么？说下你在项目开发中遇到的坑
7. 你所知道的微服务技术栈有哪些？请列举一二
8. eureka和zookeeper都可以提供服务注册与发现的功能，请说说两个的区别？

## 2. 回顾微服务和微服务架构

> [Microservices](https://martinfowler.com/articles/microservices.html)
>
> [微服务（Microservices）——Martin Flower](https://www.cnblogs.com/liuning8023/p/4493156.html)

### 2.1 基本概念

+ 微服务，它提倡将单一的应用程序划分成一组小的服务。（模块化）

+ 服务之间采用轻量级的通信机制相互沟通。（通常HTTP）

+ 微服务化的核心就是将传统的一站式应用，根据业务拆分成一个一个的服务，彻底地去耦合，每一个微服务提供单个业务功能的服务，一个服务做一件事情，从技术角度看就是一种小而独立的处理过程，类似进程的概念，能够自行单独启动或销毁，拥有自己独立的数据库。

### 2.2 微服务与微服务架构

#### 微服务

强调的是服务的大小，它关注的是某一个点，是具体解决某一个问题/提供落地对应服务的一个应用，狭义的看，可以看做IDEA中的一个个微服务工程，或者Module

```none
IDEA 工具里面使用Mven开发的一个个独立的小Module，它具体是使用springboot开发的一个小模块，专业的事情交给专业的模块来做，一个模块就做一件事情。
强调的是一个个的个体，每个个体完成一个具体的任务或功能。
```

#### 微服务架构

一种新的架构形式，Martin Fowler，2014提出

微服务架构是一种架构模式，他提倡将单一应用程序划分成一组小的服务，服务之间互相协调，互相配合，为用户提供最终价值。每个服务运行在其独立的进程中，服务于服务间采用轻量级的通信机制互相协作，每个服务都围绕着具体的业务进行构建，并且能够被独立的部署到生产环境中，另外，应尽量避免统一的，集中式的服务管理机制，对具体的一个服务而言，应根据业务上下文，选择合适的语言，工具对其进行构建。

### 2.3 微服务优缺点

**优点**

+ 单一职责原则
+ 每个服务足够内聚，足够小，代码容易理解，这样能聚焦一个指定的业务功能或业务需求
+ 开发简单，开发效率提高，一个服务可能就是专一地只干一件事
+ 微服务能够被小团队单独开发，这个小团队是2-5人的开发人员组成
+ 微服务是松耦合的，是有功能意义的服务，无论是在开发阶段或部署阶段都是独立的
+ 微服务能使用不同的语言开发
+ 易于和第三方集成，微服务允许容易且灵活的方式集成自动部署，通过持续集成工具，如jenkins、Hudson、bamboo
+ 微服务易于被一个开发人员理解，修改和维护，这样小团队能够更关注自己的工作成果。无需通过合作才能体现价值。
+ 微服务允许你利用融合最新技术。
+ 微服务只是业务逻辑的代码，不会和HTML、CSS或者其他界面混合
+ 每个微服务都有自己的存储能力，可以有自己的数据库，也可以有统一的数据库

**缺点**

+ 开发人员要处理分布式系统的复杂性
+ 多服务运维难度，随着服务的增加，运维的压力也在增大
+ 系统部署依赖
+ 数据一致性
+ 系统集成测试
+ 性能监控

### 2.4 微服务技术栈有哪些

| 微服务条目                               | 落地技术                                                     |
| ---------------------------------------- | ------------------------------------------------------------ |
| 服务开发                                 | SpringBoot、Spring、SpringMVC                                |
| 服务配置与管理                           | Netflix公司的Archaius、阿里的Diamond等                       |
| 服务注册与发现                           | Eureka、Cosul、Zookeeper等                                   |
| 服务调用                                 | Rest、RPC、gRPC                                              |
| 服务熔断器                               | Hystrix、Envoy等                                             |
| 负载均衡                                 | Ribbon、Nginx等                                              |
| 服务接口调用（客户端调用服务的简化工具） | Feign等                                                      |
| 消息队列                                 | Kafka、RabbitMQ、ActiveMQ等                                  |
| 服务配置中心管理                         | SpringCloudConfig、Chef等                                    |
| 服务路由（API网关）                      | Zuul等                                                       |
| 服务监控                                 | Zabbix、Nagios、Metrics、Specatator等                        |
| 全链路追踪                               | Zipkin、Brave、Dapper等                                      |
| 服务部署                                 | Docker、OpenStack、Kubernetes等                              |
| 数据流操作开发包                         | SpringCloud Stream（封装与Redis、Rabbit、Kafka等发送接收消息） |
| 事件消息总线                             | SpringCloud Bus                                              |

### 2.5 为什么选择SpringCloud作为微服务架构

#### 1. 选型依据
+ 整体解决方案和框架成熟度
+ 社区热度
+ 可维护性
+ 学习曲线
#### 2. 当前各大IT公司用的微服务架构有哪些
+ 阿里：dubbo+HFS
+ 京东：JSF
+ 新浪：Motan
+ 当当网：Dubbox

#### 3. 各微服务框架对比

| 功能点/服务框架 | Netflix/SpringCloud                                          | Motan                                                        | gRPC                      | Thrift   | Dubbo/DubboX                        |
| --------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------- | -------- | ----------------------------------- |
| 功能定位        | 完整的微服务框架                                             | RPC框架，但整合了ZK或Consul，实现集群环境的基本服务注册/发现 | RPC框架                   | RPC框架  | 服务框架                            |
| 支持Rest        | 是，Ribbon支持多种可拔插的序列化选择                         | 否                                                           | 否                        | 否       | 否                                  |
| 支持RPC         | 否                                                           | 是（Hession2）                                               | 是                        | 是       | 是                                  |
| 支持多语言      | 是（Rest形式）                                               | 否                                                           | 是                        | 是       | 否                                  |
| 负载均衡        | 是（服务端zuul+客户端Ribbon），zuul-服务，动态路由，云端负载均衡Eureka（针对中间层服务器） | 是（客户端）                                                 | 否                        | 否       | 是（客户端）                        |
| 配置服务        | Netfix Archaius，Spring Cloud Config Server集中配置          | 是（zookeeper提供）                                          | 否                        | 否       | 否                                  |
| 服务调用链监控  | 是（zuul），zuul提供边缘服务，API网关                        | 否                                                           | 否                        | 否       | 否                                  |
| 高可用/容错     | 是（服务端Hystrix+客户端Ribbon）                             | 是（客户端）                                                 | 否                        | 否       | 是（客户端）                        |
| 典型应用案例    | Netflix                                                      | Sina                                                         | Google                    | Facebook |                                     |
| 社区活跃程度    | 高                                                           | 一般                                                         | 高                        | 一般     | 2017年后重新开始维护，之前中断了5年 |
| 学习难度        | 中                                                           | 低                                                           | 高                        | 高       | 低                                  |
| 文档丰富程度    | 高                                                           | 一般                                                         | 一般                      | 一般     | 高                                  |
| 其他            | Spring Cloud Bus为我们的应用程序带来了更多管理端点           | 支持降级                                                     | Netflix内部在开发集成gRPC | IDL定义  | 实践的公司比较多                    |

## 3. 什么是SpringCloud

### 3.1 基本介绍	

​	Springcloud基于Spring Boot提供了一套微服务解决方案，包括服务注册与发现，配置中心，全链路监控，服务网关，负载均衡，熔断器等组件，除了基于NetFlix的开源组件做髙度抽象封装之外，还有一些选型中立的开源组件。

​	SpringCloud利用Springboot的开发便利性，巧妙地简化了分布式系统基础设施的开发，SpringCloud为开发人员提供了快速构建分布式系统的一些工具，包括**配置管理、服务发现、断路器、路由、微代理、事件总线、全局锁、决策竞选、分布式会话等等**，他们都可以用SpringBoot的开发风格做到一键启动和部署。

​	Springboot并没有重复造轮子，它只是将目前各家公司开发的比較成熟、经得起实际考研的服务框架组合起来，通过 SpringBoot风格进行再封装，屏蔽掉了复杂的配置和实现原理，**最终给开发者留出了一套简单易懂、易部署和易维护的分布式系统开发工具包**

​	SprinfCloud是分布式微服务架构下的一站式解决方案，是各个微服务架构落地技术的集合体，俗称微服务全交通。

### 3.2 SpringCloud和SpringBoot关系

+ SpringBoot专注于快速方便的开发单个个体微服务。-jar包
+ SpringCloud是专注全局的微服务协调整理治理框架，它将SpringBoot开发的一个个单体微服务整合并管理起来，为各个微服务之间提供：配置管理、服务发现、断路器、路由、微代理、事件总线、全局锁、决策竞选，分布式会话等等集成服务。
+ SpringBoot可以离开SpringCloud独立使用，开发项目，但是SpringCloud离不开SpringBoot，属于依赖关系。
+ SpringBoot专注于快速、方便地开发单个个体微服务，SpringCloud关注全局的服务治理框架

### 3.3 Dubbo和SpringCloud技术选型

#### 1. 分布式+服务治理Dubbo

​	目前成熟的互联网架构：应用服务拆分+消息中间件

​	客户端-> CDN-> Lvs集群 -> Nginx集群 -> 服务器Tomcat集群、分布式文件系统(HDFS、fastDFS)->注册中心<-服务提供者->MQ消息中间件(队列、异步)->缓存（Redis集群）、Elasticsearch搜索引擎（通常与缓存相关）->数据库集群、数据库中间件(MyCat等，解决数据库同步问题，通常读水平拆分、写垂直拆分，因为写操作比较IO耗时)。

#### 2. Dubbo和SpringCloud对比

|              | Dubbo         | SpringCloud                 |
| ------------ | ------------- | --------------------------- |
| 服务注册中心 | ZooKeeper     | SpringCloud Netflix Eureka  |
| 服务调用方式 | RPC           | REST API                    |
| 服务监控     | Dubbo-monitor | SpringBoot Admin            |
| 断路器       | 不完善        | SpringCloud Netflix Hystrix |
| 服务网关     | 无            | SpringCloud Netflix Zuul    |
| 分布式配置   | 无            | SpringCloud Config          |
| 服务跟踪     | 无            | SpringCloud Sleuth          |
| 消息总栈     | 无            | SpringCloud Bus             |
| 数据流       | 无            | SpringCloud Stream          |
| 批量任务     | 无            | SpringCloud Task            |

**最大区别:SpringCloud抛弃了Dubbo的RPC通信,采用的是基于HTTP的REST方式。**

严格来说,这两种方式各有优劣。虽然从一定程度上来说,后者牺牲了服务调用的性能,但也避免了上面提到的原生RPC带来的问题。而且REST相比RPC更为灵活,服务提供方和调用方的依赖只依靠一纸契约,不存在代码级别的强依赖,这在强调快速演化的微服务环境下,显得更加合适。

**品牌机与组装机的区别**

很明显, Spring Cloud的功能比 Dubbo更加强大,涵盖面更广,而且作为Spring的拳头项目,它也能够与 Spring Framework、 Spring Boot、 Spring Data、 Spring Batch等其他 Spring项目完美融合,这些对于微服务而言是至关重要的。使用Dubbo构建的微服务架构就像组裝电脑,各环节我们的选择自由度很高,但是最终结果很有可能因为一条内存质量不行就点不亮了,总是让人不怎么放心,但是如果你是一名高手,那这些都不是问题;而SprintCoud就像品牌机,在Spring Source的整合下,做了大量的兼容性测试,保证了机器拥有更高的稳定性,但是如果要在使用非原装组件外的东西,就需要对其基础有足够的了解。

**社区支持与更新力度**

最为重要的是, Dubbo停止了5年左右的更新,虽然2017.7重启了。对于技术发展的新需求,需要由开发者自行拓展升级(比如当当网弄出了 DubboX),这对于很多想要采用微服务架构的中小软件组织,显然是不太合适的,中小公司没有这么强大的技术能力去修改 Dubbo源码+周边的一整套解決方案,并不是每一个公司都有阿里的大牛+真实的线上生产环境测试过

**解决的问题域不一样：Dubbo的定位是一款RPC框架，Spring Cloud的目标是微服务架构下的一站式解决方案**

**设计模式+微服务拆分思想：软实力，活跃度**

> [LVS 简介及使用](https://www.jianshu.com/p/7a063123d1f1)
>
> [数据库垂直拆分 水平拆分](https://www.cnblogs.com/firstdream/p/6728106.html)
>
> [干货满满！10分钟看懂Docker和K8S](https://my.oschina.net/jamesview/blog/2994112)

### 3.4 SpringCloud能干啥

+ Distributed/versioned configuration(分布式/版本控制配置)
+ Service registration and discovery(服务注册与发现)
+ Routing(路由)
+ Service-to- service calls(服务到服务的调用)
+ Load balancing(负载均衡配置)
+ Circuit Breakers(断路器)
+ Distributed messaging(分布式消息管理)
+ ......

### 3.5 SpringCloud下载

```none
Spring Cloud是一个由众多独立子项目组成的大型综合项目，每个子项目有不同的发行节奏，都维护着自己的发布版本号。Spring Cloud通过一个资源清单 BOM(Bill of Materials)来管理每个版本的子项目清单。为了避免与子项目的发布号混淆，所以没有采用版本号的方式，而是通过命名的方式。

这些版本名称的命名方式采用了伦敦地铁站的名称，同时根据字母表的顺序来对应版本实践顺序，比如：最早的Release版本：Angel，第二个Release版本：Brixton，然后是Camden、Dalston、Edgware...
```

> [Spring Cloud Netflix 中文文档 参考手册 中文版](https://www.springcloud.cc/spring-cloud-netflix.html)
>
> [Spring Cloud Dalston 中文文档](https://www.springcloud.cc/spring-cloud-dalston.html)
>
> [Spring Cloud 中国社区](http://www.springcloud.cn/)
>
> [Spring Cloud 中文网](https://www.springcloud.cc/)

## 4. Rest学习环境搭建：服务提供者

> [jetty和tomcat的区别和联系是什么？](https://www.zhihu.com/question/39055107)
>
> [map-underscore-to-camel-case](https://blog.csdn.net/weixin_41758407/article/details/90722718)
>
> [MyBatis二级缓存应用场景以及局限性](https://blog.csdn.net/haobindayi/article/details/81776853)
>
> [MYSQL缓存：一级缓存和二级缓存](https://www.cnblogs.com/maoyizhimi/p/7778504.html)

