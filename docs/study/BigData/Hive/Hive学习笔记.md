# Hive学习笔记

> [APACHE HIVE TM -- 官方文档](https://hive.apache.org/)

# Installation and Configuration

> [安装跟着官方教程走就好了](https://cwiki.apache.org/confluence/display/Hive/GettingStarted#GettingStarted-InstallationandConfiguration)	<=	后面又出各种问题，官方这个安装可能对Linux完全没问题？后面我又找了别人mac的安装教程
>
> [Mac上配置Hive3.1.0](https://blog.csdn.net/zhangvalue/article/details/84282827)

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

creates a table called invites with two columns and a partition column called ds. The partition column is a virtual column. It is not part of the data itself but is derived from the partition that a particular dataset is loaded into.

By default, tables are assumed to be of text input format and the delimiters are assumed to be ^A(ctrl-a).