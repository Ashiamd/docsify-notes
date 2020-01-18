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

​	实现对ZooKeeper服务器端的事件监听，是客户端操作服务器端的一项重点工作，在Curator的API中，事件监听有两种模式：

**（1）一种是标准的观察者模式。**

**（2）另一种是缓存监听模式。**

​	<u>第一种标准的观察者模式是通过Watcher监听器实现的；第二种缓存监听模式是通过引入了一种本地缓存视图的Cache机制去实现的。</u>

​	第二种Cache事件监听机制，<u>可以理解为一个本地缓存视图与远程ZooKeeper视图的对比过程</u>，简单来说，Cache在客户端缓存了ZNode的各种状态，当感知到Zk集群的ZNode状态变化时，会触发事件，注册的监听器会处理这些事件。

​	虽然，<u>Cache是一种缓存机制，但是可以借助Cache实现实际的监听</u>。另外，**Cache机制提供了反复注册的能力，而观察模式的Watcher监听器只能监听一次。**

​	在类型上，Watcher监听器比较简单，只有一种。Cache事件监听器的种类有3种：Path Cache，Node Cache和Tree Cache。

#### 10.5.1 Watcher标准的事件处理器

​	在ZooKeeper中，接口类型Watcher用于表示一个标准的事件处理器，用来定义收到事件通知后相关的回调处理逻辑。

​	<u>接口类型Watcher包含KeeperState和EventType两个内部枚举类，分别代表了通知状态和事件类型。</u>

​	定义回调处理逻辑，需要使用Watcher接口的事件回调方法：process(WatchedEvent event)。定义一个Watcher的实例很简单，代码如下：

```java
Watcher w = new Watcher() {
    @Override
    public void process(WatchedEvent watchedEvent){
        log.info("监听器watchedEvent：" + watchedEvent);
    }
}
```

​	如何使用Watcher监听器实例呢？可以通过GetDataBuilder、GetChildrenBuilder和ExistsBuilder等这类实现了Watchable\<T\>接口的构造者，然后使用构造者的usingWatcher(Watcher w)方法，为构造者设置Watcher监听器实例。

​	在Curator中，Watchable\<T\>接口的源代码如下：

```java
package org.apache.curator.framework.api;
import org.apache.ZooKeeper.Watcher;
public interface Watchable<T> {
    T watched();
    T usingWatcher(Watcher w);
    T usingWatcher(CuratorWatcher cw);
}
```

​	GetDataBuilder、GetChildrenBuilder、ExistsBuilder构造者，分别通过getData()、getChildren()和checkExists()三个方法返回，也就是说，至少在以上三个方法的调用链上可以通过加上usingWatcher方法去设置监听器，典型的代码如下：

```java
//为GetDataBuilder实例设置监听器
byte[] content = client.getData().usingWatcher(w).forPath(workerPath);
```

​	<u>一个Watcher监听器在向服务器完成注册后，当服务器的一些事件触发了这个Watcher，就会向注册过的客户端会话发送一个事件通知来实现分布式的通知功能。在Curator客户端收到服务器端的通知后，会封装一个**WatchedEvent事件实例**，再传递给监听器的process(WatchedEvent e)回调方法。</u>

​	来看一下通知事件WatchedEvent实例的类型，WatchedEvent包含了三个基本属性：

+ 通知状态（keeperState）
+ 事件类型（EventType）
+ 节点路径（path）

​	**WatchedEvent并不是从ZooKeeper集群直接传递过来的事件实例，而是Curator封装过的事件案例。WatchedEvent类型没有实现序列化接口java.io.Serializable，因此不能用于网络传输**。从ZooKeeper服务器端直接通过网络传输传递过来的事件实例其实是一个**WatcherEvent类型的实例**，WatcherEvent传输实例和Curator的WatchedEvent封装实例，在名称上基本上一样的，只有一个字母之差，而且功能也是一样的，都表示的是同一个服务器端事件。

​	这里聚焦一下，只讲Curator封装过的WatchedEvent实例。WatchedEvent中所用到的通知状态和事件类型定义在Watcher接口中。Watcher接口定义的通知状态和事件类型，具体如下表。

| KeeperState      | EventType           | 触发条件                                                     | 说明                                                         |
| ---------------- | ------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
|                  | None(-1)            | 客户端与服务器成功建立连接                                   |                                                              |
| SyncConnected(0) | NodeCreated(1)      | 监听的对应数据节点被创建                                     |                                                              |
|                  | NodeDeleted(2)      | 监听的对应数据节点被删除                                     | 此时客户端和服务器处于连接状态                               |
|                  | NodeDataChanged(3)  | 监听的对应数据节点的数据内容发生变更                         |                                                              |
|                  | NodeChildChanged(4) | 监听的对应数据节点的子节点列表发生变更                       |                                                              |
| Disconnected(0)  | None(-1)            | 客户端与ZooKeeper服务器断开连接                              | 此时客户端和服务器处于断开连接状态                           |
| Expired(-112)    | Node(-1)            | 会话超时                                                     | 此时客户端会话失效，通常同时也会收到SessionExpiredException异常 |
| AuthFailed(4)    | None(-1)            | 通常有两种情况，1：使用错误的schema进行权限检查，2：SASL权限检查失败 | 通常同时也会收到AuthFailedException异常                      |

​	利用Watcher来对节点事件进行监听，来看一个简单的实例程序:

```java
@Slf4j
@Data
public class ZkWatcherDemo {
    private String workerPath = "/test/listener/remoteNode";
    private String subWorkerPath = "/test/listener/remoteNode/id-";

    @Test
    public void testWatcher() {
        CuratorFramework client = ZKclient.instance.getClient();

        //检查节点是否存在，没有则创建
        boolean isExist = ZKclient.instance.isNodeExist(workerPath);
        if (!isExist) {
            ZKclient.instance.createNode(workerPath, null);
        }

        try {

            Watcher w = new Watcher() {
                @Override
                public void process(WatchedEvent watchedEvent) {
                    System.out.println("监听到的变化 watchedEvent = " + watchedEvent);
                }
            };

            byte[] content = client.getData()
                .usingWatcher(w).forPath(workerPath);

            log.info("监听节点内容：" + new String(content));

            // 第一次变更节点数据
            client.setData().forPath(workerPath, "第1次更改内容".getBytes());

            // 第二次变更节点数据
            client.setData().forPath(workerPath, "第2次更改内容".getBytes());

            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    // ...其他方法
}
```

