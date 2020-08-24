# SpringCloud最新教程IDEA版【狂神说Java系列】学习笔记

> 视频链接 https://www.bilibili.com/video/av76020761

## 1. 这个阶段该如何学习

> [maven模块化项目总共模块相互引用打包失败问题](https://blog.csdn.net/liqi_q/article/details/80557157) 很重要，说三遍，而且容易忘记，用了旧版的jar包！
>
> [maven模块化项目总共模块相互引用打包失败问题](https://blog.csdn.net/liqi_q/article/details/80557157) 很重要，说三遍，而且容易忘记，用了旧版的jar包！
>
> [maven模块化项目总共模块相互引用打包失败问题](https://blog.csdn.net/liqi_q/article/details/80557157) 很重要，说三遍，而且容易忘记，用了旧版的jar包！

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
>
> SOA和微服务等概念
>
> [SOA架构和微服务架构的区别](https://blog.csdn.net/zpoison/article/details/80729052)

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
>
> [【MyBatis】MyBatis一级缓存和二级缓存](https://blog.csdn.net/qq_26525215/article/details/79182637)

## 5.Rest学习环境搭建：服务消费者

> [@RequestBody和@RequestParam区别](https://blog.csdn.net/weixin_38004638/article/details/99655322)
>
> [@ResponseBody详解](https://blog.csdn.net/originations/article/details/89492884)
>
> [@Reference 、@Resource和@Autowired](https://blog.csdn.net/u014662858/article/details/84262544)
>
> [为什么一般是给post请求设置content-type,get请求不需要设置吗？](https://www.zhihu.com/question/363207841/answer/962651986)
>
> [@Bean 注解全解析](https://www.cnblogs.com/cxuanBlog/p/11179439.html)
>
> [Spring @Configuration 注解介绍](https://www.jianshu.com/p/721c76c1529c)

## 6. Eureka: 什么是Eureka

**Eureka服务注册与发现**

### 6.1 什么是Eureka

+ Euerka：怎么读？
+ Netflix在设计Eureka时，遵循的就是AP原则
+ Eureka是Netflix的一个子模块，也是核心模块之一。Eureka是一个基于REST的服务，用于定位服务，以实现云端中间层服务发现和故障转移，服务注册与发现对于微服务来说是非常重要的，有了服务发现与注册，只需要使用服务的标识符，就可以访问到服务器，而不需修改服务调用的配置文件了，功能类似Dubbo的注册中心，比如ZooKeeper；

### 6.2 原理讲解

+ Eureka的基本架构

  + Springcloud封装了Netflix公司开发的Eureka模块来实现服务注册和发现(对比 Zookeeper)
  + Eureka采用了C-S的架构设计, EurekaServer作为服务注册功能的服务器,他是服务注册中心
  + 而系统中的其他微服务。使用 Eureka的客户端连接到 EurekaServer并维持心跳连接。这样系统的维护人员就可以通过EurekaServer来监控系统中各个微服务是否正常运行, SpringCloud的一些其他模块(比如Zuul)就可以通过EurekaServer来发现系统中的其他微服务,并执行相关的逻辑
  + 和 Dubbo架构对比

  + Eureka包含两个组件：**Eureka Server**和**Eureka Client**
  + Eureka Server提供服务注册服务,各个节点启动后,会在 Eureka Server中进行注册,这样Eureka Server中的服务注册表中将会存储所有可用服务节点的信息,服务节点的信息可以在界面中直观的看到。
  + Eureka Client是一个java客户端,用于简化 Eureka Server的交互,客户端同时也具备一个内置的,<u>使用轮询负载算法的负载均衡器</u>。在应用启动后,将会向 Eureka Server发送心跳(默认周期为30秒)。如果Eureka Server在多个心跳周期内没有接收到某个节点的心跳, Eureka Server将会从服务注册表中把这个服务节点移除掉(默认周期为90秒)

+ 三大角色
  + Eureka Server:提供服务的注册与发现。
  + Service Provider:将自身服务注册到 Eureka中,从而使消费方能够找到。
  + Service Consumer:服务消费方从 Eureka中获取注册服务列表,从而找到消费服务。

**微服务开发套路步骤：**

1. **导入依赖**
2. **编写配置文件**
3. **开启这个功能 @EnableXXXX**
4. **配置类**

> [Eureka](https://www.jianshu.com/p/e2e3ded1f54a)
>
> [【Spring Cloud总结】14.Eureka常用配置详解](https://blog.csdn.net/acmman/article/details/99670419)
>
> [Spring Cloud Eureka 配置](https://www.cnblogs.com/april-chen/p/10617066.html)
>
> [EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP WHEN THEY'RE NOT. RENEWALS ARE LESSER THAN THRESHOLD AND HENCE THE INSTANCES ARE NOT BEING EXPIRED JUST TO BE SAFE.](https://www.cnblogs.com/breath-taking/articles/7940364.html)

## 7. Eureka: 服务注册-信息配置-自我保护机制

> [Spring Cloud学习过程中遇到的Bug Error creating bean with name 'scopedTarget.eurekaClient' defined in class](https://blog.csdn.net/QFYJ_TL/article/details/84066451)
>
> [Spring Cloud下配置eureka.instance.instance-id使得服务实例在eureka界面增加显示版本号](https://blog.csdn.net/qq_27680317/article/details/79181236)
>
> [Spring Cloud Eureka 自我保护机制](https://www.cnblogs.com/xishuai/p/spring-cloud-eureka-safe.html)
>
> [spring Cloud Eureka增加security后注册失败解决方案](https://blog.csdn.net/jerry_player/article/details/85952023)
>
> [Eureka的自我保护机制](https://www.cnblogs.com/ericnie/p/9393995.html)

**自我保护机制：好死不如赖活着**

一句话总结:某时刻某一个微服务不可以用了, eureka不会立刻清理,依旧会对该微服务的信息进行保存!

+ 默认情况下,如果Eureka Server在一定时间内没有接收到某个微服务实例的心跳, Eureka Server将会注销该实例(默认90秒)。但是当网络分区故障发生时,微服务与Eureka之间无法正常通行,以上行为可能变得非常危险了-因为微服务本身其实是健康的,**此时本不应该注销这个服务**。 Eureka通过自我保护机制来解决这个问题-当 Eureka Server节点在短时间内丢失过多客户端时(可能发生了网络分区故障),那么这个节点就会进入自我保护模式。一旦进入该模式, Eureka Server就会保护服务注册表中的信息,不再删除服务注册表中的数据(也就是不会注销仼何微服务)。当网络故障恢复后,该Eureka Server节点会自动退岀自我保护模式
+ 在自我保护模式中, Eureka Server会保护服务注册表中的信息,不再注销仼何服务实例。当它收到的心跳数重新恢复到阈值以上时,该 Eureka Server节点就会自动退出自我保护模式。它的设计哲学就是宁可保留错误的服务注册信息,也不盲目注销任何可能健康的服务实例。一句话:好死不如赖活着
+ 综上,自我保护模式是一种应对网络异常的安全保护措施。它的架构哲学是宁可同时保留所有微服务(健康的微服务和不健康的微服务都会保留),也不盲目注销任何健康的微服务。使用自我保护模式,可以让 Eureka集群更加的健壮和稳定
+ 在 SpringCloud中,可以使用`eureka. server, enable-self- preservation= false`禁用自我保护模式【不推荐关闭自我保护机制】

## 8. Eureka：集群环境配置

> [Spring Cloud：使用Eureka集群搭建高可用服务注册中心](http://blog.itpub.net/31558358/viewspace-2375380/)
>
> [**relay start error: java.lang.SecurityException: Cannot locate policy or framework files!**](https://www.ibm.com/mysupport/s/question/0D50z00005pgfSQCAY/relay-start-error-javalangsecurityexception-cannot-locate-policy-or-framework-files?language=zh_CN&sort=newest)
>
> [Eureka控制台相关介绍及自我保护机制解说](https://blog.csdn.net/qq_25112523/article/details/83028529)

## 9. CAP原则及对比ZooKeeper

### 回顾CAP原则

RDBMS （Mysql、Oracle、SqlServer）===>ACID

NoSQL（Redis、mongdb）===>CAP

### ACID是什么？

+ A（Atomicity）原子性
+ C（Consistency）一致性
+ I（Isolation）隔离性
+ D（Durability）持久性

### CAP是什么？

+ C（Consistency）强一致性
+ A（Availability）可用性
+ P（Partition tolerance）分区容错性

CAP的三进二：CA、AP、CP

**CAP理论的核心**

+ 一个分布式系统不可能同时很好的满足一致性,可用性和分区容错性这三个需求
+ 根据CAP原理，将NoSQL数据库分成了满足CA原则，满足CP原则和满足AP原则三个大类：
  + CA：单点集群，满足一致性，可用性的系统，通常可扩展性较差
  + CP：满足一致性，分区容错性的系统，通常性能不是特别高
  + AP：满足可用性，分区容错性的系统，通常可能对一致性要求低一点

### 作为服务注册中心，Eureka比ZooKeeper好在哪里？

​	著名的CAP理论指出，一个分布式系统不可能同时满足C（一致性）、A（可用性）、P（容错性）。

​	由于分区容错性P在分布式系统中是必须要保证的，因此我们只能在A和C之间进行权衡。

+ ZooKeeper保证的是CP；
+ Eureka保证的是AP；

### ZooKeeper保证的是CP

​	当向注册中心查询服务列表时,我们可以容忍注册中心返回的是几分钟以前的注册信息,但不能接受服务直接down掉不可用。也就是说,服务注册功能对可用性的要求要高于一致性。但是zk会出现这样一种情况,当 master节点因为网络故障与其他节点失去联系时,剩余节点会重新进行leader选举。问题在于,选举 leader的时间太长,30-120s,且选举期间整个zk集群都是不可用的,这就导致在选举期间注册服务瘫痪。在云部署的环境下,因为网络问题使得zk集群失去master节点是较大概率会发生的事件,虽然服务最终能够恢复,但是漫长的选举时间导致的注册长期不可用是不能容忍的。

### Eureka保证的是AP

​	Eureka看明白了这一点,因此在设计时就优先保证可用性。**Eureka各个节点都是平等的**,几个节点挂掉不会影响正常节点的工作,剩余的节点依然可以提供注册和查询服务。而 Eureka的客户端在向某个 Eureka注册时,如果发现连接失败,则会自动切換至其他节点,只要有一台 Eureka还在,就能保住注册服务的可用性,只不过查到的信息可能不是最新的,除此之外, Eureka还有一种自我保护机制,如果在15分钟内超过85%的节点都没有正常的心跳,那么 Eureka就认为客户端与注册中心出现了网络故障,此时会出现以下几种情况:

1. Eureka不再从注册列表中移除因为长时间没收到心跳而应该过期的服务
2. Eureka仍然能够接收新服务的注册和查询请求，但是不会被同步到其他节点上（即保证当前节点仍然可用）
3. 当网络稳定时，当前实例新的注册信息会被同步到其他节点中

​	**因此，Eureka可以很好地应对因网络故障导致部分节点失去联系的情况，而不会像zookeeper那样使整个注册服务瘫痪**。

## 10. Ribbon：负载均衡及Ribbon

> [30张图带你彻底理解红黑树](https://www.jianshu.com/p/e136ec79235c)
>
> [Spring Cloud之Ribbon转发请求头(header参数)](http://www.manongjc.com/article/59734.html)
>
> [详解 RestTemplate 操作](https://blog.csdn.net/itguangit/article/details/78825505)
>
> [Spring Cloud - Ribbon 使用以及自动请求头签名信息](https://www.jianshu.com/p/2e87d96023c8)
>
> [Springboot -- 用更优雅的方式发HTTP请求(RestTemplate详解)](https://www.jianshu.com/p/27a82c494413)
>
> [SpringCloud（五） 使用Ribbon进行Restful请求](https://www.cnblogs.com/hellxz/p/8875452.html)
>
> [Ribbon详解](https://www.jianshu.com/p/1bd66db5dc46)
>
> 出错or异常
>
> [RestTemplate踩坑笔记-中文乱码与json被解析成xml](https://www.jianshu.com/p/8be6efeb17b1)

### Ribbon是什么？

+ Spring Cloud Ribbon是基于Netflix Ribbon实现的一套**客户端负载均衡的工具**。
+ 简单的说, Ribbon是Netflix发布的开源项目,主要功能是提供客户端的软件负载均衡算法,将NetFlix的中间层服务连接在一起。Ribbon的客户端组件提供一系列完整的配置项如:连接超时、重试等等。简单的说,就是在配置文件中列岀 LoadBalancer(简称LB:负载均衡)后面所有的机器, Ribbon会自动的帮助你基于某种规则(如简单轮询,随机连接等等)去连接这些机器。我们也很容易使用 Ribbon实现自定义的负载均衡算法。

### Ribbon能干嘛?

+ LB，即负载均衡( Load Balance)，在微服务或分布式集群中经常用的一种应用。
+ **负載均衡简单的说就是将用户的请求平摊地分配到多个服务上,从而达到系统的HA(高可用)**。
+ 常见的负载均衡软件有 Nginx,`LVs`等等
+ dubbo、SpringCloud中均给我们提供了负载均衡, **SpringCloud的负载均衡算法可以自定义**
+ 负载均衡简单分类:
  + 集中式LB
    + 即在服务的消费方和提供方之间使用独立的LB设施,如Nginx:反向代理服务器,由该设施负责把访问请求通过某种策略转发至服务的提供方!
  + 进程式LB
    + 将LB逻辑集成到消费方，消费方从服务注册中心获知有哪些地址可用，然后自己再从这些地址中选出一个合适的服务器。
    + **Ribbon就属于进程内LB**,它只是一个类库,集成于消费方进程,消费方通过它来获取到服务提供方的地址!

## 11. Ribbon：使用Ribbon实现负载均衡

> [关联mysql失败_Server returns invalid timezone. Go to 'Advanced' tab and set 'serverTimezon'](https://www.cnblogs.com/sunchunmei/p/11426758.html)

SpringCloud中使用Ribbon时，Ribbon的默认算法即轮询算法

## 12. Ribbon：自定义负载均衡算法

> [Thread类中的方法：join()、sleep()、yield()之间的区别](https://blog.csdn.net/xzp_12345/article/details/81129735)
>
> [Ribbon核心组件IRule及配置指定的负载均衡算法](https://www.cnblogs.com/yufeng218/p/10952590.html)
>
> [SpringCloud Ribbon介绍及总结](https://www.jianshu.com/p/ad6a7e497ac0)
>
> [F5负载均衡综合实例详解](https://blog.csdn.net/weixin_43089453/article/details/87937994)
>
> [F5负载均衡原理](https://blog.csdn.net/panxueji/article/details/42647193)
>
> [java.lang.IllegalStateException: No instances available for SPRINGCLOUDTEST-PROVIDER-DEPT解决](https://blog.csdn.net/Ashiamd/article/details/104096878)

## 13. Feign：使用接口方式调用服务

> [Spring Cloud 学习教程——第三篇：服务消费者（Ribbon / Feign）](https://www.jianshu.com/p/e1f23a998b82)

### 13.1 Feign负载均衡-简介

​	feign是声明式的 web service客户端,它让微服务之间的调用变得更简单了,类似 controller调用 service。SpringCloud集成了Ribbon和Eureka,可在使用Feign时提供负载均衡的http客户端。

​	只需要创建一个接口,然后添加注解即可



​	feign，主要是社区，大家都习惯面向接口编程。这个是很多开发人员的规范。调用微服务访问两种方法

1. 微服务名字【Ribbon】
2. 接口和注解【feign】

### Feign能干什么？

+ Feign旨在使编写Java Http客户端变得更容易
+ 前面在使用 Ribbon+RestTemplatel时,利用RestTemplate对Http请求的封装处理,形成了一套模板化的调用方法。但是在实际开发中,由于对服务依赖的调用可能不止一处,往往一个接口会被多处调用,所以通常都会针对每个微服务自行封装一些客户端类来包装这些依赖服务的调用。所以, Feign在此基础上做了进一步封装,由他来帮助我们定义和实现依赖服务接口的定义,**在 Feign的实现下,我们只需要创建一个接口并使用注解的方式来配置它(类似于以前Dao接口上标注Mapper注解,现在是一个微服务接口上面标注一个 Feign注解即可。)**即可完成对服务提供方的接口绑定,简化了使用Spring Cloud Ribbon时,自动封装服务调用客户端的开发量。

### Feign继承了Ribbon

+ 利用Ribbon维护了MicroServiceCloud-Dept的服务列表信息,并且通过轮询实现了客户端的负载均衡,而与Ribbon不同的是,通过 Feign只需要定义服务绑定接口且以声明式的方法,优雅而且简单的实现了服务调用。

##  14. Hystrix：服务熔断

### 分布式系统面临的问题

复杂分布式体系结构中的应用程序有数十个依赖关系，每个依赖关系在某些时候将不可避免的失败

### 服务雪崩

​	多个微服务之间调用的时候,假设微服务A调用微服务B和微服务C,微服务B和微服务C又调用其他的微服务，这就是所谓的“扇出”、如果扇出的链路上某个微服务的调用响应时间过长或者不可用,对微服务A的调用就会占用越来越多的系统资源,进而引起系统崩溃,所谓的“雪崩效应”。

​	对于高流量的应用来说,单一的后端依赖可能会导致所有服务器上的所有资源都在几秒中内饱和。比失败更糟糕的是,这些应用程序还可能导致服务之间的延迟增加,备份队列,线程和其他系统资源紧张,导致整个系统发生更多的级联故障,这些都表示需要对故障和延迟进行隔离和管理,以便单个依赖关系的失败,不能取消整个应用程序或系统。

​	我们需要"弃车保帅"

### 什么是Hystrix

​	Hystrix是一个用于处理分布式系统的延迟和容错的开源库,在分布式系统里,许多依赖不可避免的会调用失败,比如超时,异常等, Hystrix能够保证在一个依赖出问题的情况下,不会导致整体服务失败,避免级联故障，以提高分布式系统的弹性。

​	"断路器"本身是一种开关装置,当某个服务单元发生故障之后,通过断路器的故障监控(类似熔断保险丝),**向调用方返回一个服务预期的,可处理的备选响应(FallBack),而不是长时间的等待或者抛出调用方法无法处理的异常,这样就可以保证了服务调用方的线程不会被长时间**,不必要的占用,从而避免了故障在分布式系统中的蔓延,乃至雪崩

### 能干嘛

+ 服务降级
+ 服务熔断
+ 服务限流
+ 接近实时的监控
+ ...

### 服务熔断

#### 是什么

​	熔断机制是对应雪崩效应的一种微服务链路保护机制。

​	当扇出链路的某个微服务不可用或者响应时间太长时,会进行服务的降级，**进而熔断该节点微服务的调用,快速返回错误的响应信息** 。当检测到该节点微服务调用响应正常后恢复调用链路。在 SpringCloud框架里熔断机制通过 Hystrix实现。 Hystrix会监控微服务间调用的状况,当失败的调用到一定阈值,缺省是5秒内20次调用失败就会启动熔断机制。熔断机制的注解是@HystrixCommand.

> [官方资料](https://github.com/Netflix/Hystrix/wiki)
>
> [7、何时进行服务熔断、服务降级、服务限流?_llianlianpay的博客-CSDN博客](https://blog.csdn.net/llianlianpay/article/details/79768890)
>
> [高并发架构设计之--「服务降级」、「服务限流」与「服务熔断」](https://blog.csdn.net/l18848956739/article/details/100132409)
>
> [mock数据的基础使用](https://www.cnblogs.com/missme-lina/p/10267770.html)
>
> [应对接口级故障：服务降级、熔断、限流、排队](https://www.jianshu.com/p/33f394c0ee2d)
>
> [谈谈我对服务熔断、服务降级的理解 专题](https://www.cnblogs.com/softidea/p/6346727.html)
>
> [高并发之服务降级与熔断](https://cloud.tencent.com/developer/article/1457494)
>
> [Hystrix的服务熔断和服务降级](https://www.cnblogs.com/guanyuehao0107/p/11848286.html)

## 15. Hystrix：服务降级

**服务熔断在服务端实现，而服务降级在客户端实现**

+ 服务熔断：服务端~ 某个服务超时或者异常，引起熔断~， 保险丝~

+ 服务降级：客户端~ 从整体网站请求负载考虑，当某个服务熔断或者关闭后，服务将不再被调用。此时在客户端，我们可以准备一个FallbackFactory，返回一个默认的值（缺省值）

## 16. Hystrix：Dashboard流监控

> [解决Hystrix Dashboard 一直是Loading ...的情况](https://www.cnblogs.com/hejianjun/p/8670693.html)
>
> [hystrix-dashboard](https://blog.csdn.net/Leon_Jinhai_Sun/article/details/100633310)
>
> [hystrixDashboard(服务监控)](https://www.cnblogs.com/yufeng218/p/11489175.html)

## 17. Zuul：路由网关

### 概述

#### 什么是Zuul？

​	Zuul包含了对请求的路由和过滤两个最主要的功能：

​	其中路由功能负责将外部请求转发到具体的微服务实例上,是实现外部访问统一入口的基础,而过滤器功能则负责对请求的处理过程进行干预,是实现请求校验,服务聚合等功能的基础。Zuul和Eureka进行整合,将Zuul自身注册为 Eureka服务治理下的应用,同时从Eureka中获得其他微服务的消息,也即以后的访问微服务都是通过Zuul跳转后获得。

​	注意:Zuul服务最终还是会注册进Eureka

​	提供:代理+路由+过滤三大功能!

#### Zuul能干嘛？

+ 路由
+ 过滤

>  [官方文档](https://github.com/Netflix/zuul)
>
> [ZUUL-API网关](https://blog.csdn.net/qq_27384769/article/details/82991261)
>
> [zuul路由网关](https://www.jianshu.com/p/ca76a4f396d1)

## 18. Config：Git环境搭建

### SpringCloud config分布式配置

#### 概述

##### 分布式系统面临的--配置文件的问题

​	微服务意味着要将单体应用中的业务拆分成一个个子服务,每个服务的粒度相对较小,因此系统中会出现大量的服务,由于每个服务都需要必要的配置信息才能运行,所以一套集中式的,动态的配置管理设施是必不可少的。Spring Cloud提供了ConfigServer来解決这个问题,我们每一个微服务自己帯着一个application.yml,那上百的的配置文件要修改起来,岂不是要发疯!

##### 什么是SpringCloud config分布式配置中心

​	Spring Cloud Config为微服务架构中的微服务提供集中化的外部配置支持,配置服务器为**各个不同微服务应用**的所有环节提供了一个**中心化的外部配置**。

​	Spring Cloud Config分为**服务端**和**客户端**两部分

​	服务端也称为分布式配置中心,它是一个独立的微服务应用,用来连接配置服务器并为客户端提供获取配置信息,加密,解密信息等访问接口。

​	客户端则是通过指定的配置中心来管理应用资源,以及与业务相关的配置内容,并在启动的时候从配置中心获取和加载配置信息。配置服务器默认采用git来存储配置信息,这样就有助于对环境配置进行版本管理。并且可以通过git客户端工具来方便的管理和访问配置内容。

##### SpringCloud config分布式配置中心能干嘛

+ 集中管理配置文件
+ 不同环境,不同配置,动态化的配置更新,分环境部署,比如 /dev /test /prod /beta /release
+ 运行期间动态调整配置,不再需要在每个服务部署的机器上编写配置文件,服务会向配置中心统一拉取配置自己的信息。
+ 当配置发生变动时,服务不需要重启,即可感知到配置的变化,并应用新的配置
+ 将配置信息以REST接口的形式暴露

##### Spring Cloud config分布式配置中心与github整合

​	由于Spring Cloud Config默认使用Git来存储配置文件(也有其他方式,比如支持SVN和本地文件),但是最推荐的还是Git,而且使用的是http / https访可的形式

## 19. Config：服务端链接Git配置

## 20. Config：客户端连接服务端访问远程

+ bootstrap.yml 系统级配置
+ application.yml 用户级配置

配置与代码分离，解耦

## 21. Config：远程配置实战测试

## 22. SpringCloud总结与展望

### SpringCloud：

+ 知识点
+ 看看面试题
+ 微服务和微服务架构
+ Euerka AP
  + 集群配置
  + 对比ZooKeeper CP
+ Ribbon （IRule负载均衡决策）
+ Feign 接口、社区要求、更加面向接口编程
+ Hystrix
  + 熔断
  + 降级
  + dashboard
+ Zuul
+ Spring Cloud Config （C-S-GIT）

### 开发流程套路

1. 导入依赖
2. 编写配置
3. @EnableXXX等

### 未来发展方向：

+ 框架源码
+ 设计模式
+ 新知识探索
+ Java新特性
+ Netty / MyCat / Http
+ JVM

> [Spring Cloud Alibaba 实战(四) - NACOS 服务发现与注册中心](https://blog.csdn.net/qq_33589510/article/details/102020934)
>
> [redis集群架构（含面试题解析）](https://www.cnblogs.com/wanghaokun/p/10366689.html)
>
> [Ribbon与Nginx区别](https://blog.csdn.net/Websphere_zxf/article/details/92833670)
>
> [SpringCloudConfig私有库服务端搭建](https://my.oschina.net/u/3420587/blog/893140)
>
> [spring cloud config将配置存储在数据库中](https://www.cnblogs.com/forezp/p/10414586.html)
>
> [maven多模块打包---必成功](https://blog.csdn.net/majipeng19950610/article/details/89358448)
>
> [Docker SpringCloud微服务集群 Eureka、Config、Zuul](https://blog.csdn.net/weixin_34556470/article/details/85001182)
>
> [关于docker里面为什么要运行linux容器的疑问？求大神科普。](https://bbs.csdn.net/topics/393081770)
>
> [docker的宿主系统是centos，为什么可以运行ubuntu的镜像呢？](https://blog.csdn.net/llsmingyi/article/details/79255373)
>
> [请教为什么大多数使用 docker 是在 ubuntu/debian, 而 centos 下很少](https://www.v2ex.com/t/636092)
>
> [Docker中选择CentOS还是Ubuntu/Debian？](https://www.zhihu.com/question/32160729?sort=created)
>
> [Docker上定制CentOS7镜像](https://www.linuxidc.com/Linux/2018-12/155993.htm)
>
> [springcloud(十三)：注册中心 Consul 使用详解](https://blog.csdn.net/love_zngy/article/details/82216696)
>
> [Spring Cloud官方文档中文版-Spring Cloud Config（上）-服务端（配置中心）](https://www.cnblogs.com/dreamingodd/p/7737318.html)
>
> [spring-cloud-starter-security与spring-boot-starter-security有什么不同？](https://www.oschina.net/question/2489154_2283940)
>
> [我们能否在 spring-boot-starter-web 中用 jetty 代替 tomcat？](https://www.koofun.com/pro/queanswers?proquestionId=7505)

## X1. 开发中遇到的坑

1. springcloud多模块项目，每个项目执行maven package后生成的jar包大小很小，并不是可执行的jar包。

+ 解决方案(修改pom依赖文件)：
  1. spring-boot-maven-plugin的版本使用与当前springgboot相符合的
  2. 下面的`<configuration>`里面的`mainClass`修改值为自己服务启动类的全限制类名

```xml
<!-- 修改原本的build字段 -->

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <version>${springboot.version}</version>
            <configuration>
                <!-- 指定该Main Class为全局的唯一入口 -->
                <mainClass>com.ash.springcloud.TestApplication</mainClass>
                <layout>ZIP</layout>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>repackage</goal><!--可以把依赖的包都打包到生成的Jar包中-->
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

2. 共用的api模块，比如存放pojo、工具类utils，如果用Feign可能还存放service接口等。这个打包不需要生成可执行文件，但是记得，别的module模块要是使用到其po类等，需要在pom依赖文件中声明

```xml
<!-- api，示例如下，根据自己实际的api模块修改对应的3个值 -->
<dependency>
    <groupId>com.ash.springcloud</groupId>
    <artifactId>http-api-n0-0000</artifactId>
    <version>0.0.1-SNAPSHOT</version>
</dependency>
```

3. 引用到公共api模块的项目，除了在pom声明外，如果该项目mvn package成执行jar包，找不到po类之类的，那就是之前没有对api模块执行mvn clean -> mvn install。install才能把之前的公共api模块存到本地maven仓库，这时候别的模块的pom对api模块依赖才算真正起到作用（指可执行jar包可以使用到公共的po类）

+ api模块不需要打成可执行jar包，所以不需要在自带的`spring-boot-maven-plugin`里配置`mainClass`等

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

+ 假如provider用到了api模块的po类，那么要生成provider的可执行jar包需要执行以下几步骤
  1. 对api模块执行 mvn clean -> mvn install（注意一定要记得每次都install，因为我之前有次忘记install结果用的旧的api的po类，打包没问题，但是运行项目mybatis提示mapper里面的属性却好啊setter之类的。因为用的不是最新的po类！！）
  2. provider的pom依赖中打包插件`spring-boot-maven-plugin`里配置`mainClass`等属性（前面提过了）。
  3. provider模块执行 mvn clean -> mvn package就可以生成而执行jar包了

4. 项目布置在几台不同公网IP的服务器，每个Eureka互相注册，需要用到真实ip。

```yaml
# 服务启动项
server:
  port: 7001

#spring配置
spring:
  profiles: dev
  application:
    name: http-eureka-dev-n1-7001
 # security: # 使得Eureka需要账号密码才能访问 ,没有用到 security依赖就不用。之前文章应该有提过这个配置了
 #   user:
 #     name: test
 #     password: 123456
 #     roles: SUPERUSER

#eureka配置
eureka:
  server:
    enable-self-preservation: false
  instance:
    hostname: http-eureka-n1-7001 
    appname: http-eureka-7001
    instance-id: n1-XXX.YYY.ZZZ.NNN
    prefer-ip-address: true # 使用真实的ip注册
    ip-address: XXX.YYY.ZZZ.NNN # 真实的公网ip
  client: 
    service-url: 
      #单机 defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
      #设置与Eureka Server交互的地址查询服务和注册服务都需要依赖这个地址（单机）。
      defaultZone: #这个自己根据实际情况配置了

```

5. Provider注册到Eureka后，Ribbon根据服务名访问不到

+ 可能是没有设置`spring.application.name`属性，Ribbon负载均衡要用到的serviceId就是这个。
+ 因为是不同公网IP服务器，加上如果用到docker之类的，Eureka会识别成本地局域网网桥的ip地址，需要自己手动配置

```yaml
# 服务启动项
server:
  port: 8081
  
# spring配置
spring:
  profiles: prod
  application:
    name: http-provider-n1 # Ribbon负载均衡要用到的serviceId就是这个
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    driver-class-name: com.mysql.cj.jdbc.Driver # 用的mysql 8
    url: jdbc:mysql://服务器IP:3306/表名?useUnicode=true&serverTimezone=Asia/Shanghai&characterEncoding=UTF-8&useSSL=false
    username: 数据库账号
    password: 数据库密码
  
#mybatis配置
mybatis:
  type-aliases-package: com.ash.springcloud.po
  config-location: classpath:mybatis/mybatis-config.xml
  mapper-locations: classpath:mybatis/mapper/*.xml
  
#eureka配置
eureka:
  instance:
    appname: http-provider-n1
    instance-id: n1-XXX.YYY.ZZZ.NNN
    prefer-ip-address: true
    ip-address: 你的服务器公网ip  # 服务提供者通过我们自己指定的ip注册到Eureka
    non-secure-port: 你的服务用到的端口port # 服务提供者通过我们自己指定的port注册到Eureka
  client:
    service-url:
      defaultZone: # 这个根据你自己的实际配置
  
#info配置
info: #点击Eureka的页面的服务的Status进入的页面显示的信息
  app.name: provider
  author: ash
```

