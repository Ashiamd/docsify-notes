# Sentinel官方文档-学习笔记

> [Sentinel中文官方文档](https://sentinelguard.io/zh-cn/docs/introduction.html) <= 下面内容基本都摘自官方文档。
>
> [Sentinel 微服务保护 - 安浩阳 - 博客园 (cnblogs.com)](https://www.cnblogs.com/anhaoyang/p/sentinel-microservice-protection-1kmwvn.html)

# 1. Sentinel介绍

## 1.1 基本概念

+ 资源

  具体的进程，某个HTTP接口，某段代码逻辑，等。只要通过 Sentinel API 定义的代码，就是资源，能够被 Sentinel 保护起来。大部分情况下，可以使用方法签名，URL，甚至服务名称作为资源名来标示资源。

+ 规则

  围绕资源的实时状态设定的规则，可以包括流量控制规则、熔断降级规则以及系统保护规则。所有规则可以动态实时调整。

## 1.2 功能和设计理念

### 1.2.1 流量控制

![image-20210715173555158](https://i-blog.csdnimg.cn/blog_migrate/a7f9c42ea00bb04a0a77ee9e1b1a698a.png)

多角度进行流量控制：

- 资源的调用关系，例如资源的调用链路，资源和资源之间的关系；
- 运行指标，例如 QPS、线程池、系统负载等；
- 控制的效果，例如直接限流、冷启动、排队等。

Sentinel 的设计理念是让您自由选择控制的角度，并进行灵活组合，从而达到想要的效果。

### 1.2.2 熔断降级

什么是熔断降级：

除了流量控制以外，降低调用链路中的不稳定资源也是 Sentinel 的使命之一。由于调用关系的复杂性，如果调用链路中的某个资源出现了不稳定，最终会导致请求发生堆积。这个问题和 [Hystrix](https://github.com/Netflix/Hystrix/wiki#what-problem-does-hystrix-solve) 里面描述的问题是一样的。

![image](https://user-images.githubusercontent.com/9434884/62410811-cd871680-b61d-11e9-9df7-3ee41c618644.png)

Sentinel 和 Hystrix 的原则是一致的: 当调用链路中某个资源出现不稳定，例如，表现为 timeout，异常比例升高的时候，则对这个资源的调用进行限制，并让请求快速失败，避免影响到其它的资源，最终产生雪崩的效果。

> 熔断降级，Hystrix 通过[线程池](https://github.com/Netflix/Hystrix/wiki/How-it-Works#benefits-of-thread-pools)的方式，来对依赖(在Sentinel概念中对应资源)进行了隔离。好处是资源和资源之间做到了最彻底的隔离。缺点是除了增加了线程切换的成本，还需要预先给各个资源做线程池大小的分配。

Sentinel通过两种手段实现熔断降级：

1. **通过并发线程数进行限制**

   当某个资源出现不稳定的情况下，例如响应时间变长，对资源的直接影响就是会造成线程数的逐步堆积。当线程数在特定资源上堆积到一定的数量之后，对该资源的新请求就会被拒绝。堆积的线程完成任务后才开始继续接收请求。

   （不需要像Hystrix一样预先分配线程池，也没有了线程切换的损耗）

2. **通过响应时间对资源进行降级**

   当依赖的资源出现响应时间过长后，所有对该资源的访问都会被直接拒绝，直到过了指定的时间窗口之后才重新恢复。

### 1.2.3 系统负载保护

Sentinel 同时提供[系统维度的自适应保护能力](https://sentinelguard.io/zh-cn/docs/system-adaptive-protection.html)。防止雪崩，是系统防护中重要的一环。当系统负载较高的时候，如果还持续让请求进入，可能会导致系统崩溃，无法响应。在集群环境下，网络负载均衡会把本应这台机器承载的流量转发到其它的机器上去。如果这个时候其它的机器也处在一个边缘状态的时候，这个增加的流量就会导致这台机器也崩溃，最后导致整个集群不可用。

针对这个情况，Sentinel 提供了对应的保护机制，让系统的入口流量和系统的负载达到一个平衡，保证系统在能力范围之内处理最多的请求。

## 1.3 Sentinel工作机制

Sentinel 的主要工作机制如下：

- 对主流框架提供适配或者显示的 API，来定义需要保护的资源，并提供设施对资源进行实时统计和调用链路分析。
- 根据预设的规则，结合对资源的实时统计信息，对流量进行控制。同时，Sentinel 提供开放的接口，方便您定义及改变规则。
- Sentinel 提供实时的监控系统，方便您快速了解目前系统的状态。

## 1.4 流控降级与容错标准

Sentinel 社区正在将流量治理相关标准抽出到 [OpenSergo spec](https://opensergo.io/zh-cn/) 中，Sentinel 作为流量治理标准实现。有关 Sentinel 流控降级与容错 spec 的最新进展，请参考 [opensergo-specification](https://github.com/opensergo/opensergo-specification/blob/main/specification/zh-Hans/fault-tolerance.md)，也欢迎社区一起来完善标准与实现。

![opensergo-and-sentinel](https://sentinelguard.io/docs/zh-cn/img/opensergo/opensergo-sentinel-fault-tolerance-zh-cn.jpg)

# 2. 入门参考

## 2.1 快速开始

Sentinel 的使用可以分为两个部分:

- 核心库（Java 客户端）：不依赖任何框架/库，能够运行于 Java 8 及以上的版本的运行时环境，同时对 Dubbo / Spring Cloud 等框架也有较好的支持（见 [主流框架适配](https://sentinelguard.io/zh-cn/docs/open-source-framework-integrations.html)）。
- 控制台（Dashboard）：Dashboard 主要负责管理推送规则、监控、管理机器信息等。

### 2.1.1 本地Demo

1. 引入maven依赖：

```xml
<dependency>
  <groupId>com.alibaba.csp</groupId>
  <artifactId>sentinel-core</artifactId>
  <version>1.8.6</version>
</dependency>
```

2. 定义资源

**资源** 是 Sentinel 中的核心概念之一。最常用的资源是我们代码中的 Java 方法。 当然，您也可以更灵活的定义你的资源，例如，把需要控制流量的代码用 Sentinel API `SphU.entry("HelloWorld")` 和 `entry.exit()` 包围起来即可。在下面的例子中，我们将 `System.out.println("hello world");` 作为资源（被保护的逻辑），用 API 包装起来。参考代码如下:

```java
public static void main(String[] args) {
    // 配置规则.
    initFlowRules();

    while (true) {
        // 1.5.0 版本开始可以直接利用 try-with-resources 特性
        try (Entry entry = SphU.entry("HelloWorld")) {
            // 被保护的逻辑
            System.out.println("hello world");
	} catch (BlockException ex) {
            // 处理被流控的逻辑
	    System.out.println("blocked!");
	}
    }
}
```

完成以上两步后，代码端的改造就完成了。

您也可以通过我们提供的 [注解支持模块](https://sentinelguard.io/zh-cn/docs/annotation-support.html)，来定义我们的资源，类似于下面的代码：

```java
@SentinelResource("HelloWorld")
public void helloWorld() {
    // 资源中的逻辑
    System.out.println("hello world");
}
```

这样，`helloWorld()` 方法就成了我们的一个资源。注意注解支持模块需要配合 Spring AOP 或者 AspectJ 一起使用。

3. 定义规则

接下来，通过流控规则来指定允许该资源通过的请求次数，例如下面的代码定义了资源 `HelloWorld` 每秒最多只能通过 20 个请求。

```java
private static void initFlowRules(){
    List<FlowRule> rules = new ArrayList<>();
    FlowRule rule = new FlowRule();
    rule.setResource("HelloWorld");
    rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
    // Set limit QPS to 20.
    rule.setCount(20);
    rules.add(rule);
    FlowRuleManager.loadRules(rules);
}
```

完成上面 3 步，Sentinel 就能够正常工作了。更多的信息可以参考 [使用文档](https://sentinelguard.io/zh-cn/docs/basic-api-resource-rule.html)。

4. 检查效果

Demo 运行之后，我们可以在日志 `~/logs/csp/${appName}-metrics.log.xxx` 里看到下面的输出:

```
|--timestamp-|------date time----|--resource-|p |block|s |e|rt
1529998904000|2018-06-26 15:41:44|hello world|20|0    |20|0|0
1529998905000|2018-06-26 15:41:45|hello world|20|5579 |20|0|728
1529998906000|2018-06-26 15:41:46|hello world|20|15698|20|0|0
1529998907000|2018-06-26 15:41:47|hello world|20|19262|20|0|0
1529998908000|2018-06-26 15:41:48|hello world|20|19502|20|0|0
1529998909000|2018-06-26 15:41:49|hello world|20|18386|20|0|0
```

其中 `p` 代表通过的请求, `block` 代表被阻止的请求, `s` 代表成功执行完成的请求个数, `e` 代表用户自定义的异常, `rt` 代表平均响应时长。

可以看到，这个程序每秒稳定输出 "hello world" 20 次，和规则中预先设定的阈值是一样的。

更详细的说明可以参考: [如何使用](https://sentinelguard.io/zh-cn/docs/basic-api-resource-rule.html)

更多的例子可以参考: [Sentinel Demo 集锦](https://github.com/alibaba/Sentinel/tree/master/sentinel-demo)

5. 启动Sentinel控制台

Sentinel 开源控制台支持实时监控和规则管理。接入控制台的步骤如下：

（1）下载控制台 jar 包并在本地启动：可以参见 [此处文档](https://sentinelguard.io/zh-cn/docs/dashboard.html)。

（2）客户端接入控制台，需要：

- 客户端需要引入 Transport 模块来与 Sentinel 控制台进行通信。您可以通过 `pom.xml` 引入 JAR 包:

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-transport-simple-http</artifactId>
    <version>1.8.6</version>
</dependency>
```

- 启动时加入 JVM 参数 `-Dcsp.sentinel.dashboard.server=consoleIp:port` 指定控制台地址和端口。更多的参数参见 [启动参数文档](https://sentinelguard.io/zh-cn/docs/startup-configuration.html)。
- 确保应用端有访问量

完成以上步骤后即可在 Sentinel 控制台上看到对应的应用，机器列表页面可以看到对应的机器：

![machine-discovery](https://user-images.githubusercontent.com/9434884/50627838-5cd92800-0f70-11e9-891e-31430adcbbf4.png)

详细介绍和使用文档可参考：[Sentinel 控制台文档](https://sentinelguard.io/zh-cn/docs/dashboard.html)。

### 2.1.2 公网Demo

![Remote push rules to config center](https://user-images.githubusercontent.com/9434884/53381986-a0b73f00-39ad-11e9-90cf-b49158ae4b6f.png)

## 2.2 基本原理

在 Sentinel 里面，**所有的资源都对应一个资源名称以及一个 Entry**。Entry 可以通过对主流框架的适配自动创建，也可以通过注解的方式或调用 API 显式创建；**每一个 Entry 创建的时候，同时也会创建一系列功能插槽（slot chain）**。这些插槽有不同的职责，例如:

- `NodeSelectorSlot` 负责收集资源的路径，并将这些资源的调用路径，以树状结构存储起来，用于根据调用路径来限流降级；
- `ClusterBuilderSlot` 则用于存储资源的统计信息以及调用者信息，例如该资源的 RT, QPS, thread count 等等，这些信息将用作为多维度限流，降级的依据；
- `StatisticSlot` 则用于记录、统计不同纬度的 runtime 指标监控信息；
- `FlowSlot` 则用于根据预设的限流规则以及前面 slot 统计的状态，来进行流量控制；
- `AuthoritySlot` 则根据配置的黑白名单和调用来源信息，来做黑白名单控制；
- `DegradeSlot` 则通过统计信息以及预设的规则，来做熔断降级；
- `SystemSlot` 则通过系统的状态，例如 load1 等，来控制总的入口流量；

总体的框架如下：

![img](https://img2020.cnblogs.com/blog/1135670/202109/1135670-20210914210753751-2014209806.png)

Sentinel 将 `ProcessorSlot` 作为 SPI 接口进行扩展（1.7.2 版本以前 `SlotChainBuilder` 作为 SPI），使得 Slot Chain 具备了扩展的能力。您可以自行加入自定义的 slot 并编排 slot 间的顺序，从而可以给 Sentinel 添加自定义的功能。

![Slot Chain SPI](https://user-images.githubusercontent.com/9434884/46783631-93324d00-cd5d-11e8-8ad1-a802bcc8f9c9.png)

下面介绍一下各个 slot 的功能。

### 2.2.1 NodeSelectorSlot

这个 slot 主要负责收集资源的路径，并将这些资源的调用路径，以树状结构存储起来，用于根据调用路径来限流降级。

```java
 ContextUtil.enter("entrance1", "appA");
 Entry nodeA = SphU.entry("nodeA");
 if (nodeA != null) {
    nodeA.exit();
 }
 ContextUtil.exit();
```

上述代码通过 `ContextUtil.enter()` 创建了一个名为 `entrance1` 的上下文，同时指定调用发起者为 `appA`；接着通过 `SphU.entry()`请求一个 token，如果该方法顺利执行没有抛 `BlockException`，表明 token 请求成功。

以上代码将在内存中生成以下结构：

```
 	     machine-root
                 /     
                /
         EntranceNode1
              /
             /   
      DefaultNode(nodeA)
```

注意：每个 `DefaultNode` 由资源 ID 和输入名称来标识。换句话说，一个资源 ID 可以有多个不同入口的 DefaultNode。

```java
  ContextUtil.enter("entrance1", "appA");
  Entry nodeA = SphU.entry("nodeA");
  if (nodeA != null) {
    nodeA.exit();
  }
  ContextUtil.exit();

  ContextUtil.enter("entrance2", "appA");
  nodeA = SphU.entry("nodeA");
  if (nodeA != null) {
    nodeA.exit();
  }
  ContextUtil.exit();
```

以上代码将在内存中生成以下结构：

```
                   machine-root
                   /         \
                  /           \
          EntranceNode1   EntranceNode2
                /               \
               /                 \
       DefaultNode(nodeA)   DefaultNode(nodeA)
```

上面的结构可以通过调用 `curl http://localhost:8719/tree?type=root` 来显示：

```pseudocode
EntranceNode: machine-root(t:0 pq:1 bq:0 tq:1 rt:0 prq:1 1mp:0 1mb:0 1mt:0)
-EntranceNode1: Entrance1(t:0 pq:1 bq:0 tq:1 rt:0 prq:1 1mp:0 1mb:0 1mt:0)
--nodeA(t:0 pq:1 bq:0 tq:1 rt:0 prq:1 1mp:0 1mb:0 1mt:0)
-EntranceNode2: Entrance1(t:0 pq:1 bq:0 tq:1 rt:0 prq:1 1mp:0 1mb:0 1mt:0)
--nodeA(t:0 pq:1 bq:0 tq:1 rt:0 prq:1 1mp:0 1mb:0 1mt:0)

t:threadNum  pq:passQps  bq:blockedQps  tq:totalQps  rt:averageRt  prq: passRequestQps 1mp:1m-passed 1mb:1m-blocked 1mt:1m-total
```

### 2.2.2 ClusterBuilderSlot

此插槽用于构建资源的 `ClusterNode` 以及调用来源节点。`ClusterNode` 保持资源运行统计信息（响应时间、QPS、block 数目、线程数、异常数等）以及原始调用者统计信息列表。来源调用者的名字由 `ContextUtil.enter(contextName，origin)` 中的 `origin` 标记。可通过如下命令查看某个资源不同调用者的访问情况：`curl http://localhost:8719/origin?id=caller`：

```
id: nodeA
idx origin  threadNum passedQps blockedQps totalQps aRt   1m-passed 1m-blocked 1m-total 
1   caller1 0         0         0          0        0     0         0          0        
2   caller2 0         0         0          0        0     0         0          0        
```

### 2.2.3 StatisticSlot

`StatisticSlot` 是 Sentinel 的核心功能插槽之一，用于统计实时的调用数据。

- `clusterNode`：资源唯一标识的 ClusterNode 的 runtime 统计
- `origin`：根据来自不同调用者的统计信息
- `defaultnode`: 根据上下文条目名称和资源 ID 的 runtime 统计
- 入口的统计

**Sentinel 底层采用高性能的滑动窗口数据结构 `LeapArray` 来统计实时的秒级指标数据，可以很好地支撑写多于读的高并发场景。**

![sliding-window-leap-array](https://user-images.githubusercontent.com/9434884/51955215-0af7c500-247e-11e9-8895-9fc0e4c10c8c.png)

### 2.2.4 FlowSlot

这个 slot 主要根据预设的资源的统计信息，按照固定的次序，依次生效。如果一个资源对应两条或者多条流控规则，则会根据如下次序依次检验，直到全部通过或者有一个规则生效为止:

- 指定应用生效的规则，即针对调用方限流的；
- 调用方为 other 的规则；
- 调用方为 default 的规则。

### 2.2.5 DegradeSlot

这个 slot 主要针对资源的平均响应时间（RT）以及异常比率，来决定资源是否在接下来的时间被自动熔断掉。

### 2.2.6 SystemSlot

这个 slot 会根据对于当前系统的整体情况，对入口资源的调用进行动态调配。其原理是让入口的流量和当前系统的预计容量达到一个动态平衡。

注意系统规则只对入口流量起作用（调用类型为 `EntryType.IN`），对出口流量无效。可通过 `SphU.entry(res, entryType)` 指定调用类型，如果不指定，默认是`EntryType.OUT`。

### 2.2.7 其他

> [Sentinel 核心类解析](https://github.com/alibaba/Sentinel/wiki/Sentinel-核心类解析)

# 3. 使用文档

## 3.1 基本使用

使用 Sentinel 来进行资源保护，主要分为几个步骤:

1. 定义资源
2. 定义规则
3. 检验规则是否生效

### 3.1.1 定义资源

1. 方式一：主流框架的默认适配

   为了减少开发的复杂程度，Sentinel对大部分的主流框架，例如 Web Servlet、Dubbo、Spring Cloud、gRPC、Spring WebFlux、Reactor 等都做了适配。您只需要引入对应的依赖即可方便地整合 Sentinel。可以参见：[主流框架的适配](https://sentinelguard.io/zh-cn/docs/open-source-framework-integrations.html)。

2. 方式二：抛出异常的方式定义资源

   `SphU` 包含了 try-catch 风格的 API。用这种方式，当资源发生了限流之后会抛出 `BlockException`。这个时候可以捕捉异常，进行限流之后的逻辑处理。示例代码如下:

   ```java
   // 1.5.0 版本开始可以利用 try-with-resources 特性
   // 资源名可使用任意有业务语义的字符串，比如方法名、接口名或其它可唯一标识的字符串。
   try (Entry entry = SphU.entry("resourceName")) {
     // 被保护的业务逻辑
     // do something here...
   } catch (BlockException ex) {
     // 资源访问阻止，被限流或被降级
     // 在此处进行相应的处理操作
   }
   ```

   **特别地**，若 entry 的时候传入了热点参数，那么 exit 的时候也一定要带上对应的参数（`exit(count, args)`），否则可能会有统计错误。这个时候不能使用 try-with-resources 的方式。另外通过 `Tracer.trace(ex)` 来统计异常信息时，由于 try-with-resources 语法中 catch 调用顺序的问题，会导致无法正确统计异常数，因此统计异常信息时也不能在 try-with-resources 的 catch 块中调用 `Tracer.trace(ex)`。

   1.5.0 之前的版本的示例：

   ```java
   Entry entry = null;
   // 务必保证finally会被执行
   try {
     // 资源名可使用任意有业务语义的字符串
     entry = SphU.entry("自定义资源名");
     // 被保护的业务逻辑
     // do something...
   } catch (BlockException e1) {
     // 资源访问阻止，被限流或被降级
     // 进行相应的处理操作
   } finally {
     if (entry != null) {
       entry.exit();
     }
   }
   ```

   **注意：** `SphU.entry(xxx)` 需要与 `entry.exit()` 方法成对出现，匹配调用，否则会导致调用链记录异常，抛出 `ErrorEntryFreeException` 异常。

3. 方式三：返回布尔值方式定义资源

   `SphO` 提供 if-else 风格的 API。用这种方式，当资源发生了限流之后会返回 `false`，这个时候可以根据返回值，进行限流之后的逻辑处理。示例代码如下:

   ```java
   // 资源名可使用任意有业务语义的字符串
   if (SphO.entry("自定义资源名")) {
     // 务必保证finally会被执行
     try {
       /**
         * 被保护的业务逻辑
         */
     } finally {
       SphO.exit();
     }
   } else {
     // 资源访问阻止，被限流或被降级
     // 进行相应的处理操作
   }
   ```

4. 方式四：注解方式定义资源

   Sentinel 支持通过 `@SentinelResource` 注解定义资源并配置 `blockHandler` 和 `fallback` 函数来进行限流之后的处理。示例：

   ```java
   // 原本的业务方法.
   @SentinelResource(blockHandler = "blockHandlerForGetUser")
   public User getUserById(String id) {
       throw new RuntimeException("getUserById command failed");
   }
   
   // blockHandler 函数，原方法调用被限流/降级/系统保护的时候调用
   public User blockHandlerForGetUser(String id, BlockException ex) {
       return new User("admin");
   }
   ```

   **注意 `blockHandler` 函数会在原方法被限流/降级/系统保护的时候调用，而 `fallback` 函数会针对所有类型的异常**。请注意 `blockHandler` 和 `fallback` 函数的形式要求，更多指引可以参见 [Sentinel 注解支持文档](https://sentinelguard.io/zh-cn/docs/annotation-support.html)。

5. 异步调用支持

   Sentinel 支持异步调用链路的统计。在异步调用中，需要通过 `SphU.asyncEntry(xxx)` 方法定义资源，并通常需要在异步的回调函数中调用 `exit` 方法。以下是一个简单的示例：

   ```java
   try {
       AsyncEntry entry = SphU.asyncEntry(resourceName);
   
       // 异步调用.
       doAsync(userId, result -> {
           try {
               // 在此处处理异步调用的结果.
           } finally {
               // 在回调结束后 exit.
               entry.exit();
           }
       });
   } catch (BlockException ex) {
       // Request blocked.
       // Handle the exception (e.g. retry or fallback).
   }
   ```

   **`SphU.asyncEntry(xxx)` 不会影响当前（调用线程）的 Context，因此以下两个 entry 在调用链上是平级关系（处于同一层），而不是嵌套关系**：

   ```java
   // 调用链类似于：
   // -parent
   // ---asyncResource
   // ---syncResource
   asyncEntry = SphU.asyncEntry(asyncResource);
   entry = SphU.entry(normalResource);
   ```

   若在异步回调中需要嵌套其它的资源调用（无论是 `entry` 还是 `asyncEntry`），只需要借助 Sentinel 提供的上下文切换功能，在对应的地方通过 `ContextUtil.runOnContext(context, f)` 进行 Context 变换，将对应资源调用处的 Context 切换为生成的异步 Context，即可维持正确的调用链路关系。示例如下：

   ```java
   public void handleResult(String result) {
       Entry entry = null;
       try {
           entry = SphU.entry("handleResultForAsync");
           // Handle your result here.
       } catch (BlockException ex) {
           // Blocked for the result handler.
       } finally {
           if (entry != null) {
               entry.exit();
           }
       }
   }
   
   public void someAsync() {
       try {
           AsyncEntry entry = SphU.asyncEntry(resourceName);
   
           // Asynchronous invocation.
           doAsync(userId, result -> {
               // 在异步回调中进行上下文变换，通过 AsyncEntry 的 getAsyncContext 方法获取异步 Context
               ContextUtil.runOnContext(entry.getAsyncContext(), () -> {
                   try {
                       // 此处嵌套正常的资源调用.
                       handleResult(result);
                   } finally {
                       entry.exit();
                   }
               });
           });
       } catch (BlockException ex) {
           // Request blocked.
           // Handle the exception (e.g. retry or fallback).
       }
   }
   ```

   此时的调用链就类似于：

   ```
   -parent
   ---asyncInvocation
   -----handleResultForAsync
   ```

   更详细的示例可以参考 Demo 中的 [AsyncEntryDemo](https://github.com/alibaba/Sentinel/blob/master/sentinel-demo/sentinel-demo-basic/src/main/java/com/alibaba/csp/sentinel/demo/AsyncEntryDemo.java)，里面包含了普通资源与异步资源之间的各种嵌套示例。

### 3.1.2 规则种类

Sentinel 支持以下几种规则：

1. **流量控制规则**

2. **熔断降级规则**

3. **系统保护规则**

4. **来源访问控制规则** 

5. **热点参数规则**

---

#### 流量控制规则(FlowRule)

**流量规则的定义**

重要属性：

|      Field      | 说明                                                         | 默认值                        |
| :-------------: | :----------------------------------------------------------- | :---------------------------- |
|    resource     | 资源名，资源名是限流规则的作用对象                           |                               |
|      count      | 限流阈值                                                     |                               |
|      grade      | 限流阈值类型，**QPS 或线程数模式**                           | QPS 模式                      |
|    limitApp     | 流控针对的调用来源                                           | `default`，代表不区分调用来源 |
|    strategy     | 调用关系限流策略：直接、链路、关联                           | 根据资源本身（直接）          |
| controlBehavior | 流控效果（直接拒绝 / 排队等待 / 慢启动模式），不支持按调用关系限流 | 直接拒绝                      |

同一个资源可以同时有多个限流规则。

**通过代码定义流量控制规则**

理解上面规则的定义之后，我们可以通过调用 `FlowRuleManager.loadRules()` 方法来用硬编码的方式定义流量控制规则，比如：

```java
private static void initFlowQpsRule() {
    List<FlowRule> rules = new ArrayList<>();
    FlowRule rule1 = new FlowRule();
    rule1.setResource(resource);
    // Set max qps to 20
    rule1.setCount(20);
    rule1.setGrade(RuleConstant.FLOW_GRADE_QPS);
    rule1.setLimitApp("default");
    rules.add(rule1);
    FlowRuleManager.loadRules(rules);
}
```

更多详细内容可以参考 [流量控制](https://sentinelguard.io/zh-cn/docs/flow-control.html)。

#### 熔断降级规则 (DegradeRule)

熔断降级规则包含下面几个重要的属性：

|       Field        | 说明                                                         | 默认值     |
| :----------------: | :----------------------------------------------------------- | :--------- |
|      resource      | 资源名，即规则的作用对象                                     |            |
|       grade        | 熔断策略，支持**慢调用比例/异常比例/异常数策略**             | 慢调用比例 |
|       count        | 慢调用比例模式下为慢调用临界 RT（超出该值计为慢调用）；异常比例/异常数模式下为对应的阈值 |            |
|     timeWindow     | 熔断时长，单位为 s                                           |            |
|  minRequestAmount  | **熔断触发的最小请求数，请求数小于该值时即使异常比率超出阈值也不会熔断**（1.7.0 引入） | 5          |
|   statIntervalMs   | 统计时长（单位为 ms），如 60*1000 代表分钟级（1.8.0 引入）   | 1000 ms    |
| slowRatioThreshold | 慢调用比例阈值，仅慢调用比例模式有效（1.8.0 引入）           |            |

同一个资源可以同时有多个降级规则。

理解上面规则的定义之后，我们可以通过调用 `DegradeRuleManager.loadRules()` 方法来用硬编码的方式定义流量控制规则。

```java
private static void initDegradeRule() {
    List<DegradeRule> rules = new ArrayList<>();
    DegradeRule rule = new DegradeRule(resource);
        .setGrade(CircuitBreakerStrategy.ERROR_RATIO.getType());
        .setCount(0.7); // Threshold is 70% error ratio
        .setMinRequestAmount(100)
        .setStatIntervalMs(30000) // 30s
        .setTimeWindow(10);
    rules.add(rule);
    DegradeRuleManager.loadRules(rules);
}
```

更多详情可以参考 [熔断降级](https://sentinelguard.io/zh-cn/docs/circuit-breaking.html)。

#### 系统保护规则 (SystemRule)

Sentinel 系统自适应限流从整体维度对应用入口流量进行控制，结合应用的 Load、CPU 使用率、总体平均 RT、入口 QPS 和并发线程数等几个维度的监控指标，通过自适应的流控策略，让系统的入口流量和系统的负载达到一个平衡，让系统尽可能跑在最大吞吐量的同时保证系统整体的稳定性。

系统规则包含下面几个重要的属性：

|       Field       | 说明                                   | 默认值      |
| :---------------: | :------------------------------------- | :---------- |
| highestSystemLoad | `load1` 触发值，用于触发自适应控制阶段 | -1 (不生效) |
|       avgRt       | 所有入口流量的平均响应时间             | -1 (不生效) |
|     maxThread     | 入口流量的最大并发数                   | -1 (不生效) |
|        qps        | 所有入口资源的 QPS                     | -1 (不生效) |
|  highestCpuUsage  | 当前系统的 CPU 使用率（0.0-1.0）       | -1 (不生效) |

理解上面规则的定义之后，我们可以通过调用 `SystemRuleManager.loadRules()` 方法来用硬编码的方式定义流量控制规则：

```java
private void initSystemProtectionRule() {
  List<SystemRule> rules = new ArrayList<>();
  SystemRule rule = new SystemRule();
  rule.setHighestSystemLoad(10);
  rules.add(rule);
  SystemRuleManager.loadRules(rules);
}
```

更多详情可以参考 [系统自适应保护](https://sentinelguard.io/zh-cn/docs/system-adaptive-protection.html)。

#### 访问控制规则 (AuthorityRule)

很多时候，我们需要根据调用方来限制资源是否通过，这时候可以使用 Sentinel 的访问控制（黑白名单）的功能。黑白名单根据资源的请求来源（`origin`）限制资源是否通过，若配置白名单则只有请求来源位于白名单内时才可通过；若配置黑名单则请求来源位于黑名单时不通过，其余的请求通过。

授权规则，即黑白名单规则（`AuthorityRule`）非常简单，主要有以下配置项：

- `resource`：资源名，即限流规则的作用对象
- `limitApp`：对应的黑名单/白名单，不同 origin 用 `,` 分隔，如 `appA,appB`
- `strategy`：限制模式，`AUTHORITY_WHITE` 为白名单模式，`AUTHORITY_BLACK` 为黑名单模式，默认为白名单模式

更多详情可以参考 [来源访问控制](https://sentinelguard.io/zh-cn/docs/origin-authority-control.html)。

#### 热点规则 (ParamFlowRule)

详情可以参考 [热点参数限流](https://sentinelguard.io/zh-cn/docs/parameter-flow-control.html)。

### 3.1.3 查询更改规则

引入了 transport 模块后，可以通过以下的 HTTP API 来获取所有已加载的规则：

```
http://localhost:8719/getRules?type=<XXXX>
```

其中，`type=flow` 以 JSON 格式返回现有的限流规则，degrade 返回现有生效的降级规则列表，system 则返回系统保护规则。

获取所有热点规则：

```
http://localhost:8719/getParamRules
```

其中，type 可以输入 `flow`、`degrade` 等方式来制定更改的规则种类，`data` 则是对应的 JSON 格式的规则。

### 3.1.4 定制自己的持久化规则

上面的规则配置，都是存在内存中的。即如果应用重启，这个规则就会失效。可以通过实现 [`DataSource`](https://github.com/alibaba/Sentinel/blob/master/sentinel-extension/sentinel-datasource-extension/src/main/java/com/alibaba/csp/sentinel/datasource/AbstractDataSource.java) 接口的方式，来自定义规则的存储数据源。通常建议：

- 整合动态配置系统，如 ZooKeeper、[Nacos](https://github.com/alibaba/Nacos) 等，动态地实时刷新配置规则
- 结合 RDBMS、NoSQL、VCS 等来实现该规则
- 配合 Sentinel Dashboard 使用

更多详情请参考 [动态规则配置](https://sentinelguard.io/zh-cn/docs/dynamic-rule-configuration.html)。

### 3.1.5 规则生效的效果

#### 判断限流降级异常

通过以下方法判断是否为 Sentinel 的流控降级异常：

```java
BlockException.isBlockException(Throwable t);
```

除了在业务代码逻辑上看到规则生效，我们也可以通过下面简单的方法，来校验规则生效的效果：

- **暴露的 HTTP 接口**：通过运行下面命令 `curl http://localhost:8719/cnode?id=<资源名称>`，观察返回的数据。如果规则生效，在返回的数据栏中的 `block` 以及 `block(m)` 中会有显示
- **日志**：Sentinel 提供秒级的资源运行日志以及限流日志，详情可以参考 [日志文档](https://sentinelguard.io/zh-cn/docs/logs.html)

#### block 事件

Sentinel 提供以下扩展接口，可以通过 `StatisticSlotCallbackRegistry` 向 `StatisticSlot` 注册回调函数：

- `ProcessorSlotEntryCallback`: callback when resource entry passed (`onPass`) or blocked (`onBlocked`)
- `ProcessorSlotExitCallback`: callback when resource entry successfully completed (`onExit`)

可以利用这些回调接口来实现报警等功能，实时的监控信息可以从 `ClusterNode` 中实时获取。

### 3.1.6 其它 API

#### 业务异常统计 Tracer

业务异常记录类 `Tracer` 用于记录业务异常。相关方法：

- `trace(Throwable e)`：记录业务异常（非 `BlockException` 异常），对应的资源为当前线程 context 下 entry 对应的资源。
- `trace(Throwable e, int count)`：记录业务异常（非 `BlockException` 异常），异常数目为传入的 `count`。
- `traceEntry(Throwable, int, Entry)`：向传入 entry 对应的资源记录业务异常（非 `BlockException` 异常），异常数目为传入的 `count`。

如果用户通过 `SphU` 或 `SphO` 手动定义资源，则 Sentinel 不能感知上层业务的异常，需要手动调用 `Tracer.trace(ex)` 来记录业务异常，否则对应的异常不会统计到 Sentinel 异常计数中。注意不要在 try-with-resources 形式的 `SphU.entry(xxx)` 中使用，否则会统计不上。

从 1.3.1 版本开始，注解方式定义资源支持自动统计业务异常，无需手动调用 `Tracer.trace(ex)` 来记录业务异常。Sentinel 1.3.1 以前的版本需要手动记录。

#### 上下文工具类 ContextUtil

相关静态方法：

**标识进入调用链入口（上下文）**：

以下静态方法用于标识调用链路入口，用于区分不同的调用链路：

- `public static Context enter(String contextName)`
- `public static Context enter(String contextName, String origin)`

其中 `contextName` 代表调用链路入口名称（上下文名称），`origin` 代表调用来源名称。默认调用来源为空。返回值类型为 `Context`，即生成的调用链路上下文对象。

**注意**：`ContextUtil.enter(xxx)` 方法仅在调用链路入口处生效，即仅在当前线程的初次调用生效，后面再调用不会覆盖当前线程的调用链路，直到 exit。`Context` 存于 ThreadLocal 中，因此切换线程时可能会丢掉，如果需要跨线程使用可以结合 `runOnContext` 方法使用。

流控规则中若选择“流控方式”为“链路”方式，则入口资源名即为上面的 `contextName`。

**退出调用链（清空上下文）**：

- `public static void exit()`：该方法用于退出调用链，清理当前线程的上下文。

**获取当前线程的调用链上下文**：

- `public static Context getContext()`：获取当前线程的调用链路上下文对象。

**在某个调用链上下文中执行代码**：

- `public static void runOnContext(Context context, Runnable f)`：常用于异步调用链路中 context 的变换。

#### 指标统计配置

**Sentinel 底层采用高性能的滑动窗口数据结构来统计实时的秒级指标数据，并支持对滑动窗口进行配置**。主要有以下两个配置：

- `windowIntervalMs`：滑动窗口的总的时间长度，默认为 1000 ms
- `sampleCount`：滑动窗口划分的格子数目，默认为 2；格子越多则精度越高，但是内存占用也会越多

![sliding-window-leap-array](https://user-images.githubusercontent.com/9434884/51955215-0af7c500-247e-11e9-8895-9fc0e4c10c8c.png)

我们可以通过 `SampleCountProperty` 来动态地变更滑动窗口的格子数目，通过 `IntervalProperty` 来动态地变更滑动窗口的总时间长度。注意这两个配置都是**全局生效**的，会影响所有资源的所有指标统计。

#### Dashboard

详情请参考：[Sentinel Dashboard 文档](https://sentinelguard.io/zh-cn/docs/dashboard.html)。

## 3.2 流量控制

`FlowSlot` 会根据预设的规则，结合前面 `NodeSelectorSlot`、`ClusterNodeBuilderSlot`、`StatistcSlot` 统计出来的实时信息进行流量控制。

**限流的直接表现是在执行 `Entry nodeA = SphU.entry(资源名字)` 的时候抛出 `FlowException` 异常。`FlowException` 是 `BlockException` 的子类，您可以捕捉 `BlockException` 来自定义被限流之后的处理逻辑**。

同一个资源可以对应多条限流规则。`FlowSlot` 会对该资源的所有限流规则依次遍历，直到有规则触发限流或者所有规则遍历完毕。

一条限流规则主要由下面几个因素组成，我们可以组合这些元素来实现不同的限流效果：

- `resource`：资源名，即限流规则的作用对象
- `count`: 限流阈值
- `grade`: 限流阈值类型，QPS 或线程数
- `strategy`: 根据调用关系选择策略

### 3.2.1 基于QPS/并发数的流量控制

**流量控制主要有两种统计类型，一种是统计线程数，另外一种则是统计 QPS**。类型由 `FlowRule.grade` 字段来定义。其中，0 代表根据并发数量来限流，1 代表根据 QPS 来进行流量控制。其中线程数、QPS 值，都是由 `StatisticSlot` 实时统计获取的。

可以通过下面的命令查看实时统计信息：

```shell
curl http://localhost:8719/cnode?id=resourceName
```

输出内容格式如下：

```
idx id   thread  pass  blocked   success  total Rt   1m-pass   1m-block   1m-all   exeption
2   abc647 0     46     0           46     46   1       2763      0         2763     0
```

其中：

- thread： 代表当前处理该资源的线程数；
- pass： 代表一秒内到来到的请求；
- blocked： 代表一秒内被流量控制的请求数量；
- success： 代表一秒内成功处理完的请求；
- total： 代表到一秒内到来的请求以及被阻止的请求总和；
- RT： 代表一秒内该资源的平均响应时间；
- 1m-pass： 则是一分钟内到来的请求；
- 1m-block： 则是一分钟内被阻止的请求；
- 1m-all： 则是一分钟内到来的请求和被阻止的请求的总和；
- exception： 则是一秒内业务本身异常的总和。

#### 并发线程数流量控制

线程数限流用于保护业务线程数不被耗尽。

<small>例如，当应用所依赖的下游应用由于某种原因导致服务不稳定、响应延迟增加，对于调用者来说，意味着吞吐量下降和更多的线程数占用，极端情况下甚至导致线程池耗尽。为应对高线程占用的情况，业内有使用隔离的方案，比如通过不同业务逻辑使用不同线程池来隔离业务自身之间的资源争抢（线程池隔离），或者使用信号量来控制同时请求的个数（信号量隔离）。这种隔离方案虽然能够控制线程数量，但无法控制请求排队时间。当请求过多时排队也是无益的，直接拒绝能够迅速降低系统压力。</small>

**Sentinel线程数限流不负责创建和管理线程池，而是简单统计当前请求上下文的线程个数，如果超出阈值，新的请求会被立即拒绝**。

#### QPS流量控制

**当 QPS 超过某个阈值的时候，则采取措施进行流量控制**。流量控制的手段包括下面 3 种，对应 `FlowRule` 中的 `controlBehavior` 字段：

1. **直接拒绝（`RuleConstant.CONTROL_BEHAVIOR_DEFAULT`）方式。该方式是默认的流量控制方式，当QPS超过任意规则的阈值后，新的请求就会被立即拒绝，拒绝方式为抛出`FlowException`**。这种方式适用于对系统处理能力确切已知的情况下，比如通过压测确定了系统的准确水位时。具体的例子参见 [FlowqpsDemo](https://github.com/alibaba/Sentinel/blob/master/sentinel-demo/sentinel-demo-basic/src/main/java/com/alibaba/csp/sentinel/demo/flow/FlowQpsDemo.java)。
2. 冷启动（`RuleConstant.CONTROL_BEHAVIOR_WARM_UP`）方式。该方式主要用于系统长期处于低水位的情况下，当流量突然增加时，直接把系统拉升到高水位可能瞬间把系统压垮。**通过"冷启动"，让通过的流量缓慢增加，在一定时间内逐渐增加到阈值上限，给冷系统一个预热的时间，避免冷系统被压垮的情况**。具体的例子参见 [WarmUpFlowDemo](https://github.com/alibaba/Sentinel/blob/master/sentinel-demo/sentinel-demo-basic/src/main/java/com/alibaba/csp/sentinel/demo/flow/WarmUpFlowDemo.java)。

通常冷启动的过程系统允许通过的 QPS 曲线如下图所示：

![img](https://pic2.zhimg.com/80/v2-655fbd07193f5195e5e5405ce483b61d_1440w.webp)

1. 匀速器（`RuleConstant.CONTROL_BEHAVIOR_RATE_LIMITER`）方式。这种方式严格控制了请求通过的间隔时间，也即是让请求以均匀的速度通过，对应的是**漏桶算法**。具体的例子参见 [PaceFlowDemo](https://github.com/alibaba/Sentinel/blob/master/sentinel-demo/sentinel-demo-basic/src/main/java/com/alibaba/csp/sentinel/demo/flow/PaceFlowDemo.java)。

该方式的作用如下图所示：

![img](https://github.com/alibaba/Sentinel/wiki/image/queue.gif)

<small>（图片，当阈值QPS=2时，每隔500ms才允许通过下一个请求）</small>

**这种方式主要用于处理间隔性突发的流量，例如消息队列**。想象一下这样的场景，在某一秒有大量的请求到来，而接下来的几秒则处于空闲状态，我们希望系统能够在接下来的空闲期间逐渐处理这些请求，而不是在第一秒直接拒绝多余的请求。

> [算法：限流之漏桶算法实现-CSDN博客](https://blog.csdn.net/xiaobaiPlayGame/article/details/127597031)

### 3.2.2 基于调用关系的流量控制

调用关系包括调用方、被调用方；方法又可能会调用其它方法，形成一个调用链路的层次关系。Sentinel 通过 `NodeSelectorSlot` 建立不同资源间的调用的关系，并且通过 `ClusterNodeBuilderSlot` 记录每个资源的实时统计信息。

有了调用链路的统计信息，我们可以衍生出多种流量控制手段。

#### 根据调用方限流

`ContextUtil.enter(resourceName, origin)` 方法中的 `origin` 参数标明了调用方身份。这些信息会在 `ClusterBuilderSlot` 中被统计。可通过以下命令来展示不同的调用方对同一个资源的调用数据：

```shell
curl http://localhost:8719/origin?id=nodeA
```

调用数据示例：

```
id: nodeA
idx origin  threadNum passedQps blockedQps totalQps aRt   1m-passed 1m-blocked 1m-total 
1   caller1 0         0         0          0        0     0         0          0
2   caller2 0         0         0          0        0     0         0          0
```

上面这个命令展示了资源名为 `nodeA` 的资源被两个不同的调用方调用的统计。

限流规则中的 `limitApp` 字段用于根据调用方进行流量控制。该字段的值有以下三种选项，分别对应不同的场景：

- `default`：表示不区分调用者，来自任何调用者的请求都将进行限流统计。如果这个资源名的调用总和超过了这条规则定义的阈值，则触发限流。
- `{some_origin_name}`：表示针对特定的调用者，只有来自这个调用者的请求才会进行流量控制。例如 `NodeA` 配置了一条针对调用者`caller1`的规则，那么当且仅当来自 `caller1` 对 `NodeA` 的请求才会触发流量控制。
- `other`：表示针对除 `{some_origin_name}` 以外的其余调用方的流量进行流量控制。例如，资源`NodeA`配置了一条针对调用者 `caller1` 的限流规则，同时又配置了一条调用者为 `other` 的规则，那么任意来自非 `caller1` 对 `NodeA` 的调用，都不能超过 `other` 这条规则定义的阈值。

同一个资源名可以配置多条规则，规则的生效顺序为：**{some_origin_name} > other > default**

#### 根据调用链路入口限流：链路限流

`NodeSelectorSlot` 中记录了资源之间的调用链路，这些资源通过调用关系，相互之间构成一棵调用树。这棵树的根节点是一个名字为 `machine-root` 的虚拟节点，调用链的入口都是这个虚节点的子节点。

一棵典型的调用树如下图所示：

```
     	          machine-root
                    /       \
                   /         \
             Entrance1     Entrance2
                /             \
               /               \
      DefaultNode(nodeA)   DefaultNode(nodeA)
```

上图中来自入口 `Entrance1` 和 `Entrance2` 的请求都调用到了资源 `NodeA`，Sentinel 允许只根据某个入口的统计信息对资源限流。比如我们可以设置 `FlowRule.strategy` 为 `RuleConstant.CHAIN`，同时设置 `FlowRule.ref_identity` 为 `Entrance1` 来表示只有从入口 `Entrance1` 的调用才会记录到 `NodeA` 的限流统计当中，而对来自 `Entrance2` 的调用漠不关心。

调用链的入口是通过 API 方法 `ContextUtil.enter(name)` 定义的。

#### 具有关系的资源流量控制：关联流量控制

当两个资源之间具有资源争抢或者依赖关系的时候，这两个资源便具有了关联。比如对数据库同一个字段的读操作和写操作存在争抢，读的速度过高会影响写得速度，写的速度过高会影响读的速度。如果放任读写操作争抢资源，则争抢本身带来的开销会降低整体的吞吐量。可使用关联限流来避免具有关联关系的资源之间过度的争抢，举例来说，`read_db` 和 `write_db` 这两个资源分别代表数据库读写，我们可以给 `read_db` 设置限流规则来达到写优先的目的：设置 `FlowRule.strategy` 为 `RuleConstant.RELATE` 同时设置 `FlowRule.ref_identity` 为 `write_db`。这样当写库操作过于频繁时，读数据的请求会被限流。

## 3.3 熔断降级

除了流量控制以外，对调用链路中不稳定的资源进行熔断降级也是保障高可用的重要措施之一。

如果依赖的服务出现了不稳定的情况，请求的响应时间变长，那么调用服务的方法的响应时间也会变长，线程会产生堆积，最终可能耗尽业务自身的线程池，服务本身也变得不可用。

![chain](https://user-images.githubusercontent.com/9434884/62410811-cd871680-b61d-11e9-9df7-3ee41c618644.png)

现代微服务架构都是分布式的，由非常多的服务组成。不同服务之间相互调用，组成复杂的调用链路。以上的问题在链路调用中会产生放大的效果。<u>复杂链路上的某一环不稳定，就可能会层层级联，最终导致整个链路都不可用。因此我们需要对不稳定的**弱依赖服务调用**进行熔断降级，暂时切断不稳定调用，避免局部不稳定因素导致整体的雪崩。熔断降级作为保护自身的手段，通常在客户端（调用端）进行配置</u>。

> **注意**：本文档针对 Sentinel 1.8.0 及以上版本。1.8.0 版本对熔断降级特性进行了全新的改进升级，请使用最新版本以更好地利用熔断降级的能力。

### 3.3.1 熔断策略

Sentinel 提供以下几种熔断策略：

- **慢调用比例 (`SLOW_REQUEST_RATIO`)**：选择以慢调用比例作为阈值，需要设置允许的慢调用 RT（即最大的响应时间），请求的响应时间大于该值则统计为慢调用。当单位统计时长（`statIntervalMs`）内请求数目大于设置的最小请求数目，并且慢调用的比例大于阈值，则接下来的熔断时长内请求会自动被熔断。经过熔断时长后熔断器会进入探测恢复状态（HALF-OPEN 状态），若接下来的一个请求响应时间小于设置的慢调用 RT 则结束熔断，若大于设置的慢调用 RT 则会再次被熔断。
- **异常比例 (`ERROR_RATIO`)**：当单位统计时长（`statIntervalMs`）内请求数目大于设置的最小请求数目，并且异常的比例大于阈值，则接下来的熔断时长内请求会自动被熔断。经过熔断时长后熔断器会进入探测恢复状态（HALF-OPEN 状态），若接下来的一个请求成功完成（没有错误）则结束熔断，否则会再次被熔断。异常比率的阈值范围是 `[0.0, 1.0]`，代表 0% - 100%。
- **异常数 (`ERROR_COUNT`)**：当单位统计时长内的异常数目超过阈值之后会自动进行熔断。经过熔断时长后熔断器会进入探测恢复状态（HALF-OPEN 状态），若接下来的一个请求成功完成（没有错误）则结束熔断，否则会再次被熔断。

注意异常降级**仅针对业务异常**，对 Sentinel 限流降级本身的异常（`BlockException`）不生效。为了统计异常比例或异常数，需要通过 `Tracer.trace(ex)` 记录业务异常。示例：

```java
Entry entry = null;
try {
  entry = SphU.entry(resource);

  // Write your biz code here.
  // <<BIZ CODE>>
} catch (Throwable t) {
  if (!BlockException.isBlockException(t)) {
    Tracer.trace(t);
  }
} finally {
  if (entry != null) {
    entry.exit();
  }
}
```

开源整合模块，如 Sentinel Dubbo Adapter, Sentinel Web Servlet Filter 或 `@SentinelResource` 注解会自动统计业务异常，无需手动调用。

### 3.3.2 熔断降级规则说明

熔断降级规则（DegradeRule）包含下面几个重要的属性：

|       Field        | 说明                                                         | 默认值     |
| :----------------: | :----------------------------------------------------------- | :--------- |
|      resource      | 资源名，即规则的作用对象                                     |            |
|       grade        | 熔断策略，支持慢调用比例/异常比例/异常数策略                 | 慢调用比例 |
|       count        | 慢调用比例模式下为慢调用临界 RT（超出该值计为慢调用）；异常比例/异常数模式下为对应的阈值 |            |
|     timeWindow     | 熔断时长，单位为 s                                           |            |
|  minRequestAmount  | 熔断触发的最小请求数，请求数小于该值时即使异常比率超出阈值也不会熔断（1.7.0 引入） | 5          |
|   statIntervalMs   | 统计时长（单位为 ms），如 60*1000 代表分钟级（1.8.0 引入）   | 1000 ms    |
| slowRatioThreshold | 慢调用比例阈值，仅慢调用比例模式有效（1.8.0 引入）           |            |

### 3.3.3 熔断器事件监听

Sentinel 支持注册自定义的事件监听器监听熔断器状态变换事件（state change event）。示例：

```java
EventObserverRegistry.getInstance().addStateChangeObserver("logging",
    (prevState, newState, rule, snapshotValue) -> {
        if (newState == State.OPEN) {
            // 变换至 OPEN state 时会携带触发时的值
            System.err.println(String.format("%s -> OPEN at %d, snapshotValue=%.2f", prevState.name(),
                TimeUtil.currentTimeMillis(), snapshotValue));
        } else {
            System.err.println(String.format("%s -> %s at %d", prevState.name(), newState.name(),
                TimeUtil.currentTimeMillis()));
        }
    });
```

### 3.3.4 示例

> 慢调用比例熔断示例：[SlowRatioCircuitBreakerDemo](https://github.com/alibaba/Sentinel/blob/master/sentinel-demo/sentinel-demo-basic/src/main/java/com/alibaba/csp/sentinel/demo/degrade/SlowRatioCircuitBreakerDemo.java)

## 3.4 系统自适应保护

Sentinel 系统自适应保护从整体维度对应用入口流量进行控制，结合应用的 Load、总体平均 RT、入口 QPS 和线程数等几个维度的监控指标，让系统的入口流量和系统的负载达到一个平衡，让系统尽可能跑在最大吞吐量的同时保证系统整体的稳定性。

### 3.4.1 背景

在开始之前，先回顾一下 Sentinel 做系统自适应保护的目的：

- 保证系统不被拖垮
- 在系统稳定的前提下，保持系统的吞吐量

长期以来，系统自适应保护的思路是根据硬指标，即系统的负载 (load1) 来做系统过载保护。当系统负载高于某个阈值，就禁止或者减少流量的进入；当 load 开始好转，则恢复流量的进入。这个思路给我们带来了不可避免的两个问题：

- load 是一个“果”，如果根据 load 的情况来调节流量的通过率，那么就始终有延迟性。也就意味着通过率的任何调整，都会过一段时间才能看到效果。当前通过率是使 load 恶化的一个动作，那么也至少要过 1 秒之后才能观测到；同理，如果当前通过率调整是让 load 好转的一个动作，也需要 1 秒之后才能继续调整，这样就浪费了系统的处理能力。所以我们看到的曲线，总是会有抖动。
- 恢复慢。想象一下这样的一个场景（真实），出现了这样一个问题，下游应用不可靠，导致应用 RT 很高，从而 load 到了一个很高的点。过了一段时间之后下游应用恢复了，应用 RT 也相应减少。这个时候，其实应该大幅度增大流量的通过率；但是由于这个时候 load 仍然很高，通过率的恢复仍然不高。

[TCP BBR](https://en.wikipedia.org/wiki/TCP_congestion_control#TCP_BBR) 的思想给了我们一个很大的启发。我们应该根据系统能够处理的请求，和允许进来的请求，来做平衡，而不是根据一个间接的指标（系统 load）来做限流。最终我们追求的目标是 **在系统不被拖垮的情况下，提高系统的吞吐率，而不是 load 一定要到低于某个阈值**。如果我们还是按照固有的思维，超过特定的 load 就禁止流量进入，系统 load 恢复就放开流量，这样做的结果是无论我们怎么调参数，调比例，都是按照果来调节因，都无法取得良好的效果。

**Sentinel 在系统自适应保护的做法是，用 load1 作为启动控制流量的值，而允许通过的流量由处理请求的能力，即请求的响应时间以及当前系统正在处理的请求速率来决定。**

### 3.4.2 系统规则

系统保护规则是从应用级别的入口流量进行控制，从单台机器的<u>总体 Load、RT、入口 QPS 和线程数</u>四个维度监控应用数据，让系统尽可能跑在最大吞吐量的同时保证系统整体的稳定性。

<u>**系统保护规则是应用整体维度的，而不是资源维度的**，并且**仅对入口流量生效**</u>。入口流量指的是进入应用的流量（`EntryType.IN`），比如 Web 服务或 Dubbo 服务端接收的请求，都属于入口流量。

系统规则支持以下的阈值类型：

- **Load**（仅对 Linux/Unix-like 机器生效）：当系统 load1 超过阈值，且系统当前的并发线程数超过系统容量时才会触发系统保护。系统容量由系统的 `maxQps * minRt` 计算得出。设定参考值一般是 `CPU cores * 2.5`。
- **CPU usage**（1.5.0+ 版本）：当系统 CPU 使用率超过阈值即触发系统保护（取值范围 0.0-1.0）。
- **RT**：当单台机器上所有入口流量的平均 RT 达到阈值即触发系统保护，单位是毫秒。
- **线程数**：当单台机器上所有入口流量的并发线程数达到阈值即触发系统保护。
- **入口 QPS**：当单台机器上所有入口流量的 QPS 达到阈值即触发系统保护。

> [系统负载Load的理解和分析 - 简书 (jianshu.com)](https://www.jianshu.com/p/735210d3e2dc)
>
> 系统负载平均值是操作系统报告的一个统计信息，表示在特定时间段内处于运行状态或等待CPU时间片的进程数量。常见的负载平均值包括`load1`（过去一分钟）、`load5`（过去五分钟）和`load15`（过去十五分钟）。这些值越高，说明系统越忙。

### 3.4.3 原理

先用经典图来镇楼:

![TCP-BBR-pipe](https://user-images.githubusercontent.com/9434884/50813887-bff10300-1352-11e9-9201-437afea60a5a.png)

我们把系统处理请求的过程想象为一个水管，到来的请求是往这个水管灌水，当系统处理顺畅的时候，请求不需要排队，直接从水管中穿过，这个请求的RT是最短的；反之，当请求堆积的时候，那么处理请求的时间则会变为：排队时间 + 最短处理时间。

- 推论一: 如果我们能够保证水管里的水量，能够让水顺畅的流动，则不会增加排队的请求；也就是说，这个时候的系统负载不会进一步恶化。

我们用 T 来表示(水管内部的水量)，用RT来表示请求的处理时间，用P来表示进来的请求数，那么一个请求从进入水管道到从水管出来，这个水管会存在 `P * RT`　个请求。换一句话来说，当 `T ≈ QPS * Avg(RT)` 的时候，我们可以认为系统的处理能力和允许进入的请求个数达到了平衡，系统的负载不会进一步恶化。

接下来的问题是，水管的水位是可以达到了一个平衡点，但是这个平衡点只能保证水管的水位不再继续增高，但是还面临一个问题，就是在达到平衡点之前，这个水管里已经堆积了多少水。如果之前水管的水已经在一个量级了，那么这个时候系统允许通过的水量可能只能缓慢通过，RT会大，之前堆积在水管里的水会滞留；反之，如果之前的水管水位偏低，那么又会浪费了系统的处理能力。

- 推论二:　当保持入口的流量使水管出来的流量达到最大值的时候，可以最大利用水管的处理能力。

然而，和 TCP BBR 的不一样的地方在于，还需要用一个系统负载的值（load1）来激发这套机制启动。

> 注：这种系统自适应算法对于低 load 的请求，它的效果是一个“兜底”的角色。**对于不是应用本身造成的 load 高的情况（如其它进程导致的不稳定的情况），效果不明显。**

### 3.4.4 示例

> 我们提供了系统自适应限流的示例：[SystemGuardDemo](https://github.com/alibaba/Sentinel/blob/master/sentinel-demo/sentinel-demo-basic/src/main/java/com/alibaba/csp/sentinel/demo/system/SystemGuardDemo.java)。

## 3.5 集群流量控制

### 3.5.1 介绍

为什么要使用集群流控呢？假设我们希望给某个用户限制调用某个 API 的总 QPS 为 50，但机器数可能很多（比如有 100 台）。这时候我们很自然地就想到，找一个 server 来专门来统计总的调用量，其它的实例都与这台 server 通信来判断是否可以调用。这就是最基础的集群流控的方式。

另外集群流控还可以解决流量不均匀导致总体限流效果不佳的问题。假设集群中有 10 台机器，我们给每台机器设置单机限流阈值为 10 QPS，理想情况下整个集群的限流阈值就为 100 QPS。不过实际情况下流量到每台机器可能会不均匀，会导致总量没有到的情况下某些机器就开始限流。因此仅靠单机维度去限制的话会无法精确地限制总体流量。而集群流控可以精确地控制整个集群的调用总量，结合单机限流兜底，可以更好地发挥流量控制的效果。

集群流控中共有两种身份：

- **Token Client：集群流控客户端，用于向所属 Token Server 通信请求 token。集群限流服务端会返回给客户端结果，决定是否限流。**
- **Token Server：即集群流控服务端，处理来自 Token Client 的请求，根据配置的集群规则判断是否应该发放 token（是否允许通过）。**

![image](https://user-images.githubusercontent.com/9434884/65305357-8f39bc80-dbb5-11e9-96d6-d1111fc365a9.png)

### 3.5.2 模块结构

Sentinel 1.4.0 开始引入了集群流控模块，主要包含以下几部分：

- `sentinel-cluster-common-default`: 公共模块，包含公共接口和实体
- `sentinel-cluster-client-default`: 默认集群流控 client 模块，使用 Netty 进行通信，提供接口方便序列化协议扩展
- `sentinel-cluster-server-default`: 默认集群流控 server 模块，使用 Netty 进行通信，提供接口方便序列化协议扩展；同时提供扩展接口对接规则判断的具体实现（`TokenService`），默认实现是复用 `sentinel-core` 的相关逻辑

> 注意：集群流控模块要求 JDK 版本最低为 1.7。

### 3.5.3 集群流控规则

#### 规则

`FlowRule` 添加了两个字段用于集群限流相关配置：

```java
private boolean clusterMode; // 标识是否为集群限流配置
private ClusterFlowConfig clusterConfig; // 集群限流相关配置项
```

其中 用一个专门的 `ClusterFlowConfig` 代表集群限流相关配置项，以与现有规则配置项分开：

```java
// 全局唯一的规则 ID，由集群限流管控端分配.
private Long flowId;

// 阈值模式，默认（0）为单机均摊，1 为全局阈值.
private int thresholdType = ClusterRuleConstant.FLOW_THRESHOLD_AVG_LOCAL;

private int strategy = ClusterRuleConstant.FLOW_CLUSTER_STRATEGY_NORMAL;

// 在 client 连接失败或通信失败时，是否退化到本地的限流模式
private boolean fallbackToLocalWhenFail = true;
```

- `flowId` 代表全局唯一的规则 ID，Sentinel 集群限流服务端通过此 ID 来区分各个规则，因此**务必保持全局唯一**。一般 flowId 由统一的管控端进行分配，或写入至 DB 时生成。
- <u>`thresholdType` 代表集群限流阈值模式。其中**单机均摊模式**下配置的阈值等同于单机能够承受的限额，token server 会根据客户端对应的 namespace（默认为 `project.name` 定义的应用名）下的连接数来计算总的阈值（比如独立模式下有 3 个 client 连接到了 token server，然后配的单机均摊阈值为 10，则计算出的集群总量就为 30）；而全局模式下配置的阈值等同于**整个集群的总阈值**。</u>

`ParamFlowRule` 热点参数限流相关的集群配置与 `FlowRule` 相似。

#### 配置方式
在集群流控的场景下，我们推荐使用动态规则源来动态地管理规则。

对于客户端，我们可以按照原有的方式来向 FlowRuleManager 和 ParamFlowRuleManager 注册动态规则源，例如：

```java
ReadableDataSource<String, List<FlowRule>> flowRuleDataSource = new NacosDataSource<>(remoteAddress, groupId, dataId, parser);
FlowRuleManager.register2Property(flowRuleDataSource.getProperty());
```

对于集群流控 token server，由于集群限流服务端有作用域（namespace）的概念，因此我们需要注册一个自动根据 namespace 生成动态规则源的 PropertySupplier:

```java
// Supplier 类型：接受 namespace，返回生成的动态规则源，类型为 SentinelProperty<List<FlowRule>>
// ClusterFlowRuleManager 针对集群限流规则，ClusterParamFlowRuleManager 针对集群热点规则，配置方式类似
ClusterFlowRuleManager.setPropertySupplier(namespace -> {
    return new SomeDataSource(namespace).getProperty();
});
```

<u>然后每当集群限流服务端 namespace set 产生变更时，Sentinel 会自动针对新加入的 namespace 生成动态规则源并进行自动监听，并删除旧的不需要的规则源。</u>

### 3.5.4 集群限流客户端

要想使用集群限流功能，必须引入集群限流 client 相关依赖：

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-cluster-client-default</artifactId>
    <version>1.8.6</version>
</dependency>
```

用户可以通过 API 将当前模式置为客户端模式：

```bash
http://<ip>:<port>/setClusterMode?mode=<xxx>
```

其中 mode 为 0 代表 client，1 代表 server。设置成功后，若已有客户端的配置，集群限流客户端将会开启并连接远程的 token server。我们可以在 `sentinel-record.log` 日志中查看连接的相关日志。

若集群限流客户端未进行配置，则用户需要对客户端进行基本的配置，比如指定集群限流 token server。我们提供了 API 进行配置：

```bash
http://<ip>:<port>/cluster/client/modifyConfig?data=<config>
```

其中 data 是 JSON 格式的 `ClusterClientConfig`，对应的配置项：

- `serverHost`: token server host
- `serverPort`: token server 端口
- `requestTimeout`: 请求的超时时间（默认为 20 ms）

当然也可以通过动态配置源进行配置。我们可以通过 `ClusterClientConfigManager` 的 `registerServerAssignProperty` 方法注册动态配置源。配置源注册的相关逻辑可以置于 `InitFunc` 实现类中，并通过 SPI 注册，在 Sentinel 初始化时即可自动进行配置源加载监听。

若用户未引入集群限流 client 相关依赖，或者 client 未开启/连接失败/通信失败，则对于开启了集群模式的规则：

- 集群热点限流默认直接通过
- 普通集群限流会退化到 local 模式的限流，即在本地按照单机阈值执行限流检查

当 token client 与 server 之间的连接意外断开时，token client 会不断进行重试，每次重试的间隔时间以 `n * 2000 ms` 的形式递增。

### 3.5.5 集群限流服务端

要想使用集群限流服务端，必须引入集群限流 server 相关依赖：

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-cluster-server-default</artifactId>
    <version>1.8.6</version>
</dependency>
```

#### 启动方式

Sentinel 集群限流服务端有两种启动方式：

- 独立模式（Alone），即作为独立的 token server 进程启动，独立部署，隔离性好，但是需要额外的部署操作。<u>独立模式适合作为 Global Rate Limiter 给集群提供流控服务</u>。

![image](https://user-images.githubusercontent.com/9434884/50463606-c3d26c00-09c7-11e9-8373-1c27e2408f8b.png)

- 嵌入模式（Embedded），即作为内置的 token server 与服务在同一进程中启动。在此模式下，集群中各个实例都是对等的，token server 和 client 可以随时进行转变，因此无需单独部署，灵活性比较好。但是隔离性不佳，需要限制 token server 的总 QPS，防止影响应用本身。嵌入模式适合某个应用集群内部的流控。

![image](https://user-images.githubusercontent.com/9434884/50463600-b7e6aa00-09c7-11e9-9580-6919f0d0a8a4.png)

我们提供了 API 用于在 embedded 模式下转换集群流控身份：

```bash
http://<ip>:<port>/setClusterMode?mode=<xxx>
```

其中 mode 为 `0` 代表 client，`1` 代表 server，`-1` 代表关闭。注意应用端需要引入集群限流客户端或服务端的相应依赖。

在独立模式下，我们可以直接创建对应的 `ClusterTokenServer` 实例并在 main 函数中通过 `start` 方法启动 Token Server。

#### 规则配置

见前面“规则配置”相关内容。

#### 属性配置

我们推荐给集群限流服务端注册动态配置源来动态地进行配置。配置类型有以下几种：

- namespace set: 集群限流服务端服务的作用域（命名空间），可以设置为自己服务的应用名。集群限流 client 在连接到 token server 后会上报自己的命名空间（默认为 `project.name` 配置的应用名），token server 会根据上报的命名空间名称统计连接数。
- transport config: 集群限流服务端通信相关配置，如 server port
- flow config: 集群限流服务端限流相关配置，如滑动窗口统计时长、格子数目、最大允许总 QPS等

我们可以通过 `ClusterServerConfigManager` 的各个 `registerXxxProperty` 方法来注册相关的配置源。

从 1.4.1 版本开始，Sentinel 支持给 token server 配置最大允许的总 QPS（`maxAllowedQps`），来对 token server 的资源使用进行限制，防止在嵌入模式下影响应用本身。

### 3.5.6 Token Server 分配配置

![image](https://user-images.githubusercontent.com/9434884/58071181-60dbae80-7bce-11e9-9dc8-8e27e2161b0d.png)

#### 示例

sentinel-demo-cluster 提供了嵌入模式和独立模式的示例：

- [sentinel-demo-cluster-server-alone](https://github.com/alibaba/Sentinel/blob/master/sentinel-demo/sentinel-demo-cluster/sentinel-demo-cluster-server-alone/src/main/java/com/alibaba/csp/sentinel/demo/cluster/ClusterServerDemo.java)：独立模式 Demo
- [sentinel-demo-cluster-embedded](https://github.com/alibaba/Sentinel/tree/master/sentinel-demo/sentinel-demo-cluster/sentinel-demo-cluster-embedded)：嵌入模式 Demo，以 Web 应用为示例，可以启动多个实例分别作为 Token Server 和 Token Client。数据源的相关配置可以参考 [DemoClusterInitFunc](https://github.com/alibaba/Sentinel/blob/master/sentinel-demo/sentinel-demo-cluster/sentinel-demo-cluster-embedded/src/main/java/com/alibaba/csp/sentinel/demo/cluster/init/DemoClusterInitFunc.java)。

> 注意：若在本地启动多个 Demo 示例，需要加上 `-Dcsp.sentinel.log.use.pid=true` 参数，否则控制台显示监控会不准确。

#### 集群限流控制台

使用集群限流功能需要对 Sentinel 控制台进行相关的改造，推送规则时直接推送至配置中心，接入端引入 push 模式的动态数据源。可以参考 [Sentinel 控制台（集群流控管理文档）](https://github.com/alibaba/Sentinel/wiki/Sentinel-控制台（集群流控管理）)。

同时云上版本 [AHAS Sentinel](https://help.aliyun.com/document_detail/174871.html) 提供**开箱即用的全自动托管集群流控能力**，无需手动指定/分配 token server 以及管理连接状态，同时支持分钟小时级别流控、大流量低延时场景流控场景，同时支持 Istio/Envoy 场景的 Mesh 流控能力。

#### 扩展接口设计

**整体扩展架构**

![overview-arch](https://user-images.githubusercontent.com/9434884/49844934-b03bd880-fdff-11e8-838e-6299ee24ff08.png)

**通用扩展接口**

以下通用接口位于 `sentinel-core` 中：

- `TokenService`: 集群限流功能接口，server / client 均可复用
- `ClusterTokenClient`: 集群限流功能客户端
- `ClusterTokenServer`: 集群限流服务端接口
- `EmbeddedClusterTokenServer`: 集群限流服务端接口（embedded 模式）

以下通用接口位于 `sentinel-cluster-common-default`:

- `EntityWriter`
- `EntityDecoder`

**Client 扩展接口**

集群流控 Client 端通信相关扩展接口：

- `ClusterTransportClient`：集群限流通信客户端
- `RequestEntityWriter`
- `ResponseEntityDecoder`

**Server 扩展接口**

集群流控 Server 端通信相关扩展接口：

- `ResponseEntityWriter`
- `RequestEntityDecoder`

集群流控 Server 端请求处理扩展接口：

- `RequestProcessor`: 请求处理接口 (request -> response)

## 3.6 网关流量控制

Sentinel 支持对 Spring Cloud Gateway、Zuul 等主流的 API Gateway 进行限流。

![sentinel-api-gateway-common-arch](https://user-images.githubusercontent.com/9434884/58381714-266d7980-7ff3-11e9-8617-d0d7c325d703.png)

Sentinel 1.6.0 引入了 Sentinel API Gateway Adapter Common 模块，此模块中包含网关限流的规则和自定义 API 的实体和管理逻辑：

- `GatewayFlowRule`：网关限流规则，针对 API Gateway 的场景定制的限流规则，<u>可以针对不同 route 或自定义的 API 分组进行限流，支持针对请求中的参数、Header、来源 IP 等进行定制化的限流</u>。
- `ApiDefinition`：用户自定义的 API 定义分组，可以看做是一些 URL 匹配的组合。比如我们可以定义一个 API 叫 `my_api`，请求 path 模式为 `/foo/**` 和 `/baz/**` 的都归到 `my_api` 这个 API 分组下面。限流的时候可以针对这个自定义的 API 分组维度进行限流。

其中网关限流规则 `GatewayFlowRule` 的字段解释如下：

- `resource`：资源名称，可以是网关中的 route 名称或者用户自定义的 API 分组名称。
- `resourceMode`：规则是针对 API Gateway 的 route（`RESOURCE_MODE_ROUTE_ID`）还是用户在 Sentinel 中定义的 API 分组（`RESOURCE_MODE_CUSTOM_API_NAME`），默认是 route。
- `grade`：限流指标维度，同限流规则的 `grade` 字段。
- `count`：限流阈值
- `intervalSec`：统计时间窗口，单位是秒，默认是 1 秒。
- `controlBehavior`：流量整形的控制效果，同限流规则的 `controlBehavior` 字段，目前支持快速失败和匀速排队两种模式，默认是快速失败。
- `burst`：应对突发请求时额外允许的请求数目。
- `maxQueueingTimeoutMs`：匀速排队模式下的最长排队时间，单位是毫秒，仅在匀速排队模式下生效。
- `paramItem`：参数限流配置。若不提供，则代表不针对参数进行限流，该网关规则将会被转换成普通流控规则；否则会转换成热点规则。其中的字段：
  - `parseStrategy`：从请求中提取参数的策略，目前支持提取来源 IP（`PARAM_PARSE_STRATEGY_CLIENT_IP`）、Host（`PARAM_PARSE_STRATEGY_HOST`）、任意 Header（`PARAM_PARSE_STRATEGY_HEADER`）和任意 URL 参数（`PARAM_PARSE_STRATEGY_URL_PARAM`）四种模式。
  - `fieldName`：若提取策略选择 Header 模式或 URL 参数模式，则需要指定对应的 header 名称或 URL 参数名称。
  - `pattern`：参数值的匹配模式，只有匹配该模式的请求属性值会纳入统计和流控；若为空则统计该请求属性的所有值。（1.6.2 版本开始支持）
  - `matchStrategy`：参数值的匹配策略，目前支持精确匹配（`PARAM_MATCH_STRATEGY_EXACT`）、子串匹配（`PARAM_MATCH_STRATEGY_CONTAINS`）和正则匹配（`PARAM_MATCH_STRATEGY_REGEX`）。（1.6.2 版本开始支持）

用户可以通过 `GatewayRuleManager.loadRules(rules)` 手动加载网关规则，或通过 `GatewayRuleManager.register2Property(property)` 注册动态规则源动态推送（推荐方式）。

### 3.6.1 Spring Cloud Gateway

从 1.6.0 版本开始，Sentinel 提供了 Spring Cloud Gateway 的适配模块，可以提供两种资源维度的限流：

- route 维度：即在 Spring 配置文件中配置的路由条目，资源名为对应的 routeId
- 自定义 API 维度：用户可以利用 Sentinel 提供的 API 来自定义一些 API 分组

使用时需引入以下模块（以 Maven 为例）：

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-spring-cloud-gateway-adapter</artifactId>
    <version>x.y.z</version>
</dependency>
```

使用时只需注入对应的 `SentinelGatewayFilter` 实例以及 `SentinelGatewayBlockExceptionHandler` 实例即可。比如：

```java
@Configuration
public class GatewayConfiguration {

    private final List<ViewResolver> viewResolvers;
    private final ServerCodecConfigurer serverCodecConfigurer;

    public GatewayConfiguration(ObjectProvider<List<ViewResolver>> viewResolversProvider,
                                ServerCodecConfigurer serverCodecConfigurer) {
        this.viewResolvers = viewResolversProvider.getIfAvailable(Collections::emptyList);
        this.serverCodecConfigurer = serverCodecConfigurer;
    }

    @Bean
    @Order(Ordered.HIGHEST_PRECEDENCE)
    public SentinelGatewayBlockExceptionHandler sentinelGatewayBlockExceptionHandler() {
        // Register the block exception handler for Spring Cloud Gateway.
        return new SentinelGatewayBlockExceptionHandler(viewResolvers, serverCodecConfigurer);
    }

    @Bean
    @Order(-1)
    public GlobalFilter sentinelGatewayFilter() {
        return new SentinelGatewayFilter();
    }
}
```

Demo 示例：[sentinel-demo-spring-cloud-gateway](https://github.com/alibaba/Sentinel/tree/master/sentinel-demo/sentinel-demo-spring-cloud-gateway)

比如我们在 Spring Cloud Gateway 中配置了以下路由：

```yaml
server:
  port: 8090
spring:
  application:
    name: spring-cloud-gateway
  cloud:
    gateway:
      enabled: true
      discovery:
        locator:
          lower-case-service-id: true
      routes:
        # Add your routes here.
        - id: product_route
          uri: lb://product
          predicates:
            - Path=/product/**
        - id: httpbin_route
          uri: https://httpbin.org
          predicates:
            - Path=/httpbin/**
          filters:
            - RewritePath=/httpbin/(?<segment>.*), /$\{segment}
```

同时自定义了一些 API 分组：

```java
private void initCustomizedApis() {
    Set<ApiDefinition> definitions = new HashSet<>();
    ApiDefinition api1 = new ApiDefinition("some_customized_api")
        .setPredicateItems(new HashSet<ApiPredicateItem>() {{
            add(new ApiPathPredicateItem().setPattern("/product/baz"));
            add(new ApiPathPredicateItem().setPattern("/product/foo/**")
                .setMatchStrategy(SentinelGatewayConstants.PARAM_MATCH_STRATEGY_PREFIX));
        }});
    ApiDefinition api2 = new ApiDefinition("another_customized_api")
        .setPredicateItems(new HashSet<ApiPredicateItem>() {{
            add(new ApiPathPredicateItem().setPattern("/ahas"));
        }});
    definitions.add(api1);
    definitions.add(api2);
    GatewayApiDefinitionManager.loadApiDefinitions(definitions);
}
```

那么这里面的 route ID（如 `product_route`）和 API name（如 `some_customized_api`）都会被标识为 Sentinel 的资源。比如访问网关的 URL 为 `http://localhost:8090/product/foo/22` 的时候，对应的统计会加到 `product_route` 和 `some_customized_api` 这两个资源上面，而 `http://localhost:8090/httpbin/json` 只会对应到 `httpbin_route` 资源上面。

> **注意**：有的时候 Spring Cloud Gateway 会自己在 route 名称前面拼一个前缀，如 `ReactiveCompositeDiscoveryClient_xxx` 这种。请观察簇点链路页面实际的资源名。

您可以在 `GatewayCallbackManager` 注册回调进行定制：

- `setBlockHandler`：注册函数用于实现自定义的逻辑处理被限流的请求，对应接口为 `BlockRequestHandler`。默认实现为 `DefaultBlockRequestHandler`，当被限流时会返回类似于下面的错误信息：`Blocked by Sentinel: FlowException`。

**注意**：

- Sentinel 网关流控默认的粒度是 route 维度以及自定义 API 分组维度，默认**不支持 URL 粒度**。若通过 Spring Cloud Alibaba 接入，请将 `spring.cloud.sentinel.filter.enabled` 配置项置为 false（若在网关流控控制台上看到了 URL 资源，就是此配置项没有置为 false）。
- 若使用 Spring Cloud Alibaba Sentinel 数据源模块，需要注意网关流控规则数据源类型是 `gw-flow`，若将网关流控规则数据源指定为 flow 则不生效。

### 3.6.2 Zuul 1.x

Sentinel 提供了 Zuul 1.x 的适配模块，可以为 Zuul Gateway 提供两种资源维度的限流：

- route 维度：即在 Spring 配置文件中配置的路由条目，资源名为对应的 route ID（对应 `RequestContext` 中的 `proxy` 字段）
- 自定义 API 维度：用户可以利用 Sentinel 提供的 API 来自定义一些 API 分组

使用时需引入以下模块（以 Maven 为例）：

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-zuul-adapter</artifactId>
    <version>x.y.z</version>
</dependency>
```

若使用的是 Spring Cloud Netflix Zuul，我们可以直接在配置类中将三个 filter 注入到 Spring 环境中即可：

```java
@Configuration
public class ZuulConfig {

    @Bean
    public ZuulFilter sentinelZuulPreFilter() {
        // We can also provider the filter order in the constructor.
        return new SentinelZuulPreFilter();
    }

    @Bean
    public ZuulFilter sentinelZuulPostFilter() {
        return new SentinelZuulPostFilter();
    }

    @Bean
    public ZuulFilter sentinelZuulErrorFilter() {
        return new SentinelZuulErrorFilter();
    }
}
```

Sentinel Zuul Adapter 生成的调用链路类似于下面，其中的资源名都是 route ID 或者自定义的 API 分组名称：

```
-EntranceNode: sentinel_gateway_context$$route$$another-route-b(t:0 pq:0.0 bq:0.0 tq:0.0 rt:0.0 prq:0.0 1mp:8 1mb:1 1mt:9)
--another-route-b(t:0 pq:0.0 bq:0.0 tq:0.0 rt:0.0 prq:0.0 1mp:4 1mb:1 1mt:5)
--another_customized_api(t:0 pq:0.0 bq:0.0 tq:0.0 rt:0.0 prq:0.0 1mp:4 1mb:0 1mt:4)
-EntranceNode: sentinel_gateway_context$$route$$my-route-1(t:0 pq:0.0 bq:0.0 tq:0.0 rt:0.0 prq:0.0 1mp:6 1mb:0 1mt:6)
--my-route-1(t:0 pq:0.0 bq:0.0 tq:0.0 rt:0.0 prq:0.0 1mp:2 1mb:0 1mt:2)
--some_customized_api(t:0 pq:0.0 bq:0.0 tq:0.0 rt:0.0 prq:0.0 1mp:2 1mb:0 1mt:2)
```

发生限流之后的处理流程 ：

- 发生限流之后可自定义返回参数，通过实现 `SentinelFallbackProvider` 接口，默认的实现是 `DefaultBlockFallbackProvider`。
- 默认的 fallback route 的规则是 route ID 或自定义的 API 分组名称。

比如：

```java
// 自定义 FallbackProvider 
public class MyBlockFallbackProvider implements ZuulBlockFallbackProvider {

    private Logger logger = LoggerFactory.getLogger(DefaultBlockFallbackProvider.class);
    
    // you can define route as service level 
    @Override
    public String getRoute() {
        return "/book/app";
    }

    @Override
        public BlockResponse fallbackResponse(String route, Throwable cause) {
            RecordLog.info(String.format("[Sentinel DefaultBlockFallbackProvider] Run fallback route: %s", route));
            if (cause instanceof BlockException) {
                return new BlockResponse(429, "Sentinel block exception", route);
            } else {
                return new BlockResponse(500, "System Error", route);
            }
        }
 }
 
 // 注册 FallbackProvider
 ZuulBlockFallbackManager.registerProvider(new MyBlockFallbackProvider());
```

限流发生之后的默认返回：

```json
{
    "code":429,
    "message":"Sentinel block exception",
    "route":"/"
}
```

**注意**：

- Sentinel 网关流控默认的粒度是 route 维度以及自定义 API 分组维度，默认**不支持 URL 粒度**。若通过 Spring Cloud Alibaba 接入，请将 `spring.cloud.sentinel.filter.enabled` 配置项置为 false（若在网关流控控制台上看到了 URL 资源，就是此配置项没有置为 false）。
- 若使用 Spring Cloud Alibaba Sentinel 数据源模块，需要注意网关流控规则数据源类型是 `gw-flow`，若将网关流控规则数据源指定为 flow 则不生效。

### 3.6.3 Zuul 2.x

> 注：从 1.7.2 版本开始支持，需要 Java 8 及以上版本。

Sentinel 提供了 Zuul 2.x 的适配模块，可以为 Zuul Gateway 提供两种资源维度的限流：

- route 维度：对应 SessionContext 中的 `routeVIP`
- 自定义 API 维度：用户可以利用 Sentinel 提供的 API 来自定义一些 API 分组

使用时需引入以下模块（以 Maven 为例）：

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-zuul2-adapter</artifactId>
    <version>x.y.z</version>
</dependency>
```

然后配置对应的 filter 即可：

```java
filterMultibinder.addBinding().toInstance(new SentinelZuulInboundFilter(500));
filterMultibinder.addBinding().toInstance(new SentinelZuulOutboundFilter(500));
filterMultibinder.addBinding().toInstance(new SentinelZuulEndpoint());
```

可以参考 [sentinel-demo-zuul2-gateway 示例](https://github.com/alibaba/Sentinel/tree/master/sentinel-demo/sentinel-demo-zuul2-gateway)。

### 3.6.4 网关流控实现原理

**当通过 `GatewayRuleManager` 加载网关流控规则（`GatewayFlowRule`）时，无论是否针对请求属性进行限流，Sentinel 底层都会将网关流控规则转化为热点参数规则（`ParamFlowRule`），存储在 `GatewayRuleManager` 中，与正常的热点参数规则相隔离。转换时 Sentinel 会根据请求属性配置，为网关流控规则设置参数索引（`idx`），并同步到生成的热点参数规则中**。

外部请求进入 API Gateway 时会经过 Sentinel 实现的 filter，其中会依次进行 **路由/API 分组匹配**、**请求属性解析**和**参数组装**。Sentinel 会根据配置的网关流控规则来解析请求属性，并依照参数索引顺序组装参数数组，最终传入 `SphU.entry(res, args)` 中。Sentinel API Gateway Adapter Common 模块向 Slot Chain 中添加了一个 `GatewayFlowSlot`，专门用来做网关规则的检查。`GatewayFlowSlot` 会从 `GatewayRuleManager` 中提取生成的热点参数规则，根据传入的参数依次进行规则检查。若某条规则不针对请求属性，则会在参数最后一个位置置入预设的常量，达到普通流控的效果。

![image](https://user-images.githubusercontent.com/9434884/58381786-5406f280-7ff4-11e9-9020-016ccaf7ab7d.png)

### 3.6.5 网关流控控制台

Sentinel 1.6.3 引入了网关流控控制台的支持，用户可以直接在 Sentinel 控制台上查看 API Gateway 实时的 route 和自定义 API 分组监控，管理网关规则和 API 分组配置。

在 API Gateway 端，用户只需要在[原有启动参数](https://github.com/alibaba/Sentinel/wiki/控制台#32-配置启动参数)的基础上添加如下启动参数即可标记应用为 API Gateway 类型：

```bash
# 注：通过 Spring Cloud Alibaba Sentinel 自动接入的 API Gateway 整合则无需此参数
-Dcsp.sentinel.app.type=1
```

添加正确的启动参数并有访问量后，我们就可以在 Sentinel 上面看到对应的 API Gateway 了。我们可以查看实时的 route 和自定义 API 分组的监控和调用信息：

![sentinel-dashboard-api-gateway-route-list](https://sentinelguard.io/blog/zh-cn/img/sentinel-dashboard-api-gateway-route-list.png)

我们可以在控制台配置自定义的 API 分组，将一些 URL 匹配模式归为一个 API 分组：

![sentinel-dashboard-api-gateway-customized-api-group](https://sentinelguard.io/blog/zh-cn/img/sentinel-dashboard-api-gateway-customized-api-group.png)

然后我们可以在控制台针对预设的 route ID 或自定义的 API 分组配置网关流控规则：

![sentinel-dashboard-api-gateway-flow-rule](https://sentinelguard.io/blog/zh-cn/img/sentinel-dashboard-api-gateway-flow-rule.png)

网关规则动态配置及网关集群流控可以参考 [AHAS Sentinel 网关流控](https://help.aliyun.com/document_detail/118482.html)。

## 3.7 热点参数限流

何为热点？热点即经常访问的数据。很多时候我们希望统计某个热点数据中访问频次最高的 Top K 数据，并对其访问进行限制。比如：

- 商品 ID 为参数，统计一段时间内最常购买的商品 ID 并进行限制
- 用户 ID 为参数，针对一段时间内频繁访问的用户 ID 进行限制

热点参数限流会统计传入参数中的热点参数，并根据配置的限流阈值与模式，对包含热点参数的资源调用进行限流。热点参数限流可以看做是一种特殊的流量控制，仅对包含热点参数的资源调用生效。

![Sentinel Parameter Flow Control](https://github.com/alibaba/Sentinel/wiki/image/sentinel-hot-param-overview-1.png)

Sentinel 利用 **LRU 策略**统计最近最常访问的热点参数，结合**令牌桶算法**来进行参数级别的流控。

### 3.7.1 基本使用

要使用热点参数限流功能，需要引入以下依赖：

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-parameter-flow-control</artifactId>
    <version>x.y.z</version>
</dependency>
```

然后为对应的资源配置热点参数限流规则，并在 `entry` 的时候传入相应的参数，即可使热点参数限流生效。

> 注：若自行扩展并注册了自己实现的 `SlotChainBuilder`，并希望使用热点参数限流功能，则可以在 chain 里面合适的地方插入 `ParamFlowSlot`。

那么如何传入对应的参数以便 Sentinel 统计呢？我们可以通过 `SphU` 类里面几个 `entry` 重载方法来传入：

```java
public static Entry entry(String name, EntryType type, int count, Object... args) throws BlockException

public static Entry entry(Method method, EntryType type, int count, Object... args) throws BlockException
```

其中最后的一串 `args` 就是要传入的参数，有多个就按照次序依次传入。比如要传入两个参数 `paramA` 和 `paramB`，则可以：

```java
// paramA in index 0, paramB in index 1.
// 若需要配置例外项或者使用集群维度流控，则传入的参数只支持基本类型。
SphU.entry(resourceName, EntryType.IN, 1, paramA, paramB);
```

<u>**注意**：若 entry 的时候传入了热点参数，那么 exit 的时候也一定要带上对应的参数（`exit(count, args)`），否则可能会有统计错误</u>。正确的示例：

```java
Entry entry = null;
try {
    entry = SphU.entry(resourceName, EntryType.IN, 1, paramA, paramB);
    // Your logic here.
} catch (BlockException ex) {
    // Handle request rejection.
} finally {
    if (entry != null) {
        entry.exit(1, paramA, paramB);
    }
}
```

对于 `@SentinelResource` 注解方式定义的资源，若注解作用的方法上有参数，Sentinel 会将它们作为参数传入 `SphU.entry(res, args)`。比如以下的方法里面 `uid` 和 `type` 会分别作为第一个和第二个参数传入 Sentinel API，从而可以用于热点规则判断：

```java
@SentinelResource("myMethod")
public Result doSomething(String uid, int type) {
  // some logic here...
}
```

### 3.7.2 热点参数规则

热点参数规则（`ParamFlowRule`）类似于流量控制规则（`FlowRule`）：

|       属性        | 说明                                                         | 默认值   |
| :---------------: | :----------------------------------------------------------- | :------- |
|     resource      | 资源名，必填                                                 |          |
|       count       | 限流阈值，必填                                               |          |
|       grade       | 限流模式                                                     | QPS 模式 |
|   durationInSec   | 统计窗口时间长度（单位为秒），1.6.0 版本开始支持             | 1s       |
|  controlBehavior  | 流控效果（支持快速失败和匀速排队模式），1.6.0 版本开始支持   | 快速失败 |
| maxQueueingTimeMs | 最大排队等待时长（仅在匀速排队模式生效），1.6.0 版本开始支持 | 0ms      |
|     paramIdx      | 热点参数的索引，必填，对应 `SphU.entry(xxx, args)` 中的参数索引位置 |          |
| paramFlowItemList | 参数例外项，可以针对指定的参数值单独设置限流阈值，不受前面 `count` 阈值的限制。**仅支持基本类型和字符串类型** |          |
|    clusterMode    | 是否是集群参数流控规则                                       | `false`  |
|   clusterConfig   | 集群流控相关配置                                             |          |

我们可以通过 `ParamFlowRuleManager` 的 `loadRules` 方法更新热点参数规则，下面是一个示例：

```java
ParamFlowRule rule = new ParamFlowRule(resourceName)
    .setParamIdx(0)
    .setCount(5);
// 针对 int 类型的参数 PARAM_B，单独设置限流 QPS 阈值为 10，而不是全局的阈值 5.
ParamFlowItem item = new ParamFlowItem().setObject(String.valueOf(PARAM_B))
    .setClassType(int.class.getName())
    .setCount(10);
rule.setParamFlowItemList(Collections.singletonList(item));

ParamFlowRuleManager.loadRules(Collections.singletonList(rule));
```

### 3.7.3 示例

> 示例可参见 [sentinel-demo-parameter-flow-control](https://github.com/alibaba/Sentinel/blob/master/sentinel-demo/sentinel-demo-parameter-flow-control/src/main/java/com/alibaba/csp/sentinel/demo/flow/param/ParamFlowQpsDemo.java)。

## 3.8 注解支持

Sentinel 提供了 `@SentinelResource` 注解用于定义资源，并提供了 AspectJ 的扩展用于自动定义资源、处理 `BlockException` 等。使用 [Sentinel Annotation AspectJ Extension](https://github.com/alibaba/Sentinel/tree/master/sentinel-extension/sentinel-annotation-aspectj) 的时候需要引入以下依赖：

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-annotation-aspectj</artifactId>
    <version>x.y.z</version>
</dependency>
```

### 3.8.1 @SentinelResource 注解

> 注意：注解方式埋点不支持 private 方法。

`@SentinelResource` 用于定义资源，并提供可选的异常处理和 fallback 配置项。 `@SentinelResource` 注解包含以下属性：

- `value`：资源名称，必需项（不能为空）
- `entryType`：entry 类型，可选项（默认为 `EntryType.OUT`）
- `blockHandler` / `blockHandlerClass`: `blockHandler` 对应处理 `BlockException` 的函数名称，可选项。blockHandler 函数访问范围需要是 `public`，返回类型需要与原方法相匹配，参数类型需要和原方法相匹配并且最后加一个额外的参数，类型为 `BlockException`。<u>blockHandler 函数默认需要和原方法在同一个类中。若希望使用其他类的函数，则可以指定 `blockHandlerClass` 为对应的类的 `Class` 对象，注意对应的函数必需为 static 函数，否则无法解析</u>。
- `fallback`：fallback 函数名称，可选项，用于在抛出异常的时候提供 fallback 处理逻辑。fallback 函数可以针对所有类型的异常（除了`exceptionsToIgnore`里面排除掉的异常类型）进行处理。fallback 函数签名和位置要求：
  - **返回值类型必须与原函数返回值类型一致；**
  - **方法参数列表需要和原函数一致，或者可以额外多一个 `Throwable` 类型的参数用于接收对应的异常。**
  - **fallback 函数默认需要和原方法在同一个类中。若希望使用其他类的函数，则可以指定 `fallbackClass` 为对应的类的 `Class` 对象，注意对应的函数必需为 static 函数，否则无法解析。**
- `defaultFallback`（since 1.6.0）：默认的 fallback 函数名称，可选项，通常用于通用的 fallback 逻辑（即可以用于很多服务或方法）。默认 fallback 函数可以针对所以类型的异常（除了`exceptionsToIgnore`里面排除掉的异常类型）进行处理。若同时配置了 fallback 和 defaultFallback，则只有 fallback 会生效。defaultFallback 函数签名要求：
  - 返回值类型必须与原函数返回值类型一致；
  - 方法参数列表需要为空，或者可以额外多一个 `Throwable` 类型的参数用于接收对应的异常。
  - defaultFallback 函数默认需要和原方法在同一个类中。若希望使用其他类的函数，则可以指定 `fallbackClass` 为对应的类的 `Class` 对象，注意对应的函数必需为 static 函数，否则无法解析。
- `exceptionsToIgnore`（since 1.6.0）：用于指定哪些异常被排除掉，不会计入异常统计中，也不会进入 fallback 逻辑中，而是会原样抛出。

> 注：1.6.0 之前的版本 fallback 函数只针对降级异常（`DegradeException`）进行处理，**不能针对业务异常进行处理**。

特别地，若 blockHandler 和 fallback 都进行了配置，则被限流降级而抛出 `BlockException` 时只会进入 `blockHandler` 处理逻辑。若未配置 `blockHandler`、`fallback` 和 `defaultFallback`，则被限流降级时会将 `BlockException` **直接抛出**。

示例：

```java
public class TestService {

    // 对应的 `handleException` 函数需要位于 `ExceptionUtil` 类中，并且必须为 static 函数.
    @SentinelResource(value = "test", blockHandler = "handleException", blockHandlerClass = {ExceptionUtil.class})
    public void test() {
        System.out.println("Test");
    }

    // 原函数
    @SentinelResource(value = "hello", blockHandler = "exceptionHandler", fallback = "helloFallback")
    public String hello(long s) {
        return String.format("Hello at %d", s);
    }
    
    // Fallback 函数，函数签名与原函数一致或加一个 Throwable 类型的参数.
    public String helloFallback(long s) {
        return String.format("Halooooo %d", s);
    }

    // Block 异常处理函数，参数最后多一个 BlockException，其余与原函数一致.
    public String exceptionHandler(long s, BlockException ex) {
        // Do some log here.
        ex.printStackTrace();
        return "Oops, error occurred at " + s;
    }
}
```

从 1.4.0 版本开始，注解方式定义资源支持自动统计业务异常，无需手动调用 `Tracer.trace(ex)` 来记录业务异常。Sentinel 1.4.0 以前的版本需要自行调用 `Tracer.trace(ex)` 来记录业务异常。

### 3.8.2 配置

**AspectJ**

若您的应用直接使用了 AspectJ，那么您需要在 `aop.xml` 文件中引入对应的 Aspect：

```xml
<aspects>
    <aspect name="com.alibaba.csp.sentinel.annotation.aspectj.SentinelResourceAspect"/>
</aspects>
```

**Spring AOP**

若您的应用使用了 Spring AOP，您需要通过配置的方式将 `SentinelResourceAspect` 注册为一个 Spring Bean：

```java
@Configuration
public class SentinelAspectConfiguration {

    @Bean
    public SentinelResourceAspect sentinelResourceAspect() {
        return new SentinelResourceAspect();
    }
}
```

我们提供了 Spring AOP 的示例，可以参见 [sentinel-demo-annotation-spring-aop](https://github.com/alibaba/Sentinel/tree/master/sentinel-demo/sentinel-demo-annotation-spring-aop)。er/sentinel-demo/sentinel-demo-annotation-spring-aop)。

## 3.9 动态规则扩展

### 3.9.1 规则

Sentinel 的理念是开发者只需要关注资源的定义，当资源定义成功后可以动态增加各种流控降级规则。Sentinel 提供两种方式修改规则：

- 通过 API 直接修改 (`loadRules`)
- 通过 `DataSource` 适配不同数据源修改

通过 API 修改比较直观，可以通过以下几个 API 修改不同的规则：

```Java
FlowRuleManager.loadRules(List<FlowRule> rules); // 修改流控规则
DegradeRuleManager.loadRules(List<DegradeRule> rules); // 修改降级规则
```

手动修改规则（硬编码方式）一般仅用于测试和演示，生产上一般通过动态规则源的方式来动态管理规则。

### 3.9.2 DataSource 扩展

上述 `loadRules()` 方法只接受内存态的规则对象，但更多时候规则存储在文件、数据库或者配置中心当中。`DataSource` 接口给我们提供了对接任意配置源的能力。相比直接通过 API 修改规则，实现 `DataSource` 接口是更加可靠的做法。

我们推荐**通过控制台设置规则后将规则推送到统一的规则中心，客户端实现** `ReadableDataSource` **接口端监听规则中心实时获取变更**，流程如下：

![push-rules-from-dashboard-to-config-center](https://user-images.githubusercontent.com/9434884/45406233-645e8380-b698-11e8-8199-0c917403238f.png)

`DataSource` 扩展常见的实现方式有:

- **拉模式**：客户端主动向某个规则管理中心定期轮询拉取规则，这个规则中心可以是 RDBMS、文件，甚至是 VCS 等。这样做的方式是简单，缺点是无法及时获取变更；
- **推模式**：规则中心统一推送，客户端通过注册监听器的方式时刻监听变化，比如使用 [Nacos](https://github.com/alibaba/nacos)、Zookeeper 等配置中心。这种方式有更好的实时性和一致性保证。

Sentinel 目前支持以下数据源扩展：

- Pull-based: 动态文件数据源、[Consul](https://github.com/alibaba/Sentinel/tree/master/sentinel-extension/sentinel-datasource-consul), [Eureka](https://github.com/alibaba/Sentinel/tree/master/sentinel-extension/sentinel-datasource-eureka)
- Push-based: [ZooKeeper](https://github.com/alibaba/Sentinel/tree/master/sentinel-extension/sentinel-datasource-zookeeper), [Redis](https://github.com/alibaba/Sentinel/tree/master/sentinel-extension/sentinel-datasource-redis), [Nacos](https://github.com/alibaba/Sentinel/tree/master/sentinel-extension/sentinel-datasource-nacos), [Apollo](https://github.com/alibaba/Sentinel/tree/master/sentinel-extension/sentinel-datasource-apollo), [etcd](https://github.com/alibaba/Sentinel/tree/master/sentinel-extension/sentinel-datasource-etcd)

流量治理标准数据源：[OpenSergo](https://sentinelguard.io/zh-cn/docs/opensergo-data-source.html)

####  拉模式扩展

实现拉模式的数据源最简单的方式是继承 [`AutoRefreshDataSource`](https://github.com/alibaba/Sentinel/blob/master/sentinel-extension/sentinel-datasource-extension/src/main/java/com/alibaba/csp/sentinel/datasource/AutoRefreshDataSource.java) 抽象类，然后实现 `readSource()` 方法，在该方法里从指定数据源读取字符串格式的配置数据。比如 [基于文件的数据源](https://github.com/alibaba/Sentinel/blob/master/sentinel-demo/sentinel-demo-dynamic-file-rule/src/main/java/com/alibaba/csp/sentinel/demo/file/rule/FileDataSourceDemo.java)。

#### 推模式扩展

实现推模式的数据源最简单的方式是继承 [`AbstractDataSource`](https://github.com/alibaba/Sentinel/blob/master/sentinel-extension/sentinel-datasource-extension/src/main/java/com/alibaba/csp/sentinel/datasource/AbstractDataSource.java) 抽象类，在其构造方法中添加监听器，并实现 `readSource()` 从指定数据源读取字符串格式的配置数据。比如 [基于 Nacos 的数据源](https://github.com/alibaba/Sentinel/blob/master/sentinel-extension/sentinel-datasource-nacos/src/main/java/com/alibaba/csp/sentinel/datasource/nacos/NacosDataSource.java)。

#### 注册数据源

通常需要调用以下方法将数据源注册至指定的规则管理器中：

```java
ReadableDataSource<String, List<FlowRule>> flowRuleDataSource = new NacosDataSource<>(remoteAddress, groupId, dataId, parser);
FlowRuleManager.register2Property(flowRuleDataSource.getProperty());
```

若不希望手动注册数据源，可以借助 Sentinel 的 `InitFunc` SPI 扩展接口。只需要实现自己的 `InitFunc` 接口，在 `init` 方法中编写注册数据源的逻辑。比如：

```java
package com.test.init;

public class DataSourceInitFunc implements InitFunc {

    @Override
    public void init() throws Exception {
        final String remoteAddress = "localhost";
        final String groupId = "Sentinel:Demo";
        final String dataId = "com.alibaba.csp.sentinel.demo.flow.rule";

        ReadableDataSource<String, List<FlowRule>> flowRuleDataSource = new NacosDataSource<>(remoteAddress, groupId, dataId,
            source -> JSON.parseObject(source, new TypeReference<List<FlowRule>>() {}));
        FlowRuleManager.register2Property(flowRuleDataSource.getProperty());
    }
}
```

接着将对应的类名添加到位于资源目录（通常是 `resource` 目录）下的 `META-INF/services` 目录下的 `com.alibaba.csp.sentinel.init.InitFunc` 文件中，比如：

```
com.test.init.DataSourceInitFunc
```

这样，当初次访问任意资源的时候，Sentinel 就可以自动去注册对应的数据源了。

### 3.9.3 示例

#### API 模式：使用客户端规则 API 配置规则

[Sentinel Dashboard](https://sentinelguard.io/zh-cn/docs/dashboard.html) 通过 Sentinel 客户端自带的规则 API 来实时查询和更改内存中的规则。

注意: 要使客户端具备规则 API，需在客户端引入以下依赖：

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentienl-http-simple-transport</artifactId>
    <version>x.y.z</version>
</dependency>
```

#### 拉模式：使用文件配置规则

[这个示例](https://github.com/alibaba/Sentinel/blob/master/sentinel-demo/sentinel-demo-dynamic-file-rule/src/main/java/com/alibaba/csp/sentinel/demo/file/rule/FileDataSourceDemo.java)展示 Sentinel 是如何从文件获取规则信息的。[`FileRefreshableDataSource`](https://github.com/alibaba/Sentinel/blob/master/sentinel-extension/sentinel-datasource-extension/src/main/java/com/alibaba/csp/sentinel/datasource/FileRefreshableDataSource.java) 会周期性的读取文件以获取规则，当文件有更新时会及时发现，并将规则更新到内存中。使用时只需添加以下依赖：

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-extension</artifactId>
    <version>x.y.z</version>
</dependency>
```

#### 推模式：使用 Nacos 配置规则

[Nacos](https://github.com/alibaba/Nacos) 是阿里中间件团队开源的服务发现和动态配置中心。Sentinel 针对 Nacos 作了适配，底层可以采用 Nacos 作为规则配置数据源。使用时只需添加以下依赖：

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-nacos</artifactId>
    <version>x.y.z</version>
</dependency>
```

然后创建 `NacosDataSource` 并将其注册至对应的 RuleManager 上即可。比如：

```java
// remoteAddress 代表 Nacos 服务端的地址
// groupId 和 dataId 对应 Nacos 中相应配置
ReadableDataSource<String, List<FlowRule>> flowRuleDataSource = new NacosDataSource<>(remoteAddress, groupId, dataId,
    source -> JSON.parseObject(source, new TypeReference<List<FlowRule>>() {}));
FlowRuleManager.register2Property(flowRuleDataSource.getProperty());
```

详细示例可以参见 [sentinel-demo-nacos-datasource](https://github.com/alibaba/Sentinel/tree/master/sentinel-demo/sentinel-demo-nacos-datasource)。

#### 推模式：使用 ZooKeeper 配置规则

Sentinel 针对 ZooKeeper 作了相应适配，底层可以采用 ZooKeeper 作为规则配置数据源。使用时只需添加以下依赖：

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-zookeeper</artifactId>
    <version>x.y.z</version>
</dependency>
```

然后创建 `ZookeeperDataSource` 并将其注册至对应的 RuleManager 上即可。比如：

```java
// remoteAddress 代表 ZooKeeper 服务端的地址
// path 对应 ZK 中的数据路径
ReadableDataSource<String, List<FlowRule>> flowRuleDataSource = new ZookeeperDataSource<>(remoteAddress, path, source -> JSON.parseObject(source, new TypeReference<List<FlowRule>>() {}));
FlowRuleManager.register2Property(flowRuleDataSource.getProperty());
```

详细示例可以参见 [sentinel-demo-zookeeper-datasource](https://github.com/alibaba/Sentinel/tree/master/sentinel-demo/sentinel-demo-zookeeper-datasource)。

#### 推模式：使用 Apollo 配置规则

Sentinel 针对 [Apollo](https://github.com/ctripcorp/apollo) 作了相应适配，底层可以采用 Apollo 作为规则配置数据源。使用时只需添加以下依赖：

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-apollo</artifactId>
    <version>x.y.z</version>
</dependency>
```

然后创建 `ApolloDataSource` 并将其注册至对应的 RuleManager 上即可。比如：

```java
// namespaceName 对应 Apollo 的命名空间名称
// ruleKey 对应规则存储的 key
// defaultRules 对应连接不上 Apollo 时的默认规则
ReadableDataSource<String, List<FlowRule>> flowRuleDataSource = new ApolloDataSource<>(namespaceName, ruleKey, defaultRules, source -> JSON.parseObject(source, new TypeReference<List<FlowRule>>() {}));
FlowRuleManager.register2Property(flowRuleDataSource.getProperty());
```

详细示例可以参见 [sentinel-demo-apollo-datasource](https://github.com/alibaba/Sentinel/tree/master/sentinel-demo/sentinel-demo-apollo-datasource)。

## 3.10 日志

Sentinel 日志目录可通过 `csp.sentinel.log.dir` 启动参数进行配置，详情请参考[通用配置项文档](https://sentinelguard.io/zh-cn/docs/general-configuration.html)。

### 3.10.1 拦截详情日志（block 日志）

无论触发了限流、熔断降级还是系统保护，它们的秒级拦截详情日志都在 `${user_home}/logs/csp/sentinel-block.log`里。如果没有发生拦截，则该日志不会出现。日志格式如下:

```
2014-06-20 16:35:10|1|sayHello(java.lang.String,long),FlowException,default,origin|61,0
2014-06-20 16:35:11|1|sayHello(java.lang.String,long),FlowException,default,origin|1,0
```

日志含义：

| index |               例子                | 说明                                                         |
| :---- | :-------------------------------: | :----------------------------------------------------------- |
| 1     |       `2014-06-20 16:35:10`       | 时间戳                                                       |
| 2     |                `1`                | 该秒发生的第一个资源                                         |
| 3     | `sayHello(java.lang.String,long)` | 资源名称                                                     |
| 4     |          `XXXException`           | 拦截的原因, 通常 `FlowException` 代表是被限流规则拦截，`DegradeException` 则表示被降级，`SystemBlockException` 则表示被系统保护拦截 |
| 5     |             `default`             | 生效规则的调用来源（参数限流中代表生效的参数）               |
| 6     |             `origin`              | 被拦截资源的调用者，可以为空                                 |
| 7     |              `61,0`               | 61 被拦截的数量，０无意义可忽略                              |

### 3.10.2 秒级监控日志

所有的资源访问都会产生秒级监控日志，日志文件默认为 `${user_home}/logs/csp/${app_name}-${pid}-metrics.log`（会随时间滚动）。格式如下:

```
1532415661000|2018-07-24 15:01:01|sayHello(java.lang.String)|12|3|4|2|295|0|0|1
```

1. `1532415661000`：时间戳
2. `2018-07-24 15:01:01`：格式化之后的时间戳
3. `sayHello(java.lang.String)`：资源名
4. `12`：表示到来的数量，即此刻通过 Sentinel 规则 check 的数量（passed QPS）
5. `3`：实际该资源被拦截的数量（blocked QPS）
6. `4`：每秒结束的资源个数（完成调用），包括正常结束和异常结束的情况（exit QPS）
7. `2`：异常的数量
8. `295`：资源的平均响应时间（RT）
9. `0`: 该秒占用未来请求的数目（since 1.5.0）
10. `0`: 最大并发数（预留用）
11. `1`: 资源分类（since 1.7.0）

### 3.10.3 业务日志

其它的日志在 `${user_home}/logs/csp/sentinel-record.log.xxx` 里。该日志包含规则的推送、接收、处理等记录，排查问题的时候会非常有帮助。

### 3.10.4 集群限流日志

- `${log_dir}/sentinel-cluster-client.log`：Token Client 日志，会记录请求失败的信息

### 3.10.5 SPI 扩展机制

1.7.2 版本开始，Sentinel 支持 Logger 扩展机制，可以实现自定义的 Logger SPI 来将 record log 等日志自行处理。metric/block log 暂不支持定制。

## 3.11 实时监控

Sentinel 提供对所有资源的实时监控。如果需要实时监控，客户端需引入以下依赖（以 Maven 为例）：

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-transport-simple-http</artifactId>
    <version>x.y.z</version>
</dependency>
```

引入上述依赖后，客户端便会主动连接 Sentinel 控制台。通过 [Sentinel 控制台](https://sentinelguard.io/zh-cn/docs/dashboard.html) 即可查看客户端的实时监控。

通常您并不需要关心以下 API，但是如果您对开发控制台感兴趣，以下为监控 API 的文档。

### 3.11.1 簇点监控

#### 获取簇点列表

相关 API: `GET /clusterNode`

当应用启动之后，可以运行下列命令，获得当前所有簇点（`ClusterNode`）的列表（JSON 格式）：

```bash
curl http://localhost:8719/clusterNode
```

结果示例：

```javascript
[
 {"avgRt":0.0, //平均响应时间
 "blockRequest":0, //每分钟拦截的请求个数
 "blockedQps":0.0, //每秒拦截个数
 "curThreadNum":0, //并发个数
 "passQps":1.0, // 每秒成功通过请求
 "passReqQps":1.0, //每秒到来的请求
 "resourceName":"/registry/machine", 资源名称
 "timeStamp":1529905824134, //时间戳
 "totalQps":1.0, // 每分钟请求数
 "totalRequest":193}, 
  ....
]
```

#### 查询某个簇点的详细信息

可以用下面命令模糊查询该簇点的具体信息，其中 `id` 对应 resource name，支持模糊查询：

```bash
curl http://localhost:8719/cnode?id=xxxx
```

结果示例：

```shell
idx id                                thread    pass      blocked   success    total    aRt   1m-pass   1m-block   1m-all   exeption   
6   /app/aliswitch2/machines.json     0         0         0         0          0        0     0         0          0        0          
7   /app/sentinel-admin/machines.json 0         1         0         1          1        6     0         0          0        0          
8   /identity/machine.json            0         0         0         0          0        0     0         0          0        0          
9   /registry/machine                 0         2         0         2          2        1     192       0          192      0          
10  /app/views/machine.html           0         1         0         1          1        2     0         0          0        0   
```

#### 簇点调用者统计信息

可以用下列命令查询该簇点的调用者统计信息：

```bash
curl http://localhost:8719/origin?id=xxxx
```

结果示例：

```
id: nodeA
idx origin  threadNum passedQps blockedQps totalQps aRt   1m-passed 1m-blocked 1m-total 
1   caller1 0         0         0          0        0     0         0          0        
2   caller2 0         0         0          0        0     0         0          0      
```

其中的 origin 由 `ContextUtil.enter(resourceName，origin)` 方法中的 `origin` 指定。

### 3.11.2 链路监控

我们可以通过命令　`curl http://localhost:8719/tree` 来查询链路入口的链路树形结构：

```
EntranceNode: machine-root(t:0 pq:1 bq:0 tq:1 rt:0 prq:1 1mp:0 1mb:0 1mt:0)
-EntranceNode1: Entrance1(t:0 pq:1 bq:0 tq:1 rt:0 prq:1 1mp:0 1mb:0 1mt:0)
--nodeA(t:0 pq:1 bq:0 tq:1 rt:0 prq:1 1mp:0 1mb:0 1mt:0)
-EntranceNode2: Entrance1(t:0 pq:1 bq:0 tq:1 rt:0 prq:1 1mp:0 1mb:0 1mt:0)
--nodeA(t:0 pq:1 bq:0 tq:1 rt:0 prq:1 1mp:0 1mb:0 1mt:0)

t:threadNum  pq:passQps  bq:blockedQps  tq:totalQps  rt:averageRt  prq: passRequestQps 1mp:1m-passed 1mb:1m-blocked 1mt:1m-total
```

### 3.11.3 历史资源数据

#### 资源的秒级日志

所有资源的秒级日志在 `${home}/logs/csp/${appName}-${pid}-metrics.log.${date}.xx`。例如，该日志的名字可能为 `app-3518-metrics.log.2018-06-22.1`

```
1529573107000|2018-06-21 17:25:07|sayHello(java.lang.String,long)|10|3601|10|0|2
```

| index |               例子                | 说明                                             |
| :---- | :-------------------------------: | :----------------------------------------------- |
| 1     |          `1529573107000`          | 时间戳                                           |
| 2     |       `2018-06-21 17:25:07`       | 日期                                             |
| 3     | `sayHello(java.lang.String,long)` | 资源名称                                         |
| 4     |               `10`                | **每秒通过的资源请求个数**                       |
| 5     |              `3601`               | **每秒资源被拦截的个数**                         |
| 6     |               `10`                | 每秒结束的资源个数，包括正常结束和异常结束的情况 |
| 7     |                `0`                | 每秒资源的异常个数                               |
| 8     |                `2`                | 资源平均响应时间                                 |

#### 被拦截的秒级日志

同样的，每秒的拦截日志也会出现在 `<用户目录>/logs/csp/sentinel-block.log` 文件下。如果没有发生拦截，则该日志不会出现。

```
2014-06-20 16:35:10|1|sayHello(java.lang.String,long),FlowException,default,origin|61,0
2014-06-20 16:35:11|1|sayHello(java.lang.String,long),FlowException,default,origin|1,0
```

| index |               例子                | 说明                                                         |
| :---- | :-------------------------------: | :----------------------------------------------------------- |
| 1     |       `2014-06-20 16:35:10`       | 时间戳                                                       |
| 2     |                `1`                | 该秒发生的第一个资源                                         |
| 3     | `sayHello(java.lang.String,long)` | 资源名称                                                     |
| 4     |          `XXXException`           | 拦截的原因, 通常 `FlowException` 代表是被限流规则拦截, `DegradeException` 则表示被降级，`SystemException` 则表示被系统保护拦截 |
| 5     |             `default`             | 生效规则的调用应用                                           |
| 6     |             `origin`              | 被拦截资源的调用者。可以为空                                 |
| 7     |              `61,0`               | 61 被拦截的数量，０则代表可以忽略                            |

#### 实时查询

相关 API: `GET /metric`

```shell
curl http://localhost:8719/metric?identity=XXX&startTime=XXXX&endTime=XXXX&maxLines=XXXX
```

需指定以下 URL 参数：

- `identity`：资源名称
- `startTime`：开始时间（时间戳）
- `endTime`：结束时间
- `maxLines`：监控数据最大行数

返回和 [资源的秒级日志](https://sentinelguard.io/zh-cn/docs/logs.html) 格式一样的内容。例如：

```
1529998904000|2018-06-26 15:41:44|abc|100|0|0|0|0
1529998905000|2018-06-26 15:41:45|abc|4|5579|104|0|728
1529998906000|2018-06-26 15:41:46|abc|0|15698|0|0|0
1529998907000|2018-06-26 15:41:47|abc|0|19262|0|0|0
1529998908000|2018-06-26 15:41:48|abc|0|19502|0|0|0
1529998909000|2018-06-26 15:41:49|abc|0|18386|0|0|0
1529998910000|2018-06-26 15:41:50|abc|0|19189|0|0|0
1529998911000|2018-06-26 15:41:51|abc|0|16543|0|0|0
1529998912000|2018-06-26 15:41:52|abc|0|18471|0|0|0
1529998913000|2018-06-26 15:41:53|abc|0|19405|0|0|0
```

## 3.12 启动配置项

### 3.12.1 配置方式

Sentinel 提供如下的配置方式：

- JVM -D 参数方式
- properties 文件方式（1.7.0 版本开始支持）

其中，`project.name` 参数只能通过 JVM -D 参数方式配置（since 1.8.0 取消该限制），其它参数支持所有的配置方式。

**优先级顺序**：JVM -D 参数的优先级最高。若 properties 和 JVM 参数中有相同项的配置，以 JVM 参数配置的为准。

用户可以通过 `-Dcsp.sentinel.config.file` 参数配置 properties 文件的路径，支持 classpath 路径配置（如 `classpath:sentinel.properties`）。默认 Sentinel 会尝试从 `classpath:sentinel.properties` 文件读取配置，读取编码默认为 UTF-8。

> 注：1.7.0 以下版本可以通过旧的 `${user_home}/logs/csp/${project.name}.properties` 配置文件进行配置（除 `project.name` 和日志相关配置项）。

> 注：若您的应用为 Spring Boot 或 Spring Cloud 应用，您可以使用 [Spring Cloud Alibaba](https://github.com/alibaba/spring-cloud-alibaba)，通过 Spring 配置文件来指定配置，详情请参考 [Spring Cloud Alibaba Sentinel 文档](https://github.com/alibaba/spring-cloud-alibaba/wiki/Sentinel)。

### 3.12.2 配置项列表

#### sentinel-core 的配置项

**基础配置项**

| 名称                                   | 含义                                                     | 类型     | 默认值                | 是否必需 | 备注                                                         |
| -------------------------------------- | -------------------------------------------------------- | -------- | --------------------- | -------- | ------------------------------------------------------------ |
| `project.name`                         | 指定应用的名称                                           | `String` | `null`                | 否       |                                                              |
| `csp.sentinel.app.type`                | 指定应用的类型                                           | `int`    | 0 (`APP_TYPE_COMMON`) | 否       | 1.6.0 引入                                                   |
| `csp.sentinel.metric.file.single.size` | 单个监控日志文件的大小                                   | `long`   | 52428800 (50MB)       | 否       |                                                              |
| `csp.sentinel.metric.file.total.count` | 监控日志文件的总数上限                                   | `int`    | 6                     | 否       |                                                              |
| `csp.sentinel.statistic.max.rt`        | 最大的有效响应时长（ms），超出此值则按照此值记录         | `int`    | 4900                  | 否       | 1.4.1 引入                                                   |
| `csp.sentinel.spi.classloader`         | SPI 加载时使用的 ClassLoader，默认为给定类的 ClassLoader | `String` | `default`             | 否       | 若配置 `context` 则使用 thread context ClassLoader。1.7.0 引入 |

其中 `project.name` 项用于指定应用名（appName）。若未指定，则默认解析 main 函数的类名作为应用名。**实际项目使用中建议手动指定应用名**。

**日志相关配置项**

| 名称                           | 含义                                                         | 类型      | 默认值                   | 是否必需 | 备注       |
| ------------------------------ | ------------------------------------------------------------ | --------- | ------------------------ | -------- | ---------- |
| `csp.sentinel.log.dir`         | Sentinel 日志文件目录                                        | `String`  | `${user.home}/logs/csp/` | 否       | 1.3.0 引入 |
| `csp.sentinel.log.use.pid`     | 日志文件名中是否加入进程号，用于单机部署多个应用的情况       | `boolean` | `false`                  | 否       | 1.3.0 引入 |
| `csp.sentinel.log.output.type` | Record 日志输出的类型，`file` 代表输出至文件，`console` 代表输出至终端 | `String`  | `file`                   | 否       | 1.6.2 引入 |

> **注意**：若需要在单台机器上运行相同服务的多个实例，则需要加入 `-Dcsp.sentinel.log.use.pid=true` 来保证不同实例日志的独立性。

#### sentinel-transport-common 的配置项

| 名称                                 | 含义                                                         | 类型     | 默认值 | 是否必需                                                     |
| ------------------------------------ | ------------------------------------------------------------ | -------- | ------ | ------------------------------------------------------------ |
| `csp.sentinel.dashboard.server`      | 控制台的地址，指定控制台后客户端会自动向该地址发送心跳包。地址格式为：`hostIp:port` | `String` | `null` | 是                                                           |
| `csp.sentinel.heartbeat.interval.ms` | 心跳包发送周期，单位毫秒                                     | `long`   | `null` | 非必需，若不进行配置，则会从相应的 `HeartbeatSender` 中提取默认值 |
| `csp.sentinel.api.port`              | 本地启动 HTTP API Server 的端口号                            | `int`    | 8719   | 否                                                           |
| `csp.sentinel.heartbeat.client.ip`   | 指定心跳包中本机的 IP                                        | `String` | -      | 若不指定则通过 `HostNameUtil` 解析；该配置项多用于多网卡环境 |

> 注：`csp.sentinel.api.port` 可不提供，默认为 8719，若端口冲突会自动向下探测可用的端口。

# 4. Sentinel控制台

> 需要的时候查查文档就好了

# 5. OpenSergo 动态规则源

Sentinel 2.0 将作为 [OpenSergo 流量治理](https://opensergo.io/zh-cn/)的标准实现，原生支持 OpenSergo 流量治理相关 CRD 配置及能力，结合 Sentinel 提供的各框架的适配模块，让 Dubbo, Spring Cloud Alibaba, gRPC, CloudWeGo 等20+框架能够无缝接入到 OpenSergo 生态中，用统一的 CRD 来配置流量路由、流控降级、服务容错等治理规则。无论是 Java 还是 Go 还是 Mesh 服务，无论是 HTTP 请求还是 RPC 调用，还是数据库 SQL 访问，用户都可以用统一的容错治理规则 CRD 来给微服务架构中的每一环配置治理，来保障服务链路的稳定性。

Sentinel 社区提供对接 OpenSergo spec 的动态数据源模块 [`sentinel-datasource-opensergo`](https://github.com/alibaba/Sentinel/tree/master/sentinel-extension/sentinel-datasource-opensergo)，只需要按照 Sentinel 数据源的方式接入即可。

> 注意：该适配模块目前还在 beta 阶段。目前 Sentinel OpenSergo 数据源支持流控规则、匀速排队规则以及熔断规则。

![image](https://sentinelguard.io/img/opensergo/sentinel-opensergo-datasource-arch.jpg)

以 Java 版本为例，首先 Maven 引入依赖：

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-opensergo</artifactId>
    <!-- 对应 Sentinel 1.8.6 版本 -->
    <version>0.1.0-beta</version>
</dependency>
```

然后在项目合适的位置（如 Spring 初始化 hook 或 Sentinel `InitFunc` 中）中创建并注册 Sentinel OpenSergo 数据源。在应用启动前，确保 OpenSergo 控制面及 CRD 已经部署在 Kubernetes 集群中，可以参考[控制面快速部署文档](https://opensergo.io/zh-cn/docs/quick-start/opensergo-control-plane/)。

```java
// 传入 OpenSergo Control Plane 的 endpoint，以及希望监听的应用名.
// 在我们的例子中，假定应用名为 foo-app
OpenSergoDataSourceGroup openSergo = new OpenSergoDataSourceGroup("localhost", 10246, "default", "foo-app");
// 初始化 OpenSergo 数据源
openSergo.start();

// 订阅 OpenSergo 流控规则，并注册数据源到 Sentinel 流控规则数据源中
FlowRuleManager.register2Property(openSergo.subscribeFlowRules());
```

启动应用后，即可编写 [FaultToleranceRule、RateLimitStrategy 等 CR YAML](https://github.com/opensergo/opensergo-specification/blob/main/specification/zh-Hans/fault-tolerance.md) 来动态配置流控容错规则，通过 kubectl apply 到集群中即可生效。

Sentinel 社区提供了一个 [Spring Boot 应用接入 OpenSergo 数据源的 demo](https://github.com/alibaba/Sentinel/blob/master/sentinel-demo/sentinel-demo-opensergo-datasource/README.zh-cn.md)，可作参考。

# 6. 开源框架适配

- 云原生微服务体系
  - Spring Boot/Spring Cloud
  - Quarkus
- Web 适配
  - Web Servlet
  - Spring Web
  - Spring WebFlux
  - JAX-RS (Java EE)
- RPC 适配
  - Apache Dubbo
  - gRPC
  - Feign
  - SOFARPC
- HTTP client 适配
  - Apache HttpClient
  - OkHttp
- Reactive 适配
  - Reactor
- API Gateway 适配
  - Spring Cloud Gateway
  - Netflix Zuul 1.x
  - Netflix Zuul 2.x
- Apache RocketMQ

> **注：适配模块仅提供相应适配功能，若希望接入 Sentinel 控制台，请务必参考 [Sentinel 控制台文档](https://sentinelguard.io/zh-cn/docs/dashboard.html)。**