​	运行上面的程序代码，输出结果如下：

```java
// ... 省略其他的输出
监听到的变化watchedEvent = 
    WatchedEventstate:SyncConnectedtype:NodeDataChanged path:/test/listener/node
```

​	在这个程序中，节点路径"/list/listener/node"注册了一个Watcher监听器实例，随后调用setData方法改变节点内容，虽然改变了两次，但是监听器仅仅监听到了一个事件。换句话说，<u>监听器的注册时一次性的，当第二次改变节点内容时，注册已经失效，无法再次捕获节点的变动事件</u>。

​	<u>既然Watcher监听器时一次性的，如果要反复使用，需要反复地通过构造者的usingWatcher方法去提前进行注册。所以，Watcher监听器不适用于节点的数据频繁变动或者节点频繁变动这样的业务场景，而是适用于一些特殊的、变动不频繁的场景，例如**会话超时、授权失败**等这样的特殊场景</u>。

​	Watcher需要反复注册比较烦琐，所以Curator引入了Cache来监听ZooKeeper服务器端的事件。**Cache机制对ZooKeeper事件监听进行了封装，能够自动处理反复注册监听。**

#### 10.5.2 NodeCache节点缓存的监听

​	Curator引入的Cache缓存机制的实现拥有一个系列的类型，包括了Node Cache、Path Cache和Tree Cache三组类。其中：

（1）Node Cache节点缓存可用于ZNode节点的监听；

（2）Path Cache子节点缓存可用于ZNode的子节点的监听；

（3）Tree Cache树缓存是Path Cache的增强，不光能监听子节点，还能监听ZNode节点自身。

​	先看Node Cache，用于监控节点的增加，删除和更新。

​	有两个构造方法，具体如下：

```java
NodeCache(CuratorFramework client, String path)
NodeCache(CuratorFramework client, String path, boolean dataIsCompressed)
```

​	第一个参数就是传入创建的Curator的框架客户端，第二个参数就是监听节点的路径，第三个重载参数dataIsCompressed表示是否对数据进行压缩。

​	NodeCache使用的第二步，就是构造一个NodeCacheListener监听器实例。该接口的定义如下：

```java
package org.apache.curator.framework.recipes.cache;
public interface NodeCacheListener {
    void nodeChanged() throws Exception;
}
```

​	NodeCacheListener监听器接口只定义了一个简单的方法nodeChanged，当节点变化时，这个方法就会被回调。实例代码如下：

```java
NodeCacheListener l = new NodeCacheListener() {
    @Override
    public void nodeChanged() throws Exception {
        ChildData childData = nodeCache.getCurrentData();
        log.info("ZNode节点状态改变, path={}", childData.getPath());
        log.info("ZNode节点状态改变, data={}", new String(childData.getData(), "Utf-8"));
        log.info("ZNode节点状态改变, stat={}", childData.getStat());
    }
};
```

​	在创建完NodeCacheListener的实例之后，需要将这个实例注册到NodeCache缓存实例，使用缓存实例的addListener方法，然后使用缓存实例nodeCache的start方法来启动节点的事件监听。

```java
//启动节点的事件监听
nodeCache.getListenable().addListener(listener);
nodeCache.start();
```

​	再强调一下，需要调用nodeCache的start方法进行缓存和事件监听，这个方法有两个版本。

```java
void start(); // Start the cache.
void start(boolean buildInitial); // true代表缓存当前节点
```

​	唯一的一个参数buildInitail代表是否将该节点的数据立即进行缓存。如果设置为true的话，在start启动时立即调用NodeCache的getCurrentData方法能够得到对应节点的信息ChildData类，如果设置为false，就得不到对应的信息。

​	使用NodeCache来监听节点的事件，完整的实例代码如下：

```java
@Slf4j
@Data
public class ZkWatcherDemo {

    private String workerPath = "/test/listener/remoteNode";
    private String subWorkerPath = "/test/listener/remoteNode/id-";

    /**
     * NodeCache节点缓存的监听
     */
    @Test
    public void testNodeCache() {

        //检查节点是否存在，没有则创建
        boolean isExist = ZKclient.instance.isNodeExist(workerPath);
        if (!isExist) {
            ZKclient.instance.createNode(workerPath, null);
        }

        CuratorFramework client = ZKclient.instance.getClient();
        try {
            NodeCache nodeCache =
                new NodeCache(client, workerPath, false);
            NodeCacheListener l = new NodeCacheListener() {
                @Override
                public void nodeChanged() throws Exception {
                    ChildData childData = nodeCache.getCurrentData();
                    log.info("ZNode节点状态改变, path={}", childData.getPath());
                    log.info("ZNode节点状态改变, data={}", new String(childData.getData(), "Utf-8"));
                    log.info("ZNode节点状态改变, stat={}", childData.getStat());
                }
            };
            nodeCache.getListenable().addListener(l);
            nodeCache.start();

            // 第1次变更节点数据
            client.setData().forPath(workerPath, "第1次更改内容".getBytes());
            Thread.sleep(1000);

            // 第2次变更节点数据
            client.setData().forPath(workerPath, "第2次更改内容".getBytes());

            Thread.sleep(1000);

            // 第3次变更节点数据
            client.setData().forPath(workerPath, "第3次更改内容".getBytes());
            Thread.sleep(1000);

            // 第4次变更节点数据
            //            client.delete().forPath(workerPath);
            Thread.sleep(Integer.MAX_VALUE);
        } catch (Exception e) {
            log.error("创建NodeCache监听失败, path={}", workerPath);
        }
    }
}
```

​	通过运行的结果我们可以看到，NodeCache节点缓存能够重复地进行事件节点的监听。代码中的第三次监听的输出节选如下：

