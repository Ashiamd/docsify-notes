# Flink学习笔记

> [Apache Flink® — Stateful Computations over Data Streams](https://flink.apache.org/)

# Try FLink

> [【笔记】Flink 官方教程 Section1 Try Fink](https://blog.csdn.net/m0_37809890/article/details/107933643)

## Local Installation

> [Flink不同版本间的编译区别](https://www.jianshu.com/p/2bccee5c2c8b)
>
> **Flink1.11.0版本之后，Flink官方为了让Flink变得Hadoop Free，现在能支持hadoop2和hadoop3，同时可以指定不同的Hadoop环境。**

Follow these few steps to download the latest stable versions and get started.

## Step 1: Download

To be able to run Flink, the only requirement is to have a working **Java 8 or 11** installation. You can check the correct installation of Java by issuing the following command:

```shell
java -version
```

[Download](https://flink.apache.org/downloads.html) the 1.12.0 release and un-tar it.

```shell
$ tar -xzf flink-1.12.0-bin-scala_2.11.tgz
$ cd flink-1.12.0-bin-scala_2.11
```

## Step 2: Start a Cluster

Flink ships with a single bash script to start a local cluster.

```shell
$ ./bin/start-cluster.sh
Starting cluster.
Starting standalonesession daemon on host.
Starting taskexecutor daemon on host.
```

## Step 3: Submit a Job

Releases of Flink come with a number of example Jobs. You can quickly deploy one of these applications to the running cluster.

```
$ ./bin/flink run examples/streaming/WordCount.jar
$ tail log/flink-*-taskexecutor-*.out
  (to,1)
  (be,1)
  (or,1)
  (not,1)
  (to,2)
  (be,2)
```

Additionally, you can check Flink’s [Web UI](http://localhost:8081/) to monitor the status of the Cluster and running Job.

## Step 4: Stop the Cluster

When you are finished you can quickly stop the cluster and all running components.

```
$ ./bin/stop-cluster.sh
```

# 基于 DataStream API 实现欺诈检测

​	Apache Flink 提供了 DataStream API 来实现稳定可靠的、有状态的流处理应用程序。 Flink 支持对状态和时间的细粒度控制，以此来实现复杂的事件驱动数据处理系统。 这个入门指导手册讲述了如何通过 Flink DataStream API 来实现一个有状态流处理程序。

## 你要搭建一个什么系统

​	在当今数字时代，信用卡欺诈行为越来越被重视。 罪犯可以通过诈骗或者入侵安全级别较低系统来盗窃信用卡卡号。 用盗得的信用卡进行很小额度的例如一美元或者更小额度的消费进行测试。 如果测试消费成功，那么他们就会用这个信用卡进行大笔消费，来购买一些他们希望得到的，或者可以倒卖的财物。

​	在这个教程中，你将会建立一个针对可疑信用卡交易行为的反欺诈检测系统。 通过使用一组简单的规则，你将了解到 Flink 如何为我们实现复杂业务逻辑并实时执行。

## 准备条件

​	这个代码练习假定你对 Java 或 Scala 有一定的了解，当然，如果你之前使用的是其他开发语言，你也应该能够跟随本教程进行学习。

## 困难求助

​	如果遇到困难，可以参考 [社区支持资源](https://flink.apache.org/zh/gettinghelp.html)。 当然也可以在邮件列表提问，Flink 的 [用户邮件列表](https://flink.apache.org/zh/community.html#mailing-lists) 一直被评为所有Apache项目中最活跃的一个，这也是快速获得帮助的好方法。

## 怎样跟着教程练习

​	首先，你需要在你的电脑上准备以下环境：

- Java 8 or 11
- Maven

​	一个准备好的 Flink Maven Archetype 能够快速创建一个包含了必要依赖的 Flink 程序骨架，基于此，你可以把精力集中在编写业务逻辑上即可。 这些已包含的依赖包括 `flink-streaming-java`、`flink-walkthrough-common` 等，他们分别是 Flink 应用程序的核心依赖项和这个代码练习需要的数据生成器，当然还包括其他本代码练习所依赖的类。

> **说明:** 为简洁起见，本练习中的代码块中可能不包含完整的类路径。完整的类路径可以在文档底部 [链接](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/try-flink/datastream_api.html#final-application) 中找到。

```shell
$ mvn archetype:generate \
    -DarchetypeGroupId=org.apache.flink \
    -DarchetypeArtifactId=flink-walkthrough-datastream-java \
    -DarchetypeVersion=1.12.0 \
    -DgroupId=frauddetection \
    -DartifactId=frauddetection \
    -Dversion=0.1 \
    -Dpackage=spendreport \
    -DinteractiveMode=false
```

​	你可以根据自己的情况修改 `groupId`、 `artifactId` 和 `package`。通过这三个参数， Maven 将会创建一个名为 `frauddetection` 的文件夹，包含了所有依赖的整个工程项目将会位于该文件夹下。 将工程目录导入到你的开发环境之后，你可以找到 `FraudDetectionJob.java` （或 `FraudDetectionJob.scala`） 代码文件，文件中的代码如下所示。你可以在 IDE 中直接运行这个文件。 同时，你可以试着在数据流中设置一些断点或者以 DEBUG 模式来运行程序，体验 Flink 是如何运行的。

### FraudDetectionJob.java

> [运行flink程序出现：ERROR: A JNI error has occurred, pLease check your installation and try again](https://www.cnblogs.com/zlshtml/p/13796793.html)

```java
package spendreport;

import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.walkthrough.common.sink.AlertSink;
import org.apache.flink.walkthrough.common.entity.Alert;
import org.apache.flink.walkthrough.common.entity.Transaction;
import org.apache.flink.walkthrough.common.source.TransactionSource;

public class FraudDetectionJob {

    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        DataStream<Transaction> transactions = env
            .addSource(new TransactionSource())
            .name("transactions");

        DataStream<Alert> alerts = transactions
            .keyBy(Transaction::getAccountId)
            .process(new FraudDetector())
            .name("fraud-detector");

        alerts
            .addSink(new AlertSink())
            .name("send-alerts");

        env.execute("Fraud Detection");
    }
}
```

### FraudDetector.java

```java
package spendreport;

import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
import org.apache.flink.util.Collector;
import org.apache.flink.walkthrough.common.entity.Alert;
import org.apache.flink.walkthrough.common.entity.Transaction;

public class FraudDetector extends KeyedProcessFunction<Long, Transaction, Alert> {

    private static final long serialVersionUID = 1L;

    private static final double SMALL_AMOUNT = 1.00;
    private static final double LARGE_AMOUNT = 500.00;
    private static final long ONE_MINUTE = 60 * 1000;

    @Override
    public void processElement(
            Transaction transaction,
            Context context,
            Collector<Alert> collector) throws Exception {

        Alert alert = new Alert();
        alert.setId(transaction.getAccountId());

        collector.collect(alert);
    }
}
```

## 代码分析

​	让我们一步步地来分析一下这两个代码文件。`FraudDetectionJob` 类定义了程序的数据流，而 `FraudDetector` 类定义了欺诈交易检测的业务逻辑。

​	下面我们开始讲解整个 Job 是如何组装到 `FraudDetectionJob` 类的 `main` 函数中的。

### 执行环境

​	第一行的 `StreamExecutionEnvironment` 用于设置你的执行环境。 任务执行环境用于定义任务的属性、创建数据源以及最终启动任务的执行。

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
```

### 创建数据源

​	数据源从外部系统例如 Apache Kafka、Rabbit MQ 或者 Apache Pulsar 接收数据，然后将数据送到 Flink 程序中。 这个代码练习使用的是一个能够无限循环生成信用卡模拟交易数据的数据源。 每条交易数据包括了信用卡 ID （`accountId`），交易发生的时间 （`timestamp`） 以及交易的金额（`amount`）。 绑定到数据源上的 `name` 属性是为了调试方便，如果发生一些异常，我们能够通过它快速定位问题发生在哪里。

```java
DataStream<Transaction> transactions = env
    .addSource(new TransactionSource())
    .name("transactions");
```

### 对事件分区 & 欺诈检测

​	`transactions` 这个数据流包含了大量的用户交易数据，需要被划分到多个并发上进行欺诈检测处理。<u>由于欺诈行为的发生是基于某一个账户的，所以，必须要要保证同一个账户的所有交易行为数据要被同一个并发的 task 进行处理</u>。

​	为了保证同一个 task 处理同一个 key 的所有数据，你可以使用 `DataStream#keyBy` 对流进行分区。 **`process()` 函数对流绑定了一个操作，这个操作将会对流上的每一个消息调用所定义好的函数**。 通常，一个操作会紧跟着 `keyBy` 被调用，在这个例子中，这个操作是`FraudDetector`，该操作是在一个 *keyed context* 上执行的。

```java
DataStream<Alert> alerts = transactions
    .keyBy(Transaction::getAccountId)
    .process(new FraudDetector())
    .name("fraud-detector");
```

### 输出结果

​	sink 会将 `DataStream` 写出到外部系统，例如 Apache Kafka、Cassandra 或者 AWS Kinesis 等。 `AlertSink` 使用 **INFO** 的日志级别打印每一个 `Alert` 的数据记录，而不是将其写入持久存储，以便你可以方便地查看结果。

```java
alerts.addSink(new AlertSink());
```

### 运行作业

​	**Flink 程序是懒加载的，并且只有在完全搭建好之后，才能够发布到集群上执行**。 调用 `StreamExecutionEnvironment#execute` 时给任务传递一个任务名参数，就可以开始运行任务。

```java
env.execute("Fraud Detection");
```

### 欺诈检测器

​	欺诈检查类 `FraudDetector` 是 `KeyedProcessFunction` 接口的一个实现。 他的方法 `KeyedProcessFunction#processElement` 将会在每个交易事件上被调用。 这个程序里边会对每笔交易发出警报，有人可能会说这做法过于保守了。

​	本教程的后续步骤将指导你对这个欺诈检测器进行更有意义的业务逻辑扩展。

```java
public class FraudDetector extends KeyedProcessFunction<Long, Transaction, Alert> {

    private static final double SMALL_AMOUNT = 1.00;
    private static final double LARGE_AMOUNT = 500.00;
    private static final long ONE_MINUTE = 60 * 1000;

    @Override
    public void processElement(
            Transaction transaction,
            Context context,
            Collector<Alert> collector) throws Exception {

        Alert alert = new Alert();
        alert.setId(transaction.getAccountId());

        collector.collect(alert);
    }
}
```

## 实现一个真正的应用程序

​	我们先实现第一版报警程序，对于一个账户，如果出现小于 $1 美元的交易后紧跟着一个大于 $500 的交易，就输出一个报警信息。

​	假设你的欺诈检测器所处理的交易数据如下：

![Transactions](https://ci.apache.org/projects/flink/flink-docs-release-1.12/fig/fraud-transactions.svg)

​	交易 3 和交易 4 应该被标记为欺诈行为，因为交易 3 是一个 $0.09 的小额交易，而紧随着的交易 4 是一个 $510 的大额交易。 另外，交易 7、8 和 交易 9 就不属于欺诈交易了，因为在交易 7 这个 $0.02 的小额交易之后，并没有跟随一个大额交易，而是一个金额适中的交易，这使得交易 7 到 交易 9 不属于欺诈行为。

​	欺诈检测器需要在多个交易事件之间记住一些信息。仅当一个大额的交易紧随一个小额交易的情况发生时，这个大额交易才被认为是欺诈交易。 在多个事件之间存储信息就需要使用到 [状态](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/concepts/glossary.html#managed-state)，这也是我们选择使用 [KeyedProcessFunction](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/stream/operators/process_function.html) 的原因。 它能够同时提供对状态和时间的细粒度操作，这使得我们能够在接下来的代码练习中实现更复杂的算法。

​	最直接的实现方式是使用一个 boolean 型的标记状态来表示是否刚处理过一个小额交易。 当处理到该账户的一个大额交易时，你只需要检查这个标记状态来确认上一个交易是是否小额交易即可。

​	<u>然而，仅使用一个标记作为 `FraudDetector` 的类成员来记录账户的上一个交易状态是不准确的。 Flink 会在同一个 `FraudDetector` 的并发实例中处理多个账户的交易数据，假设，当账户 A 和账户 B 的数据被分发的同一个并发实例上处理时，账户 A 的小额交易行为可能会将标记状态设置为真，随后账户 B 的大额交易可能会被误判为欺诈交易。 当然，我们可以使用如 `Map` 这样的数据结构来保存每一个账户的状态，但是常规的类成员变量是无法做到容错处理的，当任务失败重启后，之前的状态信息将会丢失。 这样的话，如果程序曾出现过失败重启的情况，将会漏掉一些欺诈报警。</u>

​	为了应对这个问题，**Flink 提供了一套支持容错状态的原语**，这些原语几乎与常规成员变量一样易于使用。

​	**Flink 中最基础的状态类型是 [ValueState](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/stream/state/state.html#using-managed-keyed-state)，这是一种能够为被其封装的变量添加容错能力的类型。**

​	 **`ValueState` 是一种 *keyed state*，也就是说它只能被用于 *keyed context* 提供的 operator 中，即所有能够紧随 `DataStream#keyBy` 之后被调用的operator**。 **一个 operator 中的 *keyed state* 的作用域默认是属于它所属的 key 的**。 这个例子中，key 就是当前正在处理的交易行为所属的信用卡账户（key 传入 keyBy() 函数调用），而 `FraudDetector` 维护了每个帐户的标记状态。 `ValueState` 需要使用 `ValueStateDescriptor` 来创建，`ValueStateDescriptor` 包含了 Flink 如何管理变量的一些元数据信息。状态在使用之前需要先被注册。 状态需要使用 `open()` 函数来注册状态。

```java
public class FraudDetector extends KeyedProcessFunction<Long, Transaction, Alert> {

    private static final long serialVersionUID = 1L;

    private transient ValueState<Boolean> flagState;

    @Override
    public void open(Configuration parameters) {
        ValueStateDescriptor<Boolean> flagDescriptor = new ValueStateDescriptor<>(
                "flag",
                Types.BOOLEAN);
        flagState = getRuntimeContext().getState(flagDescriptor);
    }
```

​	**`ValueState` 是一个包装类，类似于 Java 标准库里边的 `AtomicReference` 和 `AtomicLong`。 它提供了三个用于交互的方法。**

+ `update` 用于更新状态
+ `value` 用于获取状态值
+  `clear` 用于清空状态。 

​	如果一个 key 还没有状态，例如当程序刚启动或者调用过 `ValueState#clear` 方法时，`ValueState#value` 将会返回 `null`。 

​	**如果需要更新状态，需要调用 `ValueState#update` 方法，直接更改 `ValueState#value` 的返回值可能不会被系统识别**。 <u>**容错处理将在 Flink 后台自动管理，你可以像与常规变量那样与状态变量进行交互。**</u>

​	下边的示例，说明了如何使用标记状态来追踪可能的欺诈交易行为。

```java
@Override
public void processElement(
        Transaction transaction,
        Context context,
        Collector<Alert> collector) throws Exception {

    // Get the current state for the current key
    Boolean lastTransactionWasSmall = flagState.value();

    // Check if the flag is set
    if (lastTransactionWasSmall != null) {
        if (transaction.getAmount() > LARGE_AMOUNT) {
            // Output an alert downstream
            Alert alert = new Alert();
            alert.setId(transaction.getAccountId());

            collector.collect(alert);
        }

        // Clean up our state
        flagState.clear();
    }

    if (transaction.getAmount() < SMALL_AMOUNT) {
        // Set the flag to true
        flagState.update(true);
    }
}
```

​	对于每笔交易，欺诈检测器都会检查该帐户的标记状态。 **请记住，`ValueState` 的作用域始终限于当前的 key**，即信用卡帐户。 如果标记状态不为空，则该帐户的上一笔交易是小额的，因此，如果当前这笔交易的金额很大，那么检测程序将输出报警信息。

​	在检查之后，不论是什么状态，都需要被清空。 不管是当前交易触发了欺诈报警而造成模式的结束，还是当前交易没有触发报警而造成模式的中断，都需要重新开始新的模式检测。

​	最后，检查当前交易的金额是否属于小额交易。 如果是，那么需要设置标记状态，以便可以在下一个事件中对其进行检查。 注意，`ValueState<Boolean>` 实际上有 3 种状态：unset (`null`)，`true`，和 `false`，`ValueState` 是允许空值的。 我们的程序只使用了 unset (`null`) 和 `true` 两种来判断标记状态被设置了与否。

## 欺诈检测器 v2：状态 + 时间 = ❤️

​	骗子们在小额交易后不会等很久就进行大额消费，这样可以降低小额测试交易被发现的几率。 比如，假设你为欺诈检测器设置了一分钟的超时，对于上边的例子，交易 3 和 交易 4 只有间隔在一分钟之内才被认为是欺诈交易。 **Flink 中的 `KeyedProcessFunction` 允许您设置计时器，该计时器在将来的某个时间点执行回调函数**。

让我们看看如何修改程序以符合我们的新要求：

- 当标记状态被设置为 `true` 时，设置一个在当前时间一分钟后触发的定时器。
- 当定时器被触发时，重置标记状态。
- 当标记状态被重置时，删除定时器。

​	要删除一个定时器，你需要记录这个定时器的触发时间，这同样需要状态来实现，所以你需要在标记状态后也创建一个记录定时器时间的状态。

```java
private transient ValueState<Boolean> flagState;
private transient ValueState<Long> timerState;

@Override
public void open(Configuration parameters) {
    ValueStateDescriptor<Boolean> flagDescriptor = new ValueStateDescriptor<>(
            "flag",
            Types.BOOLEAN);
    flagState = getRuntimeContext().getState(flagDescriptor);

    ValueStateDescriptor<Long> timerDescriptor = new ValueStateDescriptor<>(
            "timer-state",
            Types.LONG);
    timerState = getRuntimeContext().getState(timerDescriptor);
}
```

​	**`KeyedProcessFunction#processElement` 需要使用提供了定时器服务的 `Context` 来调用。 定时器服务可以用于查询当前时间、注册定时器和删除定时器**。 使用它，你可以在标记状态被设置时，也设置一个当前时间一分钟后触发的定时器，同时，将触发时间保存到 `timerState` 状态中。

```java
if (transaction.getAmount() < SMALL_AMOUNT) {
    // set the flag to true
    flagState.update(true);

    // set the timer and timer state
    long timer = context.timerService().currentProcessingTime() + ONE_MINUTE;
    context.timerService().registerProcessingTimeTimer(timer);
    timerState.update(timer);
}
```

​	**处理时间是本地时钟时间，这是由运行任务的服务器的系统时间来决定的。**

​	当定时器触发时，将会调用 `KeyedProcessFunction#onTimer` 方法。 通过重写这个方法来实现一个你自己的重置状态的回调逻辑。

```java
@Override
public void onTimer(long timestamp, OnTimerContext ctx, Collector<Alert> out) {
    // remove flag after 1 minute
    timerState.clear();
    flagState.clear();
}
```

​	最后，如果要取消定时器，你需要删除已经注册的定时器，并同时清空保存定时器的状态。 你可以把这些逻辑封装到一个助手函数中，而不是直接调用 `flagState.clear()`。

```java
private void cleanUp(Context ctx) throws Exception {
    // delete timer
    Long timer = timerState.value();
    ctx.timerService().deleteProcessingTimeTimer(timer);

    // clean up all state
    timerState.clear();
    flagState.clear();
}
```

​	这就是一个功能完备的，有状态的分布式流处理程序了。

## 完整的程序

```java
package spendreport;

import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
import org.apache.flink.util.Collector;
import org.apache.flink.walkthrough.common.entity.Alert;
import org.apache.flink.walkthrough.common.entity.Transaction;

public class FraudDetector extends KeyedProcessFunction<Long, Transaction, Alert> {

    private static final long serialVersionUID = 1L;

    private static final double SMALL_AMOUNT = 1.00;
    private static final double LARGE_AMOUNT = 500.00;
    private static final long ONE_MINUTE = 60 * 1000;

    private transient ValueState<Boolean> flagState;
    private transient ValueState<Long> timerState;

    @Override
    public void open(Configuration parameters) {
        ValueStateDescriptor<Boolean> flagDescriptor = new ValueStateDescriptor<>(
                "flag",
                Types.BOOLEAN);
        flagState = getRuntimeContext().getState(flagDescriptor);

        ValueStateDescriptor<Long> timerDescriptor = new ValueStateDescriptor<>(
                "timer-state",
                Types.LONG);
        timerState = getRuntimeContext().getState(timerDescriptor);
    }

    @Override
    public void processElement(
            Transaction transaction,
            Context context,
            Collector<Alert> collector) throws Exception {

        // Get the current state for the current key
        Boolean lastTransactionWasSmall = flagState.value();

        // Check if the flag is set
        if (lastTransactionWasSmall != null) {
            if (transaction.getAmount() > LARGE_AMOUNT) {
                //Output an alert downstream
                Alert alert = new Alert();
                alert.setId(transaction.getAccountId());

                collector.collect(alert);
            }
            // Clean up our state
            cleanUp(context);
        }

        if (transaction.getAmount() < SMALL_AMOUNT) {
            // set the flag to true
            flagState.update(true);

            long timer = context.timerService().currentProcessingTime() + ONE_MINUTE;
            context.timerService().registerProcessingTimeTimer(timer);

            timerState.update(timer);
        }
    }

    @Override
    public void onTimer(long timestamp, OnTimerContext ctx, Collector<Alert> out) {
        // remove flag after 1 minute
        timerState.clear();
        flagState.clear();
    }

    private void cleanUp(Context ctx) throws Exception {
        // delete timer
        Long timer = timerState.value();
        ctx.timerService().deleteProcessingTimeTimer(timer);

        // clean up all state
        timerState.clear();
        flagState.clear();
    }
}
```

### 期望的结果

​	使用已准备好的 `TransactionSource` 数据源运行这个代码，将会检测到账户 3 的欺诈行为，并输出报警信息。 你将能够在你的 task manager 的日志中看到下边输出：

```shell
2019-08-19 14:22:06,220 INFO  org.apache.flink.walkthrough.common.sink.AlertSink            - Alert{id=3}
2019-08-19 14:22:11,383 INFO  org.apache.flink.walkthrough.common.sink.AlertSink            - Alert{id=3}
2019-08-19 14:22:16,551 INFO  org.apache.flink.walkthrough.common.sink.AlertSink            - Alert{id=3}
2019-08-19 14:22:21,723 INFO  org.apache.flink.walkthrough.common.sink.AlertSink            - Alert{id=3}
2019-08-19 14:22:26,896 INFO  org.apache.flink.walkthrough.common.sink.AlertSink            - Alert{id=3}
```

# 基于 Table API 实现实时报表

​	Apache Flink offers a Table API as a unified, relational API for batch and stream processing, i.e., queries are executed with the same semantics on unbounded, real-time streams or bounded, batch data sets and produce the same results. The Table API in Flink is commonly used to ease the definition of data analytics, data pipelining, and ETL applications.

## What Will You Be Building?

> [Grafana](https://grafana.com/)
>
> [可视化工具Grafana：简介及安装](https://www.cnblogs.com/imyalost/p/9873641.html)

​	In this tutorial, you will learn how to build a real-time dashboard to track financial transactions by account. The pipeline will read data from Kafka and write the results to MySQL visualized via Grafana.

## Prerequisites

​	This walkthrough assumes that you have some familiarity with Java or Scala, but you should be able to follow along even if you come from a different programming language. It also assumes that you are familiar with basic relational concepts such as `SELECT` and `GROUP BY` clauses.

## Help, I’m Stuck!

​	If you get stuck, check out the [community support resources](https://flink.apache.org/community.html). In particular, Apache Flink’s [user mailing list](https://flink.apache.org/community.html#mailing-lists) consistently ranks as one of the most active of any Apache project and a great way to get help quickly.

> If running docker on windows and your data generator container is failing to start, then please ensure that you're using the right shell. For example **docker-entrypoint.sh** for **table-walkthrough_data-generator_1** container requires bash. If unavailable, it will throw an error **standard_init_linux.go:211: exec user process caused "no such file or directory"**. A workaround is to switch the shell to **sh** on the first line of **docker-entrypoint.sh**.

## How To Follow Along

​	If you want to follow along, you will require a computer with:

- Java 8 or 11
- Maven
- Docker

​	The required configuration files are available in the [flink-playgrounds](https://github.com/apache/flink-playgrounds) repository. Once downloaded, open the project `flink-playground/table-walkthrough` in your IDE and navigate to the file `SpendReport`.

```java
EnvironmentSettings settings = EnvironmentSettings.newInstance().build();
TableEnvironment tEnv = TableEnvironment.create(settings);

tEnv.executeSql("CREATE TABLE transactions (\n" +
    "    account_id  BIGINT,\n" +
    "    amount      BIGINT,\n" +
    "    transaction_time TIMESTAMP(3),\n" +
    "    WATERMARK FOR transaction_time AS transaction_time - INTERVAL '5' SECOND\n" +
    ") WITH (\n" +
    "    'connector' = 'kafka',\n" +
    "    'topic'     = 'transactions',\n" +
    "    'properties.bootstrap.servers' = 'kafka:9092',\n" +
    "    'format'    = 'csv'\n" +
    ")");

tEnv.executeSql("CREATE TABLE spend_report (\n" +
    "    account_id BIGINT,\n" +
    "    log_ts     TIMESTAMP(3),\n" +
    "    amount     BIGINT\n," +
    "    PRIMARY KEY (account_id, log_ts) NOT ENFORCED" +
    ") WITH (\n" +
    "   'connector'  = 'jdbc',\n" +
    "   'url'        = 'jdbc:mysql://mysql:3306/sql-demo',\n" +
    "   'table-name' = 'spend_report',\n" +
    "   'driver'     = 'com.mysql.jdbc.Driver',\n" +
    "   'username'   = 'sql-demo',\n" +
    "   'password'   = 'demo-sql'\n" +
    ")");

Table transactions = tEnv.from("transactions");
report(transactions).executeInsert("spend_report");
```

## Breaking Down The Code

### The Execution Environment

​	The first two lines set up your `TableEnvironment`. The table environment is how you can set properties for your Job, specify whether you are writing a batch or a streaming application, and create your sources. This walkthrough creates a standard table environment that uses the streaming execution.

```java
EnvironmentSettings settings = EnvironmentSettings.newInstance().build();
TableEnvironment tEnv = TableEnvironment.create(settings);
```

### Registering Tables

> [Catalogs](https://ci.apache.org/projects/flink/flink-docs-release-1.12/dev/table/catalogs.html#catalogs)
>
> Catalogs provide metadata, such as databases, tables, partitions, views, and functions and information needed to access data stored in a database or other external systems.
>
> **One of the most crucial aspects of data processing is managing metadata. It may be transient metadata like temporary tables, or UDFs registered against the table environment. Or permanent metadata, like that in a Hive Metastore. Catalogs provide a unified API for managing metadata and making it accessible from the Table API and SQL Queries.**
>
> Catalog enables users to reference existing metadata in their data systems, and automatically maps them to Flink’s corresponding metadata. For example, Flink can map JDBC tables to Flink table automatically, and users don’t have to manually re-writing DDLs in Flink. Catalog greatly simplifies steps required to get started with Flink with users’ existing system, and greatly enhanced user experiences.

​	Next, tables are registered in the current [catalog](https://ci.apache.org/projects/flink/flink-docs-release-1.12/dev/table/catalogs.html) that you can use to connect to external systems for reading and writing both batch and streaming data. A table source provides access to data stored in external systems, such as a database, a key-value store, a message queue, or a file system. A table sink emits a table to an external storage system. Depending on the type of source and sink, they support different formats such as CSV, JSON, Avro, or Parquet.

```java
tEnv.executeSql("CREATE TABLE transactions (\n" +
     "    account_id  BIGINT,\n" +
     "    amount      BIGINT,\n" +
     "    transaction_time TIMESTAMP(3),\n" +
     "    WATERMARK FOR transaction_time AS transaction_time - INTERVAL '5' SECOND\n" +
     ") WITH (\n" +
     "    'connector' = 'kafka',\n" +
     "    'topic'     = 'transactions',\n" +
     "    'properties.bootstrap.servers' = 'kafka:9092',\n" +
     "    'format'    = 'csv'\n" +
     ")");
```

​	Two tables are registered; a transaction input table, and a spend report output table. The transactions (`transactions`) table lets us read credit card transactions, which contain account ID’s (`account_id`), timestamps (`transaction_time`), and US$ amounts (`amount`). The table is a logical view over a Kafka topic called `transactions` containing CSV data.

```java
tEnv.executeSql("CREATE TABLE spend_report (\n" +
    "    account_id BIGINT,\n" +
    "    log_ts     TIMESTAMP(3),\n" +
    "    amount     BIGINT\n," +
    "    PRIMARY KEY (account_id, log_ts) NOT ENFORCED" +
    ") WITH (\n" +
    "    'connector'  = 'jdbc',\n" +
    "    'url'        = 'jdbc:mysql://mysql:3306/sql-demo',\n" +
    "    'table-name' = 'spend_report',\n" +
    "    'driver'     = 'com.mysql.jdbc.Driver',\n" +
    "    'username'   = 'sql-demo',\n" +
    "    'password'   = 'demo-sql'\n" +
    ")");
```

​	The second table, `spend_report`, stores the final results of the aggregation. Its underlying storage is a table in a MySql database.

### The Query

​	With the environment configured and tables registered, you are ready to build your first application. **From the `TableEnvironment` you can read `from` an input table to read its rows and then write those results into an output table using `executeInsert`.** The `report` function is where you will implement your business logic. It is currently unimplemented.

```java
Table transactions = tEnv.from("transactions");
report(transactions).executeInsert("spend_report");
```

## Testing

The project contains a secondary testing class `SpendReportTest` that validates the logic of the report. It creates a table environment in batch mode.

```java
EnvironmentSettings settings = EnvironmentSettings.newInstance().inBatchMode().build();
TableEnvironment tEnv = TableEnvironment.create(settings); 
```

​	**One of Flink’s unique properties is that it provides consistent semantics across batch and streaming**. 

​	This means **you can develop and test applications in batch mode on static datasets, and deploy to production as streaming applications**.

## Attempt One

Now with the skeleton of a Job set-up, you are ready to add some business logic. The goal is to build a report that shows the total spend for each account across each hour of the day. This means the timestamp column needs be be rounded down from millisecond to hour granularity.

Flink supports developing relational applications in pure [SQL](https://ci.apache.org/projects/flink/flink-docs-release-1.12/dev/table/sql/) or using the [Table API](https://ci.apache.org/projects/flink/flink-docs-release-1.12/dev/table/tableApi.html). The Table API is a fluent DSL inspired by SQL, that can be written in Python, Java, or Scala and supports strong IDE integration. Just like a SQL query, Table programs can select the required fields and group by your keys. These features, allong with [built-in functions](https://ci.apache.org/projects/flink/flink-docs-release-1.12/dev/table/functions/systemFunctions.html) like `floor` and `sum`, you can write this report.

```java
public static Table report(Table transactions) {
    return transactions.select(
            $("account_id"),
            $("transaction_time").floor(TimeIntervalUnit.HOUR).as("log_ts"),
            $("amount"))
        .groupBy($("account_id"), $("log_ts"))
        .select(
            $("account_id"),
            $("log_ts"),
            $("amount").sum().as("amount"));
}
```

## User Defined Functions

​	Flink contains a limited number of built-in functions, and sometimes you need to extend it with a [user-defined function](https://ci.apache.org/projects/flink/flink-docs-release-1.12/dev/table/functions/udfs.html). If `floor` wasn’t predefined, you could implement it yourself.

```java
import java.time.LocalDateTime;
import java.time.temporal.ChronoUnit;

import org.apache.flink.table.annotation.DataTypeHint;
import org.apache.flink.table.functions.ScalarFunction;

public class MyFloor extends ScalarFunction {

    public @DataTypeHint("TIMESTAMP(3)") LocalDateTime eval(
        @DataTypeHint("TIMESTAMP(3)") LocalDateTime timestamp) {

        return timestamp.truncatedTo(ChronoUnit.HOURS);
    }
}
```

And then quickly integrate it in your application.

```java
public static Table report(Table transactions) {
    return transactions.select(
            $("account_id"),
            call(MyFloor.class, $("transaction_time")).as("log_ts"),
            $("amount"))
        .groupBy($("account_id"), $("log_ts"))
        .select(
            $("account_id"),
            $("log_ts"),
            $("amount").sum().as("amount"));
}
```

This query consumes all records from the `transactions` table, calculates the report, and outputs the results in an efficient, scalable manner. Running the test with this implementation will pass.

## Adding Windows

Grouping data based on time is a typical operation in data processing, especially when working with infinite streams. **A grouping based on time is called a [window](https://ci.apache.org/projects/flink/flink-docs-release-1.12/dev/stream/operators/windows.html) and Flink offers flexible windowing semantics**. **The most basic type of window is called a `Tumble` window, which has a fixed size and whose buckets do not overlap**.

```java
public static Table report(Table transactions) {
    return transactions
        .window(Tumble.over(lit(1).hour()).on($("transaction_time")).as("log_ts"))
        .groupBy($("account_id"), $("log_ts"))
        .select(
            $("account_id"),
            $("log_ts").start().as("log_ts"),
            $("amount").sum().as("amount"));
}
```

This defines your application as using one hour tumbling windows based on the timestamp column. So a row with timestamp `2019-06-01 01:23:47` is put in the `2019-06-01 01:00:00` window.

Aggregations based on time are unique because time, as opposed to other attributes, generally moves forward in a continuous streaming application. Unlike `floor` and your UDF, **window functions are [intrinsics](https://en.wikipedia.org/wiki/Intrinsic_function), which allows the runtime to apply additional optimizations**. In a batch context, windows offer a convenient API for grouping records by a timestamp attribute.

Running the test with this implementation will also pass.

## Once More, With Streaming!

And that’s it, a fully functional, stateful, distributed streaming application! The query continuously consumes the stream of transactions from Kafka, computes the hourly spendings, and emits results as soon as they are ready. **Since the input is unbounded, the query keeps running until it is manually stopped. And because the Job uses time window-based aggregations, Flink can perform specific optimizations such as state clean up when the framework knows that no more records will arrive for a particular window.**

The table playground is fully dockerized and runnable locally as streaming application. The environment contains a Kafka topic, a continuous data generator, MySql, and Grafana.

From within the `table-walkthrough` folder start the docker-compose script.

```shell
$ docker-compose build
$ docker-compose up -d
```

You can see information on the running job via the [Flink console](http://localhost:8082/).

![Flink Console](https://ci.apache.org/projects/flink/flink-docs-release-1.12/fig/spend-report-console.png)

Explore the results from inside MySQL.

```mysql
$ docker-compose exec mysql mysql -Dsql-demo -usql-demo -pdemo-sql

mysql> use sql-demo;
Database changed

mysql> select count(*) from spend_report;
+----------+
| count(*) |
+----------+
|      110 |
+----------+
```

Finally, go to [Grafana](http://localhost:3000/d/FOe0PbmGk/walkthrough?viewPanel=2&orgId=1&refresh=5s) to see the fully visualized result!

![Grafana](https://ci.apache.org/projects/flink/flink-docs-release-1.12/fig/spend-report-grafana.png)

# Flink Operations Playground

​	There are many ways to deploy and operate Apache Flink in various environments. Regardless of this variety, the fundamental building blocks of a Flink Cluster remain the same, and similar operational principles apply.

​	In this playground, you will learn how to manage and run Flink Jobs. You will see how to deploy and monitor an application, experience how Flink recovers from Job failure, and perform everyday operational tasks like upgrades and rescaling.

## Anatomy of this Playground

​	This playground consists of a long living [Flink Session Cluster](https://ci.apache.org/projects/flink/flink-docs-release-1.12/concepts/glossary.html#flink-session-cluster) and a Kafka Cluster.

​	**A Flink Cluster always consists of a [JobManager](https://ci.apache.org/projects/flink/flink-docs-release-1.12/concepts/glossary.html#flink-jobmanager) and one or more [Flink TaskManagers](https://ci.apache.org/projects/flink/flink-docs-release-1.12/concepts/glossary.html#flink-taskmanager).** The JobManager is responsible for handling [Job](https://ci.apache.org/projects/flink/flink-docs-release-1.12/concepts/glossary.html#flink-job) submissions, the supervision of Jobs as well as resource management. The Flink TaskManagers are the worker processes and are responsible for the execution of the actual [Tasks](https://ci.apache.org/projects/flink/flink-docs-release-1.12/concepts/glossary.html#task) which make up a Flink Job. In this playground you will start with a single TaskManager, but scale out to more TaskManagers later. Additionally, this playground comes with a dedicated *client* container, which we use to submit the Flink Job initially and to perform various operational tasks later on. The *client* container is not needed by the Flink Cluster itself but only included for ease of use.

​	The Kafka Cluster consists of a Zookeeper server and a Kafka Broker.

![Flink Docker Playground](https://ci.apache.org/projects/flink/flink-docs-release-1.12/fig/flink-docker-playground.svg)

​	When the playground is started a Flink Job called *Flink Event Count* will be submitted to the JobManager. Additionally, two Kafka Topics *input* and *output* are created.

![Click Event Count Example](https://ci.apache.org/projects/flink/flink-docs-release-1.12/fig/click-event-count-example.svg)

​	The Job consumes `ClickEvent`s from the *input* topic, each with a `timestamp` and a `page`. The events are then keyed by `page` and counted in 15 second [windows](https://ci.apache.org/projects/flink/flink-docs-release-1.12/dev/stream/operators/windows.html). The results are written to the *output* topic.

​	There are six different pages and we generate 1000 click events per page and 15 seconds. Hence, the output of the Flink job should show 1000 views per page and window.

## Starting the Playground

​	The playground environment is set up in just a few steps. We will walk you through the necessary commands and show how to validate that everything is running correctly.

​	We assume that you have [Docker](https://docs.docker.com/) (1.12+) and [docker-compose](https://docs.docker.com/compose/) (2.1+) installed on your machine.

​	The required configuration files are available in the [flink-playgrounds](https://github.com/apache/flink-playgrounds) repository. Check it out and spin up the environment:

```shell
git clone --branch release-1.12 https://github.com/apache/flink-playgrounds.git
cd flink-playgrounds/operations-playground
docker-compose build
docker-compose up -d
```

Afterwards, you can inspect the running Docker containers with the following command:

```shell
docker-compose ps

                    Name                                  Command               State                   Ports                
-----------------------------------------------------------------------------------------------------------------------------
operations-playground_clickevent-generator_1   /docker-entrypoint.sh java ...   Up       6123/tcp, 8081/tcp                  
operations-playground_client_1                 /docker-entrypoint.sh flin ...   Exit 0                                       
operations-playground_jobmanager_1             /docker-entrypoint.sh jobm ...   Up       6123/tcp, 0.0.0.0:8081->8081/tcp    
operations-playground_kafka_1                  start-kafka.sh                   Up       0.0.0.0:9094->9094/tcp              
operations-playground_taskmanager_1            /docker-entrypoint.sh task ...   Up       6123/tcp, 8081/tcp                  
operations-playground_zookeeper_1              /bin/sh -c /usr/sbin/sshd  ...   Up       2181/tcp, 22/tcp, 2888/tcp, 3888/tcp
```

This indicates that the client container has successfully submitted the Flink Job (`Exit 0`) and all cluster components as well as the data generator are running (`Up`).

You can stop the playground environment by calling:

```shell
docker-compose down -v
```

## Entering the Playground

​	There are many things you can try and check out in this playground. In the following two sections we will show you how to interact with the Flink Cluster and demonstrate some of Flink’s key features.

### Flink WebUI

​	The most natural starting point to observe your Flink Cluster is the WebUI exposed under [http://localhost:8081](http://localhost:8081/). If everything went well, you’ll see that the cluster initially consists of one TaskManager and executes a Job called *Click Event Count*.

![Playground Flink WebUI](https://ci.apache.org/projects/flink/flink-docs-release-1.12/fig/playground-webui.png)

​	The Flink WebUI contains a lot of useful and interesting information about your Flink Cluster and its Jobs (JobGraph, Metrics, Checkpointing Statistics, TaskManager Status,…).

### Logs

**JobManager**

The JobManager logs can be tailed via `docker-compose`.

```shell
docker-compose logs -f jobmanager
```

After the initial startup you should mainly see log messages for every checkpoint completion.

**TaskManager**

The TaskManager log can be tailed in the same way.

```shell
docker-compose logs -f taskmanager
```

After the initial startup you should mainly see log messages for every checkpoint completion.

### Flink CLI

The [Flink CLI](https://ci.apache.org/projects/flink/flink-docs-release-1.12/deployment/cli.html) can be used from within the client container. For example, to print the `help` message of the Flink CLI you can run

```shell
docker-compose run --no-deps client flink --help
```

### Flink REST API

The [Flink REST API](https://ci.apache.org/projects/flink/flink-docs-release-1.12/ops/rest_api.html#api) is exposed via `localhost:8081` on the host or via `jobmanager:8081` from the client container, e.g. to list all currently running jobs, you can run:

```shell
curl localhost:8081/jobs
```

### Kafka Topics

You can look at the records that are written to the Kafka Topics by running

```shell
//input topic (1000 records/s)
docker-compose exec kafka kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 --topic input

//output topic (24 records/min)
docker-compose exec kafka kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 --topic output
```

## Time to Play!

​	Now that you learned how to interact with Flink and the Docker containers, let’s have a look at some common operational tasks that you can try out on our playground. All of these tasks are independent of each other, i.e. you can perform them in any order. Most tasks can be executed via the [CLI](https://ci.apache.org/projects/flink/flink-docs-release-1.12/try-flink/flink-operations-playground.html#flink-cli) and the [REST API](https://ci.apache.org/projects/flink/flink-docs-release-1.12/try-flink/flink-operations-playground.html#flink-rest-api).

### Listing Running Jobs

**Command**

```
docker-compose run --no-deps client flink list
```

**Expected Output**

```
Waiting for response...
------------------ Running/Restarting Jobs -------------------
16.07.2019 16:37:55 : <job-id> : Click Event Count (RUNNING)
--------------------------------------------------------------
No scheduled jobs.
```

The JobID is assigned to a Job upon submission and is needed to perform actions on the Job via the CLI or REST API.

### Observing Failure & Recovery

Flink provides exactly-once processing guarantees under (partial) failure. In this playground you can observe and - to some extent - verify this behavior.

#### Step 1: Observing the Output

As described [above](https://ci.apache.org/projects/flink/flink-docs-release-1.12/try-flink/flink-operations-playground.html#anatomy-of-this-playground), the events in this playground are generate such that each window contains exactly one thousand records. So, in order to verify that Flink successfully recovers from a TaskManager failure without data loss or duplication you can tail the output topic and check that - after recovery - all windows are present and the count is correct.

For this, start reading from the *output* topic and leave this command running until after recovery (Step 3).

```
docker-compose exec kafka kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 --topic output
```

#### Step 2: Introducing a Fault

In order to simulate a partial failure you can kill a TaskManager. In a production setup, this could correspond to a loss of the TaskManager process, the TaskManager machine or simply a transient exception being thrown from the framework or user code (e.g. due to the temporary unavailability of an external resource).

```
docker-compose kill taskmanager
```

After a few seconds, the JobManager will notice the loss of the TaskManager, cancel the affected Job, and immediately resubmit it for recovery. When the Job gets restarted, its tasks remain in the `SCHEDULED` state, which is indicated by the purple colored squares (see screenshot below).

![Playground Flink WebUI](https://ci.apache.org/projects/flink/flink-docs-release-1.12/fig/playground-webui-failure.png)

**Note**: Even though the tasks of the job are in SCHEDULED state and not RUNNING yet, the overall status of a Job is shown as RUNNING.

At this point, the tasks of the Job cannot move from the `SCHEDULED` state to `RUNNING` because there are no resources (TaskSlots provided by TaskManagers) to the run the tasks. Until a new TaskManager becomes available, the Job will go through a cycle of cancellations and resubmissions.

In the meantime, the data generator keeps pushing `ClickEvent`s into the *input* topic. This is similar to a real production setup where data is produced while the Job to process it is down.

#### Step 3: Recovery

Once you restart the TaskManager, it reconnects to the JobManager.

```
docker-compose up -d taskmanager
```

When the JobManager is notified about the new TaskManager, it schedules the tasks of the recovering Job to the newly available TaskSlots. Upon restart, the tasks recover their state from the last successful [checkpoint](https://ci.apache.org/projects/flink/flink-docs-release-1.12/learn-flink/fault_tolerance.html) that was taken before the failure and switch to the `RUNNING` state.

The Job will quickly process the full backlog of input events (accumulated during the outage) from Kafka and produce output at a much higher rate (> 24 records/minute) until it reaches the head of the stream. In the *output* you will see that all keys (`page`s) are present for all time windows and that every count is exactly one thousand. Since we are using the [FlinkKafkaProducer](https://ci.apache.org/projects/flink/flink-docs-release-1.12/dev/connectors/kafka.html#kafka-producers-and-fault-tolerance) in its “at-least-once” mode, there is a chance that you will see some duplicate output records.

**Note**: Most production setups rely on a resource manager (Kubernetes, Yarn, Mesos) to automatically restart failed processes.

### Upgrading & Rescaling a Job

Upgrading a Flink Job always involves two steps: First, the Flink Job is gracefully stopped with a [Savepoint](https://ci.apache.org/projects/flink/flink-docs-release-1.12/ops/state/savepoints.html). A Savepoint is a consistent snapshot of the complete application state at a well-defined, globally consistent point in time (similar to a checkpoint). Second, the upgraded Flink Job is started from the Savepoint. In this context “upgrade” can mean different things including the following:

- An upgrade to the configuration (incl. the parallelism of the Job)
- An upgrade to the topology of the Job (added/removed Operators)
- An upgrade to the user-defined functions of the Job

Before starting with the upgrade you might want to start tailing the *output* topic, in order to observe that no data is lost or corrupted in the course the upgrade.

```
docker-compose exec kafka kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 --topic output
```

#### Step 1: Stopping the Job

To gracefully stop the Job, you need to use the “stop” command of either the CLI or the REST API. For this you will need the JobID of the Job, which you can obtain by [listing all running Jobs](https://ci.apache.org/projects/flink/flink-docs-release-1.12/try-flink/flink-operations-playground.html#listing-running-jobs) or from the WebUI. With the JobID you can proceed to stopping the Job:

**Command**

```
docker-compose run --no-deps client flink stop <job-id>
```

**Expected Output**

```
Suspending job "<job-id>" with a savepoint.
Savepoint completed. Path: file:<savepoint-path>
```

The Savepoint has been stored to the `state.savepoint.dir` configured in the *flink-conf.yaml*, which is mounted under */tmp/flink-savepoints-directory/* on your local machine. You will need the path to this Savepoint in the next step.

#### Step 2a: Restart Job without Changes

You can now restart the upgraded Job from this Savepoint. For simplicity, you can start by restarting it without any changes.

**Command**

```
docker-compose run --no-deps client flink run -s <savepoint-path> \
  -d /opt/ClickCountJob.jar \
  --bootstrap.servers kafka:9092 --checkpointing --event-time
```

**Expected Output**

```
Job has been submitted with JobID <job-id>
```

Once the Job is `RUNNING` again, you will see in the *output* Topic that records are produced at a higher rate while the Job is processing the backlog accumulated during the outage. Additionally, you will see that no data was lost during the upgrade: all windows are present with a count of exactly one thousand.

#### Step 2b: Restart Job with a Different Parallelism (Rescaling)

Alternatively, you could also rescale the Job from this Savepoint by passing a different parallelism during resubmission.

**Command**

```
docker-compose run --no-deps client flink run -p 3 -s <savepoint-path> \
  -d /opt/ClickCountJob.jar \
  --bootstrap.servers kafka:9092 --checkpointing --event-time
```

**Expected Output**

```
Starting execution of program
Job has been submitted with JobID <job-id>
```

Now, the Job has been resubmitted, but it will not start as there are not enough TaskSlots to execute it with the increased parallelism (2 available, 3 needed). With

```
docker-compose scale taskmanager=2
```

you can add a second TaskManager with two TaskSlots to the Flink Cluster, which will automatically register with the JobManager. Shortly after adding the TaskManager the Job should start running again.

Once the Job is “RUNNING” again, you will see in the *output* Topic that no data was lost during rescaling: all windows are present with a count of exactly one thousand.

## Querying the Metrics of a Job

The JobManager exposes system and user [metrics](https://ci.apache.org/projects/flink/flink-docs-release-1.12/ops/metrics.html) via its REST API.

The endpoint depends on the scope of these metrics. Metrics scoped to a Job can be listed via `jobs/<job-id>/metrics`. The actual value of a metric can be queried via the `get` query parameter.

**Request**

```
curl "localhost:8081/jobs/<jod-id>/metrics?get=lastCheckpointSize"
```

**Expected Response (pretty-printed; no placeholders)**

```
[
  {
    "id": "lastCheckpointSize",
    "value": "9378"
  }
]
```

The REST API can not only be used to query metrics, but you can also retrieve detailed information about the status of a running Job.

**Request**

```
# find the vertex-id of the vertex of interest
curl localhost:8081/jobs/<jod-id>
```

**Expected Response (pretty-printed)**

```
{
  "jid": "<job-id>",
  "name": "Click Event Count",
  "isStoppable": false,
  "state": "RUNNING",
  "start-time": 1564467066026,
  "end-time": -1,
  "duration": 374793,
  "now": 1564467440819,
  "timestamps": {
    "CREATED": 1564467066026,
    "FINISHED": 0,
    "SUSPENDED": 0,
    "FAILING": 0,
    "CANCELLING": 0,
    "CANCELED": 0,
    "RECONCILING": 0,
    "RUNNING": 1564467066126,
    "FAILED": 0,
    "RESTARTING": 0
  },
  "vertices": [
    {
      "id": "<vertex-id>",
      "name": "ClickEvent Source",
      "parallelism": 2,
      "status": "RUNNING",
      "start-time": 1564467066423,
      "end-time": -1,
      "duration": 374396,
      "tasks": {
        "CREATED": 0,
        "FINISHED": 0,
        "DEPLOYING": 0,
        "RUNNING": 2,
        "CANCELING": 0,
        "FAILED": 0,
        "CANCELED": 0,
        "RECONCILING": 0,
        "SCHEDULED": 0
      },
      "metrics": {
        "read-bytes": 0,
        "read-bytes-complete": true,
        "write-bytes": 5033461,
        "write-bytes-complete": true,
        "read-records": 0,
        "read-records-complete": true,
        "write-records": 166351,
        "write-records-complete": true
      }
    },
    {
      "id": "<vertex-id>",
      "name": "Timestamps/Watermarks",
      "parallelism": 2,
      "status": "RUNNING",
      "start-time": 1564467066441,
      "end-time": -1,
      "duration": 374378,
      "tasks": {
        "CREATED": 0,
        "FINISHED": 0,
        "DEPLOYING": 0,
        "RUNNING": 2,
        "CANCELING": 0,
        "FAILED": 0,
        "CANCELED": 0,
        "RECONCILING": 0,
        "SCHEDULED": 0
      },
      "metrics": {
        "read-bytes": 5066280,
        "read-bytes-complete": true,
        "write-bytes": 5033496,
        "write-bytes-complete": true,
        "read-records": 166349,
        "read-records-complete": true,
        "write-records": 166349,
        "write-records-complete": true
      }
    },
    {
      "id": "<vertex-id>",
      "name": "ClickEvent Counter",
      "parallelism": 2,
      "status": "RUNNING",
      "start-time": 1564467066469,
      "end-time": -1,
      "duration": 374350,
      "tasks": {
        "CREATED": 0,
        "FINISHED": 0,
        "DEPLOYING": 0,
        "RUNNING": 2,
        "CANCELING": 0,
        "FAILED": 0,
        "CANCELED": 0,
        "RECONCILING": 0,
        "SCHEDULED": 0
      },
      "metrics": {
        "read-bytes": 5085332,
        "read-bytes-complete": true,
        "write-bytes": 316,
        "write-bytes-complete": true,
        "read-records": 166305,
        "read-records-complete": true,
        "write-records": 6,
        "write-records-complete": true
      }
    },
    {
      "id": "<vertex-id>",
      "name": "ClickEventStatistics Sink",
      "parallelism": 2,
      "status": "RUNNING",
      "start-time": 1564467066476,
      "end-time": -1,
      "duration": 374343,
      "tasks": {
        "CREATED": 0,
        "FINISHED": 0,
        "DEPLOYING": 0,
        "RUNNING": 2,
        "CANCELING": 0,
        "FAILED": 0,
        "CANCELED": 0,
        "RECONCILING": 0,
        "SCHEDULED": 0
      },
      "metrics": {
        "read-bytes": 20668,
        "read-bytes-complete": true,
        "write-bytes": 0,
        "write-bytes-complete": true,
        "read-records": 6,
        "read-records-complete": true,
        "write-records": 0,
        "write-records-complete": true
      }
    }
  ],
  "status-counts": {
    "CREATED": 0,
    "FINISHED": 0,
    "DEPLOYING": 0,
    "RUNNING": 4,
    "CANCELING": 0,
    "FAILED": 0,
    "CANCELED": 0,
    "RECONCILING": 0,
    "SCHEDULED": 0
  },
  "plan": {
    "jid": "<job-id>",
    "name": "Click Event Count",
    "nodes": [
      {
        "id": "<vertex-id>",
        "parallelism": 2,
        "operator": "",
        "operator_strategy": "",
        "description": "ClickEventStatistics Sink",
        "inputs": [
          {
            "num": 0,
            "id": "<vertex-id>",
            "ship_strategy": "FORWARD",
            "exchange": "pipelined_bounded"
          }
        ],
        "optimizer_properties": {}
      },
      {
        "id": "<vertex-id>",
        "parallelism": 2,
        "operator": "",
        "operator_strategy": "",
        "description": "ClickEvent Counter",
        "inputs": [
          {
            "num": 0,
            "id": "<vertex-id>",
            "ship_strategy": "HASH",
            "exchange": "pipelined_bounded"
          }
        ],
        "optimizer_properties": {}
      },
      {
        "id": "<vertex-id>",
        "parallelism": 2,
        "operator": "",
        "operator_strategy": "",
        "description": "Timestamps/Watermarks",
        "inputs": [
          {
            "num": 0,
            "id": "<vertex-id>",
            "ship_strategy": "FORWARD",
            "exchange": "pipelined_bounded"
          }
        ],
        "optimizer_properties": {}
      },
      {
        "id": "<vertex-id>",
        "parallelism": 2,
        "operator": "",
        "operator_strategy": "",
        "description": "ClickEvent Source",
        "optimizer_properties": {}
      }
    ]
  }
}
```

Please consult the [REST API reference](https://ci.apache.org/projects/flink/flink-docs-release-1.12/ops/rest_api.html#api) for a complete list of possible queries including how to query metrics of different scopes (e.g. TaskManager metrics);

## Variants

You might have noticed that the *Click Event Count* application was always started with `--checkpointing` and `--event-time` program arguments. By omitting these in the command of the *client* container in the `docker-compose.yaml`, you can change the behavior of the Job.

- `--checkpointing` enables [checkpoint](https://ci.apache.org/projects/flink/flink-docs-release-1.12/learn-flink/fault_tolerance.html), which is Flink’s fault-tolerance mechanism. If you run without it and go through [failure and recovery](https://ci.apache.org/projects/flink/flink-docs-release-1.12/try-flink/flink-operations-playground.html#observing-failure--recovery), you should will see that data is actually lost.
- `--event-time` enables [event time semantics](https://ci.apache.org/projects/flink/flink-docs-release-1.12/dev/event_time.html) for your Job. When disabled, the Job will assign events to windows based on the wall-clock time instead of the timestamp of the `ClickEvent`. Consequently, the number of events per window will not be exactly one thousand anymore.

The *Click Event Count* application also has another option, turned off by default, that you can enable to explore the behavior of this job under backpressure. You can add this option in the command of the *client* container in `docker-compose.yaml`.

- `--backpressure` adds an additional operator into the middle of the job that causes severe backpressure during even-numbered minutes (e.g., during 10:12, but not during 10:13). This can be observed by inspecting various [network metrics](https://ci.apache.org/projects/flink/flink-docs-release-1.12/ops/metrics.html#default-shuffle-service) such as `outputQueueLength` and `outPoolUsage`, and/or by using the [backpressure monitoring](https://ci.apache.org/projects/flink/flink-docs-release-1.12/ops/monitoring/back_pressure.html#monitoring-back-pressure) available in the WebUI.

# Learn Flink: Hands-on Training

## Goals and Scope of this Training

> [ETL讲解（很详细！！！）](https://www.cnblogs.com/yjd_hycf_space/p/7772722.html)

​	This training presents an introduction to Apache Flink that includes just enough to get you started writing scalable streaming ETL, analytics, and event-driven applications, while leaving out a lot of (ultimately important) details. The focus is on providing straightforward introductions to Flink’s APIs for managing state and time, with the expectation that having mastered these fundamentals, you’ll be much better equipped to pick up the rest of what you need to know from the more detailed reference documentation. The links at the end of each section will lead you to where you can learn more.

​	Specifically, you will learn:

- how to implement streaming data processing pipelines
- how and why Flink manages state
- how to use event time to consistently compute accurate analytics
- how to build event-driven applications on continuous streams
- how Flink is able to provide fault-tolerant, stateful stream processing with exactly-once semantics

​	This training focuses on four critical concepts: continuous processing of streaming data, event time, stateful stream processing, and state snapshots. This page introduces these concepts.

​	**Note** Accompanying this training is a set of hands-on exercises that will guide you through learning how to work with the concepts being presented. A link to the relevant exercise is provided at the end of each section.

## Stream Processing

​	Streams are data’s natural habitat. Whether it is events from web servers, trades from a stock exchange, or sensor readings from a machine on a factory floor, data is created as part of a stream. But when you analyze data, you can either organize your processing around *bounded* or *unbounded* streams, and which of these paradigms you choose has profound consequences.

![Bounded and unbounded streams](https://ci.apache.org/projects/flink/flink-docs-release-1.12/fig/bounded-unbounded.png)

​	**Batch processing** is the paradigm at work when you process a bounded data stream. In this mode of operation you can choose to ingest the entire dataset before producing any results, which means that it is possible, for example, to sort the data, compute global statistics, or produce a final report that summarizes all of the input.

​	**Stream processing**, on the other hand, involves unbounded data streams. Conceptually, at least, the input may never end, and so you are forced to continuously process the data as it arrives.

​	In Flink, applications are composed of **streaming dataflows** that may be transformed by user-defined **operators**. These dataflows form directed graphs that start with one or more **sources**, and end in one or more **sinks**.

![A DataStream program, and its dataflow.](https://ci.apache.org/projects/flink/flink-docs-release-1.12/fig/program_dataflow.svg)

​	Often there is a one-to-one correspondence between the transformations in the program and the operators in the dataflow. Sometimes, however, one transformation may consist of multiple operators.

​	An application may consume real-time data from streaming sources such as message queues or distributed logs, like Apache Kafka or Kinesis. But flink can also consume bounded, historic data from a variety of data sources. Similarly, the streams of results being produced by a Flink application can be sent to a wide variety of systems that can be connected as sinks.

![Flink application with sources and sinks](https://ci.apache.org/projects/flink/flink-docs-release-1.12/fig/flink-application-sources-sinks.png)

### Parallel Dataflows

​	Programs in Flink are inherently parallel and distributed. During execution, a *stream* has one or more **stream partitions**, and each *operator* has one or more **operator subtasks**. The operator subtasks are independent of one another, and execute in different threads and possibly on different machines or containers.

​	The number of operator subtasks is the **parallelism** of that particular operator. Different operators of the same program may have different levels of parallelism.

![A parallel dataflow](https://ci.apache.org/projects/flink/flink-docs-release-1.12/fig/parallel_dataflow.svg)

​	Streams can transport data between two operators in a *one-to-one* (or *forwarding*) pattern, or in a *redistributing* pattern:

- **One-to-one** streams (for example between the *Source* and the *map()* operators in the figure above) preserve the partitioning and ordering of the elements. That means that subtask[1] of the *map()* operator will see the same elements in the same order as they were produced by subtask[1] of the *Source* operator.
- **Redistributing** streams (as between *map()* and *keyBy/window* above, as well as between *keyBy/window* and *Sink*) change the partitioning of streams. Each *operator subtask* sends data to different target subtasks, depending on the selected transformation. Examples are *keyBy()* (which re-partitions by hashing the key), *broadcast()*, or *rebalance()* (which re-partitions randomly). In a *redistributing* exchange the ordering among the elements is only preserved within each pair of sending and receiving subtasks (for example, subtask[1] of *map()* and subtask[2] of *keyBy/window*). So, for example, the redistribution between the keyBy/window and the Sink operators shown above introduces non-determinism regarding the order in which the aggregated results for different keys arrive at the Sink.

## Timely Stream Processing

​	For most streaming applications it is very valuable to be able re-process historic data with the same code that is used to process live data – and to produce deterministic, consistent results, regardless.

​	It can also be crucial to pay attention to the order in which events occurred, rather than the order in which they are delivered for processing, and to be able to reason about when a set of events is (or should be) complete. For example, consider the set of events involved in an e-commerce transaction, or financial trade.

​	These requirements for timely stream processing can be met by using event time timestamps that are recorded in the data stream, rather than using the clocks of the machines processing the data.

## Stateful Stream Processing

​	Flink’s operations can be stateful. This means that how one event is handled can depend on the accumulated effect of all the events that came before it. State may be used for something simple, such as counting events per minute to display on a dashboard, or for something more complex, such as computing features for a fraud detection model.

​	A Flink application is run in parallel on a distributed cluster. The various parallel instances of a given operator will execute independently, in separate threads, and in general will be running on different machines.

​	The set of parallel instances of a stateful operator is effectively a sharded key-value store. Each parallel instance is responsible for handling events for a specific group of keys, and the state for those keys is kept locally.

​	The diagram below shows a job running with a parallelism of two across the first three operators in the job graph, terminating in a sink that has a parallelism of one. The third operator is stateful, and you can see that a fully-connected network shuffle is occurring between the second and third operators. This is being done to partition the stream by some key, so that all of the events that need to be processed together, will be.

![State is sharded](https://ci.apache.org/projects/flink/flink-docs-release-1.12/fig/parallel-job.png)

​	**State is always accessed locally, which helps Flink applications achieve high throughput and low-latency. You can choose to keep state on the JVM heap, or if it is too large, in efficiently organized on-disk data structures.**

![State is local](https://ci.apache.org/projects/flink/flink-docs-release-1.12/fig/local-state.png)

## Fault Tolerance via State Snapshots

​	Flink is able to provide fault-tolerant, exactly-once semantics through a combination of state snapshots and stream replay. These snapshots capture the entire state of the distributed pipeline, recording offsets into the input queues as well as the state throughout the job graph that has resulted from having ingested the data up to that point. When a failure occurs, the sources are rewound, the state is restored, and processing is resumed. As depicted above, these state snapshots are captured asynchronously, without impeding the ongoing processing.

# Intro to the DataStream API

> [Intro to the DataStream API](https://ci.apache.org/projects/flink/flink-docs-release-1.12/learn-flink/datastream_api.html)

​	The focus of this training is to broadly cover the DataStream API well enough that you will be able to get started writing streaming applications.

## What can be Streamed?

Flink’s DataStream APIs for Java and Scala will let you stream anything they can serialize. Flink’s own serializer is used for

- basic types, i.e., String, Long, Integer, Boolean, Array
- composite types: Tuples, POJOs, and Scala case classes

and Flink falls back to Kryo for other types. It is also possible to use other serializers with Flink. Avro, in particular, is well supported.

## Java tuples and POJOs

Flink’s native serializer can operate efficiently on tuples and POJOs.