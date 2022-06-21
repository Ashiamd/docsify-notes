# MySQL技术内幕-InnoDB存储引擎第2版-学习笔记

# 第一章 MySQL体系结构和存储引擎

## 1.1 定义数据库和实例

+ 数据库database：

  物理操作系统文件或其他形式文件类型的集合。

  当使用NDB引擎时，数据库的文件可能不是操作系统上的文件，而是存放于内存中的文件，定义不变。

+ 实例instance：

  MySQL数据库由后台线程以及一个共享内存区组成。共享内存可以被运行的后台线程所共享。需要牢记的是，**数据库实例才是真正用于操作数据库文件的**。

​	一个实例对应一个数据库，一个数据库对应一个实例。**在集群情况下，可能存在一个数据库被多个数据实例使用的情况**。

​	**MySQL单进程多线程，MySQL数据库实例在系统上的表现就是一个进程**。

## 1.2 MySQL体系结构

> [MySQL体系结构_贾维斯的博客-CSDN博客_mysql体系结构](https://blog.csdn.net/u010735147/article/details/81330201)

![img](https://images2015.cnblogs.com/blog/540235/201609/540235-20160927083854563-2139392246.jpg)

+ 连接池组件
+ 管理服务和工具组件
+ SQL接口组件
+ 查询分析器组件
+ 优化器组件
+ 缓冲（Cache）组件
+ **插件式存储引擎**
+ 物理文件

MySQL数据库区别于其他数据库的最重要的一个特点就是<u>插件式</u>的表存储引擎。

**存储引擎基于表，而不是数据库**。

## 1.3 MySQL存储引擎

### 1.3.1 InnoDB存储引擎

​	MySQL5.5.8开始，InnoDB是默认的存储引擎。

+ InnoDB存储引擎**支持事务**
+ 主要面向在线事务处理（**OLTP**）应用
+ **行锁设计**
+ 支持外键
+ 支持类似Oracle的非锁定读，默认读操作不会产生锁
+ 数据存储在逻辑表内
+ MySQL4.1之后，InnoDB表单独存放到独立的ibd文件中
+ 支持用裸设备（raw disk）建立表空间
+ **多版本并发控制（MVCC）以支持高并发性**
+ **实现SQL标准的4种隔离级别，默认为REPEATABLE**
+ **使用next-key locking策略避免幻读（phantom）现象**
+ 提供**插入缓冲（insert buffer）**、**二次写（double write）**、**自适应哈希索引（adaptive hash index）**、**预读（read ahead）**等高性能和高可用的功能

+ 对于表中数据的存储，InnoDB存储引擎采用了聚集（clustered）的方式，因此**每张表的存储都是按主键的顺序进行存放**
+ **如果没有显式地在表定义时指定主键，InnoDB存储引擎会为每一行生成一个6字节的ROWID，并以此作为主键**

### 1.3.2 MyISAM存储引擎

在MySQL5.5.8之前MyISAM存储引擎是默认的存储引擎（除WIndows版本外）

+ **不支持事务**
+ **表锁设计**
+ 支持全文索引
+ 主要面向**OLAP**数据库应用
+ MyISAM的缓存池**只缓存(cache)索引文件，而不缓冲数据文件**

+ MyISAM存储引擎表由MYD和MYI组成，MYD用来存放数据文件，MYI用来存放索引文件

+ 可以使用myisampack工具进一步压缩数据文件

+ myisampack使用哈夫曼（Huffman）编码静态算法压缩数据，**压缩后的表是只读的**

+ MySQL5.0之前，MyISAM默认支持的表大小为4GB

  （如需支持更大的表，需指定MAX_ROWS和AVG_ROW_LENGTH属性）

+ MySQL5.0起，MyISAM默认支持256TB的单表数据

> 对于MyISAM存储引擎表，MySQL数据库只缓存索引文件，数据文件的缓存由操作系统本身完成，这与其他使用LRU算法缓存数据的大部分数据库不同。
>
> MySQL5.1.23之前，32位和64位的操作系统缓存索引的缓冲区最大只能设置为4GB，往后的版本64位系统可以支持大于4GB的索引缓冲区。

### 1.3.3 NDB存储引擎

​	NDB是集群存储引擎，类似Oracle的RAC集群。与Oracle RAC share everything架构不同，**NDB采用的是share nothing的集群架构，可用性更高**。

+ **数据全部存放在内存中**，主键查找（primaru key looups）速度极快

  （MySQL5.1开始，可以将非索引数据放在磁盘上）

+ 通过添加NDB数据存储节点（Data Node）可以线性提高数据库性能，是高可用、高性能的集群系统

+ **连接操作（JOIN）在MySQL数据库层完成，而不是在存储引擎层完成**

  （**复杂的连接操作需要巨大的网络开销，因此查询速度很慢**）

### 1.3.4 Memory存储引擎

​	之前称HEAP存储引擎。

+ **将表中的数据存放在内存中**

  （如果数据库重启或者发生崩溃，表中的数据都将消失）

+ 适合用于存储临时数据的临时表，以及数据仓库中的维度表

+ **默认使用哈希索引，而不是B+树索引**

+ **只支持表锁**，并发性能较差

+ **不支持TEXT和BLOB类型数据**

+ **存储变长字段（varchar）时，按照定长字段（char）方式存储，会造成内存浪费**

  （eBay的工程师Igor Chernyshev给出patch解决方案）

+ **<u>MySQL数据库使用Memory存储引擎作为临时表来存放查询的中间结果集（intermediate result）。如果中间结果集大雨Memory存储引擎表的容量设置，又或者中间含有TEXT或BLOB列类型字段，则MySQL数据库会把其转换到MyISAM存储引擎表而存放到磁盘中。MyISAM不缓存数据文件，此时产生的临时表的性能对于查询会有损失</u>**。

### 1.3.5 Archive存储引擎

+ **只支持INSERT和SELECT操作**
+ MySQL5.1起，支持索引
+ 使用zlib算法将数据行（row）进行压缩后存储，压缩比一般可达到1:10
+ **适合存储归档数据，如日志信息**
+ 使用**行锁**实现高并发插入操作
+ **非事务安全**，设计目标主要是提供高速插入和压缩功能

### 1.3.6 Federated存储引擎

+ **不存放数据，只是指向一台远程MySQL数据库服务器上的表**

  （类似SQL Server的链接服务器和Oracle的透明网关）

+ 只支持MySQL数据库表，不支持异构数据库表

### 1.3.7 Maria存储引擎

+ 设计目标是取代MyISAM，作为MySQL的默认存储引擎

  （可视为是MyISAM的后续版本）

+ **支持缓存数据和索引文件**

+ **行锁设计**

+ **提供MVCC功能**

+ **支持事务和非事务安全的选项**

+ 对BLOB字符类型的处理性能更好

### 1.3.8 其他存储引擎

Merge、CSV、Sphinx、Infobright等。

+ 支持全文索引：MyISAM、InnoDB（1.2版本）和Sphinx

+ 对于ETL操作，MyISAM更有优势；OLTP环境，InnoDB效率更高

+ 当表的数据量大于1000万时MySQL的性能会急剧下降吗？

  根据场景选择合适的存储引擎（以及正确的配置）。官方手册提及，Mytrix和Inc.在InnoDB上存储超过1TB的数据，还有一些网站使用InnoDB存储引擎，<u>处理插入/更新的操作平均800次/秒</u>

## 1.4 各储存引擎之间的比较

> [什么是存储引擎以及不同存储引擎特点 - wildfox - 博客园 (cnblogs.com)](https://www.cnblogs.com/wildfox/p/5815414.html)

![img](https://images2015.cnblogs.com/blog/798756/201608/798756-20160828170533471-1740775159.jpg)

+ 通过`SHOW ENGINES`或查找`information_schema`架构下的`ENGINES`表，可查看当前使用的MySQL数据库所支持的存储引擎

## 1.5 连接MySQL

连接MySQL操作是一个连接进程和MySQL数据库实例进行通信，本质上是**进程通信**。

进程通信的方式有管道、命名管道、TCP/IP套接字、UNIX域套接字等。

MySQL数据库提供的连接方式从本质上看即上述提及的进程通信方式。

### 1.5.1 TCP/IP

​	MySQL实例(server)在一台服务器上，客户端client在另一台服务器上与其通过TCP/IP网络进行通信。

​	在通过TCP/IP连接到MySQL实例时，MySQL数据库会先检查一张权限视图，用来判断发起请求的客户端IP是否允许连接到MySQL实例。

​	在mysql架构下，表名为user。

### 1.5.2 命名管道和共享内存

+ 如果两个需要进程通信的进程在同一台服务器上，可以使用命名管道，Microsoft SQL Server数据库默认安装后的本地连接也是使用命名管道。在MySQL数据库中须在配置文件中启用`--enable-named-pipe`选项。

+ 在MySQL4.1之后，提供共享内存的连接方式，通过配置文件中添加`--shared-memory`实现。如果想使用共享内存的方式，在连接时，MySQL客户端还需要使用`--protocol=memory`选项。

### 1.5.3 UNIX域套接字

​	在Linux和Unix环境下，还可以使用UNIX域套接字。UNIX域套接字不是网络协议，只能在MySQL客户端和数据库实例在一台服务器上的情况下使用。用户可以在配置文件中指定套接字文件的路径，如`--socket=/tmp/mysql.sock`。当数据库实例启动后，用户可以通过下列命令来进行UNIX域套接字文件的查找：

`SHOW VARIABLES LIKE 'socket';`

查出UNIX域套接字文件路径后，进行连接`mysql -udavid -S 套接字文件`

## 1.6 小结

+ 数据库、数据库实例
+ 常见的表存储引擎的特性
+ MySQL独有的插件式存储引擎概念

# 2. InnoDB存储引擎

​	InnoDB是事务安全的MySQL存储引擎，设计上采用了类似Oracle数据库的结构。

​	通常来说，InnoDB存储引擎是OLTP应用中核心表的首选存储引擎。

## 2.1 InnoDB存储引擎概述

​	MySQL5.5开始，InnoDB是MySQL默认的表存储引擎（之前的版本InnoDB存储引擎尽在Windows下为默认的存储引擎）。

​	InnoDB是第一个完整支持ACID事务的MySQL存储引擎（BDB是第一个支持事务的MySQL存储引擎，已经停止开发）

特点：

+ **行锁设计**
+ **支持MVCC**
+ **支持外键**
+ **提供一致性非锁定读**
+ 被设计用来最有效地利用以及使用内存和CPU

## 2.2 InnoDB存储引擎的版本

| 版本         | 功能                                                     |
| ------------ | -------------------------------------------------------- |
| 老版本InnoDB | 支持ACID、行锁设计、MVCC                                 |
| InnoDB 1.0.x | 继承了上述版本所有功能，增加了compress和dynamic页格式    |
| InnoDB 1.1.x | 继承了上述版本所有功能，增加了Linux AIO、多回滚段        |
| InnoDB 1.2.x | 继承了上述版本所有功能，增加了全文索引支持、在线索引添加 |

## 2.3 InnoDB体系架构

> [InnoDB体系架构详解_qqqq0199181的博客-CSDN博客_innodb架构](https://blog.csdn.net/qqqq0199181/article/details/80659856)

![img](https://img-blog.csdn.net/20180612011119133)

​	InnoDB存储引擎有多个内存块，可以认为这些内存块组成了一个大的内存池，负责如下工作：

+ 维护所有进程/线程需要访问的多个内部数据结构。
+ 缓存磁盘上的数据，方便快速地读取，同时在对磁盘文件的数据修改之前在这里缓存。
+ 重做日志（redo log）缓冲。

​	后台线程的主要作用是负责刷新内存池中的数据，保证缓冲池中的内存缓存是最近的数据。此外将已修改的数据文件刷新到磁盘文件，同时保证在数据库发生异常的情况下InnoDB能恢复到正常运行状态。

### 2.3.1 后台线程

1. Master Thread

   ​	主要负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性。

   + 脏页的刷新
   + 合并插入缓冲（INSERT BUFFER）
   + UNDO页的回收
   + ...

   *ps：2.5节会详细地介绍各个版本中的Master Thread的工作方式。*

2. IO Thread

   在InnoDB存储引擎中大量使用了AIO（Async IO）来处理写IO请求，极大提高数据库的性能。

   IO Thread主要负责这些IO请求的回调（call back）处理。

   （此处省略参数配置、使用介绍）

3. Purge Thread

   事务被提交后，其所使用的undolog可能不再需要，因此需要PurgeThread回收已经使用并分配的undo页。

   + InnoDB1.1之前，purge操作仅在Master Thread中完成
   + InnoDB1.1之后，purge操作可以独立到单独的线程中进行

   （此处省略配置文件设置启用独立的Purge Thread）

4. Page Cleaner Thread

   InnoDB 1.2.x版本中引入。

   ​	将之前版本中脏页的刷新操作都放入到单独的线程中完成，减轻原Master Thread的工作及对于用户查询线程的阻塞，进一步提高InnoDB存储引擎的性能。

### 2.3.2 内存

​	InnoDB存储引擎基于磁盘存储，并将其中的记录按照页的方式进行管理。因此可以将其视为基于磁盘的数据库系统（Disk-base Database）。

1. 缓冲池

   ​	简单来说就是一块内存区域，通过内存的速度来弥补磁盘速度较慢对数据库性能的影响。

   ​	在数据库中进行读取页的操作，首先将从磁盘读到的页放在缓冲池中。这个过程称为将页"FIX"在缓冲池中。下一次再读取相同的页时，首先判断该页是否在缓冲池中。若在缓冲池中，称该页在缓冲池中被命中，直接读取该页，否则读取磁盘上的页。

   ​	对于数据库中过的页操作，则首先修改在缓冲池中的页，然后再以一定的频率刷新到磁盘上。

   + **并非每次缓冲池中的页发生更新时触发刷新数据到磁盘，而是通过Checkpoint机制刷新到磁盘**。

   （此处省略缓冲池大小配置）

   + **缓冲池缓存的数据页数据类型有：索引页、数据页、undo页、插入缓冲（insert buffer）、自适应哈希索引（data dictionary）等**。

     ![InnoDB内存结构](https://static.oschina.net/uploads/img/201611/22145927_7wWa.png)

     > [InnoDB存储引擎内部结构_weixin_33860528的博客-CSDN博客](https://blog.csdn.net/weixin_33860528/article/details/92001932)

   ​	**从InnoDB 1.0.x开始，允许有多个缓冲池实例。每个页根据哈希值平均分配到不同的缓冲池实例中**。可通过innodb_buffer_pool_instances进行配置，默认值为1。

   （此处省略关于innodb_buffer_pool_instances相关的配置介绍和使用）

---

2. LRU List、Free List和Flush List

   ​	通常来说，数据库中的缓冲池通过LRU算法进行管理。当缓冲池不能存放新读取到的页时，首先释放LRU列表中尾端的页。

   ​	InnoDB存储引擎中，缓冲池中的页大小默认为16KB，同样使用LRU算法对缓冲池进行管理。

   ​	<u>InnoDB对传统LRU进行优化，在LRU列表中加入了midpoint位置。新读取到的页，并非放到LRU列表首部，而是放到LRU列表的midpoint位置</u>。（在InnoDB称为midpoint insertion strategy）

   ​	默认配置下，midpoint在LRU列表长度的5/8处。（可由innodb_old_blocks_pct控制）

   （此处省略innodb_old_blocks_pct配置和说明）

   ​	InnoDB中，把midpoint之后的列表称为old列表，之前的称为new列表。可以简单理解为new列表中的页都是最为活跃的热点数据。

   + **为什么不适用朴素的LRU算法，将新读取的页插入LRU首部？**

     ​	**若直接插入LRU首部，诸如索引或者数据的扫描操作，会使缓冲池中的页被刷新出，这些数据一般仅在本次查询中使用，非热点数据。这使得原先真正的热点数据从LRU列表中被移除，下一次读取该页，需要重新从磁盘读取**。

     ​	为解决该问题，InnoDB引入innodb_old_blocks_time，表示该页读取到mid位置后需要等待多久才会被加入到LRU列表的热端。

     （省略配置介绍）

   ---

   ​	数据库启动时，LRU列表空，所有页存放在Free列表中。当需要从缓冲池分页时，首先从Free列表中查找是否有可用的空闲页，若有则将该页从Free列表中删除，放入LRU列表中。

   + 当页从LRU的old部分加入到new部分时，称该操作为page made young
   + 因innodb_old_blocks_time的设置而导致页没有从old部分移动到new部分的操作称为page not made young

   可以通过`SHOW ENGINE INNODB STATUS`观察LRU列表以及Free列表的使用情况和运行状态。

   （此处省略指令的使用）

   + 缓冲池中的页还可能被分配给自适应哈希索引、Lock信息、Insert Buffer等页，这部分页不需要LRU算法维护，因此不再LRU列表中

   > `SHOW ENGINE INNODB STATUS`，输出的`Buffer pool hit rate`表示缓冲池命中率，通常不应该小于95%，出现异常时需要观察是否是由于全表扫描引起LRU列表被污染的问题。
   >
   > `SHOW ENGINE INNODB STATUS`显示的非当前状态，而是过去某个时间范围的InnoDB存储引擎的状态。

   （省略压缩页unzip_LRU内容介绍）

---

3. 重做日志缓冲（redo log buffer）

   InnoDB首先将重做日志信息放入该缓冲区，然后按照一定频率将其刷新到重做日志文件。

   ​	redo log buffer一般不用设置很大，一般每一秒钟会将重做日志缓冲刷新到日志文件，只需保证每秒产生的事务量在这个缓冲大小之内即可。（配置参数innodb_log_buffer_size，默认8MB）

   ​	通常8MB足以，下列三种情况下会将刷新缓冲到磁盘的重做日志文件中：

   + **Master Thread每一秒将重做日志缓冲刷新到重做日志文件**
   + **每个事物提交时会将重做日志缓冲到重做日志文件**
   + **当重做日志缓冲池剩余空间小于1/2时，重做日志缓冲刷新到重做日志文件**

---

4. 额外的内存池

   ​	在InnoDB中，通过内存堆（heap）的方式管理内存。对一些数据结构本身的内存进行分配时，需要从额外的内存池中进行申请，当该区域内存不足时，会从缓冲池中进行申请。

   > 例如，分配了缓冲池（innodb_buffer_pool），但是每个缓冲池中的帧缓冲（frame buffer）还有对应的缓冲池控制对象（buffer control block），这些对象记录了一些诸如LRU、锁、等待等信息，而这个对象的内存需要从额外内存池中申请。
   >
   > 因此，申请了很大的InnoDB缓冲池时，也应考虑相应增加这个值。

## * 2.4 Checkpoint技术

​	为了避免数据丢失，当前**事务数据库系统**普遍采用了**Write Ahead Log**策略。即当食物提交时，先写重做日志（磁盘），再修改页（内存）。*（宕机导致内存数据丢失时，即通过重做日志完成数据恢复，保证事务ACID的D，Durability）*

​	Checkpoint(检查点)技术主要解决以下几个问题：

+ **缩短数据库的恢复时间**（宕机时，Checkpoint之前的页都已经刷新回磁盘，只需对Checkpoint之后的重做日志进行恢复）
+ **缓冲池不够用时，将脏页刷新到磁盘**（当缓冲池不够用时，根据LRU算法会溢出最近最少使用的页，若此页为脏页，那么需要强制执行Checkpoint，将脏页也就是页的新版本刷回磁盘）
+ **重做日志不可用时，刷新脏页**（当前事务数据库系统对重做日志的设计都是循环使用的，并非无限增大。不再需要的重做日志片段在宕机时可被覆盖重用。如果这部分重做日志还需要使用，那必须强制产生Checkpoint，将缓冲池中的页纸烧刷新到当前重做日志的位置。）

​	InnoDB存储引擎，通过LSN（Log Sequence Number）标记版本。LSN是8字节数字，单位是字节。每个页有LSN，重做日志也有LSN，Checkpoint也有LSN。可通过`SHOW ENGINE INNODB STATUS`来观察。

​	概述，**Checkpoint主要作用即将缓冲池中的脏页刷回到磁盘**。不同的细节即：每次刷新多少页到磁盘、从哪里取脏页面、什么时候触发Checkpoint。

​	InnoDB引擎内部，有两种Checkpoint：

+ Sharp Checkpoint（默认<u>在数据库关闭时</u>将**所有**脏页刷新回磁盘，参数`innodb_fast_shutdown=1`）
+ **Fuzzy Checkpoint（InnoDB存储引擎内部使用，只刷新一部分脏页）**

---

InnoDB存储引擎中可能发生以下几种情况的Fuzzy Checkpoint：

+ Master Thread Checkpoint

  每秒or每十秒速度异步从缓冲池刷新一定比例脏页回磁盘，不阻塞用户查询线程。

+ FLUSH_LRU_LIST Checkpoint

  InnoDB 1.1.版本之前，在用户查询线程需保证LRU列表有差不多100个空闲页供用户查询时使用。若没有100个可用空闲页，那么InnoDB存储引擎会将LRU列表尾端的页移除，如果这些页中有脏页，那么需要进行Checkpoint，而这些页来自LRU列表，所以称为FLUSH_LRU_LIST Checkpoint。

  MySQL5.6，InnoDB1.2.x版本开始，这项检查放在单独的Page Cleaner线程中运行，用户可通过参数`innodb_lru_scan_depth`控制LRU列表中可用页的数量（默认1024）。

+ Async/Sync Flush Checkpoint

  重做日志文件不可用时，需强制从脏页列表中选取部分脏页刷新回磁盘。Async/Sync Flush Checkpoint保证重做日志的循环使用的可用性。InnoDB 1.2.x之前，Async Flush Checkpoint阻塞发现问题的用户查询线程，而Sync Flush Checkpoint阻塞所有用户查询线程，并等待脏页刷新完成。InnoDB 1.2.x开始，这部分的刷新操作同样放到单独的Page Cleaner Thread中，不阻塞用户查询线程。`SHOW ENGINE INNODB STATUS`可观察具体细节。

+ Dirty Page too much Checkpoint

  为保证缓存池有足够可用的页，脏页过多会触发InnoDB存储引擎强制进行Checkpoint。阈值油`innodb_max_dirty_pages_pct`控制，值75表示脏页数据占比75%强制Checkpoint刷新部分脏页到磁盘。InnoDB 1.0.x之前默认值90，之后默认值75。

## 2.5 Master Thread工作方式

InnoDB存储引擎的主要工作在单独的后台线程Master Thread中完成。

### 2.5.1 InnoDB 1.0.x版本之前的Master Thread

​	Master Thread具有最高线程优先级。其内部由多个循环(loop)组成：主循环（loop）、后台循环（backgroup loop）、刷新循环（flush loop）、暂停循环（suspend loop）。Master Thread根据数据库运行状态在上述几种loop中进行切换。

​	大部分操作在主循环Loop中，其有两大部分的操作——每秒钟的操作和每10秒的操作。

---

​	每秒钟的操作包括：

+ **日志缓冲刷新到磁盘，即使这个事务还没有提交**(总是);

  这就是为什么再大的食物提交（commit）时间也是很短的。

+ 合并插入缓冲(可能);

  前一秒内发生的IO次数小于5次则认为当前IO压力小，才执行合并插入缓冲的操作。

+ 至多刷新100个Innodb的缓冲池中的脏页到磁盘(可能);

  缓冲池中脏页比例超过`buf_get_modified_ratio_pct`（默认90，表示90%）则将100个脏页写入磁盘中。

+ 如果当前没有用户活动，则切换到background loop(可能)。

---

​	每10秒的操作包括：

+ 刷新100个脏页到磁盘(可能的情况下);

  过去10秒内磁盘IO操作小于200则刷新100个脏页到磁盘。

+ 合并至多5个插人缓冲(总是);

  在每10秒触发了刷新100脏页到磁盘的动作后，InnoDB存储引擎会合并插入缓冲。

+ 将日志缓冲刷新到磁盘(总是);

  每10秒操作中合并插入缓冲后，将日志缓冲刷新到磁盘。

+ 删除无用的Undo页(总是);

  上述操作后，进一步执行full purge操作，删除无用Undo页。对表进行update、delete这类操作时，原先的行被标记为删除，由于一致性要求暂时保留行版本信息。full purge过程中，判断当前事务系统中已被删除的行是否可删除，比如有时候可能在查询操作时需要读取之前版本的undo信息，如果可以删除，InnoDB会立即将其删除。从源代码中可发现，InnoDB存储引擎在执行full purge操作时，每次最多尝试回收20个undo页。

+ 刷新100个或者10个脏页到磁盘(总是)

  随后，同样判断缓冲池脏页占比是否超过`buf_get_modified_ratio_pct`，超过则刷新100个脏页到磁盘，不超过则刷新10%脏页到磁盘。

---

background loop，若当前没有用户活动（数据库空闲时）活着数据库关闭（shutdown），即切换到该循环。background loop执行以下操作：

+ 删除无用的Undo页（总是）；
+ 合并20个插入缓冲（总是）；
+ 跳回到主循环（总是）；
+ 不断刷新100个页直到符合条件（可能，跳转到flush loop中完成）

若 flush loop中也没有什么事情可以做了， InnoDB存储引擎会切换到 suspend loop，将 Master Thread挂起，等待事件的发生。若用户启用(enable)了InnoDB存储引擎，却没有使用任何 InnoDB存储引擎的表，那么Master Thread总是处于挂起的状态。

### 2.5.2 InnoDB 1.2.x 版本之前的Master Thread

`innodb_io_capacity`

​	由于早期hard coding，即使磁盘能够在1秒内处理多于100个页的写入和20个插入缓冲的合并，Master Thread仍只会选择刷新100个脏页和合并20个插入缓冲。此时若发生宕机，由于很多数据没有刷新回磁盘，会导致恢复的时间较久，尤其是对于insert buffer来说。

​	InnoDB Plugin（从InnoDB 1.0.x版本开始）提供了参数`innodb_io_capacity`，用来表示磁盘IO的吞吐量（默认值200）。

+ 在合并插入缓冲时，合并插入缓冲的数量为`innodb_io_capacity`值的5%
+ 在从缓冲区刷新脏页时，刷新脏页的数量为`innodb_io_capacity`

​	若使用SSD类的磁盘或RAID磁盘阵列，存储设备拥有更高的IO速度，可以调高`innodb_io_capacity`的值。

---

`innodb_max_dirty_pages_pct`

​	`innodb_max_dirty_pages_pct`在InnoDB 1.0.x版本之前，默认值90，当脏页占据缓冲池90%，才会刷新100个脏页。如果有很大的内存活着数据库服务器的压力很大，这时刷新脏页的速度反而会降低。从InnoDB 1.0.x版本开始，`innodb_max_dirty_pages_pct`默认值变为75，加快刷新脏页的频率并保证磁盘IO的负载。

---

`innodb_adaptive_flushing`

​	随着`innodb_adaptive_flushing`参数的引入，InnoDB存储引擎会通过`buf_flush_get_desired_flush_rate`函数来判断需要刷新脏页最合适的数量。（`buf_flush_get_desired_flush_rate`通过判断产生重做日志redo log的速度来决定最合适的刷新脏页数量）。此时，即使脏页的比例小于`innodb_max_dirty_pages_pct`，也会刷新一定量的脏页。

---

`innodb_purge_batch_size`

​	InnoDB 1.0.x版本引入`innodb_purge_batch_size`参数，控制每次full purge回收的Undo页数量（默认值20，可动态修改）。

---

​	从InnoDB 1.0.x开始，通过命令`SHOW ENGINE INNODB STATUS`可查看当前Master Thread的状态信息。

### 2.5.3 InnoDB 1.2.x版本的Master Thread

​	InnoDB 1.2.x 再次对Master Thread进行优化。其中刷新脏页的操作，从Master Thread线程分离到一个单独的Page Cleaner Thread。

## * 2.6 InnoDB关键特性

​	InnoDB存储引擎的关键特性包括：

+ 插入缓冲（Insert Buffer）
+ 两次写（Double Write）
+ 自适应哈希索引（Adaptive Hash Index）
+ 异步IO（Async IO）
+ 刷新邻接页（Flush Neighbor Page）

### * 2.6.1 插入缓冲

1. Insert Buffer

   InnoDB缓冲池中有Insert Buffer信息，Insert Buffer和数据页一样，也是物理页的组成部分。

   对于非聚集索引的插入或更新操作，不是每次直接插入到索引页中，而是先判断插入的非聚集索引页是否在缓冲池中，若在则直接插入；若不在，则先放入一个Insert Buffer对象中，之后以一定的频率和情况进行Insert Buffer和辅助索引页子节点的merge（合并）操作，此时通常能合并多个插入操作（因为在同一个索引页中），大大提高了对非聚集索引插入的性能。

   使用Insert Buffer需要同时满足以下两个条件：

   + **索引是辅助索引（secondary index）**
   + **索引不是唯一的（unique）**

   然而，应用程序进行大量的插入操作，这些数据如果都涉及不唯一的非聚集索引，使用到Insert Buffer。此时MySQL数据库发生宕机，就会有大量的Insert Buffer没有合并到实际的非聚集索引中，数据恢复时间在极端情况下需要几个小时。

   **辅助索引不能是唯一的，因为在插入缓冲时，数据库并不去查找索引页来判断插入的记录的唯一性。如果去查找肯定又会有离散读取的情况发生，从而导致 Insert Buffer失去了意义。**

   通过`SHOW ENGINE INNODB STATUS`可查看插入缓冲的信息。

   修改`IBUF_POOL_SIZE_PER_MAX_SIZE`限制插入缓冲的大小（设置3表示最大只能使用1/3的缓冲池内存）。

2. Changer Buffer

   InnoDB 1.0.x版本开始，引入Change Buffer，可视为Insert Buffer的升级。该版本起，InnoDB可对DML操作——INSERT、DELETE、UPDATE都进行缓冲，分别是Insert Buffer、Delete Buffer、Purge Buffer。

   **和Insert Buffer一样，Change Buffer适用对象仍是非唯一的辅助索引**。

   对一条记录进行UPDATE操作可分为两个过程：

   + 将记录标记为已删除（Delete Buufer对应这一过程）
   + 真正将记录删除（Purge Buffer对应这一过程）

   *InnoDB存储引擎提供了参数`innodb_change_buffering`，用来开启各种Buffer的选项，默认值all。该参数的可选值为：inserts、deletes、purges、changes、all、none。insets、deletes、purge对应前面讨论的3种情况。change表示启用inserts和deletes，all表示启用所有，none表示都不启用。*

   InnoDB 1.2.x版本开始，通过参数`innodb_change_buffer_max_size`可控制Change Buffer最大使用内存的数量（默认值25，表示最多使用1/4缓冲池内存空间，该参数最大有效值为50）。

3. Insert Buffer的内部实现

   Insert Buffer的数据结构是一棵B+树。MySQL4.1之前，每张表有一颗Insert Buffer B+树。后续版本中，全局只有一颗Insert Buffer B+树，负责对所有的表的辅助索引进行Insert Buffer。该B+树存放在共享表空间中，默认也就是ibdata1中。

   <u>试图通过独立表空间ibd文件恢复表中数据时，往往会导致CHECK TABLE失败。这是因为表的辅助索引中的数据可能还在 Insert Buffer中，也就是共享表空间中，所以通过ibd文件进行恢复后，还需要进行REPAIR TABLE操作来重建表上所有的辅助索引</u>。

   Insert Buffer是一棵B+树，因此其也由叶节点和非叶节点组成。非叶节点存放的是查询的 search key(键值)，其构造如图2-3所示。

   ![image](https://yqfile.alicdn.com/ea0da5467e09edbf2cabb8503c30bf37fa5415db.png)

   search key一共占用9个字节，其中 space表示待插入记录所在表的表空间id，在Innodb存储引擎中，每个表有一个唯一的space id，可以通过space id查询得知是哪张表。space占用4字节。marker占用1字节，它是用来兼容老版本的 Insert Buffer。offset表示页所在的偏移量，占用4字节。

   当一个辅助索引要插入到页（space，offset）时，如果这个页不在缓冲池中，那么InnoDB存储引擎首先根据上述规则构造一个search key，接下来查询Insert Buffer这棵B+树，然后再将这条记录插入到Insert Buffer B+树的叶子节点中。

   对于插入到Insert Buffer B+树叶子节点的记录（如图2-4所示），并不是直接将待插入的记录插入，而是需要根据如下的规则进行构造：

   ![image](https://yqfile.alicdn.com/f5955632cb034e946706f0bcc33e7b8529f6e087.png)

   space、marker、page_no字段和之前非叶节点中的含义相同，一共占用9字节。第4个字段metadata占用4字节，其存储的内容如表2-2所示。

   ![image](https://yqfile.alicdn.com/f6938a3faa3ebd43e9ed2b20b6dc40f2b8b9edfd.png)

   IBUF_REC_OFFSET_COUNT是保存两个字节的整数，用来排序每个记录进入Insert Buffer的顺序。因为从InnoDB1.0.x开始支持Change Buffer，所以这个值同样记录进入Insert Buffer的顺序。通过这个顺序回放（replay）才能得到记录的正确值。

   从Insert Buffer叶子节点的第5列开始，就是实际插入记录的各个字段了。因此较之原插入记录，Insert Buffer B+树的叶子节点记录需要额外13字节的开销。

   因为启用Insert Buffer索引后，辅助索引页（space，page_no）中的记录可能被插入到Insert Buffer B+树中，所以为了保证每次Merge Insert Buffer页必须成功，还需要有一个特殊的页用来标记每个辅助索引页（space，page_no）的可用空间。这个页的类型为Insert Buffer Bitmap。

   每个Insert Buffer Bitmap页用来追踪16384个辅助索引页，也就是256个区（Extent）。每个Insert Buffer Bitmap页都在16384个页的第二个页中。关于Insert Buffer Bitmap页的作用会在下一小节中详细介绍。

   每个辅助索引页在Insert Buffer Bitmap页中占用4位（bit），由表2-3中的三个部分组成。

   ![image](https://yqfile.alicdn.com/ff1179b0c12c7608e00ed74445b294d8b6082d74.png)

4. Merge Insert Buffer

   概括地说，Merge Insert Buffer的操作可能发生在以下几种情况下：

   + 辅助索引页被读取到缓冲池时； 

   + Insert Buffer Bitmap页追踪到该辅助索引页已无可用空间时；

   + Master Thread。

   第一种情况为当辅助索引页被读取到缓冲池中时，例如这在执行正常的SELECT查询操作，这时需要检查Insert Buffer Bitmap页，然后确认该辅助索引页是否有记录存放于Insert Buffer B+树中。若有，则将Insert Buffer B+树中该页的记录插入到该辅助索引页中。可以看到对该页多次的记录操作通过一次操作合并到了原有的辅助索引页中，因此性能会有大幅提高。

   Insert Buffer Bitmap页用来追踪每个辅助索引页的可用空间，并至少有1/32页的空间。若插入辅助索引记录时检测到插入记录后可用空间会小于1/32页，则会强制进行一个合并操作，即强制读取辅助索引页，将Insert Buffer B+树中该页的记录及待插入的记录插入到辅助索引页中。这就是上述所说的第二种情况。

   还有一种情况，之前在分析Master Thread时曾讲到，在Master Thread线程中每秒或每10秒会进行一次Merge Insert Buffer的操作，不同之处在于每次进行merge操作的页的数量不同。

   在Master Thread中，执行merge操作的不止是一个页，而是根据srv_innodb_io_capactiy的百分比来决定真正要合并多少个辅助索引页。但InnoDB存储引擎又是根据怎样的算法来得知需要合并的辅助索引页呢？

   在Insert Buffer B+树中，辅助索引页根据（space，offset）都已排序好，故可以根据（space，offset）的排序顺序进行页的选择。然而，对于Insert Buffer页的选择，InnoDB存储引擎并非采用这个方式，它随机地选择Insert Buffer B+树的一个页，读取该页中的space及之后所需要数量的页。该算法在复杂情况下应有更好的公平性。同时，若进行merge时，要进行merge的表已经被删除，此时可以直接丢弃已经被Insert/Change Buffer的数据记录。

### * 2.6.2 两次写

​	如果说Insert Buffer带给InnoDB存储引擎的是性能上的提升，那么doublewrite（两次写）带给InnoDB存储引擎的是数据页的可靠性。

​	当发生数据库宕机时，可能InnoDB存储引擎正在写入某个页到表中，而这个页只写了一部分，比如16KB的页，只写了前4KB，之后就发生了宕机，这种情况被称为**部分写失效（partial page write）**。在InnoDB存储引擎未使用doublewrite技术前，曾经出现过因为部分写失效而导致数据丢失的情况。

​	有经验的DBA也许会想，如果发生写失效，可以通过重做日志进行恢复。这是一个办法。但是必须清楚地认识到，重做日志中记录的是对页的物理操作，如偏移量800，写'aaaa'记录。如果这个页本身已经发生了损坏，再对其进行重做是没有意义的。这就是说，**在应用（apply）重做日志前，用户需要一个页的副本，当写入失效发生时，先通过页的副本来还原该页，再进行重做，这就是doublewrite**。在InnoDB存储引擎中doublewrite的体系架构如图2-5所示。

![image](https://yqfile.alicdn.com/c1eb90077242e090818e14fc626ec87420d3f430.png)

​	doublewrite由两部分组成，一部分是内存中的doublewrite buffer，大小为2MB，另一部分是物理磁盘上共享表空间中连续的128个页，即2个区（extent），大小同样为2MB。**在对缓冲池的脏页进行刷新时，并不直接写磁盘，而是会通过memcpy函数将脏页先复制到内存中的doublewrite buffer，之后通过doublewrite buffer再分两次，每次1MB顺序地写入共享表空间的物理磁盘上，然后马上调用fsync函数，同步磁盘，避免缓冲写带来的问题**。在这个过程中，因为doublewrite页是连续的，因此这个过程是顺序写的，开销并不是很大。在完成doublewrite页的写入后，再将doublewrite buffer中的页写入各个表空间文件中，此时的写入则是离散的。

​	**如果操作系统在将页写入磁盘的过程中发生了崩溃，在恢复过程中，InnoDB存储引擎可以从共享表空间中的doublewrite中找到该页的一个副本，将其复制到表空间文件，再应用重做日志**。

​	参数skip_innodb_doublewrite可以禁止使用doublewrite功能，这时可能会发生前面提及的写失效问题。不过如果用户有多个从服务器（slave server），需要提供较快的性能（如在slaves erver上做的是RAID0），也许启用这个参数是一个办法。不过**对于需要提供数据高可靠性的主服务器（master server），任何时候用户都应确保开启doublewrite功能**。

> 注意：有些文件系统本身就提供了部分写失效的防范机制，如ZFS文件系统。在这种情况下，用户就不要启用doublewrite了。

### * 2.6.3 自适应哈希索引

​	哈希（hash）是一种非常快的查找方法，在一般情况下这种查找的时间复杂度为O(1)，即一般仅需要一次查找就能定位数据。而B+树的查找次数，取决于B+树的高度，**在生产环境中，B+树的高度一般为3～4层，故需要3～4次的查询**。

​	**InnoDB存储引擎会监控对表上各索引页的查询。如果观察到建立哈希索引可以带来速度提升，则建立哈希索引，称之为自适应哈希索引（Adaptive Hash Index，AHI）**。AHI是通过缓冲池的B+树页构造而来，因此建立的速度很快，而且不需要对整张表构建哈希索引。<u>InnoDB存储引擎会自动根据访问的频率和模式来自动地为某些热点页建立哈希索引</u>。

​	**AHI有一个要求，即对这个页的连续访问模式必须是一样的**。例如对于（a，b）这样的联合索引页，其访问模式可以是以下情况：

+ WHERE a=xxx

+ WHERE a=xxx and b=xxx

​	访问模式一样指的是查询的条件一样，<u>若交替进行上述两种查询，那么InonDB存储引擎不会对该页构造AHI</u>。此外AHI还有如下的要求：

+ 以该模式访问了100次
+ 页通过该模式访问了N次，其中N=页中记录*1/16

​	<u>根据InnoDB存储引擎官方的文档显示，启用AHI后，读取和写入速度可以提高2倍，辅助索引的连接操作性能可以提高5倍</u>。毫无疑问，AHI是非常好的优化模式，其设计思想是数据库自优化的（self-tuning），即无需DBA对数据库进行人为调整。

​	通过命令SHOW ENGINE INNODB STATUS可以看到当前AHI的使用状况。

​	值得注意的是，**哈希索引只能用来搜索等值的查询**，如SELECT*FROM table WHERE index_col='xxx'。而对于其他查找类型，如范围查找，是不能使用哈希索引的。

​	AHI是由InnoDB存储引擎控制的。用户可以通过观察SHOW ENGINE INNODB STATUS的结果及参数`innodb_adaptive_hash_index`来考虑是禁用或启动此特性，**默认AHI为开启状态**。

### 2.6.4 异步IO

​	为了提高磁盘操作性能，当前的数据库系统都采用**异步IO**（Asynchronous IO，AIO）的方式来处理磁盘操作。InnoDB存储引擎亦是如此。

​	一次SQL查询包括扫描索引页、数据页等操作，每个操作一般需要多次IO，AIO能够一次发起多个IO请求，最后在统一整合IO响应结果。另外<u>AIO能够进行IO Merge操作，将几个请求访问地址连续的IO请求合并成一个</u>，提高IOPS性能。

​	在InnoDB1.1.x之前，AIO的实现通过InnoDB存储引擎中的代码来模拟实现。而从InnoDB 1.1.x开始（InnoDB Plugin不支持），提供了内核级别AIO的支持，称为Native AIO（Mac OS不支持）。因此在编译或者运行该版本MySQL时，需要libaio库的支持。	

> 参数`innodb_use_native_aio`用来控制是否启用Native AIO，在Linux操作系统下，默认值为ON。
>
> 用户可以通过开启和关闭Native AIO功能来比较InnoDB性能的提升。官方的测试显示，启用Native AIO，恢复速度可以提高75%。

​	**在InnoDB存储引擎中，read ahead方式的读取都是通过AIO完成，脏页的刷新，即磁盘的写入操作则全部由AIO完成**。

### 2.6.5 刷新邻接页

​	InnoDB存储引擎还提供了Flush Neighbor Page（刷新邻接页）的特性。其工作原理为：当刷新一个脏页时，InnoDB存储引擎会检测该页所在区（extent）的所有页，如果是脏页，那么一起进行刷新。这样做的好处显而易见，通过AIO可以将多个IO写入操作合并为一个IO操作，故该工作机制在传统机械磁盘下有着显著的优势。但是需要考虑到下面两个问题：

+ 是不是可能将不怎么脏的页进行了写入，而该页之后又会很快变成脏页？

+ 固态硬盘有着较高的IOPS，是否还需要这个特性？

​	为此，InnoDB存储引擎从1.2.x版本开始提供了参数`innodb_flush_neighbors`，用来控制是否启用该特性。对于传统机械硬盘建议启用该特性，而对于固态硬盘有着超高IOPS性能的磁盘，则建议将该参数设置为0，即关闭此特性。

## 2.7 启动、关闭与恢复

​	在关闭时，参数`innodb_fast_shutdown`影响着表的存储引擎为InnoDB的行为。该参数可取值为0、1、2，默认值为1。

+ 0表示在MySQL数据库关闭时，InnoDB需要完成所有的full purge和merge insert buffer，并且将所有的脏页刷新回磁盘。这需要一些时间，有时甚至需要几个小时来完成。如果在进行InnoDB升级时，必须将这个参数调为0，然后再关闭数据库。
+ 1是参数`innodb_fast_shutdown`的默认值，表示不需要完成上述的full purge和merge insert buffer操作，但是在缓冲池中的一些数据脏页还是会刷新回磁盘。
+ 2表示不完成full purge和merge insert buffer操作，也不将缓冲池中的数据脏页写回磁盘，而是将日志都写入日志文件。这样不会有任何事务的丢失，但是下次MySQL数据库启动时，会进行恢复操作（recovery）。

​	当正常关闭MySQL数据库时，下次的启动应该会非常“正常”。但是如果没有正常地关闭数据库，<u>如用kill命令关闭数据库，在MySQL数据库运行中重启了服务器，或者在关闭数据库时，将参数`innodb_fast_shutdown`设为了2时，下次MySQL数据库启动时都会对InnoDB存储引擎的表进行恢复操作</u>。

​	参数`innodb_force_recovery`影响了整个InnoDB存储引擎恢复的状况。该参数值默认为0，代表当发生需要恢复时，进行所有的恢复操作，当不能进行有效恢复时，如数据页发生了corruption，MySQL数据库可能发生宕机（crash），并把错误写入错误日志中去。

​	但是，在某些情况下，可能并不需要进行完整的恢复操作，因为用户自己知道怎么进行恢复。比如在对一个表进行alter table操作时发生意外了，数据库重启时会对InnoDB表进行回滚操作，对于一个大表来说这需要很长时间，可能是几个小时。这时用户可以自行进行恢复，如可以把表删除，从备份中重新导入数据到表，可能这些操作的速度要远远快于回滚操作。

​	参数`innodb_force_recovery`还可以设置为6个非零值：1～6。大的数字表示包含了前面所有小数字表示的影响。具体情况如下：

+ 1(SRV_FORCE_IGNORE_CORRUPT)：忽略检查到的corrupt页。
+ 2(SRV_FORCE_NO_BACKGROUND)：阻止Master Thread线程的运行，如Master Thread线程需要进行full purge操作，而这会导致crash。
+ 3(SRV_FORCE_NO_TRX_UNDO)：不进行事务的回滚操作。
+ 4(SRV_FORCE_NO_IBUF_MERGE)：不进行插入缓冲的合并操作。
+ 5(SRV_FORCE_NO_UNDO_LOG_SCAN)：不查看撤销日志（Undo Log），InnoDB存储引擎会将未提交的事务视为已提交。
+ 6(SRV_FORCE_NO_LOG_REDO)：不进行前滚的操作。

​	需要注意的是，**在设置了参数innodb_force_recovery大于0后，用户可以对表进行select、create和drop操作，但insert、update和delete这类DML操作是不允许的**。

## 2.8 小结

+ InnoDB存储引擎及其体系结构
+ InnoDB存储引擎的关键特性
+ 启动和关闭MYSQL时一些配置文件参数对InnoDB存储引擎的影响

​	第3章开始介绍 MYSQL的文件，包括 My SQL本身的文件和与InnoDB存储引擎本身有关的文件。之后本书将介绍基于InnoDB存储引擎的表，并揭示内部的存储构造。

# 3. 文件

​	本章将分析构成MySQL数据库和InnoDB存储引擎表的各种类型文件。这些文件有以下这些。

+ 参数文件：告诉MySQL实例启动时在哪里可以找到数据库文件，并且指定某些初始化参数，这些参数定义了某种内存结构的大小等设置，还会介绍各种参数的类型。
+ 日志文件：用来记录MySQL实例对某种条件做出响应时写入的文件，如错误日志文件、二进制日志文件、慢查询日志文件、查询日志文件等。
+ socket文件：当用UNIX域套接字方式进行连接时需要的文件。
+ pid文件：MySQL实例的进程ID文件。
+ MySQL表结构文件：用来存放MySQL表结构定义文件。
+ 存储引擎文件：因为MySQL表存储引擎的关系，每个存储引擎都会有自己的文件来保存各种数据。这些存储引擎真正存储了记录和索引等数据。本章主要介绍与InnoDB有关的存储引擎文件。

## 3.1 参数文件

​	MySQL实例启动时，数据库会按照一定的顺序读取配置文件，用户可通过命令`mysql--help | grep my.cnf`来寻找。

### 3.1.1 什么是参数

​	可以通过命令`SHOW VARIABLES`查看数据库中的所有参数，也可以通过LIKE来过滤参数名。从MySQL 5.1版本开始，还可以通过`information_schema`架构下的`GLOBAL_VARIABLES`视图来进行查找。

### 3.1.2 参数类型

​	MySQL数据库中的参数可以分为两类：

+ 动态（dynamic）参数
+ 静态（static）参数

​	**动态参数意味着可以在MySQL实例运行中进行更改，静态参数说明在整个实例生命周期内都不得进行更改，就好像是只读（read only）的**。可以通过SET命令对动态的参数值进行修改，SET的语法如下：

```
SET 
| [global | session] system_var_name= expr
| [@@global. | @@session. | @@]system_var_name= expr
```

​	**这里可以看到global和session关键字，<u>它们表明该参数的修改是基于当前会话还是整个实例的生命周期</u>**。有些动态参数只能在会话中进行修改，如autocommit；而有些参数修改完后，在整个实例生命周期中都会生效，如binlog_cache_size；而有些参数既可以在会话中又可以在整个实例的生命周期内生效，如read_buffer_size。

​	**这次把read_buffer_size全局值更改为1MB，而当前会话的read_buffer_size的值还是512KB**。<u>这里需要注意的是，对变量的全局值进行了修改，在**这次的实例生命周期内**都有效，但MySQL实例本身并不会对参数文件中的该值进行修改</u>。也就是说，**在下次启动时MySQL实例还是会读取参数文件。若想在数据库实例下一次启动时该参数还是保留为当前修改的值，那么用户必须去修改参数文件**。要想知道MySQL所有动态变量的可修改范围，可以参考MySQL官方手册的Dynamic System Variables的相关内容。

> [mysql的session、global变量详解_勤天的博客-CSDN博客_mysql session](https://blog.csdn.net/demored/article/details/122962605)

## 3.2 日志文件

日志文件记录了影响MySQL数据库的各种类型活动。MySQL数据库中常见的日志文件有：

+ 错误日志（error log）
+ 二进制日志（binlog）
+ 慢查询日志（slow query log）

+ 查询日志（log）

这些日志文件可以帮助DBA对MySQL数据库的运行状态进行诊断，从而更好地进行数据库层面的优化。

### 3.2.1 错误日志

​	<u>错误日志文件对MySQL的启动、运行、关闭过程进行了记录</u>。MySQL DBA在遇到问题时应该首先查看该文件以便定位问题。该文件不仅记录了所有的错误信息，也记录一些警告信息或正确的信息。用户可以通过命令SHOW VARIABLES LIKE 'log_error'来定位该文件。

​	当出现MySQL数据库不能正常启动时，第一个必须查找的文件应该就是错误日志文件，该文件记录了错误信息，能很好地指导用户发现问题。

### 3.2.2 慢查询日志

​	3.2.1小节提到可以通过错误日志得到一些关于数据库优化的信息，而慢查询日志（slow log）可帮助DBA定位可能存在问题的SQL语句，从而进行SQL语句层面的优化。

​	在默认情况下，MySQL数据库并不启动慢查询日志，用户需要手工将这个参数设为ON。

​	另一个和慢查询日志有关的参数是`log_queries_not_using_indexes`，如果运行的SQL语句没有使用索引，则MySQL数据库同样会将这条SQL语句记录到慢查询日志文件。

​	MySQL 5.6.5版本开始新增了一个参数`log_throttle_queries_not_using_indexes`，用来表示每分钟允许记录到slow log的且未使用索引的SQL语句次数。该值默认为0，表示没有限制。在生产环境下，若没有使用索引，此类SQL语句会频繁地被记录到slow log，从而导致slow log文件的大小不断增加，故DBA可通过此参数进行配置。

​	DBA可以通过慢查询日志来找出有问题的SQL语句，对其进行优化。然而随着MySQL数据库服务器运行时间的增加，可能会有越来越多的SQL查询被记录到了慢查询日志文件中，此时要分析该文件就显得不是那么简单和直观的了。而这时MySQL数据库提供的`mysqldumpslow`命令，可以很好地帮助DBA解决该问题。

​	MySQL 5.1开始可以将慢查询的日志记录放入一张表中，这使得用户的查询更加方便和直观。慢查询表在mysql架构下，名为slow_log。

​	InnoSQL版本加强了对于SQL语句的捕获方式。在原版MySQL的基础上在slow log中增加了**对于逻辑读取（logical reads）和物理读取（physical reads）的统计**。这里的物理读取是指从磁盘进行IO读取的次数，逻辑读取包含所有的读取，不管是磁盘还是缓冲池。

​	用户可以通过额外的参数`long_query_io`将超过指定逻辑IO次数的SQL语句记录到slow log中。该值默认为100，即表示对于逻辑读取次数大于100的SQL语句，记录到slow log中。而为了兼容原MySQL数据库的运行方式，还添加了参数`slow_query_type`，用来表示启用slow log的方式，可选值为：

+ 0表示不将SQL语句记录到slow log
+ 1表示根据运行时间将SQL语句记录到slow log
+ 2表示根据逻辑IO次数将SQL语句记录到slow log
+ 3表示根据运行时间及逻辑IO次数将SQL语句记录到slow log

### 3.2.3 查询日志

​	查询日志记录了所有对MySQL数据库请求的信息，无论这些请求是否得到了正确的执行。默认文件名为：`主机名.log`。

​	同样地，从MySQL 5.1开始，可以将查询日志的记录放入mysql架构下的general_log表中，该表的使用方法和前面小节提到的slow_log基本一样。

### * 3.2.4 二进制日志

​	**二进制日志（binary log）记录了对MySQL数据库执行更改的所有操作，但是不包括SELECT和SHOW这类操作，因为这类操作对数据本身并没有修改**。然而，若操作本身并没有导致数据库发生变化，那么该操作可能也会写入二进制日志。

​	如果用户想记录SELECT和SHOW操作，那只能使用查询日志，而不是二进制日志。此外，二进制日志还包括了执行数据库更改操作的时间等其他额外信息。总的来说，二进制日志主要有以下几种作用。

+ 恢复（recovery）：某些数据的恢复需要二进制日志，例如，在一个数据库全备文件恢复后，用户可以通过二进制日志进行point-in-time的恢复。
+ 复制（replication）：其原理与恢复类似，通过复制和执行二进制日志使一台远程的MySQL数据库（一般称为slave或standby）与一台MySQL数据库（一般称为master或primary）进行实时同步。
+ 审计（audit）：用户可以通过二进制日志中的信息来进行审计，判断是否有对数据库进行注入的攻击。

> 通过配置参数log-bin［=name］可以启动二进制日志。如果不指定name，则默认二进制日志文件名为主机名，后缀名为二进制日志的序列号，所在路径为数据库所在目录（datadir）。

​	二进制日志文件在默认情况下并没有启动，需要手动指定参数来启动。可能有人会质疑，开启这个选项是否会对数据库整体性能有所影响。不错，开启这个选项的确会影响性能，但是性能的损失十分有限。根据MySQL官方手册中的测试表明，开启二进制日志会使性能下降1%。但考虑到可以使用复制（replication）和point-in-time的恢复，这些性能损失绝对是可以且应该被接受的。
以下配置文件的参数影响着二进制日志记录的信息和行为：

+ max_binlog_size
+ binlog_cache_size
+ sync_binlog
+ binlog-do-db
+ binlog-ignore-db
+ log-slave-update
+ binlog_format

​	参数`max_binlog_size`指定了单个二进制日志文件的最大值，如果超过该值，则产生新的二进制日志文件，后缀名+1，并记录到.index文件。从MySQL 5.0开始的默认值为1073741824，代表1G（在之前版本中`max_binlog_size`默认大小为1.1G）。

​	**当使用事务的表存储引擎（如InnoDB存储引擎）时，所有未提交（uncommitted）的二进制日志会被记录到一个缓存中去，等该事务提交（committed）时直接将缓冲中的二进制日志写入二进制日志文件**，而该缓冲的大小由`binlog_cache_size`决定，默认大小为32K。

​	此外，**`binlog_cache_size`是基于会话（session）**的，也就是说，当一个线程开始一个事务时，MySQL会自动分配一个大小为`binlog_cache_size`的缓存，因此该值的设置需要相当小心，不能设置过大。<u>当一个事务的记录大于设定的`binlog_cache_size`时，MySQL会把缓冲中的日志写入一个临时文件中，因此该值又不能设得太小</u>。

> 通过`SHOW GLOBAL STATUS`命令查看`binlog_cache_use`、`binlog_cache_disk_use`的状态，可以判断当前`binlog_cache_size`的设置是否合适。`Binlog_cache_use`记录了使用缓冲写二进制日志的次数，`binlog_cache_disk_use`记录了使用临时文件写二进制日志的次数。

​	**在默认情况下，二进制日志并不是在每次写的时候同步到磁盘（用户可以理解为缓冲写）。因此，当数据库所在操作系统发生宕机时，可能会有最后一部分数据没有写入二进制日志文件中，这会给恢复和复制带来问题**。

> 参数sync_binlog=［N］表示每写缓冲多少次就同步到磁盘。如果将N设为1，即sync_binlog=1表示采用同步写磁盘的方式来写二进制日志，这时写操作不使用操作系统的缓冲来写二进制日志。sync_binlog的默认值为0，如果使用InnoDB存储引擎进行复制，并且想得到最大的高可用性，建议将该值设为ON。不过该值为ON时，确实会对数据库的IO系统带来一定的影响。

​	但是，即使将sync_binlog设为1，还是会有一种情况导致问题的发生。当使用InnoDB存储引擎时，在一个事务发出COMMIT动作之前，由于sync_binlog为1，因此会将二进制日志立即写入磁盘。**如果这时已经写入了二进制日志，但是提交还没有发生，并且此时发生了宕机，那么在MySQL数据库下次启动时，由于COMMIT操作并没有发生，这个事务会被回滚掉。但是二进制日志已经记录了该事务信息，不能被回滚**。**这个问题可以通过将参数`innodb_support_xa`设为1来解决，虽然`innodb_support_xa`与XA事务有关，但它同时也确保了二进制日志和InnoDB存储引擎数据文件的同步**。

> 参数binlog-do-db和binlog-ignore-db表示需要写入或忽略写入哪些库的日志。默认为空，表示需要同步所有库的日志到二进制日志。
>
> 如果当前数据库是复制中的slave角色，则它不会将从master取得并执行的二进制日志写入自己的二进制日志文件中去。如果需要写入，要设置log-slave-update。如果需要搭建master=>slave=>slave架构的复制，则必须设置该参数。

​	**`binlog_format`参数十分重要，它影响了记录二进制日志的格式**。在MySQL 5.1版本之前，没有这个参数。所有二进制文件的格式都是基于SQL语句（statement）级别的，因此基于这个格式的二进制日志文件的复制（Replication）和Oracle 的逻辑Standby有点相似。同时，对于复制是有一定要求的。如在主服务器运行rand、uuid等函数，又或者使用触发器等操作，这些都可能会导致主从服务器上表中数据的不一致（not sync）。另一个影响是，会发现**InnoDB存储引擎的默认事务隔离级别是REPEATABLE READ**。这其实也是因为二进制日志文件格式的关系，如果使用READ COMMITTED的事务隔离级别（大多数数据库，如Oracle，Microsoft SQL Server数据库的默认隔离级别），会出现类似丢失更新的现象，从而出现主从数据库上的数据不一致。

​	MySQL 5.1开始引入了binlog_format参数，该参数可设的值有STATEMENT、ROW和MIXED。
（1）STATEMENT格式和之前的MySQL版本一样，二进制日志文件记录的是日志的逻辑SQL语句。
（2）**在ROW格式下，二进制日志记录的不再是简单的SQL语句了，而是记录表的行更改情况**。基于ROW格式的复制类似于Oracle的物理Standby（当然，还是有些区别）。同时，对上述提及的Statement格式下复制的问题予以解决。从MySQL 5.1版本开始，如果设置了binlog_format为ROW，可以将InnoDB的事务隔离基本设为READ COMMITTED，以获得更好的并发性。
（3）**在MIXED格式下，MySQL默认采用STATEMENT格式进行二进制日志文件的记录，但是在一些情况下会使用ROW格式，可能的情况有**：
1）表的存储引擎为NDB，这时对表的DML操作都会以ROW格式记录。
2）<u>使用了UUID()、USER()、CURRENT_USER()、FOUND_ROWS()、ROW_COUNT()等不确定函数</u>。
3）使用了INSERT DELAY语句。
4）使用了用户定义函数（UDF）。
5）使用了临时表（temporary table）。
此外，`binlog_format`参数还有对于存储引擎的限制，如表3-1所示。
![image](https://yqfile.alicdn.com/919af1f916ebe40d7088c1576f5cc15a828a7cfb.png)

**`binlog_format`是动态参数**，因此可以在数据库运行环境下进行更改。

> 当然，也可以将全局的binlog_format设置为想要的格式，不过通常这个操作会带来问题，运行时要确保更改后不会对复制带来影响。

​	**<u>在通常情况下，我们将参数binlog_format设置为ROW，这可以为数据库的恢复和复制带来更好的可靠性。但是不能忽略的一点是，这会带来二进制文件大小的增加，有些语句下的ROW格式可能需要更大的容量</u>**。

​	**<u>将参数`binlog_format`设置为ROW，会对磁盘空间要求有一定的增加。而由于复制是采用传输二进制日志方式实现的，因此复制的网络开销也有所增加</u>**。

> 图书作者在某张17MB大小的表。在`binlog_format='STATEMENT'`时执行某UPDATE只增加binlog日志大小约200字节，但是`binlog_format='ROW'`执行相同UPDATE语句增加13MB大小的日志。**因为这时MySQL数据库不再将逻辑的SQL操作记录到二进制日志中，而是记录对于每行的更改。**

> 二进制日志文件的文件格式为二进制（好像有点废话），不能像错误日志文件、慢查询日志文件那样用cat、head、tail等命令来查看。要查看二进制日志文件的内容，必须通过MySQL提供的工具mysqlbinlog。对于STATEMENT格式的二进制日志文件，在使用mysqlbinlog后，看到的就是执行的逻辑SQL语句。

## 3.3 套接字文件

​	前面提到过，在UNIX系统下本地连接MySQL可以采用UNIX域套接字方式，这种方式需要一个套接字（socket）文件。套接字文件可由参数socket控制。一般在/tmp目录下，名为`mysql.sock`。

## 3.4 pid文件

​	当MySQL实例启动时，会将自己的进程ID写入一个文件中——该文件即为pid文件。该文件可由参数`pid_file`控制，默认位于数据库目录下，文件名为`主机名.pid`。

## 3.5 表结构定义文件

​	因为MySQL插件式存储引擎的体系结构的关系，MySQL数据的存储是根据表进行的，每个表都会有与之对应的文件。但**不论表采用何种存储引擎，MySQL都有一个以frm为后缀名的文件，这个文件记录了该表的表结构定义**。

​	frm还用来存放视图的定义，如用户创建了一个v_a视图，那么对应地会产生一个v_a.frm文件，用来记录视图的定义，该文件是文本文件，可以直接使用cat命令进行查看。

## 3.6 InnoDB存储引擎文件

​	之前介绍的文件都是MySQL数据库本身的文件，和存储引擎无关。除了这些文件外，每个表存储引擎还有其自己独有的文件。本节将具体介绍与InnoDB存储引擎密切相关的文件，这些文件包括重做日志文件、表空间文件。

### 3.6.1 表空间文件

​	**InnoDB采用将存储的数据按表空间（tablespace）进行存放的设计**。在默认配置下会有一个初始大小为10MB，名为ibdata1的文件。该文件就是默认的表空间文件（tablespace file），用户可以通过参数innodb_data_file_path对其进行设置，格式如下：

```shell
innodb_data_file_path=datafile_spec1[;datafile_spec2]...
```

​	用户可以通过多个文件组成一个表空间，同时制定文件的属性，如：

```
[mysqld]
innodb_data_file_path = /db/ibdata1:2000M;/dr2/db/ibdata2:2000M:autoextend
```

​	这里将/db/ibdata1和/dr2/db/ibdata2两个文件用来组成表空间。<u>若这两个文件位于不同的磁盘上，磁盘的负载可能被平均，因此可以提高数据库的整体性能</u>。同时，两个文件的文件名后都跟了属性，表示文件idbdata1的大小为2000MB，文件ibdata2的大小为2000MB，如果用完了这2000MB，该文件可以自动增长（autoextend）。

> 设置`innodb_data_file_path`参数后，所有基于InnoDB存储引擎的表的数据都会记录到该共享表空间中。若设置了参数`innodb_file_per_table`，则用户可以将每个基于InnoDB存储引擎的表产生一个独立表空间。独立表空间的命名规则为：`表名.ibd`。通过这样的方式，用户不用将所有数据都存放于默认的表空间中。

​	**设置`innodb_file_per_table=ON`时，这些单独的表空间文件仅存储该表的数据、索引和插入缓冲BITMAP等信息，其余信息还是存放在默认的表空间中**。图3-1显示了InnoDB存储引擎对于文件的存储方式：

![image](https://yqfile.alicdn.com/b642d3dd798919aed8ab11833c05a796f9616514.png)

### * 3.6.2 重做日志文件

​	在默认情况下，在InnoDB存储引擎的数据目录下会有两个名为`ib_logfile0`和`ib_logfile1`的文件。在MySQL官方手册中将其称为InnoDB存储引擎的日志文件，不过更准确的定义应该是重做日志文件（redo log file）。为什么强调是重做日志文件呢？因为重做日志文件对于InnoDB存储引擎至关重要，它们记录了对于InnoDB存储引擎的事务日志。

​	<u>当实例或介质失败（media failure）时，重做日志文件就能派上用场。例如，数据库由于所在主机掉电导致实例失败，InnoDB存储引擎会使用重做日志恢复到掉电前的时刻，以此来保证数据的完整性</u>。

​	**每个InnoDB存储引擎至少有1个重做日志文件组（group），每个文件组下至少有2个重做日志文件，如默认的`ib_logfile0`和`ib_logfile1`**。<u>为了得到更高的可靠性，用户可以设置多个的镜像日志组（mirrored log groups），将不同的文件组放在不同的磁盘上，以此提高重做日志的高可用性。在日志组中每个重做日志文件的大小一致，并以**循环写入**的方式运行</u>。InnoDB存储引擎先写重做日志文件1，当达到文件的最后时，会切换至重做日志文件2，再当重做日志文件2也被写满时，会再切换到重做日志文件1中。图3-2显示了一个拥有3个重做日志文件的重做日志文件组。

![image](https://yqfile.alicdn.com/55f1a1c1a5eacc5301d4fbfb3e7503ea959b05f5.png)

下列参数影响着重做日志文件的属性：

+ innodb_log_file_size
+ innodb_log_files_in_group
+ innodb_mirrored_log_groups
+ innodb_log_group_home_dir

> 参数innodb_log_file_size指定每个重做日志文件的大小。在InnoDB1.2.x版本之前，重做日志文件总的大小不得大于等于4GB，而1.2.x版本将该限制扩大为了512GB。
>
> 参数innodb_log_files_in_group指定了日志文件组中重做日志文件的数量，默认为2。
>
> 参数innodb_mirrored_log_groups指定了日志镜像文件组的数量，默认为1，表示只有一个日志文件组，没有镜像。若磁盘本身已经做了高可用的方案，如磁盘阵列，那么可以不开启重做日志镜像的功能。
>
> 参数innodb_log_group_home_dir指定了日志文件组所在路径，默认为`./`，表示在MySQL数据库的数据目录下。

​	重做日志文件的大小设置对于InnoDB存储引擎的性能有着非常大的影响。一方面重做日志文件不能设置得太大，如果设置得很大，在恢复时可能需要很长的时间；另一方面又不能设置得太小了，否则可能导致一个事务的日志需要多次切换重做日志文件。此外，重做日志文件太小会导致频繁地发生async checkpoint，导致性能的抖动。

​	重做日志有一个capacity变量，该值代表了最后的检查点不能超过这个阈值，如果超过则必须将缓冲池（innodb buffer pool）中脏页列表（flush list）中的部分脏数据页写回磁盘，这时会导致**用户线程的阻塞**。

​	同样是记录事务日志，**和二进制日志的区别**：

+ 首先，二进制日志会记录所有与MySQL数据库有关的日志记录，包括InnoDB、MyISAM、Heap等其他存储引擎的日志。而InnoDB存储引擎的重做日志只记录有关该存储引擎本身的事务日志。

+ 其次，记录的内容不同，无论用户将二进制日志文件记录的格式设为STATEMENT还是ROW，又或者是MIXED，其记录的都是关于一个事务的具体操作内容，即该日志是**逻辑日志**。而InnoDB存储引擎的重做日志文件记录的是关于每个页（Page）的更改的物理情况。

+ 此外，写入的时间也不同，二进制日志文件仅在事务提交前进行提交，即只写磁盘一次，不论这时该事务多大。而在事务进行的过程中，却不断有重做日志条目（redo entry）被写入到重做日志文件中。

​	在InnoDB存储引擎中，对于各种不同的操作有着不同的重做日志格式。到InnoDB 1.2.x版本为止，总共定义了51种重做日志类型。虽然各种重做日志的类型不同，但是它们有着基本的格式，表3-2显示了重做日志条目的结构：
![image](https://yqfile.alicdn.com/21119fc5462f28ffd52c48fc803e4145aa885731.png)

从表3-2可以看到重做日志条目是由4个部分组成：

+ redo_log_type占用1字节，表示重做日志的类型
+ space表示表空间的ID，但采用压缩的方式，因此占用的空间可能小于4字节
+ page_no表示页的偏移量，同样采用压缩的方式
+ redo_log_body表示每个重做日志的数据部分，恢复时需要调用相应的函数进行解析

​	<u>在第2章中已经提到，写入重做日志文件的操作不是直接写，而是先写入一个重做日志缓冲（redo log buffer）中，然后按照一定的条件顺序地写入日志文件</u>。图3-3很好地诠释了重做日志的写入过程。![image](https://yqfile.alicdn.com/8b6b2eabf3c46659228889e59ab641dbc0798443.png)

​	**从重做日志缓冲往磁盘写入时，是按512个字节，也就是一个扇区的大小进行写入。因为扇区是写入的最小单位，因此可以保证写入必定是成功的。因此在重做日志的写入过程中不需要有doublewrite**。

​	前面提到了从日志缓冲写入磁盘上的重做日志文件是按一定条件进行的，那这些条件有哪些呢？

+ 第2章分析了主线程（master thread），知道在主线程中每秒会将重做日志缓冲写入磁盘的重做日志文件中，不论事务是否已经提交。
+ 另一个触发写磁盘的过程是由参数`innodb_flush_log_at_trx_commit`控制，表示在提交（commit）操作时，处理重做日志的方式。

> 参数`innodb_flush_log_at_trx_commit`的有效值有0、1、2。0代表当提交事务时，并不将事务的重做日志写入磁盘上的日志文件，而是等待主线程每秒的刷新。1和2不同的地方在于：1表示在执行commit时将重做日志缓冲同步写到磁盘，即伴有fsync的调用。2表示将重做日志异步写到磁盘，即写到文件系统的缓存中。因此不能完全保证在执行commit时肯定会写入重做日志文件，只是有这个动作发生。

​	**为了保证事务的ACID中的持久性，必须将`innodb_flush_log_at_trx_commit`设置为1，也就是每当有事务提交时，就必须确保事务都已经写入重做日志文件**。那么当数据库因为意外发生宕机时，可以通过重做日志文件恢复，并保证可以恢复已经提交的事务。而将重做日志文件设置为0或2，都有可能发生恢复时部分事务的丢失。不同之处在于，设置为2时，当MySQL数据库发生宕机而操作系统及服务器并没有发生宕机时，由于此时未写入磁盘的事务日志保存在文件系统缓存中，当恢复时同样能保证数据不丢失。

## 3.7 小结

​	本章介绍了与MySQL数据库相关的一些文件，并了解了文件可以分为**MySQL数据库文件**以及与**各存储引擎相关的文件**。与MySQL数据库有关的文件中，错误文件和二进制日志文件非常重要。当MySQL数据库发生任何错误时，DBA首先就应该去查看错误文件，从文件提示的内容中找出问题的所在。当然，错误文件不仅记录了错误的内容，也记录了警告的信息，通过一些警告也有助于DBA对于数据库和存储引擎进行优化。

​	**二进制日志的作用非常关键，可以用来进行point in time的恢复以及复制（replication）环境的搭建**。因此，**建议在任何时候时都启用二进制日志的记录**。从MySQL 5.1开始，二进制日志支持STATEMENT、ROW、MIX三种格式，这样可以更好地保证从数据库与主数据库之间数据的一致性。当然DBA应该十分清楚这三种不同格式之间的差异。

​	本章的最后介绍了和InnoDB存储引擎相关的文件，包括表空间文件和**重做日志文件**。表空间文件是用来管理InnoDB存储引擎的存储，分为共享表空间和独立表空间。**重做日志非常的重要，用来记录InnoDB存储引擎的事务日志，也因为重做日志的存在，才使得InnoDB存储引擎可以提供可靠的事务**。

# 4. 表

​	表的物理存储方式。

## 4.1 索引组织表

​	InnoDB存储引擎中，表按照主键顺序组织存放，这种存储方式的表称为**索引组织表**（index organized table）。InnoDB存储引擎表中，每张表都有主键（Primary Key），如果没有显式定义主键，InnoDB存储引擎会按如下方式选择or创建主键：

+ **首先判读表中是否有非空的唯一索引（Unique NOT NULL），如果有，则该列即为主键。**
+ **如果不符合上述条件，InnoDB存储引擎自动创建一个6字节大小的指针**。

> 当表中有多个非空唯一索引时， InnoDB存储引擎将选择建表时**第一个**定义的非空唯索引为主键。这里需要非常注意的是，**主键的选择根据的是定义索引的顺序，而不是建表时列的顺序**。
>
> **在单个列为主键的情况下，`_rowid`可以显示表的主键值**

##  * 4.2 InnoDB逻辑存储结构

​	从 InnoDB存储引擎的逻辑存储结构看，所有数据都被逻辑地存放在一个空间中，称之为表空间(tablespace)。表空间又由段(segment)、区(extent)、页(page)组成。页在一些文档中有时也称为块(block)， InnoDB存储引擎的逻辑存储结构大致如图4-1所示。

![c1e4cc3772d032509b85c3f2d06b5567.png](https://img-blog.csdnimg.cn/img_convert/c1e4cc3772d032509b85c3f2d06b5567.png)

​	图4-1 InnoDB逻辑存储结构

### 4.2.1 表空间

​	<u>表空间可以看做是 Innodb存储引擎逻辑结构的最高层，所有的数据都存放在表空间中</u>。

​	第3章中已经介绍了在默认情况下 Innodb存储引擎有一个共享表空间 ibdata1，即所有数据都存放在这个表空间内。如果用户启用了参数`innodb_file_per_table`，则每张表内的数据可以单独放到一个表空间内。

> 如果启用了`innodb_ file_per_table`的参数，需要注意的是每张表的表空间内存放的只是数据、索引和插入缓冲Bitmap页，其他类的数据，如回滚(undo)信息，插入缓冲索引页、系统事务信息，二次写缓冲(Double write buffer)等还是存放在原来的共享表空间内。这同时也说明了另一个问题：即使在启用了参数`innodb _file_per_table`之后，共享表空间还是会不断地增加其大小。
>
> **产生undo信息后，即使对事务进行回滚rollback，共享表空间新增的undo信息占用的空间也不会被回收，但是InnoDB会在不需要这些undo信息时，将其占据的空间标记为可用空间，供下次undo使用**。

### 4.2.2 段

​	图4-1中显示了表空间是由各个段组成的，常见的段有数据段、索引段、回滚段等。因为前面已经介绍过了InnoDB存储引擎表是索引组织的(index organized)，因此数据即索引，索引即数据。那么数据段即为B+树的叶子节点(图4-1的Leaf node segment)，索引段即为B+树的非索引节点(图4-1的 Non-leaf node segment)。回滚段较为特殊，将会在后面的章节进行单独的介绍。

​	在 Innodb存储引擎中，对段的管理都是由引擎自身所完成，DBA不能也没有必要对其进行控制。这和 Oracle数据库中的自动段空间管理(ASSM)类似，从一定程度上简化了DBA对于段的管理。

### 4.2.3 区

​	<u>区是由连续页组成的空间，在任何情况下每个区的大小都为1MB</u>。为了保证区中页的连续性， InnoDB存储引擎一次从磁盘申请4~5个区。在默认情况下， InnoDB存储引擎页的大小为16KB，即一个区中一共有64个连续的页。

> InnoDB1.0.x版本开始引人压缩页，即每个页的大小可以通过参数`KEY_BLOCKSIZE`设置为2K、4K、8K，因此每个区对应页的数量就应该为512、256、128。
>
> InnoDB1.2.x版本新增了参数`innodb_page_size`，通过该参数可以将默认页的大小设置为4K、8K，但是页中的数据库不是压缩。这时区中页的数量同样也为256、128。总之，不论页的大小怎么变化，区的大小总是为1M。

​	但是，这里还有这样一个问题：在用户启用了参数`innodb file_per_tabe`后，创建的表默认大小是96KB。区中是64个连续的页，创建的表的大小至少是1MB才对啊？其实这是因为**在每个段开始时，先用32个页大小的碎片页(fragment page)来存放数据，在使用完这些页之后才是64个连续页的申请。这样做的目的是，对于一些小表，或者是undo这类的段，可以在开始时申请较少的空间，节省磁盘容量的开销**。

### 4.2.4 页

​	同大多数数据库一样， **InnoDB有页(Page)的概念(也可以称为块)，页是InnoDB磁盘管理的最小单位**。在Innodb存储引擎中，默认每个页的大小为16KB。而从InnoDB 1.2.x版本开始，可以通过参数`innodb_page_size`将页的大小设置为4K、8K、16K。若设置完成，则所有表中页的大小都为`innodb_page_size`，不可以对其再次进行修改。除非通过 mysqldump导入和导出操作来产生新的库。

​	在 InnoDB存储引擎中，常见的页类型有：

+ 数据页(B-tree Node)
+ undo页(undo Log Page)
+ 系统页(System Page)
+ 事务数据页(Transaction system Page)
+ 插入缓冲位图页(Insert Buffer Bitmap)
+ 插入缓冲空闲列表页(Insert Buffer Free List)
+ 未压缩的二进制大对象页(Uncompressed BLOB Page)
+ 压缩的二进制大对象页(compressed BLOB Page)

### * 4.2.5 行

​	InnoDB存储引擎是面向行的(row-oriented)，也就说数据是按行进行存放的。**每个页存放的行记录也是有硬性定义的，最多允许存放16KB/2-200行的记录，即7992行记录**。这里提到了row-oriented的数据库，也就是说，存在有column-oriented的数据库。MySQL infobright存储引擎就是按列来存放数据的，这对于数据仓库下的分析类SQL语句的执行及数据压缩非常有帮助。类似的数据库还有 Sybase IQ、 Google Big Table。面向列的数据库是当前数据库发展的一个方向，但这超出了本书涵盖的内容，有兴趣的读者可以在网上寻找相关资料。

## * 4.3 InnoDB行记录格式

​	InnoDB存储引檠和大多数数据库一样(如 Oracle和 Microsoft SQL Server数据库)，记录是以行的形式存储的。这意味着页中保存着表中一行行的数据。

​	在 InnoDB 1.0.x版本之前， InnoDB存储引擎提供了 Compact和 Redundant两种格式来存放行记录数据，这也是目前使用最多的一种格式。 <u>Redundant格式是为兼容之前版本而保留的</u>，如果阅读过 InnoDB的源代码，用户会发现源代码中是用 PHYSICAL RECORD (NEW STYLE)和 PHYSICAL RECORD (OLD STYLE)来区分两种格式的。

​	在 MYSQL5.1版本中，默认设置为 Compact 行格式。用户可以通过命令 SHOW TABLE STATUS LIKE 'tablename'来査看当前表使用的行格式，其中 row_format 属性表示当前所使用的行记录结构类型。

### * 4.3.1 Compact 行记录格式

​	Compact行记录是在 MySQL5.0 中引入的，其设计目标是高效地存储数据。**简单来说一个页中存放的行数据越多，其性能就越高**。图4-2显示了 Compact行记录的存储方式：

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020112517190174.png#pic_center)

​	图4-2 Compact 行记录的格式

​	从图4-2 可以观察到， **Compact 行记录格式的首部是一个<u>非NULL 变长字段长度列表</u>，并且其是<u>按照列的顺序逆序放置</u>的**，其长度为：

+ 若列的长度小于255 字节，用1字节表示；

+ 若大于255个字节，用2字节表示。

​	<u>变长字段的长度最大不可以超过2字节，这是因在MySQL数据库中VARCHAR类型的最大长度限制为65535</u> 。变长字段之后的第二个部分是NULL 标志位，该位指示了该行数据中是否有NULL 值，有则用1 表示。该部分所占的字节应该为1 字节。接下来的部分是记录头信息(record header) ，固定占用5 字节(40 位），每位的含义见表4-1 。

​	表4-1 Compact 记录头信息

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201125171846565.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1MzcxMzIz,size_16,color_FFFFFF,t_70#pic_center)

​	最后的部分就是实际存储每个列的数据。**<u>需要特别注意的是， NULL 不占该部分任何空间，即NULL 除了占有NULL 标志位，实际存储不占有任何空间</u>。**另外有一点需要注意的是，<u>每行数据除了用户定义的列外，还有两个隐藏列，**事务ID 列**和**回滚指针列**</u>，分别为6 字节和7 字节的大小。<u>**若InnoDB 表没有定义主键，每行还会增加一个6 字节的rowid 列**</u>。

​	**<u>不管是CHAR类型还是VARCHAR类型，在compact格式下，NULL值都不占用任何存储空间</u>**。

### * 4.3.2 Redundant行记录格式

​	**Redundant 是MySQL5.0 版本之前InnoDB 的行记录存储方式， MySQL5.0 支持Redundant 是为了兼容之前版本的页格式**。Redundant 行记录采用如图4-3 所示的方式存储。

![在这里插入图片描述](https://img-blog.csdnimg.cn/3ffe498c6b3d4e3aa46c36a5761a47fb.png)

​	从图4-3 可以看到，不同于Compact 行记录格式， Redundant 行记录格式的首部是一个字段长度<u>偏移</u>列表，<u>同样是按照列的顺序逆序放置的</u>。若列的长度小于255 字节，用1 字节表示：若大于255 字节，用2 字节表示。

​	第二个部分为记录头信息(recordheader) ，不同于Compact 行记录格式， Redundant 行记录格式的记录头占用6 字节(48位），每位的含义见表4-2 。从表4-2 中可以发现， **`n_fields` 值代表一行中列的数扯，占用10 位。同时这也很好地解释了为什么MySQL 数据库一行支持最多的列为1023** 。另一个需要注意的值为`1byte_offs_ flags` ，该值定义了偏移列表占用1字节还是2字节。而最后的部分就是实际存储的每个列的数据了。

![在这里插入图片描述](https://img-blog.csdnimg.cn/4694f6ee888546ec9dd14f9f2a03c8b9.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5Zyf5ouo6byg6aWy5YW75ZGY,size_20,color_FFFFFF,t_70,g_se,x_16)

​	**<u>对于VARCHAR类型的NULL值，Redudant行记录格式同样不会占用任何存储空间，而CHAR类型的NULL值需要占用空间</u>**。

> 当表的字符集为Latin1, 每个字符最多只占用1 字节。若用户将表的字符集转换为utf8，CHAR字段固定长度类型将占用原本的`字节数*3`大小的字节数。**在Redundant 行记录格式下， CHAR 类型将会占用可能存放的最大值字节数**。

### * 4.3.3 行溢出数据

​	**InnoDB 存储引擎可以将一条记录中的某些数据存储在真正的数据页面之外**。

​	一般认为BLOB 、LOB 这类的大对象列类型的存储会把数据存放在数据页面之外。但是，这个理解有点偏差， <u>BLOB 可以不将数据放在溢出页面，而且即便是VARCHAR 列数据类型，依然有可能被存放为行溢出数据</u>。

​	首先对VARCHAR 数据类型进行研究。很多DBA 喜欢MySQL 数据库提供的VARCHAR 类型，因为相对于Oracle VARCHAR2 最大存放4000 字节， SQL Server 最大存放8000 字节， <u>MySQL 数据库的VARCHAR 类型可以存放65535 字节</u>。

​	但是，这是真的吗？真的可以存放65535 字节吗？如果创建VARCHAR 长度为65535 的表，用户会得到下面的错误信息：

 ```shell
 mysql> CREATE TABLE test (
 -> a VARCHAR(65535)
 ->)CHARSET=latinl ENGINE=InnoDB;
 ERROR 1118 (42000): Row size too large. The maximum row size for the used table
 type, not counting BLOBs, is 65535. You have to change some columns to TEXT or
 BLOBS
 ```

​	从错误消息可以看到**lnnoDB 存储引擎并不支持65535 长度的VARCHAR。这是因为还有别的开销，通过实际测试发现能存放VARCHAR 类型的最大长度为65532** 。例如，按下面的命令创建表就不会报错了。

 ```shell
 mysql> CREATE TABLE test (
 -> a VARCHAR(65532)
 ->)CHARSET=latinl ENGINE=InnoDB;
 Query OK, 0 rows affected (0.15 sec)
 ```

​	需要注意的是，如果在执行上述示例的时候没有将SQL_MODE设为严格模式，或许可以建立表，但是MySQL 数据库会抛出一个warning，如：

```shell
mysql> CREATE TABLE test (
-> a VARCHAR(65535)
->)CHARSET=latinl ENGINE=InnoDB;
Query OK, 0 rows affected, 1 warning (0.14 sec)

mysql> SHOW WARNINGS\G;
*************************** 1. row***************************
Level: Note
Code: 124 6
Message: Converting column'a'from VARCHAR to TEXT
1 row in set (0.00 sec)
```

<u>	warning 信息提示了这次可以创建是因为MySQL 数据库自动地将VARCHAR 类型转换成了TEXT 类型</u>。查看test 的表结构会发现：

```shell
mysql> SHOW CREATE TABLE test\G;
*************************** 1. row***************************
Table: test
Create Table: CREATE TABLE'test'(
'a'mediumtext
ENGINE=InnoDB DEFAULT CHARSET=utf8
1 row in set (0.00 sec)
```

​	还需要注意上述创建的VARCHAR 长度为65 532 的表，其字符类型是latinl 的，如果换成GBK 又或UTF-8 的，会产生怎样的结果呢？

```shell
mysql> CREATE TABLE test (
-> a VARCHAR(65532)
->)CHARSET=GBK ENGINE=InnoDB;
ERROR 1074 (42000): Column length too big for column'a'(max = 32767); use
BLOB or TEXT instead

rnysql> mysql> CREATE TABLE test (
-> a VARCHAR(65532)
->)CHARSET=UTF8 ENGINE=InnoDB;
ERROR 1074 (42000):
```

​	这次即使创建列的VARCHAR 长度为65532，也会提示报错，但是两次报错对max值的提示是不同的。因此从这个例子中用户也应该理解**VARCHAR (N) 中的N 指的是字符的长度。而文档中说明VARCHAR 类型最大支待65535，单位是字节。**

​	**<u>此外需要注意的是， MySQL 官方手册中定义的65535 长度是指所有VARCHAR列的长度总和，如果列的长度总和超出这个长度，依然无法创建</u>**，如下所示：

```shell
mysql> CREATE TABLE test2
-> a VARCHAR(22000),
-> b VARCHAR(22000),
-> c VARCHAR(22000)
->)CHARSET=latinl ENGINE=InnoDB;
ERROR 1118 (42000): Row size too large. The maximum row size for the used table
type, not counting BLOBs, is 65535. You have to change some columns to TEXT or
BLOBS
```

​	3 个列长度总和是66000, 因此InnoDB 存储引擎再次报了同样的错误。

​	即使能存放65532 个字节，但是有没有想过， InnoDB 存储引擎的页为16KB， 即16384 字节，怎么能存放65532 字节呢？**<u>在一般情况下， InnoDB 存储引擎的数据都是存放在页类型为B-tree node 中。但是当发生行溢出时，数据存放在页类型为Uncompress BLOB 页中</u>**。

​	对于行溢出数据，其存放采用图4-4的方式。数据页面其实只保存了VARCHAR (65532)的前768 字节的前缀(prefix) 数据（举例这里都是a) ，之后是偏移量，指向行溢出页，也就是前面用户看到的Uncompressed BLOB Page。

![在这里插入图片描述](https://img-blog.csdnimg.cn/90fbe66f21e64d2799eda4c54d4dfa8a.png)

​	那多长的VARCHAR 是保存在单个数据页中的，从多长开始又会保存在BLOB页呢？可以这样进行思考： InnoDB 存储引擎表是索引组织的，即B+Tree 的结构，这样每个页中至少应该有两条行记录（否则失去了B+Tree 的意义，变成链表了）。因此，**<u>如果页中只能存放下一条记录，那么InnoDB 存储引擎会自动将行数据存放到溢出页中</u>**。

​	但是，**<u>如果可以在一个页中至少放入两行数据，那VARCHAR 类型的行数据就不会存放到BLOB 页中去。经过多次试验测试，发现这个阔值的长度为8098</u>**。

​	另一个问题是，对于TEXT 或BLOB 的数据类型，用户总是以为它们是存放在Uncompressed BLOB Page 中的，其实这也是不准确的。<u>**（TEXT或BLOB数据类型）是放在数据页中还是BLOB 页中，和前面讨论的VARCHAR 一样，至少保证一个页能存放两条记录**</u>。 

> 当然既然用户使用了BLOB 列类型，一般不可能存放长度较小的数据。因此在大多数的情况下BLOB 的行数据还是会发生行溢出，实际数据保存在BLOB 页中，数据页只保存数据的前768 字节。

### * 4.3.4 Compressed 和Dynamic 行记录格式

​	InnoDB 1.0.x 版本开始引入了新的文件格式(file format, 用户可以理解为新的**页格式**），<u>以前支持的Compact 和Redundant 格式称为Antelope 文件格式</u>，新的文件格式称为Barracuda文件格式。Barracuda 文件格式下拥有两种新的行记录格式： Compressed 和Dynamic。

​	**新的两种记录格式对千存放在BLOB 中的数据采用了<u>完全的行溢出</u>的方式**，如图4-5 所示，<u>在数据页中只存放20 个字节的指针，实际的数据都存放在Off Page 中，而之前的Compact 和Redundant 两种格式会存放768 个前缀字节</u>。

![在这里插入图片描述](https://img-blog.csdnimg.cn/6b81ac040aba44a98f81b1b5c9b0a673.png)

​	**<u>Compressed 行记录格式的另一个功能就是，存储在其中的行数据会以zlib 的算法进行压缩，因此对于BLOB 、TEXT 、VARCHAR 这类大长度类型的数据能够进行非常有效的存储</u>**。

### * 4.3.5 CHAR 的行结构存储

​	通常理解VARCHAR 是存储变长长度的字符类型， CHAR 是存储固定长度的字符类型。而在前面的小节中，用户已经了解行结构的内部的存储，并可以发现每行的变长字段长度的列表都没有存储CHAR 类型的长度。

​	**从MySQL4.l 版本开始， CHR(N) 中的N 指的是字符的长度，而不是之前版本的字节长度。也就说在不同的字符集下， CHAR 类型列内部存储的可能不是定长的数据**。

​	**<u>对于多字节字符编码的CHAR 数据类型的存储， lnnoDB 存储引擎在内部将其视为变长字符类型。这也就意味着在变长长度列表中会记录CHAR数据类型的长度</u>**。此时，CHAR 类型被明确视为了变长字符类型，对于未能占满长度的字符还是填充0x20。

​	InnoDB 存储引擎内部对字符的存储和我们用HEX函数看到的也是一致的。因此可以认为**在多字节字符集的情况下， CHAR 和VARCHAR 的实际行存储基本是没有区别的**。

## 4.4 lnnoDB 数据页结构

P133