```java
//...省略前两次的输出
- ZNode节点状态改变，path=/test/listener/node
- ZNode节点状态改变，data=第三次更改内容
- ZNode节点状态改变，stat=17179869191，
...
```

​	**最后说明一下，如果在监听的时候NodeCache监听的节点为空（也就是说ZNode路径不存在），也是可以的。之后，如果创建了对应的节点，也是会触发事件从而回调nodeChanged方法。**

#### 10.5.3 PathChildrenCache子节点监听

​	PathChildrenCache子节点缓存用于子节点的监听，监控当前节点的子节点被创建、根本更新或者删除，需要强调：

+ **只能监听子节点，监听不到当前节点。**
+ **不能递归监听，子节点下的子节点不能递归监控**。

​	PathChildrenCache子节点缓存使用的第一步就是构造一个缓存实例。PathChildrenCache有多个重载版本的构造方法，下面选择4个进行寿命，具体如下：

```java
	//重载版本一
	public PathChildrenCache(CuratorFramework client, String path, boolean cacheData)

    //重载版本二
    public PathChildrenCache(CuratorFramework client, String path, boolean cacheData, boolean dataIsCompressed, final ExecutorService executorService)

    //重载版本三
    public PathChildrenCache(CuratorFramework client, String path, boolean cacheData, boolean dataIsCompressed, ThreadFactory threadFactory)

    //重载版本三
    public PathChildrenCache(CuratorFramework client, String path, boolean cacheData, ThreadFactory threadFactory)
```

​	所有的PathChildrenCache构造方法的前三个参数都是一样的。

+ 第一个参数就是传入创建的Curator框架的客户端。
+ 第二个参数就是监听节点的路径。
+ 第三个重载参数cacheData表示是否把节点的内容缓存起来。如果cacheData为true，那么接收到节点列表变更事件的同时会将获得节点内容。

​	除了上边的三个参数，其他参数的说明如下：

+ dataIsCompressed参数，表示是否对节点数据进行压缩。
+ threadFactory参数表示线程池工厂，当PathChildrenCache内部需要启动新的线程执行时，使用该线程池工厂来创建线程。
+ executorService和threadFactory参数差不多，表示通过传入的线程池或者线程工厂来异步处理监听事件。

​	构造完缓冲实例之后，PathChildrenCache缓存的第二步是构造一个子节点缓存监听器PathChildrenCacheListener实例。

​	PathChildrenCacheListener监听器接口的定义如下：

```java
package org.apache.curator.framework.recipes.cache;
import org.apache.curator.framework.CuratorFramework;
public interface PathChildrenCacheListener {
    void childEvent(CucratorFramework client, PathChildrenCacheEvent e) throws Exception;
}
```

​	**在PathChildrenCacheListener监听接口中只定义了一个简单的方法childEvent，当子节点有变化时，这个方法就会被回调。**PathChildrenCacheListener的使用实例如下：

```java
PathChildrenCacheListener l =
    new PathChildrenCacheListener() {
    @Override
    public void childEvent(CuratorFramework client,
                           PathChildrenCacheEvent event) {
        try {
            ChildData data = event.getData();
            switch (event.getType()) {
                case CHILD_ADDED:
                    log.info("子节点增加, path={}, data={}",
                             data.getPath(), new String(data.getData(), "UTF-8"));
                    break;
                case CHILD_UPDATED:
                    log.info("子节点更新, path={}, data={}",
                             data.getPath(), new String(data.getData(), "UTF-8"));
                    break;
                case CHILD_REMOVED:
                    log.info("子节点删除, path={}, data={}",
                             data.getPath(), new String(data.getData(), "UTF-8"));
                    break;
                default:
                    break;
            }
        } catch (
            UnsupportedEncodingException e) {
            e.printStackTrace();
        }
    }
};
```

​	在创建完PathChildrenCacheListener的实例之后，需要将这个实例注册到PathChildrenCache缓存实例，在调用缓存实例的addListenr方法，然后调用缓存实例nodeCache的start方法，启动节点的事件监听。

​	start方法可以传入启动的模式，定义在StartMode枚举中，具体如下：

+ NORMAL——异步初始化cache
+ BUILD_INITIAL_CACHE——同步初始化cache
+ POST_INITAILIZED_EVENT——异步初始化cache，并触发完成事件

​	StartMode枚举的三种启动方式，详细说明如下：

（1） BULID_INITIAL_CACHE模式：启动时同步初始化cache，表示创建cache后就从服务器提取对应的数据；

（2）POST_INITIALIZED_EVENT模式：启动时异步初始化cache，表示创建cache后从服务器提取对应的数据，完成后触发PathChildrenCacheEvent.Type#INITIALIZED事件，cache中Listener会收到该事件的通知；

（3）NORMAL模式：启动时异步初始化cache，完成后不会发出通知。

​	使用PathChildrenCache来监听节点的事件，完整的实例代码如下：

```java
@Slf4j
@Data
public class ZkWatcherDemo {
    /**
     * 子节点监听
     */
    @Test
    public void testPathChildrenCache() {

        //检查节点是否存在，没有则创建
        boolean isExist = ZKclient.instance.isNodeExist(workerPath);
        if (!isExist) {
            ZKclient.instance.createNode(workerPath, null);
        }

        CuratorFramework client = ZKclient.instance.getClient();

        try {
            PathChildrenCache cache =
                new PathChildrenCache(client, workerPath, true);
            PathChildrenCacheListener l =
                new PathChildrenCacheListener() {
                @Override
                public void childEvent(CuratorFramework client,
                                       PathChildrenCacheEvent event) {
                    try {
                        ChildData data = event.getData();
                        switch (event.getType()) {
                            case CHILD_ADDED:

                                log.info("子节点增加, path={}, data={}",
                                         data.getPath(), new String(data.getData(), "UTF-8"));

                                break;
                            case CHILD_UPDATED:
                                log.info("子节点更新, path={}, data={}",
                                         data.getPath(), new String(data.getData(), "UTF-8"));
                                break;
                            case CHILD_REMOVED:
                                log.info("子节点删除, path={}, data={}",
                                         data.getPath(), new String(data.getData(), "UTF-8"));
                                break;
                            default:
                                break;
                        }

                    } catch (
                        UnsupportedEncodingException e) {
                        e.printStackTrace();
                    }
                }
            };
            // 增加监听器
            cache.getListenable().addListener(l);
            // 设置启动模式
            cache.start(PathChildrenCache.StartMode.BUILD_INITIAL_CACHE);
            Thread.sleep(1000);
            
            // 创建3个子节点
            for (int i = 0; i < 3; i++) {
                ZKclient.instance.createNode(subWorkerPath + i, null);
            }

            Thread.sleep(1000);
            
            // 删除3个子节点
            for (int i = 0; i < 3; i++) {
                ZKclient.instance.deleteNode(subWorkerPath + i);
            }

        } catch (Exception e) {
            log.error("PathCache监听失败, path=", workerPath);
        }

    }
}
```

