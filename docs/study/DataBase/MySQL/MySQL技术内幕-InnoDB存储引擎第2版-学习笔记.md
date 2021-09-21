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

MySQL数据库区别于其他数据库的最重要的一个特点就是插件式的表存储引擎。

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

+ **MySQL数据库使用Memory存储引擎作为临时表来存放查询的中间结果集（intermediate result）。如果中间结果集大雨Memory存储引擎表的容量设置，又或者中间含有TEXT或BLOB列类型字段，则MySQL数据库会把其转换到MyISAM存储引擎表而存放到磁盘中。MyISAM不缓存数据文件，此时产生的临时表的性能对于查询会有损失**。

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

## 2.4 Checkpoint技术

p45