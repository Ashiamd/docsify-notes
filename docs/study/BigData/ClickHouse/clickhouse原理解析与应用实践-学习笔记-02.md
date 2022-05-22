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

# 10. 副本与分片

P429