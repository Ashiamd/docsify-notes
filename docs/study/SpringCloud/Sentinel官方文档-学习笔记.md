# Sentinel官方文档-学习笔记

> [Sentinel中文官方文档](https://sentinelguard.io/zh-cn/docs/introduction.html) <= 下面内容基本都摘自官方文档。

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
