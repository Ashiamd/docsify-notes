# 《Netty、Redis、ZooKeeper高并发实战》学习笔记

> 对亏了系统的学习和各种社区的帖子帮助，总算完成了手机App和服务器的IM通讯了

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

​	<u>ZooKeeper为了保证**高吞吐和低延迟**，整个树形的目录结构全部都放在**内存**中</u>。与硬盘和其他的外存设备相比，计算机的内存比较有限，使得ZooKeeper的目录结构不能用于存放大量的数据。ZooKeeper官方的要求是，**每个节点存放的有效负载数据（Payload）的上限仅为1MB**。

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

​	缓存是一个很简单的问题，为什么要用缓存？主要原因是数据库的查询比较耗时，而使用缓存能大大节省数据访问的时间。举个例子，假如表中有两千万个用户信息，在加载用户信息时，一次数据库查询大致的时间在数百毫秒级别。这仅仅是一次查询，如果是频繁多次的数据库查询，效率就会更低。

​	<u>提升效率的通用做法是把数据加入缓存，每次加载数据之前，先去缓存中加载，如果为空，再去查询数据库并将数据加入缓存，这样可以大大提高数据访问的效率</u>。 

### 11.1 Redis入门

​	本节主要介绍Redis的安装和配置，以及Redis的客户端操作。

#### 11.1.1 Redis安装和配置

​	Redis在Windows下的安装很简单，根据系统的实际情况选择32位或者64位的Redis安装版本。

​	查看和修改Redis的配置项，有两种方式：

（1）是通过配置文件查看和修改；

（2）是通过配置命令查看和修改。

​	第一种方法，通过配置文件修改Redis的配置项。Redis在Windows中安装完成后，配置文件在Redis安装目录下，文件名为redis.windows.conf，可以复制它，保存一份自己的配置版本redis.conf，以自己的这份来作为运行时的配置文件。Redis在Linux中安装完后，redis.conf是一个默认的配置文件。通过redis.conf文件，可以查看和修改配置项的值。

​	第二种方式，通过命令修改Redis的配置项。启动Redis的命令客户端工具，连接上Redis服务，可以使用以下命令来查看和修改Redis配置项：

```shell
CONFIG GET CONFIG_SETTING_NAME
CONFIG SET CONFIG_SETTING_NAME NEW_CONFIG_VALUE
```

​	前一个命令CONFIG GET是查看命令，后面加配置项的名称；后一个命令CONFIG SET，是修改命令，后面加配置项的名称和要设置的新值.还要注意的是：**Redis的客户端命令是不区分字母大小的**；另外CONFIG GET查看命令可以使用通配符。

​	举个例子，查看Redis的服务端口，使用config get port，具体如下：

```shell
127.0.0.1:6379>config getport
1) "port"
2) "6379"
```

​	通过控制台输出的结果，我们可以看到，当前的Redis服务的端口为6379。

​	Redis的配置项比较好，大致的清单如下：
（1）port：端口配置项，查看和设置Redis监听端口，默认端口为6379。

（2）bind：主机地址配置项，查看和绑定的主机地址，默认地址的值为127.0.0.1。这个选项，在单机网卡的机器上，一般不需要修改。

（3）timeout：连接空闲多长要关闭连接，表示客户端闲置一段时间后，要关闭连接。如果指定为0，表示时长不限制。这个选项的默认值为0，表示默认不限制连接的空闲时长。

（4）dbfilename：指定保存缓存数据库的本地文件名，默认值为dump.rdb。

（5）dir：指定保存缓存数据的本地文件所存放的目录，默认的值为安装目录。

（6）rdbcompression：指定存储至本地数据库是否压缩数据，默认为yes，Redis采用LZF压缩，如果为了节省CPU时间，可以关闭该选项，但会导致数据库文件变得巨大。

（7）save：指定在多长时间内，有多少次Key-Value更新操作，就将数据同步到本地数据库文件。save配置项的格式为save\<seconds\>\<changes\>：seconds表示时间段的长度，changes表示变化的次数。如果在seconds时间段内，变化了changes次，则将Redis缓存数据同步到文件。

​	设置为900秒（15分钟）内有1个更改，则同步到文件：

```shell
127.0.0.1:6379> config set save "900 1"
OK
127.0.0.1:6379> config get save
1) "save"
2) "jd 900"
```

​	设置为900秒（15分钟）内有1个更改，300秒（5分钟）内有10个更改以及60秒内有10000个更改，三者满足一个条件，则同步到文件：

```shell
127.0.0.1:6379> config set save "900 1 300 10 60 10000"
OK
127.0.0.1:6379> config get save
1) "save"
2) "jd 900 jd 300 jd 60"
```

（8）requirepass：设置Redis连接密码，如果配置了连接密码，客户端在连接Reids时需要通过AUTH\<password\>命令提供密码，默认这个选项是关闭的。

（9）slaveof：在主从复制的模式下，设置当前节点为slave（从）节点时，设置master（主）节点的IP地址即端口，在Redis启动时，它会自动从master（主）节点进行数据同步。如果已经是slave（从）服务器，则会丢掉旧数据集，从新的master主服务器同步缓存数据。

```none
设置为slave节点命令的格式为：slaveof<masterip><masterport>
```

（10）masterauth：在主从复制的模式下，当master（主）服务器节点设置了密码保护时，slave（从）服务器连接master（主）服务器的密码。

```none
master服务器节点设置密码的格式为：masterauth<master-password>
```

（11）databases：<u>设置缓存数据库的数量，默认数据库数量为16个</u>。这个16个数据库的id为0-15，默认使用的数据库是第0个。可以使用SELECT\<dbid\>命令在连接时通过数据库id来指定要使用的数据库。

​	databases配置选项，可以设置多个缓存数据库，不同的数据库存放不同应用的缓存数据。类似mysql数据库中，不同的应用程序数据存在不同的数据库下。**在Redis中，数据库的名称由一个整数索引标识，而不是由一个字符串名称来标识**。在默认情况下，一个客户端连接到数据库0。可以通过SELECT <dbid\>命令来切换到不同的数据库。例如，命令select 2，将Redis数据库切换到第3个数据库，随后所有的Redis客户端命令将使用数据库3。

​	<u>Redis存储的形式是Key-Value（键-值对），其中Key（键）不能发生冲突。每个数据库都有属于自己的空间，不必担心之间的Key相冲突。在不同的数据库中，相同的Key可以分别取到各自的值。</u> 

​	<u>当清除缓存数据时，使用flushdb命令，只会清除当前数据库中的数据，而不会影响到其他数据库；而flushall命令，则会清除这个Redis实例所有数据库（从0-15）的数据，因此在执行这个命令前要格外小心。</u>

​	在Java编程中，配置连接Reids的uri连接字符串时，可以指定到具体的数据库，格式为：

```shell
redis://用户名:密码@host:port/Redis库名
```

​	举例如下

```shell
redis://testRedis:foobared@119.254.166.136:6379/1
```

​	表示连接到第2个Redis缓存库，其中的用户名是可以随意填写的。

