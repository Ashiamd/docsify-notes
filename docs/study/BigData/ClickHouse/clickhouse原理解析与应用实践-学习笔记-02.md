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

P341