​	运行的结果如下：

```java
- 子节点增加，patt=/test/listener/node/id-0, data=to set content
- 子节点增加，patt=/test/listener/node/id-2, data=to set content
- 子节点增加，patt=/test/listener/node/id-1, data=to set content
......
- 子节点删除，patt=/test/listener/node/id-2, data=to set content
- 子节点删除，patt=/test/listener/node/id-0, data=to set content
- 子节点删除，patt=/test/listener/node/id-1, data=to set content
```

​	从执行结果可以看到，PathChildrenCache能偶反复地监听到节点的增加和删除。

​	**Curator监听的原理：无论是PathChildrenCache，还是TreeCache，所谓的监听都是在进行Curator本地缓存视图和ZooKeeper服务器远程的数据节点的对比，并且在进行数据同步时会触发相应的事件**。

​	<u>这里，以NODE\_ADDED(节点增加事件)的触发为例来进行说明。在本地缓存视图开始创建的时候，本地视图为空，从服务器进行数据同步时，本地的监听器就能监听到NODE\_ADDED事件。为什么呢？刚开始本地缓存并没有内容，然后本地缓存和服务器缓存进行对比，发现ZooKeeper服务器是有节点数据的，这才将服务器的节点缓存到本地，也会触发本地缓存的NODE\_ADDED事件</u>。

​	至此，已经讲完了两个系列的缓存监听。简单回顾一下：

（1）Node Cache用来观察ZNode自身，如果ZNode节点自身被创建，更新或者删除，那么Node Cache会更新缓存，并触发事件给注册的监听器。Node Cache是通过NodeCache类来实现的，监听器对应的接口为NodeCacheListener。

（2）Path Cache子节点缓存用来观察ZNode的子节点、并缓存子节点的状态，如果ZNode的子节点被创建，更新或者删除，那么Path Cache会更新缓存，并且触发事件给注册的监听器。Path Cache是通过PathChildrenCache类来实现的，监听器注册是通过PathChildrenCacheListener。

#### 10.5.4 Tree Cache节点树缓存

​	最后的一个系列是Tree Cache。

​	Tree Cache可以看作是Node Cache和Path Cache的合体。Tree Cache不光能监听子节点，这能监听节点自身。

​	Tree Cache使用的第一步就是构造一个TreeCache缓存实例。

​	Tree Cache类有两个构造方法，具体如下：

```java
// TreeCache构造器之一
TreeCache(CuratorFramework client, String path)

// TreeCache构造器之二
TreeCache(CuratorFramework client, String path, boolean cacheData, boolean dataIsCompressed, int maxDepth, ExecutorService executorService, boolean createParentNodes, TreeCacheSelector selector)
```

​	第一个参数就是传入创建的Curator框架的客户端，第二个参数就是监听节点的路径，其他参数的简单说明如下：

+ dataIsCompressed：表示是否对数据进行压缩。
+ maxDepth：表示缓存的层次深度，默认为整数最大值。
+ executorService：表示监听的执行线程池，默认会创建一个单一线程的线程池。
+ createParentNodes：表示是否创建父亲节点，默认为false。

​	如果要监听一个ZNode节点，在一般情况下，使用TreeCache的第一个构造函数即可。

​	TreeCache使用的第二步就是构造一个TreeCacheListener监听器实例。该接口的定义如下：

```java
//监听器接口定义
package org.apache.curator.framework.recipes.cache;
import org.apache.curator.framework.CuratorFramework;
public interface TreeCacheListener {
    void childEvent(CutatorFramework var1, TreeCacheEvent var2) throws Exception;
}
```

​	在TreeCacheListener监听器接口中也只定义了一个简单的方法childEvent，当子节点有变化时，这个方法就会被回调。TreeCacheListener的使用实例如下：

```java
TreeCacheListener l =
    new TreeCacheListener() {
    @Override
    public void childEvent(CuratorFramework client,
                           TreeCacheEvent event) {
        try {
            ChildData data = event.getData();
            if (data == null) {
                log.info("数据为空");
                return;
            }
            switch (event.getType()) {
                case NODE_ADDED:
                    log.info("[TreeCache]节点增加, path={}, data={}",
                             data.getPath(), new String(data.getData(), "UTF-8"));
                    break;
                case NODE_UPDATED:
                    log.info("[TreeCache]节点更新, path={}, data={}",
                             data.getPath(), new String(data.getData(), "UTF-8"));
                    break;
                case NODE_REMOVED:
                    log.info("[TreeCache]节点删除, path={}, data={}",
                             data.getPath(), new String(data.getData(), "UTF-8"));
                    break;
                default:
                    break;
            }

        } catch (
            UnsupportedEncodingException e) {
            e.printStackTrace();
        }
    }
};
```

​	在创建完TreeCacheListener的实例之后，调用缓存实例的addListener方法，将TreeCacheListener监听器实例注册到TreeCache缓存实例。然后调用缓存实例nodeCache的start方法来启动节点的事件监听。整个实例的代码如下：

