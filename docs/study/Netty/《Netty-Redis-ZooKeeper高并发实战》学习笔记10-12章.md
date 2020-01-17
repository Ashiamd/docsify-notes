# 《Netty、Redis、ZooKeeper高并发实战》学习笔记

> 最近一段时间学习IM编程，但是之前急于需要成品，所以学得不是很懂，最后也没能弄好，所以打算系统地学习一下，然后再继续IM编程。

## 第10章 ZooKeeper分布式协调

​	ZooKeeper（本书也简称ZK）是Hadoop的正式子项目，它是一个针对大型分布式系统的可靠协调系统，提供的功能包括：**配置维护、名字服务、分布式同步、组服务**等。

​	<u>ZooKeeper的目标就是封装好复杂易出错的关键服务，将简单易用的接口和性能高效、功能稳定的系统提供给用户</u>。

​	<u>ZooKeeper在实际生产环境中应用非常广泛，例如SOA的服务监控系统、Hadoop、Spark的分布式调度系统</u>。

### 10.1 ZooKeeper伪集群安装和配置

​	现在，我们开始使用三台机器来搭建一个ZooKeeper集群。在学习环境中，由于没有多余的服务器，这里就将三个ZooKeeper节点都安装到本地机器上，故称之为**伪集群模式**。

​	虽然伪集群模式只是便于开发、普通测试，但是不能用于生产环境。实际上，如果了解伪集群模式下的安装和配置，那么在生产环境下的配置也就大致差不多了。

​	首先是下载ZooKeeper。

