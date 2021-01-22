# Hadoop官方教程笔记

# 1. Hadoop: 部署单节点集群

## 1. 目的

​	部署单节点Hadoop，以便之后使用Hadoop MapReduce和分布式文件系统HDFS

## 2. 准备工作

### 2.1 平台

​	一般建议用Linux

### 2.2 需要的软件

1. Java
2. ssh（顺带建议安装pdsh管理ssh资源）

### 2.3 安装软件

在Ubuntu发行版本上

```shell
$ sudo apt-get install ssh
$ sudo apt-get install pdsh
```

## 3. 下载

> [下载地址](https://www.apache.org/dyn/closer.cgi/hadoop/common/)
>
> [文件签名校验教程](https://www.apache.org/info/verification.html)

这里我选择[最新的稳定版本3.3.0](https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/stable/hadoop-3.3.0.tar.gz)进行下载。（mac Big Sur 11.0.1）

## 4. 部署Hadoop集群

1. 解压压缩包
2. 确保有配置java的环境变量"JAVA_HOME"
3. 尝试运行hadoop。`bin/hadoop`。正常应该有输出关于hadoop如何使用的一些文字介绍

Hadoop支持3种集群部署模式：

- [Local (Standalone) Mode](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html#Standalone_Operation)	（单机模式）
- [Pseudo-Distributed Mode](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html#Pseudo-Distributed_Operation)	（伪分布式模式）
- [Fully-Distributed Mode](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html#Fully-Distributed_Operation)	（完全分布式模式）

### 4.1 单机操作

​	默认情况下，hadoop就是作为单个java程序，以非分布式的模式运行。单节点模式适合debug调试。

​	下面以解压后的conf目录作为输入，查找匹配指定正则表达式规则的匹配项，将结果输出到我们自定义的目录。

```shell
  $ mkdir input
  $ cp etc/hadoop/*.xml input
  $ bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.2.1.jar grep input output 'dfs[a-z.]+'
  $ cat output/*
```

​	最后我获取到的内容是`1	dfsadmin`

### 4.2 伪分布式操作

​	Hadoop可以在一个节点上以伪分布式模式运行，每个Hadoop守护进程都在单独的Java进程中运行。

####  4.2.1 配置

Use the following:

etc/hadoop/core-site.xml:

```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```

etc/hadoop/hdfs-site.xml:

```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```

> 这两个文件的\<configuration\>\</configuration\>	本来都是空的。

#### 4.2.2 设置免密ssh

> mac需要在系统设置的"共享"里设置"允许远程登录"
>
> [关于Mac中ssh: connect to host localhost port 22: Connection refused](https://blog.csdn.net/u011068475/article/details/52883677)

Now check that you can ssh to the localhost without a passphrase:

```ssh
  $ ssh localhost
```

If you cannot ssh to localhost without a passphrase, execute the following commands:

```ssh
  $ ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
  $ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
  $ chmod 0600 ~/.ssh/authorized_keys
```

#### 4.2.3 执行

> [MapReduce和YARN区别](https://blog.csdn.net/hahachenchen789/article/details/80527706)

The following instructions are to run a MapReduce job locally. If you want to execute a job on YARN, see [YARN on Single Node](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html#YARN_on_Single_Node).

1. Format the filesystem:

   ```
     $ bin/hdfs namenode -format
   ```

2. Start NameNode daemon and DataNode daemon:

   ```
     $ sbin/start-dfs.sh
   ```

   The hadoop daemon log output is written to the `$HADOOP_LOG_DIR` directory (defaults to `$HADOOP_HOME/logs`).

3. Browse the web interface for the NameNode; by default it is available at:

   - NameNode - `http://localhost:9870/`

4. Make the HDFS directories required to execute MapReduce jobs:

   ```
     $ bin/hdfs dfs -mkdir /user
     $ bin/hdfs dfs -mkdir /user/<username>
   ```

5. Copy the input files into the distributed filesystem:

   ```
     $ bin/hdfs dfs -mkdir input
     $ bin/hdfs dfs -put etc/hadoop/*.xml input
   ```

6. Run some of the examples provided:

   ```
     $ bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.2.1.jar grep input output 'dfs[a-z.]+'
   ```

7. Examine the output files: Copy the output files from the distributed filesystem to the local filesystem and examine them:

   ```
     $ bin/hdfs dfs -get output output
     $ cat output/*
   ```

   or

   View the output files on the distributed filesystem:

   ```
     $ bin/hdfs dfs -cat output/*
   ```

8. When you’re done, stop the daemons with:

   ```
     $ sbin/stop-dfs.sh
   ```

#### 4.2.4 YARN on a Single Node

You can run a MapReduce job on YARN in a pseudo-distributed mode by setting a few parameters and running ResourceManager daemon and NodeManager daemon in addition.

The following instructions assume that 1. ~ 4. steps of [the above instructions](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html#Execution) are already executed.

1. Configure parameters as follows:

   `etc/hadoop/mapred-site.xml`:(这个文件的`<configuration></configuration>`原本是空的)

   ```
   <configuration>
       <property>
           <name>mapreduce.framework.name</name>
           <value>yarn</value>
       </property>
       <property>
           <name>mapreduce.application.classpath</name>
           <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
       </property>
   </configuration>
   ```

   `etc/hadoop/yarn-site.xml`:(这个文件的`<configuration></configuration>`原本是空的)

   ```
   <configuration>
       <property>
           <name>yarn.nodemanager.aux-services</name>
           <value>mapreduce_shuffle</value>
       </property>
       <property>
           <name>yarn.nodemanager.env-whitelist</name>
           <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
       </property>
   </configuration>
   ```

2. Start ResourceManager daemon and NodeManager daemon:

   ```
     $ sbin/start-yarn.sh
   ```

3. Browse the web interface for the ResourceManager; by default it is available at:

   - ResourceManager - `http://localhost:8088/`

4. Run a MapReduce job.

5. When you’re done, stop the daemons with:

   ```
     $ sbin/stop-yarn.sh
   ```

### 4.3 完全分布式操作

​	For information on setting up fully-distributed, non-trivial clusters see [Cluster Setup](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/ClusterSetup.html).

# 5. mac安装hadoop

>[mac安装hadoop及配置伪分布式](https://blog.csdn.net/weixin_44570264/article/details/106872445)	<=	跟着这个确实可以
>
>```shell
>vim ~/.bash_profile
>
># hadoop-3.3.0 第一条是hadoop的解压目录
>export HADOOP_HOME=/Users/ashiamd/mydocs/dev-tools/hadoop/hadoop-3.3.0
>export HADOOP_INSTALL=$HADOOP_HOME
>export HADOOP_MAPRED_HOME=$HADOOP_HOME
>export HADOOP_COMMON_HOME=$HADOOP_HOME
>export HADOOP_HDFS_HOME=$HADOOP_HOME
>export YARN_HOME=$HADOOP_HOME
>export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
>export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib"
>export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
>```