```java
@Slf4j
@Data
public class ZkWatcherDemo {
    @Test
    public void testTreeCache() {
        //检查节点是否存在，没有则创建
        boolean isExist = ZKclient.instance.isNodeExist(workerPath);
        if (!isExist) {
            ZKclient.instance.createNode(workerPath, null);
        }
        CuratorFramework client = ZKclient.instance.getClient();

        try {
            TreeCache treeCache =
                new TreeCache(client, workerPath);
            TreeCacheListener l =
                new TreeCacheListener() {
                @Override
                public void childEvent(CuratorFramework client,
                                       TreeCacheEvent event) {
                    try {
                        ChildData data = event.getData();
                        if (data == null) {
                            log.info("数据为空");
                            return;
                        }
                        switch (event.getType()) {
                            case NODE_ADDED:
                                log.info("[TreeCache]节点增加, path={}, data={}",
                                         data.getPath(), new String(data.getData(), "UTF-8"));
                                break;
                            case NODE_UPDATED:
                                log.info("[TreeCache]节点更新, path={}, data={}",
                                         data.getPath(), new String(data.getData(), "UTF-8"));
                                break;
                            case NODE_REMOVED:
                                log.info("[TreeCache]节点删除, path={}, data={}",
                                         data.getPath(), new String(data.getData(), "UTF-8"));
                                break;
                            default:
                                break;
                        }
                    } catch (
                        UnsupportedEncodingException e) {
                        e.printStackTrace();
                    }
                }
            };
            treeCache.getListenable().addListener(l);
            treeCache.start();
            Thread.sleep(1000);
            for (int i = 0; i < 3; i++) {
                ZKclient.instance.createNode(subWorkerPath + i, null);
            }
            Thread.sleep(1000);
            for (int i = 0; i < 3; i++) {
                ZKclient.instance.deleteNode(subWorkerPath + i);
            }
            Thread.sleep(1000);
            ZKclient.instance.deleteNode(workerPath);
            Thread.sleep(Integer.MAX_VALUE);
        } catch (Exception e) {
            log.error("PathCache监听失败, path=", workerPath);
        }
    }
}
```

​	运行的结果如下：

```java
- [TreeCache]节点增加，path=/test/listener/node，data=to set content

- [TreeCache]节点增加，path=/test/listener/node/id-0，data=to set content
- [TreeCache]节点增加，path=/test/listener/node/id-1，data=to set content
- [TreeCache]节点增加，path=/test/listener/node/id-2，data=to set content
    
- [TreeCache]节点删除，path=/test/listener/node/id-2，data=to set content
- [TreeCache]节点删除，path=/test/listener/node/id-1，data=to set content
- [TreeCache]节点删除，path=/test/listener/node/id-0，data=to set content
    
- [TreeCache]节点删除，path=/test/listener/node，data=to set content
```

​	最后，说明一下TreeCacheEvent的事件类型，具体为：

+ NODE_ADDED对应于节点的增加。
+ NODE_UPDATED对应于节点的修改。
+ NODE_REMOVED对应于节点的删除。

​	对比TreeCacheEvent的事件类型与Path Cache的事件类型是不同的。回忆一下，Path Cache的事件类型具体如下：

+ CHILD_ADDED对应于子节点的增加。
+ CHILD_UPDATED对应于子节点的修改。
+ CHILD_REMOVED对应于子节点的删除。

### 10.6 分布式锁的原理与实践

​	在单体的应用开发场景中涉及并发同步的时候，大家往往采用Synchronized(同步)或者其他同一个JVM内Lock机制来解决多线程之间的同步问题。在分布式集群工作的开发场景中，就需要一种更加高级的锁机制来处理跨机器的进程之间的数据同步问题。这种跨机器的锁就是分布式锁。

#### 10.6.1 公平锁和可重入锁的原理

​	最经典的分布式锁是可重入的公平锁。什么是可重入的公平锁呢？

​	直接讲解概念和原理会比较抽象难懂，还是从具体的实例入手吧！这里尽量用一个简单的故事来类比，估计就简单多了。

​	故事发生在一个没有自来水的古代，在一个村子里有一口井，水质非常的好，村民们都抢着取井里的水。井就那么一口，存里的人很多，村民为争抢取水大家斗殴，甚至头破血流。

​	问题总是要解决，于是村长绞尽脑汁最终想出了一个凭号取水的方案。井边安排一个看井人，维护取水的秩序。取水秩序很简单：

（1）取水之前，先取号

（2）号排在前面的人，就可以先取水。

（3）先到的人排在前面，那些后到的人，一个一个挨着在井边排成一队。

​	这种排队取水模型就是一种锁的模型。排在最前面的号拥有取水权，就是一种典型的独占锁。另外，先到先得，号排在前面的人先取到水，取水之后就轮到下一个号取水，停公平的，说明它是一种公平锁。

​	什么是可重入锁呢？假定取水时以家庭为单位，家庭的某人拿到号，其他的家庭成员过来打水，这时候不用再取号。如果是同一个家庭，可以直接复用排号，不用重复取号从后面排起。

​	**只要满足条件，同一个取水号，可以用来多次取水，相当于重入锁的模型。在重入锁的模型中，一把独占锁可以被多次锁定，这就叫作可重入锁**。

#### 10.6.2 ZooKeeper分布式锁的原理

​	理解了经典的公平可重入锁的原理后，再来看在分布式场景下的公平可重入锁的原理。

​	通过前面的分析基本可以判定：ZooKeeper的临时顺序节点，天生就有一副实现分布式锁的胚子。为什么呢？

1. ZooKeeper的每一个节点都是一个天然的顺序发号器

   ​	<u>在每一个节点下面创建临时顺序节点（EPHEMERAL_SEQUENTIAL）类型，新的子节点后面会加上一个顺序编号。这个顺序编号是在上一个生成的顺序编号加1。</u>

   ​	例如，用一个发号的节点"/test/lock"为父亲节点，可以在这个父节点下面创建相同前缀的临时顺序子节点，假定相同的前缀为"/test/lock/seq-"。如果是第一个创建的子节点，那么生成的子节点为"/test/lock/seq-0000000000"，下一个节点则为"/test/lock/seq-0000000001"。

