# RocketMQ官方文档-学习笔记

> [为什么选择RocketMQ | RocketMQ (apache.org)](https://rocketmq.apache.org/zh/docs/) <= 下面的图文基本都来自官方文档

# 1. 基本概念

## 1.1 MQ产品对比

​	Apache RocketMQ 自诞生以来，因其架构简单、业务功能丰富、具备极强可扩展性等特点被众多企业开发者以及云厂商广泛采用。历经十余年的大规模场景打磨，RocketMQ 已经成为业内共识的金融级可靠业务消息首选方案，被广泛应用于互联网、大数据、移动互联网、物联网等领域的业务场景。

| Messaging Product | Client SDK           | Protocol and Specification                           | Ordered Message                                              | Scheduled Message | Batched Message                                 | BroadCast Message | Message Filter                                          | Server Triggered Redelivery | Message Storage                                              | Message Retroactive                          | Message Priority | High Availability and Failover                               | Message Track | Configuration                                                | Management and Operation Tools                         |
| ----------------- | -------------------- | ---------------------------------------------------- | ------------------------------------------------------------ | ----------------- | ----------------------------------------------- | ----------------- | ------------------------------------------------------- | --------------------------- | ------------------------------------------------------------ | -------------------------------------------- | ---------------- | ------------------------------------------------------------ | ------------- | ------------------------------------------------------------ | ------------------------------------------------------ |
| ActiveMQ          | Java, .NET, C++ etc. | Push model, support OpenWire, STOMP, AMQP, MQTT, JMS | Exclusive Consumer or Exclusive Queues can ensure ordering   | Supported         | Not Supported                                   | Supported         | Supported                                               | Not Supported               | Supports very fast persistence using JDBC along with a high performance journal，such as levelDB, kahaDB | Supported                                    | Supported        | Supported, depending on storage,if using levelDB it requires a ZooKeeper server | Not Supported | The default configuration is low level, user need to optimize the configuration parameters | Supported                                              |
| Kafka             | Java, Scala etc.     | Pull model, support TCP                              | Ensure ordering of messages within a partition               | Not Supported     | Supported, with async producer                  | Not Supported     | Supported, you can use Kafka Streams to filter messages | Not Supported               | High performance file storage                                | Supported offset indicate                    | Not Supported    | Supported, requires a ZooKeeper server                       | Not Supported | Kafka uses key-value pairs format for configuration. These values can be supplied either from a file or programmatically. | Supported, use terminal command to expose core metrics |
| RocketMQ          | Java, C++, Go        | Pull model, support TCP, JMS, OpenMessaging          | Ensure strict ordering of messages,and can scale out gracefully | Supported         | Supported, with sync mode to avoid message loss | Supported         | Supported, property filter expressions based on SQL92   | Supported                   | High performance and low latency file storage                | Supported timestamp and offset two indicates | Not Supported    | Supported, Master-Slave model, without another kit           | Supported     | Work out of box,user only need to pay attention to a few configurations |                                                        |

> [消息中间件push和pull模式 - 简书 (jianshu.com)](https://www.jianshu.com/p/714afde33af6)
>
> ![img](https://upload-images.jianshu.io/upload_images/17019759-1ed2f7f70a5a6691.png?imageMogr2/auto-orient/strip|imageView2/2/w/780/format/webp)
>
> **Push Model 示例**
>
> 1. 生产者发送消息到消息队列。
> 2. 消息队列检测到有新的消费者注册，将消息推送给消费者。
> 3. 消费者处理消息后，向消息队列发送确认。
>
> **Pull Model 示例**
>
> 1. 生产者发送消息到消息队列。
> 2. 消费者定期轮询消息队列，请求新的消息。
> 3. 消息队列返回可用的消息给消费者。
> 4. 消费者处理完消息后，向消息队列发送确认。
>
> **总结**
>
> **Push Model** 更适用于实时性要求较高的场景，而 **Pull Model** 更适合于可靠性要求较高的场景。

## 1.2 基本概念

1. 主题(Topic)

   Apache RocketMQ 中消息传输和存储的顶层容器，用于标识同一类业务逻辑的消息。主题通过TopicName来做唯一标识和区分。更多信息，请参见[主题（Topic）](https://rocketmq.apache.org/zh/docs/domainModel/02topic)。

2. 消息类型(MessageType)

   Apache RocketMQ 中按照消息传输特性的不同而定义的分类，用于类型管理和安全校验。 Apache RocketMQ 支持的消息类型有：

   + 普通消息
   + 顺序消息
   + 事务消息
   + 定时/延时消息

   > Apache RocketMQ 从5.0版本开始，支持强制校验消息类型，即每个主题Topic只允许发送一种消息类型的消息，这样可以更好的运维和管理生产系统，避免混乱。但同时保证向下兼容4.x版本行为，强制校验功能默认开启。

   > 虽然 Kafka 不像 RocketMQ 那样显式地区分消息类型，但它也支持一些类似于 RocketMQ 的功能，例如：
   >
   > - **顺序消息**：通过将所有需要按顺序消费的消息发送到同一个分区，**Kafka 可以保证消息在单个分区内的顺序**。
   > - **事务消息**：Kafka 支持事务性的消息生产和消费，可以保证消息的原子性和一致性。
   > - **定时消息**：虽然 Kafka 没有直接提供定时消息的功能，但可以通过将消息发送到未来的偏移量来模拟定时消息的效果。
   >
   > [Kafka事务是怎么实现的？Kafka事务消息原理详解-CSDN博客](https://blog.csdn.net/weixin_44837153/article/details/136474339)

3. 消息队列(MessageQueue)

   **队列是 Apache RocketMQ 中消息存储和传输的实际容器，也是消息的最小存储单元**。 Apache RocketMQ 的所有主题都是由多个队列组成，以此实现队列数量的水平拆分和队列内部的流式存储。队列通过QueueId来做唯一标识和区分。更多信息，请参见[队列（MessageQueue）](https://rocketmq.apache.org/zh/docs/domainModel/03messagequeue)。

4. 消息(Message)

   **消息是 Apache RocketMQ 中的最小数据传输单元**。生产者将业务数据的负载和拓展属性包装成消息发送到服务端，服务端按照相关语义将消息投递到消费端进行消费。更多信息，请参见[消息（Message）](https://rocketmq.apache.org/zh/docs/domainModel/04message)。

5. 消息视图(MessageView)

   **消息视图是 Apache RocketMQ 面向开发视角提供的一种消息只读接口**。通过消息视图可以读取消息内部的多个属性和负载信息，但是不能对消息本身做任何修改。

6. 消息标签(MessageTag)

   消息标签是Apache RocketMQ 提供的细粒度消息分类属性，可以在主题层级之下做消息类型的细分。消费者通过订阅特定的标签来实现细粒度过滤。更多信息，请参见[消息过滤](https://rocketmq.apache.org/zh/docs/featureBehavior/07messagefilter)。

7. 消息位点(MessageQueueOffset)

   消息是按到达Apache RocketMQ 服务端的先后顺序存储在指定主题的多个队列中，每条消息在队列中都有一个唯一的Long类型坐标，这个坐标被定义为消息位点。更多信息，请参见[消费进度管理](https://rocketmq.apache.org/zh/docs/featureBehavior/09consumerprogress)。

8. 消费位点(onsumerOffset)

   一条消息被某个消费者消费完成后不会立即从队列中删除，Apache RocketMQ 会基于每个消费者分组记录消费过的最新一条消息的位点，即消费位点。更多信息，请参见[消费进度管理](https://rocketmq.apache.org/zh/docs/featureBehavior/09consumerprogress)。

9. 消息索引(MessageKey)

   消息索引是Apache RocketMQ 提供的面向消息的索引属性。通过设置的消息索引可以快速查找到对应的消息内容。

10. 生产者(Producer)

    生产者是Apache RocketMQ 系统中用来构建并传输消息到服务端的运行实体。生产者通常被集成在业务系统中，将业务消息按照要求封装成消息并发送至服务端。更多信息，请参见[生产者（Producer）](https://rocketmq.apache.org/zh/docs/domainModel/04producer)。

11. 事务检查器(TransactionChecker)

    Apache RocketMQ 中生产者用来执行本地事务检查和异常事务恢复的监听器。事务检查器应该通过业务侧数据的状态来检查和判断事务消息的状态。更多信息，请参见[事务消息](https://rocketmq.apache.org/zh/docs/featureBehavior/04transactionmessage)。

12. 事务状态(TransactionResolution)

    Apache RocketMQ 中事务消息发送过程中，事务提交的状态标识，服务端通过事务状态控制事务消息是否应该提交和投递。事务状态包括事务提交、事务回滚和事务未决。更多信息，请参见[事务消息](https://rocketmq.apache.org/zh/docs/featureBehavior/04transactionmessage)。

13. 消费者分组(ConsumerGroup)

    消费者分组是Apache RocketMQ 系统中承载多个消费行为一致的消费者的负载均衡分组。和消费者不同，消费者分组并不是运行实体，而是一个逻辑资源。在 Apache RocketMQ 中，通过消费者分组内初始化多个消费者实现消费性能的水平扩展以及高可用容灾。更多信息，请参见[消费者分组（ConsumerGroup）](https://rocketmq.apache.org/zh/docs/domainModel/07consumergroup)。

14. 消费者(Consumer)

    消费者是Apache RocketMQ 中用来接收并处理消息的运行实体。消费者通常被集成在业务系统中，从服务端获取消息，并将消息转化成业务可理解的信息，供业务逻辑处理。更多信息，请参见[消费者（Consumer）](https://rocketmq.apache.org/zh/docs/domainModel/08consumer)。

15. 消费结果(ConsumeResult)

    Apache RocketMQ 中PushConsumer消费监听器处理消息完成后返回的处理结果，用来标识本次消息是否正确处理。消费结果包含消费成功和消费失败。

16. 订阅关系(Subscription)

    订阅关系是Apache RocketMQ 系统中消费者获取消息、处理消息的规则和状态配置。订阅关系由消费者分组动态注册到服务端系统，并在后续的消息传输中按照订阅关系定义的过滤规则进行消息匹配和消费进度维护。更多信息，请参见[订阅关系（Subscription）](https://rocketmq.apache.org/zh/docs/domainModel/09subscription)。

17. 消息过滤

    消费者可以通过订阅指定消息标签（Tag）对消息进行过滤，确保最终只接收被过滤后的消息合集。过滤规则的计算和匹配在Apache RocketMQ 的服务端完成。更多信息，请参见[消息过滤](https://rocketmq.apache.org/zh/docs/featureBehavior/07messagefilter)。

18. 重置消费位点

    以时间轴为坐标，在消息持久化存储的时间范围内，重新设置消费者分组对已订阅主题的消费进度，设置完成后消费者将接收设定时间点之后，由生产者发送到Apache RocketMQ 服务端的消息。更多信息，请参见[重置消费位点](https://rocketmq.apache.org/zh/docs/featureBehavior/09consumerprogress)。

19. 消息轨迹

    在一条消息从生产者发出到消费者接收并处理过程中，由各个相关节点的时间、地点等数据汇聚而成的完整链路信息。通过消息轨迹，您能清晰定位消息从生产者发出，经由Apache RocketMQ 服务端，投递给消费者的完整链路，方便定位排查问题。

20. 消息堆积

    生产者已经将消息发送到Apache RocketMQ 的服务端，但由于消费者的消费能力有限，未能在短时间内将所有消息正确消费掉，此时在服务端保存着未被消费的消息，该状态即消息堆积。

21. 事务消息

    **事务消息是Apache RocketMQ 提供的一种高级消息类型，支持在分布式场景下保障消息生产和本地事务的最终一致性**。

22. 定时/延时消息

    **定时/延时消息是Apache RocketMQ 提供的一种高级消息类型，消息被发送至服务端后，在指定时间后才能被消费者消费。通过设置一定的定时时间可以实现分布式场景的延时调度触发效果**。

23. 顺序消息

    **顺序消息是Apache RocketMQ 提供的一种高级消息类型，支持消费者按照发送消息的先后顺序获取消息，从而实现业务场景中的顺序处理**。

## 1.3 参数约束和建议

Apache RocketMQ 系统中存在很多自定义参数和资源命名，您在使用 Apache RocketMQ 时建议参考如下说明规范系统设置，避对某些具体参数设置不合理导致应用出现异常。

| 参数                         | 建议范围                                                     | 说明                                                         |
| ---------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Topic名称                    | 字符建议：字母a~z或A~Z、数字0~9以及下划线（_）、短划线（\-）和百分号（%）。 长度建议：1~64个字符。 系统保留字符：Topic名称不允许使用以下保留字符或含有特殊前缀的字符命名。 保留字符: TBW102 \*BenchmarkTest\* SELF_TEST_TOPIC \*OFFSET_MOVED_EVENT\* SCHEDULE_TOPIC_XXXX \*RMQ_SYS_TRANS_HALF_TOPIC\* RMQ_SYS_TRACE_TOPIC \*RMQ_SYS_TRANS_OP_HALF_TOPIC 特殊前缀:\* rmq_sys* %RETRY% *%DLQ%* rocketmq-broker- | Topic命名应该尽量使用简短、常用的字符，避免使用特殊字符。特殊字符会导致系统解析出现异常，字符过长可能会导致消息收发被拒绝。 |
| ConsumerGroup名称            | 字符建议：支持字母a~z或A~Z、数字0~9以及下划线（\_）、短划线（-）和百分号（%）。 长度建议：1~64个字符。 系统保留字符：ConsumerGroup不允许使用以下保留字符或含有特殊前缀的字符命名。 保留字符: \*DEFAULT_CONSUMER\* DEFAULT_PRODUCER \*TOOLS_CONSUMER\* FILTERSRV_CONSUMER \*__MONITOR_CONSUMER\* CLIENT_INNER_PRODUCER \*SELF_TEST_P_GROUP\* SELF_TEST_C_GROUP \*CID_ONS-HTTP-PROXY\* CID_ONSAPI_PERMISSION \*CID_ONSAPI_OWNER\* CID_ONSAPI_PULL \*CID_RMQ_SYS_TRANS\* 特殊字符 \* CID_RMQ_SYS* * CID_HOUSEKEEPING | 无。                                                         |
| ACL Credentials              | 字符建议：AK（AccessKey ID）、SK（AccessKey Secret）和Token仅支持字母a~z或A~Z、数字0~9。 长度建议：不超过1024个字符。 | 无。                                                         |
| 请求超时时间                 | **默认值：3000毫秒。 取值范围：该参数为客户端本地行为，取值范围建议不要超过30000毫秒。** | 请求超时时间是客户端本地同步调用的等待时间，请根据实际应用设置合理的取值，避免线程阻塞时间过长。 |
| 消息大小                     | **默认值：不超过4 MB。不涉及消息压缩，仅计算消息体body的大小。 取值范围：建议不超过4 MB。** | 消息传输应尽量压缩和控制负载大小，避免超大文件传输。若消息大小不满足限制要求，可以尝试分割消息或使用OSS存储，用消息传输URL。 |
| 消息自定义属性               | 字符限制：所有可见字符。 长度建议：属性的Key和Value总长度不超过16 KB。 系统保留属性：不允许使用以下保留属性作为自定义属性的Key。 保留属性Key | 无。                                                         |
| MessageGroup                 | 字符限制：所有可见字符。 长度建议：1~64字节。                | MessageGroup是顺序消息的分组标识。一般设置为需要保证顺序的一组消息标识，例如订单ID、用户ID等。 |
| 消息发送重试次数             | 默认值：3次。 取值范围：无限制。                             | 消息发送重试是客户端SDK内置的重试策略，对应用不可见，建议取值不要过大，避免阻塞业务线程。 如果消息达到最大重试次数后还未发送成功，建议业务侧做好兜底处理，保证消息可靠性。 |
| 消息消费重试次数             | 默认值：16次。                                               | 消费重试次数应根据实际业务需求设置合理的参数值，避免使用重试进行无限触发。重试次数过大容易造成系统压力过量增加。 |
| 事务异常检查间隔             | 默认值：60秒。                                               | 事务异常检查间隔指的是，半事务消息因系统重启或异常情况导致没有提交，生产者客户端会按照该间隔时间进行事务状态回查。 **间隔时长不建议设置过短，否则频繁的回查调用会影响系统性能。** |
| 半事务消息第一次回查时间     | 默认值：取值等于[事务异常检查间隔] * 最大限制：不超过1小时。 | 无。                                                         |
| 半事务消息最大超时时长       | 默认值：4小时。 * 取值范围：不支持自定义修改。               | **半事务消息因系统重启或异常情况导致没有提交，生产者客户端会按照事务异常检查间隔时间进行回查，若超过半事务消息超时时长后没有返回结果，半事务消息将会被强制回滚**。 您可以通过监控该指标避免异常事务。 |
| PushConsumer本地缓存         | 默认值： *最大缓存数量：1024条。 *最大缓存大小：64 M。 取值范围：支持用户自定义设置，无限制。 | 消费者类型为PushConsumer时，为提高消费者吞吐量和性能，客户端会在SDK本地缓存部分消息。缓存的消息的数量和大小应设置在系统内存允许的范围内。 |
| PushConsumer重试间隔时长     | 默认值： *非顺序性投递：间隔时间阶梯变化，具体取值，请参见PushConsumer消费重试策略。 *顺序性投递：3000毫秒。 | 无。                                                         |
| PushConsumer消费并发度       | 默认值：20个线程。                                           | 无。                                                         |
| 获取消息最大批次             | 默认值：32条。                                               | 消费者从服务端获取消息时，一次获取到最大消息条数。建议按照实际业务设置合理的参数值，**一次获取消息数量过大容易在消费失败时造成大批量消息重复**。 |
| SimpleConsumer最大不可见时间 | 默认值：用户必填参数，无默认值。 取值范围建议：最小10秒；最大12小时。 | **消费不可见时间指的是消息处理+失败后重试间隔的总时长，建议设置时取值比实际需要耗费的时间稍微长一些**。 |

# 2. 快速开始

> [本地部署 RocketMQ | RocketMQ (apache.org)](https://rocketmq.apache.org/zh/docs/quickStart/01quickstart)

## 2.1 本地部署RocketMQ

## 2.2 Docker部署RocketMQ

## 2.3 Docker Compose部署RocketMQ

# 3. 领域模型

## 3.1 领域模型介绍

### 3.1.1 Apache RocketMQ领域模型

Apache RocketMQ 是一款典型的分布式架构下的中间件产品，使用**异步通信**方式和**发布订阅**的消息传输模型。通信方式和传输模型的具体说明，请参见下文**通信方式介绍**和**消息传输模型**介绍。 Apache RocketMQ 产品具备异步通信的优势，系统拓扑简单、上下游耦合较弱，主要应用于异步解耦，<u>流量削峰填谷</u>等场景。

![领域模型](https://rocketmq.apache.org/zh/assets/images/mainarchi-9b036e7ff5133d050950f25838367a17.png)

如上图所示，Apache RocketMQ 中消息的生命周期主要分为消息生产、消息存储、消息消费这三部分。

生产者生产消息并发送至 Apache RocketMQ 服务端，消息被存储在服务端的主题中，消费者通过订阅主题消费消息。

**消息生产**

+ [生产者（Producer）](https://rocketmq.apache.org/zh/docs/domainModel/04producer)：

  Apache RocketMQ 中用于产生消息的运行实体，一般集成于业务调用链路的上游。<u>生产者是轻量级匿名无身份的</u>。

**消息存储**

- [主题（Topic）](https://rocketmq.apache.org/zh/docs/domainModel/02topic)：

  Apache RocketMQ 消息传输和存储的分组容器，主题内部由多个队列组成，**消息的存储和水平扩展实际是通过主题内的队列实现的**。

  > Kafka的Topic则是由多个parition分区组成，然后只保证每个分区内消息有序，不保证整个topic范围消息有序。和RocketMQ的topic组成部分队列概念类似。

- [队列（MessageQueue）](https://rocketmq.apache.org/zh/docs/domainModel/03messagequeue)：

  Apache RocketMQ 消息传输和存储的实际单元容器，类比于其他消息队列中的分区。 Apache RocketMQ 通过流式特性的无限队列结构来存储消息，消息在队列内具备顺序性存储特征。

- [消息（Message）](https://rocketmq.apache.org/zh/docs/domainModel/04message)：

  Apache RocketMQ 的最小传输单元。消息具备不可变性，在初始化发送和完成存储后即不可变。

**消息消费**

- [消费者分组（ConsumerGroup）](https://rocketmq.apache.org/zh/docs/domainModel/07consumergroup)：

  Apache RocketMQ 发布订阅模型中定义的独立的消费身份分组，用于统一管理底层运行的多个消费者（Consumer）。同一个消费组的多个消费者必须保持消费逻辑和配置一致，共同分担该消费组订阅的消息，实现消费能力的水平扩展。

- [消费者（Consumer）](https://rocketmq.apache.org/zh/docs/domainModel/08consumer)：

  Apache RocketMQ 消费消息的运行实体，一般集成在业务调用链路的下游。消费者必须被指定到某一个消费组中。

- [订阅关系（Subscription）](https://rocketmq.apache.org/zh/docs/domainModel/09subscription)：

  Apache RocketMQ 发布订阅模型中消息过滤、重试、消费进度的规则配置。**订阅关系以消费组粒度进行管理**，消费组通过定义订阅关系控制指定消费组下的消费者如何实现消息过滤、消费重试及消费进度恢复等。

  Apache RocketMQ 的订阅关系除过滤表达式之外都是持久化的，即服务端重启或请求断开，订阅关系依然保留。

### 3.1.2 通信方式介绍

分布式系统架构思想下，将复杂系统拆分为多个独立的子模块，例如微服务模块。此时就需要考虑子模块间的远程通信，典型的通信模式分为以下两种，一种是同步的RPC远程调用；一种是基于中间件代理的异步通信方式。

同步RPC调用模型 ![同步调用](https://rocketmq.apache.org/zh/assets/images/syncarchi-ebbd41e1afd6adf432792ee2d7a91748.png)

同步RPC调用模型下，不同系统之间直接进行调用通信，每个请求直接从调用方发送到被调用方，然后要求被调用方立即返回响应结果给调用方，以确定本次调用结果是否成功。 **注意** 此处的同步并不代表RPC的编程接口方式，RPC也可以有异步非阻塞调用的编程方式，但本质上仍然是需要在指定时间内得到目标端的直接响应。

异步通信模型 ![异步调用](https://rocketmq.apache.org/zh/assets/images/asyncarchi-e7ee18dd77aca472fb80bb2238d9528b.png)

异步消息通信模式下，各子系统之间无需强耦合直接连接，调用方只需要将请求转化成异步事件（消息）发送给中间代理，发送成功即可认为该异步链路调用完成，剩下的工作中间代理会负责将事件可靠通知到下游的调用系统，确保任务执行完成。该中间代理一般就是消息中间件。

异步通信的优势如下：

- 系统拓扑简单。由于调用方和被调用方统一和中间代理通信，系统是星型结构，易于维护和管理。

- 上下游耦合性弱。上下游系统之间弱耦合，结构更灵活，由中间代理负责缓冲和异步恢复。 上下游系统间可以独立升级和变更，不会互相影响。

- 容量削峰填谷。基于消息的中间代理往往具备很强的流量缓冲和整形能力，业务流量高峰到来时不会击垮下游。

### 3.1.3 消息传输模型介绍

主流的消息中间件的传输模型主要为点对点模型和发布订阅模型。

点对点模型 ![点对点模型](https://rocketmq.apache.org/zh/assets/images/p2pmode-fefdc2fbe4792e757e26befc0b3acbff.png)

点对点模型也叫队列模型，具有如下特点：

- 消费匿名：消息上下游沟通的唯一的身份就是队列，下游消费者从队列获取消息无法申明独立身份。
- 一对一通信：基于消费匿名特点，下游消费者即使有多个，但都没有自己独立的身份，因此共享队列中的消息，每一条消息都只会被唯一一个消费者处理。因此点对点模型只能实现一对一通信。

发布订阅模型 ![发布订阅模型](https://rocketmq.apache.org/zh/assets/images/pubsub-042a4e5e5d76806943bd7dcfb730c5d5.png)

发布订阅模型具有如下特点：

- 消费独立：相比队列模型的匿名消费方式，发布订阅模型中消费方都会具备的身份，一般叫做订阅组（订阅关系），不同订阅组之间相互独立不会相互影响。
- 一对多通信：**基于独立身份的设计，同一个主题内的消息可以被多个订阅组处理，每个订阅组都可以拿到全量消息**。因此发布订阅模型可以实现一对多通信。

传输模型对比

点对点模型和发布订阅模型各有优势，点对点模型更为简单，而发布订阅模型的扩展性更高。

**Apache RocketMQ 使用的传输模型为发布订阅模型，因此也具有发布订阅模型的特点**。

## 3.2 主题(Topic)

本文介绍 Apache RocketMQ 中主题（Topic）的定义、模型关系、内部属性、行为约束、版本兼容性及使用建议。

### 3.2.1 定义

主题是 Apache RocketMQ 中消息传输和存储的顶层容器，用于标识同一类业务逻辑的消息。 主题的作用主要如下：

- **定义数据的分类隔离：** 在 Apache RocketMQ 的方案设计中，<u>建议将不同业务类型的数据拆分到不同的主题中管理，通过主题实现存储的隔离性和订阅隔离性</u>。
- **定义数据的身份和权限：** Apache RocketMQ 的<u>消息本身是匿名无身份的</u>，同一分类的消息使用相同的主题来做身份识别和权限管理。

### 3.2.2 模型关系

在整个 Apache RocketMQ 的领域模型中，主题所处的流程和位置如下：

![主题](https://rocketmq.apache.org/zh/assets/images/archifortopic-ef512066703a22865613ea9216c4c300.png)

主题是 Apache RocketMQ 的顶层存储，所有消息资源的定义都在主题内部完成，但**主题是一个逻辑概念，并不是实际的消息容器**。

**<u>主题内部由多个队列组成，消息的存储和水平扩展能力最终是由队列实现的；并且针对主题的所有约束和属性设置，最终也是通过主题内部的队列来实现</u>**。

### 3.2.3 内部属性

**主题名称**

- 定义：主题的名称，用于标识主题，<u>主题名称集群内全局唯一</u>。
- 取值：由用户创建主题时定义。
- 约束：请参见[参数限制](https://rocketmq.apache.org/zh/docs/introduction/03limits)。

**队列列表**

- 定义：队列作为主题的组成单元，是消息存储的实际容器，一个主题内包含一个或多个队列，消息实际存储在主题的各队列内。更多信息，请参见[队列（MessageQueue）](https://rocketmq.apache.org/zh/docs/domainModel/03messagequeue)。
- 取值：系统根据队列数量给主题分配队列，队列数量创建主题时定义。
- 约束：一个主题内至少包含一个队列。（类比Kafka的一个topic内至少一个分区partition）

**消息类型**

- 定义：主题所支持的消息类型。
- 取值：创建主题时选择消息类型。Apache RocketMQ 支持的主题类型如下：
  - Normal：[普通消息](https://rocketmq.apache.org/zh/docs/featureBehavior/01normalmessage)，消息本身无特殊语义，消息之间也没有任何关联。
  - FIFO：[顺序消息](https://rocketmq.apache.org/zh/docs/featureBehavior/03fifomessage)，Apache RocketMQ 通过消息分组MessageGroup标记一组特定消息的先后顺序，可以保证消息的投递顺序严格按照消息发送时的顺序。
  - Delay：[定时/延时消息](https://rocketmq.apache.org/zh/docs/featureBehavior/02delaymessage)，通过指定延时时间控制消息生产后不要立即投递，而是在延时间隔后才对消费者可见。
  - Transaction：[事务消息](https://rocketmq.apache.org/zh/docs/featureBehavior/04transactionmessage)，Apache RocketMQ 支持分布式事务消息，支持应用数据库更新和消息调用的事务一致性保障。
- **约束：Apache RocketMQ 从5.0版本开始，支持强制校验消息类型，即每个主题只允许发送一种消息类型的消息，这样可以更好的运维和管理生产系统，避免混乱。为保证向下兼容4.x版本行为，强制校验功能默认开启**。

### 3.2.4 行为约束

**消息类型强制校验**

Apache RocketMQ 5.x版本支持将消息类型拆分到主题中进行独立运维和处理，因此系统会对发送的消息类型和主题定的消息类型进行强制校验，若校验不通过，则消息发送请求会被拒绝，并返回类型不匹配异常。校验原则如下：

- **消息类型必须一致：发送的消息的类型，必须和目标主题定义的消息类型一致。**
- **主题类型必须单一：每个主题只支持一种消息类型，不允许将多种类型的消息发送到同一个主题中**。

> **为保证向下兼容4.x版本行为，上述强制校验功能默认开启**。

**常见错误使用场景**

- 发送的消息类型不匹配。例如：创建主题时消息类型定义为顺序消息，发送消息时发送事务消息到该主题中，此时消息发送请求会被拒绝，并返回类型不匹配异常。
- 单一消息主题混用。例如：创建主题时消息类型定义为普通消息，发送消息时同时发送普通消息和顺序消息到该主题中，则顺序消息的发送请求会被拒绝，并返回类型不匹配异常。

### 3.2.5 版本兼容性

消息类型的强制校验，仅 Apache RocketMQ 服务端5.x版本支持，且默认开启，推荐部署时打开配置。 Apache RocketMQ 服务端4.x和3.x历史版本的SDK不支持强制校验，您需要自己保证消息类型一致。 如果您使用的服务端版本为历史版本，建议您升级到 Apache RocketMQ 服务端5.x版本。

### 3.2.6 使用示例

Apache RocketMQ 5.0版本下创建主题操作，推荐使用mqadmin工具，需要注意的是，对于消息类型需要通过属性参数添加。示例如下：

```shell
sh mqadmin updateTopic -n <nameserver_address> -t <topic_name> -c <cluster_name> -a +message.type=<message_type>
```

其中message_type根据消息类型设置成Normal/FIFO/Delay/Transaction。如果不设置，默认为Normal类型。

### 3.2.7 使用建议

**按照业务分类合理拆分主题**

Apache RocketMQ 的主题拆分设计应遵循大类统一原则，即将相同业务域内同一功能属性的消息划分为同一主题。拆分主题时，您可以从以下角度考虑拆分粒度：

- 消息类型是否一致：不同类型的消息，如顺序消息和普通消息需要使用不同的主题。
- 消息业务是否关联：如果业务没有直接关联，比如，淘宝交易消息和盒马物流消息没有业务交集，需要使用不同的消息主题；同样是淘宝交易消息，女装类订单和男装类订单可以使用同一个订单。当然，如果业务量较大或其他子模块应用处理业务时需要进一步拆分订单类型，您也可以将男装订单和女装订单的消息拆分到两个主题中。
- 消息量级是否一样：数量级不同或时效性不同的业务消息建议使用不同的主题，例如某些业务消息量很小但是时效性要求很强，如果跟某些万亿级消息量的业务使用同一个主题，会增加消息的等待时长。

**正确拆分示例：** 线上商品购买场景下，订单交易如订单创建、支付、取消等流程消息使用一个主题，物流相关消息使用一个主题，积分管理相关消息使用一个主题。

**错误拆分示例：**

- 拆分粒度过粗：会导致业务隔离性差，不利于独立运维和故障处理。例如，所有交易消息和物流消息都共用一个主题。
- 拆分粒度过细：会消耗大量主题资源，造成系统负载过重。例如，按照用户ID区分，每个用户ID使用一个主题。

**单一主题只收发一种类型消息，避免混用**

Apache RocketMQ 主题的设计原则为通过主题隔离业务，不同业务逻辑的消息建议使用不同的主题。同一业务逻辑消息的类型都相同，因此，对于指定主题，应该只收发同一种类型的消息。

**主题管理尽量避免自动化机制**

在 Apache RocketMQ 架构中，主题属于顶层资源和容器，拥有独立的权限管理、可观测性指标采集和监控等能力，创建和管理主题会占用一定的系统资源。因此，生产环境需要严格管理主题资源，请勿随意进行增、删、改、查操作。

Apache RocketMQ 虽然提供了自动创建主题的功能，但是建议仅在测试环境使用，生产环境请勿打开，避免产生大量垃圾主题，无法管理和回收并浪费系统资源。

## 3.3 队列(MessageQueue)

本文介绍 Apache RocketMQ 中队列（MessageQueue）的定义、模型关系、内部属性、版本兼容性及使用建议。

### 3.3.1 定义

队列是 Apache RocketMQ 中消息存储和传输的实际容器，也是 Apache RocketMQ 消息的最小存储单元。 Apache RocketMQ 的所有主题都是由多个队列组成，以此实现队列数量的水平拆分和队列内部的流式存储。

队列的主要作用如下：

- 存储顺序性

  队列天然具备顺序性，即消息按照进入队列的顺序写入存储，同一队列间的消息天然存在顺序关系，队列头部为最早写入的消息，队列尾部为最新写入的消息。消息在队列中的位置和消息之间的顺序通过位点（Offset）进行标记管理。

- 流式操作语义

  Apache RocketMQ 基于队列的存储模型可确保消息从任意位点读取任意数量的消息，以此实现类似聚合读取、回溯读取等特性，这些特性是RabbitMQ、ActiveMQ等非队列存储模型不具备的。

### 3.3.2 模型关系

在整个 Apache RocketMQ 的领域模型中，队列所处的流程和位置如下：![队列](https://rocketmq.apache.org/zh/assets/images/archiforqueue-dd6788b33bf2fc96b4a1dab83a1b0d71.png)

**Apache RocketMQ 默认提供消息可靠存储机制，所有发送成功的消息都被持久化存储到队列中，配合生产者和消费者客户端的调用可实现至少投递一次的可靠性语义**。

**Apache RocketMQ 队列模型和Kafka的分区（Partition）模型类似**。在 Apache RocketMQ 消息收发模型中，队列属于主题的一部分，虽然所有的消息资源以主题粒度管理，但实际的操作实现是面向队列。例如，生产者指定某个主题，向主题内发送消息，但实际消息发送到该主题下的某个队列中。

Apache RocketMQ 中通过修改队列数量，以此实现横向的水平扩容和缩容。

### 3.3.3 内部属性

读写权限

- 定义：当前队列是否可以读写数据。
- 取值：由服务端定义，枚举值如下
  - 6：读写状态，当前队列允许读取消息和写入消息。
  - 4：只读状态，当前队列只允许读取消息，不允许写入消息。
  - 2：只写状态，当前队列只允许写入消息，不允许读取消息。
  - 0：不可读写状态，当前队列不允许读取消息和写入消息。

- 约束：队列的读写权限属于运维侧操作，不建议频繁修改。

### 3.3.4 行为约束

每个主题下会由一到多个队列来存储消息，每个主题对应的队列数与消息类型以及实例所处地域（Region）相关。

### 3.3.5 版本兼容性

队列的名称属性在 Apache RocketMQ 服务端的不同版本中有如下差异：

- 服务端3.x/4.x版本：队列名称由{主题名称}+{BrokerID}+{QueueID}三元组组成，和物理节点绑定。
- <u>服务端5.x版本：队列名称为一个集群分配的全局唯一的字符串组成，和物理节点解耦</u>。

**因此，在开发过程中，建议不要对队列名称做任何假设和绑定。如果您在代码中自定义拼接队列名称并和其他操作进行绑定，一旦服务端版本升级，可能会出现队列名称无法解析的兼容性问题**。

### 3.3.6 使用建议

**按照实际业务消耗设置队列数**

Apache RocketMQ 的队列数可在创建主题或变更主题时设置修改，队列数量的设置应遵循少用够用原则，避免随意增加队列数量。

主题内队列数过多可能对导致如下问题：

- 集群元数据膨胀

  <u>Apache RocketMQ 会以队列粒度采集指标和监控数据，队列过多容易造成管控元数据膨胀</u>。

- 客户端压力过大

  Apache RocketMQ 的消息读写都是针对队列进行操作，队列过多对应更多的轮询请求，增加系统负荷。

**常见队列增加场景**

- 需要增加队列实现物理节点负载均衡

  Apache RocketMQ 每个主题的多个队列可以分布在不同的服务节点上，在集群水平扩容增加节点后，为了保证集群流量的负载均衡，建议在新的服务节点上新增队列，或将旧的队列迁移到新的服务节点上。

- 需要增加队列实现顺序消息性能扩展

  <u>在 Apache RocketMQ 服务端4.x版本中，顺序消息的顺序性在队列内生效的</u>，因此顺序消息的并发度会在一定程度上受队列数量的影响，因此建议仅在系统性能瓶颈时再增加队列。

## 3.4 消息(Message)

本文介绍 Apache RocketMQ 中消息（Message）的定义、模型关系、内部属性、行为约束及使用建议。

### 3.4.1 定义

消息是 Apache RocketMQ 中的最小数据传输单元。生产者将业务数据的负载和拓展属性包装成消息发送到 Apache RocketMQ 服务端，服务端按照相关语义将消息投递到消费端进行消费。

Apache RocketMQ 的消息模型具备如下特点：

- **消息不可变性**

  消息本质上是已经产生并确定的事件，一旦产生后，消息的内容不会发生改变。即使经过传输链路的控制也不会发生变化，**消费端获取的消息都是只读消息视图**。

- **消息持久化**

  Apache RocketMQ 会默认对消息进行持久化，即将接收到的消息存储到 Apache RocketMQ 服务端的存储文件中，保证消息的可回溯性和系统故障场景下的可恢复性。

### 3.4.2 模型关系

在整个 Apache RocketMQ 的领域模型中，消息所处的流程和位置如下：![消息](https://rocketmq.apache.org/zh/assets/images/archiforqueue-dd6788b33bf2fc96b4a1dab83a1b0d71.png)

1. 消息由生产者初始化并发送到Apache RocketMQ 服务端。
2. 消息按照到达Apache RocketMQ 服务端的顺序存储到队列中。
3. 消费者按照指定的订阅关系从Apache RocketMQ 服务端中获取消息并消费。

### 3.4.3 消息内部属性

**系统保留属性**

**主题名称**

- 定义：当前消息所属的主题的名称。集群内全局唯一。更多信息，请参见[主题（Topic）](https://rocketmq.apache.org/zh/docs/domainModel/02topic)。
- 取值：从客户端SDK接口获取。

**消息类型**

- 定义：当前消息的类型。
- 取值：从客户端SDK接口获取。Apache RocketMQ 支持的消息类型如下：
  - Normal：[普通消息](https://rocketmq.apache.org/zh/docs/featureBehavior/01normalmessage)，消息本身无特殊语义，消息之间也没有任何关联。
  - FIFO：[顺序消息](https://rocketmq.apache.org/zh/docs/featureBehavior/03fifomessage)，Apache RocketMQ 通过消息分组MessageGroup标记一组特定消息的先后顺序，可以保证消息的投递顺序严格按照消息发送时的顺序。
  - Delay：[定时/延时消息](https://rocketmq.apache.org/zh/docs/featureBehavior/02delaymessage)，通过指定延时时间控制消息生产后不要立即投递，而是在延时间隔后才对消费者可见。
  - Transaction：[事务消息](https://rocketmq.apache.org/zh/docs/featureBehavior/04transactionmessage)，Apache RocketMQ 支持分布式事务消息，支持应用数据库更新和消息调用的事务一致性保障。

**消息队列**

- 定义：实际存储当前消息的队列。更多信息，请参见[队列（MessageQueue）](https://rocketmq.apache.org/zh/docs/domainModel/03messagequeue)。
- 取值：由服务端指定并填充。

**消息位点**

- 定义：当前消息存储在队列中的位置。更多信息，请参见[消费进度原理](https://rocketmq.apache.org/zh/docs/featureBehavior/09consumerprogress)。
- 取值：由服务端指定并填充。取值范围：0~long.Max。

**消息ID**

- 定义：消息的唯一标识，**集群内每条消息的ID全局唯一**。
- 取值：生产者客户端系统自动生成。固定为数字和大写字母组成的32位字符串。

**索引Key列表（可选）**

- 定义：消息的索引键，可通过设置不同的Key区分消息和快速查找消息。
- 取值：由生产者客户端定义。

**过滤标签Tag（可选）**

- 定义：消息的过滤标签。消费者可通过Tag对消息进行过滤，仅接收指定标签的消息。
- 取值：由生产者客户端定义。
- 约束：一条消息仅支持设置一个标签。

**定时时间（可选）**

- 定义：定时场景下，消息触发延时投递的毫秒级时间戳。更多信息，请参见[定时/延时消息](https://rocketmq.apache.org/zh/docs/featureBehavior/02delaymessage)。
- 取值：由消息生产者定义。
- 约束：最大可设置定时时长为40天。

**消息发送时间**

- 定义：消息发送时，生产者客户端系统的本地毫秒级时间戳。
- 取值：由生产者客户端系统填充。
- 说明：客户端系统时钟和服务端系统时钟可能存在偏差，消息发送时间是以客户端系统时钟为准。

**消息保存时间戳**

- 定义：消息在Apache RocketMQ服务端完成存储时，服务端系统的本地毫秒级时间戳。 对于定时消息和事务消息，消息保存时间指的是消息生效对消费方可见的服务端系统时间。

- 取值：由服务端系统填充。
- 说明：客户端系统时钟和服务端系统时钟可能存在偏差，消息保留时间是以服务端系统时钟为准。

**消费重试次数**

- 定义：消息消费失败后，Apache RocketMQ 服务端重新投递的次数。每次重试后，重试次数加1。更多信息，请参见[消费重试](https://rocketmq.apache.org/zh/docs/featureBehavior/10consumerretrypolicy)。
- 取值：由服务端系统标记。首次消费，重试次数为0；消费失败首次重试时，重试次数为1。

**业务自定义属性**

- 定义：生产者可以自定义设置的扩展信息。
- 取值：由消息生产者自定义，按照字符串键值对设置。

**消息负载**

- 定义：业务消息的实际报文数据。
- 取值：由生产者负责序列化编码，按照二进制字节传输。
- 约束：请参见[参数限制](https://rocketmq.apache.org/zh/docs/introduction/03limits)。

### 3.4.4 行为约束

消息大小不得超过其类型所对应的限制，否则消息会发送失败。

系统默认的消息最大限制如下：

- 普通和顺序消息：4 MB
- 事务和定时或延时消息：64 KB

### 3.4.5 使用建议

**单条消息不建议传输超大负载**

作为一款消息中间件产品，Apache RocketMQ 一般传输的是都是业务事件数据。单个原子消息事件的数据大小需要严格控制，如果单条消息过大容易造成网络传输层压力，不利于异常重试和流量控制。

生产环境中如果需要传输超大负载，建议按照固定大小做报文拆分，或者结合文件存储等方法进行传输。

**消息中转时做好不可变设计**

**Apache RocketMQ 服务端5.x版本中，消息本身不可编辑，消费端获取的消息都是只读消息视图**。 但在历史版本3.x和4.x版本中消息不可变性没有强约束，因此如果您需要在使用过程中对消息进行中转操作，务必将消息重新初始化。

- 正确使用示例如下：

  ```java
  Message m = Consumer.receive();
  Message m2= MessageBuilder.buildFrom(m);
  Producer.send(m2);
  ```

- 错误使用示例如下：

  ```java
  Message m = Consumer.receive();
  m.update()；
  Producer.send(m);
  ```

## 3.5 生产者(Producer)

本文介绍 Apache RocketMQ 中生产者（Producer）的定义、模型关系、内部属性、版本兼容性及使用建议。

### 3.5.1 定义

生产者是 Apache RocketMQ 系统中用来构建并传输消息到服务端的运行实体。

生产者通常被集成在业务系统中，将业务消息按照要求封装成 Apache RocketMQ 的[消息（Message）](https://rocketmq.apache.org/zh/docs/domainModel/04message)并发送至服务端。

在消息生产者中，可以定义如下传输行为：

- 发送方式：生产者可通过API接口设置消息发送的方式。Apache RocketMQ 支持同步传输和异步传输。
- 批量发送：生产者可通过API接口设置消息批量传输的方式。例如，批量发送的消息条数或消息大小。
- 事务行为：Apache RocketMQ 支持事务消息，对于事务消息需要生产者配合进行事务检查等行为保障事务的最终一致性。具体信息，请参见[事务消息](https://rocketmq.apache.org/zh/docs/featureBehavior/04transactionmessage)。

生产者和主题的关系为多对多关系，即同一个生产者可以向多个主题发送消息，对于平台类场景如果需要发送消息到多个主题，并不需要创建多个生产者；同一个主题也可以接收多个生产者的消息，以此可以实现生产者性能的水平扩展和容灾。 ![生产者主题关联](https://rocketmq.apache.org/zh/assets/images/producer_topic-f9a6348396228a2976e34a5ad0774314.png)

### 3.5.2 模型关系

在 Apache RocketMQ 的领域模型中，生产者的位置和流程如下：![生产者](https://rocketmq.apache.org/zh/assets/images/archiforproducer-ebb8ff832f6e857cbebc2c17c2044a3b.png)

1. 消息由生产者初始化并发送到Apache RocketMQ 服务端。
2. 消息按照到达Apache RocketMQ 服务端的顺序存储到主题的指定队列中。
3. 消费者按照指定的订阅关系从Apache RocketMQ 服务端中获取消息并消费。

### 3.5.3 内部属性

**客户端ID**

- 定义：生产者客户端的标识，用于区分不同的生产者。**集群内全局唯一**。
- 取值：客户端ID由Apache RocketMQ 的SDK自动生成，主要用于日志查看、问题定位等运维场景，不支持修改。

**通信参数**

- 接入点信息 **（必选）** ：连接服务端的接入地址，用于识别服务端集群。 <u>接入点必须按格式配置，建议使用域名，避免使用IP地址，防止节点变更无法进行热点迁移</u>。
- 身份认证信息 **（可选）** ：客户端用于身份验证的凭证信息。 仅在服务端开启身份识别和认证时需要传输。
- 请求超时时间 **（可选）** ：客户端网络请求调用的超时时间。取值范围和默认值，请参见[参数限制](https://rocketmq.apache.org/zh/docs/introduction/03limits)。

**预绑定主题列表**

- 定义：Apache RocketMQ 的生产者需要将消息发送到的目标主题列表，主要作用如下：

  - 事务消息 **（必须设置）** ：事务消息场景下，生产者在故障、重启恢复时，需要检查事务消息的主题中是否有未提交的事务消息。避免生产者发送新消息后，主题中的旧事务消息一直处于未提交状态，造成业务延迟。

  - 非事务消息 **（建议设置）** ：服务端会在生产者初始化时根据预绑定主题列表，检查目标主题的访问权限和合法性，而不需要等到应用启动后再检查。

    若未设置，或后续消息发送的目标主题动态变更， Apache RocketMQ 会对目标主题进行动态补充检验。

- 约束：对于事务消息，预绑定列表必须设置，且需要和事务检查器一起配合使用。

**事务检查器**

- 定义：Apache RocketMQ 的事务消息机制中，为保证异常场景下事务的最终一致性，生产者需要主动实现事务检查器的接口。具体信息，请参见[事务消息](https://rocketmq.apache.org/zh/docs/featureBehavior/04transactionmessage)。
- **发送事务消息时，事务检查器必须设置，且需要和预绑定主题列表一起配合使用**。

**发送重试策略**：

- 定义: 生产者在消息发送失败时的重试策略。具体信息，请参见[消息发送重试机制](https://rocketmq.apache.org/zh/docs/featureBehavior/05sendretrypolicy)。

### 3.5.4 版本兼容性

<u>Apache RocketMQ 服务端5.x版本开始，生产者是匿名的，无需管理生产者分组（ProducerGroup）；对于历史版本服务端3.x和4.x版本，已经使用的生产者分组可以废弃无需再设置，且不会对当前业务产生影响</u>。

### 3.5.5 使用建议

**不建议单一进程创建大量生产者**

Apache RocketMQ 的生产者和主题是多对多的关系，支持同一个生产者向多个主题发送消息。对于生产者的创建和初始化，建议遵循够用即可、最大化复用原则，如果有需要发送消息到多个主题的场景，无需为每个主题都创建一个生产者。

**不建议频繁创建和销毁生产者**

Apache RocketMQ 的生产者是可以重复利用的底层资源，类似数据库的连接池。因此不需要在每次发送消息时动态创建生产者，且在发送结束后销毁生产者。这样频繁的创建销毁会在服务端产生大量短连接请求，严重影响系统性能。

- 正确示例

  ```java
  Producer p = ProducerBuilder.build();
  for (int i =0;i<n;i++){
      Message m= MessageBuilder.build();
      p.send(m);
   }
  p.shutdown();
  ```

  

- 典型错误示例

  ```java
  for (int i =0;i<n;i++){
      Producer p = ProducerBuilder.build();
      Message m= MessageBuilder.build();
      p.send(m);
      p.shutdown();
    }
  ```

## 3.6 消费者分组(ConsumerGroup)

本文介绍 Apache RocketMQ 中消费者分组（ConsumerGroup）的定义、模型关系、内部属性、行为约束、版本兼容性及使用建议。

### 3.6.1 定义

消费者分组是 Apache RocketMQ 系统中承载多个消费行为一致的消费者的负载均衡分组。

和消费者不同，消费者分组并不是运行实体，而是一个逻辑资源。在 Apache RocketMQ 中，通过消费者分组内初始化多个消费者实现消费性能的水平扩展以及高可用容灾。

在消费者分组中，统一定义以下消费行为，同一分组下的多个消费者将按照分组内统一的消费行为和负载均衡策略消费消息。

- 订阅关系：Apache RocketMQ 以消费者分组的粒度管理订阅关系，实现订阅关系的管理和追溯。具体信息，请参见[订阅关系（Subscription）](https://rocketmq.apache.org/zh/docs/domainModel/09subscription)。
- 投递顺序性：Apache RocketMQ 的服务端将消息投递给消费者消费时，支持顺序投递和并发投递，投递方式在消费者分组中统一配置。具体信息，请参见[顺序消息](https://rocketmq.apache.org/zh/docs/featureBehavior/03fifomessage)。
- 消费重试策略： 消费者消费消息失败时的重试策略，包括重试次数、死信队列设置等。具体信息，请参见[消费重试](https://rocketmq.apache.org/zh/docs/featureBehavior/10consumerretrypolicy)。

### 3.6.2 模型关系

在 Apache RocketMQ 的领域模型中，消费者分组的位置和流程如下：![消费组](https://rocketmq.apache.org/zh/assets/images/archiforconsumergroup-9d98f4f7fc0302aa2363454a552477d9.png)

1. 消息由生产者初始化并发送到Apache RocketMQ 服务端。
2. 消息按照到达Apache RocketMQ 服务端的顺序存储到主题的指定队列中。
3. 消费者按照指定的订阅关系从Apache RocketMQ 服务端中获取消息并消费。

### 3.6.3 内部属性

**消费者分组名称**

- 定义：消费者分组的名称，用于区分不同的消费者分组。**集群内全局唯一**。
- 取值：消费者分组由用户设置并创建。具体命名规范，请参见[参数限制](https://rocketmq.apache.org/zh/docs/introduction/03limits)。

**投递顺序性**

- 定义：消费者消费消息时，Apache RocketMQ 向消费者客户端投递消息的顺序。

  根据不同的消费场景，Apache RocketMQ 提供顺序投递和并发投递两种方式。具体信息，请参见[顺序消息](https://rocketmq.apache.org/zh/docs/featureBehavior/03fifomessage)。

- 取值：**默认投递方式为并发投递**。

**消费重试策略**

- 定义：消费者消费消息失败时，系统的重试策略。消费者消费消息失败时，系统会按照重试策略，将指定消息投递给消费者重新消费。具体信息，请参见[消费重试](https://rocketmq.apache.org/zh/docs/featureBehavior/10consumerretrypolicy)。
- 取值：重试策略包括：
  - 最大重试次数：表示消息可以重新被投递的最大次数，**超过最大重试次数还没被成功消费，消息将被投递至死信队列或丢弃**。
  - 重试间隔：Apache RocketMQ 服务端重新投递消息的间隔时间。 最大重试次数和重试间隔的取值范围及默认值，请参见[参数限制](https://rocketmq.apache.org/zh/docs/introduction/03limits)。
- 约束：<u>重试间隔仅在PushConsumer消费类型下有效</u>。

**订阅关系**

- 定义：当前消费者分组关联的订阅关系集合。包括消费者订阅的主题，以及消息的过滤规则等。订阅关系由消费者动态注册到消费者分组中，Apache RocketMQ 服务端会持久化订阅关系并匹配消息的消费进度。更多信息，请参见[订阅关系（Subscription）](https://rocketmq.apache.org/zh/docs/domainModel/09subscription)。

### 3.6.4 行为约束

在 Apache RocketMQ 领域模型中，消费者的管理通过消费者分组实现，同一分组内的消费者共同分摊消息进行消费。因此，为了保证分组内消息的正常负载和消费，

Apache RocketMQ 要求同一分组下的所有消费者以下消费行为保持一致：

- **投递顺序**
- **消费重试策略**

### 3.6.5 版本兼容性

如行为约束中所述，同一分组内所有消费者的投递顺序和消费重试策略需要保持一致。

- <u>Apache RocketMQ 服务端5.x版本：上述消费者的消费行为从关联的消费者分组中统一获取，因此，同一分组内所有消费者的消费行为必然是一致的，客户端无需关注</u>。
- Apache RocketMQ 服务端3.x/4.x历史版本：上述消费逻辑由消费者客户端接口定义，因此，您需要自己在消费者客户端设置时保证同一分组下的消费者的消费行为一致。

若您使用 Apache RocketMQ 服务端5.x版本，客户端使用历史版本SDK，则消费者的消费逻辑以消费者客户端接口的设置为准。

### 3.6.6 使用建议

**按照业务合理拆分分组**

Apache RocketMQ 的消费者和主题是多对多的关系，对于消费者分组的拆分设计，建议遵循以下原则：

- 消费者的投递顺序一致：同一消费者分组下所有消费者的消费投递顺序是相同的，统一都是顺序投递或并发投递，不同业务场景不能混用消费者分组。
- 消费者业务类型一致：一般消费者分组和主题对应，不同业务域对消息消费的要求不同，例如消息过滤属性、消费重试策略不同。因此，不同业务域主题的消费建议使用不同的消费者分组，避免一个消费者分组消费超过10个主题。

**消费者分组管理尽量避免自动化机制**

在 Apache RocketMQ 架构中，消费分组属于状态管理类的逻辑资源，每个消费分组都会涉及关联的消费状态、堆积信息、可观测指标和监控采集数据。因此，生产环境需要严格管理消费者分组资源，请勿随意进行增、删、改、查操作。

<u>Apache RocketMQ 虽然提供了自动创建消费者分组的功能，但是建议仅在测试环境使用，生产环境请勿打开，避免产生大量消费者分组，无法管理和回收，且浪费系统资源</u>。

## 3.7 消费者(Consumer)

本文介绍 Apache RocketMQ 中消费者（Consumer）的定义、模型关系、内部属性、行为约束、版本兼容性及使用建议。

### 3.7.1 定义

消费者是 Apache RocketMQ 中用来接收并处理消息的运行实体。 消费者通常被集成在业务系统中，从 Apache RocketMQ 服务端获取消息，并将消息转化成业务可理解的信息，供业务逻辑处理。

在消息消费端，可以定义如下传输行为：

- 消费者身份：**消费者必须关联一个指定的消费者分组，以获取分组内统一定义的行为配置和消费状态。**
- 消费者类型：Apache RocketMQ 面向不同的开发场景提供了多样的消费者类型，包括PushConsumer类型、SimpleConsumer类型、PullConsumer类型（仅推荐流处理场景使用）等。具体信息，请参见[消费者分类](https://rocketmq.apache.org/zh/docs/featureBehavior/06consumertype)。
- 消费者本地运行配置：消费者根据不同的消费者类型，控制消费者客户端本地的运行配置。例如消费者客户端的线程数，消费并发度等，实现不同的传输效果。

### 3.7.2 模型关系

在 Apache RocketMQ 的领域模型中，消费者的位置和流程如下：![消费者](https://rocketmq.apache.org/zh/assets/images/archiforconsumer-24914573add839fdf2ba2cbc0fcab7c4.png)

1. 消息由生产者初始化并发送到Apache RocketMQ 服务端。
2. 消息按照到达Apache RocketMQ 服务端的顺序存储到主题的指定队列中。
3. 消费者按照指定的订阅关系从Apache RocketMQ 服务端中获取消息并消费。

### 3.7.3 内部属性

**消费者分组名称**

- 定义：当前消费者关联的消费者分组名称，消费者必须关联到指定的消费者分组，通过消费者分组获取消费行为。更多信息，请参见[消费者分组（ConsumerGroup）](https://rocketmq.apache.org/zh/docs/domainModel/07consumergroup)。
- 取值：消费者分组为Apache RocketMQ 的逻辑资源，需要您提前通过控制台或OpenAPI创建。具体命名格式，请参见[使用限制](https://rocketmq.apache.org/zh/docs/introduction/03limits)。

**客户端ID**

- 定义：消费者客户端的标识，用于区分不同的消费者。集群内全局唯一。
- 取值：客户端ID由Apache RocketMQ 的SDK自动生成，主要用于日志查看、问题定位等运维场景，不支持修改。

**通信参数**

- 接入点信息 **（必选）** ：连接服务端的接入地址，用于识别服务端集群。 接入点必须按格式配置，建议使用域名，避免使用IP地址，防止节点变更无法进行热点迁移。
- 身份认证信息 **（可选）** ：客户端用于身份验证的凭证信息。 仅在服务端开启身份识别和认证时需要传输。
- 请求超时时间 **（可选）** ：客户端网络请求调用的超时时间。取值范围和默认值，请参见[参数限制](https://rocketmq.apache.org/zh/docs/introduction/03limits)。

**预绑定订阅关系列表**

- 定义：指定消费者的订阅关系列表。 Apache RocketMQ 服务端可在消费者初始化阶段，根据预绑定的订阅关系列表对目标主题进行权限及合法性校验，无需等到应用启动后才能校验。

- 取值：建议在消费者初始化阶段明确订阅关系即要订阅的主题列表，若未设置，或订阅的主题动态变更，Apache RocketMQ 会对目标主题进行动态补充校验。

**消费监听器**

- 定义：Apache RocketMQ 服务端将消息推送给消费者后，消费者调用消息消费逻辑的监听器。
- 取值：由消费者客户端本地配置。
- 约束：使用PushConsumer类型的消费者消费消息时，消费者客户端必须设置消费监听器。消费者类型的具体信息，请参见[消费者分类](https://rocketmq.apache.org/zh/docs/featureBehavior/06consumertype)。

### 3.7.4 行为约束

在 Apache RocketMQ 领域模型中，消费者的管理通过消费者分组实现，同一分组内的消费者共同分摊消息进行消费。因此，为了保证分组内消息的正常负载和消费，

Apache RocketMQ 要求同一分组下的所有消费者以下消费行为保持一致：

- **投递顺序**
- **消费重试策略**

### 3.7.5 版本兼容性

如行为约束中所述，同一分组内所有消费者的投递顺序和消费重试策略需要保持一致。

- Apache RocketMQ 服务端5.x版本：上述消费者的消费行为从关联的消费者分组中统一获取，因此，同一分组内所有消费者的消费行为必然是一致的，客户端无需关注。
- Apache RocketMQ 服务端3.x/4.x历史版本：上述消费逻辑由消费者客户端接口定义，因此，您需要自己在消费者客户端设置时保证同一分组下的消费者的消费行为一致。

若您使用 Apache RocketMQ 服务端5.x版本，客户端使用历史版本SDK，则消费者的消费逻辑以消费者客户端接口的设置为准。

### 3.7.6 使用建议

**不建议在单一进程内创建大量消费者**

Apache RocketMQ 的消费者在通信协议层面支持非阻塞传输模式，网络通信效率较高，并且支持多线程并发访问。因此，大部分场景下，单一进程内同一个消费分组只需要初始化唯一的一个消费者即可，开发过程中应避免以相同的配置初始化多个消费者。

**不建议频繁创建和销毁消费者**

Apache RocketMQ 的消费者是可以重复利用的底层资源，类似数据库的连接池。因此不需要在每次接收消息时动态创建消费者，且在消费完成后销毁消费者。这样频繁地创建销毁会在服务端产生大量短连接请求，严重影响系统性能。

- 正确示例

  ```java
  Consumer c = ConsumerBuilder.build();
  for (int i =0;i<n;i++){
        Message m= c.receive();
        //process message
      }
  c.shutdown();
  ```

  

- 典型错误示例

  ```java
  for (int i =0;i<n;i++){
      Consumer c = ConsumerBuilder.build();
      Message m= c.receive();
      //process message
      c.shutdown();
    }
  ```

## 3.8 订阅关系(Subscription)

本文介绍 Apache RocketMQ 中订阅关系（Subscription）的定义、模型关系、内部属性及使用建议。

### 3.8.1 定义

订阅关系是 Apache RocketMQ 系统中消费者获取消息、处理消息的规则和状态配置。

订阅关系由<u>消费者分组</u>动态注册到服务端系统，并在后续的消息传输中按照订阅关系定义的过滤规则进行消息匹配和消费进度维护。

通过配置订阅关系，可控制如下传输行为：

- 消息过滤规则：用于控制消费者在消费消息时，选择主题内的哪些消息进行消费，设置消费过滤规则可以高效地过滤消费者需要的消息集合，灵活根据不同的业务场景设置不同的消息接收范围。具体信息，请参见[消息过滤](https://rocketmq.apache.org/zh/docs/featureBehavior/07messagefilter)。
- 消费状态：Apache RocketMQ 服务端默认提供订阅关系持久化的能力，即消费者分组在服务端注册订阅关系后，当消费者离线并再次上线后，可以获取离线前的消费进度并继续消费。

### 3.8.2 订阅关系判断原则

Apache RocketMQ 的订阅关系按照消费者分组和主题粒度设计，因此，一个订阅关系指的是指定某个消费者分组对于某个主题的订阅，判断原则如下：

- 不同消费者分组对于同一个主题的订阅相互独立如下图所示，消费者分组Group A和消费者分组Group B分别以不同的订阅关系订阅了同一个主题Topic A，这两个订阅关系互相独立，可以各自定义，不受影响。

  ![订阅关系不同分组](https://rocketmq.apache.org/zh/assets/images/subscription_diff_group-0b215b9bb822b4bf43c388e9155ecca1.png)

- 同一个消费者分组对于不同主题的订阅也相互独立如下图所示，消费者分组Group A订阅了两个主题Topic A和Topic B，对于Group A中的消费者来说，订阅的Topic A为一个订阅关系，订阅的Topic B为另外一个订阅关系，且这两个订阅关系互相独立，可以各自定义，不受影响。

  ![订阅关系相同分组](https://rocketmq.apache.org/zh/assets/images/subscription_one_group-77bd92b987e8264ad3c5f27b29463942.png)

> 整体和kafka类似，只不过kafka没有支持上图中的Tag过滤（消息过滤）的功能，而订阅关系的维护和RocketMQ基本没区别。

### 3.8.3 模型关系

在 Apache RocketMQ 的领域模型中，订阅关系的位置和流程如下：![订阅关系](https://rocketmq.apache.org/zh/assets/images/archiforsubsciption-a495c04e71ed64b9403b689f9413ed08.png)

1. 消息由生产者初始化并发送到Apache RocketMQ 服务端。
2. 消息按照到达Apache RocketMQ 服务端的顺序存储到主题的指定队列中。
3. 消费者按照指定的订阅关系从Apache RocketMQ 服务端中获取消息并消费。

### 3.8.4 内部属性

**过滤类型**

- 定义：消息过滤规则的类型。订阅关系中设置消息过滤规则后，系统将按照过滤规则匹配主题中的消息，只将符合条件的消息投递给消费者消费，实现消息的再次分类。
- 取值：
  - TAG过滤：按照Tag字符串进行全文过滤匹配。
  - SQL92过滤：按照SQL语法对消息属性进行过滤匹配。

**过滤表达式**

- 定义：自定义的过滤规则表达式。
- 取值：具体取值规范，请参见[过滤表达式语法规范](https://rocketmq.apache.org/zh/docs/featureBehavior/07messagefilter)。

### 3.8.5 行为约束

**订阅关系一致**

Apache RocketMQ 是按照消费者分组粒度管理订阅关系，因此，同一消费者分组内的消费者在消费逻辑上必须保持一致，否则会出现消费冲突，导致部分消息消费异常。

- 正确示例

  ```java
  //Consumer c1
  Consumer c1 = ConsumerBuilder.build(groupA);
  c1.subscribe(topicA,"TagA");
  //Consumer c2
  Consumer c2 = ConsumerBuilder.build(groupA);
  c2.subscribe(topicA,"TagA");
  ```

- 错误示例

  ```java
  //Consumer c1
  Consumer c1 = ConsumerBuilder.build(groupA);
  c1.subscribe(topicA,"TagA");
  //Consumer c2
  Consumer c2 = ConsumerBuilder.build(groupA);
  c2.subscribe(topicA,"TagB");
  ```

### 3.8.6 使用建议

**建议不要频繁修改订阅关系**

在 Apache RocketMQ 领域模型中，订阅关系关联了过滤规则、消费进度等元数据和相关配置，同时系统需要保证消费者分组下的所有消费者的消费行为、消费逻辑、负载策略等一致，整体运算逻辑比较复杂。因此，不建议在生产环境中通过频繁修改订阅关系来实现业务逻辑的变更，这样可能会导致客户端一直处于负载均衡调整和变更的过程，从而影响消息接收。

# 4. 功能特性

## 4.1 普通消息

普通消息为 Apache RocketMQ 中最基础的消息，区别于有特性的顺序消息、定时/延时消息和事务消息。本文为您介绍普通消息的应用场景、功能原理、使用方法和使用建议。

### 4.1.1 应用场景

普通消息一般应用于微服务解耦、事件驱动、数据集成等场景，这些场景大多数要求数据传输通道具有可靠传输的能力，且对消息的处理时机、处理顺序没有特别要求。

**典型场景一：微服务异步解耦** ![在线消息处理](https://rocketmq.apache.org/zh/assets/images/onlineprocess-cfd38e3de3a5fc1ee76f17331cc5b828.png)

如上图所示，以在线的电商交易场景为例，上游订单系统将用户下单支付这一业务事件封装成独立的普通消息并发送至Apache RocketMQ服务端，下游按需从服务端订阅消息并按照本地消费逻辑处理下游任务。每个消息之间都是相互独立的，且不需要产生关联。

**典型场景二：数据集成传输** ![数据传输](https://rocketmq.apache.org/zh/assets/images/offlineprocess-027f6f1642db3d78ff29890abbe38bf8.png)

如上图所示，以离线的日志收集场景为例，通过埋点组件收集前端应用的相关操作日志，并转发到 Apache RocketMQ 。每条消息都是一段日志数据，Apache RocketMQ 不做任何处理，只需要将日志数据可靠投递到下游的存储系统和分析系统即可，后续功能由后端应用完成。

### 4.1.2 功能原理

**什么是普通消息**

定义：普通消息是Apache RocketMQ基本消息功能，支持生产者和消费者的异步解耦通信。 ![生命周期](https://rocketmq.apache.org/zh/assets/images/lifecyclefornormal-e8a2a7e42a0722f681eb129b51e1bd66.png)

**普通消息生命周期**

- 初始化：消息被生产者构建并完成初始化，待发送到服务端的状态。
- 待消费：消息被发送到服务端，对消费者可见，等待消费者消费的状态。
- **消费中：消息被消费者获取，并按照消费者本地的业务逻辑进行处理的过程。 此时服务端会等待消费者完成消费并提交消费结果，如果一定时间后没有收到消费者的响应，Apache RocketMQ会对消息进行重试处理。具体信息，请参见[消费重试](https://rocketmq.apache.org/zh/docs/featureBehavior/10consumerretrypolicy)。**
- 消费提交：消费者完成消费处理，并向服务端提交消费结果，服务端标记当前消息已经被处理（包括消费成功和失败）。 Apache RocketMQ默认支持保留所有消息，此时消息数据并不会立即被删除，只是逻辑标记已消费。**消息在保存时间到期或存储空间不足被删除前，消费者仍然可以回溯消息重新消费**。
- 消息删除：Apache RocketMQ按照消息保存机制滚动清理最早的消息数据，将消息从物理文件中删除。更多信息，请参见[消息存储和清理机制](https://rocketmq.apache.org/zh/docs/featureBehavior/11messagestorepolicy)。

### 4.1.3 使用限制

普通消息仅支持使用MessageType为Normal主题，即普通消息只能发送至类型为普通消息的主题中，发送的消息的类型必须和主题的类型一致。

### 4.1.4 使用示例

**创建主题**

Apache RocketMQ 5.0版本下创建主题操作，推荐使用mqadmin工具，需要注意的是，对于消息类型需要通过属性参数添加。示例如下：

```shell
sh mqadmin updateTopic -n <nameserver_address> -t <topic_name> -c <cluster_name> -a +message.type=NORMAL
```

**发送消息** 普通消息支持设置消息索引键、消息过滤标签等信息，用于消息过滤和搜索查找。以Java语言为例，收发普通消息的示例代码如下：

```java
//普通消息发送。
MessageBuilder messageBuilder = new MessageBuilderImpl();
Message message = messageBuilder.setTopic("topic")
    //设置消息索引键，可根据关键字精确查找某条消息。
    .setKeys("messageKey")
    //设置消息Tag，用于消费端根据指定Tag过滤消息。
    .setTag("messageTag")
    //消息体。
    .setBody("messageBody".getBytes())
    .build();
try {
    //发送消息，需要关注发送结果，并捕获失败等异常。
    SendReceipt sendReceipt = producer.send(message);
    System.out.println(sendReceipt.getMessageId());
} catch (ClientException e) {
    e.printStackTrace();
}
//消费示例一：使用PushConsumer消费普通消息，只需要在消费监听器中处理即可。
MessageListener messageListener = new MessageListener() {
    @Override
    public ConsumeResult consume(MessageView messageView) {
        System.out.println(messageView);
        //根据消费结果返回状态。
        return ConsumeResult.SUCCESS;
    }
};
//消费示例二：使用SimpleConsumer消费普通消息，主动获取消息进行消费处理并提交消费结果。
List<MessageView> messageViewList = null;
try {
    messageViewList = simpleConsumer.receive(10, Duration.ofSeconds(30));
    messageViewList.forEach(messageView -> {
        System.out.println(messageView);
        //消费处理完成后，需要主动调用ACK提交消费结果。
        try {
            simpleConsumer.ack(messageView);
        } catch (ClientException e) {
            e.printStackTrace();
        }
    });
} catch (ClientException e) {
    //如果遇到系统流控等原因造成拉取失败，需要重新发起获取消息请求。
    e.printStackTrace();
}
```

### 4.1.5 使用建议

**设置全局唯一业务索引键，方便问题追踪**

Apache RocketMQ支持自定义索引键（消息的Key），在消息查询和轨迹查询时，可以通过索引键高效精确地查询到消息。

因此，发送消息时，建议设置业务上唯一的信息作为索引，方便后续快速定位消息。例如，订单ID，用户ID等。

## 4.2 定时/延时消息

定时/延时消息为 Apache RocketMQ 中的高级特性消息，本文为您介绍定时/延时消息的应用场景、功能原理、使用限制、使用方法和使用建议。

> **定时消息和延时消息本质相同，都是服务端根据消息设置的定时时间在某一固定时刻将消息投递给消费者消费。因此，下文统一用定时消息描述。**

### 4.2.1 应用场景

在分布式定时调度触发、任务超时处理等场景，需要实现精准、可靠的定时事件触发。使用 Apache RocketMQ 的定时消息可以简化定时调度任务的开发逻辑，实现高性能、可扩展、高可靠的定时触发能力。

**典型场景一：分布式定时调度** ![定时消息](https://rocketmq.apache.org/zh/assets/images/delaywork-e9647b539ae35898102a336a27d3ad94.png)

在分布式定时调度场景下，需要实现各类精度的定时任务，例如每天5点执行文件清理，每隔2分钟触发一次消息推送等需求。传统基于数据库的定时调度方案在分布式场景下，性能不高，实现复杂。基于 Apache RocketMQ 的定时消息可以封装出多种类型的定时触发器。

**典型场景二：任务超时处理** ![超时任务处理](https://rocketmq.apache.org/zh/assets/images/scheduletask-1944aea7bf2a4a4c56be4d90ead4f1f3.png)

以电商交易场景为例，订单下单后暂未支付，此时不可以直接关闭订单，而是需要等待一段时间后才能关闭订单。使用 Apache RocketMQ 定时消息可以实现超时任务的检查触发。

基于定时消息的超时任务处理具备如下优势：

- 精度高、开发门槛低：基于消息通知方式不存在定时阶梯间隔。可以轻松实现任意精度事件触发，无需业务去重。
- 高性能可扩展：传统的数据库扫描方式较为复杂，需要频繁调用接口扫描，容易产生性能瓶颈。 Apache RocketMQ 的定时消息具有高并发和水平扩展的能力。

### 4.2.2 功能原理

**什么是定时消息**

定时消息是 Apache RocketMQ 提供的一种高级消息类型，消息被发送至服务端后，在指定时间后才能被消费者消费。通过设置一定的定时时间可以实现分布式场景的延时调度触发效果。

**定时时间设置原则**

- Apache RocketMQ 定时消息设置的定时时间是一个预期触发的系统时间戳，延时时间也需要转换成当前系统时间后的某一个时间戳，而不是一段延时时长。
- 定时时间的格式为毫秒级的Unix时间戳，您需要将要设置的时刻转换成时间戳形式。具体方式，请参见[Unix时间戳转换工具](https://www.unixtimestamp.com/)。
- 定时时间必须设置在定时时长范围内，超过范围则定时不生效，服务端会立即投递消息。
- **定时时长最大值默认为24小时**，不支持自定义修改，更多信息，请参见[参数限制](https://rocketmq.apache.org/zh/docs/introduction/03limits)。
- **定时时间必须设置为当前时间之后，若设置到当前时间之前，则定时不生效，服务端会立即投递消息**。

**示例如下：**

- 定时消息：例如，当前系统时间为2022-06-09 17:30:00，您希望消息在下午19:20:00定时投递，则定时时间为2022-06-09 19:20:00，转换成时间戳格式为1654773600000。
- 延时消息：例如，当前系统时间为2022-06-09 17:30:00，您希望延时1个小时后投递消息，则您需要根据当前时间和延时时长换算成定时时刻，即消息投递时间为2022-06-09 18:30:00，转换为时间戳格式为1654770600000。

**定时消息生命周期**

![定时消息生命周期](https://rocketmq.apache.org/zh/assets/images/lifecyclefordelay-2ce8278df69cd026dd11ffd27ab09a17.png)

- 初始化：消息被生产者构建并完成初始化，待发送到服务端的状态。
- **定时中：消息被发送到服务端，和普通消息不同的是，服务端不会直接构建消息索引，而是会将定时消息单独存储在定时存储系统中，等待定时时刻到达**。
- **待消费：定时时刻到达后，服务端将消息重新写入普通存储引擎，对下游消费者可见，等待消费者消费的状态**。
- 消费中：消息被消费者获取，并按照消费者本地的业务逻辑进行处理的过程。 此时服务端会等待消费者完成消费并提交消费结果，如果一定时间后没有收到消费者的响应，Apache RocketMQ会对消息进行重试处理。具体信息，请参见[消费重试](https://rocketmq.apache.org/zh/docs/featureBehavior/10consumerretrypolicy)。
- 消费提交：消费者完成消费处理，并向服务端提交消费结果，服务端标记当前消息已经被处理（包括消费成功和失败）。 Apache RocketMQ 默认支持保留所有消息，此时消息数据并不会立即被删除，只是逻辑标记已消费。消息在保存时间到期或存储空间不足被删除前，消费者仍然可以回溯消息重新消费。
- 消息删除：Apache RocketMQ按照消息保存机制滚动清理最早的消息数据，将消息从物理文件中删除。更多信息，请参见[消息存储和清理机制](https://rocketmq.apache.org/zh/docs/featureBehavior/11messagestorepolicy)。

### 4.2.3 使用限制

**消息类型一致性**

<u>定时消息仅支持在 MessageType为Delay 的主题内使用，即定时消息只能发送至类型为定时消息的主题中，发送的消息的类型必须和主题的类型一致</u>。

**定时精度约束**

Apache RocketMQ 定时消息的定时时长参数精确到毫秒级，但是默认精度为1000ms，即定时消息为秒级精度。

Apache RocketMQ 定时消息的状态支持持久化存储，系统由于故障重启后，仍支持按照原来设置的定时时间触发消息投递。若存储系统异常重启，可能会导致定时消息投递出现一定延迟。

### 4.2.4 使用示例

**创建主题**

Apache RocketMQ 5.0版本下创建主题操作，推荐使用mqadmin工具，需要注意的是，对于消息类型需要通过属性参数添加。示例如下：

```shell
sh mqadmin updateTopic -n <nameserver_address> -t <topic_name> -c <cluster_name> -a +message.type=DELAY
```

**发送消息**

<u>和普通消息相比，定时消费发送时，必须设置定时触发的目标时间戳</u>。

**创建延迟主题**

```bash
/bin/mqadmin updateTopic -c DefaultCluster -t DelayTopic -n 127.0.0.1:9876 -a +message.type=DELAY
```

- -c 集群名称
- -t Topic名称
- -n nameserver地址
- -a 额外属性，本例给主题添加了`message.type`为`DELAY`的属性用来支持延迟消息

以Java语言为例，使用定时消息示例参考如下：

```java
        //定时/延时消息发送
        MessageBuilder messageBuilder = new MessageBuilderImpl();;
        //以下示例表示：延迟时间为10分钟之后的Unix时间戳。
        Long deliverTimeStamp = System.currentTimeMillis() + 10L * 60 * 1000;
        Message message = messageBuilder.setTopic("topic")
                //设置消息索引键，可根据关键字精确查找某条消息。
                .setKeys("messageKey")
                //设置消息Tag，用于消费端根据指定Tag过滤消息。
                .setTag("messageTag")
                .setDeliveryTimestamp(deliverTimeStamp)
                //消息体
                .setBody("messageBody".getBytes())
                .build();
        try {
            //发送消息，需要关注发送结果，并捕获失败等异常。
            SendReceipt sendReceipt = producer.send(message);
            System.out.println(sendReceipt.getMessageId());
        } catch (ClientException e) {
            e.printStackTrace();
        }
        //消费示例一：使用PushConsumer消费定时消息，只需要在消费监听器处理即可。
        MessageListener messageListener = new MessageListener() {
            @Override
            public ConsumeResult consume(MessageView messageView) {
                System.out.println(messageView.getDeliveryTimestamp());
                //根据消费结果返回状态。
                return ConsumeResult.SUCCESS;
            }
        };
        //消费示例二：使用SimpleConsumer消费定时消息，主动获取消息进行消费处理并提交消费结果。
        List<MessageView> messageViewList = null;
        try {
            messageViewList = simpleConsumer.receive(10, Duration.ofSeconds(30));
            messageViewList.forEach(messageView -> {
                System.out.println(messageView);
                //消费处理完成后，需要主动调用ACK提交消费结果。
                try {
                    simpleConsumer.ack(messageView);
                } catch (ClientException e) {
                    e.printStackTrace();
                }
            });
        } catch (ClientException e) {
            //如果遇到系统流控等原因造成拉取失败，需要重新发起获取消息请求。
            e.printStackTrace();
        }
```

### 4.2.5 使用建议

**避免大量相同定时时刻的消息**

<u>定时消息的实现逻辑需要先经过定时存储等待触发，定时时间到达后才会被投递给消费者。因此，如果将大量定时消息的定时时间设置为同一时刻，则到达该时刻后会有大量消息同时需要被处理，会造成系统压力过大，导致消息分发延迟，影响定时精度</u>。

## 4.3 顺序消息

顺序消息为 Apache RocketMQ 中的高级特性消息，本文为您介绍顺序消息的应用场景、功能原理、使用限制、使用方法和使用建议。

### 4.3.1 应用场景

在有序事件处理、撮合交易、数据实时增量同步等场景下，异构系统间需要维持强一致的状态同步，上游的事件变更需要按照顺序传递到下游进行处理。在这类场景下使用 Apache RocketMQ 的顺序消息可以有效保证数据传输的顺序性。

**典型场景一：撮合交易** ![交易撮合](https://rocketmq.apache.org/zh/assets/images/fifo_trade-a8bac55b8fb3fceb995891c64c2f0a5a.png)

以证券、股票交易撮合场景为例，对于出价相同的交易单，坚持按照先出价先交易的原则，下游处理订单的系统需要严格按照出价顺序来处理订单。

**典型场景二：数据实时增量同步**

普通消息![普通消息](https://rocketmq.apache.org/zh/assets/images/tradewithnormal-5273283ffa54ec08017f356227411f83.png) 顺序消息![顺序消息](https://rocketmq.apache.org/zh/assets/images/tradewithfifo-30884dfeb909c54d7379641fcec437fa.png)

以数据库变更增量同步场景为例，上游源端数据库按需执行增删改操作，将二进制操作日志作为消息，通过 Apache RocketMQ 传输到下游搜索系统，下游系统按顺序还原消息数据，实现状态数据按序刷新。如果是普通消息则可能会导致状态混乱，和预期操作结果不符，基于顺序消息可以实现下游状态和上游操作结果一致。

### 4.3.2 功能原理

**什么是顺序消息**

顺序消息是 Apache RocketMQ 提供的一种高级消息类型，支持消费者按照发送消息的先后顺序获取消息，从而实现业务场景中的顺序处理。 相比其他类型消息，顺序消息在发送、存储和投递的处理过程中，更多强调多条消息间的先后顺序关系。

**Apache RocketMQ 顺序消息的<u>顺序关系通过消息组（MessageGroup）判定和识别</u>，发送顺序消息时需要为每条消息设置归属的消息组，相同消息组的多条消息之间遵循先进先出的顺序关系，<u>不同消息组、无消息组的消息之间不涉及顺序性</u>**。

基于消息组的顺序判定逻辑，支持按照业务逻辑做细粒度拆分，可以在满足业务局部顺序的前提下提高系统的并行度和吞吐能力。

**如何保证消息的顺序性**

Apache RocketMQ 的消息的顺序性分为两部分，生产顺序性和消费顺序性。

- **生产顺序性** ：

  Apache RocketMQ 通过生产者和服务端的协议保障单个生产者串行地发送消息，并按序存储和持久化。

  如需保证消息生产的顺序性，则必须满足以下条件：

  - **单一生产者：消息生产的顺序性仅支持单一生产者**，不同生产者分布在不同的系统，即使设置相同的消息组，不同生产者之间产生的消息也无法判定其先后顺序。
  - **串行发送**：Apache RocketMQ 生产者客户端支持多线程安全访问，但如果生产者使用多线程并行发送，则不同线程间产生的消息将无法判定其先后顺序。

  满足以上条件的生产者，将顺序消息发送至 Apache RocketMQ 后，会保证设置了同一消息组的消息，按照发送顺序存储在同一队列中。服务端顺序存储逻辑如下：

  - 相同消息组的消息按照先后顺序被存储在同一个队列。
  - **不同消息组的消息可以混合在同一个队列中，且不保证连续**。

![顺序存储逻辑](https://rocketmq.apache.org/zh/assets/images/fifomessagegroup-aad0a1b7e64089075db956c0eca0cbf4.png)

如上图所示，消息组1和消息组4的消息混合存储在队列1中， Apache RocketMQ 保证消息组1中的消息G1-M1、G1-M2、G1-M3是按发送顺序存储，且消息组4的消息G4-M1、G4-M2也是按顺序存储，但消息组1和消息组4中的消息不涉及顺序关系。

+ **消费顺序性** ：

  Apache RocketMQ 通过消费者和服务端的协议保障消息消费严格按照存储的先后顺序来处理。

  如需保证消息消费的顺序性，则必须满足以下条件：

  - 投递顺序

    Apache RocketMQ 通过客户端SDK和服务端通信协议保障消息按照服务端存储顺序投递，但业务方消费消息时需要严格按照接收---处理---应答的语义处理消息，避免因异步处理导致消息乱序。

    > 消费者类型为PushConsumer时， Apache RocketMQ 保证消息按照存储顺序一条一条投递给消费者，<u>若消费者类型为SimpleConsumer，则消费者有可能一次拉取多条消息。此时，消息消费的顺序性需要由业务方自行保证</u>。消费者类型的具体信息，请参见[消费者分类](https://rocketmq.apache.org/zh/docs/featureBehavior/06consumertype)。

  + 有限重试

    Apache RocketMQ 顺序消息投递仅在重试次数限定范围内，即**一条消息如果一直重试失败，超过最大重试次数后将不再重试，跳过这条消息消费，不会一直阻塞后续消息处理**。

    对于需要严格保证消费顺序的场景，请务设置合理的重试次数，避免参数不合理导致消息乱序。

**生产顺序性和消费顺序性组合**

<u>如果消息需要严格按照先进先出（FIFO）的原则处理，即先发送的先消费、后发送的后消费，则必须要同时满足生产顺序性和消费顺序性</u>。

一般业务场景下，同一个生产者可能对接多个下游消费者，不一定所有的消费者业务都需要顺序消费，您可以将生产顺序性和消费顺序性进行差异化组合，应用于不同的业务场景。例如发送顺序消息，但使用非顺序的并发消费方式来提高吞吐能力。更多组合方式如下表所示：

| 生产顺序                       | 消费顺序 | 顺序性效果                                                   |
| ------------------------------ | -------- | ------------------------------------------------------------ |
| 设置消息组，保证消息顺序发送。 | 顺序消费 | 按照消息组粒度，严格保证消息顺序。 同一消息组内的消息的消费顺序和发送顺序完全一致。 |
| 设置消息组，保证消息顺序发送。 | 并发消费 | 并发消费，尽可能按时间顺序处理。                             |
| 未设置消息组，消息乱序发送。   | 顺序消费 | 按队列存储粒度，严格顺序。 基于 Apache RocketMQ 本身队列的属性，消费顺序和队列存储的顺序一致，但不保证和发送顺序一致。 |
| 未设置消息组，消息乱序发送。   | 并发消费 | 并发消费，尽可能按照时间顺序处理。                           |

**顺序消息生命周期** ![生命周期](https://rocketmq.apache.org/zh/assets/images/lifecyclefornormal-e8a2a7e42a0722f681eb129b51e1bd66.png)

- 初始化：消息被生产者构建并完成初始化，待发送到服务端的状态。
- 待消费：消息被发送到服务端，对消费者可见，等待消费者消费的状态。
- 消费中：消息被消费者获取，并按照消费者本地的业务逻辑进行处理的过程。 此时服务端会等待消费者完成消费并提交消费结果，如果一定时间后没有收到消费者的响应，Apache RocketMQ会对消息进行重试处理。具体信息，请参见[消费重试](https://rocketmq.apache.org/zh/docs/featureBehavior/10consumerretrypolicy)。
- 消费提交：消费者完成消费处理，并向服务端提交消费结果，服务端标记当前消息已经被处理（包括消费成功和失败）。 Apache RocketMQ 默认支持保留所有消息，此时消息数据并不会立即被删除，只是逻辑标记已消费。消息在保存时间到期或存储空间不足被删除前，消费者仍然可以回溯消息重新消费。
- 消息删除：Apache RocketMQ按照消息保存机制滚动清理最早的消息数据，将消息从物理文件中删除。更多信息，请参见[消息存储和清理机制](https://rocketmq.apache.org/zh/docs/featureBehavior/11messagestorepolicy)。

> - 消息消费失败或消费超时，会触发服务端重试逻辑，**重试消息属于新的消息，原消息的生命周期已结束**。
> - **顺序消息消费失败进行消费重试时，为保障消息的顺序性，后续消息不可被消费，<u>必须等待前面的消息消费完成后才能被处理</u>**。

### 4.3.3 使用限制

**顺序消息仅支持使用MessageType为FIFO的主题，即顺序消息只能发送至类型为顺序消息的主题中，发送的消息的类型必须和主题的类型一致。**

### 4.3.4 使用示例

**创建主题**

Apache RocketMQ 5.0版本下创建主题操作，推荐使用mqadmin工具，需要注意的是，对于消息类型需要通过属性参数添加。示例如下：

```shell
sh mqadmin updateTopic -n <nameserver_address> -t <topic_name> -c <cluster_name> -a +message.type=FIFO
```

**创建订阅消费组**

Apache RocketMQ 5.0版本下创建订阅消费组操作，推荐使用mqadmin工具，需要注意的是，对于订阅消费组顺序类型需要通过 `-o` 选项设置。示例如下：

```shell
sh mqadmin updateSubGroup -c <cluster_name> -g <consumer_group_name> -n <nameserver_address> -o true
```

**发送消息**

和普通消息发送相比，顺序消息发送必须要设置消息组。消息组的粒度建议按照业务场景，尽可能细粒度设计，以便实现业务拆分和并发扩展。

**创建FIFO主题**

```bash
./bin/mqadmin updateTopic -c DefaultCluster -t FIFOTopic -o true -n 127.0.0.1:9876 -a +message.type=FIFO
```

- -c 集群名称
- -t Topic名称
- -n Nameserver地址
- **-o 创建顺序消息主题**

**创建FIFO订阅消费组**

```bash
./bin/mqadmin updateSubGroup -c DefaultCluster -g FIFOGroup -n 127.0.0.1:9876 -o true
```

- -c 集群名称
- -g ConsumerGroup名称
- -n Nameserver地址
- **-o 创建顺序订阅消费组**

以Java语言为例，收发顺序消息的示例代码如下：

```java
        //顺序消息发送。
        MessageBuilder messageBuilder = new MessageBuilderImpl();;
        Message message = messageBuilder.setTopic("topic")
                //设置消息索引键，可根据关键字精确查找某条消息。
                .setKeys("messageKey")
                //设置消息Tag，用于消费端根据指定Tag过滤消息。
                .setTag("messageTag")
                //设置顺序消息的排序分组，该分组尽量保持离散，避免热点排序分组。
                .setMessageGroup("fifoGroup001")
                //消息体。
                .setBody("messageBody".getBytes())
                .build();
        try {
            //发送消息，需要关注发送结果，并捕获失败等异常
            SendReceipt sendReceipt = producer.send(message);
            System.out.println(sendReceipt.getMessageId());
        } catch (ClientException e) {
            e.printStackTrace();
        }
        //消费顺序消息时，需要确保当前消费者分组是顺序投递模式，否则仍然按并发乱序投递。
        //消费示例一：使用PushConsumer消费顺序消息，只需要在消费监听器处理即可。
        MessageListener messageListener = new MessageListener() {
            @Override
            public ConsumeResult consume(MessageView messageView) {
                System.out.println(messageView);
                //根据消费结果返回状态。
                return ConsumeResult.SUCCESS;
            }
        };
        //消费示例二：使用SimpleConsumer消费顺序消息，主动获取消息进行消费处理并提交消费结果。
        //需要注意的是，同一个MessageGroup的消息，如果前序消息没有消费完成，再次调用Receive是获取不到后续消息的。
        List<MessageView> messageViewList = null;
        try {
            messageViewList = simpleConsumer.receive(10, Duration.ofSeconds(30));
            messageViewList.forEach(messageView -> {
                System.out.println(messageView);
                //消费处理完成后，需要主动调用ACK提交消费结果。
                try {
                    simpleConsumer.ack(messageView);
                } catch (ClientException e) {
                    e.printStackTrace();
                }
            });
        } catch (ClientException e) {
            //如果遇到系统流控等原因造成拉取失败，需要重新发起获取消息请求。
            e.printStackTrace();
        }
```

### 4.3.5 使用建议

**串行消费，避免批量消费导致乱序**

<u>消息消费建议串行处理，避免一次消费多条消息，否则可能出现乱序情况</u>。

例如：发送顺序为1->2->3->4，消费时批量消费，消费顺序为1->23（批量处理，失败）->23（重试处理）->4，此时可能由于消息3的失败导致消息2被重复处理，最后导致消息消费乱序。

**消息组尽可能打散，避免集中导致热点**

Apache RocketMQ 保证相同消息组的消息存储在同一个队列中，如果不同业务场景的消息都集中在少量或一个消息组中，则这些消息存储压力都会集中到服务端的少量队列或一个队列中。容易导致性能热点，且不利于扩展。一般建议的消息组设计会采用订单ID、用户ID作为顺序参考，即同一个终端用户的消息保证顺序，不同用户的消息无需保证顺序。

因此建议将业务以消息组粒度进行拆分，例如，将订单ID、用户ID作为消息组关键字，可实现同一终端用户的消息按照顺序处理，不同用户的消息无需保证顺序。

## 4.4 事务消息

事务消息为 Apache RocketMQ 中的高级特性消息，本文为您介绍事务消息的应用场景、功能原理、使用限制、使用方法和使用建议。

### 4.4.1 应用场景

**分布式事务的诉求**

分布式系统调用的特点为一个核心业务逻辑的执行，同时需要调用多个下游业务进行处理。因此，如何保证核心业务和多个下游业务的执行结果完全一致，是分布式事务需要解决的主要问题。 ![事务消息诉求](https://rocketmq.apache.org/zh/assets/images/tradetrans01-636d42fb6584de6c51692d0889af5c2d.png)

以电商交易场景为例，用户支付订单这一核心操作的同时会涉及到下游物流发货、积分变更、购物车状态清空等多个子系统的变更。当前业务的处理分支包括：

- 主分支订单系统状态更新：由未支付变更为支付成功。
- 物流系统状态新增：新增待发货物流记录，创建订单物流记录。
- 积分系统状态变更：变更用户积分，更新用户积分表。
- 购物车系统状态变更：清空购物车，更新用户购物车记录。

**传统XA事务方案：性能不足**

为了保证上述四个分支的执行结果一致性，典型方案是基于XA协议的分布式事务系统来实现。将四个调用分支封装成包含四个独立事务分支的大事务。基于XA分布式事务的方案可以满足业务处理结果的正确性，但最大的缺点是多分支环境下资源锁定范围大，并发度低，随着下游分支的增加，系统性能会越来越差。

**基于普通消息方案：一致性保障困难**

将上述基于XA事务的方案进行简化，将订单系统变更作为本地事务，剩下的系统变更作为普通消息的下游来执行，事务分支简化成普通消息+订单表事务，充分利用消息异步化的能力缩短链路，提高并发度。 ![普通消息方案](https://rocketmq.apache.org/zh/assets/images/transwithnormal-f7d951385520fc18aea8d85f0cd86c27.png)

该方案中消息下游分支和订单系统变更的主分支很容易出现不一致的现象，例如：

- 消息发送成功，订单没有执行成功，需要回滚整个事务。
- 订单执行成功，消息没有发送成功，需要额外补偿才能发现不一致。
- 消息发送超时未知，此时无法判断需要回滚订单还是提交订单变更。

**基于Apache RocketMQ分布式事务消息：支持最终一致性**

上述普通消息方案中，普通消息和订单事务无法保证一致的原因，本质上是由于普通消息无法像单机数据库事务一样，具备提交、回滚和统一协调的能力。

而**基于Apache RocketMQ实现的分布式事务消息功能，在普通消息基础上，支持二阶段的提交能力。将二阶段提交和本地事务绑定，实现全局提交结果的一致性。** ![事务消息](https://rocketmq.apache.org/zh/assets/images/tradewithtrans-25be17fcdedb8343a0d2633e693d126d.png)

Apache RocketMQ事务消息的方案，具备高性能、可扩展、业务开发简单的优势。具体事务消息的原理和流程，请参见下文的功能原理。

### 4.4.2 功能原理

**什么是事务消息**

事务消息是 Apache RocketMQ 提供的一种高级消息类型，支持在分布式场景下保障消息生产和本地事务的最终一致性。

**事务消息处理流程**

事务消息交互流程如下图所示。![事务消息](https://rocketmq.apache.org/zh/assets/images/transflow-0b07236d124ddb814aeaf5f6b5f3f72c.png)

1. 生产者将消息发送至Apache RocketMQ服务端。
2. Apache RocketMQ服务端将消息持久化成功之后，向生产者返回Ack确认消息已经发送成功，此时消息被标记为"暂不能投递"，这种状态下的消息即为**半事务消息**。
3. 生产者开始执行本地事务逻辑。
4. 生产者根据本地事务执行结果向服务端**提交二次确认结果（Commit或是Rollback）**，服务端收到确认结果后处理逻辑如下：
   - **二次确认结果为Commit：服务端将半事务消息标记为可投递，并投递给消费者。**
   - **二次确认结果为Rollback：服务端将回滚事务，不会将半事务消息投递给消费者。**
5. 在断网或者是生产者应用重启的特殊情况下，若服务端未收到发送者提交的二次确认结果，或服务端收到的二次确认结果为Unknown未知状态，经过固定时间后，服务端将对消息生产者即生产者集群中任一生产者实例发起消息回查。 **说明** 服务端回查的间隔时间和最大回查次数，请参见[参数限制](https://rocketmq.apache.org/zh/docs/introduction/03limits)。
6. **生产者收到消息回查后，需要检查对应消息的本地事务执行的最终结果**。
7. **生产者根据检查到的本地事务的最终状态再次提交二次确认，服务端仍按照步骤4对半事务消息进行处理**。

**事务消息生命周期** ![事务消息](https://rocketmq.apache.org/zh/assets/images/lifecyclefortrans-fe4a49f1c9fdae5d590a64546722036f.png)

- 初始化：半事务消息被生产者构建并完成初始化，待发送到服务端的状态。
- **事务待提交：半事务消息被发送到服务端，和普通消息不同，并不会直接被服务端持久化，而是会被单独存储到<u>事务存储系统</u>中，等待第二阶段本地事务返回执行结果后再提交。此时消息对下游消费者不可见**。
- 消息回滚：第二阶段如果事务执行结果明确为回滚，服务端会将半事务消息回滚，该事务消息流程终止。
- **提交待消费：第二阶段如果事务执行结果明确为提交，服务端会将半事务消息重新存储到普通存储系统中，此时消息对下游消费者可见，等待被消费者获取并消费**。
- 消费中：消息被消费者获取，并按照消费者本地的业务逻辑进行处理的过程。 此时服务端会等待消费者完成消费并提交消费结果，如果一定时间后没有收到消费者的响应，Apache RocketMQ会对消息进行重试处理。具体信息，请参见[消费重试](https://rocketmq.apache.org/zh/docs/featureBehavior/10consumerretrypolicy)。
- 消费提交：消费者完成消费处理，并向服务端提交消费结果，服务端标记当前消息已经被处理（包括消费成功和失败）。 Apache RocketMQ默认支持保留所有消息，此时消息数据并不会立即被删除，只是逻辑标记已消费。消息在保存时间到期或存储空间不足被删除前，消费者仍然可以回溯消息重新消费。
- 消息删除：Apache RocketMQ按照消息保存机制滚动清理最早的消息数据，将消息从物理文件中删除。更多信息，请参见[消息存储和清理机制](https://rocketmq.apache.org/zh/docs/featureBehavior/11messagestorepolicy)。

### 4.4.3 使用限制

**消息类型一致性**

<u>事务消息仅支持在 MessageType 为 Transaction 的主题内使用</u>，即事务消息只能发送至类型为事务消息的主题中，发送的消息的类型必须和主题的类型一致。

**消费事务性**

**Apache RocketMQ <u>事务消息保证本地主分支事务和下游消息发送事务的一致性，但不保证消息消费结果和上游事务的一致性</u>。因此需要下游业务分支自行保证消息正确处理，建议消费端做好[消费重试](https://rocketmq.apache.org/zh/docs/featureBehavior/10consumerretrypolicy)，如果有短暂失败可以利用重试机制保证最终处理成功**。

**中间状态可见性**

Apache RocketMQ 事务消息为**最终一致性**，即在消息提交到下游消费端处理完成之前，下游分支和上游事务之间的状态会不一致。因此，**事务消息仅适合接受<u>异步执行</u>的事务场景**。

**事务超时机制**

Apache RocketMQ 事务消息的生命周期存在超时机制，即<u>半事务消息被生产者发送服务端后，如果在指定时间内服务端无法确认提交或者回滚状态，则消息默认会被回滚</u>。事务超时时间，请参见[参数限制](https://rocketmq.apache.org/zh/docs/introduction/03limits)。

### 4.4.4 使用示例

**创建主题**

Apache RocketMQ 5.0版本下创建主题操作，推荐使用mqadmin工具，需要注意的是，对于消息类型需要通过属性参数添加。示例如下：

```shell
sh mqadmin updateTopic -n <nameserver_address> -t <topic_name> -c <cluster_name> -a +message.type=TRANSACTION
```

**发送消息**

事务消息相比普通消息发送时需要修改以下几点：

- 发送事务消息前，需要开启事务并关联本地的事务执行。
- 为保证事务一致性，在构建生产者时，<u>必须设置事务检查器和预绑定事务消息发送的主题列表，客户端内置的事务检查器会对绑定的事务主题做异常状态恢复</u>。

**创建事务主题**

*NORMAL类型Topic不支持TRANSACTION类型消息，生产消息会报错。*

```bash
./bin/mqadmin updatetopic -n localhost:9876 -t TestTopic -c DefaultCluster -a +message.type=TRANSACTION
```

- -c 集群名称
- -t Topic名称
- -n nameserver地址
- -a 额外属性，本例给主题添加了`message.type`为`TRANSACTION`的属性用来支持事务消息

以Java语言为例，使用事务消息示例参考如下：

```java
    //演示demo，模拟订单表查询服务，用来确认订单事务是否提交成功。
    private static boolean checkOrderById(String orderId) {
        return true;
    }
    //演示demo，模拟本地事务的执行结果。
    private static boolean doLocalTransaction() {
        return true;
    }
    public static void main(String[] args) throws ClientException {
        ClientServiceProvider provider = new ClientServiceProvider();
        MessageBuilder messageBuilder = new MessageBuilderImpl();
        //构造事务生产者：事务消息需要生产者构建一个事务检查器，用于检查确认异常半事务的中间状态。
        Producer producer = provider.newProducerBuilder()
                .setTransactionChecker(messageView -> {
                    /**
                     * 事务检查器一般是根据业务的ID去检查本地事务是否正确提交还是回滚，此处以订单ID属性为例。
                     * 在订单表找到了这个订单，说明本地事务插入订单的操作已经正确提交；如果订单表没有订单，说明本地事务已经回滚。
                     */
                    final String orderId = messageView.getProperties().get("OrderId");
                    if (Strings.isNullOrEmpty(orderId)) {
                        // 错误的消息，直接返回Rollback。
                        return TransactionResolution.ROLLBACK;
                    }
                    return checkOrderById(orderId) ? TransactionResolution.COMMIT : TransactionResolution.ROLLBACK;
                })
                .build();
        //开启事务分支。
        final Transaction transaction;
        try {
            transaction = producer.beginTransaction();
        } catch (ClientException e) {
            e.printStackTrace();
            //事务分支开启失败，直接退出。
            return;
        }
        Message message = messageBuilder.setTopic("topic")
                //设置消息索引键，可根据关键字精确查找某条消息。
                .setKeys("messageKey")
                //设置消息Tag，用于消费端根据指定Tag过滤消息。
                .setTag("messageTag")
                //一般事务消息都会设置一个本地事务关联的唯一ID，用来做本地事务回查的校验。
                .addProperty("OrderId", "xxx")
                //消息体。
                .setBody("messageBody".getBytes())
                .build();
        //发送半事务消息
        final SendReceipt sendReceipt;
        try {
            sendReceipt = producer.send(message, transaction);
        } catch (ClientException e) {
            //半事务消息发送失败，事务可以直接退出并回滚。
            return;
        }
        /**
         * 执行本地事务，并确定本地事务结果。
         * 1. 如果本地事务提交成功，则提交消息事务。
         * 2. 如果本地事务提交失败，则回滚消息事务。
         * 3. 如果本地事务未知异常，则不处理，等待事务消息回查。
         *
         */
        boolean localTransactionOk = doLocalTransaction();
        if (localTransactionOk) {
            try {
                transaction.commit();
            } catch (ClientException e) {
                // 业务可以自身对实时性的要求选择是否重试，如果放弃重试，可以依赖事务消息回查机制进行事务状态的提交。
                e.printStackTrace();
            }
        } else {
            try {
                transaction.rollback();
            } catch (ClientException e) {
                // 建议记录异常信息，回滚异常时可以无需重试，依赖事务消息回查机制进行事务状态的提交。
                e.printStackTrace();
            }
        }
    }
```

### 4.4.5 使用建议

**避免大量未决事务导致超时**

Apache RocketMQ支持在事务提交阶段异常的情况下发起事务回查，保证事务一致性。但生产者应该尽量避免本地事务返回未知结果。<u>大量的事务检查会导致系统性能受损，容易导致事务处理延迟</u>。

**正确处理"进行中"的事务**

消息回查时，对于正在进行中的事务不要返回Rollback或Commit结果，应继续保持Unknown的状态。 一般出现消息回查时事务正在处理的原因为：事务执行较慢，消息回查太快。解决方案如下：

- 将第一次事务回查时间设置较大一些，但可能导致依赖回查的事务提交延迟较大。
- 程序能正确识别正在进行中的事务。

## 4.5 消息发送重试和流控机制

本文为您介绍 Apache RocketMQ 的消息发送重试机制和消息流控机制。

### 4.5.1 背景信息

**消息发送重试**

Apache RocketMQ的消息发送重试机制主要为您解答如下问题：

- 部分节点异常是否影响消息发送？
- 请求重试是否会阻塞业务调用？
- 请求重试会带来什么不足？

**消息流控**

Apache RocketMQ 的流控机制主要为您解答如下问题：

- 系统在什么情况下会触发流控？
- 触发流控时客户端行为是什么？
- 应该如何避免触发流控，以及如何应对突发流控？

### 4.5.2 消息发送重试机制

#### 4.5.2.1 重试基本概念

Apache RocketMQ 客户端连接服务端发起消息发送请求时，可能会因为网络故障、服务异常等原因导致调用失败。为保证消息的可靠性， Apache RocketMQ 在客户端SDK中内置请求重试逻辑，尝试通过重试发送达到最终调用成功的效果。

同步发送和异步发送模式均支持消息发送重试。

**重试触发条件**

触发消息发送重试机制的条件如下：

- 客户端消息发送请求调用失败或请求超时
- 网络异常造成连接失败或请求超时。
- 服务端节点处于重启或下线等状态造成连接失败。
- 服务端运行慢造成请求超时。
- 服务端返回失败错误码
  - 系统逻辑错误：因运行逻辑不正确造成的错误。
  - 系统流控错误：因容量超限造成的流控错误。

> 对于事务消息，只会进行[透明重试（transparent retries）](https://github.com/grpc/proposal/blob/master/A6-client-retries.md#transparent-retries)，网络超时或异常等场景不会进行重试。

#### 4.5.2.2 重试流程

生产者在初始化时设置消息发送最大重试次数，当出现上述触发条件的场景时，生产者客户端会按照设置的重试次数一直重试发送消息，直到消息发送成功或达到最大重试次数重试结束，并在最后一次重试失败后返回调用错误响应。

- 同步发送：调用线程会一直阻塞，直到某次重试成功或最终重试失败，抛出错误码和异常。
- 异步发送：调用线程不会阻塞，但调用结果会通过异常事件或者成功事件返回。

#### 4.5.2.3 重试间隔

- 除服务端返回系统流控错误场景，其他触发条件触发重试后，均会立即进行重试，无等待间隔。
- **若由于服务端返回流控错误触发重试，系统会按照指数退避策略进行延迟重试**。指数退避算法通过以下参数控制重试行为：
  - INITIAL_BACKOFF： 第一次失败重试前后需等待多久，默认值：1秒。
  - MULTIPLIER ：指数退避因子，即退避倍率，默认值：1.6。
  - JITTER ：随机抖动因子，默认值：0.2。
  - MAX_BACKOFF ：等待间隔时间上限，默认值：120秒
  - MIN_CONNECT_TIMEOUT ：最短重试间隔，默认值：20秒。

**建议算法如下：**

```unknow
ConnectWithBackoff()
  current_backoff = INITIAL_BACKOFF
  current_deadline = now() + INITIAL_BACKOFF
  while (TryConnect(Max(current_deadline, now() + MIN_CONNECT_TIMEOUT))!= SUCCESS)
    SleepUntil(current_deadline)
    current_backoff = Min(current_backoff * MULTIPLIER, MAX_BACKOFF)
    current_deadline = now() + current_backoff + UniformRandom(-JITTER * current_backoff, JITTER * current_backoff)
```

更多信息，请参见[connection-backoff 策略](https://github.com/grpc/grpc/blob/master/doc/connection-backoff.md)。

#### 4.5.2.4 功能约束

- 链路耗时阻塞评估：从上述重试机制可以看出，在重试流程中生产者仅能控制最大重试次数。若由于系统异常触发了SDK内置的重试逻辑，则服务端需要等待最终重试结果，可能会导致消息发送请求链路被阻塞。对于某些实时调用类场景，您需要合理评估每次调用请求的超时时间以及最大重试次数，避免影响全链路的耗时。
- 最终异常兜底： Apache RocketMQ 客户端内置的发送请求重试机制并不能保证消息发送一定成功。当最终重试仍然失败时，业务方调用需要捕获异常，并做好冗余保护处理，避免消息发送结果不一致。
- **消息重复问题：因远程调用的不确定性，当Apache RocketMQ客户端因请求超时触发消息发送重试流程，此时客户端无法感知服务端的处理结果，客户端进行的消息发送重试可能会产生消息重复问题，业务逻辑需要自行处理消息重复问题**。

### 4.5.3 消息流控机制

#### 4.5.3.1 消息流控基本概念

消息流控指的是系统容量或水位过高， Apache RocketMQ 服务端会通过快速失败返回流控错误来避免底层资源承受过高压力。

#### 4.5.3.2 触发条件

Apache RocketMQ 的消息流控触发条件如下：

- **存储压力大**：参考[消费进度管理](https://rocketmq.apache.org/zh/docs/featureBehavior/09consumerprogress)的原理机制，消费者分组的初始消费位点为当前队列的最大消费位点。若某些场景例如业务上新等需要回溯到指定时刻前开始消费，此时队列的存储压力会瞬间飙升，触发消息流控。
- **服务端请求任务排队溢出**：若消费者消费能力不足，导致队列中有大量堆积消息，当堆积消息超过一定数量后会触发消息流控，减少下游消费系统压力。

#### 4.5.3.3 流控行为

当系统触发消息发送流控时，客户端会收到系统限流错误和异常，错误码信息如下：

- reply-code：530
- reply-text：TOO_MANY_REQUESTS

客户端收到系统流控错误码后，会根据指数退避策略进行消息发送重试。

#### 4.5.3.4 处理建议

- 如何避免触发消息流控：触发限流的根本原因是系统容量或水位过高，您可以利用可观测性功能监控系统水位容量等，保证底层资源充足，避免触发流控机制。
- 突发消息流控处理：如果因为突发原因触发消息流控，且客户端内置的重试流程执行失败，则建议业务方将请求调用临时替换到其他系统进行应急处理。

## 4.6 消费者分类

Apache RocketMQ 支持 PushConsumer 、 SimpleConsumer 以及 PullConsumer 这三种类型的消费者，本文分别从使用方式、实现原理、可靠性重试和适用场景等方面为您介绍这三种类型的消费者。

### 4.6.1 背景信息

Apache RocketMQ 面向不同的业务场景提供了不同消费者类型，每种消费者类型的集成方式和控制方式都不一样。了解如下问题，可以帮助您选择更匹配业务场景的消费者类型。

- 如何实现并发消费：消费者如何使用并发的多线程机制处理消息，以此提高消息处理效率？
- 如何实现同步、异步消息处理：对于不同的集成场景，消费者获取消息后可能会将消息异步分发到业务逻辑中处理，此时，消息异步化处理如何实现？
- 如何实现消息可靠处理：消费者处理消息时如何返回响应结果？如何在消息异常情况进行重试，保证消息的可靠处理？

以上问题的具体答案，请参考下文。

### 4.6.2 功能概述

![消息消费流程](https://rocketmq.apache.org/zh/assets/images/consumerflow-eaa625a6a01a048a155a3809a603529a.png)

如上图所示， Apache RocketMQ 的消费者处理消息时主要经过以下阶段：消息获取--->消息处理--->消费状态提交。

针对以上几个阶段，Apache RocketMQ 提供了不同的消费者类型： PushConsumer 、SimpleConsumer 和 PullConsumer。这几种类型的消费者通过不同的实现方式和接口可满足您在不同业务场景下的消费需求。具体差异如下：

> 在实际使用场景中，PullConsumer 仅推荐在流处理框架中集成使用，**大多数消息收发场景使用 PushConsumer 和 SimpleConsumer 就可以满足需求**。
>
> 若您的业务场景发生变更，或您当前使用的消费者类型不适合当前业务，您可以选择在 PushConsumer 和SimpleConsumer 之间变更消费者类型。**变更消费者类型不影响当前Apache RocketMQ 资源的使用和业务处理**。

> **<u>生产环境中相同的 ConsumerGroup 下严禁混用 PullConsumer 和其他两种消费者，否则会导致消息消费异常</u>**。

| 对比项         | PushConsumer                                                 | SimpleConsumer                                       | PullConsumer(类比kafka)                            |
| -------------- | ------------------------------------------------------------ | ---------------------------------------------------- | -------------------------------------------------- |
| 接口方式       | 使用监听器回调接口返回消费结果，消费者仅允许在监听器范围内处理消费逻辑。 | 业务方自行实现消息处理，并主动调用接口返回消费结果。 | 业务方自行按队列拉取消息，并可选择性地提交消费结果 |
| 消费并发度管理 | 由SDK管理消费并发度。                                        | 由业务方消费逻辑自行管理消费线程。                   | 由业务方消费逻辑自行管理消费线程。                 |
| 负载均衡粒度   | 5.0 SDK是消息粒度，更均衡，早期版本是队列维度                | 消息粒度，更均衡                                     | **队列粒度，吞吐攒批性能更好，但容易不均衡**       |
| 接口灵活度     | 高度封装，不够灵活。                                         | 原子接口，可灵活自定义。                             | 原子接口，可灵活自定义。                           |
| 适用场景       | 适用于无自定义流程的业务消息开发场景。                       | 适用于需要高度自定义业务流程的业务开发场景。         | 仅推荐在流处理框架场景下集成使用                   |

### 4.6.3 PushConsumer

PushConsumers是一种高度封装的消费者类型，消费消息仅通过消费监听器处理业务并返回消费结果。消息的获取、消费状态提交以及消费重试都通过 Apache RocketMQ 的客户端SDK完成。

**使用方式**

PushConsumer的使用方式比较固定，在消费者初始化时注册一个消费监听器，并在消费监听器内部实现消息处理逻辑。由 Apache RocketMQ 的SDK在后台完成消息获取、触发监听器调用以及进行消息重试处理。

示例代码如下：

```java
// 消费示例：使用PushConsumer消费普通消息。
ClientServiceProvider provider = ClientServiceProvider.loadService();
String topic = "YourTopic";
FilterExpression filterExpression = new FilterExpression("YourFilterTag", FilterExpressionType.TAG);
PushConsumer pushConsumer = provider.newPushConsumerBuilder()
    // 设置消费者分组。
    .setConsumerGroup("YourConsumerGroup")
    // 设置接入点。
    .setClientConfiguration(ClientConfiguration.newBuilder().setEndpoints("YourEndpoint").build())
    // 设置预绑定的订阅关系。
    .setSubscriptionExpressions(Collections.singletonMap(topic, filterExpression))
    // 设置消费监听器。
    .setMessageListener(new MessageListener() {
        @Override
        public ConsumeResult consume(MessageView messageView) {
            // 消费消息并返回处理结果。
            return ConsumeResult.SUCCESS;
        }
    })
    .build();
                
```

PushConsumer的消费监听器执行结果分为以下三种情况：

- 返回消费成功：以Java SDK为例，返回`ConsumeResult.SUCCESS`，表示该消息处理成功，服务端按照消费结果更新消费进度。
- 返回消费失败：以Java SDK为例，返回`ConsumeResult.FAILURE`，表示该消息处理失败，需要根据消费重试逻辑判断是否进行重试消费。
- 出现非预期失败：例如抛异常等行为，该结果按照消费失败处理，需要根据消费重试逻辑判断是否进行重试消费。

**PushConsumer 消费消息时，若消息处理逻辑出现预期之外的阻塞导致消息处理一直无法执行成功，SDK会按照消费超时处理强制提交消费失败结果，并按照消费重试逻辑进行处理**。消息超时，请参见[PushConsumer消费重试策略](https://rocketmq.apache.org/zh/docs/featureBehavior/10consumerretrypolicy)。

> **出现消费超时情况时，SDK虽然提交消费失败结果，但是当前消费线程可能仍然无法响应中断，还会继续处理消息。**

**内部原理**

**在PushConsumer类型中，消息的实时处理能力是基于SDK内部的典型<u>Reactor线程模型</u>实现的**。<u>如下图所示，SDK内置了一个长轮询线程，先将消息异步拉取到SDK内置的缓存队列中，再分别提交到消费线程中，触发监听器执行本地消费逻辑</u>。 ![PushConsumer原理](https://rocketmq.apache.org/zh/assets/images/pushconsumer-26b909b090d4f911a40d5050d3ceba1d.png)

**可靠性重试**

PushConsumer 消费者类型中，客户端SDK和消费逻辑的唯一边界是消费监听器接口。客户端SDK严格按照监听器的返回结果判断消息是否消费成功，并做可靠性重试。所有消息必须以同步方式进行消费处理，并在监听器接口结束时返回调用结果，不允许再做异步化分发。消息重试具体信息，请参见[PushConsumer消费重试策略](https://rocketmq.apache.org/zh/docs/featureBehavior/10consumerretrypolicy)。

使用PushConsumer消费者消费时，不允许使用以下方式处理消息，否则 Apache RocketMQ 无法保证消息的可靠性。

- <u>错误方式一：消息还未处理完成，就提前返回消费成功结果。此时如果消息消费失败，Apache RocketMQ 服务端是无法感知的，因此不会进行消费重试</u>。
- **错误方式二：在消费监听器内将消息再次分发到自定义的其他线程，消费监听器提前返回消费结果。此时如果消息消费失败，Apache RocketMQ 服务端同样无法感知，因此也不会进行消费重试**。

**顺序性保障**

基于 Apache RocketMQ [顺序消息](https://rocketmq.apache.org/zh/docs/featureBehavior/03fifomessage)的定义，如果消费者分组设置了顺序消费模式，则PushConsumer在触发消费监听器时，严格遵循消息的先后顺序。业务处理逻辑无感知即可保证消息的消费顺序。

> **消息消费按照顺序处理的前提是遵循同步提交原则，如果业务逻辑自定义实现了异步分发，则Apache RocketMQ 无法保证消息的顺序性。**

**适用场景**

PushConsumer严格限制了消息同步处理及每条消息的处理超时时间，适用于以下场景：

- 消息处理时间可预估：如果不确定消息处理耗时，经常有预期之外的长时间耗时的消息，<u>PushConsumer的可靠性保证会频繁触发消息重试机制造成大量重复消息</u>。
- 无异步化、高级定制场景：PushConsumer限制了消费逻辑的线程模型，由客户端SDK内部按最大吞吐量触发消息处理。**该模型开发逻辑简单，但是不允许使用异步化和自定义处理流程**。

### 4.6.4 SimpleConsumer

SimpleConsumer 是一种接口原子型的消费者类型，消息的获取、消费状态提交以及消费重试都是通过消费者业务逻辑<u>主动发起</u>调用完成。

**使用方式**

SimpleConsumer 的使用涉及多个接口调用，由业务逻辑按需调用接口获取消息，然后分发给业务线程处理消息，最后按照处理的结果调用提交接口，返回服务端当前消息的处理结果。示例如下：

```java
// 消费示例：使用 SimpleConsumer 消费普通消息，主动获取消息处理并提交。 
ClientServiceProvider provider = ClientServiceProvider.loadService();
String topic = "YourTopic";
FilterExpression filterExpression = new FilterExpression("YourFilterTag", FilterExpressionType.TAG);
SimpleConsumer simpleConsumer = provider.newSimpleConsumerBuilder()
        // 设置消费者分组。
        .setConsumerGroup("YourConsumerGroup")
        // 设置接入点。
        .setClientConfiguration(ClientConfiguration.newBuilder().setEndpoints("YourEndpoint").build())
        // 设置预绑定的订阅关系。
        .setSubscriptionExpressions(Collections.singletonMap(topic, filterExpression))
        // 设置从服务端接受消息的最大等待时间
        .setAwaitDuration(Duration.ofSeconds(1))
        .build();
try {
    // SimpleConsumer 需要主动获取消息，并处理。
    List<MessageView> messageViewList = simpleConsumer.receive(10, Duration.ofSeconds(30));
    messageViewList.forEach(messageView -> {
        System.out.println(messageView);
        // 消费处理完成后，需要主动调用 ACK 提交消费结果。
        try {
            simpleConsumer.ack(messageView);
        } catch (ClientException e) {
            logger.error("Failed to ack message, messageId={}", messageView.getMessageId(), e);
        }
    });
} catch (ClientException e) {
    // 如果遇到系统流控等原因造成拉取失败，需要重新发起获取消息请求。
    logger.error("Failed to receive message", e);
}
```

SimpleConsumer主要涉及以下几个接口行为：

| 接口名称                  | 主要作用                                                     | 可修改参数                                                   |
| ------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `ReceiveMessage`          | 消费者主动调用该接口从服务端获取消息。 **说明** 由于服务端存储为分布式，可能会出现服务端实际有消息，但是返回为空的现象。 一般可通过重新发起ReceiveMessage调用或提高ReceiveMessage的并发度解决。 | *批量拉取消息数：SimpleConsumer可以一次性批量获取多条消息实现批量消费，该接口可修改批量获取的消息数量。* 消费不可见时间：消息的最长处理耗时，该参数用于控制消费失败时的消息重试间隔。具体信息，请参见[SimpleConsumer消费重试策略](https://rocketmq.apache.org/zh/docs/featureBehavior/10consumerretrypolicy)。消费者调用`ReceiveMessage`接口时需要指定消费不可见时间。 |
| `AckMessage`              | 消费者成功消费消息后，主动调用该接口向服务端返回消费成功响应。 | 无                                                           |
| `ChangeInvisibleDuration` | 消费重试场景下，消费者可通过该接口修改消息处理时长，即控制消息的重试间隔。 | 消费不可见时间：调用本接口可修改`ReceiveMessage`接口预设的消费不可见时间的参数值。一般用于需要延长消息处理时长的场景。 |

**可靠性重试**

SimpleConsumer消费者类型中，客户端SDK和服务端通过`ReceiveMessage`和`AckMessage`接口通信。客户端SDK如果处理消息成功则调用`AckMessage`接口；<u>如果处理失败只需要不回复ACK响应，即可在定义的消费不可见时间到达后触发消费重试流程</u>。更多信息，请参见[SimpleConsumer消费重试策略](https://rocketmq.apache.org/zh/docs/featureBehavior/10consumerretrypolicy)。

**顺序性保障**

基于 Apache RocketMQ [顺序消息](https://rocketmq.apache.org/zh/docs/featureBehavior/03fifomessage)的定义，SimpleConsumer在处理顺序消息时，会按照消息存储的先后顺序获取消息。即需要保持顺序的一组消息中，如果前面的消息未处理完成，则无法获取到后面的消息。

适用场景

SimpleConsumer提供原子接口，用于消息获取和提交消费结果，相对于PushConsumer方式更加灵活。SimpleConsumer适用于以下场景：

- <u>消息处理时长不可控</u>：如果消息处理时长无法预估，经常有长时间耗时的消息处理情况。建议使用SimpleConsumer消费类型，可以在消费时自定义消息的预估处理时长，若实际业务中预估的消息处理时长不符合预期，也可以通过接口提前修改。
- <u>需要异步化、批量消费等高级定制场景</u>：SimpleConsumer在SDK内部没有复杂的线程封装，完全由业务逻辑自由定制，可以实现异步分发、批量消费等高级定制场景。
- <u>需要自定义消费速率</u>：SimpleConsumer是由业务逻辑主动调用接口获取消息，因此可以自由调整获取消息的频率，自定义控制消费速率。

### 4.6.5 PullConsumer

暂无描述

### 4.6.6 使用建议

**PushConsumer合理控制消费耗时，避免无限阻塞**

<u>对于PushConsumer消费类型，需要严格控制消息的消费耗时，尽量避免出现消息处理超时导致消息重复。如果业务经常会出现一些预期外的长时间耗时的消息，建议使用SimpleConsumer，并设置好消费不可见时间</u>。

## 4.7 消息过滤

消费者订阅了某个主题后，Apache RocketMQ 会将该主题中的所有消息投递给消费者。若消费者只需要关注部分消息，可通过设置过滤条件在 Apache RocketMQ 服务端进行过滤，只获取到需要关注的消息子集，避免接收到大量无效的消息。本文介绍消息过滤的定义、原理、分类及不同过滤方式的使用方法、配置示例等。

### 4.7.1 应用场景

Apache RocketMQ 作为发布订阅模型的消息中间件广泛应用于上下游业务集成场景。在实际业务场景中，同一个主题下的消息往往会被多个不同的下游业务方处理，各下游的处理逻辑不同，只关注自身逻辑需要的消息子集。

使用 Apache RocketMQ 的消息过滤功能，可以帮助消费者更高效地过滤自己需要的消息集合，避免大量无效消息投递给消费者，降低下游系统处理压力。

Apache RocketMQ 主要解决的单个业务域即同一个主题内不同消息子集的过滤问题，一般是基于同一业务下更具体的分类进行过滤匹配。如果是需要对不同业务域的消息进行拆分，建议使用不同主题处理不同业务域的消息。

### 4.7.2 功能概述

**消息过滤定义**

<u>过滤的含义指的是将符合条件的消息投递给消费者，而不是将匹配到的消息过滤掉</u>。

Apache RocketMQ 的消息过滤功能通过生产者和消费者对消息的属性、标签进行定义，并在 Apache RocketMQ 服务端根据过滤条件进行筛选匹配，将符合条件的消息投递给消费者进行消费。

**消息过滤原理** ![消息过滤](https://rocketmq.apache.org/zh/assets/images/messagefilter0-ad2c8360f54b9a622238f8cffea12068.png)

消息过滤主要通过以下几个关键流程实现：

- 生产者：生产者在初始化消息时预先为消息设置一些属性和标签，用于后续消费时指定过滤目标。
- 消费者：消费者在初始化及后续消费流程中通过调用订阅关系注册接口，向服务端上报需要订阅指定主题的哪些消息，即过滤条件。
- 服务端：消费者获取消息时会触发服务端的动态过滤计算，Apache RocketMQ 服务端根据消费者上报的过滤条件的表达式进行匹配，并将符合条件的消息投递给消费者。

**消息过滤分类**

Apache RocketMQ 支持Tag标签过滤和SQL属性过滤，这两种过滤方式对比如下：

| 对比项   | Tag标签过滤                      | SQL属性过滤                                                  |
| -------- | -------------------------------- | ------------------------------------------------------------ |
| 过滤目标 | 消息的Tag标签。                  | 消息的属性，包括用户自定义属性以及系统属性（Tag是一种系统属性）。 |
| 过滤能力 | 精准匹配。                       | SQL语法匹配。                                                |
| 适用场景 | 简单过滤场景、计算逻辑简单轻量。 | 复杂过滤场景、计算逻辑较复杂。                               |

具体的使用方式及示例，请参见下文的Tag标签过滤和SQL属性过滤。

### 4.7.3 订阅关系一致性

过滤表达式属于订阅关系的一部分，Apache RocketMQ 的领域模型规定，同一消费者分组内的多个消费者的订阅关系包括过滤表达式，必须保持一致，否则可能会导致部分消息消费不到。更多信息，请参见[订阅关系（Subscription）](https://rocketmq.apache.org/zh/docs/domainModel/09subscription)。

### 4.7.4 Tag标签过滤

Tag标签过滤方式是 Apache RocketMQ 提供的基础消息过滤能力，基于生产者为消息设置的Tag标签进行匹配。生产者在发送消息时，设置消息的Tag标签，消费者需指定已有的Tag标签来进行匹配订阅。

**场景示例**

以下图电商交易场景为例，从客户下单到收到商品这一过程会生产一系列消息：

- 订单消息
- 支付消息
- 物流消息

这些消息会发送到名称为Trade_Topic的Topic中，被各个不同的下游系统所订阅：

- 支付系统：只需订阅支付消息。
- 物流系统：只需订阅物流消息。
- 交易成功率分析系统：需订阅订单和支付消息。
- 实时计算系统：需要订阅所有和交易相关的消息。

过滤效果如下图所示：![Tag过滤](https://rocketmq.apache.org/zh/assets/images/messagefilter-09e82bf396d7c4100ed742e8d0d2c185.png)

**Tag标签设置**

- Tag由生产者发送消息时设置，**每条消息允许设置一个Tag标签**。
- Tag使用可见字符，建议长度不超过128字符。

**Tag标签过滤规则**

Tag标签过滤为精准字符串匹配，过滤规则设置格式如下：

- 单Tag匹配：过滤表达式为目标Tag。表示只有消息标签为指定目标Tag的消息符合匹配条件，会被发送给消费者。
- 多Tag匹配：多个Tag之间为或的关系，不同Tag间使用两个竖线（||）隔开。例如，Tag1||Tag2||Tag3，表示标签为Tag1或Tag2或Tag3的消息都满足匹配条件，都会被发送给消费者进行消费。
- 全部匹配：使用星号（*）作为全匹配表达式。表示主题下的所有消息都将被发送给消费者进行消费。

**使用示例**

- 发送消息，设置Tag标签。

  ```java
  Message message = messageBuilder.setTopic("topic")
  //设置消息索引键，可根据关键字精确查找某条消息。
  .setKeys("messageKey")
  //设置消息Tag，用于消费端根据指定Tag过滤消息。
  //该示例表示消息的Tag设置为"TagA"。
  .setTag("TagA")
  //消息体。
  .setBody("messageBody".getBytes())
  .build();
  ```

- 订阅消息，匹配单个Tag标签。

  ```java
  String topic = "Your Topic";
  //只订阅消息标签为"TagA"的消息。
  FilterExpression filterExpression = new FilterExpression("TagA", FilterExpressionType.TAG);
  pushConsumer.subscribe(topic, filterExpression);
  ```

- 订阅消息，匹配多个Tag标签。

  ```java
  String topic = "Your Topic";
  //只订阅消息标签为"TagA"、"TagB"或"TagC"的消息。
  FilterExpression filterExpression = new FilterExpression("TagA||TagB||TagC", FilterExpressionType.TAG);
  pushConsumer.subscribe(topic, filterExpression);
  ```

- 订阅消息，匹配Topic中的所有消息，不进行过滤。

  ```java
  String topic = "Your Topic";
  //使用Tag标签过滤消息，订阅所有消息。
  FilterExpression filterExpression = new FilterExpression("*", FilterExpressionType.TAG);
  pushConsumer.subscribe(topic, filterExpression);
  ```

### 4.7.5 SQL属性过滤

SQL属性过滤是 Apache RocketMQ 提供的高级消息过滤方式，通过生产者为消息设置的属性（Key）及属性值（Value）进行匹配。生产者在发送消息时**可设置多个属性**，消费者订阅时可设置SQL语法的过滤表达式过滤多个属性。

> <u>Tag是一种系统属性，所以SQL过滤方式也兼容Tag标签过滤。在SQL语法中，Tag的属性名称为TAGS</u>。

**场景示例**

以下图电商交易场景为例，从客户下单到收到商品这一过程会生产一系列消息，按照类型将消息分为订单消息和物流消息，其中给物流消息定义地域属性，按照地域分为杭州和上海：

- 订单消息
- 物流消息
  - 物流消息且地域为杭州
  - 物流消息且地域为上海

这些消息会发送到名称为Trade_Topic的Topic中，被各个不同的系统所订阅：

- 物流系统1：只需订阅物流消息且消息地域为杭州。
- 物流系统2：只需订阅物流消息且消息地域为杭州或上海。
- 订单跟踪系统：只需订阅订单消息。
- 实时计算系统：需要订阅所有和交易相关的消息。

过滤效果如下图所示：![sql过滤](https://rocketmq.apache.org/zh/assets/images/messagefilter2-dbf55cf4a63ac6d3b9c5f02603ce92ce.png)

**消息属性设置**

生产者发送消息时可以自定义消息属性，每个属性都是一个自定义的键值对（Key-Value）。

每条消息支持设置**多个属性**。

**SQL属性过滤规则**

SQL属性过滤使用SQL92语法作为过滤规则表达式，语法规范如下：

| 语法                    | 说明                                                         | 示例                                                         |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| IS NULL                 | 判断属性不存在。                                             | `a IS NULL` ：属性a不存在。                                  |
| IS NOT NULL             | 判断属性存在。                                               | `a IS NOT NULL`：属性a存在。                                 |
| *>* >= *<* <=           | 用于比较数字，不能用于比较字符串，否则消费者客户端启动时会报错。 **说明** 可转化为数字的字符串也被认为是数字。 | *`a IS NOT NULL AND a > 100`：属性a存在且属性a的值大于100。* `a IS NOT NULL AND a > 'abc'`：错误示例，abc为字符串，不能用于比较大小。 |
| BETWEEN xxx AND xxx     | 用于比较数字，不能用于比较字符串，否则消费者客户端启动时会报错。等价于>= xxx AND \<= xxx。表示属性值在两个数字之间。 | `a IS NOT NULL AND (a BETWEEN 10 AND 100)`：属性a存在且属性a的值大于等于10且小于等于100。 |
| NOT BETWEEN xxx AND xxx | 用于比较数字，不能用于比较字符串，否则消费者客户端启动会报错。等价于\< xxx OR > xxx，表示属性值在两个值的区间之外。 | `a IS NOT NULL AND (a NOT BETWEEN 10 AND 100)`：属性a存在且属性a的值小于10或大于100。 |
| IN (xxx, xxx)           | 表示属性的值在某个集合内。集合的元素只能是字符串。           | `a IS NOT NULL AND (a IN ('abc', 'def'))`：属性a存在且属性a的值为abc或def。 |
| *=* <>                  | 等于和不等于。可用于比较数字和字符串。                       | `a IS NOT NULL AND (a = 'abc' OR a<>'def')`：属性a存在且属性a的值为abc或a的值不为def。 |
| *AND* OR                | 逻辑与、逻辑或。可用于组合任意简单的逻辑判断，需要将每个逻辑判断内容放入括号内。 | `a IS NOT NULL AND (a > 100) OR (b IS NULL)`：属性a存在且属性a的值大于100或属性b不存在。 |

由于SQL属性过滤是生产者定义消息属性，消费者设置SQL过滤条件，因此过滤条件的计算结果具有不确定性，服务端的处理方式如下：

- 异常情况处理：如果过滤条件的表达式计算抛异常，消息默认被过滤，不会被投递给消费者。例如比较数字和非数字类型的值。
- 空值情况处理：如果过滤条件的表达式计算值为null或不是布尔类型（true和false），则消息默认被过滤，不会被投递给消费者。例如发送消息时未定义某个属性，在订阅时过滤条件中直接使用该属性，则过滤条件的表达式计算结果为null。
- 数值类型不符处理：如果消息自定义属性为浮点型，但过滤条件中使用整数进行判断，则消息默认被过滤，不会被投递给消费者。

**使用示例**

- 发送消息，同时设置消息Tag标签和自定义属性。

  ```java
  Message message = messageBuilder.setTopic("topic")
  //设置消息索引键，可根据关键字精确查找某条消息。
  .setKeys("messageKey")
  //设置消息Tag，用于消费端根据指定Tag过滤消息。
  //该示例表示消息的Tag设置为"messageTag"。
  .setTag("messageTag")
  //消息也可以设置自定义的分类属性，例如环境标签、地域、逻辑分支。
  //该示例表示为消息自定义一个属性，该属性为地域，属性值为杭州。
  .addProperty("Region", "Hangzhou")
  //消息体。
  .setBody("messageBody".getBytes())
  .build();
  ```

- 订阅消息，根据单个自定义属性匹配消息。

  ```java
  String topic = "topic";
  //只订阅地域属性为杭州的消息。
  FilterExpression filterExpression = new FilterExpression("Region IS NOT NULL AND Region='Hangzhou'", FilterExpressionType.SQL92);
  simpleConsumer.subscribe(topic, filterExpression);
  ```

- 订阅消息，同时根据多个自定义属性匹配消息。

  ```java
  String topic = "topic";
  //只订阅地域属性为杭州且价格属性大于30的消息。
  FilterExpression filterExpression = new FilterExpression("Region IS NOT NULL AND price IS NOT NULL AND Region = 'Hangzhou' AND price > 30", FilterExpressionType.SQL92);
  simpleConsumer.subscribe(topic, filterExpression);
  ```

- 订阅消息，匹配Topic中的所有消息，不进行过滤。

  ```java
  String topic = "topic";
  //订阅所有消息。
  FilterExpression filterExpression = new FilterExpression("True", FilterExpressionType.SQL92);
  simpleConsumer.subscribe(topic, filterExpression);
  ```

### 4.7.6 使用建议

**合理划分主题和Tag标签**

从消息的过滤机制和主题的原理机制可以看出，业务消息的拆分可以基于主题进行筛选，也可以基于主题内消息的Tag标签及属性进行筛选。关于拆分方式的选择，应遵循以下原则：

- 消息类型是否一致：不同类型的消息，如顺序消息和普通消息需要使用不同的主题进行拆分，无法通过Tag标签进行分类。
- 业务域是否相同：不同业务域和部门的消息应该拆分不同的主题。例如物流消息和支付消息应该使用两个不同的主题；同样是一个主题内的物流消息，普通物流消息和加急物流消息则可以通过不同的Tag进行区分。
- 消息量级和重要性是否一致：如果消息的量级规模存在巨大差异，或者说消息的链路重要程度存在差异，则应该使用不同的主题进行隔离拆分。

## 4.8 消费者负载均衡

消费者从 Apache RocketMQ 获取消息消费时，通过消费者负载均衡策略，可将主题内的消息分配给指定消费者分组中的多个消费者共同分担，提高消费并发能力和消费者的水平扩展能力。本文介绍 Apache RocketMQ 消费者的负载均衡策略。

### 4.8.1 背景信息

了解消费者负载均衡策略，可以帮助您解决以下问题：

- 消息消费处理的容灾策略：您可以根据消费者负载均衡策略，明确当局部节点出现故障时，消息如何进行消费重试和容灾切换。
- 消息消费的顺序性机制：通过消费者负载均衡策略，您可以进一步了解消息消费时，如何保证同一消息组内消息的先后顺序。
- 消息分配的水平拆分策略：了解消费者负载均衡策略，您可以明确消息消费压力如何被分配到不同节点，有针对性地进行流量迁移和水平扩缩容。

### 4.8.2 广播消费和共享消费

在 Apache RocketMQ 领域模型中，同一条消息支持被多个消费者分组订阅，同时，对于每个消费者分组可以初始化多个消费者。您可以根据消费者分组和消费者的不同组合，实现以下两种不同的消费效果：![消费方式](https://rocketmq.apache.org/zh/assets/images/consumemode-74d53c59b3266f1f633b1392f5a0f279.png)

- **消费组间广播消费** ：如上图所示，每个消费者分组只初始化唯一一个消费者，每个消费者可消费到消费者分组内所有的消息，各消费者分组都订阅相同的消息，以此实现单客户端级别的广播一对多推送效果。

  该方式一般可用于网关推送、配置推送等场景。

- **消费组内共享消费** ：如上图所示，每个消费者分组下初始化了多个消费者，这些消费者共同分担消费者分组内的所有消息，实现消费者分组内流量的水平拆分和均衡负载。

  该方式一般可用于微服务解耦场景。

### 4.8.3 什么是消费者负载均衡

如上文所述，消费组间广播消费场景下，每个消费者分组内只有一个消费者，因此不涉及消费者的负载均衡。

消费组内共享消费场景下，消费者分组内多个消费者共同分担消息，消息按照哪种逻辑分配给哪个消费者，就是由消费者负载均衡策略所决定的。

根据消费者类型的不同，消费者负载均衡策略分为以下两种模式：

- [消息粒度负载均衡](https://rocketmq.apache.org/zh/docs/featureBehavior/08consumerloadbalance#section-x2b-2cu-gpf)：PushConsumer和SimpleConsumer默认负载策略
- [队列粒度负载均衡](https://rocketmq.apache.org/zh/docs/featureBehavior/08consumerloadbalance#section-n9m-6xy-y77)：PullConsumer默认负载策略

> 类比Kafka的话，kafka对应消费者类型即只有PullConsumer，并且负载均衡策略即partition粒度。

### 4.8.4 消息粒度负载均衡

**使用范围**

对于PushConsumer和SimpleConsumer类型的消费者，默认且仅使用消息粒度负载均衡策略。

> 上述说明是指5.0 SDK下，PushConsumer默认使用消息粒度负载均衡，对于3.x/4.x等Remoting协议SDK 仍然使用了队列粒度负载均衡。业务集成如无特殊需求，建议使用新版本机制。

**策略原理**

消息粒度负载均衡策略中，同一消费者分组内的多个消费者将按照消息粒度平均分摊主题中的所有消息，即同一个队列中的消息，可被平均分配给多个消费者共同消费。 ![消息粒度负载](https://rocketmq.apache.org/zh/assets/images/clustermode-dfd781d08bc0c69111841bda537aa302.png)

如上图所示，消费者分组Group A中有三个消费者A1、A2和A3，这三个消费者将共同消费主题中同一队列Queue1中的多条消息。 **注意** 消息粒度负载均衡策略保证同一个队列的消息可以被多个消费者共同处理，但是该策略使用的消息分配算法结果是随机的，并不能指定消息被哪一个特定的消费者处理。

<u>消息粒度的负载均衡机制，是**基于内部的单条消息确认语义实现**的。消费者获取某条消息后，**服务端会将该消息加锁**，保证这条消息对其他消费者不可见，直到该消息消费成功或消费超时</u>。因此，即使多个消费者同时消费同一队列的消息，服务端也可保证消息不会被多个消费者重复消费。

**顺序消息负载机制**

在顺序消息中，消息的顺序性指的是同一消息组内的多个消息之间的先后顺序。因此，顺序消息场景下，消息粒度负载均衡策略还需要保证同一消息组内的消息，按照服务端存储的先后顺序进行消费。**不同消费者处理同一个消息组内的消息时，会严格按照先后顺序锁定消息状态，确保同一消息组的消息串行消费**。 ![顺序消息负载策略](https://rocketmq.apache.org/zh/assets/images/fifoinclustermode-60b2f917ab49333f93029cee178b13f0.png)

如上图所述，队列Queue1中有4条顺序消息，这4条消息属于同一消息组G1，存储顺序由M1到M4。在消费过程中，前面的消息M1、M2被消费者Consumer A1处理时，只要消费状态没有提交，消费者A2是无法并行消费后续的M3、M4消息的，必须等前面的消息提交消费状态后才能消费后面的消息。

**策略特点**

相对于队列粒度负载均衡策略，消息粒度负载均衡策略有以下特点：

- 消费分摊更均衡：对于传统队列级的负载均衡策略，如果队列数量和消费者数量不均衡，则可能会出现部分消费者空闲，或部分消费者处理过多消息的情况。消息粒度负载均衡策略无需关注消费者和队列的相对数量，能够更均匀地分摊消息。
- 对非对等消费者更友好：在线上生产环境中，由于网络机房分区延迟、消费者物理资源规格不一致等原因，消费者的处理能力可能会不一致，如果按照队列分配消息，则可能出现部分消费者消息堆积、部分消费者空闲的情况。消息粒度负载均衡策略按需分配，消费者处理任务更均衡。
- 队列分配运维更方便：**传统基于绑定队列的负载均衡策略必须保证队列数量大于等于消费者数量，以免产生部分消费者获取不到队列出现空转的情况，而消息粒度负载均衡策略则无需关注队列数**。

> kafka只能基于partition负载均衡，当消费者数量>分区数量时，就注定会有消费者空闲。

**适用场景**

消息粒度消费负载均衡策略下，同一队列内的消息离散地分布于多个消费者，适用于绝大多数在线事件处理的场景。只需要基本的消息处理能力，对消息之间没有批量聚合的诉求。而**对于流式处理、聚合计算场景，需要明确地对消息进行聚合、批处理时，更适合使用队列粒度的负载均衡策略**。

**使用示例**

消息粒度负载均衡策略不需要额外设置，对于PushConsumer和SimpleConsumer消费者类型默认启用。

```java
SimpleConsumer simpleConsumer = null;
        //消费示例一：使用PushConsumer消费普通消息，只需要在消费监听器处理即可，无需关注消息负载均衡。
        MessageListener messageListener = new MessageListener() {
            @Override
            public ConsumeResult consume(MessageView messageView) {
                System.out.println(messageView);
                //根据消费结果返回状态。
                return ConsumeResult.SUCCESS;
            }
        };
        //消费示例二：使用SimpleConsumer消费普通消息，主动获取消息处理并提交。会按照订阅的主题自动获取，无需关注消息负载均衡。
        List<MessageView> messageViewList = null;
        try {
            messageViewList = simpleConsumer.receive(10, Duration.ofSeconds(30));
            messageViewList.forEach(messageView -> {
                System.out.println(messageView);
                //消费处理完成后，需要主动调用ACK提交消费结果。
                try {
                    simpleConsumer.ack(messageView);
                } catch (ClientException e) {
                    e.printStackTrace();
                }
            });
        } catch (ClientException e) {
            //如果遇到系统流控等原因造成拉取失败，需要重新发起获取消息请求。
            e.printStackTrace();
        }
```

### 4.8.5 队列粒度负载均衡

**使用范围**

对于历史版本（服务端4.x/3.x版本）的消费者，包括PullConsumer、DefaultPushConsumer、DefaultPullConsumer、LitePullConsumer等，默认且仅能使用队列粒度负载均衡策略。

**策略原理**

队列粒度负载均衡策略中，同一消费者分组内的多个消费者将按照队列粒度消费消息，即每个队列仅被一个消费者消费。 ![队列级负载均衡原理](https://rocketmq.apache.org/zh/assets/images/clusterqueuemode-ce4f88dc594c1237ba95db2fa9146b8c.png)

如上图所示，主题中的三个队列Queue1、Queue2、Queue3被分配给消费者分组中的两个消费者，每个队列只能分配给一个消费者消费，该示例中由于队列数大于消费者数，因此，消费者A2被分配了两个队列。若队列数小于消费者数量，可能会出现部分消费者无绑定队列的情况。

队列粒度的负载均衡，基于队列数量、消费者数量等运行数据进行统一的算法分配，将每个队列绑定到特定的消费者，然后每个消费者按照取消息>提交消费位点>持久化消费位点的消费语义处理消息，**取消息过程不提交消费状态，因此，为了避免消息被多个消费者重复消费，每个队列仅支持被一个消费者消费**。

> 队列粒度负载均衡策略保证同一个队列仅被一个消费者处理，该策略的实现依赖消费者和服务端的信息协商机制，**Apache RocketMQ 并不能保证协商结果完全强一致**。
>
> 因此，<u>**在消费者数量、队列数量发生变化时，可能会出现短暂的队列分配结果不一致，从而导致少量消息被重复处理**</u>。

**策略特点**

相对于消息粒度负载均衡策略，队列粒度负载均衡策略分配粒度较大，不够灵活。但**该策略在流式处理场景下有天然优势，能够保证同一队列的消息被相同的消费者处理，对于批量处理、聚合处理更友好**。

**适用场景**

<u>队列粒度负载均衡策略适用于流式计算、数据聚合等需要明确对消息进行聚合、批处理的场景</u>。

**使用示例**

队列粒度负载均衡策略不需要额外设置，对于历史版本（服务端4.x/3.x版本）的消费者类型PullConsumer默认启用。

具体示例代码，请访问[RocketMQ代码库](https://github.com/apache/rocketmq/blob/develop/example/src/main/java/org/apache/rocketmq/example/simple/LitePullConsumerAssign.java)获取。

### 4.8.6 版本兼容性

消息粒度的负载均衡策略从 Apache RocketMQ 服务端5.0版本开始支持，历史版本4.x/3.x版本仅支持队列粒度的负载均衡策略。

当您使用的 Apache RocketMQ 服务端版本为5.x版本时，两种消费者负载均衡策略均支持，具体生效的负载均衡策略依客户端版本和消费者类型而定。

### 4.8.7 使用建议

**针对消费逻辑做消息幂等**

<u>无论是消息粒度负载均衡策略还是队列粒度负载均衡策略，在消费者上线或下线、服务端扩缩容等场景下，都会触发短暂的重新负载均衡动作。此时可能会存在短暂的负载不一致情况，出现少量消息重复的现象。因此，需要在下游消费逻辑中做好消息幂等去重处理</u>。

## 4.9 消费者进度管理

Apache RocketMQ 通过消费位点管理消费进度，本文为您介绍 Apache RocketMQ 的消费进度管理机制。

### 4.9.1 背景信息

Apache RocketMQ 的生产者和消费者在进行消息收发时，必然会涉及以下场景，消息先生产后订阅或先订阅后生产。这两种场景下，消费者客户端启动后从哪里开始消费？如何标记已消费的消息？这些都是由 Apache RocketMQ 的消费进度管理机制来定义的。

通过了解 Apache RocketMQ 的消费进度管理机制，可以帮助您解答以下问题：

- 消费者启动后从哪里开始消费消息？
- 消费者每次消费成功后如何标记消息状态，确保下次不会再重复处理该消息？
- 某消息被指定消费者消费过一次后，如果业务出现异常需要做故障恢复，该消息能否被重新消费？

### 4.9.2 消费进度原理

**消息位点（Offset）**

参考 Apache RocketMQ [主题](https://rocketmq.apache.org/zh/docs/domainModel/02topic)和[队列](https://rocketmq.apache.org/zh/docs/domainModel/03messagequeue)的定义，消息是按到达服务端的先后顺序存储在指定主题的多个队列中，**每条消息在队列中都有一个唯一的Long类型坐标，这个坐标被定义为消息位点**。

任意一个消息队列在逻辑上都是无限存储，即消息位点会从0到Long.MAX无限增加。通过主题、队列和位点就可以定位任意一条消息的位置，具体关系如下图所示：![消息位点](https://rocketmq.apache.org/zh/assets/images/consumerprogress-da5f38e59a7fcb4ff40325b0f7fbf8a3.png)

Apache RocketMQ 定义队列中最早一条消息的位点为最小消息位点（MinOffset）；最新一条消息的位点为最大消息位点（MaxOffset）。**虽然消息队列逻辑上是无限存储，但由于服务端物理节点的存储空间有限， Apache RocketMQ 会滚动删除队列中存储最早的消息。因此，消息的最小消费位点和最大消费位点会一直<u>递增变化</u>**。![消费位点更新](https://rocketmq.apache.org/zh/assets/images/updateprogress-02d1a9de72aa4f72c3b1e1c6e03d2407.png)

**消费位点（ConsumerOffset）**

Apache RocketMQ 领域模型为发布订阅模式，每个主题的队列都可以被多个消费者分组订阅。若某条消息被某个消费者消费后直接被删除，则其他订阅了该主题的消费者将无法消费该消息。

因此，Apache RocketMQ 通过消费位点管理消息的消费进度。<u>每条消息被某个消费者消费完成后不会立即在队列中删除，Apache RocketMQ 会基于每个消费者分组维护一份消费记录，该记录指定消费者分组消费某一个队列时，消费过的最新一条消息的位点，即消费位点</u>。

当消费者客户端离线，又再次重新上线时，会严格按照服务端保存的消费进度继续处理消息。**如果服务端保存的历史位点信息已过期被删除，此时消费位点向前移动至服务端存储的最小位点**。

> **消费位点的保存和恢复是基于 Apache RocketMQ 服务端的存储实现，<u>和任何消费者无关</u>。因此 Apache RocketMQ 支持跨消费者的消费进度恢复**。

队列中消息位点MinOffset、MaxOffset和每个消费者分组的消费位点ConsumerOffset的关系如下：![消费进度](https://rocketmq.apache.org/zh/assets/images/consumerprogress1-07d9f77dd7e62f2250330ed36f36fe3c.png)

- ConsumerOffset≤MaxOffset：
  - 当消费速度和生产速度一致，且全部消息都处理完成时，最大消息位点和消费位点相同，即ConsumerOffset=MaxOffset。
  - 当消费速度较慢小于生产速度时，队列中会有部分消息未消费，此时消费位点小于最大消息位点，即ConsumerOffset<MaxOffset，两者之差就是该队列中堆积的消息量。
- ConsumerOffset≥MinOffset：正常情况下有效的消费位点ConsumerOffset必然大于等于最小消息位点MinOffset。消费位点小于最小消息位点时是无效的，相当于消费者要消费的消息已经从队列中删除了，是无法消费到的，此时服务端会将消费位点强制纠正到合法的消息位点。

**消费位点初始值**

<u>消费位点初始值指的是消费者分组首次启动消费者消费消息时，服务端保存的消费位点的初始值</u>。

**Apache RocketMQ 定义消费位点的初始值为消费者首次获取消息时，该时刻队列中的最大消息位点**。相当于消费者将从队列中最新的消息开始消费。

### 4.9.3 重置消费位点

若消费者分组的初始消费位点或当前消费位点不符合您的业务预期，您可以通过重置消费位点调整您的消费进度。

**适用场景**

- 初始消费位点不符合需求：因初始消费位点为当前队列的最大消息位点，即客户端会直接从最新消息开始消费。若业务上线时需要消费部分历史消息，您可以通过重置消费位点功能消费到指定时刻前的消息。
- 消费堆积快速清理：当下游消费系统性能不足或消费速度小于生产速度时，会产生大量堆积消息。若这部分堆积消息可以丢弃，您可以通过重置消费位点快速将消费位点更新到指定位置，绕过这部分堆积的消息，减少下游处理压力。
- 业务回溯，纠正处理：由于业务消费逻辑出现异常，消息被错误处理。若您希望重新消费这些已被处理的消息，可以通过重置消费位点快速将消费位点更新到历史指定位置，实现消费回溯。

**重置功能**

Apache RocketMQ 的重置消费位点提供以下能力：

- 重置到队列中的指定位点。
- <u>重置到某一时刻对应的消费位点，匹配位点时，服务端会根据自动匹配到该时刻最接近的消费位点</u>。

**使用限制**

- <u>重置消费位点后消费者将直接从重置后的位点开始消费，对于回溯重置类场景，重置后的历史消息大多属于存储冷数据，可能会造成系统压力上升，一般称为冷读现象</u>。因此，需要谨慎评估重置消费位点后的影响。建议严格控制重置消费位点接口的调用权限，避免无意义、高频次的消费位点重置。
- Apache RocketMQ 重置消费位点功能只能重置对消费者可见的消息，不能重置定时中、重试等待中的消息。更多信息，请参见[定时/延时消息](https://rocketmq.apache.org/zh/docs/featureBehavior/02delaymessage)和[消费重试](https://rocketmq.apache.org/zh/docs/featureBehavior/10consumerretrypolicy)。

### 4.9.4 版本兼容性

关于消费者分组的消费位点初始值，不同的服务端版本中定义如下：

- 服务端历史版本（4.x/3.x版本）：消息位点初始值受当前队列消息状态的影响。
- 服务端5.x版本：明确定义消费位点初始值为消费者获取消息时刻队列中的最大消息位点。

因此，若您将服务端版本从历史版本升级到最新的5.x版本时，需要自行对消费者首次启动时的情况做兼容性判断。

### 4.9.5 使用建议

**严格控制消费位点重置的权限**

重置消费位点会给系统带来额外处理压力，可能会影响新消息的读写性能。 因此该操作请在适用场景下谨慎执行，并提前做好合理性和必要性评估。

## 4.10 消费重试

消费者出现异常，消费某条消息失败时， Apache RocketMQ 会根据消费重试策略重新投递该消息进行故障恢复。本文介绍消费重试机制的原理、版本兼容性和使用建议。

### 4.10.1 应用场景

Apache RocketMQ 的消费重试主要解决的是业务处理逻辑失败导致的消费完整性问题，是一种为业务兜底的策略，**不应该被用做业务流程控制**。建议以下消费失败场景使用重试机制：

推荐使用消息重试场景如下：

- 业务处理失败，且失败原因跟当前的消息内容相关，比如该消息对应的事务状态还未获取到，预期一段时间后可执行成功。
- 消费失败的原因不会导致连续性，即当前消息消费失败是一个小概率事件，不是常态化的失败，后面的消息大概率会消费成功。此时可以对当前消息进行重试，避免进程阻塞。

典型错误使用场景如下：

- 消费处理逻辑中使用消费失败来做条件判断的结果分流，是不合理的，因为处理逻辑已经预见了一定会大量出现该判断分支。
- 消费处理中使用消费失败来做处理速率限流，是不合理的。限流的目的是将超出流量的消息暂时堆积在队列中达到削峰的作用，而不是让消息进入重试链路。

### 4.10.2 应用目的

消息中间件做异步解耦时的一个典型问题是如果下游服务处理消息事件失败，如何保证整个调用链路的完整性。Apache RocketMQ 作为金融级的可靠业务消息中间件，在消息投递处理机制的设计上天然支持可靠传输策略，通过完整的确认和重试机制保证每条消息都按照业务的预期被处理。

了解 Apache RocketMQ 的消息确认机制以及消费重试策略可以帮助您分析如下问题：

- 如何保证业务完整处理消息：了解消费重试策略，可以在设计实现消费者逻辑时保证每条消息处理的完整性，避免部分消息出现异常时被忽略，导致业务状态不一致。
- 系统异常时处理中的消息状态如何恢复：帮助您了解当系统出现异常（宕机故障）等场景时，处理中的消息状态如何恢复，是否会出现状态不一致。

### 4.10.3 消费重试策略概述

消费重试指的是，消费者在消费某条消息失败后，Apache RocketMQ 服务端会根据重试策略重新消费该消息，<u>超过一定次数后若还未消费成功，则该消息将不再继续重试，直接被发送到**死信队列**中</u>。

**消息重试的触发条件**

- 消费失败，包括消费者返回消息失败状态标识或抛出非预期异常。
- <u>消息处理超时</u>，包括在PushConsumer中排队超时。

**消息重试策略主要行为**

- 重试过程状态机：控制消息在重试流程中的状态和变化逻辑。
- 重试间隔：上一次消费失败或超时后，下次重新尝试消费的间隔时间。
- 最大重试次数：消息可被重试消费的最大次数。

**消息重试策略差异**

根据消费者类型不同，消息重试策略的具体内部机制和设置方法有所不同，具体差异如下：

| 消费者类型     | 重试过程状态机                       | 重试间隔                                                     | 最大重试次数                   |
| -------------- | ------------------------------------ | ------------------------------------------------------------ | ------------------------------ |
| PushConsumer   | *已就绪* 处理中 *待重试* 提交 * 死信 | 消费者分组创建时元数据控制。 *无序消息：阶梯间隔* 顺序消息：固定间隔时间 | 消费者分组创建时的元数据控制。 |
| SimpleConsumer | *已就绪* 处理中 *提交* 死信          | 通过API修改获取消息时的不可见时间。                          | 消费者分组创建时的元数据控制。 |

具体的重试策略，请参见下文[PushConsumer消费重试策略](https://rocketmq.apache.org/zh/docs/featureBehavior/10consumerretrypolicy#section-qqo-bil-rc6)和[SimpleConsumer消费重试策略](https://rocketmq.apache.org/zh/docs/featureBehavior/10consumerretrypolicy#section-my2-2au-7gl)。

### 4.10.4 PushConsumer消费重试策略

**重试状态机**

PushConsumer消费消息时，消息的几个主要状态如下：![Push消费状态机](https://rocketmq.apache.org/zh/assets/images/retrymachinestatus-37ddbd0a20b8736e34bb88f565945d16.png)

- Ready：已就绪状态。消息在Apache RocketMQ服务端已就绪，可以被消费者消费。
- Inflight：处理中状态。消息被消费者客户端获取，处于消费中还未返回消费结果的状态。
- **WaitingRetry：待重试状态，PushConsumer独有的状态**。当消费者消息处理失败或消费超时，会触发消费重试逻辑判断。如果当前重试次数未达到最大次数，则该消息变为待重试状态，经过重试间隔后，消息将重新变为已就绪状态可被重新消费。多次重试之间，可通过重试间隔进行延长，防止无效高频的失败。
- Commit：提交状态。消费成功的状态，消费者返回成功响应即可结束消息的状态机。
- **DLQ：死信状态**。<u>消费逻辑的最终兜底机制，若消息一直处理失败并不断进行重试，直到超过最大重试次数还未成功，此时消息不会再重试，会被投递至死信队列。您可以通过消费死信队列的消息进行业务恢复</u>。

消息重试过程中，每次重试消息状态都会经过已就绪>处理中>待重试的变化，两次消费间的间隔时间实际由消费耗时及重试间隔控制，消费耗时的最大上限受服务端系统参数控制，一般不应该超过上限时间。![消息间隔时间](https://rocketmq.apache.org/zh/assets/images/retrytimeline-27247ef53fbcf08c745b9f7d356de6f9.png)

**最大重试次数**

PushConsumer的最大重试次数由消费者分组创建时的元数据控制，具体参数，请参见[消费者分组](https://rocketmq.apache.org/zh/docs/domainModel/07consumergroup)。

例如，最大重试次数为3次，则该消息最多可被投递4次，1次为原始消息，3次为重试投递次数。

**重试间隔时间**

- 无序消息（非顺序消息）：重试间隔为阶梯时间，具体时间如下：

  | 第几次重试 | 与上次重试的间隔时间 | 第几次重试 | 与上次重试的间隔时间 |
  | ---------- | -------------------- | ---------- | -------------------- |
  | 1          | 10秒                 | 9          | 7分钟                |
  | 2          | 30秒                 | 10         | 8分钟                |
  | 3          | 1分钟                | 11         | 9分钟                |
  | 4          | 2分钟                | 12         | 10分钟               |
  | 5          | 3分钟                | 13         | 20分钟               |
  | 6          | 4分钟                | 14         | 30分钟               |
  | 7          | 5分钟                | 15         | 1小时                |
  | 8          | 6分钟                | 16         | 2小时                |

> 若重试次数超过16次，后面每次重试间隔都为2小时。

- 顺序消息：重试间隔为固定时间，具体取值，请参见[参数限制](https://rocketmq.apache.org/zh/docs/introduction/03limits)。

**使用示例**

**PushConsumer触发消息重试只需要返回消费失败的状态码即可，当出现非预期的异常时，也会被SDK捕获**。

```java
SimpleConsumer simpleConsumer = null;
        //消费示例：使用PushConsumer消费普通消息，如果消费失败返回错误，即可触发重试。
        MessageListener messageListener = new MessageListener() {
            @Override
            public ConsumeResult consume(MessageView messageView) {
                System.out.println(messageView);
                //返回消费失败，会自动重试，直至到达最大重试次数。
                return ConsumeResult.FAILURE;
            }
        };
```

### 4.10.5 SimpleConsumer消费重试策略

**重试状态机**

SimpleConsumer消费消息时，消息的几个主要状态如下：![SimpleConsumer状态机](https://rocketmq.apache.org/zh/assets/images/simplemachinestatus-1844bd0115b315e32661cf20b1732db0.png)

- Ready：已就绪状态。消息在Apache RocketMQ服务端已就绪，可以被消费者消费。
- Inflight：处理中状态。消息被消费者客户端获取，处于消费中还未返回消费结果的状态。
- Commit：提交状态。消费成功的状态，消费者返回成功响应即可结束消息的状态机。
- **DLQ：死信状态**。消费逻辑的最终兜底机制，若消息一直处理失败并不断进行重试，直到超过最大重试次数还未成功，此时消息不会再重试，会被投递至死信队列。您可以通过消费死信队列的消息进行业务恢复。

<u>和PushConsumer消费重试策略不同的是，SimpleConsumer消费者的重试间隔是预分配的，每次获取消息消费者会在调用API时设置一个不可见时间参数 InvisibleDuration，即消息的最大处理时长</u>。若消息消费失败触发重试，不需要设置下一次重试的时间间隔，直接复用不可见时间参数的取值。 ![simpleconsumer重试](https://rocketmq.apache.org/zh/assets/images/simpletimeline-130218b5dca33422638d2ee6409a8330.png)

由于不可见时间为预分配的，可能和实际业务中的消息处理时间差别较大，您可以通过API接口修改不可见时间。

例如，您预设消息处理耗时最多20 ms，但实际业务中20 ms内消息处理不完，您可以修改消息不可见时间，延长消息处理时间，避免消息触发重试机制。

修改消息不可见时间需要满足以下条件：

- 消息处理未超时
- 消息处理未提交消费状态

如下图所示，**<u>消息不可见时间修改后立即生效</u>，即从调用API时刻开始，重新计算消息不可见时间**。 ![修改不可见时间](https://rocketmq.apache.org/zh/assets/images/changeInvisibletime-769fd45237e26f2ff333ee1149e66d47.png)

**最大重试次数**

**SimpleConsumer的最大重试次数由<u>消费者分组</u>创建时的元数据控制**，具体参数，请参见[消费者分组](https://rocketmq.apache.org/zh/docs/domainModel/07consumergroup)。

**消息重试间隔**

**消息重试间隔=不可见时间－消息实际处理时长**

<u>SimpleConsumer 的消费重试间隔通过消息的不可见时间控制</u>。例如，消息不可见时间为30 ms，实际消息处理用了10 ms就返回失败响应，则距下次消息重试还需要20 ms，此时的消息重试间隔即为20 ms；若直到30 ms消息还未处理完成且未返回结果，则消息超时，立即重试，此时重试间隔即为0 ms。

**使用示例**

SimpleConsumer 触发消息重试只需要等待即可。

```java
 //消费示例：使用SimpleConsumer消费普通消息，如果希望重试，只需要静默等待超时即可，服务端会自动重试。
        List<MessageView> messageViewList = null;
        try {
            messageViewList = simpleConsumer.receive(10, Duration.ofSeconds(30));
            messageViewList.forEach(messageView -> {
                System.out.println(messageView);
                //如果处理失败，希望服务端重试，只需要忽略即可，等待消息再次可见后即可重试获取。
            });
        } catch (ClientException e) {
            //如果遇到系统流控等原因造成拉取失败，需要重新发起获取消息请求。
            e.printStackTrace();
        }
```

### 4.10.6 使用建议

**合理重试，避免因限流等诉求触发消费重试**

上文[应用场景](https://rocketmq.apache.org/zh/docs/featureBehavior/10consumerretrypolicy#section-d2i-0sk-rtf)中提到，消息重试适用业务处理失败且当前消费为小概率事件的场景，**不适合在连续性失败的场景下使用，例如消费限流场景**。

- 错误示例：如果当前消费速度过高触发限流，则返回消费失败，等待下次重新消费。
- 正确示例：<u>如果当前消费速度过高触发限流，则延迟获取消息，稍后再消费</u>。

**合理控制重试次数，避免无限重试**

虽然Apache RocketMQ支持自定义消费重试次数，但是**建议通过减少重试次数+延长重试间隔来降低系统压力**，避免出现无限重试或大量重试的情况。

## 4.11 消息存储和清理机制

本文为您介绍 Apache RocketMQ 中消息的存储机制，包括消息的存储粒度、判断依据及后续处理策略等。

### 4.11.1 背景信息

参考 Apache RocketMQ 中[队列](https://rocketmq.apache.org/zh/docs/domainModel/03messagequeue)的定义，<u>消息按照达到服务器的先后顺序被存储到队列中</u>，理论上每个队列都支持无限存储。

但是在实际部署场景中，服务端节点的物理存储空间有限，消息无法做到永久存储。因此，在实际使用中需要考虑以下问题，消息在服务端中的存储以什么维度为判定条件？消息存储以什么粒度进行管理？消息存储超过限制后如何处理？这些问题都是由消息存储和过期清理机制来定义的。

了解消息存储和过期清理机制，可以从以下方面帮助您更好的进行运维管理：

- 提供消息存储时间SLA，为业务提供安全冗余空间：消息存储时间的承诺本质上代表业务侧可以自由获取消息的时间范围。对于消费时间长、消息堆积、故障恢复等场景非常关键。
- 评估和控制存储成本：Apache RocketMQ 消息一般存储于磁盘介质上，您可以通过存储机制评估消息存储空间，提前预留存储资源。

### 4.11.2 消息存储机制

**原理机制**

Apache RocketMQ **使用存储时长作为消息存储的依据**，即每个节点对外承诺消息的存储时长。在存储时长范围内的消息都会被保留，无论消息是否被消费；超过时长限制的消息则会被清理掉。

消息存储机制主要定义以下关键问题：

- 消息存储管理粒度：Apache RocketMQ 按存储节点管理消息的存储时长，并不是按照主题或队列粒度来管理。
- 消息存储判断依据：消息存储按照存储时间作为判断依据，相对于消息数量、消息大小等条件，使用存储时间作为判断依据，更利于业务方对消息数据的价值进行评估。
- 消息存储和是否消费状态无关：Apache RocketMQ 的消息存储是按照消息的生产时间计算，和消息是否被消费无关。按照统一的计算策略可以有效地简化存储机制。

消息在队列中的存储情况如下：![消息存储](https://rocketmq.apache.org/zh/assets/images/cleanpolicy-aa812156263be0605a22b9348ebdc22c.png)

> **消息存储管理粒度说明**
>
> Apache RocketMQ 按照服务端节点粒度管理存储时长而非队列或主题，原因如下：
>
> - 消息存储优势权衡：Apache RocketMQ 基于统一的物理日志队列和轻量化逻辑队列的二级组织方式，管理物理数据。这种机制可以带来顺序读写、高吞吐、高性能等优势，但缺点是不支持按主题和队列单独管理。
> - 安全生产和容量保障风险要求：即使Apache RocketMQ 按照主题或者队列独立生成存储文件，但存储层本质还是共享存储介质。**单独根据主题或队列控制存储时长，这种方式看似更灵活，但实际上整个集群仍然存在容量风险，可能会导致存储时长SLA被打破**。<u>从安全生产角度考虑，最合理的方式是将不同存储时长的消息通过不同集群进行分离治理</u>。

**消息存储和消费状态关系说明**

Apache RocketMQ 统一管理消息的存储时长，无论消息是否被消费。

当消费者不在线或消息消费异常时，会造成队列中大量消息堆积，且该现象暂时无法有效控制。若此时按照消费状态考虑将未消费的消息全部保留，则很容易导致存储空间不足，进而影响到新消息的读写速度。

根据统一地存储时长管理消息，可以帮助消费者业务清晰地判断每条消息的生命周期。只要消息在有效期内可以随时被消费，或通过[重置消费位点](https://rocketmq.apache.org/zh/docs/featureBehavior/09consumerprogress)功能使消息可被消费多次。

**消息存储文件结构说明** Apache RocketMQ 消息默认存储在本地磁盘文件中，存储文件的根目录由配置参数 storePathRootDir 决定，存储结构如下图所示，<u>其中 commitlog 文件夹存储消息物理文件，consumeCQueue文件夹存储逻辑队列索引</u>，其他文件的详细作用可以参考代码解析。 ![消息存储](https://rocketmq.apache.org/zh/assets/images/store-2eb2d519dd4030480ca3ea63f2dc1b70.jpg)

### 4.11.3 消息过期清理机制

在 Apache RocketMQ中，消息保存时长并不能完整控制消息的实际保存时间，因为消息存储仍然使用本地磁盘，**本地磁盘空间不足时，为保证服务稳定性消息仍然会被强制清理，导致消息的实际保存时长小于设置的保存时长**。

### 4.11.4 使用建议

**消息存储时长建议适当增加**

Apache RocketMQ 按存储时长统一控制消息是否保留。建议在存储成本可控的前提下，尽可能延长消息存储时长。延长消息存储时长，可以为紧急故障恢复、应急问题排查和消息回溯带来更多的可操作空间。

# 5. 部署 & 运维

# 6. 可观测

# 7. 客户端SDK

# 8. 最佳实践

# 9. RocketMQ EventBridge

# 10. RocketMQ MQTT

# 11. RocketMQ Streams

# X. 其他

