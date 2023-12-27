# 《bytebuddy进阶实战》-学习笔记

> + 视频教程: [bytebuddy进阶实战-skywalking agent可插拔式架构实现_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1Jv4y1a7Kw)
>
> + ByteBuddy github项目: [raphw/byte-buddy: Runtime code generation for the Java virtual machine. (github.com)](https://github.com/raphw/byte-buddy)
>
> + ByteBuddy官方教程: [Byte Buddy - runtime code generation for the Java virtual machine](http://bytebuddy.net/#/tutorial-cn)
>
> + SkyWalking官网: [Apache SkyWalking](https://skywalking.apache.org/)
>
> + ByteBuddy个人学习中记录demo代码的github仓库: [Ashiamd/ash_bytebuddy_study](https://github.com/Ashiamd/ash_bytebuddy_study/tree/main)
>
> + SkyWalking Agent个人学习中记录demo代码的github仓库: [Ashiamd/ash_skywalking_agent_study](https://github.com/Ashiamd/ash_skywalking_agent_study)
>
> ---
>
> 下面的图片全部来自网络博客/文章
>
> 下面学习笔记主要由视频内容, 官方教程, 网络文章, 个人简述组成。

# 1. 网络文章收集

> [深入理解 Skywalking Agent - 简书 (jianshu.com)](https://www.jianshu.com/p/61d2edbe2275) <= 特别棒的文章，推荐阅读

# 2. SkyWalking简述

**SkyWalking**: an APM(application performance monitor) system, especially designed for microservices, cloud native and container-based architectures.

![img](https://upload-images.jianshu.io/upload_images/23610677-1bd4fe5f5ea9b253.png?imageMogr2/auto-orient/strip|imageView2/2/w/943/format/webp)

​	SkyWalking主要可以划分为4部分：

+ Probes: agent项目（sniffer）
+ Platform backend：oap
+ Storage：oap使用的存储
+ UI: 界面

​	大致流程即：

（1）agent(sniff)探针在服务端程序中执行SkyWalking的插件的拦截器逻辑，完成监测数据获取

（2）agent(sniff)探针通过gRPC(默认)等形式将获取的监测数据传输到OAP系统

（3）OAP系统将数据进行处理/整合后，存储到Storage中(ES, MySQL, H2等)

（4）用户可通过UI界面快捷查询数据，了解请求调用链路，调用耗时等信息

---

> [Overview | Apache SkyWalking](https://skywalking.apache.org/docs/main/v9.7.0/en/concepts-and-designs/overview/)

![img](https://skywalking.apache.org/images/home/architecture_2160x720.png?t=20220617)

- **Probe**s collect telemetry data, including metrics, traces, logs and events in various formats(SkyWalking, Zipkin, OpenTelemetry, Prometheus, Zabbix, etc.)
- **Platform backend** supports data aggregation, analysis and streaming process covers traces, metrics, logs and events. Work as Aggregator Role, Receiver Role or both.
- **Storage** houses SkyWalking data through an open/plugable interface. You can choose an existing implementation, such as ElasticSearch, H2, MySQL, TiDB, BanyanDB, or implement your own.
- **UI** is a highly customizable web based interface allowing SkyWalking end users to visualize and manage SkyWalking data.

# 3. SkyWalking Agent

> [深入理解 Skywalking Agent - 简书 (jianshu.com)](https://www.jianshu.com/p/61d2edbe2275) <= 推荐先阅读，很详细

+ 基础介绍

  SkyWalking Agent，是SkyWalking中的组件之一(`skywalking-agent.jar`)，在Java中通过Java Agent实现。

+ 使用简述

  （1）通过`java -javaagent:skywalking-agent.jar包的绝对路径 -jar 目标服务jar包绝对路径`指令，在目标服务main方法执行前，会先执行`skywalking-agent.jar`的`org.apache.skywalking.apm.agent.SkyWalkingAgent#premain`方法

  （2）premian方法执行时，加载与`skywalking-agent.jar`同层级文件目录的`./activations`和`./plusgins`目录内的所有插件jar包，根据jar文件内的`skywalking-plugin.def`配置文件作相关解析工作

  （3）解析工作完成后，将所有插件实现的拦截器逻辑，通过JVM工具类`Instrumentation`提供的redefine/retransform能力，修改目标类的`.class`内容，将拦截器内指定的增强逻辑附加到被拦截的类的原方法实现中

  （4）目标服务的main方法执行，此时被拦截的多个类内部方法逻辑已经被增强，比如某个方法执行前后额外通过gRPC将方法耗时记录并发送到OAP。简单理解的话，类似Spring中常用的AOP切面技术，这里相当于字节码层面完成切面增强

+ 实现思考

  Byte Buddy可以指定一个拦截器对指定拦截的类中指定的方法进行增强。SkyWalking Java Agent使用ByteBuddy实现，从0到1实现时，避不开以下几个问题需要考虑：

  （1）每个插件都需要有各自的拦截器逻辑，如何让Byte Buddy使用我们指定的多个拦截器

  （2）多个拦截器怎么区分各自需要拦截的类和方法

  （3）如何在premain中使用Byte Buddy时加载多个插件的拦截器逻辑

# 4. SkyWalking Agent Demo实现

## 4.1 学习目标

1. 下面跟着视频教程完成Java版本的SkyWalking Agent实现。因为视频主要是对Byte Buddy实际应用的扩展讲解，所以下面demo编写时，也主要围绕和Byte Buddy有关的部分，如：

+ 通过Byte Buddy完成不同插件不同类拦截范围的需求
+ 通过Byte Buddy完成不同插件不同方法拦截范围的需求
+ 通过Byte Buddy完成不同插件不同增强逻辑的需求

2. 学习SkyWalking Agent Demo实现中的一些代码设计技巧

   不同的工具，框架等，往往根据其使用场景，都有特别的设计或实现思想，值得学习和效仿

## 4.2 阶段一: Byte Buddy Agent回顾

### 4.2.1 目标

1. 完成项目初始化搭建
2. 回顾Byte Buddy Agent的使用

### 4.2.2 代码实现

1. 搭建简易的SpringBoot项目
2. 分别编写MySQL和SpringMVC的Byte Buddy Agent代码
3. 指定多个javagent启动参数，观察MySQL和SpringMVC的agent代码是否生效

### 4.2.3 小结

1. 使用多个`-javaagent:{agent的jar包绝对路径}`指定使用多个javaagent

## 4.3 阶段二: 可插拔分析

### 4.3.1 目标

### 4.3.2 代码实现

### 4.3.3 小结

## 4.4 阶段三: 抽象插件拦截器

### 4.4.1 目标

### 4.4.2 代码实现

### 4.4.3 小结

## 4.5 阶段四: 