2. ZooKeeper节点的递增有序性可以确保锁的公平

   ​	一个ZooKeeper分布式锁，首先需要创建一个父节点，尽量是持久节点（PERSISTENT类型），然后每个要获得锁的线程都在这个节点下创建个临时的顺序节点。<u>由于Zk节点是按照创建的顺序依次递增的，为了确保公平，可以简单地规定，编号最小的那个节点表示获得了锁。因此，每个线程在尝试占用锁之前，首先判断自己的排号是不是当前最小的，如果是，则获取锁。</u>

3. ZooKeeper的节点监听机制可以保障占有锁的传递有序而且高效

   ​	<u>每个线程抢占锁之前，先抢号创建自己的ZNode。同样，释放锁的时候，就需要删除抢号的ZNode。</u>在抢号成之后，如果不是排号最小的节点，就处于等待通知的状态。等谁的通知呢？不需要其他人，只需要等前一个ZNode节点的通知即可。当前一个ZNode被删除的时候，就是轮到了自己占有锁的时候。第一个通知第二个、第二个通知第三个，击鼓传花似的依次向后传递。

   ​	ZooKeeper内部优越的机制，能保证由于网络异常或者其他原因造成集群中的占有锁的客户端失联时，锁能够被有效释放。一旦占用锁的ZNode客户端与ZooKeeper集群服务器失去联系，这个临时ZNode也将自动删除。排在它户面的那个节点也能收到删除事件，从而获得锁。所以，在创建取号节点的时候，尽量创建临时ZNode节点。

4. ZooKeeper的节点监听机制能够避免羊群效应

   ​	**ZooKeeper这种首尾相接，后面监听前面的方式，可以避免羊群效应。所谓羊群效应就是一个节点挂掉了，所有节点都去监听，然后作出反应，这样会给服务器带来巨大压力，所以有了临时顺序节点，当一个节点挂掉，只有它后面的那一个节点才作出反应。**

#### 10.6.3 分布式锁的基本流程

​	接下来就基于ZooKeeper实现一下分布式锁。

​	首先，定义了一个锁的接口Lock，很简单，仅仅两个抽象方法：一个加锁方法，一个解锁方法。Lock接口的代码如下：

```java
public interface Lock {
	/**
	 * 加锁方法
	 * @return 是否成功加锁
	 */
    boolean lock() throws Exception;
	
    /**
     * 解锁方法
     * @return 是否成功解锁
     */
    boolean unlock();
}
```

​	使用ZooKeeper实现分布式锁的算法，流程大致如下：

（1）一把锁，使用一个ZNode节点表示，如果锁对应的ZNode节点不存在，那么先创建ZNode节点。这里假设为"/test/lock"，代表了一把需要创建的分布式锁。

（2）抢占锁的所有客户端，使用锁的ZNode节点的子节点列表来表示；如果某个客户端需要占用锁，则在"/test/lock"下创建一个临时有序的子节点。

​	这里，所有子节点尽量共用一个有意义的子节点前缀。

​	如果子节点前缀为"/test/lock/seq-"，则第一个客户端对应的子节点为"/test/lock/seq-000000000"，第二个为"/test/lock/seq-000000001"，以此类推，也非常直观。

（3）如果判定客户端是否占有锁呢？很简单，客户端创建子节点后，需要进行判断：自己创建的子节点是否为当前子节点列表中序号最小的子节点。如果是，则认为加锁成功；如果不是，则监听前一个ZNode子节点的变更消息，等待前一个节点释放锁。

（4）一旦队列中后面的节点获得前一个子节点的变更通知，则开始进行判断，判断自己是否为当前子节点列表中序号最小的子节点，如果是，则认为加锁成功；如果不是，则持续监听，一直到获得锁。

（5）获得锁后，开始处理业务流程。在完成业务流程后，删除自己对应的子节点，完成释放锁的工作，以便后面的节点能捕获到节点的变更通知，获得分布式锁。

#### 10.6.4 加锁的实现

​	在Lock接口中，加锁的方法是lock()。lock()方法的大致流程是，首先尝试着去加锁，如果加锁失败就去等待，然后再重复。代码如下：

```java
@Slf4j
public class ZkLock implements Lock {
    //ZkLock的节点链接
    private static final String ZK_PATH = "/test/lock";
    private static final String LOCK_PREFIX = ZK_PATH + "/";
    private static final long WAIT_TIME = 1000;
    //Zk客户端
    CuratorFramework client = null;

    private String locked_short_path = null;
    private String locked_path = null;
    private String prior_path = null;
    final AtomicInteger lockCount = new AtomicInteger(0);
    private Thread thread;

    public ZkLock() {
        ZKclient.instance.init();
        if (!ZKclient.instance.isNodeExist(ZK_PATH)) {
            ZKclient.instance.createNode(ZK_PATH, null);
        }
        client = ZKclient.instance.getClient();
    }
    
    @Override
    public boolean lock() {

        synchronized (this) {
            if (lockCount.get() == 0) {
                thread = Thread.currentThread();
                lockCount.incrementAndGet();
            } else {
                if (!thread.equals(Thread.currentThread())) {
                    return false;
                }
                lockCount.incrementAndGet();
                return true;
            }
        }

        try {
            boolean locked = false;

            locked = tryLock();

            if (locked) {
                return true;
            }
            while (!locked) {

                await();

                //获取等待的子节点列表

                List<String> waiters = getWaiters();

                if (checkLocked(waiters)) {
                    locked = true;
                }
            }
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            unlock();
        }

        return false;
    }
    
    //...省略其他的方法
}
```

​	尝试加锁的tryLock方法是关键，它做了两件很重要的事情：

（1）创建临时顺序节点，并且保存自己的节点路径。

（2）判断是否是第一个，如果是第一个，则加锁成功。如果不是，就找到前一个ZNode节点，并且把它的路径把它的路径保存到prior_path。

​	myLock()方法的代码如下：

