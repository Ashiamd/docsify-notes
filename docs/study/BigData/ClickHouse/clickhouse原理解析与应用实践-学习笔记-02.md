# clickhouse原理解析与应用实践-学习笔记-02

# 8. 其他常见类型表引擎

​	Everything is table（万物皆为表）是ClickHouse一个非常有意思的设计思路，正因为ClickHouse是一款数据库，所以自然而然的，数据表就是它的武器，是它与外部进行交互的接口层。在数据表背后无论连接的是本地文件、HDFS、Zookeeper还是其他服务，终端用户始终只需面对数据表，只需使用SQL查询语言。

​	本章将继续介绍其他常见类型的表引擎，它们以表作为接口，极大地丰富了ClickHouse的查询能力。这些表引擎各自特点突出，或是独立地应用于特定场景，或是能够与MergeTree一起搭配使用。例如，外部存储系列的表引擎，能够直接读取其他系统的数据，ClickHouse自身只负责元数据管理，类似使用外挂表的形式；内存系列的表引擎，能够充当数据分发的临时存储载体或消息通道；日志文件系列的表引擎，拥有简单易用的特点；接口系列表引擎，能够串联已有的数据表，起到黏合剂的作用。在本章后续的内容中，会按照表引擎的分类逐个进行介绍，包括它们的特点和使用方法。

## 8.1 外部存储类型

​	顾名思义，**外部存储表引擎直接从其他的存储系统读取数据**，例如直接读取HDFS的文件或者MySQL数据库的表。**<u>这些表引擎只负责元数据管理和数据查询，而它们自身通常并不负责数据的写入，数据文件直接由外部系统提供</u>**。

### 8.1.1 HDFS

​	HDFS是一款分布式文件系统，堪称Hadoop生态的基石，HDFS表引擎则能够直接与它对接，读取HDFS内的文件。关于HDFS环境的安装部署，相关资料很多，《企业级大数据平台构建：架构与实现》一书中也有详细介绍，此处不再赘述。

​	假设HDFS环境已经准备就绪，在正式使用HDFS表引擎之前还需要做一些准备工作：首先需要关闭HDFS的Kerberos认证（因为**HDFS表引擎目前还不支持Kerberos**）；接着在HDFS上创建用于存放文件的目录：

```shell
hadoop fs -mkdir /clickhouse
```

​	最后，在HDFS上给ClickHouse用户授权。例如，为默认用户clickhouse授权的方法如下：

```shell
hadoop fs -chown -R clickhouse:clickhouse /clickhouse
```

​	至此，前期的准备工作就算告一段落了。

​	HDFS表引擎的定义方法如下：

```sql
ENGINE = HDFS(hdfs_uri,format)
```

其中：

+ hdfs_uri表示HDFS的文件存储路径；

+ format表示文件格式（指ClickHouse支持的文件格式，常见的有CSV、TSV和JSON等）。

HDFS表引擎通常有两种使用形式：

+ 既负责读文件，又负责写文件。
+ 只负责读文件，文件写入工作则由其他外部系统完成。

​	首先，介绍第一种形式的使用方法，即HDFS文件的创建与查询均使用HDFS数据表。创建HDFS数据表的方式如下：

```sql
CREATE TABLE hdfs_table1(
  id UInt32,
  code String,
  name String
)ENGINE = HDFS('hdfs://hdp1.nauu.com:8020/clickhouse/hdfs_table1','CSV')
```

​	在上面的配置中，hdfs_uri是数据文件的<u>绝对路径</u>，文件格式为CSV。

​	接着写入测试数据：

```sql
INSERT INTO hdfs_table1 SELECT number,concat('code',toString(number)),
concat('n',toString(number)) FROM numbers(5)
```

​	在数据写入之后，就可以通过数据表查询了：

```sql
SELECT * FROM hdfs_table1
┌─id─┬─code─┬─name─┐
│ 0 │ code0 │ n0 │
│ 1 │ code1 │ n1 │
│ 2 │ code2 │ n2 │
│ 3 │ code3 │ n3 │
│ 4 │ code4 │ n4 │
└───┴─────┴────┘
```

​	接着再看看在HDFS上发生了什么变化。执行hadoop fs -cat查看文件：

```shell
$ hadoop fs -cat /clickhouse/hdfs_table1
0,"code0","n0"
1,"code1","n1"
2,"code2","n2"
3,"code3","n3"
4,"code4","n4"
```

​	可以发现，通过HDFS表引擎，ClickHouse在HDFS的指定目录下创建了一个名为hdfs_table1的文件，并且按照CSV格式写入了数据。不过**目前ClickHouse并没有提供删除HDFS文件的方法，即便将数据表hdfs_table1删除**：

```sql
DROP Table hdfs_table1
```

​	在HDFS上文件依然存在：

```shell
$ hadoop fs -ls /clickhouse
Found 1 items
-rwxrwxrwx 3 clickhouse clickhouse /clickhouse/hdfs_table1
```

​	接下来，介绍<u>第二种形式的使用方法，这种形式类似Hive的外挂表，由其他系统直接将文件写入HDFS</u>。通过HDFS表引擎的hdfs_uri和format参数分别与HDFS的文件路径、文件格式建立映射。其中，hdfs_uri支持以下几种常见的配置方法：

+ 绝对路径：会读取指定路径的单个文件，例如/clickhouse/hdfs_table1。

+ \*通配符：匹配所有字符，例如路径为/clickhouse/hdfs_table/\*，则会读取/click-house/hdfs_table路径下的所有文件。

+ ?通配符：匹配单个字符，例如路径为/clickhouse/hdfs_table/organization_?.csv，则会读

  取/clickhouse/hdfs_table路径下与organization_?.csv匹配的文件，其中?代表任意一个合法字符。

+ {M..N}数字区间：匹配指定数字的文件，例如路径为/clickhouse/hdfs_table/organization_{1..3}.csv，则会读取/clickhouse/hdfs_table/路径下的文件organization_1.csv、organization_2.csv和organization_3.csv。

​	现在用一个具体示例验证表引擎的效果。首先，将事先准备好的3个CSV测试文件上传至HDFS的/clickhouse/hdfs_table2路径（用于测试的CSV文件，可以在本书的github仓库获取）：

```shell
--上传文件至HDFS
$ hadoop fs -put /chbase/demo-data/ /clickhouse/hdfs_table2
--查询路径
$ hadoop fs -ls /clickhouse/hdfs_table2
Found 3 items
-rw-r--r-- 3 hdfs clickhouse /clickhouse/hdfs_table2/organization_1.csv
-rw-r--r-- 3 hdfs clickhouse /clickhouse/hdfs_table2/organization_2.csv
-rw-r--r-- 3 hdfs clickhouse /clickhouse/hdfs_table2/organization_3.csv
```

​	接着，创建HDFS测试表：

```sql
CREATE TABLE hdfs_table2(
  id UInt32,
  code String,
  name String
) ENGINE = HDFS('hdfs://hdp1.nauu.com:8020/clickhouse/hdfs_table2/*','CSV')
```

​	其中，下列几种配置路径的方式，针对上面的测试表，效果是等价的：

+ \*通配符：

  ```shell
  HDFS('hdfs://hdp1.nauu.com:8020/clickhouse/hdfs_table2/*','CSV')
  ```

+ ?通配符：

  ```shell
  HDFS('hdfs://hdp1.nauu.com:8020/clickhouse/hdfs_table2/organization_?.csv','CSV')
  ```

+ {M..N}数字区间：

  ```shell
  HDFS('hdfs://hdp1.nauu.com:8020/clickhouse/hdfs_table2/organization_{1..3}.csv','
  CSV')
  ```

​	选取上面任意一种配置方式后，查询数据表：

```sql
SELECT * FROM hdfs_table2
┌─id─┬─code─┬─name───┐
│ 4 │ a0004 │ 测试部 │
│ 5 │ a0005 │ 运维部 │
└───┴─────┴──────┘
┌─id─┬─code─┬─name───┐
│ 1 │ a0001 │ 研发部 │
│ 2 │ a0002 │ 产品部 │
│ 3 │ a0003 │ 数据部 │
└───┴─────┴──────┘
┌─id─┬─code─┬─name───┐
│ 6 │ a0006 │ 规划部 │
│ 7 │ a0007 │ 市场部 │
└───┴─────┴──────┘
```

​	可以看到，<u>3个文件的数据以3个分区的形式合并返回了</u>。

### 8.1.2 MySQL

​	MySQL表引擎可以与MySQL数据库中的数据表建立映射，并通过SQL向其发起远程查询，包括SELECT和INSERT，它的声明方式如下：

```sql
ENGINE = MySQL('host:port', 'database', 'table', 'user', 'password'[,
replace_query, 'on_duplicate_clause'])
```

​	其中各参数的含义分别如下：

+ host:port表示MySQL的地址和端口。

+ database表示数据库的名称。

+ table表示需要映射的表名称。

+ user表示MySQL的用户名。

+ password表示MySQL的密码。

+ **replace_query默认为0，对应MySQL的REPLACE INTO语法。如果将它设置为1，则会用REPLACE INTO代替INSERT INTO**。

+ **on_duplicate_clause默认为0，对应MySQL的ON DUPLICATE KEY语法。如果需要使用该设置，则必须将replace_query设置成0**。

​	现在用一个具体的示例说明MySQL表引擎的用法。假设MySQL数据库已准备就绪，则使用MySQL表引擎与其建立映射：

```sql
CREATE TABLE dolphin_scheduler_table(
  id UInt32,
  name String
)ENGINE = MySQL('10.37.129.2:3306', 'escheduler', 't_escheduler_process_definition', 'root', '')
```

​	创建成功之后，就可以通过这张数据表代为查询MySQL中的数据了，例如：

```sql
SELECT * FROM dolphin_scheduler_table
┌─id─┬─name───┐
│ 1 │ 流程1 │
│ 2 │ 流程2 │
│ 3 │ 流程3 │
└──┴────────┘
```

​	接着，尝试写入数据：

```sql
INSERT INTO TABLE dolphin_scheduler_table VALUES (4,'流程4')
```

​	再次查询t_escheduler_proess_definition，可以发现数据已被写入远端的MySQL表内了。

​	在具备了INSERT写入能力之后，就可以尝试一些组合玩法了。例如执行下面的语句，创建一张物化视图，将MySQL表引擎与物化视图一起搭配使用：

```sql
CREATE MATERIALIZED VIEW view_mysql1
ENGINE = MergeTree()
ORDER BY id
AS SELECT * FROM dolphin_scheduler_table
```

​	<u>当通过MySQL表引擎向远端MySQL数据库写入数据的同时，物化视图也会同步更新数据</u>。

​	不过比较遗憾的是，**目前MySQL表引擎不支持任何UPDATE和DELETE操作**，如果有数据更新方面的诉求，可以考虑使用CollapsingMergeTree作为视图的表引擎。

### 8.1.3 JDBC

​	相对MySQL表引擎而言，JDBC表引擎不仅可以对接MySQL数据库，还能够与PostgreSQL、SQLite和H2数据库对接。但是，JDBC表引擎无法单独完成所有的工作，它需要依赖名为clickhouse-jdbc-bridge的查询代理服务。clickhouse-jdbc-bridge是一款基于Java语言实现的SQL代理服务，它的项目地址为https://github.com/ClickHouse/clickhouse-jdbc-bridge 。clickhouse-jdbc-bridge可以为ClickHouse代理访问其他的数据库，并自动转换数据类型。数据类型的映射规则如表8-1所示：

​	表8-1 ClickHouse数据类型与JDBC标准类型的对应关系

| ClickHouse数据类型 | JDBC标准数据类型 | ClickHouse数据类型 | JDBC标准数据类型 |
| ------------------ | ---------------- | ------------------ | ---------------- |
| Int8               | TINYINT          | DateTime           | TIME             |
| Int16              | SMALLINT         | Date               | DATE             |
| Int32              | INTEGER          | UInt8              | BIT              |
| Int64              | BIGINT           | UInt8              | BOOLEAN          |
| Float32            | FLOAT            | String             | CHAR             |
| Float32            | REAL             | String             | VARCHAR          |
| Float64            | DOUBLE           | String             | LONGVARCHAR      |
| DateTime           | TIMESTAMP        |                    |                  |

​	关于clickhouse-jdbc-bridge的构建方法，其项目首页有详细说明，此处不再赘述。简单而言，在通过Maven构建之后，最终会生成一个名为clickhouse-jdbc-bridge-1.0.jar的服务jar包。在本书对应的Github仓库上，我已经为大家编译好了jar包，可以直接下载使用。

​	在使用JDBC表引擎之前，首先需要启动clickhouse-jdbc-bridge代理服务，启动的方式如下：

```shell
java -jar ./clickhouse-jdbc-bridge-1.0.jar --driver-path /chbase/jdbc-bridge --
listen-host ch5.nauu.com
```

​	一些配置参数的含义如下：

+ --driver-path用于指定放置数据库驱动的目录，例如要代理查询PostgreSQL数据库，则需要将它的驱动jar放置到这个目录。

+ --listen-host用于代理服务的监听端口，通过这个地址访问代理服务，ClickHouse的jdbc_bridge配置项与此参数对应。

​	至此，代理服务就配置好了。接下来，需要在config.xml全局配置中增加代理服务的访问地址：

```xml
……
  <jdbc_bridge>
    <host>ch5.nauu.com</host>
    <port>9019</port>
  </jdbc_bridge>
</yandex>
```

​	前期准备工作全部就绪之后就可以开始创建JDBC表了。JDBC表引擎的声明方式如下所示：

```sql
ENGINE = JDBC('jdbc:url', 'database', 'table')
```

​	其中，url表示需要对接的数据库的驱动地址；database表示数据库名称；table则表示对接的数据表的名称。现在创建一张JDBC表：

```sql
CREATE TABLE t_ds_process_definition (
  id Int32,
  name String
)ENGINE = JDBC('jdbc:postgresql://ip:5432/dolphinscheduler?user=test&password=test, '', 't_ds_process_definition')
```

​	查询这张数据表：

```sql
SELECT id,name FROM t_ds_process_definition
```

​	观察ClickHouse的服务日志：

```shell
<Debug>executeQuery:(from ip:36462)SELECT id, name FROM t_ds_process_definition
```

​	可以看到，伴随着每一次的SELECT查询，JDBC表引擎首先会向clickhouse-jdbc-bridge发送一次ping请求，以探测代理是否启动：

```shell
<Trace> ReadWriteBufferFromHTTP: Sending request to http://ch5.nauu.com:9019/ping
```

​	如果ping服务访问不到，或者返回值不是“Ok.”，那么将会得到下面这条错误提示：

```shell
DB::Exception: jdbc-bridge is not running. Please, start it manually
```

​	在这个示例中，clickhouse-jdbc-bridge早已启动，所以在ping探测之后，JDBC表引擎向代理服务发送了查询请求：

```shell
<Trace> ReadWriteBufferFromHTTP: Sending request to http://ch5.nauu.com:9019/ ?
connection_string=jdbc%3Apo.....
```

​	之后，代理查询通过JDBC协议访问数据库，并将数据返回给JDBC表引擎：

```sql
┌─id─┬─name───┐
│ 3 │ db2测试 │
│ 4 │ dag2 │
│ 5 │ hive │
│ 6 │ db2 │
│ 7 │ flink-A │
└───┴───────┘
```

​	除了JDBC表引擎之外，jdbc函数也能够通过clickhouse-jdbcbridge代理访问其他数据库，例如执行下面的语句：

```sql
SELECT id,name FROM
jdbc('jdbc:postgresql://ip:5432/dolphinscheduler?user=test&password=test, '','t_ds_process_definition')
```

​	得到的查询结果将会与示例中的JDBC表相同。

### 8.1.4 Kafka

​	Kafka是大数据领域非常流行的一款分布式消息系统。Kafka表引擎能够直接与Kafka系统对接，进而订阅Kafka中的主题并实时接收消息数据。

​	众所周知，在消息系统中存在三层语义，它们分别是：

+ 最多一次（At most once）：可能出现丢失数据的情况，因为在这种情形下，一行数据在消费端最多只会被接收一次。
+ 最少一次（At least once）：可能出现重复数据的情况，因为在这种情形下，一行数据在消费端允许被接收多次。
+ 恰好一次（Exactly once）：数据不多也不少，这种情形是最为理想的状况。

​	虽然Kafka本身能够支持上述三层语义，但是**<u>目前ClickHouse还不支持恰好一次（Exactly once）的语义，因为这需要应用端与Kafka深度配合才能实现</u>**。Kafka使用offset标志位记录主题数据被消费的位置信息，当应用端接收到消息之后，通过自动或手动执行Kafka commit，提交当前的offset信息，以保障消息的语义，所以ClickHouse在这方面还有进步的空间。

​	Kafka表引擎的声明方式如下所示：

```sql
ENGINE = Kafka()
SETTINGS
  kafka_broker_list = 'host:port,... ',
  kafka_topic_list = 'topic1,topic2,...',
  kafka_group_name = 'group_name',
  kafka_format = 'data_format'[,]
  [kafka_row_delimiter = 'delimiter_symbol']
  [kafka_schema = '']
  [kafka_num_consumers = N]
  [kafka_skip_broken_messages = N]
  [kafka_commit_every_batch = N]
```

​	其中，带有方括号的参数表示选填项，现在依次介绍这些参数的作用。首先是必填参数：

+ kafka_broker_list：表示Broker服务的地址列表，多个地址之间使用逗号分隔，例如'hdp1.nauu.com:6667，hdp2.nauu.com:6667'。

+ kafka_topic_list：表示订阅消息主题的名称列表，多个主题之间使用逗号分隔，例如'topic1,topic2'。多个主题中的数据均会被消费。

+ kafka_group_name：表示消费组的名称，表引擎会依据此名称创建Kafka的消费组。

+ kafka_format：表示用于解析消息的数据格式，在消息的发送端，必须按照此格式发送消息。数据格式必须是ClickHouse提供的格式之一，例如TSV、JSONEachRow和CSV等。

​	接下来是选填参数：

+ kafka_row_delimiter：表示判定一行数据的结束符，默认值为'Ä0'。

+ kafka_schema：对应Kafka的schema参数。

+ kafka_num_consumers：表示消费者的数量，默认值为1。表引擎会依据此参数在消费组中开启相应数量的消费者线程。**在Kafka的主题中，一个Partition分区只能使用一个消费者**。

+ <u>kafka_skip_broken_messages：当表引擎按照预定格式解析数据出现错误时，允许跳过失败的数据行数，默认值为0，即不允许任何格式错误的情形发生。在此种情形下，只要Kafka主题中存在无法解析的数据，数据表都将不会接收任何数据。如果将其设置为非0正整数，例如kafka_skip_broken_messages=10，表示只要Kafka主题中存在无法解析的数据的总数小于10，数据表就能正常接收消息数据，而解析错误的数据会被自动跳过</u>。

+ kafka_commit_every_batch：<u>表示执行Kafka commit的频率，默认值为0，即当一整个Block数据块完全写入数据表后才执行Kafka commit。如果将其设置为1，则每写完一个Batch批次的数据就会执行一次Kafka commit（一次Block写入操作，由多次Batch写入操作组成）</u>。

  除此之外，还有一些配置参数可以调整表引擎的行为。在默认情况下，Kafka表引擎每间隔500毫秒会拉取一次数据，时间由stream_poll_timeout_ms参数控制（默认500毫秒）。数据首先会被放入缓存，在时机成熟的时候，缓存数据会被刷新到数据表。

  触发Kafka表引擎刷新缓存的条件有两个，当满足其中的任意一个时，便会触发刷新动作：

  + 当一个数据块完成写入的时候（一个数据块的大小由kafka_max_block_size参数控制，默认情况下kafka_max_block_size=max_block_size=65536）。

  + 等待间隔超过7500毫秒，由stream_flush_interval_ms参数控制（默认7500 ms）。

​	Kafka表引擎底层负责与Kafka通信的部分，是基于librdkafka实现的，这是一个由C++实现的Kafka库，项目地址为https://github.com/edenhill/librdkafka 。librdkafka提供了许多自定义的配置参数，例如在默认的情况下，它每次只会读取Kafka主题中最新的数据（auto.offset.reset=largest），如果将其改为earlies后，数据将会从头读取。更多的自定义参数可以在如下地址找到：https://github.com/edenhill/librdkafka/blob/master/CONFIGURATION.md 。

​	ClickHouse对librdkafka的自定义参数提供了良好的扩展支持。在ClickHouse的全局设置中，提供了一组Kafka标签，专门用于定义librdkafka的自定义参数。不过需要注意的是，<u>librdkafka的原生参数使用了点连接符，在ClickHouse中需要将其改为下划线的形式</u>，例如：

```xml
<kafka>
  //librdkafka中，此参数名是auto.offset.reset
  <auto_offset_reset>smallest</auto_offset_reset>
</kafka>
```

​	现在用一个例子说明Kafka表引擎的使用方法。首先需要准备Kafka的测试数据。假设Kafka(V 0.10.1)的环境已准备就绪，执行下面的命令在Kafka中创建主题：

```shell
# ./kafka-topics.sh --create --zookeeper hdp1.nauu.com:2181 --replication-factor
1 --partitions 1 --topic sales-queue
Created topic "sales-queue".
```

​	接着发送测试消息：

```shell
# ./kafka-console-producer.sh --broker-list hdp1.nauu.com:6667 --topic salesqueue
{"id":1,"code":"code1","name":"name1"}
{"id":2,"code":"code2","name":"name2"}
{"id":3,"code":"code3","name":"name3"}
```

​	验证测试消息是否发送成功：

```shell
# ./kafka-console-consumer.sh --bootstrap-server hdp1.nauu.com:6667 --topic
sales-queue --from-beginning
{"id":1,"code":"code1","name":"name1"}
{"id":2,"code":"code2","name":"name2"}
{"id":3,"code":"code3","name":"name3"}
```

​	Kafka端的相关准备工作完成之后就可以开始ClickHouse部分的工作了。首先新建一张数据表：

```sql
CREATE TABLE kafka_test(
  id UInt32,
  code String,
  name String
) ENGINE = Kafka()
SETTINGS
  kafka_broker_list = 'hdp1.nauu.com:6667',
  kafka_topic_list = 'sales-queue',
  kafka_group_name = 'chgroup',
  kafka_format = 'JSONEachRow',
  kafka_skip_broken_messages = 100
```

​	该数据表订阅了名为sales-queue的消息主题，且消费组的名称为chgroup，而消息的格式采用了JSONEachRow。在此之后，查询这张数据表就能够看到Kafka的数据了。

​	到目前为止，似乎一切都进展顺利，但如果此时再次执行SELECT查询会发现kafka_test数据表空空如也，这是怎么回事呢？这是因**为<u>Kafka表引擎在执行查询之后就会删除表内的数据</u>**。读到这里，各位应该能够猜到，刚才介绍的Kafka表引擎使用方法一定不是ClickHouse设计团队所期望的模式。Kafka表引擎的正确使用方式，如图8-1所示。

![img](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211124_2b1881da-4cc6-11ec-817d-38f9d3cd240d.png)

​	图8-1 使用Kafka表引擎作为数据管道用途的示意图

​	在上图中，整个拓扑分为三类角色：

+ 首先是Kafka数据表A，它充当的角色是一条数据管道，负责拉取Kafka中的数据。

+ 接着是另外一张任意引擎的数据表B，它充当的角色是面向终端用户的查询表，在生产环境中通常是MergeTree系列。

+ 最后，是一张物化视图C，它负责将表A的数据实时同步到表B。

​	现在用一个具体的示例演示这种使用方法。首先新建一张Kafka引擎的表，让其充当数据管道：

```sql
CREATE TABLE kafka_queue(
  id UInt32,
  code String,
  name String
) ENGINE = Kafka()
SETTINGS
  kafka_broker_list = 'hdp1.nauu.com:6667',
  kafka_topic_list = 'sales-queue',
  kafka_group_name = 'chgroup',
  kafka_format = 'JSONEachRow',
  kafka_skip_broken_messages = 100
```

​	接着，新建一张面向终端用户的查询表，这里使用MergeTree表引擎：

```sql
CREATE TABLE kafka_table (
  id UInt32,
  code String,
  name String
) ENGINE = MergeTree()
ORDER BY id
```

​	最后，新建一张物化视图，用于将数据从kafka_queue同步到kafka_table：

```sql
CREATE MATERIALIZED VIEW consumer TO kafka_table
AS SELECT id,code,name FROM kafka_queue
```

​	至此，全部的工作就完成了。现在可以继续向Kafka主题发送消息，数据查询则只需面向kafka_table：

```sql
SELECT * FROM kafka_table
┌─id─┬─code──┬─name──┐
│ 1 │ code1 │ name1 │
│ 1 │ code1 │ name1 │
│ 2 │ code2 │ name2 │
│ 3 │ code3 │ name3 │
└───┴─────┴─────┘
```

​	如果需要停止数据同步，则可以删除视图：

```sql
DROP TABLE consumer
```

​	或者将其卸载：

```sql
DETACH TABLE consumer
```

​	在卸载了视图之后，如果想要再次恢复，可以使用装载命令：

```sql
ATTACH MATERIALIZED VIEW consumer TO kafka_table(
  id UInt32,
  code String,
  name String
)
AS SELECT id, code, name FROM kafka_queue
```

### * 8.1.5 File

​	<u>File表引擎能够直接读取本地文件的数据，通常被作为一种扩充手段来使用</u>。例如：它可以读取由其他系统生成的数据文件，如果外部系统直接修改了文件，则变相达到了数据更新的目的；它可以将ClickHouse数据导出为本地文件；它还可以用于数据格式转换等场景。除此以外，File表引擎也被应用于clickhouse-local工具（参见第3章相关内容）。

​	File表引擎的声明方式如下所示：

```sql
ENGINE = File(format)
```

​	其中，format表示文件中的数据格式，其类型必须是ClickHouse支持的数据格式，例如TSV、CSV和JSONEachRow等。可以发现，<u>在File表引擎的定义参数中，并没有包含文件路径这一项</u>。所以，**File表引擎的数据文件只能保存在config.xml配置中由path指定的路径下**。

​	<u>每张File数据表均由目录和文件组成，其中目录以表的名称命名，而数据文件则固定以data.format命名</u>，例如：

```shell
<ch-path>/data/default/test_file_table/data.CSV
```

​	创建File表目录和文件的方式有自动和手动两种。首先介绍自动创建的方式，即由File表引擎全权负责表目录和数据文件的创建：

```sql
CREATE TABLE file_table (
  name String,
  value UInt32
) ENGINE = File("CSV")
```

​	当执行完上面的语句后，在\<ch-path\>/data/default路径下便会创建一个名为file_table的目录。此时在该目录下还没有数据文件，接着写入数据：

```sql
INSERT INTO file_table VALUES ('one', 1), ('two', 2), ('three', 3)
```

​	在数据写入之后，file_table目录下便会生成一个名为data.CSV的数据文件：

```shell
# pwd
/chbase/data/default/file_table
# cat ./data.CSV
"one",1
"two",2
"three",3
```

​	可以看到数据被写入了文件之中。

​	接下来介绍手动创建的形式，即表目录和数据文件由ClickHouse之外的其他系统创建，例如使用shell创建：

```shell
//切换到clickhouse用户，以确保ClickHouse有权限读取目录和文件
# su clickhouse
//创建表目录
# mkdir /chbase/data/default/file_table1
//创建数据文件
# mv /chbase/data/default/file_table/data.CSV /chbase/data/default/file_table1
```

​	在表目录和数据文件准备妥当之后，挂载这张数据表：

```sql
ATTACH TABLE file_table1(
  name String,
  value UInt32
)ENGINE = File(CSV)
```

​	查询file_table1内的数据：

```sql
SELECT * FROM file_table1
┌─name──┬─value─┐
│ one │ 1 │
│ two │ 2 │
│ three │ 3 │
└─────┴─────┘
```

​	可以看到，file_table1同样读取到了文件内的数据。

​	即便是手动创建的表目录和数据文件，仍然可以对数据表插入数据，例如：

```sql
INSERT INTO file_table1 VALUES ('four', 4), ('five', 5)
```

​	File表引擎会在数据文件中追加数据：

```sql
# cat /chbase/data/default/file_table1/data.CSV
"one",1
"two",2
"three",3
"four",4
"five",5
```

​	可以看到，新写入的数据被追加到了文件尾部。

​	至此，File表引擎的基础用法就介绍完毕了。灵活运用这些方法，就能够实现开篇提到的一些典型场景。

## 8.2 内存类型

​	接下来将要介绍的几款表引擎，都是<u>面向内存查询的，数据会从内存中被直接访问，所以它们被归纳为内存类型</u>。但这并不意味着内存类表引擎不支持物理存储，事实上，**除了Memory表引擎之外，其余的几款表引擎都会将数据写入磁盘，这是为了防止数据丢失，是一种故障恢复手段**。而在数据表被加载时，它们会将数据全部加载至内存，以供查询之用。将数据全量放在内存中，对于表引擎来说是一把双刃剑：一方面，这意味着拥有较好的查询性能；而另一方面，如果表内装载的数据量过大，可能会带来极大的内存消耗和负担。

### * 8.2.1 Memory

​	**Memory表引擎直接将数据保存在内存中，数据既不会被压缩也不会被格式转换，数据在内存中保存的形态与查询时看到的如出一辙**。正因为如此，当ClickHouse服务重启的时候，Memory表内的数据会全部丢失。所以在一些场合，会将Memory作为测试表使用，很多初学者在学习ClickHouse的时候所写的Hello World程序很可能用的就是Memory表。<u>由于不需要磁盘读取、序列化以及反序列等操作，所以Memory表引擎支持并行查询，并且在简单的查询场景中能够达到与MergeTree旗鼓相当的查询性能（一亿行数据量以内）</u>。Memory表的创建方法如下所示：

```sql
CREATE TABLE memory_1 (
  id UInt64
)ENGINE = Memory()
```

​	当数据被写入之后，磁盘上不会创建任何数据文件。

​	最后需要说明的是，**相较于被当作测试表使用，Memory表更为广泛的应用场景是在ClickHouse的内部，它会作为集群间分发数据的存储载体来使用**。<u>例如**在分布式IN查询的场合中，会利用Memory临时表保存IN子句的查询结果，并通过网络将它传输到远端节点**</u>。关于这方面的更多细节，会在后续章节介绍。

### 8.2.2 Set