> [Window配置Redis环境和简单使用](https://www.cnblogs.com/wxjnew/p/9160855.html)

#### 11.1.2 Redis客户端命令

​	通过安装目录下的redis-cli命令客户端，可以连接到Redis本地服务。如果需要在远程Redis服务上执行命令，我们使用的也是redis-cli命令。Windows/Linux命令的格式为：

```shell
redis-cli -h host -p port -a password
```

​	实例如下：

```shell
redis-cli -h 127.0.0.1 -p 6379 -a "123456"
```

​	此命令实例表示使用Redis命令客户端，连接到的远程主机为127.0.0.1，端口为6379，密码为"123456"的Redis服务上。

​	一旦连接上Redis本地服务或者远程服务，既可以通过命令客户端，完成Redis的命令执行，包括了基础Redis的Key-Value缓存操作。

（1）set命令：根据Key，设置Value值。

（2）get命令：根据Key，获取Value值。当Key不存在，会返回空结果。

set、get两个命令的使用很简单，与Java中Map数据类型的Key-Value设置与获取非常相似。如Key为"foo"、Value为"bar"的设置和获取，示例如下：

```shell
127.0.0.1:6379> set foo bar
OK
127.0.0.1:6379> get foo
"bar"
```

（3）keys命令：查找所有符合给定模式（Pattern）的Key。模式支持多种通配符，大致的规则如下表

| 符号 | 含义                                                 |
| ---- | ---------------------------------------------------- |
| ?    | 匹配一个字符                                         |
| *    | 匹配任意个（包括0个）字符                            |
| [-]  | 匹配取键内的任一字符，如a[b-d]可以匹配"ab","ac","ad" |
| \    | 转义符。使用\？，可以匹配"?"字符                     |

（4）exists命令：判断一个Key是否存在。如果Key存在，则返回整数类型1，否则返回0。

例如：

```shell
127.0.0.1:6379> exists foo
(integer) 1
127.0.0.1:6379> exists bar
(integer) 0
```

（5）expire命令：为指定的Key设置过期时间，以秒为单位。

（6）ttl命名：返回指定Key的剩余生存时间（ttl，time to live），以秒为单位

```shell
127.0.0.1:6379>set foo2 bar2
OK
127.0.0.1:6379>expire foo2 10000
(integer) 1
127.0.0.1:6379>ttl foo2
(integer) 9995
127.0.0.1:6379>ttl foo2
(integer) 9987
127.0.0.1:6379>ttl foo
(integer) -1
```

​	**如果没有指定剩余时间，默认的剩余生存时间为-1，表示永久存在。**

（7）type命令：返回Key所存储的Value值的类型。最简单的类型为string类型。**Redis中有5种数据类型：String（字符串类型）、Hash（哈希类型）、List（列表类型）、Set（集合类型）、Zset（有序集合类型）**。

（8）del命令：删除Key，可以删除一个或多个Key，返回值是删除的Key的个数。实例如下：

```shell
127.0.0.1:6379>del foo
(integer) 1
127.0.0.1:6379>del foo2
(integer) 1
```

（9）ping命令：检查客户端是否连接成功，如果连接成功，则返回pong。

#### 11.1.3 Redis Key的命名规范

​	在实际开发中，为了更好地进行命令空间的区分，Key会有很多的层次间隔，就像一棵目录树一样。例如"疯狂创客圈"的CrazyIM系统中，有缓存用户的Key，也有缓存IM消息的Key。为了以示区分，方便统计、更新、清除，可以将Key的命令组织成一种目录树一样的层次关系。

​	很多人习惯用英文句号来作为层次关系的Key的分隔符，例如：

```shell
superkey.subkey.subsubkey.subsubsubkey....
```

​	**而使用Redis，建议使用冒号作为superkey和subkey直接的风格符**，如下：

```shell
superkey:subkey:subsubkey:subsubsubkey....
```

​	例如，在"疯狂创客圈"的CrazyIM系统中有缓存用户的Key，也有缓存IM消息的Key，使用上面的规范，进行命名的规则如下：

+ 缓存用户的Key，命名规则为：CrazyIMKey:User:0001
+ 缓存消息的Key，命名规则为：CrazyIMKey:ImMessage:0001

​	最后的部分（如0001），表示的是业务ID。

​	Key的命名规范使用冒号分割，大致的优势如下：

（1）方便分层展示。Redis的很多客户端可视化管理工具，如Redis Desktop Manager，是以冒号作为分类展示的，方便快速查到要查询的Redis Key对应的Value值。

（2）方便删除与维护。可以对于某一层次下面的Key，使用通配符进行批量查询和批量删除。

### 11.2 Redis数据类型

​	**Redis中有5种数据类型：String（字符串类型）、Hash（哈希类型）、List（列表类型）、Set（集合类型）、Zset（有序集合类型）。**

#### 11.2.1 String字符串

​	String类型是Redis中最简单的数据结构。它既可以存储文字（例如"hello world"），又可以存储数字（例如整数10086和浮点数3.14），还可以存储二进制数据（例如10010100）。下面对String类型的主要操作进行简要介绍。

1. 设值：SET Key Value [EX seconds]

   ​	将Key键设置成指定的Value值。如果Key键已经存在，并且保存了一个旧值的话，旧的值会被覆盖，不论旧的类型是否为String都会被忽略掉。如果Key值不存在，那么会在数据库中添加一个Key键，保存的Value值就是刚刚设置的新值。

   ​	[EXseconds]选项表示Key键过期的时间，单位为秒。如果不加设置，表示Key键永不过期。另外，SET命令还有一些选项，由于使用较少，这里就展开说明了。

2. 批量设值：MSET Key Value [Key Value ...]

   ​	一次性设置多个Key-Value（键-值对）。相当于同时调用多次SET命令。不过要注意的是，**这个操作是原子的**。也就是说，所有的Key键都一次性设置的。如果同时运行两个MSET来设置相同的Key键，那么操作的结果也只会是两次MSET中后一次的结果，而不会是混杂的结果。

3. 批量添加：MSETNX Key Value [Key Value ...]

   ​	一次性添加多个Key-Value(键-值对)。**如果任何一个Key键已经存在，那么这个操作都不会执行**。所以，当使用MSETNX时，要么全部Key键被添加，要么全部不被添加。这个命令是在MSET命令后面增加了一个后缀NX（if Not eXist），表示只有Key键不存在的时候，才会设置Key键的Value值。

4. 获取：GET Key

   ​	使用GET命令，可以取得单个Key键所绑定的String值。

5. 批量获取：MGET Key [Key ...]

   ​	在GRT命令的前面增加了一个前缀M，表示多个（Multi）。使用MGET命令一次性获取多个Value值，这和多次使用GET命令取得单个值，有什么区别呢？主要在于减少网络传输的次数，提升了性能。

6. 获取长度：STRLEN Key

   ​	返回Key键对应的String的长度，如果Key键对应的不是String，则报错。如果Key键不存在，则返回0。

7. 为Key键对应的整数Value值增加1：INCR Key

8. 为Key键对应的整数Value值减少1：DECR Key

9. 为Key键对应的整数Value值增加increment：INCRBY Key increment
10. 为Key键对应的整数Value值减少decrement：DECRBY Key decrement

​	说明一下：**Redis并没有为浮点数Value值减少decrement的操作DECRBYFLOAT。如果要为浮点数Value值减少decrement，只需要把INCRBYFLOAT命令的increment设成负值即可。**

```shell
127.0.0.1:6379>set foo 1.0
OK
127.0.0.1:6379>incrbyfloat foo 10.01
"11.01"
127.0.0.1:6379>incrbyfloat foo -5.0
"6.01"
```

​	在例子中，首先为foo设置了一个浮点数，然后使用INCRBYFLOAT命令，为foo的值加上了10.01；最后将INCRBYFLOAT命令的参数设置成负数，为foo的值减少了5.0。

#### 11.2.2 List列表

​	**Redis的List类型是基于双向链表实现的**，可以支持正向、反向查找和遍历。从用户角度来说，List列表是简单的字符串列表，字符串按照添加的顺序排序。可以添加一个元素到List列表的头部（左边）或者尾部（右边）。一个List列表最多可以包含2<sup>32</sup>-1个元素（最多可以存储超过40亿个元素，4294967295）。

​	**List列表的典型应用场景：网络社区中最新的发帖列表、简单的消息队列、最新新闻的分页列表、博客的评论列表、排队系统等等**。举个具体的例子，在"双11"秒杀、抢购这样的大型活动中，短时间内有大量的用户请求发向服务器，而后台的程序不可能立即响应每一个用户的请求，有什么好的方法来解决这个问题呢？我们需要一个排队系统。<u>根据用户的请求时间，将用户的请求放入List队列中，后台程序依次从队列中获取任务，处理并将结果返回到结果队列</u>。换句话说，<u>通过List队列，可以将并行的请求转换成串行的任务队列，之后依次处理</u>。总体来说，List队列的使用场景，是非常多的。

​	下面对List类型的主要操作，进行简要介绍。

1. 右推入：RPUSH Key Value [Value ...]

   ​	也叫后推入。将一个或多个的Value值依次推入到列表的尾部（右端）。如果Key键不存在，那么RPUSH之前会先自动创建一个空的List列表。如果Key键的Value值不是一个List类型，则会返回一个错误。如果同时RPUSH多个Value值，则多个Value值会依次从尾部进入List列表。<u>RPUSH命令的返回值为操作完成后List包含的元素量</u>。RPUSH时间复杂度为O(N)，如果只推入一个值，那么命令的复杂度为O(1)。

2. 左推入：LPUSH Key Value [Value ...]

   ​	也叫前推入。这个命令和RPUSH几乎一样，只是推入元素的地点不同，是从List列表的头部（左侧）推入的。

3. 左弹出：LPOP Key

   ​	PUSH操作是增加元素；而POP操作，则是<u>获取元素并删除</u>。LPOP命令是从List队列的左边（前端），获取并移除一个元素，复杂度O(1)。如果List列表为空，则返回nil。

4. 右弹出：RPOP Key

   ​	与LPOP功能基本相同，是从队列的右边（后端）获取并移除一个元素，复杂度O(1)。

5. 获取列表的长度：LLEN Key

6. 获取列表指定位置上的元素：LINDEX Key index

7. 获取指定索引范围之内的所有元素：LRANGE Key start stop

8. 设置指定索引上的元素：LSET Key index Value

   不能设置超过原本范围的索引的元素值，比如原本就2个，不能设置index为2的元素值，会报错(error) ERR index out of range

​	**List列表的下标是从0开始的，index为负的时候是从后向前数。-1表示最后一个元素。当下标超出边界时，会返回nil**。

#### 11.2.3 Hash哈希表

​	Redis中的Hash表是一个String类型的Field字段和Value值之间的映射表，类似于Java中的HashMap。一个哈希表由多个字段-值对（Field-Value Pair）组成，Value值可以是文字、整性、浮点数或者二进制数据。在同一个Hash哈希表中，每个Field字段的名称必须时唯一的。这一点和Java中的HashMap的Key键的规范要求也是八九不离十的。下面对Hash哈希表的主要操作进行简要介绍。

1. 设置字段-值：HSET Key Field Value；

   ​	在Key哈希表中，给Field字段设置Value值。如果Field字段之前没有设置值，那么命令返回1；如果Field字段已经有关联值，那么命令用新值覆盖旧值，并返回0。

2. 获取字段-值：HGET Key Field；

   ​	返回Key哈希表中Field字段所关联的Value值。如果Field字段没有关联Values，那么返回nil。

3. 检查字段是否存在：HEXISTS Key Field；

   ​	查看在Key哈希表中，指定Field字段是否存在：存在则返回1，不存在则返回0。

4. 删除指定的字段：HDEL Key Field [Field ...]

   ​	删除Key哈希表中，一个或多个指定Field字段，以及那些Field字段所关联的值。不存在的Field字段将被忽略。命令返回被成功删除的Field-Value对的数量。

5. 查看指定的Field字段是否存在：HEXISTS Key Field；

6. 获取所有的Field字段：HEKYS Key；

7. 获取所有的Value值：HVALS Key。

​	总结一下，使用Hash哈希列表的好处：

（1）将数据集中存放。通过Hash哈希表，可以将一些相关的信息存储在同一个缓存Key键中，不仅方便了数据管理，还可以尽量避免误操作的发生。

（2）避免键名冲突。<u>在介绍缓存Key命名规范时，可以在命名键的时候，使用冒号分隔符来避免命名冲突，但更好的避免冲突的办法是直接使用哈希键来存储"键-值对"数据</u>。

（3）减少Key键的内存占用。**在一般情况下，保存相同数量的"键-值对"信息，使用哈希键比使用字符串键更节约内存**。因为Redis创建一个Key都带有很多的附加管理信息（例如这个Key键的类型、最后一次被访问的时间等），所以缓存的Key键越多，耗费的内存就越多，花在管理数据库Key键上的CPU也会越多。

​	总之，**应该尽量使用Hash哈希表而不是字符串键来缓存Key-Value"键-值对"数据，优势为：方便管理、能够避免键名冲突、并且还能够节约内存**。

#### 11.2.4 Set集合

​	Set集合也是一个列表，不过它的特殊之处在于它是可以自动去掉重复元素的。Set集合类型的使用场景是：当需要存储一个列表，而又不希望有重复的元素(例如ID的集合)时，使用Set是一个很好的选择。<u>并且Set类型拥有一个命令，它可用于判断某个元素是否存在，而List类型并没有这种功能的命令</u>。

​	通过Set集合类型的命令可以快速地向集合添加元素，或者从集合里面删除元素，也可以对多个Set集合进行集合运算，例如并集、交集、差集。

1. 添加元素：SADD Key member1 [member2 ...]

   ​	可以向Key集合中，添加一个或者多个成员。  

2. 移除元素：SREM Key  member1 [member2 ...]

   ​	从Key集合中移除一个或者多个成员。

3. 判断某个元素：SISMEMBER Key member

   ​	判断member元素是否为Key集合的成员。

4. 获取集合的成员数：**SCARD Key**

5. 获取集合中的所有成员：SMEMBERS Key

#### 11.2.5 Zset有序集合

​	Zset有序集合和Set集合的使用场景类似，区别是**有序集合会根据提供的score参数来进行自动排序**。当需要一个不重复的且有序的集合列表，那么就可以选择Zset有序集合列表。<u>常用案例：游戏中的排行榜。</u>

​	Zset有序集合和Set集合不同的是，有序集合的每个元素，都关联着一个分值（Score），这是一个浮点数格式的关联值。Zset有序集合会按照分值（score），按照**从小到大**的顺序来排列有序集合中的各个元素。

1. 添加成员：ZADD Key Score1 member1 [ScoreN memberN ...]

   ​	向有序集合Key中添加一个或者多个成员。如果memberN已经存在，则更新已存在成员的分数。

2. 移除元素：ZREM Key member1 [memberN ...]

   ​	从有序集合Key中移除一个或者多个成员

3. 取得分数：ZSCORE Key member

   ​	从有序集合Key中，取得member成员的分数值。

4. 取得成员排序：ZRANK Key member

   ​	从有序集合Key中，取得member成员的分数值的排名。

5. 成员加分：ZINCRBY Key increment member

   ​	在有序集合Key中，对指定成员的分数加上增量Score。

6. 区间获取：ZRANGEBYSCORE Key min max[WITHSCORES] [LIMIT]

   ​	从有序集合Key中，获取指定**分数区间范围**内的成员。WITHSCORES表示带上分数值返回；LIMIT选项，类似于mysql查询的limit选项，有offset、count两个参数，表示返回的偏移量和成员数量。

   ​	在默认情况下，min和max表示的范围，是闭包间范围，而不是开区间范围，即min <= score <= max 内的成员将被返回。另外，**可以使用-inf 和+inf分别表示有序集合中分数的最小值和最大值。**

7. 获取成员数：ZCARD Key

8. 区间计数：ZCOUNT Key min max

   ​	在有序集合Key中，计算<u>指定区间分数</u>的成员数。

### 11.3 Jedis基础编程的实践案例

​	Jedis是一个高性能的Java客户端，是Redis官方推荐的Java开发工具。要在Java开发中访问Redis缓存服务器，必须对Jedis熟悉才能编写出"漂亮"的代码。[Jedis的项目地址](https://github.com/xetorthio/jedis)

​	使用Jedis，可以在Maven的pom文件中，增加以下依赖：

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>${redis.version}</version>
</dependency>
```

​	本实践实例所使用的依赖版本为2.9.0。

​	Jedis基本的使用十分简单，在每次使用时，构建Jedis对象即可。<u>一个Jedis对象代表一条和Reids服务进行连接的Socket通道。使用完Jedis对象之后，需要调用Jedis.close()方法把连接关闭，否则会占用系统资源</u>。

​	创建Jedis对象时，可以指定Redis服务的host，port和password。大致的伪代码如下：

```java
Jedis jedis = new Jedis("localhost",6379); // 指定Redis服务的主机和端口
jedis.auth("XXXX");	// 如果Redis服务连接需要密码，就设置密码
// .... 访问Redis服务
jedis.close(); // 使用完，就关闭连接
```

#### 11.3.1 Jedis操作String字符串

​	Jedis的String字符串操作函数和Redis客户端操作String字符串的命令，基本上可以一比一的相互对应。正因为如此，本节部对Jedis的String字符串操作函数进行清单式的说明，只设计了一个比较全面的String字符串操作的示例程序，演示一下这些函数的使用。

​	Jedis操作String字符串具体的示例程序代码，如下：

```java
public class StringDemo {

    /**
     * Redis 字符串数据类型的相关命令用于操作 redis 字符串值
     */
    @Test
    public void operateString() {
        Jedis jedis = new Jedis("localhost", 6379);
        //如果返回 pang 代表链接成功
        System.out.println("jedis.ping():" + jedis.ping());
        //设置key0的值 123456
        jedis.set("key0", "123456");
        //返回数据类型  string
        System.out.println("jedis.type(key0): " + jedis.type("key0"));
        //get key
        System.out.println("jedis.get(key0): " + jedis.get("key0"));
        // key是否存在
        System.out.println("jedis.exists(key0):" + jedis.exists("key0"));
        //返回key的长度
        System.out.println("jedis.strlen(key0): " + jedis.strlen("key0"));
        //返回截取字符串, 范围 0,-1 表示截取全部
        System.out.println("jedis.getrange(key0): " + jedis.getrange("key0", 0, -1));
        //返回截取字符串, 范围 1,4 表示从表示区间[1,4]
        System.out.println("jedis.getrange(key0): " + jedis.getrange("key0", 1, 4));

        //追加
        System.out.println("jedis.append(key0): " + jedis.append("key0", "appendStr"));
        System.out.println("jedis.get(key0): " + jedis.get("key0"));

        //重命名
        jedis.rename("key0", "key0_new");
        //判断key 是否存在
        System.out.println("jedis.exists(key0): " + jedis.exists("key0"));

        //批量插入
        jedis.mset("key1", "val1", "key2", "val2", "key3", "100");
        //批量取出
        System.out.println("jedis.mget(key1,key2,key3): " + jedis.mget("key1", "key2", "key3"));
        //删除
        System.out.println("jedis.del(key1): " + jedis.del("key1"));
        System.out.println("jedis.exists(key1): " + jedis.exists("key1"));
        //取出旧值 并set新值
        System.out.println("jedis.getSet(key2): " + jedis.getSet("key2", "value3"));
        //自增1 要求数值类型
        System.out.println("jedis.incr(key3): " + jedis.incr("key3"));
        //自增15 要求数值类型
        System.out.println("jedis.incrBy(key3): " + jedis.incrBy("key3", 15));
        //自减1 要求数值类型
        System.out.println("jedis.decr(key3): " + jedis.decr("key3"));
        //自减5 要求数值类型
        System.out.println("jedis.decrBy(key3): " + jedis.decrBy("key3", 15));
        //增加浮点类型
        System.out.println("jedis.incrByFloat(key3): " + jedis.incrByFloat("key3", 1.1));

        //返回0 只有在key不存在的时候才设置
        System.out.println("jedis.setnx(key3): " + jedis.setnx("key3", "existVal"));
        System.out.println("jedis.get(key3): " + jedis.get("key3"));// 3.1

        //只有key都不存在的时候才设置,这里返回 null
        System.out.println("jedis.msetnx(key2,key3): " + jedis.msetnx("key2", "exists1", "key3", "exists2"));
        System.out.println("jedis.mget(key2,key3): " + jedis.mget("key2", "key3"));

        //设置key 2 秒后失效
        jedis.setex("key4", 2, "2 seconds is no Val");
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        // 2 seconds is no Val
        System.out.println("jedis.get(key4): " + jedis.get("key4"));


        jedis.set("key6", "123456789");
        //下标从0开始，从第三位开始,将新值覆盖旧值
        jedis.setrange("key6", 3, "abcdefg");
        //返回：123abcdefg
        System.out.println("jedis.get(key6): " + jedis.get("key6"));

        //返回所有匹配的key
        System.out.println("jedis.get(key*): " + jedis.keys("key*"));

        jedis.close();

    }
}
```

运行结果如下：

```java
jedis.ping():PONG
jedis.type(key0): string
jedis.get(key0): 123456
jedis.exists(key0):true
jedis.strlen(key0): 6
jedis.getrange(key0): 123456
jedis.getrange(key0): 2345
jedis.append(key0): 15
jedis.get(key0): 123456appendStr
jedis.exists(key0): false
jedis.mget(key1,key2,key3): [val1, val2, 100]
jedis.del(key1): 1
jedis.exists(key1): false
jedis.getSet(key2): val2
jedis.incr(key3): 101
jedis.incrBy(key3): 116
jedis.decr(key3): 115
jedis.decrBy(key3): 100
jedis.incrByFloat(key3): 101.1
jedis.setnx(key3): 0
jedis.get(key3): 101.09999999999999
jedis.msetnx(key2,key3): 0
jedis.mget(key2,key3): [value3, 101.09999999999999]
jedis.get(key4): null
jedis.get(key6): 123abcdefg
jedis.get(key*): [key0_new, key2, key6, key3]
```

#### 11.3.2 Jedis操作List列表

​	Jedis的List列表操作函数和Redis客户端操作List列表的命令，基本上可以一比一的相互对应。正因为如此，本节部对Jedis的List列表操作函数进行清单式的说明，只设计了一个比较全面的List列表操作的示例程序，演示一下这些函数的使用。

```java
public class ListDemo {

    /**
     * Redis列表是简单的字符串列表，按照插入顺序排序。
     * 可以添加一个元素到列表的头部（左边）或者尾部（右边）
     */
    @Test
    public void operateList() {
        Jedis jedis = new Jedis("localhost");
        System.out.println("jedis.ping(): " +jedis.ping());
        jedis.del("list1");

        //从list尾部添加3个元素
        jedis.rpush("list1", "zhangsan", "lisi", "wangwu");

        //取得类型, list
        System.out.println("jedis.type(): " +jedis.type("list1"));

        //遍历区间[0,-1]，取得全部的元素
        System.out.println("jedis.lrange(0,-1): " +jedis.lrange("list1", 0, -1));
        //遍历区间[1,2]，取得区间的元素
        System.out.println("jedis.lrange(1,2): " +jedis.lrange("list1", 1, 2));

        //获取list长度
        System.out.println("jedis.llen(list1): " +jedis.llen("list1"));
        //获取下标为 1 的元素
        System.out.println("jedis.lindex(list1,1): " +jedis.lindex("list1", 1));
        //左侧弹出元素
        System.out.println("jedis.lpop(): " +jedis.lpop("list1"));
        //右侧弹出元素
        System.out.println("jedis.rpop(): " +jedis.rpop("list1"));
        //设置下标为0的元素val
        jedis.lset("list1", 0, "lisi2");
        //最后，遍历区间[0,-1]，取得全部的元素
        System.out.println("jedis.lrange(0,-1): " +jedis.lrange("list1", 0, -1));

        jedis.close();
    }

}
```

运行结果如下：

```java
jedis.ping(): PONG
jedis.type(): list
jedis.lrange(0,-1): [zhangsan, lisi, wangwu]
jedis.lrange(1,2): [lisi, wangwu]
jedis.llen(list1): 3
jedis.lindex(list1,1): lisi
jedis.lpop(): zhangsan
jedis.rpop(): wangwu
jedis.lrange(0,-1): [lisi2]
```

#### 11.3.3 Jedis操作Hash哈希表

​	Jedis的Hash哈希表操作函数和Redis客户端操作Hash哈希表的命令，基本上可以一比一的相互对应。正因为如此，本节部对Jedis的Hash哈希表操作函数进行清单式的说明，只设计了一个比较全面的Hash哈希表操作的示例程序，演示一下这些函数的使用。

```java
public class HashDemo {

    /**
     * Redis hash 是一个string类型的field和value的映射表，
     * hash特别适合用于存储对象。
     * Redis 中每个 hash 可以存储 2^32 - 1 键值对（40多亿）
     */
    @Test
    public void operateHash() {

        Jedis jedis = new Jedis("localhost");
        jedis.del("config");
        //设置hash的 field-value 对
        jedis.hset("config", "ip", "127.0.0.1");

        //取得hash的 field的关联的value值
        System.out.println("jedis.hget(): " + jedis.hget("config", "ip"));
        //取得类型：hash
        System.out.println("jedis.type(): " + jedis.type("config"));

        //批量添加 field-value 对，参数为java map
        Map<String, String> configFields = new HashMap<String, String>();
        configFields.put("port", "8080");
        configFields.put("maxalive", "3600");
        configFields.put("weight", "1.0");
        //执行批量添加
        jedis.hmset("config", configFields);
        //批量获取：取得全部 field-value 对，返回 java map
        System.out.println("jedis.hgetAll(): " + jedis.hgetAll("config"));
        //批量获取：取得部分 field对应的value，返回 java map
        System.out.println("jedis.hmget(): " + jedis.hmget("config", "ip", "port"));

        //浮点数增加: 类似于String的 incrByFloat
        jedis.hincrByFloat("config", "weight", 1.2);
        System.out.println("jedis.hget(weight): " + jedis.hget("config", "weight"));

        //获取所有的key
        System.out.println("jedis.hkeys(config): " + jedis.hkeys("config"));
        //获取所有的val
        System.out.println("jedis.hvals(config): " + jedis.hvals("config"));

        //获取长度
        System.out.println("jedis.hlen(): " + jedis.hlen("config"));
        //判断field是否存在
        System.out.println("jedis.hexists(weight): " + jedis.hexists("config", "weight"));

        //删除一个field
        jedis.hdel("config", "weight");
        System.out.println("jedis.hexists(weight): " + jedis.hexists("config", "weight"));
        jedis.close();
    }
}
```

运行结果如下：

```java
jedis.hget(): 127.0.0.1
jedis.type(): hash
jedis.hgetAll(): {port=8080, weight=1.0, maxalive=3600, ip=127.0.0.1}
jedis.hmget(): [127.0.0.1, 8080]
jedis.hget(weight): 2.2
jedis.hkeys(config): [port, weight, maxalive, ip]
jedis.hvals(config): [127.0.0.1, 3600, 8080, 2.2]
jedis.hlen(): 4
jedis.hexists(weight): true
jedis.hexists(weight): false
```

#### 11.3.4 Jedis操作Set集合

​	Jedis的Set集合操作函数和Redis客户端操作Set集合的命令，基本上可以一比一的相互对应。正因为如此，本节部对Jedis的Set集合操作函数进行清单式的说明，只设计了一个比较全面的Set集合操作的示例程序，演示一下这些函数的使用。

```java
public class SetDemo {

    /**
     * Redis 的 Set 是 String 类型的无序集合。
     * 集合成员是唯一的，这就意味着集合中不能出现重复的数据。
     * Redis 中集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是 O(1)。
     * 集合中最大的成员数为 2^32 - 1 (4294967295, 每个集合可存储40多亿个成员)。
     */
    @Test
    public void operateSet() {
        Jedis jedis = new Jedis("localhost");
        jedis.del("set1");
        System.out.println("jedis.ping(): " + jedis.ping());
        System.out.println("jedis.type(): " + jedis.type("set1"));

        //sadd函数: 向集合添加元素
        jedis.sadd("set1", "user01", "user02", "user03");
        //smembers函数: 遍历所有元素
        System.out.println("jedis.smembers(): " + jedis.smembers("set1"));
        //scard函数: 获取集合元素个数
        System.out.println("jedis.scard(): " + jedis.scard("set1"));
        //sismember 判断是否是集合元素
        System.out.println("jedis.sismember(user04): " + jedis.sismember("set1", "user04"));
        //srem函数：移除元素
        System.out.println("jedis.srem(): " + jedis.srem("set1", "user02", "user01"));
        //smembers函数: 遍历所有元素
        System.out.println("jedis.smembers(): " + jedis.smembers("set1"));

        jedis.close();
    }

}
```

运行结果如下：

```java
jedis.ping(): PONG
jedis.type(): none
jedis.smembers(): [user02, user01, user03]
jedis.scard(): 3
jedis.sismember(user04): false
jedis.srem(): 2
jedis.smembers(): [user03]
```

#### 11.3.5 Jedis操作Zset有序集合

​	Jedis的Zset有序集合操作函数和Redis客户端操作Zset有序集合的命令，基本上可以一比一的相互对应。正因为如此，本节部对Jedis的Zset有序集合操作函数进行清单式的说明，只设计了一个比较全面的Zset有序集合操作的示例程序，演示一下这些函数的使用。

```java
public class ZSetDemo {

    /**
     * Redis 有序集合和集合一样也是string类型元素的集合,且不允许重复的成员。
     * 不同的是每个元素都会关联一个double类型的分数。
     * redis正是通过分数来为集合中的成员进行从小到大的排序。
     * 有序集合的成员是唯一的,但分数(score)却可以重复。
     * 集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是O(1)。
     * 集合中最大的成员数为 2^32 - 1 (4294967295, 每个集合可存储40多亿个成员)。
     */
    @Test
    public void operateZset() {

        Jedis jedis = new Jedis("localhost");
        System.out.println("jedis.get(): " + jedis.ping());

        jedis.del("salary");
        Map<String, Double> members = new HashMap<String, Double>();
        members.put("u01", 1000.0);
        members.put("u02", 2000.0);
        members.put("u03", 3000.0);
        members.put("u04", 13000.0);
        members.put("u05", 23000.0);
        //批量添加元素
        jedis.zadd("salary", members);
        //类型,zset
        System.out.println("jedis.type(): " + jedis.type("salary"));

        //获取集合元素个数
        System.out.println("jedis.zcard(): " + jedis.zcard("salary"));
        //按照下标[起,止]遍历元素
        System.out.println("jedis.zrange(): " + jedis.zrange("salary", 0, -1));
        //按照下标[起,止]倒序遍历元素
        System.out.println("jedis.zrevrange(): " + jedis.zrevrange("salary", 0, -1));

        //按照分数（薪资）[起,止]遍历元素
        System.out.println("jedis.zrangeByScore(): " + jedis.zrangeByScore("salary", 1000, 10000));
        //按照薪资[起,止]遍历元素,带分数返回
        Set<Tuple> res0 = jedis.zrangeByScoreWithScores("salary", 1000, 10000);
        for (Tuple temp : res0) {
            System.out.println("Tuple.get(): " + temp.getElement() + " -> " + temp.getScore());
        }
        //按照分数[起,止]倒序遍历元素
        System.out.println("jedis.zrevrangeByScore(): " + jedis.zrevrangeByScore("salary", 1000, 4000));
        //获取元素[起,止]分数区间的元素数量
        System.out.println("jedis.zcount(): " + jedis.zcount("salary", 1000, 4000));

        //获取元素score值：薪资
        System.out.println("jedis.zscore(): " + jedis.zscore("salary", "u01"));
        //获取元素下标
        System.out.println("jedis.zrank(u01): " + jedis.zrank("salary", "u01"));
        //倒序获取元素下标
        System.out.println("jedis.zrevrank(u01): " + jedis.zrevrank("salary", "u01"));
        //删除元素
        System.out.println("jedis.zrem(): " + jedis.zrem("salary", "u01", "u02"));
        //删除元素,通过下标范围
        System.out.println("jedis.zremrangeByRank(): " + jedis.zremrangeByRank("salary", 0, 1));
        //删除元素,通过分数范围
        System.out.println("jedis.zremrangeByScore(): " + jedis.zremrangeByScore("salary", 20000, 30000));
        //按照下标[起,止]遍历元素
        System.out.println("jedis.zrange(): " + jedis.zrange("salary", 0, -1));

        Map<String, Double> members2 = new HashMap<String, Double>();
        members2.put("u11", 1136.0);
        members2.put("u12", 2212.0);
        members2.put("u13", 3324.0);
        //批量添加元素
        jedis.zadd("salary", members2);
        //增加指定分数
        System.out.println("jedis.zincrby(10000): " + jedis.zincrby("salary", 10000, "u13"));
        //按照下标[起,止]遍历元素
        System.out.println("jedis.zrange(): " + jedis.zrange("salary", 0, -1));

        jedis.close();

    }
}
```

运行结果如下：

```java
jedis.get(): PONG
jedis.type(): zset
jedis.zcard(): 5
jedis.zrange(): [u01, u02, u03, u04, u05]
jedis.zrevrange(): [u05, u04, u03, u02, u01]
jedis.zrangeByScore(): [u01, u02, u03]
Tuple.get(): u01 -> 1000.0
Tuple.get(): u02 -> 2000.0
Tuple.get(): u03 -> 3000.0
jedis.zrevrangeByScore(): []
jedis.zcount(): 3
jedis.zscore(): 1000.0
jedis.zrank(u01): 0
jedis.zrevrank(u01): 4
jedis.zrem(): 2
jedis.zremrangeByRank(): 2
jedis.zremrangeByScore(): 1
jedis.zrange(): []
jedis.zincrby(10000): 13324.0
jedis.zrange(): [u11, u12, u13]
```

### 11.4 JedisPool连接池的实践案例

​	使用Jedis API可以方便地在Java程序中操作Redis，就像通过JDBC API操作数据库一样。但是仅仅实现这一点是不够地。为什么呢？大家知道，**数据库连接的底层是一条Socket通道，创建和销毁很耗时**。在数据库连接过程中，为了防止数据库连接的频繁创建、销毁带来的性能损耗，常常会用到连接池(Connection Pool)，例如淘宝的Druid连接池、Tomcat的DBCP连接池。Jedis连接和数据库连接一样，也需要使用连接池(Connection Pool)来管理。

​	Jedis开源库提供了一个负责管理Jedis连接对象的池，名为JedisPool类，位于redis.clients.jedis包中。

> [Jedis和RedisTemplate有何区别？](https://blog.csdn.net/varyall/article/details/83476970)
>
> [Redis深入学习：Jedis和Spring的RedisTemplate](https://blog.csdn.net/CSDN2497242041/article/details/102675435)
>
> [[Redis的三个框架：Jedis,Redisson,Lettuce](https://www.cnblogs.com/liyan492/p/9858548.html)]

#### 11.4.1 JedisPool的配置

​	在使用JedisPool类创建Jedis连接池之前，首先要了解一个很重要的配置类——JedisPoolConfig配置类，它也位于redis.clients.jedis包中。这个连接池的配置类负责配置JedisPool的参数。JedisPoolConfig配置类涉及到很多与连接管理和使用有关的参数，下面对它的一些重要参数进行说明。

（1）maxTotal：资源池中最大的连接数，默认值为8。

（2）maxIdle：资源池允许最大空闲的连接数，默认值为8。

（3）minIdle：资源池确保最少空闲的连接数，默认值为0。如果JedisPool开启了空闲连接的有效性检测，如果空闲连接无效，就销毁。销毁连接后，连接数量减少了，如果小于minIdle数量，就新建连接，维护数量不少于minIdle的数量。minIdle确保了线程池中有最小的空闲Jedis实例的数量。

（4）blockWhenExhausted：当资源池用尽后，调用者是否要等待，默认值为true。当为true时，maxWaitMillis才会生效。

（5）maxWaitMillis：当资源池连接用尽后，调用者的最大等待时间（单位为毫秒）。默认值为-1，表示永不超时，不建议使用默认值。

（6）testOnBorrow：向资源池借用连接时，是否做有效性检查（ping命令），如果是无效连接，会被移除，默认值为false，表示不做检测。如果为true，则得到的Jedis实例均是可用的。**在业务量小的应用场景，建议设置为true，确保连接可用；在业务量很大的应用场景，建议设置为false（默认值），少一次ping命令的开销，有助于提升性能**。

（7）testOnReturn：向资源池归还连接时，是否做有效性检测（ping命令），如果是无效连接，会被移除，默认值为false，表示不做检测。同样，在业务量很大的应用场景，建议设置为false(默认值)，少一次ping命令的开销。

（8）testWhileIdle：如果为true，表示用一个专门的线程对空闲的连接进行有效性的检测扫描，如果有效性检测失败，则表示无效连接会从资源池中移除。默认值为true，表示进行空闲连接的检测。这个选项存在一个附加条件，需要配置项timeBetweenEvictionRunsMillis的值大于0；否则testWhileIdle不会生效。

（9）timeBetweenEvictionRunsMillis：表示两次空闲连接扫描的活动之间要睡眠的毫秒数，默认为30000毫秒，也就是30秒钟。

（10）minEvictableIdleTimeMillis：表示一个Jedis连接至少停留在空闲状态的最短时间，然后才能被空闲连接扫描线程进行有效性检测，默认值为60000毫秒，即60秒。也就是说在默认情况下，一条Jedis连接只有空闲60秒后，才会参与空闲线程的有效性检测。这个选项存在一个附加条件，需要在timeBetweenEvictionRunsMillis大于0时才会生效。也就是说，如果不启动空闲检测线程，这个参数也没有什么意义。

（11）numTestsPerEvictionRun：表示空闲检测线程每次最多扫描的Jedis连接数，默认值为-1，表示扫描全部的空闲连接。

​	空闲扫描的选项在JedisPoolConfig的构造器中都有默认值，具体如下：

```java
package redis.clients.jedis;
import org.apache.commons.pool2.impl.GenericObjectPoolConfig;
public class JredisPoolBuilder extends GenericObjectPoolConfig {
    public JedisPoolConfig() {
        this.setTestWhileIdle(true);
		this.setMinEvictableIdleTimeMillis(60000L);
        this.setTimeBetweenEvictionRunsMillis(30000L);
        this.setNumTestsPerEvictionRun(-1);
    }
}
```

（12）jmxEnabled：是否开启jmx监控，默认值为true，建议开启。

​	有个实际的问题：如何推算一个连接池的最大连接数maxTotal呢？

​	实际上，这是一个很难精准回答的问题，主要是依赖的因素比较多。大致的推算方法是：**业务QPS/单连接的QPS = 最大连接数。**

​	如果推算单个Jedis连接的QPS呢？假设一个Jedis命令操作的时间约为5ms(包含borrow+return+Jedis执行命令+网络延迟)，那么，单个Jedis连接的QPS大约是100/5=200。如果业务期望的QPS是100000，那么需要的最大连接数为100000、200 = 500。

​	实际上，上面的估算仅仅是个理论值。在实际的生产场景中，还要预留一些资源，通常来讲所配置的maxTotal要比理论值大一些。

​	**如果连接数确实太多，可以考虑Redis集群，那么单个Redis节点的最大连接数的公式为：maxToal = 预估的连接数 / nodes节点数**

​	在并发量不大时，maxTotal设置过高会导致不必要的连接资源的浪费。可以根据实际总QPS和nodes节点数，合理评估每个节点所使用的最大连接数。

​	在看一个问题：如何推算连接池的最大空闲连接数maxIdle值呢？

​	**实际上，maxTotal只是给出了一个连接数量的上限，maxIdle实际上才是业务可用的最大连接数**，从这个层面来说，maxIdle不能设置国小，否则会有创建、销毁连接的开销。使得连接池达到最佳性能的设置是maxTotal = maxIdle，应尽可能地避免由于频繁地创建和销毁Jedis连接所带来的连接池性能的下降。

#### 11.4.2 JedisPool创建和预热

​	创建JedisPool连接池的一般步骤为创建一个JedisPoolConfig配置实例；以JedisPoolConfig实例、Redis IP、Redis端口和其他可选选项（如超时时间、Auth密码）为参数，构造一个JedisPool连接池实例。

```java
public class JredisPoolBuilder {

    public static final int MAX_IDLE = 50;
    public static final int MAX_TOTAL = 50;
    private static JedisPool pool = null;
    //....
   
    //创建连接池
    private static JedisPool buildPool() {
        if (pool == null) {
            long start = System.currentTimeMillis();
            JedisPoolConfig config = new JedisPoolConfig();
            config.setMaxTotal(MAX_TOTAL);
            config.setMaxIdle(MAX_IDLE);
            config.setMaxWaitMillis(1000 * 10);
            // 在borrow一个jedis实例时，是否提前进行validate操作；
            // 如果为true，则得到的jedis实例均是可用的；
            config.setTestOnBorrow(true);
            //new JedisPool(config, ADDR, PORT, TIMEOUT, AUTH);
            pool = new JedisPool(config, "127.0.0.1", 6379, 10000);
            long end = System.currentTimeMillis();
            Logger.info("buildPool  毫秒数:", end - start);
        }
        return pool;
    }
}
```

​	<u>虽然JedisPool定义了最大空闲资源数、最小空闲资源数，但是在创建的时候，不会真的创建好Jedis连接并放到JedisPool池子里。这样会导致一个问题，刚创建好的连接池，池子没有Jedis连接资源在使用，在初次访问请求到来的时候，才开始创建新的连接，不过，这样会导致一定的时间开销。为了提升初次访问的性能，可以考虑在JedisPool创建后，为JedisPool提前进行预热，一般以最小空闲数量作为预热数量。</u>

```java
public class JredisPoolBuilder {
    //...
    
    //连接池的预热
    public static void hotPool() {

        long start = System.currentTimeMillis();
        List<Jedis> minIdleJedisList = new ArrayList<Jedis>(MAX_IDLE);
        Jedis jedis = null;

        for (int i = 0; i < MAX_IDLE; i++) {
            try {
                jedis = pool.getResource();
                minIdleJedisList.add(jedis);
                jedis.ping();
            } catch (Exception e) {
                Logger.error(e.getMessage());
            } finally {
            }
        }

        for (int i = 0; i < MAX_IDLE; i++) {
            try {
                jedis = minIdleJedisList.get(i);
                jedis.close();
            } catch (Exception e) {
                Logger.error(e.getMessage());
            } finally {

            }
        }
        long end = System.currentTimeMillis();
        Logger.info("hotPool  毫秒数:", end - start);

    }
}
```

​	在自己定义的JredisPoolBuilder连接池Builder类中，创建好连接池实例，并且进行预热，然后，定义一个从连接池中获取Jedis连接的新方法——getJedis()，供其他模块调用。

```java
public class JredisPoolBuilder {
    private static JedisPool pool = null;
    //...
    
    static {
        //创建连接池
        buildPool();
        //预热连接池
        hotPool();
    }
    
    //获取连接
    public synchronized static Jedis getJedis() {
        return pool.getResource();
    }
    //...
}
```

#### 11.4.3 JedisPool的使用

​	可以使用前面定义好的getJedis()方法，间接地通过pool.getResource()从连接池获取连接；也可以直接通过pool.getResource()方法获取Jedis连接。

​	**主要的要求是Jedis连接使用完之后，一定要调用close方法关闭连接，这个关闭操作不是真正地关闭连接，而是归还给连接池**。<u>这一点和使用数据库连接池是一样的。一般来说，关闭操作放在finally代码段中，确保Jedis的关闭最终都会被执行到。</u>

```java
public class JredisPoolTester {

    public static final int NUM = 200;
    public static final String ZSET_KEY = "zset1";

    //测试删除
    @Test
    public void testDel() {
        Jedis redis =null;
        try  {
            redis = JredisPoolBuilder.getJedis();
            long start = System.currentTimeMillis();
            redis.del(ZSET_KEY);
            long end = System.currentTimeMillis();
            Logger.info("删除 zset1  毫秒数:", end - start);
        } finally {
            //使用后一定关闭，还给连接池
            if (redis != null) {
                redis.close();
            }
        }
    }
    // ...
}
```

​	**由于Jedis类实现了java.io.Closeable接口，故而在JDK1.7或者以上版本可以使用try-with-resources语句，在其隐藏的finally部分自动调用close方法。**

```java
public class JredisPoolTester {

    public static final int NUM = 200;
    public static final String ZSET_KEY = "zset1";

    //测试创建zset
    @Test
    public void testSet() {
        testDel();

        try (Jedis redis = JredisPoolBuilder.getJedis()) {
            int loop = 0;
            long start = System.currentTimeMillis();
            while (loop < NUM) {
                redis.zadd(ZSET_KEY, loop, "field-" + loop);
                loop++;
            }
            long end = System.currentTimeMillis();
            Logger.info("设置 zset :", loop, "次, 毫秒数:", end - start);
        }
    }
}
```

​	**这里使用try-with-resources的效果和使用try-finally写法是一样的，只是它会默认调用jedis.close()方法。这里优先推荐try-with-resources写法，因为比较简洁、干净。大家平时常用的数据库连接、输入输出流的关闭，都可以使用这个方法。**

### 11.5 使用spring-data-redis完成CRUD的实践案例

​	无论是Jedis还是JedisPool，都只是完成对Redis操作的极为基础的API，在不依靠任何中间件的开发环境中，可以使用他们。但是，<u>一般的Java开发，都会使用了Spring框架，可以使用spring-data-redis开源库来简化Redis操作的代码逻辑，做到最大程度的业务聚焦。</u>

​	下面从缓存的应用场景入手，介绍spring-data-redis开源库的使用。

#### 11.5.1 CRUD中应用缓存的场景

​	在普通CRUD应用场景中，很多情况下需要同步操作缓存，推荐使用Spring的spring-data-redis开源库。注：CRUD是指Create创建，Retrieve查询，Update更新和Delete删除。

​	一般来说，在普通的CRUD应用场景中，大致涉及到的缓存操作为：

1. 创建缓存

   ​	在创建Create一个POJO实例的时候，对POJO实例进行分布式缓存，一般以"缓存前缀+ID"为缓存的Key键，POJO对象为缓存的Value值，直接缓存POJO的二进制字节。前提是：POJO必须可序列化，实现java.Serializable空接口。<u>如果POJO不可序列化，也是可以缓存的，但是必须自己实现序列化的方法，例如使用JSON方式序列化。</u>

2. 查询缓存

   ​	<u>在查询Retrieve一个POJO实例的时候，首先应该根据POJO缓存的Key键，从Redis缓存中返回结果。如果不存在，才去查询数据库，并且能够将数据库的结果缓存起来。</u>

3. 更新缓存

   ​	在更新Update一个POJO实例的时候，既需要更新数据库的POJO数据记录，也需要更新POJO的缓存记录。

4. 删除缓存

   ​	在删除Delete一个POJO实例的时候，既需要删除数据库的POJO数据记录，也需要删除POJO的缓存记录。

   ​	使用spring-data-redis开源库可以快速地完成上述的缓存CRUD操作。

   ​	为了演示CRUD场景下的Redis缓存操作，首先定义一个简单的POJO实体类：聊天系统的用户类。此类拥有一些简单的属性，如uid和nickName，且这些属性都具备基本的getter和setter方法。

   ```java
   public class User implements Serializable {
   
       String uid;
       String devId;
       String token;
       String nickName;
       // .... 
       //...省略getter和setter等方法
   }
   ```

   ​	然后定义一个完成CRUD操作的Service接口，定义三个方法：

   （1）saveUser完成创建C、更新操作U

   （2）getUser完成查询操作R

   （3）deleteUser完成删除操作D

   Service接口的代码如下：

   ```java
   public interface UserService {
   
       /**
        * CRUD 之   查询
        *
        * @param id id
        * @return 用户
        */
       User getUser(long id);
   
       /**
        * CRUD 之  新增/更新
        *
        * @param user 用户
        */
       User saveUser(final User user);
   
       /**
        * CRUD 之 删除
        *
        * @param id id
        */
   
       void deleteUser(long id);
   
       /**
        * 删除全部
        */
       public void deleteAll();
   
   }
   ```

   ​	定义完了Service接口之后，接下来就是定义Service服务的具体实现。不过，这里聚焦的是：如何通过spring-data-redis库，使Service实现待缓存的功能？

#### 11.5.2 配置spring-redis.xml

​	使用spring-data-redis库的第一步是，要在Maven的pom文件中加上spring-data-redis库的依赖，具体如下：

```xml
<!-- https://mvnrepository.com/artifact/org.springframework.data/spring-data-redis -->
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-redis</artifactId>
    <version>2.2.4.RELEASE</version>
</dependency>
```

​	使用spring-data-redis库的第二步，即配置spring-data-redis库的连接池实例和RedisTemplate模板实例。这是两个spring bean，可以配置在项目统一的spring xml配置文件中，也可以编写一个独立的spring-redis.xml配置文件。这里使用的是第二种方式。

​	连接池实例和RedisTemplate模板实例的配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:cache="http://www.springframework.org/schema/cache"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           http://www.springframework.org/schema/beans/spring-beans.xsd
                           http://www.springframework.org/schema/context
                           http://www.springframework.org/schema/context/spring-context-4.0.xsd
                           http://www.springframework.org/schema/cache
                           http://www.springframework.org/schema/cache/spring-cache.xsd"
       default-lazy-init="false">


    <context:component-scan base-package="com.crazymakercircle.redis.springJedis"/>

    <context:annotation-config/>

    <!-- 启用缓存注解功能，这个是必须的，否则注解不会生效 -->
    <cache:annotation-driven/>

    <!-- 加载配置文件 -->
    <context:property-placeholder location="classpath:redis.properties"/>
    <!-- redis数据源 -->
    <bean id="poolConfig" class="redis.clients.jedis.JedisPoolConfig">
        <!-- 最大空闲数 -->
        <property name="maxIdle" value="${redis.maxIdle}"/>
        <!-- 最大空连接数 -->
        <property name="maxTotal" value="${redis.maxTotal}"/>
        <!-- 最大等待时间 -->
        <property name="maxWaitMillis" value="${redis.maxWaitMillis}"/>
        <!-- 连接超时时是否阻塞，false时报异常,ture阻塞直到超过maxWaitMillis, 默认true -->
        <property name="blockWhenExhausted" value="${redis.blockWhenExhausted}"/>
        <!-- 返回连接时，检测连接是否成功 -->
        <property name="testOnBorrow" value="${redis.testOnBorrow}"/>
    </bean>

    <!-- Spring-redis连接池管理工厂 -->
    <bean id="jedisConnectionFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
        <!-- IP地址 -->
        <property name="hostName" value="${redis.host}"/>
        <!-- 端口号 -->
        <property name="port" value="${redis.port}"/>
        <!-- 连接池配置引用 -->
        <property name="poolConfig" ref="poolConfig"/>
        <!-- usePool：是否使用连接池 -->
        <property name="usePool" value="true"/>
    </bean>

    <!-- redis template definition -->
    <bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
        <property name="connectionFactory" ref="jedisConnectionFactory"/>
        <property name="keySerializer">
            <bean class="org.springframework.data.redis.serializer.StringRedisSerializer"/>
        </property>
        <property name="valueSerializer">
            <bean class="org.springframework.data.redis.serializer.JdkSerializationRedisSerializer"/>
        </property>
        <property name="hashKeySerializer">
            <bean class="org.springframework.data.redis.serializer.StringRedisSerializer"/>
        </property>
        <property name="hashValueSerializer">
            <bean class="org.springframework.data.redis.serializer.JdkSerializationRedisSerializer"/>
        </property>
        <!--开启事务  -->
        <property name="enableTransactionSupport" value="true"></property>
    </bean>

    <!--自定义redis工具类,在需要缓存的地方注入此类  -->
    <bean id="cacheManager" class="org.springframework.data.redis.cache.RedisCacheManager">
        <constructor-arg ref="redisTemplate"/>
        <constructor-arg name="cacheNames">
            <set>
                <!--声明userCache-->
                <value>userCache</value>
            </set>
        </constructor-arg>
    </bean>


    <!--将redisTemplate 封装成缓存service-->
    <bean id="cacheOperationService" class="com.crazymakercircle.redis.springJedis.CacheOperationService">
        <property name="redisTemplate" ref="redisTemplate"/>
    </bean>
    <!--业务service,依赖缓存service-->
    <bean id="serviceImplWithTemplate" class="com.crazymakercircle.redis.springJedis.UserServiceImplWithTemplate">
        <property name="cacheOperationService" ref="cacheOperationService"/>
    </bean>

    <bean id="serviceImplInTemplate" class="com.crazymakercircle.redis.springJedis.UserServiceImplInTemplate">
        <property name="redisTemplate" ref="redisTemplate"/>
    </bean>

</beans>
```

​	spring-data-redis库在JedisPool提供连接池的基础上封装了自己的连接池——RedisConnectionFactory连接工厂；并且spring-data-redis封装了一个短期、非线程安全的连接类，名为RedisConnection连接类。RedisConnection类和Jedis库中的Jedis类原理一样，提供了与Redis客户端命令一对一的API函数，用于操作远程Redis服务。

​	<u>在使用spring-data-redis时，虽然没有直接用到Jedis库，但是spring-data-redis库底层对Redis服务的操作还是调用Jedis库完成的</u>。也就是说，spring-data-redis库从一定程度上使大家更好地使用Jedis库。

​	RedisConnection的API命令操作的对象都是字节级别的Key键和Value值。为了更进一步地减少开发的工作，spring-data-redis库在RedisConnection连接类的基础上，针对不同的缓存类型，设计了五大数据类型的命令API集合，用于完成不同类型的数据缓存操作，并封装在RedisTemplate模板类中。

#### 11.5.3 使用RedisTemplate模板API

​	RedisTemplate模板类位于核心包org.springframework.data.redis.core中，它封装了五大数据类型的命令API集合：

1. ValueOperations字符串类型操作API集合
2. ListOperations列表类型操作API集合
3. SetOperations集合类型操作API集合
4. ZSetOperations有序集合类型API集合
5. HashOperations哈希类型操作API集合

​	每一种类型的操作API基本上都和每一种类型的Redis客户端命令一一对应。但是在API的名称上并不完全一致，RedisTemplate的API名称更加人性化。例如，Redis客户端命令setNX——Key-Value不存在才设值，非常不直观，但是RedisTemplate的API名称为setIfAbsent，翻译过来就是——如果不存在，则设值。selfAbsent比setNX易懂多了。

​	除了名称存在略微的调整，总体上而言，RedisTemplate模板类的API函数和Redis客户端命令是一一对应的关系。所以，本节不再一一赘述RedisTemplate模板类中的API函数，大家可以自行阅读API的源代码。

​	在实际开发中，为了尽可能地减少第三方库的"入侵"，或者为了在不同的第三方库之间进行方便的切换，一般来说，要对第三方库进行封装。

​	下面将RedisTemplate模板类的大部分缓存操作封装成一个自己的缓存操作Service服务——CacheOperationService，部分代码（完整500+行）如下：

```java
public class CacheOperationService {


    private RedisTemplate redisTemplate;

    public void setRedisTemplate(RedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }
    // --------------RedisTemplate 基础操作  --------------------


    /**
     * 取得指定格式的所有的key
     *
     * @param patens 匹配的表达式
     * @return key 的集合
     */
    public Set getKeys(Object patens) {
        try {
            return redisTemplate.keys(patens);
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }

    /**
     * 指定缓存失效时间
     *
     * @param key  键
     * @param time 时间(秒)
     * @return
     */
    public boolean expire(String key, long time) {
        try {
            if (time > 0) {
                redisTemplate.expire(key, time, TimeUnit.SECONDS);
            }
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    //....

    // --------------RedisTemplate 操作 String --------------------


    /**
     * 普通缓存获取
     *
     * @param key 键
     * @return 值
     */
    public Object get(String key) {
        return key == null ? null : redisTemplate.opsForValue().get(key);
    }

    /**
     * 普通缓存放入
     *
     * @param key   键
     * @param value 值
     * @return true成功 false失败
     */
    public boolean set(String key, Object value) {
        try {
            redisTemplate.opsForValue().set(key, value);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }

    }
    // ...

    /**
     * 移除N个值为value
     *
     * @param key   键
     * @param count 移除多少个
     * @param value 值
     * @return 移除的个数
     */
    public long lRemove(String key, long count, Object value) {
        try {
            Long remove = redisTemplate.opsForList().remove(key, count, value);
            return remove;
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }
}
```

​	在代码中，<u>除了基本数据类型的Redis操作（如keys、hasKey）直接使用redisTemplate实例完成。其他的API命令，都是在不同类型的命令集合类上完成的。</u>

​	redisTemplate提供了5个方法，取得不同类型的命令集合，具体为：

1. redisTemplate.opsForValue()取得String类型命令API集合。
2. redisTemplate.opsForList()取得List类型命令API集合。
3. redisTemplate.opsForSet()取得Set类型命令API集合。
4. redisTemplate.opsForHash()取得Hash类型命令API集合。
5. redisTemplate.opsForZset()取得Zset类型命令API集合。

​	然后，在不同类型的命令API集合上，使用各种数据类型特有的API函数，完成具体的Redis API操作。

#### 11.5.4 使用RedisTemplate模板API完成CRUD的实践案例

​	封装完成了自己的CacheOperationService缓存管理服务后，可以注入到Spring的业务Service中，就可以完成缓存的CRUD操作了。

​	这里的业务类是UserServiceImplWithTemplate类，用于完成User实例缓存的CRUD。使用CacheOperationService后，就能非常方便地进行缓存的管理，同时，在进行POJO的查询时，能优先使用缓存数据，省去了数据库访问的时间。

```java
@Service
@CacheConfig(cacheNames = "userCache")
public class UserServiceImplWithAnno implements UserService {

    public static final String USER_UID_PREFIX = "'userCache:'+";

    /**
     * CRUD 之  新增/更新
     *
     * @param user 用户
     */
    @CachePut(key = USER_UID_PREFIX + "T(String).valueOf(#user.uid)")
    @Override
    public User saveUser(final User user) {
        //保存到数据库
        //返回值，将保存到缓存
        Logger.info("user : save to redis");
        return user;
    }

    /**
     * 带条件缓存
     *
     * @param user 用户
     * @return 用户
     */
    @CachePut(key = "T(String).valueOf(#user.uid)", condition = "#user.uid>1000")
    public User cacheUserWithCondition(final User user) {
        //保存到数据库
        //返回值，将保存到缓存
        Logger.info("user : save to redis");
        return user;
    }

    /**
     * CRUD 之   查询
     *
     * @param id id
     * @return 用户
     */
    @Cacheable(key = USER_UID_PREFIX + "T(String).valueOf(#id)")
    @Override
    public User getUser(final long id) {
        //如果缓存没有,则从数据库中加载
        Logger.info("user : is null");
        return null;
    }

    /**
     * CRUD 之 删除
     *
     * @param id id
     */

    @CacheEvict(key = USER_UID_PREFIX + "T(String).valueOf(#id)")
    @Override
    public void deleteUser(long id) {

        //从数据库中删除
        Logger.info("delete  User:", id);
    }

    /**
     * 删除userCache中的全部缓存
     */
    @CacheEvict(value = "userCache", allEntries = true)
    public void deleteAll() {

    }

    /**
     * 一个方法上，加上三类cache处理
     */
    @Caching(cacheable = @Cacheable(key = "'userCache:'+ #uid"),
             put = @CachePut(key = "'userCache:'+ #uid"),
             evict = {
                 @CacheEvict(key = "'userCache:'+ #uid"),
                 @CacheEvict(key = "'addressCache:'+ #uid"),
                 @CacheEvict(key = "'messageCache:'+ #uid")
             }
            )
    public User updateRef(String uid) {
        //....业务逻辑
        return null;
    }
}
```

​	在业务Service类使用CacheOperationService缓存管理之前，还需要在配置文件（这里为spring-redis.xml）中配置好依赖：

```xml
<!--将redisTemplate 封装成缓存service-->
<bean id="cacheOperationService" class="com.crazymakercircle.redis.springJedis.CacheOperationService">
    <property name="redisTemplate" ref="redisTemplate"/>
</bean>
<!--业务service,依赖缓存service-->
<bean id="serviceImplWithTemplate" class="com.crazymakercircle.redis.springJedis.UserServiceImplWithTemplate">
    <property name="cacheOperationService" ref="cacheOperationService"/>
</bean>
```

​	编写一个用例，测试一下UserServiceImplWithTemplate，运行之后，可以从Redis客户端输入命令来查看缓存的数据。至此，缓存机制已经成功生效，数据访问的时间可以从数据库的百毫秒级别缩小到毫秒级别，性能提升了100倍。

```java
public class SpringRedisTester {


    /**
     * 测试 直接使用redisTemplate
     */
    @Test
    public void testServiceImplWithTemplate() {
        ApplicationContext ac = new ClassPathXmlApplicationContext("classpath:spring-redis.xml");
        UserService userService =
            (UserService) ac.getBean("serviceImplWithTemplate");
        long userId = 1L;
        userService.deleteUser(userId);
        User userInredis = userService.getUser(userId);
        Logger.info("delete user", userInredis);
        User user = new User();
        user.setUid("1");
        user.setNickName("foo");
        userService.saveUser(user);
        Logger.info("save user:", user);
        userInredis = userService.getUser(userId);
        Logger.info("get user", userInredis);
    }
    //...其他测试用例
}
```

#### 11.5.5 使用RedisCallback回调完成CRUD的实践案例

​	前面讲到，RedisConnection连接类和RedisTemplate模板类都提供了整套Redis操作的API，只不过，它们的层次不同。**RedisConnection连接类更加底层，它负责二进制层面的Reids操作，Key、Value都是二进制字节数组。而RedisTemplate模板类，在RedisConnection的基础上，使用在spring-redis.xml中配置的序列化、反序列化的工具类，完成上层类型（如String、Object、POJO等类）的Redis操作**。

​	<u>如果不需要RedisTemplate配置的序列化、反序列化的工具类，或者由于其他的原因，需要直接使用RedisConnection去操作Redis，怎么办呢？可以使用RedisCallback的doInRedis回调方法，在doInRedis回调方法中，直接使用实参RedisConnection连接类实例来完成Redis的操作。</u>

​	当然，完成RedisCallback回调业务逻辑后，还需要使用RedisTemplate模板实例去执行，调用的是RedisTemplate.execute(ReidsCallback)方法。

​	通过RedisCallback回调方法实现CRUD的实例代码如下：

```java
public class UserServiceImplInTemplate implements UserService {

    public static final String USER_UID_PREFIX = "user:uid:";

    private RedisTemplate redisTemplate;

    public void setRedisTemplate(RedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    private static final long CASHE_LONG = 60 * 4;//4分钟

    /**
     * CRUD 之  新增/更新
     *
     * @param user 用户
     */
    @Override
    public User saveUser(final User user) {
        //保存到缓存
        redisTemplate.execute(new RedisCallback<User>() {

            @Override
            public User doInRedis(RedisConnection connection)
                throws DataAccessException {
                byte[] key = serializeKey(USER_UID_PREFIX + user.getUid());
                connection.set(key, serializeValue(user));
                connection.expire(key, CASHE_LONG);
                return user;
            }
        });
        //保存到数据库
        //...如mysql
        return user;
    }

    private byte[] serializeValue(User s) {
        return redisTemplate
            .getValueSerializer().serialize(s);
    }

    private byte[] serializeKey(String s) {
        return redisTemplate
            .getKeySerializer().serialize(s);
    }

    private User deSerializeValue(byte[] b) {
        return (User) redisTemplate
            .getValueSerializer().deserialize(b);
    }

    /**
     * CRUD 之   查询
     *
     * @param id id
     * @return 用户
     */
    @Override
    public User getUser(final long id) {
        //首先从缓存中获取
        User value =  (User) redisTemplate.execute(new RedisCallback<User>() {
            @Override
            public User doInRedis(RedisConnection connection)
                throws DataAccessException {
                byte[] key = serializeKey(USER_UID_PREFIX + id);
                if (connection.exists(key)) {
                    byte[] value = connection.get(key);
                    return deSerializeValue(value);
                }

                return null;
            }
        });
        if (null == value) {
            //如果缓存中没有，从数据库中获取
            //...如mysql
            //并且，保存到缓存
        }
        return value;
    }

    /**
     * CRUD 之 删除
     * @param id id
     */
    @Override
    public void deleteUser(long id) {
        //从缓存删除
        redisTemplate.execute(new RedisCallback<Boolean>() {
            @Override
            public Boolean doInRedis(RedisConnection connection)
                throws DataAccessException {
                byte[] key = serializeKey(USER_UID_PREFIX + id);
                if (connection.exists(key)) {
                    connection.del(key);
                }
                return true;
            }
        });
        //从数据库删除
        //...如mysql
    }

    /**
     * 删除全部
     */
    @Override
    public void deleteAll() {

    }
}
```

​	同样的，在使用UserServiceImplTemplate之前，也需要在配置文件（这里为spring-redis.xml）配置好依赖关系：

```xml
<bean id="serviceImplInTemplate" class="com.crazymakercircle.redis.springJedis.UserServiceImplInTemplate">
    <property name="redisTemplate" ref="redisTemplate"/>
</bean>
```

### 11.6 Spring的Redis缓存注释

​	前面讲的Redis缓存实现都是基于Java代码实现的。在Spring中，通过合理的添加缓存注解，也能实现和前面示例程序中一样的缓存功能。

​	为了方便地提供缓存能力，Spring提供了一组缓存注解。但是，这组注解不仅仅是针对Redis，它本质上并不是一种具体的缓存实现方案（例如Redis、EHCache等），而是对缓存使用的统一抽象。通过这组缓存注解，然后加上与具体缓存相互匹配的Spring配置，不用编码就可以快速达到缓存的效果。

​	下面先给大家展示一下Spring缓存注解的应用实例，然后对Spring cache的几个注解进行详细的介绍。

#### 11.6.1 使用Spring缓存注解完成CRUD的实践案例

​	这里简单介绍一下Spring的三个缓存注解：@CachePut、@CacheEvict、@Cacheable。这三个注解通常都加在方法的前面，大致的作用如下：

（1）@CachePut作用是设置缓存。先执行方法，并将执行结果缓存起来。

（2）@CacheEvict的作用是删除缓存。在执行方法前，删除缓存。

（3）@Cacheable的作用更多是查询缓存。首先检查注解中的Key键是否在缓存中，如果是，则返回Key的缓存值，不再执行方法；否则，执行方法并将方法结果缓存起来。从后半部分来看，@Cacheable也具备@CachePut的能力。

​	在展开介绍三个注释之前，先演示下它们的使用：用它们实现一个带缓存功能的用户操作UserService实现类，名为UserServiceImplWithAnno类。其功能和前面介绍的UserServiceImplWithTemplate类是一样的，只是这里使用注解去实现缓存，代码如下：

```java
@Service
@CacheConfig(cacheNames = "userCache")
public class UserServiceImplWithAnno implements UserService {

    public static final String USER_UID_PREFIX = "'userCache:'+";

    /**
     * CRUD 之  新增/更新
     *
     * @param user 用户
     */
    @CachePut(key = USER_UID_PREFIX + "T(String).valueOf(#user.uid)")
    @Override
    public User saveUser(final User user) {
        //保存到数据库
        //返回值，将保存到缓存
        Logger.info("user : save to redis");
        return user;
    }

    /**
     * 带条件缓存
     *
     * @param user 用户
     * @return 用户
     */
    @CachePut(key = "T(String).valueOf(#user.uid)", condition = "#user.uid>1000")
    public User cacheUserWithCondition(final User user) {
        //保存到数据库
        //返回值，将保存到缓存
        Logger.info("user : save to redis");
        return user;
    }


    /**
     * CRUD 之   查询
     *
     * @param id id
     * @return 用户
     */
    @Cacheable(key = USER_UID_PREFIX + "T(String).valueOf(#id)")
    @Override
    public User getUser(final long id) {
        //如果缓存没有,则从数据库中加载
        Logger.info("user : is null");
        return null;
    }

    /**
     * CRUD 之 删除
     *
     * @param id id
     */

    @CacheEvict(key = USER_UID_PREFIX + "T(String).valueOf(#id)")
    @Override
    public void deleteUser(long id) {

        //从数据库中删除
        Logger.info("delete  User:", id);
    }

    /**
     * 删除userCache中的全部缓存
     */
    @CacheEvict(value = "userCache", allEntries = true)
    public void deleteAll() {

    }


    /**
     * 一个方法上，加上三类cache处理
     */
    @Caching(cacheable = @Cacheable(key = "'userCache:'+ #uid"),
             put = @CachePut(key = "'userCache:'+ #uid"),
             evict = {
                 @CacheEvict(key = "'userCache:'+ #uid"),
                 @CacheEvict(key = "'addressCache:'+ #uid"),
                 @CacheEvict(key = "'messageCache:'+ #uid")
             }
            )
    public User updateRef(String uid) {
        //....业务逻辑
        return null;
    }

}
```

#### 11.6.2 spring-redis.xml中配置的调整

​	在使用Spring缓存注解前，首先需要配置文件中启用Spring对基类的Cache的支持：在spring-redis.xml中，加上\<cache:annotation-driven />配置项。

​	\<cache:annotation-driven />有一个cache-manager属性，用来指定所需要用到的缓存管理器（CacheManager）的Spring Bean的名称。如果不进行特别设置，默认的名称是CacheManager。也就是说，如果使用了\<cache:annotation-driven />，还需要配置一个名为CacheManager的缓存管理器Spring Bean，这个Bean，要求实现CacheManager接口。而<u>CacheManager接口是Spring定义的一个用来管理Cache缓存的通用接口。对应于不同的缓存，需要使用不同的CacheManager实现</u>。Spring自身已经提供了一种CacheManager的实现，是基于Java API的ConcurrentMap简单的内存Key-Value缓存实现。但是，这里需要使用的缓存是Redis，所以使用spring-data-redis包中的RedisCacheManager实现。

​	spring-reids.xml增加的配置项，具体如下：

```xml
<!-- 启用缓存注解功能，这个是必须的，否则注解不会生效 -->
<cache:annotation-driven/>
<!--自定义redis工具类,在需要缓存的地方注入此类  -->
<bean id="cacheManager" class="org.springframework.data.redis.cache.RedisCacheManager">
    <constructor-arg ref="redisTemplate"/>
    <constructor-arg name="cacheNames">
        <set>
            <!--声明userCache-->
            <value>userCache</value>
        </set>
    </constructor-arg>
</bean>
```

​	\<cache:annotation-driven />还可以指定一个mode属性，可选值有proxy和aspectj，<u>默认是使用proxy</u>。**当mode为proxy时，只有当缓存注解的地方被对象外部的方法调用时，Spring Cache才会发生作用，反过来说，如果一个缓存方法，被其所在对象的内部方法调用时，Spring Cache是不会发生作用的。而mode为aspectj模式时，就不会发生上面的情况，只有注解方法被内部调用，缓存才会生效；非public类型的方法上，也可以使用Spring Cache注解。**

​	\<cache:annotation-driven />还可以指定一个proxy-target-class属性，设置代理类的创建机制，有两个值：

（1）值为true，表示使用CGLib创建代理类。

（2）值为false，表示使用JDK的动态代理机制创建代理类，默认为false。

​	大家知道，**JDK的动态代理是利用反射机制生成一个实现代理接口的匿名类(Class-based-Proxies)，在调用具体方法前，通过调用InvokeHandler来调用实际的代理方法**。而使用CGLib创建代理类，则不同。**CGLib底层采用ASM开源.class字节码生成框架，生成字节码级别的代理类（Inter-based Proxies）。对比来说，在实际的运行时，CGLib代理类比使用Java反射代理类的效率更高。**

​	当proxy-target-class为true时，@Cacheable和@CacheInvalidate等注解，必须标记在具体类（Concrete Class）类上，不能标记在接口上，否则不会发生作用。当proxy-target-class为false时，@Cacheable和@CacheInvalidate等可以标记在接口上，也能发挥作用。

​	在配置RedisCacheManager缓存管理器Bean时，需要配置两个构造函数：

（1）redisTemplate模板Bean

（2）cacheNames缓存名称

​	但是，不同的spring-data-redis版本，构造函数不同，这里使用的是spring-data-redis的版本是1.4.3。对于2.0版本，在配置上发生了一些变化，但是原理大致是相同的，自行研究。

> [CGLIB(Code Generation Library) 介绍与原理 | 菜鸟教程](https://www.runoob.com/w3cnote/cglibcode-generation-library-intro.html)

#### 11.6.3 详解@CachePut和@Cacheable注解

​	简单来说，这两个注解都可以增加缓存，但是有细微的区别：

（1）@CachePut负责增加缓存。

（2）@Cacheable负责查询缓存，如果没有查到，则将执行方法，并将方法的结果增加到缓存。

​	下面我们呢将来详细介绍一下@CachePut和@Cacheable两个注解。

​	在支持Spring Cache的环境下，如果@CachePut加在方法上，每次执行方法后，会将结果存入指定缓存的Key键上，如下所示：

```java
/**
     * CRUD 之  新增/更新
     * @param user 用户
     */
@CachePut(key = USER_UID_PREFIX + "T(String).valueOf(#user.uid)")
@Override
public User saveUser(final User user) {
    //保存到数据库
    //返回值，将保存到缓存
    Logger.info("user : save to redis");
    return user;
}
```

​	大家知道，Redis缓存都是键-值对（Key-Value Pair）。Redis缓存中的Key键即为@CachePut注解配置的key属性值，一般是一个字符串，或者是结果为字符串的一个SpEL(StringEL)表达式。Redis缓存的Value值就是方法的返回结果，在经过序列化后所产生的序列化数据。

​	一般来说，可以给@CachePut设置三个属性，Value、Key和Condition

（1）value属性，指定Cache缓存的名字

​	value值表示当前Key键被缓存在哪个Cache上，对应于Spring配置文件中的CacheManager缓存管理器的cacheNames属性中配置的某个Cache名称，如userCache。可以配置一个Cache，也可以是多个Cache，当配置多个Cache时，value值是一个数组，如value={userCache,otherCache1,otherCache2....}。

​	<u>Value属性中的Cache名称，相当于缓存Key所属的命名空间。当使用@CacheEvict注解清除缓存时，可以通过合理配置清除指定Cache名称下的所有Key。</u>

（2）key属性，指定Redis的Key属性值

​	key属性，是用来指定Spring缓存方法的Key键，该属性支持SpringEL表达式。当没有指定该属性时，Spring将使用默认策略生成Key键。有关SpringEL表达式，稍后再详细介绍。

（3）condition属性，指定缓存的条件

​	并不是所有的函数结果都希望加入Redis缓存，可以通过condition属性来实现这一功能。condition属性值默认为空，表示将缓存所有的结果。可以通过SpringEL表达式设置，当表达式的值为true时，表示进行缓存处理；否则不进行缓存处理。如下示例程序表示只当user的id大于1000时，才会进行缓存，代码如下：

```java
@CachePut(key = "T(String).valueOf(#user.uid)", condition = "#user.uid>1000")
public User cacheUserWithCondition(final User user){
    // 保存到数据库
    // 返回值将保存到缓存
    Logger.info("user : save to redis");
    return user;
}
```

​	再来看一看@Cacheable注解，主要是查询缓存。

​	<u>对于加上了@Cacheable注解的方法，Spring在每次执行前会检查Redis缓存中是否存在相同的Key键，如果存在，就不再直接该方法，而是直接从缓存中获取结果并返回；如果不存在，才会执行方法，并将返回结果存入Redis缓存中。</u>与@CachePut注解一样，@Cacheable也具备增加缓存的能力。

​	@Cacheable与@CachePut不同之处的是：@Cacheable只有当Key键在Redis缓存不存在时，才会执行方法，将方法的结果缓存起来；如果Key键在Redis缓存中存在，则直接返回缓存结果。而加了@CachePut注解的方法，则缺少了检查的环节：<u>@CachePut在方法执行前不去进行缓存检查，无论之前是否有缓存，都会将新的执行结果加入到缓存中。</u>

​	使用@Cacheable注解，一般也能指定三个属性：value、key和condition。三个属性的配置方法和@CachePut的三个属性的配置方法也是一样的，不再赘述。

​	**@CachePut和@Cacheable注解也可以标注在类上，表示所有的方法都具缓存处理的功能。但是这种情况，用得比较少。**

#### 11.6.4 详解@CacheEvict注解

​	注解@CacheEvict主要用来清除缓存，可以指定的属性有value、key、condition、allEntries和beforeInvocation。其中value、key和condition的语义与@Cacheable对应的属性类似。value表示清除哪些Cache（对应Cache的名称）；key表示清除哪个Key键；condition 表示清除的条件。下面主要看一下两个属性allEnries和beforeInvocation。

​	（1）allEntries属性：表示是否全部清空

​	allEntries表示是否需要清除缓存中的所有Key键，是boolean类型，默认为false，表示不需要清除全部。当指定了allEntries为true时，表示清空value名称属性所指向的Cache中所有的缓存，这时候，所配置的key属性值已经没有意义，将被忽略。allEntries为true，用于需要全部清空某个Cache的场景，这比一个一个地清除Key键，效率更高。

​	在下面的例子中，一次清理Cache名称为userCache中的所有Redis缓存，代码如下：

```java
@Service
@CacheConfig(cacheNames = "userCache")
public class UserServiceImplWithAnno implements UserService {
    //... 省略其他内容

    /**
     * 删除userCache名字空间的全部缓存
     */
    @CacheEvict(value = "userCache", allEntries = true)
    public void deleteAll() {

    }
}
```

​	（2）beforeInvocation属性：表示是否在方法执行前执行操作缓存

​	一般情况下，是在对应方法成功执行之后，在触发清除操作。但是，如果方法执行过程中，有异常抛出，或者由于其他的原因，导致线程终止，就不会触发清除操作。所以，通过设置beforeInvocation属性来确保清理。

​	beforeInvocation属性是boolean属性，当设置为true时，可以改变触发清除操作的次序，Spring会在执行注解的方法之前完成缓存的清理工作。

​	<u>最后说明一下：注解@CacheEvict，除了加在方法上，还可以加在类上。当加在一个类上时，表示该类所有的方法都会触发缓存清除，一般情况下，很少这样使用。</u>

#### 11.6.5 详解@Caching组合注解

​		**@Caching注解，是一个缓存处理的组合注解。通过@Caching，可以一次指定多个Spring Cache注解的组合。**@Caching注解拥有三个属性：cacheable、put和evict。

​		@Caching的组合能力，主要通过三个属性完成，具体如下：

（1）cacheable属性：用于指定一个或者多个@Cacheable注解的组合，可以指定一个，也可以指定多个，如果指定多个@Cacheable注解，则直接使用数组的形式，即使用花括号，将多个@Cacheable注解包围起来。用于查询一个或者多个key的缓存，如果没有，则按照条件将结果加入缓存。

（2）put属性：用于指定一个或者多个@CachePut注解的组合，可以指定一个，也可以指定多个，用于设置一个或多个key的缓存。如果指定多个@CachePut注解，则直接使用数组的形式。

（3）evict属性：用于指定一个或者多个@CacheEvict注解的组合，可以指定一个，也可以指定多个，用于删除一个或多个key的缓存。如果指定多个@CacheEvict注解，则直接使用数组的形式。

​	**在数据库中，往往需要进行外键的级联删除：在删除一个主键时，需要将一个主键的所有级联的外键，通通删除掉。如果外键都进行了缓存，在级联删除时，则可以使用@Caching注解，组合多个@CacheEvict注解，在删除主键缓存时，删除所有的外键缓存。**

​	下面有一个简单的示例，模拟在更新一个用户时，需要删除与用户关联的多个缓存：用户信息、地址信息、用户的消息等等。

​	使用@Caching注解，为各个方法加上一大票缓存注解，具体如下：

```java
/**
 * 一个方法上，加上三类cache处理
 */
@Caching(cacheable = @Cacheable(key = "'userCache:'+ #uid"),
         put = @CachePut(key = "'userCache:'+ #uid"),
         evict = {
             @CacheEvict(key = "'userCache:'+ #uid"),
             @CacheEvict(key = "'addressCache:'+ #uid"),
             @CacheEvict(key = "'messageCache:'+ #uid")
         }
        )
public User updateRef(String uid) {
    //....业务逻辑
    return null;
}
```

​	以上示例程序仅仅是一个组合注解的演示。@Caching有cacheable、put、evict三大类型属性，在实际使用时，可以进行类型的灵活裁剪。例如，实际的开发场景并不需要添加缓存，完全可以不给@Caching注解配置cacheable属性。

​	至此，缓存注解已经介绍完毕。注解中需要用到SpEL表达式。

### 11.7 详解SpringEL（SpEL）

​	Spring表达式语言全称为"Spring Expression Language"，缩写为"SpEL"。SpEL提供一种强大、简洁的Spring Bean的动态操作表达式。**SpEL表达式可以在运行期间执行，表达式的值可以动态装配到Spring Bean属性或者构造函数中，表达式可以调用Java静态方法，可以访问Properties文件中的配置值等等，SpringEL能与Spring功能完美整合，给静态Java语言增加了动态功能**。

​	大家知道，JSP页面的表达式使用${}进行声明。而SpringEL表达式使用#{}进行声明。SpEL支持如下的表达式：

1. 基本表达式：字面量表达式、关系、逻辑与算数运算表达式、字符串连接及截取表达式、三目运算及Elivis表达式、正则表达式、括号优先级表达式。
2. 类型表达式：类型访问、静态方法/属性访问、实例访问、实例属性值获取、实例属性导航、instanceof、变量定义及引用、赋值表达式、自定义函数等等。
3. 集合相关表达式：内联列表、内联数组、集合，字典访问、列表、字典，数组修改、集合投影、集合选择；不支持多位内联数组初始化；不支持内联字典定义；
4. 其他表达式：模板表达式。

#### 11.7.1 SpEL运算符

​	SpEL基本表达式是由各种基础运算符、常量、变量引用一起进行组合所构成的表达式。基础的运算符主要包括：算数运算符、关系运算符、逻辑运算符、字符串运算符、三目运算符、正则表达式匹配符、类型运算符、变量引用符等。

1. 算数运算符：SpEL提供了以下算数运算符：如加（+）、减（-）、乘（*）、除（/）、求余（%）、幂（\^）、求余（MOD）和除（DIV）等算数运算符。MOD与"%"等价，DIV与"/"等价，并且**不区分大小写**。例如，#{1+2\*3/4-2}、#{2\^3}、#{100mod9}都是算数运算SpEL表达式。
2. 关系运算符：SpEL提供了以下关系运算符：等于（==）、不等于（！=）、大于（>）、大于等于（>=）、小于（<）、小于等于（<=），区间（between）运算等等。例如：#{2>3}值为false。

3. 逻辑运算符：SpEL提供了以下逻辑运算符：与（and）、或（or）、非（！或者NOT）。例如：#{2>3 or 4>3} 值为true。与Java逻辑运算不同，SqEL不支持"&&"和"||"。

4. 字符串运算符：SpEL提供了以下字符串运算符：连接（+）和截取（\[\]）。例如：#{\'Hello\' + \'World!\'}的结果为"Hello World"。#{\'Hello World！\'[0]}截取第一个字符"H"，**目前只支持获取一个字符**。

5. 三目运算符：SpEL提供了和Java一样的三目运算符："逻辑表达式 ？ 表达式1：表达式2"。例如：#{3>4？\'Hello\'：\'World\'}将返回’World‘。

6. 正则表达式匹配符：SpEL提供了字符串的正则表达式匹配符：matches。例如：#{\'123\'matches\'\\d{3}\'}返回true。

7. **类型访问运算符**：**SpEL提供了一个类型访问运算符："T（Type）"，"Type"表示某个Java类型，实际上对应于Java类java.lang.Class实例。**"Type"必须是类的全限定名（包括包名），但是核心包"java.lang"中的类除外。也就是说，<u>"java.lang"包下的类，可以不用指定完整的包名</u>。例如：T（String）表示访问的是java.lang.String类。#{T(String).valueOf(1)}，表示将整数1转换成字符串。

8. **变量引用符：SpEL提供了一个上下文变量的引用符"#"，在表达式中使用"#variableName"引用上下文变量。**

   <u>SpEL提供了一个变量定义的上下文接口——EvaluationContext，并且提供了标准的上下文实现——StandardEvaluationContext</u>。通过EvaluationContext接口的setVariable(variableName，value)方法，可以定义"上下文变量"，这些变量在表达式中采用"#variableName"的方式予以引用。**在创建变量上下文Context实例时，还可以在构造器参数中设置一个rootObject作为根，可以使用"#root"引用对象，也可以使用"#this"引用跟对象**。

​	下面使用前面介绍的运算符定义几个SpEL表达式，示例程序如下：

```java
@Component
@Data
public class SpElBean {
    /**
     * 算术运算符
     */
    @Value("#{10+2*3/4-2}")
    private int algDemoValue;


    /**
     * 字符串运算符
     */
    @Value("#{'Hello ' + 'World!'}")
    private String stringConcatValue;

    /**
     * 类型运算符
     */
    @Value("#{ T(java.lang.Math).random() * 100.0 }")
    private int randomInt;


    /**
     * 展示SpEl 上下文变量
     */
    public void showContextVar() {
        ExpressionParser parser = new SpelExpressionParser();
        EvaluationContext context = new StandardEvaluationContext();
        context.setVariable("foo", "bar");
        String foo = parser.parseExpression("#foo").getValue(context, String.class);
        Logger.info(" foo:=", foo);

        context = new StandardEvaluationContext("I am root");
        String root = parser.parseExpression("#root").getValue(context, String.class);
        Logger.info(" root:=", root);

        String result3 = parser.parseExpression("#this").getValue(context, String.class);
        Logger.info(" this:=", root);
    }

}
```

​	以上示例程序代码的测试用例如下：

```java
public class SpringRedisTester {
    /**
     * 测试   SpEl 表达式
     */
    @Test
    public void testSpElBean() {
        ApplicationContext ac = new ClassPathXmlApplicationContext("classpath:spring-redis.xml");
        SpElBean spElBean =
            (SpElBean) ac.getBean("spElBean");

        /**
         * 演示算术运算符
         */
        Logger.info(" spElBean.getAlgDemoValue():="
                    , spElBean.getAlgDemoValue());

        /**
         * 演示 字符串运算符
         */
        Logger.info(" spElBean.getStringConcatValue():="
                    , spElBean.getStringConcatValue());

        /**
         * 演示 类型运算符
         */
        Logger.info(" spElBean.getRandomInt():="
                    , spElBean.getRandomInt());

        /**
         * 展示SpEl 上下文变量
         */
        spElBean.showContextVar();

    }
}
```

​	**一般来说，SpringEL表达式使用#{}进行声明。但是，不是所有注解中的SpringEL表达式都需要#{}进行声明。例如，@Value注解中的SpringEL表达式需要#{}进行声明；而ExpressionParser.parseExpression实例方法中的SpringEL表达式不需要#{}进行声明；另外，@CachePut和@Cacheable等缓存注解中的key属性值的SpringEL表达式，也不需要#{}进行声明。**

#### 11.7.2 缓存注解中的SpringEL表达式

​	对应于加在方法上的缓存注解（如@CachePut和@Cacheable），spring提供了专门的上下文类CacheEvaluationContext，这个类继承于基础的方法注解上下文MethodBasedEvaluationContext，而这个方法则继承于StandardEvaluationContext（大家熟悉的标准注解上下文）。

​	CacheEvaluationContext的构造器如下：

```java
class CacheEvaluationContext extends MethodBasedEvaluationContext {
    // 构造器
    CacheEvaluationContext(Object rootObject,//根对象
                          Method method,//当前方法
                          Object[] arguments,//当前方法的参数
                          ParameterNameDiscoverer parameterNameDiscoverer)
    {
        super(rootObject, method, arguments, parameterNameDiscoverera);
    }
    //...省略其他方法
}
```

​	在配置缓存注解（如@CachePut）的Key时，可以用到CacheEvaluationContext的rootObject根对象。通过该根对象，可以获取到如表所示的属性。

| 属性名称    | 说明                                                         | 示例                                                 |
| ----------- | ------------------------------------------------------------ | ---------------------------------------------------- |
| methodName  | 当前被调用的方法名                                           | 获取当前被调用的方法名：#root.methodName             |
| Method      | 当前被调用的方法                                             | 获取当前被调用的方法：#root.method.name              |
| Target      | 当前被调用的目标对象                                         | 当前被调用的目标对象：#root.target                   |
| targetClass | 当前按被调用的目标对象类                                     | 当前被调用的目标对象类型：#root.targetClass          |
| Args        | 当前被调用的方法的参数列表                                   | 当前被调用的方法的第0个参数：#root.args[0]           |
| Caches      | 当前方法调用使用的缓存之列表，如：@Cacheable(value={"cache1"，"cache2"}),则有两个cache | 当前被调用方法的第0个cache名称：#root.caches[0].name |

​	**在配置key属性时，如果用到SpEL表达式root对象的属性，也可以将"#root"省略，因为Spring默认使用的就是root对象的属性**。如：

```java
@Cacheable(value={"cache1"，"cache2"}, key="caches[1].name")
public User find(User user){
    // ...省略：查询数据库的代码
}
```

​	在SpEL表达式中，处理访问SpEL表达式root对象，还可以访问当前方法的参数以及它们的属性，访问方法的参数有以下两种形式：

（1）**方式一：使用"#p 参数index"形式访问方法的参数**

​	展示使用"#p 参数 index"形式访问arguments参数的示例程序：

```java
//访问第0个参数，参数id
@Cacheable(value="users", key="#p0")
public User find(String id){
    // ...省略：查询数据库的代码
}
```

​	下面的示例程序中访问参数的属性，这里是参数user的id属性，具体如下：

```java
//访问参数user的id属性
@Cacheable(value="users", key="#p0.id")
public User find(User user){
    // ...省略：查询数据库的代码
}
```

（2）**方式二：使用"#参数名"形式访问方法的参数**

​	可以使用"#参数名"的形式直接访问方法的参数。例如，使用"#user.id"的形式访问参数user的id属性，代码如下：

```java
//访问参数user的id属性
@Cacheable(value="users", key="#user.id")
public User find(User user){
    // ...省略：查询数据库的代码
}
```

​	通过对比可以看出，<u>在访问方法的参数以及参数的属性时，使用方式二"#参数名"的形式，比方式一"#p 参数index"的形式更加直观</u>。

### 11.8 本章小结

​	本章介绍了Redis的安装、客户端的使用、Jedis编程。对于如何使用spring-data-redis操作缓存进行了详细的介绍。同时介绍了如何使用Spring的缓存注解，节省编程使用缓存的编码工作量。

​	本章重点内容是缓存操作的一个全面的知识基础，对大家的实际开发，有指导作用，尤其目前的市面上，还没有一本书具体介绍spring-data-redis和缓存注解的使用。然而，由于篇幅的原因，本章的内容没有涉及到Redis的高端架构和开发。包括如何搭建高可用、高性能的Redis集群，以及如何在Java中操作高可用、高性能的Redis集群等等。后续"疯狂创客圈"将结合本书，提供一些更加高端的教学视频，将这方面的内容呈现给大家，尽可能为大家的开发、面试等尽一份绵薄之力。

## 第12章 亿级高并发IM架构的开发实践

​	本章结合分布式缓存Redis、分布式协调ZooKeeper、高性能通信Netty，从架构的维度，设计一套亿级IM通信的高并发应用方案。并从学习和实战的角度出发，将联合"疯狂创客圈"社群的高性能发烧友们，一起持续迭代出一个支持亿级流量的IM项目，暂时命名为"CrazyIM"。

### 12.1 如何支持亿级流量的高并发IM架构的理论基础

​	支持亿级流量的高并发IM通信，需要用到Netty集群、ZooKeeper集群、Redis集群、MySql集群、SpringCloud WEB服务集群、RocketMQ消息队列集群等等。

#### 12.1.1 亿级流量的系统架构的开发实践

​	支持亿级流量的高并发IM通信的几大集群中，最为核心的是Netty集群、ZooKeeper集群、Redis集群，它们是主要实现亿级流量通信功能不可缺少的集群。其次是SpringCloud WEB 服务集群、MySql集群，完成海量用户的登录和存储，以及离线消息的存储。最后是RocketMQ消息队列集群，用于离线消息的保存。

​	主要的集群介绍如下：

（1）Netty服务集群

​	主要用来负责维持和客户端的TCP连接，完成消息的发送和转发。

（2）ZooKeeper集群

​	负责Netty Server集群的管理，包括注册、路由、负载均衡。集群IP注册和节点ID分配。主要在基于ZooKeeper集群提供底层服务。

（3）Redis集群

​	负责用户、用户绑定关系、用户群组关系、用户远程会话等等数据的缓存。缓存其他的配置数据或者临时数据，加快读取速度。

（4）MySql集群

​	保存用户、群组、离线消息等。

（5）RocketMQ消息队列集群

​	主要是将优先级不高的操作，从高并发模式转成低并发的模式。例如，可以将离线消息发向消息队列，然后通过低并发的异步任务保存到数据库。

​	上面的架构是"疯狂创客圈"高性能社群的"CrazyIM"学习项目的架构，并且只是涉及核心功能，并不是实践开发亿级流量系统架构的全部。从迭代的角度来看，还有很多的完善的空间，"疯狂创客圈"高性能社群将持续对"CrazyIM"高性能项目的架构和实现，进行不断的更新和迭代，所以最终的架构图和实现，以最后的版本为准。

​	理论上来说，以上集群具备完全的扩展能力，进行合理的横向扩展和局部的优化，支持亿级流量是没有任何问题的。为什么这么说呢？

​	单体的Netty服务器，远远不止支持10万个并发，在CPU、内存还不错的情况下，如果配置得当。甚至能撑到100万级别的并发。通过合理的高并发架构，能够让系统动态扩展到成百上千的Netty节点，支持亿级流量是没有任何问题的。

​	至于如何通过配置，让单体的Netty服务器支撑100万高并发，请查询疯狂创客圈社群的文章《Netty100万级高并发服务器配置》

#### 12.1.2 高并发架构的技术选型

​	明确了架构之后，接下来就是平台的计数选型，大致如下：

（1）核心

Netty4.x+spring4.x+ZooKeeper3.x+redis3.x+rocketMQ3.x+mysql5.x+mongo3.x

（2）短连接服务：spring cloud

基于RESTful短链接的分布式微服务架构，完成用户在线管理、单点登录系统

（3）长连接服务：Netty

主要用来负责维护和客户端的TCP连接，完成消息的发送和转发

（4）消息队列：rocketMQ高速消息队列

（5）数据库：mysql+mongodb

**mysql用来存储结构化数据，如用户数据。mongodb很重要，用来存储非结构化的离线消息。**

（6）序列化协议：Protobuf+JSON

**Protobuf是最高效的二进制序列化协议，用于长连接。JSON是最紧凑的文本协议，用于短链接。**

#### 12.1.3 详解IM消息的序列化协议选型

​	<u>IM系统的客户端和服务器节点之间，需要按照同一种数据序列化协议进行数据的交换。简而言之：就是规定网络中的字节流数据，如何与应用程序需要的结构化数据相互转换。</u>

​	序列化协议主要的工作有两部分，结构化数据到二进制数据的序列化和反序列化。**序列化协议的类型：文本协议和二进制协议。**

​	常见的文本协议包括XML和JSON。文本协议序列化之后，可读性好，便于调试，方便扩展。但文本协议的缺点在于解析效率一般，有很多的冗余数据，这一点主要体现在XML格式上。

​	常见的二进制协议包括Protobuf、Thrift，这些协议都自带了数据压缩，编解码效率高，同时兼具扩展性。二进制协议的优势很明显，但是劣势也非常的突出。二进制协议和文本协议相反，序列化之后的二进制协议报文数据，基本上没有什么可读性，很显然，这点不利于大家开发和调试。

​	因此，在协议的选择上，给大家的建议是：

+ **对于并发度不高的IM系统，建议使用文本协议，例如JSON。**
+ **对于并发度非常之高，QPS在千万级、亿级的通信系统，尽量选择二进制的协议。**

​	"疯狂创客圈"社群持续迭代的"CrazyIM"项目，序列化协议选择的是Protobuf二进制协议，以便于容易达到对亿级流量的支持。

#### 12.1.4 详解长连接和短连接

​	<u>什么是长连接呢？客户端向服务器发起连接，服务器接受客户端的连接，双方建立连接。客户端与服务器完成一次读写之后，它们的连接并不会主动关闭，后续的读写操作会继续使用这个连接。</u>

​	大家知道，TCP协议的连接过程比较烦琐，建立连接是需要三次握手的，而释放连接则需要4次握手，所以说每个连接的建立都需要消耗资源和时间。

​	在高并发的IM系统中，客户端和服务器之间，需要大量的发送通信的消息，如果每次发送消息，都去建立连接，客户端的和服务器的连接建立和断开的开销是非常巨大的。所以，IM消息的发送，肯定是需要长连接的。

​	<u>什么是短连接呢？客户端向服务器发起连接，服务器接收客户端连接，在三次握手之后，双方建立连接。客户端与服务器完成一次读写，发送数据包并得到返回的结果之后，通过客户端和服务器的四次握手断开连接。</u>

​	短连接适用于数据请求频度较低的应用场景。例如网站的浏览和普通的Web请求。短连接的优点是，管理起来比较简单，存在的连接都是有用的连接，不需要额外的控制手段。

​	在高并发的IM系统中，客户端和服务器之间，除了消息的通信外，还需要用户的登录与认证、好友的更新与获取等等一些低频的请求，这些都使用短连接来实现。

​	综上所述，<u>在这个高并发IM系统中，存在两类的服务器。一类短连接服务器和一个长连接服务器。</u>

​	**<u>短连接服务器也叫做Web服务器</u>，主要功能是实现用户的登录鉴权和拉取好友、群组、数据档案等相对低频的请求操作**。

​	**<u>长连接服务器也叫做IM即时通信服务器</u>，主要作用就是用来和客户端建立并维护长连接，实现消息的传递和即时的转发**。并且，**分布式网络非常复杂，长连接管理是重中之重，需要考虑到连接的保活、连接检测、自动重连等方方面面的工作**。

​	<u>短连接Web服务器和长连接IM服务器之间，是相互配合的。在分布式集群的环境下，用户首先通过短连接登录Web服务器。Web服务器在完成用户的账号/密码验证，返回uid和token时，还需通过一定策略，获取目标IM服务器的IP地址和端口号列表，并返回给客户端。客户端开始连接IM服务器，连接成功后，发送鉴权请求，鉴权成功则授权的长连接正式建立。</u>

​	如果用户规模庞大，无论是短连接Web服务器，还是长连接IM服务器，都需要进行横向的扩展，都需要扩展到上十台、百台、甚至上千台服务器。只有这样，才能有良好性能的用户体验。因此，需要引入一个新的角色，短连接Web网关（WebGate）。

​	**WebGate短连接网关的职责，首先是代理大量的Web服务器，从而无感知地实现短连接的高并发。在客户端登录时和进行其他短连接时，不直接连接Web服务器，而是连接Web网关**。<u>围绕Web网关和Web高并发的相关技术，目前非常成熟，可以使用SpringCloud或者Dubbo等分布式Web技术，也很容易扩展。</u>

​	<u>除此之外，大量的IM服务器，又如何协同和管理呢？基于ZooKeeper或者其他的分布式协调中间件，可以非常方便、轻松地实现一个IM服务器集群的管理，包括而且不限于命名服务、服务注册、服务发现、负载均衡等管理。</u>

​	当用户登录成功的时候，WebGate短连接网关可以通过负载均衡技术，从ZooKeeper集群中，找出一个可用的IM服务器的地址，返回给用户，让用户建立长连接。

### 12.2 分布式IM的命名服务的实践案例

​	前面提到，一个高并发系统是由很多的节点所组成，而且节点的数量是不断动态变化的。在一个即时消息（IM）通信系统中，从0到1到N，用户量可能会越来越多，或者说由于某些活动影响，会不断出现流量洪流。这时需要动态加入大量的节点。另外，由于服务器或者网络的原因，一些节点主动离开了集群。如何为大量的动态节点命名呢？最好的方法是使用分布式命名服务，按照一定的规则，为动态上线和下线的工作节点命名。

​	疯狂创客圈的高并发"CrazyIM"实战学习项目，基于ZooKeeper构建分布式命名服务，为每一个IM工作服务器节点动态命名。

#### 12.2.1 IM节点的POJO类

​	首先定义一个POJO类，保存IM Worker节点的基础信息如Netty服务IP、Netty服务端口，以及Netty的服务连接数。具体如下：

```java
@Data
public class ImNode implements Comparable<ImNode>, Serializable {

    private static final long serialVersionUID = -499010884211304846L;


    //worker 的Id,zookeeper负责生成
    private long id;

    //Netty 服务 的连接数
    private Integer balance = 0;

    //Netty 服务 IP
    private String host;

    //Netty 服务 端口
    private Integer port;

    public ImNode() {
    }

    public ImNode(String host, Integer port) {
        this.host = host;
        this.port = port;
    }


    @Override
    public String toString() {
        return "ImNode{" +
            "id='" + id + '\'' +
            "host='" + host + '\'' +
            ", port='" + port + '\'' +
            ",balance=" + balance +
            '}';
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        ImNode node = (ImNode) o;
        //        return id == node.id &&
        return Objects.equals(host, node.host) &&
            Objects.equals(port, node.port);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, host, port);
    }

    /**
     * 用来按照负载升序排列
     */
    public int compareTo(ImNode o) {
        int weight1 = this.balance;
        int weight2 = o.balance;
        if (weight1 > weight2) {
            return 1;
        } else if (weight1 < weight2) {
            return -1;
        }
        return 0;
    }


    public void incrementBalance() {
        balance++;
    }

    public void decrementBalance() {
        balance--;
    }
}
```

​	这个POJO类的IP、端口、balance负载和每一个节点的Netty服务器相关。而id属性，则利用ZooKeeper中的ZNode子节点可以顺序编号的性质，由ZooKeeper生成。

#### 12.2.2 IM节点的ImWorker类

​	节点的命名服务的思路是，所有的工作节点都在ZooKeeper的同一个父节点下，创建顺序节点。然后从返回的临时路径上，取得属于自己的那个后缀的编号。主要的代码如下：

```java
public class ImWorker {

    //Zk curator 客户端
    private CuratorFramework client = null;

    //保存当前Znode节点的路径，创建后返回
    private String pathRegistered = null;

    private ImNode localNode = null;

    private static ImWorker singleInstance = null;

    //取得单例
    public static ImWorker getInst() {

        if (null == singleInstance) {

            singleInstance = new ImWorker();
            singleInstance.client =
                ZKclient.instance.getClient();
            singleInstance.localNode = new ImNode();
        }
        return singleInstance;
    }

    private ImWorker() {

    }

    // 在zookeeper中创建临时节点
    public void init() {

        createParentIfNeeded(ServerConstants.MANAGE_PATH);

        // 创建一个 ZNode 节点
        // 节点的 payload 为当前worker 实例

        try {
            byte[] payload = JsonUtil.object2JsonBytes(localNode);

            pathRegistered = client.create()
                .creatingParentsIfNeeded()
                .withMode(CreateMode.EPHEMERAL_SEQUENTIAL)
                .forPath(ServerConstants.PATH_PREFIX, payload);

            //为node 设置id
            localNode.setId(getId());

        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    public void setLocalNode(String ip, int port) {
        localNode.setHost(ip);
        localNode.setPort(port);
    }

    /**
     * 取得IM 节点编号
     *
     * @return 编号
     */
    public long getId() {

        return getIdByPath(pathRegistered);

    }

    /**
     * 取得IM 节点编号
     *
     * @return 编号
   * @param path  路径
     */
    public long getIdByPath(String path) {
        String sid = null;
        if (null == path) {
            throw new RuntimeException("节点路径有误");
        }
        int index = path.lastIndexOf(ServerConstants.PATH_PREFIX);
        if (index >= 0) {
            index += ServerConstants.PATH_PREFIX.length();
            sid = index <= path.length() ? path.substring(index) : null;
        }

        if (null == sid) {
            throw new RuntimeException("节点ID获取失败");
        }

        return Long.parseLong(sid);

    }


    /**
     * 增加负载，表示有用户登录成功
     *
     * @return 成功状态
     */
    public boolean incBalance() {
        if (null == localNode) {
            throw new RuntimeException("还没有设置Node 节点");
        }
        // 增加负载：增加负载，并写回zookeeper
        while (true) {
            try {
                localNode.incrementBalance();
                byte[] payload = JsonUtil.object2JsonBytes(localNode);
                client.setData().forPath(pathRegistered, payload);
                return true;
            } catch (Exception e) {
                return false;
            }
        }

    }

    /**
     * 减少负载，表示有用户下线，写回zookeeper
     *
     * @return 成功状态
     */
    public boolean decrBalance() {
        if (null == localNode) {
            throw new RuntimeException("还没有设置Node 节点");
        }
        while (true) {
            try {

                localNode.decrementBalance();

                byte[] payload = JsonUtil.object2JsonBytes(localNode);
                client.setData().forPath(pathRegistered, payload);
                return true;
            } catch (Exception e) {
                return false;
            }
        }

    }

    /**
     * 创建父节点
     *
     * @param managePath 父节点路径
     */
    private void createParentIfNeeded(String managePath) {

        try {
            Stat stat = client.checkExists().forPath(managePath);
            if (null == stat) {
                client.create()
                    .creatingParentsIfNeeded()
                    .withProtection()
                    .withMode(CreateMode.PERSISTENT)
                    .forPath(managePath);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

    }


    /**
     * 返回本地的节点信息
     *
     * @return 本地的节点信息
     */
    public ImNode getLocalNodeInfo() {
        return localNode;
    }

}
```

​	注意，这里有三个ZNode相关的路径：

（1）MANAGE_PATH

（2）PATH_PREFIX

（3）pathRegistered

​	第一个MANAGE_PATH是一个常量，值为"/im/nodes"，<u>为所有Worker临时工作节点的父亲节点的路径</u>，在创建Worker节点之前，首先要检查一下，父亲ZNode节点是否存在，否则的话，先创建父亲节点。<u>"/im/nodes"父亲节点的创建方式是，持久化节点，而不是临时节点</u>。ps：(10.3.5提过，临时节点下面不能创建子节点)

​	第二路径PATH_PREFIX是所有临时节点的前缀，值为MANAGE_PATH + "/seq-"，即"/im/nodes/seq-"。通常可以是在工作路径后，加上一个"/"分隔符，也可以是在工作路径的后面，加上"/"分隔符和其他的前缀字符，如"/im/nodes/id-"、"/im/nodes/seq-"等等。

​	第三路径pathRegistered是临时节点创建成功之后，返回的完整路径。例如：/im/nodes/seq-0000000000，/im/nodes/seq-0000000001等等。后面的编号是顺序的。

​	创建节点成功后，截取后边的编号数字，放在POJO对象的id属性中供后边使用：

```java
//为node设置id
node.setId(getId());
```

### 12.3 Worker集群的负载均衡之实践案例

​	**理论上来说，负载均衡是一种手段，用来把对某种资源的访问分摊给不同的服务器，从而减轻单点的压力。在高并发的IM系统中，负载均衡就是需要将IM长连接分摊给不同的Netty服务器，防止单个Netty服务器负载过大，而导致其不可用**。

​	前面讲到，当用户登录成功的时候，短连接网关WebGate需要返回给用户一个可用的Netty服务器地址，让用户来建立Netty长连接。而每台Netty工作服务器在启动时，都会去ZooKeeper的"/im/nodes"节点下注册临时节点。

​	因此，<u>短连接网关WebGate可以在用户登录成功之后，去"/im/nodes"节点下面取得所有可用的Netty服务器列表，并通过一定的负载均衡算法计算得出一台Netty工作服务器，并且返回给客户端</u>。

#### 12.3.1 ImLoadBalance负载均衡器

​	短连接网关WebGate如何获得最佳的Netty服务器呢？需要通过查询ZooKeeper集群来实现。定义一个负载均衡器ImLoadBalance类，将计算最佳Netty服务器的算法，放在负载均衡器中，ImLoadBalance的代码，大致如下：

```java
@Data
@Slf4j
@Service
public class ImLoadBalance {

    //Zk客户端
    private CuratorFramework client = null;
    //工作节点的路径
    private String managerPath;

    public ImLoadBalance() {
        this.client = ZKclient.instance.getClient();
        //        managerPath=ServerConstants.MANAGE_PATH+"/";
        managerPath=ServerConstants.MANAGE_PATH;
    }

    /**
     * 获取负载最小的IM节点
     *
     * @return
     */
    public ImNode getBestWorker() {
        List<ImNode> workers = getWorkers();

        log.info("全部节点如下：");
        workers.stream().forEach(node -> {
            log.info("节点信息：{}", JsonUtil.pojoToJson(node));
        });
        ImNode best = balance(workers);

        return best;
    }

    /**
     * 按照负载排序
     *
     * @param items 所有的节点
     * @return 负载最小的IM节点
     */
    protected ImNode balance(List<ImNode> items) {
        if (items.size() > 0) {
            // 根据balance值由小到大排序
            Collections.sort(items);

            // 返回balance值最小的那个
            ImNode node = items.get(0);

            log.info("最佳的节点为：{}", JsonUtil.pojoToJson(node));
            return node;
        } else {
            return null;
        }
    }


    /**
     * 从zookeeper中拿到所有IM节点
     */
    protected List<ImNode> getWorkers() {

        List<ImNode> workers = new ArrayList<ImNode>();

        List<String> children = null;
        try {
            children = client.getChildren().forPath(managerPath);
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }

        for (String child : children) {
            log.info("child:", child);
            byte[] payload = null;
            try {
                payload = client.getData().forPath(managerPath+"/"+child);

            } catch (Exception e) {
                e.printStackTrace();
            }
            if (null == payload) {
                continue;
            }
            ImNode worker = JsonUtil.jsonBytes2Object(payload, ImNode.class);
            workers.add(worker);
        }
        return workers;

    }
    /**
     * 从zookeeper中删除所有IM节点
     */
    public void removeWorkers() {


        try {
            client.delete().deletingChildrenIfNeeded().forPath(managerPath);
        } catch (Exception e) {
            e.printStackTrace();
        }


    }

}
```

​	短连接网关WebGate会调用getBestWorker()方法，取得最佳的IM服务器。而在这个方法中，有两个很重要的方法，一是取得所有的IM服务器列表，注意是带负载的；二是通过负载信息，计算最小负载的服务器。

​	代码中的getWorkers()方法，调用了Cutator的getChildren()方法获取子节点，取得"/im/nodes"目录下的所有的临时节点。然后，调用getData方法取得每一个子节点的二进制负载。最后，将负载信息转成POJOImNode对象。

​	取到了工作节点的POJO列表之后，在balance()方法中，通过一个简单的排序算法，计算出balance值最小的ImNode对象。

#### 12.3.2 与WebGate的整合

​	<u>短连接网关WebGate登录成功之后，需要通过负载均衡器ImLoadBalance类，查询到最佳的Netty服务器，并且返回给客户端</u>，代码如下：

```java
//@EnableAutoConfiguration
@RestController
@RequestMapping(value = "/user", produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
@Api("User 相关的api")
public class UserAction extends BaseController {
    @Resource
    private UserService userService;
    @Resource
    private ImLoadBalance imLoadBalance;

    /**
     * Web短连接登录
     *
     * @param username 用户名
     * @param password 命名
     * @return 登录结果
     */
    @ApiOperation(value = "登录", notes = "根据用户信息登录")
    @RequestMapping(value = "/login/{username}/{password}",method = RequestMethod.GET)
    public String loginAction(
        @PathVariable("username") String username,
        @PathVariable("password") String password) {
        UserPO user = new UserPO();
        user.setUserName(username);
        user.setPassWord(password);
        user.setUserId(user.getUserName());

        //        User loginUser = userService.login(user);

        LoginBack back = new LoginBack();
        /**
         * 取得最佳的Netty服务器
         */
        ImNode bestWorker = imLoadBalance.getBestWorker();
        back.setImNode(bestWorker);
        UserDTO userDTO = new UserDTO();
        BeanUtils.copyProperties(user, userDTO);
        back.setUserDTO(userDTO);
        back.setToken(user.getUserId().toString());
        String r = JsonUtil.pojoToJson(back);
        return r;
    }


    /**
     * 从zookeeper中删除所有IM节点
     *
     * @return 删除结果
     */
    @ApiOperation(value = "删除节点", notes = "从zookeeper中删除所有IM节点")
    @RequestMapping(value = "/removeWorkers",method = RequestMethod.GET)
    public String removeWorkers(){
        imLoadBalance.removeWorkers();
        return "已经删除";
    }

}
```

> [领域驱动设计系列文章（2）——浅析VO、DTO、DO、PO的概念、区别和用处](https://www.cnblogs.com/qixuejia/p/4390086.html)

### 12.4 即时通讯消息的路由和转发的实践案例

​	**如果连接在不同的Netty Worker工作站点的客户端之间，需要相互进行消息的发送，那么就需要在不同的Worker节点之间进行路由和转发。Worker节点的路由是指，根据消息需要转发的目标用户，找到用户的连接所在的Worker节点。由于节点和节点之间都有可能需要相互转发， 因此节点之间的连接是一种网状结构。每一个节点都需要具备路由的能力。**

#### 12.4.1 IM路由器WorkerRouter(代码中改为PeerManager)

​	为每一个Worker节点增加一个IM路由器类，名为PeerManager。<u>为了能够转发到所有的节点，一是要订阅到集群中所有的在线Netty服务器，并且保存起来，二是要其他的Netty服务器建立一个长连接，用于转发消息。</u>

​	PeerManager初始化代码，节选如下：

```java
@Slf4j
public class PeerManager {
    //Zk客户端
    private CuratorFramework client = null;

    private String pathRegistered = null;
    private ImNode node = null;

	//唯一实例模式
    private static PeerManager singleInstance = null;
    //监听路径
    private static final String path = ServerConstants.MANAGE_PATH;
	//节点的容器
    private ConcurrentHashMap<Long, PeerSender> peerMap =
        new ConcurrentHashMap<>();


    public static PeerManager getInst() {
        if (null == singleInstance) {
            singleInstance = new PeerManager();
            singleInstance.client = ZKclient.instance.getClient();
        }
        return singleInstance;
    }

    private PeerManager() {

    }


    /**
     * 初始化节点管理
     */
    public void init() {
        try {

            //订阅节点的增加和删除事件

            PathChildrenCache childrenCache = new PathChildrenCache(client, path, true);
            PathChildrenCacheListener childrenCacheListener = new PathChildrenCacheListener() {

                @Override
                public void childEvent(CuratorFramework client,
                                       PathChildrenCacheEvent event) throws Exception {
                    log.info("开始监听其他的ImWorker子节点:-----");
                    ChildData data = event.getData();
                    switch (event.getType()) {
                        case CHILD_ADDED:
                            log.info("CHILD_ADDED : " + data.getPath() + "  数据:" + data.getData());
                            processNodeAdded(data);
                            break;
                        case CHILD_REMOVED:
                            log.info("CHILD_REMOVED : " + data.getPath() + "  数据:" + data.getData());
                            processNodeRemoved(data);
                            break;
                        case CHILD_UPDATED:
                            log.info("CHILD_UPDATED : " + data.getPath() + "  数据:" + new String(data.getData()));
                            break;
                        default:
                            log.debug("[PathChildrenCache]节点数据为空, path={}", data == null ? "null" : data.getPath());
                            break;
                    }

                }

            };

            childrenCache.getListenable().addListener(childrenCacheListener);
            System.out.println("Register zk watcher successfully!");
            childrenCache.start(PathChildrenCache.StartMode.POST_INITIALIZED_EVENT);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    //...省略其他方法
}
```

​	在上一小节中，我们已经知道，一个节点上线时，首先要通过命名服务加入到Netty集群中。在上面的代码中，PeerManager路由器使用Curator的PathChildrenCache缓存订阅了节点的CHILD_ADDED子节点添加消息。当一个新的Netty节点加入时，调用processNodeAdded(data)方法在本地保存一份节点的POJO消息，并且建立一个消息中转的Netty客户连接。

​	处理节点添加的方法processNodeAdded(data)比较重要，代码如下：

```java
/**
 * 节点增加的处理
 * @param data 新节点
 */
private void processNodeAdded(ChildData data) {
    byte[] payload = data.getData();
    ImNode n = ObjectUtil.JsonBytes2Object(payload, ImNode.class);

    long id = ImWorker.getInst().getIdByPath(data.getPath());
    n.setId(id);

    log.info("[TreeCache]节点更新端口, path={}, data={}",
             data.getPath(), JsonUtil.pojoToJson(n));

    if(n.equals(getLocalNode()))
    {
        log.info("[TreeCache]本地节点, path={}, data={}",
                 data.getPath(), JsonUtil.pojoToJson(n));
        return;
    }
    PeerSender peerSender = peerMap.get(n.getId());
    if (null != peerSender && peerSender.getNode().equals(n)) {

        log.info("[TreeCache]节点重复增加, path={}, data={}",
                 data.getPath(), JsonUtil.pojoToJson(n));
        return;
    }
    if (null != peerSender) {
        //关闭老的连接
        peerSender.stopConnecting();
    }
    peerSender = new PeerSender(n);
    peerSender.doConnect();

    peerMap.put(n.getId(), peerSender);
}
```

​	PeerManager路由器有一个容器成员peerMap，用于封装和保存所有的在线节点。当一个节点添加时，PeerManager取到添加的ZNode路径和负载。ZNode路径中所有新节点的ID，ZNode的payload负载中的有新节点的Netty服务的IP地址和端口号，这三个信息共同够成新节点的POJO消息——ImNode节点信息。PeerManager在检查完和确定本地不存在该节点的准发器后，添加一个转发器peerSender，将新节点的转发器保存在自己的容器中。

​	<u>这里有一个问题，为什么在PeerManager路由器中不简单地保存新节点的POJO信息呢？因为PeerManager路由器的主要作用，除了路由节点，还需要进行消息的转发，所以PeerManager路由器保存的是转发器PeerSender，而添加的远程Netty节点的POJO信息被封装在转发器中</u>。

#### 12.4.2 IM转发器WorkerReSender(代码中改为PeerSender)

​	IM转发器PeerSender封装了远程节点的IP地址、端口号以及ID。另外，PeerSender还维持了一个到远程节点的长连接。也就是说，<u>它是一个Netty的NIO客户端，维护了一个远程节点的Netty Channel通道，通过这个通道将消息转发给远程的节点</u>。

​	IM转发器PeerSender的核心代码如下：

```java
@Slf4j
@Data
public class PeerSender {
	//连接远程节点的Netty通道
    private Channel channel;
	//连接远程节点的POJO信息
    private ImNode node;
    /**
     * 唯一标记
     */
    private boolean connectFlag = false;
    private UserDTO user;

    GenericFutureListener<ChannelFuture> closeListener = (ChannelFuture f) ->
    {
        log.info("分布式连接已经断开……{}", node.toString());
        channel = null;
        connectFlag = false;
    };

    private GenericFutureListener<ChannelFuture> connectedListener = (ChannelFuture f) ->
    {
        final EventLoop eventLoop = f.channel().eventLoop();
        if (!f.isSuccess()) {
            log.info("连接失败!在10s之后准备尝试重连!");
            eventLoop.schedule(() -> PeerSender.this.doConnect(), 10, TimeUnit.SECONDS);

            connectFlag = false;
        } else {
            connectFlag = true;

            log.info(new Date() + "分布式节点连接成功:{}", node.toString());

            channel = f.channel();
            channel.closeFuture().addListener(closeListener);

            /**
             * 发送链接成功的通知
             */
            Notification<ImNode> notification=new Notification<>(ImWorker.getInst().getLocalNodeInfo());
            notification.setType(Notification.CONNECT_FINISHED);
            String json= JsonUtil.pojoToJson(notification);
            ProtoMsg.Message pkg = NotificationMsgBuilder.buildNotification(json);
            writeAndFlush(pkg);
        }
    };


    private Bootstrap b;
    private EventLoopGroup g;

    public PeerSender(ImNode n) {
        this.node = n;

        /**
         * 客户端的是Bootstrap，服务端的则是 ServerBootstrap。
         * 都是AbstractBootstrap的子类。
         **/

        b = new Bootstrap();
        /**
         * 通过nio方式来接收连接和处理连接
         */

        g = new NioEventLoopGroup();


    }

    /**
     * 重连
     */
    public void doConnect() {

        // 服务器ip地址
        String host = node.getHost();
        // 服务器端口
        int port =node.getPort();

        try {
            if (b != null && b.group() == null) {
                b.group(g);
                b.channel(NioSocketChannel.class);
                b.option(ChannelOption.SO_KEEPALIVE, true);
                b.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
                b.remoteAddress(host, port);

                // 设置通道初始化
                b.handler(
                    new ChannelInitializer<SocketChannel>() {
                        public void initChannel(SocketChannel ch) {
                            ch.pipeline().addLast("decoder", new ProtobufDecoder());
                            ch.pipeline().addLast("encoder", new ProtobufEncoder());
                            ch.pipeline().addLast("imNodeHeartBeatClientHandler",new ImNodeHeartBeatClientHandler());
                            ch.pipeline().addLast("exceptionHandler",new ImNodeExceptionHandler());
                        }
                    }
                );
                log.info(new Date() + "开始连接分布式节点:{}", node.toString());

                ChannelFuture f = b.connect();
                f.addListener(connectedListener);


                // 阻塞
                //                 f.channel().closeFuture().sync();
            } else if (b.group() != null) {
                log.info(new Date() + "再一次开始连接分布式节点", node.toString());
                ChannelFuture f = b.connect();
                f.addListener(connectedListener);
            }
        } catch (Exception e) {
            log.info("客户端连接失败!" + e.getMessage());
        }

    }

    public void stopConnecting() {
        g.shutdownGracefully();
        connectFlag = false;
    }

    /**
 	 * 消息转发的方法
 	 * @param pkg 聊天信息
 	 */
    public void writeAndFlush(Object pkg) {
        if (connectFlag == false) {
            log.error("分布式节点未连接:", node.toString());
            return;
        }
        channel.writeAndFlush(pkg);
    }
}
```

​	在IM转发器中，主体是与Netty相关的代码，比较简单。严格来说，IM转发器是一个Netty客户端，它比Netty服务器的代码简单一点。

​	转发器有一个消息转发的方法，直接通过Netty Channel通道将消息发送到远程节点，代码如下：

```java
/**
 * 消息转发的方法
 * @param pkg 聊天信息
 */
public void writeAndFlush(Object pkg) {
    if (connectFlag == false) {
        log.error("分布式节点未连接:", node.toString());
        return;
    }
    channel.writeAndFlush(pkg);
}
```

### 12.5 Feign短连接RESTful调用

​	<u>一般来说，短连接的服务接口都是基于应用层HTTP协议的HTTP API或者RESTful API实现的，通过JSON文本格式返回数据</u>。如何在Java服务器端调用其他节点的HTTP API或者RESTful API呢？

​	至少有以下几种方式：

+ JDK原生的URLConnection
+ **Apache的HttpClient/HttpComponents**
+ Netty的异步HttpClient
+ Spring的RestTemplate

​	目前用的最多的，基本上是第二种，这也是在单体服务时代最为成熟和稳定的方式，也是效率最高的短连接方式。

​	首先做一个解释，什么是RESTful API。**REST的全称是Representational State Transfer（表征状态转移，也有译成表述性状态转移），它是一种API接口的风格而不是标准，只是提供了一组调用的原则和约束条件。也就是说，在短连接服务的领域，它算是一种特殊格式的HTTP API**。

​	**言归正传，如果同一个HTTP API/RESTful API接口，倘若不止一个短连接服务器提供服务，而是有多个节点提供服务，那么简单使用HttpClient就无能为力了。**

​	HttpClient/HttpComponents调用不能根据接口的负载或者其他的条件，去判断哪一个接口应该调用，哪一个接口不应该调用。解决这个问题的方式如何呢？

​	<u>可以使用Feign来调用多个服务器的同一个接口。Feign不仅可以进行同接口多服务器的负载均衡，一旦使用了Feign作为HTTP API的客户端，调用远程的HTTP接口就会变得像调用本地方法一样简单</u>。

​	Feign是Netflix开发的一个声明式、模板化的HTTP客户端，Feign的目标是帮助Java工程师更快捷、优雅地调用HTTP API/RESTful API。Netflix Feign目前改名为[OpenFeign](https://github.com/OpenFeign/feign)。Feign在Java应用中负责处理与远程Web服务的请求响应，最大限度地降低了编程的复杂性。另外，Feign被无缝集成到了SpringCloud微服务框架，使用Feign后，可以非常方便地在项目中使用SpringCloud微服务技术。如果项目使用了SpringCloud技术，同样可以更方便地使用Feign。即便项目中没有使用SpringCloud，使用Feign也非常简单。总之，它是Java中调用Web服务的客户端历器。

​	下面就看看在单独使用Feign的应用场景下是怎么调用远程的HTTP服务的。

​	引入Feign依赖的jar包到pom.xml

```java
<!-- https://mvnrepository.com/artifact/io.github.openfeign/feign-core -->
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-core</artifactId>
    <version>10.7.3</version>
</dependency>
<!-- https://mvnrepository.com/artifact/io.github.openfeign/feign-gson -->
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-gson</artifactId>
    <version>10.7.3</version>
</dependency>
```

​	接下来就可以开始使用Feign来调用远程的HTTP API了。

#### 12.5.1 短连接API的接口准备

​	前面讲到，在高并发的IM系统中，用户的登陆与认证、好友的更新与获取等等一些低频的请求都使用短连接来实现。

​	作为演示，这里仅仅举例两个短连接的API接口：

（1）http://localhost:8080/user/{userid}

​	这个接口的功能是获取用户信息，是一个典型的RESTful类型的接口。{userid}是一个占位符，在调用的时候需要替换成用户id。例如，如果用户id为1，Feign最终生成的实际调用的链接为：http://localhost:8080/user/1

（2）http://localhost:8080/login/{username}/{password}

​	这个接口实现的功能是用户登录的认证。这里有两个占位符，占位符{username}表示用户名称，占位符{password}表示用户密码。例如，如果用户名称为zhangsan，密码为123，那么Feign最终生成的实际调用的链接为：http://localhost:8080/login/zhangsan/123

​	上面的接口很简单，仅仅是为了演示，不能用于生产场景。这些API的开发和实现可以利用Spring MVC/JSP Servlet等常见的WEB技术。

#### 12.5.2 声明远程接口的本地代理

​	如何通过Feign技术来调用上面的这些HTTP API呢？

​	第一步，需要创建一个本地的API的代理接口。具体如下：

```java
/**
 * 远程接口的本地代理
 * Created by 尼恩 at 疯狂创客圈
 */
public interface UserAction {

    /**
     * 登录代理
     * @param username  用户名
     * @param password  密码
     * @return  登录结果
     */
    @RequestLine("GET /user/login/{username}/{password}")
    public String loginAction(
        @Param("username") String username,
        @Param("password") String password);

    /**
     * 获取用户信息代理
     * @param userid  用户id
     * @return  用户信息
     */
    @RequestLine("GET /{userid}")
    public String getById(@Param("userid") Integer userid);

}
```

​	在代理接口中，为每一个远程HTTP API定义一个本地代理方法。

​	<u>如何将方法对应到远程接口呢？在方法的前面加上一个@RequestLine注解，注明远程HTTP API的请求地址</u>。这个地址不需要从域名和端口开始，只需要从URI的根目录"/"开始即可。例如，如果远程HTTP API的URL为：http://localhost:8080/user/{userid}，@RequestLine声明的值只需要配成/user/{userid}即可。

​		如何给接口传递参数值呢？在方法的参数前面加上一个@Param注解即可。@Param内容为HTTP链接中参数占位符的名称。绑定好之后，实际这个Java接口中的参数值会替换到@Param注解的占位符。例如，由于在getById的唯一参数userid的@Param注解中用到的占位符是userid，那么通过调用userAction.getById(100)，即userid的值为100，就会用来替换掉请求链接http://localhost:8080/user/{userid}中的占位符userid，最终得到的请求连接为：http://localhost:8080/user/100。

#### 12.5.3 远程API的本地调用

​	在完成远程API的本地代理接口的定义之后，接下来的工作就是调用本地代理，这个工作也是非常简单的。

​	还是以疯狂创客圈"CrazyIM"实战项目中获取用户信息和用户登录这两个API的代理接口的调用为例。实践案例的代码如下：

```java
public class LoginActionTest {
    /**
     * 测试登录
     */
    @Test
    public void testLogin() {

        UserAction action = Feign.builder()
            //                .decoder(new GsonDecoder())
            .decoder(new StringDecoder())
            .target(UserAction.class, "http://localhost:8080/user");

        String s = action.loginAction("zhangsan", "zhangsan");

        LoginBack back= JsonUtil.jsonToPojo(s,LoginBack.class);
        System.out.println("s = " + s);

    }

    /**
     * 测试登录
     */
    @Test
    public void testLogin2() {

        LoginBack back= WebOperator.login("lisi","lisi");
        System.out.println("s = " + "");

    }


    /**
     * 测试获取用户信息
     */
    @Test
    public void testGetById() {

        UserAction action = Feign.builder()
            //                .decoder(new GsonDecoder())
            .decoder(new StringDecoder())
            .target(UserAction.class, "http://localhost:8080/user");

        String s = action.getById(2);
        System.out.println("s = " + s);

    }
}
```

​	**最为核心的就一步，构建一个远程代理接口的本地实例**。调用Feign.builder()构造器模式的方法，带上一票配置方法的链式调用。主要的链式调用的配置方法介绍如下：

（1）options配置方法

​	options方法指定连接超时的时长以及响应超时的时长。

（2）retryer配置方法

​	retryer方法主要是指定重试策略。

（3）decoder配置方法

​	decoder方法指定对象解码方式，这里用的是基于String字符串的解码方式。如果需要使用Jackson的解码方式，需要在pom.xml中添加Jackson的依赖。

（4）client配置方法

​	<u>此方法用于底层的请求客户端。Feign默认使用Java的HttpURLConnection作为HTTP请求客户端</u>。Feign也可以直接使用现有的公共第三方HTTP客户端类库，如Apache HttpClient，OKHttp，来编写Java客户端以访问HTTP服务。

​	集成Apache HttpComponentsHttpClient的例子为：

```java
Feign.builder().client(new ApacheHttpClient())
```

​	集成OKHttp的例子为：

```java
Feign.builder().client(new OkHttpClient()).target(...)
```

​	继承Ribbon的例子为：

```java
Feign.builder().client(RibbonClient.create()).target(...)
```

​	**Feign集成了Ribbon后，利用Ribbon维护了API服务列表信息，并且通过轮询实现了客户端的负载均衡**。而与Ribbon不同的是，Feign只需要定义服务绑定接口且以声明的方法，可以优雅而简单地实现服务调用。

（5）target方法

​	是构造器模式最后面的方法，通过它可以最终得到本地代理实例。它有两个参数，第一个是本地的代理接口的class类型，第二个是远程URL的根目录地址。第一个代理接口类很重要，最终Feign.builder()构造器返回的本地代理实例类型就这个接口的类型。代理接口类中每一个接口方法前用@RequestLine声明的URI链接，最终都会加上target方法的第二个参数的根目录值，来形成最终的URL。

​	**target方法是最后面的一个方法，也就是说它的后面不能再链接调用其他的配置方法**。

​	主要的构造器方法就介绍这些，具体的使用细节和其他方法看官网说明文档。

​	使用配置方法完成配置之后，再通过Feign.builder()构造完成代理实例，调用远程API，这就和调用Java函数一样简单了。

​	<u>总之，如果是独立调用HTTP服务，那么尽量使用Feign。原因：一是很简单；二是如果采用HttpClient或其他相对较"重"的框架，对初学者来说编码量与学习曲线都会是一个挑战；三是既可以独立使用Feign，又方便后续Spring Cloud微服务架构的集成，那么使用Feign代替HttpClient等其他方式，何乐而不为呢？</u>

### 12.6 分布式的在线用户统计的实践案例

​	顾名思义，计数器是用来计数的。在分布式环境中，常规的计数器是不能使用的，再次介绍ZooKeeper实现的分布式计数器。**利用ZooKeeper可以实现一个集群共享的计数器，只要使用相同的path就可以得到最新的计数器值，这是由ZooKeeper的一致性保证的**。

#### 12.6.1 Curator的分布式计数器

​	Curator有两个计数器，一个是用int类型来计数（SharedCount），一个用long类型来计数（DistributedAtomicLong）。下面使用DistributedAtomicLong来实现高并发IM系统中的在线用户统计，代码如下：

```java
/**
 * 分布式计数器
 * create by 尼恩 @ 疯狂创客圈
 **/
@Data
public class OnlineCounter {

    private static final String PATH =
        ServerConstants.COUNTER_PATH;

    //Zk客户端
    private CuratorFramework client = null;

    //单例模式
    private static OnlineCounter singleInstance = null;

    DistributedAtomicLong distributedAtomicLong = null;
    private Long curValue;

    public static OnlineCounter getInst() {
        if (null == singleInstance) {
            singleInstance = new OnlineCounter();
            singleInstance.client = ZKclient.instance.getClient();
            singleInstance.init();
        }
        return singleInstance;
    }

    private void init() {

        /**
         *  分布式计数器，失败时重试10，每次间隔30毫秒
         */

        distributedAtomicLong = new DistributedAtomicLong(client, PATH, new RetryNTimes(10, 30));

    }

    private OnlineCounter() {

    }

    /**
     * 增加计数
     */
    public boolean increment() {
        boolean result = false;
        AtomicValue<Long> val = null;
        try {
            val = distributedAtomicLong.increment();
            result = val.succeeded();
            System.out.println("old cnt: " + val.preValue()
                               + "   new cnt : " + val.postValue()
                               + "  result:" + val.succeeded());
            curValue = val.postValue();

        } catch (Exception e) {
            e.printStackTrace();
        }
        return result;
    }

    /**
     * 减少计数
     */
    public boolean decrement() {
        boolean result = false;
        AtomicValue<Long> val = null;
        try {
            val = distributedAtomicLong.decrement();
            result = val.succeeded();
            System.out.println("old cnt: " + val.preValue()
                               + "   new cnt : " + val.postValue()
                               + "  result:" + val.succeeded());
            curValue = val.postValue();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return result;

    }

}
```

#### 12.6.2 用户上线和下线的统计

​	当用户上线的时候，调用increase方法分布式地增加一次计数：

```java
/**
 * 增加 远程的 session
 */
public void addRemoteSession(RemoteSession remoteSession) {
    String sessionId = remoteSession.getSessionId();
    if (localSessionMap.containsKey(sessionId)) {
        log.error("通知有误，通知到了会话所在的节点");
        return;
    }

    remoteSessionMap.put(sessionId, remoteSession);
    //删除本地保存的 远程session
    String uid = remoteSession.getUserId();
    UserSessions sessions = sessionsLocalCache.get(uid);
    if (null == sessions) {
        sessions = new UserSessions(uid);
        sessionsLocalCache.put(uid, sessions);
    }

    sessions.addSession(sessionId, remoteSession.getImNode());
}
```

​	当用户下线地时候，调用decrease方法分布式地减少一次计数：

```java
/**
 * 删除 远程的 session
 */
public void removeRemoteSession(String sessionId) {
    if (localSessionMap.containsKey(sessionId)) {
        log.error("通知有误，通知到了会话所在的节点");
        return;
    }

    RemoteSession s = remoteSessionMap.get(sessionId);
    remoteSessionMap.remove(sessionId);

    //删除本地保存的 远程session
    String uid = s.getUserId();
    UserSessions sessions = sessionsLocalCache.get(uid);
    sessions.removeSession(sessionId);

}
```

### 12.7 本章小结

​	本章介绍了支亿级流量地高并发IM架构以及高并发架构下的技术选型。然后，集中介绍了Netty集群所涉及的分布式IM的命名服务、Worker集群的负载均衡、即时通信消息的路由和转发、分布式的在线用户统计等技术实现。

​	本章的示例代码来自"疯狂创客圈"社群的高并发学习项目"CrazyIM"，由于项目不断地迭代，因此在看书的时候，建议参考最新版本的代码。不过，细节如何迭代，设计思路基本都是一致的。

​	本章的目的仅仅是抛砖引玉。寥寥数千字，无法彻底地将一个支持亿级流量的IM项目的架构及其实现剖析得非常清楚，后续"疯狂创客圈"会结合本书将内容更加全面呈现给大家。

> [netty无缝切换rabbitmq、activemq、rocketmq实现聊天室单聊、群聊功能](https://segmentfault.com/a/1190000020220432)
>
> [基于Netty与RabbitMQ的消息服务](https://www.cnblogs.com/luxiaoxun/p/4257105.html)
>
> [netty与MQ使用心得](https://www.cnblogs.com/xingjunli/p/4923986.html)
>
> [Netty中Channel的生命周期（SimpleChannelInboundHandler）](https://blog.csdn.net/u014131617/article/details/86476522)
>
> [Reactor详解](https://blog.csdn.net/bingxuesiyang/article/details/89888664)
>
> [Netty ChannelOption参数详解](https://www.jianshu.com/p/975b30171352)
>
> EventLoopGroup 反应器线程组线程数量，现在想想之前看过一个代码一下子给工作线程配置CPU内核数*10，感觉有点迷惑行为；
>
> [netty实战之百万级流量NioEventLoopGroup线程数配置](https://blog.csdn.net/linsongbin1/article/details/77698479)
>
> [NioEventLoopGroup源码分析与线程设定](https://www.cnblogs.com/linlf03/p/11373834.html)
>
> [[netty源码分析]--EventLoopGroup与EventLoop 分析netty的线程模型](https://blog.csdn.net/u010853261/article/details/62043709)