```java
@Slf4j
public class ZkLock implements Lock {
    //...省略其他成员变量和方法
    /**
     * 尝试枷锁
     * @return 是否加锁成功
     * @throws Exception异常
     */
    private boolean tryLock() throws Exception {
        //创建临时Znode节点
        List<String> waiters = getWaiters();
        locked_path = ZKclient.instance
            .createEphemeralSeqNode(LOCK_PREFIX);
        if (null == locked_path) {
            throw new Exception("zk error");
        }
        
        //取得加锁的排队编号
        locked_short_path = getShorPath(locked_path);
        
        //获得加锁的队列
        List<String> waiters = getWaiters();

        //获取等待的子节点列表，判断自己是否第一个
        if (checkLocked(waiters)) {
            return true;
        }

        // 判断自己排第几个
        int index = Collections.binarySearch(waiters, locked_short_path);
        if (index < 0) { // 网络抖动，获取到的子节点列表里可能已经没有自己了
            throw new Exception("节点没有找到: " + locked_short_path);
        }

        //如果自己没有获得锁，则要监听前一个节点
        prior_path = ZK_PATH + "/" + waiters.get(index - 1);

        return false;
    }
}
```

​	创建临时顺序节点后，它的完整路径存放在locked_path成员中。另外，还截取了一个后缀路径，放在locked_short_path成员中，后缀路径是一个短路径，只有完整路径的最后一层。为什么要单独保存段路径呢？<u>因为获取的远程子节点列表中的其他路径都是短路径，只有最后一层。为了方便进行比较，也把自己的短路经保存下来</u>。

​	创建了自己的临时节点后，调用checkLocked方法，判断是否锁定成功。如果锁定成功，则返回true；如果自己没有获得锁，则要监听前一个节点。找出前一个节点的路径，保存在prior_path成员中，供后面的await()等待方法用于监听。在介绍await()等待方法之前，先说下checkLocked锁定判断方法。

​	在checkLocked()方法中，判断是否可以持有锁。判断规则很简单：当前创建的节点是否在上一步获取到的子节点列表的第一个位置。

+ 如果是，说明可以持有锁，返回true，表示加锁成功。
+ 如果不是，说明有其他线程早已先持有了锁， 返回false。

​	checkLocked()方法的代码如下：

```java
@Slf4j
public class ZkLock implements Lock {
    //...省略其他成员变量和方法
    
    /**
     * 判断是否加锁成功
     * @param waiters 排队列表
     * @return 成功状态
     */
    private boolean checkLocked(List<String> waiters) {

        //节点按照编号，升序排列
        Collections.sort(waiters);

        // 如果是第一个，代表自己已经获得了锁
        if (locked_short_path.equals(waiters.get(0))) {
            log.info("成功的获取分布式锁,节点为{}", locked_short_path);
            return true;
        }
        return false;
    }
}
```

​	checkLocked方法比较简单，将参与排队的所有子节点列表从小到大根据节点名称进行排序，排序主要依靠节点的编号，也就是后10位数字，因为前缀都是一样的。排序之后，进行判断，如果自己的locked_short_path编号位置排在第一个，代表自己已经获得了锁。如果不是，则会返回false。

​	如果checkLocked()为false，那么外层的调用方法一般会执行等待方法await()执行争夺锁失败以后的等待逻辑。

​	await()也很简单，就是监听前一个ZNode节点prior_path成员的删除事件，代码如下：

```java
@Slf4j
public class ZkLock implements Lock {
    //...省略其他成员变量和方法
    
    /**
     * 等待，监听前一个节点的删除事件
     * @throws Exception 可能会有Zk异常、网络异常
     */
    private void await() throws Exception {

        if (null == prior_path) {
            throw new Exception("prior_path error");
        }

        final CountDownLatch latch = new CountDownLatch(1);

		//监听方式一：Watcher一次性订阅
        //订阅比自己次小顺序节点的删除事件
        Watcher w = new Watcher() {
            @Override
            public void process(WatchedEvent watchedEvent) {
                System.out.println("监听到的变化 watchedEvent = " + watchedEvent);
                log.info("[WatchedEvent]节点删除");

                latch.countDown();
            }
        };
		
        //开始监听
        client.getData().usingWatcher(w).forPath(prior_path);
        
        /*
        //监听方式二：TreeCache订阅
        //订阅比自己次小顺序节点的删除事件
        TreeCache treeCache = new TreeCache(client, prior_path);
        TreeCacheListener l = new TreeCacheListener() {
            @Override
            public void childEvent(CuratorFramework client,
                                   TreeCacheEvent event) throws Exception {
                ChildData data = event.getData();
                if (data != null) {
                    switch (event.getType()) {
                        case NODE_REMOVED:
                            log.debug("[TreeCache]节点删除, path={}, data={}",
                                    data.getPath(), data.getData());

                            latch.countDown();
                            break;
                        default:
                            break;
                    }
                }
            }
        };
        
		//开始监听
        treeCache.getListenable().addListener(l);
        treeCache.start();
        */
        
        //限时等待，最长加锁时间为3s
        latch.await(WAIT_TIME, TimeUnit.SECONDS);
    }
}
```

​	首先添加一个Watcher监听，而监听的节点正是前面所保存在prior_path成员的前一个节点的路径。这里，仅仅去监听自己的前一个节点的变动，而不是其他节点的变动，以提高效率。完成监听之后，调用latch.await()，线程进入等待状态，一直到线程被监听回调代码中的latch.countDown()所唤醒，或者等待超时。

​	在上面的代码中，监听前一个节点的删除可以使用两种监听方式：

+ Watcher订阅。
+ TreeCache订阅。

​	两种方式的效果都差不多。<u>但是，这里的删除事件只需要监听一次即可，不需要反复监听，所以，建议使用Watcher一次性订阅</u>。而程序中有关TreeCache订阅的代码已经被注释，供大家参考。**一旦前一个节点prior_path被删除，那么就将线程从等待状态唤醒，重新开始一轮锁的争夺，直到获得锁，再完成业务处理**。

​	至此，关于lock加锁的算法还差一点点就介绍完成。这一点，就是实现锁的可重入。<u>什么是可重入呢？只需要保障同一个线程进入加锁的代码，可以重复加锁成功即可。</u>修改前面的lock方法，在前面加上可重入的判读逻辑。代码如下：

```java
@Override
public boolean lock() {
    // 可重入的判断
    synchronized (this) {
        if(lockCount.get() == 0) {
            thread = Thread.currentThread();
            lockCount.incrementAndGet();
        } else {
            if(!thread.equals(Thread.currentThread())) {
                return false;
            }
            lockCount.incrementAndGet();
            return true;
        }
    }
}
```