​	Set表引擎是拥有物理存储的，**数据首先会被写至内存，然后被同步到磁盘文件中**。<u>所以当服务重启时，它的数据不会丢失，当数据表被重新装载时，文件数据会再次被全量加载至内存</u>。众所周知，在Set数据结构中，所有元素都是唯一的。Set表引擎具有去重的能力，在数据写入的过程中，重复的数据会被自动忽略。然而Set表引擎的使用场景既特殊又有限，**它虽然支持正常的INSERT写入，但并不能直接使用SELECT对其进行查询，Set表引擎只能间接作为IN查询的右侧条件被查询使用**。

​	Set表引擎的存储结构由两部分组成，它们分别是：

+ [num].bin数据文件：保存了所有列字段的数据。其中，num是一个自增id，从1开始。**伴随着每一批数据的写入（每一次INSERT），都会生成一个新的.bin文件，num也会随之加1**。
+ tmp临时目录：<u>数据文件首先会被写到这个目录，当一批数据写入完毕之后，数据文件会被移出此目录</u>。

​	现在用一个示例说明Set表引擎的使用方法，首先新建一张数据表：

```sql
CREATE TABLE set_1 (
  id UInt8
)ENGINE = Set()
```

​	接着写入数据：

```sql
INSERT INTO TABLE set_1 SELECT number FROM numbers(10)
```

​	如果直接查询set_1，则会出现错误，因为Set表引擎不能被直接查询，例如：

```sql
SELECT * FROM set_1
DB::Exception: Method read is not supported by storage Set.
```

​	正确的查询方法是将Set表引擎作为IN查询的右侧条件，例如：

```sql
SELECT arrayJoin([1, 2, 3]) AS a WHERE a IN set_1
```

### 8.2.3 Join

​	Join表引擎可以说是为JOIN查询而生的，它等同于将JOIN查询进行了一层简单封装。<u>在Join表引擎的底层实现中，它与Set表引擎共用了大部分的处理逻辑，所以Join和Set表引擎拥有许多相似之处</u>。例如，Join表引擎的存储也由[num].bin数据文件和tmp临时目录两部分组成；<u>数据首先会被写至内存，然后被同步到磁盘文件</u>。**但是相比Set表引擎，Join表引擎有着更加广泛的应用场景，它既能够作为JOIN查询的连接表，也能够被直接查询使用**。

​	Join表引擎的声明方式如下所示：

```sql
ENGINE = Join(join_strictness, join_type, key1[, key2, ...])
```

​	其中，各参数的含义分别如下：

+ join_strictness：连接精度，它决定了JOIN查询在连接数据时所使用的策略，目前支持ALL、ANY和ASOF三种类型。

+ join_type：连接类型，它决定了JOIN查询组合左右两个数据集合的策略，它们所形成的结果是交集、并集、笛卡儿积或其他形式，目前支持INNER、OUTER和CROSS三种类型。<u>当join_type被设置为ANY时，在数据写入时，join_key重复的数据会被自动忽略</u>。

+ join_key：连接键，它决定了使用哪个列字段进行关联。

​	上述这些参数中，每一条都对应了JOIN查询子句的语法规则，关于JOIN查询子句的更多介绍将会在后续章节展开。

​	接下来用一个具体的示例演示Join表引擎的用法。首先建立一张主表：

```sql
CREATE TABLE join_tb1(
  id UInt8,
  name String,
  time Datetime
) ENGINE = Log
```

​	向主表写入数据：

```sql
INSERT INTO TABLE join_tb1 VALUES (1,'ClickHouse','2019-05-01 12:00:00'),
(2,'Spark', '2019-05-01 12:30:00'),(3,'ElasticSearch','2019-05-01 13:00:00')
```

​	接着创建Join表：

```sql
CREATE TABLE id_join_tb1(
  id UInt8,
  price UInt32,
  time Datetime
) ENGINE = Join(ANY, LEFT, id)
```

​	其中，<u>join_strictness为ANY，所以join_key重复的数据会被忽略</u>：

```shell
INSERT INTO TABLE id_join_tb1 VALUES (1,100,'2019-05-01 11:55:00'),(1,105,'2019-
05-01 11:10:00'),(2,90,'2019-05-01 12:01:00'),(3,80,'2019-05-01 13:10:00'),
(5,70,'2019-05-01 14:00:00'),(6,60,'2019-05-01 13:50:00')
```

​	在刚才写入的数据中，存在两条id为1的重复数据，现在查询这张表：

```sql
SELECT * FROM id_join_tb1
┌─id─┬─price─┬─────────time────┐
│ 1 │ 100 │ 2019-05-01 11:55:00 │
│ 2 │ 90 │ 2019-05-01 12:01:00 │
│ 3 │ 80 │ 2019-05-01 13:10:00 │
│ 5 │ 70 │ 2019-05-01 14:00:00 │
│ 6 │ 60 │ 2019-05-01 13:50:00 │
└───┴─────┴───────────────┘
```

​	可以看到，只有第一条id为1的数据写入成功，重复的数据被自动忽略了。

​	从刚才的示例可以得知，Join表引擎是可以被直接查询的，但这种方式并不是Join表引擎的主战场，它的主战场显然应该是JOIN查询，例如：

```sql
SELECT id,name,price FROM join_tb1 LEFT JOIN id_join_tb1 USING(id)
┌─id─┬─name───────┬─price─┐
│ 1 │ ClickHouse │ 100 │
│ 2 │ Spark │ 90 │
│ 3 │ ElasticSearch │ 80 │
└───┴───────────┴─────┘
```

​	Join表引擎除了可以直接使用SELECT和JOIN访问之外，还可以通过join函数访问，例如：

```sql
SELECT joinGet('id_join_tb1', 'price', toUInt8(1))
┌─joinGet('id_join_tb1', 'price', toUInt8(1))─┐
│ 100 │
└────────────────────────────┘
```

### * 8.2.4 Buffer

​	**Buffer表引擎完全使用内存装载数据，不支持文件的持久化存储，所以当服务重启之后，表内的数据会被清空**。**<u>Buffer表引擎不是为了面向查询场景而设计的，它的作用是充当缓冲区的角色</u>**。

​	假设有这样一种场景，我们需要将数据写入目标MergeTree表A，由于写入的并发数很高，这可能会导致MergeTree表A的合并速度慢于写入速度（因为每一次INSERT都会生成一个新的分区目录）。此时，可以引入Buffer表来缓解这类问题，将Buffer表作为数据写入的缓冲区。<u>数据首先被写入Buffer表，当满足预设条件时，Buffer表会自动将数据刷新到目标表</u>。

​	Buffer表引擎的声明方式如下所示：

```sql
ENGINE = Buffer(database, table, num_layers, min_time, max_time, min_rows,
max_rows, min_bytes, max_bytes)
```

​	其中，参数可以分成基础参数和条件参数两类，首先说明基础参数的作用：

+ database：目标表的数据库。
+ table：目标表的名称，Buffer表内的数据会自动刷新到目标表。
+ num_layers：可以理解成线程数，Buffer表会按照num_layers的数量开启线程，以并行的方式将数据刷新到目标表，官方建议设为16。

​	**Buffer表并不是实时刷新数据的，只有在阈值条件满足时它才会刷新**。阈值条件由三组最小和最大值组成。接下来说明三组极值条件参数的具体含义：

+ min_time和max_time：时间条件的最小和最大值，单位为秒，从第一次向表内写入数据的时候开始计算；

+ min_rows和max_rows：数据行条件的最小和最大值；

+ min_bytes和max_bytes：数据体量条件的最小和最大值，单位为字节。

​	**根据上述条件可知，Buffer表刷新的判断依据有三个，满足其中任意一个，Buffer表就会刷新数据**，它们分别是：

+ 如果三组条件中所有的最小阈值都已满足，则触发刷新动作；

+ 如果三组条件中至少有一个最大阈值条件满足，则触发刷新动作；

+ 如果写入的一批数据的数据行大于max_rows，或者数据体量大于max_bytes，则数据直接被写入目标表。

​	还有一点需要注意，**<u>上述三组条件在每一个num_layers中都是单独计算的</u>**。假设num_layers=16，则Buffer表最多会开启16个线程来响应数据的写入，**它们以轮询的方式接收请求**，**在每个线程内，会独立进行上述条件判断的过程**。<u>也就是说，假设一张Buffer表的max_bytes=100000000（约100 MB），num_layers=16，那么这张Buffer表能够同时处理的最大数据量约是1.6 GB</u>。

​	现在用一个示例演示它的用法。首先新建一张Buffer表buffer_to_memory_1：

```sql
CREATE TABLE buffer_to_memory_1 AS memory_1
ENGINE = Buffer(default, memory_1, 16, 10, 100, 10000, 1000000, 10000000,
100000000)
```

​	buffer_to_memory_1将memory_1作为数据输送的目标，所以必须使用与memory_1相同的表结构。

​	接着向Buffer表写入100万行数据：

```sql
INSERT INTO TABLE buffer_to_memory_1 SELECT number FROM numbers(1000000)
```

​	此时，buffer_to_memory_1内有数据，而目标表memory_1是没有的，因为目前不论从时间、数据行还是数据大小来判断，没有一个达到了最大阈值。所以在大致100秒之后，数据才会从buffer_to_memory_1刷新到memory_1。可以在ClickHouse的日志中发现相关记录信息：

```sql
<Trace> StorageBuffer (buffer_to_memory_1): Flushing buffer with 1000000 rows,
1000000 bytes, age 101 seconds.
```

​	接着，再次写入数据，这一次写入一百万零一行数据：

```sql
INSERT INTO TABLE buffer_to_memory_1 SELECT number FROM numbers(1000001)
```

​	查询目标表，可以看到数据不经等待即被直接写入目标表：

```sql
SELECT COUNT(*) FROM memory_1
┌─COUNT()─┐
│ 2000001 │
└──────┘
```

## 8.3 日志类型

​	**如果使用的数据量很小（100万以下），面对的数据查询场景也比较简单，并且是<u>“一次”写入多次查询</u>的模式，那么使用日志家族系列的表引擎将会是一种不错的选择**。

​	与合并树家族表引擎类似，日志家族系列的表引擎也拥有一些共性特征。例如：<u>它们均不支持索引、分区等高级特性；不支持并发读写，当针对一张日志表写入数据时，针对这张表的查询会被阻塞，直至写入动作结束；但它们也同时拥有切实的物理存储，数据会被保存到本地文件中</u>。除了这些共同的特征之外，日志家族系列的表引擎也有着各自的特点。接下来，会按照性能由低到高的顺序逐个介绍它们的使用方法。

### 8.3.1 TinyLog

​	**TinyLog是日志家族系列中性能最低的表引擎**，它的存储结构由<u>数据文件</u>和<u>元数据</u>两部分组成。其中，**数据文件是按列独立存储的**，也就是说每一个列字段都拥有一个与之对应的.bin文件。这种结构和MergeTree有些相似，但是**TinyLog既不支持分区，也没有.mrk标记文件**。<u>**由于没有标记文件，它自然无法支持.bin文件的并行读取操作，所以它只适合在非常简单的场景下使用**</u>。接下来用一个示例说明它的用法。首先创建一张TinyLog表：

```sql
CREATE TABLE tinylog_1 (
  id UInt64,
  code UInt64
)ENGINE = TinyLog()
```

​	接着，对其写入数据：

```sql
INSERT INTO TABLE tinylog_1 SELECT number,number+1 FROM numbers(100)
```

​	数据写入后就能够通过SELECT语句对它进行查询了。现在找到它的文件目录，分析一下它的存储结构：

```shell
# pwd
/chbase/data/default/tinylog_1
ll
total 12
-rw-r-----. 1 clickhouse clickhouse 432 23:39 code.bin
-rw-r-----. 1 clickhouse clickhouse 430 23:39 id.bin
-rw-r-----. 1 clickhouse clickhouse 66 23:39 sizes.json
```

​	可以看到，在表目录之下，id和code字段分别生成了各自的.bin数据文件。现在进一步查看sizes.json文件：

```shell
# cat ./sizes.json
{"yandex":{"code%2Ebin":{"size":"432"},"id%2Ebin":{"size":"430"}}}
```

​	由上述操作发现，在sizes.json文件内使用JSON格式记录了每个.bin文件内对应的数据大小的信息。

### 8.3.2 StripeLog

​	StripeLog表引擎的存储结构由固定的3个文件组成，它们分别是：

+ data.bin：数据文件，所有的列字段使用同一个文件保存，它们的数据都会被写入data.bin。
+ index.mrk：数据标记，保存了数据在data.bin文件中的位置信息。利用数据标记能够使用多个线程，以并行的方式读取data.bin内的压缩数据块，从而提升数据查询的性能。
+ sizes.json：元数据文件，记录了data.bin和index.mrk大小的信息。

​	从上述信息能够得知，相比TinyLog而言，**StripeLog拥有更高的查询性能（拥有.mrk标记文件，支持并行查询），同时其使用了更少的文件描述符（所有数据使用同一个文件保存）**。

​	接下来用一个示例说明它的用法。首先创建一张StripeLog表：

```sql
CREATE TABLE spripelog_1 (
  id UInt64,
  price Float32
)ENGINE = StripeLog()
```

​	接着，对其写入数据：

```sql
INSERT INTO TABLE spripelog_1 SELECT number,number+100 FROM numbers(1000)
```

​	写入之后，就可以使用SELECT语句对它进行查询了。现在，同样找到它的文件目录，下面分析它的存储结构：

```shell
# pwd
/chbase/data/default/spripelog_1
# ll
total 16
-rw-r-----. 1 clickhouse clickhouse 8121 01:10 data.bin
-rw-r-----. 1 clickhouse clickhouse 70 01:10 index.mrk
-rw-r-----. 1 clickhouse clickhouse 69 01:10 sizes.json
```

​	在表目录下，StripeLog表引擎创建了3个固定文件，现在进一步查看sizes.json：

```shell
# cd /chbase/data/default/spripelog_1
# cat ./sizes.json
{"yandex":{"data%2Ebin":{"size":"8121"},"index%2Emrk":{"size":"70"}}}
```

​	在sizes.json文件内，使用JSON格式记录了每个data.bin和index.mrk文件内对应的数据大小的信息。

### 8.3.3 Log

​	**Log表引擎结合了TinyLog表引擎和StripeLog表引擎的长处，是日志家族系列中性能最高的表引擎**。Log表引擎的存储结构由3个部分组成：

+ [column].bin：数据文件，数据文件按列独立存储，每一个列字段都拥有一个与之对应的.bin文件。
+ __marks.mrk：数据标记，统一保存了数据在各个[column].bin文件中的位置信息。利用数据标记能够使用多个线程，以并行的方式读取.bin内的压缩数据块，从而提升数据查询的性能。
+ sizes.json：元数据文件，记录了[column].bin和__marks.mrk大小的信息。

​	从上述信息能够得知，**由于拥有数据标记且各列数据独立存储，所以Log既能够支持并行查询，又能够按列按需读取，而付出的代价仅仅是比StripeLog消耗更多的文件描述符（每个列字段都拥有自己的.bin文件）**。

​	接下来用一个示例说明它的用法。首先创建一张Log：

```sql
CREATE TABLE log_1 (
  id UInt64,
  code UInt64
)ENGINE = Log()
```

​	接着，对其写入数据：

```sql
INSERT INTO TABLE log_1 SELECT number,number+1 FROM numbers(200)
```

​	数据写入之后就能够通过SELECT语句对它进行查询了。现在，再次找到它的文件目录，对它的存储结构进行分析：

```shell
# pwd
/chbase/data/default/log_1
# ll
total 16
-rw-r-----. 1 clickhouse clickhouse 432 23:55 code.bin
-rw-r-----. 1 clickhouse clickhouse 430 23:55 id.bin
-rw-r-----. 1 clickhouse clickhouse 32 23:55 __marks.mrk
-rw-r-----. 1 clickhouse clickhouse 96 23:55 sizes.json
```

​	可以看到，在表目录下，各个文件与先前表引擎中的文件如出一辙。现在进一步查看sizes.json文件：

```shell
# cd /chbase/data/default/log_1
# cat ./sizes.json
{"yandex":{"__marks%2Emrk":{"size":"32"},"code%2Ebin":{"size":"432"},"id%2Ebin":
{"size":"430"}}}
```

​	在sizes.json文件内，使用JSON格式记录了每个[column].bin和__marks.mrk文件内对应的数据大小的信息。

## * 8.4 接口类型

​	有这么一类表引擎，它们自身并不存储任何数据，而是像黏合剂一样可以整合其他的数据表。**在使用这类表引擎的时候，不用担心底层的复杂性，它们就像接口一样，为用户提供了统一的访问界面，所以我将它们归为接口类表引擎**。

### * 8.4.1 Merge

​	假设有这样一种场景：在数据仓库的设计中，数据按年分表存储，例如test_table_2018、test_table_2019和test_table_2020。假如现在需要跨年度查询这些数据，应该如何实现呢？在这情形下，使用Merge表引擎是一种合适的选择了。

​	Merge表引擎就如同一层使用了门面模式的代理，**它本身不存储任何数据，也不支持数据写入**。它的作用就如其名，即**负责合并多个查询的结果集**。<u>Merge表引擎可以代理查询任意数量的数据表，这些查询会异步且并行执行，并最终合成一个结果集返回</u>。

​	**<u>被代理查询的数据表被要求处于同一个数据库内，且拥有相同的表结构，但是它们可以使用不同的表引擎以及不同的分区定义（对于MergeTree而言）</u>**。Merge表引擎的声明方式如下所示：

```sql
ENGINE = Merge(database, table_name)
```

​	其中：database表示数据库名称；table_name表示数据表的名称，它**支持使用正则表达式**，例如^test表示合并查询所有以test为前缀的数据表。

​	现在用一个简单示例说明Merge的使用方法，假设数据表test_table_2018保存了整个2018年度的数据，它数据结构如下所示：

```sql
CREATE TABLE test_table_2018(
  id String,
  create_time DateTime,
  code String
)ENGINE = MergeTree
PARTITION BY toYYYYMM(create_time)
ORDER BY id
```

​	表test_table_2019的结构虽然与test_table_2018相同，但是它使用了不同的表引擎：

```sql
CREATE TABLE test_table_2019(
  id String,
  create_time DateTime,
  code String
)ENGINE = Log
```

​	现在创建一张Merge表，将上述两张表组合：

```sql
CREATE TABLE test_table_all as test_table_2018
ENGINE = Merge(currentDatabase(), '^test_table_')
```

​	其中，Merge表test_table_all直接复制了test_table_2018的表结构，它会合并当前数据库中所有以^test_table_开头的数据表。创建Merge之后，就可以查询这张Merge表了：	

```sql
SELECT _table,* FROM test_table_all
┌─_table────────┬─id──┬───────create_time─┬─code─┐
│ test_table_2018 │ A001 │ 2018-06-01 11:00:00 │ C2 │
└────────────┴────┴──────────────┴────┘
┌─_table────────┬─id──┬───────create_time─┬─code─┐
│ test_table_2018 │ A000 │ 2018-05-01 17:00:00 │ C1 │
│ test_table_2018 │ A002 │ 2018-05-01 12:00:00 │ C3 │
└────────────┴────┴──────────────┴────┘
┌─_table────────┬─id──┬───────create_time─┬─code─┐
│ test_table_2019 │ A020 │ 2019-05-01 17:00:00 │ C1 │
│ test_table_2019 │ A021 │ 2019-06-01 11:00:00 │ C2 │
│ test_table_2019 │ A022 │ 2019-05-01 12:00:00 │ C3 │
└────────────┴────┴──────────────┴────┘
```

​	通过返回的结果集可以印证，所有以^test_table_为前缀的数据表被分别查询后进行了合并返回。

​	值得一提的是，在上述示例中用到了**虚拟字段\_table，它表示某行数据的来源表**。如果在查询语句中，将虚拟字段\_table作为过滤条件：

```sql
SELECT _table,* FROM test_table_all WHERE _table = 'test_table_2018'
```

​	那么**它将等同于索引，Merge表会忽略那些被排除在外的数据表，不会向它们发起查询请求**。

### 8.4.2 Dictionary

​	**Dictionary表引擎是数据字典的一层代理封装，它可以取代字典函数，让用户通过数据表查询字典**。字典内的数据被加载后，会全部保存到内存中，所以使用Dictionary表对字典性能不会有任何影响。声明Dictionary表的方式如下所示：

```sql
ENGINE = Dictionary(dict_name)
```

​	其中，dict_name对应一个已被加载的字典名称，例如下面的例子：

```sql
CREATE TABLE tb_test_flat_dict (
  id UInt64,
  code String,
  name String
)Engine = Dictionary(test_flat_dict);
```

​	tb_test_flat_dict等同于数据字典test_flat_dict的代理表，现在对它使用SELECT语句进行查询：

```sql
SELECT * FROM tb_test_flat_dict
┌─id─┬─code──┬─name─┐
│ 1 │ a0001 │ 研发部 │
│ 2 │ a0002 │ 产品部 │
│ 3 │ a0003 │ 数据部 │
│ 4 │ a0004 │ 测试部 │
└───┴─────┴────┘
```

​	由上可以看到，字典数据被如数返回。

​	<u>如果字典的数量很多，逐一为它们创建各自的Dictionary表未免过于烦琐。这时候可以使用Dictionary引擎类型的数据库来解决这个问题</u>，例如：

```sql
CREATE DATABASE test_dictionaries ENGINE = Dictionary
```

​	<u>上述语句创建了一个名为test_dictionaries的数据库，它使用了Dictionary类型的引擎。在这个数据库中，ClickHouse会自动为每个字典分别创建它们的Dictionary表</u>：

```sql
SELECT database,name,engine_full FROM system.tables WHERE database =
'test_dictionaries'
┌─database──────┬─name─────────────────┬─engine───┐
│ test_dictionaries │ test_cache_dict │ Dictionary │
│ test_dictionaries │ test_ch_dict │ Dictionary │
│ test_dictionaries │ test_flat_dict │ Dictionary │
└────────────┴────────────────────┴────────┘
```

​	由上可以看到，当前系统中所有已加载的数据字典都在这个数据库下创建了各自的Dictionary表。

### 8.4.3 Distributed

​	在数据库领域，当面对海量业务数据的时候，一种主流的做法是实施Sharding方案，即将一张数据表横向扩展到多个数据库实例。其中，每个数据库实例称为一个Shard分片，数据在写入时，需要按照预定的业务规则均匀地写至各个Shard分片；而在数据查询时，则需要在每个Shard分片上分别查询，最后归并结果集。所以为了实现Sharding方案，一款支持分布式数据库的中间件是必不可少的，例如Apache ShardingSphere。

​	**ClickHouse作为一款性能卓越的分布式数据库，自然是支持Sharding方案的，而Distributed表引擎就等同于Sharding方案中的数据库中间件。Distributed表引擎自身不存储任何数据，它能够作为分布式表的一层透明代理，在集群内部自动开展数据的写入分发以及查询路由工作**。关于Distributed表引擎的详细介绍，将会在后续章节展开。

## 8.5 其他类型

​	接下来将要介绍的几款表引擎，由于各自用途迥异，所以只好把它们归为其他类型。虽然这些表引擎的使用场景并不广泛，但仍建议大家了解它们的特性和使用方法。因为这些表引擎扩充了ClickHouse的能力边界，在一些特殊的场合，它们也能够发挥重要作用。

### * 8.5.1 Live View

​	虽然ClickHouse已经提供了准实时的数据处理手段，例如Kafka表引擎和物化视图，但是在应用层面，一直缺乏开放给用户的**事件监听机制**。所以从19.14版本开始，Click-House提供了一种全新的图——Live View。

​	Live View是一种特殊的视图，虽然它并不属于表引擎，但是因为它与数据表息息相关，所以我还是把Live View归类到了这里。**Live View的作用类似事件监听器，它能够将一条SQL查询结果作为监控目标，当目标数据增加时，Live View可以及时发出响应**。

​	若要使用Live View，首先需要将allow_experimental_live_view参数设置为1，可以执行如下语句确认参数是否设置正确：

```sql
SELECT name, value FROM system.settings WHERE name LIKE '%live_view%'
┌─name──────────────────────────────┬─value─┐
│ allow_experimental_live_view │ 1 │
└──────────────────────────────────┴─────┘
```

​	现在用一个示例说明它的使用方法。首先创建一张数据表，它将作为Live View的监听目标：

```sql
CREATE TABLE origin_table1(
  id UInt64
) ENGINE = Log
```

​	接着，创建一张Live View表示：

```sql
CREATE LIVE VIEW lv_origin AS SELECT COUNT(*) FROM origin_table1
```

​	然后，**执行watch命令以开启监听模式**：

```sql
WATCH lv_origin
┌─COUNT()─┬─_version─┐
│ 0 │ 1 │
└───────┴───────┘
↖ Progress: 1.00 rows, 16.00 B (0.07 rows/s., 1.07 B/s.)
```

​	如此一来，Live View就进入监听模式了。接着再开启另外一个客户端，向origin_table1写入数据：

```sql
INSERT INTO TABLE origin_table1 SELECT rand() FROM numbers(5)
```

​	此时再观察Live View，可以看到它做出了实时响应：

```sql
WATCH lv_origin
┌─COUNT()─┬─_version──┐
│ 0 │ 1 │
└──────┴────────┘
┌─COUNT()─┬─_version──┐
│ 5 │ 2 │
└──────┴────────┘
↓ Progress: 2.00 rows, 32.00 B (0.04 rows/s., 0.65 B/s.)
```

​	注意，**虚拟字段_version伴随着每一次数据的同步，它的位数都会加1**。

### * 8.5.2 Null

​	**Null表引擎的功能与作用，与Unix系统的空设备/dev/null很相似。如果用户向Null表写入数据，系统会正确返回，但是Null表会自动忽略数据，永远不会将它们保存。如果用户向Null表发起查询，那么它将返回一张空表**。

​	**<u>在使用物化视图的时候，如果不希望保留源表的数据，那么将源表设置成Null引擎将会是极好的选择</u>**。接下来，用一个具体示例来说明这种使用方法。

​	首先新建一张Null表：

```sql
CREATE TABLE null_table1(
  id UInt8
) ENGINE = Null
```

​	接着以null_table1为源表，建立一张**物化视图**：

```sql
CREATE MATERIALIZED VIEW view_table10
ENGINE = TinyLog
AS SELECT * FROM null_table1
```

​	现在向null_table1写入数据，会发现数据被顺利同步到了视图view_table10中，而源表null_table1依然空空如也。

### 8.5.3 URL

​	**URL表引擎的作用等价于HTTP客户端，它可以通过HTTP/HTTPS协议，直接访问远端的REST服务**。

+ 当执行SELECT查询的时候，底层会将其转换为GET请求的远程调用。
+ 而执行INSERT查询的时候，会将其转换为POST请求的远程调用。

​	URL表引擎的声明方式如下所示：

```sql
ENGINE = URL('url', format)
```

​	其中，url表示远端的服务地址，而format则是ClickHouse支持的数据格式，如TSV、CSV和JSON等。

​	接下来，用一个具体示例说明它的用法。下面的这段代码片段来自于一个通过NodeJS模拟实现的REST服务：

```javascript
/* GET users listing. */
router.get('/users', function(req, res, next) {
	var result = '';
	for (let i = 0; i < 5; i++) {
		result += '{"name":"nauu'+i+'"}\n';
	}
	res.send(result);
});
/* Post user. */
router.post('/ users'', function(req, res) {
	res.sendStatus(200)
});
```

​	该服务的访问路径是/users，其中，GET请求对应了用户查询功能；而POST请求对应了新增用户功能。现在新建一张URL表：

```sql
CREATE TABLE url_table(
  name String
)ENGINE = URL('http://client1.nauu.com:3000/users', JSONEachRow)
```

​	其中，url参数对应了REST服务的访问地址，数据格式使用了JSONEachRow。

​	按如下方式执行SELECT查询：

```sql
SELECT * FROM url_table
```

​	此时SELECT会转换成一次GET请求，访问远端的HTTP服务：

```shell
<Debug> executeQuery: (from 10.37.129.2:62740) SELECT * FROM url_table
<Trace> ReadWriteBufferFromHTTP: Sending request to
http://client1.nauu.com:3000/users
```

​	最终，数据以表的形式被呈现在用户面前：

```sql
┌─name──┐
│ nauu0 │
│ nauu1 │
│ nauu2 │
│ nauu3 │
│ nauu4 │
└─────┘
```

​	按如下方式执行INSERT查询：

```sql
INSERT INTO TABLE url_table VALUES('nauu-insert')
```

​	INSERT会转换成一次POST请求，访问远端的HTTP服务：

```shell
<Debug> executeQuery: (from 10.37.129.2:62743) INSERT INTO TABLE url_table VALUES
<Trace> WriteBufferToHTTP: Sending request to http://client1.nauu.com:3000/users
```

## 8.6 本章小结

​	本章全面介绍了除第7章介绍的表引擎之外的其他类型的表引擎，知道了MergeTree家族表引擎之外还有另外5类表引擎。这些表引擎丰富了ClickHouse的使用场景，扩充了ClickHouse的能力界限。

​	**外部存储类型的表引擎与Hive的外挂表很相似，它们只负责元数据管理和数据查询，自身并不负责数据的生成，数据文件直接由外部系统维护**。它们可以直接读取HDFS、本地文件、常见关系型数据库和KafKa的数据。

​	**内存类型的表引擎中的数据是常驻内存的，所以它们拥有堪比MergeTree的查询性能（1亿数据量以内**）。其中Set和Join表引擎拥有物理存储，数据在写入内存的同时也会被刷新到磁盘；而Memory和Buffer表引擎在服务重启之后，数据便会被清空。**内存类表引擎是一把双刃剑，在数据大于1亿的场景下不建议使用内存类表引擎**。

​	**日志类型表引擎适用于数据量在100万以下，并且是“一次”写入多次查询的场景**。其中TinyLog、StripeLog和Log的性能依次升高的。

​	接口类型的表引擎自身并不存储任何数据，而是像黏合剂一样可以整合其他的数据表。其中Merge表引擎能够合并查询任意张表结构相同的数据表；Dictionary表引擎能够代理查询数据字典；而Distributed表引擎的作用类似分布式数据库的分表中间件，能够帮助用户简化数据的分发和路由工作。

​	助用户简化数据的分发和路由工作。其他类型的表引擎用途迥异。其中Live View是一种特殊的视图，能够对SQL查询进行准实时监听；<u>Null表引擎类似于Unix系统的空设备/dev/null，通常与物化视图搭配使用</u>；而URL表引擎类似于HTTP客户端，能够代理调用远端的REST服务。

# 9. 数据查询

