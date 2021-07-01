# 尚硅谷Flink入门到实战-学习笔记

> [尚硅谷2021最新Java版Flink](https://www.bilibili.com/video/BV1qy4y1q728)
>
> 下面笔记来源（尚硅谷公开资料、网络博客、个人小结）
>
> 中间会把自己认为较重要的点做做标记（下划线、加粗等）

# 1. Flink的特点

+ 事件驱动（Event-driven）

+ 基于流处理

  一切皆由流组成，离线数据是有界的流；实时数据是一个没有界限的流。（有界流、无界流）

+ 分层API

  + 越顶层越抽象，表达含义越简明，使用越方便
  + 越底层越具体，表达能力越丰富，使用越灵活

## 1.1 Flink vs Spark Streaming

+ 数据模型
  + Spark采用RDD模型，spark streaming的DStream实际上也就是一组组小批数据RDD的集合
  + flink基本数据模型是数据流，以及事件（Event）序列
+ 运行时架构
  + spark是批计算，将DAG划分为不同的stage，一个完成后才可以计算下一个
  + flink是标准的流执行模式，一个事件在一个节点处理完后可以直接发往下一个节点处理

# 2. 快速上手

## 2.1 批处理实现WordCount

> *flink-streaming-scala_2.12 => org.apache.flink:flink-runtime_2.12:1.12.1 => com.typesafe.akka:akka-actor_2.12:2.5.21，akka就是用scala实现的。即使这里我们用java语言，还是用到了scala实现的包*

pom依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>Flink_Tutorial</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
        <flink.version>1.12.1</flink.version>
        <scala.binary.version>2.12</scala.binary.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-java</artifactId>
            <version>${flink.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-scala_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-clients_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
        </dependency>
    </dependencies>

</project>
```

代码实现

```java
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.java.DataSet;
import org.apache.flink.api.java.ExecutionEnvironment;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.util.Collector;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2021/1/29 10:46 PM
 * 批处理 wordcount
 */
public class WordCount {
  public static void main(String[] args) throws Exception {
    // 创建执行环境
    ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

    // 从文件中读取数据
    String inputPath = "/tmp/Flink_Tutorial/src/main/resources/hello.txt";
    DataSet<String> inputDataSet = env.readTextFile(inputPath);

    // 对数据集进行处理，按空格分词展开，转换成(word, 1)二元组进行统计
    // 按照第一个位置的word分组
    // 按照第二个位置上的数据求和
    DataSet<Tuple2<String, Integer>> resultSet = inputDataSet.flatMap(new MyFlatMapper())
      .groupBy(0)
      .sum(1);

    resultSet.print();
  }

  // 自定义类，实现FlatMapFunction接口
  public static class MyFlatMapper implements FlatMapFunction<String, Tuple2<String, Integer>> {

    @Override
    public void flatMap(String s, Collector<Tuple2<String, Integer>> out) throws Exception {
      // 按空格分词
      String[] words = s.split(" ");
      // 遍历所有word，包成二元组输出
      for (String str : words) {
        out.collect(new Tuple2<>(str, 1));
      }
    }
  }

}
```

输出：

```shell
(scala,1)
(flink,1)
(world,1)
(hello,4)
(and,1)
(fine,1)
(how,1)
(spark,1)
(you,3)
(are,1)
(thank,1)
```

> [解决 Flink 升级1.11 报错 No ExecutorFactory found to execute the application](https://blog.csdn.net/qq_41398614/article/details/107553604)

## 2.2 流处理实现WordCount

在2.1批处理的基础上，新建一个类进行改动。

+ 批处理=>几组或所有数据到达后才处理；流处理=>有数据来就直接处理，不等数据堆叠到一定数量级

+ **这里不像批处理有groupBy => 所有数据统一处理，而是用流处理的keyBy => 每一个数据都对key进行hash计算，进行类似分区的操作，来一个数据就处理一次，所有中间过程都有输出！**

+ **并行度：开发环境的并行度默认就是计算机的CPU逻辑核数**

代码实现

```java
package wc;

import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.client.program.StreamContextEnvironment;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2021/1/29 11:13 PM
 */
public class StreamWordCount {

    public static void main(String[] args) throws Exception {

        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamContextEnvironment.getExecutionEnvironment();

      	// 设置并行度，默认值 = 当前计算机的CPU逻辑核数（设置成1即单线程处理）
        // env.setMaxParallelism(32);
      
        // 从文件中读取数据
        String inputPath = "/tmp/Flink_Tutorial/src/main/resources/hello.txt";
        DataStream<String> inputDataStream = env.readTextFile(inputPath);

        // 基于数据流进行转换计算
        DataStream<Tuple2<String,Integer>> resultStream = inputDataStream.flatMap(new WordCount.MyFlatMapper())
                .keyBy(item->item.f0)
                .sum(1);

        resultStream.print();

        // 执行任务
        env.execute();
    }
}
```

输出：

*这里因为是流处理，所以所有中间过程都会被输出，前面的序号就是并行执行任务的线程编号。*

```shell
9> (world,1)
5> (hello,1)
8> (are,1)
10> (you,1)
11> (how,1)
6> (thank,1)
9> (fine,1)
10> (you,2)
10> (you,3)
15> (and,1)
5> (hello,2)
13> (flink,1)
1> (spark,1)
5> (hello,3)
1> (scala,1)
5> (hello,4)
```

​	这里`env.execute();`之前的代码，可以理解为是在定义任务，只有执行`env.execute()`后，Flink才把前面的代码片段当作一个任务整体（每个线程根据这个任务操作，并行处理流数据）。

## 2.3 流式数据源测试

1. 通过`nc -lk <port>`打开一个socket服务，用于模拟实时的流数据

   ```shell
   nc -lk 7777
   ```

2. 代码修改inputStream的部分

   ```java
   package wc;
   
   import org.apache.flink.api.java.tuple.Tuple2;
   import org.apache.flink.client.program.StreamContextEnvironment;
   import org.apache.flink.streaming.api.datastream.DataStream;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   
   /**
    * @author : Ashiamd email: ashiamd@foxmail.com
    * @date : 2021/1/29 11:13 PM
    */
   public class StreamWordCount {
   
       public static void main(String[] args) throws Exception {
   
           // 创建流处理执行环境
           StreamExecutionEnvironment env = StreamContextEnvironment.getExecutionEnvironment();
   
           // 设置并行度，默认值 = 当前计算机的CPU逻辑核数（设置成1即单线程处理）
           // env.setMaxParallelism(32);
   
           // 从文件中读取数据
   //        String inputPath = "/tmp/Flink_Tutorial/src/main/resources/hello.txt";
   //        DataStream<String> inputDataStream = env.readTextFile(inputPath);
   
           // 从socket文本流读取数据
           DataStream<String> inputDataStream = env.socketTextStream("localhost", 7777);
   
           // 基于数据流进行转换计算
           DataStream<Tuple2<String,Integer>> resultStream = inputDataStream.flatMap(new WordCount.MyFlatMapper())
                   .keyBy(item->item.f0)
                   .sum(1);
   
           resultStream.print();
   
           // 执行任务
           env.execute();
       }
   }
   
   ```

3. 在本地开启的socket中输入数据，观察IDEA的console输出。

   ​	本人测试后发现，同一个字符串，前面输出的编号是一样的，因为key => hashcode,同一个key的hash值固定，分配给相对应的线程处理。

# 3. Flink部署

## 3.1 Standalone模式

> [Flink任务调度原理之TaskManager 与Slots](https://blog.csdn.net/qq_39657909/article/details/105823127)	<=	下面内容出自该博文

1. Flink 中每一个 TaskManager 都是一个JVM进程，它可能会在独立的线程上执行一个或多个 subtask
2. 为了控制一个 TaskManager 能接收多少个 task， TaskManager 通过 task slot 来进行控制（一个 TaskManager 至少有一个 slot）
3. 每个task slot表示TaskManager拥有资源的一个固定大小的子集。假如一个TaskManager有三个slot，那么它会将其管理的内存分成三份给各个slot(注：这里不会涉及CPU的隔离，slot仅仅用来隔离task的受管理内存)
4. 可以通过调整task slot的数量去自定义subtask之间的隔离方式。如一个TaskManager一个slot时，那么每个task group运行在独立的JVM中。而**当一个TaskManager多个slot时，多个subtask可以共同享有一个JVM,而在同一个JVM进程中的task将共享TCP连接和心跳消息，也可能共享数据集和数据结构，从而减少每个task的负载**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200428203404161.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5NjU3OTA5,size_16,color_FFFFFF,t_70#pic_center)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200428205219327.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5NjU3OTA5,size_16,color_FFFFFF,t_70#pic_center)

1. 默认情况下，Flink 允许子任务共享 slot，即使它们是不同任务的子任务（前提是它们来自同一个job）。 这样的结果是，一个 slot 可以保存作业的整个管道。
2. Task Slot 是静态的概念，是指 TaskManager 具有的并发执行能力，可以通过参数taskmanager.numberOfTaskSlots进行配置；而并行度parallelism是动态概念，即TaskManager运行程序时实际使用的并发能力，可以通过参数parallelism.default进行配置。
   举例：如果总共有3个TaskManager,每一个TaskManager中分配了3个TaskSlot,也就是每个TaskManager可以接收3个task,这样我们总共可以接收9个TaskSot。但是如果我们设置parallelism.default=1，那么当程序运行时9个TaskSlot将只有1个运行，8个都会处于空闲状态，所以要学会合理设置并行度！具体图解如下：
   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200902165040619.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5NjU3OTA5,size_16,color_FFFFFF,t_70#pic_center)

`conf/flink-conf.yaml`配置文件中

+ `taskmanager.numberOfTaskSlots`
+ `parallelism.default`

```yaml
# The number of task slots that each TaskManager offers. Each slot runs one parallel pipeline.

taskmanager.numberOfTaskSlots: 1

# The parallelism used for programs that did not specify and other parallelism.

parallelism.default: 1
```

注：**Flink存储State用的是堆外内存**，所以web UI里`JVM Heap Size`和`Flink Managed MEM`是两个分开的值。

### 3.1.1 Web UI提交job

> [Flink Savepoint简单介绍](https://blog.csdn.net/qq_37142346/article/details/91385333)

启动Flink后，可以在[Web UI](lcoalhost:8081)的`Submit New Job`提交jar包，然后指定Job参数。

+ Entry Class

  程序的入口，指定入口类（类的全限制名）

+ Program Arguments

  程序启动参数，例如`--host localhost --port 7777`

+ Parallelism

  设置Job并行度。

  Ps：并行度优先级（从上到下优先级递减）

  + 代码中算子`setParallelism()`
  + `ExecutionEnvironment env.setMaxParallelism()`
  + 设置的Job并行度
  + 集群conf配置文件中的`parallelism.default`

  ps：**socket等特殊的IO操作，本身不能并行处理，并行度只能是1**

+ Savepoint Path

  savepoint是通过checkpoint机制为streaming job创建的一致性快照，比如数据源offset，状态等。
  
  (savepoint可以理解为手动备份，而checkpoint为自动备份)

ps：提交job要注意分配的slot总数是否足够使用，如果slot总数不够，那么job执行失败。（资源不够调度）

这里提交前面demo项目的StreamWordCount，在本地socket即`nc -lk 7777`中输入字符串，查看结果

输入：

```shell
hello world, and thank you!
```

输出：

可以看出来输出的顺序并不是和输入的字符串严格相同的，因为是多个线程并行处理的。

```shell
1> (world,,1)
2> (and,1)
1> (thank,1)
2> (you!,1)
2> (hello,1)
```

### 3.1.2 命令行提交job

1. 查看已提交的所有job

   ```shell
   $ bin/flink list      
   Waiting for response...
   ------------------ Running/Restarting Jobs -------------------
   30.01.2021 17:09:45 : 30d9dda946a170484d55e41358973942 : Flink Streaming Job (RUNNING)
   --------------------------------------------------------------
   No scheduled jobs.
   ```

2. 提交job

   + `-c`指定入口类
   + `-p`指定job的并行度

   `bin/flink run -c <入口类> -p <并行度> <jar包路径> <启动参数>`

   ```shell
   $ bin/flink run -c wc.StreamWordCount -p 3 /tmp/Flink_Tutorial-1.0-SNAPSHOT.jar --host localhost --port 7777
   Job has been submitted with JobID 33a5d1f00688a362837830f0b85fd75e
   ```

3. 取消job

   `bin/flink cancel <Job的ID>`

   ```shell
   $ bin/flink cancel 30d9dda946a170484d55e41358973942
   Cancelling job 30d9dda946a170484d55e41358973942.
   Cancelled job 30d9dda946a170484d55e41358973942.
   ```

**注：Total Task Slots只要不小于Job中Parallelism最大值即可。**

eg：这里我配置文件设置`taskmanager.numberOfTaskSlots: 4`，实际Job运行时总Tasks显示9，但是里面具体4个任务步骤分别需求（1，3，3，2）数量的Tasks，4>3，满足最大的Parallelism即可运行成功。

## 3.2 yarn模式

> [4.6 Flink-流处理框架-Flink On Yarn（Session-cluster+Per-Job-Cluster）](https://blog.csdn.net/suyebiubiu/article/details/111874245)	<=	下面内容出自此处，主要方便索引图片URL

 	以Yarn模式部署Flink任务时，要求Flink是有 Hadoop 支持的版本，Hadoop 环境需要保证版本在 2.2 以上，并且集群中安装有 HDFS 服务。

### 3.2.1 Flink on Yarn

​	Flink提供了两种在yarn上运行的模式，分别为Session-Cluster和Per-Job-Cluster模式。

#### 1. Sesstion Cluster模式

​	Session-Cluster 模式需要先启动集群，然后再提交作业，接着会向 yarn 申请一块空间后，**资源永远保持不变**。如果资源满了，下一个作业就无法提交，只能等到 yarn 中的其中一个作业执行完成后，释放了资源，下个作业才会正常提交。**所有作业共享 Dispatcher 和 ResourceManager**；**共享资源；适合规模小执行时间短的作业。**

![img](https://img-blog.csdnimg.cn/20201228202616146.png)

​	**在 yarn 中初始化一个 flink 集群，开辟指定的资源，以后提交任务都向这里提交。这个 flink 集群会常驻在 yarn 集群中，除非手工停止。**

#### 2. Per Job Cluster 模式

​	一个 Job 会对应一个集群，每提交一个作业会根据自身的情况，都会单独向 yarn 申请资源，直到作业执行完成，一个作业的失败与否并不会影响下一个作业的正常提交和运行。**独享 Dispatcher 和 ResourceManager**，按需接受资源申请；适合规模大长时间运行的作业。

​	**每次提交都会创建一个新的 flink 集群，任务之间互相独立，互不影响，方便管理。任务执行完成之后创建的集群也会消失。**

![img](https://img-blog.csdnimg.cn/20201228202718916.png)

### 3.2.2 Session Cluster

1. 启动*hadoop*集群（略）

2. 启动*yarn-session*

   ```shell
   ./yarn-session.sh -n 2 -s 2 -jm 1024 -tm 1024 -nm test -d
   ```

   其中：

   + `-n(--container)`：TaskManager的数量。

   + `-s(--slots)`：每个TaskManager的slot数量，默认一个slot一个core，默认每个taskmanager的slot的个数为1，有时可以多一些taskmanager，做冗余。

   + `-jm`：JobManager的内存（单位MB)。

   + `-tm`：每个taskmanager的内存（单位MB)。

   + `-nm`：yarn 的appName(现在yarn的ui上的名字)。

   + `-d`：后台执行。

3. 执行任务

   ```shell
   ./flink run -c com.atguigu.wc.StreamWordCount FlinkTutorial-1.0-SNAPSHOT-jar-with-dependencies.jar --host lcoalhost –port 7777
   ```

4. 去 yarn 控制台查看任务状态

   ![img](https://img-blog.csdnimg.cn/20201228202911116.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1N1eWViaXViaXU=,size_16,color_FFFFFF,t_70)

5. 取消 yarn-session

   ```shell
   yarn application --kill application_1577588252906_0001
   ```

### 3.2.3 Per Job Cluster

1. 启动*hadoop*集群（略）

2. 不启动**yarn-session**，直接执行*job*

   ```shell
   ./flink run –m yarn-cluster -c com.atguigu.wc.StreamWordCount FlinkTutorial-1.0-SNAPSHOT-jar-with-dependencies.jar --host lcoalhost –port 7777
   ```

## 3.3 Kubernetes部署

​	容器化部署时目前业界很流行的一项技术，基于Docker镜像运行能够让用户更加方便地对应用进行管理和运维。容器管理工具中最为流行的就是Kubernetes（k8s），而Flink也在最近的版本中支持了k8s部署模式。

1. 搭建*Kubernetes*集群（略）

2. 配置各组件的*yaml*文件

​	在k8s上构建Flink Session Cluster，需要将Flink集群的组件对应的docker镜像分别在k8s上启动，包括JobManager、TaskManager、JobManagerService三个镜像服务。每个镜像服务都可以从中央镜像仓库中获取。

3. 启动*Flink Session Cluster*

   ```shell
   // 启动jobmanager-service 服务
   kubectl create -f jobmanager-service.yaml
   // 启动jobmanager-deployment服务
   kubectl create -f jobmanager-deployment.yaml
   // 启动taskmanager-deployment服务
   kubectl create -f taskmanager-deployment.yaml
   ```

4. 访问*Flink UI*页面

   集群启动后，就可以通过JobManagerServicers中配置的WebUI端口，用浏览器输入以下url来访问Flink UI页面了：

   `http://{JobManagerHost:Port}/api/v1/namespaces/default/services/flink-jobmanager:ui/proxy`

# 4. Flink运行架构

> [Flink-运行时架构中的四大组件|任务提交流程|任务调度原理|Slots和并行度中间的关系|数据流|执行图|数据得传输形式|任务链](https://blog.csdn.net/qq_40180229/article/details/106321149)

## 4.1 Flink运行时的组件

​	Flink运行时架构主要包括四个不同的组件，它们会在运行流处理应用程序时协同工作：

+ **作业管理器（JobManager）**
+ **资源管理器（ResourceManager）**
+ **任务管理器（TaskManager）**
+ **分发器（Dispatcher）**

​	因为Flink是用Java和Scala实现的，所以所有组件都会运行在Java虚拟机上。每个组件的职责如下：

### 作业管理器（JobManager）

​	控制一个应用程序执行的主进程，也就是说，每个应用程序都会被一个不同的JobManager所控制执行。

​	JobManager会先接收到要执行的应用程序，这个应用程序会包括：

+ 作业图（JobGraph）
+ 逻辑数据流图（logical dataflow graph）
+ 打包了所有的类、库和其它资源的JAR包。

​	JobManager会把JobGraph转换成一个物理层面的数据流图，这个图被叫做“执行图”（ExecutionGraph），包含了所有可以并发执行的任务。

​	**JobManager会向资源管理器（ResourceManager）请求执行任务必要的资源，也就是任务管理器（TaskManager）上的插槽（slot）。一旦它获取到了足够的资源，就会将执行图分发到真正运行它们的TaskManager上**。

​	在运行过程中，JobManager会负责所有需要中央协调的操作，比如说检查点（checkpoints）的协调。

### 资源管理器（ResourceManager）

​	主要负责管理任务管理器（TaskManager）的插槽（slot），TaskManger插槽是Flink中定义的处理资源单元。

​	Flink为不同的环境和资源管理工具提供了不同资源管理器，比如YARN、Mesos、K8s，以及standalone部署。

​	**当JobManager申请插槽资源时，ResourceManager会将有空闲插槽的TaskManager分配给JobManager**。如果ResourceManager没有足够的插槽来满足JobManager的请求，它还可以向资源提供平台发起会话，以提供启动TaskManager进程的容器。

​	另外，**ResourceManager还负责终止空闲的TaskManager，释放计算资源**。

### 任务管理器（TaskManager）

​	Flink中的工作进程。通常在Flink中会有多个TaskManager运行，每一个TaskManager都包含了一定数量的插槽（slots）。**插槽的数量限制了TaskManager能够执行的任务数量**。

​	启动之后，TaskManager会向资源管理器注册它的插槽；收到资源管理器的指令后，TaskManager就会将一个或者多个插槽提供给JobManager调用。JobManager就可以向插槽分配任务（tasks）来执行了。

​	**在执行过程中，一个TaskManager可以跟其它运行同一应用程序的TaskManager交换数据**。

### 分发器（Dispatcher）

​	可以跨作业运行，它为应用提交提供了REST接口。

​	当一个应用被提交执行时，分发器就会启动并将应用移交给一个JobManager。由于是REST接口，所以Dispatcher可以作为集群的一个HTTP接入点，这样就能够不受防火墙阻挡。Dispatcher也会启动一个Web UI，用来方便地展示和监控作业执行的信息。

​	*Dispatcher在架构中可能并不是必需的，这取决于应用提交运行的方式。*

## 4.2 任务提交流程

​	我们来看看当一个应用提交执行时，Flink的各个组件是如何交互协作的：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524212126844.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

​	*ps：上图中7.指TaskManager为JobManager提供slots，8.表示JobManager提交要在slots中执行的任务给TaskManager。*

​	上图是从一个较为高层级的视角来看应用中各组件的交互协作。

​	如果部署的集群环境不同（例如YARN，Mesos，Kubernetes，standalone等），其中一些步骤可以被省略，或是有些组件会运行在同一个JVM进程中。

​	具体地，如果我们将Flink集群部署到YARN上，那么就会有如下的提交流程：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524212247873.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

1. Flink任务提交后，Client向HDFS上传Flink的Jar包和配置
2. 之后客户端向Yarn ResourceManager提交任务，ResourceManager分配Container资源并通知对应的NodeManager启动ApplicationMaster
3. ApplicationMaster启动后加载Flink的Jar包和配置构建环境，去启动JobManager，之后**JobManager向Flink自身的RM进行申请资源，自身的RM向Yarn 的ResourceManager申请资源(因为是yarn模式，所有资源归yarn RM管理)启动TaskManager**
4. Yarn ResourceManager分配Container资源后，由ApplicationMaster通知资源所在节点的NodeManager启动TaskManager
5. NodeManager加载Flink的Jar包和配置构建环境并启动TaskManager，TaskManager启动后向JobManager发送心跳包，并等待JobManager向其分配任务。

## 4.3 任务调度原理

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524213145755.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

1. 客户端不是运行时和程序执行的一部分，但它用于准备并发送dataflow(JobGraph)给Master(JobManager)，然后，客户端断开连接或者维持连接以等待接收计算结果。而Job Manager会产生一个执行图(Dataflow Graph)

2. 当 Flink 集群启动后，首先会启动一个 JobManger 和一个或多个的 TaskManager。由 Client 提交任务给 JobManager，JobManager 再调度任务到各个 TaskManager 去执行，然后 TaskManager 将心跳和统计信息汇报给 JobManager。TaskManager 之间以流的形式进行数据的传输。上述三者均为独立的 JVM 进程。

3. Client 为提交 Job 的客户端，可以是运行在任何机器上（与 JobManager 环境连通即可）。提交 Job 后，Client 可以结束进程（Streaming的任务），也可以不结束并等待结果返回。

4. JobManager 主要负责调度 Job 并协调 Task 做 checkpoint，职责上很像 Storm 的 Nimbus。从 Client 处接收到 Job 和 JAR 包等资源后，会生成优化后的执行计划，并以 Task 的单元调度到各个 TaskManager 去执行。

5. TaskManager 在启动的时候就设置好了槽位数（Slot），每个 slot 能启动一个 Task，Task 为线程。从 JobManager 处接收需要部署的 Task，部署启动后，与自己的上游建立 Netty 连接，接收数据并处理。

   *注：如果一个Slot中启动多个线程，那么这几个线程类似CPU调度一样共用同一个slot*

### 4.3.1 TaskManger与Slots

要点：

+ 考虑到Slot分组，所以实际运行Job时所需的Slot总数 = 每个Slot组中的最大并行度。

  eg（1，1，2，1）,其中第一个归为组“red”、第二个归组“blue”、第三个和第四归组“green”，那么运行所需的slot即max（1）+max（1）+max（2，1） =  1+1+2  = 4

---

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524213557113.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ <u>Flink中每一个worker(TaskManager)都是一个**JVM**进程，它可能会在独立的线程上执行一个或多个subtask</u>。
+ 为了控制一个worker能接收多少个task，worker通过task slot来进行控制（一个worker至少有一个task slot）。

**上图这个每个子任务各自占用一个slot，可以在代码中通过算子的`.slotSharingGroup("组名")`指定算子所在的Slot组名，默认每一个算子的SlotGroup和上一个算子相同，而默认的SlotGroup就是"default"**。

**同一个SlotGroup的算子能共享同一个slot，不同组则必须另外分配独立的Slot。**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524214555469.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ 默认情况下，Flink允许子任务共享slot，即使它们是不同任务的子任务（前提需要来自同一个Job）。这样结果是，**一个slot可以保存作业的整个管道pipeline**。

  + **不同任务共享同一个Slot的前提：这几个任务前后顺序不同，如上图中Source和keyBy是两个不同步骤顺序的任务，所以可以在同一个Slot执行**。
  + 一个slot可以保存作业的整个管道的好处：
    + 如果有某个slot执行完了整个任务流程，那么其他任务就可以不用继续了，这样也省去了跨slot、跨TaskManager的通信损耗（降低了并行度）
    + 同时slot能够保存整个管道，使得整个任务执行健壮性更高，因为某些slot执行出异常也能有其他slot补上。
    + 有些slot分配到的子任务非CPU密集型，有些则CPU密集型，如果每个slot只完成自己的子任务，将出现某些slot太闲，某些slot过忙的现象。

  + *假设拆分的多个Source子任务放到同一个Slot，那么任务不能并行执行了=>因为多个相同步骤的子任务需要抢占的具体资源相同，比如抢占某个锁，这样就不能并行。*

+ Task Slot是静态的概念，是指TaskManager具有的并发执行能力，可以通过参数`taskmanager.numberOfTaskSlots`进行配置。

  *而并行度**parallelism**是动态概念，即**TaskManager**运行程序时实际使用的并发能力，可以通过参数`parallelism.default`进行配置。*

​	每个task slot表示TaskManager拥有资源的一个固定大小的子集。假如一个TaskManager有三个slot，那么它会将其管理的内存分成三份给各个slot。资源slot化意味着一个subtask将不需要跟来自其他job的subtask竞争被管理的内存，取而代之的是它将拥有一定数量的内存储备。

​	**需要注意的是，这里不会涉及到CPU的隔离，slot目前仅仅用来隔离task的受管理的内存**。

​	通过调整task slot的数量，允许用户定义subtask之间如何互相隔离。如果一个TaskManager一个slot，那将意味着每个task group运行在独立的JVM中（该JVM可能是通过一个特定的容器启动的），而一个TaskManager多个slot意味着更多的subtask可以共享同一个JVM。<u>而在同一个JVM进程中的task将共享TCP连接（基于多路复用）和心跳消息。它们也可能共享数据集和数据结构，因此这减少了每个task的负载。</u>

### 4.3.2 Slot和并行度

1. **一个特定算子的 子任务（subtask）的个数被称之为其并行度（parallelism）**，我们可以对单独的每个算子进行设置并行度，也可以直接用env设置全局的并行度，更可以在页面中去指定并行度。
2. 最后，由于并行度是实际Task Manager处理task 的能力，而一般情况下，**一个 stream 的并行度，可以认为就是其所有算子中最大的并行度**，则可以得出**在设置Slot时，在所有设置中的最大设置的并行度大小则就是所需要设置的Slot的数量。**（如果Slot分组，则需要为每组Slot并行度最大值的和）

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524215554488.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020052421520496.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524215251380.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

​	假设一共有3个TaskManager，每一个TaskManager中的分配3个TaskSlot，也就是每个TaskManager可以接收3个task，一共9个TaskSlot，如果我们设置`parallelism.default=1`，即运行程序默认的并行度为1，9个TaskSlot只用了1个，有8个空闲，因此，设置合适的并行度才能提高效率。

​	*ps：上图最后一个因为是输出到文件，避免多个Slot（多线程）里的算子都输出到同一个文件互相覆盖等混乱问题，直接设置sink的并行度为1。*

### 4.3.3 程序和数据流（DataFlow）

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524215944234.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ **所有的Flink程序都是由三部分组成的： Source 、Transformation 和 Sink。**

+ Source 负责读取数据源，Transformation 利用各种算子进行处理加工，Sink 负责输出

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524220037630.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ 在运行时，Flink上运行的程序会被映射成“逻辑数据流”（dataflows），它包含了这三部分

+ 每一个dataflow以一个或多个sources开始以一个或多个sinks结束。dataflow类似于任意的有向无环图（DAG）

+ 在大部分情况下，程序中的转换运算（transformations）跟dataflow中的算子（operator）是一一对应的关系

### 4.3.4 执行图（**ExecutionGraph**）

​	由Flink程序直接映射成的数据流图是StreamGraph，也被称为**逻辑流图**，因为它们表示的是计算逻辑的高级视图。为了执行一个流处理程序，Flink需要将**逻辑流图**转换为**物理数据流图**（也叫**执行图**），详细说明程序的执行方式。

+ Flink 中的执行图可以分成四层：StreamGraph -> JobGraph -> ExecutionGraph -> 物理执行图。

  + **StreamGraph**：是根据用户通过Stream API 编写的代码生成的最初的图。用来表示程序的拓扑结构。

  + **JobGraph**：StreamGraph经过优化后生成了JobGraph，提交给JobManager 的数据结构。主要的优化为，将多个符合条件的节点chain 在一起作为一个节点，这样可以减少数据在节点之间流动所需要的序列化/反序列化/传输消耗。

  + **ExecutionGraph**：JobManager 根据JobGraph 生成ExecutionGraph。ExecutionGraph是JobGraph的并行化版本，是调度层最核心的数据结构。

  + 物理执行图：JobManager 根据ExecutionGraph 对Job 进行调度后，在各个TaskManager 上部署Task 后形成的“图”，并不是一个具体的数据结构。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524220232635.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

### 4.3.5 数据传输形式

+ 一个程序中，不同的算子可能具有不同的并行度

+ 算子之间传输数据的形式可以是 one-to-one (forwarding) 的模式也可以是redistributing 的模式，具体是哪一种形式，取决于算子的种类

  + **One-to-one**：stream维护着分区以及元素的顺序（比如source和map之间）。这意味着map 算子的子任务看到的元素的个数以及顺序跟 source 算子的子任务生产的元素的个数、顺序相同。**map、fliter、flatMap等算子都是one-to-one的对应关系**。

  + **Redistributing**：stream的分区会发生改变。每一个算子的子任务依据所选择的transformation发送数据到不同的目标任务。例如，keyBy 基于 hashCode 重分区、而 broadcast 和 rebalance 会随机重新分区，这些算子都会引起redistribute过程，而 redistribute 过程就类似于 Spark 中的 shuffle 过程。

### 4.3.6 任务链（OperatorChains）

​	Flink 采用了一种称为任务链的优化技术，可以在特定条件下减少本地通信的开销。为了满足任务链的要求，必须将两个或多个算子设为**相同的并行度**，并通过本地转发（local forward）的方式进行连接

+ **相同并行度**的 **one-to-one 操作**，Flink 这样相连的算子链接在一起形成一个 task，原来的算子成为里面的 subtask
  + 并行度相同、并且是 one-to-one 操作，两个条件缺一不可

​	**为什么需要并行度相同，因为若flatMap并行度为1，到了之后的map并行度为2，从flatMap到map的数据涉及到数据由于并行度map为2会往两个slot处理，数据会分散，所产生的元素个数和顺序发生的改变所以有2个单独的task，不能成为任务链**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524220815415.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

​	**如果前后任务逻辑上可以是OneToOne，且并行度一致，那么就能合并在一个Slot里**（并行度原本是多少就是多少，两者并行度一致）执行。

+ keyBy需要根据Hash值分配给不同slot执行，所以只能Hash，不能OneToOne。
+ 逻辑上可OneToOne但是并行度不同，那么就会Rebalance，轮询形式分配给下一个任务的多个slot。

---

+ **代码中如果`算子.disableChaining()`，能够强制当前算子的子任务不参与任务链的合并，即不和其他Slot资源合并，但是仍然可以保留“Slot共享”的特性**。

+ **如果`StreamExecutionEnvironment env.disableOperatorChaining()`则当前执行环境全局设置算子不参与"任务链的合并"。**

+ **如果`算子.startNewChain()`表示不管前面任务链合并与否，从当前算子往后重新计算任务链的合并。通常用于前面强制不要任务链合并，而当前往后又需要任务链合并的特殊场景。**

*ps：如果`算子.shuffle()`，能够强制算子之后重分区到不同slot执行下一个算子操作，逻辑上也实现了任务不参与任务链合并=>但是仅为“不参与任务链的合并”，这个明显不是最优解操作*

> [Flink slotSharingGroup disableChain startNewChain 用法案例](https://blog.csdn.net/qq_31866793/article/details/102786249)

# 5. Flink流处理API

## 5.1 Environment

![img](https://img-blog.csdnimg.cn/20191124113558631.png)

### 5.1.1 getExecutionEnvironment

​	创建一个执行环境，表示当前执行程序的上下文。如果程序是独立调用的，则此方法返回本地执行环境；如果从命令行客户端调用程序以提交到集群，则此方法返回此集群的执行环境，也就是说，getExecutionEnvironment会根据查询运行的方式决定返回什么样的运行环境，是最常用的一种创建执行环境的方式。

`ExecutionEnvironment env = ExecutionEnvironment.*getExecutionEnvironment*(); `

`StreamExecutionEnvironment env = StreamExecutionEnvironment.*getExecutionEnvironment*(); `

如果没有设置并行度，会以flink-conf.yaml中的配置为准，默认是1。

![img](https://img-blog.csdnimg.cn/20191124113636435.png)

### 5.1.2 createLocalEnvironment

​	返回本地执行环境，需要在调用时指定默认的并行度。

`LocalStreamEnvironment env = StreamExecutionEnvironment.*createLocalEnvironment*(1); `

### 5.1.3 createRemoteEnvironment

​	返回集群执行环境，将Jar提交到远程服务器。需要在调用时指定JobManager的IP和端口号，并指定要在集群中运行的Jar包。

`StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment(1);`

## 5.2 Source

> [Flink-Environment的三种方式和Source的四种读取方式-从集合中、从kafka中、从文件中、自定义](https://blog.csdn.net/qq_40180229/article/details/106335725)

### 5.2.1 从集合读取数据

java代码：

```java
package apitest.source;

import apitest.beans.SensorReading;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

import java.util.Arrays;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2021/1/31 5:13 PM
 * 测试Flink从集合中获取数据
 */
public class SourceTest1_Collection {
    public static void main(String[] args) throws Exception {
        // 创建执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // 设置env并行度1，使得整个任务抢占同一个线程执行
        env.setParallelism(1);

        // Source: 从集合Collection中获取数据
        DataStream<SensorReading> dataStream = env.fromCollection(
                Arrays.asList(
                        new SensorReading("sensor_1", 1547718199L, 35.8),
                        new SensorReading("sensor_6", 1547718201L, 15.4),
                        new SensorReading("sensor_7", 1547718202L, 6.7),
                        new SensorReading("sensor_10", 1547718205L, 38.1)
                )
        );

        DataStream<Integer> intStream = env.fromElements(1,2,3,4,5,6,7,8,9);

        // 打印输出
        dataStream.print("SENSOR");
        intStream.print("INT");

        // 执行
        env.execute("JobName");

    }

}
```

输出：

```shell
INT> 1
INT> 2
SENSOR> SensorReading{id='sensor_1', timestamp=1547718199, temperature=35.8}
INT> 3
SENSOR> SensorReading{id='sensor_6', timestamp=1547718201, temperature=15.4}
INT> 4
SENSOR> SensorReading{id='sensor_7', timestamp=1547718202, temperature=6.7}
INT> 5
SENSOR> SensorReading{id='sensor_10', timestamp=1547718205, temperature=38.1}
INT> 6
INT> 7
INT> 8
INT> 9
```

### 5.2.2 从文件读取数据

java代码如下：

```java
package apitest.source;

import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2021/1/31 5:26 PM
 * Flink从文件中获取数据
 */
public class SourceTest2_File {
    public static void main(String[] args) throws Exception {
        // 创建执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // 使得任务抢占同一个线程
        env.setParallelism(1);

        // 从文件中获取数据输出
        DataStream<String> dataStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");

        dataStream.print();

        env.execute();
    }
}

```

sensor.txt文件内容

```txt
sensor_1,1547718199,35.8
sensor_6,1547718201,15.4
sensor_7,1547718202,6.7
sensor_10,1547718205,38.1
sensor_1,1547718207,36.3
sensor_1,1547718209,32.8
sensor_1,1547718212,37.1
```

输出：

```shell
sensor_1,1547718199,35.8
sensor_6,1547718201,15.4
sensor_7,1547718202,6.7
sensor_10,1547718205,38.1
sensor_1,1547718207,36.3
sensor_1,1547718209,32.8
sensor_1,1547718212,37.1
```

### 5.2.3 从Kafka读取数据

1. pom依赖

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
       <modelVersion>4.0.0</modelVersion>
   
       <groupId>org.example</groupId>
       <artifactId>Flink_Tutorial</artifactId>
       <version>1.0-SNAPSHOT</version>
   
       <properties>
           <maven.compiler.source>8</maven.compiler.source>
           <maven.compiler.target>8</maven.compiler.target>
           <flink.version>1.12.1</flink.version>
           <scala.binary.version>2.12</scala.binary.version>
       </properties>
   
       <dependencies>
           <dependency>
               <groupId>org.apache.flink</groupId>
               <artifactId>flink-java</artifactId>
               <version>${flink.version}</version>
           </dependency>
           <dependency>
               <groupId>org.apache.flink</groupId>
               <artifactId>flink-streaming-scala_${scala.binary.version}</artifactId>
               <version>${flink.version}</version>
           </dependency>
           <dependency>
               <groupId>org.apache.flink</groupId>
               <artifactId>flink-clients_${scala.binary.version}</artifactId>
               <version>${flink.version}</version>
           </dependency>
   
           <!-- kafka -->
           <dependency>
               <groupId>org.apache.flink</groupId>
               <artifactId>flink-connector-kafka_${scala.binary.version}</artifactId>
               <version>${flink.version}</version>
           </dependency>
       </dependencies>
   </project>
   ```

2. 启动zookeeper

   ```shell
   $ bin/zookeeper-server-start.sh config/zookeeper.properties
   ```

3. 启动kafka服务

   ```shell
   $ bin/kafka-server-start.sh config/server.properties
   ```

4. 启动kafka生产者

   ```shell
   $ bin/kafka-console-producer.sh --broker-list localhost:9092  --topic sensor
   ```

5. 编写java代码

   ```java
   package apitest.source;
   
   import org.apache.flink.api.common.serialization.SimpleStringSchema;
   import org.apache.flink.streaming.api.datastream.DataStream;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer;
   
   import java.util.Properties;
   
   /**
    * @author : Ashiamd email: ashiamd@foxmail.com
    * @date : 2021/1/31 5:44 PM
    */
   public class SourceTest3_Kafka {
   
       public static void main(String[] args) throws Exception {
           // 创建执行环境
           StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
   
           // 设置并行度1
           env.setParallelism(1);
   
           Properties properties = new Properties();
           properties.setProperty("bootstrap.servers", "localhost:9092");
           // 下面这些次要参数
           properties.setProperty("group.id", "consumer-group");
           properties.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
           properties.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
           properties.setProperty("auto.offset.reset", "latest");
   
           // flink添加外部数据源
           DataStream<String> dataStream = env.addSource(new FlinkKafkaConsumer<String>("sensor", new SimpleStringSchema(),properties));
   
           // 打印输出
           dataStream.print();
   
           env.execute();
       }
   }
   ```

6. 运行java代码，在Kafka生产者console中输入

   ```shell
   $ bin/kafka-console-producer.sh --broker-list localhost:9092  --topic sensor
   >sensor_1,1547718199,35.8
   >sensor_6,1547718201,15.4
   >
   ```

7. java输出

   ```shell
   sensor_1,1547718199,35.8
   sensor_6,1547718201,15.4
   ```

### 5.2.4 自定义Source

java代码：

```java
package apitest.source;

import apitest.beans.SensorReading;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.source.SourceFunction;

import java.util.HashMap;
import java.util.Random;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2021/1/31 6:44 PM
 */
public class SourceTest4_UDF {
    public static void main(String[] args) throws Exception {
        // 创建执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        DataStream<SensorReading> dataStream = env.addSource(new MySensorSource());

        dataStream.print();

        env.execute();
    }

    // 实现自定义的SourceFunction
    public static class MySensorSource implements SourceFunction<SensorReading> {

        // 标示位，控制数据产生
        private volatile boolean running = true;


        @Override
        public void run(SourceContext<SensorReading> ctx) throws Exception {
            //定义一个随机数发生器
            Random random = new Random();

            // 设置10个传感器的初始温度
            HashMap<String, Double> sensorTempMap = new HashMap<>();
            for (int i = 0; i < 10; ++i) {
                sensorTempMap.put("sensor_" + (i + 1), 60 + random.nextGaussian() * 20);
            }

            while (running) {
                for (String sensorId : sensorTempMap.keySet()) {
                    // 在当前温度基础上随机波动
                    Double newTemp = sensorTempMap.get(sensorId) + random.nextGaussian();
                    sensorTempMap.put(sensorId, newTemp);
                    ctx.collect(new SensorReading(sensorId,System.currentTimeMillis(),newTemp));
                }
                // 控制输出评率
                Thread.sleep(2000L);
            }
        }

        @Override
        public void cancel() {
            this.running = false;
        }
    }
}
```

输出：

```shell
7> SensorReading{id='sensor_9', timestamp=1612091759321, temperature=83.80320976056609}
15> SensorReading{id='sensor_10', timestamp=1612091759321, temperature=68.77967856820972}
1> SensorReading{id='sensor_1', timestamp=1612091759321, temperature=45.75304941852771}
6> SensorReading{id='sensor_6', timestamp=1612091759321, temperature=71.80036477804133}
3> SensorReading{id='sensor_7', timestamp=1612091759321, temperature=55.262086521569564}
2> SensorReading{id='sensor_2', timestamp=1612091759321, temperature=64.0969570576537}
5> SensorReading{id='sensor_5', timestamp=1612091759321, temperature=51.09761352612651}
14> SensorReading{id='sensor_3', timestamp=1612091759313, temperature=32.49085393551031}
4> SensorReading{id='sensor_8', timestamp=1612091759321, temperature=64.83732456896752}
16> SensorReading{id='sensor_4', timestamp=1612091759321, temperature=88.88318538017865}
12> SensorReading{id='sensor_2', timestamp=1612091761325, temperature=65.21522804626638}
16> SensorReading{id='sensor_6', timestamp=1612091761325, temperature=70.49210870668041}
15> SensorReading{id='sensor_5', timestamp=1612091761325, temperature=50.32349231082738}
....
```

## 5.3 Transform

map、flatMap、filter通常被统一称为**基本转换算子**（**简单转换算子**）。

### 5.3.1 基本转换算子(map/flatMap/filter)

> [到处是map、flatMap，啥意思？](https://zhuanlan.zhihu.com/p/66196174)

java代码：

```java
package apitest.transform;

import org.apache.flink.api.common.functions.FilterFunction;
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.Collector;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2021/1/31 7:31 PM
 */
public class TransformTest1_Base {
    public static void main(String[] args) throws Exception {
        // 创建执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // 使得任务抢占同一个线程
        env.setParallelism(1);

        // 从文件中获取数据输出
        DataStream<String> dataStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");

        // 1. map, String => 字符串长度INT
        DataStream<Integer> mapStream = dataStream.map(new MapFunction<String, Integer>() {
            @Override
            public Integer map(String value) throws Exception {
                return value.length();
            }
        });

        // 2. flatMap，按逗号分割字符串
        DataStream<String> flatMapStream = dataStream.flatMap(new FlatMapFunction<String, String>() {
            @Override
            public void flatMap(String value, Collector<String> out) throws Exception {
                String[] fields = value.split(",");
                for(String field:fields){
                    out.collect(field);
                }
            }
        });

        // 3. filter,筛选"sensor_1"开头的数据
        DataStream<String> filterStream = dataStream.filter(new FilterFunction<String>() {
            @Override
            public boolean filter(String value) throws Exception {
                return value.startsWith("sensor_1");
            }
        });

        // 打印输出
        mapStream.print("map");
        flatMapStream.print("flatMap");
        filterStream.print("filter");

        env.execute();
    }
}

```

输出：

```shell
map> 24
flatMap> sensor_1
flatMap> 1547718199
flatMap> 35.8
filter> sensor_1,1547718199,35.8
map> 24
flatMap> sensor_6
flatMap> 1547718201
flatMap> 15.4
map> 23
flatMap> sensor_7
flatMap> 1547718202
flatMap> 6.7
map> 25
flatMap> sensor_10
flatMap> 1547718205
flatMap> 38.1
filter> sensor_10,1547718205,38.1
map> 24
flatMap> sensor_1
flatMap> 1547718207
flatMap> 36.3
filter> sensor_1,1547718207,36.3
map> 24
flatMap> sensor_1
flatMap> 1547718209
flatMap> 32.8
filter> sensor_1,1547718209,32.8
map> 24
flatMap> sensor_1
flatMap> 1547718212
flatMap> 37.1
filter> sensor_1,1547718212,37.1
```

### 5.3.2 聚合操作算子

> [Flink_Trasform算子](https://blog.csdn.net/dongkang123456/article/details/108361376)

+ DataStream里没有reduce和sum这类聚合操作的方法，因为**Flink设计中，所有数据必须先分组才能做聚合操作**。
+ **先keyBy得到KeyedStream，然后调用其reduce、sum等聚合操作方法。（先分组后聚合）**

---

常见的聚合操作算子主要有：

+ keyBy
+ 滚动聚合算子Rolling Aggregation

+ reduce

---

#### keyBy

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200902141943335.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvbmdrYW5nMTIzNDU2,size_16,color_FFFFFF,t_70#pic_center)

**DataStream -> KeyedStream**：逻辑地将一个流拆分成不相交的分区，每个分区包含具有相同key的元素，在内部以hash的形式实现的。

1、KeyBy会重新分区；
2、不同的key有可能分到一起，因为是通过hash原理实现的；

#### Rolling Aggregation

这些算子可以针对KeyedStream的每一个支流做聚合。

+ sum()
+ min()
+ max()
+ minBy()
+ maxBy()

---

测试maxBy的java代码一

```java
package apitest.transform;

import apitest.beans.SensorReading;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.KeyedStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2021/1/31 9:51 PM
 * 滚动聚合，测试
 */
public class TransformTest2_RollingAggregation {
    public static void main(String[] args) throws Exception {
        // 创建 执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // 执行环境并行度设置1
        env.setParallelism(1);

        DataStream<String> dataStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");

//        DataStream<SensorReading> sensorStream = dataStream.map(new MapFunction<String, SensorReading>() {
//            @Override
//            public SensorReading map(String value) throws Exception {
//                String[] fields = value.split(",");
//                return new SensorReading(fields[0],new Long(fields[1]),new Double(fields[2]));
//            }
//        });

        DataStream<SensorReading> sensorStream = dataStream.map(line -> {
            String[] fields = line.split(",");
            return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
        });
        // 先分组再聚合
        // 分组
        KeyedStream<SensorReading, String> keyedStream = sensorStream.keyBy(SensorReading::getId);

        // 滚动聚合，max和maxBy区别在于，maxBy除了用于max比较的字段以外，其他字段也会更新成最新的，而max只有比较的字段更新，其他字段不变
        DataStream<SensorReading> resultStream = keyedStream.maxBy("temperature");

        resultStream.print("result");

        env.execute();
    }
}
```

其中`sensor.txt`文件内容如下

```txt
sensor_1,1547718199,35.8
sensor_6,1547718201,15.4
sensor_7,1547718202,6.7
sensor_10,1547718205,38.1
sensor_1,1547718207,36.3
sensor_1,1547718209,32.8
sensor_1,1547718212,37.1
```

输出如下：

*由于是滚动更新，每次输出历史最大值，所以下面36.3才会出现两次*

```shell
result> SensorReading{id='sensor_1', timestamp=1547718199, temperature=35.8}
result> SensorReading{id='sensor_6', timestamp=1547718201, temperature=15.4}
result> SensorReading{id='sensor_7', timestamp=1547718202, temperature=6.7}
result> SensorReading{id='sensor_10', timestamp=1547718205, temperature=38.1}
result> SensorReading{id='sensor_1', timestamp=1547718207, temperature=36.3}
result> SensorReading{id='sensor_1', timestamp=1547718207, temperature=36.3}
result> SensorReading{id='sensor_1', timestamp=1547718212, temperature=37.1}
```

#### reduce

​	**Reduce适用于更加一般化的聚合操作场景**。java中需要实现`ReduceFunction`函数式接口。

---

​	在前面Rolling Aggregation的前提下，对需求进行修改。获取同组历史温度最高的传感器信息，同时要求实时更新其时间戳信息。

java代码如下：

```java
package apitest.transform;

import apitest.beans.SensorReading;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.KeyedStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.kafka.common.metrics.stats.Max;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2021/1/31 10:14 PM
 * 复杂场景，除了获取最大温度的整个传感器信息以外，还要求时间戳更新成最新的
 */
public class TransformTest3_Reduce {
    public static void main(String[] args) throws Exception {
        // 创建 执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // 执行环境并行度设置1
        env.setParallelism(1);

        DataStream<String> dataStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");

        DataStream<SensorReading> sensorStream = dataStream.map(line -> {
            String[] fields = line.split(",");
            return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
        });
        // 先分组再聚合
        // 分组
        KeyedStream<SensorReading, String> keyedStream = sensorStream.keyBy(SensorReading::getId);

        // reduce，自定义规约函数，获取max温度的传感器信息以外，时间戳要求更新成最新的
        DataStream<SensorReading> resultStream = keyedStream.reduce(
                (curSensor,newSensor)->new SensorReading(curSensor.getId(),newSensor.getTimestamp(), Math.max(curSensor.getTemperature(), newSensor.getTemperature()))
        );

        resultStream.print("result");

        env.execute();
    }
}
```

`sensor.txt`文件内容如下：

```txt
sensor_1,1547718199,35.8
sensor_6,1547718201,15.4
sensor_7,1547718202,6.7
sensor_10,1547718205,38.1
sensor_1,1547718207,36.3
sensor_1,1547718209,32.8
sensor_1,1547718212,37.1
```

输出如下：

*和前面“Rolling Aggregation”小节不同的是，倒数第二条数据的时间戳用了当前比较时最新的时间戳。*

```shell
result> SensorReading{id='sensor_1', timestamp=1547718199, temperature=35.8}
result> SensorReading{id='sensor_6', timestamp=1547718201, temperature=15.4}
result> SensorReading{id='sensor_7', timestamp=1547718202, temperature=6.7}
result> SensorReading{id='sensor_10', timestamp=1547718205, temperature=38.1}
result> SensorReading{id='sensor_1', timestamp=1547718207, temperature=36.3}
result> SensorReading{id='sensor_1', timestamp=1547718209, temperature=36.3}
result> SensorReading{id='sensor_1', timestamp=1547718212, temperature=37.1}
```

### 5.3.3 多流转换算子

> [Flink_Trasform算子](https://blog.csdn.net/dongkang123456/article/details/108361376)

多流转换算子一般包括：

+ Split和Select （新版已经移除）
+ Connect和CoMap

+ Union

#### Split和Select

**注：新版Flink已经不存在Split和Select这两个API了（至少Flink1.12.1没有！）**

##### Split
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200902194203248.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvbmdrYW5nMTIzNDU2,size_16,color_FFFFFF,t_70#pic_center)
**DataStream -> SplitStream**：根据某些特征把DataStream拆分成SplitStream;

**SplitStream虽然看起来像是两个Stream，但是其实它是一个特殊的Stream**;

##### Select
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200902194442828.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvbmdrYW5nMTIzNDU2,size_16,color_FFFFFF,t_70#pic_center)
**SplitStream -> DataStream**：从一个SplitStream中获取一个或者多个DataStream;

**我们可以结合split&select将一个DataStream拆分成多个DataStream。**

---

测试场景：根据传感器温度高低，划分成两组，high和low（>30归入high）：

*这个我发现在Flink当前时间最新版1.12.1已经不是DataStream的方法了，被去除了*

这里直接附上教程代码（Flink1.10.1）

```java
package com.atguigu.apitest.transform;/**
 * Copyright (c) 2018-2028 尚硅谷 All Rights Reserved
 * <p>
 * Project: FlinkTutorial
 * Package: com.atguigu.apitest.transform
 * Version: 1.0
 * <p>
 * Created by wushengran on 2020/11/7 16:14
 */

import com.atguigu.apitest.beans.SensorReading;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.api.java.tuple.Tuple3;
import org.apache.flink.streaming.api.collector.selector.OutputSelector;
import org.apache.flink.streaming.api.datastream.ConnectedStreams;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.datastream.SplitStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.co.CoMapFunction;

import java.util.Collections;

/**
 * @ClassName: TransformTest4_MultipleStreams
 * @Description:
 * @Author: wushengran on 2020/11/7 16:14
 * @Version: 1.0
 */
public class TransformTest4_MultipleStreams {
  public static void main(String[] args) throws Exception {
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    env.setParallelism(1);

    // 从文件读取数据
    DataStream<String> inputStream = env.readTextFile("D:\\Projects\\BigData\\FlinkTutorial\\src\\main\\resources\\sensor.txt");

    // 转换成SensorReading
    DataStream<SensorReading> dataStream = inputStream.map(line -> {
      String[] fields = line.split(",");
      return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
    } );

    // 1. 分流，按照温度值30度为界分为两条流
    SplitStream<SensorReading> splitStream = dataStream.split(new OutputSelector<SensorReading>() {
      @Override
      public Iterable<String> select(SensorReading value) {
        return (value.getTemperature() > 30) ? Collections.singletonList("high") : Collections.singletonList("low");
      }
    });

    DataStream<SensorReading> highTempStream = splitStream.select("high");
    DataStream<SensorReading> lowTempStream = splitStream.select("low");
    DataStream<SensorReading> allTempStream = splitStream.select("high", "low");

    highTempStream.print("high");
    lowTempStream.print("low");
    allTempStream.print("all");
    
    env.execute();
  }
}
```

输出结果如下：

```shell
high> SensorReading{id='sensor_1', timestamp=1547718199, temperature=35.8}
all > SensorReading{id='sensor_1', timestamp=1547718199, temperature=35.8}
low > SensorReading{id='sensor_6', timestamp=1547718201, temperature=15.4}
all > SensorReading{id='sensor_6', timestamp=1547718201, temperature=15.4}
...
```

#### Connect和CoMap

##### Connect

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200902202832986.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvbmdrYW5nMTIzNDU2,size_16,color_FFFFFF,t_70#pic_center)
**DataStream,DataStream -> ConnectedStreams**: 连接两个保持他们类型的数据流，两个数据流被Connect 之后，只是被放在了一个流中，内部依然保持各自的数据和形式不发生任何变化，两个流相互独立。

##### CoMap

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200902203333640.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvbmdrYW5nMTIzNDU2,size_16,color_FFFFFF,t_70#pic_center)
**ConnectedStreams -> DataStream**: 作用于ConnectedStreams 上，功能与map和flatMap一样，对ConnectedStreams 中的**每一个Stream分别进行map和flatMap操作**；

---

虽然Flink1.12.1的DataStream有connect和map方法，但是教程基于前面的split和select编写，所以这里直接附上教程的代码：

```java
package com.atguigu.apitest.transform;/**
 * Copyright (c) 2018-2028 尚硅谷 All Rights Reserved
 * <p>
 * Project: FlinkTutorial
 * Package: com.atguigu.apitest.transform
 * Version: 1.0
 * <p>
 * Created by wushengran on 2020/11/7 16:14
 */

import com.atguigu.apitest.beans.SensorReading;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.api.java.tuple.Tuple3;
import org.apache.flink.streaming.api.collector.selector.OutputSelector;
import org.apache.flink.streaming.api.datastream.ConnectedStreams;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.datastream.SplitStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.co.CoMapFunction;

import java.util.Collections;

/**
 * @ClassName: TransformTest4_MultipleStreams
 * @Description:
 * @Author: wushengran on 2020/11/7 16:14
 * @Version: 1.0
 */
public class TransformTest4_MultipleStreams {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        // 从文件读取数据
        DataStream<String> inputStream = env.readTextFile("D:\\Projects\\BigData\\FlinkTutorial\\src\\main\\resources\\sensor.txt");

        // 转换成SensorReading
        DataStream<SensorReading> dataStream = inputStream.map(line -> {
            String[] fields = line.split(",");
            return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
        } );

        // 1. 分流，按照温度值30度为界分为两条流
        SplitStream<SensorReading> splitStream = dataStream.split(new OutputSelector<SensorReading>() {
            @Override
            public Iterable<String> select(SensorReading value) {
                return (value.getTemperature() > 30) ? Collections.singletonList("high") : Collections.singletonList("low");
            }
        });

        DataStream<SensorReading> highTempStream = splitStream.select("high");
        DataStream<SensorReading> lowTempStream = splitStream.select("low");
        DataStream<SensorReading> allTempStream = splitStream.select("high", "low");

        // highTempStream.print("high");
        // lowTempStream.print("low");
        // allTempStream.print("all");

        // 2. 合流 connect，将高温流转换成二元组类型，与低温流连接合并之后，输出状态信息
        DataStream<Tuple2<String, Double>> warningStream = highTempStream.map(new MapFunction<SensorReading, Tuple2<String, Double>>() {
            @Override
            public Tuple2<String, Double> map(SensorReading value) throws Exception {
                return new Tuple2<>(value.getId(), value.getTemperature());
            }
        });

        ConnectedStreams<Tuple2<String, Double>, SensorReading> connectedStreams = warningStream.connect(lowTempStream);

        DataStream<Object> resultStream = connectedStreams.map(new CoMapFunction<Tuple2<String, Double>, SensorReading, Object>() {
            @Override
            public Object map1(Tuple2<String, Double> value) throws Exception {
                return new Tuple3<>(value.f0, value.f1, "high temp warning");
            }

            @Override
            public Object map2(SensorReading value) throws Exception {
                return new Tuple2<>(value.getId(), "normal");
            }
        });

        resultStream.print();
        
        env.execute();
    }
}
```

输出如下：

```shell
(sensor_1,35.8,high temp warning)
(sensor_6,normal)
(sensor_10,38.1,high temp warning)
(sensor_7,normal)
(sensor_1,36.3,high temp warning)
(sensor_1,32.8,high temp warning)
(sensor_1,37.1,high temp warning)
```

#### Union

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200902205220165.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvbmdrYW5nMTIzNDU2,size_16,color_FFFFFF,t_70#pic_center)

**DataStream -> DataStream**：对**两个或者两个以上**的DataStream进行Union操作，产生一个包含多有DataStream元素的新DataStream。

**问题：和Connect的区别？**

1. Connect 的数据类型可以不同，**Connect 只能合并两个流**；
2. **Union可以合并多条流，Union的数据结构必须是一样的**；

```java
// 3. union联合多条流
//        warningStream.union(lowTempStream); 这个不行，因为warningStream类型是DataStream<Tuple2<String, Double>>，而highTempStream是DataStream<SensorReading>
        highTempStream.union(lowTempStream, allTempStream);
```

### 5.3.4 算子转换

> [Flink常用算子Transformation（转换）](https://blog.csdn.net/a_drjiaoda/article/details/89357916)

​	在Storm中，我们常常用Bolt的层级关系来表示各个数据的流向关系，组成一个拓扑。

​	在Flink中，**Transformation算子就是将一个或多个DataStream转换为新的DataStream**，可以将多个转换组合成复杂的数据流拓扑。
​	如下图所示，DataStream会由不同的Transformation操作，转换、过滤、聚合成其他不同的流，从而完成我们的业务要求。

![img](https://img-blog.csdnimg.cn/20190417171341810.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FfZHJqaWFvZGE=,size_16,color_FFFFFF,t_70)

## 5.4 支持的数据类型

​	Flink流应用程序处理的是以数据对象表示的事件流。所以在Flink内部，我们需要能够处理这些对象。它们**需要被序列化和反序列化**，以便通过网络传送它们；或者从状态后端、检查点和保存点读取它们。为了有效地做到这一点，Flink需要明确知道应用程序所处理的数据类型。Flink使用类型信息的概念来表示数据类型，并为每个数据类型生成特定的序列化器、反序列化器和比较器。

​	Flink还具有一个类型提取系统，该系统分析函数的输入和返回类型，以自动获取类型信息，从而获得序列化器和反序列化器。但是，在某些情况下，例如lambda函数或泛型类型，需要显式地提供类型信息，才能使应用程序正常工作或提高其性能。

​	Flink支持Java和Scala中所有常见数据类型。使用最广泛的类型有以下几种。

### 5.4.1 基础数据类型

​	Flink支持所有的Java和Scala基础数据类型，Int, Double, Long, String, …

```java
DataStream<Integer> numberStream = env.fromElements(1, 2, 3, 4);
numberStream.map(data -> data * 2);
```

### 5.4.2 Java和Scala元组(Tuples)

java不像Scala天生支持元组Tuple类型，java的元组类型由Flink的包提供，默认提供Tuple0~Tuple25

```java
DataStream<Tuple2<String, Integer>> personStream = env.fromElements( 
  new Tuple2("Adam", 17), 
  new Tuple2("Sarah", 23) 
); 
personStream.filter(p -> p.f1 > 18);
```

### 5.4.3 Scala样例类(case classes)

```scala
case class Person(name:String,age:Int)

val numbers: DataStream[(String,Integer)] = env.fromElements(
  Person("张三",12),
  Person("李四"，23)
)
```

### 5.4.4 Java简单对象(POJO)

java的POJO这里要求必须提供无参构造函数

+ 成员变量要求都是public（或者private但是提供get、set方法）

```java
public class Person{
  public String name;
  public int age;
  public Person() {}
  public Person( String name , int age) {
    this.name = name;
    this.age = age;
  }
}
DataStream Pe rson > persons = env.fromElements(
  new Person (" Alex", 42),
  new Person (" Wendy",23)
);
```

### 5.4.5 其他(Arrays, Lists, Maps, Enums,等等)

Flink对Java和Scala中的一些特殊目的的类型也都是支持的，比如Java的ArrayList，HashMap，Enum等等。

## 5.5 实现UDF函数——更细粒度的控制流

### 5.5.1 函数类(Function Classes)

​	Flink暴露了所有UDF函数的接口(实现方式为接口或者抽象类)。例如MapFunction, FilterFunction, ProcessFunction等等。

​	下面例子实现了FilterFunction接口：

```java
DataStream<String> flinkTweets = tweets.filter(new FlinkFilter()); 
public static class FlinkFilter implements FilterFunction<String> { 
  @Override public boolean filter(String value) throws Exception { 
    return value.contains("flink");
  }
}
```

​	还可以将函数实现成匿名类

```java
DataStream<String> flinkTweets = tweets.filter(
  new FilterFunction<String>() { 
    @Override public boolean filter(String value) throws Exception { 
      return value.contains("flink"); 
    }
  }
);
```

​	我们filter的字符串"flink"还可以当作参数传进去。

```java
DataStream<String> tweets = env.readTextFile("INPUT_FILE "); 
DataStream<String> flinkTweets = tweets.filter(new KeyWordFilter("flink")); 
public static class KeyWordFilter implements FilterFunction<String> { 
  private String keyWord; 

  KeyWordFilter(String keyWord) { 
    this.keyWord = keyWord; 
  } 

  @Override public boolean filter(String value) throws Exception { 
    return value.contains(this.keyWord); 
  } 
}
```

### 5.5.2 匿名函数(Lambda Functions)

```java
DataStream<String> tweets = env.readTextFile("INPUT_FILE"); 
DataStream<String> flinkTweets = tweets.filter( tweet -> tweet.contains("flink") );
```

### 5.5.3 富函数(Rich Functions)

​	“富函数”是DataStream API提供的一个函数类的接口，所有Flink函数类都有其Rich版本。

​	**它与常规函数的不同在于，可以获取运行环境的上下文，并拥有一些生命周期方法，所以可以实现更复杂的功能**。

+ RichMapFunction

+ RichFlatMapFunction

+ RichFilterFunction

+ …

​	Rich Function有一个**生命周期**的概念。典型的生命周期方法有：

+ **`open()`方法是rich function的初始化方法，当一个算子例如map或者filter被调用之前`open()`会被调用。**

+ **`close()`方法是生命周期中的最后一个调用的方法，做一些清理工作。**

+ **`getRuntimeContext()`方法提供了函数的RuntimeContext的一些信息，例如函数执行的并行度，任务的名字，以及state状态**

```java
public static class MyMapFunction extends RichMapFunction<SensorReading, Tuple2<Integer, String>> { 

  @Override public Tuple2<Integer, String> map(SensorReading value) throws Exception {
    return new Tuple2<>(getRuntimeContext().getIndexOfThisSubtask(), value.getId()); 
  } 

  @Override public void open(Configuration parameters) throws Exception { 
    System.out.println("my map open"); // 以下可以做一些初始化工作，例如建立一个和HDFS的连接 
  } 

  @Override public void close() throws Exception { 
    System.out.println("my map close"); // 以下做一些清理工作，例如断开和HDFS的连接 
  } 
}
```

---

测试代码：

```java
package apitest.transform;

import apitest.beans.SensorReading;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.common.functions.RichMapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2021/2/1 12:21 AM
 */
public class TransformTest5_RichFunction {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(4);

        DataStream<String> inputStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");

        // 转换成SensorReading类型
        DataStream<SensorReading> dataStream = inputStream.map(line -> {
            String[] fields = line.split(",");
            return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
        });

        DataStream<Tuple2<String, Integer>> resultStream = dataStream.map( new MyMapper() );

        resultStream.print();

        env.execute();
    }

    // 传统的Function不能获取上下文信息，只能处理当前数据，不能和其他数据交互
    public static class MyMapper0 implements MapFunction<SensorReading, Tuple2<String, Integer>> {
        @Override
        public Tuple2<String, Integer> map(SensorReading value) throws Exception {
            return new Tuple2<>(value.getId(), value.getId().length());
        }
    }

    // 实现自定义富函数类（RichMapFunction是一个抽象类）
    public static class MyMapper extends RichMapFunction<SensorReading, Tuple2<String, Integer>> {
        @Override
        public Tuple2<String, Integer> map(SensorReading value) throws Exception {
//            RichFunction可以获取State状态
//            getRuntimeContext().getState();
            return new Tuple2<>(value.getId(), getRuntimeContext().getIndexOfThisSubtask());
        }

        @Override
        public void open(Configuration parameters) throws Exception {
            // 初始化工作，一般是定义状态，或者建立数据库连接
            System.out.println("open");
        }

        @Override
        public void close() throws Exception {
            // 一般是关闭连接和清空状态的收尾操作
            System.out.println("close");
        }
    }
}

```

输出如下：

由于设置了执行环境env的并行度为4，所以有4个slot执行自定义的RichFunction，输出4次open和close

```shell
open
open
open
open
4> (sensor_1,3)
4> (sensor_6,3)
close
2> (sensor_1,1)
2> (sensor_1,1)
close
3> (sensor_1,2)
close
1> (sensor_7,0)
1> (sensor_10,0)
close
```

## 5.6 数据重分区操作

重分区操作，在DataStream类中可以看到很多`Partitioner`字眼的类。

**其中`partitionCustom(...)`方法用于自定义重分区**。

java代码：

```java
package apitest.transform;

import apitest.beans.SensorReading;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2021/2/1 12:38 AM
 */
public class TransformTest6_Partition {
  public static void main(String[] args) throws Exception{

    // 创建执行环境
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

    // 设置并行度 = 4
    env.setParallelism(4);

    // 从文件读取数据
    DataStream<String> inputStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");

    // 转换成SensorReading类型
    DataStream<SensorReading> dataStream = inputStream.map(line -> {
      String[] fields = line.split(",");
      return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
    });

    // SingleOutputStreamOperator多并行度默认就rebalance,轮询方式分配
    dataStream.print("input");

    // 1. shuffle (并非批处理中的获取一批后才打乱，这里每次获取到直接打乱且分区)
    DataStream<String> shuffleStream = inputStream.shuffle();
    shuffleStream.print("shuffle");

    // 2. keyBy (Hash，然后取模)
    dataStream.keyBy(SensorReading::getId).print("keyBy");

    // 3. global (直接发送给第一个分区，少数特殊情况才用)
    dataStream.global().print("global");

    env.execute();
  }
}
```

输出：

```shell
input:3> SensorReading{id='sensor_1', timestamp=1547718199, temperature=35.8}
input:3> SensorReading{id='sensor_6', timestamp=1547718201, temperature=15.4}
input:1> SensorReading{id='sensor_1', timestamp=1547718207, temperature=36.3}
input:1> SensorReading{id='sensor_1', timestamp=1547718209, temperature=32.8}
shuffle:2> sensor_6,1547718201,15.4
shuffle:1> sensor_1,1547718199,35.8
input:4> SensorReading{id='sensor_7', timestamp=1547718202, temperature=6.7}
input:4> SensorReading{id='sensor_10', timestamp=1547718205, temperature=38.1}
shuffle:1> sensor_1,1547718207,36.3
shuffle:2> sensor_1,1547718209,32.8
global:1> SensorReading{id='sensor_1', timestamp=1547718199, temperature=35.8}
keyBy:3> SensorReading{id='sensor_1', timestamp=1547718199, temperature=35.8}
global:1> SensorReading{id='sensor_6', timestamp=1547718201, temperature=15.4}
keyBy:3> SensorReading{id='sensor_6', timestamp=1547718201, temperature=15.4}
keyBy:3> SensorReading{id='sensor_1', timestamp=1547718207, temperature=36.3}
keyBy:3> SensorReading{id='sensor_1', timestamp=1547718209, temperature=32.8}
global:1> SensorReading{id='sensor_1', timestamp=1547718207, temperature=36.3}
shuffle:1> sensor_7,1547718202,6.7
global:1> SensorReading{id='sensor_1', timestamp=1547718209, temperature=32.8}
shuffle:2> sensor_10,1547718205,38.1
input:2> SensorReading{id='sensor_1', timestamp=1547718212, temperature=37.1}
global:1> SensorReading{id='sensor_7', timestamp=1547718202, temperature=6.7}
keyBy:4> SensorReading{id='sensor_7', timestamp=1547718202, temperature=6.7}
keyBy:2> SensorReading{id='sensor_10', timestamp=1547718205, temperature=38.1}
global:1> SensorReading{id='sensor_10', timestamp=1547718205, temperature=38.1}
shuffle:1> sensor_1,1547718212,37.1
keyBy:3> SensorReading{id='sensor_1', timestamp=1547718212, temperature=37.1}
global:1> SensorReading{id='sensor_1', timestamp=1547718212, temperature=37.1}
```

## 5.7 Sink

> [Flink之流处理API之Sink](https://blog.csdn.net/lixinkuan328/article/details/104116894)

​	Flink没有类似于spark中foreach方法，让用户进行迭代的操作。虽有对外的输出操作都要利用Sink完成。最后通过类似如下方式完成整个任务最终输出操作。

```java
stream.addSink(new MySink(xxxx)) 
```

​	官方提供了一部分的框架的sink。除此以外，需要用户自定义实现sink。

![img](https://img-blog.csdnimg.cn/20200130221249884.png)

### 5.7.1 Kafka

1. pom依赖

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
       <modelVersion>4.0.0</modelVersion>
   
       <groupId>org.example</groupId>
       <artifactId>Flink_Tutorial</artifactId>
       <version>1.0-SNAPSHOT</version>
   
       <properties>
           <maven.compiler.source>8</maven.compiler.source>
           <maven.compiler.target>8</maven.compiler.target>
           <flink.version>1.12.1</flink.version>
           <scala.binary.version>2.12</scala.binary.version>
       </properties>
   
       <dependencies>
           <dependency>
               <groupId>org.apache.flink</groupId>
               <artifactId>flink-java</artifactId>
               <version>${flink.version}</version>
           </dependency>
           <dependency>
               <groupId>org.apache.flink</groupId>
               <artifactId>flink-streaming-scala_${scala.binary.version}</artifactId>
               <version>${flink.version}</version>
           </dependency>
           <dependency>
               <groupId>org.apache.flink</groupId>
               <artifactId>flink-clients_${scala.binary.version}</artifactId>
               <version>${flink.version}</version>
           </dependency>
   
           <!-- kafka -->
           <dependency>
               <groupId>org.apache.flink</groupId>
               <artifactId>flink-connector-kafka_${scala.binary.version}</artifactId>
               <version>${flink.version}</version>
           </dependency>
       </dependencies>
   
   </project>
   ```

2. 编写java代码

   ```java
   package apitest.sink;
   
   import apitest.beans.SensorReading;
   import org.apache.flink.api.common.serialization.SimpleStringSchema;
   import org.apache.flink.streaming.api.datastream.DataStream;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer;
   import org.apache.flink.streaming.connectors.kafka.FlinkKafkaProducer;
   
   import java.util.Properties;
   
   /**
    * @author : Ashiamd email: ashiamd@foxmail.com
    * @date : 2021/2/1 1:11 AM
    */
   public class SinkTest1_Kafka {
       public static void main(String[] args) throws Exception{
           // 创建执行环境
           StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
   
           // 并行度设置为1
           env.setParallelism(1);
   
           Properties properties = new Properties();
           properties.setProperty("bootstrap.servers", "localhost:9092");
           properties.setProperty("group.id", "consumer-group");
           properties.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
           properties.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
           properties.setProperty("auto.offset.reset", "latest");
   
           // 从Kafka中读取数据
           DataStream<String> inputStream = env.addSource( new FlinkKafkaConsumer<String>("sensor", new SimpleStringSchema(), properties));
   
           // 序列化从Kafka中读取的数据
           DataStream<String> dataStream = inputStream.map(line -> {
               String[] fields = line.split(",");
               return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2])).toString();
           });
   
           // 将数据写入Kafka
           dataStream.addSink( new FlinkKafkaProducer<String>("localhost:9092", "sinktest", new SimpleStringSchema()));
           
           env.execute();
       }
   }
   ```

3. 启动zookeeper

   ```shell
   $ bin/zookeeper-server-start.sh config/zookeeper.properties
   ```

4. 启动kafka服务

   ```shell
   $ bin/kafka-server-start.sh config/server.properties
   ```

5. 新建kafka生产者console

   ```shell
   $ bin/kafka-console-producer.sh --broker-list localhost:9092  --topic sensor
   ```

6. 新建kafka消费者console

   ```shell
   $ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic sinktest
   ```

7. 运行Flink程序，在kafka生产者console输入数据，查看kafka消费者console的输出结果

   输入(kafka生产者console)

   ```shell
   >sensor_1,1547718199,35.8
   >sensor_6,1547718201,15.4
   ```

   输出(kafka消费者console)

   ```shell
   SensorReading{id='sensor_1', timestamp=1547718199, temperature=35.8}
   SensorReading{id='sensor_6', timestamp=1547718201, temperature=15.4}
   ```

这里Flink的作用相当于pipeline了。

### 5.7.2 Redis

> [flink-connector-redis](https://mvnrepository.com/search?q=flink-connector-redis)
>
> 查询Flink连接器，最简单的就是查询关键字`flink-connector-`

这里将Redis当作sink的输出对象。

1. pom依赖

   这个可谓相当老的依赖了，2017年的。

   ```xml
   <!-- https://mvnrepository.com/artifact/org.apache.bahir/flink-connector-redis -->
   <dependency>
       <groupId>org.apache.bahir</groupId>
       <artifactId>flink-connector-redis_2.11</artifactId>
       <version>1.0</version>
   </dependency>
   ```

2. 编写java代码

   ```java
   package apitest.sink;
   
   import apitest.beans.SensorReading;
   import org.apache.flink.streaming.api.datastream.DataStream;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.connectors.redis.RedisSink;
   import org.apache.flink.streaming.connectors.redis.common.config.FlinkJedisPoolConfig;
   import org.apache.flink.streaming.connectors.redis.common.mapper.RedisCommand;
   import org.apache.flink.streaming.connectors.redis.common.mapper.RedisCommandDescription;
   import org.apache.flink.streaming.connectors.redis.common.mapper.RedisMapper;
   
   /**
    * @author : Ashiamd email: ashiamd@foxmail.com
    * @date : 2021/2/1 1:47 AM
    */
   public class SinkTest2_Redis {
       public static void main(String[] args) throws Exception {
           StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
           env.setParallelism(1);
   
           // 从文件读取数据
           DataStream<String> inputStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");
   
           // 转换成SensorReading类型
           DataStream<SensorReading> dataStream = inputStream.map(line -> {
               String[] fields = line.split(",");
               return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
           });
   
           // 定义jedis连接配置(我这里连接的是docker的redis)
           FlinkJedisPoolConfig config = new FlinkJedisPoolConfig.Builder()
                   .setHost("localhost")
                   .setPort(6379)
                   .setPassword("123456")
                   .setDatabase(0)
                   .build();
   
           dataStream.addSink(new RedisSink<>(config, new MyRedisMapper()));
   
           env.execute();
       }
   
       // 自定义RedisMapper
       public static class MyRedisMapper implements RedisMapper<SensorReading> {
           // 定义保存数据到redis的命令，存成Hash表，hset sensor_temp id temperature
           @Override
           public RedisCommandDescription getCommandDescription() {
               return new RedisCommandDescription(RedisCommand.HSET, "sensor_temp");
           }
   
           @Override
           public String getKeyFromData(SensorReading data) {
               return data.getId();
           }
   
           @Override
           public String getValueFromData(SensorReading data) {
               return data.getTemperature().toString();
           }
       }
   }
   
   ```

3. 启动redis服务（我这里是docker里的）

4. 启动Flink程序

5. 查看Redis里的数据

   *因为最新数据覆盖前面的，所以最后redis里呈现的是最新的数据。*

   ```shell
   localhost:0>hgetall sensor_temp
   1) "sensor_1"
   2) "37.1"
   3) "sensor_6"
   4) "15.4"
   5) "sensor_7"
   6) "6.7"
   7) "sensor_10"
   8) "38.1"
   ```

### 5.7.3 Elasticsearch

> [Flink 1.12.1 ElasticSearch连接 Sink](https://blog.csdn.net/weixin_42066446/article/details/113243977)

1. pom依赖

   ```xml
   <!-- ElasticSearch7 -->
   <dependency>
       <groupId>org.apache.flink</groupId>
       <artifactId>flink-connector-elasticsearch7_2.12</artifactId>
       <version>1.12.1</version>
   </dependency>
   ```

2. 编写java代码

   ```java
   package apitest.sink;
   
   import apitest.beans.SensorReading;
   import org.apache.flink.api.common.functions.RuntimeContext;
   import org.apache.flink.streaming.api.datastream.DataStream;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.connectors.elasticsearch.ElasticsearchSinkFunction;
   import org.apache.flink.streaming.connectors.elasticsearch.RequestIndexer;
   import org.apache.flink.streaming.connectors.elasticsearch7.ElasticsearchSink;
   import org.apache.http.HttpHost;
   import org.elasticsearch.action.index.IndexRequest;
   import org.elasticsearch.client.Requests;
   
   import java.util.ArrayList;
   import java.util.HashMap;
   import java.util.List;
   
   /**
    * @author : Ashiamd email: ashiamd@foxmail.com
    * @date : 2021/2/1 2:13 AM
    */
   public class SinkTest3_Es {
       public static void main(String[] args) throws Exception {
           StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
           env.setParallelism(1);
   
           // 从文件读取数据
           DataStream<String> inputStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");
   
           // 转换成SensorReading类型
           DataStream<SensorReading> dataStream = inputStream.map(line -> {
               String[] fields = line.split(",");
               return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
           });
   
           // 定义es的连接配置
           List<HttpHost> httpHosts = new ArrayList<>();
           httpHosts.add(new HttpHost("localhost", 9200));
   
           dataStream.addSink( new ElasticsearchSink.Builder<SensorReading>(httpHosts, new MyEsSinkFunction()).build());
   
           env.execute();
       }
   
       // 实现自定义的ES写入操作
       public static class MyEsSinkFunction implements ElasticsearchSinkFunction<SensorReading> {
           @Override
           public void process(SensorReading element, RuntimeContext ctx, RequestIndexer indexer) {
               // 定义写入的数据source
               HashMap<String, String> dataSource = new HashMap<>();
               dataSource.put("id", element.getId());
               dataSource.put("temp", element.getTemperature().toString());
               dataSource.put("ts", element.getTimestamp().toString());
   
               // 创建请求，作为向es发起的写入命令(ES7统一type就是_doc，不再允许指定type)
               IndexRequest indexRequest = Requests.indexRequest()
                       .index("sensor")
                       .source(dataSource);
   
               // 用index发送请求
               indexer.add(indexRequest);
           }
       }
   }
   ```

3. 启动ElasticSearch（我这里是docker启动的

4. 运行Flink程序，查看ElasticSearch是否新增数据

   ```shell
   $ curl "localhost:9200/sensor/_search?pretty"
   {
     "took" : 1,
     "timed_out" : false,
     "_shards" : {
       "total" : 1,
       "successful" : 1,
       "skipped" : 0,
       "failed" : 0
     },
     "hits" : {
       "total" : {
         "value" : 7,
         "relation" : "eq"
       },
       "max_score" : 1.0,
       "hits" : [
         {
           "_index" : "sensor",
           "_type" : "_doc",
           "_id" : "jciyWXcBiXrGJa12kSQt",
           "_score" : 1.0,
           "_source" : {
             "temp" : "35.8",
             "id" : "sensor_1",
             "ts" : "1547718199"
           }
         },
         {
           "_index" : "sensor",
           "_type" : "_doc",
           "_id" : "jsiyWXcBiXrGJa12kSQu",
           "_score" : 1.0,
           "_source" : {
             "temp" : "15.4",
             "id" : "sensor_6",
             "ts" : "1547718201"
           }
         },
         {
           "_index" : "sensor",
           "_type" : "_doc",
           "_id" : "j8iyWXcBiXrGJa12kSQu",
           "_score" : 1.0,
           "_source" : {
             "temp" : "6.7",
             "id" : "sensor_7",
             "ts" : "1547718202"
           }
         },
         {
           "_index" : "sensor",
           "_type" : "_doc",
           "_id" : "kMiyWXcBiXrGJa12kSQu",
           "_score" : 1.0,
           "_source" : {
             "temp" : "38.1",
             "id" : "sensor_10",
             "ts" : "1547718205"
           }
         },
         {
           "_index" : "sensor",
           "_type" : "_doc",
           "_id" : "kciyWXcBiXrGJa12kSQu",
           "_score" : 1.0,
           "_source" : {
             "temp" : "36.3",
             "id" : "sensor_1",
             "ts" : "1547718207"
           }
         },
         {
           "_index" : "sensor",
           "_type" : "_doc",
           "_id" : "ksiyWXcBiXrGJa12kSQu",
           "_score" : 1.0,
           "_source" : {
             "temp" : "32.8",
             "id" : "sensor_1",
             "ts" : "1547718209"
           }
         },
         {
           "_index" : "sensor",
           "_type" : "_doc",
           "_id" : "k8iyWXcBiXrGJa12kSQu",
           "_score" : 1.0,
           "_source" : {
             "temp" : "37.1",
             "id" : "sensor_1",
             "ts" : "1547718212"
           }
         }
       ]
     }
   }
   ```

### 5.7.4 JDBC自定义sink

> [Flink之Mysql数据CDC](https://www.cnblogs.com/ywjfx/p/14263718.html)
>
> [JDBC Connector](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/connectors/jdbc.html)	<=	官方目前没有专门针对MySQL的，我们自己实现就好了

这里测试的是连接MySQL。

1. pom依赖（我本地docker里的mysql是8.0.19版本的）

   ```xml
   <!-- https://mvnrepository.com/artifact/mysql/mysql-connector-java -->
   <dependency>
       <groupId>mysql</groupId>
       <artifactId>mysql-connector-java</artifactId>
       <version>8.0.19</version>
   </dependency>
   ```

2. 启动mysql服务（我本地是docker启动的）

3. 新建数据库

   ```sql
   CREATE DATABASE `flink_test` DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
   ```

4. 新建schema

   ```sql
   CREATE TABLE `sensor_temp` (
     `id` varchar(32) NOT NULL,
     `temp` double NOT NULL
   ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
   ```

5. 编写java代码

   ```java
   package apitest.sink;
   
   import apitest.beans.SensorReading;
   import apitest.source.SourceTest4_UDF;
   import org.apache.flink.configuration.Configuration;
   import org.apache.flink.streaming.api.datastream.DataStream;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.api.functions.sink.RichSinkFunction;
   
   import java.sql.Connection;
   import java.sql.DriverManager;
   import java.sql.PreparedStatement;
   
   /**
    * @author : Ashiamd email: ashiamd@foxmail.com
    * @date : 2021/2/1 2:48 AM
    */
   public class SinkTest4_Jdbc {
       public static void main(String[] args) throws Exception {
   
           // 创建执行环境
           StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
   
           // 设置并行度 = 1
           env.setParallelism(1);
   
           // 从文件读取数据
   //        DataStream<String> inputStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");
   //
   //        // 转换成SensorReading类型
   //        DataStream<SensorReading> dataStream = inputStream.map(line -> {
   //            String[] fields = line.split(",");
   //            return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
   //        });
   
           // 使用之前编写的随机变动温度的SourceFunction来生成数据
           DataStream<SensorReading> dataStream = env.addSource(new SourceTest4_UDF.MySensorSource());
   
           dataStream.addSink(new MyJdbcSink());
   
           env.execute();
       }
   
       // 实现自定义的SinkFunction
       public static class MyJdbcSink extends RichSinkFunction<SensorReading> {
           // 声明连接和预编译语句
           Connection connection = null;
           PreparedStatement insertStmt = null;
           PreparedStatement updateStmt = null;
   
           @Override
           public void open(Configuration parameters) throws Exception {
               connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/flink_test?useUnicode=true&serverTimezone=Asia/Shanghai&characterEncoding=UTF-8&useSSL=false", "root", "example");
               insertStmt = connection.prepareStatement("insert into sensor_temp (id, temp) values (?, ?)");
               updateStmt = connection.prepareStatement("update sensor_temp set temp = ? where id = ?");
           }
   
           // 每来一条数据，调用连接，执行sql
           @Override
           public void invoke(SensorReading value, Context context) throws Exception {
               // 直接执行更新语句，如果没有更新那么就插入
               updateStmt.setDouble(1, value.getTemperature());
               updateStmt.setString(2, value.getId());
               updateStmt.execute();
               if (updateStmt.getUpdateCount() == 0) {
                   insertStmt.setString(1, value.getId());
                   insertStmt.setDouble(2, value.getTemperature());
                   insertStmt.execute();
               }
           }
   
           @Override
           public void close() throws Exception {
               insertStmt.close();
               updateStmt.close();
               connection.close();
           }
       }
   }
   ```

6. 运行Flink程序，查看MySQL数据（可以看到MySQL里的数据一直在变动）

   ```shell
   mysql> SELECT * FROM sensor_temp;
   +-----------+--------------------+
   | id        | temp               |
   +-----------+--------------------+
   | sensor_3  | 20.489172407885917 |
   | sensor_10 |  73.01289164711463 |
   | sensor_4  | 43.402500895809744 |
   | sensor_1  |  6.894772325662007 |
   | sensor_2  | 101.79309911751122 |
   | sensor_7  | 63.070612021580324 |
   | sensor_8  |  63.82606628090501 |
   | sensor_5  |  57.67115738487047 |
   | sensor_6  |  50.84442627975055 |
   | sensor_9  |  52.58400793021675 |
   +-----------+--------------------+
   10 rows in set (0.00 sec)
   
   mysql> SELECT * FROM sensor_temp;
   +-----------+--------------------+
   | id        | temp               |
   +-----------+--------------------+
   | sensor_3  | 19.498209543035923 |
   | sensor_10 |  71.92981963197121 |
   | sensor_4  | 43.566017489470426 |
   | sensor_1  |  6.378208186786803 |
   | sensor_2  | 101.71010087830145 |
   | sensor_7  |  62.11402602179431 |
   | sensor_8  |  64.33196455020062 |
   | sensor_5  |  56.39071692662006 |
   | sensor_6  | 48.952784757264894 |
   | sensor_9  | 52.078086096436685 |
   +-----------+--------------------+
   10 rows in set (0.00 sec)
   ```

## 5.8 Joining

> [Joining](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/stream/operators/joining.html)

### 5.8.1 Window Join

​	A window join joins the elements of two streams that share a common key and lie in the same window. These windows can be defined by using a [window assigner](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/stream/operators/windows.html#window-assigners) and are evaluated on elements from both of the streams.

​	The elements from both sides are then passed to a user-defined `JoinFunction` or `FlatJoinFunction` 	where the user can emit results that meet the join criteria.

​	The general usage can be summarized as follows:

```java
stream.join(otherStream)
    .where(<KeySelector>)
    .equalTo(<KeySelector>)
    .window(<WindowAssigner>)
    .apply(<JoinFunction>)
```

#### Tumbling Window Join

​	When performing a tumbling window join, all elements with a common key and a common tumbling window are joined as pairwise combinations and passed on to a `JoinFunction` or `FlatJoinFunction`. Because this behaves like an inner join, elements of one stream that do not have elements from another stream in their tumbling window are not emitted!

![img](https://ci.apache.org/projects/flink/flink-docs-release-1.12/fig/tumbling-window-join.svg)

​	As illustrated in the figure, we define a tumbling window with the size of 2 milliseconds, which results in windows of the form `[0,1], [2,3], ...`. The image shows the pairwise combinations of all elements in each window which will be passed on to the `JoinFunction`. Note that in the tumbling window `[6,7]` nothing is emitted because no elements exist in the green stream to be joined with the orange elements ⑥ and ⑦.

```java
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows;
import org.apache.flink.streaming.api.windowing.time.Time;
 
...

DataStream<Integer> orangeStream = ...
DataStream<Integer> greenStream = ...

orangeStream.join(greenStream)
    .where(<KeySelector>)
    .equalTo(<KeySelector>)
    .window(TumblingEventTimeWindows.of(Time.milliseconds(2)))
    .apply (new JoinFunction<Integer, Integer, String> (){
        @Override
        public String join(Integer first, Integer second) {
            return first + "," + second;
        }
    });
```

####  Sliding Window Join

​	When performing a sliding window join, all elements with a common key and common sliding window are joined as pairwise combinations and passed on to the `JoinFunction` or `FlatJoinFunction`. Elements of one stream that do not have elements from the other stream in the current sliding window are not emitted! Note that some elements might be joined in one sliding window but not in another!

![img](https://ci.apache.org/projects/flink/flink-docs-release-1.12/fig/sliding-window-join.svg)

​	In this example we are using sliding windows with a size of two milliseconds and slide them by one millisecond, resulting in the sliding windows `[-1, 0],[0,1],[1,2],[2,3], …`. The joined elements below the x-axis are the ones that are passed to the `JoinFunction` for each sliding window. Here you can also see how for example the orange ② is joined with the green ③ in the window `[2,3]`, but is not joined with anything in the window `[1,2]`.

```java
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.streaming.api.windowing.assigners.SlidingEventTimeWindows;
import org.apache.flink.streaming.api.windowing.time.Time;

...

DataStream<Integer> orangeStream = ...
DataStream<Integer> greenStream = ...

orangeStream.join(greenStream)
    .where(<KeySelector>)
    .equalTo(<KeySelector>)
    .window(SlidingEventTimeWindows.of(Time.milliseconds(2) /* size */, Time.milliseconds(1) /* slide */))
    .apply (new JoinFunction<Integer, Integer, String> (){
        @Override
        public String join(Integer first, Integer second) {
            return first + "," + second;
        }
    });
```

#### Session Window Join

​	When performing a session window join, all elements with the same key that when *“combined”* fulfill the session criteria are joined in pairwise combinations and passed on to the `JoinFunction` or `FlatJoinFunction`. Again this performs an inner join, so if there is a session window that only contains elements from one stream, no output will be emitted!

![img](https://ci.apache.org/projects/flink/flink-docs-release-1.12/fig/session-window-join.svg)

​	Here we define a session window join where each session is divided by a gap of at least 1ms. There are three sessions, and in the first two sessions the joined elements from both streams are passed to the `JoinFunction`. In the third session there are no elements in the green stream, so ⑧ and ⑨ are not joined!

```java
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.streaming.api.windowing.assigners.EventTimeSessionWindows;
import org.apache.flink.streaming.api.windowing.time.Time;
 
...

DataStream<Integer> orangeStream = ...
DataStream<Integer> greenStream = ...

orangeStream.join(greenStream)
    .where(<KeySelector>)
    .equalTo(<KeySelector>)
    .window(EventTimeSessionWindows.withGap(Time.milliseconds(1)))
    .apply (new JoinFunction<Integer, Integer, String> (){
        @Override
        public String join(Integer first, Integer second) {
            return first + "," + second;
        }
    });
```

### 5.8.2 Interval Join

The interval join joins elements of two streams (we’ll call them A & B for now) with a common key and where elements of stream B have timestamps that lie in a relative time interval to timestamps of elements in stream A.

This can also be expressed more formally as `b.timestamp ∈ [a.timestamp + lowerBound; a.timestamp + upperBound]` or `a.timestamp + lowerBound <= b.timestamp <= a.timestamp + upperBound`

where a and b are elements of A and B that share a common key. Both the lower and upper bound can be either negative or positive as long as as the lower bound is always smaller or equal to the upper bound. The interval join currently only performs inner joins.

When a pair of elements are passed to the `ProcessJoinFunction`, they will be assigned with the larger timestamp (which can be accessed via the `ProcessJoinFunction.Context`) of the two elements.

<u>**Note** The interval join currently only supports event time.</u>

![img](https://ci.apache.org/projects/flink/flink-docs-release-1.12/fig/interval-join.svg)

In the example above, we join two streams ‘orange’ and ‘green’ with a lower bound of -2 milliseconds and an upper bound of +1 millisecond. Be default, these boundaries are inclusive, but `.lowerBoundExclusive()` and `.upperBoundExclusive` can be applied to change the behaviour.

Using the more formal notation again this will translate to

```
orangeElem.ts + lowerBound <= greenElem.ts <= orangeElem.ts + upperBound
```

as indicated by the triangles.

```java
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.streaming.api.functions.co.ProcessJoinFunction;
import org.apache.flink.streaming.api.windowing.time.Time;

...

DataStream<Integer> orangeStream = ...
DataStream<Integer> greenStream = ...

orangeStream
    .keyBy(<KeySelector>)
    .intervalJoin(greenStream.keyBy(<KeySelector>))
    .between(Time.milliseconds(-2), Time.milliseconds(1))
    .process (new ProcessJoinFunction<Integer, Integer, String(){

        @Override
        public void processElement(Integer left, Integer right, Context ctx, Collector<String> out) {
            out.collect(first + "," + second);
        }
    });
```

# 6. Flink的Window

## 6.1 Window

> [Flink_Window](https://blog.csdn.net/dongkang123456/article/details/108374799)

### 6.1.1 概述

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200903082944202.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvbmdrYW5nMTIzNDU2,size_16,color_FFFFFF,t_70#pic_center)

​	streaming流式计算是一种被设计用于处理无限数据集的数据处理引擎，而无限数据集是指一种不断增长的本质上无限的数据集，而**window是一种切割无限数据为有限块进行处理的手段**。

​	**Window是无限数据流处理的核心，Window将一个无限的stream拆分成有限大小的”buckets”桶，我们可以在这些桶上做计算操作**。

*举例子：假设按照时间段划分桶，接收到的数据马上能判断放到哪个桶，且多个桶的数据能并行被处理。（迟到的数据也可判断是原本属于哪个桶的）*

### 6.1.2 Window类型

+ 时间窗口（Time Window）
  + 滚动时间窗口
  + 滑动时间窗口
  + 会话窗口
+ 计数窗口（Count Window）
  + 滚动计数窗口
  + 滑动计数窗口

**TimeWindow：按照时间生成Window**

**CountWindow：按照指定的数据条数生成一个Window，与时间无关**

----

#### 滚动窗口(Tumbling Windows)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200903083725483.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvbmdrYW5nMTIzNDU2,size_16,color_FFFFFF,t_70#pic_center)

+ 依据**固定的窗口长度**对数据进行切分

+ 时间对齐，窗口长度固定，没有重叠

#### 滑动窗口(Sliding Windows)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200903084127244.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvbmdrYW5nMTIzNDU2,size_16,color_FFFFFF,t_70#pic_center)

+ 可以按照固定的长度向后滑动固定的距离

+ 滑动窗口由**固定的窗口长度**和**滑动间隔**组成

+ 可以有重叠(是否重叠和滑动距离有关系)

+ 滑动窗口是固定窗口的更广义的一种形式，滚动窗口可以看做是滑动窗口的一种特殊情况（即窗口大小和滑动间隔相等）

#### 会话窗口(Session Windows)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200903085034747.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvbmdrYW5nMTIzNDU2,size_16,color_FFFFFF,t_70#pic_center)

+ 由一系列事件组合一个指定时间长度的timeout间隙组成，也就是一段时间没有接收到新数据就会生成新的窗口
+ 特点：时间无对齐

## 6.2 Window API

### 6.2.1 概述

+ 窗口分配器——`window()`方法

+ 我们可以用`.window()`来定义一个窗口，然后基于这个window去做一些聚合或者其他处理操作。

  **注意`window()`方法必须在keyBy之后才能使用**。

+ Flink提供了更加简单的`.timeWindow()`和`.countWindow()`方法，用于定义时间窗口和计数窗口。

```java
DataStream<Tuple2<String,Double>> minTempPerWindowStream = 
  datastream
  .map(new MyMapper())
  .keyBy(data -> data.f0)
  .timeWindow(Time.seconds(15))
  .minBy(1);
```

#### 窗口分配器(window assigner)

+ `window()`方法接收的输入参数是一个WindowAssigner
+ WindowAssigner负责将每条输入的数据分发到正确的window中
+ Flink提供了通用的WindowAssigner
  + 滚动窗口（tumbling window）
  + 滑动窗口（sliding window）
  + 会话窗口（session window）
  + **全局窗口（global window）**

#### 创建不同类型的窗口

+ 滚动时间窗口（tumbling time window）

  `.timeWindow(Time.seconds(15))`

+ 滑动时间窗口（sliding time window）

  `.timeWindow(Time.seconds(15),Time.seconds(5))`

+ 会话窗口（session window）

  `.window(EventTimeSessionWindows.withGap(Time.minutes(10)))`

+ 滚动计数窗口（tumbling count window）

  `.countWindow(5)`

+ 滑动计数窗口（sliding count window）

  `.countWindow(10,2)`

*DataStream的`windowAll()`类似分区的global操作，这个操作是non-parallel的(并行度强行为1)，所有的数据都会被传递到同一个算子operator上，官方建议如果非必要就不要用这个API*

### 6.2.2 TimeWindow

​	TimeWindow将指定时间范围内的所有数据组成一个window，一次对一个window里面的所有数据进行计算。

#### 滚动窗口

​	Flink默认的时间窗口根据ProcessingTime进行窗口的划分，将Flink获取到的数据根据进入Flink的时间划分到不同的窗口中。

```java
DataStream<Tuple2<String, Double>> minTempPerWindowStream = dataStream 
  .map(new MapFunction<SensorReading, Tuple2<String, Double>>() { 
    @Override 
    public Tuple2<String, Double> map(SensorReading value) throws Exception {
      return new Tuple2<>(value.getId(), value.getTemperature()); 
    } 
  }) 
  .keyBy(data -> data.f0) 
  .timeWindow( Time.seconds(15) ) 
  .minBy(1);
```

​	时间间隔可以通过`Time.milliseconds(x)`，`Time.seconds(x)`，`Time.minutes(x)`等其中的一个来指定。

#### 滑动窗口

​	滑动窗口和滚动窗口的函数名是完全一致的，只是在传参数时需要传入两个参数，一个是window_size，一个是sliding_size。

​	下面代码中的sliding_size设置为了5s，也就是说，每5s就计算输出结果一次，每一次计算的window范围是15s内的所有元素。

```java
DataStream<SensorReading> minTempPerWindowStream = dataStream 
  .keyBy(SensorReading::getId) 
  .timeWindow( Time.seconds(15), Time.seconds(5) ) 
  .minBy("temperature");
```

​	时间间隔可以通过`Time.milliseconds(x)`，`Time.seconds(x)`，`Time.minutes(x)`等其中的一个来指定。

### 6.2.3 CountWindow

​	CountWindow根据窗口中相同key元素的数量来触发执行，执行时只计算元素数量达到窗口大小的key对应的结果。

​	**注意：CountWindow的window_size指的是相同Key的元素的个数，不是输入的所有元素的总数。**

#### 滚动窗口

​	默认的CountWindow是一个滚动窗口，只需要指定窗口大小即可，**当元素数量达到窗口大小时，就会触发窗口的执行**。

```java
DataStream<SensorReading> minTempPerWindowStream = dataStream 
  .keyBy(SensorReading::getId) 
  .countWindow( 5 ) 
  .minBy("temperature");
```

#### 滑动窗口

​	滑动窗口和滚动窗口的函数名是完全一致的，只是在传参数时需要传入两个参数，一个是window_size，一个是sliding_size。

​	下面代码中的sliding_size设置为了2，也就是说，每收到两个相同key的数据就计算一次，每一次计算的window范围是10个元素。

```java
DataStream<SensorReading> minTempPerWindowStream = dataStream 
  .keyBy(SensorReading::getId) 
  .countWindow( 10, 2 ) 
  .minBy("temperature");
```

### 6.2.4 window function

window function 定义了要对窗口中收集的数据做的计算操作，主要可以分为两类：

+ 增量聚合函数（incremental aggregation functions）
+ 全窗口函数（full window functions）

#### 增量聚合函数

+ **每条数据到来就进行计算**，保持一个简单的状态。（来一条处理一条，但是不输出，到窗口临界位置才输出）
+ 典型的增量聚合函数有ReduceFunction, AggregateFunction。

#### 全窗口函数

+ **先把窗口所有数据收集起来，等到计算的时候会遍历所有数据**。（来一个放一个，窗口临界位置才遍历且计算、输出）
+ ProcessWindowFunction，WindowFunction。

### 6.2.5 其它可选API

> [Flink-Window概述 | Window类型 | TimeWindow、CountWindow、SessionWindow、WindowFunction](https://blog.csdn.net/qq_40180229/article/details/106359443)

+ `.trigger()` ——触发器

  定义window 什么时候关闭，触发计算并输出结果

+ `.evitor()` ——移除器

  定义移除某些数据的逻辑

+ `.allowedLateness()` ——允许处理迟到的数据

+ `.sideOutputLateData()` ——将迟到的数据放入侧输出流

+ `.getSideOutput()` ——获取侧输出流

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200526181340668.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

### 6.2.6 代码测试

> [Flink之Window的使用（2）：时间窗口](https://www.cnblogs.com/yangshibiao/p/14133628.html)

1. 测试滚动时间窗口的**增量聚合函数**

   增量聚合函数，特点即每次数据过来都处理，但是**到了窗口临界才输出结果**。

   + 编写java代码

     ```java
     package apitest.window;
     
     import apitest.beans.SensorReading;
     import org.apache.flink.api.common.functions.AggregateFunction;
     import org.apache.flink.streaming.api.datastream.DataStream;
     import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
     import org.apache.flink.streaming.api.windowing.assigners.TumblingProcessingTimeWindows;
     import org.apache.flink.streaming.api.windowing.time.Time;
     
     /**
      * @author : Ashiamd email: ashiamd@foxmail.com
      * @date : 2021/2/1 7:14 PM
      */
     public class WindowTest1_TimeWindow {
       public static void main(String[] args) throws Exception {
     
         // 创建执行环境
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
     
         // 并行度设置1，方便看结果
         env.setParallelism(1);
     
         //        // 从文件读取数据
         //        DataStream<String> dataStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");
     
         // 从socket文本流获取数据
         DataStream<String> inputStream = env.socketTextStream("localhost", 7777);
     
         // 转换成SensorReading类型
         DataStream<SensorReading> dataStream = inputStream.map(line -> {
           String[] fields = line.split(",");
           return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
         });
     
         // 开窗测试
     
         // 1. 增量聚合函数 (这里简单统计每个key组里传感器信息的总数)
         DataStream<Integer> resultStream = dataStream.keyBy("id")
           //                .countWindow(10, 2);
           //                .window(EventTimeSessionWindows.withGap(Time.minutes(1)));
           //                .window(TumblingProcessingTimeWindows.of(Time.seconds(15)))
           //                .timeWindow(Time.seconds(15)) // 已经不建议使用@Deprecated
           .window(TumblingProcessingTimeWindows.of(Time.seconds(15)))
           .aggregate(new AggregateFunction<SensorReading, Integer, Integer>() {
     
             // 新建的累加器
             @Override
             public Integer createAccumulator() {
               return 0;
             }
     
             // 每个数据在上次的基础上累加
             @Override
             public Integer add(SensorReading value, Integer accumulator) {
               return accumulator + 1;
             }
     
             // 返回结果值
             @Override
             public Integer getResult(Integer accumulator) {
               return accumulator;
             }
     
             // 分区合并结果(TimeWindow一般用不到，SessionWindow可能需要考虑合并)
             @Override
             public Integer merge(Integer a, Integer b) {
               return a + b;
             }
           });
     
         resultStream.print("result");
     
         env.execute();
       }
     }
     ```

   + 本地开启socket服务

     ```shell
     nc -lk 7777
     ```

   + 启动Flink程序，在socket窗口输入数据

     + 输入(下面用“换行”区分每个15s内的输入，实际输入时无换行)

       ```none
       sensor_1,1547718199,35.8
       sensor_6,1547718201,15.4
       
       sensor_7,1547718202,6.7
       sensor_10,1547718205,38.1
       sensor_1,1547718207,36.3
       sensor_1,1547718209,32.8
       
       sensor_1,1547718212,37.1
       ```

     + 输出（下面用“换行”区分每个15s内的输出，实际输出无换行）

       *因为代码实现每15s一个window，所以"sensor_1"中间一组才累计2，最初一次不累计，最后一次也是另外的window，重新从1计数。*

       ```none
       result> 1
       result> 1
       
       result> 1
       result> 1
       result> 2
       
       result> 1
       ```

2. 测试滚动时间窗口的**全窗口函数**

   全窗口函数，特点即数据过来先不处理，等到窗口临界再遍历、计算、输出结果。

   + 编写java测试代码

     ```java
     package apitest.window;
     
     import apitest.beans.SensorReading;
     import org.apache.commons.collections.IteratorUtils;
     import org.apache.flink.api.common.functions.AggregateFunction;
     import org.apache.flink.api.java.tuple.Tuple;
     import org.apache.flink.api.java.tuple.Tuple3;
     import org.apache.flink.streaming.api.datastream.DataStream;
     import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
     import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
     import org.apache.flink.streaming.api.functions.windowing.WindowFunction;
     import org.apache.flink.streaming.api.windowing.assigners.TumblingProcessingTimeWindows;
     import org.apache.flink.streaming.api.windowing.time.Time;
     import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
     import org.apache.flink.util.Collector;
     
     /**
      * @author : Ashiamd email: ashiamd@foxmail.com
      * @date : 2021/2/1 7:14 PM
      */
     public class WindowTest1_TimeWindow {
         public static void main(String[] args) throws Exception {
     
             // 创建执行环境
             StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
     
             // 并行度设置1，方便看结果
             env.setParallelism(1);
     
     //        // 从文件读取数据
     //        DataStream<String> dataStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");
     
             // 从socket文本流获取数据
             DataStream<String> inputStream = env.socketTextStream("localhost", 7777);
     
             // 转换成SensorReading类型
             DataStream<SensorReading> dataStream = inputStream.map(line -> {
                 String[] fields = line.split(",");
                 return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
             });
     
             // 2. 全窗口函数 （WindowFunction和ProcessWindowFunction，后者更全面）
             SingleOutputStreamOperator<Tuple3<String, Long, Integer>> resultStream2 = dataStream.keyBy(SensorReading::getId)
                     .window(TumblingProcessingTimeWindows.of(Time.seconds(15)))
     //                .process(new ProcessWindowFunction<SensorReading, Object, Tuple, TimeWindow>() {
     //                })
                     .apply(new WindowFunction<SensorReading, Tuple3<String, Long, Integer>, String, TimeWindow>() {
                         @Override
                         public void apply(String s, TimeWindow window, Iterable<SensorReading> input, Collector<Tuple3<String, Long, Integer>> out) throws Exception {
                             String id = s;
                             long windowEnd = window.getEnd();
                             int count = IteratorUtils.toList(input.iterator()).size();
                             out.collect(new Tuple3<>(id, windowEnd, count));
                         }
                     });
     
             resultStream2.print("result2");
     
             env.execute();
         }
     }
     ```

   + 启动本地socket

     ```shell
     nc -lk 7777
     ```

   + 在本地socket输入，查看Flink输出结果

     + 输入（以“空行”表示每个15s时间窗口内的输入，实际没有“空行”）

       ```none
       sensor_1,1547718199,35.8
       sensor_6,1547718201,15.4
       
       sensor_7,1547718202,6.7
       sensor_10,1547718205,38.1
       sensor_1,1547718207,36.3
       sensor_1,1547718209,32.8
       ```

     + 输出（以“空行”表示每个15s时间窗口内的输入，实际没有“空行”）

       *这里每个window都是分开计算的，所以第一个window里的sensor_1和第二个window里的sensor_1并没有累计。*

       ```none
       result2> (sensor_1,1612190820000,1)
       result2> (sensor_6,1612190820000,1)
       
       result2> (sensor_7,1612190835000,1)
       result2> (sensor_1,1612190835000,2)
       result2> (sensor_10,1612190835000,1)
       ```

3. 测试滑动计数窗口的**增量聚合函数**

   滑动窗口，当窗口不足设置的大小时，会先按照步长输出。

   eg：窗口大小10，步长2，那么前5次输出时，窗口内的元素个数分别是（2，4，6，8，10），再往后就是10个为一个窗口了。

   + 编写java代码：

     这里获取每个窗口里的温度平均值

     ```java
     package apitest.window;
     
     import apitest.beans.SensorReading;
     import org.apache.flink.api.common.functions.AggregateFunction;
     import org.apache.flink.api.java.tuple.Tuple2;
     import org.apache.flink.streaming.api.datastream.DataStream;
     import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
     
     /**
      * @author : Ashiamd email: ashiamd@foxmail.com
      * @date : 2021/2/1 11:03 PM
      */
     public class WindowTest2_CountWindow {
       public static void main(String[] args) throws Exception {
     
         // 创建执行环境
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
     
         // 并行度设置1，方便看结果
         env.setParallelism(1);
     
         // 从socket文本流获取数据
         DataStream<String> inputStream = env.socketTextStream("localhost", 7777);
     
         // 转换成SensorReading类型
         DataStream<SensorReading> dataStream = inputStream.map(line -> {
           String[] fields = line.split(",");
           return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
         });
     
         DataStream<Double> resultStream = dataStream.keyBy(SensorReading::getId)
           .countWindow(10, 2)
           .aggregate(new MyAvgFunc());
     
         resultStream.print("result");
     
         env.execute();
       }
     
       public static class MyAvgFunc implements AggregateFunction<SensorReading, Tuple2<Double, Integer>, Double> {
     
         @Override
         public Tuple2<Double, Integer> createAccumulator() {
           return new Tuple2<>(0.0, 0);
         }
     
         @Override
         public Tuple2<Double, Integer> add(SensorReading value, Tuple2<Double, Integer> accumulator) {
           // 温度累加求和，当前统计的温度个数+1
           return new Tuple2<>(accumulator.f0 + value.getTemperature(), accumulator.f1 + 1);
         }
     
         @Override
         public Double getResult(Tuple2<Double, Integer> accumulator) {
           return accumulator.f0 / accumulator.f1;
         }
     
         @Override
         public Tuple2<Double, Integer> merge(Tuple2<Double, Integer> a, Tuple2<Double, Integer> b) {
           return new Tuple2<>(a.f0 + b.f0, a.f1 + b.f1);
         }
       }
     }
     ```

   + 启动socket服务

     ```shell
     nc -lk 7777
     ```

   + 本地socket输入，Flink控制台查看输出结果

     + 输入

       这里为了方便，就只输入同一个keyBy组的数据`sensor_1`

       ```none
       sensor_1,1547718199,1
       sensor_1,1547718199,2
       sensor_1,1547718199,3
       sensor_1,1547718199,4
       sensor_1,1547718199,5
       sensor_1,1547718199,6
       sensor_1,1547718199,7
       sensor_1,1547718199,8
       sensor_1,1547718199,9
       sensor_1,1547718199,10
       sensor_1,1547718199,11
       sensor_1,1547718199,12
       sensor_1,1547718199,13
       sensor_1,1547718199,14
       ```

     + 输出

       输入时，会发现，每次到达一个窗口步长（这里为2），就会计算得出一次结果。

       第一次计算前2个数的平均值

       第二次计算前4个数的平均值

       第三次计算前6个数的平均值

       第四次计算前8个数的平均值

       第五次计算前10个数的平均值

       **第六次计算前最近10个数的平均值**

       **第七次计算前最近10个数的平均值**

       ```none
       result> 1.5
       result> 2.5
       result> 3.5
       result> 4.5
       result> 5.5
       result> 7.5
       result> 9.5
       ```

4. 其他可选API代码片段

   ```java
   // 3. 其他可选API
   OutputTag<SensorReading> outputTag = new OutputTag<SensorReading>("late") {
   };
   
   SingleOutputStreamOperator<SensorReading> sumStream = dataStream.keyBy("id")
     .timeWindow(Time.seconds(15))
     //                .trigger() // 触发器，一般不使用 
     //                .evictor() // 移除器，一般不使用
     .allowedLateness(Time.minutes(1)) // 允许1分钟内的迟到数据<=比如数据产生时间在窗口范围内，但是要处理的时候已经超过窗口时间了
     .sideOutputLateData(outputTag) // 侧输出流，迟到超过1分钟的数据，收集于此
     .sum("temperature"); // 侧输出流 对 温度信息 求和。
   
   // 之后可以再用别的程序，把侧输出流的信息和前面窗口的信息聚合。（可以把侧输出流理解为用来批处理来补救处理超时数据）
   ```

# 7. 时间语义和Watermark

> [Flink_Window](https://blog.csdn.net/dongkang123456/article/details/108374799)

## 7.1 Flink中的时间语义

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200903145920356.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvbmdrYW5nMTIzNDU2,size_16,color_FFFFFF,t_70#pic_center)

+ **Event Time：事件创建时间；**

+ Ingestion Time：数据进入Flink的时间；

+ Processing Time：执行操作算子的本地系统时间，与机器相关；

​	*Event Time是事件创建的时间。它通常由事件中的时间戳描述，例如采集的日志数据中，每一条日志都会记录自己的生成时间，Flink通过时间戳分配器访问事件时间戳。*	

---

> [Flink-时间语义与Wartmark及EventTime在Window中的使用](https://blog.csdn.net/qq_40180229/article/details/106363815)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200526200231905.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ 不同的时间语义有不同的应用场合
+ **我们往往更关心事件事件（Event Time）**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200526200432798.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

​	这里假设玩游戏，两分钟内如果过5关就有奖励。用户坐地铁玩游戏，进入隧道前已经过3关，在隧道中又过了2关。但是信号不好，后两关通关的信息，等到出隧道的时候（8:23:20）才正式到达服务器。

​	如果为了用户体验，那么应该按照Event Time处理信息，保证用户获得游戏奖励。

+ Event Time可以从日志数据的时间戳（timestamp）中提取

  ```shell
  2017-11-02 18:27:15.624 INFO Fail over to rm
  ```

## 7.2 EventTime的引入

​	**在Flink的流式处理中，绝大部分的业务都会使用eventTime**，一般只在eventTime无法使用时，才会被迫使用ProcessingTime或者IngestionTime。

​	*（虽然默认环境里使用的就是ProcessingTime，使用EventTime需要另外设置）*

​	如果要使用EventTime，那么需要引入EventTime的时间属性，引入方式如下所示：

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
// 从调用时刻开始给env创建的每一个stream追加时间特征
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
```

**注：具体的时间，还需要从数据中提取时间戳。**

## 7.3 Watermark

> [Flink流计算编程--watermark（水位线）简介](https://blog.csdn.net/lmalds/article/details/52704170)	<=	不错的文章，建议阅读

### 7.3.1 概念

* **Flink对于迟到数据有三层保障**，先来后到的保障顺序是：
  * WaterMark => 约等于放宽窗口标准
  * allowedLateness => 允许迟到（ProcessingTime超时，但是EventTime没超时）
  * sideOutputLateData => 超过迟到时间，另外捕获，之后可以自己批处理合并先前的数据

---

> [Flink-时间语义与Wartmark及EventTime在Window中的使用](https://blog.csdn.net/qq_40180229/article/details/106363815)

​	我们知道，流处理从事件产生，到流经source，再到operator，中间是有一个过程和时间的，虽然大部分情况下，流到operator的数据都是按照事件产生的时间顺序来的，但是也不排除由于网络、分布式等原因，导致乱序的产生，所谓乱序，就是指Flink接收到的事件的先后顺序不是严格按照事件的Event Time顺序排列的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200526201305372.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

​	那么此时出现一个问题，一旦出现乱序，如果只根据eventTime决定window的运行，我们不能明确数据是否全部到位，但又不能无限期的等下去，此时必须要有个机制来保证一个特定的时间后，必须触发window去进行计算了，这个特别的机制，就是Watermark。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200526201418333.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ 当Flink以**Event Time模式**处理数据流时，它会根据**数据里的时间戳**来处理基于时间的算子。

  （比如5s一个窗口，那么理想情况下，遇到时间戳是5s的数据时，就认为[0,5s)时间段的桶bucket就可以关闭了。）

+ 实际由于网络、分布式传输处理等原因，会导致乱序数据的产生

+ 乱序数据会导致窗口计算不准确

  （如果按照前面说法，获取到5s时间戳的数据，但是2s，3s乱序数据还没到，理论上不应该关闭桶）

---

+ 怎样避免乱序数据带来的计算不正确？
+ 遇到一个时间戳达到了窗口关闭时间，不应该立即触发窗口计算，而是等待一段时间，等迟到的数据来了再关闭窗口

1. Watermark是一种衡量Event Time进展的机制，可以设定延迟触发

2. Watermark是用于处理乱序事件的，而正确的处理乱序事件，通常用Watermark机制结合window来实现

3. 数据流中的Watermark用于表示”timestamp小于Watermark的数据，都已经到达了“，因此，window的执行也是由Watermark触发的。

4. Watermark可以理解成一个延迟触发机制，我们可以设置Watermark的延时时长t，<u>每次系统会校验已经到达的数据中最大的maxEventTime，然后认定eventTime小于maxEventTime - t的所有数据都已经到达</u>，**如果有窗口的停止时间等于maxEventTime – t，那么这个窗口被触发执行。**

   `Watermark = maxEventTime-延迟时间t`

5. watermark 用来让程序自己平衡延迟和结果正确性

*watermark可以理解为把原本的窗口标准稍微放宽了一点。（比如原本5s，设置延迟时间=2s，那么实际等到7s的数据到达时，才认为是[0,5）的桶需要关闭了）*

有序流的Watermarker如下图所示：（延迟时间设置为0s）

<small>*此时以5s一个窗口，那么EventTime=5s的元素到达时，关闭第一个窗口，下图即W(5)，W(10)同理。*</small>

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200526201731274.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

乱序流的Watermarker如下图所示：（延迟时间设置为2s）

<small>*乱序流，所以可能出现EventTime前后顺序不一致的情况，这里延迟时间设置2s，第一个窗口则为`5s+2s`，当EventTime=7s的数据到达时，关闭第一个窗口。第二个窗口则是`5*2+2=12s`，当12s这个EventTime的数据到达时，关闭第二个窗口。*</small>

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020052620175060.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

​	当Flink接收到数据时，会按照一定的规则去生成Watermark，这条<u>Watermark就等于当前所有到达数据中的maxEventTime-延迟时长</u>，也就是说，**Watermark是基于数据携带的时间戳生成的**，一旦Watermark比当前未触发的窗口的停止时间要晚，那么就会触发相应窗口的执行。

​	**由于event time是由数据携带的，因此，如果运行过程中无法获取新的数据，那么没有被触发的窗口将永远都不被触发**。

​	上图中，我们设置的允许最大延迟到达时间为2s，所以时间戳为7s的事件对应的Watermark是5s，时间戳为12s的事件的Watermark是10s，如果我们的窗口1是`1s~5s`，窗口2是`6s~10s`，那么时间戳为7s的事件到达时的Watermarker恰好触发窗口1，时间戳为12s的事件到达时的Watermark恰好触发窗口2。

​	**Watermark 就是触发前一窗口的“关窗时间”，一旦触发关门那么以当前时刻为准在窗口范围内的所有所有数据都会收入窗中。**

​	**只要没有达到水位那么不管现实中的时间推进了多久都不会触发关窗。**

### 7.3.2 Watermark的特点

> [Flink-时间语义与Wartmark及EventTime在Window中的使用](https://blog.csdn.net/qq_40180229/article/details/106363815)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200526204111817.png)

+ watermark 是一条特殊的数据记录

+ **watermark 必须单调递增**，以确保任务的事件时间时钟在向前推进，而不是在后退

+ watermark 与数据的时间戳相关

### 7.3.3 Watermark的传递

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200526204125805.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

1. 图一，当前Task有四个上游Task给自己传输WaterMark信息，通过比较，只取当前最小值作为自己的本地Event-time clock，上图中，当前Task[0,2)的桶就可关闭了，因为所有上游中2s最小，能保证2s的WaterMark是准确的（所有上游Watermark都已经>=2s)。这时候将Watermark=2广播到当前Task的下游。
2. 图二，上游的Watermark持续变动，此时Watermark=3成为新的最小值，更新本地Task的event-time clock，同时将最新的Watermark=3广播到下游
3. 图三，上游的Watermark虽然更新了，但是当前最小值还是3，所以不更新event-time clock，也不需要广播到下游
4. 图四，和图二同理，更新本地event-time clock，同时向下游广播最新的Watermark=4

### 7.3.4 Watermark的引入

​	watermark的引入很简单，对于乱序数据，最常见的引用方式如下：

```scala
dataStream.assignTimestampsAndWatermarks( new BoundedOutOfOrdernessTimestampExtractor<SensorReading>(Time.milliseconds(1000)) {
  @Override
  public long extractTimestamp(element: SensorReading): Long = { 
    return element.getTimestamp() * 1000L;
  } 
});
```

​	**Event Time的使用一定要指定数据源中的时间戳。否则程序无法知道事件的事件时间是什么(数据源里的数据没有时间戳的话，就只能使用Processing Time了)**。

​	我们看到上面的例子中创建了一个看起来有点复杂的类，这个类实现的其实就是分配时间戳的接口。Flink暴露了TimestampAssigner接口供我们实现，使我们可以自定义如何从事件数据中抽取时间戳。

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
// 设置事件时间语义 env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
DataStream<SensorReading> dataStream = env.addSource(new SensorSource()) .assignTimestampsAndWatermarks(new MyAssigner());
```

MyAssigner有两种类型

+ AssignerWithPeriodicWatermarks

+ AssignerWithPunctuatedWatermarks

以上两个接口都继承自TimestampAssigner。

#### TimestampAssigner

##### AssignerWithPeriodicWatermarks

+ 周期性的生成 watermark：系统会周期性的将 watermark 插入到流中

+ 默认周期是200毫秒，可以使用 `ExecutionConfig.setAutoWatermarkInterval()` 方法进行设置

+ **升序和前面乱序的处理 BoundedOutOfOrderness ，都是基于周期性 watermark 的**。

##### AssignerWithPunctuatedWatermarks

+ 没有时间周期规律，可打断的生成 watermark（即可实现每次获取数据都更新watermark）

### 7.3.5 Watermark的设定

+ 在Flink中，Watermark由应用程序开发人员生成，这通常需要对相应的领域有一定的了解
+ 如果Watermark设置的延迟太久，收到结果的速度可能就会很慢，解决办法是在水位线到达之前输出一个近似结果
+ 如果Watermark到达得太早，则可能收到错误结果，不过Flink处理迟到数据的机制可以解决这个问题

​	*一般大数据场景都是考虑高并发情况，所以一般使用周期性生成Watermark的方式，避免频繁地生成Watermark。*

---

**注：一般认为Watermark的设置代码，在里Source步骤越近的地方越合适。**

### 7.3.6 测试代码

测试Watermark和迟到数据

java代码（旧版Flink），新版的代码我暂时不打算折腾，之后用上再说吧。

**这里设置的Watermark的延时时间是2s，实际一般设置和window大小一致。**

```java
public class WindowTest3_EventTimeWindow {
  public static void main(String[] args) throws Exception {
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

    // Flink1.12.X 已经默认就是使用EventTime了，所以不需要这行代码
    //        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
    env.getConfig().setAutoWatermarkInterval(100);

    // socket文本流
    DataStream<String> inputStream = env.socketTextStream("localhost", 7777);

    // 转换成SensorReading类型，分配时间戳和watermark
    DataStream<SensorReading> dataStream = inputStream.map(line -> {
      String[] fields = line.split(",");
      return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
    })
      //              
      //                // 旧版 (新版官方推荐用assignTimestampsAndWatermarks(WatermarkStrategy) )
      // 升序数据设置事件时间和watermark
      //.assignTimestampsAndWatermarks(new AscendingTimestampExtractor<SensorReading>() {
      //  @Override
      //  public long extractAscendingTimestamp(SensorReading element) {
      //    return element.getTimestamp() * 1000L;
      //  }
      //})
      
      // 旧版 (新版官方推荐用assignTimestampsAndWatermarks(WatermarkStrategy) )
      // 乱序数据设置时间戳和watermark
      .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<SensorReading>(Time.seconds(2)) {
        @Override
        public long extractTimestamp(SensorReading element) {
          return element.getTimestamp() * 1000L;
        }
      });

    OutputTag<SensorReading> outputTag = new OutputTag<SensorReading>("late") {
    };

    // 基于事件时间的开窗聚合，统计15秒内温度的最小值
    SingleOutputStreamOperator<SensorReading> minTempStream = dataStream.keyBy("id")
      .timeWindow(Time.seconds(15))
      .allowedLateness(Time.minutes(1))
      .sideOutputLateData(outputTag)
      .minBy("temperature");

    minTempStream.print("minTemp");
    minTempStream.getSideOutput(outputTag).print("late");

    env.execute();
  }
}
```

#### 并行任务Watermark传递测试

在前面代码的基础上，修改执行环境并行度为4，进行测试

```java
public class WindowTest3_EventTimeWindow {
  public static void main(String[] args) throws Exception {
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

    env.setParallelism(4);

    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
    env.getConfig().setAutoWatermarkInterval(100);

    // socket文本流
    DataStream<String> inputStream = env.socketTextStream("localhost", 7777);

    // 转换成SensorReading类型，分配时间戳和watermark
    DataStream<SensorReading> dataStream = inputStream.map(line -> {
      String[] fields = line.split(",");
      return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
    })
      
      // 乱序数据设置时间戳和watermark
      .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<SensorReading>(Time.seconds(2)) {
        @Override
        public long extractTimestamp(SensorReading element) {
          return element.getTimestamp() * 1000L;
        }
      });

    OutputTag<SensorReading> outputTag = new OutputTag<SensorReading>("late") {
    };

    // 基于事件时间的开窗聚合，统计15秒内温度的最小值
    SingleOutputStreamOperator<SensorReading> minTempStream = dataStream.keyBy("id")
      .timeWindow(Time.seconds(15))
      .allowedLateness(Time.minutes(1))
      .sideOutputLateData(outputTag)
      .minBy("temperature");

    minTempStream.print("minTemp");
    minTempStream.getSideOutput(outputTag).print("late");

    env.execute();
  }
}
```

启动本地socket，输入数据，查看结果

```shell
nc -lk 7777
```

输入：

```shell
sensor_1,1547718199,35.8
sensor_6,1547718201,15.4
sensor_7,1547718202,6.7
sensor_10,1547718205,38.1
sensor_1,1547718207,36.3
sensor_1,1547718211,34
sensor_1,1547718212,31.9
sensor_1,1547718212,31.9
sensor_1,1547718212,31.9
sensor_1,1547718212,31.9
```

输出

*注意：上面输入全部输入后，才突然有下面4条输出！*

```shell
minTemp:2> SensorReading{id='sensor_10', timestamp=1547718205, temperature=38.1}
minTemp:3> SensorReading{id='sensor_1', timestamp=1547718199, temperature=35.8}
minTemp:4> SensorReading{id='sensor_7', timestamp=1547718202, temperature=6.7}
minTemp:3> SensorReading{id='sensor_6', timestamp=1547718201, temperature=15.4}
```

##### 分析

1. **计算窗口起始位置Start和结束位置End**

   从`TumblingProcessingTimeWindows`类里的`assignWindows`方法，我们可以得知窗口的起点计算方法如下：
   $$
   窗口起点start = timestamp - (timestamp -offset+WindowSize) \% WindowSize
   $$
   由于我们没有设置offset，所以这里`start=第一个数据的时间戳1547718199-(1547718199-0+15)%15=1547718195`

   计算得到窗口初始位置为`Start = 1547718195`，那么这个窗口理论上本应该在1547718195+15的位置关闭，也就是`End=1547718210`

   ```java
   @Override
   public Collection<TimeWindow> assignWindows(
     Object element, long timestamp, WindowAssignerContext context) {
     final long now = context.getCurrentProcessingTime();
     if (staggerOffset == null) {
       staggerOffset =
         windowStagger.getStaggerOffset(context.getCurrentProcessingTime(), size);
     }
     long start =
       TimeWindow.getWindowStartWithOffset(
       now, (globalOffset + staggerOffset) % size, size);
     return Collections.singletonList(new TimeWindow(start, start + size));
   }
   
   // 跟踪 getWindowStartWithOffset 方法得到TimeWindow的方法
   public static long getWindowStartWithOffset(long timestamp, long offset, long windowSize) {
     return timestamp - (timestamp - offset + windowSize) % windowSize;
   }
   ```

2. **计算修正后的Window输出结果的时间**

   测试代码中Watermark设置的`maxOutOfOrderness`最大乱序程度是2s，所以实际获取到End+2s的时间戳数据时（达到Watermark），才认为Window需要输出计算的结果（不关闭，因为设置了允许迟到1min）

   **所以实际应该是1547718212的数据到来时才触发Window输出计算结果。**

   ```java
   .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<SensorReading>(Time.seconds(2)) {
     @Override
     public long extractTimestamp(SensorReading element) {
       return element.getTimestamp() * 1000L;
     }
   });
   
   
   // BoundedOutOfOrdernessTimestampExtractor.java
   public BoundedOutOfOrdernessTimestampExtractor(Time maxOutOfOrderness) {
     if (maxOutOfOrderness.toMilliseconds() < 0) {
       throw new RuntimeException(
         "Tried to set the maximum allowed "
         + "lateness to "
         + maxOutOfOrderness
         + ". This parameter cannot be negative.");
     }
     this.maxOutOfOrderness = maxOutOfOrderness.toMilliseconds();
     this.currentMaxTimestamp = Long.MIN_VALUE + this.maxOutOfOrderness;
   }
   @Override
   public final Watermark getCurrentWatermark() {
     // this guarantees that the watermark never goes backwards.
     long potentialWM = currentMaxTimestamp - maxOutOfOrderness;
     if (potentialWM >= lastEmittedWatermark) {
       lastEmittedWatermark = potentialWM;
     }
     return new Watermark(lastEmittedWatermark);
   }
   ```

3. 为什么上面输入中，最后连续四条相同输入，才触发Window输出结果？

   + **Watermark会向子任务广播**
     + 我们在map才设置Watermark，map根据Rebalance轮询方式分配数据。所以前4个输入分别到4个slot中，4个slot计算得出的Watermark不同（分别是1547718199-2，1547718201-2，1547718202-2，1547718205-2）

   + **Watermark传递时，会选择当前接收到的最小一个作为自己的Watermark**
     + 前4次输入中，有些map子任务还没有接收到数据，所以其下游的keyBy后的slot里watermark就是`Long.MIN_VALUE`（因为4个上游的Watermark广播最小值就是默认的`Long.MIN_VALUE`）
     + 并行度4，在最后4个相同的输入，使得Rebalance到4个map子任务的数据的`currentMaxTimestamp`都是1547718212，经过`getCurrentWatermark()`的计算（`currentMaxTimestamp-maxOutOfOrderness`），4个子任务都计算得到watermark=1547718210，4个map子任务向4个keyBy子任务广播`watermark=1547718210`，使得keyBy子任务们获取到4个上游的Watermark最小值就是1547718210，然后4个KeyBy子任务都更新自己的Watermark为1547718210。
   + **根据Watermark的定义，我们认为>=Watermark的数据都已经到达。由于此时watermark >= 窗口End，所以Window输出计算结果（4个子任务，4个结果）。**

### 7.3.7 窗口起始点和偏移量

> [flink-Window Assingers(窗口分配器)中offset偏移量](https://juejin.cn/post/6844904110941011976)

​	时间偏移一个很大的用处是用来调准非0时区的窗口，例如:在中国你需要指定一个8小时的时间偏移。

# 8. Flink状态管理

> [Flink_Flink中的状态](https://blog.csdn.net/dongkang123456/article/details/108430338)
>
> [Flink状态管理详解：Keyed State和Operator List State深度解析](https://zhuanlan.zhihu.com/p/104171679)	<=	不错的文章，建议阅读

+ 算子状态（Operator State）
+ 键控状态（Keyed State）
+ 状态后端（State Backends）

## 8.1 状态概述

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200906125916475.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvbmdrYW5nMTIzNDU2,size_16,color_FFFFFF,t_70#pic_center)

- 由一个任务维护，并且用来计算某个结果的所有数据，都属于这个任务的状态
- 可以认为任务状态就是一个本地变量，可以被任务的业务逻辑访问
- **Flink 会进行状态管理，包括状态一致性、故障处理以及高效存储和访问，以便于开发人员可以专注于应用程序的逻辑**

---

- **在Flink中，状态始终与特定算子相关联**
- 为了使运行时的Flink了解算子的状态，算子需要预先注册其状态

**总的来说，有两种类型的状态：**

+ **算子状态（Operator State）**
  + 算子状态的作用范围限定为**算子任务**（也就是不能跨任务访问）
+ **键控状态（Keyed State）**
  + 根据输入数据流中定义的键（key）来维护和访问

## 8.2 算子状态 Operator State

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200906173949148.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvbmdrYW5nMTIzNDU2,size_16,color_FFFFFF,t_70#pic_center)

+ 算子状态的作用范围限定为算子任务，同一并行任务所处理的所有数据都可以访问到相同的状态。

+ 状态对于**同一任务**而言是共享的。（**不能跨slot**）

+ 状态算子不能由相同或不同算子的另一个任务访问。

### 算子状态数据结构

+ 列表状态(List state) 
  +  将状态表示为一组数据的列表

+ 联合列表状态(Union list state)
  + 也将状态表示未数据的列表。它与常规列表状态的区别在于，在发生故障时，或者从保存点(savepoint)启动应用程序时如何恢复

+ 广播状态(Broadcast state)
  + 如果一个算子有多项任务，而它的每项任务状态又都相同，那么这种特殊情况最适合应用广播状态

### 测试代码

实际一般用算子状态比较少，一般还是键控状态用得多一点。

```java
package apitest.state;

import apitest.beans.SensorReading;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.streaming.api.checkpoint.ListCheckpointed;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

import java.util.Collections;
import java.util.List;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2021/2/2 4:05 AM
 */
public class StateTest1_OperatorState {

  public static void main(String[] args) throws Exception {
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    env.setParallelism(1);

    // socket文本流
    DataStream<String> inputStream = env.socketTextStream("localhost", 7777);

    // 转换成SensorReading类型
    DataStream<SensorReading> dataStream = inputStream.map(line -> {
      String[] fields = line.split(",");
      return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
    });

    // 定义一个有状态的map操作，统计当前分区数据个数
    SingleOutputStreamOperator<Integer> resultStream = dataStream.map(new MyCountMapper());

    resultStream.print();

    env.execute();
  }

  // 自定义MapFunction
  public static class MyCountMapper implements MapFunction<SensorReading, Integer>, ListCheckpointed<Integer> {
    // 定义一个本地变量，作为算子状态
    private Integer count = 0;

    @Override
    public Integer map(SensorReading value) throws Exception {
      count++;
      return count;
    }

    @Override
    public List<Integer> snapshotState(long checkpointId, long timestamp) throws Exception {
      return Collections.singletonList(count);
    }

    @Override
    public void restoreState(List<Integer> state) throws Exception {
      for (Integer num : state) {
        count += num;
      }
    }
  }
}
```

输入(本地开启socket后输入)

```shell
sensor_1,1547718199,35.8
sensor_1,1547718199,35.8
sensor_1,1547718199,35.8
sensor_1,1547718199,35.8
sensor_1,1547718199,35.8
```

输出

```shell
1
2
3
4
5
```

## 8.3 键控状态 Keyed State

> [Flink_Flink中的状态](https://blog.csdn.net/dongkang123456/article/details/108430338)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200906182710217.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvbmdrYW5nMTIzNDU2,size_16,color_FFFFFF,t_70#pic_center)

+ 键控状态是根据输入数据流中定义的键（key）来维护和访问的。

+ **Flink 为每个key维护一个状态实例，并将具有相同键的所有数据，都分区到同一个算子任务中，这个任务会维护和处理这个key对应的状态。**

+ **当任务处理一条数据时，他会自动将状态的访问范围限定为当前数据的key**。

### 键控状态数据结构

+ 值状态(value state)
  + 将状态表示为单个的值

+ 列表状态(List state)
  + 将状态表示为一组数据的列表

+ 映射状态(Map state)
  + 将状态表示为一组key-value对

+ **聚合状态(Reducing state & Aggregating State)**
  + 将状态表示为一个用于聚合操作的列表

### 测试代码

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200906183806458.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvbmdrYW5nMTIzNDU2,size_16,color_FFFFFF,t_70#pic_center)

*注：声明一个键控状态，一般在算子的open()中声明，因为运行时才能获取上下文信息*

+ java测试代码

  ```java
  package apitest.state;
  
  import apitest.beans.SensorReading;
  import org.apache.flink.api.common.functions.RichMapFunction;
  import org.apache.flink.api.common.state.*;
  import org.apache.flink.configuration.Configuration;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/2 5:41 PM
   */
  public class StateTest2_KeyedState {
  
    public static void main(String[] args) throws Exception {
      // 创建执行环境
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      // 设置并行度 = 1
      env.setParallelism(1);
      // 从本地socket读取数据
      DataStream<String> inputStream = env.socketTextStream("localhost", 7777);
  
      // 转换成SensorReading类型
      DataStream<SensorReading> dataStream = inputStream.map(line -> {
        String[] fields = line.split(",");
        return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
      });
  
      // 使用自定义map方法，里面使用 我们自定义的Keyed State
      DataStream<Integer> resultStream = dataStream
        .keyBy(SensorReading::getId)
        .map(new MyMapper());
  
      resultStream.print("result");
      env.execute();
    }
  
    // 自定义map富函数，测试 键控状态
    public static class MyMapper extends RichMapFunction<SensorReading,Integer>{
  
      //        Exception in thread "main" java.lang.IllegalStateException: The runtime context has not been initialized.
      //        ValueState<Integer> valueState = getRuntimeContext().getState(new ValueStateDescriptor<Integer>("my-int", Integer.class));
  
      private ValueState<Integer> valueState;
  
  
      // 其它类型状态的声明
      private ListState<String> myListState;
      private MapState<String, Double> myMapState;
      private ReducingState<SensorReading> myReducingState;
  
      @Override
      public void open(Configuration parameters) throws Exception {
        valueState = getRuntimeContext().getState(new ValueStateDescriptor<Integer>("my-int", Integer.class));
  
        myListState = getRuntimeContext().getListState(new ListStateDescriptor<String>("my-list", String.class));
        myMapState = getRuntimeContext().getMapState(new MapStateDescriptor<String, Double>("my-map", String.class, Double.class));
        //            myReducingState = getRuntimeContext().getReducingState(new ReducingStateDescriptor<SensorReading>())
  
      }
  
      // 这里就简单的统计每个 传感器的 信息数量
      @Override
      public Integer map(SensorReading value) throws Exception {
        // 其它状态API调用
        // list state
        for(String str: myListState.get()){
          System.out.println(str);
        }
        myListState.add("hello");
        // map state
        myMapState.get("1");
        myMapState.put("2", 12.3);
        myMapState.remove("2");
        // reducing state
        //            myReducingState.add(value);
  
        myMapState.clear();
  
  
        Integer count = valueState.value();
        // 第一次获取是null，需要判断
        count = count==null?0:count;
        ++count;
        valueState.update(count);
        return count;
      }
    }
  }
  ```

### 场景测试

假设做一个温度报警，如果一个传感器前后温差超过10度就报警。这里使用键控状态Keyed State + flatMap来实现

+ java代码

  ```java
  package apitest.state;
  
  import apitest.beans.SensorReading;
  import org.apache.flink.api.common.functions.RichFlatMapFunction;
  import org.apache.flink.api.common.state.ValueState;
  import org.apache.flink.api.common.state.ValueStateDescriptor;
  import org.apache.flink.api.java.tuple.Tuple3;
  import org.apache.flink.configuration.Configuration;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.util.Collector;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/2 6:37 PM
   */
  public class StateTest3_KeyedStateApplicationCase {
  
    public static void main(String[] args) throws Exception {
      // 创建执行环境
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      // 设置并行度 = 1
      env.setParallelism(1);
      // 从socket获取数据
      DataStream<String> inputStream = env.socketTextStream("localhost", 7777);
      // 转换为SensorReading类型
      DataStream<SensorReading> dataStream = inputStream.map(line -> {
        String[] fields = line.split(",");
        return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
      });
  
      SingleOutputStreamOperator<Tuple3<String, Double, Double>> resultStream = dataStream.keyBy(SensorReading::getId).flatMap(new MyFlatMapper(10.0));
  
      resultStream.print();
  
      env.execute();
    }
  
    // 如果 传感器温度 前后差距超过指定温度(这里指定10.0),就报警
    public static class MyFlatMapper extends RichFlatMapFunction<SensorReading, Tuple3<String, Double, Double>> {
  
      // 报警的温差阈值
      private final Double threshold;
  
      // 记录上一次的温度
      ValueState<Double> lastTemperature;
  
      public MyFlatMapper(Double threshold) {
        this.threshold = threshold;
      }
  
      @Override
      public void open(Configuration parameters) throws Exception {
        // 从运行时上下文中获取keyedState
        lastTemperature = getRuntimeContext().getState(new ValueStateDescriptor<Double>("last-temp", Double.class));
      }
  
      @Override
      public void close() throws Exception {
        // 手动释放资源
        lastTemperature.clear();
      }
  
      @Override
      public void flatMap(SensorReading value, Collector<Tuple3<String, Double, Double>> out) throws Exception {
        Double lastTemp = lastTemperature.value();
        Double curTemp = value.getTemperature();
  
        // 如果不为空，判断是否温差超过阈值，超过则报警
        if (lastTemp != null) {
          if (Math.abs(curTemp - lastTemp) >= threshold) {
            out.collect(new Tuple3<>(value.getId(), lastTemp, curTemp));
          }
        }
  
        // 更新保存的"上一次温度"
        lastTemperature.update(curTemp);
      }
    }
  }
  ```

+ 启动socket

  ```shell
  nc -lk 7777
  ```

+ 输入数据，查看结果

  + 输入

    ```shell
    sensor_1,1547718199,35.8
    sensor_1,1547718199,32.4
    sensor_1,1547718199,42.4
    sensor_10,1547718205,52.6   
    sensor_10,1547718205,22.5
    sensor_7,1547718202,6.7
    sensor_7,1547718202,9.9
    sensor_1,1547718207,36.3
    sensor_7,1547718202,19.9
    sensor_7,1547718202,30
    ```

  + 输出

    *中间没有输出（sensor_7,9.9,19.9)，应该是double浮点数计算精度问题，不管它*

    ```shell
    (sensor_1,32.4,42.4)
    (sensor_10,52.6,22.5)
    (sensor_7,19.9,30.0)
    ```

## 8.4 状态后端 State Backends

> [Flink_Flink中的状态](https://blog.csdn.net/dongkang123456/article/details/108430338)

### 8.4.1 概述

+ 每传入一条数据，有状态的算子任务都会读取和更新状态。

+ 由于有效的状态访问对于处理数据的低延迟至关重要，因此每个并行任务都会在本地维护其状态，以确保快速的状态访问。

+ 状态的存储、访问以及维护，由一个可插入的组件决定，这个组件就叫做**状态后端( state backend)**

+ **状态后端主要负责两件事：本地状态管理，以及将检查点(checkPoint)状态写入远程存储**

### 8.4.2 选择一个状态后端

+ MemoryStateBackend
  + 内存级的状态后端，会将键控状态作为内存中的对象进行管理，将它们存储在TaskManager的JVM堆上，而将checkpoint存储在JobManager的内存中
  + 特点：快速、低延迟，但不稳定
+ FsStateBackend（默认）
  + 将checkpoint存到远程的持久化文件系统（FileSystem）上，而对于本地状态，跟MemoryStateBackend一样，也会存在TaskManager的JVM堆上
  + 同时拥有内存级的本地访问速度，和更好的容错保证
+ RocksDBStateBackend
  + 将所有状态序列化后，存入本地的RocksDB中存储

### 8.4.3 配置文件

`flink-conf.yaml`

```yaml
#==============================================================================
# Fault tolerance and checkpointing
#==============================================================================

# The backend that will be used to store operator state checkpoints if
# checkpointing is enabled.
#
# Supported backends are 'jobmanager', 'filesystem', 'rocksdb', or the
# <class-name-of-factory>.
#
# state.backend: filesystem
上面这个就是默认的checkpoint存在filesystem


# Directory for checkpoints filesystem, when using any of the default bundled
# state backends.
#
# state.checkpoints.dir: hdfs://namenode-host:port/flink-checkpoints

# Default target directory for savepoints, optional.
#
# state.savepoints.dir: hdfs://namenode-host:port/flink-savepoints

# Flag to enable/disable incremental checkpoints for backends that
# support incremental checkpoints (like the RocksDB state backend). 
#
# state.backend.incremental: false

# The failover strategy, i.e., how the job computation recovers from task failures.
# Only restart tasks that may have been affected by the task failure, which typically includes
# downstream tasks and potentially upstream tasks if their produced data is no longer available for consumption.

jobmanager.execution.failover-strategy: region

上面这个region指，多个并行度的任务要是有个挂掉了，只重启那个任务所属的region（可能含有多个子任务），而不需要重启整个Flink程序
```

### 8.4.4 样例代码

+ 其中使用RocksDBStateBackend需要另外加入pom依赖

  ```xml
  <!-- RocksDBStateBackend -->
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-statebackend-rocksdb_${scala.binary.version}</artifactId>
    <version>${flink.version}</version>
  </dependency>
  ```

+ java代码

  ```java
  package apitest.state;
  
  import apitest.beans.SensorReading;
  import org.apache.flink.api.common.restartstrategy.RestartStrategies;
  import org.apache.flink.api.common.time.Time;
  import org.apache.flink.contrib.streaming.state.RocksDBStateBackend;
  import org.apache.flink.runtime.state.filesystem.FsStateBackend;
  import org.apache.flink.runtime.state.memory.MemoryStateBackend;
  import org.apache.flink.streaming.api.CheckpointingMode;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/2 11:35 PM
   */
  public class StateTest4_FaultTolerance {
      public static void main(String[] args) throws Exception {
          StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
          env.setParallelism(1);
  
          // 1. 状态后端配置
          env.setStateBackend(new MemoryStateBackend());
          env.setStateBackend(new FsStateBackend("checkpointDataUri"));
          // 这个需要另外导入依赖
          env.setStateBackend(new RocksDBStateBackend("checkpointDataUri"));
  
          // socket文本流
          DataStream<String> inputStream = env.socketTextStream("localhost", 7777);
  
          // 转换成SensorReading类型
          DataStream<SensorReading> dataStream = inputStream.map(line -> {
              String[] fields = line.split(",");
              return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
          });
  
          dataStream.print();
          env.execute();
      }
  }
  ```

# 9. ProcessFunction API(底层API)

​	我们之前学习的**转换算子**是无法访问事件的<u>时间戳信息和水位线信息</u>的。而这在一些应用场景下，极为重要。例如MapFunction这样的map转换算子就无法访问时间戳或者当前事件的事件时间。

​	基于此，DataStream API提供了一系列的Low-Level转换算子。可以**访问时间戳**、**watermark**以及**注册定时事件**。还可以输出**特定的一些事件**，例如超时事件等。<u>Process Function用来构建事件驱动的应用以及实现自定义的业务逻辑(使用之前的window函数和转换算子无法实现)。例如，FlinkSQL就是使用Process Function实现的</u>。

Flink提供了8个Process Function：

- ProcessFunction
- KeyedProcessFunction
- CoProcessFunction
- ProcessJoinFunction
- BroadcastProcessFunction
- KeyedBroadcastProcessFunction
- ProcessWindowFunction
- ProcessAllWindowFunction

## 9.1 KeyedProcessFunction

​	这个是相对比较常用的ProcessFunction，根据名字就可以知道是用在keyedStream上的。

​	KeyedProcessFunction用来操作KeyedStream。KeyedProcessFunction会处理流的每一个元素，输出为0个、1个或者多个元素。所有的Process Function都继承自RichFunction接口，所以都有`open()`、`close()`和`getRuntimeContext()`等方法。而`KeyedProcessFunction<K, I, O>`还额外提供了两个方法:

+ `processElement(I value, Context ctx, Collector<O> out)`，流中的每一个元素都会调用这个方法，调用结果将会放在Collector数据类型中输出。Context可以访问元素的时间戳，元素的 key ，以及TimerService 时间服务。 Context 还可以将结果输出到别的流(side outputs)。
+ `onTimer(long timestamp, OnTimerContext ctx, Collector<O> out)`，是一个回调函数。当之前注册的定时器触发时调用。参数timestamp 为定时器所设定的触发的时间戳。Collector 为输出结果的集合。OnTimerContext和processElement的Context 参数一样，提供了上下文的一些信息，例如定时器触发的时间信息(事件时间或者处理时间)。

### 测试代码

设置一个获取数据后第5s给出提示信息的定时器。

```java
package processfunction;

import apitest.beans.SensorReading;
import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.api.java.tuple.Tuple;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
import org.apache.flink.util.Collector;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2021/2/3 12:30 AM
 */
public class ProcessTest1_KeyedProcessFunction {
  public static void main(String[] args) throws Exception{
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    env.setParallelism(1);

    // socket文本流
    DataStream<String> inputStream = env.socketTextStream("localhost", 7777);

    // 转换成SensorReading类型
    DataStream<SensorReading> dataStream = inputStream.map(line -> {
      String[] fields = line.split(",");
      return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
    });

    // 测试KeyedProcessFunction，先分组然后自定义处理
    dataStream.keyBy("id")
      .process( new MyProcess() )
      .print();

    env.execute();
  }

  // 实现自定义的处理函数
  public static class MyProcess extends KeyedProcessFunction<Tuple, SensorReading, Integer> {
    ValueState<Long> tsTimerState;

    @Override
    public void open(Configuration parameters) throws Exception {
      tsTimerState =  getRuntimeContext().getState(new ValueStateDescriptor<Long>("ts-timer", Long.class));
    }

    @Override
    public void processElement(SensorReading value, Context ctx, Collector<Integer> out) throws Exception {
      out.collect(value.getId().length());

      // context
      // Timestamp of the element currently being processed or timestamp of a firing timer.
      ctx.timestamp();
      // Get key of the element being processed.
      ctx.getCurrentKey();
      //            ctx.output();
      ctx.timerService().currentProcessingTime();
      ctx.timerService().currentWatermark();
      // 在5处理时间的5秒延迟后触发
      ctx.timerService().registerProcessingTimeTimer( ctx.timerService().currentProcessingTime() + 5000L);
      tsTimerState.update(ctx.timerService().currentProcessingTime() + 1000L);
      //            ctx.timerService().registerEventTimeTimer((value.getTimestamp() + 10) * 1000L);
      // 删除指定时间触发的定时器
      //            ctx.timerService().deleteProcessingTimeTimer(tsTimerState.value());
    }

    @Override
    public void onTimer(long timestamp, OnTimerContext ctx, Collector<Integer> out) throws Exception {
      System.out.println(timestamp + " 定时器触发");
      ctx.getCurrentKey();
      //            ctx.output();
      ctx.timeDomain();
    }

    @Override
    public void close() throws Exception {
      tsTimerState.clear();
    }
  }
}
```

启动本地socket

```shell
nc -lk 7777
```

输入

```shell
sensor_1,1547718207,36.3
```

输出

```shell
8
1612283803911 定时器触发
```

## 9.2 TimerService和定时器(Timers)

​	Context 和OnTimerContext 所持有的TimerService 对象拥有以下方法：

+ `long currentProcessingTime()` 返回当前处理时间

+ `long currentWatermark()` 返回当前watermark 的时间戳

+ `void registerProcessingTimeTimer( long timestamp)` 会注册当前key的processing time的定时器。当processing time 到达定时时间时，触发timer。

+ **`void registerEventTimeTimer(long timestamp)` 会注册当前key 的event time 定时器。当Watermark水位线大于等于定时器注册的时间时，触发定时器执行回调函数。**

+ `void deleteProcessingTimeTimer(long timestamp)` 删除之前注册处理时间定时器。如果没有这个时间戳的定时器，则不执行。

+ `void deleteEventTimeTimer(long timestamp)` 删除之前注册的事件时间定时器，如果没有此时间戳的定时器，则不执行。

​	**当定时器timer 触发时，会执行回调函数onTimer()。注意定时器timer 只能在keyed streams 上面使用。**

### 测试代码

下面举个例子说明KeyedProcessFunction 如何操作KeyedStream。

需求：监控温度传感器的温度值，如果温度值在10 秒钟之内(processing time)连续上升，则报警。

+ java代码

  ```java
  package processfunction;
  
  import apitest.beans.SensorReading;
  import org.apache.flink.api.common.state.ValueState;
  import org.apache.flink.api.common.state.ValueStateDescriptor;
  import org.apache.flink.api.common.time.Time;
  import org.apache.flink.configuration.Configuration;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
  import org.apache.flink.util.Collector;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/3 1:02 AM
   */
  public class ProcessTest2_ApplicationCase {
  
    public static void main(String[] args) throws Exception {
      // 创建执行环境
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      // 设置并行度为1
      env.setParallelism(1);
      // 从socket中获取数据
      DataStream<String> inputStream = env.socketTextStream("localhost", 7777);
      // 转换数据为SensorReading类型
      DataStream<SensorReading> sensorReadingStream = inputStream.map(line -> {
        String[] fields = line.split(",");
        return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
      });
      // 如果存在连续10s内温度持续上升的情况，则报警
      sensorReadingStream.keyBy(SensorReading::getId)
        .process(new TempConsIncreWarning(Time.seconds(10).toMilliseconds()))
        .print();
      env.execute();
    }
  
    // 如果存在连续10s内温度持续上升的情况，则报警
    public static class TempConsIncreWarning extends KeyedProcessFunction<String, SensorReading, String> {
  
      public TempConsIncreWarning(Long interval) {
        this.interval = interval;
      }
  
      // 报警的时间间隔(如果在interval时间内温度持续上升，则报警)
      private Long interval;
  
      // 上一个温度值
      private ValueState<Double> lastTemperature;
      // 最近一次定时器的触发时间(报警时间)
      private ValueState<Long> recentTimerTimeStamp;
  
      @Override
      public void open(Configuration parameters) throws Exception {
        lastTemperature = getRuntimeContext().getState(new ValueStateDescriptor<Double>("lastTemperature", Double.class));
        recentTimerTimeStamp = getRuntimeContext().getState(new ValueStateDescriptor<Long>("recentTimerTimeStamp", Long.class));
      }
  
      @Override
      public void close() throws Exception {
        lastTemperature.clear();
        recentTimerTimeStamp.clear();
      }
  
      @Override
      public void processElement(SensorReading value, Context ctx, Collector<String> out) throws Exception {
        // 当前温度值
        double curTemp = value.getTemperature();
        // 上一次温度(没有则设置为当前温度)
        double lastTemp = lastTemperature.value() != null ? lastTemperature.value() : curTemp;
        // 计时器状态值(时间戳)
        Long timerTimestamp = recentTimerTimeStamp.value();
  
        // 如果 当前温度 > 上次温度 并且 没有设置报警计时器，则设置
        if (curTemp > lastTemp && null == timerTimestamp) {
          long warningTimestamp = ctx.timerService().currentProcessingTime() + interval;
          ctx.timerService().registerProcessingTimeTimer(warningTimestamp);
          recentTimerTimeStamp.update(warningTimestamp);
        }
        // 如果 当前温度 < 上次温度，且 设置了报警计时器，则清空计时器
        else if (curTemp <= lastTemp && timerTimestamp != null) {
          ctx.timerService().deleteProcessingTimeTimer(timerTimestamp);
          recentTimerTimeStamp.clear();
        }
        // 更新保存的温度值
        lastTemperature.update(curTemp);
      }
  
      // 定时器任务
      @Override
      public void onTimer(long timestamp, OnTimerContext ctx, Collector<String> out) throws Exception {
        // 触发报警，并且清除 定时器状态值
        out.collect("传感器" + ctx.getCurrentKey() + "温度值连续" + interval + "ms上升");
        recentTimerTimeStamp.clear();
      }
    }
  }
  ```

+ 启动本地socket，之后输入数据

  ```shell
  nc -lk 7777
  ```

  + 输入

    ```shell
    sensor_1,1547718199,35.8
    sensor_1,1547718199,34.1
    sensor_1,1547718199,34.2
    sensor_1,1547718199,35.1
    sensor_6,1547718201,15.4
    sensor_7,1547718202,6.7
    sensor_10,1547718205,38.1
    sensor_10,1547718205,39  
    sensor_6,1547718201,18  
    sensor_7,1547718202,9.1
    ```

  + 输出

    ```shell
    传感器sensor_1温度值连续10000ms上升
    传感器sensor_10温度值连续10000ms上升
    传感器sensor_6温度值连续10000ms上升
    传感器sensor_7温度值连续10000ms上升
    ```

## 9.3 侧输出流（SideOutput）

+ **一个数据可以被多个window包含，只有其不被任何window包含的时候(包含该数据的所有window都关闭之后)，才会被丢到侧输出流。**
+ **简言之，如果一个数据被丢到侧输出流，那么所有包含该数据的window都由于已经超过了"允许的迟到时间"而关闭了，进而新来的迟到数据只能被丢到侧输出流！**

----

+ 大部分的DataStream API 的算子的输出是单一输出，也就是某种数据类型的流。除了split 算子，可以将一条流分成多条流，这些流的数据类型也都相同。

+ **processfunction 的side outputs 功能可以产生多条流，并且这些流的数据类型可以不一样。**

+ 一个side output 可以定义为OutputTag[X]对象，X 是输出流的数据类型。

+ processfunction 可以通过Context 对象发射一个事件到一个或者多个side outputs。

### 测试代码

场景：温度>=30放入高温流输出，反之放入低温流输出

+ java代码

  ```java
  package processfunction;
  
  import apitest.beans.SensorReading;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.ProcessFunction;
  import org.apache.flink.util.Collector;
  import org.apache.flink.util.OutputTag;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/3 2:07 AM
   */
  public class ProcessTest3_SideOuptCase {
    public static void main(String[] args) throws Exception {
      // 创建执行环境
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      // 设置并行度 = 1
      env.setParallelism(1);
      // 从本地socket读取数据
      DataStream<String> inputStream = env.socketTextStream("localhost", 7777);
      // 转换成SensorReading类型
      DataStream<SensorReading> dataStream = inputStream.map(line -> {
        String[] fields = line.split(",");
        return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
      });
  
      // 定义一个OutputTag，用来表示侧输出流低温流
      // An OutputTag must always be an anonymous inner class
      // so that Flink can derive a TypeInformation for the generic type parameter.
      OutputTag<SensorReading> lowTempTag = new OutputTag<SensorReading>("lowTemp"){};
  
      // 测试ProcessFunction，自定义侧输出流实现分流操作
      SingleOutputStreamOperator<SensorReading> highTempStream = dataStream.process(new ProcessFunction<SensorReading, SensorReading>() {
        @Override
        public void processElement(SensorReading value, Context ctx, Collector<SensorReading> out) throws Exception {
          // 判断温度，大于30度，高温流输出到主流；小于低温流输出到侧输出流
          if (value.getTemperature() > 30) {
            out.collect(value);
          } else {
            ctx.output(lowTempTag, value);
          }
        }
      });
  
      highTempStream.print("high-temp");
      highTempStream.getSideOutput(lowTempTag).print("low-temp");
  
      env.execute();
    }
  }
  ```

+ 本地启动socket

  + 输入

    ```shell
    sensor_1,1547718199,35.8
    sensor_6,1547718201,15.4
    sensor_7,1547718202,6.7
    sensor_10,1547718205,38.1
    ```

  + 输出

    ```shell
    high-temp> SensorReading{id='sensor_1', timestamp=1547718199, temperature=35.8}
    low-temp> SensorReading{id='sensor_6', timestamp=1547718201, temperature=15.4}
    low-temp> SensorReading{id='sensor_7', timestamp=1547718202, temperature=6.7}
    high-temp> SensorReading{id='sensor_10', timestamp=1547718205, temperature=38.1}
    ```

## 9.4 CoProcessFunction

+ 对于两条输入流，DataStream API 提供了CoProcessFunction 这样的low-level操作。CoProcessFunction 提供了操作每一个输入流的方法: `processElement1()`和`processElement2()`。

+ **类似于ProcessFunction，这两种方法都通过Context 对象来调用**。<u>这个Context对象可以访问事件数据，定时器时间戳，TimerService，以及side outputs</u>。
+ **CoProcessFunction 也提供了onTimer()回调函数**。

# 10. 容错机制

> [Flink-容错机制 | 一致性检查点 | 检查点到恢复状态过程 | Flink检查点算法(Chandy-Lamport) | 算法操作解析 | 保存点简介](https://blog.csdn.net/qq_40180229/article/details/106433621)

## 10.1 一致性检查点(checkpoint)

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020052922013278.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ Flink 故障恢复机制的核心，就是应用状态的一致性检查点

+ 有状态流应用的一致检查点，其实就是**所有任务的状态**，在某个时间点的一份拷贝（一份快照）；**这个时间点，应该是所有任务都恰好处理完一个相同的输入数据的时候**

  *(5这个数据虽然进了奇数流但是偶数流也应该做快照，因为属于同一个相同数据，只是没有被他处理)*

  *（这里根据奇偶性分流，偶数流求偶数和，奇数流求奇数和，5这里明显已经被sum_odd（1+3+5）处理了，且sum_even不需要处理该数据，因为前面已经判断该数据不需要到sum_even流，相当于所有任务都已经处理完source的数据5了。）*

+ 在JobManager中也有个Chechpoint的指针，指向了仓库的状态快照的一个拓扑图，为以后的数据故障恢复做准备

## 10.2 从检查点恢复状态

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200529220326395.png)

+ 在执行流应用程序期间，Flink 会定期保存状态的一致检查点

+ 如果发生故障， Flink 将会使用最近的检查点来一致恢复应用程序的状态，并重新启动处理流程

  （**如图中所示，7这个数据被source读到了，准备传给奇数流时，奇数流宕机了，数据传输发生中断**）

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200529220452315.png)

+ 遇到故障之后，第一步就是重启应用

  (**重启后，起初流都是空的**)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200529220546658.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ 第二步是从 checkpoint 中读取状态，将状态重置

  *(**读取在远程仓库**(Storage，这里的仓库指状态后端保存数据指定的三种方式之一)**保存的状态**)*

+ 从检查点重新启动应用程序后，其内部状态与检查点完成时的状态完全相同

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200529220850257.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ 第三步：开始消费并处理检查点到发生故障之间的所有数据

+ **这种检查点的保存和恢复机制可以为应用程序状态提供“精确一次”（exactly-once）的一致性，因为所有算子都会保存检查点并恢复其所有状态，这样一来所有的输入流就都会被重置到检查点完成时的位置**

  *（这里要求source源也能记录状态，回退到读取数据7的状态，kafka有相应的偏移指针能完成该操作）*

## 10.3 Flink检查点算法

### 概述

**checkpoint和Watermark一样，都会以广播的形式告诉所有下游。**

---

+ 一种简单的想法

  暂停应用，保存状态到检查点，再重新恢复应用（当然Flink 不是采用这种简单粗暴的方式）

+ Flink的改进实现

  + 基于Chandy-Lamport算法的分布式快照
  + 将检查点的保存和数据处理分离开，不暂停整个应用

  （就是每个任务单独拍摄自己的快照到内存，之后再到jobManager整合）

---

+ 检查点分界线（Checkpoint Barrier）
  + Flink的检查点算法用到了一种称为分界线（barrier）的特殊数据形式，用来把一条流上数据按照不同的检查点分开
  + **分界线之前到来的数据导致的状态更改，都会被包含在当前分界线所属的检查点中；而基于分界线之后的数据导致的所有更改，就会被包含在之后的检查点中**

### 具体讲解

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200529224034243.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ 现在是一个有两个输入流的应用程序，用并行的两个 Source 任务来读取

+ 两条自然数数据流，蓝色数据流已经输出完`蓝3`了，黄色数据流输出完`黄4`了

+ 在Souce端 Source1 接收到了数据`蓝3` 正在往下游发向一个数据`蓝2 和 蓝3`； Source2 接受到了数据`黄4`，且往下游发送数据`黄4`

+ 偶数流已经处理完`黄2` 所以后面显示为2， 奇数流处理完`蓝1 和 黄1 黄3` 所以为5，并分别往下游发送每次聚合后的结果给Sink

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200529224517502.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ **JobManager 会向每个 source 任务发送一条带有新检查点 ID 的消息**，通过这种方式来启动检查点

  *（这个带有新检查点ID的东西为**barrier**，由图中三角型表示，数值2只是ID）*

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200529224705177.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ 数据源将它们的状态写入检查点，并发出一个检查点barrier
+ 状态后端在状态存入检查点之后，会返回通知给source任务，source任务就会向JobManager确认检查点完成

​	*上图，在Source端接受到barrier后，将自己此身的3 和 4 的数据的状态写入检查点，且向JobManager发送checkpoint成功的消息，然后向下游分别发出一个检查点 barrier*

​	*可以看出在Source接受barrier时，数据流也在不断的处理，不会进行中断*

​	*此时的偶数流已经处理完`蓝2`变成了4，但是还没处理到`黄4`，只是下游sink发送了一个数据4，而奇数流已经处理完`蓝3`变成了8（黄1+蓝1+黄3+蓝3），并向下游sink发送了8*

​	*此时检查点barrier都还未到Sum_odd奇数流和Sum_even偶数流*

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200529225235834.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ **分界线对齐：barrier向下游传递，sum任务会等待所有输入分区的barrier到达**
+ **对于barrier已经达到的分区，继续到达的数据会被缓存**
+ **而barrier尚未到达的分区，数据会被正常处理**

​	*此时蓝色流的barrier先一步抵达了偶数流，黄色的barrier还未到，但是因为数据的不中断一直处理，此时的先到的蓝色的barrier会将此时的偶数流的数据4进行缓存处理，流接着处理接下来的数据等待着黄色的barrier的到来，而黄色barrier之前的数据将会对缓存的数据相加*

​	*这次处理的总结：**分界线对齐**：**barrier 向下游传递，sum 任务会等待所有输入分区的 barrier 到达，对于barrier已经到达的分区，继续到达的数据会被缓存。而barrier尚未到达的分区，数据会被正常处理***

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200529225656902.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ **当收到所有输入分区的 barrier 时，任务就将其状态保存到状态后端的检查点中，然后将 barrier 继续向下游转发**

​	*当蓝色的barrier和黄色的barrier(所有分区的)都到达后，进行状态保存到远程仓库，**然后对JobManager发送消息，说自己的检查点保存完毕了***

​	*此时的偶数流和奇数流都为8*

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200529230413317.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ 向下游转发检查点 barrier 后，任务继续正常的数据处理

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020052923042436.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ **Sink 任务向 JobManager 确认状态保存到 checkpoint 完毕**
+ **当所有任务都确认已成功将状态保存到检查点时，检查点就真正完成了**

## 10.4 保存点(Savepoints)

**CheckPoint为自动保存，SavePoint为手动保存**

+ Flink还提供了可以自定义的镜像保存功能，就是保存点（save points）
+ 原则上，创建保存点使用的算法与检查点完全相同，因此保存点可以认为就是具有一些额外元数据的检查点
+ Flink不会自动创建保存点，因此用户（或者外部调度程序）必须明确地触发创建操作
+ 保存点是一个强大的功能。除了故障恢复外，保存点可以用于：有计划的手动备份、更新应用程序、版本迁移、暂停和重启程序，等等

## 10.5 检查点和重启策略配置

+ java样例代码

  ```java
  package apitest.state;
  
  import apitest.beans.SensorReading;
  import org.apache.flink.api.common.restartstrategy.RestartStrategies;
  import org.apache.flink.api.common.time.Time;
  import org.apache.flink.contrib.streaming.state.RocksDBStateBackend;
  import org.apache.flink.runtime.state.filesystem.FsStateBackend;
  import org.apache.flink.runtime.state.memory.MemoryStateBackend;
  import org.apache.flink.streaming.api.CheckpointingMode;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/2 11:35 PM
   */
  public class StateTest4_FaultTolerance {
    public static void main(String[] args) throws Exception {
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
  
      // 1. 状态后端配置
      env.setStateBackend(new MemoryStateBackend());
      env.setStateBackend(new FsStateBackend(""));
      // 这个需要另外导入依赖
      env.setStateBackend(new RocksDBStateBackend(""));
  
      // 2. 检查点配置 (每300ms让jobManager进行一次checkpoint检查)
      env.enableCheckpointing(300);
  
      // 高级选项
      env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
      //Checkpoint的处理超时时间
      env.getCheckpointConfig().setCheckpointTimeout(60000L);
      // 最大允许同时处理几个Checkpoint(比如上一个处理到一半，这里又收到一个待处理的Checkpoint事件)
      env.getCheckpointConfig().setMaxConcurrentCheckpoints(2);
      // 与上面setMaxConcurrentCheckpoints(2) 冲突，这个时间间隔是 当前checkpoint的处理完成时间与接收最新一个checkpoint之间的时间间隔
      env.getCheckpointConfig().setMinPauseBetweenCheckpoints(100L);
      // 如果同时开启了savepoint且有更新的备份，是否倾向于使用更老的自动备份checkpoint来恢复，默认false
      env.getCheckpointConfig().setPreferCheckpointForRecovery(true);
      // 最多能容忍几次checkpoint处理失败（默认0，即checkpoint处理失败，就当作程序执行异常）
      env.getCheckpointConfig().setTolerableCheckpointFailureNumber(0);
  
      // 3. 重启策略配置
      // 固定延迟重启(最多尝试3次，每次间隔10s)
      env.setRestartStrategy(RestartStrategies.fixedDelayRestart(3, 10000L));
      // 失败率重启(在10分钟内最多尝试3次，每次至少间隔1分钟)
      env.setRestartStrategy(RestartStrategies.failureRateRestart(3, Time.minutes(10), Time.minutes(1)));
  
      // socket文本流
      DataStream<String> inputStream = env.socketTextStream("localhost", 7777);
  
      // 转换成SensorReading类型
      DataStream<SensorReading> dataStream = inputStream.map(line -> {
        String[] fields = line.split(",");
        return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
      });
  
      dataStream.print();
      env.execute();
    }
  }
  ```

## 10.6 状态一致性

> [Flink-状态一致性 | 状态一致性分类 | 端到端状态一致性 | 幂等写入 | 事务写入 | WAL | 2PC](https://blog.csdn.net/qq_40180229/article/details/106445029)

### 10.6.1 概述

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530181851687.png)

+ 有状态的流处理，内部每个算子任务都可以有自己的状态

+ 对于流处理器内部来说，所谓的状态一致性，其实就是我们所说的计算结果要保证准确。

+ 一条数据不应该丢失，也不应该重复计算

+ 在遇到故障时可以恢复状态，恢复以后的重新计算，结果应该也是完全正确的。

### 10.6.2 分类

​	**Flink的一个重大价值在于，它既保证了exactly-once，也具有低延迟和高吞吐的处理能力。**

1. AT-MOST-ONCE（最多一次）
   当任务故障时，最简单的做法是什么都不干，既不恢复丢失的状态，也不重播丢失的数据。At-most-once 语义的含义是最多处理一次事件。

   *这其实是没有正确性保障的委婉说法——故障发生之后，计算结果可能丢失。类似的比如网络协议的udp。*

2. AT-LEAST-ONCE（至少一次）
   在大多数的真实应用场景，我们希望不丢失事件。这种类型的保障称为 at-least-once，意思是所有的事件都得到了处理，而一些事件还可能被处理多次。

   *这表示计数结果可能大于正确值，但绝不会小于正确值。也就是说，计数程序在发生故障后可能多算，但是绝不会少算。*

3. EXACTLY-ONCE（精确一次）
   **恰好处理一次是最严格的保证，也是最难实现的。恰好处理一次语义不仅仅意味着没有事件丢失，还意味着针对每一个数据，内部状态仅仅更新一次。**

   *这指的是系统保证在发生故障后得到的计数结果与正确值一致。*

### 10.6.3 一致性检查点(Checkpoints)

+ Flink使用了一种轻量级快照机制——检查点（checkpoint）来保证exactly-once语义
+ 有状态流应用的一致检查点，其实就是：所有任务的状态，在某个时间点的一份备份（一份快照）。而这个时间点，应该是所有任务都恰好处理完一个相同的输入数据的时间。
+ 应用状态的一致检查点，是Flink故障恢复机制的核心

#### 端到端(end-to-end)状态一致性

+ 目前我们看到的一致性保证都是由流处理器实现的，也就是说都是在Flink流处理器内部保证的；而在真实应用中，流处理应用除了流处理器以外还包含了数据源（例如Kafka）和输出到持久化系统

+ 端到端的一致性保证，意味着结果的正确性贯穿了整个流处理应用的始终；每一个组件都保证了它自己的一致性
+ **整个端到端的一致性级别取决于所有组件中一致性最弱的组件**

#### 端到端 exactly-once

+ 内部保证——checkpoint
+ source端——可重设数据的读取位置
+ sink端——从故障恢复时，数据不会重复写入外部系统
  + 幂等写入
  + 事务写入

##### 幂等写入

+ 所谓幂等操作，是说一个操作，可以重复执行很多次，但只导致一次结果更改，也就是说，后面再重复执行就不起作用了。

  *（中间可能会存在不正确的情况，只能保证最后结果正确。比如5=>10=>15=>5=>10=>15，虽然最后是恢复到了15，但是中间有个恢复的过程，如果这个过程能够被读取，就会出问题。）*

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020053019091138.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

##### 事务写入

+ 事务（Transaction）

  + 应用程序中一系列严密的操作，所有操作必须成功完成，否则在每个操作中所作的所有更改都会被撤销
  + 具有原子性：一个事务中的一系列的操作要么全部成功，要么一个都不做

+ 实现思想

  **构建的事务对应着checkpoint，等到checkpoint真正完成的时候，才把所有对应的结果写入sink系统中**。

+ 实现方式
  + 预习日志
  + 两阶段提交

###### 预写日志(Write-Ahead-Log，WAL)

+ 把结果数据先当成状态保存，然后在收到checkpoint完成的通知时，一次性写入sink系统
+ 简单易于实现，由于数据提前在状态后端中做了缓存，所以无论什么sink系统，都能用这种方式一批搞定
+ DataStream API提供了一个模版类：GenericWriteAheadSink，来实现这种事务性sink

###### 两阶段提交(Two-Phase-Commit，2PC)

+ 对于每个checkpoint，sink任务会启动一个事务，并将接下来所有接收到的数据添加到事务里
+ 然后将这些数据写入外部sink系统，但不提交它们——这时只是"预提交"
+ **这种方式真正实现了exactly-once，它需要一个提供事务支持的外部sink系统**。Flink提供了TwoPhaseCommitSinkFunction接口

#### 不同Source和Sink的一致性保证

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530194322578.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

## 10.7 Flink+Kafka 端到端状态一致性的保证

> [Flink-状态一致性 | 状态一致性分类 | 端到端状态一致性 | 幂等写入 | 事务写入 | WAL | 2PC](https://blog.csdn.net/qq_40180229/article/details/106445029)

+ 内部——利用checkpoint机制，把状态存盘，发生故障的时候可以恢复，保证内部的状态一致性
+ source——kafka consumer作为source，可以将偏移量保存下来，如果后续任务出现了故障，恢复的时候可以由连接器重制偏移量，重新消费数据，保证一致性
+ sink——kafka producer作为sink，采用两阶段提交sink，需要实现一个TwoPhaseCommitSinkFunction

### Exactly-once 两阶段提交

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530194434435.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ JobManager 协调各个 TaskManager 进行 checkpoint 存储
+ checkpoint保存在 StateBackend中，默认StateBackend是内存级的，也可以改为文件级的进行持久化保存

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530194627287.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ 当 checkpoint 启动时，JobManager 会将检查点分界线（barrier）注入数据流
+ barrier会在算子间传递下去

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530194657186.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ 每个算子会对当前的状态做个快照，保存到状态后端
+ checkpoint 机制可以保证内部的状态一致性

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530194835593.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ 每个内部的 transform 任务遇到 barrier 时，都会把状态存到 checkpoint 里

+ sink 任务首先把数据写入外部 kafka，**这些数据都属于预提交的事务**；**遇到 barrier 时，把状态保存到状态后端，并开启新的预提交事务**

  *(barrier之前的数据还是在之前的事务中没关闭事务，遇到barrier后的数据另外新开启一个事务)*

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020053019485194.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ 当所有算子任务的快照完成，也就是这次的 checkpoint 完成时，JobManager 会向所有任务发通知，确认这次 checkpoint 完成
+ sink 任务收到确认通知，正式提交之前的事务，kafka 中未确认数据改为“已确认”

### Exactly-once 两阶段提交步骤总结

1. 第一条数据来了之后，开启一个 kafka 的事务（transaction），正常写入 kafka 分区日志但标记为未提交，这就是“预提交”
2. jobmanager 触发 checkpoint 操作，barrier 从 source 开始向下传递，遇到 barrier 的算子将状态存入状态后端，并通知 jobmanager
3. sink 连接器收到 barrier，保存当前状态，存入 checkpoint，通知 jobmanager，并开启下一阶段的事务，用于提交下个检查点的数据
4. jobmanager 收到所有任务的通知，发出确认信息，表示 checkpoint 完成
5. sink 任务收到 jobmanager 的确认信息，正式提交这段时间的数据
6. 外部kafka关闭事务，提交的数据可以正常消费了。

# 11. Table API和Flink SQL

> [Flink-Table API 和 Flink SQL简介 | 新老版本Flink批流处理对比 | 读取文件和Kafka消费数据 | API 和 SQL查询表](https://blog.csdn.net/qq_40180229/article/details/106457648)
>
> [flink-Table&sql-碰到的几个问题记录](https://blog.csdn.net/weixin_41956627/article/details/110050094)

## 11.1 概述

+ Flink 对批处理和流处理，提供了统一的上层 API
+ Table API 是一套内嵌在 Java 和 Scala 语言中的查询API，它允许以非常直观的方式组合来自一些关系运算符的查询
+ Flink 的 SQL 支持基于实现了 SQL 标准的 Apache Calcite

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200531165328668.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

###  使用样例

+ 导入pom依赖，1.11.X之后，推荐使用blink版本

  ```xml
  <!-- Table API 和 Flink SQL -->
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-planner-blink_${scala.binary.version}</artifactId>
    <version>${flink.version}</version>
  </dependency>
  ```

+ java样例代码

  ```java
  package apitest.tableapi;
  
  import apitest.beans.SensorReading;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.table.api.Table;
  import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
  import org.apache.flink.types.Row;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/3 5:47 AM
   */
  public class TableTest1_Example {
    public static void main(String[] args) throws Exception {
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
  
      // 1. 读取数据
      DataStream<String> inputStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");
  
      // 2. 转换成POJO
      DataStream<SensorReading> dataStream = inputStream.map(line -> {
        String[] fields = line.split(",");
        return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
      });
  
      // 3. 创建表环境
      StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);
  
      // 4. 基于流创建一张表
      Table dataTable = tableEnv.fromDataStream(dataStream);
  
      // 5. 调用table API进行转换操作
      Table resultTable = dataTable.select("id, temperature")
        .where("id = 'sensor_1'");
  
      // 6. 执行SQL
      tableEnv.createTemporaryView("sensor", dataTable);
      String sql = "select id, temperature from sensor where id = 'sensor_1'";
      Table resultSqlTable = tableEnv.sqlQuery(sql);
  
      tableEnv.toAppendStream(resultTable, Row.class).print("result");
      tableEnv.toAppendStream(resultSqlTable, Row.class).print("sql");
  
      env.execute();
    }
  }
  ```

+ Txt文件

  ```txt
  sensor_1,1547718199,35.8
  sensor_6,1547718201,15.4
  sensor_7,1547718202,6.7
  sensor_10,1547718205,38.1
  sensor_1,1547718207,36.3
  sensor_1,1547718209,32.8
  sensor_1,1547718212,37.1
  ```

+ 输出结果

  ```shell
  result> sensor_1,35.8
  sql> sensor_1,35.8
  result> sensor_1,36.3
  sql> sensor_1,36.3
  result> sensor_1,32.8
  sql> sensor_1,32.8
  result> sensor_1,37.1
  sql> sensor_1,37.1
  ```

## 11.2 基本程序结构

+ Table API和SQL的程序结构，与流式处理的程序结构十分类似

  ```java
  StreamTableEnvironment tableEnv = ... // 创建表的执行环境
  
  // 创建一张表，用于读取数据
  tableEnv.connect(...).createTemporaryTable("inputTable");
  
  // 注册一张表，用于把计算结果输出
  tableEnv.connect(...).createTemporaryTable("outputTable");
  
  // 通过 Table API 查询算子，得到一张结果表
  Table result = tableEnv.from("inputTable").select(...);
  
  // 通过SQL查询语句，得到一张结果表
  Table sqlResult = tableEnv.sqlQuery("SELECT ... FROM inputTable ...");
  
  // 将结果表写入输出表中
  result.insertInto("outputTable");
  ```

## 11.3 Table API批处理和流处理

新版本blink，真正把批处理、流处理都以DataStream实现。

### 创建环境-样例代码

```java
package apitest.tableapi;


import org.apache.flink.api.java.ExecutionEnvironment;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.TableEnvironment;
import org.apache.flink.table.api.bridge.java.BatchTableEnvironment;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.table.api.internal.TableEnvironmentImpl;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2021/2/3 3:56 PM
 */
public class TableTest2_CommonApi {
  public static void main(String[] args) {
    // 创建执行环境
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    // 设置并行度为1
    env.setParallelism(1);

    StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

    // 1.1 基于老版本planner的流处理
    EnvironmentSettings oldStreamSettings = EnvironmentSettings.newInstance()
      .useOldPlanner()
      .inStreamingMode()
      .build();
    StreamTableEnvironment oldStreamTableEnv = StreamTableEnvironment.create(env,oldStreamSettings);

    // 1.2 基于老版本planner的批处理
    ExecutionEnvironment batchEnv = ExecutionEnvironment.getExecutionEnvironment();
    BatchTableEnvironment oldBatchTableEnv = BatchTableEnvironment.create(batchEnv);

    // 1.3 基于Blink的流处理
    EnvironmentSettings blinkStreamSettings = EnvironmentSettings.newInstance()
      .useBlinkPlanner()
      .inStreamingMode()
      .build();
    StreamTableEnvironment blinkStreamTableEnv = StreamTableEnvironment.create(env,blinkStreamSettings);

    // 1.4 基于Blink的批处理
    EnvironmentSettings blinkBatchSettings = EnvironmentSettings.newInstance()
      .useBlinkPlanner()
      .inBatchMode()
      .build();
    TableEnvironment blinkBatchTableEnv = TableEnvironment.create(blinkBatchSettings);
  }
}
```

### 11.3.1  表(Table)

+ TableEnvironment可以注册目录Catalog，并可以基于Catalog注册表
+ **表(Table)是由一个"标示符"(identifier)来指定的，由3部分组成：Catalog名、数据库(database)名和对象名**
+ 表可以是常规的，也可以是虚拟的(视图，View)
+ 常规表(Table)一般可以用来描述外部数据，比如文件、数据库表或消息队列的数据，也可以直接从DataStream转换而来
+ 视图(View)可以从现有的表中创建，通常是table API或者SQL查询的一个结果集

### 11.3.2 创建表

+ TableEnvironment可以调用`connect()`方法，连接外部系统，并调用`.createTemporaryTable()`方法，在Catalog中注册表

  ```java
  tableEnv
    .connect(...)	//	定义表的数据来源，和外部系统建立连接
    .withFormat(...)	//	定义数据格式化方法
    .withSchema(...)	//	定义表结构
    .createTemporaryTable("MyTable");	//	创建临时表
  ```

### 11.3.3 创建TableEnvironment

+ 创建表的执行环境，需要将flink流处理的执行环境传入

  `StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);`

+ TableEnvironment是flink中集成Table API和SQL的核心概念，所有对表的操作都基于TableEnvironment

  + 注册Catalog
  + 在Catalog中注册表
  + 执行SQL查询
  + 注册用户自定义函数（UDF）

#### 测试代码

+ pom依赖

  ```xml
  <!-- Table API 和 Flink SQL -->
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-planner-blink_${scala.binary.version}</artifactId>
    <version>${flink.version}</version>
  </dependency>
  
  <!-- csv -->
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-csv</artifactId>
    <version>${flink.version}</version>
  </dependency>
  ```

+ java代码

  ```java
  package apitest.tableapi;
  
  
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.table.api.DataTypes;
  import org.apache.flink.table.api.Table;
  import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
  import org.apache.flink.table.descriptors.Csv;
  import org.apache.flink.table.descriptors.FileSystem;
  import org.apache.flink.table.descriptors.Schema;
  import org.apache.flink.types.Row;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/3 3:56 PM
   */
  public class TableTest2_CommonApi {
    public static void main(String[] args) throws Exception {
      // 创建执行环境
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      // 设置并行度为1
      env.setParallelism(1);
  
      StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);
  
      // 2. 表的创建：连接外部系统，读取数据
      // 2.1 读取文件
      String filePath = "/tmp/Flink_Tutorial/src/main/resources/sensor.txt";
  
      tableEnv.connect(new FileSystem().path(filePath)) // 定义到文件系统的连接
        .withFormat(new Csv()) // 定义以csv格式进行数据格式化
        .withSchema(new Schema()
                    .field("id", DataTypes.STRING())
                    .field("timestamp", DataTypes.BIGINT())
                    .field("temp", DataTypes.DOUBLE())
                   ) // 定义表结构
        .createTemporaryTable("inputTable"); // 创建临时表
  
      Table inputTable = tableEnv.from("inputTable");
      inputTable.printSchema();
      tableEnv.toAppendStream(inputTable, Row.class).print();
  
      env.execute();
    }
  }
  ```

+ 输入文件

  ```txt
  sensor_1,1547718199,35.8
  sensor_6,1547718201,15.4
  sensor_7,1547718202,6.7
  sensor_10,1547718205,38.1
  sensor_1,1547718207,36.3
  sensor_1,1547718209,32.8
  sensor_1,1547718212,37.1
  ```

+ 输出

  ```shell
  root
   |-- id: STRING
   |-- timestamp: BIGINT
   |-- temp: DOUBLE
   
  sensor_1,1547718199,35.8
  sensor_6,1547718201,15.4
  sensor_7,1547718202,6.7
  sensor_10,1547718205,38.1
  sensor_1,1547718207,36.3
  sensor_1,1547718209,32.8
  sensor_1,1547718212,37.1
  ```

### 11.3.4 表的查询

+ Table API是集成在Scala和Java语言内的查询API

+ Table API基于代表"表"的Table类，并提供一整套操作处理的方法API；这些方法会返回一个新的Table对象，表示对输入表应用转换操作的结果

+ 有些关系型转换操作，可以由多个方法调用组成，构成链式调用结构

  ```java
  Table sensorTable = tableEnv.from("inputTable");
  Table resultTable = sensorTable
    .select("id","temperature")
    .filter("id = 'sensor_1'");
  ```

#### 从文件获取数据

+ java代码

  ```java
  package apitest.tableapi;
  
  
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.table.api.DataTypes;
  import org.apache.flink.table.api.Table;
  import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
  import org.apache.flink.table.descriptors.Csv;
  import org.apache.flink.table.descriptors.FileSystem;
  import org.apache.flink.table.descriptors.Schema;
  import org.apache.flink.types.Row;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/3 3:56 PM
   */
  public class TableTest2_CommonApi {
    public static void main(String[] args) throws Exception {
      // 创建执行环境
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      // 设置并行度为1
      env.setParallelism(1);
  
      StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);
  
      // 2. 表的创建：连接外部系统，读取数据
      // 2.1 读取文件
      String filePath = "/tmp/Flink_Tutorial/src/main/resources/sensor.txt";
  
      tableEnv.connect(new FileSystem().path(filePath))
        .withFormat(new Csv())
        .withSchema(new Schema()
                    .field("id", DataTypes.STRING())
                    .field("timestamp", DataTypes.BIGINT())
                    .field("temp", DataTypes.DOUBLE())
                   )
        .createTemporaryTable("inputTable");
  
      Table inputTable = tableEnv.from("inputTable");
      //        inputTable.printSchema();
      //        tableEnv.toAppendStream(inputTable, Row.class).print();
  
      // 3. 查询转换
      // 3.1 Table API
      // 简单转换
      Table resultTable = inputTable.select("id, temp")
        .filter("id === 'sensor_6'");
  
      // 聚合统计
      Table aggTable = inputTable.groupBy("id")
        .select("id, id.count as count, temp.avg as avgTemp");
  
      // 3.2 SQL
      tableEnv.sqlQuery("select id, temp from inputTable where id = 'senosr_6'");
      Table sqlAggTable = tableEnv.sqlQuery("select id, count(id) as cnt, avg(temp) as avgTemp from inputTable group by id");
  
      // 打印输出
      tableEnv.toAppendStream(resultTable, Row.class).print("result");
      tableEnv.toRetractStream(aggTable, Row.class).print("agg");
      tableEnv.toRetractStream(sqlAggTable, Row.class).print("sqlagg");
  
      env.execute();
    }
  }
  ```

+ 输出结果

  *里面的false表示上一条保存的记录被删除，true则是新加入的数据*

  *所以Flink的Table API在更新数据时，实际是先删除原本的数据，再添加新数据。*

  ```shell
  result> sensor_6,15.4
  sqlagg> (true,sensor_1,1,35.8)
  sqlagg> (true,sensor_6,1,15.4)
  sqlagg> (true,sensor_7,1,6.7)
  sqlagg> (true,sensor_10,1,38.1)
  agg> (true,sensor_1,1,35.8)
  agg> (true,sensor_6,1,15.4)
  sqlagg> (false,sensor_1,1,35.8)
  sqlagg> (true,sensor_1,2,36.05)
  agg> (true,sensor_7,1,6.7)
  sqlagg> (false,sensor_1,2,36.05)
  sqlagg> (true,sensor_1,3,34.96666666666666)
  agg> (true,sensor_10,1,38.1)
  sqlagg> (false,sensor_1,3,34.96666666666666)
  sqlagg> (true,sensor_1,4,35.5)
  agg> (false,sensor_1,1,35.8)
  agg> (true,sensor_1,2,36.05)
  agg> (false,sensor_1,2,36.05)
  agg> (true,sensor_1,3,34.96666666666666)
  agg> (false,sensor_1,3,34.96666666666666)
  agg> (true,sensor_1,4,35.5)
  ```

#### 数据写入到文件

> [flink Sql 1.11 executeSql报No operators defined in streaming topology. Cannot generate StreamGraph.](https://blog.csdn.net/qq_26502245/article/details/107376528)

​	写入到文件有局限，只能是批处理，且只能是追加写，不能是更新式的随机写。

+ java代码

  ```java
  package apitest.tableapi;
  
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.table.api.DataTypes;
  import org.apache.flink.table.api.Table;
  import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
  import org.apache.flink.table.descriptors.Csv;
  import org.apache.flink.table.descriptors.FileSystem;
  import org.apache.flink.table.descriptors.Schema;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/3 5:53 PM
   */
  public class TableTest3_FileOutput {
    public static void main(String[] args) throws Exception {
      // 1. 创建环境
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
  
      StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);
  
      // 2. 表的创建：连接外部系统，读取数据
      // 读取文件
      String filePath = "/tmp/Flink_Tutorial/src/main/resources/sensor.txt";
      tableEnv.connect(new FileSystem().path(filePath))
        .withFormat(new Csv())
        .withSchema(new Schema()
                    .field("id", DataTypes.STRING())
                    .field("timestamp", DataTypes.BIGINT())
                    .field("temp", DataTypes.DOUBLE())
                   )
        .createTemporaryTable("inputTable");
  
      Table inputTable = tableEnv.from("inputTable");
      //        inputTable.printSchema();
      //        tableEnv.toAppendStream(inputTable, Row.class).print();
  
      // 3. 查询转换
      // 3.1 Table API
      // 简单转换
      Table resultTable = inputTable.select("id, temp")
        .filter("id === 'sensor_6'");
  
      // 聚合统计
      Table aggTable = inputTable.groupBy("id")
        .select("id, id.count as count, temp.avg as avgTemp");
  
      // 3.2 SQL
      tableEnv.sqlQuery("select id, temp from inputTable where id = 'senosr_6'");
      Table sqlAggTable = tableEnv.sqlQuery("select id, count(id) as cnt, avg(temp) as avgTemp from inputTable group by id");
  
      // 4. 输出到文件
      // 连接外部文件注册输出表
      String outputPath = "/tmp/Flink_Tutorial/src/main/resources/out.txt";
      tableEnv.connect(new FileSystem().path(outputPath))
        .withFormat(new Csv())
        .withSchema(new Schema()
                    .field("id", DataTypes.STRING())
                    //                        配合 aggTable.insertInto("outputTable"); 才使用下面这条
                    //                        .field("cnt", DataTypes.BIGINT())
                    .field("temperature", DataTypes.DOUBLE())
                   )
        .createTemporaryTable("outputTable");
  
      resultTable.insertInto("outputTable");
      // 这条会报错(文件系统输出，不支持随机写，只支持附加写)
      // Exception in thread "main" org.apache.flink.table.api.TableException:
      // AppendStreamTableSink doesn't support consuming update changes which is produced by
      // node GroupAggregate(groupBy=[id], select=[id, COUNT(id) AS EXPR$0, AVG(temp) AS EXPR$1])
      //        aggTable.insertInto("outputTable");
  
      // 旧版可以用下面这条
      //        env.execute();
  
      // 新版需要用这条，上面那条会报错，报错如下
      // Exception in thread "main" java.lang.IllegalStateException:
      // No operators defined in streaming topology. Cannot execute.
      tableEnv.execute("");
    }
  }
  ```

+ 输出结果（输出到out.txt文件）

  ```txt
  sensor_6,15.4
  ```

+ 这个程序只能运行一次，再运行一次报错

  ```shell
  Exception in thread "main" org.apache.flink.runtime.client.JobExecutionException: Job execution failed.
  	at org.apache.flink.runtime.jobmaster.JobResult.toJobExecutionResult(JobResult.java:144)
  	at org.apache.flink.runtime.minicluster.MiniClusterJobClient.lambda$getJobExecutionResult$2(MiniClusterJobClient.java:117)
  	at java.util.concurrent.CompletableFuture.uniApply(CompletableFuture.java:616)
  	at java.util.concurrent.CompletableFuture$UniApply.tryFire(CompletableFuture.java:591)
  	at java.util.concurrent.CompletableFuture.postComplete(CompletableFuture.java:488)
  	at java.util.concurrent.CompletableFuture.complete(CompletableFuture.java:1975)
  	at org.apache.flink.runtime.rpc.akka.AkkaInvocationHandler.lambda$invokeRpc$0(AkkaInvocationHandler.java:238)
  	at java.util.concurrent.CompletableFuture.uniWhenComplete(CompletableFuture.java:774)
  	at java.util.concurrent.CompletableFuture$UniWhenComplete.tryFire(CompletableFuture.java:750)
  	at java.util.concurrent.CompletableFuture.postComplete(CompletableFuture.java:488)
  	at java.util.concurrent.CompletableFuture.complete(CompletableFuture.java:1975)
  	at org.apache.flink.runtime.concurrent.FutureUtils$1.onComplete(FutureUtils.java:1046)
  	at akka.dispatch.OnComplete.internal(Future.scala:264)
  	at akka.dispatch.OnComplete.internal(Future.scala:261)
  	at akka.dispatch.japi$CallbackBridge.apply(Future.scala:191)
  	at akka.dispatch.japi$CallbackBridge.apply(Future.scala:188)
  	at scala.concurrent.impl.CallbackRunnable.run$$$capture(Promise.scala:60)
  	at scala.concurrent.impl.CallbackRunnable.run(Promise.scala)
  	at org.apache.flink.runtime.concurrent.Executors$DirectExecutionContext.execute(Executors.java:73)
  	at scala.concurrent.impl.CallbackRunnable.executeWithValue(Promise.scala:68)
  	at scala.concurrent.impl.Promise$DefaultPromise.$anonfun$tryComplete$1(Promise.scala:284)
  	at scala.concurrent.impl.Promise$DefaultPromise.$anonfun$tryComplete$1$adapted(Promise.scala:284)
  	at scala.concurrent.impl.Promise$DefaultPromise.tryComplete(Promise.scala:284)
  	at akka.pattern.PromiseActorRef.$bang(AskSupport.scala:573)
  	at akka.pattern.PipeToSupport$PipeableFuture$$anonfun$pipeTo$1.applyOrElse(PipeToSupport.scala:22)
  	at akka.pattern.PipeToSupport$PipeableFuture$$anonfun$pipeTo$1.applyOrElse(PipeToSupport.scala:21)
  	at scala.concurrent.Future.$anonfun$andThen$1(Future.scala:532)
  	at scala.concurrent.impl.Promise.liftedTree1$1(Promise.scala:29)
  	at scala.concurrent.impl.Promise.$anonfun$transform$1(Promise.scala:29)
  	at scala.concurrent.impl.CallbackRunnable.run$$$capture(Promise.scala:60)
  	at scala.concurrent.impl.CallbackRunnable.run(Promise.scala)
  	at akka.dispatch.BatchingExecutor$AbstractBatch.processBatch(BatchingExecutor.scala:55)
  	at akka.dispatch.BatchingExecutor$BlockableBatch.$anonfun$run$1(BatchingExecutor.scala:91)
  	at scala.runtime.java8.JFunction0$mcV$sp.apply(JFunction0$mcV$sp.java:12)
  	at scala.concurrent.BlockContext$.withBlockContext(BlockContext.scala:81)
  	at akka.dispatch.BatchingExecutor$BlockableBatch.run(BatchingExecutor.scala:91)
  	at akka.dispatch.TaskInvocation.run(AbstractDispatcher.scala:40)
  	at akka.dispatch.ForkJoinExecutorConfigurator$AkkaForkJoinTask.exec(ForkJoinExecutorConfigurator.scala:44)
  	at akka.dispatch.forkjoin.ForkJoinTask.doExec(ForkJoinTask.java:260)
  	at akka.dispatch.forkjoin.ForkJoinPool$WorkQueue.runTask(ForkJoinPool.java:1339)
  	at akka.dispatch.forkjoin.ForkJoinPool.runWorker(ForkJoinPool.java:1979)
  	at akka.dispatch.forkjoin.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:107)
  Caused by: org.apache.flink.runtime.JobException: Recovery is suppressed by NoRestartBackoffTimeStrategy
  	at org.apache.flink.runtime.executiongraph.failover.flip1.ExecutionFailureHandler.handleFailure(ExecutionFailureHandler.java:118)
  	at org.apache.flink.runtime.executiongraph.failover.flip1.ExecutionFailureHandler.getFailureHandlingResult(ExecutionFailureHandler.java:80)
  	at org.apache.flink.runtime.scheduler.DefaultScheduler.handleTaskFailure(DefaultScheduler.java:233)
  	at org.apache.flink.runtime.scheduler.DefaultScheduler.maybeHandleTaskFailure(DefaultScheduler.java:224)
  	at org.apache.flink.runtime.scheduler.DefaultScheduler.updateTaskExecutionStateInternal(DefaultScheduler.java:215)
  	at org.apache.flink.runtime.scheduler.SchedulerBase.updateTaskExecutionState(SchedulerBase.java:665)
  	at org.apache.flink.runtime.scheduler.SchedulerNG.updateTaskExecutionState(SchedulerNG.java:89)
  	at org.apache.flink.runtime.jobmaster.JobMaster.updateTaskExecutionState(JobMaster.java:447)
  	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
  	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
  	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
  	at java.lang.reflect.Method.invoke(Method.java:498)
  	at org.apache.flink.runtime.rpc.akka.AkkaRpcActor.handleRpcInvocation(AkkaRpcActor.java:306)
  	at org.apache.flink.runtime.rpc.akka.AkkaRpcActor.handleRpcMessage(AkkaRpcActor.java:213)
  	at org.apache.flink.runtime.rpc.akka.FencedAkkaRpcActor.handleRpcMessage(FencedAkkaRpcActor.java:77)
  	at org.apache.flink.runtime.rpc.akka.AkkaRpcActor.handleMessage(AkkaRpcActor.java:159)
  	at akka.japi.pf.UnitCaseStatement.apply(CaseStatements.scala:26)
  	at akka.japi.pf.UnitCaseStatement.apply(CaseStatements.scala:21)
  	at scala.PartialFunction.applyOrElse(PartialFunction.scala:123)
  	at scala.PartialFunction.applyOrElse$(PartialFunction.scala:122)
  	at akka.japi.pf.UnitCaseStatement.applyOrElse(CaseStatements.scala:21)
  	at scala.PartialFunction$OrElse.applyOrElse(PartialFunction.scala:171)
  	at scala.PartialFunction$OrElse.applyOrElse(PartialFunction.scala:172)
  	at scala.PartialFunction$OrElse.applyOrElse(PartialFunction.scala:172)
  	at akka.actor.Actor.aroundReceive(Actor.scala:517)
  	at akka.actor.Actor.aroundReceive$(Actor.scala:515)
  	at akka.actor.AbstractActor.aroundReceive(AbstractActor.scala:225)
  	at akka.actor.ActorCell.receiveMessage(ActorCell.scala:592)
  	at akka.actor.ActorCell.invoke(ActorCell.scala:561)
  	at akka.dispatch.Mailbox.processMailbox(Mailbox.scala:258)
  	at akka.dispatch.Mailbox.run(Mailbox.scala:225)
  	at akka.dispatch.Mailbox.exec(Mailbox.scala:235)
  	... 4 more
  Caused by: java.io.IOException: File or directory /Users/ashiamd/mydocs/docs/study/javadocument/javadocument/IDEA_project/Flink_Tutorial/src/main/resources/out.txt already exists. Existing files and directories are not overwritten in NO_OVERWRITE mode. Use OVERWRITE mode to overwrite existing files and directories.
  	at org.apache.flink.core.fs.FileSystem.initOutPathLocalFS(FileSystem.java:874)
  	at org.apache.flink.core.fs.SafetyNetWrapperFileSystem.initOutPathLocalFS(SafetyNetWrapperFileSystem.java:142)
  	at org.apache.flink.api.common.io.FileOutputFormat.open(FileOutputFormat.java:234)
  	at org.apache.flink.api.java.io.TextOutputFormat.open(TextOutputFormat.java:92)
  	at org.apache.flink.streaming.api.functions.sink.OutputFormatSinkFunction.open(OutputFormatSinkFunction.java:65)
  	at org.apache.flink.api.common.functions.util.FunctionUtils.openFunction(FunctionUtils.java:34)
  	at org.apache.flink.streaming.api.operators.AbstractUdfStreamOperator.open(AbstractUdfStreamOperator.java:102)
  	at org.apache.flink.streaming.api.operators.StreamSink.open(StreamSink.java:46)
  	at org.apache.flink.streaming.runtime.tasks.OperatorChain.initializeStateAndOpenOperators(OperatorChain.java:426)
  	at org.apache.flink.streaming.runtime.tasks.StreamTask.lambda$beforeInvoke$2(StreamTask.java:535)
  	at org.apache.flink.streaming.runtime.tasks.StreamTaskActionExecutor$1.runThrowing(StreamTaskActionExecutor.java:50)
  	at org.apache.flink.streaming.runtime.tasks.StreamTask.beforeInvoke(StreamTask.java:525)
  	at org.apache.flink.streaming.runtime.tasks.StreamTask.invoke(StreamTask.java:565)
  	at org.apache.flink.runtime.taskmanager.Task.doRun(Task.java:755)
  	at org.apache.flink.runtime.taskmanager.Task.run(Task.java:570)
  	at java.lang.Thread.run(Thread.java:748)
  	Suppressed: java.lang.NullPointerException
  		at org.apache.flink.streaming.api.functions.source.ContinuousFileReaderOperator.lambda$cleanUp$1(ContinuousFileReaderOperator.java:499)
  		at org.apache.flink.streaming.api.functions.source.ContinuousFileReaderOperator.cleanUp(ContinuousFileReaderOperator.java:512)
  		at org.apache.flink.streaming.api.functions.source.ContinuousFileReaderOperator.dispose(ContinuousFileReaderOperator.java:441)
  		at org.apache.flink.streaming.runtime.tasks.StreamTask.disposeAllOperators(StreamTask.java:783)
  		at org.apache.flink.streaming.runtime.tasks.StreamTask.runAndSuppressThrowable(StreamTask.java:762)
  		at org.apache.flink.streaming.runtime.tasks.StreamTask.cleanUpInvoke(StreamTask.java:681)
  		at org.apache.flink.streaming.runtime.tasks.StreamTask.invoke(StreamTask.java:585)
  		... 3 more
  ```

#### 读写Kafka

Kafka作为消息队列，和文件系统类似的，只能往里追加数据，不能修改数据。

+ 测试代码

  *（我用的新版Flink和新版kafka连接器，所以version指定"universal"）*

  ```java
  package apitest.tableapi;
  
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.table.api.DataTypes;
  import org.apache.flink.table.api.Table;
  import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
  import org.apache.flink.table.descriptors.Csv;
  import org.apache.flink.table.descriptors.Kafka;
  import org.apache.flink.table.descriptors.Schema;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/3 6:33 PM
   */
  public class TableTest4_KafkaPipeLine {
    public static void main(String[] args) throws Exception {
      // 1. 创建环境
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
  
      StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);
  
      // 2. 连接Kafka，读取数据
      tableEnv.connect(new Kafka()
                       .version("universal")
                       .topic("sensor")
                       .property("zookeeper.connect", "localhost:2181")
                       .property("bootstrap.servers", "localhost:9092")
                      )
        .withFormat(new Csv())
        .withSchema(new Schema()
                    .field("id", DataTypes.STRING())
                    .field("timestamp", DataTypes.BIGINT())
                    .field("temp", DataTypes.DOUBLE())
                   )
        .createTemporaryTable("inputTable");
  
      // 3. 查询转换
      // 简单转换
      Table sensorTable = tableEnv.from("inputTable");
      Table resultTable = sensorTable.select("id, temp")
        .filter("id === 'sensor_6'");
  
      // 聚合统计
      Table aggTable = sensorTable.groupBy("id")
        .select("id, id.count as count, temp.avg as avgTemp");
  
      // 4. 建立kafka连接，输出到不同的topic下
      tableEnv.connect(new Kafka()
                       .version("universal")
                       .topic("sinktest")
                       .property("zookeeper.connect", "localhost:2181")
                       .property("bootstrap.servers", "localhost:9092")
                      )
        .withFormat(new Csv())
        .withSchema(new Schema()
                    .field("id", DataTypes.STRING())
                    //                        .field("timestamp", DataTypes.BIGINT())
                    .field("temp", DataTypes.DOUBLE())
                   )
        .createTemporaryTable("outputTable");
  
      resultTable.insertInto("outputTable");
  
      tableEnv.execute("");
    }
  }
  ```

+ 启动kafka目录里自带的zookeeper

  ```shell
  $ bin/zookeeper-server-start.sh config/zookeeper.properties
  ```

+ 启动kafka服务

  ```shell
  $ bin/kafka-server-start.sh config/server.properties
  ```

+ 新建kafka生产者和消费者

  ```shell
  $ bin/kafka-console-producer.sh --broker-list localhost:9092  --topic sensor
  
  $ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic sinktest
  ```

+ 启动Flink程序，在kafka生产者console中输入数据，查看输出

  + 输入（kafka-console-producer）

    ```shell
    >sensor_1,1547718199,35.8
    >sensor_6,1547718201,15.4
    >sensor_7,1547718202,6.7
    >sensor_10,1547718205,38.1
    >sensor_6,1547718209,34.5
    ```

  + 输出（kafka-console-consumer）

    代码中，只筛选id为`sensor_6`的，所以输出没有问题

    ```shell
    sensor_6,15.4           
    
    sensor_6,34.5
    
    
    ```

### 11.3.5 更新模式

+ 对于流式查询，需要声明如何在表和外部连接器之间执行转换
+ 与外部系统交换的消息类型，由更新模式（Uadate Mode）指定
+ 追加（Append）模式
  + 表只做插入操作，和外部连接器只交换插入（Insert）消息
+ 撤回（Retract）模式
  + 表和外部连接器交换添加（Add）和撤回（Retract）消息
  + 插入操作（Insert）编码为Add消息；删除（Delete）编码为Retract消息；**更新（Update）编码为上一条的Retract和下一条的Add消息**
+ 更新插入（Upsert）模式
  + 更新和插入都被编码为Upsert消息；删除编码为Delete消息

### 11.3.6 输出到ES

+ 可以创建Table来描述ES中的数据，作为输出的TableSink

  ```java
  tableEnv.connect(
    new Elasticsearch()
    .version("6")
    .host("localhost",9200,"http")
    .index("sensor")
    .documentType("temp")
  )
    .inUpsertMode()
    .withFormat(new Json())
    .withSchema(new Schema()
                .field("id",DataTypes.STRING())
                .field("count",DataTypes.BIGINT())
               )
    .createTemporaryTable("esOutputTable");
  aggResultTable.insertInto("esOutputTable");
  ```

### 11.3.7 输出到MySQL

+ 需要的pom依赖

  Flink专门为Table API的jdbc连接提供了flink-jdbc连接器，需要先引入依赖

  ```xml
  <!-- https://mvnrepository.com/artifact/org.apache.flink/flink-jdbc -->
  <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-jdbc_2.12</artifactId>
      <version>1.10.3</version>
      <scope>provided</scope>
  </dependency>
  ```

+ 可以创建Table来描述MySql中的数据，作为输入和输出

  ```java
  String sinkDDL = 
    "create table jdbcOutputTable (" +
    " id varchar(20) not null, " +
    " cnt bigint not null " +
    ") with (" +
    " 'connector.type' = 'jdbc', " +
    " 'connector.url' = 'jdbc:mysql://localhost:3306/test', " +
    " 'connector.table' = 'sensor_count', " +
    " 'connector.driver' = 'com.mysql.jdbc.Driver', " +
    " 'connector.username' = 'root', " +
    " 'connector.password' = '123456' )";
  tableEnv.sqlUpdate(sinkDDL);	// 执行DDL创建表
  aggResultSqlTable.insertInto("jdbcOutputTable");
  ```

## 11.4 表和流的转换

> [Flink- 将表转换成DataStream | 查看执行计划 | 流处理和关系代数的区别 | 动态表 | 流式持续查询的过程 | 将流转换成动态表 | 持续查询 | 将动态表转换成 DS](https://blog.csdn.net/qq_40180229/article/details/106479537)

### 11.4.1 将Table转换成DataStream

+ 表可以转换为 DataStream 或 DataSet ，这样自定义流处理或批处理程序就可以继续在 Table API 或 SQL 查询的结果上运行了

+ 将表转换为 DataStream 或 DataSet 时，需要指定生成的数据类型，即要将表的每一行转换成的数据类型

+ 表作为流式查询的结果，是动态更新的

+ 转换有两种转换模式：追加（Appende）模式和撤回（Retract）模式

+ 追加模式

  + 用于表只会被插入（Insert）操作更改的场景

  ```java
  DataStream<Row> resultStream = tableEnv.toAppendStream(resultTable,Row.class);
  ```

+ 撤回模式

  + 用于任何场景。有些类似于更新模式中Retract模式，它只有Insert和Delete两类操作。

  + **得到的数据会增加一个Boolean类型的标识位（返回的第一个字段），用它来表示到底是新增的数据（Insert），还是被删除的数据（Delete）**。

    *(更新数据，会先删除旧数据，再插入新数据)*

### 11.4.2 将DataStream转换成表

+ 对于一个DataStream，可以直接转换成Table，进而方便地调用Table API做转换操作

  ```java
  DataStream<SensorReading> dataStream = ...
  Table sensorTable = tableEnv.fromDataStream(dataStream);
  ```

+ 默认转换后的Table schema和DataStream中的字段定义一一对应，也可以单独指定出来

  ```java
  DataStream<SensorReading> dataStream = ...
  Table sensorTable = tableEnv.fromDataStream(dataStream,
                                             "id, timestamp as ts, temperature");
  ```

### 11.4.3 创建临时视图(Temporary View)

+ 基于DataStream创建临时视图

  ```java
  tableEnv.createTemporaryView("sensorView",dataStream);
  tableEnv.createTemporaryView("sensorView",
                              dataStream, "id, timestamp as ts, temperature");
  ```

+ 基于Table创建临时视图

  ```java
  tableEnv.createTemporaryView("sensorView", sensorTable);
  ```

## 11.5 查看执行计划

+ Table API 提供了一种机制来解释计算表的逻辑和优化查询计划

+ 查看执行计划，可以通过`TableEnvironment.explain(table)`方法或`TableEnvironment.explain()`方法完成，返回一个字符串，描述三个计划

  + 优化的逻辑查询计划
  + 优化后的逻辑查询计划
  + 实际执行计划

  ```java
  String explaination = tableEnv.explain(resultTable);
  System.out.println(explaination);
  ```

## 11.6 流处理和关系代数的区别

> [Flink- 将表转换成DataStream | 查看执行计划 | 流处理和关系代数的区别 | 动态表 | 流式持续查询的过程 | 将流转换成动态表 | 持续查询 | 将动态表转换成 DS](https://blog.csdn.net/qq_40180229/article/details/106479537)

​	Table API和SQL，本质上还是基于关系型表的操作方式；而关系型表、关系代数，以及SQL本身，一般是有界的，更适合批处理的场景。这就导致在进行流处理的过程中，理解会稍微复杂一些，需要引入一些特殊概念。

​	可以看到，其实**关系代数（主要就是指关系型数据库中的表）和SQL，主要就是针对批处理的，这和流处理有天生的隔阂。**

|                           | 关系代数(表)/SQL           | 流处理                                       |
| ------------------------- | -------------------------- | -------------------------------------------- |
| 处理的数据对象            | 字段元组的有界集合         | 字段元组的无限序列                           |
| 查询（Query）对数据的访问 | 可以访问到完整的数据输入   | 无法访问所有数据，必须持续"等待"流式输入     |
| 查询终止条件              | 生成固定大小的结果集后终止 | 永不停止，根据持续收到的数据不断更新查询结果 |

### 11.6.1 动态表(Dynamic Tables)

​	我们可以**随着新数据的到来，不停地在之前的基础上更新结果**。这样得到的表，在Flink Table API概念里，就叫做“动态表”（Dynamic Tables）。

+ 动态表是 Flink 对流数据的 Table API 和 SQL 支持的核心概念
+ 与表示批处理数据的静态表不同，动态表是随时间变化的
+ 持续查询(Continuous Query)
  + 动态表可以像静态的批处理表一样进行查询，查询一个动态表会产生**持续查询（Continuous Query）**
  + **连续查询永远不会终止，并会生成另一个动态表**
  + 查询（Query）会不断更新其动态结果表，以反映其动态输入表上的更改。

### 11.6.2 动态表和持续查询

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200601190927869.png)

流式表查询的处理过程：

1. 流被转换为动态表

2. 对动态表计算连续查询，生成新的动态表

3. 生成的动态表被转换回流

### 11.6.3 将流转换成动态表

+ 为了处理带有关系查询的流，必须先将其转换为表
+ 从概念上讲，流的每个数据记录，都被解释为对结果表的插入（Insert）修改操作

*本质上，我们其实是从一个、只有插入操作的changelog（更新日志）流，来构建一个表*

*来一条数据插入一条数据*

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200601191016684.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

### 11.6.4 持续查询

+ 持续查询，会在动态表上做计算处理，并作为结果生成新的动态表。

  *与批处理查询不同，连续查询从不终止，并根据输入表上的更新更新其结果表。*

  *在任何时间点，连续查询的结果在语义上，等同于在输入表的快照上，以批处理模式执行的同一查询的结果。*

​	下图为一个点击事件流的持续查询，是一个分组聚合做count统计的查询。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200601191112183.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

### 11.6.4 将动态表转换成 DataStream

+ 与常规的数据库表一样，动态表可以通过插入（Insert）、更新（Update）和删除（Delete）更改，进行持续的修改
+ **将动态表转换为流或将其写入外部系统时，需要对这些更改进行编码**

---

+ 仅追加（Append-only）流

  + 仅通过插入（Insert）更改来修改的动态表，可以直接转换为仅追加流

+ 撤回（Retract）流

  + 撤回流是包含两类消息的流：添加（Add）消息和撤回（Retract）消息

    *动态表通过将INSERT 编码为add消息、DELETE 编码为retract消息、UPDATE编码为被更改行（前一行）的retract消息和更新后行（新行）的add消息，转换为retract流。*

+ Upsert（更新插入流）

  + Upsert流也包含两种类型的消息：Upsert消息和删除（Delete）消息

    *通过将INSERT和UPDATE更改编码为upsert消息，将DELETE更改编码为DELETE消息，就可以将具有唯一键（Unique Key）的动态表转换为流。*

### 11.6.5 将动态表转换成DataStream

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200601191549610.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

# 12. 时间特性(Time Attributes)

## 12.1 概述

+ 基于时间的操作（比如 Table API 和 SQL 中窗口操作），需要定义相关的时间语义和时间数据来源的信息
+ Table 可以提供一个逻辑上的时间字段，用于在表处理程序中，指示时间和访问相应的时间戳
+ **时间属性，可以是每个表schema的一部分。一旦定义了时间属性，它就可以作为一个字段引用，并且可以在基于时间的操作中使用**
+ 时间属性的行为类似于常规时间戳，可以访问，并且进行计算

## 12.2 定义处理时间(Processing Time)

+ 处理时间语义下，允许表处理程序根据机器的本地时间生成结果。它是时间的最简单概念。它既不需要提取时间戳，也不需要生成 watermark

### 由DataStream转换成表时指定

+ 在定义 Table Schema 期间，可以使用`.proctime`，指定字段名定义处理时间字段

+ **这个proctime属性只能通过附加逻辑字段，来扩展物理schema。因此，只能在schema定义的末尾定义它**

  ```java
  Table sensorTable = tableEnv.fromDataStream(dataStream,
                                             "id, temperature, pt.proctime");
  ```

### 定义Table Schema时指定

```java
.withSchema(new Schema()
            .field("id", DataTypes.STRING())
            .field("timestamp",DataTypes.BIGINT())
            .field("temperature",DataTypes.DOUBLE())
            .field("pt",DataTypes.TIMESTAMP(3))
            .proctime()
           )
```

### 创建表的DDL中定义

```java
String sinkDDL = 
  "create table dataTable (" +
  " id varchar(20) not null, " +
  " ts bigint, " +
  " temperature double, " +
  " pt AS PROCTIME() " +
  " ) with (" +
  " 'connector.type' = 'filesystem', " +
  " 'connector.path' = '/sensor.txt', " +
  " 'format.type' = 'csv')";
tableEnv.sqlUpdate(sinkDDL);
```

### 测试代码

```java
package apitest.tableapi;

import apitest.beans.SensorReading;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.table.api.Over;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.Tumble;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.types.Row;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2021/2/4 12:47 AM
 */
public class TableTest5_TimeAndWindow {
  public static void main(String[] args) throws Exception {
    // 1. 创建环境
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    env.setParallelism(1);

    StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

    // 2. 读入文件数据，得到DataStream
    DataStream<String> inputStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");

    // 3. 转换成POJO
    DataStream<SensorReading> dataStream = inputStream.map(line -> {
      String[] fields = line.split(",");
      return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
    });

    // 4. 将流转换成表，定义时间特性
    Table dataTable = tableEnv.fromDataStream(dataStream, "id, timestamp as ts, temperature as temp, pt.proctime");

    dataTable.printSchema();
    tableEnv.toAppendStream(dataTable, Row.class).print();

    env.execute();
  }
}
```

输出如下：

```shell
root
 |-- id: STRING
 |-- ts: BIGINT
 |-- temp: DOUBLE
 |-- pt: TIMESTAMP(3) *PROCTIME*

sensor_1,1547718199,35.8,2021-02-03T16:50:58.048
sensor_6,1547718201,15.4,2021-02-03T16:50:58.048
sensor_7,1547718202,6.7,2021-02-03T16:50:58.050
sensor_10,1547718205,38.1,2021-02-03T16:50:58.050
sensor_1,1547718207,36.3,2021-02-03T16:50:58.051
sensor_1,1547718209,32.8,2021-02-03T16:50:58.051
sensor_1,1547718212,37.1,2021-02-03T16:50:58.051
```

## 12.3 定义事件事件(Event Time)

+ 事件时间语义，允许表处理程序根据每个记录中包含的时间生成结果。这样即使在有乱序事件或者延迟事件时，也可以获得正确的结果。
+ **为了处理无序事件，并区分流中的准时和迟到事件；Flink需要从事件数据中，提取时间戳，并用来推送事件时间的进展**
+ 定义事件事件，同样有三种方法：
  + 由DataStream转换成表时指定
  + 定义Table Schema时指定
  + 在创建表的DDL中定义

### 由DataStream转换成表时指定

+ 由DataStream转换成表时指定（推荐）

+ 在DataStream转换成Table，使用`.rowtime`可以定义事件事件属性

  ```java
  // 将DataStream转换为Table，并指定时间字段
  Table sensorTable = tableEnv.fromDataStream(dataStream,
                                             "id, timestamp.rowtime, temperature");
  // 或者，直接追加时间字段
  Table sensorTable = tableEnv.fromDataStream(dataStream,
                                            "id, temperature, timestamp, rt.rowtime");
  ```

### 定义Table Schema时指定

```java
.withSchema(new Schema()
            .field("id", DataTypes.STRING())
            .field("timestamp",DataTypes.BIGINT())
            .rowtime(
              new Rowtime()
              .timestampsFromField("timestamp") // 从字段中提取时间戳
              .watermarksPeriodicBounded(1000) // watermark延迟1秒
            )
            .field("temperature",DataTypes.DOUBLE())
           )
```

### 创建表的DDL中定义

```java
String sinkDDL = 
  "create table dataTable (" +
  " id varchar(20) not null, " +
  " ts bigint, " +
  " temperature double, " +
  " rt AS TO_TIMESTAMP( FROM_UNIXTIME(ts) ), " +
  " watermark for rt as rt - interval '1' second"
  " ) with (" +
  " 'connector.type' = 'filesystem', " +
  " 'connector.path' = '/sensor.txt', " +
  " 'format.type' = 'csv')";
tableEnv.sqlUpdate(sinkDDL);
```

### 测试代码

```java
package apitest.tableapi;

import apitest.beans.SensorReading;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.table.api.Over;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.Tumble;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.types.Row;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2021/2/4 12:47 AM
 */
public class TableTest5_TimeAndWindow {
  public static void main(String[] args) throws Exception {
    // 1. 创建环境
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    env.setParallelism(1);

    StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

    // 2. 读入文件数据，得到DataStream
    DataStream<String> inputStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");

    // 3. 转换成POJO
    DataStream<SensorReading> dataStream = inputStream.map(line -> {
      String[] fields = line.split(",");
      return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
    })
      .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<SensorReading>(Time.seconds(2)) {
        @Override
        public long extractTimestamp(SensorReading element) {
          return element.getTimestamp() * 1000L;
        }
      });

    // 4. 将流转换成表，定义时间特性
    //        Table dataTable = tableEnv.fromDataStream(dataStream, "id, timestamp as ts, temperature as temp, pt.proctime");
    Table dataTable = tableEnv.fromDataStream(dataStream, "id, timestamp as ts, temperature as temp, rt.rowtime");

    dataTable.printSchema();

    tableEnv.toAppendStream(dataTable,Row.class).print();

    env.execute();
  }
}
```

输出如下：

*注：这里最后一列rt里显示的是EventTime，而不是Processing Time*

```shell
root
 |-- id: STRING
 |-- ts: BIGINT
 |-- temp: DOUBLE
 |-- rt: TIMESTAMP(3) *ROWTIME*

sensor_1,1547718199,35.8,2019-01-17T09:43:19
sensor_6,1547718201,15.4,2019-01-17T09:43:21
sensor_7,1547718202,6.7,2019-01-17T09:43:22
sensor_10,1547718205,38.1,2019-01-17T09:43:25
sensor_1,1547718207,36.3,2019-01-17T09:43:27
sensor_1,1547718209,32.8,2019-01-17T09:43:29
sensor_1,1547718212,37.1,2019-01-17T09:43:32
```

## 12.4 窗口

> [Flink-分组窗口 | Over Windows | SQL 中的 Group Windows | SQL 中的 Over Windows](https://blog.csdn.net/qq_40180229/article/details/106482095)

+ 时间语义，要配合窗口操作才能发挥作用。
+ 在Table API和SQL中，主要有两种窗口
  + Group Windows（分组窗口）
    + **根据时间戳或行计数间隔，将行聚合到有限的组（Group）中，并对每个组的数据执行一次聚合函数**
  + Over Windows
    + 针对每个输入行，计算相邻行范围内的聚合

### 12.4.1 Group Windows

+ Group Windows 是使用 window（w:GroupWindow）子句定义的，并且**必须由as子句指定一个别名**。

+ 为了按窗口对表进行分组，窗口的别名必须在 group by 子句中，像常规的分组字段一样引用

  ```scala
  Table table = input
  .window([w:GroupWindow] as "w") // 定义窗口，别名为w
  .groupBy("w, a") // 按照字段 a和窗口 w分组
  .select("a,b.sum"); // 聚合
  ```

+ Table API 提供了一组具有特定语义的预定义 Window 类，这些类会被转换为底层 DataStream 或 DataSet 的窗口操作

+ 分组窗口分为三种：

  + 滚动窗口
  + 滑动窗口
  + 会话窗口

#### 滚动窗口(Tumbling windows)

+ 滚动窗口（Tumbling windows）要用Tumble类来定义

```java
// Tumbling Event-time Window（事件时间字段rowtime）
.window(Tumble.over("10.minutes").on("rowtime").as("w"))

// Tumbling Processing-time Window（处理时间字段proctime）
.window(Tumble.over("10.minutes").on("proctime").as("w"))

// Tumbling Row-count Window (类似于计数窗口，按处理时间排序，10行一组)
.window(Tumble.over("10.rows").on("proctime").as("w"))
```

+ over：定义窗口长度
+ on：用来分组（按时间间隔）或者排序（按行数）的时间字段
+ as：别名，必须出现在后面的groupBy中

#### 滑动窗口(Sliding windows)

+ 滑动窗口（Sliding windows）要用Slide类来定义

```java
// Sliding Event-time Window
.window(Slide.over("10.minutes").every("5.minutes").on("rowtime").as("w"))

// Sliding Processing-time window 
.window(Slide.over("10.minutes").every("5.minutes").on("proctime").as("w"))

// Sliding Row-count window
.window(Slide.over("10.rows").every("5.rows").on("proctime").as("w"))
```

+ over：定义窗口长度
+ every：定义滑动步长
+ on：用来分组（按时间间隔）或者排序（按行数）的时间字段
+ as：别名，必须出现在后面的groupBy中

#### 会话窗口(Session windows)

+ 会话窗口（Session windows）要用Session类来定义

```java
// Session Event-time Window
.window(Session.withGap("10.minutes").on("rowtime").as("w"))

// Session Processing-time Window 
.window(Session.withGap("10.minutes").on("proctime").as("w"))
```

+ withGap：会话时间间隔
+ on：用来分组（按时间间隔）或者排序（按行数）的时间字段
+ as：别名，必须出现在后面的groupBy中

### 12.4.2 SQL中的Group Windows

Group Windows定义在SQL查询的Group By子句中

+ TUMBLE(time_attr, interval)
  + 定义一个滚动窗口，每一个参数是时间字段，第二个参数是窗口长度
+ HOP(time_attr，interval，interval)
  + 定义一个滑动窗口，第一个参数是时间字段，**第二个参数是窗口滑动步长，第三个是窗口长度**
+ SESSION(time_attr，interval)
  + 定义一个绘画窗口，第一个参数是时间字段，第二个参数是窗口间隔

#### 测试代码

```java
package apitest.tableapi;

import apitest.beans.SensorReading;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.table.api.Over;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.Tumble;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.types.Row;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2021/2/4 12:47 AM
 */
public class TableTest5_TimeAndWindow {
  public static void main(String[] args) throws Exception {
    // 1. 创建环境
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    env.setParallelism(1);

    StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

    // 2. 读入文件数据，得到DataStream
    DataStream<String> inputStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");

    // 3. 转换成POJO
    DataStream<SensorReading> dataStream = inputStream.map(line -> {
      String[] fields = line.split(",");
      return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
    })
      .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<SensorReading>(Time.seconds(2)) {
        @Override
        public long extractTimestamp(SensorReading element) {
          return element.getTimestamp() * 1000L;
        }
      });

    // 4. 将流转换成表，定义时间特性
    //        Table dataTable = tableEnv.fromDataStream(dataStream, "id, timestamp as ts, temperature as temp, pt.proctime");
    Table dataTable = tableEnv.fromDataStream(dataStream, "id, timestamp as ts, temperature as temp, rt.rowtime");

    //        dataTable.printSchema();
    //
    //        tableEnv.toAppendStream(dataTable,Row.class).print();

    tableEnv.createTemporaryView("sensor", dataTable);

    // 5. 窗口操作
    // 5.1 Group Window
    // table API
    Table resultTable = dataTable.window(Tumble.over("10.seconds").on("rt").as("tw"))
      .groupBy("id, tw")
      .select("id, id.count, temp.avg, tw.end");

    // SQL
    Table resultSqlTable = tableEnv.sqlQuery("select id, count(id) as cnt, avg(temp) as avgTemp, tumble_end(rt, interval '10' second) " +
                                             "from sensor group by id, tumble(rt, interval '10' second)");

    dataTable.printSchema();
    tableEnv.toAppendStream(resultTable, Row.class).print("result");
    tableEnv.toRetractStream(resultSqlTable, Row.class).print("sql");

    env.execute();
  }
}
```

输出：

```java
root
 |-- id: STRING
 |-- ts: BIGINT
 |-- temp: DOUBLE
 |-- rt: TIMESTAMP(3) *ROWTIME*

result> sensor_1,1,35.8,2019-01-17T09:43:20
result> sensor_6,1,15.4,2019-01-17T09:43:30
result> sensor_1,2,34.55,2019-01-17T09:43:30
result> sensor_10,1,38.1,2019-01-17T09:43:30
result> sensor_7,1,6.7,2019-01-17T09:43:30
sql> (true,sensor_1,1,35.8,2019-01-17T09:43:20)
result> sensor_1,1,37.1,2019-01-17T09:43:40
sql> (true,sensor_6,1,15.4,2019-01-17T09:43:30)
sql> (true,sensor_1,2,34.55,2019-01-17T09:43:30)
sql> (true,sensor_10,1,38.1,2019-01-17T09:43:30)
sql> (true,sensor_7,1,6.7,2019-01-17T09:43:30)
sql> (true,sensor_1,1,37.1,2019-01-17T09:43:40)
```

### 12.4.3 Over Windows

> [SQL中over的用法](https://blog.csdn.net/liuyuehui110/article/details/42736667)
>
> [sql over的作用及用法](https://www.cnblogs.com/xiayang/articles/1886372.html)

+ **Over window 聚合是标准 SQL 中已有的（over 子句），可以在查询的 SELECT 子句中定义**

+ Over window 聚合，会**针对每个输入行**，计算相邻行范围内的聚合

+ Over windows 使用 window（w:overwindows*）子句定义，并在 select（）方法中通过**别名**来引用

  ```scala
  Table table = input
  .window([w: OverWindow] as "w")
  .select("a, b.sum over w, c.min over w");
  ```

+ Table API 提供了 Over 类，来配置 Over 窗口的属性

#### 无界Over Windows

+ 可以在事件时间或处理时间，以及指定为时间间隔、或行计数的范围内，定义 Over windows
+ 无界的 over window 是使用常量指定的

```java
// 无界的事件时间over window (时间字段 "rowtime")
.window(Over.partitionBy("a").orderBy("rowtime").preceding(UNBOUNDED_RANGE).as("w"))

//无界的处理时间over window (时间字段"proctime")
.window(Over.partitionBy("a").orderBy("proctime").preceding(UNBOUNDED_RANGE).as("w"))

// 无界的事件时间Row-count over window (时间字段 "rowtime")
.window(Over.partitionBy("a").orderBy("rowtime").preceding(UNBOUNDED_ROW).as("w"))

//无界的处理时间Row-count over window (时间字段 "rowtime")
.window(Over.partitionBy("a").orderBy("proctime").preceding(UNBOUNDED_ROW).as("w"))
```

*partitionBy是可选项*

#### 有界Over Windows

+ 有界的over window是用间隔的大小指定的

```java
// 有界的事件时间over window (时间字段 "rowtime"，之前1分钟)
.window(Over.partitionBy("a").orderBy("rowtime").preceding("1.minutes").as("w"))

// 有界的处理时间over window (时间字段 "rowtime"，之前1分钟)
.window(Over.partitionBy("a").orderBy("porctime").preceding("1.minutes").as("w"))

// 有界的事件时间Row-count over window (时间字段 "rowtime"，之前10行)
.window(Over.partitionBy("a").orderBy("rowtime").preceding("10.rows").as("w"))

// 有界的处理时间Row-count over window (时间字段 "rowtime"，之前10行)
.window(Over.partitionBy("a").orderBy("proctime").preceding("10.rows").as("w"))
```

### 12.4.4 SQL中的Over Windows

+ 用 Over 做窗口聚合时，所有聚合必须在同一窗口上定义，也就是说必须是相同的分区、排序和范围
+ 目前仅支持在当前行范围之前的窗口
+ ORDER BY 必须在单一的时间属性上指定

```sql
SELECT COUNT(amount) OVER (
  PARTITION BY user
  ORDER BY proctime
  ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)
FROM Orders

// 也可以做多个聚合
SELECT COUNT(amount) OVER w, SUM(amount) OVER w
FROM Orders
WINDOW w AS (
  PARTITION BY user
  ORDER BY proctime
  ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)
```

#### 测试代码

java代码

```java
package apitest.tableapi;

import apitest.beans.SensorReading;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.table.api.Over;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.Tumble;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.types.Row;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2021/2/4 12:47 AM
 */
public class TableTest5_TimeAndWindow {
  public static void main(String[] args) throws Exception {
    // 1. 创建环境
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    env.setParallelism(1);

    StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

    // 2. 读入文件数据，得到DataStream
    DataStream<String> inputStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");

    // 3. 转换成POJO
    DataStream<SensorReading> dataStream = inputStream.map(line -> {
      String[] fields = line.split(",");
      return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
    })
      .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<SensorReading>(Time.seconds(2)) {
        @Override
        public long extractTimestamp(SensorReading element) {
          return element.getTimestamp() * 1000L;
        }
      });

    // 4. 将流转换成表，定义时间特性
    //        Table dataTable = tableEnv.fromDataStream(dataStream, "id, timestamp as ts, temperature as temp, pt.proctime");
    Table dataTable = tableEnv.fromDataStream(dataStream, "id, timestamp as ts, temperature as temp, rt.rowtime");

    //        dataTable.printSchema();
    //
    //        tableEnv.toAppendStream(dataTable,Row.class).print();

    tableEnv.createTemporaryView("sensor", dataTable);

    // 5. 窗口操作
    // 5.1 Group Window
    // table API
    Table resultTable = dataTable.window(Tumble.over("10.seconds").on("rt").as("tw"))
      .groupBy("id, tw")
      .select("id, id.count, temp.avg, tw.end");

    // SQL
    Table resultSqlTable = tableEnv.sqlQuery("select id, count(id) as cnt, avg(temp) as avgTemp, tumble_end(rt, interval '10' second) " +
                                             "from sensor group by id, tumble(rt, interval '10' second)");

    // 5.2 Over Window
    // table API
    Table overResult = dataTable.window(Over.partitionBy("id").orderBy("rt").preceding("2.rows").as("ow"))
      .select("id, rt, id.count over ow, temp.avg over ow");

    // SQL
    Table overSqlResult = tableEnv.sqlQuery("select id, rt, count(id) over ow, avg(temp) over ow " +
                                            " from sensor " +
                                            " window ow as (partition by id order by rt rows between 2 preceding and current row)");

    //        dataTable.printSchema();
    //        tableEnv.toAppendStream(resultTable, Row.class).print("result");
    //        tableEnv.toRetractStream(resultSqlTable, Row.class).print("sql");
    tableEnv.toAppendStream(overResult, Row.class).print("result");
    tableEnv.toRetractStream(overSqlResult, Row.class).print("sql");

    env.execute();
  }
}
```

输出:

*因为`partition by id order by rt rows between 2 preceding and current row`，所以最后2次关于`sensor_1`的输出的`count(id)`都是3,但是计算出来的平均值不一样。（前者计算倒数3条sensor_1的数据，后者计算最后最新的3条sensor_1数据的平均值）*

```shell
result> sensor_1,2019-01-17T09:43:19,1,35.8
sql> (true,sensor_1,2019-01-17T09:43:19,1,35.8)
result> sensor_6,2019-01-17T09:43:21,1,15.4
sql> (true,sensor_6,2019-01-17T09:43:21,1,15.4)
result> sensor_7,2019-01-17T09:43:22,1,6.7
sql> (true,sensor_7,2019-01-17T09:43:22,1,6.7)
result> sensor_10,2019-01-17T09:43:25,1,38.1
sql> (true,sensor_10,2019-01-17T09:43:25,1,38.1)
result> sensor_1,2019-01-17T09:43:27,2,36.05
sql> (true,sensor_1,2019-01-17T09:43:27,2,36.05)
sql> (true,sensor_1,2019-01-17T09:43:29,3,34.96666666666666)
result> sensor_1,2019-01-17T09:43:29,3,34.96666666666666
result> sensor_1,2019-01-17T09:43:32,3,35.4
sql> (true,sensor_1,2019-01-17T09:43:32,3,35.4)
```

# 13. 函数(Functions)

> [Flink-函数 | 用户自定义函数（UDF）标量函数 | 表函数 | 聚合函数 | 表聚合函数](https://blog.csdn.net/qq_40180229/article/details/106482550)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200601214323293.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020060121433777.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

## 13.1 用户自定义函数(UDF)

+ 用户定义函数（User-defined Functions，UDF）是一个重要的特性，它们显著地扩展了查询的表达能力

  *一些系统内置函数无法解决的需求，我们可以用UDF来自定义实现*

+ **在大多数情况下，用户定义的函数必须先注册，然后才能在查询中使用**

+ 函数通过调用 `registerFunction()` 方法在 TableEnvironment 中注册。当用户定义的函数被注册时，它被插入到 TableEnvironment 的函数目录中，这样Table API 或 SQL 解析器就可以识别并正确地解释它

### 13.1.1 标量函数(Scalar Functions)

**Scalar Funcion类似于map，一对一**

**Table Function类似flatMap，一对多**

---

+ 用户定义的标量函数，可以将0、1或多个标量值，映射到新的标量值

+ 为了定义标量函数，必须在 org.apache.flink.table.functions 中扩展基类Scalar Function，并实现（一个或多个）求值（eval）方法

+ **标量函数的行为由求值方法决定，求值方法必须public公开声明并命名为 eval**

  ```java
  public static class HashCode extends ScalarFunction {
  
    private int factor = 13;
  
    public HashCode(int factor) {
      this.factor = factor;
    }
  
    public int eval(String id) {
      return id.hashCode() * 13;
    }
  }
  ```

#### 测试代码

```java
package apitest.tableapi.udf;

import apitest.beans.SensorReading;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.table.functions.ScalarFunction;
import org.apache.flink.types.Row;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2021/2/4 3:28 AM
 */
public class UdfTest1_ScalarFunction {
  public static void main(String[] args) throws Exception {
    // 创建执行环境
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    // 并行度设置为1
    env.setParallelism(1);

    // 创建Table执行环境
    StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

    // 1. 读取数据
    DataStream<String> inputStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");

    // 2. 转换成POJO
    DataStream<SensorReading> dataStream = inputStream.map(line -> {
      String[] fields = line.split(",");
      return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
    });

    // 3. 将流转换为表
    Table sensorTable = tableEnv.fromDataStream(dataStream, "id,timestamp as ts,temperature");

    // 4. 自定义标量函数，实现求id的hash值
    HashCode hashCode = new HashCode(23);
    // 注册UDF
    tableEnv.registerFunction("hashCode", hashCode);

    // 4.1 table API
    Table resultTable = sensorTable.select("id, ts, hashCode(id)");

    // 4.2 SQL
    tableEnv.createTemporaryView("sensor", sensorTable);
    Table resultSqlTable = tableEnv.sqlQuery("select id, ts, hashCode(id) from sensor");

    // 打印输出
    tableEnv.toAppendStream(resultTable, Row.class).print();
    tableEnv.toAppendStream(resultSqlTable, Row.class).print();

    env.execute();
  }

  public static class HashCode extends ScalarFunction {

    private int factor = 13;

    public HashCode(int factor) {
      this.factor = factor;
    }

    public int eval(String id) {
      return id.hashCode() * 13;
    }
  }
}
```

输出结果

```shell
sensor_1,1547718199,-772373508
sensor_1,1547718199,-772373508
sensor_6,1547718201,-772373443
sensor_6,1547718201,-772373443
sensor_7,1547718202,-772373430
sensor_7,1547718202,-772373430
sensor_10,1547718205,1826225652
sensor_10,1547718205,1826225652
sensor_1,1547718207,-772373508
sensor_1,1547718207,-772373508
sensor_1,1547718209,-772373508
sensor_1,1547718209,-772373508
sensor_1,1547718212,-772373508
sensor_1,1547718212,-772373508
```

### 13.1.2 表函数(Table Fcuntions)

**Scalar Funcion类似于map，一对一**

**Table Function类似flatMap，一对多**

---

+ 用户定义的表函数，也可以将0、1或多个标量值作为输入参数；**与标量函数不同的是，它可以返回任意数量的行作为输出，而不是单个值**

+ 为了定义一个表函数，必须扩展 org.apache.flink.table.functions 中的基类 TableFunction 并实现（一个或多个）求值方法

+ **表函数的行为由其求值方法决定，求值方法必须是 public 的，并命名为 eval**

  ```java
  public static class Split extends TableFunction<Tuple2<String, Integer>> {
  
    // 定义属性，分隔符
    private String separator = ",";
  
    public Split(String separator) {
      this.separator = separator;
    }
  
    public void eval(String str) {
      for (String s : str.split(separator)) {
        collect(new Tuple2<>(s, s.length()));
      }
    }
  }
  ```

#### 测试代码

```java
package apitest.tableapi.udf;

import apitest.beans.SensorReading;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.table.functions.TableFunction;
import org.apache.flink.types.Row;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2021/2/4 3:58 AM
 */
public class UdfTest2_TableFunction {
  public static void main(String[] args) throws Exception {
    // 创建执行环境
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    // 并行度设置为1
    env.setParallelism(1);

    // 创建Table执行环境
    StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

    // 1. 读取数据
    DataStream<String> inputStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");

    // 2. 转换成POJO
    DataStream<SensorReading> dataStream = inputStream.map(line -> {
      String[] fields = line.split(",");
      return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
    });

    // 3. 将流转换为表
    Table sensorTable = tableEnv.fromDataStream(dataStream, "id,timestamp as ts,temperature");

    // 4. 自定义表函数，实现将id拆分，并输出（word, length）
    Split split = new Split("_");
    // 需要在环境中注册UDF
    tableEnv.registerFunction("split", split);

    // 4.1 table API
    Table resultTable = sensorTable
      .joinLateral("split(id) as (word, length)")
      .select("id, ts, word, length");

    // 4.2 SQL
    tableEnv.createTemporaryView("sensor", sensorTable);
    Table resultSqlTable = tableEnv.sqlQuery("select id, ts, word, length " +
                                             " from sensor, lateral table(split(id)) as splitid(word, length)");

    // 打印输出
    tableEnv.toAppendStream(resultTable, Row.class).print("result");
    tableEnv.toAppendStream(resultSqlTable, Row.class).print("sql");

    env.execute();
  }

  // 实现自定义 Table Function
  public static class Split extends TableFunction<Tuple2<String, Integer>> {

    // 定义属性，分隔符
    private String separator = ",";

    public Split(String separator) {
      this.separator = separator;
    }

    public void eval(String str) {
      for (String s : str.split(separator)) {
        collect(new Tuple2<>(s, s.length()));
      }
    }
  }
}
```

输出结果

```shell
result> sensor_1,1547718199,sensor,6
result> sensor_1,1547718199,1,1
sql> sensor_1,1547718199,sensor,6
sql> sensor_1,1547718199,1,1
result> sensor_6,1547718201,sensor,6
result> sensor_6,1547718201,6,1
sql> sensor_6,1547718201,sensor,6
sql> sensor_6,1547718201,6,1
result> sensor_7,1547718202,sensor,6
result> sensor_7,1547718202,7,1
sql> sensor_7,1547718202,sensor,6
sql> sensor_7,1547718202,7,1
result> sensor_10,1547718205,sensor,6
result> sensor_10,1547718205,10,2
sql> sensor_10,1547718205,sensor,6
sql> sensor_10,1547718205,10,2
result> sensor_1,1547718207,sensor,6
result> sensor_1,1547718207,1,1
sql> sensor_1,1547718207,sensor,6
sql> sensor_1,1547718207,1,1
result> sensor_1,1547718209,sensor,6
result> sensor_1,1547718209,1,1
sql> sensor_1,1547718209,sensor,6
sql> sensor_1,1547718209,1,1
result> sensor_1,1547718212,sensor,6
result> sensor_1,1547718212,1,1
sql> sensor_1,1547718212,sensor,6
sql> sensor_1,1547718212,1,1
```

### 13.1.3 聚合函数(Aggregate Functions)

**聚合，多对一，类似前面的窗口聚合**

---

+ 用户自定义聚合函数（User-Defined Aggregate Functions，UDAGGs）可以把一个表中的数据，聚合成一个标量值
+ 用户定义的聚合函数，是通过继承 AggregateFunction 抽象类实现的

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200601221643915.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ AggregationFunction要求必须实现的方法
  + `createAccumulator()`
  + `accumulate()`
  + `getValue()`
+ AggregateFunction 的工作原理如下：
  + 首先，它需要一个累加器（Accumulator），用来保存聚合中间结果的数据结构；可以通过调用 `createAccumulator()` 方法创建空累加器
  + 随后，对每个输入行调用函数的 `accumulate()` 方法来更新累加器
  + 处理完所有行后，将调用函数的 `getValue()` 方法来计算并返回最终结果

#### 测试代码

```java
package apitest.tableapi.udf;

import apitest.beans.SensorReading;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.table.functions.AggregateFunction;
import org.apache.flink.types.Row;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2021/2/4 4:24 AM
 */
public class UdfTest3_AggregateFunction {
  public static void main(String[] args) throws Exception {
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    env.setParallelism(1);

    StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

    // 1. 读取数据
    DataStream<String> inputStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");

    // 2. 转换成POJO
    DataStream<SensorReading> dataStream = inputStream.map(line -> {
      String[] fields = line.split(",");
      return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
    });

    // 3. 将流转换成表
    Table sensorTable = tableEnv.fromDataStream(dataStream, "id, timestamp as ts, temperature as temp");

    // 4. 自定义聚合函数，求当前传感器的平均温度值
    // 4.1 table API
    AvgTemp avgTemp = new AvgTemp();

    // 需要在环境中注册UDF
    tableEnv.registerFunction("avgTemp", avgTemp);
    Table resultTable = sensorTable
      .groupBy("id")
      .aggregate("avgTemp(temp) as avgtemp")
      .select("id, avgtemp");

    // 4.2 SQL
    tableEnv.createTemporaryView("sensor", sensorTable);
    Table resultSqlTable = tableEnv.sqlQuery("select id, avgTemp(temp) " +
                                             " from sensor group by id");

    // 打印输出
    tableEnv.toRetractStream(resultTable, Row.class).print("result");
    tableEnv.toRetractStream(resultSqlTable, Row.class).print("sql");

    env.execute();
  }

  // 实现自定义的AggregateFunction
  public static class AvgTemp extends AggregateFunction<Double, Tuple2<Double, Integer>> {
    @Override
    public Double getValue(Tuple2<Double, Integer> accumulator) {
      return accumulator.f0 / accumulator.f1;
    }

    @Override
    public Tuple2<Double, Integer> createAccumulator() {
      return new Tuple2<>(0.0, 0);
    }

    // 必须实现一个accumulate方法，来数据之后更新状态
    // 这里方法名必须是这个，且必须public。
    // 累加器参数，必须得是第一个参数；随后的才是我们自己传的入参
    public void accumulate(Tuple2<Double, Integer> accumulator, Double temp) {
      accumulator.f0 += temp;
      accumulator.f1 += 1;
    }
  }
}
```

输出结果：

```shell
result> (true,sensor_1,35.8)
result> (true,sensor_6,15.4)
result> (true,sensor_7,6.7)
result> (true,sensor_10,38.1)
result> (false,sensor_1,35.8)
result> (true,sensor_1,36.05)
sql> (true,sensor_1,35.8)
result> (false,sensor_1,36.05)
sql> (true,sensor_6,15.4)
result> (true,sensor_1,34.96666666666666)
sql> (true,sensor_7,6.7)
result> (false,sensor_1,34.96666666666666)
sql> (true,sensor_10,38.1)
result> (true,sensor_1,35.5)
sql> (false,sensor_1,35.8)
sql> (true,sensor_1,36.05)
sql> (false,sensor_1,36.05)
sql> (true,sensor_1,34.96666666666666)
sql> (false,sensor_1,34.96666666666666)
sql> (true,sensor_1,35.5)
```

### 13.1.4 表聚合函数

+ 用户定义的表聚合函数（User-Defined Table Aggregate Functions，UDTAGGs），可以把一个表中数据，聚合为具有多行和多列的结果表
+ 用户定义表聚合函数，是通过继承 TableAggregateFunction 抽象类来实现的

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200601223517314.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ AggregationFunction 要求必须实现的方法：
  + `createAccumulator()`
  + `accumulate()`
  + `emitValue()`
+ TableAggregateFunction 的工作原理如下：
  + 首先，它同样需要一个累加器（Accumulator），它是保存聚合中间结果的数据结构。通过调用 `createAccumulator()` 方法可以创建空累加器。
  + 随后，对每个输入行调用函数的 `accumulate()` 方法来更新累加器。
  + 处理完所有行后，将调用函数的 `emitValue()` 方法来计算并返回最终结果。

#### 测试代码

> [Flink-函数 | 用户自定义函数（UDF）标量函数 | 表函数 | 聚合函数 | 表聚合函数](https://blog.csdn.net/qq_40180229/article/details/106482550)

```scala
import com.atguigu.bean.SensorReading
import org.apache.flink.streaming.api.TimeCharacteristic
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor
import org.apache.flink.streaming.api.scala._
import org.apache.flink.streaming.api.windowing.time.Time
import org.apache.flink.table.api.Table
import org.apache.flink.table.api.scala._
import org.apache.flink.table.functions.TableAggregateFunction
import org.apache.flink.types.Row
import org.apache.flink.util.Collector

object TableAggregateFunctionTest {
  def main(args: Array[String]): Unit = {

    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)

    // 开启事件时间语义
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

    // 创建表环境
    val tableEnv: StreamTableEnvironment = StreamTableEnvironment.create(env)

    val inputDStream: DataStream[String] = env.readTextFile("D:\\MyWork\\WorkSpaceIDEA\\flink-tutorial\\src\\main\\resources\\SensorReading.txt")

    val dataDStream: DataStream[SensorReading] = inputDStream.map(
      data => {
        val dataArray: Array[String] = data.split(",")
        SensorReading(dataArray(0), dataArray(1).toLong, dataArray(2).toDouble)
      })
    .assignTimestampsAndWatermarks( new BoundedOutOfOrdernessTimestampExtractor[SensorReading]
                                   ( Time.seconds(1) ) {
                                     override def extractTimestamp(element: SensorReading): Long = element.timestamp * 1000L
                                   } )

    // 用proctime定义处理时间
    val dataTable: Table = tableEnv
    .fromDataStream(dataDStream, 'id, 'temperature, 'timestamp.rowtime as 'ts)

    // 使用自定义的hash函数，求id的哈希值
    val myAggTabTemp = MyAggTabTemp()

    // 查询 Table API 方式
    val resultTable: Table = dataTable
    .groupBy('id)
    .flatAggregate( myAggTabTemp('temperature) as ('temp, 'rank) )
    .select('id, 'temp, 'rank)


    // SQL调用方式，首先要注册表
    tableEnv.createTemporaryView("dataTable", dataTable)
    // 注册函数
    tableEnv.registerFunction("myAggTabTemp", myAggTabTemp)

    /*
    val resultSqlTable: Table = tableEnv.sqlQuery(
      """
        |select id, temp, `rank`
        |from dataTable, lateral table(myAggTabTemp(temperature)) as aggtab(temp, `rank`)
        |group by id
        |""".stripMargin)
*/


    // 测试输出
    resultTable.toRetractStream[ Row ].print( "scalar" )
    //resultSqlTable.toAppendStream[ Row ].print( "scalar_sql" )
    // 查看表结构
    dataTable.printSchema()

    env.execute(" table ProcessingTime test job")
  }
}

// 自定义状态类
case class AggTabTempAcc() {
  var highestTemp: Double = Double.MinValue
  var secondHighestTemp: Double = Double.MinValue
}

case class MyAggTabTemp() extends TableAggregateFunction[(Double, Int), AggTabTempAcc]{
  // 初始化状态
  override def createAccumulator(): AggTabTempAcc = new AggTabTempAcc()

  // 每来一个数据后，聚合计算的操作
  def accumulate( acc: AggTabTempAcc, temp: Double ): Unit ={
    // 将当前温度值，跟状态中的最高温和第二高温比较，如果大的话就替换
    if( temp > acc.highestTemp ){
      // 如果比最高温还高，就排第一，其它温度依次后移
      acc.secondHighestTemp = acc.highestTemp
      acc.highestTemp = temp
    } else if( temp > acc.secondHighestTemp ){
      acc.secondHighestTemp = temp
    }
  }

  // 实现一个输出数据的方法，写入结果表中
  def emitValue( acc: AggTabTempAcc, out: Collector[(Double, Int)] ): Unit ={
    out.collect((acc.highestTemp, 1))
    out.collect((acc.secondHighestTemp, 2))
  }
}
```

# 14. 基于flink的电商用户行为数据分析

> [Flink电商项目第一天-电商用户行为分析及完整图步骤解析-热门商品统计TopN的实现](https://blog.csdn.net/qq_40180229/article/details/106502286)

+ 批处理和流处理
+ 电商用户行为分析
+ 数据源解析
+ 项目模块划分

## 14.1 批处理和流处理

### 批处理

批处理主要操作大容量静态数据集，并在计算过程完成后返回结果。

可以认为，处理的是用一个固定时间间隔分组的数据点集合。

批处理模式中使用的数据集通常符合下列特征：

+ **有界：批处理数据集代表数据的有限集合**
+ **持久：数据通常始终存储在某种类型的持久存储位置中**
+ **大量：批处理操作通常是处理极为海量数据集的唯一方法**

### 流处理

流处理可以对随时进入系统的数据进行计算。

流处理方式无需针对整个数据集执行操作，而是对通过系统传输的每个数据项执行操作。

流处理中的数据集是“无边界”的，这就产生了几个重要的影响：

+ 可以处理几乎无限量的数据，但**同一时间只能处理一条数据，不同记录间只维持最少量的状态**

+ 处理工作是基于事件的，除非明确停止否则没有“尽头”

+ 处理结果立刻可用，并会随着新数据的抵达继续更新。

## 14.2 电商用户行为分析

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200602182834724.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ 统计分析
  + 点击、浏览
  + 热门商品、近期热门商品、分类热门商品，流量统计
+ 偏好统计
  + 收藏、喜欢、评分、打标签
  + 用户画像，推荐列表（结合特征工程和机器学习算法）
+ 风险控制
  + 下订单、支付、登录
  + 刷单监控，订单失效监控，恶意登录（短时间内频繁登录失败）监控

### 项目模块设计

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200602183018421.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200602183121950.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

### 数据源

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200602183227549.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

### 数据源-数据结构

**UserBehavior**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200602183249723.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

**ApacheLogEvent**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200602183323920.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

## 14.3 项目模块

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200602183434342.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

### 14.3.1 热门实时商品统计

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200602183513841.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200602183537531.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ 按照商品id进行分区

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200602183613384.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ 设置窗口时间

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200602183637249.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ 时间窗口（timeWindow）区间为左闭右开
+ 同一份数据会被分发到不同的窗口

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200602183733756.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ 窗口聚合

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200602183749122.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ 窗口聚合策略——没出现一条记录就加一

+ 实现AggregateFunction接口

  `interface AggregateFunction<IN, ACC, OUT>`

+ 定义输出结构——`ItemViewCount(itemId,windowEnd,count)`

+ 实现WindowFunction接口

  + `interface WindowFunction<IN,OUT,KEY,W extends Window>`

    + IN：输入为累加器的类型，Long
    + OUT：窗口累加以后输出的类型为`ItemViewCount(itemId: Long,windowEnd: Long,count: Long)`，windowEnd为窗口的结束时间，也是窗口的唯一标识
    + KEY：Tuple泛型，在这里是itemId，窗口根据itemId聚合
    + W：聚合的窗口，`w.getEnd`就能拿到窗口的结束时间

    ```java
    public void apply(Tuple tuple, TimeWindow window,
                     Iterable<Long> input, Collector<ItemViewCount> out)throws Exception {
      Long itemId = tuple.getField(0);
      Long windowEnd - window.getEnd();
      Long count = input.iterator().next();
      out.collect(new ItemViewCount(itemId, windowEnd, count));
    }
    ```

+ 窗口聚合示例

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200602183856808.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+  **进行统计整理 —— keyBy(“windowEnd”)**

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020060218391775.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ 状态编程

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200602183935754.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ **最终排序输出——keyedProcessFunction**

  + 针对有状态流的底层API
  + KeyedProcessFunction会对分区后的每一条子流进行处理
  + 以windowEnd作为key，保证分流以后每一条流的数据都在一个时间窗口内
  + 从ListState中读取当前流的状态，存储数据进行排序输出

  ---

  + 用ProcessFunction定义KeyedStream的处理逻辑
  + 分区之后，每个KeyedStream都有其自己的生命周期
    + open：初始化，在这里可以获取当前流的状态
    + processElement：处理流中每一个元素时调用
    + onTimer：定时调用，注册定时器Timer并触发之后的回调操作

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200602184003237.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

#### POJO

需要生成get/set、无参/有参构造函数、toString

+ ItemViewCount

  ```java
  private Long itemId;
  private Long windowEnd;
  private Long count;
  ```

+ UserBehavior

  ```java
  private Long uerId;
  private Long itemId;
  private Integer categoryId;
  private String behavior;
  private Long timestamp;
  ```

#### 代码1-文件

+ 父pom依赖

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <project xmlns="http://maven.apache.org/POM/4.0.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
  
    <groupId>org.example</groupId>
    <artifactId>UserBehaviorAnalysis</artifactId>
    <packaging>pom</packaging>
    <version>1.0-SNAPSHOT</version>
    <modules>
      <module>HotItemsAnalysis</module>
    </modules>
  
    <properties>
      <maven.compiler.source>8</maven.compiler.source>
      <maven.compiler.target>8</maven.compiler.target>
      <flink.version>1.12.1</flink.version>
      <scala.binary.version>2.12</scala.binary.version>
      <kafka.version>2.7.0</kafka.version>
      <mysql.version>8.0.19</mysql.version>
    </properties>
  
    <dependencies>
  
      <!-- flink -->
      <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-java</artifactId>
        <version>${flink.version}</version>
      </dependency>
      <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-streaming-java_${scala.binary.version}</artifactId>
        <version>${flink.version}</version>
      </dependency>
      <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-clients_${scala.binary.version}</artifactId>
        <version>${flink.version}</version>
      </dependency>
  
      <!-- kafka -->
      <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka_${scala.binary.version}</artifactId>
        <version>${kafka.version}</version>
      </dependency>
      <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-connector-kafka_${scala.binary.version}</artifactId>
        <version>${flink.version}</version>
      </dependency>
    </dependencies>
  
    <build>
      <plugins>
        <plugin>
          <artifactId>maven-compiler-plugin</artifactId>
          <configuration>
            <source>1.8</source>
            <target>1.8</target>
            <encoding>UTF-8</encoding>
          </configuration>
        </plugin>
      </plugins>
    </build>
  
  </project>
  ```

+ java代码

  ```java
  import beans.ItemViewCount;
  import beans.UserBehavior;
  import org.apache.commons.compress.utils.Lists;
  import org.apache.flink.api.common.functions.AggregateFunction;
  import org.apache.flink.api.common.state.ListState;
  import org.apache.flink.api.common.state.ListStateDescriptor;
  import org.apache.flink.configuration.Configuration;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
  import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
  import org.apache.flink.streaming.api.functions.windowing.WindowFunction;
  import org.apache.flink.streaming.api.windowing.assigners.SlidingEventTimeWindows;
  import org.apache.flink.streaming.api.windowing.assigners.SlidingProcessingTimeWindows;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
  import org.apache.flink.streaming.runtime.operators.util.AssignerWithPeriodicWatermarksAdapter;
  import org.apache.flink.util.Collector;
  
  import java.sql.Timestamp;
  import java.util.ArrayList;
  import java.util.concurrent.TimeUnit;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/4 6:07 PM
   */
  public class HotItems {
    public static void main(String[] args) throws Exception {
      // 1. 创建执行环境
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      // 设置并行度为1
      env.setParallelism(1);
  
      // 2. 从csv文件中获取数据
      DataStream<String> inputStream = env.readTextFile("/tmp/UserBehaviorAnalysis/HotItemsAnalysis/src/main/resources/UserBehavior.csv");
  
      // 3. 转换成POJO,分配时间戳和watermark
      DataStream<UserBehavior> userBehaviorDataStream = inputStream.map(line -> {
        String[] fields = line.split(",");
        return new UserBehavior(new Long(fields[0]), new Long(fields[1]), new Integer(fields[2]), fields[3], new Long(fields[4]));
      }).assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
        new BoundedOutOfOrdernessTimestampExtractor<UserBehavior>(Time.of(200, TimeUnit.MILLISECONDS)) {
          @Override
          public long extractTimestamp(UserBehavior element) {
            return element.getTimestamp() * 1000L;
          }
        }
      ));
  
      // 4. 分组开窗聚合，得到每个窗口内各个商品的count值
      //        DataStream<ItemViewCount> windowAggStream = userBehaviorDataStream
      DataStream<ItemViewCount> windowAggStream = userBehaviorDataStream
        // 过滤只保留pv行为
        .filter(userBehavior -> "pv".equals(userBehavior.getBehavior()))
        // 按照商品ID分组
        .keyBy(UserBehavior::getItemId)
        // 滑动窗口
        .window(SlidingEventTimeWindows.of(Time.hours(1), Time.minutes(5)))
        .aggregate(new ItemCountAgg(), new WindowItemCountResult());
  
      // 5. 收集同一窗口的所有商品的count数据，排序输出top n
      DataStream<String> resultStream = windowAggStream
        // 按照窗口分组
        .keyBy(ItemViewCount::getWindowEnd)
        // 用自定义处理函数排序取前5
        .process(new TopNHotItems(5));
  
      resultStream.print();
  
      env.execute("hot items analysis");
    }
  
    // 实现自定义增量聚合函数
    public static class ItemCountAgg implements AggregateFunction<UserBehavior, Long, Long> {
  
      @Override
      public Long createAccumulator() {
        return 0L;
      }
  
      @Override
      public Long add(UserBehavior value, Long accumulator) {
        return accumulator + 1;
      }
  
      @Override
      public Long getResult(Long accumulator) {
        return accumulator;
      }
  
      @Override
      public Long merge(Long a, Long b) {
        return a + b;
      }
    }
  
    // 自定义全窗口函数
    public static class WindowItemCountResult implements WindowFunction<Long, ItemViewCount, Long, TimeWindow> {
  
      @Override
      public void apply(Long itemId, TimeWindow window, Iterable<Long> input, Collector<ItemViewCount> out) throws Exception {
        Long windowEnd = window.getEnd();
        Long count = input.iterator().next();
        out.collect(new ItemViewCount(itemId, windowEnd, count));
      }
    }
  
    // 实现自定义KeyedProcessFunction
    public static class TopNHotItems extends KeyedProcessFunction<Long, ItemViewCount, String> {
  
      // 定义属性， TopN的大小
      private Integer topSize;
  
      // 定义状态列表，保存当前窗口内所有输出的ItemViewCount
      ListState<ItemViewCount> itemViewCountListState;
  
      @Override
      public void open(Configuration parameters) throws Exception {
        itemViewCountListState = getRuntimeContext().getListState(
          new ListStateDescriptor<ItemViewCount>("item-view-count-list", ItemViewCount.class));
      }
  
      //        @Override
      //        public void close() throws Exception {
      //            itemViewCountListState.clear();
      //        }
  
      public TopNHotItems(Integer topSize) {
        this.topSize = topSize;
      }
  
      @Override
      public void processElement(ItemViewCount value, Context ctx, Collector<String> out) throws Exception {
        // 每来一条数据，存入List中，并注册定时器
        itemViewCountListState.add(value);
        // 模拟等待，所以这里时间设的比较短(1ms)
        ctx.timerService().registerEventTimeTimer(value.getWindowEnd() + 1);
      }
  
      @Override
      public void onTimer(long timestamp, OnTimerContext ctx, Collector<String> out) throws Exception {
        // 定时器触发，当前已收集到所有数据，排序输出
        ArrayList<ItemViewCount> itemViewCounts = Lists.newArrayList(itemViewCountListState.get().iterator());
        // 从多到少(越热门越前面)
        itemViewCounts.sort((a, b) -> -Long.compare(a.getCount(), b.getCount()));
        StringBuilder resultBuilder = new StringBuilder();
        resultBuilder.append("============================").append(System.lineSeparator());
        resultBuilder.append("窗口结束时间：").append(new Timestamp(timestamp - 1)).append(System.lineSeparator());
  
        // 遍历列表，取top n输出
        for (int i = 0; i < Math.min(topSize, itemViewCounts.size()); i++) {
          ItemViewCount currentItemViewCount = itemViewCounts.get(i);
          resultBuilder.append("NO ").append(i + 1).append(":")
            .append(" 商品ID = ").append(currentItemViewCount.getItemId())
            .append(" 热门度 = ").append(currentItemViewCount.getCount())
            .append(System.lineSeparator());
        }
        resultBuilder.append("===============================").append(System.lineSeparator());
  
        // 控制输出频率
        Thread.sleep(1000L);
  
        out.collect(resultBuilder.toString());
      }
    }
  }
  ```

+ 输入文件如下：

  ```shell
  543462,1715,1464116,pv,1511658000
  662867,2244074,1575622,pv,1511658000
  561558,3611281,965809,pv,1511658000
  894923,3076029,1879194,pv,1511658000
  834377,4541270,3738615,pv,1511658000
  ...
  ```

+ 输出如下：

  ```shell
  ============================
  窗口结束时间：2017-11-26 10:10:00.0
  NO 1: 商品ID = 2338453 热门度 = 30
  NO 2: 商品ID = 812879 热门度 = 18
  NO 3: 商品ID = 2563440 热门度 = 14
  NO 4: 商品ID = 138964 热门度 = 12
  NO 5: 商品ID = 3244134 热门度 = 12
  ===============================
  
  ============================
  窗口结束时间：2017-11-26 10:15:00.0
  NO 1: 商品ID = 2338453 热门度 = 33
  NO 2: 商品ID = 812879 热门度 = 18
  NO 3: 商品ID = 3244134 热门度 = 13
  NO 4: 商品ID = 2563440 热门度 = 13
  NO 5: 商品ID = 2364679 热门度 = 13
  ===============================
  
  ============================
  窗口结束时间：2017-11-26 10:20:00.0
  NO 1: 商品ID = 2338453 热门度 = 32
  NO 2: 商品ID = 812879 热门度 = 18
  NO 3: 商品ID = 3244134 热门度 = 15
  NO 4: 商品ID = 4649427 热门度 = 13
  NO 5: 商品ID = 2364679 热门度 = 12
  ===============================
  
  ...
  ```

#### 代码2-kafka

+ java代码

  ```java
  // 仅修改 获取数据源的部分
  
  // 2. 从csv文件中获取数据
  //        DataStream<String> inputStream = env.readTextFile("/tmp/UserBehaviorAnalysis/HotItemsAnalysis/src/main/resources/UserBehavior.csv");
  
  Properties properties = new Properties();
  properties.setProperty("bootstrap.servers", "localhost:9092");
  properties.setProperty("group.id", "consumer");
  // 下面是一些次要参数
  properties.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
  properties.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
  properties.setProperty("auto.offset.reset", "latest");
  
  // 2. 从kafka消费数据
  DataStream<String> inputStream = env.addSource(new FlinkKafkaConsumer<>("hotitems", new SimpleStringSchema(), properties ));
  
  
  ```

+ 启动本地kafka里自带的zookeeper

  ```shell
  $ bin/zookeeper-server-start.sh config/zookeeper.properties
  ```

+ 启动kafka

  ```shell
  $ bin/kafka-server-start.sh config/server.properties
  ```

+ 启动kafka生产者console

  ```shell
  $ bin/kafka-console-producer.sh --broker-list localhost:9092  --topic hotitems
  ```

+ 运行Flink程序，输入数据（kafka-console-producer）

  ```shell
  $ bin/kafka-console-producer.sh --broker-list localhost:9092  --topic hotitems
  >543462,1715,1464116,pv,1511658000
  >662867,2244074,1575622,pv,1511658060
  >561558,3611281,965809,pv,1511658120
  >894923,1715,1879194,pv,1511658180
  >834377,2244074,3738615,pv,1511658240
  >625915,3611281,570735,pv,1511658300
  >625915,3611281,570735,pv,1511658301
  ```

+ 输出

  ```shell
  ============================
  窗口结束时间：2017-11-26 09:05:00.0
  NO 1: 商品ID = 1715 热门度 = 2
  NO 2: 商品ID = 2244074 热门度 = 2
  NO 3: 商品ID = 3611281 热门度 = 1
  ===============================
  ```

#### 代码3-kafka批量数据测试

+ java代码

  ```java
  import org.apache.kafka.clients.producer.KafkaProducer;
  import org.apache.kafka.clients.producer.ProducerRecord;
  
  import java.io.BufferedReader;
  import java.io.BufferedWriter;
  import java.io.FileReader;
  import java.util.Properties;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/4 11:53 PM
   */
  public class KafkaProducerUtil {
    public static void main(String[] args) throws Exception {
      writeToKafka("hotitems");
    }
  
    public static void writeToKafka(String topic)throws Exception{
      // Kafka配置
      Properties properties = new Properties();
      properties.setProperty("bootstrap.servers", "localhost:9092");
      properties.setProperty("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
      properties.setProperty("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
  
      // 定义一个Kafka Producer
      KafkaProducer<String, String> kafkaProducer = new KafkaProducer<String, String>(properties);
  
      // 用缓冲方式来读取文本
      BufferedReader bufferedReader = new BufferedReader(new FileReader("/tmp/UserBehaviorAnalysis/HotItemsAnalysis/src/main/resources/UserBehavior.csv"));
      String line;
      while((line = bufferedReader.readLine())!=null){
        ProducerRecord<String,String> producerRecord = new ProducerRecord<>(topic,line );
        // 用producer发送数据
        kafkaProducer.send(producerRecord);
      }
      kafkaProducer.close();
    }
  }
  
  ```

+ 启动zookeeper

+ 启动kafka服务

+ 运行该java程序，之后就可以直接启动HotItems程序，读取本地已有的kafka数据了

#### 代码4-Flink-SQL实现

+ java代码

  **下面用最新的Expression写法实现<=新版本推荐的写法**

  ```java
  import beans.UserBehavior;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.streaming.runtime.operators.util.AssignerWithPeriodicWatermarksAdapter;
  import org.apache.flink.table.api.EnvironmentSettings;
  import org.apache.flink.table.api.Slide;
  import org.apache.flink.table.api.Table;
  import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
  import org.apache.flink.types.Row;
  
  import java.util.concurrent.TimeUnit;
  
  import static org.apache.flink.table.api.Expressions.$;
  import static org.apache.flink.table.api.Expressions.lit;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/5 12:18 AM
   */
  public class HotItemsWithSql {
    public static void main(String[] args) throws Exception {
      // 1. 创建执行环境
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
  
      // 2. 从csv文件中获取数据
      DataStream<String> inputStream = env.readTextFile("/tmp/UserBehaviorAnalysis/HotItemsAnalysis/src/main/resources/UserBehavior.csv");
  
      // 3. 转换成POJO,分配时间戳和watermark
      DataStream<UserBehavior> userBehaviorDataStream = inputStream.map(line -> {
        String[] fields = line.split(",");
        return new UserBehavior(new Long(fields[0]), new Long(fields[1]), new Integer(fields[2]), fields[3], new Long(fields[4]));
      }).assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
        new BoundedOutOfOrdernessTimestampExtractor<UserBehavior>(Time.of(200, TimeUnit.MILLISECONDS)) {
          @Override
          public long extractTimestamp(UserBehavior element) {
            return element.getTimestamp() * 1000L;
          }
        }
      ));
  
      // 4. 创建表执行环境,使用blink版本
      EnvironmentSettings settings = EnvironmentSettings.newInstance()
        .useBlinkPlanner()
        .inStreamingMode()
        .build();
      StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env, settings);
  
      // 5. 将流转换成表
      Table dataTable = tableEnv.fromDataStream(userBehaviorDataStream,
                                                $("itemId"),
                                                $("behavior"),
                                                $("timestamp").rowtime().as("ts")
                                               );
  
      // 6. 分组开窗
      // Table API
      Table windowAggTable = dataTable
        .filter($("behavior").isEqual("pv"))
        .window(Slide.over(lit(1).hours()).every(lit(5).minutes()).on($("ts")).as("w"))
        .groupBy($("itemId"), $("w"))
        .select($("itemId"), $("w").end().as("windowEnd"), $("itemId").count().as("cnt"));
  
      // 7. 利用开创函数，对count值进行排序，并获取Row number，得到Top N
      // SQL
      DataStream<Row> aggStream = tableEnv.toAppendStream(windowAggTable, Row.class);
      tableEnv.createTemporaryView("agg", aggStream, $("itemId"), $("windowEnd"), $("cnt"));
  
      Table resultTable = tableEnv.sqlQuery("select * from " +
                                            "  ( select *, ROW_NUMBER() over (partition by windowEnd order by cnt desc) as row_num " +
                                            "  from agg) " +
                                            " where row_num <= 5 ");
  
      // 纯SQL实现
      tableEnv.createTemporaryView("data_table", userBehaviorDataStream, $("itemId"), $("behavior"), $("timestamp").rowtime().as("ts"));
      Table resultSqlTable = tableEnv.sqlQuery("select * from " +
                                               "  ( select *, ROW_NUMBER() over (partition by windowEnd order by cnt desc) as row_num " +
                                               "  from ( " +
                                               "    select itemId, count(itemId) as cnt, HOP_END(ts, interval '5' minute, interval '1' hour) as windowEnd " +
                                               "    from data_table " +
                                               "    where behavior = 'pv' " +
                                               "    group by itemId, HOP(ts, interval '5' minute, interval '1' hour)" +
                                               "    )" +
                                               "  ) " +
                                               " where row_num <= 5 ");
  
   //   tableEnv.toRetractStream(resultTable, Row.class).print();
      tableEnv.toRetractStream(resultSqlTable, Row.class).print();
  
      env.execute("hot items with sql job");
    }
  }
  ```

+ 输出

  ```shell
  ....
  (true,2288408,2017-11-26T03:00,15,4)
  (false,279675,2017-11-26T03:00,14,5)
  (true,291932,2017-11-26T03:00,15,5)
  (false,3715112,6,2017-11-26T03:05,1)
  (true,3244931,7,2017-11-26T03:05,1)
  (false,710777,6,2017-11-26T03:05,2)
  (true,3715112,6,2017-11-26T03:05,2)
  (false,724262,6,2017-11-26T03:05,3)
  (true,710777,6,2017-11-26T03:05,3)
  (false,1303734,5,2017-11-26T03:05,4)
  (true,724262,6,2017-11-26T03:05,4)
  (false,4622270,5,2017-11-26T03:05,5)
  (true,1303734,5,2017-11-26T03:05,5)
  (false,1303734,5,2017-11-26T03:05,5)
  ....
  ```

### 14.3.2 实时流量统计——热门页面

+ 基本需求
  + 从web服务器的日志中，统计实时的热门访问页面
  + 统计每分钟的ip访问量，取出访问量最大的5个地址，每5秒更新一次
+ 解决思路1
  + 将apache服务器日志中的时间，转换为时间戳，作为Event Time
  + 构建滑动窗口，窗口长度为1分钟，滑动距离为5秒

#### POJO

+ ApacheLogEvent

  ```java
  private String ip;
  private String userId;
  private Long timestamp;
  private String method;
  private String url;
  ```

+ PageViewCount

  ```java
  private String url;
  private Long windowEnd;
  private Long count;
  ```

#### 代码1-文件

+ Java代码

  ```java
  import beans.ApacheLogEvent;
  import beans.PageViewCount;
  import org.apache.commons.compress.utils.Lists;
  import org.apache.flink.api.common.functions.AggregateFunction;
  import org.apache.flink.api.common.state.ListState;
  import org.apache.flink.api.common.state.ListStateDescriptor;
  import org.apache.flink.configuration.Configuration;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
  import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
  import org.apache.flink.streaming.api.functions.windowing.WindowFunction;
  import org.apache.flink.streaming.api.windowing.assigners.SlidingEventTimeWindows;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
  import org.apache.flink.streaming.runtime.operators.util.AssignerWithPeriodicWatermarksAdapter;
  import org.apache.flink.util.Collector;
  
  import java.net.URL;
  import java.sql.Timestamp;
  import java.text.SimpleDateFormat;
  import java.util.ArrayList;
  import java.util.concurrent.TimeUnit;
  import java.util.regex.Pattern;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/5 1:27 AM
   */
  public class HotPages {
    public static void main(String[] args) throws Exception {
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
  
      // 读取文件，转换成POJO
      URL resource = HotPages.class.getResource("/apache.log");
      DataStream<String> inputStream = env.readTextFile(resource.getPath());
  
      DataStream<ApacheLogEvent> dataStream = inputStream
        .map(line -> {
          String[] fields = line.split(" ");
          SimpleDateFormat simpleDateFormat = new SimpleDateFormat("dd/MM/yyyy:HH:mm:ss");
          Long timestamp = simpleDateFormat.parse(fields[3]).getTime();
          return new ApacheLogEvent(fields[0], fields[1], timestamp, fields[5], fields[6]);
        })
        .assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
          new BoundedOutOfOrdernessTimestampExtractor<ApacheLogEvent>(Time.of(1, TimeUnit.SECONDS)) {
            @Override
            public long extractTimestamp(ApacheLogEvent element) {
              return element.getTimestamp();
            }
          }
        ));
  
      // 分组开窗聚合
      SingleOutputStreamOperator<PageViewCount> windowAggStream = dataStream
        // 过滤get请求
        .filter(data -> "GET".equals(data.getMethod()))
        .filter(data -> {
          String regex = "^((?!\\.(css|js|png|ico)$).)*$";
          return Pattern.matches(regex, data.getUrl());
        })
        // 按照url分组
        .keyBy(ApacheLogEvent::getUrl)
        .window(SlidingEventTimeWindows.of(Time.minutes(10), Time.seconds(5)))
        .aggregate(new PageCountAgg(), new PageCountResult());
  
  
      // 收集同一窗口count数据，排序输出
      DataStream<String> resultStream = windowAggStream
        .keyBy(PageViewCount::getWindowEnd)
        .process(new TopNHotPages(3));
  
      resultStream.print();
  
      env.execute("hot pages job");
    }
  
    // 自定义预聚合函数
    public static class PageCountAgg implements AggregateFunction<ApacheLogEvent, Long, Long> {
      @Override
      public Long createAccumulator() {
        return 0L;
      }
  
      @Override
      public Long add(ApacheLogEvent value, Long accumulator) {
        return accumulator + 1;
      }
  
      @Override
      public Long getResult(Long accumulator) {
        return accumulator;
      }
  
      @Override
      public Long merge(Long a, Long b) {
        return a + b;
      }
    }
  
    // 实现自定义的窗口函数
    public static class PageCountResult implements WindowFunction<Long, PageViewCount, String, TimeWindow> {
      @Override
      public void apply(String url, TimeWindow window, Iterable<Long> input, Collector<PageViewCount> out) throws Exception {
        out.collect(new PageViewCount(url, window.getEnd(), input.iterator().next()));
      }
    }
  
    // 实现自定义的处理函数
    public static class TopNHotPages extends KeyedProcessFunction<Long, PageViewCount, String> {
      private Integer topSize;
  
      public TopNHotPages(Integer topSize) {
        this.topSize = topSize;
      }
  
      // 定义状态，保存当前所有PageViewCount到list中
      ListState<PageViewCount> pageViewCountListState;
  
      @Override
      public void open(Configuration parameters) throws Exception {
        pageViewCountListState = getRuntimeContext().getListState(new ListStateDescriptor<PageViewCount>("page-count-list", PageViewCount.class));
      }
  
      @Override
      public void processElement(PageViewCount value, Context ctx, Collector<String> out) throws Exception {
        pageViewCountListState.add(value);
        ctx.timerService().registerEventTimeTimer(value.getWindowEnd() + 1);
      }
  
      @Override
      public void onTimer(long timestamp, OnTimerContext ctx, Collector<String> out) throws Exception {
        ArrayList<PageViewCount> pageViewCounts = Lists.newArrayList(pageViewCountListState.get().iterator());
  
        pageViewCounts.sort((a, b) -> -Long.compare(a.getCount(), b.getCount()));
  
        // 格式化成String输出
        StringBuilder resultBuilder = new StringBuilder();
        resultBuilder.append("============================").append(System.lineSeparator());
        resultBuilder.append("窗口结束时间：").append(new Timestamp(timestamp - 1)).append(System.lineSeparator());
  
        // 遍历列表，取top n输出
        for (int i = 0; i < Math.min(topSize, pageViewCounts.size()); i++) {
          PageViewCount pageViewCount = pageViewCounts.get(i);
          resultBuilder.append("NO ").append(i + 1).append(":")
            .append(" 页面URL = ").append(pageViewCount.getUrl())
            .append(" 浏览量 = ").append(pageViewCount.getCount())
            .append(System.lineSeparator());
        }
        resultBuilder.append("===============================").append(System.lineSeparator());
  
        // 控制输出频率
        Thread.sleep(1000L);
  
        out.collect(resultBuilder.toString());
      }
    }
  }
  ```

+ 输出结果

  ```shell
  ....
  
  ============================
  窗口结束时间：2015-05-17 10:05:25.0
  NO 1: 页面URL = /blog/tags/puppet?flav=rss20 浏览量 = 2
  NO 2: 页面URL = /blog/geekery/eventdb-ideas.html 浏览量 = 1
  NO 3: 页面URL = /blog/geekery/installing-windows-8-consumer-preview.html 浏览量 = 1
  ===============================
  
  ============================
  窗口结束时间：2015-05-17 10:05:30.0
  NO 1: 页面URL = /blog/tags/puppet?flav=rss20 浏览量 = 2
  NO 2: 页面URL = /blog/geekery/eventdb-ideas.html 浏览量 = 1
  NO 3: 页面URL = /blog/geekery/installing-windows-8-consumer-preview.html 浏览量 = 1
  ===============================
  
  ============================
  窗口结束时间：2015-05-17 10:05:35.0
  NO 1: 页面URL = /blog/tags/puppet?flav=rss20 浏览量 = 2
  NO 2: 页面URL = /blog/tags/firefox?flav=rss20 浏览量 = 2
  NO 3: 页面URL = /blog/geekery/eventdb-ideas.html 浏览量 = 1
  ===============================
  
  ....
  ```

#### 代码2-乱序数据测试

+ java代码

  ```java
  import beans.ApacheLogEvent;
  import beans.PageViewCount;
  import org.apache.commons.compress.utils.Lists;
  import org.apache.flink.api.common.functions.AggregateFunction;
  import org.apache.flink.api.common.state.ListState;
  import org.apache.flink.api.common.state.ListStateDescriptor;
  import org.apache.flink.configuration.Configuration;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
  import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
  import org.apache.flink.streaming.api.functions.windowing.WindowFunction;
  import org.apache.flink.streaming.api.windowing.assigners.SlidingEventTimeWindows;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
  import org.apache.flink.streaming.runtime.operators.util.AssignerWithPeriodicWatermarksAdapter;
  import org.apache.flink.util.Collector;
  import org.apache.flink.util.OutputTag;
  
  import java.net.URL;
  import java.sql.Timestamp;
  import java.text.SimpleDateFormat;
  import java.util.ArrayList;
  import java.util.concurrent.TimeUnit;
  import java.util.regex.Pattern;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/5 1:27 AM
   */
  public class HotPages {
    public static void main(String[] args) throws Exception {
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
  
      // 读取文件，转换成POJO
      //        URL resource = HotPages.class.getResource("/apache.log");
      //        DataStream<String> inputStream = env.readTextFile(resource.getPath());
  
      // 方便测试，使用本地Socket输入数据
      DataStream<String> inputStream = env.socketTextStream("localhost", 7777);
  
  
      DataStream<ApacheLogEvent> dataStream = inputStream
        .map(line -> {
          String[] fields = line.split(" ");
          SimpleDateFormat simpleDateFormat = new SimpleDateFormat("dd/MM/yyyy:HH:mm:ss");
          Long timestamp = simpleDateFormat.parse(fields[3]).getTime();
          return new ApacheLogEvent(fields[0], fields[1], timestamp, fields[5], fields[6]);
        })
        .assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
          new BoundedOutOfOrdernessTimestampExtractor<ApacheLogEvent>(Time.of(1, TimeUnit.SECONDS)) {
            @Override
            public long extractTimestamp(ApacheLogEvent element) {
              return element.getTimestamp();
            }
          }
        ));
  
      dataStream.print("data");
  
  
      // 定义一个侧输出流标签
      OutputTag<ApacheLogEvent> lateTag = new OutputTag<ApacheLogEvent>("late"){};
  
      // 分组开窗聚合
      SingleOutputStreamOperator<PageViewCount> windowAggStream = dataStream
        // 过滤get请求
        .filter(data -> "GET".equals(data.getMethod()))
        .filter(data -> {
          String regex = "^((?!\\.(css|js|png|ico)$).)*$";
          return Pattern.matches(regex, data.getUrl());
        })
        // 按照url分组
        .keyBy(ApacheLogEvent::getUrl)
        .window(SlidingEventTimeWindows.of(Time.minutes(10), Time.seconds(5)))
        .allowedLateness(Time.minutes(1))
        .sideOutputLateData(lateTag)
        .aggregate(new PageCountAgg(), new PageCountResult());
  
  
      windowAggStream.print("agg");
      windowAggStream.getSideOutput(lateTag).print("late");
  
  
      // 收集同一窗口count数据，排序输出
      DataStream<String> resultStream = windowAggStream
        .keyBy(PageViewCount::getWindowEnd)
        .process(new TopNHotPages(3));
  
      resultStream.print();
  
      env.execute("hot pages job");
    }
  
    // 自定义预聚合函数
    public static class PageCountAgg implements AggregateFunction<ApacheLogEvent, Long, Long> {
      @Override
      public Long createAccumulator() {
        return 0L;
      }
  
      @Override
      public Long add(ApacheLogEvent value, Long accumulator) {
        return accumulator + 1;
      }
  
      @Override
      public Long getResult(Long accumulator) {
        return accumulator;
      }
  
      @Override
      public Long merge(Long a, Long b) {
        return a + b;
      }
    }
  
    // 实现自定义的窗口函数
    public static class PageCountResult implements WindowFunction<Long, PageViewCount, String, TimeWindow> {
      @Override
      public void apply(String url, TimeWindow window, Iterable<Long> input, Collector<PageViewCount> out) throws Exception {
        out.collect(new PageViewCount(url, window.getEnd(), input.iterator().next()));
      }
    }
  
    // 实现自定义的处理函数
    public static class TopNHotPages extends KeyedProcessFunction<Long, PageViewCount, String> {
      private Integer topSize;
  
      public TopNHotPages(Integer topSize) {
        this.topSize = topSize;
      }
  
      // 定义状态，保存当前所有PageViewCount到list中
      ListState<PageViewCount> pageViewCountListState;
  
      @Override
      public void open(Configuration parameters) throws Exception {
        pageViewCountListState = getRuntimeContext().getListState(new ListStateDescriptor<PageViewCount>("page-count-list", PageViewCount.class));
      }
  
      @Override
      public void processElement(PageViewCount value, Context ctx, Collector<String> out) throws Exception {
        pageViewCountListState.add(value);
        ctx.timerService().registerEventTimeTimer(value.getWindowEnd() + 1);
      }
  
      @Override
      public void onTimer(long timestamp, OnTimerContext ctx, Collector<String> out) throws Exception {
        ArrayList<PageViewCount> pageViewCounts = Lists.newArrayList(pageViewCountListState.get().iterator());
  
        pageViewCounts.sort((a, b) -> -Long.compare(a.getCount(), b.getCount()));
  
        // 格式化成String输出
        StringBuilder resultBuilder = new StringBuilder();
        resultBuilder.append("============================").append(System.lineSeparator());
        resultBuilder.append("窗口结束时间：").append(new Timestamp(timestamp - 1)).append(System.lineSeparator());
  
        // 遍历列表，取top n输出
        for (int i = 0; i < Math.min(topSize, pageViewCounts.size()); i++) {
          PageViewCount pageViewCount = pageViewCounts.get(i);
          resultBuilder.append("NO ").append(i + 1).append(":")
            .append(" 页面URL = ").append(pageViewCount.getUrl())
            .append(" 浏览量 = ").append(pageViewCount.getCount())
            .append(System.lineSeparator());
        }
        resultBuilder.append("===============================").append(System.lineSeparator());
  
        // 控制输出频率
        Thread.sleep(1000L);
  
        out.collect(resultBuilder.toString());
  
        pageViewCountListState.clear();
      }
    }
  }
  ```

+ 启动本地socket

  ```shell
  nc -lk 7777
  ```

+ 在本地socket输入数据，查看输出

  + 输入

    ```shell
    83.149.9.216 - - 17/05/2015:10:25:49 +0000 GET /presentations/
    83.149.9.216 - - 17/05/2015:10:25:50 +0000 GET /presentations/
    83.149.9.216 - - 17/05/2015:10:25:51 +0000 GET /presentations/
    83.149.9.216 - - 17/05/2015:10:25:52 +0000 GET /presentations/
    83.149.9.216 - - 17/05/2015:10:25:55 +0000 GET /presentations/
    83.149.9.216 - - 17/05/2015:10:25:56 +0000 GET /presentations/
    83.149.9.216 - - 17/05/2015:10:25:56 +0000 GET /present
    83.149.9.216 - - 17/05/2015:10:25:57 +0000 GET /present
    83.149.9.216 - - 17/05/2015:10:26:01 +0000 GET /
    83.149.9.216 - - 17/05/2015:10:26:02 +0000 GET /pre
    83.149.9.216 - - 17/05/2015:10:25:46 +0000 GET /presentations/
    83.149.9.216 - - 17/05/2015:10:26:02 +0000 GET /pre
    83.149.9.216 - - 17/05/2015:10:26:03 +0000 GET /pre
    ```

  + 输出

    *由于`onTimer`简单粗暴直接`pageViewCountListState.clear();`导致前面几次排名信息中丢失第一名以外的数据 => 下面代码3-乱序数据代码改进 中解决问题*

    ```shell
    data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829549000, method='GET', url='/presentations/'}
    data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829550000, method='GET', url='/presentations/'}
    data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829551000, method='GET', url='/presentations/'}
    data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829552000, method='GET', url='/presentations/'}
    data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829555000, method='GET', url='/presentations/'}
    data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829556000, method='GET', url='/presentations/'}
    data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829556000, method='GET', url='/present'}
    data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829557000, method='GET', url='/present'}
    data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829561000, method='GET', url='/'}
    data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829562000, method='GET', url='/pre'}
    data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829546000, method='GET', url='/presentations/'}
    data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829562000, method='GET', url='/pre'}
    agg> PageViewCount{url='/presentations/', windowEnd=1431829550000, count=2}
    agg> PageViewCount{url='/presentations/', windowEnd=1431829555000, count=5}
    agg> PageViewCount{url='/presentations/', windowEnd=1431829560000, count=7}
    agg> PageViewCount{url='/present', windowEnd=1431829560000, count=2}
    ============================
    窗口结束时间：2015-05-17 10:25:50.0
    NO 1: 页面URL = /presentations/ 浏览量 = 2
    ===============================
    
    ============================
    窗口结束时间：2015-05-17 10:25:55.0
    NO 1: 页面URL = /presentations/ 浏览量 = 5
    ===============================
    
    ============================
    窗口结束时间：2015-05-17 10:26:00.0
    NO 1: 页面URL = /presentations/ 浏览量 = 7
    NO 2: 页面URL = /present 浏览量 = 2
    ===============================
    ```

#### 代码3-乱序数据-代码改进

**一个数据只有不属于任何窗口了，才会被丢进侧输出流！**

---

+ java代码

  ```java
  import beans.ApacheLogEvent;
  import beans.PageViewCount;
  import org.apache.commons.compress.utils.Lists;
  import org.apache.flink.api.common.functions.AggregateFunction;
  import org.apache.flink.api.common.state.MapState;
  import org.apache.flink.api.common.state.MapStateDescriptor;
  import org.apache.flink.configuration.Configuration;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
  import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
  import org.apache.flink.streaming.api.functions.windowing.WindowFunction;
  import org.apache.flink.streaming.api.windowing.assigners.SlidingEventTimeWindows;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
  import org.apache.flink.streaming.runtime.operators.util.AssignerWithPeriodicWatermarksAdapter;
  import org.apache.flink.util.Collector;
  import org.apache.flink.util.OutputTag;
  
  import java.sql.Timestamp;
  import java.text.SimpleDateFormat;
  import java.util.ArrayList;
  import java.util.Map;
  import java.util.concurrent.TimeUnit;
  import java.util.regex.Pattern;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/5 1:27 AM
   */
  public class HotPages {
    public static void main(String[] args) throws Exception {
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
  
      // 读取文件，转换成POJO
      //        URL resource = HotPages.class.getResource("/apache.log");
      //        DataStream<String> inputStream = env.readTextFile(resource.getPath());
  
      // 方便测试，使用本地Socket输入数据
      DataStream<String> inputStream = env.socketTextStream("localhost", 7777);
  
  
      DataStream<ApacheLogEvent> dataStream = inputStream
        .map(line -> {
          String[] fields = line.split(" ");
          SimpleDateFormat simpleDateFormat = new SimpleDateFormat("dd/MM/yyyy:HH:mm:ss");
          Long timestamp = simpleDateFormat.parse(fields[3]).getTime();
          return new ApacheLogEvent(fields[0], fields[1], timestamp, fields[5], fields[6]);
        })
        .assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
          new BoundedOutOfOrdernessTimestampExtractor<ApacheLogEvent>(Time.of(1, TimeUnit.SECONDS)) {
            @Override
            public long extractTimestamp(ApacheLogEvent element) {
              return element.getTimestamp();
            }
          }
        ));
  
      dataStream.print("data");
  
  
      // 定义一个侧输出流标签
      OutputTag<ApacheLogEvent> lateTag = new OutputTag<ApacheLogEvent>("late") {
      };
  
      // 分组开窗聚合
      SingleOutputStreamOperator<PageViewCount> windowAggStream = dataStream
        // 过滤get请求
        .filter(data -> "GET".equals(data.getMethod()))
        .filter(data -> {
          String regex = "^((?!\\.(css|js|png|ico)$).)*$";
          return Pattern.matches(regex, data.getUrl());
        })
        // 按照url分组
        .keyBy(ApacheLogEvent::getUrl)
        .window(SlidingEventTimeWindows.of(Time.minutes(10), Time.seconds(5)))
        .allowedLateness(Time.minutes(1))
        .sideOutputLateData(lateTag)
        .aggregate(new PageCountAgg(), new PageCountResult());
  
  
      windowAggStream.print("agg");
      windowAggStream.getSideOutput(lateTag).print("late");
  
  
      // 收集同一窗口count数据，排序输出
      DataStream<String> resultStream = windowAggStream
        .keyBy(PageViewCount::getWindowEnd)
        .process(new TopNHotPages(3));
  
      resultStream.print();
  
      env.execute("hot pages job");
    }
  
    // 自定义预聚合函数
    public static class PageCountAgg implements AggregateFunction<ApacheLogEvent, Long, Long> {
      @Override
      public Long createAccumulator() {
        return 0L;
      }
  
      @Override
      public Long add(ApacheLogEvent value, Long accumulator) {
        return accumulator + 1;
      }
  
      @Override
      public Long getResult(Long accumulator) {
        return accumulator;
      }
  
      @Override
      public Long merge(Long a, Long b) {
        return a + b;
      }
    }
  
    // 实现自定义的窗口函数
    public static class PageCountResult implements WindowFunction<Long, PageViewCount, String, TimeWindow> {
      @Override
      public void apply(String url, TimeWindow window, Iterable<Long> input, Collector<PageViewCount> out) throws Exception {
        out.collect(new PageViewCount(url, window.getEnd(), input.iterator().next()));
      }
    }
  
    // 实现自定义的处理函数
    public static class TopNHotPages extends KeyedProcessFunction<Long, PageViewCount, String> {
      private Integer topSize;
  
      public TopNHotPages(Integer topSize) {
        this.topSize = topSize;
      }
  
      // 定义状态，保存当前所有PageViewCount到Map中
      //        ListState<PageViewCount> pageViewCountListState;
      MapState<String, Long> pageViewCountMapState;
  
      @Override
      public void open(Configuration parameters) throws Exception {
        //            pageViewCountListState = getRuntimeContext().getListState(new ListStateDescriptor<PageViewCount>("page-count-list", PageViewCount.class));
        pageViewCountMapState = getRuntimeContext().getMapState(new MapStateDescriptor<String, Long>("page-count-map", String.class, Long.class));
      }
  
      @Override
      public void processElement(PageViewCount value, Context ctx, Collector<String> out) throws Exception {
        //            pageViewCountListState.add(value);
        pageViewCountMapState.put(value.getUrl(), value.getCount());
        ctx.timerService().registerEventTimeTimer(value.getWindowEnd() + 1);
        // 注册一个1分钟之后的定时器，用来清空状态
        ctx.timerService().registerEventTimeTimer(value.getWindowEnd() + 60 * 1000L);
      }
  
  
      @Override
      public void onTimer(long timestamp, OnTimerContext ctx, Collector<String> out) throws Exception {
        // 先判断是否到了窗口关闭清理时间，如果是，直接清空状态返回
        if (timestamp == ctx.getCurrentKey() + 60 * 1000L) {
          pageViewCountMapState.clear();
          return;
        }
  
  
        //            ArrayList<PageViewCount> pageViewCounts = Lists.newArrayList(pageViewCountListState.get().iterator());
        ArrayList<Map.Entry<String, Long>> pageViewCounts = Lists.newArrayList(pageViewCountMapState.entries().iterator());
  
        pageViewCounts.sort((a, b) -> -Long.compare(a.getValue(), b.getValue()));
  
        // 格式化成String输出
        StringBuilder resultBuilder = new StringBuilder();
        resultBuilder.append("============================").append(System.lineSeparator());
        resultBuilder.append("窗口结束时间：").append(new Timestamp(timestamp - 1)).append(System.lineSeparator());
  
        // 遍历列表，取top n输出
        for (int i = 0; i < Math.min(topSize, pageViewCounts.size()); i++) {
          //                PageViewCount pageViewCount = pageViewCounts.get(i);
          Map.Entry<String, Long> pageViewCount = pageViewCounts.get(i);
          resultBuilder.append("NO ").append(i + 1).append(":")
            .append(" 页面URL = ").append(pageViewCount.getKey())
            .append(" 浏览量 = ").append(pageViewCount.getValue())
            .append(System.lineSeparator());
        }
        resultBuilder.append("===============================").append(System.lineSeparator());
  
        // 控制输出频率
        Thread.sleep(1000L);
  
        out.collect(resultBuilder.toString());
  
  
        //            pageViewCountListState.clear();
      }
    }
  }
  
  ```

+ 输入

  ```shell
  83.149.9.216 - - 17/05/2015:10:25:49 +0000 GET /presentations/
  83.149.9.216 - - 17/05/2015:10:25:50 +0000 GET /presentations/
  83.149.9.216 - - 17/05/2015:10:25:51 +0000 GET /presentations/
  83.149.9.216 - - 17/05/2015:10:25:52 +0000 GET /presentations/
  83.149.9.216 - - 17/05/2015:10:25:55 +0000 GET /presentations/
  83.149.9.216 - - 17/05/2015:10:25:56 +0000 GET /presentations/
  83.149.9.216 - - 17/05/2015:10:25:56 +0000 GET /present
  83.149.9.216 - - 17/05/2015:10:25:57 +0000 GET /present
  83.149.9.216 - - 17/05/2015:10:26:01 +0000 GET /
  83.149.9.216 - - 17/05/2015:10:26:02 +0000 GET /pre
  83.149.9.216 - - 17/05/2015:10:25:46 +0000 GET /presentations/
  83.149.9.216 - - 17/05/2015:10:26:03 +0000 GET /pre
  ```

+ 输出

  ```shell
  data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829549000, method='GET', url='/presentations/'}
  data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829550000, method='GET', url='/presentations/'}
  data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829551000, method='GET', url='/presentations/'}
  data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829552000, method='GET', url='/presentations/'}
  data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829555000, method='GET', url='/presentations/'}
  data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829556000, method='GET', url='/presentations/'}
  data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829556000, method='GET', url='/present'}
  data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829557000, method='GET', url='/present'}
  data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829561000, method='GET', url='/'}
  data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829562000, method='GET', url='/pre'}
  data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829546000, method='GET', url='/presentations/'}
  agg> PageViewCount{url='/presentations/', windowEnd=1431829550000, count=2}
  agg> PageViewCount{url='/presentations/', windowEnd=1431829555000, count=5}
  agg> PageViewCount{url='/presentations/', windowEnd=1431829560000, count=7}
  agg> PageViewCount{url='/present', windowEnd=1431829560000, count=2}
  data> ApacheLogEvent{ip='83.149.9.216', userId='-', timestamp=1431829563000, method='GET', url='/pre'}
  ============================
  窗口结束时间：2015-05-17 10:25:50.0
  NO 1: 页面URL = /presentations/ 浏览量 = 2
  ===============================
  
  ============================
  窗口结束时间：2015-05-17 10:25:55.0
  NO 1: 页面URL = /presentations/ 浏览量 = 5
  ===============================
  
  ============================
  窗口结束时间：2015-05-17 10:26:00.0
  NO 1: 页面URL = /presentations/ 浏览量 = 7
  NO 2: 页面URL = /present 浏览量 = 2
  ===============================
  ```

### 14.3.3 实时流量统计——PV和UV

+ 基本需求
  + 从埋点日志中，统计实时的PV和UV
  + 统计每小时的访问量（PV），并且对用户进行去重（UV）
+ 解决思路
  + 统计埋点日志中的pv行为，利用Set数据结构进行去重
  + **对于超大规模的数据，可以考虑用布隆过滤器进行去重**

#### 代码1-PV统计-基本实现

+ java代码

  ```java
  
  import beans.UserBehavior;
  import org.apache.flink.api.common.functions.MapFunction;
  import org.apache.flink.api.java.tuple.Tuple2;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
  import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.streaming.runtime.operators.util.AssignerWithPeriodicWatermarksAdapter;
  
  import java.net.URL;
  import java.util.concurrent.TimeUnit;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/5 3:11 AM
   */
  public class PageView {
    public static void main(String[] args) throws Exception {
      // 1. 创建执行环境
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      // 设置并行度为1
      env.setParallelism(1);
  
      // 2. 从csv文件中获取数据
      URL resource = PageView.class.getResource("/UserBehavior.csv");
      DataStream<String> inputStream = env.readTextFile(resource.getPath());
  
      // 3. 转换成POJO,分配时间戳和watermark
      DataStream<UserBehavior> userBehaviorDataStream = inputStream.map(line -> {
        String[] fields = line.split(",");
        return new UserBehavior(new Long(fields[0]), new Long(fields[1]), new Integer(fields[2]), fields[3], new Long(fields[4]));
      }).assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
        new BoundedOutOfOrdernessTimestampExtractor<UserBehavior>(Time.of(200, TimeUnit.MILLISECONDS)) {
          @Override
          public long extractTimestamp(UserBehavior element) {
            return element.getTimestamp() * 1000L;
          }
        }
      ));
  
      // 4. 分组开窗聚合，得到每个窗口内各个商品的count值
      DataStream<Tuple2<String, Long>> pvResultStream = userBehaviorDataStream
        // 过滤只保留pv行为
        .filter(userBehavior -> "pv".equals(userBehavior.getBehavior()))
        .map(new MapFunction<UserBehavior, Tuple2<String, Long>>() {
          @Override
          public Tuple2<String, Long> map(UserBehavior value) throws Exception {
            return new Tuple2<>("pv", 1L);
          }
        })
        // 按照商品ID分组
        .keyBy(item -> item.f0)
        // 1小时滚动窗口
        .window(TumblingEventTimeWindows.of(Time.hours(1)))
        .sum(1);
  
  
      pvResultStream.print();
  
      env.execute("pv count job");
    }
  }
  
  ```

+ 输出

  ```shell
  (pv,41890)
  (pv,48022)
  (pv,47298)
  (pv,44499)
  (pv,48649)
  (pv,50838)
  (pv,52296)
  (pv,52552)
  (pv,48292)
  (pv,13)
  ```

#### 代码2 PV统计-并行和数据倾斜优化

+ java代码

  ```java
  
  import beans.PageViewCount;
  import beans.UserBehavior;
  import org.apache.flink.api.common.functions.AggregateFunction;
  import org.apache.flink.api.common.functions.MapFunction;
  import org.apache.flink.api.common.state.ValueState;
  import org.apache.flink.api.common.state.ValueStateDescriptor;
  import org.apache.flink.api.java.tuple.Tuple2;
  import org.apache.flink.configuration.Configuration;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
  import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
  import org.apache.flink.streaming.api.functions.windowing.WindowFunction;
  import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
  import org.apache.flink.streaming.runtime.operators.util.AssignerWithPeriodicWatermarksAdapter;
  
  import org.apache.flink.util.Collector;
  
  import java.net.URL;
  import java.util.Random;
  import java.util.concurrent.TimeUnit;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/5 3:11 AM
   */
  public class PageView {
    public static void main(String[] args) throws Exception {
      // 1. 创建执行环境
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      // 设置并行度为4
      env.setParallelism(4);
  
      // 2. 从csv文件中获取数据
      URL resource = PageView.class.getResource("/UserBehavior.csv");
      DataStream<String> inputStream = env.readTextFile(resource.getPath());
  
      // 3. 转换成POJO,分配时间戳和watermark
      DataStream<UserBehavior> userBehaviorDataStream = inputStream.map(line -> {
        String[] fields = line.split(",");
        return new UserBehavior(new Long(fields[0]), new Long(fields[1]), new Integer(fields[2]), fields[3], new Long(fields[4]));
      }).assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
        new BoundedOutOfOrdernessTimestampExtractor<UserBehavior>(Time.of(200, TimeUnit.MILLISECONDS)) {
          @Override
          public long extractTimestamp(UserBehavior element) {
            return element.getTimestamp() * 1000L;
          }
        }
      ));
  
      // 4. 分组开窗聚合，得到每个窗口内各个商品的count值
      DataStream<Tuple2<String, Long>> pvResultStream0 = userBehaviorDataStream
        // 过滤只保留pv行为
        .filter(userBehavior -> "pv".equals(userBehavior.getBehavior()))
        .map(new MapFunction<UserBehavior, Tuple2<String, Long>>() {
          @Override
          public Tuple2<String, Long> map(UserBehavior value) throws Exception {
            return new Tuple2<>("pv", 1L);
          }
        })
        // 按照商品ID分组
        .keyBy(item -> item.f0)
        // 1小时滚动窗口
        .window(TumblingEventTimeWindows.of(Time.hours(1)))
        .sum(1);
  
      //  并行任务改进，设计随机key，解决数据倾斜问题
      SingleOutputStreamOperator<PageViewCount> pvStream = userBehaviorDataStream.filter(data -> "pv".equals(data.getBehavior()))
        .map(new MapFunction<UserBehavior, Tuple2<Integer, Long>>() {
          @Override
          public Tuple2<Integer, Long> map(UserBehavior value) throws Exception {
            Random random = new Random();
            return new Tuple2<>(random.nextInt(10), 1L);
          }
        })
        .keyBy(data -> data.f0)
        .window(TumblingEventTimeWindows.of(Time.hours(1)))
        .aggregate(new PvCountAgg(), new PvCountResult());
  
      // 将各分区数据汇总起来
      DataStream<PageViewCount> pvResultStream = pvStream
        .keyBy(PageViewCount::getWindowEnd)
        .process(new TotalPvCount());
      //                .sum("count");
  
      pvResultStream.print();
  
      env.execute("pv count job");
    }
  
    // 实现自定义预聚合函数
    public static class PvCountAgg implements AggregateFunction<Tuple2<Integer, Long>, Long, Long> {
      @Override
      public Long createAccumulator() {
        return 0L;
      }
  
      @Override
      public Long add(Tuple2<Integer, Long> value, Long accumulator) {
        return accumulator + 1;
      }
  
      @Override
      public Long getResult(Long accumulator) {
        return accumulator;
      }
  
      @Override
      public Long merge(Long a, Long b) {
        return a + b;
      }
    }
  
    // 实现自定义窗口
    public static class PvCountResult implements WindowFunction<Long, PageViewCount, Integer, TimeWindow> {
      @Override
      public void apply(Integer integer, TimeWindow window, Iterable<Long> input, Collector<PageViewCount> out) throws Exception {
        out.collect( new PageViewCount(integer.toString(), window.getEnd(), input.iterator().next()) );
      }
    }
  
    // 实现自定义处理函数，把相同窗口分组统计的count值叠加
    public static class TotalPvCount extends KeyedProcessFunction<Long, PageViewCount, PageViewCount> {
      // 定义状态，保存当前的总count值
      ValueState<Long> totalCountState;
  
      @Override
      public void open(Configuration parameters) throws Exception {
        totalCountState = getRuntimeContext().getState(new ValueStateDescriptor<Long>("total-count", Long.class));
      }
  
      @Override
      public void processElement(PageViewCount value, Context ctx, Collector<PageViewCount> out) throws Exception {
        Long totalCount = totalCountState.value();
        if(null == totalCount){
          totalCount = 0L;
          totalCountState.update(totalCount);
        }
        totalCountState.update( totalCount + value.getCount() );
        ctx.timerService().registerEventTimeTimer(value.getWindowEnd() + 1);
      }
  
      @Override
      public void onTimer(long timestamp, OnTimerContext ctx, Collector<PageViewCount> out) throws Exception {
        // 定时器触发，所有分组count值都到齐，直接输出当前的总count数量
        Long totalCount = totalCountState.value();
        out.collect(new PageViewCount("pv", ctx.getCurrentKey(), totalCount));
        // 清空状态
        totalCountState.clear();
      }
    }
  }
  
  ```

+ 输出

  ```shell
  2> PageViewCount{url='pv', windowEnd=1511661600000, count=41890}
  2> PageViewCount{url='pv', windowEnd=1511679600000, count=50838}
  1> PageViewCount{url='pv', windowEnd=1511676000000, count=48649}
  4> PageViewCount{url='pv', windowEnd=1511668800000, count=47298}
  2> PageViewCount{url='pv', windowEnd=1511686800000, count=52552}
  4> PageViewCount{url='pv', windowEnd=1511672400000, count=44499}
  3> PageViewCount{url='pv', windowEnd=1511665200000, count=48022}
  4> PageViewCount{url='pv', windowEnd=1511683200000, count=52296}
  2> PageViewCount{url='pv', windowEnd=1511690400000, count=48292}
  3> PageViewCount{url='pv', windowEnd=1511694000000, count=13}
  ```

#### 代码3-UV统计-Set去重

+ java代码

  ```java
  import beans.PageViewCount;
  import beans.UserBehavior;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
  import org.apache.flink.streaming.api.functions.windowing.AllWindowFunction;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
  import org.apache.flink.streaming.runtime.operators.util.AssignerWithPeriodicWatermarksAdapter;
  import org.apache.flink.util.Collector;
  
  import java.net.URL;
  import java.util.HashSet;
  import java.util.concurrent.TimeUnit;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/5 3:51 AM
   */
  public class UniqueVisitor {
    public static void main(String[] args) throws Exception {
      // 1. 创建执行环境
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      // 设置并行度为1
      env.setParallelism(1);
  
      // 2. 从csv文件中获取数据
      URL resource = UniqueVisitor.class.getResource("/UserBehavior.csv");
      DataStream<String> inputStream = env.readTextFile(resource.getPath());
  
      // 3. 转换成POJO,分配时间戳和watermark
      DataStream<UserBehavior> dataStream = inputStream.map(line -> {
        String[] fields = line.split(",");
        return new UserBehavior(new Long(fields[0]), new Long(fields[1]), new Integer(fields[2]), fields[3], new Long(fields[4]));
      }).assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
        new BoundedOutOfOrdernessTimestampExtractor<UserBehavior>(Time.of(200, TimeUnit.MILLISECONDS)) {
          @Override
          public long extractTimestamp(UserBehavior element) {
            return element.getTimestamp() * 1000L;
          }
        }
      ));
  
      // 开窗统计uv值
      SingleOutputStreamOperator<PageViewCount> uvStream = dataStream.filter(data -> "pv".equals(data.getBehavior()))
        .timeWindowAll(Time.hours(1))
        .apply(new UvCountResult());
  
      uvStream.print();
  
  
      env.execute("uv count job");
    }
  
    // 实现自定义全窗口函数
    public static class UvCountResult implements AllWindowFunction<UserBehavior, PageViewCount, TimeWindow> {
      @Override
      public void apply(TimeWindow window, Iterable<UserBehavior> values, Collector<PageViewCount> out) throws Exception {
        // 定义一个Set结构，保存窗口中的所有userId，自动去重
        HashSet<Long> uidSet = new HashSet<>();
        for (UserBehavior ub : values) {
          uidSet.add(ub.getUerId());
        }
        out.collect(new PageViewCount("uv", window.getEnd(), (long) uidSet.size()));
      }
    }
  }
  ```

+ 输出

  ```shell
  PageViewCount{url='uv', windowEnd=1511661600000, count=28196}
  PageViewCount{url='uv', windowEnd=1511665200000, count=32160}
  PageViewCount{url='uv', windowEnd=1511668800000, count=32233}
  PageViewCount{url='uv', windowEnd=1511672400000, count=30615}
  PageViewCount{url='uv', windowEnd=1511676000000, count=32747}
  PageViewCount{url='uv', windowEnd=1511679600000, count=33898}
  PageViewCount{url='uv', windowEnd=1511683200000, count=34631}
  PageViewCount{url='uv', windowEnd=1511686800000, count=34746}
  PageViewCount{url='uv', windowEnd=1511690400000, count=32356}
  PageViewCount{url='uv', windowEnd=1511694000000, count=13}
  ```

#### 代码4-UV统计-布隆过滤器

+ pom依赖

  ```xml
  <dependencies>
    <dependency>
      <groupId>redis.clients</groupId>
      <artifactId>jedis</artifactId>
      <version>3.5.1</version>
    </dependency>
  </dependencies>
  ```

+ java代码

  ```java
  import beans.PageViewCount;
  import beans.UserBehavior;
  import org.apache.flink.configuration.Configuration;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
  import org.apache.flink.streaming.api.functions.windowing.ProcessAllWindowFunction;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.streaming.api.windowing.triggers.Trigger;
  import org.apache.flink.streaming.api.windowing.triggers.TriggerResult;
  import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
  import org.apache.flink.streaming.runtime.operators.util.AssignerWithPeriodicWatermarksAdapter;
  import org.apache.flink.util.Collector;
  import redis.clients.jedis.Jedis;
  
  import java.net.URL;
  import java.util.concurrent.TimeUnit;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/5 4:03 AM
   */
  public class UvWithBloomFilter {
    public static void main(String[] args) throws Exception {
      // 1. 创建执行环境
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      // 设置并行度为1
      env.setParallelism(1);
  
      // 2. 从csv文件中获取数据
      URL resource = UniqueVisitor.class.getResource("/UserBehavior.csv");
      DataStream<String> inputStream = env.readTextFile(resource.getPath());
  
      // 3. 转换成POJO,分配时间戳和watermark
      DataStream<UserBehavior> dataStream = inputStream.map(line -> {
        String[] fields = line.split(",");
        return new UserBehavior(new Long(fields[0]), new Long(fields[1]), new Integer(fields[2]), fields[3], new Long(fields[4]));
      }).assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
        new BoundedOutOfOrdernessTimestampExtractor<UserBehavior>(Time.of(200, TimeUnit.MILLISECONDS)) {
          @Override
          public long extractTimestamp(UserBehavior element) {
            return element.getTimestamp() * 1000L;
          }
        }
      ));
  
      // 开窗统计uv值
      SingleOutputStreamOperator<PageViewCount> uvStream = dataStream
        .filter(data -> "pv".equals(data.getBehavior()))
        .timeWindowAll(Time.hours(1))
        .trigger(new MyTrigger())
        .process(new UvCountResultWithBloomFliter());
  
  
      uvStream.print();
  
      env.execute("uv count with bloom filter job");
    }
  
    // 自定义触发器
    public static class MyTrigger extends Trigger<UserBehavior, TimeWindow> {
      @Override
      public TriggerResult onElement(UserBehavior element, long timestamp, TimeWindow window, TriggerContext ctx) throws Exception {
        // 每一条数据来到，直接触发窗口计算，并且直接清空窗口
        return TriggerResult.FIRE_AND_PURGE;
      }
  
      @Override
      public TriggerResult onProcessingTime(long time, TimeWindow window, TriggerContext ctx) throws Exception {
        return TriggerResult.CONTINUE;
      }
  
      @Override
      public TriggerResult onEventTime(long time, TimeWindow window, TriggerContext ctx) throws Exception {
        return TriggerResult.CONTINUE;
      }
  
      @Override
      public void clear(TimeWindow window, TriggerContext ctx) throws Exception {
      }
    }
  
    // 自定义一个布隆过滤器
    public static class MyBloomFilter {
      // 定义位图的大小，一般需要定义为2的整次幂
      private Integer cap;
  
      public MyBloomFilter(Integer cap) {
        this.cap = cap;
      }
  
      // 实现一个hash函数
      public Long hashCode(String value, Integer seed) {
        Long result = 0L;
        for (int i = 0; i < value.length(); i++) {
          result = result * seed + value.charAt(i);
        }
        return result & (cap - 1);
      }
    }
  
    // 实现自定义的处理函数
    public static class UvCountResultWithBloomFliter extends ProcessAllWindowFunction<UserBehavior, PageViewCount, TimeWindow> {
      // 定义jedis连接和布隆过滤器
      Jedis jedis;
      MyBloomFilter myBloomFilter;
  
      @Override
      public void open(Configuration parameters) throws Exception {
        jedis = new Jedis("localhost", 6379);
        myBloomFilter = new MyBloomFilter(1 << 29);    // 要处理1亿个数据，用64MB大小的位图
      }
  
      @Override
      public void process(Context context, Iterable<UserBehavior> elements, Collector<PageViewCount> out) throws Exception {
        // 将位图和窗口count值全部存入redis，用windowEnd作为key
        Long windowEnd = context.window().getEnd();
        String bitmapKey = windowEnd.toString();
        // 把count值存成一张hash表
        String countHashName = "uv_count";
        String countKey = windowEnd.toString();
  
        // 1. 取当前的userId
        Long userId = elements.iterator().next().getUerId();
  
        // 2. 计算位图中的offset
        Long offset = myBloomFilter.hashCode(userId.toString(), 61);
  
        // 3. 用redis的getbit命令，判断对应位置的值
        Boolean isExist = jedis.getbit(bitmapKey, offset);
  
        if (!isExist) {
          // 如果不存在，对应位图位置置1
          jedis.setbit(bitmapKey, offset, true);
  
          // 更新redis中保存的count值
          Long uvCount = 0L;    // 初始count值
          String uvCountString = jedis.hget(countHashName, countKey);
          if (uvCountString != null && !"".equals(uvCountString)) {
            uvCount = Long.valueOf(uvCountString);
          }
          jedis.hset(countHashName, countKey, String.valueOf(uvCount + 1));
  
          out.collect(new PageViewCount("uv", windowEnd, uvCount + 1));
        }
      }
  
      @Override
      public void close() throws Exception {
        jedis.close();
      }
    }
  }
  
  ```

+ 输出

  ```shell
  ....
  PageViewCount{url='uv', windowEnd=1511661600000, count=7469}
  PageViewCount{url='uv', windowEnd=1511661600000, count=7470}
  PageViewCount{url='uv', windowEnd=1511661600000, count=7471}
  PageViewCount{url='uv', windowEnd=1511661600000, count=7472}
  PageViewCount{url='uv', windowEnd=1511661600000, count=7473}
  PageViewCount{url='uv', windowEnd=1511661600000, count=7474}
  ...
  ```

### 14.3.4 市场营销分析——APP市场推广统计

+ 基本需求
  + 从埋点日志中，统计APP市场推广的数据指标
  + 按照不同的推广渠道，分别统计数据
+ 解决思路
  + 通过过滤日志中的用户行为，按照不同的渠道进行统计
  + 可以用process function处理，得到自定义的输出数据信息

#### POJO

+ MarketingUserBehavior

  ```java
  private Long userId;
  private String behavior;
  private String channel;
  private Long timestamp;
  ```

+ ChannelPromotionCount

  ```java
  private String channel;
  private String behavior;
  private String windowEnd;
  private Long count;
  ```

#### 代码1-自定义测试数据源

+ java代码

  ```java
  import beans.MarketingUserBehavior;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.source.SourceFunction;
  
  import java.util.Arrays;
  import java.util.List;
  import java.util.Random;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/5 5:32 AM
   */
  public class AppMarketingByChannel {
    public static void main(String[] args) throws Exception {
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
  
      // 1. 从自定义数据源中读取数据
      DataStream<MarketingUserBehavior> dataStream = env.addSource(new SimulatedMarketingUserBehaviorSource());
  
    }
  
    // 实现自定义的模拟市场用户行为数据源
    public static class SimulatedMarketingUserBehaviorSource implements SourceFunction<MarketingUserBehavior> {
      // 控制是否正常运行的标识位
      Boolean running = true;
  
      // 定义用户行为和渠道的范围
      List<String> behaviorList = Arrays.asList("CLICK", "DOWNLOAD", "INSTALL", "UNINSTALL");
      List<String> channelList = Arrays.asList("app store", "wechat", "weibo");
  
      Random random = new Random();
  
      @Override
      public void run(SourceContext<MarketingUserBehavior> ctx) throws Exception {
        while (running) {
          // 随机生成所有字段
          Long id = random.nextLong();
          String behavior = behaviorList.get(random.nextInt(behaviorList.size()));
          String channel = channelList.get(random.nextInt(channelList.size()));
          Long timestamp = System.currentTimeMillis();
  
          // 发出数据
          ctx.collect(new MarketingUserBehavior(id, behavior, channel, timestamp));
  
          Thread.sleep(100L);
        }
      }
  
      @Override
      public void cancel() {
        running = false;
      }
    }
  }
  ```


#### 代码2-具体实现

+ java代码

  ```java
  import beans.ChannelPromotionCount;
  import beans.MarketingUserBehavior;
  import org.apache.flink.api.common.functions.AggregateFunction;
  import org.apache.flink.api.java.functions.KeySelector;
  import org.apache.flink.api.java.tuple.Tuple2;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.source.SourceFunction;
  import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
  import org.apache.flink.streaming.api.functions.windowing.ProcessWindowFunction;
  import org.apache.flink.streaming.api.windowing.assigners.SlidingEventTimeWindows;
  import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows;
  import org.apache.flink.streaming.api.windowing.assigners.TumblingProcessingTimeWindows;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
  import org.apache.flink.streaming.runtime.operators.util.AssignerWithPeriodicWatermarksAdapter;
  import org.apache.flink.util.Collector;
  
  import java.sql.Timestamp;
  import java.util.Arrays;
  import java.util.List;
  import java.util.Random;
  import java.util.concurrent.TimeUnit;
  
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/5 5:32 AM
   */
  public class AppMarketingByChannel {
    public static void main(String[] args) throws Exception {
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
  
      // 1. 从自定义数据源中读取数据
      DataStream<MarketingUserBehavior> dataStream = env.addSource(new SimulatedMarketingUserBehaviorSource())
        .assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
          new BoundedOutOfOrdernessTimestampExtractor<MarketingUserBehavior>(Time.of(200, TimeUnit.MILLISECONDS)) {
            @Override
            public long extractTimestamp(MarketingUserBehavior element) {
              return element.getTimestamp();
            }
          }
        ));
  
      // 2. 分渠道开窗统计
      DataStream<ChannelPromotionCount> resultStream = dataStream
        .filter(data -> !"UNINSTALL".equals(data.getBehavior()))
        .keyBy(new KeySelector<MarketingUserBehavior, Tuple2<String, String>>() {
          @Override
          public Tuple2<String, String> getKey(MarketingUserBehavior value) throws Exception {
            return new Tuple2<>(value.getChannel(), value.getBehavior());
          }
        })
        // 定义滑窗
        .window(SlidingEventTimeWindows.of(Time.hours(1), Time.seconds(5)))
        .aggregate(new MarketingCountAgg(), new MarketingCountResult());
  
      resultStream.print();
  
      env.execute("app marketing by channel job");
  
    }
  
    // 实现自定义的模拟市场用户行为数据源
    public static class SimulatedMarketingUserBehaviorSource implements SourceFunction<MarketingUserBehavior> {
      // 控制是否正常运行的标识位
      Boolean running = true;
  
      // 定义用户行为和渠道的范围
      List<String> behaviorList = Arrays.asList("CLICK", "DOWNLOAD", "INSTALL", "UNINSTALL");
      List<String> channelList = Arrays.asList("app store", "wechat", "weibo");
  
      Random random = new Random();
  
      @Override
      public void run(SourceContext<MarketingUserBehavior> ctx) throws Exception {
        while (running) {
          // 随机生成所有字段
          Long id = random.nextLong();
          String behavior = behaviorList.get(random.nextInt(behaviorList.size()));
          String channel = channelList.get(random.nextInt(channelList.size()));
          Long timestamp = System.currentTimeMillis();
  
          // 发出数据
          ctx.collect(new MarketingUserBehavior(id, behavior, channel, timestamp));
  
          Thread.sleep(100L);
        }
      }
  
      @Override
      public void cancel() {
        running = false;
      }
    }
  
    // 实现自定义的增量聚合函数
    public static class MarketingCountAgg implements AggregateFunction<MarketingUserBehavior, Long, Long> {
  
      @Override
      public Long createAccumulator() {
        return 0L;
      }
  
      @Override
      public Long add(MarketingUserBehavior value, Long accumulator) {
        return accumulator + 1;
      }
  
      @Override
      public Long getResult(Long accumulator) {
        return accumulator;
      }
  
      @Override
      public Long merge(Long a, Long b) {
        return a + b;
      }
    }
  
    // 实现自定义的全窗口函数
    public static class MarketingCountResult extends ProcessWindowFunction<Long, ChannelPromotionCount, Tuple2<String, String>, TimeWindow> {
  
      @Override
      public void process(Tuple2<String, String> stringStringTuple2, Context context, Iterable<Long> elements, Collector<ChannelPromotionCount> out) throws Exception {
        String channel = stringStringTuple2.f0;
        String behavior = stringStringTuple2.f1;
        String windowEnd = new Timestamp(context.window().getEnd()).toString();
        Long count = elements.iterator().next();
        out.collect(new ChannelPromotionCount(channel, behavior, windowEnd, count));
      }
    }
  }
  
  ```

+ 输出

  ```shell
  beans.ChannelPromotionCount{channel='app store', behavior='CLICK', windowEnd='2021-02-05 17:54:40.0', count=4}
  beans.ChannelPromotionCount{channel='weibo', behavior='DOWNLOAD', windowEnd='2021-02-05 17:54:40.0', count=1}
  beans.ChannelPromotionCount{channel='weibo', behavior='INSTALL', windowEnd='2021-02-05 17:54:40.0', count=1}
  beans.ChannelPromotionCount{channel='wechat', behavior='DOWNLOAD', windowEnd='2021-02-05 17:54:40.0', count=1}
  beans.ChannelPromotionCount{channel='wechat', behavior='INSTALL', windowEnd='2021-02-05 17:54:40.0', count=1}
  beans.ChannelPromotionCount{channel='weibo', behavior='INSTALL', windowEnd='2021-02-05 17:54:45.0', count=1}
  beans.ChannelPromotionCount{channel='app store', behavior='DOWNLOAD', windowEnd='2021-02-05 17:54:45.0', count=10}
  beans.ChannelPromotionCount{channel='weibo', behavior='CLICK', windowEnd='2021-02-05 17:54:45.0', count=2}
  beans.ChannelPromotionCount{channel='app store', behavior='CLICK', windowEnd='2021-02-05 17:54:45.0', count=9}
  .....
  ```

#### 代码3-不分渠道代码实现

+ java代码

  ```java
  import beans.ChannelPromotionCount;
  import beans.MarketingUserBehavior;
  import org.apache.flink.api.common.functions.AggregateFunction;
  import org.apache.flink.api.common.functions.MapFunction;
  import org.apache.flink.api.java.tuple.Tuple2;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.source.SourceFunction;
  import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
  import org.apache.flink.streaming.api.functions.windowing.WindowFunction;
  import org.apache.flink.streaming.api.windowing.assigners.SlidingEventTimeWindows;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
  import org.apache.flink.streaming.runtime.operators.util.AssignerWithPeriodicWatermarksAdapter;
  import org.apache.flink.util.Collector;
  
  import java.sql.Timestamp;
  import java.util.Arrays;
  import java.util.List;
  import java.util.Random;
  import java.util.concurrent.TimeUnit;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/5 6:19 PM
   */
  public class AppMarketingStatistics {
    public static void main(String[] args) throws Exception {
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
  
      // 1. 从自定义数据源中读取数据
      DataStream<MarketingUserBehavior> dataStream = env.addSource(new AppMarketingByChannel.SimulatedMarketingUserBehaviorSource())
        .assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
          new BoundedOutOfOrdernessTimestampExtractor<MarketingUserBehavior>(Time.of(200, TimeUnit.MILLISECONDS)) {
            @Override
            public long extractTimestamp(MarketingUserBehavior element) {
              return element.getTimestamp();
            }
          }
        ));
  
      // 2. 开窗统计总量
      DataStream<ChannelPromotionCount> resultStream = dataStream
        .filter(data -> !"UNINSTALL".equals(data.getBehavior()))
        .map(new MapFunction<MarketingUserBehavior, Tuple2<String, Long>>() {
          @Override
          public Tuple2<String, Long> map(MarketingUserBehavior value) throws Exception {
            return new Tuple2<>("total", 1L);
          }
        })
        .keyBy(tuple2 -> tuple2.f0)
        // 定义滑窗
        .window(SlidingEventTimeWindows.of(Time.hours(1), Time.seconds(5)))
        .aggregate(new MarketingStatisticsAgg(), new MarketingStatisticsResult());
  
      resultStream.print();
  
      env.execute("app marketing by channel job");
  
    }
  
    // 实现自定义的模拟市场用户行为数据源
    public static class SimulatedMarketingUserBehaviorSource implements SourceFunction<MarketingUserBehavior> {
      // 控制是否正常运行的标识位
      Boolean running = true;
  
      // 定义用户行为和渠道的范围
      List<String> behaviorList = Arrays.asList("CLICK", "DOWNLOAD", "INSTALL", "UNINSTALL");
      List<String> channelList = Arrays.asList("app store", "wechat", "weibo");
  
      Random random = new Random();
  
      @Override
      public void run(SourceContext<MarketingUserBehavior> ctx) throws Exception {
        while (running) {
          // 随机生成所有字段
          Long id = random.nextLong();
          String behavior = behaviorList.get(random.nextInt(behaviorList.size()));
          String channel = channelList.get(random.nextInt(channelList.size()));
          Long timestamp = System.currentTimeMillis();
  
          // 发出数据
          ctx.collect(new MarketingUserBehavior(id, behavior, channel, timestamp));
  
          Thread.sleep(100L);
        }
      }
  
      @Override
      public void cancel() {
        running = false;
      }
    }
  
    // 实现自定义的增量聚合函数
    public static class MarketingStatisticsAgg implements AggregateFunction<Tuple2<String, Long>, Long, Long> {
  
      @Override
      public Long createAccumulator() {
        return 0L;
      }
  
      @Override
      public Long add(Tuple2<String, Long> value, Long accumulator) {
        return accumulator + 1;
      }
  
      @Override
      public Long getResult(Long accumulator) {
        return accumulator;
      }
  
      @Override
      public Long merge(Long a, Long b) {
        return a + b;
      }
    }
  
    // 实现自定义的全窗口函数
    public static class MarketingStatisticsResult implements WindowFunction<Long, ChannelPromotionCount, String, TimeWindow> {
  
      @Override
      public void apply(String s, TimeWindow window, Iterable<Long> input, Collector<ChannelPromotionCount> out) throws Exception {
        String windowEnd = new Timestamp(window.getEnd()).toString();
        Long count = input.iterator().next();
        out.collect(new ChannelPromotionCount("total", "total", windowEnd, count));
      }
    }
  }
  ```

+ 输出

  ```java
  beans.ChannelPromotionCount{channel='total', behavior='total', windowEnd='2021-02-05 18:34:15.0', count=40}
  beans.ChannelPromotionCount{channel='total', behavior='total', windowEnd='2021-02-05 18:34:20.0', count=75}
  beans.ChannelPromotionCount{channel='total', behavior='total', windowEnd='2021-02-05 18:34:25.0', count=109}
  ....
  ```

### 14.3.5 市场营销分析——页面广告统计

+ 基本需求
  + 从埋点日志中，统计每小时页面广告的点击量，5秒刷新一次，并按照不同省份进行划分
  + 对于"刷单"式的频繁点击行为进行过滤，并将该用户加入黑名单
+ 解决思路
  + 根据省份进行分组，创建长度为1小时、滑动距离为5秒的时间窗口进行统计
  + 可以用`process function`进行黑名单过滤，检测用户对同一广告的点击量，如果超过上限则将用户信息以侧输出流输出到黑名单中

#### POJO

+ AdClickEvent

  ```java
  private Long userId;
  private Long adId;
  private String province;
  private String city;
  private Long timestamp;
  ```

+ BlackListUserWarning

  ```java
  private Long userId;
  private Long adId;
  private String warningMsg;
  ```

#### 代码1-基本实现

+ java代码

  ```java
  import beans.AdClickEvent;
  import beans.AdCountViewByProvince;
  import org.apache.flink.api.common.functions.AggregateFunction;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
  import org.apache.flink.streaming.api.functions.windowing.WindowFunction;
  import org.apache.flink.streaming.api.windowing.assigners.SlidingEventTimeWindows;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
  import org.apache.flink.streaming.runtime.operators.util.AssignerWithPeriodicWatermarksAdapter;
  import org.apache.flink.util.Collector;
  
  
  import java.net.URL;
  import java.sql.Timestamp;
  import java.util.concurrent.TimeUnit;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/5 6:41 PM
   */
  public class AdStatisticsByProvince {
  
    public static void main(String[] args) throws Exception {
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
  
      // 1. 从文件中读取数据
      URL resource = AdStatisticsByProvince.class.getResource("/AdClickLog.csv");
      DataStream<AdClickEvent> adClickEventDataStream = env.readTextFile(resource.getPath())
        .map(line -> {
          String[] fields = line.split(",");
          return new AdClickEvent(new Long(fields[0]), new Long(fields[1]), fields[2], fields[3], new Long(fields[4]));
        })
        .assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
          new BoundedOutOfOrdernessTimestampExtractor<AdClickEvent>(Time.of(200, TimeUnit.MILLISECONDS)) {
            @Override
            public long extractTimestamp(AdClickEvent element) {
              return element.getTimestamp() * 1000L;
            }
          }
        ));
  
      // 2. 基于省份分组，开窗聚合
      DataStream<AdCountViewByProvince> adCountStream = adClickEventDataStream
        .keyBy(AdClickEvent::getProvince)
        // 定义滑窗,5min输出一次
        .window(SlidingEventTimeWindows.of(Time.hours(1), Time.minutes(5)))
        .aggregate(new AdCountAgg(), new AdCountResult());
  
  
      adCountStream.print();
  
      env.execute("ad count by province job");
    }
  
    public static class AdCountAgg implements AggregateFunction<AdClickEvent, Long, Long> {
  
      @Override
      public Long createAccumulator() {
        return 0L;
      }
  
      @Override
      public Long add(AdClickEvent value, Long accumulator) {
        return accumulator + 1;
      }
  
      @Override
      public Long getResult(Long accumulator) {
        return accumulator;
      }
  
      @Override
      public Long merge(Long a, Long b) {
        return a + b;
      }
    }
  
    public static class AdCountResult implements WindowFunction<Long, AdCountViewByProvince, String, TimeWindow> {
  
      @Override
      public void apply(String province, TimeWindow window, Iterable<Long> input, Collector<AdCountViewByProvince> out) throws Exception {
        String windowEnd = new Timestamp(window.getEnd()).toString();
        Long count = input.iterator().next();
        out.collect(new AdCountViewByProvince(province, windowEnd, count));
      }
    }
  }
  
  ```

+ 输出

  ```shell
  beans.AdCountViewByProvince{province='beijing', windowEnd='2017-11-26 09:05:00.0', count=2}
  beans.AdCountViewByProvince{province='shanghai', windowEnd='2017-11-26 09:05:00.0', count=1}
  beans.AdCountViewByProvince{province='guangdong', windowEnd='2017-11-26 09:05:00.0', count=2}
  beans.AdCountViewByProvince{province='guangdong', windowEnd='2017-11-26 09:10:00.0', count=4}
  beans.AdCountViewByProvince{province='shanghai', windowEnd='2017-11-26 09:10:00.0', count=2}
  beans.AdCountViewByProvince{province='beijing', windowEnd='2017-11-26 09:10:00.0', count=2}
  beans.AdCountViewByProvince{province='shanghai', windowEnd='2017-11-26 09:15:00.0', count=2}
  beans.AdCountViewByProvince{province='beijing', windowEnd='2017-11-26 09:15:00.0', count=2}
  beans.AdCountViewByProvince{province='guangdong', windowEnd='2017-11-26 09:15:00.0', count=5}
  beans.AdCountViewByProvince{province='shanghai', windowEnd='2017-11-26 09:20:00.0', count=2}
  ....
  ```

#### 代码2-点击异常行为黑名单过滤

+ java代码

  ```java
  import beans.AdClickEvent;
  import beans.AdCountViewByProvince;
  import beans.BlackListUserWarning;
  import org.apache.commons.lang3.time.DateUtils;
  import org.apache.flink.api.common.functions.AggregateFunction;
  import org.apache.flink.api.common.state.ValueState;
  import org.apache.flink.api.common.state.ValueStateDescriptor;
  import org.apache.flink.api.java.functions.KeySelector;
  import org.apache.flink.api.java.tuple.Tuple2;
  import org.apache.flink.configuration.Configuration;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
  import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
  import org.apache.flink.streaming.api.functions.windowing.WindowFunction;
  import org.apache.flink.streaming.api.windowing.assigners.SlidingEventTimeWindows;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
  import org.apache.flink.streaming.runtime.operators.util.AssignerWithPeriodicWatermarksAdapter;
  import org.apache.flink.util.Collector;
  import org.apache.flink.util.OutputTag;
  
  import java.net.URL;
  import java.sql.Timestamp;
  import java.util.Date;
  import java.util.concurrent.TimeUnit;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/5 6:41 PM
   */
  public class AdStatisticsByProvince {
  
    public static void main(String[] args) throws Exception {
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
  
      // 1. 从文件中读取数据
      URL resource = AdStatisticsByProvince.class.getResource("/AdClickLog.csv");
      DataStream<AdClickEvent> adClickEventDataStream = env.readTextFile(resource.getPath())
        .map(line -> {
          String[] fields = line.split(",");
          return new AdClickEvent(new Long(fields[0]), new Long(fields[1]), fields[2], fields[3], new Long(fields[4]));
        })
        .assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
          new BoundedOutOfOrdernessTimestampExtractor<AdClickEvent>(Time.of(200, TimeUnit.MILLISECONDS)) {
            @Override
            public long extractTimestamp(AdClickEvent element) {
              return element.getTimestamp() * 1000L;
            }
          }
        ));
  
      // 2. 对同一个用户点击同一个广告的行为进行检测报警
      SingleOutputStreamOperator<AdClickEvent> filterAdClickStream = adClickEventDataStream
        .keyBy(new KeySelector<AdClickEvent, Tuple2<Long, Long>>() {
          @Override
          public Tuple2<Long, Long> getKey(AdClickEvent value) throws Exception {
            return new Tuple2<>(value.getUserId(), value.getAdId());
          }
        })
        .process(new FilterBlackListUser(100));
  
      // 3. 基于省份分组，开窗聚合
      DataStream<AdCountViewByProvince> adCountResultStream = filterAdClickStream
        .keyBy(AdClickEvent::getProvince)
        // 定义滑窗,5min输出一次
        .window(SlidingEventTimeWindows.of(Time.hours(1), Time.minutes(5)))
        .aggregate(new AdCountAgg(), new AdCountResult());
  
  
      adCountResultStream.print();
      filterAdClickStream
        .getSideOutput(new OutputTag<BlackListUserWarning>("blacklist"){})
        .print("blacklist-user");
  
      env.execute("ad count by province job");
    }
  
    public static class AdCountAgg implements AggregateFunction<AdClickEvent, Long, Long> {
  
      @Override
      public Long createAccumulator() {
        return 0L;
      }
  
      @Override
      public Long add(AdClickEvent value, Long accumulator) {
        return accumulator + 1;
      }
  
      @Override
      public Long getResult(Long accumulator) {
        return accumulator;
      }
  
      @Override
      public Long merge(Long a, Long b) {
        return a + b;
      }
    }
  
    public static class AdCountResult implements WindowFunction<Long, AdCountViewByProvince, String, TimeWindow> {
  
      @Override
      public void apply(String province, TimeWindow window, Iterable<Long> input, Collector<AdCountViewByProvince> out) throws Exception {
        String windowEnd = new Timestamp(window.getEnd()).toString();
        Long count = input.iterator().next();
        out.collect(new AdCountViewByProvince(province, windowEnd, count));
      }
    }
  
    // 实现自定义处理函数
    public static class FilterBlackListUser extends KeyedProcessFunction<Tuple2<Long, Long>, AdClickEvent, AdClickEvent> {
  
      // 定义属性：点击次数上线
      private Integer countUpperBound;
  
      public FilterBlackListUser(Integer countUpperBound) {
        this.countUpperBound = countUpperBound;
      }
  
      // 定义状态，保存当前用户对某一广告的点击次数
      ValueState<Long> countState;
      // 定义一个标志状态，保存当前用户是否已经被发送到了黑名单里
      ValueState<Boolean> isSentState;
  
      @Override
      public void open(Configuration parameters) throws Exception {
        countState = getRuntimeContext().getState(new ValueStateDescriptor<Long>("ad-count", Long.class));
        isSentState = getRuntimeContext().getState(new ValueStateDescriptor<Boolean>("is-sent", Boolean.class));
      }
  
      @Override
      public void onTimer(long timestamp, OnTimerContext ctx, Collector<AdClickEvent> out) throws Exception {
        // 清空所有状态
        countState.clear();
        isSentState.clear();
      }
  
      @Override
      public void processElement(AdClickEvent value, Context ctx, Collector<AdClickEvent> out) throws Exception {
        // 判断当前用户对同一广告的点击次数，如果不够上限，该count加1正常输出；
        // 如果到达上限，直接过滤掉，并侧输出流输出黑名单报警
  
        // 首先获取当前count值
        Long curCount = countState.value();
  
        Boolean isSent = isSentState.value();
  
        if(null == curCount){
          curCount = 0L;
        }
  
        if(null == isSent){
          isSent = false;
        }
  
        // 1. 判断是否是第一个数据，如果是的话，注册一个第二天0点的定时器
        if (curCount == 0) {
          long ts = ctx.timerService().currentProcessingTime();
          long fixedTime = DateUtils.addDays(new Date(ts), 1).getTime();
          ctx.timerService().registerProcessingTimeTimer(fixedTime);
        }
  
        // 2. 判断是否报警
        if (curCount >= countUpperBound) {
          // 判断是否输出到黑名单过，如果没有的话就输出到侧输出流
          if (!isSent) {
            isSentState.update(true);
            ctx.output(new OutputTag<BlackListUserWarning>("blacklist"){},
                       new BlackListUserWarning(value.getUserId(), value.getAdId(), "click over " + countUpperBound + "times."));
          }
          // 不再进行下面操作
          return;
        }
  
        // 如果没有返回，点击次数加1，更新状态，正常输出当前数据到主流
        countState.update(curCount + 1);
        out.collect(value);
      }
  
    }
  }
  ```

+ 输出

  ```java
  blacklist-user> beans.BlackListUserWarning{userId=937166, adId=1715, warningMsg='click over 100times.'}
  beans.AdCountViewByProvince{province='beijing', windowEnd='2017-11-26 09:05:00.0', count=2}
  beans.AdCountViewByProvince{province='shanghai', windowEnd='2017-11-26 09:05:00.0', count=1}
  beans.AdCountViewByProvince{province='guangdong', windowEnd='2017-11-26 09:05:00.0', count=2}
  beans.AdCountViewByProvince{province='guangdong', windowEnd='2017-11-26 09:10:00.0', count=4}
  beans.AdCountViewByProvince{province='shanghai', windowEnd='2017-11-26 09:10:00.0', count=2}
  beans.AdCountViewByProvince{province='beijing', windowEnd='2017-11-26 09:10:00.0', count=2}
  beans.AdCountViewByProvince{province='shanghai', windowEnd='2017-11-26 09:15:00.0', count=2}
  beans.AdCountViewByProvince{province='beijing', windowEnd='2017-11-26 09:15:00.0', count=2}
  beans.AdCountViewByProvince{province='guangdong', windowEnd='2017-11-26 09:15:00.0', count=5}
  beans.AdCountViewByProvince{province='shanghai', windowEnd='2017-11-26 09:20:00.0', count=2}
  beans.AdCountViewByProvince{province='guangdong', windowEnd='2017-11-26 09:20:00.0', count=5}
  ....
  ```

### 14.3.6 恶意登录监控

+ 基本需求
  + 用户在短时间内频繁登录失败，有程序恶意攻击的可能
  + 同一用户（可以是不同IP）在2秒内连续两次登录失败，需要报警
+ 解决思路
  + 将用户的登录失败行为存入ListState，设定定时器2秒后出发，查看ListState中有几次失败登录
  + 更加精确的检测，可以使用CEP库实现事件流的模式匹配

#### POJO

+ LoginEvent

  ```java
  private Long userId;
  private String ip;
  private String loginState;
  private Long timestamp;
  ```

+ LoginFailWarning

  ```java
  private Long userId;
  private Long firstFailTime;
  private Long lastFailTime;
  private String warningMsg;
  ```

#### 代码1-简单代码实现

+ java代码

  ```java
  import beans.LoginEvent;
  import beans.LoginFailWarning;
  import org.apache.commons.compress.utils.Lists;
  import org.apache.flink.api.common.state.ListState;
  import org.apache.flink.api.common.state.ListStateDescriptor;
  import org.apache.flink.api.common.state.ValueState;
  import org.apache.flink.api.common.state.ValueStateDescriptor;
  import org.apache.flink.configuration.Configuration;
  import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
  import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.streaming.runtime.operators.util.AssignerWithPeriodicWatermarksAdapter;
  import org.apache.flink.util.Collector;
  
  import java.net.URL;
  import java.util.ArrayList;
  import java.util.concurrent.TimeUnit;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/6 1:49 AM
   */
  public class LoginFail {
    public static void main(String[] args) throws Exception {
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
  
      // 1. 从文件中读取数据
      URL resource = LoginFail.class.getResource("/LoginLog.csv");
      SingleOutputStreamOperator<LoginEvent> loginEventStream = env.readTextFile(resource.getPath())
        .map(line -> {
          String[] fields = line.split(",");
          return new LoginEvent(new Long(fields[0]), fields[1], fields[2], new Long(fields[3]));
        }).assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
        new BoundedOutOfOrdernessTimestampExtractor<LoginEvent>(Time.of(3, TimeUnit.SECONDS)) {
          @Override
          public long extractTimestamp(LoginEvent element) {
            return element.getTimestamp() * 1000L;
          }
        }
      ));
      // 自定义处理函数检测连续登录失败事件
      SingleOutputStreamOperator<LoginFailWarning> warningStream = loginEventStream
        .keyBy(LoginEvent::getUserId)
        .process(new LoginFailDetectWarning(1));
  
      warningStream.print();
  
      env.execute("login fail detect job");
    }
  
    // 实现自定义KeyedProcessFunction
    public static class LoginFailDetectWarning extends KeyedProcessFunction<Long, LoginEvent, LoginFailWarning> {
      // 定义属性，最大连续登录失败次数
      private Integer maxFailTimes;
  
      // 定义状态：保存2秒内所有的登录失败事件
      ListState<LoginEvent> loginFailEventListState;
      // 定义状态：保存注册的定时器时间戳
      ValueState<Long> timerTsState;
  
      public LoginFailDetectWarning(Integer maxFailTimes) {
        this.maxFailTimes = maxFailTimes;
      }
  
      @Override
      public void open(Configuration parameters) throws Exception {
        loginFailEventListState = getRuntimeContext().getListState(new ListStateDescriptor<LoginEvent>("login-fail-list", LoginEvent.class));
        timerTsState = getRuntimeContext().getState(new ValueStateDescriptor<Long>("timer-ts", Long.class));
      }
  
      @Override
      public void onTimer(long timestamp, OnTimerContext ctx, Collector<LoginFailWarning> out) throws Exception {
        // 定时器触发，说明2秒内没有登录成功，判读ListState中失败的个数
        ArrayList<LoginEvent> loginFailEvents = Lists.newArrayList(loginFailEventListState.get().iterator());
        int failTimes = loginFailEvents.size();
  
        if (failTimes >= maxFailTimes) {
          // 如果超出设定的最大失败次数，输出报警
          out.collect(new LoginFailWarning(ctx.getCurrentKey(),
                                           loginFailEvents.get(0).getTimestamp(),
                                           loginFailEvents.get(failTimes - 1).getTimestamp(),
                                           "login fail in 2s for " + failTimes + " times"));
        }
  
        // 清空状态
        loginFailEventListState.clear();
        timerTsState.clear();
      }
  
      @Override
      public void processElement(LoginEvent value, Context ctx, Collector<LoginFailWarning> out) throws Exception {
        // 判断当前登录事件类型
        if ("fail".equals(value.getLoginState())) {
          // 1. 如果是失败事件，添加到表状态中
          loginFailEventListState.add(value);
          // 如果没有定时器，注册一个2秒之后的定时器
          if (null == timerTsState.value()) {
            long ts = (value.getTimestamp() + 2) * 1000L;
            ctx.timerService().registerEventTimeTimer(ts);
            timerTsState.update(ts);
          } else {
            // 2. 如果是登录成功，删除定时器，清空状态，重新开始
            if (null != timerTsState.value()) {
              ctx.timerService().deleteEventTimeTimer(timerTsState.value());
            }
            loginFailEventListState.clear();
            timerTsState.clear();
          }
        }
      }
    }
  }
  ```

+ 输出

  ```shell
  LoginFailWarning{userId=23064, firstFailTime=1558430826, lastFailTime=1558430826, warningMsg='login fail in 2s for 1 times'}
  LoginFailWarning{userId=5692, firstFailTime=1558430833, lastFailTime=1558430833, warningMsg='login fail in 2s for 1 times'}
  LoginFailWarning{userId=1035, firstFailTime=1558430844, lastFailTime=1558430844, warningMsg='login fail in 2s for 1 times'}
  LoginFailWarning{userId=76456, firstFailTime=1558430859, lastFailTime=1558430859, warningMsg='login fail in 2s for 1 times'}
  LoginFailWarning{userId=23565, firstFailTime=1558430862, lastFailTime=1558430862, warningMsg='login fail in 2s for 1 times'}
  ```

#### 代码2-代码实效性改进

+ java代码

  ```java
  import beans.LoginEvent;
  import beans.LoginFailWarning;
  import org.apache.commons.compress.utils.Lists;
  import org.apache.flink.api.common.state.ListState;
  import org.apache.flink.api.common.state.ListStateDescriptor;
  import org.apache.flink.api.common.state.ValueState;
  import org.apache.flink.api.common.state.ValueStateDescriptor;
  import org.apache.flink.configuration.Configuration;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
  import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.streaming.runtime.operators.util.AssignerWithPeriodicWatermarksAdapter;
  import org.apache.flink.util.Collector;
  
  import java.net.URL;
  import java.util.ArrayList;
  import java.util.Iterator;
  import java.util.concurrent.TimeUnit;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/6 1:49 AM
   */
  public class LoginFail {
    public static void main(String[] args) throws Exception{
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
  
      // 1. 从文件中读取数据
      URL resource = LoginFail.class.getResource("/LoginLog.csv");
      DataStream<LoginEvent> loginEventStream = env.readTextFile(resource.getPath())
        .map(line -> {
          String[] fields = line.split(",");
          return new LoginEvent(new Long(fields[0]), fields[1], fields[2], new Long(fields[3]));
        })
        .assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
          new BoundedOutOfOrdernessTimestampExtractor<LoginEvent>(Time.of(200, TimeUnit.MILLISECONDS)) {
            @Override
            public long extractTimestamp(LoginEvent element) {
              return element.getTimestamp() * 1000L;
            }
          }
        ));
  
      // 自定义处理函数检测连续登录失败事件
      SingleOutputStreamOperator<LoginFailWarning> warningStream = loginEventStream
        .keyBy(LoginEvent::getUserId)
        .process(new LoginFailDetectWarning(2));
  
      warningStream.print();
  
      env.execute("login fail detect job");
    }
  
    // 实现自定义KeyedProcessFunction
    public static class LoginFailDetectWarning0 extends KeyedProcessFunction<Long, LoginEvent, LoginFailWarning>{
      // 定义属性，最大连续登录失败次数
      private Integer maxFailTimes;
  
      public LoginFailDetectWarning0(Integer maxFailTimes) {
        this.maxFailTimes = maxFailTimes;
      }
  
      // 定义状态：保存2秒内所有的登录失败事件
      ListState<LoginEvent> loginFailEventListState;
      // 定义状态：保存注册的定时器时间戳
      ValueState<Long> timerTsState;
  
      @Override
      public void open(Configuration parameters) throws Exception {
        loginFailEventListState = getRuntimeContext().getListState(new ListStateDescriptor<LoginEvent>("login-fail-list", LoginEvent.class));
        timerTsState = getRuntimeContext().getState(new ValueStateDescriptor<Long>("timer-ts", Long.class));
      }
  
      @Override
      public void processElement(LoginEvent value, Context ctx, Collector<LoginFailWarning> out) throws Exception {
        // 判断当前登录事件类型
        if( "fail".equals(value.getLoginState()) ){
          // 1. 如果是失败事件，添加到列表状态中
          loginFailEventListState.add(value);
          // 如果没有定时器，注册一个2秒之后的定时器
          if( timerTsState.value() == null ){
            Long ts = (value.getTimestamp() + 2) * 1000L;
            ctx.timerService().registerEventTimeTimer(ts);
            timerTsState.update(ts);
          }
        } else {
          // 2. 如果是登录成功，删除定时器，清空状态，重新开始
          if( timerTsState.value() != null )
            ctx.timerService().deleteEventTimeTimer(timerTsState.value());
          loginFailEventListState.clear();
          timerTsState.clear();
        }
      }
  
      @Override
      public void onTimer(long timestamp, OnTimerContext ctx, Collector<LoginFailWarning> out) throws Exception {
        // 定时器触发，说明2秒内没有登录成功来，判断ListState中失败的个数
        ArrayList<LoginEvent> loginFailEvents = Lists.newArrayList(loginFailEventListState.get().iterator());
        Integer failTimes = loginFailEvents.size();
  
        if( failTimes >= maxFailTimes ){
          // 如果超出设定的最大失败次数，输出报警
          out.collect( new LoginFailWarning(ctx.getCurrentKey(),
                                            loginFailEvents.get(0).getTimestamp(),
                                            loginFailEvents.get(failTimes - 1).getTimestamp(),
                                            "login fail in 2s for " + failTimes + " times") );
        }
  
        // 清空状态
        loginFailEventListState.clear();
        timerTsState.clear();
      }
    }
  
    // 实现自定义KeyedProcessFunction
    public static class LoginFailDetectWarning extends KeyedProcessFunction<Long, LoginEvent, LoginFailWarning> {
      // 定义属性，最大连续登录失败次数
      private Integer maxFailTimes;
  
      public LoginFailDetectWarning(Integer maxFailTimes) {
        this.maxFailTimes = maxFailTimes;
      }
  
      // 定义状态：保存2秒内所有的登录失败事件
      ListState<LoginEvent> loginFailEventListState;
  
      @Override
      public void open(Configuration parameters) throws Exception {
        loginFailEventListState = getRuntimeContext().getListState(new ListStateDescriptor<LoginEvent>("login-fail-list", LoginEvent.class));
      }
  
      // 以登录事件作为判断报警的触发条件，不再注册定时器
      @Override
      public void processElement(LoginEvent value, Context ctx, Collector<LoginFailWarning> out) throws Exception {
        // 判断当前事件登录状态
        if( "fail".equals(value.getLoginState()) ){
          // 1. 如果是登录失败，获取状态中之前的登录失败事件，继续判断是否已有失败事件
          Iterator<LoginEvent> iterator = loginFailEventListState.get().iterator();
          if( iterator.hasNext() ){
            // 1.1 如果已经有登录失败事件，继续判断时间戳是否在2秒之内
            // 获取已有的登录失败事件
            LoginEvent firstFailEvent = iterator.next();
            if( value.getTimestamp() - firstFailEvent.getTimestamp() <= 2 ){
              // 1.1.1 如果在2秒之内，输出报警
              out.collect( new LoginFailWarning(value.getUserId(), firstFailEvent.getTimestamp(), value.getTimestamp(), "login fail 2 times in 2s") );
            }
  
            // 不管报不报警，这次都已处理完毕，直接更新状态
            loginFailEventListState.clear();
            loginFailEventListState.add(value);
          } else {
            // 1.2 如果没有登录失败，直接将当前事件存入ListState
            loginFailEventListState.add(value);
          }
        } else {
          // 2. 如果是登录成功，直接清空状态
          loginFailEventListState.clear();
        }
      }
    }
  }
  ```

+ 输出

  ```shell
  LoginFailWarning{userId=1035, firstFailTime=1558430842, lastFailTime=1558430843, warningMsg='login fail 2 times in 2s'}
  LoginFailWarning{userId=1035, firstFailTime=1558430843, lastFailTime=1558430844, warningMsg='login fail 2 times in 2s'}
  ```

#### 代码3-CEP代码实现

+ pom依赖

  CEP编程

  ```xml
  <dependencies>
    <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-cep_${scala.binary.version}</artifactId>
      <version>${flink.version}</version>
    </dependency>
  </dependencies>
  ```

+ java代码

  ```java
  import beans.LoginEvent;
  import beans.LoginFailWarning;
  import org.apache.flink.cep.CEP;
  import org.apache.flink.cep.PatternSelectFunction;
  import org.apache.flink.cep.PatternStream;
  import org.apache.flink.cep.pattern.Pattern;
  import org.apache.flink.cep.pattern.conditions.SimpleCondition;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.streaming.runtime.operators.util.AssignerWithPeriodicWatermarksAdapter;
  
  import java.net.URL;
  import java.util.List;
  import java.util.Map;
  import java.util.concurrent.TimeUnit;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/6 3:41 AM
   */
  public class LoginFailWithCep {
  
    public static void main(String[] args) throws Exception {
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
  
      // 1. 从文件中读取数据
      URL resource = LoginFail.class.getResource("/LoginLog.csv");
      DataStream<LoginEvent> loginEventStream = env.readTextFile(resource.getPath())
        .map(line -> {
          String[] fields = line.split(",");
          return new LoginEvent(new Long(fields[0]), fields[1], fields[2], new Long(fields[3]));
        })
        .assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
          new BoundedOutOfOrdernessTimestampExtractor<LoginEvent>(Time.of(200, TimeUnit.MILLISECONDS)) {
            @Override
            public long extractTimestamp(LoginEvent element) {
              return element.getTimestamp() * 1000L;
            }
          }
        ));
  
      // 1. 定义一个匹配模式
      // firstFail -> secondFail, within 2s
      Pattern<LoginEvent, LoginEvent> loginFailPattern = Pattern
        .<LoginEvent>begin("firstFail").where(new SimpleCondition<LoginEvent>() {
        @Override
        public boolean filter(LoginEvent value) throws Exception {
          return "fail".equals(value.getLoginState());
        }
      })
        .next("secondFail").where(new SimpleCondition<LoginEvent>() {
        @Override
        public boolean filter(LoginEvent value) throws Exception {
          return "fail".equals(value.getLoginState());
        }
      })
        .within(Time.seconds(2));
  
      // 2. 将匹配模式应用到数据流上，得到一个pattern stream
      PatternStream<LoginEvent> patternStream = CEP.pattern(loginEventStream.keyBy(LoginEvent::getUserId), loginFailPattern);
  
      // 3. 检出符合匹配条件的复杂事件，进行转换处理，得到报警信息
      SingleOutputStreamOperator<LoginFailWarning> warningStream = patternStream.select(new LoginFailMatchDetectWarning());
  
      warningStream.print();
  
      env.execute("login fail detect with cep job");
    }
  
    // 实现自定义的PatternSelectFunction
    public static class LoginFailMatchDetectWarning implements PatternSelectFunction<LoginEvent, LoginFailWarning> {
      @Override
      public LoginFailWarning select(Map<String, List<LoginEvent>> pattern) throws Exception {
        LoginEvent firstFailEvent = pattern.get("firstFail").iterator().next();
        LoginEvent lastFailEvent = pattern.get("secondFail").get(0);
        return new LoginFailWarning(firstFailEvent.getUserId(), firstFailEvent.getTimestamp(), lastFailEvent.getTimestamp(), "login fail 2 times");
      }
    }
  }
  
  ```

+ 输出

  ```java
  LoginFailWarning{userId=1035, firstFailTime=1558430842, lastFailTime=1558430843, warningMsg='login fail 2 times'}
  LoginFailWarning{userId=1035, firstFailTime=1558430843, lastFailTime=1558430844, warningMsg='login fail 2 times'}
  ```

#### 代码4-CEP利用循环模式优化

+ java代码

  ```java
  import beans.LoginEvent;
  import beans.LoginFailWarning;
  import org.apache.flink.cep.CEP;
  import org.apache.flink.cep.PatternSelectFunction;
  import org.apache.flink.cep.PatternStream;
  import org.apache.flink.cep.pattern.Pattern;
  import org.apache.flink.cep.pattern.conditions.SimpleCondition;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.streaming.runtime.operators.util.AssignerWithPeriodicWatermarksAdapter;
  
  import java.net.URL;
  import java.util.List;
  import java.util.Map;
  import java.util.concurrent.TimeUnit;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/6 3:41 AM
   */
  public class LoginFailWithCep {
  
    public static void main(String[] args) throws Exception {
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
  
      // 1. 从文件中读取数据
      URL resource = LoginFail.class.getResource("/LoginLog.csv");
      DataStream<LoginEvent> loginEventStream = env.readTextFile(resource.getPath())
        .map(line -> {
          String[] fields = line.split(",");
          return new LoginEvent(new Long(fields[0]), fields[1], fields[2], new Long(fields[3]));
        })
        .assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
          new BoundedOutOfOrdernessTimestampExtractor<LoginEvent>(Time.of(200, TimeUnit.MILLISECONDS)) {
            @Override
            public long extractTimestamp(LoginEvent element) {
              return element.getTimestamp() * 1000L;
            }
          }
        ));
  
      // 1. 定义一个匹配模式
      // firstFail -> secondFail, within 2s
      Pattern<LoginEvent, LoginEvent> loginFailPattern = Pattern
        .<LoginEvent>begin("failEvents").where(new SimpleCondition<LoginEvent>() {
        @Override
        public boolean filter(LoginEvent value) throws Exception {
          return "fail".equals(value.getLoginState());
        }
      }).times(3).consecutive()
        .within(Time.seconds(5));
  
      // 2. 将匹配模式应用到数据流上，得到一个pattern stream
      PatternStream<LoginEvent> patternStream = CEP.pattern(loginEventStream.keyBy(LoginEvent::getUserId), loginFailPattern);
  
      // 3. 检出符合匹配条件的复杂事件，进行转换处理，得到报警信息
      SingleOutputStreamOperator<LoginFailWarning> warningStream = patternStream.select(new LoginFailMatchDetectWarning());
  
      warningStream.print();
  
      env.execute("login fail detect with cep job");
    }
  
    // 实现自定义的PatternSelectFunction
    public static class LoginFailMatchDetectWarning implements PatternSelectFunction<LoginEvent, LoginFailWarning> {
      @Override
      public LoginFailWarning select(Map<String, List<LoginEvent>> pattern) throws Exception {
        LoginEvent firstFailEvent = pattern.get("failEvents").get(0);
        LoginEvent lastFailEvent = pattern.get("failEvents").get(pattern.get("failEvents").size() - 1);
        return new LoginFailWarning(firstFailEvent.getUserId(), firstFailEvent.getTimestamp(), lastFailEvent.getTimestamp(), "login fail " + pattern.get("failEvents").size() + " times");
      }
    }
  }
  ```

+ 输出

  ```shell
  LoginFailWarning{userId=1035, firstFailTime=1558430842, lastFailTime=1558430844, warningMsg='login fail 3 times'}
  ```

### 14.3.7 订单支付实时监控

+ 基本需求
  + 用户下单之后，应设置订单失效事件，以提高用户支付的意愿，并降低系统风险
  + 用户下单后15分钟未支付，则输出监控信息
+ 解决思路
  + 利用CEP库进行事件流的模式匹配，并设定匹配的时间间隔
  + 也可以利用状态编程，用process function实现处理逻辑

#### POJO

+ OrderEvent

  ```java
  private Long orderId;
  private String eventType;
  private String txId;
  private Long timestamp;
  ```

+ OrderResult

  ```java
  private Long orderId;
  private String resultState;
  ```

+ ReceiptEvent

  ```java
  private String txId;
  private String payChannel;
  private Long timestamp;
  ```

#### 代码1-CEP代码实现

+ pom依赖

  ```java
  <dependencies>
    <dependency>
  <groupId>org.apache.flink</groupId>
    <artifactId>flink-cep_${scala.binary.version}</artifactId>
    <version>${flink.version}</version>
    </dependency>
    </dependencies>
  ```
  
+ java代码

  （实际如果处理超时订单，应该修改对应的数据库数据，好让下次用户再次操作超时订单时失效）

  ```java
  import beans.OrderEvent;
  import beans.OrderResult;
  import org.apache.flink.cep.CEP;
  import org.apache.flink.cep.PatternSelectFunction;
  import org.apache.flink.cep.PatternStream;
  import org.apache.flink.cep.PatternTimeoutFunction;
  import org.apache.flink.cep.pattern.Pattern;
  import org.apache.flink.cep.pattern.conditions.SimpleCondition;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.streaming.runtime.operators.util.AssignerWithPeriodicWatermarksAdapter;
  import org.apache.flink.util.OutputTag;
  
  import java.net.URL;
  import java.util.List;
  import java.util.Map;
  import java.util.concurrent.TimeUnit;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/6 5:50 AM
   */
  public class OrderPayTimeout {
    public static void main(String[] args) throws Exception {
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
  
      // 读取数据并转换成POJO类型
      URL resource = OrderPayTimeout.class.getResource("/OrderLog.csv");
      DataStream<OrderEvent> orderEventDataStream = env.readTextFile(resource.getPath())
        .map(line -> {
          String[] fields = line.split(",");
          return new OrderEvent(new Long(fields[0]), fields[1], fields[2], new Long(fields[3]));
        }).assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
        new BoundedOutOfOrdernessTimestampExtractor<OrderEvent>(Time.of(200, TimeUnit.MILLISECONDS)) {
          @Override
          public long extractTimestamp(OrderEvent element) {
            return element.getTimestamp() * 1000L;
          }
        }
      ));
  
      // 1. 定义一个待时间限制的模式
      Pattern<OrderEvent, OrderEvent> orderPayPattern = Pattern.<OrderEvent>begin("create").where(new SimpleCondition<OrderEvent>() {
        @Override
        public boolean filter(OrderEvent value) throws Exception {
          return "create".equals(value.getEventType());
        }
      })
        .followedBy("pay").where(new SimpleCondition<OrderEvent>() {
        @Override
        public boolean filter(OrderEvent value) throws Exception {
          return "pay".equals(value.getEventType());
        }
      })
        .within(Time.minutes(5));
  
      // 2. 定义侧输出流标签，用来表示超时事件
      OutputTag<OrderResult> orderTimeoutTag = new OutputTag<OrderResult>("order-timeout") {
      };
  
      // 3. 将pattern应用到输入数据上，得到pattern stream
      PatternStream<OrderEvent> patternStream = CEP.pattern(orderEventDataStream.keyBy(OrderEvent::getOrderId), orderPayPattern);
  
      // 4. 调用select方法，实现对匹配复杂事件和超时复杂事件的提取和处理
      SingleOutputStreamOperator<OrderResult> resultStream = patternStream
        .select(orderTimeoutTag, new OrderTimeoutSelect(), new OrderPaySelect());
  
      resultStream.print("payed normally");
      resultStream.getSideOutput(orderTimeoutTag).print("timeout");
  
      env.execute("order timeout detect job");
  
    }
  
    // 实现自定义的超时事件处理函数
    public static class OrderTimeoutSelect implements PatternTimeoutFunction<OrderEvent, OrderResult> {
  
      @Override
      public OrderResult timeout(Map<String, List<OrderEvent>> pattern, long timeoutTimestamp) throws Exception {
        Long timeoutOrderId = pattern.get("create").iterator().next().getOrderId();
        return new OrderResult(timeoutOrderId, "timeout " + timeoutTimestamp);
      }
    }
  
    // 实现自定义的正常匹配事件处理函数
    public static class OrderPaySelect implements PatternSelectFunction<OrderEvent, OrderResult> {
      @Override
      public OrderResult select(Map<String, List<OrderEvent>> pattern) throws Exception {
        Long payedOrderId = pattern.get("pay").iterator().next().getOrderId();
        return new OrderResult(payedOrderId, "payed");
      }
    }
  }
  ```

+ 输出

  ```shell
  payed normally> OrderResult{orderId=34729, resultState='payed'}
  timeout> OrderResult{orderId=34767, resultState='timeout 1558431249000'}
  payed normally> OrderResult{orderId=34766, resultState='payed'}
  payed normally> OrderResult{orderId=34765, resultState='payed'}
  payed normally> OrderResult{orderId=34764, resultState='payed'}
  payed normally> OrderResult{orderId=34763, resultState='payed'}
  payed normally> OrderResult{orderId=34762, resultState='payed'}
  payed normally> OrderResult{orderId=34761, resultState='payed'}
  payed normally> OrderResult{orderId=34760, resultState='payed'}
  payed normally> OrderResult{orderId=34759, resultState='payed'}
  payed normally> OrderResult{orderId=34758, resultState='payed'}
  payed normally> OrderResult{orderId=34757, resultState='payed'}
  timeout> OrderResult{orderId=34756, resultState='timeout 1558431213000'}
  payed normally> OrderResult{orderId=34755, resultState='payed'}
  payed normally> OrderResult{orderId=34754, resultState='payed'}
  payed normally> OrderResult{orderId=34753, resultState='payed'}
  payed normally> OrderResult{orderId=34752, resultState='payed'}
  payed normally> OrderResult{orderId=34751, resultState='payed'}
  payed normally> OrderResult{orderId=34750, resultState='payed'}
  payed normally> OrderResult{orderId=34749, resultState='payed'}
  payed normally> OrderResult{orderId=34748, resultState='payed'}
  payed normally> OrderResult{orderId=34747, resultState='payed'}
  payed normally> OrderResult{orderId=34746, resultState='payed'}
  payed normally> OrderResult{orderId=34745, resultState='payed'}
  payed normally> OrderResult{orderId=34744, resultState='payed'}
  payed normally> OrderResult{orderId=34743, resultState='payed'}
  payed normally> OrderResult{orderId=34742, resultState='payed'}
  payed normally> OrderResult{orderId=34741, resultState='payed'}
  payed normally> OrderResult{orderId=34740, resultState='payed'}
  payed normally> OrderResult{orderId=34739, resultState='payed'}
  payed normally> OrderResult{orderId=34738, resultState='payed'}
  payed normally> OrderResult{orderId=34737, resultState='payed'}
  payed normally> OrderResult{orderId=34736, resultState='payed'}
  payed normally> OrderResult{orderId=34735, resultState='payed'}
  payed normally> OrderResult{orderId=34734, resultState='payed'}
  payed normally> OrderResult{orderId=34733, resultState='payed'}
  payed normally> OrderResult{orderId=34732, resultState='payed'}
  payed normally> OrderResult{orderId=34731, resultState='payed'}
  payed normally> OrderResult{orderId=34730, resultState='payed'}
  ```

#### 代码2-ProcessFunction实现

CEP虽然更加简洁，但是ProcessFunction能控制的细节操作更多。

CEP还是比较适合事件之间有复杂联系的场景；

ProcessFunction用来处理每个独立且靠状态就能联系的事件，灵活性更高。

+ java代码

  ```java
  import beans.OrderEvent;
  import beans.OrderResult;
  import org.apache.flink.api.common.state.ValueState;
  import org.apache.flink.api.common.state.ValueStateDescriptor;
  import org.apache.flink.configuration.Configuration;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
  import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.streaming.runtime.operators.util.AssignerWithPeriodicWatermarksAdapter;
  import org.apache.flink.util.Collector;
  import org.apache.flink.util.OutputTag;
  
  import java.net.URL;
  import java.util.concurrent.TimeUnit;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/6 4:59 PM
   */
  public class OrderTimeoutWithoutCep {
  
    // 定义超时事件的侧输出流标签
    private final static OutputTag<OrderResult> orderTimeoutTag = new OutputTag<OrderResult>("order-timeout") {
    };
  
    public static void main(String[] args) throws Exception {
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
  
      // 读取数据并转换成POJO类型
      URL resource = OrderPayTimeout.class.getResource("/OrderLog.csv");
      DataStream<OrderEvent> orderEventDataStream = env.readTextFile(resource.getPath())
        .map(line -> {
          String[] fields = line.split(",");
          return new OrderEvent(new Long(fields[0]), fields[1], fields[2], new Long(fields[3]));
        }).assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
        new BoundedOutOfOrdernessTimestampExtractor<OrderEvent>(Time.of(200, TimeUnit.MILLISECONDS)) {
          @Override
          public long extractTimestamp(OrderEvent element) {
            return element.getTimestamp() * 1000L;
          }
        }
      ));
  
      // 定义自定义处理函数，主流输出正常匹配订单事件，侧输出流输出超时报警事件
      SingleOutputStreamOperator<OrderResult> resultStream = orderEventDataStream.keyBy(OrderEvent::getOrderId)
        .process(new OrderPayMatchDetect());
  
      resultStream.print("pay normally");
      resultStream.getSideOutput(orderTimeoutTag).print("timeout");
  
      env.execute("order timeout detect without cep job");
    }
  
    public static class OrderPayMatchDetect extends KeyedProcessFunction<Long, OrderEvent, OrderResult> {
      // 定义状态，保存之前点单是否已经来过create、pay的事件
      ValueState<Boolean> isPayedState;
      ValueState<Boolean> isCreatedState;
      // 定义状态，保存定时器时间戳
      ValueState<Long> timerTsState;
  
      @Override
      public void open(Configuration parameters) throws Exception {
        isPayedState = getRuntimeContext().getState(new ValueStateDescriptor<Boolean>("is-payed", Boolean.class, false));
        isCreatedState = getRuntimeContext().getState(new ValueStateDescriptor<Boolean>("is-created", Boolean.class, false));
        timerTsState = getRuntimeContext().getState(new ValueStateDescriptor<Long>("timer-ts", Long.class));
      }
  
      @Override
      public void processElement(OrderEvent value, Context ctx, Collector<OrderResult> out) throws Exception {
        // 先获取当前状态
        Boolean isPayed = isPayedState.value();
        Boolean isCreated = isCreatedState.value();
        Long timerTs = timerTsState.value();
  
        // 判断当前事件类型
        if ("create".equals(value.getEventType())) {
          // 1. 如果来的是create，要判断是否支付过
          if (isPayed) {
            // 1.1 如果已经正常支付，输出正常匹配结果
            out.collect(new OrderResult(value.getOrderId(), "payed successfully"));
            // 清空状态，删除定时器
            isCreatedState.clear();
            isPayedState.clear();
            timerTsState.clear();
            ctx.timerService().deleteEventTimeTimer(timerTs);
          } else {
            // 1.2 如果没有支付过，注册15分钟后的定时器，开始等待支付事件
            Long ts = (value.getTimestamp() + 15 * 60) * 1000L;
            ctx.timerService().registerEventTimeTimer(ts);
            // 更新状态
            timerTsState.update(ts);
            isCreatedState.update(true);
          }
        } else if ("pay".equals(value.getEventType())) {
          // 2. 如果来的是pay，要判断是否有下单事件来过
          if (isCreated) {
            // 2.1 已经有过下单事件，要继续判断支付的时间戳是否超过15分钟
            if (value.getTimestamp() * 1000L < timerTs) {
              // 2.1.1 在15分钟内，没有超时，正常匹配输出
              out.collect(new OrderResult(value.getOrderId(), "payed successfully"));
            } else {
              // 2.1.2 已经超时，输出侧输出流报警
              ctx.output(orderTimeoutTag, new OrderResult(value.getOrderId(), "payed but already timeout"));
            }
            // 统一清空状态
            isCreatedState.clear();
            isPayedState.clear();
            timerTsState.clear();
            ctx.timerService().deleteEventTimeTimer(timerTs);
          } else {
            // 2.2 没有下单事件，乱序，注册一个定时器，等待下单事件
            ctx.timerService().registerEventTimeTimer(value.getTimestamp() * 1000L);
            // 更新状态
            timerTsState.update(value.getTimestamp() * 1000L);
            isPayedState.update(true);
          }
        }
      }
  
      @Override
      public void onTimer(long timestamp, OnTimerContext ctx, Collector<OrderResult> out) throws Exception {
        // 定时器触发，说明一定有一个事件没来
        if (isPayedState.value()) {
          // 如果pay来了，说明create没来
          ctx.output(orderTimeoutTag, new OrderResult(ctx.getCurrentKey(), "payed but not found created log"));
        } else {
          // 如果pay没来，支付超时
          ctx.output(orderTimeoutTag, new OrderResult(ctx.getCurrentKey(), "timeout"));
        }
        // 清空状态
        isCreatedState.clear();
        isPayedState.clear();
        timerTsState.clear();
      }
    }
  }
  
  ```

+ 输出

  ```shell
  pay normally> OrderResult{orderId=34729, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34730, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34731, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34732, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34734, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34733, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34735, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34736, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34746, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34738, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34745, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34741, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34747, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34743, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34737, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34744, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34742, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34739, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34740, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34753, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34749, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34755, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34752, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34748, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34751, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34750, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34761, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34759, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34754, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34758, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34760, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34757, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34762, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34763, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34764, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34765, resultState='payed successfully'}
  pay normally> OrderResult{orderId=34766, resultState='payed successfully'}
  timeout> OrderResult{orderId=34767, resultState='payed but already timeout'}
  timeout> OrderResult{orderId=34768, resultState='payed but not found created log'}
  timeout> OrderResult{orderId=34756, resultState='timeout'}
  ```

### 14.3.8 订单支付实时对帐

+ 基本需求
  + 用户下单并支付之后，应查询到账信息，进行实时对帐
  + 如果有不匹配的支付信息或者到账信息，输出提示信息
+ 解决思路
  + 从两条流中分别读取订单支付信息和到账信息，合并处理
  + 用connect连接合并两条流，用coProcessFunction做匹配处理

#### POJO

+ ReceiptEvent

  ```java
  private String txId;
  private String payChannel;
  private Long timestamp;
  ```

#### 代码1-具体实现

+ java代码实现

  ```java
  import beans.OrderEvent;
  import beans.ReceiptEvent;
  import org.apache.flink.api.common.state.ValueState;
  import org.apache.flink.api.common.state.ValueStateDescriptor;
  import org.apache.flink.api.java.tuple.Tuple2;
  import org.apache.flink.configuration.Configuration;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.co.CoProcessFunction;
  import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.streaming.runtime.operators.util.AssignerWithPeriodicWatermarksAdapter;
  import org.apache.flink.util.Collector;
  import org.apache.flink.util.OutputTag;
  
  import java.net.URL;
  import java.util.concurrent.TimeUnit;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/6 5:34 PM
   */
  public class TxPayMatch {
  
    // 定义侧输出流标签
    private final static OutputTag<OrderEvent> unmatchedPays = new OutputTag<OrderEvent>("unmatched-pays"){};
    private final static OutputTag<ReceiptEvent> unmatchedReceipts = new OutputTag<ReceiptEvent>("unmatched-receipts"){};
  
    public static void main(String[] args) throws Exception {
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
  
      // 读取数据并转换成POJO类型
      // 读取订单支付事件数据
      URL orderResource = TxPayMatch.class.getResource("/OrderLog.csv");
      DataStream<OrderEvent> orderEventStream = env.readTextFile(orderResource.getPath())
        .map(line -> {
          String[] fields = line.split(",");
          return new OrderEvent(new Long(fields[0]), fields[1], fields[2], new Long(fields[3]));
        })
        .assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
          new BoundedOutOfOrdernessTimestampExtractor<OrderEvent>(Time.of(200, TimeUnit.MILLISECONDS)) {
            @Override
            public long extractTimestamp(OrderEvent element) {
              return element.getTimestamp() * 1000L;
            }
          }
        ))
        // 交易id不为空，必须是pay事件
        .filter(data -> !"".equals(data.getTxId()));
  
      // 读取到账事件数据
      URL receiptResource = TxPayMatch.class.getResource("/ReceiptLog.csv");
      SingleOutputStreamOperator<ReceiptEvent> receiptEventStream = env.readTextFile(receiptResource.getPath())
        .map(line -> {
          String[] fields = line.split(",");
          return new ReceiptEvent(fields[0], fields[1], new Long(fields[2]));
        })
        .assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
          new BoundedOutOfOrdernessTimestampExtractor<ReceiptEvent>(Time.of(200, TimeUnit.MILLISECONDS)) {
            @Override
            public long extractTimestamp(ReceiptEvent element) {
              return element.getTimestamp() * 1000L;
            }
          }
        ));
  
      // 将两条流进行连接合并，进行匹配处理，不匹配的事件输出到侧输出流
      SingleOutputStreamOperator<Tuple2<OrderEvent, ReceiptEvent>> resultStream = orderEventStream.keyBy(OrderEvent::getTxId)
        .connect(receiptEventStream.keyBy(ReceiptEvent::getTxId))
        .process(new TxPayMatchDetect());
  
      resultStream.print("matched-pays");
      resultStream.getSideOutput(unmatchedPays).print("unmatched-pays");
      resultStream.getSideOutput(unmatchedReceipts).print("unmatched-receipts");
  
      env.execute("tx match detect job");
    }
  
    // 实现自定义CoProcessFunction
    public static class TxPayMatchDetect extends CoProcessFunction<OrderEvent, ReceiptEvent, Tuple2<OrderEvent, ReceiptEvent>> {
      // 定义状态，保存当前已经到来的订单支付事件和到账时间
      ValueState<OrderEvent> payState;
      ValueState<ReceiptEvent> receiptState;
  
      @Override
      public void open(Configuration parameters) throws Exception {
        payState = getRuntimeContext().getState(new ValueStateDescriptor<OrderEvent>("pay", OrderEvent.class));
        receiptState = getRuntimeContext().getState(new ValueStateDescriptor<ReceiptEvent>("receipt", ReceiptEvent.class));
      }
  
      @Override
      public void processElement1(OrderEvent pay, Context ctx, Collector<Tuple2<OrderEvent, ReceiptEvent>> out) throws Exception {
        // 订单支付事件来了，判断是否已经有对应的到账事件
        ReceiptEvent receipt = receiptState.value();
        if( receipt != null ){
          // 如果receipt不为空，说明到账事件已经来过，输出匹配事件，清空状态
          out.collect( new Tuple2<>(pay, receipt) );
          payState.clear();
          receiptState.clear();
        } else {
          // 如果receipt没来，注册一个定时器，开始等待
          ctx.timerService().registerEventTimeTimer( (pay.getTimestamp() + 5) * 1000L );    // 等待5秒钟，具体要看数据
          // 更新状态
          payState.update(pay);
        }
      }
  
      @Override
      public void processElement2(ReceiptEvent receipt, Context ctx, Collector<Tuple2<OrderEvent, ReceiptEvent>> out) throws Exception {
        // 到账事件来了，判断是否已经有对应的支付事件
        OrderEvent pay = payState.value();
        if( pay != null ){
          // 如果pay不为空，说明支付事件已经来过，输出匹配事件，清空状态
          out.collect( new Tuple2<>(pay, receipt) );
          payState.clear();
          receiptState.clear();
        } else {
          // 如果pay没来，注册一个定时器，开始等待
          ctx.timerService().registerEventTimeTimer( (receipt.getTimestamp() + 3) * 1000L );    // 等待3秒钟，具体要看数据
          // 更新状态
          receiptState.update(receipt);
        }
      }
  
      @Override
      public void onTimer(long timestamp, OnTimerContext ctx, Collector<Tuple2<OrderEvent, ReceiptEvent>> out) throws Exception {
        // 定时器触发，有可能是有一个事件没来，不匹配，也有可能是都来过了，已经输出并清空状态
        // 判断哪个不为空，那么另一个就没来
        if( payState.value() != null ){
          ctx.output(unmatchedPays, payState.value());
        }
        if( receiptState.value() != null ){
          ctx.output(unmatchedReceipts, receiptState.value());
        }
        // 清空状态
        payState.clear();
        receiptState.clear();
      }
    }
  }
  ```

+ 输出

  ```shell
  matched-pays> (OrderEvent{orderId=34729, eventType='pay', txId='sd76f87d6', timestamp=1558430844},ReceiptEvent{txId='sd76f87d6', payChannel='wechat', timestamp=1558430847})
  matched-pays> (OrderEvent{orderId=34730, eventType='pay', txId='3hu3k2432', timestamp=1558430845},ReceiptEvent{txId='3hu3k2432', payChannel='alipay', timestamp=1558430848})
  matched-pays> (OrderEvent{orderId=34732, eventType='pay', txId='32h3h4b4t', timestamp=1558430861},ReceiptEvent{txId='32h3h4b4t', payChannel='wechat', timestamp=1558430852})
  matched-pays> (OrderEvent{orderId=34733, eventType='pay', txId='766lk5nk4', timestamp=1558430864},ReceiptEvent{txId='766lk5nk4', payChannel='wechat', timestamp=1558430855})
  matched-pays> (OrderEvent{orderId=34734, eventType='pay', txId='435kjb45d', timestamp=1558430863},ReceiptEvent{txId='435kjb45d', payChannel='alipay', timestamp=1558430859})
  matched-pays> (OrderEvent{orderId=34735, eventType='pay', txId='5k432k4n', timestamp=1558430869},ReceiptEvent{txId='5k432k4n', payChannel='wechat', timestamp=1558430862})
  matched-pays> (OrderEvent{orderId=34736, eventType='pay', txId='435kjb45s', timestamp=1558430875},ReceiptEvent{txId='435kjb45s', payChannel='wechat', timestamp=1558430866})
  matched-pays> (OrderEvent{orderId=34738, eventType='pay', txId='43jhin3k4', timestamp=1558430896},ReceiptEvent{txId='43jhin3k4', payChannel='wechat', timestamp=1558430871})
  matched-pays> (OrderEvent{orderId=34741, eventType='pay', txId='88df0wn92', timestamp=1558430896},ReceiptEvent{txId='88df0wn92', payChannel='alipay', timestamp=1558430882})
  matched-pays> (OrderEvent{orderId=34737, eventType='pay', txId='324jnd45s', timestamp=1558430902},ReceiptEvent{txId='324jnd45s', payChannel='wechat', timestamp=1558430868})
  matched-pays> (OrderEvent{orderId=34743, eventType='pay', txId='3hefw8jf', timestamp=1558430900},ReceiptEvent{txId='3hefw8jf', payChannel='alipay', timestamp=1558430885})
  matched-pays> (OrderEvent{orderId=34744, eventType='pay', txId='499dfano2', timestamp=1558430903},ReceiptEvent{txId='499dfano2', payChannel='wechat', timestamp=1558430886})
  matched-pays> (OrderEvent{orderId=34742, eventType='pay', txId='435kjb4432', timestamp=1558430906},ReceiptEvent{txId='435kjb4432', payChannel='alipay', timestamp=1558430884})
  matched-pays> (OrderEvent{orderId=34745, eventType='pay', txId='8xz09ddsaf', timestamp=1558430896},ReceiptEvent{txId='8xz09ddsaf', payChannel='wechat', timestamp=1558430889})
  matched-pays> (OrderEvent{orderId=34739, eventType='pay', txId='98x0f8asd', timestamp=1558430907},ReceiptEvent{txId='98x0f8asd', payChannel='alipay', timestamp=1558430874})
  matched-pays> (OrderEvent{orderId=34746, eventType='pay', txId='3243hr9h9', timestamp=1558430895},ReceiptEvent{txId='3243hr9h9', payChannel='wechat', timestamp=1558430892})
  matched-pays> (OrderEvent{orderId=34740, eventType='pay', txId='392094j32', timestamp=1558430913},ReceiptEvent{txId='392094j32', payChannel='wechat', timestamp=1558430877})
  matched-pays> (OrderEvent{orderId=34747, eventType='pay', txId='329d09f9f', timestamp=1558430893},ReceiptEvent{txId='329d09f9f', payChannel='alipay', timestamp=1558430893})
  matched-pays> (OrderEvent{orderId=34749, eventType='pay', txId='324n0239', timestamp=1558430916},ReceiptEvent{txId='324n0239', payChannel='wechat', timestamp=1558430899})
  matched-pays> (OrderEvent{orderId=34748, eventType='pay', txId='809saf0ff', timestamp=1558430934},ReceiptEvent{txId='809saf0ff', payChannel='wechat', timestamp=1558430895})
  matched-pays> (OrderEvent{orderId=34752, eventType='pay', txId='rnp435rk', timestamp=1558430925},ReceiptEvent{txId='rnp435rk', payChannel='wechat', timestamp=1558430905})
  matched-pays> (OrderEvent{orderId=34751, eventType='pay', txId='24309dsf', timestamp=1558430941},ReceiptEvent{txId='24309dsf', payChannel='alipay', timestamp=1558430902})
  matched-pays> (OrderEvent{orderId=34753, eventType='pay', txId='8c6vs8dd', timestamp=1558430913},ReceiptEvent{txId='8c6vs8dd', payChannel='wechat', timestamp=1558430906})
  matched-pays> (OrderEvent{orderId=34750, eventType='pay', txId='sad90df3', timestamp=1558430941},ReceiptEvent{txId='sad90df3', payChannel='alipay', timestamp=1558430901})
  matched-pays> (OrderEvent{orderId=34755, eventType='pay', txId='8x0zvy8w3', timestamp=1558430918},ReceiptEvent{txId='8x0zvy8w3', payChannel='alipay', timestamp=1558430911})
  matched-pays> (OrderEvent{orderId=34754, eventType='pay', txId='3245nbo7', timestamp=1558430950},ReceiptEvent{txId='3245nbo7', payChannel='alipay', timestamp=1558430908})
  matched-pays> (OrderEvent{orderId=34758, eventType='pay', txId='32499fd9w', timestamp=1558430950},ReceiptEvent{txId='32499fd9w', payChannel='alipay', timestamp=1558430921})
  matched-pays> (OrderEvent{orderId=34759, eventType='pay', txId='9203kmfn', timestamp=1558430950},ReceiptEvent{txId='9203kmfn', payChannel='alipay', timestamp=1558430922})
  matched-pays> (OrderEvent{orderId=34760, eventType='pay', txId='390mf2398', timestamp=1558430960},ReceiptEvent{txId='390mf2398', payChannel='alipay', timestamp=1558430926})
  matched-pays> (OrderEvent{orderId=34757, eventType='pay', txId='d8938034', timestamp=1558430962},ReceiptEvent{txId='d8938034', payChannel='wechat', timestamp=1558430915})
  matched-pays> (OrderEvent{orderId=34761, eventType='pay', txId='902dsqw45', timestamp=1558430943},ReceiptEvent{txId='902dsqw45', payChannel='wechat', timestamp=1558430927})
  matched-pays> (OrderEvent{orderId=34762, eventType='pay', txId='84309dw31r', timestamp=1558430983},ReceiptEvent{txId='84309dw31r', payChannel='alipay', timestamp=1558430933})
  matched-pays> (OrderEvent{orderId=34763, eventType='pay', txId='sddf9809ew', timestamp=1558431068},ReceiptEvent{txId='sddf9809ew', payChannel='alipay', timestamp=1558430936})
  matched-pays> (OrderEvent{orderId=34764, eventType='pay', txId='832jksmd9', timestamp=1558431079},ReceiptEvent{txId='832jksmd9', payChannel='wechat', timestamp=1558430938})
  matched-pays> (OrderEvent{orderId=34765, eventType='pay', txId='m23sare32e', timestamp=1558431082},ReceiptEvent{txId='m23sare32e', payChannel='wechat', timestamp=1558430940})
  matched-pays> (OrderEvent{orderId=34766, eventType='pay', txId='92nr903msa', timestamp=1558431095},ReceiptEvent{txId='92nr903msa', payChannel='wechat', timestamp=1558430944})
  matched-pays> (OrderEvent{orderId=34767, eventType='pay', txId='sdafen9932', timestamp=1558432021},ReceiptEvent{txId='sdafen9932', payChannel='alipay', timestamp=1558430949})
  unmatched-receipts> ReceiptEvent{txId='ewr342as4', payChannel='wechat', timestamp=1558430845}
  unmatched-receipts> ReceiptEvent{txId='8fdsfae83', payChannel='alipay', timestamp=1558430850}
  unmatched-pays> OrderEvent{orderId=34731, eventType='pay', txId='35jue34we', timestamp=1558430849}
  unmatched-receipts> ReceiptEvent{txId='9032n4fd2', payChannel='wechat', timestamp=1558430913}
  unmatched-pays> OrderEvent{orderId=34768, eventType='pay', txId='88snrn932', timestamp=1558430950}
  ```

#### 代码2-Join实现

**这种方法的缺陷，只能获得正常匹配的结果，不能获得未匹配成功的记录。**

+ java代码

  ```java
  import beans.OrderEvent;
  import beans.ReceiptEvent;
  import org.apache.flink.api.java.tuple.Tuple2;
  import org.apache.flink.streaming.api.TimeCharacteristic;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  import org.apache.flink.streaming.api.functions.co.ProcessJoinFunction;
  import org.apache.flink.streaming.api.functions.timestamps.AscendingTimestampExtractor;
  import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
  import org.apache.flink.streaming.api.windowing.time.Time;
  import org.apache.flink.streaming.runtime.operators.util.AssignerWithPeriodicWatermarksAdapter;
  import org.apache.flink.util.Collector;
  
  import java.net.URL;
  import java.util.concurrent.TimeUnit;
  
  /**
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2021/2/6 7:55 PM
   */
  public class TxPayMatchByJoin {
  
    public static void main(String[] args) throws Exception {
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
      env.setParallelism(1);
  
      // 读取数据并转换成POJO类型
      // 读取订单支付事件数据
      URL orderResource = TxPayMatch.class.getResource("/OrderLog.csv");
      DataStream<OrderEvent> orderEventStream = env.readTextFile(orderResource.getPath())
        .map(line -> {
          String[] fields = line.split(",");
          return new OrderEvent(new Long(fields[0]), fields[1], fields[2], new Long(fields[3]));
        })
        .assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
          new BoundedOutOfOrdernessTimestampExtractor<OrderEvent>(Time.of(200, TimeUnit.MILLISECONDS)) {
            @Override
            public long extractTimestamp(OrderEvent element) {
              return element.getTimestamp() * 1000L;
            }
          }
        ))
        // 交易id不为空，必须是pay事件
        .filter(data -> !"".equals(data.getTxId()));
  
      // 读取到账事件数据
      URL receiptResource = TxPayMatch.class.getResource("/ReceiptLog.csv");
      SingleOutputStreamOperator<ReceiptEvent> receiptEventStream = env.readTextFile(receiptResource.getPath())
        .map(line -> {
          String[] fields = line.split(",");
          return new ReceiptEvent(fields[0], fields[1], new Long(fields[2]));
        })
        .assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarksAdapter.Strategy<>(
          new BoundedOutOfOrdernessTimestampExtractor<ReceiptEvent>(Time.of(200, TimeUnit.MILLISECONDS)) {
            @Override
            public long extractTimestamp(ReceiptEvent element) {
              return element.getTimestamp() * 1000L;
            }
          }
        ));
  
      // 区间连接两条流，得到匹配的数据
      SingleOutputStreamOperator<Tuple2<OrderEvent, ReceiptEvent>> resultStream = orderEventStream
        .keyBy(OrderEvent::getTxId)
        .intervalJoin(receiptEventStream.keyBy(ReceiptEvent::getTxId))
        .between(Time.seconds(-3), Time.seconds(5))    // -3，5 区间范围
        .process(new TxPayMatchDetectByJoin());
  
      resultStream.print();
  
      env.execute("tx pay match by join job");
    }
  
    // 实现自定义ProcessJoinFunction
    public static class TxPayMatchDetectByJoin extends ProcessJoinFunction<OrderEvent, ReceiptEvent, Tuple2<OrderEvent, ReceiptEvent>> {
      @Override
      public void processElement(OrderEvent left, ReceiptEvent right, Context ctx, Collector<Tuple2<OrderEvent, ReceiptEvent>> out) throws Exception {
        out.collect(new Tuple2<>(left, right));
      }
    }
  }
  
  ```

+ 输出

  ```shell
  (OrderEvent{orderId=34729, eventType='pay', txId='sd76f87d6', timestamp=1558430844},ReceiptEvent{txId='sd76f87d6', payChannel='wechat', timestamp=1558430847})
  (OrderEvent{orderId=34730, eventType='pay', txId='3hu3k2432', timestamp=1558430845},ReceiptEvent{txId='3hu3k2432', payChannel='alipay', timestamp=1558430848})
  (OrderEvent{orderId=34746, eventType='pay', txId='3243hr9h9', timestamp=1558430895},ReceiptEvent{txId='3243hr9h9', payChannel='wechat', timestamp=1558430892})
  (OrderEvent{orderId=34747, eventType='pay', txId='329d09f9f', timestamp=1558430893},ReceiptEvent{txId='329d09f9f', payChannel='alipay', timestamp=1558430893})
  ```

# 15. CEP

> [Flink-复杂事件（CEP）](https://zhuanlan.zhihu.com/p/43448829)
>
> [Flink之CEP(复杂时间处理)](https://blog.csdn.net/qq_37135484/article/details/106327567)

## 15.1 基本概念

### 15.1.1 什么是CEP

+ 复杂事件处理（Complex Event Processing，CEP）
+ Flink CEP是在Flink中实现的复杂事件处理（CEP）库
+ CEP允许在**无休止的事件流**中检测事件模式，让我们有机会掌握数据中重要的部分

+ **一个或多个由简单事件构成的事件流通过一定的规则匹配，然后输出用户想得到的数据——满足规则的复杂事件**

### 15.1.2 CEP特点

![img](https://pic1.zhimg.com/80/v2-1c7057bda8a3ba077a3b8059f35d9bc4_1440w.jpg)

+ 目标：从有序的简单事件流中发现一些高阶特征

- 输入：一个或多个由简单事件构成的事件流

- 处理：识别简单事件之间的内在联系，多个符合一定规则的简单事件构成复杂事件

- 输出：满足规则的复杂事件

## 15.2 Pattern API

+ 处理事件的规则，被叫做"模式"（Pattern）

+ Flink CEP提供了Pattern API，用于对输入流数据进行复杂事件规则定义，用来提取符合规则的时间序列

  ```java
  DataStream<Event> input = ...
  // 定义一个Pattern
  Pattern<Event, Event> pattern = Pattern.<Event>begin("start").where(...)
    .next("middle").subtype(SubEvent.class).where(...)
    .followedBy("end").where(...);
  // 将创建好的Pattern应用到输入事件流上
  PatternStream<Event> patternStream = CEP.pattern(input,pattern);
  // 检出匹配事件序列，处理得到结果
  DataStream<Alert> result = patternStream.select(...);
  ```

### 个体模式(Individual Patterns)

+ 组成复杂规则的每一个单独的模式定义,就是"个体模式"

  ```java
  start.times(3).where(new SimpleCondition<Event>() {...})
  ```

+ 个体模式可以包括"单例(singleton)模式"和"循环(looping)模式"
+ 单例模式只接收一个事件，而循环模式可以接收多个

---

+ 量词（Quantifier）

  可以在一个个体模式后追加量词，也就是指定循环次数

  ```java
  //匹配出现4次
  start.times(4)
  //匹配出现2/3/4次
  start.time(2,4).greedy
  //匹配出现0或者4次
  start.times(4).optional
  //匹配出现1次或者多次
  start.oneOrMore
  //匹配出现2,3,4次
  start.times(2,4)
  //匹配出现0次,2次或者多次,并且尽可能多的重复匹配
  start.timesOrMore(2),optional.greedy
  ```

+ 条件（Condition）

  + **每个模式都需要指定触发条件**，作为模式是否接受事件进入的判断依据

  + CEP中的个体模式主要通过调用`.where()`，`.or()`和`.until()`来指定条件

  + 按不同的调用方式，可以分成以下几类

    + 简单条件（Simple Condition）

      通过`.where()`方法对事件中的字段进行判断筛选，决定是否接受该事件

      ```java
      start.where(new SimpleCondition<Event>){
        @Override
        public boolean filter(Event value) throws Exception{
          return value.getName.startsWith("foo");
        }
      }
      ```

    + 组合条件（Combining Condition）

      将简单条件进行合并；`.or()`方法表示或逻辑相连，where的直接组合就是AND

      `pattern.where(event => ... /* some condition */).or(event => ... /* or condition */)`

    + 终止条件（Stop Condition）

      如果使用了`oneOrMore`或者`oneOrMore.optional`，建议使用`.until()`作为终止条件，以便清理状态

    + 迭代条件（Iterative Condition）

      能够对模式之前所有接收的事件进行处理

      可以调用`ctx.getEventsForPattern("name")`

      ```java
      .where(new IterativeCondition<Event>(){...})
      ```

### 组合模式(Combining Patterns)

组合模式(Combining Patterns)也叫模式序列。

+ 很多个体模式组合起来，就形成了整个的模式序列

+ 模式序列必须以一个"初始模式"开始

  ```java
  Pattern<Event, Event> start = Pattern.<Event>begin("start")
  ```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200526221919332.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3MTM1NDg0,size_16,color_FFFFFF,t_70)

+ 严格近邻(Strict Contiguity)
  + **所有事件按照严格的顺序出现**，中间没有任何不匹配的事件，由`.next()`指定
  + 例如对于模式"a next b",事件序列[a,c,b1,b2]没有匹配
+ 宽松近邻(Relaxed Contiguity)
  + 允许中间出现不匹配的事件,由`.followedBy()`指定
  + 例如对于模式"a followedBy b",事件序列[a,c,b1,b2]匹配为[a,b1]

+ 非确定性宽松近邻(Non-Deterministic Relaxed Contiguity)

  + 进一步放宽条件,之前已经匹配过的事件也可以再次使用，由`.followByAny()`指定
  + 例如对于模式"a followedAny b",事件序列[a,c,b1,b2]匹配为{a,b1},{a,b2}

+ 除了以上模式序列外,还可以定义"不希望出现某种近邻关系":

  + `.notNext()` 不严格近邻

  + `.notFollowedBy()`不在两个事件之间发生

    （eg，a not FollowedBy c，a Followed By b，a希望之后出现b，且不希望ab之间出现c）

+ 需要注意：

  +  **所有模式序列必须以`.begin()`开始**

  +  **模式序列不能以`.notFollowedBy()`结束**

  +  **"not "类型的模式不能被optional 所修饰**

  +  此外,还可以为模式指定事件约束，用来<u>要求在多长时间内匹配有效</u>: 

    `next.within(Time.seconds(10))`

### 模式组(Groups of patterns)

+ 将一个模式序列作为条件嵌套在个体模式里，成为一组模式

## 15.3 模式的检测

- 指定要查找的模式序列后，就可以将其应用于输入流以检测潜在匹配

- 调用`CEP.pattern()`，给定输入流和模式，就能得到一个PatternStream

  ```java
  DataStream<Event> input = ...
  Pattern<Event, Event> pattern = Pattern.<Event>begin("start").where(...)...
  
  PatternStream<Event> patternStream = CEP.pattern(input, pattern);
  ```

## 15.4 匹配事件的提取

- 创建PatternStrean之后，就可以应用select或者flatselect方法，从检测到的事件序列中提取事件了

- `select()`方法需要输入一个select function作为参数,每个成功匹配的事件序列都会调用它

- `select()` 以一个Map<String，List<IN]>> 来接收匹配到的事件序列，其中Key就是每个模式的名称，而value就是所有接收到的事件的List类型

  ```java
  public OUT select(Map<String, List<IN>> pattern) throws Exception {
    IN startEvent = pattern.get("start").get(0);
    IN endEvent = pattern.get("end").get(0);
    return new OUT(startEvent, endEvent);
  }
  ```

## 15.4 超时事件的提取

+ **当一个模式通过within关键字定义了检测窗口时间时，部分事件序列可能因为超过窗口长度而被丢弃；为了能够处理这些超时的部分匹配，select和flatSelect API调用允许指定超时处理程序**

+ **超时处理程序会接收到目前为止由模式匹配到的所有事件，由一个OutputTag定义接收到的超时事件序列**

  ```java
  PatternStream<Event> patternStream = CEP.pattern(input, pattern);
  OutputTag<String> outputTag = new OutputTag<String>("side-output"){};
  
  SingleOutputStreamOperator<ComplexEvent> flatResult = 
    patternStream.flatSelect(
    outputTag,
    new PatternFlatTimeoutFunction<Event, TimeoutEvent>() {...},
    new PatternFlatSelectFunction<Event, ComplexEvent>() {...}
  );
  DataStream<TimeoutEvent> timeoutFlatResult = 
    flatResult.getSideOutput(outputTag);
  ```