​	为了变成可重入，在代码中增加了一个加锁的计数器lockCount，计算重复加锁的次数。如果是同一个线程加锁，只需要增加次数，直接返回，表示加锁成功。至此，lock()方法已经介绍完了，接下来，就是去释放锁。

#### 10.6.5 释放锁的实现

​	Lock接口中的unLock()方法用于释放锁，释放锁主要有两个工作：

+ 减少重入锁的计数，如果不是0，直接返回，表示成功地释放了一次
+ 如果计数器为0，移除Watchers监听器，并且删除创建的ZNode临时节点

​	unLock()方法的代码如下：

```java
@Slf4j
public class ZkLock implements Lock {
    //...省略其他成员变量和方法
    
    /**
     * 释放锁
     * @return 是否成功释放锁
     */
    @Override
    public boolean unlock() {
		
        // 没有加锁的线程能够解锁
        if (!thread.equals(Thread.currentThread())) {
            return false;
        }
		
        // 减少可重入的计数
        int newLockCount = lockCount.decrementAndGet();
		
        // 计数不能小于0
        if (newLockCount < 0) {
            throw new IllegalMonitorStateException("Lock count has gone negative for lock: " + locked_path);
        }
		
        // 如果计数不为0，直接返回
        if (newLockCount != 0) {
            return true;
        }
        try {
            // 删除临时节点
            if (ZKclient.instance.isNodeExist(locked_path)) {
                client.delete().forPath(locked_path);
            }
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }

        return true;
    }
}
```

​	**在程序中，为了尽量保证线程的安全，可重入计数器的类型不是int类型，而是Java并发包中的原子类型——AtomicInteger。**

#### 10.6.6 分布式锁的使用

​	编写一个用例，测试一下ZLock的使用，代码如下：

```java
/**
 * 测试分布式锁
 */
@Slf4j
public class ZkLockTester {
	//需要锁来保护的公共资源
    //变量
    int count = 0;
	/**
	 * 测试自定义分布式锁
	 * @throws InterruptedException异常
	 */
    @Test
    public void testLock() throws InterruptedException {
        // 10个并发任务
        for (int i = 0; i < 10; i++) {
            FutureTaskScheduler.add(new ExecuteTask() {
                @Override
                public void execute() {
                    //创建锁
                    ZkLock lock = new ZkLock();
                    lock.lock();
                    
					//每条线程执行10次累加
                    for (int j = 0; j < 10; j++) {
						// 公共资源变量累加
                        count++;
                    }
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    log.info("count = " + count);
                    lock.unlock();

                }
            });
        }
        Thread.sleep(Integer.MAX_VALUE);
    }

    // ... 省略其他成员变量和方法
}
```

​	ZLock仅仅对应一个ZNode路径，也就是说，仅仅代表一把锁。如果需要代表不同的ZNode路径，还需要进行简单改造。这种改造，留给各位自己去实现。

#### 10.6.7 Curator的InterProcessMutex可重入锁

​	Zlock实现的主要价值是展示一下分布式锁的原理和基础开发。<u>在实际的开发中，如果需要使用分布式锁，不建议自己“重复造轮子”，而建议直接使用Curator客户端中的各种官方实现的分布式锁，例如其中的InterProcessMutex可重入锁。</u>

​	这里提供一个InterProcessMutex可重入锁的使用实例，代码如下：

```java
/**
 * 测试分布式锁
 */
@Slf4j
public class ZkLockTester {
    //需要锁来保护的公共资源
    //变量
    int count = 0;

    /**
     * 测试ZK自带的互斥锁
     * @throws InterruptedException异常
     */
    @Test
    public void testzkMutex() throws InterruptedException {

        CuratorFramework client = ZKclient.instance.getClient();
		// 创建互斥锁
        final InterProcessMutex zkMutex =
            new InterProcessMutex(client, "/mutex");
        // 每条线程执行10次累加
        for (int i = 0; i < 10; i++) {
            FutureTaskScheduler.add(new ExecuteTask() {
                @Override
                public void execute() {

                    try {
                        // 获得互斥锁
                        zkMutex.acquire();

                        for (int j = 0; j < 10; j++) {
							// 公共的资源变量累加
                            count++;
                        }
                        try {
                            Thread.sleep(1000);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                        log.info("count = " + count);
                        //释放互斥锁
                        zkMutex.release();

                    } catch (Exception e) {
                        e.printStackTrace();
                    }

                }
            });
        }
        Thread.sleep(Integer.MAX_VALUE);
    }
}
```

​	最后，总结一下ZooKeeper分布式锁：

1. 优点：ZooKeeper分布式锁（如InterProcessMutex），能有效地解决分布式问题，不可重入问题，使用起来也较为简单。

2. 缺点：**ZooKeeper实现的分布式锁，性能并不高**。为什么呢？因为每次在创建和释放锁的过程中，都要动态创建、销毁暂时节点来实现锁功能。大家知道，**Zk中创建和删除节点只能通过Leader(主)服务器来执行**，然后Leader服务器还需要将数据同步到所有的Follower(从)服务器上，这样频繁的网络通信，性能的短板是非常突出的。

   ​	总之，**在高性能、高并发的应用场景下，不建议使用ZooKeeper的分布式锁**。而由于ZooKeeper的高可用性，因此<u>在并发量不是太高的应用场景中，还是推荐使用ZooKeeper的分布式锁</u>。

   ​	**目前分布式锁，比较成熟、主流的方案有两种：**

   **（1）基于Redis的分布式锁。适用于并发量很大、性能要求很高而可靠性问题可以通过其他方案去弥补的场景。**

   **（2）基于ZooKeeper的分布式锁。适用于高可靠（高可用），而并发量不是太高的场景。**

### 10.7 本章小结

​	在分布式系统中，ZooKeeper是一个重要的协调工具。本章介绍了<u>分布式命名服务、分布式锁</u>的原理以及基于ZooKeeper的参考实现。

## 第11章 分布式缓存Redis