​	作为一款OLAP分析型数据库，我相信大家在绝大部分时间内都在使用它的查询功能。在日常运转的过程中，数据查询也是ClickHouse的主要工作之一。ClickHouse完全使用SQL作为查询语言，能够以SELECT查询语句的形式从数据库中选取数据，这也是它具备流行潜质的重要原因。虽然ClickHouse拥有优秀的查询性能，但是我们也不能滥用查询，掌握ClickHouse所支持的各种查询子句，并选择合理的查询形式是很有必要的。使用不恰当的SQL语句进行查询不仅会带来低性能，还可能导致不可预知的系统错误。

​	虽然在之前章节的部分示例中，我们已经见识过一些查询语句的用法，但那些都是为了演示效果简化后的代码，与真正的生产环境中的代码相差较大。例如在绝大部分场景中，都应该避免使用`SELECT *`形式来查询数据，因为通配符`*`对于采用列式存储的ClickHouse而言没有任何好处。假如面对一张拥有数百个列字段的数据表，下面这两条SELECT语句的性能可能会相差100倍之多：

```sql
--使用通配符*与按列按需查询相比，性能可能相差100倍
SELECT * FROM datasets.hits_v1;
SELECT WatchID FROM datasets.hits_v1;
```

​	**<u>ClickHouse对于SQL语句的解析是大小写敏感的</u>**，这意味着SELECT a和SELECT A表示的语义是不相同的。ClickHouse目前支持的查询子句如下所示：

```sql
[WITH expr |(subquery)]
SELECT [DISTINCT] expr
[FROM [db.]table | (subquery) | table_function] [FINAL]
[SAMPLE expr]
[[LEFT] ARRAY JOIN]
[GLOBAL] [ALL|ANY|ASOF] [INNER | CROSS | [LEFT|RIGHT|FULL [OUTER]] ] JOIN
(subquery)|table ON|USING columns_list
[PREWHERE expr]
[WHERE expr]
[GROUP BY expr] [WITH ROLLUP|CUBE|TOTALS]
[HAVING expr]
[ORDER BY expr]
[LIMIT [n[,m]]
[UNION ALL]
[INTO OUTFILE filename]
[FORMAT format]
[LIMIT [offset] n BY columns]
```

​	其中，方括号包裹的查询子句表示其为可选项，所以只有SELECT子句是必须的，而ClickHouse对于查询语法的解析也大致是按照上面各个子句排列的顺序进行的。在本章后续会正视ClickHouse的本地查询部分，并大致依照各子句的解析顺序系统性地介绍它们的使用方法，而分布式查询部分则留待第10章介绍。

## * 9.1 WITH子句

​	**ClickHouse支持CTE（Common Table Expression，公共表表达式）**，以增强查询语句的表达。例如下面的函数嵌套：

```sql
SELECT pow(pow(2, 2), 3)
```

​	在改用CTE的形式后，可以极大地提高语句的可读性和可维护性，简化后的语句如下所示：

```sql
WITH pow(2, 2) AS a SELECT pow(a, 3)
```

​	CTE通过WITH子句表示，目前支持以下四种用法。

1. 定义变量

   ​	可以定义变量，这些变量能够在后续的查询子句中被直接访问。例如下面示例中的常量start，被直接用在紧接的WHERE子句中：

   ```sql
   WITH 10 AS start
   SELECT number FROM system.numbers
   WHERE number > start
   LIMIT 5
   ┌number─┐
   │ 11 │
   │ 12 │
   │ 13 │
   │ 14 │
   │ 15 │
   └─────┘
   ```

2. 调用函数

   ​	可以访问SELECT子句中的列字段，并调用函数做进一步的加工处理。例如在下面的示例中，对data_uncompressed_bytes使用聚合函数求和后，又紧接着在SELECT子句中对其进行了格式化处理：

   ```sql
   WITH SUM(data_uncompressed_bytes) AS bytes
   SELECT database , formatReadableSize(bytes) AS format FROM system.columns
   GROUP BY database
   ORDER BY bytes DESC
   ┌─database────┬─format───┐
   │ datasets │ 12.12 GiB │
   │ default │ 1.87 GiB │
   │ system │ 1.10 MiB │
   │ dictionaries │ 0.00 B │
   └─────────┴───────┘
   ```

3. 定义子查询

   **可以定义子查询**。例如在下面的示例中，借助子查询可以得出各database未压缩数据大小与数据总和大小的比例的排名：

   ```sql
   WITH (
     SELECT SUM(data_uncompressed_bytes) FROM system.columns
   ) AS total_bytes
   SELECT database , (SUM(data_uncompressed_bytes) / total_bytes) * 100 AS
   database_disk_usage
   FROM system.columns
   GROUP BY database
   ORDER BY database_disk_usage DESC
   ┌─database────┬──database_disk_usage─┐
   │ datasets │ 85.15608638238845 │
   │ default │ 13.15591656190217 │
   │ system │ 0.007523354055850406 │
   │ dictionaries │ 0 │
   └──────────┴──────────────┘
   ```

   ​	**<u>在WITH中使用子查询时有一点需要特别注意，该查询语句只能返回一行数据，如果结果集的数据大于一行则会抛出异常</u>**。

4. 在子查询中重复使用WITH

   ​	**在子查询中可以嵌套使用WITH子句**，例如在下面的示例中，在计算出各database未压缩数据大小与数据总和的比例之后，又进行了取整函数的调用：

   ```sql
   WITH (
     round(database_disk_usage)
   ) AS database_disk_usage_v1
   SELECT database,database_disk_usage, database_disk_usage_v1
   FROM (
     --嵌套
     WITH (
       SELECT SUM(data_uncompressed_bytes) FROM system.columns
     ) AS total_bytes
     SELECT database , (SUM(data_uncompressed_bytes) / total_bytes) * 100 AS
     database_disk_usage FROM system.columns
     GROUP BY database
     ORDER BY database_disk_usage DESC
   )
   ┌─database────┬───database_disk_usage─┬─database_disk_usage_v1───┐
   │ datasets │ 85.15608638238845 │ 85 │
   │ default │ 13.15591656190217 │ 13 │
   │ system │ 0.007523354055850406 │ 0 │
   └─────────┴───────────────┴─────────────────┘
   ```

## 9.2 FROM子句

​	FROM子句表示从何处读取数据，目前支持如下3种形式。

（1）从数据表中取数：

```sql
SELECT WatchID FROM hits_v1
```

（2）从子查询中取数：

```sql
SELECT MAX_WatchID
FROM (SELECT MAX(WatchID) AS MAX_WatchID FROM hits_v1)
```

（3）从表函数中取数：

```sql
SELECT number FROM numbers(5)
```

​	**<u>FROM关键字可以省略，此时会从虚拟表中取数。在ClickHouse中，并没有数据库中常见的DUAL虚拟表，取而代之的是system.one</u>**。例如下面的两条查询语句，其效果是等价的：

```sql
SELECT 1
SELECT 1 FROM system.one
┌─1─┐
│ 1 │
└───┘
```

​	**在FROM子句后，可以使用Final修饰符**。<u>它可以配合CollapsingMergeTree和Versioned-CollapsingMergeTree等表引擎进行查询操作，以强制在查询过程中合并，但由于Final修饰符会降低查询性能，所以应该尽可能避免使用它</u>。

## * 9.3 SAMPLE子句

​	**SAMPLE子句能够实现数据采样的功能，使查询仅返回采样数据而不是全部数据，从而有效减少查询负载**。**<u>SAMPLE子句的采样机制是一种幂等设计</u>**，也就是说在数据不发生变化的情况下，使用相同的采样规则总是能够返回相同的数据，所以这项特性非常适合在那些可以接受近似查询结果的场合使用。例如在数据量十分巨大的情况下，对查询时效性的要求大于准确性时就可以尝试使用SAMPLE子句。

​	**<u>SAMPLE子句只能用于MergeTree系列引擎的数据表，并且要求在CREATE TABLE时声明SAMPLE BY抽样表达式</u>**，例如下面的语句：

```sql
CREATE TABLE hits_v1 (
  CounterID UInt64,
  EventDate DATE,
  UserID UInt64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(EventDate)
ORDER BY (CounterID, intHash32(UserID))
--Sample Key声明的表达式必须也包含在主键的声明中
SAMPLE BY intHash32(UserID)
```

​	<u>SAMPLE BY表示hits_v1内的数据，可以按照intHash32(UserID)分布后的结果采样查询</u>。

​	在声明Sample Key的时候有两点需要注意：

+ **<u>SAMPLE BY所声明的表达式必须同时包含在主键的声明内</u>**；

+ **<u>Sample Key必须是Int类型，如若不是，ClickHouse在进行CREATE TABLE操作时也不会报错，但在数据查询时会得到如下类似异常</u>**：

  ```shell
  Invalid sampling column type in storage parameters: Float32. Must be unsigned
  integer type.
  ```

SAMPLE子句目前支持如下3种用法。

1. SAMPLE factor

   SAMPLE factor表示按因子系数采样，其中factor表示采样因子，它的取值支持0～1之间的小数。**如果factor设置为0或者1，则效果等同于不进行数据采样**。如下面的语句表示按10%的因子采样数据：

   ```sql
   SELECT CounterID FROM hits_v1 SAMPLE 0.1
   ```

   factor也支持使用十进制的形式表述：

   ```sql
   SELECT CounterID FROM hits_v1 SAMPLE 1/10
   ```

   **在进行统计查询时，为了得到最终的近似结果，需要将得到的直接结果乘以采样系数。例如若想按0.1的因子采样数据，则需要将统计结果放大10倍**：

   ```sql
   SELECT count() * 10 FROM hits_v1 SAMPLE 0.1
   ```

   **一种更为优雅的方法是借助虚拟字段`_sample_factor`来获取采样系数，并以此代替硬编码的形式**。`_sample_factor`可以返回当前查询所对应的采样系数：

   ```sql
   SELECT CounterID, _sample_factor FROM hits_v1 SAMPLE 0.1 LIMIT 2
   ┌─CounterID─┬─_sample_factor───┐
   │ 57 │ 10 │
   │ 57 │ 10 │
   └────────┴─────────────┘
   ```

   在使用`_sample_factor`之后，可以将之前的查询语句改写成如下形式：

   ```sql
   SELECT count() * any(_sample_factor) FROM hits_v1 SAMPLE 0.1
   ```

2. SAMPLE rows

   **SAMPLE rows表示按样本数量采样，其中rows表示至少采样多少行数据，<u>它的取值必须是大于1的整数</u>。如果rows的取值大于表内数据的总行数，则效果等于rows=1（即不使用采样）**。

   下面的语句表示采样10000行数据：

   ```sql
   SELECT count() FROM hits_v1 SAMPLE 10000
   ┌─count()─┐
   │ 9576 │
   └──────┘
   ```

   最终查询返回了9576行数据，从返回的结果中可以得知，**数据采样的范围是一个近似范围**，<u>**这是由于采样数据的最小粒度是由`index_granularity`索引粒度决定的**</u>。由此可知，设置一个小于索引粒度或者较小的rows值没有什么意义，应该设置一个较大的值。

   同样可以使用虚拟字段`_sample_factor`来获取当前查询所对应的采样系数：

   ```sql
   SELECT CounterID,_sample_factor FROM hits_v1 SAMPLE 100000 LIMIT 1
   ┌─CounterID─┬─_sample_factor─┐
   │ 63 │ 13.27104 │
   └───────┴──────────┘
   ```

3. SAMPLE factor OFFSET n

   <u>SAMPLE factor OFFSET n表示按因子系数和偏移量采样，其中factor表示采样因子，n表示偏移多少数据后才开始采样，它们两个的取值都是0～1之间的小数</u>。例如下面的语句表示偏移量为0.5并按0.4的系数采样：

   ```sql
   SELECT CounterID FROM hits_v1 SAMPLE 0.4 OFFSET 0.5
   ```

   <u>上述代码最终的查询会从数据的二分之一处开始，按0.4的系数采样数据</u>。

   **如果在计算OFFSET偏移量后，按照SAMPLE比例采样出现了溢出，则数据会被自动截断**。

   这种用法支持使用十进制的表达形式，也支持虚拟字段`_sample_factor`：

   ```sql
   SELECT CounterID,_sample_factor FROM hits_v1 SAMPLE 1/10 OFFSET 1/2
   ```

## * 9.4 ARRAY JOIN子句

​	**ARRAY JOIN子句允许在数据表的内部，与数组或嵌套类型的字段进行JOIN操作，从而将一行数组展开为多行**。接下来让我们看看它的基础用法。首先新建一张包含Array数组字段的测试表：

```sql
CREATE TABLE query_v1
(
  title String,
  value Array(UInt8)
) ENGINE = Log
```

​	接着写入测试数据，注意最后一行数据的数组为空：

```sql
INSERT INTO query_v1 VALUES ('food', [1,2,3]), ('fruit', [3,4]), ('meat', [])
SELECT title,value FROM query_v1
┌─title─┬─value───┐
│ food │ [1,2,3] │
│ fruit │ [3,4] │
│ meat │ [] │
└─────┴───────┘
```

​	**在一条SELECT语句中，只能存在一个ARRAY JOIN（使用子查询除外）。目前支持INNER和LEFT两种JOIN策略**：

1. INNER ARRAY JOIN

   **<u>ARRAY JOIN在默认情况下使用的是INNER JOIN策略</u>**，例如下面的语句：

   ```sql
   SELECT title,value FROM query_v1 ARRAY JOIN value
   ┌─title─┬─value──┐
   │ food │ 1 │
   │ food │ 2 │
   │ food │ 3 │
   │ fruit │ 3 │
   │ fruit │ 4 │
   └─────┴──────┘
   ```

   从查询结果可以发现，最终的数据基于value数组被展开成了多行，并且**<u>排除掉了空数组</u>**。在使用ARRAY JOIN时，如果为原有的数组字段添加一个别名，则能够访问展开前的数组字段，例如：

   ```sql
   SELECT title,value,v FROM query_v1 ARRAY JOIN value AS v
   ┌─title─┬─value──┬─v─┐
   │ food │ [1,2,3] │ 1 │
   │ food │ [1,2,3] │ 2 │
   │ food │ [1,2,3] │ 3 │
   │ fruit │ [3,4] │ 3 │
   │ fruit │ [3,4] │ 4 │
   └─────┴──────┴───┘
   ```

2. LEFT ARRAY JOIN

   ARRAY JOIN子句支持LEFT连接策略，例如执行下面的语句：

   ```sql
   SELECT title,value,v FROM query_v1 LEFT ARRAY JOIN value AS v
   ┌─title─┬─value──┬─v─┐
   │ food │ [1,2,3] │ 1 │
   │ food │ [1,2,3] │ 2 │
   │ food │ [1,2,3] │ 3 │
   │ fruit │ [3,4] │ 3 │
   │ fruit │ [3,4] │ 4 │
   │ meat │ [] │ 0 │
   └─────┴──────┴──┘
   ```

   **<u>在改为LEFT连接查询后，可以发现，在INNER JOIN中被排除掉的空数组出现在了返回的结果集中</u>**。

   **当同时对<u>多个数组字段</u>进行ARRAY JOIN操作时，查询的计算逻辑是<u>按行合并</u>而不是产生笛卡儿积**，例如下面的语句：

   ```sql
   -- ARRAY JOIN多个数组时，是合并，不是笛卡儿积
   SELECT title,value,v ,arrayMap(x -> x * 2,value) as mapv,v_1 FROM query_v1 LEFT
   ARRAY JOIN value AS v , mapv as v_1
   ┌─title─┬─value──┬─v─┬─mapv───┬─v_1─┐
   │ food │ [1,2,3] │ 1 │ [2,4,6] │ 2 │
   │ food │ [1,2,3] │ 2 │ [2,4,6] │ 4 │
   │ food │ [1,2,3] │ 3 │ [2,4,6] │ 6 │
   │ fruit │ [3,4] │ 3 │ [6,8] │ 6 │
   │ fruit │ [3,4] │ 4 │ [6,8] │ 8 │
   │ meat │ [] │ 0 │ [] │ 0 │
   └─────┴──────┴───┴──────┴────┘
   ```

   value和mapv**数组是按行合并的，并没有产生笛卡儿积**。

   在前面介绍数据定义时曾介绍过，**<u>嵌套数据类型的本质是数组，所以ARRAY JOIN也支持嵌套数据类型</u>**。接下来继续用一组示例说明。首先新建一张包含嵌套类型的测试表：

   ```sql
   --ARRAY JOIN嵌套类型
   CREATE TABLE query_v2
   (
     title String,
     nest Nested(
       v1 UInt32,
       v2 UInt64)
   ) ENGINE = Log
   ```

   接着写入测试数据，在写入嵌套数据类型时，记得同一行数据中各个数组的长度需要对齐，而对多行数据之间的数组长度没有限制：

   ```sql
   -- 同一行数据，数组长度要对齐
   INSERT INTO query_v2 VALUES ('food', [1,2,3], [10,20,30]), ('fruit', [4,5],
   [40,50]), ('meat', [], [])
   SELECT title, nest.v1, nest.v2 FROM query_v2
   ┌─title─┬─nest.v1─┬─nest.v2───┐
   │ food │ [1,2,3] │ [10,20,30] │
   │ fruit │ [4,5] │ [40,50] │
   │ meat │ [] │ [] │
   └─────┴──────┴────────┘
   ```

   **对嵌套类型数据的访问，ARRAY JOIN既可以直接使用字段列名**：

   ```sql
   SELECT title, nest.v1, nest.v2 FROM query_v2 ARRAY JOIN nest
   ```

   也可以使用点访问符的形式：

   ```sql
   SELECT title, nest.v1, nest.v2 FROM query_v2 ARRAY JOIN nest.v1, nest.v2
   ```

   上述两种形式的查询效果完全相同：

   ```sql
   ┌─title─┬─nest.v1─┬─nest.v2─┐
   │ food │ 1 │ 10 │
   │ food │ 2 │ 20 │
   │ food │ 3 │ 30 │
   │ fruit │ 4 │ 40 │
   │ fruit │ 5 │ 50 │
   └─────┴───────┴───────┘
   ```

   **嵌套类型也支持ARRAY JOIN部分嵌套字段**：

   ```sql
   --也可以只ARRAY JOIN其中部分字段
   SELECT title, nest.v1, nest.v2 FROM query_v2 ARRAY JOIN nest.v1
   ┌─title─┬─nest.v1─┬─nest.v2───┐
   │ food │ 1 │ [10,20,30] │
   │ food │ 2 │ [10,20,30] │
   │ food │ 3 │ [10,20,30] │
   │ fruit │ 4 │ [40,50] │
   │ fruit │ 5 │ [40,50] │
   └─────┴──────┴────────┘
   ```

   可以看到，在这种情形下，只有被ARRAY JOIN的数组才会展开。

   在查询嵌套类型时也能够通过别名的形式访问原始数组：

   ```sql
   SELECT title, nest.v1, nest.v2, n.v1, n.v2 FROM query_v2 ARRAY JOIN nest as n
   ┌─title─┬─nest.v1─┬─nest.v2───┬─n.v1─┬─n.v2─┐
   │ food │ [1,2,3] │ [10,20,30] │ 1 │ 10 │
   │ food │ [1,2,3] │ [10,20,30] │ 2 │ 20 │
   │ food │ [1,2,3] │ [10,20,30] │ 3 │ 30 │
   │ fruit │ [4,5] │ [40,50] │ 4 │ 40 │
   │ fruit │ [4,5] │ [40,50] │ 5 │ 50 │
   └──────┴──────┴────────┴────┴─────┘
   ```

## 9.5 JOIN子句

​	JOIN子句可以对左右两张表的数据进行连接，这是最常用的查询子句之一。<u>它的语法包含**连接精度**和**连接类型**两部分</u>。目前ClickHouse支持的JOIN子句形式如图9-3所示。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210616173303639.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1MTQxMTA1,size_16,color_FFFFFF,t_70)

图9-3 JOIN子句组合规则

​	由上图可知，**连接精度分为ALL、ANY和ASOF三种，而连接类型也可分为外连接、内连接和交叉连接三种**。

​	除此之外，<u>JOIN查询还可以根据其执行策略被划分为**本地查询**和**远程查询**</u>。关于远程查询的内容放在后续章节进行说明，这里着重讲解本地查询。接下来，会基于下面三张测试表介绍JOIN用法。

​	代码清单9-1 JOIN测试表join_tb1

```sql
┌─id─┬─name──────┬─────────time─┐
│ 1 │ ClickHouse │ 2019-05-01 12:00:00 │
│ 2 │ Spark │ 2019-05-01 12:30:00 │
│ 3 │ ElasticSearch │ 2019-05-01 13:00:00 │
│ 4 │ HBase │ 2019-05-01 13:30:00 │
│ NULL │ ClickHouse │ 2019-05-01 12:00:00 │
│ NULL │ Spark │ 2019-05-01 12:30:00 │
└────┴─────────┴─────────────┘
```

​	代码清单9-2 JOIN测试表join_tb2

```sql
┌─id─┬─rate─┬─────────time─┐
│ 1 │ 100 │ 2019-05-01 11:55:00 │
│ 2 │ 90 │ 2019-05-01 12:01:00 │
│ 3 │ 80 │ 2019-05-01 13:10:00 │
│ 5 │ 70 │ 2019-05-01 14:00:00 │
│ 6 │ 60 │ 2019-05-01 13:50:00 │
└───┴────┴─────────────┘
```

​	代码清单9-3 JOIN测试表join_tb3

```sql
┌─id─┬─star─┐
│ 1 │ 1000 │
│ 2 │ 900 │
└───┴────┘
```

### 9.5.1 连接精度

​	连接精度决定了JOIN查询在连接数据时所使用的策略，目前支持ALL、ANY和ASOF三种类型。如果不主动声明，则**默认是ALL**。可以通过join_default_strictness配置参数修改默认的连接精度类型。

​	**对数据是否连接匹配的判断是通过JOIN KEY进行的，<u>目前只支持等式（EQUAL JOIN）</u>**。交叉连接（CROSS JOIN）不需要使用JOIN KEY，因为它会产生笛卡儿积。

1. ALL

   **如果左表内的一行数据，在右表中有多行数据与之连接匹配，<u>则返回右表中全部连接的数据</u>**。而判断连接匹配的依据是左表与右表内的数据，基于连接键（JOIN KEY）的取值完全相等（equal），等同于left.key=right.key。例如执行下面的语句：

   ```sql
   SELECT a.id,a.name,b.rate FROM join_tb1 AS a
   ALL INNER JOIN join_tb2 AS b ON a.id = b.id
   ┌─id─┬─name──────┬─rate──┐
   │ 1 │ ClickHouse │ 100 │
   │ 1 │ ClickHouse │ 105 │
   │ 2 │ Spark │ 90 │
   │ 3 │ ElasticSearch │ 80 │
   └───┴─────────┴──────┘
   ```

   结果集返回了右表中所有与左表id相匹配的数据。

2. ANY

   **如果左表内的一行数据，在右表中有多行数据与之连接匹配，<u>则仅返回右表中第一行连接的数据</u>**。ANY与ALL判断连接匹配的依据相同。例如执行下面的语句：

   ```sql
   SELECT a.id,a.name,b.rate FROM join_tb1 AS a
   ANY INNER JOIN join_tb2 AS b ON a.id = b.id
   ┌─id─┬─name──────┬─rate──┐
   │ 1 │ ClickHouse │ 100 │
   │ 2 │ Spark │ 90 │
   │ 3 │ ElasticSearch │ 80 │
   └───┴─────────┴──────┘
   ```

   结果集仅返回了右表中与左表id相连接的第一行数据。

3. ASOF

   **<u>ASOF是一种模糊连接，它允许在连接键之后追加定义一个模糊连接的匹配条件asof_column</u>**。以下面的语句为例：

   ```sql
   SELECT a.id,a.name,b.rate,a.time,b.time
   FROM join_tb1 AS a ASOF INNER JOIN join_tb2 AS b
   ON a.id = b.id AND a.time = b.time
   ```

   **其中a.id=b.id是寻常的连接键，而紧随其后的a.time=b.time则是asof_column模糊连接条件**，这条语句的语义等同于：

   ```sql
   a.id = b.id AND a.time >= b.time
   ```

   执行上述这条语句后：

   ```sql
   ┌─id─┬─name────┬─rate─┬────time──────┬──────b.time───┐
   │ 1 │ ClickHouse │ 100 │ 2019-05-01 12:00:00 │ 2019-05-01 11:55:00 │
   │ 2 │ Spark │ 90 │ 2019-05-01 12:30:00 │ 2019-05-01 12:01:00 │
   └───┴───────┴────┴─────────────┴─────────────┘
   ```

   ​	由上可以得知，其最终返回的查询结果符合连接条件a.id=b.idAND a.time>=b.time，**且仅返回了右表中第一行连接匹配的数据**。

   ​	**ASOF支持使用USING的简写形式，USING后声明的<u>最后一个字段</u>会被自动转换成asof_colum模糊连接条件**。例如将上述语句改成USING的写法后，将会是下面的样子：

   ```sql
   SELECT a.id,a.name,b.rate,a.time,b.time FROM join_tb1 AS a ASOF
   INNER JOIN join_tb2 AS b USING(id,time)
   ```

   USING后的time字段会被转换成asof_colum。

   对于asof_colum字段的使用有两点需要注意：

   + **asof_colum必须是整型、浮点型和日期型这类有序序列的数据类型**；
   + **asof_colum不能是数据表内的唯一字段，换言之，连接键（JOIN KEY）和asof_colum不能是同一个字段**。

### * 9.5.2 连接类型

​	连接类型决定了JOIN查询组合左右两个数据集合要用的策略，它们所形成的结果是交集、并集、笛卡儿积或是其他形式。接下来会分别介绍这几种连接类型的使用方法。

