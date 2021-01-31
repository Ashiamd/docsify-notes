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
    DataStream<String> inputStream = env.readTextFile("/Users/ashiamd/mydocs/docs/study/javadocument/javadocument/IDEA_project/Flink_Tutorial/src/main/resources/sensor.txt");

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





