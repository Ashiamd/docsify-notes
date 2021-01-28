# Hive学习笔记

> [APACHE HIVE TM -- 官方文档](https://hive.apache.org/)

# Installation and Configuration

> [安装跟着官方教程走就好了](https://cwiki.apache.org/confluence/display/Hive/GettingStarted#GettingStarted-InstallationandConfiguration)	<=	后面又出各种问题，官方这个安装可能对Linux完全没问题？后面我又找了别人mac的安装教程
>
> [Mac上配置Hive3.1.0](https://blog.csdn.net/zhangvalue/article/details/84282827)	<=	没想到安装个Hive还得用到MySQL，哈哈。

## 安装中可能遇到的异常

> [启动hive报错：java.lang.NoSuchMethodError: com.google.common.base.Preconditions.checkArgument(ZLjava/lang/String;Ljava/lang/Object;)V（已解决）](https://www.cnblogs.com/guohu/p/13200879.html)
>
> [Hive 遇到 Class path contains multiple SLF4J bindings](https://www.cnblogs.com/Jesse-Li/p/7809485.html)
>
> [Hive的安装配置及原理 完美解决解压hive缺少hive-site.xml文件，和安全模式无法启动hive等问题](https://blog.csdn.net/iamboluke/article/details/103496098)

下面这个，则是没有启动Hadoop。

*Mac系统需要注意系统设置的sharing打开“Remote Login”，然后添加ssh登录（把公钥配置到`~/.ssh/known_hosts`这个文件里）*

```shell
Exception in thread "main" java.lang.RuntimeException: java.net.ConnectException: Call From ashiamddeMBP/192.168.31.17 to localhost:9000 failed on connection exception: java.net.ConnectException: Connection refused; For more details see:  http://wiki.apache.org/hadoop/ConnectionRefused
	at org.apache.hadoop.hive.ql.session.SessionState.start(SessionState.java:651)
	at org.apache.hadoop.hive.ql.session.SessionState.beginStart(SessionState.java:591)
	at org.apache.hadoop.hive.cli.CliDriver.run(CliDriver.java:747)
	at org.apache.hadoop.hive.cli.CliDriver.main(CliDriver.java:683)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.apache.hadoop.util.RunJar.run(RunJar.java:323)
	at org.apache.hadoop.util.RunJar.main(RunJar.java:236)
Caused by: java.net.ConnectException: Call From ashiamddeMBP/192.168.31.17 to localhost:9000 failed on connection exception: java.net.ConnectException: Connection refused; For more details see:  http://wiki.apache.org/hadoop/ConnectionRefused
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	at org.apache.hadoop.net.NetUtils.wrapWithMessage(NetUtils.java:837)
	at org.apache.hadoop.net.NetUtils.wrapException(NetUtils.java:757)
	at org.apache.hadoop.ipc.Client.getRpcResponse(Client.java:1566)
	at org.apache.hadoop.ipc.Client.call(Client.java:1508)
	at org.apache.hadoop.ipc.Client.call(Client.java:1405)
	at org.apache.hadoop.ipc.ProtobufRpcEngine2$Invoker.invoke(ProtobufRpcEngine2.java:234)
	at org.apache.hadoop.ipc.ProtobufRpcEngine2$Invoker.invoke(ProtobufRpcEngine2.java:119)
	at com.sun.proxy.$Proxy28.getFileInfo(Unknown Source)
	at org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolTranslatorPB.getFileInfo(ClientNamenodeProtocolTranslatorPB.java:964)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.apache.hadoop.io.retry.RetryInvocationHandler.invokeMethod(RetryInvocationHandler.java:422)
	at org.apache.hadoop.io.retry.RetryInvocationHandler$Call.invokeMethod(RetryInvocationHandler.java:165)
	at org.apache.hadoop.io.retry.RetryInvocationHandler$Call.invoke(RetryInvocationHandler.java:157)
	at org.apache.hadoop.io.retry.RetryInvocationHandler$Call.invokeOnce(RetryInvocationHandler.java:95)
	at org.apache.hadoop.io.retry.RetryInvocationHandler.invoke(RetryInvocationHandler.java:359)
	at com.sun.proxy.$Proxy29.getFileInfo(Unknown Source)
	at org.apache.hadoop.hdfs.DFSClient.getFileInfo(DFSClient.java:1731)
	at org.apache.hadoop.hdfs.DistributedFileSystem$29.doCall(DistributedFileSystem.java:1725)
	at org.apache.hadoop.hdfs.DistributedFileSystem$29.doCall(DistributedFileSystem.java:1722)
	at org.apache.hadoop.fs.FileSystemLinkResolver.resolve(FileSystemLinkResolver.java:81)
	at org.apache.hadoop.hdfs.DistributedFileSystem.getFileStatus(DistributedFileSystem.java:1737)
	at org.apache.hadoop.fs.FileSystem.exists(FileSystem.java:1729)
	at org.apache.hadoop.hive.ql.exec.Utilities.ensurePathIsWritable(Utilities.java:4486)
	at org.apache.hadoop.hive.ql.session.SessionState.createRootHDFSDir(SessionState.java:760)
	at org.apache.hadoop.hive.ql.session.SessionState.createSessionDirs(SessionState.java:701)
	at org.apache.hadoop.hive.ql.session.SessionState.start(SessionState.java:627)
	... 9 more
Caused by: java.net.ConnectException: Connection refused
	at sun.nio.ch.SocketChannelImpl.checkConnect(Native Method)
	at sun.nio.ch.SocketChannelImpl.finishConnect(SocketChannelImpl.java:715)
	at org.apache.hadoop.net.SocketIOWithTimeout.connect(SocketIOWithTimeout.java:206)
	at org.apache.hadoop.net.NetUtils.connect(NetUtils.java:533)
	at org.apache.hadoop.ipc.Client$Connection.setupConnection(Client.java:699)
	at org.apache.hadoop.ipc.Client$Connection.setupIOstreams(Client.java:812)
	at org.apache.hadoop.ipc.Client$Connection.access$3800(Client.java:413)
	at org.apache.hadoop.ipc.Client.getConnection(Client.java:1636)
	at org.apache.hadoop.ipc.Client.call(Client.java:1452)
	... 34 more
```

## HCatalog

> [利用HCatalog管理元数据](https://www.jianshu.com/p/af3fcc4511b9)
>
> [HCatalog快速入门](https://blog.csdn.net/ancony_/article/details/79903705)

## Hive, Map-Reduce and Local-Mode

Hive compiler generates map-reduce jobs for most queries. These jobs are then submitted to the Map-Reduce cluster indicated by the variable:

```
  mapred.job.tracker
```

While this usually points to a map-reduce cluster with multiple nodes, <u>Hadoop also offers a nifty option to run map-reduce jobs locally on the user's workstation</u>. This can be very useful to run queries over small data sets – in such cases local mode execution is usually significantly faster than submitting jobs to a large cluster. Data is accessed transparently from HDFS. **Conversely, local mode only runs with one reducer and can be very slow processing larger data sets.**

**Starting with release 0.7, Hive fully supports local mode execution**. To enable this, the user can enable the following option:

```
  hive> SET mapreduce.framework.name=local;
```

In addition, `mapred.local.dir` should point to a path that's valid on the local machine (for example `/tmp/<username>/mapred/local`). (Otherwise, the user will get an exception allocating local disk space.)

**Starting with release 0.7, Hive also supports a mode to run map-reduce jobs in local-mode automatically.** The relevant options are `hive.exec.mode.local.auto`, `hive.exec.mode.local.auto.inputbytes.max`, and `hive.exec.mode.local.auto.tasks.max`:

```
  hive> SET hive.exec.mode.local.auto=false;
```

**Note that this feature is *disabled* by default**. If enabled, Hive analyzes the size of each map-reduce job in a query and may run it locally if the following thresholds are satisfied:

- The total input size of the job is lower than: `hive.exec.mode.local.auto.inputbytes.max` (128MB by default)
- The total number of map-tasks is less than: `hive.exec.mode.local.auto.tasks.max` (4 by default)
- The total number of reduce tasks required is 1 or 0.

So for queries over small data sets, or for queries with multiple map-reduce jobs where the input to subsequent jobs is substantially smaller (because of reduction/filtering in the prior job), jobs may be run locally.

Note that there may be differences in the runtime environment of Hadoop server nodes and the machine running the Hive client (because of different jvm versions or different software libraries). This can cause unexpected behavior/errors while running in local mode. **Also note that local mode execution is done in a separate, child jvm (of the Hive client)**. If the user so wishes, the maximum amount of memory for this child jvm can be controlled via the option `hive.mapred.local.mem`. By default, it's set to zero, in which case Hive lets Hadoop determine the default memory limits of the child jvm.

# DDL Operations

 The Hive DDL operations are documented in [Hive Data Definition Language](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL).

## Creating Hive Tables

> [关于hive异常：Unable to instantiate org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStor](https://blog.csdn.net/hhj724/article/details/79094138)
>
> [Unable to instantiate org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient](https://www.58jb.com/html/89.html)

```shell
  hive> CREATE TABLE pokes (foo INT, bar STRING);
```

creates a table called pokes with two columns, the first being an integer and the other a string.

```shell
  hive> CREATE TABLE invites (foo INT, bar STRING) PARTITIONED BY (ds STRING);
```

**creates a table called invites with two columns and a partition column called ds. The partition column is a virtual column. It is not part of the data itself but is derived from the partition that a particular dataset is loaded into.**

By default, tables are assumed to be of text input format and the delimiters are assumed to be ^A(ctrl-a).

## Browsing through Tables

```shell
  hive> SHOW TABLES;
```

lists all the tables.

```shell
  hive> SHOW TABLES '.*s';
```

lists all the table that end with 's'. The pattern matching follows Java regular expressions. Check out this link for documentation http://java.sun.com/javase/6/docs/api/java/util/regex/Pattern.html.

```shell
hive> DESCRIBE invites;
```

shows the list of columns.

## Altering and Dropping Tables

Table names can be [changed](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL#LanguageManualDDL-RenameTable) and columns can be [added or replaced](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL#LanguageManualDDL-Add/ReplaceColumns):

```shell
  hive> ALTER TABLE events RENAME TO 3koobecaf;
  hive> ALTER TABLE pokes ADD COLUMNS (new_col INT);
  hive> ALTER TABLE invites ADD COLUMNS (new_col2 INT COMMENT 'a comment');
  hive> ALTER TABLE invites REPLACE COLUMNS (foo INT, bar STRING, baz INT COMMENT 'baz replaces new_col2');
```

**Note that REPLACE COLUMNS replaces all existing columns and only changes the table's schema, not the data**. The table must use a native SerDe. 

**REPLACE COLUMNS can also be used to drop columns from the table's schema**:

```shell
  hive> ALTER TABLE invites REPLACE COLUMNS (foo INT COMMENT 'only keep the first column');
```

Dropping tables:

```shell
  hive> DROP TABLE pokes;
```

## Metadata Store

Metadata is in an embedded Derby database whose disk storage location is determined by the Hive configuration variable named `javax.jdo.option.ConnectionURL`. By default this location is `./metastore_db` (see `conf/hive-default.xml`).

**Right now, in the default configuration, this metadata can only be seen by one user at a time.**

Metastore can be stored in any database that is supported by JPOX. The location and the type of the RDBMS can be controlled by the two variables `javax.jdo.option.ConnectionURL` and `javax.jdo.option.ConnectionDriverName`. Refer to JDO (or JPOX) documentation for more details on supported databases. The database schema is defined in JDO metadata annotations file `package.jdo` at `src/contrib/hive/metastore/src/model`.

In the future, the metastore itself can be a standalone server.

If you want to run the metastore as a network server so it can be accessed from multiple nodes, see [Hive Using Derby in Server Mode](https://cwiki.apache.org/confluence/display/Hive/HiveDerbyServerMode).

# DML Operations

The Hive DML operations are documented in [Hive Data Manipulation Language](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DML).

Loading data from flat files into Hive:

```
  hive> LOAD DATA LOCAL INPATH './examples/files/kv1.txt' OVERWRITE INTO TABLE pokes;
```

Loads a file that contains two columns separated by ctrl-a into pokes table. 'LOCAL' signifies that the input file is on the local file system. **If 'LOCAL' is omitted then it looks for the file in HDFS.**

The keyword 'OVERWRITE' signifies that existing data in the table is deleted. **If the 'OVERWRITE' keyword is omitted, data files are appended to existing data sets.**

NOTES:

- NO verification of data against the schema is performed by the load command.
- **If the file is in hdfs, it is moved into the Hive-controlled file system namespace.**
  The root of the Hive directory is specified by the option `hive.metastore.warehouse.dir` in `hive-default.xml`. We advise users to create this directory before trying to create tables via Hive.

```
  hive> LOAD DATA LOCAL INPATH './examples/files/kv2.txt' OVERWRITE INTO TABLE invites PARTITION (ds='2008-08-15');
  hive> LOAD DATA LOCAL INPATH './examples/files/kv3.txt' OVERWRITE INTO TABLE invites PARTITION (ds='2008-08-08');
```

The two LOAD statements above load data into two different partitions of the table invites. Table invites must be created as partitioned by the key ds for this to succeed.

```
  hive> LOAD DATA INPATH '/user/myname/kv2.txt' OVERWRITE INTO TABLE invites PARTITION (ds='2008-08-15');
```

The above command will load data from an HDFS file/directory to the table.
**Note that loading data from HDFS will result in moving the file/directory. As a result, the operation is almost instantaneous.**

# SQL Operations

> **下面操作前如果安装Hive配置的是MySQL，那么需要先开启MySQL服务，且保证本地Hadoop已经启动，保证Hadoop网络通讯不受影响（开放端口、如果有防火墙干扰可以考虑暂时关闭<=学习测试环境）**

The Hive query operations are documented in [Select](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Select).

## Example Queries

Some example queries are shown below. They are available in `build/dist/examples/queries`.
More are available in the Hive sources at `ql/src/test/queries/positive`.

### SELECTS and FILTERS

> [hadoop拒绝history通过19888端口连接查看已完成的job的日志](https://blog.csdn.net/u014326004/article/details/80332126)
>
> 在hadoop的sbin目录执行`./mr-jobhistory-daemon.sh start historyserver`
>
> [NoSuchFieldException: parentOffset - Hive on Spark](https://stackoverflow.com/questions/60811684/nosuchfieldexception-parentoffset-hive-on-spark)	<=	即检查自己JAVA_HOME是jdk8而不是11
>
> **注意，如果出现Socket连接问题，可以关闭防火墙试试看（我就是自己的TripMode网络防火墙干扰了Hadoop的连接）**

```
  hive> SELECT a.foo FROM invites a WHERE a.ds='2008-08-15';
```

selects column 'foo' from all rows of partition `ds=2008-08-15` of the `invites` table. **The results are not stored anywhere, but are displayed on the console.**

**Note that in all the examples that follow, `INSERT` (into a Hive table, local directory or HDFS directory) is optional.**

```
  hive> INSERT OVERWRITE DIRECTORY '/tmp/hdfs_out' SELECT a.* FROM invites a WHERE a.ds='2008-08-15';
```

selects all rows from partition `ds=2008-08-15` of the `invites` table into an HDFS directory. The result data is in files (depending on the number of mappers) in that directory.
**NOTE: partition columns if any are selected by the use of *. They can also be specified in the projection clauses.**

**Partitioned tables must always have a partition selected in the `WHERE` clause of the statement.**

```
  hive> INSERT OVERWRITE LOCAL DIRECTORY '/tmp/local_out' SELECT a.* FROM pokes a;
```

selects all rows from pokes table into a local directory.

```
  hive> INSERT OVERWRITE TABLE events SELECT a.* FROM profiles a;
  hive> INSERT OVERWRITE TABLE events SELECT a.* FROM profiles a WHERE a.key < 100;
  hive> INSERT OVERWRITE LOCAL DIRECTORY '/tmp/reg_3' SELECT a.* FROM events a;
  hive> INSERT OVERWRITE DIRECTORY '/tmp/reg_4' select a.invites, a.pokes FROM profiles a;
  hive> INSERT OVERWRITE DIRECTORY '/tmp/reg_5' SELECT COUNT(*) FROM invites a WHERE a.ds='2008-08-15';
  hive> INSERT OVERWRITE DIRECTORY '/tmp/reg_5' SELECT a.foo, a.bar FROM invites a;
  hive> INSERT OVERWRITE LOCAL DIRECTORY '/tmp/sum' SELECT SUM(a.pc) FROM pc1 a;
```

selects the sum of a column. The avg, min, or max can also be used. Note that for versions of Hive which don't include [HIVE-287](https://issues.apache.org/jira/browse/HIVE-287), you'll need to use `COUNT(1)` in place of `COUNT(*)`.

### GROUP BY

> *这里events表需要先根据前面的CREATE TABLE指令建表，且需要和invites表的列一致，才能成功清空events数据再把invites的数据覆盖上去*

```
  hive> FROM invites a INSERT OVERWRITE TABLE events SELECT a.bar, count(*) WHERE a.foo > 0 GROUP BY a.bar;
  hive> INSERT OVERWRITE TABLE events SELECT a.bar, count(*) FROM invites a WHERE a.foo > 0 GROUP BY a.bar;
```

Note that for versions of Hive which don't include [HIVE-287](https://issues.apache.org/jira/browse/HIVE-287), you'll need to use `COUNT(1)` in place of `COUNT(*)`.

### JOIN

*这个需要events这个表DDL预先定义三列，且数据类型符合SELECT投影的三列*

```
  hive> FROM pokes t1 JOIN invites t2 ON (t1.bar = t2.bar) INSERT OVERWRITE TABLE events SELECT t1.bar, t1.foo, t2.foo;
```

### MULTITABLE INSERT

```
  FROM src
  INSERT OVERWRITE TABLE dest1 SELECT src.* WHERE src.key < 100
  INSERT OVERWRITE TABLE dest2 SELECT src.key, src.value WHERE src.key >= 100 and src.key < 200
  INSERT OVERWRITE TABLE dest3 PARTITION(ds='2008-04-08', hr='12') SELECT src.key WHERE src.key >= 200 and src.key < 300
  INSERT OVERWRITE LOCAL DIRECTORY '/tmp/dest4.out' SELECT src.value WHERE src.key >= 300;
```

### STREAMING

```
  hive> FROM invites a INSERT OVERWRITE TABLE events SELECT TRANSFORM(a.foo, a.bar) AS (oof, rab) USING '/bin/cat' WHERE a.ds > '2008-08-09';
```

This streams the data in the map phase through the script `/bin/cat` (like Hadoop streaming).
**Similarly – streaming can be used on the reduce side** (please see the [Hive Tutorial](https://cwiki.apache.org/confluence/display/Hive/Tutorial#Tutorial-Custommap%2Freducescripts) for examples).

# Simple Example Use Cases

## MovieLens User Ratings

First, create a table with tab-delimited text file format:

```
CREATE TABLE u_data (
  userid INT,
  movieid INT,
  rating INT,
  unixtime STRING)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t'
STORED AS TEXTFILE;
```

Then, download the data files from **MovieLens 100k** on the [GroupLens datasets](http://grouplens.org/datasets/movielens/) page (which also has a README.txt file and index of unzipped files):

```
wget http://files.grouplens.org/datasets/movielens/ml-100k.zip
```

or:

```
curl --remote-name http://files.grouplens.org/datasets/movielens/ml-100k.zip
```

Note:  If the link to [GroupLens datasets](http://grouplens.org/datasets/movielens/) does not work, please report it on [HIVE-5341](https://issues.apache.org/jira/browse/HIVE-5341) or send a message to the [user@hive.apache.org mailing list](http://hive.apache.org/mailing_lists.html).

Unzip the data files:

```shell
unzip ml-100k.zip
```

And load `u.data` into the table that was just created:

```sql
LOAD DATA LOCAL INPATH '<path>/u.data'
OVERWRITE INTO TABLE u_data;
```

Count the number of rows in table u_data:

```sql
SELECT COUNT(*) FROM u_data;
```

Note that for older versions of Hive which don't include [HIVE-287](https://issues.apache.org/jira/browse/HIVE-287), you'll need to use COUNT(1) in place of COUNT(*).

Now we can do some complex data analysis on the table `u_data`:

Create `weekday_mapper.py`:

```sql
import sys
import datetime

for line in sys.stdin:
  line = line.strip()
  userid, movieid, rating, unixtime = line.split('\t')
  weekday = datetime.datetime.fromtimestamp(float(unixtime)).isoweekday()
  print '\t'.join([userid, movieid, rating, str(weekday)])
```

Use the mapper script:

```sql
CREATE TABLE u_data_new (
  userid INT,
  movieid INT,
  rating INT,
  weekday INT)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t';

add FILE weekday_mapper.py;

INSERT OVERWRITE TABLE u_data_new
SELECT
  TRANSFORM (userid, movieid, rating, unixtime)
  USING 'python weekday_mapper.py'
  AS (userid, movieid, rating, weekday)
FROM u_data;

SELECT weekday, COUNT(*)
FROM u_data_new
GROUP BY weekday;
```

Note that if you're using Hive 0.5.0 or earlier you will need to use `COUNT(1)` in place of `COUNT(*)`.

## Apache Weblog Data

> [hive 建表报错：ParseException - cannot recognize input near 'end' 'string'](https://blog.csdn.net/u011940366/article/details/51396152/)

The format of Apache weblog is customizable, while most webmasters use the default.
For default Apache weblog, we can create a table with the following command.

More about RegexSerDe can be found here in [HIVE-662](https://issues.apache.org/jira/browse/HIVE-662) and [HIVE-1719](https://issues.apache.org/jira/browse/HIVE-1719).

```sql
CREATE TABLE apachelog (
  host STRING,
  identity STRING,
  `user` STRING,
  `time` STRING,
  request STRING,
  status STRING,
  size STRING,
  referer STRING,
  agent STRING)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
WITH SERDEPROPERTIES (
  "input.regex" = "([^]*) ([^]*) ([^]*) (-|\\[^\\]*\\]) ([^ \"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\".*\") ([^ \"]*|\".*\"))?"
)
STORED AS TEXTFILE;
```

---

# LanguageManual

> [LanguageManual](https://cwiki.apache.org/confluence/display/Hive/LanguageManual)