1. INNER

   INNER JOIN表示内连接，在查询时会以左表为基础逐行遍历数据，然后从右表中找出与左边连接的行，它只会返回左表与右表两个数据集合中交集的部分，其余部分都会被排除。

   ![在这里插入图片描述](https://img-blog.csdnimg.cn/202106161739108.png)

   在前面介绍连接精度时所用的演示用例中，使用的正是INNER JOIN，它的使用方法如下所示：

   ```sql
   SELECT a.id,a.name,b.rate FROM join_tb1 AS a
   INNER JOIN join_tb2 AS b ON a.id = b.id
   ┌─id─┬─name──────┬─rate──┐
   │ 1 │ ClickHouse │ 100 │
   │ 1 │ ClickHouse │ 105 │
   │ 2 │ Spark │ 90 │
   │ 3 │ ElasticSearch │ 80 │
   └────┴─────────┴──────┘
   ```

   从返回的结果集能够得知，只有左表与右表中id完全相同的数据才会保留，也就是<u>只保留交集部分</u>。

2. OUTER

   <u>OUTER JOIN表示外连接，它可以进一步细分为左外连接（LEFT）、右外连接（RIGHT）和全外连接（FULL）三种形式</u>。根据连接形式的不同，其返回数据集合的逻辑也不尽相同。OUTER JOIN查询的语法如下所示：

   ```sql
   [LEFT|RIGHT|FULL [OUTER]] ] JOIN
   ```

   其中，**OUTER修饰符可以省略**。

   1. LEFT

      在进行左外连接查询时，会**以左表为基础逐行遍历数据**，然后从右表中找出与左边连接的行以补齐属性。<u>如果在右表中没有找到连接的行，则采用相应字段数据类型的默认值填充</u>。换言之，对于左连接查询而言，**左表的数据总是能够全部返回**。

      ![在这里插入图片描述](https://img-blog.csdnimg.cn/202106161739108.png)

      左外连接查询的示例语句如下所示：

      ```sql
      SELECT a.id,a.name,b.rate FROM join_tb1 AS a
      LEFT OUTER JOIN join_tb2 AS b ON a.id = b.id
      ┌─id──┬─name──────┬─rate──┐
      │ 1 │ ClickHouse │ 100 │
      │ 1 │ ClickHouse │ 105 │
      │ 2 │ Spark │ 90 │
      │ 3 │ ElasticSearch │ 80 │
      │ │ │ │
      │ 4 │ HBase │ 0 │
      └────┴─────────┴──────┘
      ```

      由查询的返回结果可知，左表join_tb1内的数据全部返回，其中id为4的数据在右表中没有连接，所以由默认值0补全。

   2. RIGHT

      右外连接查询的效果与左连接恰好相反，**右表的数据总是能够全部返回**，而<u>左表不能连接的数据则使用默认值补全</u>。

      ![在这里插入图片描述](https://img-blog.csdnimg.cn/202106161739108.png)

      在进行右外连接查询时，内部的执行逻辑大致如下：

      （1）<u>在内部进行类似INNER JOIN的内连接查询，在计算交集部分的同时，顺带记录右表中那些未能被连接的数据行</u>。

      （2）**将那些未能被连接的数据行追加到交集的尾部**。

      （3）将追加数据中那些属于左表的列字段用默认值补全。

      右外连接查询的示例语句如下所示：

      ```sql
      SELECT a.id,a.name,b.rate FROM join_tb1 AS a
      RIGHT JOIN join_tb2 AS b ON a.id = b.id
      ┌─id──┬─name──────┬─rate──┐
      │ 1 │ ClickHouse │ 100 │
      │ 1 │ ClickHouse │ 105 │
      │ 2 │ Spark │ 90 │
      │ 3 │ ElasticSearch │ 80 │
      │ 5 │ │ 70 │
      │ │ │ │
      │ 6 │ │ 60 │
      └────┴─────────┴──────┘
      ```

      由查询的返回结果可知，右表join_tb2内的数据全部返回，在左表中没有被连接的数据由默认值补全。

   3. FULL

      全外连接查询会返回左表与右表两个数据集合的并集。

      ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210616174401891.png)

      全外连接内部的执行逻辑大致如下：

      （1）<u>会在内部进行类似LEFT JOIN的查询，在左外连接的过程中，顺带记录右表中已经被连接的数据行</u>。

      （2）通过在右表中记录已被连接的数据行，得到未被连接的数据行。

      （3）**将右表中未被连接的数据追加至结果集，并将那些属于左表中的列字段以默认值补全**。

      全外连接查询的示例如下所示。

      ```sql
      SELECT a.id,a.name,b.rate FROM join_tb1 AS a
      FULL JOIN join_tb2 AS b ON a.id = b.id
      ┌─id──┬─name──────┬─rate──┐
      │ 1 │ ClickHouse │ 100 │
      │ │ │ │
      │ 1 │ ClickHouse │ 105 │
      │ 2 │ Spark │ 90 │
      │ 3 │ ElasticSearch │ 80 │
      │ 4 │ HBase │ 0 │
      │ 5 │ │ 70 │
      │ 6 │ │ 60 │
      └────┴─────────┴──────┘
      ```

3. CROSS

   CROSS JOIN表示交叉连接，它会返回左表与右表两个数据集合的**笛卡儿积**。也正因为如此，CROSS JOIN不需要声明JOIN KEY，因为结果会包含它们的所有组合，如下面的语句所示：

   ```sql
   SELECT a.id,a.name,b.rate FROM join_tb1 AS a
   CROSS JOIN join_tb2 AS b
   ┌─id──┬─name──────┬─rate──┐
   │ 1 │ ClickHouse │ 100 │
   │ 1 │ ClickHouse │ 105 │
   │ 1 │ ClickHouse │ 90 │
   │ 1 │ ClickHouse │ 80 │
   │ 1 │ ClickHouse │ 70 │
   │ 1 │ ClickHouse │ 60 │
   │ 2 │ Spark │ 100 │
   │ 2 │ Spark │ 105 │
   │ 2 │ Spark │ 90 │
   │ 2 │ Spark │ 80 │
   …省略
   ```

   上述语句返回的结果是两张数据表的笛卡儿积。

   在进行交叉连接查询时，会**以左表为基础，逐行与右表全集相乘**。

### * 9.5.3 多表连接

​	**<u>在进行多张数据表的连接查询时，ClickHouse会将它们转为两两连接的形式</u>**，例如执行下面的语句，对三张测试表进行内连接查询：

```sql
SELECT a.id,a.name,b.rate,c.star FROM join_tb1 AS a
INNER JOIN join_tb2 AS b ON a.id = b.id
LEFT JOIN join_tb3 AS c ON a.id = c.id
┌─a.id─┬─a.name────┬─b.rate──┬─c.star──┐
│ 1 │ ClickHouse │ 100 │ 1000 │
│ 1 │ ClickHouse │ 105 │ 1000 │
│ 2 │ Spark │ 90 │ 900 │
│ 3 │ ElasticSearch │ 80 │ 0 │
└─────┴─────────┴───────┴───────┘
```

​	在执行上述查询时，**会先将join_tb1与join_tb2进行内连接，之后再将它们的结果集与join_tb3左连接**。

​	**<u>ClickHouse虽然也支持关联查询的语法，但是会自动将其转换成指定的连接查询</u>**。<u>要想使用这项特性，需要将allow_experimental_cross_to_join_conversion参数设置为1（默认为1，该参数在新版本中已经取消）</u>，它的转换规则如下：

+ 转换为CROSS JOIN：**如果查询语句中不包含WHERE条件，则会转为CROSS JOIN**。

  ```sql
  SELECT a.id,a.name,b.rate,c.star FROM join_tb1 AS a , join_tb2 AS b ,join_tb3 AS
  c
  ```

+ 转换为INNER JOIN：**如果查询语句中包含WHERE条件，则会转为INNER JOIN**。

  ```sql
  SELECT a.id,a.name,b.rate,c.star FROM join_tb1 AS a , join_tb2 AS b ,join_tb3 AS
  c WHERE a.id = b.id AND a.id = c.id
  ```

​	**<u>虽然ClickHouse支持上述语法转换特性，但并不建议使用，因为在编写复杂的业务查询语句时，我们无法确定最终的查询意图</u>**。

### * 9.5.4 注意事项

​	最后，还有两个关于JOIN查询的注意事项。

1. 关于性能

   为了能够优化JOIN查询性能，首先应该**遵循左大右小的原则** ，即将数据量小的表放在右侧。这是因为在执行JOIN查询时，<u>无论使用的是哪种连接方式，**右表都会被全部加载到内存中**与左表进行比较</u>。

   <u>其次，**JOIN查询目前没有缓存的支持** ，这意味着每一次JOIN查询，即便是连续执行相同的SQL，也都会生成一次全新的执行计划。如果应用程序会大量使用JOIN查询，则需要进一步考虑借助上层应用侧的缓存服务或使用JOIN表引擎来改善性能</u>。

   最后，<u>**如果是在大量维度属性补全的查询场景中，则建议使用字典代替JOIN查询** 。因为在进行多表的连接查询时，查询会转换成两两连接的形式，这种“滚雪球”式的查询很可能带来性能问题</u>。

2. 关于空值策略与简写形式

   细心的读者应该能够发现，在之前的介绍中，连接查询的空值（那些未被连接的数据）是由默认值填充的，这与其他数据库所采取的策略不同（由Null填充）。<u>连接查询的空值策略是通过join_use_nulls参数指定的，默认为0。当参数值为0时，空值由数据类型的默认值填充；而当参数值为1时，空值由Null填充</u>。

   <u>JOIN KEY支持简化写法，当数据表的连接字段名称相同时，可以使用USING语法简写</u>，例如下面两条语句的效果是等同的：

   ```sql
   SELECT a.id,a.name,b.rate FROM join_tb1 AS a
   INNER JOIN join_tb2 AS b ON a.id = b.id
   --USING简写
   SELECT id,name,rate FROM join_tb1 INNER JOIN join_tb2 USING id
   ```

## * 9.6 WHERE与PREWHERE子句

​	WHERE子句基于条件表达式来实现数据过滤。<u>如果过滤条件恰好是主键字段，则能够进一步借助索引加速查询，所以WHERE子句是一条查询语句能否启用索引的判断依据（前提是表引擎支持索引特性）</u>。例如下面的查询语句：

```sql
SELECT id,url,v1,v2,v3,v4 FROM query_v3 WHERE id = 'A000'
(SelectExecutor): Key condition: (column 0 in ['A000', 'A000'])
```

​	WHERE表达式中包含主键，所以它能够使用索引过滤数据区间。除此之外，ClickHouse还提供了全新的PREWHERE子句。

​	**<u>PREWHERE目前只能用于MergeTree系列的表引擎</u>**，它可以看作对WHERE的一种优化，其作用与WHERE相同，均是用来过滤数据。它们的不同之处在于，**<u>使用PREWHERE时，首先只会读取PREWHERE指定的列字段数据，用于数据过滤的条件判断。待数据过滤之后再读取SELECT声明的列字段以补全其余属性</u>**。所以在一些场合下，PREWHERE相比WHERE而言，处理的数据量更少，性能更高。

​	接下来，就让我们用一组具体的例子对比两者的差异。首先执行`set optimize_move_to_prewhere=0`强制关闭自动优化（关于这个参数，之后会进一步介绍），然后执行下面的语句：

```sql
-- 执行set optimize_move_to_prewhere=0关闭PREWHERE自动优化
SELECT WatchID,Title,GoodEvent FROM hits_v1 WHERE JavaEnable = 1
981110 rows in set. Elapsed: 0.095 sec. Processed 1.34 million rows, 124.65 MB
(639.61 thousand rows/s., 59.50 MB/s.)
```

​	从查询统计结果可以看到，此次查询总共处理了134万行数据，其数据大小为124.65 MB。

​	现在，将语句中的WHERE替换为PREWHERE，其余保持不变：

```sql
SELECT WatchID,Title,GoodEvent FROM hits_v1 PREWHERE JavaEnable = 1
981110 rows in set. Elapsed: 0.080 sec. Processed 1.34 million rows, 91.61 MB
(740.98 thousand rows/s., 50.66 MB/s.)
```

​	从PREWHERE语句的查询统计结果可以发现，虽然处理数据的总量没有发生变化，仍然是134万行数据，但是其数据大小从124.65MB减少至91.61 MB，从而提高了每秒处理数据的吞吐量，这种现象充分印证了PREWHERE的优化效果。**这是因为在执行PREWHERE查询时，只需获取JavaEnable字段进行数据过滤，减少了需要处理的数据量大小**。

​	进一步观察两次查询的执行计划，也能够发现它们查询过程之间的差异：

```sql
--WHERE查询
Union
	Expression × 2
		Expression
			Filter
				MergeTreeThread
--PREWHERE查询
Union
	Expression × 2
		Expression
			MergeTreeThread
```

​	由上可以看到，**PREWHERE查询省去了一次Filter操作**。

​	既然PREWHERE性能更优，那么是否需要将所有的WHERE子句都替换成PREWHERE呢？其实大可不必，因为**<u>ClickHouse实现了自动优化的功能，会在条件合适的情况下将WHERE替换为PREWHERE</u>**。**如果想开启这项特性，需要将optimize_move_to_prewhere设置为1（默认值为1，即开启状态）**，例如执行下面的语句：

```sql
SELECT id,url FROM query_v3 WHERE v1 = 10
```

​	通过观察执行日志可以发现，谓词v1=10被移动到了PREWHERE子句：

```shell
<Debug> InterpreterSelectQuery: MergeTreeWhereOptimizer: condition "v1 = 10"
moved to PREWHERE
```

​	但是也有例外情况，假设数据表query_v3所有的字段类型如下：

```sql
desc query_v3
┌─name──┬─type──────┬─default_type─┬─default_expression───┐
│ id │ String │ │ │
│ url │ String │ │ │
│ time │ Date │ │ │
│ v1 │ UInt8 │ │ │
│ v2 │ UInt8 │ │ │
│ nest.v1 │ Array(UInt32) │ │ │
│ nest.v2 │ Array(UInt64) │ │ │
│ v3 │ UInt8 │ MATERIALIZED │ CAST(v1 / v2, 'UInt8') │
│ v4 │ String │ ALIAS │ id │
└─────┴─────────┴─────────┴───────────────┘
```

则在**<u>以下情形时并不会自动优化</u>**：

+ **使用了常量表达式**：

  ```sql
  SELECT id,url,v1,v2,v3,v4 FROM query_v3 WHERE 1=1
  ```

+ **使用了默认值为ALIAS类型的字段**：

  ```sql
  SELECT id,url,v1,v2,v3,v4 FROM query_v3 WHERE v4 = 'A000'
  ```

+ **包含了arrayJoin、globalIn、globalNotIn或者indexHint的查询**：

  ```sql
  SELECT title, nest.v1, nest.v2 FROM query_v2 ARRAY JOIN nest WHERE nest.v1=1
  ```

+ SELECT查询的列字段与WHERE谓词相同：

  ```sql
  SELECT v3 FROM query_v3 WHERE v3 = 1
  ```

+ 使用了主键字段：

  ```sql
  SELECT id FROM query_v3 WHERE id = 'A000'
  ```

​	虽然在上述情形中ClickHouse不会自动将谓词移动到PREWHERE，但仍然可以主动使用PREWHERE。以主键字段为例，当使用PREWHERE进行主键查询时，首先会通过稀疏索引过滤数据区间（index_granularity粒度），接着会读取PREWHERE指定的条件列以进一步过滤，这样一来就有可能截掉数据区间的尾巴，从而返回低于index_granularity粒度的数据范围。<u>即便如此，相比其他场合移动谓词所带来的性能提升，这类效果还是比较有限的，所以目前ClickHouse在这类场合下仍然保持不移动的处理方式</u>。

## 9.7 GROUP BY子句

​	GROUP BY又称聚合查询，是最常用的子句之一，它是让ClickHouse最凸显卓越性能的地方。在GROUP BY后声明的表达式，通常称为聚合键或者Key，数据会按照聚合键进行聚合。在ClickHouse的聚合查询中，SELECT可以声明聚合函数和列字段，**如果SELECT后只声明了聚合函数，则可以省略GROUP BY关键字**：

```sql
--如果只有聚合函数，可以省略GROUP BY
SELECT SUM(data_compressed_bytes) AS compressed ,
SUM(data_uncompressed_bytes) AS uncompressed
FROM system.parts
```

​	**如若声明了列字段，则只能使用聚合键包含的字段，否则会报错**：

```sql
--除了聚合函数外，只能使用聚合key中包含的table字段
SELECT table,COUNT() FROM system.parts GROUP BY table
--使用聚合key中未声明的rows字段，则会报错
SELECT table,COUNT(),rows FROM system.parts GROUP BY table
```

​	但是在某些场合下，可以借助any、max和min等聚合函数访问聚合键之外的列字段：

```sql
SELECT table,COUNT(),any(rows) FROM system.parts GROUP BY table
┌─table────┬─COUNT()─┬─any(rows)─┐
│ partition_v1 │ 1 │ 4 │
│ agg_merge2 │ 1 │ 1 │
│ hits_v1 │ 2 │ 8873898 │
└────────┴───────┴────────┘
```

​	**<u>当聚合查询内的数据存在NULL值时，ClickHouse会将NULL作为NULL=NULL的特定值处理</u>**，例如：

```sql
SELECT arrayJoin([1, 2, 3,null,null,null]) AS v GROUP BY v
┌───v───┐
│ │
│ 1 │
│ 2 │
│ 3 │
│ NULL │
└───────┘
```

​	可以看到所有的NULL值都被聚合到了NULL分组。

​	除了上述特性之外，<u>聚合查询目前还能配合WITH ROLLUP、WITH CUBE和WITH TOTALS三种修饰符获取额外的汇总信息</u>。

### * 9.7.1 WITH ROLLUP

​	顾名思义，**ROLLUP能够按照聚合键从右向左上卷数据，<u>基于聚合函数依次生成分组小计和总计</u>。如果设聚合键的个数为n，则最终会生成小计的个数为n+1**。例如执行下面的语句：

```sql
SELECT table, name, SUM(bytes_on_disk) FROM system.parts GROUP BY table,name WITH
ROLLUP
ORDER BY table ┌─table──────┬─name──────┬─SUM(bytes_on_disk)─┐
│ │ │ 2938852157 │
│partition_v1 │ │ 670 │
│ partition_v1 │ 201906_6_6_0 │ 160 │
│ partition_v1 │ 201905_1_3_1 │ 175 │
│ partition_v1 │ 201905_5_5_0 │ 160 │
│ partition_v1 │ 201906_2_4_1 │ 175 │
│query_v4 │ │ 459 │
│ query_v4 │ 201906_2_2_0 │ 203 │
│ query_v4 │ 201905_1_1_0 │ 256 │
省略…
└─────────┴─────────┴─────────────┘
```

​	可以看到在最终返回的结果中，**附加返回了显示名称为空的小计汇总行**，<u>包括所有表分区磁盘大小的汇总合计以及每张table内所有分区大小的合计信息</u>。

### * 9.7.2 WITH CUBE

​	顾名思义，CUBE会像立方体模型一样，**基于聚合键之间所有的组合生成小计信息**。**<u>如果设聚合键的个数为n，则最终小计组合的个数为2的n次方</u>**。

​	接下来用示例说明它的用法。假设在数据库(database)default和datasets中分别拥有一张数据表hits_v1，现在需要通过查询system.parts系统表分别按照数据库database、表table和分区name分组合计汇总，则可以执行下面的语句：

```sql
SELECT database, table, name, SUM(bytes_on_disk) FROM
(SELECT database, table, name, bytes_on_disk FROM system.parts WHERE table
='hits_v1') GROUP BY database,table,name
WITH CUBE
ORDER BY database,table ,name
┌─database─┬─table──┬─name──────┬─SUM(bytes_on_disk) ──┐
│ │ │ │ 1460381504 │
│ │ │ 201403_1_29_2 │ 1271367153 │
│ │ │ 201403_1_6_1 │ 189014351 │
│ │ hits_v1 │ │ 1460381504 │
│ │ hits_v1 │ 201403_1_29_2 │ 1271367153 │
│ │ hits_v1 │ 201403_1_6_1 │ 189014351 │
│ │ │ │ │
│ datasets │ │ │ 1271367153 │
│ datasets │ │ 201403_1_29_2 │ 1271367153 │
│ datasets │ hits_v1 │ │ 1271367153 │
│ datasets │ hits_v1 │ 201403_1_29_2 │ 1271367153 │
│ default │ │ │ 189014351 │
│ default │ │ 201403_1_6_1 │ 189014351 │
│ default │ hits_v1 │ │ 189014351 │
│ default │ hits_v1 │ 201403_1_6_1 │ 189014351 │
└───────┴──────┴─────────┴──────────────┘
```

​	由返回结果可知，基于3个分区键database、table与name按照立方体CUBE组合后，形成了[空]、[database,table,name]、[database]、[table]、[name]、[database,table]、[database,name]和[table,name]共计8种小计组合，恰好是2的3次方。

### * 9.7.3 WITH TOTALS

​	**使用TOTALS修饰符后，会基于聚合函数对所有数据进行总计**，例如执行下面的语句：

```sql
SELECT database, SUM(bytes_on_disk),COUNT(table) FROM system.parts GROUP BY
database WITH TOTALS
┌─database─┬─SUM(bytes_on_disk)─┬─COUNT(table)─┐
│ default │ 378059851 │ 46 │
│ datasets │ 2542748913 │ 3 │
│ system │ 152144 │ 3 │
└───────┴─────────────┴─────────┘
Totals:
┌─database─┬─SUM(bytes_on_disk)─┬─COUNT(table)─┐
│ │ 2920960908 │ 52 │
└───────┴─────────────┴─────────┘
```

​	其结果附加了一行Totals汇总合计，**这一结果是基于聚合函数对所有数据聚合总计的结果**。

## 9.8 HAVING子句

​	HAVING子句需要与GROUP BY同时出现，不能单独使用。它能够在聚合计算之后实现二次过滤数据。例如下面的语句是一条普通的聚合查询，会按照table分组并计数：

```sql
SELECT COUNT() FROM system.parts GROUP BY table --执行计划
Expression
	Expression
		Aggregating
			Concat
				Expression One
```

​	现在增加HAVING子句后再次执行上述操作，则数据在按照table聚合之后，进一步截掉了table='query_v3'的部分。

```sql
SELECT COUNT() FROM system.parts GROUP BY table HAVING table = 'query_v3'
--执行计划
Expression
	Expression
		Filter
			Aggregating
				Concat
					Expression One
```

​	观察两次查询的执行计划，可以发现**<u>HAVING的本质是在聚合之后增加了Filter过滤动作</u>**。

​	对于类似上述的查询需求，除了使用HAVING之外，通过嵌套的WHERE也能达到相同的目的，例如下面的语句：

```sql
SELECT COUNT() FROM
(SELECT table FROM system.parts WHERE table = 'query_v3') GROUP BY table
--执行计划
Expression
	Expression
		Aggregating
			Concat
				Expression Expression Expression Filter One
```

​	分析上述查询的执行计划，**<u>相比使用HAVING，嵌套WHERE的执行计划效率更高</u>**。**因为WHERE等同于使用了谓词下推，在聚合之前就进行了数据过滤，从而减少了后续聚合时需要处理的数据量**。

​	既然如此，那是否意味着HAVING子句没有存在的意义了呢？其实不然，现在来看另外一种查询诉求。假设现在需要按照table分组聚合，并且返回均值bytes_on_disk大于10 000字节的数据表，在这种情形下需要使用HAVING子句：

```sql
SELECT table ,avg(bytes_on_disk) as avg_bytes FROM system.parts GROUP BY table
HAVING avg_bytes > 10000
┌─table─────┬───avg_bytes───┐
│ hits_v1 │ 730190752 │
└─────────┴────────────┘
```

​	这是因为WHERE的执行优先级大于GROUP BY，所以<u>如果需要按照聚合值进行过滤，就必须借助HAVING实现</u>。

## * 9.9 ORDER BY子句

​	ORDER BY子句通过声明排序键来指定查询数据返回时的顺序。通过先前的介绍大家知道，在MergeTree表引擎中也有ORDER BY参数用于指定排序键，那么这两者有何不同呢？**在MergeTree中指定ORDER BY后，数据在各个分区内会按照其定义的规则排序，这是一种分区内的局部排序**。如果在查询时数据跨越了多个分区，则它们的返回顺序是无法预知的，每一次查询返回的顺序都可能不同。<u>在这种情形下，如果需要数据总是能够按照期望的顺序返回，就需要借助ORDER BY子句来指定全局顺序</u>。

​	ORDER BY在使用时可以定义多个排序键，每个排序键后需紧跟ASC（升序）或DESC（降序）来确定排列顺序。**如若不写，则默认为ASC（升序）**。例如下面的两条语句即是等价的：

```sql
--按照v1升序、v2降序排序
SELECT arrayJoin([1,2,3]) as v1 , arrayJoin([4,5,6]) as v2
ORDER BY v1 ASC, v2 DESC
SELECT arrayJoin([1,2,3]) as v1 , arrayJoin([4,5,6]) as v2
ORDER BY v1, v2 DESC
```

​	数据首先会按照v1升序，接着再按照v2降序。

​	**对于数据中NULL值的排序，目前ClickHouse拥有NULL值最后和NULL值优先两种策略**，可以通过NULLS修饰符进行设置：

1. NULLS LAST

   **<u>NULL值排在最后，这也是默认行为</u>，修饰符可以省略**。在这种情形下，数据的排列顺序为其他值（value）→NaN→NULL。

   ```sql
   -- 顺序是value -> NaN -> NULL
   WITH arrayJoin([30,null,60.5,0/0,1/0,-1/0,30,null,0/0]) AS v1
   SELECT v1 ORDER BY v1 DESC NULLS LAST
   ┌───v1─┐
   │ inf │
   │ 60.5 │
   │ 30 │
   │ 30 │
   │ -inf │
   │ nan │
   │ nan │
   │ NULL │
   │ NULL │
   └─────┘
   ```

2. NULLS FIRST

   NULL值排在最前，在这种情形下，数据的排列顺序为NULL→NaN→其他值（value）：

   ```sql
   -- 顺序是NULL -> NaN -> value
   WITH arrayJoin([30,null,60.5,0/0,1/0,-1/0,30,null,0/0]) AS v1
   SELECT v1 ORDER BY v1 DESC NULLS FIRST
   ┌──v1─┐
   │ NULL │
   │ NULL │
   │ nan │
   │ nan │
   │ inf │
   │ 60.5 │
   │ 30 │
   │ 30 │
   │ -inf │
   └────┘
   ```

   ​	从上述的两组测试中不难发现，对于NaN而言，它总是紧跟在NULL的身边。

   + **在使用NULLS LAST策略时，NaN好像比所有非NULL值都小**；
   + **而在使用NULLS FIRST时，NaN又好像比所有非NULL值都大**。

## * 9.10 LIMIT BY子句

​	**<u>LIMIT BY子句和大家常见的LIMIT所有不同，它运行于ORDER BY之后和LIMIT之前，能够按照指定分组，最多返回前n行数据（如果数据少于n行，则按实际数量返回），常用于TOP N的查询场景</u>**。LIMIT BY的常规语法如下：

```sql
LIMIT n BY express
```

​	例如执行下面的语句后便能够在基于数据库和数据表分组的情况下，查询返回数据占磁盘空间最大的前3张表：

```sql
-- limit n by
SELECT database,table,MAX(bytes_on_disk) AS bytes FROM system.parts GROUP BY
database,table ORDER BY database ,bytes DESC
LIMIT 3 BY database
┌─database─┬─table───────┬───bytes──┐
│ datasets │ hits_v1 │ 1271367153 │
│ datasets │ hits_v1_1 │ 1269636153 │
│ default │ hits_v1_1 │ 189025442 │
│ default │ hits_v1 │ 189014351 │
│ default │ partition_v5 │ 5344 │
│ │ │ │
│ system │ query_log │ 81127 │
│ system │ query_thread_log │ 68838 │
└───────┴───────────┴────────┘
```

​	声明多个表达式需使用逗号分隔，例如下面的语句能够得到每张数据表所定义的字段中，使用最频繁的前5种数据类型：

```sql
SELECT database,table,type,COUNT(name) AS col_count FROM system.columns GROUP BY
database,table,type ORDER BY col_count DESC
LIMIT 5 BY database,table
```

​	除了常规语法以外，**LIMIT BY也支持跳过OFFSET偏移量获取数据**，具体语法如下：

```sql
LIMIT n OFFSET y BY express --简写
LIMIT y,n BY express
```

​	例如在执行下面的语句时，查询会从跳过1行数据的位置开始：

```sql
SELECT database,table,MAX(bytes_on_disk) AS bytes FROM system.parts GROUP BY
database,table ORDER BY bytes DESC
LIMIT 3 OFFSET 1 BY database
```

​	使用简写形式也能够得到相同效果：

```sql
SELECT database,table,MAX(bytes_on_disk) AS bytes FROM system.parts GROUP BY
database,table ORDER BY bytes DESC
LIMIT 1，3 BY database
```

## 9.11 LIMIT子句

​	LIMIT子句用于返回指定的前n行数据，常用于分页场景，它的三种语法形式如下所示：

```sql
LIMIT n
LIMIT n OFFSET m
LIMIT m，n
```

​	例如下面的语句，会返回前10行数据：

```sql
SELECT number FROM system.numbers LIMIT 10
```

​	从指定m行开始并返回前n行数据的语句如下：

```sql
SELECT number FROM system.numbers LIMIT 10 OFFSET 5
```

​	上述语句的简写形式如下：

```sql
SELECT number FROM system.numbers LIMIT 5 ,10
```

​	**LIMIT子句可以和LIMIT BY一同使用**，以下面的语句为例：

```sql
SELECT database,table,MAX(bytes_on_disk) AS bytes FROM system.parts
GROUP BY database,table ORDER BY bytes DESC
LIMIT 3 BY database
LIMIT 10
```

​	上述语句表示，查询返回数据占磁盘空间最大的前3张表，而返回的总数据行等于10。

​	**<u>在使用LIMIT子句时有一点需要注意，如果数据跨越了多个分区，在没有使用ORDER BY指定全局顺序的情况下，每次LIMIT查询所返回的数据有可能不同。如果对数据的返回顺序敏感，则应搭配ORDER BY一同使用</u>**。

## * 9.12 SELECT子句

​	SELECT子句决定了一次查询语句最终返回哪些列字段或表达式。与直观的感受不同，**虽然SELECT位于SQL语句的起始位置，但它却是在上述一众子句之后执行的**。<u>**在其他子句执行之后，SELECT会将选取的字段或表达式作用于每行数据之上**</u>。<u>如果使用`*`通配符，则会返回数据表的所有字段。正如本章开篇所言，在大多数情况下都不建议这么做，因为对于一款列式存储的数据库而言，这绝对是劣势而不是优势</u>。

​	**<u>在选择列字段时，ClickHouse还为特定场景提供了一种基于正则查询的形式</u>**。例如执行下面的语句后，查询会返回名称以字母n开头和包含字母p的列字段：

```sql
SELECT COLUMNS('^n'), COLUMNS('p') FROM system.databases
┌─name────┬─data_path────┬─metadata_path───┐
│ default │ /data/default/ │ /metadata/default/ │
│ system │ /data/system/ │ /metadata/system/ │
└───────┴──────────┴────────────┘
```

## 9.13 DISTINCT子句

​	DISTINCT子句能够去除重复数据，使用场景广泛。有时候，人们会拿它与GROUP BY子句进行比较。假设数据表query_v5的数据如下所示：

```sql
┌─name─┬─v1─┐
│ a │ 1 │
│ c │ 2 │
│ b │ 3 │
│ NULL │ 4 │
│ d │ 5 │
│ a │ 6 │
│ a │ 7 │
│ NULL │ 8 │
└────┴───┘
```

​	则下面两条SQL查询的返回结果相同：

```sql
-- DISTINCT查询
SELECT DISTINCT name FROM query_v5
-- DISTINCT查询执行计划
Expression
	Distinct
		Expression
		Log
-- GROUP BY查询
SELECT name FROM query_v5 GROUP BY name
-- GROUP BY查询执行计划
Expression
	Expression
		Aggregating
		Concat
			Expression
				Log
```

​	其中，第一条SQL语句使用了DISTINCT子句，第二条SQL语句使用了GROUP BY子句。但是观察它们执行计划不难发现，DISTINCT子句的执行计划更简单。与此同时，**DISTINCT也能够与GROUP BY同时使用，所以它们是互补而不是互斥的关系**。

​	**<u>如果使用了LIMIT且没有ORDER BY子句，则DISTINCT在满足条件时能够迅速结束查询，这样可避免多余的处理逻辑</u>；而当DISTINCT与ORDER BY同时使用时，其执行的优先级是先DISTINCT后ORDER BY**。例如执行下面的语句，首先以升序查询：

```sql
SELECT DISTINCT name FROM query_v5 ORDER BY v1 ASC
┌─name─┐
│ a │
│ c │
│ b │
│ NULL │
│ d │
└─────┘
```

​	接着再反转顺序，以倒序查询：

```sql
SELECT DISTINCT name FROM query_v5 ORDER BY v1 DESC
┌─name─┐
│ d │
│ NULL │
│ b │
│ c │
│ a │
└─────┘
```

​	从两组查询结果中能够明显看出，执行逻辑是先DISTINCT后ORDER BY。**对于NULL值而言，DISTINCT也遵循着NULL=NULL的语义，所有的NULL值都会归为一组**。

## * 9.14 UNION ALL子句

​	UNION ALL子句能够联合左右两边的两组子查询，将结果一并返回。**在一次查询中，可以声明多次UNION ALL以便联合多组查询，<u>但UNION ALL不能直接使用其他子句（例如ORDER BY、LIMIT等），这些子句只能在它联合的子查询中使用</u>**。下面用一组示例说明它的用法。首先准备用于测试的数据表以及测试数据：

```sql
CREATE TABLE union_v1
(
  name String,
  v1 UInt8
) ENGINE = Log
INSERT INTO union_v1 VALUES('apple',1),('cherry',2),('banana',3)
CREATE TABLE union_v2
(
  title Nullable(String), v1 Float32
) ENGINE = Log
INSERT INTO union_v2 VALUES('apple',20), (null,4.5),('orange',1.1),('pear',2.0),
('lemon',3.54)
```

现在执行联合查询：

```sql
SELECT name,v1 FROM union_v1
UNION ALL
SELECT title,v1 FROM union_v2
```

​	在上述查询中，对于UNION ALL两侧的子查询能够得到几点信息：

+ 首先，**列字段的数量必须相同**；
+ 其次，**列字段的数据类型必须相同或相兼容**；
+ 最后，**列字段的名称可以不同，查询结果中的列名会以左边的子查询为准**。

​	对于联合查询还有一点要说明，**目前ClickHouse只支持UNION ALL子句**，如果想得到UNION DISTINCT子句的效果，可以使用嵌套查询来变相实现，例如：

```sql
SELECT DISTINCT name FROM
(
  SELECT name,v1 FROM union_v1
  UNION ALL
  SELECT title,v1 FROM union_v2
)
```

## * 9.15 查看SQL执行计划

​	**ClickHouse目前并没有直接提供EXPLAIN查询**，但是<u>借助后台的服务日志，能变相实现该功能</u>。例如，执行下面的语句，就能看到SQL的执行计划：

```shell
clickhouse-client -h ch7.nauu.com --send_logs_level=trace <<< 'SELECT * FROM
hits_v1' > /dev/null
```

​	假设数据表hits_v1的关键属性如下所示：

```sql
CREATE TABLE hits_v1 (
  WatchID UInt64,
  EventDate DATE,
  CounterID UInt32,
  ...
)ENGINE = MergeTree()
PARTITION BY toYYYYMM(EventDate)
ORDER BY CounterID
SETTINGS index_granularity = 8192
```

​	其中，分区键是EventDate，主键是CounterID。在写入测试数据后，这张表的数据约900万行：

```shell
SELECT COUNT(*) FROM hits_v1
┌─COUNT()─┐
│ 8873910 │
└──────┘
1 rows in set. Elapsed: 0.010 sec.
```

​	另外，测试数据还拥有12个分区：

```shell
SELECT partition_id ,name FROM 'system'.parts WHERE 'table' = 'hits_v1' AND
active = 1
┌─partition_id┬─name──────┐
│ 201403 │ 201403_1_7_1 │
│ 201403 │ 201403_8_13_1 │
│ 201403 │ 201403_14_19_1 │
│ 201403 │ 201403_20_25_1 │
│ 201403 │ 201403_26_26_0 │
│ │ │
│ 201403 │ 201403_27_27_0 │
│ 201403 │ 201403_28_28_0 │
│ 201403 │ 201403_29_29_0 │
│ 201403 │ 201403_30_30_0 │
│ 201405 │ 201405_31_40_2 │
│ 201405 │ 201405_41_41_0 │
│ 201406 │ 201406_42_42_0 │
└────────┴─────────┘
12 rows in set. Elapsed: 0.008 sec.
```

​	因为数据刚刚写入完毕，所以名为201403的分区目前有8个，该阶段还没有将其最终合并成1个。

1. 全字段、全表扫描

   ​	首先，执行SELECT * FROM hits_v进行全字段、全表扫描，具体语句如下：

   ```shell
   [root@ch7 ~]# clickhouse-client -h ch7.nauu.com --send_logs_level=trace <<<
   'SELECT * FROM hits_v1' > /dev/null
   [ch7.nauu.com] 2020.03.24 21:17:18.197960 {910ebccd-6af9-4a3e-82f4-d2291686e319}
   [ 45 ] <Debug> executeQuery: (from 10.37.129.15:47198) SELECT * FROM hits_v1
   [ch7.nauu.com] 2020.03.24 21:17:18.200324 {910ebccd-6af9-4a3e-82f4-d2291686e319}
   [ 45 ] <Debug> default.hits_v1 (SelectExecutor): Key condition: unknown
   [ch7.nauu.com] 2020.03.24 21:17:18.200350 {910ebccd-6af9-4a3e-82f4-d2291686e319}
   [ 45 ] <Debug> default.hits_v1 (SelectExecutor): MinMax index condition: unknown
   [ch7.nauu.com] 2020.03.24 21:17:18.200453 {910ebccd-6af9-4a3e-82f4-d2291686e319}
   [ 45 ] <Debug> default.hits_v1 (SelectExecutor): Selected 12 parts by date, 12
   parts by key, 1098 marks to read from 12 ranges
   [ch7.nauu.com] 2020.03.24 21:17:18.205865 {910ebccd-6af9-4a3e-82f4-d2291686e319}
   [ 45 ] <Trace> default.hits_v1 (SelectExecutor): Reading approx. 8917216 rows
   with 2 streams
   [ch7.nauu.com] 2020.03.24 21:17:18.206333 {910ebccd-6af9-4a3e-82f4-d2291686e319}
   [ 45 ] <Trace> InterpreterSelectQuery: FetchColumns -> Complete
   [ch7.nauu.com] 2020.03.24 21:17:18.207143 {910ebccd-6af9-4a3e-82f4-d2291686e319}
   [ 45 ] <Debug> executeQuery: Query pipeline:
   Union
   Expression × 2
   Expression
   MergeTreeThread
   [ch7.nauu.com] 2020.03.24 21:17:46.460028 {910ebccd-6af9-4a3e-82f4-d2291686e319}
   [ 45 ] <Trace> UnionBlockInputStream: Waiting for threads to finish
   [ch7.nauu.com] 2020.03.24 21:17:46.463029 {910ebccd-6af9-4a3e-82f4-d2291686e319}
   [ 45 ] <Trace> UnionBlockInputStream: Waited for threads to finish
   [ch7.nauu.com] 2020.03.24 21:17:46.466535 {910ebccd-6af9-4a3e-82f4-d2291686e319}
   [ 45 ] <Information> executeQuery: Read 8873910 rows, 8.50 GiB in 28.267 sec.,
   313928 rows/sec., 308.01 MiB/sec.
   [ch7.nauu.com] 2020.03.24 21:17:46.466603 {910ebccd-6af9-4a3e-82f4-d2291686e319}
   [ 45 ] <Debug> MemoryTracker: Peak memory usage (for query): 340.03 MiB.
   ```

   ​	现在我们分析一下，从上述日志中能够得到什么信息。日志中打印了该SQL的执行计划：

   ```sql
   Union
   	Expression × 2
   		Expression
   			MergeTreeThread
   ```

   ​	<u>这条查询语句使用了2个线程执行，并最终通过Union合并了结果集</u>。

   ​	**该查询语句没有使用主键索引**，具体如下：

   ```shell
   Key condition: unknown
   ```

   ​	**该查询语句没有使用分区索引**，具体如下：

   ```shell
   MinMax index condition: unknown
   ```

   ​	该查询语句共扫描了12个分区目录，共计1098个MarkRange，具体如下：

   ```shell
   Selected 12 parts by date, 12 parts by key, 1098 marks to read from 12 ranges
   ```

   ​	该查询语句总共读取了8 873 910行数据（全表），共8.50GB，具体如下：

   ```shell
   Read 8873910 rows, 8.50 GiB in 28.267 sec., 313928 rows/sec., 308.01 MiB/sec.
   ```

   ​	该查询语句消耗内存最大时为340 MB：

   ```shell
   MemoryTracker: Peak memory usage (for query): 340.03 MiB.
   ```

   ​	接下来尝试优化这条查询语句。

2. 单个字段、全表扫描

   ​	首先，还是全表扫描，但只访问1个字段：

   ```sql
   SELECT WatchID FROM hits_v，
   ```

   ​	执行下面的语句：

   ```shell
   clickhouse-client -h ch7.nauu.com --send_logs_level=trace <<< 'SELECT WatchID
   FROM hits_v1' > /dev/null
   ```

   ​	再次观察执行日志，会发现该查询语句仍然会扫描所有的12个分区，并读取8873910行数据，但结果集大小由之前的8.50 GB降低到了现在的67.70 MB：

   ```shell
   Read 8873910 rows, 67.70 MiB in 0.195 sec., 45505217 rows/sec., 347.18 MiB/sec.
   ```

   ​	内存的峰值消耗也从先前的340 MB降低为现在的17.56 MB：

   ```shell
   MemoryTracker: Peak memory usage (for query): 17.56 MiB.
   ```

3. 使用分区索引

   ​	继续修改SQL语句，增加WHERE子句，并将分区字段EventDate作为查询条件：

   ```sql
   SELECT WatchID FROM hits_v1 WHERE EventDate = '2014-03-17',
   ```

   ​	执行下面的语句：

   ```shell
   clickhouse-client -h ch7.nauu.com --send_logs_level=trace <<< "SELECT WatchID
   FROM hits_v1 WHERE EventDate = '2014-03-17'" > /dev/null
   ```

   ​	这一次会看到执行日志发生了一些变化。首先，**WHERE子句被自动优化成了PREWHERE子句**：

   ```shell
   InterpreterSelectQuery: MergeTreeWhereOptimizer: condition "EventDate = '2014-03-
   17'" moved to PREWHERE
   ```

   ​	其次，分区索引被启动了：

   ```shell
   MinMax index condition: (column 0 in [16146, 16146])
   ```

   ​	借助分区索引，这次查询只需要扫描9个分区目录，剪枝了3个分区：

   ```shell
   Selected 9 parts by date, 9 parts by key, 1095 marks to read from 9 ranges
   ```

   ​	**由于仍然没有启用主键索引，所以该查询仍然需要扫描9个分区内，即所有的1095个MarkRange**。所以，最终需要读取到内存的预估数据量为8892640行：

   ```shell
   Reading approx. 8892640 rows with 2 streams
   ```

4. 使用主键索引

   ​	继续修改SQL语句，在WHERE子句中增加主键字段CounterID的过滤条件：

   ```sql
   SELECT WatchID FROM hits_v1 WHERE EventDate = '2014-03-17'
   AND CounterID = 67141
   ```

   ​	执行下面的语句：

   ```shell
   clickhouse-client -h ch7.nauu.com --send_logs_level=trace <<< "SELECT WatchID
   FROM hits_v1 WHERE EventDate = '2014-03-17' AND CounterID = 67141 " > /dev/null
   ```

   ​	再次观察日志，会发现在上次的基础上主键索引也被启动了：

   ```shell
   Key condition: (column 0 in [67141, 67141])
   ```

   ​	<u>由于启用了主键索引，所以需要扫描的MarkRange由1095个降到了8个</u>：

   ```shell
   Selected 9 parts by date, 8 parts by key, 8 marks to read from 8 ranges
   ```

   ​	现在最终需要读取到内存的预估数据量只有65536行（8192*8）：

   ```shell
   Reading approx. 65536 rows with 2 streams
   ```

---

好了，现在总结一下：

（1）**通过将ClickHouse服务日志设置到DEBUG或者TRACE级别，可以变相实现EXPLAIN查询，以分析SQL的执行日志**。

（2）**需要真正执行了SQL查询，CH才能打印计划日志，所以如果表的数据量很大，最好借助LIMIT子句以减小查询返回的数据量**。

（3）在日志中，分区过滤信息部分如下所示：

```shell
Selected xxx parts by date,
```

​	**<u>其中by date是固定的，无论我们的分区键是什么字段，这里都不会变。这是由于在早期版本中，MergeTree分区键只支持日期字段</u>**。

（4）不要使用SELECT * 全字段查询。

（5）尽可能利用各种索引（分区索引、一级索引、二级索引），这样可避免全表扫描。

## 9.16 本章小结

​	本章按照ClickHouse对SQL大致的解析顺序，依次介绍了各种查询子句的用法。包括用于简化SQL写法的WITH子句、用于数据采样的SAMPLE子句、能够优化查询的PREWHERE子句以及常用的JOIN和GROUP BY子句等。但是到目前为止，我们还是只介绍了ClickHouse的本地查询部分，当面对海量数据的时候，单节点服务是不足以支撑的，所以下一章将进一步介绍与ClickHouse分布式相关的知识。

# * 10. 副本与分片

​	纵使单节点性能再强，也会有遇到瓶颈的那一天。业务量的持续增长、服务器的意外故障，都是ClickHouse需要面对的洪水猛兽。常言道，“一个篱笆三个桩，一个好汉三个帮”，而**集群**、**副本**与**分片**，就是ClickHouse的三个“桩”和三个“帮手”。

## 10.1 概述

​	集群是副本和分片的基础，它将ClickHouse的服务拓扑由单节点延伸到多个节点，但它并不像Hadoop生态的某些系统那样，要求所有节点组成一个单一的大集群。**ClickHouse的集群配置非常灵活，用户既可以将所有节点组成一个单一集群，也可以按照业务的诉求，把节点划分为多个小的集群**。在每个小的集群区域之间，它们的节点、分区和副本数量可以各不相同，如图10-1所示。

![preview](https://segmentfault.com/img/bVbI4JZ/view)

​	图10-1 单集群和多集群的示意图

​	<u>从作用来看，**ClickHouse集群的工作更多是针对逻辑层面的**。集群定义了多个节点的拓扑关系，这些节点在后续服务过程中可能会协同工作，而**执行层面的具体工作则交给了副本和分片来执行**</u>。

​	副本和分片这对双胞胎兄弟，有时候看起来泾渭分明，有时候又让人分辨不清。这里有两种区分的方法。

+ 一种是从**数据层面区分**，假设ClickHouse的N个节点组成了一个集群，在集群的各个节点上，都有一张结构相同的数据表Y。<u>如果N1的Y和N2的Y中的数据完全不同，则N1和N2互为分片；如果它们的数据完全相同，则它们互为副本</u>。换言之，**分片之间的数据是不同的，而副本之间的数据是完全相同的**。所以抛开表引擎的不同，单纯从数据层面来看，副本和分片有时候只有一线之隔。

+ 另一种是从**功能作用层面区分**，**<u>使用副本的主要目的是防止数据丢失，增加数据存储的冗余；而使用分片的主要目的是实现数据的水平切分</u>**，如图10-2所示。

![image.png](https://segmentfault.com/img/bVbI4J3)

​	图10-2 区分副本和分片的示意图

​	本章接下来会按照由易到难的方式介绍副本、分片和集群的使用方法。从数据表的初始形态1分片、0副本开始介绍；接着介绍如何为它添加副本，从而形成1分片、1副本的状态；再介绍如何引入分片，将其转换为多分片、1副本的形态（多副本的形态以此类推），如图10-3所示。

![image.png](https://segmentfault.com/img/bVbI4J4)

​	图10-3 由1分片、0副本发展到多分片、1副本的示意图

​	这种形态的变化过程像极了企业内的业务发展过程。在业务初期，我们从单张数据表开始；在业务上线之后，可能会为它增加副本，以保证数据的安全，或者希望进行读写分离；随着业务量的发展，单张数据表可能会遇到瓶颈，此时会进一步为它增加分片，从而实现数据的水平切分。在接下来的示例中，也会遵循这样的演示路径进行说明。

## 10.2 数据副本

​	不知大家是否还记得，在介绍MergeTree的时候，曾经讲过它的命名规则。如果在\*MergeTree的前面增加Replicated的前缀，则能够组合成一个新的变种引擎，即Replicated-MergeTree复制表，如图10-4所示。

![image.png](https://segmentfault.com/img/bVbI4Kb)

​	图10-4 ReplicatedMergeTree系列表引擎的命名规则示意图

​	换言之，**只有使用了ReplicatedMergeTree复制表系列引擎，才能应用副本的能力（后面会介绍另一种副本的实现方式）**。或者用一种更为直接的方式理解，即**使用ReplicatedMergeTree的数据表就是副本**。

​	ReplicatedMergeTree是MergeTree的派生引擎，它在MergeTree的基础上加入了分布式协同的能力，如图10-5所示。

![image.png](https://segmentfault.com/img/bVbI4Kc)

​	图10-5 ReplicatedMergeTree与MergeTree的逻辑关系示意

​	在MergeTree中，一个数据分区由开始创建到全部完成，会历经两类存储区域。

（1）内存：数据首先会被写入内存缓冲区。

（2）本地磁盘：**<u>数据接着会被写入tmp临时目录分区</u>，待全部完成后再将临时目录重命名为正式分区**。

​	<u>ReplicatedMergeTree在上述基础之上增加了ZooKeeper的部分，它会进一步在ZooKeeper内创建一系列的监听节点，并以此实现多个实例之间的通信。**在整个通信过程中，ZooKeeper并不会涉及表数据的传输**</u>。

### * 10.2.1 副本的特点

​	作为数据副本的主要实现载体，ReplicatedMergeTree在设计上有一些显著特点。

+ 依赖ZooKeeper：<u>在执行INSERT和ALTER查询的时候，ReplicatedMergeTree需要借助ZooKeeper的分布式协同能力，以实现多个副本之间的同步</u>。但是**在查询副本的时候，并不需要使用ZooKeeper**。关于这方面的更多信息，会在稍后详细介绍。

+ **表级别的副本**：副本是在表级别定义的，所以每张表的副本配置都可以按照它的实际需求进行个性化定义，包括副本的数量，以及副本在集群内的分布位置等。

+ **多主架构（Multi Master）：可以在任意一个副本上执行INSERT和ALTER查询，它们的效果是相同的。这些操作会借助ZooKeeper的协同能力被分发至每个副本以本地形式执行**。

+ **Block数据块**：在执行INSERT命令写入数据时，会依据max_insert_block_size的大小（默认1048576行）将数据切分成若干个Block数据块。所以**Block数据块是数据写入的基本单元，并且<u>具有写入的原子性和唯一性</u>**。

+ **原子性：在数据写入时，一个Block块内的数据要么全部写入成功，要么全部失败**。

+ 唯一性：在写入一个Block数据块的时候，会按照当前Block数据块的数据顺序、数据行和数据大小等指标，计算Hash信息摘要并记录在案。在此之后，**如果某个待写入的Block数据块与先前已被写入的Block数据块拥有相同的Hash摘要（Block数据块内数据顺序、数据大小和数据行均相同），则该Block数据块会被忽略**。这项设计可以**预防由异常原因引起的Block数据块重复写入的问题**。

​	如果只是单纯地看这些特点的说明，可能不够直观。没关系，接下来会逐步展开，并附带一系列具体的示例。

### 10.2.2 ZooKeeper的配置方式

​	在正式开始之前，还需要做一些准备工作，那就是安装并配置ZooKeeper，因为ReplicatedMergeTree必须对接到它才能工作。关于ZooKeeper的安装，此处不再赘述，使用3.4.5及以上版本均可。这里着重讲解如何在ClickHouse中增加ZooKeeper的配置。

​	ClickHouse使用一组zookeeper标签定义相关配置，默认情况下，在全局配置config.xml中定义即可。但是各个副本所使用的Zookeeper配置通常是相同的，为了便于在多个节点之间复制配置文件，更常见的做法是将这一部分配置抽离出来，独立使用一个文件保存。

​	首先，在服务器的/etc/clickhouse-server/config.d目录下创建一个名为metrika.xml的配置文件：

```xml
<?xml version="1.0"?>
<yandex>
  <zookeeper-servers> <!—ZooKeeper配置，名称自定义 -->
    <node index="1"> <!—节点配置，可以配置多个地址-->
      <host>hdp1.nauu.com</host>
      <port>2181</port>
    </node>
  </zookeeper-servers>
</yandex>
```

​	接着，在全局配置config.xml中使用<include_from>标签导入刚才定义的配置：

```xml
<include_from>/etc/clickhouse-server/config.d/metrika.xml</include_from>
```

​	并引用ZooKeeper配置的定义：

```xml
<zookeeper incl="zookeeper-servers" optional="false" />
```

​	其中，incl与metrika.xml配置文件内的节点名称要彼此对应。至此，整个配置过程就完成了。

​	**ClickHouse在它的系统表中，颇为贴心地提供了一张名为zookeeper的代理表。通过这张表，可以使用SQL查询的方式读取远端ZooKeeper内的数据**。有一点需要注意，在用于查询的SQL语句中，必须指定path条件，例如查询根路径：

```sql
SELECT * FROM system.zookeeper where path = '/'
┌─name─────────┬─value─┬─czxid─┐
│ dolphinscheduler │ │ 2627 │
│ clickhouse │ │ 92875 │
└─────────────┴─────┴─────┘
```

​	进一步查询clickhouse目录：

```sql
SELECT name, value, czxid, mzxid FROM system.zookeeper where path = '/clickhouse'
┌─name─────┬─value─┬─czxid─┬──mzxid─┐
│ tables │ │ 134107 │ 134107 │
│ task_queue │ │ 92876 │ 92876 │
└────────┴─────┴─────┴──────┘
```

### * 10.2.3 副本的定义形式

​	正如前文所言，使用副本的好处甚多。首先，由于增加了数据的冗余存储，所以降低了数据丢失的风险；其次，由于副本采用了多主架构，所以每个副本实例都可以作为数据读、写的入口，这无疑分摊了节点的负载。

​	**在使用副本时，不需要依赖任何集群配置（关于集群配置，在后续小节会详细介绍），ReplicatedMergeTree结合ZooKeeper就能完成全部工作**。

​	ReplicatedMergeTree的定义方式如下：

```sql
ENGINE = ReplicatedMergeTree('zk_path', 'replica_name')
```

​	在上述配置项中，有zk_path和replica_name两项配置，首先介绍zk_path的作用。

​	**zk_path用于指定在ZooKeeper中创建的数据表的路径**，路径名称是自定义的，并没有固定规则，用户可以设置成自己希望的任何路径。即便如此，ClickHouse还是提供了一些约定俗成的配置模板以供参考，例如：

```shell
/clickhouse/tables/{shard}/table_name
```

​	其中：

+ /clickhouse/tables/是约定俗成的路径固定前缀，表示存放数据表的根路径。

+ {shard}表示分片编号，通常用数值替代，例如01、02、03。一张数据表可以有多个分片，而每个分片都拥有自己的副本。
+ table_name表示数据表的名称，为了方便维护，通常与物理表的名字相同（虽然ClickHouse并不强制要求路径中的表名称和物理表名相同）；

​	而**replica_name的作用是定义在ZooKeeper中创建的副本名称，该名称是区分不同副本实例的唯一标识**。<u>一种约定成俗的命名方式是使用所在服务器的域名称</u>。

​	<u>**对于zk_path而言，同一张数据表的同一个分片的不同副本，应该定义相同的路径**；**而对于replica_name而言，同一张数据表的同一个分片的不同副本，应该定义不同的名称**</u>。

​	是不是有些绕口呢？下面列举几个示例。

​	1个分片、1个副本的情形：

```shell
//1分片，1副本. zk_path相同，replica_name不同
ReplicatedMergeTree('/clickhouse/tables/01/test_1, 'ch5.nauu.com')
ReplicatedMergeTree('/clickhouse/tables/01/test_1, 'ch6.nauu.com')
```

​	多个分片、1个副本的情形：

```shell
//分片1
//2分片，1副本. zk_path相同，其中{shard}=01, replica_name不同
ReplicatedMergeTree('/clickhouse/tables/01/test_1, 'ch5.nauu.com')
ReplicatedMergeTree('/clickhouse/tables/01/test_1, 'ch6.nauu.com')
//分片2
//2分片，1副本. zk_path相同，其中{shard}=02, replica_name不同
ReplicatedMergeTree('/clickhouse/tables/02/test_1, 'ch7.nauu.com')
ReplicatedMergeTree('/clickhouse/tables/02/test_1, 'ch8.nauu.com')
```

## 10.3 ReplicatedMergeTree原理解析

​	<u>ReplicatedMergeTree作为复制表系列的基础表引擎，涵盖了数据副本最为核心的逻辑，将它拿来作为副本的研究标本是最合适不过了</u>。因为只要剖析了ReplicatedMergeTree的核心原理，就能掌握整个ReplicatedMergeTree系列表引擎的使用方法。

### 10.3.1 数据结构

​	在ReplicatedMergeTree的核心逻辑中，大量运用了ZooKeeper的能力，以实现多个ReplicatedMergeTree副本实例之间的协同，包括**主副本选举**、**副本状态感知**、**操作日志分发**、**任务队列**和**BlockID去重判断**等。<u>在执行INSERT数据写入、MERGE分区和MUTATION操作的时候，都会涉及与ZooKeeper的通信</u>。但是<u>**在通信的过程中，并不会涉及任何表数据的传输，在查询数据的时候也不会访问ZooKeeper，所以不必过于担心ZooKeeper的承载压力**</u>。

​	因为ZooKeeper对ReplicatedMergeTree非常重要，所以下面首先从它的数据结构开始介绍。

1. ZooKeeper内的节点结构

   ​	**ReplicatedMergeTree需要依靠ZooKeeper的事件监听机制以实现各个副本之间的协同**。所以，在每张ReplicatedMergeTree表的创建过程中，它会以zk_path为根路径，在Zoo-Keeper中为这张表创建一组监听节点。按照作用的不同，监听节点可以大致分成如下几类：

   （1）元数据：

   + /metadata：保存元数据信息，包括主键、分区键、采样表达式等。

   + /columns：保存列字段信息，包括列名称和数据类型。

   + /replicas：保存副本名称，对应设置参数中的replica_name。

   （2）判断标识：

   + /leader_election：用于主副本的选举工作，主副本会主导MERGE和MUTATION操作（ALTER DELETE和ALTER UPDATE）。**这些任务在主副本完成之后再借助ZooKeeper将消息事件分发至其他副本**。

   + /blocks：记录Block数据块的Hash信息摘要，以及对应的partition_id。<u>通过Hash摘要能够判断Block数据块是否重复；通过partition_id，则能够找到需要同步的数据分区</u>。

   + /block_numbers：按照分区的写入顺序，以相同的顺序记录partition_id。**各个副本在本地进行MERGE时，都会依照相同的block_numbers顺序进行**。

   + /quorum：**记录quorum的数量，当至少有quorum数量的副本写入成功后，整个写操作才算成功**。quorum的数量由insert_quorum参数控制，默认值为0。

   （3）操作日志：

   + /log：常规操作日志节点（INSERT、MERGE和DROP PARTITION），它是整个工作机制中最为重要的一环，保存了副本需要执行的任务指令。<u>log使用了ZooKeeper的持久顺序型节点</u>，每条指令的名称以log-为前缀递增，例如log-0000000000、log-0000000001等。**每一个副本实例都会监听/log节点，当有新的指令加入时，它们会把指令加入副本各自的任务队列，并执行任务**。关于这方面的执行逻辑，稍后会进一步展开。

   + /mutations：MUTATION操作日志节点，作用与log日志类似，当执行ALERT DELETE和ALERT UPDATE查询时，操作指令会被添加到这个节点。<u>mutations同样使用了ZooKeeper的持久顺序型节点</u>，但是它的命名没有前缀，每条指令直接以递增数字的形式保存，例如0000000000、0000000001等。关于这方面的执行逻辑，同样稍后展开。
   + /replicas/{replica_name}/*：<u>每个副本各自的节点下的一组监听节点，用于指导副本在本地执行具体的任务指令</u>，其中较为重要的节点有如下几个：
   + /queue：**任务队列节点，用于执行具体的操作任务。当副本从/log或/mutations节点监听到操作指令时，会将执行任务添加至该节点下，并基于队列执行**。
   + /log_pointer：log日志指针节点，记录了最后一次执行的log日志下标信息，例如log_pointer：4对应了/log/log-0000000003（从0开始计数）。
   + /mutation_pointer：mutations日志指针节点，记录了最后一次执行的mutations日志名称，例如mutation_pointer：0000000000对应了/mutations/000000000。

2. Entry日志对象的数据结构

   ​	从上一小节的介绍中能够得知，**ReplicatedMergeTree在ZooKeeper中有两组非常重要的父节点，那就是/log和/mutations。它们的作用犹如一座通信塔，是分发操作指令的信息通道，而发送指令的方式，则是为这些父节点添加子节点**。<u>所有的副本实例，都会监听父节点的变化，当有子节点被添加时，它们能实时感知</u>。

   ​	这些被添加的子节点在ClickHouse中被统一抽象为Entry对象，而具体实现则由Log-Entry和MutationEntry对象承载，分别对应/log和/mutations节点。

   1）LogEntry

   ​	LogEntry用于封装/log的子节点信息，它拥有如下几个核心属性：

   + source replica：发送这条Log指令的副本来源，对应replica_name。

   + type：操作指令类型，主要有get、merge和mutate三种，分别对应从远程副本下载分区、合并分区和MUTATION操作。

   + block_id：当前分区的BlockID，对应/blocks路径下子节点的名称。

   + partition_name：当前分区目录的名称。

   2）MutationEntry

   ​	MutationEntry用于封装/mutations的子节点信息，它同样拥有如下几个核心属性：

   + source replica：发送这条MUTATION指令的副本来源，对应replica_name。

   + commands：操作指令，主要有ALTER DELETE和ALTER UPDATE。

   + mutation_id：MUTATION操作的版本号。

   + partition_id：当前分区目录的ID。

   以上就是Entry日志对象的数据结构信息，在接下来将要介绍的核心流程中，将会看到它们的身影。

### * 10.3.2 副本协同的核心流程

​	**副本协同的核心流程主要有INSERT、MERGE、MUTATION和ALTER四种，分别对应了数据写入、分区合并、数据修改和元数据修改**。**INSERT和ALTER查询是分布式执行的**。借助ZooKeeper的事件通知机制，多个副本之间会自动进行有效协同，但是<u>它们不会使用ZooKeeper存储任何分区数据</u>。而**其他查询并不支持分布式执行，包括SELECT、CREATE、DROP、RENAME和ATTACH**。例如，为了创建多个副本，我们需要分别登录每个ClickHouse节点，在它们本地执行各自的CREATE语句（后面将会介绍如何利用集群配置简化这一操作）。接下来，会依次介绍上述流程的工作机理。为了便于理解，我先来整体认识一下各个流程的介绍方法。

​	首先，拟定一个演示场景，即使用ReplicatedMergeTree实现一张拥有1分片、1副本的数据表，并以此来贯穿整个讲解过程（对于大于1个副本的场景，流程以此类推）。

​	接着，通过对ReplicatedMergeTree分别执行INSERT、MERGE、MUTATION和ALTER操作，以此来讲解相应的工作原理。与此同时，通过实际案例，论证工作原理。

1. INSERT的核心执行流程

   ​	当需要在ReplicatedMergeTree中执行INSERT查询以写入数据时，即会进入INSERT核心流程，其整体示意如图10-6所示。

   + 

   ![preview](https://img-blog.csdnimg.cn/img_convert/cd601542ad22b4adc8b303b917b08e08.png)

   ![preview](https://img-blog.csdnimg.cn/img_convert/790d422b8d4c2d7a3708f65563b6985f.png)

   ​	图10-6 ReplicatedMergeTree与其他副本协同的核心流程

   ​	整个流程从上至下按照时间顺序进行，其大致可分成8个步骤。现在，根据图10-6所示编号讲解整个过程。

   1）创建第一个副本实例

   ​	假设首先从CH5节点开始，对CH5节点执行下面的语句后，会创建第一个副本实例：

   ```sql
   CREATE TABLE replicated_sales_1(
     id String,
     price Float64,
     create_time DateTime
   ) ENGINE =
   ReplicatedMergeTree('/clickhouse/tables/01/replicated_sales_1','ch5.nauu.com')
   PARTITION BY toYYYYMM(create_time)
   ORDER BY id
   ```

   ​	在创建的过程中，ReplicatedMergeTree会进行一些初始化操作，例如：

   + 根据zk_path初始化所有的ZooKeeper节点。

   + 在/replicas/节点下注册自己的副本实例ch5.nauu.com。

   + 启动监听任务，监听/log日志节点。

   + **参与副本选举，选举出主副本，选举的方式是向/leader_election/插入子节点，第一个插入成功的副本就是主副本**。

   2）创建第二个副本实例

   ​	接着，在CH6节点执行下面的语句，创建第二个副本实例。表结构和zk_path需要与第一个副本相同，而replica_name则需要设置成CH6的域名：

   ```sql
   CREATE TABLE replicated_sales_1(
     //相同结构
   ) ENGINE =
   ReplicatedMergeTree('/clickhouse/tables/01/replicated_sales_1','ch6.nauu.com')
   //相同结构
   ```

   ​	在创建过程中，第二个ReplicatedMergeTree同样会进行一些初始化操作，例如：

   + 在/replicas/节点下注册自己的副本实例ch6.nauu.com。

   + 启动监听任务，监听/log日志节点。

   + 参与副本选举，选举出主副本。在这个例子中，CH5副本成为主副本。

   3）向第一个副本实例写入数据

   ​	现在尝试向第一个副本CH5写入数据。执行如下命令：

   ```sql
   INSERT INTO TABLE replicated_sales_1 VALUES('A001',100,'2019-05-10 00:00:00')
   ```

   ​	上述命令执行之后，首先会在本地完成分区目录的写入：

   ```shell
   Renaming temporary part tmp_insert_201905_1_1_0 to 201905_0_0_0
   ```

   ​	接着向/blocks节点写入该数据分区的block_id：

   ```shell
   Wrote block with ID '201905_2955817577822961065_12656761735954722499'
   ```

   ​	**该block_id将作为后续去重操作的判断依据**。如果此时再次执行刚才的INSERT语句，试图写入重复数据，则会出现如下提示：

   ```shell
   Block with ID 201905_2955817577822961065_12656761735954722499 already exists;
   ignoring it.
   ```

   ​	即副本会自动忽略block_id重复的待写入数据。

   ​	此外，**如果设置了insert_quorum参数（默认为0），并且insert_quorum>=2，则CH5会进一步监控已完成写入操作的副本个数，只有当写入副本个数大于或等于insert_quorum时，整个写入操作才算成功**。

   4）由第一个副本实例推送Log日志

   ​	在3步骤完成之后，会继续由执行了INSERT的副本向/log节点推送操作日志。在这个例子中，会由第一个副本CH5担此重任。日志的编号是/log/log-0000000000，而LogEntry的核心属性如下：

   ```shell
   /log/log-0000000000
   source replica: ch5.nauu.com
   block_id: 201905_...
   type : get
   partition_name :201905_0_0_0
   ```

   ​	从日志内容中可以看出，<u>操作类型为get下载，而需要下载的分区是201905_0_0_0。其余所有副本都会基于Log日志以相同的顺序执行命令</u>。

   5）第二个副本实例拉取Log日志

   ​	CH6副本会一直监听/log节点变化，当CH5推送了/log/log-0000000000之后，CH6便会触发日志的拉取任务并更新log_pointer，将其指向最新日志下标：

   ```shell
   /replicas/ch6.nauu.com/log_pointer : 0
   ```

   ​	**在拉取了LogEntry之后，它并不会直接执行，而是将其转为任务对象放至队列**：

   ```shell
   /replicas/ch6.nauu.com/queue/
   Pulling 1 entries to queue: log-0000000000 - log-0000000000
   ```

   ​	<u>这是因为在复杂的情况下，考虑到在同一时段内，会连续收到许多个LogEntry，所以使用队列的形式消化任务是一种更为合理的设计</u>。**注意，拉取的LogEntry是一个区间，这同样也是因为可能会连续收到多个LogEntry**。

   6）第二个副本实例向其他副本发起下载请求

   ​	**CH6基于/queue队列开始执行任务。当看到type类型为get的时候，ReplicatedMerge-Tree会明白此时在远端的其他副本中已经成功写入了数据分区，而自己需要同步这些数据**。

   ​	CH6上的第二个副本实例会开始选择一个远端的其他副本作为数据的下载来源。远端副本的选择算法大致是这样的：

   （1）从/replicas节点拿到所有的副本节点。

   （2）**遍历这些副本，选取其中一个。选取的副本需要拥有最大的log_pointer下标，并且/queue子节点数量最少。log_pointer下标最大，意味着该副本执行的日志最多，数据应该更加完整；而/queue最小，则意味着该副本目前的任务执行负担较小**。

   ​	在这个例子中，算法选择的远端副本是CH5。于是，CH6副本向CH5发起了HTTP请求，希望下载分区201905_0_0_0：

   ```sql
   Fetching part 201905_0_0_0 from replicas/ch5.nauu.com
   Sending request to http://ch5.nauu.com:9009/?endpoint=DataPartsExchange
   ```

   ​	如果第一次下载请求失败，在默认情况下，CH6再尝试请求4次，一共会尝试5次（由max_fetch_partition_retries_count参数控制，默认为5）。

   7）第一个副本实例响应数据下载

   ​	CH5的DataPartsExchange端口服务接收到调用请求，在得知对方来意之后，根据参数做出响应，将本地分区201905_0_0_0基于DataPartsExchang的服务响应发送回CH6：

   ```shell
   Sending part 201905_0_0_0
   ```

   8）第二个副本实例下载数据并完成本地写入

   ​	CH6副本在收到CH5的分区数据后，首先将其写至临时目录：

   ```shell
   tmp_fetch_201905_0_0_0
   ```

   ​	待全部数据接收完成之后，重命名该目录：

   ```shell
   Renaming temporary part tmp_fetch_201905_0_0_0 to 201905_0_0_0
   ```

   ​	至此，整个写入流程结束。

   ​	可以看到，<u>在INSERT的写入过程中，ZooKeeper不会进行任何实质性的数据传输</u>。<u>本着**谁执行谁负责**的原则，在这个案例中由CH5首先在本地写入了分区数据。之后，也由这个副本负责发送Log日志，通知其他副本下载数据。**如果设置了insert_quorum并且insert_quorum>=2，则还会由该副本监控完成写入的副本数量**。其他副本在接收到Log日志之后，会选择一个最合适的远端副本，点对点地下载分区数据</u>。

2. MERGE的核心执行流程

   ​	当ReplicatedMergeTree触发分区合并动作时，即会进入这个部分的流程，它的核心流程如图10-7所示。

   ​	**无论MERGE操作从哪个副本发起，其合并计划都会交由主副本来制定**。在INSERT的例子中，CH5节点已经成功竞选为主副本，所以为了方便论证，这个案例就从CH6节点开始。整个流程从上至下按照时间顺序进行，其大致分成5个步骤。现在，根据图10-7中所示编号讲解整个过程。

   ![image.png](https://segmentfault.com/img/bVbI4MK)

   ![image.png](https://segmentfault.com/img/bVbI4Nu)

   ​	图10-7 ReplicatedMergeTree与其他副本协同的核心流程

   1）创建远程连接，尝试与**主副本**通信

   ​	首先在CH6节点执行OPTIMIZE，强制触发MERGE合并。这个时候，CH6通过/replicas找到主副本CH5，并尝试建立与它的远程连接。

   ```shell
   optimize table replicated_sales_1
   Connection (ch5.nauu.com:9000): Connecting. Database: default. User: default
   ```

   2）主副本接收通信

   ​	主副本CH5接收并建立来自远端副本CH6的连接。

   ```shell
   Connected ClickHouse Follower replica version 19.17.0, revision: 54428, database:
   default, user: default.
   ```

   3）**由主副本制定MERGE计划并推送Log日志**

   ​	由主副本CH5制定MERGE计划，并判断哪些分区需要被合并。在选定之后，CH5将合并计划转换为Log日志对象并推送Log日志，以通知所有副本开始合并。日志的核心信息如下：

   ```shell
   /log/log-0000000002
   source replica: ch5.nauu.com
   block_id:
   type : merge
   201905_0_0_0
   201905_1_1_0
   into
   201905_0_1_1
   ```

   ​	从日志内容中可以看出，操作类型为Merge合并，而这次需要合并的分区目录是201905_0_0_0和201905_1_1_0。

   ​	**与此同时，<u>主副本还会锁住执行线程</u>，对日志的接收情况进行监听**：

   ```shell
   Waiting for queue-0000000002 to disappear from ch5.nauu.com queue
   ```

   ​	**其监听行为由replication_alter_partitions_sync参数控制，默认值为1。当此参数为0时，不做任何等待；为1时，只等待主副本自身完成；为2时，会等待所有副本拉取完成**。

   4）各个副本分别拉取Log日志

   ​	CH5和CH6两个副本实例将分别监听/log/log-0000000002日志的推送，它们也会分别拉取日志到本地，并推送到各自的/queue任务队列：

   ```shell
   Pulling 1 entries to queue: log-0000000002 - log-0000000002
   ```

   5）各个副本分别在本地执行MERGE

   ​	CH5和CH6基于各自的/queue队列开始执行任务：

   ```shell
   Executing log entry to merge parts 201905_0_0_0, 201905_1_1_0 to 201905_0_1_1
   ```

   ​	各个副本开始在本地执行MERGE：

   ```shell
   Merged 2 parts: from 201905_0_0_0 to 201905_1_1_0
   ```

   ​	至此，整个合并流程结束。

   ​	可以看到，在MERGE的合并过程中，ZooKeeper也不会进行任何实质性的数据传输，所有的合并操作，最终都是由各个副本在本地完成的。**而<u>无论合并动作在哪个副本被触发，都会首先被转交至主副本，再由主副本负责合并计划的制定、消息日志的推送以及对日志接收情况的监控</u>**。

3. MUTATION的核心执行流程

   ​	当对ReplicatedMergeTree执行ALTER DELETE或者ALTERUPDATE操作的时候，即会进入MUTATION部分的逻辑，它的核心流程如图10-8所示。

   ​	与MERGE类似，无论MUTATION操作从哪个副本发起，首先都会由主副本进行响应。所以为了方便论证，这个案例还是继续从CH6节点开始（因为CH6不是主副本）。整个流程从上至下按照时间顺序进行，其大致分成5个步骤。现在根据图10-8中所示编号讲解整个过程。

   ![image.png](https://segmentfault.com/img/bVbI4NO)

   ![image.png](https://segmentfault.com/img/bVbI4NR)

   ​	图10-8 ReplicatedMergeTree与其他副本协同的核心流程

   1）推送MUTATION日志

   ​	在CH6节点尝试通过DELETE来删除数据（执行UPDATE的效果与此相同），执行如下命令：
   ```sql
   ALTER TABLE replicated_sales_1 DELETE WHERE id = '1'
   ```

   ​	执行之后，该副本会接着进行两个重要事项：

   + 创建MUTATION ID：

     ```sql
     Created mutation with ID 0000000000
     ```

   + 将MUTATION操作转换为MutationEntry日志，并推送到/mutations/0000000000。MutationEntry的核心属性如下：

     ```shell
     /mutations/0000000000
     source replica: ch6.nauu.com
     mutation_id: 2
     partition_id: 201905
     commands: DELETE WHERE id = \'1\'
     ```

     由此也能知晓，MUTATION的操作日志是经由/mutations节点分发至各个副本的。

   2）所有副本实例各自监听MUTATION日志

   ​	**CH5和CH6都会监听/mutations节点，所以一旦有新的日志子节点加入，它们都能实时感知**：

   ```shell
   Loading 1 mutation entries: 0000000000 – 0000000000
   ```

   ​	**<u>当监听到有新的MUTATION日志加入时，并不是所有副本都会直接做出响应，它们首先会判断自己是否为主副本</u>**。

   3）**<u>由主副本实例响应MUTATION日志并推送Log日志</u>**

   ​	**<u>只有主副本才会响应MUTATION日志</u>**，在这个例子中主副本为CH5，所以CH5将MUTATION日志转换为LogEntry日志并推送至/log节点，以通知各个副本执行具体的操作。日志的核心信息如下：

   ```shell
   /log/log-0000000003
   source replica: ch5.nauu.com
   block_id:
   type : mutate
   201905_0_1_1
   to
   201905_0_1_1_2
   ```

   ​	从日志内容中可以看出，上述操作的类型为mutate，而这次需要将201905_0_1_1分区修改为201905_0_1_1_2(201905_0_1_1+"_" + mutation_id)。

   4）各个副本实例分别拉取Log日志

   ​	CH5和CH6两个副本分别监听/log/log-0000000003日志的推送，它们也会分别拉取日志到本地，并推送到各自的/queue任务队列：

   ```shell
   Pulling 1 entries to queue: log-0000000003 - log-0000000003
   ```

   5）各个副本实例分别在本地执行MUTATION

   ​	CH5和CH6基于各自的/queue队列开始执行任务：

   ```shell
   Executing log entry to mutate part 201905_0_1_1 to 201905_0_1_1_2
   ```

   ​	各个副本，开始在本地执行MUTATION：

   ```shell
   Cloning part 201905_0_1_1 to tmp_clone_201905_0_1_1_2
   Renaming temporary part tmp_clone_201905_0_1_1_2 to 201905_0_1_1_2.
   ```

   ​	至此，整个MUTATION流程结束。

   ​	可以看到，在MUTATION的整个执行过程中，ZooKeeper同样不会进行任何实质性的数据传输。所有的MUTATION操作，最终都是由各个副本在本地完成的。而MUTATION操作是经过/mutations节点实现分发的。本着**谁执行谁负责**的原则，在这个案例中由CH6负责了消息的推送。**<u>但是无论MUTATION动作从哪个副本被触发，之后都会被转交至主副本，再由主副本负责推送Log日志，以通知各个副本执行最终的MUTATION逻辑。同时也由主副本对日志接收的情况实行监控</u>**。

4. ALTER的核心执行流程

   ​	当对ReplicatedMergeTree执行ALTER操作进行元数据修改的时候，即会进入ALTER部分的逻辑，例如增加、删除表字段等。而ALTER的核心流程如图10-9所示。

   ​	与之前的几个流程相比，ALTET的流程会简单很多，其执行过程中并不会涉及/log日志的分发。整个流程从上至下按照时间顺序进行，其大致分成3个步骤。现在根据图10-9所示编号讲解整个过程。

   ![image.png](https://segmentfault.com/img/bVbI4Oa)

   ​	图10-9 ReplicatedMergeTree与其他副本协同的核心流程

   1）修改共享元数据

   ​	在CH6节点尝试增加一个列字段，执行如下语句：

   ```sql
   ALTER TABLE replicated_sales_1 ADD COLUMN id2 String
   ```

   ​	执行之后，CH6会修改ZooKeeper内的共享元数据节点：

   ```shell
   /metadata，/columns
   Updated shared metadata nodes in ZooKeeper. Waiting for replicas to apply changes.
   ```

   ​	数据修改后，节点的版本号也会同时提升：

   ```shell
   Version of metadata nodes in ZooKeeper changed. Waiting for structure write lock.
   ```

   ​	<u>与此同时，CH6还会负责监听所有副本的修改完成情况</u>：

   ```shell
   Waiting for ch5.nauu.com to apply changes
   Waiting for ch6.nauu.com to apply changes
   ```

   2）监听共享元数据变更并各自执行本地修改

   ​	CH5和CH6两个副本分别监听共享元数据的变更。之后，它们会分别对本地的元数据版本号与共享版本号进行对比。在这个案例中，它们会**发现本地版本号低于共享版本号，于是它们开始在各自的本地执行更新操作**：

   ```shell
   Metadata changed in ZooKeeper. Applying changes locally.
   Applied changes to the metadata of the table.
   ```

   3）确认所有副本完成修改

   ​	CH6确认所有副本均已完成修改：

   ```sql
   ALTER finished
   Done processing query
   ```

   ​	至此，整个ALTER流程结束。

   ​	可以看到，在ALTER整个的执行过程中，ZooKeeper不会进行任何实质性的数据传输。所有的ALTER操作，最终都是由各个副本在本地完成的。本着**谁执行谁负责**的原则，在这个案例中由CH6负责对共享元数据的修改以及对各个副本修改进度的监控。

## * 10.4 数据分片

​	<u>通过引入数据副本，虽然能够有效降低数据的丢失风险（多份存储），并提升查询的性能（分摊查询、读写分离），但是仍然有一个问题没有解决，那就是数据表的容量问题</u>。到目前为止，每个副本自身，仍然保存了数据表的全量数据。所以在业务量十分庞大的场景中，依靠副本并不能解决单表的性能瓶颈。想要从根本上解决这类问题，需要借助另外一种手段，即进一步将数据水平切分，也就是我们将要介绍的数据分片。

​	**ClickHouse中的每个服务节点都可称为一个shard（分片）**。<u>从理论上来讲，假设有N(N>=1)张数据表A，分布在N个ClickHouse服务节点，而这些数据表彼此之间没有重复数据，那么就可以说数据表A拥有N个分片</u>。然而在工程实践中，如果只有这些分片表，那么整个Sharding（分片）方案基本是不可用的。**<u>对于一个完整的方案来说，还需要考虑数据在写入时，如何被均匀地写至各个shard，以及数据在查询时，如何路由到每个shard，并组合成结果集</u>**。所以，**ClickHouse的数据分片需要结合Distributed表引擎一同使用**，如图10-10所示。

​	![image.png](https://segmentfault.com/img/bVbI4Oo)

​	图10-10 Distributed分布式表引擎与分片的关系示意图

​	**<u>Distributed表引擎自身不存储任何数据，它能够作为分布式表的一层透明代理，在集群内部自动开展数据的写入、分发、查询、路由等工作</u>**。

### 10.4.1 集群的配置方式

​	**在ClickHouse中，集群配置用shard代表分片、用replica代表副本**。那么在逻辑层面，表示1分片、0副本语义的配置如下所示：

```xml
<shard> <!-- 分片 -->
  <replica> <!-- 副本 -->
  </replica>
</shard>
```

​	而表示1分片、1副本语义的配置则是：

```xml
<shard> <!-- 分片 -->
  <replica> <!-- 副本 -->
  </replica>
  <replica>
  </replica>
</shard>
```

​	可以看到，这样的配置似乎有些反直觉，shard更像是逻辑层面的分组，而**<u>无论是副本还是分片，它们的载体都是replica，所以从某种角度来看，副本也是分片</u>**。关于这方面的详细介绍会在后续展开，现在先回到之前的话题。

​	**由于<u>Distributed表引擎需要读取集群的信息</u>，所以首先必须为ClickHouse添加集群的配置**。找到前面在介绍ZooKeeper配置时增加的metrika.xml配置文件，将其加入集群的配置信息。

​	集群有两种配置形式，下面分别介绍。

1. 不包含副本的分片

   **如果直接使用node标签定义分片节点，那么该集群将只包含分片，不包含副本**。以下面的配置为例：

   ```xml
   <yandex>
     <!--自定义配置名，与config.xml配置的incl属性对应即可 -->
     <clickhouse_remote_servers>
       <shard_2><!--自定义集群名称-->
         <node><!--定义ClickHouse节点-->
           <host>ch5.nauu.com</host>
           <port>9000</port>
           <!--选填参数
             <weight>1</weight>
             <user></user>
             <password></password>
             <secure></secure>
             <compression></compression>
            -->
         </node>
         <node>
           <host>ch6.nauu.com</host>
           <port>9000</port>
         </node>
       </shard_2>
       ……
     </clickhouse_remote_servers>
   ```

   该配置定义了一个名为shard_2的集群，其包含了2个分片节点，它们分别指向了是CH5和CH6服务器。现在分别对配置项进行说明：

   + shard_2表示自定义的集群名称，**全局唯一**，是<u>后续引用集群配置的唯一标识</u>。<u>在一个配置文件内，可以定义任意组集群</u>。

   + <u>node用于定义分片节点，**不包含副本**</u>。

   + host指定部署了ClickHouse节点的服务器地址。

   + port指定ClickHouse服务的TCP端口。

   接下来介绍选填参数：

   + weight分片权重默认为1，在后续小节中会对其详细介绍。

   + user为ClickHouse用户，默认为default。

   + password为ClickHouse的用户密码，默认为空字符串。
   + secure为SSL连接的端口，默认为9440。
   + compression表示是否开启数据压缩功能，默认为true。

2. 自定义分片与副本

   <u>集群配置支持自定义分片和副本的数量，这种形式需要使用shard标签代替先前的node，除此之外的配置完全相同</u>。在这种自定义配置的方式下，分片和副本的数量完全交由配置者掌控。其中，**shard表示逻辑上的数据分片，而物理上的分片则用replica表示**。<u>如果在1个shard标签下定义N(N>=1)组replica，则该shard的语义表示1个分片和N-1个副本</u>。接下来用几组配置示例进行说明。

   1）不包含副本的分片

   下面所示的这组集群配置的效果与先前介绍的shard_2集群相同：

   ```xml
   <!-- 2个分片、0个副本 -->
   <sharding_simple> <!-- 自定义集群名称 -->
     <shard> <!-- 分片 -->
       <replica> <!-- 副本 -->
         <host>ch5.nauu.com</host>
         <port>9000</port>
       </replica>
     </shard>
     <shard>
       <replica>
         <host>ch6.nauu.com</host>
         <port>9000</port>
       </replica>
     </shard>
   </sharding_simple>
   ```

   sharding_simple集群的语义为2分片、0副本（1分片、0副本，再加上1分片、0副本）。

   2）N个分片和N个副本

   这种形式可以按照实际需求自由组合，例如下面的这组配置，集群sharding_simple_1拥有1个分片和1个副本：

   ```xml
   <!-- 1个分片 1个副本-->
   <sharding_simple_1>
     <shard>
       <replica>
         <host>ch5.nauu.com</host>
         <port>9000</port>
       </replica>
       <replica>
         <host>ch6.nauu.com</host>
         <port>9000</port>
       </replica>
     </shard>
   </sharding_simple_1>
   ```

   下面所示集群sharding_ha拥有2个分片，而每个分片拥有1个副本：

   ```xml
   <sharding_ha>
     <shard>
       <replica>
         <host>ch5.nauu.com</host>
         <port>9000</port>
       </replica>
       <replica>
         <host>ch6.nauu.com</host>
         <port>9000</port>
       </replica>
     </shard>
     <shard>
       <replica>
         <host>ch7.nauu.com</host>
         <port>9000</port>
       </replica>
       <replica>
         <host>ch8.nauu.com</host>
         <port>9000</port>
       </replica>
     </shard>
   </sharding_ha>
   ```

   从上面的配置信息中能够得出结论，<u>集群中replica数量的上限是由ClickHouse节点的数量决定的</u>，例如为了部署集群sharding_ha，需要4个ClickHouse服务节点作为支撑。

   在完成上述配置之后，可以查询系统表验证集群配置是否已被加载：

   ```sql
   SELECT cluster, host_name FROM system.clusters
   ┌─cluster────────┬─host_name──┐
   │ shard_2 │ ch5.nauu.com │
   │ shard_2 │ ch6.nauu.com │
   │ sharding_simple │ ch5.nauu.com │
   │ sharding_simple │ ch6.nauu.com │
   │ sharding_simple_1 │ ch5.nauu.com │
   │ sharding_simple_1 │ ch6.nauu.com │
   └─────────────┴─────────┘
   ```

### * 10.4.2 基于集群实现分布式DDL

​	不知道大家是否还记得，在前面介绍数据副本时为了创建多张副本表，我们需要分别登录到每个ClickHouse节点，在它们本地执行各自的CREATE语句。这是因为<u>在默认的情况下，CREATE、DROP、RENAME和ALTER等DDL语句并不支持分布式执行</u>。而在**加入集群配置后，就可以使用新的语法实现分布式DDL执行**了，其语法形式如下：

```sql
CREATE/DROP/RENAME/ALTER TABLE ON CLUSTER cluster_name
```

​	其中，cluster_name对应了配置文件中的集群名称，**ClickHouse会根据集群的配置信息顺藤摸瓜，分别去各个节点执行DDL语句**。

​	下面是在10.2.3节中使用的多副本示例：

```shell
//1分片，2副本. zk_path相同，replica_name不同。
ReplicatedMergeTree('/clickhouse/tables/01/test_1, 'ch5.nauu.com')
ReplicatedMergeTree('/clickhouse/tables/01/test_1, 'ch6.nauu.com')
```

​	现在将它改写为分布式DDL的形式：

```sql
CREATE TABLE test_1_local ON CLUSTER shard_2(
id UInt64
--这里可以使用任意其他表引擎，
)ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/test_1', '{replica}')
ORDER BY id
┌─host──────┬─port─┬─status─┬─error─┬─num_hosts_active─┐
│ ch6.nauu.com │ 9000 │ 0 │ │ 0 │
│ ch5.nauu.com │ 9000 │ 0 │ │ 0 │
└─────────┴────┴──────┴─────┴───────────┘
```

​	在执行了上述语句之后，ClickHouse会根据集群shard_2的配置信息，分别在CH5和CH6节点本地创建test_1_local。

​	如果要删除test_1_local，则执行下面的分布式DROP：

```sql
DROP TABLE test_1_local ON CLUSTER shard_2
┌─host──────┬─port─┬─status─┬─error─┬─num_hosts_active─┐
│ ch6.nauu.com │ 9000 │ 0 │ │ 0 │
│ ch5.nauu.com │ 9000 │ 0 │ │ 0 │
└─────────┴────┴──────┴─────┴───────────┘
```

​	值得注意的是，**在改写的CREATE语句中，用{shard}和{replica}两个动态宏变量代替了先前的硬编码方式**。执行下面的语句查询系统表，能够看到当前ClickHouse节点中已存在的宏变量：

```shell
--ch5节点
SELECT * FROM system.macros
┌─macro───┬─substitution─┐
│ replica │ ch5.nauu.com │
│ shard │ 01 │
└───────┴─────────┘
--ch6节点
SELECT * FROM remote('ch6.nauu.com:9000', 'system', 'macros', 'default')
┌─macro───┬─substitution─┐
│ replica │ ch6.nauu.com │
│ shard │ 02 │
└───────┴─────────┘
```

​	**这些宏变量是通过配置文件的形式预先定义在各个节点的配置文件中的**，配置文件如下所示。

​	在CH5节点的config.xml配置中预先定义了分区01的宏变量：

```xml
<macros>
  <shard>01</shard>
  <replica>ch5.nauu.com</replica>
</macros>
```

​	在CH6节点的config.xml配置中预先定义了分区02的宏变量：

```xml
<macros>
  <shard>02</shard>
  <replica>ch6.nauu.com</replica>
</macros>
```

1. 数据结构

   **与ReplicatedMergeTree类似，分布式DDL语句在执行的过程中也需要借助ZooKeeper的协同能力，以实现日志分发**。

   1）ZooKeeper内的节点结构

   在默认情况下，分布式DDL在ZooKeeper内使用的根路径为：

   ```shell
   /clickhouse/task_queue/ddl
   ```

   该路径由config.xml内的distributed_ddl配置指定：

   ```xml
   <distributed_ddl>
     <!-- Path in ZooKeeper to queue with DDL queries -->
     <path>/clickhouse/task_queue/ddl</path>
   </distributed_ddl>
   ```

   **在此根路径之下，还有一些其他的监听节点，其中包括/query-[seq]，其是DDL操作日志，每执行一次分布式DDL查询，在该节点下就会新增一条操作日志，以记录相应的操作指令**。当各个节点监听到有新日志加入的时候，便会响应执行。DDL操作日志使用ZooKeeper的持久顺序型节点，每条指令的名称以query-为前缀，后面的序号递增，例如query-0000000000、query-0000000001等。在每条query-[seq]操作日志之下，还有两个状态节点：

   （1）/query-[seq]/active：用于状态监控等用途，在任务的执行过程中，在该节点下会临时保存当前集群内状态为active的节点。

   （2）/query-[seq]/finished：用于检查任务完成情况，在任务的执行过程中，每当集群内的某个host节点执行完毕之后，便会在该节点下写入记录。例如下面的语句。

   ```shell
   /query-000000001/finished
   ch5.nauu.com:9000 : 0
   ch6.nauu.com:9000 : 0
   ```

   上述语句表示集群内的CH5和CH6两个节点已完成任务。

   2）DDLLogEntry日志对象的数据结构

   在/query-[seq]下记录的日志信息由DDLLogEntry承载，它拥有如下几个核心属性：

   （1）query记录了DDL查询的执行语句，例如：

   ```shell
   query: DROP TABLE default.test_1_local ON CLUSTER shard_2
   ```

   （2）hosts记录了指定集群的hosts主机列表，集群由分布式DDL语句中的ON CLUSTER指定，例如：

   ```shell
   hosts: ['ch5.nauu.com:9000','ch6.nauu.com:9000']
   ```

   <u>在分布式DDL的执行过程中，会根据hosts列表逐个判断它们的执行状态</u>。

   （3）initiator记录初始化host主机的名称，hosts主机列表的取值来自于初始化host节点上的集群，例如：

   ```shell
   initiator: ch5.nauu.com:9000
   ```

   hosts主机列表的取值来源等同于下面的查询：

   ```sql
   --从initiator节点查询cluster信息
   SELECT host_name FROM
   remote('ch5.nauu.com:9000', 'system', 'clusters', 'default')
   WHERE cluster = 'shard_2'
   ┌─host_name────┐
   │ ch5.nauu.com │
   │ ch6.nauu.com │
   └──────────┘
   ```

2. 分布式DDL的核心执行流程

   ​	与副本协同的核心流程类似，接下来，就以10.4.2节中介绍的创建test_1_local的过程为例，解释分布式DDL的核心执行流程。整个流程如图10-11所示。

   ​	整个流程从上至下按照时间顺序进行，其大致分成3个步骤。现在，根据图10-11所示编号讲解整个过程。

   ![image.png](https://segmentfault.com/img/bVbI40o)

   ​	图10-11 分布式CREATE查询的核心流程示意图（其他DDL语句与此类似）

   （1）推送DDL日志：首先在CH5节点执行CREATE TABLE ONCLUSTER，本着**谁执行谁负责**的原则，在这个案例中将会由CH5节点负责创建DDLLogEntry日志并将日志推送到ZooKeeper，同时也会由这个节点负责监控任务的执行进度。

   （2）拉取日志并执行：CH5和CH6两个节点分别监听/ddl/query-0000000064日志的推送，于是它们分别拉取日志到本地。首先，它们会判断各自的host是否被包含在DDLLog-Entry的hosts列表中。如果包含在内，则进入执行流程，<u>执行完毕后将状态写入finished节点</u>；**如果不包含，则忽略这次日志的推送**。

   （3）确认执行进度：在步骤1执行DDL语句之后，客户端会阻塞等待180秒，以期望所有host执行完毕。如果等待时间大于180秒，则会转入后台线程继续等待（等待时间由distributed_ddl_task_timeout参数指定，默认为180秒）。

## * 10.5 Distributed原理解析

​	**Distributed表引擎是分布式表的代名词，<u>它自身不存储任何数据</u>，而是作为数据分片的透明代理，能够自动路由数据至集群中的各个节点，所以Distributed表引擎需要和其他数据表引擎一起协同工作**，如图10-12所示。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021062418014944.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1YW5nZmFuMzIy,size_16,color_FFFFFF,t_70#pic_center)	图10-12 分布式表与本地表一对多的映射关系示意图

​	从实体表层面来看，一张分片表由两部分组成：

+ 本地表：通常以_local为后缀进行命名。本地表是承接数据的载体，可以使用非Distributed的任意表引擎，**一张本地表对应了一个数据分片**。

+ 分布式表：通常以_all为后缀进行命名。**分布式表只能使用Distributed表引擎，它与本地表形成一对多的映射关系，日后将通过分布式表代理操作多张本地表**。

​	<u>**对于分布式表与本地表之间表结构的一致性检查，Distributed表引擎采用了读时检查的机制**，这意味着如果它们的表结构不兼容，只有在查询时才会抛出错误，而在创建表时并不会进行检查</u>。<u>不同ClickHouse节点上的本地表之间，使用不同的表引擎也是可行的，但是通常不建议这么做，保持它们的结构一致，有利于后期的维护并避免造成不可预计的错误</u>。

### 10.5.1 定义形式

​	Distributed表引擎的定义形式如下所示：

```sql
ENGINE = Distributed(cluster, database, table [,sharding_key])
```

​	其中，各个参数的含义分别如下：

+ cluster：集群名称，与集群配置中的自定义名称相对应。在对分布式表执行写入和查询的过程中，它会使用集群的配置信息来找到相应的host节点。

+ database和table：分别对应数据库和表的名称，分布式表使用这组配置映射到本地表。

+ sharding_key：分片键，选填参数。在数据写入的过程中，分布式表会依据分片键的规则，将数据分布到各个host节点的本地表。

​	现在用示例说明Distributed表的声明方式，建表语句如下所示：

```sql
CREATE TABLE test_shard_2_all ON CLUSTER sharding_simple (
  id UInt64
)ENGINE = Distributed(sharding_simple, default, test_shard_2_local,rand())
```

​	上述表引擎参数的语义可以理解为，代理的本地表为default.test_shard_2_local，它们分布在集群sharding_simple的各个shard，<u>在数据写入时会根据rand()随机函数的取值决定数据写入哪个分片</u>。**值得注意的是，此时此刻本地表还未创建，所以从这里也能看出，<u>Distributed表运用的是读时检查的机制，对创建分布式表和本地表的顺序并没有强制要求</u>。同样值得注意的是，在上面的语句中使用了ON CLUSTER分布式DDL，这意味着在集群的每个分片节点上，都会创建一张Distributed表，如此一来便可以从其中任意一端发起对所有分片的读、写请求**，如图10-13所示。

![image.png](https://segmentfault.com/img/bVbI40O)

​	图10-13 示例中分布式表与本地表的关系拓扑图

​	接着需要创建本地表，**一张本地表代表着一个数据分片**。这里同样可以利用先前已经配置好的集群配置，使用分布式DDL语句迅速的在各个节点创建相应的本地表：

```sql
CREATE TABLE test_shard_2_local ON CLUSTER sharding_simple (
  id UInt64
)ENGINE = MergeTree()
ORDER BY id
PARTITION BY id
```

​	至此，拥有两个数据分片的分布式表test_shard_2就建好了。

### * 10.5.2 查询的分类

​	Distributed表的查询操作可以分为如下几类：

+ 会作用于本地表的查询：<u>对于INSERT和SELECT查询，Distributed将会以分布式的方式作用于local本地表</u>。而对于这些查询的具体执行逻辑，将会在后续小节介绍。

+ 只会影响Distributed自身，不会作用于本地表的查询：**Distributed支持部分元数据操作，包括CREATE、DROP、RENAME和ALTER，其中ALTER并不包括分区的操作（ATTACH PARTITION、REPLACE PARTITION等）。<u>这些查询只会修改Distributed表自身，并不会修改local本地表</u>**。例如**<u>要彻底删除一张分布式表，则需要分别删除分布式表和本地表</u>**，示例如下。

  ```sql
  --删除分布式表
  DROP TABLE test_shard_2_all ON CLUSTER sharding_simple
  --删除本地表
  DROP TABLE test_shard_2_local ON CLUSTER sharding_simple
  ```

+ **不支持的查询：<u>Distributed表不支持任何MUTATION类型的操作，包括ALTER DELETE和ALTER UPDATE</u>**。

### * 10.5.3 分片规则

​	关于分片的规则这里将做进一步的展开说明。**<u>分片键要求返回一个整型类型的取值，包括Int系列和UInt系列</u>**。

​	例如分片键可以是一个具体的整型列字段：

```sql
按照用户id的余数划分
Distributed(cluster, database, table ,userid)
```

​	也可以是一个返回整型的表达式：

```sql
--按照随机数划分
Distributed(cluster, database, table ,rand())
--按照用户id的散列值划分
Distributed(cluster, database, table , intHash64(userid))
```

​	**<u>如果不声明分片键，那么分布式表只能包含一个分片，这意味着只能映射一张本地表</u>**，否则，在写入数据时将会得到如下异常：

```shell
Method write is not supported by storage Distributed with more than one shard and
no sharding key provided
```

​	<u>如果一张分布式表只包含一个分片，那就意味着其失去了使用的意义了。所以虽然分片键是选填参数，但是通常都会按照业务规则进行设置</u>。

​	那么数据具体是如何被划分的呢？想要讲清楚这部分逻辑，首先需要明确几个概念。

1. 分片权重（weight）

   在集群的配置中，有一项weight（分片权重）的设置：

   ```xml
   <sharding_simple><!-- 自定义集群名称 -->
     <shard><!-- 分片 -->
       <weight>10</weight><!-- 分片权重 -->
       ……
     </shard>
     <shard>
       <weight>20</weight>
       ……
     </shard>
     …
   ```

   <u>weight默认为1，虽然可以将它设置成任意整数，但**官方建议应该尽可能设置成较小的值**。分片权重会影响数据在分片中的倾斜程度，一个分片权重值越大，那么它被写入的数据就会越多</u>。

2. **slot（槽）**

   <u>slot可以理解成许多小的水槽，如果把数据比作是水的话，那么数据之水会顺着这些水槽流进每个数据分片。slot的数量等于所有分片的权重之和</u>，假设集群sharding_simple有两个Shard分片，第一个分片的weight为10，第二个分片的weight为20，那么slot的数量则等于30。slot按照权重元素的取值区间，与对应的分片形成映射关系。在这个示例中，如果slot值落在[0,10)区间，则对应第一个分片；如果slot值落在[10,20]区间，则对应第二个分片。

3. 选择函数

   选择函数用于判断一行待写入的数据应该被写入哪个分片，整个判断过程大致分成两个步骤：

   （1）它会找出slot的取值，其计算公式如下：

   ```sql
   slot = shard_value % sum_weight
   ```

   其中，shard_value是分片键的取值；sum_weight是所有分片的权重之和；slot等于shard_value和sum_weight的余数。假设某一行数据的shard_value是10，sum_weight是30（两个分片，第一个分片权重为10，第二个分片权重为20），那么slot值等于10（10%30=10）。

   （2）基于slot值找到对应的数据分片。当slot值等于10的时候，它属于[10,20)区间，所以这行数据会对应到第二个Shard分片。

   ​	整个过程的示意如图10-14所示。

   ![image.png](https://segmentfault.com/img/bVbI418)

   图10-14 基于选择函数实现数据分片的逻辑示意图

### * 10.5.4 分布式写入的核心流程

​	在向集群内的分片写入数据时，通常有两种思路：<u>一种是借助外部计算系统，事先将数据均匀分片，再借由计算系统直接将数据写入ClickHouse集群的各个本地表</u>，如图10-15所示。

![image.png](https://segmentfault.com/img/bVbI42t)

​	图10-15 由外部系统将数据写入本地表

​	**上述这种方案通常拥有更好的写入性能，因为分片数据是被并行点对点写入的**。但是这种方案的实现主要依赖于外部系统，而不在于ClickHouse自身，所以这里主要会介绍第二种思路。

​	第二种思路是通过Distributed表引擎代理写入分片数据的，接下来开始介绍数据写入的核心流程。

​	为了便于理解整个过程，这里会将分片写入、副本复制拆分成两个部分进行讲解。在讲解过程中，会使用两个特殊的集群分别进行演示：第一个集群拥有2个分片和0个副本，通过这个示例向大家讲解分片写入的核心流程；第二个集群拥有1个分片和1个副本，通过这个示例向大家讲解副本复制的核心流程。

1. 将数据写入分片的核心流程

   在对Distributed表执行INSERT查询的时候，会进入数据写入分片的执行逻辑，它的核心流程如图10-16所示。

   ![image.png](https://segmentfault.com/img/bVbI43s)

   ![image.png](https://segmentfault.com/img/bVbI43J)

   ​	图10-16 由Distributed表将数据写入多个分片

   ​	在这个流程中，继续使用集群sharding_simple的示例，该集群由2个分片和0个副本组成。整个流程从上至下按照时间顺序进行，其大致分成5个步骤。现在根据图10-16所示编号讲解整个过程。

   1）在第一个分片节点写入本地分片数据

   首先在CH5节点，对分布式表test_shard_2_all执行INSERT查询，尝试写入10、30、200和55四行数据。执行之后分布式表主要会做两件事情：第一，根据分片规则划分数据，在这个示例中，30会归至分片1，而10、200和55则会归至分片2；第二，将属于当前分片的数据直接写入本地表test_shard_2_local。

   2）第一个分片建立远端连接，准备发送远端分片数据

   将归至远端分片的数据**<u>以分区为单位</u>**，分别写入test_shard_2_all存储目录下的临时bin文件，数据文件的命名规则如下：

   ```shell
   /database@host:port/[increase_num].bin
   ```

   由于在这个示例中只有一个远端分片CH6，所以它的临时数据文件如下所示：

   ```shell
   /test_shard_2_all/default@ch6.nauu.com:9000/1.bin
   ```

   10、200和55三行数据会被写入上述这个临时数据文件。接着，会尝试与远端CH6分片建立连接：

   ```shell
   Connection (ch6.nauu.com:9000): Connected to ClickHouse server
   ```

   3）第一个分片向远端分片发送数据

   此时，会有另一组监听任务负责监听/test_shard_2_all目录下的文件变化，这些任务负责将目录数据发送至远端分片：

   ```shell
   test_shard_2_all.Distributed.DirectoryMonitor:
   Started processing /test_shard_2_all/default@ch6.nauu.com:9000/1.bin
   ```

   其中，**每份目录将会由独立的线程负责发送，数据在传输之前会被压缩**。

   4）第二个分片接收数据并写入本地

   CH6分片节点确认建立与CH5的连接：

   ```shell
   TCPHandlerFactory: TCP Request. Address: CH5:45912
   TCPHandler: Connected ClickHouse server
   ```

   在接收到来自CH5发送的数据后，将它们写入本地表：

   ```shell
   executeQuery: (from CH5) INSERT INTO default.test_shard_2_local
   --第一个分区
   Reserving 1.00 MiB on disk 'default'
   Renaming temporary part tmp_insert_10_1_1_0 to 10_1_1_0.
   --第二个分区
   Reserving 1.00 MiB on disk 'default'
   Renaming temporary part tmp_insert_200_2_2_0 to 200_2_2_0.
   --第三个分区
   Reserving 1.00 MiB on disk 'default'
   Renaming temporary part tmp_insert_55_3_3_0 to 55_3_3_0.
   ```

   5）由第一个分片确认完成写入

   最后，还是由CH5分片确认所有的数据发送完毕：

   ```shell
   Finished processing /test_shard_2_all/default@ch6.nauu.com:9000/1.bin
   ```

   至此，整个流程结束。

   <u>可以看到，在整个过程中，Distributed表负责所有分片的写入工作。本着**谁执行谁负责**的原则，在这个示例中，由CH5节点的分布式表负责切分数据，并向所有其他分片节点发送数据</u>。

   在由Distributed表负责向远端分片发送数据时，有**异步写**和**同步写**两种模式：

   + 如果是异步写，则在Distributed表写完本地分片之后，INSERT查询就会返回成功写入的信息；
   + 如果是同步写，则在执行INSERT查询之后，会等待所有分片完成写入。

   使用何种模式由insert_distributed_sync参数控制，默认为false，即异步写。如果将其设置为true，则可以一进步通过insert_distributed_timeout参数控制同步等待的超时时间。

2. 副本复制数据的核心流程

   如果在集群的配置中包含了副本，那么除了刚才的分片写入流程之外，还会触发副本数据的复制流程。**数据在多个副本之间，有两种复制实现方式：一种是继续借助Distributed表引擎，由它将数据写入副本；另一种则是借助ReplicatedMergeTree表引擎实现副本数据的分发**。两种方式的区别如图10-17所示。

   ![image.png](https://segmentfault.com/img/bVbI44V)

   图10-17 使用Distributed与ReplicatedMergeTree分发副本数据的对比示意图

   1）通过Distributed复制数据

   在这种实现方式下，即使本地表不使用ReplicatedMergeTree表引擎，也能实现数据副本的功能。**Distributed会同时负责分片和副本的数据写入工作，而副本数据的写入流程与分片逻辑相同**，详情参见10.5.4节。现在用一个简单示例说明。首先让我们再重温一下集群sharding_simple_1的配置，它的配置如下：

   ```xml
   <!-- 1个分片 1个副本-->
   <sharding_simple_1>
     <shard>
       <replica>
         <host>ch5.nauu.com</host>
         <port>9000</port>
       </replica>
       <replica>
         <host>ch6.nauu.com</host>
         <port>9000</port>
       </replica>
     </shard>
   </sharding_simple_1>
   ```

   现在，尝试在这个集群内创建数据表，首先创建本地表：

   ```sql
   CREATE TABLE test_sharding_simple1_local ON CLUSTER sharding_simple_1(
     id UInt64
   )ENGINE = MergeTree()
   ORDER BY id
   ```

   接着创建Distributed分布式表：

   ```sql
   CREATE TABLE test_sharding_simple1_all
   (
     id UInt64
   )ENGINE = Distributed(sharding_simple_1, default, test_sharding_simple1_local,rand())
   ```

   之后，向Distributed表写入数据，它会负责将数据写入集群内的每个replica。

   细心的朋友应该能够发现，**在这种实现方案下，Distributed节点需要同时负责分片和副本的数据写入工作，它很有可能会成为写入的单点瓶颈**，所以就有了接下来将要说明的第二种方案。

   2）通过ReplicatedMergeTree复制数据

   <u>如果在集群的shard配置中增加internal_replication参数并将其设置为true（默认为false），那么Distributed表在该shard中只会选择一个合适的replica并对其写入数据。**此时，如果使用ReplicatedMergeTree作为本地表的引擎，则在该shard内，多个replica副本之间的数据复制会交由ReplicatedMergeTree自己处理，不再由Distributed负责，从而为其减负**</u>。

   在shard中选择replica的算法大致如下：首选，在ClickHouse的服务节点中，拥有一个全局计数器errors_count，当服务出现任何异常时，该计数累积加1；接着，**<u>当一个shard内拥有多个replica时，选择errors_count错误最少的那个</u>**。

   加入internal_replication配置后示例如下所示：

   ```xml
   <shard>
     <!-- 由ReplicatedMergeTree复制表自己负责数据分发 -->
     <internal_replication>true</internal_replication>
     <replica>
       <host>ch5.nauu.com</host>
       <port>9000</port>
     </replica>
     <replica>
       <host>ch6.nauu.com</host>
       <port>9000</port>
     </replica>
   </shard>
   ```

   关于Distributed表引擎如何将数据写入分片，请参见10.5.4节；而关于Replicated-MergeTree表引擎如何复制分发数据，请参见10.3.2节。

### * 10.5.5 分布式查询的核心流程

​	**与数据写入有所不同，<u>在面向集群查询数据的时候，只能通过Distributed表引擎实现</u>**。**<u>当Distributed表接收到SELECT查询的时候，它会依次查询每个分片的数据，再合并汇总返回</u>**。接下来将对数据查询时的重点逻辑进行介绍。

1. 多副本的路由规则

   在查询数据的时候，如果集群中的一个shard，拥有多个replica，那么Distributed表引擎需要面临副本选择的问题。它会使用负载均衡算法从众多replica中选择一个，而具体使用何种负载均衡算法，则由load_balancing参数控制：

   ```shell
   load_balancing = random/nearest_hostname/in_order/first_or_random
   ```

   有如下四种负载均衡算法：

   1）random

   **random是默认的负载均衡算法**，正如前文所述，<u>在ClickHouse的服务节点中，拥有一个全局计数器errors_count，当服务发生任何异常时，该计数累积加1。而random算法会选择errors_count错误数量最少的replica，如果多个replica的errors_count计数相同，则在它们之中随机选择一个</u>。

   2）nearest_hostname

   nearest_hostname可以看作random算法的变种，首先它会选择errors_count错误数量最少的replica，如果多个replica的errors_count计数相同，则选择集群配置中host名称与当前host最相似的一个。而相似的规则是以当前host名称为基准按字节逐位比较，找出不同字节数最少的一个，例如CH5-1-1和CH5-1-2.nauu.com有一个字节不同：

   ```shell
   CH5-1-1
   CH5-1-2.nauu.com
   ```

   而CH5-1-1和CH5-2-2则有两个字节不同：

   ```shell
   CH5-1-1
   CH5-2-2
   ```

   3）in_order

   in_order同样可以看作random算法的变种，首先它会选择errors_count错误数量最少的replica，如果多个replica的errors_count计数相同，则按照集群配置中replica的定义顺序逐个选择。

   4）first_or_random

   first_or_random可以看作in_order算法的变种，首先它会选择errors_count错误数量最少的replica，如果多个replica的errors_count计数相同，它首先会选择集群配置中第一个定义的replica，如果该replica不可用，则进一步随机选择一个其他的replica。

2. 多分片查询的核心流程

   分布式查询与分布式写入类似，同样本着**谁执行谁负责**的原则，它会由接收SELECT查询的Distributed表，并负责串联起整个过程。<u>首先它会将针对分布式表的SQL语句，按照分片数量将查询拆分成若干个针对本地表的子查询，然后向各个分片发起查询，最后再汇总各个分片的返回结果</u>。如果对分布式表按如下方式发起查询：

   ```sql
   SELECT * FROM distributed_table
   ```

   那么它会将其转为如下形式之后，再发送到远端分片节点来执行：

   ```sql
   SELECT * FROM local_table
   ```

   以sharding_simple集群的test_shard_2_all为例，假设在CH5节点对分布式表发起查询：

   ```sql
   SELECT COUNT(*) FROM test_shard_2_all
   ```

   那么，**Distributed表引擎会<u>将查询计划转换为多个分片的UNION联合查询</u>**，如图10-18所示。

   ![image.png](https://segmentfault.com/img/bVbI455)

   ​	图10-18 对分布式表执行COUNT查询的执行计划

   整个执行计划从下至上大致分成两个步骤：

   1）查询各个分片数据

   在图10-18所示执行计划中，<u>One和Remote步骤是**并行执行**的，它们分别负责了本地和远端分片的查询动作</u>。其中，在One步骤会将SQL转换成对本地表的查询：

   ```sql
   SELECT COUNT() FROM default.test_shard_2_local
   ```

   而在Remote步骤中，会建立与CH6节点的连接，并向其发起远程查询：

   ```shell
   Connection (ch6.nauu.com:9000): Connecting. Database: …
   ```

   CH6节点在接收到来自CH5的查询请求后，开始在本地执行。<u>同样，SQL会转换成对本地表的查询</u>：

   ```shell
   executeQuery: (from CH5:45992, initial_query_id: 4831b93b-5ae6-4b18-bac9-
   e10cc9614353) WITH toUInt32(2) AS _shard_num
   SELECT COUNT() FROM default.test_shard_2_local
   ```

   2）合并返回结果

   多个分片数据均查询返回后，按如下方法在CH5节点将它们合并：

   ```shell
   Read 2 blocks of partially aggregated data, total 2 rows.
   Aggregator: Converting aggregated data to blocks
   ……
   ```

3. 使用Global优化分布式子查询

   **如果在分布式查询中使用子查询，可能会面临两难的局面**。下面来看一个示例。假设有这样一张分布式表test_query_all，它拥有两个分片，而表内的数据如下所示：

   ```sql
   CH5节点test_query_local
   ┌─id─┬─repo─┐
   │ 1 │ 100 │
   │ 2 │ 100 │
   │ 3 │ 100 │
   └───┴─────┘
   CH6节点test_query_local
   ┌─id─┬─repo─┐
   │ │ │
   │ 3 │ 200 │
   │ 4 │ 200 │
   └───┴─────┘
   ```

   其中，id代表用户的编号，repo代表仓库的编号。如果现在有一项查询需求，要求找到同时拥有两个仓库的用户，应该如何实现？对于这类交集查询的需求，可以使用IN子查询，此时你会面临两难的选择：IN查询的子句应该使用本地表还是分布式表？（使用JOIN面临的情形与IN类似）。

   1）使用本地表的问题

   如果在IN查询中使用本地表，例如下面的语句：

   ```sql
   SELECT uniq(id) FROM test_query_all WHERE repo = 100
   AND id IN (SELECT id FROM test_query_local WHERE repo = 200)
   ┌─uniq(id)─┐
   │ 0 │
   └───────┘
   ```

   那么你会发现返回的结果是错误的。这是为什么呢？这是因为**<u>分布式表在接收到查询之后，会将上述SQL替换成本地表的形式，再发送到每个分片进行执行</u>**：

   ```sql
   SELECT uniq(id) FROM test_query_local WHERE repo = 100
   AND id IN (SELECT id FROM test_query_local WHERE repo = 200)
   ```

   **注意，IN查询的子句使用的是本地表**：

   ```sql
   SELECT id FROM test_query_local WHERE repo = 200
   ```

   由于在单个分片上只保存了部分的数据，所以该SQL语句没有匹配到任何数据，如图10-19所示。

   ![image.png](https://segmentfault.com/img/bVbI46R)

   ​	图10-19 使用本地表作为IN查询子句的执行逻辑

   从上图中可以看到，单独在分片1或分片2内均无法找到repo同时等于100和200的数据。

   2）使用分布式表的问题

   为了解决返回结果错误的问题，现在尝试在IN查询子句中使用分布式表：

   ```sql
   SELECT uniq(id) FROM test_query_all WHERE repo = 100
   AND id IN (SELECT id FROM test_query_all WHERE repo = 200)
   ┌─uniq(id)─┐
   │ 1 │
   └───────┘
   ```

   这次返回了正确的查询结果。那是否意味着使用这种方案就万无一失了呢？通过进一步观察执行日志会发现，**情况并非如此，该查询的请求被放大了两倍**。

   **这是由于在IN查询子句中，同样也使用了分布式表查询**：

   ```sql
   SELECT id FROM test_query_all WHERE repo = 200
   ```

   所以在CH6节点接收到这条SQL之后，它将再次向其他分片发起远程查询，如图10-20所示。

   ![在这里插入图片描述](https://img-blog.csdnimg.cn/8981076c15404db495f5a2ff2f812b68.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl8zOTAyNTM2Mg==,size_16,color_FFFFFF,t_70)

   ​	图10-20 IN查询子句查询放大原因示意

   因此可以得出结论，**<u>在IN查询子句使用分布式表的时候，查询请求会被放大N的平方倍</u>**，其中N等于集群内分片节点的数量，假如集群内有10个分片节点，则在一次查询的过程中，会最终导致100次的查询请求，这显然是不可接受的。

   3）使用GLOBAL优化查询

   **<u>为了解决查询放大的问题，可以使用GLOBAL IN或JOIN进行优化</u>**。现在对刚才的SQL进行改造，为其增加**GLOBAL修饰符**：

   ```sql
   SELECT uniq(id) FROM test_query_all WHERE repo = 100
   AND id GLOBAL IN (SELECT id FROM test_query_all WHERE repo = 200)
   ```

   再次分析查询的核心过程，如图10-21所示。

   整个过程由上至下大致分成5个步骤：

   （1）将IN子句单独提出，发起了一次分布式查询。

   （2）将分布式表转local本地表后，分别在本地和远端分片执行查询。

   （3）**将IN子句查询的结果进行汇总，并放入一张临时的内存表进行保存**。

   （4）**将内存表发送到远端分片节点**。

   （5）将分布式表转为本地表后，开始执行完整的SQL语句，**IN子句直接使用临时内存表的数据**。

   至此，整个核心流程结束。可以看到，**在使用GLOBAL修饰符之后，ClickHouse使用内存表临时保存了IN子句查询到的数据，并将其发送到远端分片节点，以此到达了数据共享的目的，从而避免了查询放大的问题**。<u>**由于数据会在网络间分发，所以需要特别注意临时表的大小，IN或者JOIN子句返回的数据不宜过大。如果表内存在重复数据，也可以事先在子句SQL中增加DISTINCT以实现去重**</u>。

   ![在这里插入图片描述](https://img-blog.csdnimg.cn/caa551875a6e499e9f9cfe530d8fd5a7.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl8zOTAyNTM2Mg==,size_16,color_FFFFFF,t_70)

   ​	图10-21 使用GLOBAL IN查询的流程示意图

## 10.6 本章小结

​	本章全方面介绍了副本、分片和集群的使用方法，并且详细介绍了它们的作用以及核心工作流程。

​	首先我们介绍了数据副本的特点，并详细介绍了ReplicatedMergeTree表引擎，它是MergeTree表引擎的变种，同时也是数据副本的代名词；接着又介绍了数据分片的特点及作用，同时在这个过程中引入了ClickHouse集群的概念，并讲解了它的工作原理；最后介绍了Distributed表引擎的核心功能与工作流程，借助它的能力，可以实现分布式写入与查询。

​	下一章将介绍与ClickHouse管理与运维相关的内容。

# 11. 管理与运维

​	本章将对ClickHouse管理与运维相关的知识进行介绍，通过对本章所学知识的运用，大家在实际工作中能够让ClickHouse变得更加安全与健壮。在先前的演示案例中，为求简便，我们一直在使用默认的default用户，且采用无密码登录方式，这显然不符合生产环境的要求。所以在接下来的内容中，将会介绍ClickHouse的权限、熔断机制、数据备份和服务监控等知识。

## 11.1 用户配置

​	user.xml配置文件默认位于/etc/clickhouse-server路径下，ClickHouse使用它来定义用户相关的配置项，包括系统参数的设定、用户的定义、权限以及熔断机制等。

### 11.1.1 用户profile

​	<u>用户profile的作用类似于用户角色。可以预先在user.xml中为ClickHouse定义多组profile，并为每组profile定义不同的配置项，以实现配置的复用</u>。以下面的配置为例：

```xml
<yandex>
  <profiles><!-- 配置profile -->
    <default> <!-- 自定义名称，默认角色-->
      <max_memory_usage>10000000000</max_memory_usage>
      <use_uncompressed_cache>0</use_uncompressed_cache>
    </default>
    <test1> <!-- 自定义名称，默认角色-->
      <allow_experimental_live_view>1</allow_experimental_live_view>
      <distributed_product_mode>allow</distributed_product_mode>
    </test1>
  </profiles>
  ……
```

​	在这组配置中，预先定义了default和test1两组profile。引用相应的profile名称，便会获得相应的配置。我们可以在CLI中直接切换到想要的profile：

```sql
SET profile = test1
```

​	或是在定义用户的时候直接引用（在11.1.3节进行演示）。

​	**在所有的profile配置中，名称为default的profile将作为默认的配置被加载，所以它必须存在**。如果缺失了名为default的profile，在登录时将会出现如下错误提示：

```shell
DB::Exception: There is no profile 'default' in configuration file..
```

​	profile配置支持继承，实现继承的方式是在定义中引用其他的profile名称，例如下面的例子所示：

```xml
<normal_inherit> <!-- 只有read查询权限-->
  <profile>test1</profile>
  <profile>test2</profile>
  <distributed_product_mode>deny</distributed_product_mode>
</normal_inherit>
```

​	这个名为normal_inherit的profile继承了test1和test2的所有配置项，并且使用新的参数值覆盖了test1中原有的distributed_product_mode配置项。

### 11.1.2 配置约束

​	<u>constraints标签可以设置一组约束条件，以保障profile内的参数值不会被随意修改</u>。约束条件有如下三种规则：

+ Min：最小值约束，在设置相应参数的时候，取值不能小于该阈值。

+ Max：最大值约束，在设置相应参数的时候，取值不能大于该阈值。

+ **Readonly：只读约束，该参数值不允许被修改**。

​	现在举例说明：

```xml
<profiles><!-- 配置profiles -->
  <default> <!-- 自定义名称，默认角色-->
    <max_memory_usage>10000000000</max_memory_usage>
    <distributed_product_mode>allow</distributed_product_mode>
    <constraints><!-- 配置约束-->
      <max_memory_usage>
        <min>5000000000</min>
        <max>20000000000</max>
      </max_memory_usage>
      <distributed_product_mode>
        <readonly/>
      </distributed_product_mode>
    </constraints>
  </default>
```

​	从上面的配置定义中可以看出，在default默认的profile内，给两组参数设置了约束。其中，为max_memory_usage设置了min和max阈值；而为distributed_product_mode设置了只读约束。现在尝试修改max_memory_usage参数，将它改为50：

```shell
SET max_memory_usage = 50
DB::Exception: Setting max_memory_usage shouldn't be less than 5000000000.
```

​	可以看到，最小值约束阻止了这次修改。

​	接着继续修改distributed_product_mode参数的取值：

```shell
SET distributed_product_mode = 'deny'
DB::Exception: Setting distributed_product_mode should not be changed.
```

​	同样，配置约束成功阻止了预期外的修改。

​	**<u>还有一点需要特别明确，在default中默认定义的constraints约束，将作为默认的全局约束，自动被其他profile继承</u>**。

### 11.1.3 用户定义

​	使用users标签可以配置自定义用户。如果打开user.xml配置文件，会发现已经默认配置了default用户，在此之前的所有示例中，一直使用的正是这个用户。定义一个新用户，必须包含以下几项属性。

1. username

   username用于指定登录用户名，这是全局唯一属性。该属性比较简单，这里就不展开介绍了。

2. password

   password用于设置登录密码，支持明文、SHA256加密和double_sha1加密三种形式，可以任选其中一种进行设置。现在分别介绍它们的使用方法。

   （1）明文密码：在使用明文密码的时候，直接通过password标签定义，例如下面的代码。

   ```xml
   <password>123</password>
   ```

   **如果password为空，则表示免密码登录**：

   ```xml
   <password></password>
   ```

   （2）SHA256加密：在使用SHA256加密算法的时候，需要通过password_sha256_hex标签定义密码，例如下面的代码。

   ```xml
   <password_sha256_hex>a665a45920422f9d417e4867efdc4fb8a04a1f3fff1fa07e998e86f7f7a2
   7ae3</password_sha256_hex>
   ```

   可以执行下面的命令获得密码的加密串，例如对明文密码123进行加密：

   ```xml
   # echo -n 123 | openssl dgst -sha256
   (stdin)= a665a45920422f9d417e4867efdc4fb8a04a1f3fff1fa07e998e86f7f7a27ae3
   ```

   （3）double_sha1加密：在使用double_sha1加密算法的时候，则需要通过password_double_sha1_hex标签定义密码，例如下面的代码。

   ```xml
   <password_double_sha1_hex>23ae809ddacaf96af0fd78ed04b6a265e05aa257</password_double_sha1_hex>
   ```

   可以执行下面的命令获得密码的加密串，例如对明文密码123进行加密：

   ```shell
   # echo -n 123 | openssl dgst -sha1 -binary | openssl dgst -sha1
   (stdin)= 23ae809ddacaf96af0fd78ed04b6a265e05aa257
   3. networks
   ```

   <u>networks表示被允许登录的网络地址，用于限制用户登录的客户端地址</u>，关于这方面的介绍将会在11.2节展开。

3. profile

   用户所使用的profile配置，直接引用相应的名称即可，例如：

   ```xml
   <default>
     <profile>default</profile>
   </default>
   ```

   该配置的语义表示，用户default使用了名为default的profile。

4. quota

   <u>quota用于设置该用户能够使用的资源限额，可以理解成一种熔断机制</u>。关于这方面的介绍将会在11.3节展开。

   现在用一个完整的示例说明用户的定义方法。首先创建一个使用明文密码的用户user_plaintext：

   ```xml
   <yandex>
     <profiles>
       ……
     </profiles>
     <users>
       <default><!—默认用户 -->
         ……
       </default>
       <user_plaintext>
         <password>123</password>
         <networks>
           <ip>::/0</ip>
         </networks>
         <profile>normal_1</profile>
         <quota>default</quota>
       </user_plaintext>
   ```

   由于配置了密码，所以在登录的时候需要附带密码参数：

   ```shell
   # clickhouse-client -h 10.37.129.10 -u user_plaintext --password 123
   Connecting to 10.37.129.10:9000 as user user_plaintext.
   ```

   接下来是两组使用了加密算法的用户，首先是用户user_sha256：

   ```xml
   <user_sha256>
     <!-- echo -n 123 | openssl dgst -sha256 !-->
     <password_sha256_hex>a665a45920422f9d417e4867efdc4fb8a04a1f3fff1fa07e998e86f7f7a2
       7ae3</password_sha256_hex>
     <networks>
       <ip>::/0</ip>
     </networks>
     <profile>default</profile>
     <quota>default</quota>
   </user_sha256>
   ```

   然后是用户user_double_sha1：

   ```xml
   <user_double_sha1>
     <!-- echo -n 123 | openssl dgst -sha1 -binary | openssl dgst -sha1 !-->
     <password_double_sha1_hex>23ae809ddacaf96af0fd78ed04b6a265e05aa257</password_double_sha1_hex>
     <networks>
       <ip>::/0</ip>
     </networks>
     <profile>default</profile>
     <quota>limit_1</quota>
   </user_double_sha1>
   ```

   这些用户在登录时同样需要附带加密前的密码，例如：

   ```shell
   # clickhouse-client -h 10.37.129.10 -u user_sha256 --password 123
   Connecting to 10.37.129.10:9000 as user user_sha256.
   ```

## 11.2 权限管理

​	权限管理是一个始终都绕不开的话题，ClickHouse分别从访问、查询和数据等角度出发，层层递进，为我们提供了一个较为立体的权限体系。

### 11.2.1 访问权限

​	访问层控制是整个权限体系的第一层防护，它又可进一步细分成两类权限。

1. 网络访问权限

   网络访问权限使用networks标签设置，用于限制某个用户登录的客户端地址，有IP地址、host主机名称以及正则匹配三种形式，可以任选其中一种进行设置。

   （1）IP地址：直接使用IP地址进行设置。

   ```xml
   <ip>127.0.0.1</ip>
   ```

   （2）host主机名称：通过host主机名称设置。

   ```xml
   <host>ch5.nauu.com</host>
   ```

   （3）正则匹配：通过表达式来匹配host名称。

   ```xml
   <host>^ch\d.nauu.com$</host>
   ```

   现在用一个示例说明：

   ```xml
   <user_normal>
     <password></password>
     <networks>
       <ip>10.37.129.13</ip>
     </networks>
     <profile>default</profile>
     <quota>default</quota>
   </user_normal>
   ```

   ​	用户user_normal限制了客户端IP，在设置之后，该用户将只能从指定的地址登录。此时如果从非指定IP的地址进行登录，例如：

   ```shell
   --从10.37.129.10登录
   # clickhouse-client -u user_normal
   ```

   则将会得到如下错误：

   ```shell
   DB::Exception: User user_normal is not allowed to connect from address
   10.37.129.10.
   ```

2. 数据库与字典访问权限

   在客户端连入服务之后，可以进一步限制某个用户<u>数据库</u>和<u>字典</u>的访问权限，它们分别通过allow_databases和allow_dictionaries标签进行设置。**如果不进行任何定义，则表示不进行限制**。现在继续在用户user_normal的定义中增加权限配置：

   ```xml
   <user_normal>
     ……
     <allow_databases>
       <database>default</database>
       <database>test_dictionaries</database>
     </allow_databases>
     <allow_dictionaries>
       <dictionary>test_flat_dict</dictionary>
     </allow_dictionaries>
   </user_normal>
   ```

   通过上述操作，该用户在登录之后，将只能看到为其开放了访问权限的数据库和字典。

### 11.2.2 查询权限

​	查询权限是整个权限体系的第二层防护，它决定了一个用户能够执行的查询语句。查询权限可以分成以下四类：

+ 读权限：包括SELECT、EXISTS、SHOW和DESCRIBE查询。
+ 写权限：包括INSERT和**OPTIMIZE**查询。
+ 设置权限：包括SET查询。
+ DDL权限：包括CREATE、DROP、ALTER、RENAME、ATTACH、DETACH和TRUNCATE查询。
+ **其他权限：包括KILL和USE查询，任何用户都可以执行这些查询**。

​	上述这四类权限，通过以下两项配置标签控制：

（1）readonly：读权限、写权限和设置权限均由此标签控制，它有三种取值。

+ 当取值为0时，不进行任何限制（默认值）。
+ 当取值为1时，只拥有读权限（只能执行SELECT、EXISTS、SHOW和DESCRIBE）。
+ 当取值为2时，拥有读权限和设置权限（在读权限基础上，增加了SET查询）。

（2）allow_ddl：DDL权限由此标签控制，它有两种取值。

+ 当取值为0时，不允许DDL查询。

+ 当取值为1时，允许DDL查询（默认值）。

​	现在继续用一个示例说明。与刚才的配置项不同，readonly和allow_ddl需要定义在用户profiles中，例如：

```xml
<profiles>
  <normal> <!-- 只有read读权限-->
    <readonly>1</readonly>
    <allow_ddl>0</allow_ddl>
  </normal>
  <normal_1> <!-- 有读和设置参数权限-->
    <readonly>2</readonly>
    <allow_ddl>0</allow_ddl>
  </normal_1>
```

​	继续在先前的profiles配置中追加了两个新角色。其中，normal只有读权限，而normal_1则有读和设置参数的权限，它们都没有DDL查询的权限。

​	再次修改用户的user_normal的定义，将它的profile设置为刚追加的normal，这意味着该用户只有读权限。现在开始验证权限的设置是否生效。使用user_normal登录后尝试写操作：

```shell
--登录
# clickhouse-client -h 10.37.129.10 -u user_normal
--写操作
:) INSERT INTO TABLE test_ddl VALUES (1)
DB::Exception: Cannot insert into table in readonly mode.
```

​	可以看到，写操作如期返回了异常。接着执行DDL查询，会发现该操作同样会被限制执行：

```shell
:) CREATE DATABASE test_3
DB::Exception: Cannot create database in readonly mode.
```

​	至此，权限设置已然生效。

### 11.2.3 数据行级权限

​	数据权限是整个权限体系中的第三层防护，它决定了一个用户能够看到什么数据。数据权限使用databases标签定义，它是用户定义中的一项选填设置。database通过定义用户级别的查询过滤器来实现**数据的行级粒度权限**，它的定义规则如下所示：

```xml
<databases>
  <database_name><!--数据库名称-->
    <table_name><!--表名称-->
      <filter> id < 10</filter><!--数据过滤条件-->
        </table_name>
  </database_name>
```

​	其中，database_name表示数据库名称；table_name表示表名称；而filter则是权限过滤的关键所在，它等同于定义了一条WHERE条件子句，与WHERE子句类似，它支持组合条件。现在用一个示例说明。这里还是用user_normal，为它追加databases定义：

```xml
<user_normal>
  ……
  <databases>
    <default><!--默认数据库-->
      <test_row_level><!--表名称-->
        <filter>id < 10</filter>
      </test_row_level>
       <!-- 支持组合条件
       <test_query_all>
          <filter>id <= 100 or repo >= 100</filter>
       </test_query_all> -->
     </default>
  </databases>
```

​	基于上述配置，通过为user_normal用户增加全局过滤条件，实现了该用户在数据表default.test_row_level的行级粒度数据上的权限设置。test_row_level的表结构如下所示：

```sql
CREATE TABLE test_row_level(
  id UInt64,
  name UInt64
)ENGINE = MergeTree()
ORDER BY id
```

​	下面验证权限是否生效。首先写入测试数据：

```sql
INSERT INTO TABLE test_row_level VALUES (1,100),(5,200),(20,200),(30,110)
```

​	写入之后，登录user_normal用户并查询这张数据表：

```sql
SELECT * FROM test_row_level
┌─id─┬─name─┐
│ 1 │ 100 │
│ 5 │ 200 │
└───┴─────┘
```

​	可以看到，<u>在返回的结果数据中，只包含`<filter> id<10 </filter>`的部分</u>，证明权限设置生效了。

​	那么数据权限的设定是如何实现的呢？进一步分析它的执行日志：

```shell
Expression
	Expression
		Filter –增加了过滤的步骤
			MergeTreeThread
```

​	可以发现，上述代码**在普通查询计划的基础之上自动附加了Filter过滤的步骤**。

​	**<u>对于数据权限的使用有一点需要明确，在使用了这项功能之后，PREWHERE优化将不再生效</u>**，例如执行下面的查询语句：

```sql
SELECT * FROM test_row_level where name = 5
```

​	**此时如果使用了数据权限，那么这条SQL将不会进行PREWHERE优化；反之，如果没有设置数据权限，则会进行PREWHERE优化**，例如：

```shell
InterpreterSelectQuery: MergeTreeWhereOptimizer: condition "name = 5" moved to
PREWHERE
```

​	<u>所以，是直接利用ClickHouse的内置过滤器，还是通过拼接WHERE查询条件的方式实现行级数据权限，需要用户在具体的使用场景中进行权衡</u>。

## * 11.3 熔断机制

​	**熔断是限制资源被过度使用的一种自我保护机制，当使用的资源数量达到阈值时，那么正在进行的操作会被自动中断**。按照使用资源统计方式的不同，熔断机制可以分为两类。

1. 根据时间周期的累积用量熔断

   在这种方式下，系统资源的用量是按照时间周期累积统计的，当累积量达到阈值，则直到下个计算周期开始之前，该用户将无法继续进行操作。这种方式通过users.xml内的quotas标签来定义资源配额。以下面的配置为例：

   ```xml
   <quotas>
     <default> <!-- 自定义名称 -->
       <interval>
         <duration>3600</duration><!-- 时间周期 单位：秒 -->
         <queries>0</queries>
         <errors>0</errors>
         <result_rows>0</result_rows>
         <read_rows>0</read_rows>
         <execution_time>0</execution_time>
       </interval>
     </default>
   </quotas>
   ```

   其中，各配置项的含义如下：

   + default：表示自定义名称，全局唯一。

   + duration：表示累积的时间周期，单位是秒。

   + queries：表示在周期内允许执行的查询次数，0表示不限制。

   + errors：表示在周期内允许发生异常的次数，0表示不限制。
   + result_row：表示在周期内允许查询返回的结果行数，0表示不限制。
   + read_rows：表示在周期内在分布式查询中，<u>允许远端节点读取的数据行数</u>，0表示不限制。
   + execution_time：表示周期内允许执行的查询时间，单位是秒，0表示不限制。

   由于上述示例中各配置项的值均为0，所以对资源配额不做任何限制。现在继续声明另外一组资源配额：

   ```xml
   <limit_1>
     <interval>
       <duration>3600</duration>
       <queries>100</queries>
       <errors>100</errors>
       <result_rows>100</result_rows>
       <read_rows>2000</read_rows>
       <execution_time>3600</execution_time>
     </interval>
   </limit_1>
   ```

   ​	为了便于演示，在这个名为limit_1的配额中，在1小时（3600秒）的周期内只允许100次查询。继续修改用户user_normal的配置，为它添加limit_1配额的引用：

   ```xml
   <user_normal>
     <password></password>
     <networks>
       <ip>10.37.129.13</ip>
     </networks>
     <profile>normal</profile>
     <quota>limit_1</quota>
   </user_normal>
   ```

   ​	最后使用user_normal用户登录，测试配额是否生效。在执行了若干查询以后，会发现之后的任何一次查询都将会得到如下异常：

   ```shell
   Quota for user 'user_normal' for 1 hour has been exceeded. Total result rows:
   149, max: 100. Interval will end at 2019-08-29 22:00:00. Name of quota template:
   'limit_1'..
   ```

   ​	上述结果证明熔断机制已然生效。

2. 根据单次查询的用量熔断

   ​	在这种方式下，系统资源的用量是按照单次查询统计的，而具体的熔断规则，则是由许多不同配置项组成的，<u>这些配置项需要定义在用户profile中</u>。**如果某次查询使用的资源用量达到了阈值，则会被中断**。以配置项max_memory_usage为例，它限定了单次查询可以使用的内存用量，在默认的情况下其规定不得超过10 GB，如果一次查询的内存用量超过10 GB，则会得到异常。**<u>需要注意的是，在单次查询的用量统计中，ClickHouse是以分区为最小单元进行统计的（不是数据行的粒度），这意味着单次查询的实际内存用量是有可能超过阈值的</u>**。

   ​	由于篇幅所限，完整的熔断配置请参阅官方手册，这里只列举个别的常用配置项。

   ​	首先介绍一组针对普通查询的熔断配置。

   （1）max_memory_usage：在单个ClickHouse服务进程中，运行一次查询限制使用的最大内存量，默认值为10 GB，其配置形式如下。

   ```xml
   <max_memory_usage>10000000000</max_memory_usage>
   ```

   ​	该配置项还有max_memory_usage_for_user和max_memory_usage_for_all_queries两个变种版本。

   （2）max_memory_usage_for_user：在单个ClickHouse服务进程中，**以用户为单位进行统计**，单个用户在运行查询时限制使用的最大内存量，默认值为0，即不做限制。

   （3）max_memory_usage_for_all_queries：在单个ClickHouse服务进程中，所有运行的查询累加在一起所限制使用的最大内存量，默认为0，即不做限制。

   ​	接下来介绍的是一组与数据写入和聚合查询相关的熔断配置。

   （1）max_partitions_per_insert_block：在单次INSERT写入的时候，限制创建的最大分区个数，默认值为100个。如果超出这个阈值，将会出现如下异常：

   ```shell
   Too many partitions for single INSERT block ……
   ```

   （2）max_rows_to_group_by：在执行GROUP BY聚合查询的时候，限制去重后聚合KEY的最大个数，默认值为0，即不做限制。当超过阈值时，其处理方式由group_by_overflow_mode参数决定。

   （3）group_by_overflow_mode：当max_rows_to_group_by熔断规则触发时，group_by_overflow_mode将会提供三种处理方式。

   + throw：抛出异常，此乃默认值。

   + break：立即停止查询，并返回当前数据。

   + any：仅根据当前已存在的聚合KEY继续完成聚合查询。

   （4）max_bytes_before_external_group_by：在执行GROUP BY聚合查询的时候，限制使用的最大内存量，默认值为0，即不做限制。<u>当超过阈值时，聚合查询将会进一步借用**本地磁盘**</u>。

## 11.4 数据备份

​	在先前的章节中，我们已经知道了数据副本的使用方法，可能有的读者心中会有这样的疑问：既然已经有了数据副本，那么还需要数据备份吗？**数据备份自然是需要的，因为数据副本并不能处理误删数据这类行为**。ClickHouse自身提供了多种备份数据的方法，根据数据规模的不同，可以选择不同的形式。

### 11.4.1 导出文件备份

​	**如果数据的体量较小，可以通过dump的形式将数据导出为本地文件**。例如执行下面的语句将test_backup的数据导出：

```shell
#clickhouse-client --query="SELECT * FROM test_backup" > /chbase/test_backup.tsv
```

​	将备份数据再次导入，则可以执行下面的语句：

```shell
# cat /chbase/test_backup.tsv | clickhouse-client --query "INSERT INTO test_backup FORMAT TSV"
```

​	上述这种dump形式的优势在于，可以**利用SELECT查询并筛选数据，然后按需备份**。**如果是备份整个表的数据，也可以直接复制它的整个目录文件**，例如：

```shell
# mkdir -p /chbase/backup/default/ & cp -r /chbase/data/default/test_backup/chbase/backup/default/
```

### 11.4.2 通过快照表备份

​	快照表实质上就是普通的数据表，它通常按照业务规定的备份频率创建，例如按天或者按周创建。所以首先需要建立一张与原表结构相同的数据表，然后再使用INSERT INTO SELECT句式，点对点地将数据从原表写入备份表。假设数据表test_backup需要按日进行备份，现在为它创建当天的备份表：

```sql
CREATE TABLE test_backup_0206 AS test_backup
```

​	有了备份表之后，就可以点对点地备份数据了，例如：

```sql
INSERT INTO TABLE test_backup_0206 SELECT * FROM test_backup
```

​	如果考虑到容灾问题，也可以将备份表放置在不同的ClickHouse节点上，此时需要将上述SQL语句改成远程查询的形式：

```sql
INSERT INTO TABLE test_backup_0206 SELECT * FROM remote('ch5.nauu.com:9000',
'default', 'test_backup', 'default')
```

### 11.4.3 按分区备份

​	基于数据分区的备份，ClickHouse目前提供了FREEZE与FETCH两种方式，现在分别介绍它们的使用方法。

1. 使用FREEZE备份

   FREEZE的完整语法如下所示：

   ```sql
   ALTER TABLE tb_name FREEZE PARTITION partition_expr
   ```

   分区在被备份之后，会被统一保存到ClickHouse根路径/shadow/N子目录下。其中，**N是一个自增长的整数，它的含义是备份的次数（FREEZE执行过多少次）**，<u>具体次数由shadow子目录下的increment.txt文件记录</u>。而**<u>分区备份实质上是对原始目录文件进行硬链接操作，所以并不会导致额外的存储空间</u>**。<u>整个备份的目录会一直向上追溯至data根路径的整个链路</u>：

   ```shell
   /data/[database]/[table]/[partition_folder]
   ```

   例如执行下面的语句，会对数据表partition_v2的201908分区进行备份：

   ```shell
   :) ALTER TABLE partition_v2 FREEZE PARTITION 201908
   ```

   进入shadow子目录，即能够看到刚才备份的分区目录：

   ```shell
   # pwd
   /chbase/data/shadow/1/data/default/partition_v2
   # ll
   total 4
   drwxr-x---. 2 clickhouse clickhouse 4096 Sep 1 00:22 201908_5_5_0
   ```

   **对于备份分区的还原操作，则需要借助ATTACH装载分区的方式来实现。这意味着如果要还原数据，首先需要主动将shadow子目录下的分区文件复制到相应数据表的detached目录下，然后再使用ATTACH语句装载**。

2. 使用FETCH备份

   **FETCH只支持ReplicatedMergeTree系列的表引擎**，它的完整语法如下所示：

   ```sql
   ALTER TABLE tb_name FETCH PARTITION partition_id FROM zk_path
   ```

   **其工作原理与ReplicatedMergeTree同步数据的原理类似，FETCH通过指定的zk_path找到ReplicatedMergeTree的所有副本实例，然后从中选择一个最合适的副本，并下载相应的分区数据**。例如执行下面的语句：

   ```sql
   ALTER TABLE test_fetch FETCH PARTITION 2019 FROM
   '/clickhouse/tables/01/test_fetch'
   ```

   表示指定将test_fetch的2019分区下载到本地，并保存到对应数据表的detached目录下，目录如下所示：

   ```shell
   data/default/test_fetch/detached/2019_0_0_0
   ```

   **与FREEZE一样，对于备份分区的还原操作，也需要借助ATTACH装载分区来实现。**

   **<u>FREEZE和FETCH虽然都能实现对分区文件的备份，但是它们并不会备份数据表的元数据</u>**。所以说如果想做到万无一失的备份，还需要对数据表的元数据进行备份，它们是/data/metadata目录下的[table].sql文件。**<u>目前这些元数据需要用户通过复制的形式单独备份</u>**。

## 11.5 服务监控

​	基于原生功能对ClickHouse进行监控，可以从两方面着手——**系统表**和**查询日志**。接下来分别介绍它们的使用方法。

### 11.5.1 系统表

​	在众多的SYSTEM系统表中，主要由以下三张表支撑了对ClickHouse运行指标的查询，它们分别是metrics、events和asynchronous_metrics。

1. metrics

   metrics表用于统计ClickHouse服务在运行时，当前**正在执行**的高层次的概要信息，包括正在执行的查询总次数、正在发生的合并操作总次数等。该系统表的查询方法如下所示：

   ```sql
   SELECT * FROM system.metrics LIMIT 5
   ┌─metric──────┬─value─┬─description─────────────────────┐
   │ Query │ 1 │ Number of executing queries │
   │ Merge │ 0 │ Number of executing background merges │
   │ PartMutation │ 0 │ Number of mutations (ALTER DELETE/UPDATE) │
   │ ReplicatedFetch │ 0 │ Number of data parts being fetched from replica │
   │ ReplicatedSend │ 0 │ Number of data parts being sent to replicas │
   └──────────┴─────┴─────────────────────────────┘
   ```

2. events

   events用于统计ClickHouse服务在运行过程中**已经执行过**的高层次的累积概要信息，包括总的查询次数、总的SELECT查询次数等，该系统表的查询方法如下所示：

   ```sql
   SELECT event, value FROM system.events LIMIT 5
   ┌─event─────────────────────┬─value─┐
   │ Query │ 165 │
   │ SelectQuery │ 92 │
   │ InsertQuery │ 14 │
   │ FileOpen │ 3525 │
   │ ReadBufferFromFileDescriptorRead │ 6311 │
   └─────────────────────────┴─────┘
   ```

3. asynchronous_metrics

   asynchronous_metrics用于统计ClickHouse服务运行过程时，当前正在**后台异步运行**的高层次的概要信息，包括当前分配的内存、执行队列中的任务数量等。该系统表的查询方法如下所示：

   ```sql
   SELECT * FROM system.asynchronous_metrics LIMIT 5
   ┌─metric───────────────────────┬─────value─┐
   │ jemalloc.background_thread.run_interval │ 0 │
   │ jemalloc.background_thread.num_runs │ 0 │
   │ jemalloc.background_thread.num_threads │ 0 │
   │ jemalloc.retained │ 79454208 │
   │ jemalloc.mapped │ 531341312 │
   └────────────────────────────┴──────────┘
   ```

### 11.5.2 查询日志

​	查询日志目前主要有6种类型，它们分别从不同角度记录了ClickHouse的操作行为。**所有查询日志在默认配置下都是关闭状态**，需要在config.xml配置中进行更改，接下来分别介绍它们的开启方法。在配置被开启之后，ClickHouse会为每种类型的查询日志自动生成相应的系统表以供查询。

1. query_log

   query_log是最常用的查询日志，它记录了ClickHouse服务中所有**已经执行**的查询记录，它的全局定义方式如下所示：

   ```xml
   <query_log>
     <database>system</database>
     <table>query_log</table>
     <partition_by>toYYYYMM(event_date)</partition_by>
     <!-— 刷新周期 -->
     <flush_interval_milliseconds>7500</flush_interval_milliseconds>
   </query_log>
   ```

   如果只需要为某些用户单独开启query_log，也可以在user.xml的profile配置中按照下面的方式定义：

   ```xml
   <log_queries> 1</log_queries>
   ```

   query_log开启后，即可以通过相应的系统表对记录进行查询：

   ```sql
   SELECT type,concat(substr(query,1,20),'...')query,read_rows,
   query_duration_ms AS duration FROM system.query_log LIMIT 6
   ┌─type──────────┬─query───────────┬─read_rows─┬─duration─┐
   │ QueryStart │ SELECT DISTINCT arra... │ 0 │ 0 │
   │ QueryFinish │ SELECT DISTINCT arra... │ 2432 │ 11 │
   │ QueryStart │ SHOW DATABASES... │ 0 │ 0 │
   │ QueryFinish │ SHOW DATABASES... │ 3 │ 1 │
   │ ExceptionBeforeStart │ SELECT * FROM test_f... │ 0 │ 0 │
   │ ExceptionBeforeStart │ SELECT * FROM test_f... │ 0 │ 0 │
   └─────────────┴───────────────┴───────┴───────┘
   ```

   如上述查询结果所示，query_log日志记录的信息十分完善，涵盖了查询语句、执行时间、执行用户返回的数据量和执行用户等。

2. query_thread_log

   query_thread_log记录了所有线程的执行查询的信息，它的全局定义方式如下所示：

   ```xml
   <query_thread_log>
     <database>system</database>
     <table>query_thread_log</table>
     <partition_by>toYYYYMM(event_date)</partition_by>
     <flush_interval_milliseconds>7500</flush_interval_milliseconds>
   </query_thread_log>
   ```

   同样，如果只需要为某些用户单独开启该功能，可以在user.xml的profile配置中按照下面的方式定义：

   ```xml
   <log_query_threads> 1</log_query_threads>
   ```

   query_thread_log开启后，即可以通过相应的系统表对记录进行查询：

   ```sql
   SELECT thread_name,concat(substr(query,1,20),'...')query,query_duration_ms AS
   duration,memory_usage AS memory FROM system.query_thread_log LIMIT 6
   ┌─thread_name───┬─query───────────┬─duration─┬─memory─┐
   │ ParalInputsProc │ SELECT DISTINCT arra... │ 2 │ 210888 │
   │ ParalInputsProc │ SELECT DISTINCT arra... │ 3 │ 252648 │
   │ AsyncBlockInput │ SELECT DISTINCT arra... │ 3 │ 449544 │
   │ TCPHandler │ SELECT DISTINCT arra... │ 11 │ 0 │
   │ TCPHandler │ SHOW DATABASES... │ 2 │ 0 │
   └──────────┴───────────────┴───────┴──────┘
   ```

   如上述查询结果所示，query_thread_log日志记录的信息涵盖了线程名称、查询语句、执行时间和内存用量等。

3. part_log

   part_log日志记录了MergeTree系列表引擎的分区操作日志，其全局定义方式如下所示：

   ```xml
   <part_log>
     <database>system</database>
     <table>part_log</table>
     <flush_interval_milliseconds>7500</flush_interval_milliseconds>
   </part_log>
   ```

   part_log开启后，即可以通过相应的系统表对记录进行查询：

   ```sql
   SELECT event_type AS type,table,partition_id,event_date FROM system.part_log
   ┌─type────┬─table─────────────┬─partition_id─┬─event_date─┐
   │ NewPart │ summing_table_nested_v1 │ 201908 │ 2020-01-29 │
   │ NewPart │ summing_table_nested_v1 │ 201908 │ 2020-01-29 │
   │ MergeParts │ summing_table_nested_v1 │ 201908 │ 2020-01-29 │
   │ RemovePart │ ttl_table_v1 │ 201505 │ 2020-01-29 │
   │ RemovePart │ summing_table_nested_v1 │ 201908 │ 2020-01-29 │
   │ RemovePart │ summing_table_nested_v1 │ 201908 │ 2020-01-29 │
   └───────┴─────────────────┴─────────┴────────┘
   ```

   如上述查询结果所示，part_log日志记录的信息涵盖了操纵类型、表名称、分区信息和执行时间等。

4. text_log

   text_log日志记录了ClickHouse运行过程中产生的一系列打印日志，包括INFO、DEBUG和Trace，它的全局定义方式如下所示：

   ```xml
   <text_log>
     <database>system</database>
     <table>text_log</table>
     <flush_interval_milliseconds>7500</flush_interval_milliseconds>
   </text_log>
   ```

   text_log开启后，即可以通过相应的系统表对记录进行查询：

   ```sql
   SELECT thread_name,
   concat(substr(logger_name,1,20),'...')logger_name,
   concat(substr(message,1,20),'...')message
   FROM system.text_log LIMIT 5
   ┌─thread_name──┬─logger_name───────┬─message────────────┐
   │ SystemLogFlush │ SystemLog (system.me... │ Flushing system log... │
   │ SystemLogFlush │ SystemLog (system.te... │ Flushing system log... │
   │ SystemLogFlush │ SystemLog (system.te... │ Creating new table s... │
   │ SystemLogFlush │ system.text_log... │ Loading data parts... │
   │ SystemLogFlush │ system.text_log... │ Loaded data parts (0... │
   └──────────┴───────────────┴─────────────────┘
   ```

   如上述查询结果所示，text_log日志记录的信息涵盖了线程名称、日志对象、日志信息和执行时间等。

5. metric_log

   metric_log日志用于将system.metrics和system.events中的数据汇聚到一起，它的全局定义方式如下所示：

   ```xml
   <metric_log>
     <database>system</database>
     <table>metric_log</table>
     <flush_interval_milliseconds>7500</flush_interval_milliseconds>
     <collect_interval_milliseconds>1000</collect_interval_milliseconds>
   </metric_log>
   ```

   其中，collect_interval_milliseconds表示收集metrics和events数据的时间周期。metric_log开启后，即可以通过相应的系统表对记录进行查询。

   除了上面介绍的系统表和查询日志外，ClickHouse还能够与众多的第三方监控系统集成，限于篇幅这里就不再展开了。

## 11.6 本章小结

​	通过对本章的学习，大家可进一步了解ClickHouse的安全性和健壮性。本章首先站在安全的角度介绍了用户的定义方法和权限的设置方法。在权限设置方面，ClickHouse分别从连接访问、资源访问、查询操作和数据权限等几个维度出发，提供了一个较为立体的权限控制体系。接着站在系统运行的角度介绍了如何通过熔断机制保护ClickHouse系统资源不会被过度使用。最后站在运维的角度介绍了数据的多种备份方法以及如何通过系统表和查询日志，实现对日常运行情况的监控。