> [Linux 下ZooKeeper安装](https://blog.csdn.net/she_lock/article/details/80435176)

#### 10.1.1 创建数据目录和日志目录

​	安装ZooKeeper之前，需要规划一下节点的个数，ZooKeepr节点数有以下要求：

​	（1）**ZooKeeper集群节点数必须是奇数**。

​	为什么呢？在ZooKeeper集群中，需要一个主节点，也称为Leader节点。主节点是集群通过选举的规则从所有节点中选举出来的。在选举的规则中很重要的一条是：要求可用节点数量 > 总结点数量 / 2。如果是偶数个节点，则可能会出现不满足这个规则的情况。

​	（2）**ZooKeeper集群至少是3个。**

​	ZooKeeper可以通过一个节点，正常启动和提供服务。但是，一个节点的Zookeeper服务不能叫作集群，其可靠性会大打折扣，仅仅作为学习使用尚可。在正常情况下，搭建ZooKeeper集群，至少需要3个节点。

​	这里作为学习案例，在本地机器上规划搭建一个具有3个节点的伪集群。

​	<u>安装集群的第一步是在安装目录下提前为每一个为节点创建好两个目录：日志目录和数据目录。</u>

​	具体来说，先创建日志目录。为3个节点中的每一个伪节点创建一个日志目录，分别为：log/zoo-1、log/zoo-2、log/zoo-3。

​	其次，创建数据目录。在安装目录下，为伪集群3个节点中的每一个伪节点创建一个数据目录，分别为：data/zoo-1、data/zoo-2、data/zoo-3。

> [关于zookeeper的节点配置的个数](https://www.cnblogs.com/smy-yaphet/p/10369310.html)

#### 10.1.2 创建myid文件

​	安装集群的第二步，为每一个节点创建一个id文件。

​	每一个节点需要有一个记录节点id的文本文件，文件名为myid。myid文件的特点如下：

+ myid文件的唯一作用是记录（伪）节点的编号。
+ myid文件是一个文本文件，文件名称为myid。
+ myid文件内容为一个数字，表示节点的编号。
+ 在myid文件中只能有一个数字，不能有其他的内容。
+ myid文件的存放位置，默认处于数据目录下。

​	下面分别为3个节点，创建3个myid文件：

（1）在第一个伪节点的数据目录zookeeper安装路径\data\zoo-1\文件夹下，创建一个myid文件，文件的内容为”1“，表示第一个节点的编号为1。

（2）在第二个伪节点的数据目录zookeeper安装路径\data\zoo-2\文件夹下，创建一个myid文件，文件的内容为”2“，表示第一个节点的编号为2。

（3）在第一个伪节点的数据目录zookeeper安装路径\data\zoo-3\文件夹下，创建一个myid文件，文件的内容为”3“，表示第一个节点的编号为3。

​	<u>ZooKeeper对id的值有何要求呢？首先，myid文件中id的值只能是一个数字，即一个节点的编号ID；其次，id的范围是1~255，表示集群最多的节点个数为255个</u>。

#### 10.1.3 创建和修改配置文件

​	安装集群的第三步，为每一个节点创建一个配置文件。不需要从零开始，在ZooKeeper的配置目录conf下，官方有一个配置文件的样本——zoo_sample.cfg。复制这个样本，修改其中的某些配置项即可。

​	下面分别为三个节点，创建3个".cfg"配置文件，具体的步骤为：

​	（1）将配置文件的样本zoo_sample.cfg文件复制3份，为每一个节点复制一份，分别命名为zoo-1.cfg、zoo-2.cfg、zoo-3.cfg，对应于3个节点。

​	（2）需要修改每一个节点的配置文件，将前面准备的日志目录和数据目录，配置到”.cfg“中的正确选项中。

```none
dataDir = zookeeper安装路径/data/zoo-1/
dataLogDir = zookeeper安装路径/log/zoo-1/
```

​	两个选项的介绍如下：

+ dataDir：数据目录选项，配置为前面准备的数据目录。myid文件处于此目录下。
+ dataLogDir：日志目录选项，配置为前面准备的日志目录。如果没有设置该参数，默认将使用和dataDir相同的设置。

​	（3）配置集群中的端口信息、节点信息、时间选项等。

​	首先是端口选项的配置，示例如下：

```none
clientPort = 2181
```

​	clientPort：表示客户端连接ZooKeeper集群中的节点的端口号。**在生产环境的集群中，不同的节点处于不同的机器，端口号一般都相同，这样便于记忆和使用**。<u>由于这里是伪集群模式，三个节点集中在一台机器上，因此3个端口号配置为不一样的。</u>

​	clientPort：一般设置为2181。而伪集群下不同的节点，clientPort是不能相同的，可以按照编号进行累加。

​	其次是节点信息的配置，示例如下：

```none
server.1=127.0.0.1:2888:3888
server.1=127.0.0.1:2889:3889
server.1=127.0.0.1:2890:3890
```

​	节点信息需要配置集群中的所有节点的（id）编号、IP地址和端口号。格式为：

```none
server.id=host:port:port
```

​	在".cfg"配置文件中，可以按照这样的格式进行配置。每一行都代表一个节点。**在ZooKeeper集群中，每个节点都需要感知到整个集群是由哪些节点组成，所以，每一个配置文件都需要配置全部的节点。**

​	总体来说，配置节点的时候，需要注意四点：
（1）不能有相同id的节点，需要确保每个节点的myid文件中的id值不同。

（2）每一行“server.id=host:port:port”中的id值。需要与所对应节点的数据目录下的myid文件中的id值保持一致。

（3）每一个配置文件都需要配置全部的节点信息。不仅仅是配置自己的那份，还需要配置所有节点的id、ip、端口。

（4）在每一行“server.id=host:port:port”中，需要配置两个端口。<u>前一个端口（如实例中的2888）用于节点之间的通信，后一个端口（如示例中的3888）用于选举主节点。</u>

​	**在伪集群的模式下，每一行记录中的端口号必须修改成不一样的，主要是避免端口冲突。在实际的分布式集群模式下，由于不同节点的ip不同，每一行记录中可以配置相同的端口。**

​	最后是时间相关选项的配置，示例如下：

```none
tickTime=4000
initLimit=10
syncLimit=5
```

​	**tickTime**：配置单元时间。单元时间是ZooKeeper的时间计算单元，其他的时间间隔都是使用tickTime的倍数来表示的。如果不配置，单元时间的默认值为3000，单位是毫秒(ms)。

​	**initLimit**：节点的初始化时间。该参数用于Follower（从节点）的启动，并完成与Leader（主节点）进行数据同步的时间。Follower节点在启动过程中，会与Leader节点建立来凝结并完成对数据的同步，从而确定自己的起始状态。Leader节点允许Follower节点在initLimit时间内完成这项工作。该参数默认值为10，表示是参数tickTime的值的10倍，必须配置且为正整数。

​	**syncLimit**：心跳最大延迟周期。该参数用于配置Leader节点和Follower节点之间进行心跳检测的最大延时时间。在ZK集群运行的过程中，Leader节点会通过心跳检测来确定Follower节点是否存活。如果Leader节点在syncLimit时间内无法获取到Follower的心跳检测响应，那么Leader节点就会认为该Follower节点已经脱离了和自己的同步。该参数默认值为5，表示是参数tickTime值的5倍。此参数必须配置且为正整数。

#### 10.1.4 配置文件示例

​	完成了伪集群的日志目录、数据目录、myid文件、".cfg"文件的准备后，伪集群的安装工作就基本完成了。

​	其中，".cfg"配置文件是最为关键的环节。下面给出三份配置文件实际的代码。

​	第一个节点，配置文件zoo-1.cfg的内容如下：

```none
tickTime=4000
initLimit=10
syncLimit=5
dataDir=zookeeper安装路径/data/zoo-1/
dataLogDir=zookeeper安装路径/log/zoo-1/

clientPort=2181
server.1=127.0.0.1:2888:3888
server.2=127.0.0.1:2889:3889
server.3=127.0.0.1:2890:3890
```

​	第二个节点，配置文件zoo-2.cfg的内容如下：

```none
tickTime=4000
initLimit=10
syncLimit=5
dataDir=zookeeper安装路径/data/zoo-2/
dataLogDir=zookeeper安装路径/log/zoo-2/

clientPort=2182
server.1=127.0.0.1:2888:3888
server.2=127.0.0.1:2889:3889
server.3=127.0.0.1:2890:3890
```

​	第三个节点，配置文件zoo-3.cfg的内容如下：

```none
tickTime=4000
initLimit=10
syncLimit=5
dataDir=zookeeper安装路径/data/zoo-3/
dataLogDir=zookeeper安装路径/log/zoo-3/

clientPort=2183
server.1=127.0.0.1:2888:3888
server.2=127.0.0.1:2889:3889
server.3=127.0.0.1:2890:3890
```

​	通过三个配置文件，我们可以看出，对于不同的节点，“.cfg”配置文件中的配置项的内容大部分相同。**这里需要强调一下，每个节点的<u>集群节点信息的配置</u>都是全量的；相对而言，每个节点的数据目录dataDir、日志目录dataLogDir和对外服务端口clientPort，则仅需配置自己的那份**。

#### 10.1.5 启动ZooKeeper伪集群

​	为了方便地启动每一个节点，我们需要为每一个节点制作一份启动命令，在Windows平台上，启动命令为一份“.cmd”文件。

​	在ZooKeepr的bin目录下，通过复制zkSevrer.cmd样本文件，为每一个伪节点创建一个启动的命令文件，分别为zkServer-1.cmd、zkServer-2.cmd、zkServer-3.cmd

​	修改复制后的".cmd"文件，主要为每一个节点增加配置文件选项（ZOOCFG）。

​	修改之后，第一个节点的启动命令文件zkServer-1.cmd中的代码如下：

```shell
setlocal
call "%~dp0zkEnv.cmd"

set ZOOCFG=zookeeper安装路径/conf/zoo-1.cfg

set ZOOMAIN=org.apache.ZooKeeper.server.quorum.QuorumPeerMain
echo on
call %JAVA% "-DzooKeeper.log.dir=%ZOO_LOG_DIR%"
"-DZooKeeper.root.logger=%ZOO_LOG4J_PROP%" -cp "%CLASSPATH%" %ZOOMAIN% "%ZOOCFG%" %*
endlocal
```

​	启动一个Windows的"命令提示符"窗口，进入到bin目录，并且启动zkServer-1.cmd，这个脚本中会启动第一个节点的Java服务进程：

```shell
cd bin
zkServer-1.cmd
```

​	<u>ZooKeeper集群需要有1/2以上的节点启动才能完成集群的启动，才能对外提供服务。所以，至少需要启动两个节点</u>。

​	启动另一个Windows的"命令提示符"窗口，进入到bin目录，并且启动zkServer-2.cmd，这个脚本中会启动第2个节点的Java服务进程：

```shell
cd bin
zkServer-2.cmd
```

​	由于这里没有使用后台服务启动的模式，因此这两个节点"命令提示符"窗口在服务期间，不能关闭。

​	启动之后，如何验证集群的启动是否成功呢？有两种方法。

​	方法一，可以通过执行jps命令，我们可以看到QuorumPeerMain进程的数量。

```shell
bin目录下 > jps
```

​	方法二，启动ZooKeeper客户端，运行并查看一下是否能连接集群。

​	如果最后显示出“CONNECTED”连接状态，则表示已经成功连接。当然，这个时候ZooKeeper已经启动成功了。

```shell
bin目录 > ./zkCli.cmd -server 127.0.0.1:2181
```

​	在连接成功后，可以输入ZooKeeper的客户端命令，操作“ZNode”树的节点。

​	在Windows下，ZooKeeper是通过.cmd命令文件中的批处理命令来运行的，默认没有提供Windows后台服务方案。为了避免每次关闭后，再一次启动ZooKeeper时还需要使用cmd带来的不便，可以通过prunsrv第三方工具来讲ZooKeeper做成Windows服务，将ZooKeeper变成后台服务来运行。

### 10.2 使用ZooKeeper进行分布式存储

​	下面介绍一下ZooKeeper存储模型，然后介绍如何使用客户端命令来操作ZooKeeper的存储模型。

#### 10.2.1 详解ZooKeeper存储模型

​	<u>ZooKeeper的存储模型非常简单，它和Linux的文件系统非常的类似。简单地说，ZooKeeper的存储模型是一棵以"/"为根节点的树。ZooKeeper的存储模型中的每一个节点，叫作ZNode（ZooKeeper Node）节点。所有的ZNode节点通过树形的目录结构，按照层次关系组织在一起，构成一棵ZNode树。</u>

​	每个ZNode节点都用一个以"/"(斜杠)分隔的完整路径来唯一标识，而且每个ZNode节点都有父节点（根节点除外）。例如："/foo/bar"就表示一个ZNode节点，它的父节点为"/foo"节点，它的祖父节点的路径为"/"。"/"节点是ZNode树的根节点，它没有父节点。

​	通过ZNode树，ZooKeeper提供了一个多层级的树形命名空间。**与文件的目录系统中的目录有所不同的是，这些ZNode节点可以保存二进制有效负载数据（Payload）。而文件系统目录树中的目录，只能存放路径信息，而不能存放负载数据。**

​	<u>ZooKeeper为了保证**高吞吐和低延迟**，整个树形的目录结构全部都放在内存中</u>。与硬盘和其他的外存设备相比，计算机的内存比较有限，使得ZooKeeper的目录结构不能用于存放大量的数据。ZooKeeper官方的要求是，**每个节点存放的有效负载数据（Payload）的上限仅为1MB**。

#### 10.2.2 zkCli客户端命令清单

​	用zkCli.cmd（zkCli.sh）连接上ZooKeeper服务后，用help命令可以列出ZooKeeper的所有命令。

| zk的客户端常用命令 | 功能简介             |
| ------------------ | -------------------- |
| create             | 创建ZNode路径节点    |
| ls                 | 查看路径下的所有节点 |
| get                | 获得节点上的值       |
| set                | 修改节点上的值       |
| delete             | 删除节点             |

​	例如，使用get命令查看ZNode树的根节点"/"。

```none
[zk: localhost: 2181(CONNECTED)6] ls -s /
[zookeeper]cZxid =0x0
ctime=Thu Jan 01 08:00:00 CST 1970
mZxid= 0x0
mtime = Thu Jan 01 08:00:00 CST 1970
pZxid = 0x0
cversion = -1 
dataVersion = 0 
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 0
numChildren = 1
```

​	所返回的节点信息主要有事务id、时间戳、版本号、数据长度、子节点的数量等。比较复杂的是事务id和版本号。

​	事务id记录着节点的状态，ZooKeeper状态的每一次改变都对应着一个递增的事务id（Transaction id），该id称为Zxid，<u>**它是全局有序的**</u>，<u>每次ZooKeeper的更新操作都会产生一个新的Zxid</u>。<u>Zxid不仅仅是一个唯一的事务id，它还具有递增性</u>。例如，有两个Zxid存在着zxid1<zxid2，那么说明Zxid1事务变化发生在Zxid2事务变化之前。

​	一个ZNode的创建或者更新都会产生一个新的Zxid值，所以在节点信息中保存了3个Zxid事务id值，分别是：

+ cZxid：ZNode节点创建时的事务id（Transaction id）。
+ mZxid：ZNode节点修改时的事务id，与子节点无关。
+ pZxid：ZNode节点的子节点的最后一次创建或者修改的时间，与孙子节点无关。

​	所返回的节点信息，包含的时间戳有两个：

+ ctime：ZNode节点创建时的时间戳。
+ mtime：ZNode节点最新一次更新时的时间戳。

​	所返回的节点信息，包括的版本号有三个：

+ dataversion：数据版本号。
+ cversion：子节点版本号。
+ aclversion：节点的ACL权限修改版本号。

​	对节点的每次操作都会使节点的相应版本号增加。

​	对于ZNode节点信息的主要属性，如下表。

| 属性名称    | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| cZxid       | 创建节点时的Zxid事务id                                       |
| ctime       | 创建节点时的时间戳                                           |
| mZxid       | 最后修改节点时的事务id                                       |
| mtime       | 最后修改节点时的时间戳                                       |
| pZxid       | 表示该节点的子节点最后一次修改的事务id。添加子节点或删除子节点就会影响pZxid的值，但是修改子节点的数据内容则不影响该id |
| cversion    | 子节点版本号，子节点每次修改，该版本号就加1                  |
| dataversion | 数据版本号，数据每次修改，该版本号就加1                      |
| aclversion  | 权限版本号，权限每次修改，该版本号就加1                      |
| dataLength  | 该节点的数据长度                                             |
| numChildren | 该节点拥有子节点的数量                                       |

### 10.3 ZooKeeper应用开发的实践

​	ZooKeeper应用的开发主要通过Java客户端API去连接和操作ZooKeeper集群。可供选择的Java客户端API有：

+ ZooKeeper官方的Java客户端API
+ 第三方的Java客户端的API

​	ZooKeeper官方的客户端API提供了基本的操作。例如，创建会话、创建节点、读取节点、更新数据、删除节点和检查节点是否存在等。不过，对于实际开发来说，ZooKeeper官方API有一些不足之处，具体如下：

+ **ZooKeeper的Watcher监测是一次性的**，每次触发之后都需要重新进行注册。
+ **会话超时之后没有实现重连机制**。
+ 异常处理烦琐，ZooKeeper提供了很多异常，对于开发人员来说可能根本不知道应该如何处理这些抛出的异常。

+ 仅提供了简单的byte[]数组类型的接口，没有提供Java POJO级别的序列化数据处理接口。
+ 创建节点时如果抛出异常，需要自行检查节点是否存在。
+ <u>无法实现级联删除</u>。

​	总之，ZooKeeper官方API功能比较简单，在实际开发过程中比较笨重，一般不推荐使用。可供选择的Java客户端API之二，即第三方开源客户端API，主要有ZkClient和Curator。

#### 10.3.1 ZkClient开源客户端介绍

​	ZkClient是一个开源客户端，在ZooKeeper原生API接口的基础上进行了包装，更便于开发人员使用。ZkClient客户端在一些著名的互联网开源项目中得到了应用，例如，阿里的分布式Dubbo框架对它进行了无缝集成。

​	ZkClient解决了ZooKeeper原生API接口的很多问题。例如，ZkClient提供了更加简洁的API，实现了会话超时重连、反复注册Watcher等问题。虽然ZkClient对原生API进行了封装，但也有它自身的不足之处，具体如下：

+ ZkClient社区不活跃，文档不够完善，几乎没有参考文档。
+ 异常处理简化（抛出RuntimeException）。
+ 重试机制比较难用。
+ 没有提供各种使用场景的参考实现。

#### 10.3.2 Curator开源客户端介绍

​	Cutator是Netflix公司开源的一套ZooKeeper客户端框架，和ZkClient一样它解决了非常底层的细节开发工作，包括连接、重连、反复注册Watcher的问题以及NodeExistsException异常等。

​	Curator是Apache基金会的顶级项目之一，Curator具有更加完善的文档，另外还提供了一套易用性和可读性更强的Fluent风格的客户端API框架。

​	Curator还为ZooKeeper客户端框架提供了一些比较普通的、开箱即用的、分布式开发用的解决方案，例如Recipe、共享锁服务、Master选项机制和分布式计算器等，帮助开发者避免了"重复造轮子"的无效开发工作。

​	另外，Curator还提供了一套非常优雅的链式调用API，与ZkClient客户端API相比，Curator的API优雅太多了，以创建ZNode节点为例，让大家实际对比一下。

​	使用ZkClient客户端创建ZNode节点的代码为：

```java
ZkClient client = new ZkClient("192.168.1.105:2181",10000,10000,new SerializableSerializer());
// 根节点路径
String PATH = "/test";
// 判断是否存在
boolean rootExists = zkClient.exists(PATH);
// 如果存在，获取地址列表
if(!rootExists){
    zkClient.createPersistent(PATH);
}
String zkPath = "/test/node-1";
boolean serviceExists = zkClient.exist(zkPath);
if(!serviceExists){
    zkClient.createPersistent(zkPath);
}
```

​	使用Curator客户端创建ZNode节点的代码如下：

```java
CuratorFramework client = CuratorFrameworkFactory.newClient(connectionString, retryPolicy);
String zkPath = "/test/node-1";
client.create().writeMode(mode).forPath(zkPath);
```

​	总之，尽管Curator不是官方的客户端，但是由于Curator客户端的确非常优秀，就连ZooKeeper一书的作者，鼎鼎大名的Patrixck Hunt都对Curator给予了高度的评价，他的评语是："Guava is to Java that Curator to ZooKeeper"。

​	在实际的开发场景中，使用Curator客户但就足以应对日常的ZooKeeper集群操作的要求。对于ZooKeeper的客户端，我们这里只学习和研究Curator的使用，疯狂创客圈社群的高并发项目——“IM实战项目”，最终也是通过Curator客户端来操作ZooKeeper集群的。

#### 10.3.3 Curator开发的环境准备

​	打开Curator的官网，我们可以看到，Curator包括了以下几个包：

+ curator-framework是对ZooKeeper的底层API的一些封装。
+ curator-client提供了一些客户端的操作，例如重试策略等。
+ curator-recipes封装了一些高级特性，如：Cache事件监听、选举、分布式锁、分布式计数器、分布式Barrier等。

​	以上的三个包，在使用之前，首先要在Maven的pom文件中加上包的Maven依赖。这里使用Curator的版本为4.0.0，与之对应ZooKeeper的版本为3.4.x。

​	如果Curator与ZooKeeper的版本不是相互匹配的，就会有兼容性问题，就很可能导致节点操作的失败。如何确保Curator与ZooKeeper的具体版本是匹配的呢？可以去[Curator的官网](https://curator.apache.org/zk-compatibility.html)查看。

#### 10.3.4 Curator客户端实例的创建

​	在使用curator-framework包操作ZooKeeper前，首先要创建一个客户端实例——这是一个CuratorFramework类型的对象，有两种方法：

+ 使用工厂类CuratorFrameworkFactory的静态newClient()方法。
+ 使用工厂列CuratorFrameworkFactory的静态builder构造者方法。

​	下面实现一个通用的类，分别使用以上两种方法来创建一下Curator客户端实例，代码如下：

```java
public class ClientFactory {

    /**
     * @param connectionString zk的连接地址
     * @return CuratorFramework 实例
     */
    public static CuratorFramework createSimple(String connectionString) {
        // 重试策略:第一次重试等待1s，第二次重试等待2s，第三次重试等待4s
        // 第一个参数：等待时间的基础单位，单位为毫秒
        // 第二个参数：最大重试次数
        ExponentialBackoffRetry retryPolicy =
                new ExponentialBackoffRetry(1000, 3);

        // 获取 CuratorFramework 实例的最简单的方式
        // 第一个参数：zk的连接地址
        // 第二个参数：重试策略
        return CuratorFrameworkFactory.newClient(connectionString, retryPolicy);
    }

    /**
     * @param connectionString    zk的连接地址
     * @param retryPolicy         重试策略
     * @param connectionTimeoutMs 连接
     * @param sessionTimeoutMs
     * @return CuratorFramework 实例
     */
    public static CuratorFramework createWithOptions(
            String connectionString, RetryPolicy retryPolicy,
            int connectionTimeoutMs, int sessionTimeoutMs) {

        // builder 模式创建 CuratorFramework 实例
        return CuratorFrameworkFactory.builder()
                .connectString(connectionString)
                .retryPolicy(retryPolicy)
                .connectionTimeoutMs(connectionTimeoutMs)
                .sessionTimeoutMs(sessionTimeoutMs)
                // 其他的创建选项
                .build();
    }
}
```

​	这里用到两种创建CuratorFramework客户端实例的方式，前一个是通过newClient函数去创建，相当于是一个简化版本，只需要设置ZK集群的连接地址和重试策略。

​	后一个是通过CuratorFrameworkFactory.builder()函数去创建，相当于是一个复杂的版本，可以设置连接超时connectionTimeoutMs、会话超时sessionTimeoutMs等其他与会话创建相关的选项。

​	上面的示例程序将两种创建客户端的方式封装成了一个通用的ClientFactory连接工具类，大家可以直接使用。

#### 10.3.5 通过Curator创建ZNode节点

​	通过Curator框架创建ZNode节点，可使用create()方法。create()方法不需要传入ZNode的节点路径，所以并不会立即创建节点，仅仅返回一个CreateBuilder构造者实例。

​	通过该CreateBuilder构造者实例，可以设置创建节点时的一些行为参数，最后再通过构造者实例的forPath(String znodePath，byte[] payload)方法来完成真正的节点创建。

​	总之，一般使用链式调用来完成节点的创建。在链式调用的最后，需要使用forPath带上需要创建的节点路径，具体的代码如下：

```java
/**
* 创建节点
*/
public void createNode(String zkPath, String data) {
    try {
        // 创建一个 ZNode 节点
        // 节点的数据为 payload
        byte[] payload = "to set content".getBytes("UTF-8");
        if (data != null) {
            payload = data.getBytes("UTF-8");
        }
        client.create()
            .creatingParentsIfNeeded()
            .withMode(CreateMode.PERSISTENT)
            .forPath(zkPath, payload);

    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        CloseableUtils.closeQuietly(client);
    }
}
```

​	在上面的代码中，在链式调用的forPath创建节点之前，通过该CreateBuilder构造者实例的withMode()方法，设置了节点的类型为CreateMode.PERSISTENT类型，表示节点的类型为持久化节点。

​	ZooKeeper节点有4种类型：

1. PERSISTENT	持久化节点

2. PERSISTENT_SEQUENTIAL	持久化顺序节点

3. EPHEMERAL	临时节点

4. EPHEMERAL_SEQUENTIAL	临时顺序节点

这4种节点类型的定义和联系具体如下：

1. 持久化节点（PERSISTENT）

   ​	所谓持久节点是指在节点创建后就一直存在，直到有删除操作来主动清除这个节点。持久化节点的生命周期是永久有效的，不会因为创建该节点的客户端会话失效而消失。

2. 持久化顺序节点（PERSISTENT_SEQUENTIAL）

   ​	这类节点的生命周期和持久节点是一致的。额外的特性是，在ZooKeeper中，每个父节点的第一级子节点维护一份顺序编码，其记录每个子节点创建的先后顺序。<u>如果在创建子节点的时候，可以设置这个属性，那么在创建节点过程中，ZooKeeper会自动为给定节点名加上一个表示顺序的数字后缀作为新的节点名</u>。<u>这个顺序后缀数字上限是整形的最大值。</u>

   ​	例如，在创建节点时只需要传入节点"/test\_"，ZooKeeper自动会在"test\_"后面补充数字顺序。

3. 临时节点（EPHEMERAL）

   ​	**和持久节点不同的是，临时节点的生命周期和客户端会话绑定**。也就是说，<u>如果客户端**会话失效**了，那么这个节点就会自动被清除掉</u>。注意，这里提到的是**会话失效，而非连接断开**。<u>还要注意一件事，就是当客户端会话失效后，所产生的节点也不是一下子就消失了，也要过一段时间，大概是10秒左右</u>。大家可以试一下，本机操作产生节点，在服务器端用命令来查看当前的节点数目，我们会发现有些客户端的状态已经是stop(中止)，但是产生的节点还在。

   ​	**另外，在临时节点下面不能创建子节点**。

4. 临时顺序节点（EPHEMERAL_SEQUENTIAL）

   ​	此节点是属于临时节点，不过带有顺序编号，客户端会话结束，所产生的节点就会消失。

#### 10.3.6 在Curator中读取节点

​	在Curator框架中，与节点读取的有关的方法主要有三个：

（1）首先是判断节点是否存在，调用checkExists方法。

（2）其次是获取节点的数据，调用getData方法。

（3）最后是获取子节点列表，调用getChildren方法。

​	演示代码如下：

```java
/**
 * 读取节点
 */
@Test
public void readNode() {
    //创建客户端
    CuratorFramework client = ClientFactory.createSimple(ZK_ADDRESS);
    try {
        //启动客户端实例,连接服务器
        client.start();

        String zkPath = "/test/CRUD/remoteNode-1";


        Stat stat = client.checkExists().forPath(zkPath);
        if (null != stat) {
            //读取节点的数据
            byte[] payload = client.getData().forPath(zkPath);
            String data = new String(payload, "UTF-8");
            log.info("read data:", data);

            String parentPath = "/test";
            List<String> children = client.getChildren().forPath(parentPath);

            for (String child : children) {
                log.info("child:", child);
            }
        }
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        CloseableUtils.closeQuietly(client);
    }
}
```

​	无论是checkExists、getData还是getChildren方法，都有一个共同的特点：

+ <u>返回构造者实例，不会立即执行</u>。
+ 通过链式调用，在最末端调用forPath(String znodePath)方法执行实际的操作。

#### 10.3.7 在Curator中更新节点

​	**节点的更新分为同步更新与异步更新**。

​	同步更新就是更新线程是阻塞的，一直阻塞到更新操作执行完成为止。

​	调用setData()方法进行同步更新，代码如下：

```java
/**
 * 更新节点
 */
@Test
public void updateNode() {
    //创建客户端
    CuratorFramework client = ClientFactory.createSimple(ZK_ADDRESS);
    try {
        //启动客户端实例,连接服务器
        client.start();


        String data = "hello world";
        byte[] payload = data.getBytes("UTF-8");
        String zkPath = "/test/remoteNode-1";
        client.setData()
            .forPath(zkPath, payload);


    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        CloseableUtils.closeQuietly(client);
    }
}
```

​	在上面的代码中，通过调用setData()方法返回一个SetDataBuilder构造者实例，执行该实例的forPath(zkPath，payload)方法即可完成同步更新操作。

​	如果是异步更新呢？

​	其实很简单，<u>通过SetDataBuilder构造者实例的inBackground(AsyncCallback callback)方法，设置一个AsyncCallback回调实例。简简单单的一个函数，就将更新数据的行为从同步执行变成了异步执行。异步执行完成之后，SetDataBuilder构造者实例会再执行AsyncCallback实例的processResult(...)方法中的回调逻辑，即可完成更新后的其他操作</u>。

​	异步更新的代码如下：

```java
/**
 * 更新节点 - 异步模式
 */
@Test
public void updateNodeAsync() {
    //创建客户端
    CuratorFramework client = ClientFactory.createSimple(ZK_ADDRESS);
    try {

        //更新完成监听器
        AsyncCallback.StringCallback callback = new AsyncCallback.StringCallback() {

            @Override
            public void processResult(int i, String s, Object o, String s1) {
                System.out.println(
                    "i = " + i + " | " +
                    "s = " + s + " | " +
                    "o = " + o + " | " +
                    "s1 = " + s1
                );
            }
        };
        //启动客户端实例,连接服务器
        client.start();

        String data = "hello ,every body! ";
        byte[] payload = data.getBytes("UTF-8");
        String zkPath = "/test/CRUD/remoteNode-1";
        client.setData()
            .inBackground(callback)
            .forPath(zkPath, payload);

        Thread.sleep(10000);
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        CloseableUtils.closeQuietly(client);
    }
}
```

#### 10.3.8 在Curator中删除节点

​	删除节点非常简单，只需调用delete方法，示例代码如下：

```java
/**
 * 删除节点
 */
@Test
public void deleteNode() {
    //创建客户端
    CuratorFramework client = ClientFactory.createSimple(ZK_ADDRESS);
    try {
        //启动客户端实例,连接服务器
        client.start();

        //删除节点
        String zkPath = "/test/CRUD/remoteNode-1";
        client.delete().forPath(zkPath);


        //删除后查看结果
        String parentPath = "/test";
        List<String> children = client.getChildren().forPath(parentPath);

        for (String child : children) {
            log.info("child:", child);
        }

    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        CloseableUtils.closeQuietly(client);
    }
}
```

​	在上面的代码中，通过调用delete()方法返回一个执行删除操作的DeleteBuilder构造者实例，执行该实例的forPath(zkPath，payload)方法，即可完成同步删除操作。

​	删除和更新操作一样，也可以异步进行。如何异步删除呢？思路是一样的：异步删除同样需要用到DeleteBuilder构造者实例的inBackground(AsyncCallback asyncCallback) 方法去设置回调，实际操作很简单，这里不再累述。

​	至此，Curator的CRUD基本操作就介绍完了。下面介绍基于Curator的基本操作，完成一些基础的分布式应用。注：CRUD是指Create(创建)，Retrieve(查询)，Update(更新)和Delete(删除)。

> [zookeeper 都有哪些使用场景？](https://www.jianshu.com/p/baf966931c32)

### 10.4 分布式命名服务的实践

​	命名服务是为系统中的资源提供标识能力。ZooKeeper的命名服务主要是利用ZooKeeper节点的树形分层结构和子节点的顺序维护能力，来为分布式系统中的资源命名。

​	哪些应用场景需要用到**分布式命名服务**呢？典型的有：

1. 分布式API目录

   ​	为分布式系统中各种API接口服务的名称、链接地址，提供类似JNDI（Java命名和目录接口）中的文件系统的功能。借助于ZooKeeper的树形分层结构就能提供分布式的API调用功能。

   ​	<u>著名的Dubbo分布式框架就是应用了ZooKeeper的分布式JNDI功能</u>。

   ​	在Dubbo中，使用ZooKeeper维护的全局服务接口API的地址列表。大致的思路为：

   + 服务提供者（Service Provider）在启动的时候，向ZooKeeper上的指定节点/dubbo/${serviceName}/providers写入自己的API地址，这个操作就相当于服务的公开。
   + 服务消费者（Consumer）启动的时候，订阅节点/dubbo/(serviceName)/provider下的服务提供者的URL地址，获得所有服务提供者的API。

2. 分布式的ID生成器

   ​	在分布式系统中，为每一个数据数据资源提供唯一性的ID标识功能。在单体服务环境下，通常说，可以利用数据库的主键自增功能，唯一标识一个数据资源。<u>但是，在大量服务器集群的场景下，依赖单体服务的数据库主键自增生成唯一ID的方式，则没有办法满足**高并发和高负载**的需求</u>。

   ​	这时，就需要分布式的ID生成器，保障分布式场景下的ID唯一性。

3. 分布式节点的命名

   ​	一个分布式系统通常会由很多的节点组成，节点的数量不是固定的，而是不断动态变化的。比如说，当业务不断膨胀和流量洪峰到来时，大量的节点可能会动态加入到集群中。而一旦流量洪峰过去了，就需要下线大量的节点。再比如说，由于机器或者网络的原因，一些节点会主动离开集群。

   ​	如何为大量的动态节点命名呢？一种简单的方法就是可以通过配置文件，手动为每一个节点命名。但是，<u>如果节点数据量太大，或者说变动频繁，手动命名则是不现实的，这就需要用到分布式系节点的命名服务</u>。

   ​	疯狂创客圈的高并发项目——"IM实战项目"，也使用分布式命名服务为每一个IM节点动态命名。

上面列举了三个分布式的命名服务场景，实际上，需要用到分布式资源标识功能的场景远远不止这些，这里只是抛砖引玉。

#### 10.4.1 ID生成器

​	在分布式系统中，分布式ID生成器的使用场景非常之多：

+ 大量的数据记录，需要分布式ID。
+ 大量的系统消息，需要分布式ID。
+ 大量的请求日志，如RESTful的操作记录，需要唯一标识，以便进行后续的用户行为分析和调用链路分析。
+ 分布式节点的命名服务，往往也需要分布式ID。

​	诸如此类......

​	可以肯定的是，传统的数据库自增主键或者单体的自增主键，已经不能满足需求。在分布式系统环境中，迫切需要一种全新的唯一ID系统，这种系统需要满足以下需求：

（1）全局唯一：不能出现重复ID。

（2）高可用：ID生成系统是基础系统，被许多关键系统调用，一旦宕机，就会造成严重影响。

​	有哪些分布式的ID生成器方案呢？简单的梳理下，大致如下：

+ Java的UUID。
+ 分布式缓存Redis生成ID：利用Redis的原子操作INCR和INCRBY，生成全局唯一的ID。
+ Twitter的SnowFlake算法。
+ ZooKeeper生成ID：利用ZooKeeper的顺序节点，生成全局唯一的ID。
+ MongoDb的ObjectId：MongoDB是一个分布式的非结构化NoSQL数据库，每插入一条记录会自动生成全局唯一的一个"_id"字段值，它是一个12字节的字符串，可以作为分布式系统中全局唯一的ID。

​	以上方法有哪些利弊呢？首先，分析一下Java语言中的UUID方案。UUDI是Universally Unique Indentifier的缩写，它是在一定的范围内（从特定的名字空间到全球）唯一的机器生成的标识符，所以，UUID在其他语言中也被叫作GUID。

​	在Java中，生成UUID的代码很简单，代码如下：

```java
String uuid = UUID.randomUUID().toString();
```

​	UUID是经由一定的算法机器生成的，为了保证UUID的唯一性，规范定义了包括网卡MAC地址、时间戳、名字空间（Namespace）、随机或伪随机数、时序等元素，以及从这些元素生成的UUID的算法。UUID只能由计算机生成。

​	一个UUID是16字节长的数字，一共128位。转成字符串之后，它会变成一个36字节的字符串，例如：3F2504E0-4F89-11D3-9A0C-0305E82C3301。使用的时候，可以把中间的4个连字符去掉，剩下32字节的字符串。

​	UUID的优点是本地生成ID，不需要进行远程调用，时延低，性能高。

​	UUID的缺点是UUID过长，16字节共128位，通常以36字节长的字符串来表示，在很多应用场景不适用，例如，**由于UUID没有排序，无法保证趋势递增，因此用于数据库索引字段的效率就很低，添加记录存储入库时性能差**。

​	所以，对于高并发和大数据量的系统，不建议使用UUID。

#### 10.4.2 ZooKeeper分布式ID生成器的实践案例

大家知道，在ZooKeeper节点的四种类型中，其中有以下两种具备**自动编号**的能力：

+ PERSISTENT_SEQUENTIAL	持久化顺序节点。
+ EPHEMERAL_SEQUENTIAL	临时顺序节点

​	**ZooKeeper的每一个节点都会为它的第一级子节点维护一份顺序编号，会记录每个子节点创建的先后顺序，这个顺序编号是分布式同步的，也是全局唯一的**。

​	在创建子节点时，如果设置为上面的类型，ZooKeeper会自动为创建后的节点路径在末尾加上一个数字，用来表示顺序。<u>这个顺序值的最大上限就是整形的最大值</u>。

​	例如，在创建节点的时候只需要传入节点"/test\_"，ZooKeeper自动会在"test\_后面补充数字顺序，例如"/test_0000000010"。

​	通过创建ZooKeeper的临时顺序节点的方法，生成全局唯一的ID，演示代码如下：

```java
public class IDMaker {
    private static final String ZK_ADDRESS = "127.0.0.1:2181";
    //Zk客户端
    CuratorFramework client = null;

    public void init() {

        //创建客户端
        client = ClientFactory.createSimple(ZK_ADDRESS);

        //启动客户端实例,连接服务器
        client.start();

    }

    public void destroy() {

        if (null != client) {
            client.close();
        }
    }

    private String createSeqNode(String pathPefix) {
        try {
            // 创建一个 ZNode 顺序节点

            String destPath = client.create()
                .creatingParentsIfNeeded()
                .withMode(CreateMode.EPHEMERAL_SEQUENTIAL)
                //避免zookeeper的顺序节点暴增，需要删除创建的持久化顺序节点
                //                    .withMode(CreateMode.EPHEMERAL_SEQUENTIAL)
                .forPath(pathPefix);
            return destPath;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }

    public String makeId(String nodeName) {
        String str = createSeqNode(nodeName);
        if (null == str) {
            return null;
        }
        int index = str.lastIndexOf(nodeName);
        if (index >= 0) {
            index += nodeName.length();
            return index <= str.length() ? str.substring(index) : "";
        }
        return str;

    }
}
```

​	节点创建完成之后，会返回节点的完整路径，生成的放置在路径的末尾，一般为10位数字字符。可以通过截取路径末尾的数字作为新生成的ID。

​	基于自定义的IDMaker，编写单元测试用例来生成ID，代码如下：

```java
public class IDMakerTester {

    @Test
    public void testMakeId() {
        IDMaker idMaker = new IDMaker();
        idMaker.init();
        String nodeName = "/test/IDMaker/ID-";

        for (int i = 0; i < 10; i++) {
            String id = idMaker.makeId(nodeName);
            log.info("第" + i + "个创建的id为:" + id);
        }
        idMaker.destroy();

    }
}
```

​	下面是这个测试用例程序的部分运行结果：

```none
第0个创建的id为：0000000010
第1个创建的id为：0000000011
//...省略其他的输出
```

#### 10.4.3 集群节点的命名服务之实践案例

​	节点的命名主要是为节点进行唯一编号。主要的诉求是不同节点的编号是绝对不能重复的。一旦编号重复，就会有不同的节点发生碰撞而导致集群异常。

​	前面讲到在分布式集群中，可能需要部署大量的机器节点。在节点少的时候，节点的命名、可以手工完成。然而，在节点数量大的场景下，手工命名的维护成功高，还需要考虑到自动部署、运维等等问题，因此手工去命名不现实。总之，节点的命名最好由系统自动维护。

​	由以下两个方案可用于生成集群节点的编号：

（1）使用数据库的自增ID特性，用数据库表存储机器的MAC地址或者IP来维护。

（2）使用ZooKeeper持久顺序节点的顺序特性来维护节点的NodeId编号。

下面为大家介绍的是第二种。

在第二种方案中，集群节点命名服务的基本流程是：

+ 启动节点服务，连接ZooKeeper，检查命名服务根节点是否存在，如果不存在，就创建系统的根节点。
+ 在根节点下创建一个临时顺序ZNode节点，取回ZNode的编号把它作为分布式系统中节点的NODEID。
+ 如果临时节点太多，可以根据需要删除临时顺序ZNode节点。

集群节点命名服务的主要实现代码如下：

```java
public class PeerNode {

    //ZooKeeper客户端
    private CuratorFramework client = null;
    private String pathRegistered = null;
    private Node node = null;

    private static PeerNode singleInstance = null;
    //唯一实例模式
    public static PeerNode getInstance(){
        if(null == singleInstance){
            singleInstance = new PeerNode();
            singleInstance.client = ZKclient.instance.getClient();
            singleInstance.init();
        }
        return singleInstance;
    }

    private PeerNode(){}

    //初始化，在ZooKeeper中创建当前的分布式节点
    public void init(){
        // 使用标准的前缀，创建父节点，父节点是持久的
        createParentIfNeeded(ServerUtils.MANAGE_PATH);

        //创建一个ZNode节点
        try{
            pathRegistered = client.create().creatingParentContainersIfNeeded()
            // 创建一个非持久的临时节点
            .withMode(CreateMode.EPHEMERAL_SEQUENTIAL)
            .forPath(ServerUtils.pathPerfix);
        } catch (Exception e){
            e.printStackTrace();
        }
    }

    //获取节点的编号
    public Long getId(){
        String sid = null;
        if(null == pathRegistered){
            throw new RuntimeException("节点注册失败");
        }
        int index = pathRegistered.lastIndexOf(ServerUtils.pathPrefix);
        if(index >= 0){
            index += ServerUtils.pathPrefix.length();
            sid = index <= pathRegistered.length() ? pathRegistered.substring(index):null;
        }
        if(null == sid){
            throw new RuntimeException("分布式节点错误");
        }
        return Long.parseLong(sid);
    }
}
```

#### 10.4.4 使用ZK实现SnowFlakeID算法的实践

​	Twitter（推特）的SnowFlake算法是一种主名的分布式服务器用户ID生成算法。SnowFlake算法所生成的ID是一个64bit的长整型数字。<u>这个64bit被划分成四个部分，其中后面三个部分分别表示时间戳、工作机器ID、序列号</u>。

> [雪花算法](https://www.cnblogs.com/Hollson/p/9116218.html)

​	SnowFlake的四个部分，具体介绍如下：

（1）第一位 占用1bit，其值始终是0，没有实际作用。

（2）时间戳 占用41bit，精确好毫秒，总共可以容纳越约69年的时间。

（3）工作机器id 占用10bit，最多可以容纳1024个节点

（4）序列号 占用12bit，最多可以累加到4095。这个值在同一毫秒同一节点上从0开始不断累加。

​	总体来说，在工作节点达到1024顶配的场景下，SnowFlake算法在同一毫秒内最多可以生成多少个全局唯一ID呢？同一毫秒的ID数量ID数量大致为：1024*4096，总计400多万个ID，也就是说，在绝大多数并发场景下都是够用的。

​	在SnowFlake算法中，第三个部分是工作机器ID，可以结合上一节的命名方法，并通过ZooKeeper管理world，免了手动频繁修改集群节点去配置机器ID的麻烦。

​	上面的bit数分配只是一个官方的推荐，是可以微调的。比方说，如果1024的节点数不够，可以增加3个bit，扩大到8192个节点；再比方说，如果每毫秒生成4096个ID比较多，可以从12bit减小到使用10bit，则每秒生成1024和ID。这样，单个节点1秒可以生成1024*1000，也就是100多万个ID，数量也是巨大的；剩下的位置为剩余时间，还剩下40bit的时间戳，比原来少1位，则可以持续32年。

```java
public class SnowflakeIdGenerator {

    /**
     * 单例
     */
    public static SnowflakeIdGenerator instance =
        new SnowflakeIdGenerator();


    /**
     * 初始化单例
     *
     * @param workerId 节点Id,最大8091
     * @return the 单例
     */
    public synchronized void init(long workerId) {
        if (workerId > MAX_WORKER_ID) {
            // zk分配的workerId过大
            throw new IllegalArgumentException("woker Id wrong: " + workerId);
        }
        instance.workerId = workerId;
    }

    private SnowflakeIdGenerator() {

    }


    /**
     * 开始使用该算法的时间为: 2017-01-01 00:00:00
     */
    private static final long START_TIME = 1483200000000L;

    /**
     * worker id 的bit数，最多支持8192个节点
     */
    private static final int WORKER_ID_BITS = 13;

    /**
     * 序列号，支持单节点最高每毫秒的最大ID数1024
     */
    private final static int SEQUENCE_BITS = 10;

    /**
     * 最大的 worker id ，8091
     * -1 的补码（二进制全1）右移13位, 然后取反
     */
    private final static long MAX_WORKER_ID = ~(-1L << WORKER_ID_BITS);

    /**
     * 最大的序列号，1023
     * -1 的补码（二进制全1）右移10位, 然后取反
     */
    private final static long MAX_SEQUENCE = ~(-1L << SEQUENCE_BITS);

    /**
     * worker 节点编号的移位
     */
    private final static long APP_HOST_ID_SHIFT = SEQUENCE_BITS;

    /**
     * 时间戳的移位
     */
    private final static long TIMESTAMP_LEFT_SHIFT = WORKER_ID_BITS + APP_HOST_ID_SHIFT;

    /**
     * 该项目的worker 节点 id
     */
    private long workerId;

    /**
     * 上次生成ID的时间戳
     */
    private long lastTimestamp = -1L;

    /**
     * 当前毫秒生成的序列
     */
    private long sequence = 0L;

    /**
     * Next id long.
     *
     * @return the nextId
     */
    public Long nextId() {
        return generateId();
    }

    /**
     * 生成唯一id的具体实现
     */
    private synchronized long generateId() {
        long current = System.currentTimeMillis();

        if (current < lastTimestamp) {
            // 如果当前时间小于上一次ID生成的时间戳，说明系统时钟回退过，出现问题返回-1
            return -1;
        }

        if (current == lastTimestamp) {
            // 如果当前生成id的时间还是上次的时间，那么对sequence序列号进行+1
            sequence = (sequence + 1) & MAX_SEQUENCE;

            if (sequence == MAX_SEQUENCE) {
                // 当前毫秒生成的序列数已经大于最大值，那么阻塞到下一个毫秒再获取新的时间戳
                current = this.nextMs(lastTimestamp);
            }
        } else {
            // 当前的时间戳已经是下一个毫秒
            sequence = 0L;
        }

        // 更新上次生成id的时间戳
        lastTimestamp = current;

        // 进行移位操作生成int64的唯一ID

        //时间戳右移动23位
        long time = (current - START_TIME) << TIMESTAMP_LEFT_SHIFT;

        //workerId 右移动10位
        long workerId = this.workerId << APP_HOST_ID_SHIFT;

        return time | workerId | sequence;
    }

    /**
     * 阻塞到下一个毫秒
     */
    private long nextMs(long timeStamp) {
        long current = System.currentTimeMillis();
        while (current <= timeStamp) {
            current = System.currentTimeMillis();
        }
        return current;
    }


}
```

​	这里的二进制位移算法以及二进制"按位或"的算法都比较简单，如果不懂，可以去查看Java的基础书籍。

​	上面的代码是一个相对比较简单的SnowFlake实现版本，现在对其中的关键算法解释如下：

+ 在单节点上获得下一个ID，使用Synchroized控制并发，没有使用CAS（Compare And Swap，比较并交换）的方式，是因为CAS不适合并发量非常高的场景；
+ 如果在一台机器上当前毫秒的序列已经增长到最大值1023，则使用while循环等待直到下一毫秒；
+ 如果当前时间小于记录的上一个毫秒值，则说明这台机器的时间回拨了，于是阻塞，一直等到下一毫秒。

编写一个测试用例，测试一下SnowflakeIdGenerator，代码如下：

```java
@Slf4j
public class SnowflakeIdTest {

    /**
     * The entry point of application.
     *
     * @param args the input arguments
     * @throws InterruptedException the interrupted exception
     */
    public static void main(String[] args) throws InterruptedException {
        SnowflakeIdGenerator.instance.init(SnowflakeIdWorker.instance.getId());
        ExecutorService es = Executors.newFixedThreadPool(10);
        final HashSet idSet = new HashSet();
        Collections.synchronizedCollection(idSet);
        long start = System.currentTimeMillis();
        log.info(" start generate id *");
        for (int i = 0; i < 10; i++)
            es.execute(() -> {
                for (long j = 0; j < 5000000; j++) {
                    long id = SnowflakeIdGenerator.instance.nextId();
                    synchronized (idSet) {
                        idSet.add(id);
                    }
                }
            });
        es.shutdown();
        es.awaitTermination(10, TimeUnit.SECONDS);
        long end = System.currentTimeMillis();
        log.info(" end generate id ");
        log.info("* cost " + (end - start) + " ms!");
    }
}
```

​	在测试用例中，用到了SnowflakeIdWorker节点的命令服务，并通过它取得了节点workerId。

SnowFlake算法的优点：

+ 生成ID时不依赖于数据库，完全在内存生成，高性能和高可用性。
+ 容量大，每秒可生成几百万个ID。
+ **ID呈趋势递增，后续插入数据库索引树时，性能较高**。

SnowFlake算法的缺点：

+ 依赖于系统时钟的一致性，如果某台机器的系统时钟回拨了，有可能造成ID冲突，或者ID乱序。
+ 在启动之前，如果这台机器的系统时间回拨过，那么有可能出现ID重复的危险。

### 10.5 分布式事件监听的重点

