# 尚硅谷Flink入门到实战-学习笔记

> [尚硅谷2021最新Java版Flink](https://www.bilibili.com/video/BV1qy4y1q728)

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
    String inputPath = "/Users/ashiamd/mydocs/docs/study/javadocument/javadocument/IDEA_project/Flink_Tutorial/src/main/resources/hello.txt";
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
        String inputPath = "/Users/ashiamd/mydocs/docs/study/javadocument/javadocument/IDEA_project/Flink_Tutorial/src/main/resources/hello.txt";
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
   //        String inputPath = "/Users/ashiamd/mydocs/docs/study/javadocument/javadocument/IDEA_project/Flink_Tutorial/src/main/resources/hello.txt";
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

