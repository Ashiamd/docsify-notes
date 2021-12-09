# mysql入门很简单系列视频-学习笔记

> 视频链接：[mysql入门很简单系列视频](https://www.bilibili.com/video/av14920200/) https://www.bilibili.com/video/av14920200/
>
> 以前主要就了解DDL、DML、DCL、索引等，有一些自己之前不常用的比较不熟，看视频当作复习了。
>
> 下面惯例会贴一些我自己看的网文，因为是复习，所以网文内容可能和学习的进度无关

## 第1章 数据库概述

> [mysql中 的 ENGINE = innodb; 是什么意思？](https://zhidao.baidu.com/question/329349963.html)
>
> [MYSQL8创建、删除用户和授权、消权操作](https://www.cnblogs.com/gychomie/p/11013442.html)
>
> [存储JWT令牌的最佳MYSQL或Maria DB数据类型是什么？](http://www.voidcn.com/article/p-avmcopfc-bwm.html)
>
> [MySQL Geometry扩展在地理位置计算中的效率优势](https://www.cnblogs.com/hargen/p/9674958.html)
>
> [Mysql 5.7 的‘虚拟列’是做什么？](https://www.techug.com/post/mysql-5-7-generated-virtual-columns.html)
>
> [NULL与MySQL空字符串的区别](https://blog.csdn.net/lovemysea/article/details/82317043)
>
> [mysql partition 实战](https://blog.csdn.net/alex_xfboy/article/details/85245502)
>
> [数据库设计及ER模型](https://www.cnblogs.com/dayle/p/9946714.html)
>
> [MySQL 连接查询](https://www.cnblogs.com/xiaozhaoboke/p/11077781.html)
>
> [in 和 exist 区别](https://blog.csdn.net/lick4050312/article/details/4476333)
>
> [如何实现 MySQL 删除重复记录并且只保留一条](https://mp.weixin.qq.com/s/17F04xs7utTTHvPUzIjZHA)
>
> [mysql 内连接、左连接会出现笛卡尔积？](https://blog.csdn.net/zy_281870667/article/details/81046573)
>
> 首先说下结论：链接查询，如果on条件是非唯一字段，会出现笛卡尔积(局部笛卡尔积)；如果on条件是表的唯一字段，则不会出现笛卡尔积。
>
> 
>
> 索引
>
> [数据库中聚合索引（MySQL和SQL Server区别）](https://www.cnblogs.com/zhurong/p/9374238.html)
>
> mysql中每个表都有一个聚簇索引（clustered index ），除此之外的表上的每个非聚簇索引都是二级索引，又叫辅助索引（secondary indexes）。
> 以InnoDB来说，每个InnoDB表具有一个特殊的索引称为聚集索引。如果您的表上定义有主键，该主键索引是聚集索引。如果你不定义为您的表的主键 时，MySQL取第一个唯一索引（unique）而且只含非空列（NOT NULL）作为主键，InnoDB使用它作为聚集索引。如果没有这样的列，InnoDB就自己产生一个这样的ID值，它有六个字节，而且是隐藏的，使其作 为聚簇索引。
>
> [MYSQL索引：对聚簇索引和非聚簇索引的认识](https://blog.csdn.net/alexdamiao/article/details/51934917) <= 推荐阅读
>
> InnoDB的的二级索引的叶子节点存放的是KEY字段加主键值。因此，通过二级索引查询首先查到是主键值，然后InnoDB再根据查到的主键值通过主键索引找到相应的数据块。而MyISAM的二级索引叶子节点存放的还是列值与行号的组合，叶子节点中保存的是数据的物理地址。所以可以看出MYISAM的主键索引和二级索引没有任何区别，主键索引仅仅只是一个叫做PRIMARY的唯一、非空的索引，且MYISAM引擎中可以不设主键。
>
> [MySQL 之全文索引](https://blog.csdn.net/mrzhouxiaofei/article/details/79940958)
>
> [什么叫做覆盖索引？](https://www.cnblogs.com/happyflyingpig/p/7662881.html) <== 建议阅读
>
> 来看看什么是覆盖索引，有下面三种理解：
>
> - 解释一： 就是select的数据列只用从索引中就能够取得，不必从数据表中读取，换句话说查询列要被所使用的索引覆盖。
> - 解释二： 索引是高效找到行的一个方法，当能通过检索索引就可以读取想要的数据，那就不需要再到数据表中读取行了。如果一个索引包含了（或覆盖了）满足查询语句中字段与条件的数据就叫 做覆盖索引。
> - 解释三：是非聚集组合索引的一种形式，它包括在查询里的Select、Join和Where子句用到的所有列（即建立索引的字段正好是覆盖查询语句[select子句]与查询条件[Where子句]中所涉及的字段，也即，索引包含了查询正在查找的所有数据）。
>
> 　　不是所有类型的索引都可以成为覆盖索引。覆盖索引必须要存储索引的列，而哈希索引、空间索引和全文索引等都不存储索引列的值，所以MySQL只能使用B-Tree索引做覆盖索引
>
> ### 总结：覆盖索引的优化及限制
>
>  覆盖索引是一种非常强大的工具，能大大提高查询性能，只需要读取索引而不需要读取数据，有以下优点：
>
>  1、索引项通常比记录要小，所以MySQL访问更少的数据。
>
>  2、索引都按值得大小存储，相对于随机访问记录，需要更少的I/O。
>
>  3、数据引擎能更好的缓存索引，比如MyISAM只缓存索引。
>
>  4、覆盖索引对InnoDB尤其有用，因为InnoDB使用聚集索引组织数据，如果二级索引包含查询所需的数据，就不再需要在聚集索引中查找了。
>
>  **限制：**
>
>  1、覆盖索引也并不适用于任意的索引类型，索引必须存储列的值。
>
>  2、Hash和full-text索引不存储值，因此MySQL只能使用BTree。
>
>  3、不同的存储引擎实现覆盖索引都是不同的，并不是所有的存储引擎都支持覆盖索引。
>
>  4、如果要使用覆盖索引，一定要注意SELECT列表值取出需要的列，不可以SELECT * ，因为如果将所有字段一起做索引会导致索引文件过大，查询性能下降。
>
> 
>
> 视图View
>
> [MySQL视图(view)](https://www.cnblogs.com/cshaptx4869/p/10481749.html)
>
> 
>
> 存储过程Procedure
>
> [mysql 存储过程权限相关](https://www.cnblogs.com/woxingwoxue/p/4974991.html)
>
> [MySQL 授予普通用户PROCESS权限](https://www.cnblogs.com/kerrycode/p/7421777.html)
>
> [MySQL之存储过程（PROCEDURE）](https://www.cnblogs.com/ccstu/p/12182933.html)
>
> 
>
> 游标Cursor
>
> [MySQL 游标的使用](https://www.cnblogs.com/oukele/p/10684639.html)
>
> 
>
> 触发器Trigger
>
> [mysql触发器trigger 实例详解](https://www.cnblogs.com/phpper/p/7587031.html)
>
> 触发器会有以下两种限制：
>
> 1.触发程序不能调用将数据返回客户端的存储程序，也不能使用采用CALL语句的动态SQL语句，但是允许存储程序通过参数将数据返回触发程序，也就是存储过程或者函数通过OUT或者INOUT类型的参数将数据返回触发器是可以的，但是不能调用直接返回数据的过程。
>
> 2.不能再触发器中使用以显示或隐式方式开始或结束事务的语句，如START TRANS-ACTION,COMMIT或ROLLBACK。
>
> 注意事项：MySQL的触发器是按照BEFORE触发器、行操作、AFTER触发器的顺序执行的，其中任何一步发生错误都不会继续执行剩下的操作，如果对事务表进行的操作，如果出现错误，那么将会被回滚，如果是对非事务表进行操作，那么就无法回滚了，数据可能会出错。
>
> 
>
> 事务，4种隔离级别和7种传播行为
>
> [mysql的表锁和行锁，排他锁和共享锁。](https://www.cnblogs.com/shamgod-lct/p/9318032.html)
>
> [mysql 四种隔离级别](https://www.cnblogs.com/jian-gao/p/10795407.html)
>
> [MySQL事务隔离性说明](https://blog.csdn.net/qq_42517220/article/details/88779932)
>
> [mysql的默认隔离级别](https://www.cnblogs.com/shoshana-kong/p/10516404.html)
>
> [说说@Transactional(readOnly = true),和mysql事务隔离级别](https://blog.csdn.net/myth_g/article/details/89921025)
>
> [spring @Transactional注解参数详解](https://www.cnblogs.com/xinruyi/p/11037724.html)
>
> [Spring 事务 readOnly 到底是怎么回事？](https://www.cnblogs.com/hackem/p/3890656.html)
>
> [Spring中 PROPAGATION_REQUIRED 解释](https://blog.csdn.net/bigtree_3721/article/details/53966617)
>
> [Spring boot中配置事务管理](https://blog.csdn.net/weixin_44554160/article/details/86775915)
>
> [脱离 Spring 实现复杂嵌套事务，之六（NOT_SUPPORTED - 非事务方式）](https://blog.csdn.net/lijiangjava/article/details/51088581)
>
> [请问事务挂起什么意思啊，出自下面，谢谢？](https://zhidao.baidu.com/question/109536055.html)
>
> [mysql数据库隔离级别及其原理、Spring的7种事物传播行为](https://www.cnblogs.com/wzk-0000/p/9928883.html)
>
> [spring 事务回滚](https://www.cnblogs.com/0201zcr/p/5962578.html)
>
> [事务挂起引起的死锁问题](https://blog.csdn.net/u011330604/article/details/86481791)
>
> [十九、spring事务之创建事务](https://www.jianshu.com/p/47fb41b78dce)
>
> [spring 是如何保证一个事务内获取同一个Connection?](https://blog.csdn.net/qq_33363618/article/details/102649197)
>
> [Spring事务的传播级别 _](https://www.spldeolin.com/posts/spring-tx-propagation/)
>
> [【面试】足够“忽悠”面试官的『Spring事务管理器』源码阅读梳理（建议珍藏） ](https://www.sohu.com/a/306497169_465221)
>
> [频繁的访问数据库,sqlconnection可以一直open不close吗(2) ](https://bbs.csdn.net/topics/390843492)
>
> [MySQL的四种事务隔离级别](https://www.cnblogs.com/wyaokai/p/10921323.html)
>
> [事务及事务隔离级别](https://www.cnblogs.com/xrq730/p/5087378.html)
>
> [Mysql——通过例子理解事务的4种隔离级别](https://www.cnblogs.com/snsdzjlz320/p/5761387.html)
>
> Mysql默认的可重复读（事务中，哪时候select了，接下去是本地select结果的副本操作了，不会再从数据库读别人update的。）
>
> 
>
> 地理位置计算、处理：
>
> [使用MySQL的geometry类型处理经纬度距离问题的方法](https://www.jb51.net/article/155712.htm)
>
> [MySQL Geometry扩展在地理位置计算中的效率优势](https://www.cnblogs.com/hargen/p/9674958.html)
>
> [mysql根据经纬度查找排序](https://www.chengxiaobai.cn/sql/mysql-according-to-the-latitude-and-longitude-search-sort.html)
>
> [mysql距离函数st_distance](https://blog.csdn.net/u013628152/article/details/51560272)
>
> 
>
> 存储引擎：
>
> [InnoDB与Myisam的六大区别](https://www.cnblogs.com/vicenteforever/articles/1613119.html)
>
> [InnoDB和MyISAM的区别](https://www.cnblogs.com/amiezhang/p/10029585.html)
>
> [MyISAM与InnoDB 的区别（9个不同点）](https://blog.csdn.net/qq_35642036/article/details/82820178)
>
> 
>
> 主键：
>
> [深入分析mysql为什么不推荐使用uuid或者雪花id作为主键](https://www.cnblogs.com/wyq178/p/12548864.html)
>
> [深入浅出mybatis之返回主键ID](https://www.cnblogs.com/nuccch/p/9067305.html)
>
> 
>
> 主从复制
>
> [Mysql 主从复制](https://www.jianshu.com/p/faf0127f1cb2)
>
> [MySQL主从分离实现](https://www.cnblogs.com/reminis/p/13335292.html)
>
> 
>
> 缓存
>
> [MySQL缓存机制](https://www.cnblogs.com/yueyun00/p/10898677.html)
>
> 
>
> 锁
>
> [select for update引发死锁分析 - 活在夢裡 - 博客园 (cnblogs.com)](https://www.cnblogs.com/micrari/p/8029710.html)
>
> [求教 For Update 解锁](https://bbs.csdn.net/topics/330034068)
>
> [MySQL 共享锁 (lock in share mode)，排他锁 (for update)](https://learnku.com/articles/12800/lock-in-share-mode-mysql-shared-lock-exclusive-lock-for-update)
>
> [for update 与lock in share mode的区别](https://www.cnblogs.com/lint20/p/11384096.html)
>
> + lock in share mode意向共享锁（IS）
>
> + for update意向排它锁（IX）
>
> 有索引就锁行，没索引就锁表（InnoDB才支持事务，另一个MyISAM不支持且只有表级锁）
>
> [IS/IX/S/X等锁的区别](https://www.jianshu.com/p/94fc3c72f195)
>
> [MySql-两阶段加锁协议](https://my.oschina.net/alchemystar/blog/1438839)
>
> 在事务中只有提交(commit)或者回滚(rollback)时才是解锁阶段，其余时间为加锁阶段。
>
> **[mysql行锁+可重复读+读提交](https://www.cnblogs.com/jimmyhe/p/11013551.html) < ==== 强烈推荐， 强烈推荐， 强烈推荐(下面内容摘自于此)**
>
> **注意：begin/start transaction 命令并不是一个事务的起点，在执行到它们之后的第一个操作InnoDB表的语句（第一个快照读语句），事务才真正启动。如果你想要马上启动一个事务，可以使用start transaction with consistent snapshot 这个命令。**
>
> 可重复读的核心就是一致性读（consistent read）；而事务更新数据的时候，只能用当前读。如果当前的记录的行锁被其他事务占用的话，就需要进入锁等待。
>
> 而读提交的逻辑和可重复读的逻辑类似，它们最主要的区别是：在可重复读隔离级别下，只需要在事务开始的时候创建一致性视图，之后事务里的其他查询都共用这个一致性视图；在读提交隔离级别下，每一个语句执行前都会重新算出一个新的视图
>
> **innoDB的行数据有多个版本，每个数据版本有自己的row trx_id，每个事务或者语句有自己的一致性视图。普通查询语句是一致性读，一致性读会根据row trx_id和一致性视图确定数据版本的可见性。**
>
> - **对于可重复读，查询只承认在事务启动前就已经提交完成的数据；**
> - **对于读提交，查询只承认在语句启动前就已经提交完成的数据；**
> - **而当前读，总是读取已经提交完成的最新版本。**
>
> [Innodb锁机制：Next-Key Lock 浅谈](https://www.cnblogs.com/zhoujinyi/p/3435982.html) <= **推荐阅读！！推荐阅读！！推荐阅读！！**
>
>    数据库使用锁是为了支持更好的并发，提供数据的完整性和一致性。InnoDB是一个支持行锁的存储引擎，锁的类型有：共享锁（S）、排他锁（X）、意向共享（IS）、意向排他（IX）。为了提供更好的并发，InnoDB提供了非锁定读：不需要等待访问行上的锁释放，读取行的一个快照。该方法是通过InnoDB的一个特性：MVCC来实现的。
>
> **InnoDB有三种行锁的算法：**
>
> 1，Record Lock：单个行记录上的锁。
>
> 2，Gap Lock：间隙锁，锁定一个范围，但不包括记录本身。GAP锁的目的，是为了防止同一事务的两次当前读，出现幻读的情况。
>
> 3，Next-Key Lock：1+2，锁定一个范围，并且锁定记录本身。对于行的查询，都是采用该方法，主要目的是解决幻读的问题。
>
> **InnoDB对于行的查询都是采用了Next-Key Lock的算法，锁定的不是单个值，而是一个范围，按照这个方法是会和第一次测试结果一样。但是，当查询的索引含有唯一属性的时候，Next-Key Lock 会进行优化，将其降级为Record Lock，即仅锁住索引本身，不是范围。**
>
> 注意：通过主键或则唯一索引来锁定不存在的值，也会产生GAP锁定。
>
> [深入了解mysql--gap locks,Next-Key Locks](https://blog.csdn.net/qq_20597727/article/details/87308709) <== 推荐阅读
>
> [记一次大事务导致的数据库死锁引起线上服务器load飙升的故障_森林里的一天-CSDN博客](https://blog.csdn.net/dtlscsl/article/details/111914461) <= 现实问题，值得学习
>
> [mysql意向锁_luzhensmart的专栏-CSDN博客_mysql意向锁](https://blog.csdn.net/luzhensmart/article/details/87905553)
>
> [MySQL死锁系列-常见加锁场景分析 (toutiao.com)](https://www.toutiao.com/i6831549261650330125/?wid=1638655068270)	<=	强烈推荐阅读
>
> 
>
> MVCC
>
> [mysql事务的实现方式——mvvc+锁](https://www.cnblogs.com/tongcc/p/13084368.html)
>
> 1.1只有在InnoDB引擎下存在的一种基于多版本的并发控制协议；
>
> 1.2MVCC只在 READ COMMITTED 和 REPEATABLE READ 两个隔离级别下工作。其他两个隔离级别够和MVCC不兼容，因为 READ UNCOMMITTED 总是读取最新的数据行，而不是符合当前事务版本的数据行。而 SERIALIZABLE 则会对所有读取的行都加锁
>
> 好处：读不加锁，读写不冲突
>
> [【八】MySQL事务之MVCC、undo、redo、binlog、二阶段提交_Sid小杰的博客-CSDN博客](https://blog.csdn.net/jy02268879/article/details/105580287)
>
> 
>
> 
>
> timeout变量
>
> [mysql timeout知多少](https://blog.csdn.net/weixin_33766805/article/details/93818673?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3)
>
> 
>
> 元数据
>
> [MySQL的元数据解释说明](https://blog.csdn.net/qq_34672033/article/details/90145387)
>
> 
>
> 数据库查询流程（MySQL基本架构）
>
> [数据库基础知识](https://blog.csdn.net/qq_35190492/article/details/104203466?utm_medium=distribute.pc_feed.none-task-blog-alirecmd-20&depth_1-utm_source=distribute.pc_feed.none-task-blog-alirecmd-20&request_id=)
>
> 
>
> binlog、undolog、redolog
>
> [MySQL三种日志有啥用？如何提高MySQL并发度？_小识的博客-CSDN博客](https://blog.csdn.net/zzti_erlie/article/details/116886211)
>
> [【八】MySQL事务之MVCC、undo、redo、binlog、二阶段提交_Sid小杰的博客-CSDN博客](https://blog.csdn.net/jy02268879/article/details/105580287)
>
> [docker mysql 开启binlog - 简书 (jianshu.com)](https://www.jianshu.com/p/8398d41a9c4a)
>
> **自己试了一下，binlog是会记录insert时的自增主键的**
>
> `mysqlbinlog --base64-output=DECODE-ROWS -v -v /var/lib/mysql/binlog文件`
>
> 
>
> auto_increment实现
>
> [MySQL auto_increment实现 - zengkefu - 博客园 (cnblogs.com)](https://www.cnblogs.com/zengkefu/p/5683258.html)

### 1. 事务(各路学习后的总结)

1. 事务级别选择

   ​	MySQL默认的事务级别是可重复读（repeatable-read），但是这种模式下，如果SQL没有命中索引，就会锁表。而比这个事务低一个级别的读提交（read-committed）在没有命中索引的时候，会放弃对当前扫描到的数据行的S/X锁，只对不符合索引条件的进行S/X锁。

   ​	==所以建议分布式事务使用RC不可重复读，避免高并发的表锁拖累响应速度==。

   ​	推荐阅读文章，[mysql的默认隔离级别](https://www.cnblogs.com/shoshana-kong/p/10516404.html) 和 [mysql数据库隔离级别及其原理、Spring的7种事物传播行为](https://www.cnblogs.com/wzk-0000/p/9928883.html) 还有 [【面试】足够“忽悠”面试官的『Spring事务管理器』源码阅读梳理（建议珍藏） ](https://www.sohu.com/a/306497169_465221)

2. 据说Spring会对只读事务会做一些优化，建议get类型的方法事务指定read-only="true"

   ```java
   /**
   	 * A boolean flag that can be set to {@code true} if the transaction is
   	 * effectively read-only, allowing for corresponding optimizations at runtime.
   	 * <p>Defaults to {@code false}.
   	 * <p>This just serves as a hint for the actual transaction subsystem;
   	 * it will <i>not necessarily</i> cause failure of write access attempts.
   	 * A transaction manager which cannot interpret the read-only hint will
   	 * <i>not</i> throw an exception when asked for a read-only transaction
   	 * but rather silently ignore the hint.
   	 * @see org.springframework.transaction.interceptor.TransactionAttribute#isReadOnly()
   	 * @see org.springframework.transaction.support.TransactionSynchronizationManager#isCurrentTransactionReadOnly()
   	 */
   	boolean readOnly() default false;
   ```

3. 事务传播行为选择

   ​	建议**改变数据**的操作update/insert/delete采用”`PROPAGATION_REQUIRED`“；

   ​	建议**不改变数据**的操作update/insert/delete采用”`PROPAGATION_SUPPORTS`“；

   网上也有用`PROPAGATION_NOT_SUPPORTED`而不是`PROPAGATION_SUPPORTS`的。我个人是觉得前者挂起事务对增删改的事务流程效率有一定影响。

   ​	而且挂起事务，虽然前面update锁定的行是排他锁(X锁)使得我select无法加共享锁(S锁)，但是不加锁不代表我获取不到数据，只不过不能再加S锁而已。平时加S锁主要就是为了不能在上面继续加X锁。我select获取到的还是修改后的(本地测试了，没能获取修改前的)。更何况挂起事务=另开一个连接线程 -> 浪费连接数。

   ​	要是读频繁导致写不进入，可以考虑用乐观锁代替事务加上去的S锁，这样不会锁住数据，但是如果乐观锁验证version不同就不读取的话，那么可能出现写频繁导致读不到的情况（当然也可以设置自旋，直到version两次读取相同，返回结果，但是同样可能导致长时间自旋，浪费资源）。

4. 事务最好在impl实现类上使用`@Transactional`，而不是在接口上。如果在接口上使用`@Transactional`，只有使用基于接口的代理时才会生效。

   且`@Transactional`只对public方法生效（不是pulic不生效也不会报错） 

5. 配置类

```java
// 启动类 添加注解
@EnableTransactionManagement // 开启事务支持
public class XXXXXApplication {
    .....
}
/**************************************************************/
/**************************************************************/
/**************************************************************/

// TransactionAopConfig.java 配置类，配置事务

@Aspect
@Configuration
public class TransactionAopConfig {

    @Autowired
    private PlatformTransactionManager transactionManager;

    @Pointcut("execution(* com.XXXX.service.impl.*.*(..))")
    public String transactionPoint(){
        return "execution(* com.XXXX.service.impl.*.*(..))";
    }

    /**
     * 事务管理配置
     */
    @Bean("txAdvice")
    public TransactionInterceptor txAdvice() {
        // 事务管理规则，承载需要进行事务管理的方法名（模糊匹配）及设置的事务管理属性
        NameMatchTransactionAttributeSource source = new NameMatchTransactionAttributeSource();

        /* 配置增C删D改U INSERT DELETE UPDATE */
        RuleBasedTransactionAttribute ruleForCUD = new RuleBasedTransactionAttribute();
        ruleForCUD.setRollbackRules(Collections.singletonList(new RollbackRuleAttribute(Exception.class))); // 抛出异常 -> 事务回滚
        ruleForCUD.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);  // 事务传播行为: 已存在事务则加入事务，否则新建事务
        ruleForCUD.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);   // 事务隔离级别：读已提交，索引没有命中时时放行锁

        /* 配置读R SELECT */
        RuleBasedTransactionAttribute ruleForR = new RuleBasedTransactionAttribute();
        ruleForR.setRollbackRules(Collections.singletonList(new RollbackRuleAttribute(Exception.class))); // 抛出异常 -> 事务回滚
        ruleForR.setPropagationBehavior(TransactionDefinition.PROPAGATION_SUPPORTS);   // 事务传播行为: 已存在事务则加入，不存在就不使用事务
        ruleForR.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED); // 事务隔离级别：读已提交，索引没有命中时时放行锁
        ruleForR.setReadOnly(true); // 只读，数据库会对只读的操作进行优化

        // 需要事务管理的方法，模糊匹配
        Map<String, TransactionAttribute> txMap = new HashMap<>();
        /* 增 */
        txMap.put("insert*", ruleForCUD);
        txMap.put("add*", ruleForCUD);
        txMap.put("create*", ruleForCUD);
        txMap.put("new*", ruleForCUD);
        /* 改 */
        //        txMap.put("update*", ruleForCUD);
        txMap.put("up*", ruleForCUD);
        txMap.put("set*", ruleForCUD);
        txMap.put("alter*", ruleForCUD);
        /* 删除 */
        txMap.put("remove*", ruleForCUD);
        txMap.put("delete*", ruleForCUD);
        txMap.put("del*", ruleForCUD);
        /* 查 */
        txMap.put("query*", ruleForR);
        txMap.put("select*", ruleForR);
        txMap.put("get*", ruleForR);
        txMap.put("find*", ruleForR);

        source.setNameMap(txMap);
        // 实例化事务拦截器
        return new TransactionInterceptor(transactionManager, source);
    }

    /**
     * 设置切面规则，容器自动注入
     */
    @Bean
    public Advisor advisor() {
        // 声明切点要切入的面
        AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
        // 设置需要被拦截的路径
        pointcut.setExpression(transactionPoint());
        // 设置切面和配置好的事务管理
        return new DefaultPointcutAdvisor(pointcut, txAdvice());
    }


}
```



## 第2章Windows平台下安装与配置MySQL

## 第3章 Linux平台下安装与配置MySQL

## 第4章 MySQL数据类型

## 第5章 操作数据库

## 第6章 创建、修改和删除表

## 第7章 索引

> [单个索引与复合索引](https://www.cnblogs.com/jiqing9006/p/10130928.html)
>
> [单索引拆成多索引。性能问题猜想。不知道对不对。](https://elasticsearch.cn/question/5015)
>
> [MySQL中explain执行计划中额外信息字段(Extra)详解](https://blog.csdn.net/poxiaonie/article/details/77757471)
>
> [MySQL explain，Extra分析（转）](https://www.cnblogs.com/myseries/p/11262054.html)
>
> [mysql索引命中规则](https://www.cnblogs.com/starluke/p/11741041.html)
>
> [SQL IN 一定走索引吗？](https://www.cnblogs.com/stoneFang/p/11032746.html)
>
> 聚簇索引
>
> [MYSQL索引：对聚簇索引和非聚簇索引的认识](https://blog.csdn.net/alexdamiao/article/details/51934917)

## 第8章 视图

## 第9章 触发器

## 第10章 查询数据

## 第11章 插入、更新与删除数据

## 第12章 MySQL运算符

## 第13章 系统信息函数

## 第14章 存储过程和函数

## 第15章 MySQL数据管理

## 第16章 数据备份与还原

## 第17章 MySQL日志

## 第18章 性能优化

## 第19章 Java访问MySQL数据库

## 第20章 PHP访问MySQL数据库

## 第21章 C#访问MySQL数据库

## 第22章 驾校学员管理系统

## 第X章 个人记录

### 1. MySQL的NULL值、空值查询；模糊查询的like、%和=比较

> [mysql 用法 Explain](https://blog.csdn.net/lvhaizhen/article/details/90763799)
>
> [MySQL_执行计划详细说明](https://www.cnblogs.com/xinysu/p/7860609.html)
>
> [MySQL执行计划extra中的using index 和 using where using index 的区别](https://www.cnblogs.com/wy123/p/7366486.html)

#### 提要

​	今天正好项目要设计数据库，再纠结以前没特地纠结的问题，那就是MySQL如果有字段可能不存在，是否要设置成NULL还是用NOT NULL default ''来设置空值，自己就无聊用3W条左右的数据试了下(数据量比较小，但是我自己电脑配置比较差，这样就已经跑了3000s左右。)

#### 数据库表格原型

​	话不多说，把测试用的表格贴上来。有4个字段，分别是自增主键`id`，标题`title`，简介`profile`，密码`password`。其中密码可能压根不存在，比如房间不需要输入密码就可以直接登录，而简介是不管有没有都需要至少显示为空值的（常理上我是这么想的）。

**下面给除了本来就是主键的id以外的title、profile、password都设置了NORMAL索引，索引方法为BTREE**

```sql
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for room
-- ----------------------------
DROP TABLE IF EXISTS `room`;
CREATE TABLE `room`  (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `title` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '标题',
  `profile` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '可以为空值',
  `password` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NULL DEFAULT NULL COMMENT '可以压根没有密码',
  PRIMARY KEY (`id`) USING BTREE,
  INDEX `title`(`title`) USING BTREE,
  INDEX `profile`(`profile`) USING BTREE,
  INDEX `password`(`password`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_0900_ai_ci ROW_FORMAT = Dynamic;

SET FOREIGN_KEY_CHECKS = 1;
```

#### 测试数据插入

​	我自己测的时候，一开始之前插入了5-6左右的垃圾数据，问题不大。(我的垃圾电脑跑了差不多3000s，要是电脑也不是很好的，建议就没必要特地尝试了)

下面重点就3句：

`insert into room values (null,CONCAT("title",i),"",NULL); `

`insert into room values (null,CONCAT("title_",i),CONCAT("",i),NULL);`

`insert into room values (null,CONCAT("title__",i),"",CONCAT("",i));`

```sql
delimiter //                            #定义标识符为双斜杠
drop procedure if exists test;          #如果存在test存储过程则删除
create procedure test()                 #创建无参存储过程,名称为test
begin
    declare i int;                      #申明变量
    set i = 0;                          #变量赋值
    lp : loop                           #lp为循环体名,可随意 loop为关键字
        insert into room values (null,CONCAT("title",i),"",NULL);    #往test表添加数据
				insert into room values (null,CONCAT("title_",i),CONCAT("",i),NULL);    #往test表添加数据
				insert into room values (null,CONCAT("title__",i),"",CONCAT("",i));    #往test表添加数据
        set i = i + 1;                  #循环一次,i加一
        if i > 10000 then                  #结束循环的条件: 当i大于10时跳出loop循环
            leave lp;
        end if; 
    end loop;
    select * from room;                 #查看test表数据
end
//                                      #结束定义语句
call test(); 
```

#### 几组测试+结果

首先，由上面的存储过程procedure，可以知道，3W条数据里面应该有2W行的`profile`为空值，2W行的`password`为NULL。然后3W条`title`都以字符串"title"开头。下面EXPLAIN指令看不懂的可以看看我上面推荐的文章，前人写得很详细了。

**下面的观点比较主观，建议要是确实想知道区别，自己动手用自己设计的数据模拟。毕竟我这里就单表查询，也没有用什么复杂的查询条件啥的**

1. password 通过判断NULL，能否用到索引，效率如何？

   从下面可以看出该查询语句`is NULL`**用到了索引**。

   *按照上面推荐的文章的意思就是，这里索引用来进行键值查找，而没有被用来实际进行查找动作。（毕竟这里is NULL就够筛选了，本身其实也算是很具体的查找条件了）。*

```sql
EXPLAIN	SELECT * FROM	room WHERE `password` is NULL;
```
| id   | select_type | table | partitions | type | possible_keys | key      | key_len | ref   | rows  | filtered | Extra                 |
| ---- | ----------- | ----- | ---------- | ---- | ------------- | -------- | ------- | ----- | ----- | -------- | --------------------- |
| 1    | SIMPLE      | room  | (Null)     | ref  | password      | password | 1023    | const | 15176 | 100.00   | Using index condition |

​	下面查询(将近2W条)时间，按照运行的次数：0.224s->0.122s -> 后面基本就是0.1~0.17之间波动了。试了下删除索引，测试后，稳定下平均0.075s。索引这里设置索引等于浪费了空间，也没得到时间上的便宜。

```sql
SELECT * FROM	room WHERE `password` is NULL;
```

**对比**

​	可以看出 `is NOT NULL`没有用到索引，直接全表查询了（主要还是因为搜索的字段是*，所有列）。

```sql
EXPLAIN	SELECT * FROM	room WHERE `password` is NOT NULL;
```

| id   | select_type | table | partitions | type | possible_keys | key    | key_len | ref    | rows  | filtered | Extra       |
| ---- | ----------- | ----- | ---------- | ---- | ------------- | ------ | ------- | ------ | ----- | -------- | ----------- |
| 1    | SIMPLE      | room  | (Null)     | ALL  | password      | (Null) | (Null)  | (Null) | 30352 | 50.00    | Using Where |

​	下面查询(将近1W条)时间，按照运行的次数(最初几次已经被漏记了)：0.68s->0.57s -> 后面基本就是0.05·0.06之间波动了。虽然走的全表查询，但是时间却差不多是`is NULL`的1/2，查询出来的数据量正好也是其1/2。==这里初略判断`is NULL`实际没有走索引，所以查询(数据量2倍)出来的时间为不走索引的`is NOT NULL`的两倍==。后面我删除索引试了下，查询时间还是差不多0.058s左右波动，也就是索引确实没有起到作用，但是也没有明显“副作用”。

```sql
SELECT * FROM	room WHERE `password` is NOT NULL;
```

2. profile使用空值，而不用NULL来标识，看看效率如何。下面我试的情况比较多，就不一一说明了，自己看看就好。

+ `= ""`因为获取的是所有字段，由于索引不是所有字段，获取所有列需要回表查询，所以没标识Using Index。

```sql
EXPLAIN SELECT * FROM	room WHERE `profile` = "";
```

| id   | select_type | table | partitions | type | possible_keys | key     | key_len | ref   | rows  | filtered | Extra |
| ---- | ----------- | ----- | ---------- | ---- | ------------- | ------- | ------- | ----- | ----- | -------- | ----- |
| 1    | SIMPLE      | room  | (Null)     | ref  | profile       | profile | 1022    | const | 15176 | 100.00   |   (Null)    |

​	这里我试了删除`profile`索引和保留索引的情况，结果**有索引**平均0.125s；**没索引**平均0.081s。没错，就是没索引反而快了，我没有打错字。

```sql
SELECT * FROM	room WHERE `profile` = "";
```

+ `=''`，因为获取所有字段，所以没有用到索引。

```sql
EXPLAIN SELECT * FROM	room WHERE `profile` = '';
```

| id   | select_type | table | partitions | type | possible_keys | key     | key_len | ref   | rows  | filtered | Extra  |
| ---- | ----------- | ----- | ---------- | ---- | ------------- | ------- | ------- | ----- | ----- | -------- | ------ |
| 1    | SIMPLE      | room  | (Null)     | ref  | profile       | profile | 1022    | const | 15152 | 100.00   | (Null) |

... 实在有点太多了。都有点打算放弃打字的形式了。

​	**上面不完整，主要觉得自己讲得不是很清楚，直接贴出我测的所有情况应该会更直观一点。反正看到这里的人应该都大概懂了。我就直接按照我想到的情况，把测试结果都贴出来得了。**

*****

下面贴出我测试的各种情况的结果（下面查询时间都是取多次查询后的稳定时间）

1. `password`和`is NULL`

+ password有设置NORMAL索引，索引方式为BTREE时

```sql
EXPLAIN SELECT * FROM	room WHERE `password` is NULL; -- Using index condition
EXPLAIN SELECT `id` FROM	room WHERE `password` is NULL; -- Using where; Using index
EXPLAIN SELECT `title` FROM	room WHERE `password` is NULL; -- Using index condition
EXPLAIN SELECT `profile` FROM	room WHERE `password` is NULL; -- Using index condition
EXPLAIN SELECT `password` FROM	room WHERE `password` is NULL; -- Using where; Using index
EXPLAIN SELECT COUNT(*) FROM	room WHERE `password` is NULL; -- Using where; Using index
```

| id   | select_type              | table | partitions | type | possible_keys            | key      | key_len | ref   | rows  | filtered | Extra                    |
| ---- | ------------------------ | ----- | ---------- | ---- | ------------------------ | -------- | ------- | ----- | ----- | -------- | ------------------------ |
| 1    | SIMPLE                   | room  | (Null)     | ref  | password                 | password | 1023    | const | 15152 | 100.00   | Using index condition    |

```sql
SELECT * FROM	room WHERE `password` is NULL; -- 0.144 
SELECT `id` FROM	room WHERE `password` is NULL; -- 0.036
SELECT `title` FROM	room WHERE `password` is NULL; -- 0.124
SELECT `profile` FROM	room WHERE `password` is NULL; -- 0.101
SELECT `password` FROM	room WHERE `password` is NULL; -- 0.025
SELECT COUNT(*) FROM	room WHERE `password` is NULL; -- 0.016
```

+ password没设置索引

| id   | select_type | table | partitions | type | possible_keys | key    | key_len | ref    | rows  | filtered | Extra        |
| ---- | ----------- | ----- | ---------- | ---- | ------------- | ------ | ------- | ------ | ----- | -------- | ------------ |
| 1    | SIMPLE      | room  | (Null)     | ALL  | password      | (Null) | (Null)  | (Null) | 30304 | 10.00    | Using  where |

```sql
SELECT * FROM	room WHERE `password` is NULL; -- 0.125s
SELECT `id` FROM	room WHERE `password` is NULL; -- 0.066s
SELECT `title` FROM	room WHERE `password` is NULL; -- 0.061s 
SELECT `profile` FROM	room WHERE `password` is NULL; -- 0.065s
SELECT `password` FROM	room WHERE `password` is NULL; -- 0.034s
SELECT COUNT(*) FROM	room WHERE `password` is NULL; -- 0.024s
```

**初略得出： Using where; Using index > Using index > Using where > Using index condition**

2. `password`和`is NOT NULL`

+ password有设置NORMAL索引，索引方式为BTREE时

```sql
EXPLAIN SELECT * FROM	room WHERE `password` is NOT NULL; 
EXPLAIN SELECT `id` FROM	room WHERE `password` is NOT NULL; 
EXPLAIN SELECT `title` FROM	room WHERE `password` is NOT NULL; 
EXPLAIN SELECT `profile` FROM	room WHERE `password` is NOT NULL; 
EXPLAIN SELECT `password` FROM	room WHERE `password` is NOT NULL; 
EXPLAIN SELECT COUNT(*) FROM	room WHERE `password` is NOT NULL; 
```

| id   | select_type | table | partitions | type  | possible_keys | key      | key_len | ref    | rows  | filtered | Extra                    |
| ---- | ----------- | ----- | ---------- | ----- | ------------- | -------- | ------- | ------ | ----- | -------- | ------------------------ |
| 1    | SIMPLE      | room  | (Null)     | ALL   | password      | (Null)   | (Null)  | (Null) | 30304 | 50.00    | Using where              |
|      |             |       |            | index |               | password | 1023    |        |       |          | Using where; Using index |
|      |             |       |            | ALL   |               | (Null)   | (Null)  |        |       |          | Using where              |
|      |             |       |            |       |               |          |         |        |       |          |                          |
|      |             |       |            | index |               | password | 1023    |        |       |          | Using where; Using index |
|      |             |       |            |       |               |          |         |        |       |          |                          |

```sql
SELECT * FROM	room WHERE `password` is NOT NULL; -- 0.046
SELECT `id` FROM	room WHERE `password` is NOT NULL; -- 0.023
SELECT `title` FROM	room WHERE `password` is NOT NULL; -- 0.026
SELECT `profile` FROM	room WHERE `password` is NOT NULL; -- 0.024
SELECT `password` FROM	room WHERE `password` is NOT NULL; -- 0.018
SELECT COUNT(*) FROM	room WHERE `password` is NOT NULL; -- 0.015
```

+ password没设置索引

| id   | select_type | table | partitions | type | possible_keys | key    | key_len | ref    | rows  | filtered | Extra       |
| ---- | ----------- | ----- | ---------- | ---- | ------------- | ------ | ------- | ------ | ----- | -------- | ----------- |
| 1    | SIMPLE      | room  | (Null)     | ALL  | (Null)        | (Null) | (Null)  | (Null) | 30304 | 90.00    | Using where |

```sql
SELECT * FROM	room WHERE `password` is NOT NULL; -- 0.053
SELECT `id` FROM	room WHERE `password` is NOT NULL; -- 0.036
SELECT `title` FROM	room WHERE `password` is NOT NULL; -- 0.036
SELECT `profile` FROM	room WHERE `password` is NOT NULL; -- 0.032
SELECT `password` FROM	room WHERE `password` is NOT NULL; -- 0.029
SELECT COUNT(*) FROM	room WHERE `password` is NOT NULL; -- 0.039
```

3. `profile` 和 `=''`

+ profile有设置NORMAL索引，索引方式为BTREE时(没标注的默认和上一行一样，或者都一样)

```sql
EXPLAIN SELECT * FROM	room WHERE `profile` =''; -- (Null)
EXPLAIN SELECT `id` FROM	room WHERE `profile` =''; -- Using index 
EXPLAIN SELECT `title` FROM	room WHERE `profile` =''; -- (Null) 
EXPLAIN SELECT `profile` FROM	room WHERE `profile` =''; -- Using index 
EXPLAIN SELECT `password` FROM	room WHERE `profile` =''; -- (Null) 
EXPLAIN SELECT COUNT(*) FROM	room WHERE `profile` =''; -- Using index 
```

| id   | select_type | table | partitions | type | possible_keys | key     | key_len | ref   | rows  | filtered | Extra  |
| ---- | ----------- | ----- | ---------- | ---- | ------------- | ------- | ------- | ----- | ----- | -------- | ------ |
| 1    | SIMPLE      | room  | (Null)     | ref  | profile       | profile | 1022    | const | 15152 | 100.00   | (Null) |

```sql
SELECT * FROM	room WHERE `profile` =''; -- 0.164
SELECT `id` FROM	room WHERE `profile` =''; -- 0.053
SELECT `title` FROM	room WHERE `profile` =''; -- 0.105
SELECT `profile` FROM	room WHERE `profile` =''; -- 0.029
SELECT `password` FROM	room WHERE `profile` =''; -- 0.093
SELECT COUNT(*) FROM	room WHERE `profile` =''; -- 0.015
```

光这么看，用=''所有上面列举的情况看下来，平均会比NULL的空值查找快一点。

+ profile没设置索引

| id   | select_type | table | partitions | type | possible_keys | key    | key_len | ref    | rows  | filtered | Extra       |
| ---- | ----------- | ----- | ---------- | ---- | ------------- | ------ | ------- | ------ | ----- | -------- | ----------- |
| 1    | SIMPLE      | room  | (Null)     | ALL  | (Null)        | (Null) | (Null)  | (Null) | 30304 | 10.00    | Using where |

```sql
SELECT * FROM	room WHERE `profile` =''; -- 0.09
SELECT `id` FROM	room WHERE `profile` =''; -- 0.114
SELECT `title` FROM	room WHERE `profile` =''; -- 0.081
SELECT `profile` FROM	room WHERE `profile` =''; -- 0.049
SELECT `password` FROM	room WHERE `profile` =''; -- 0.069
SELECT COUNT(*) FROM	room WHERE `profile` =''; -- 0.003
```

4. `profile` 和 `!=''`

+ profile有设置NORMAL索引，索引方式为BTREE时(没标注的默认和上一行一样，或者都一样)

```sql
EXPLAIN SELECT * FROM	room WHERE `profile` !=''; -- ALL
EXPLAIN SELECT `id` FROM	room WHERE `profile` !=''; -- range
EXPLAIN SELECT `title` FROM	room WHERE `profile` !=''; -- ALL
EXPLAIN SELECT `profile` FROM	room WHERE `profile` !=''; -- range
EXPLAIN SELECT `password` FROM	room WHERE `profile` !=''; -- ALL
EXPLAIN SELECT COUNT(*) FROM	room WHERE `profile` !=''; -- range
```

| id   | select_type | table | partitions | type  | possible_keys | key     | key_len | ref    | rows  | filtered | Extra                    |
| ---- | ----------- | ----- | ---------- | ----- | ------------- | ------- | ------- | ------ | ----- | -------- | ------------------------ |
| 1    | SIMPLE      | room  | (Null)     | ALL   | profile       | (Null)  | (Null)  | (Null) | 30304 | 33.06    | Using where              |
|      |             |       |            | range |               | profile | 1022    |        | 10017 | 100.00   | Using where; Using index |

```sql
SELECT * FROM	room WHERE `profile` !=''; -- 0.055
SELECT `id` FROM	room WHERE `profile` !=''; -- 0.017
SELECT `title` FROM	room WHERE `profile` !=''; -- 0.039
SELECT `profile` FROM	room WHERE `profile` !=''; -- 0.015
SELECT `password` FROM	room WHERE `profile` !=''; -- 0.034
SELECT COUNT(*) FROM	room WHERE `profile` !=''; -- 0.009
```

+ profile没设置索引

| id   | select_type | table | partitions | type | possible_keys | key    | key_len | ref    | rows  | filtered | Extra       |
| ---- | ----------- | ----- | ---------- | ---- | ------------- | ------ | ------- | ------ | ----- | -------- | ----------- |
| 1    | SIMPLE      | room  | (Null)     | ALL  | (Null)        | (Null) | (Null)  | (Null) | 30304 | 90.00    | Using where |

```sql
SELECT * FROM	room WHERE `profile` !=''; -- 0.041
SELECT `id` FROM	room WHERE `profile` !=''; -- 0.023
SELECT `title` FROM	room WHERE `profile` !=''; -- 0.037
SELECT `profile` FROM	room WHERE `profile` !=''; -- 0.026
SELECT `password` FROM	room WHERE `profile` !=''; -- 0.025
SELECT COUNT(*) FROM	room WHERE `profile` !=''; -- 0.017
```

可能因为我数据量太少，`!=''`没看出来有比`is NOT NULL`好到哪去，但是还是能大致推测数据量大的时候，`!=''`会比`is NOT NULL`表现好。

​	**至少到这里为止，我靠上面的数据+一些主观推测，认为要是单表情况下考虑索引的效率，那么空值会比NULL效率高。**

5. `profile` 和 `=""`

+ profile有设置NORMAL索引，索引方式为BTREE时

```sql
EXPLAIN SELECT * FROM	room WHERE `profile` =""; 
EXPLAIN SELECT `id` FROM	room WHERE `profile` =""; 
EXPLAIN SELECT `title` FROM	room WHERE `profile` =""; 
EXPLAIN SELECT `profile` FROM	room WHERE `profile` =""; 
EXPLAIN SELECT `password` FROM	room WHERE `profile` =""; 
EXPLAIN SELECT COUNT(*) FROM	room WHERE `profile` =""; 
-- 结果和 ='' 一样，就不贴出来了
```

```sql
SELECT * FROM	room WHERE `profile` =""; -- 0.122
SELECT `id` FROM	room WHERE `profile` =""; -- 0.037
SELECT `title` FROM	room WHERE `profile` =""; -- 0.093
SELECT `profile` FROM	room WHERE `profile` =""; -- 0.026
SELECT `password` FROM	room WHERE `profile` =""; -- 0.089
SELECT COUNT(*) FROM	room WHERE `profile` =""; -- 0.016
-- 和 ='' 结果差不多下面不多做展开，毕竟也都说""和''没区别，我就好奇试试而已，看看有没有啥‘魔法’
```

6. `profile` 和 `like ''`

+ profile有设置NORMAL索引，索引方式为BTREE时

```sql
EXPLAIN SELECT * FROM	room WHERE `profile` like ''; -- ALL
EXPLAIN SELECT `id` FROM	room WHERE `profile` like ''; -- index
EXPLAIN SELECT `title` FROM	room WHERE `profile` like ''; -- ALL
EXPLAIN SELECT `profile` FROM	room WHERE `profile` like ''; -- index
EXPLAIN SELECT `password` FROM	room WHERE `profile` like ''; -- ALL
EXPLAIN SELECT COUNT(*) FROM	room WHERE `profile` like ''; -- index
```

| id   | select_type | table | partitions | type  | possible_keys | key     | key_len | ref  | rows  | filtered | Extra        |
| ---- | ----------- | ----- | ---------- | ----- | ------------- | ------- | ------- | ---- | ----- | -------- | ------------ |
| 1    | SIMPLE      | room  | (Null)     | ALL   | profile       | (Null)  | (Null)  |      | 30304 | 50.00    | Using where  |
|      |             |       |            | index |               | profile | 1022    |      | 30304 | 50.00    | Using where; Using index |

```sql
SELECT * FROM	room WHERE `profile` like ''; -- 0.071
SELECT `id` FROM	room WHERE `profile` like ''; -- 0.036
SELECT `title` FROM	room WHERE `profile` like ''; -- 0.048
SELECT `profile` FROM	room WHERE `profile` like ''; -- 0.032
SELECT `password` FROM	room WHERE `profile` like ''; -- 0.044
SELECT COUNT(*) FROM	room WHERE `profile` like ''; -- 0.022
```

+ profile没设置索引

| id   | select_type | table | partitions | type | possible_keys | key    | key_len | ref    | rows  | filtered | Extra       |
| ---- | ----------- | ----- | ---------- | ---- | ------------- | ------ | ------- | ------ | ----- | -------- | ----------- |
| 1    | SIMPLE      | room  | (Null)     | ALL  | (Null)        | (Null) | (Null)  | (Null) | 30304 | 11.11    | Using where |

```sql
SELECT * FROM	room WHERE `profile` like ''; -- 0.094
SELECT `id` FROM	room WHERE `profile` like ''; -- 0.047
SELECT `title` FROM	room WHERE `profile` like ''; -- 0.044
SELECT `profile` FROM	room WHERE `profile` like ''; -- 0.04
SELECT `password` FROM	room WHERE `profile` like ''; -- 0.042
SELECT COUNT(*) FROM	room WHERE `profile` like ''; -- 0.02
```

​	**从上面几个比较，考虑like一般用于模糊查询，且从EXPLAIN的结果和查询的时间来看，有字段属性犹豫要为NULL还是空值''，如果极端要求索引速度可以使用''空值。（我这里数据量少、查询情况也不够复杂，可能没啥太大说服力）。要是NULL没有什么特别的优化的话(这个我没怎么了解)，我个人可能偏向用空值而不是NULL，因为NULL再之后要是修改数据库属性或者java类，会有更多协商问题，但是如果用空值，就可以省去很多事情。**

7. `title`  和 `like 'title%'`

+ title有设置NORMAL索引，索引方式为BTREE时

```sql
EXPLAIN SELECT * FROM	room WHERE `title` like 'title%'; -- ALL
EXPLAIN SELECT `id` FROM	room WHERE `title` like 'title%'; -- index
EXPLAIN SELECT `title` FROM	room WHERE `title` like 'title%'; -- index
EXPLAIN SELECT `profile` FROM	room WHERE `title` like 'title%'; -- ALL
EXPLAIN SELECT `password` FROM	room WHERE `title` like 'title%'; -- ALL
EXPLAIN SELECT COUNT(*) FROM	room WHERE `title` like 'title%'; -- index
```

| id   | select_type | table | partitions | type  | possible_keys | key    | key_len | ref    | rows  | filtered | Extra           |
| ---- | ----------- | ----- | ---------- | ----- | ------------- | ------ | ------- | ------ | ----- | -------- | --------------- |
| 1    | SIMPLE      | room  | (Null)     | ALL   | title         | (Null) | (Null)  | (Null) | 30304 | 50.00    | Using  where    |
|      |             |       |            | index | title         | title  | 1022    |        |       | 50.00    | Using  where; Using index |

```sql
SELECT * FROM	room WHERE `title` like 'title%'; -- 0.103
SELECT `id` FROM	room WHERE `title` like 'title%'; -- 0.087
SELECT `title` FROM	room WHERE `title` like 'title%'; -- 0.08
SELECT `profile` FROM	room WHERE `title` like 'title%'; -- 0.046
SELECT `password` FROM	room WHERE `title` like 'title%'; -- 0.048
SELECT COUNT(*) FROM	room WHERE `title` like 'title%'; -- 0.042
```

+ title没设置索引

| id   | select_type | table | partitions | type | possible_keys | key    | key_len | ref    | rows  | filtered | Extra       |
| ---- | ----------- | ----- | ---------- | ---- | ------------- | ------ | ------- | ------ | ----- | -------- | ----------- |
| 1    | SIMPLE      | room  | (Null)     | ALL  | (Null)        | (Null) | (Null)  | (Null) | 30304 | 11.11    | Using where |

```sql
SELECT * FROM	room WHERE `title` like 'title%'; -- 0.089
SELECT `id` FROM	room WHERE `title` like 'title%'; -- 0.049
SELECT `title` FROM	room WHERE `title` like 'title%'; -- 0.053
SELECT `profile` FROM	room WHERE `title` like 'title%'; -- 0.049
SELECT `password` FROM	room WHERE `title` like 'title%'; -- 0.045
SELECT COUNT(*) FROM	room WHERE `title` like 'title%'; -- 0.027
```

8. `title`  和 `like '%title%'`

+ title有设置NORMAL索引，索引方式为BTREE时

```sql
EXPLAIN SELECT * FROM	room WHERE `title` like '%title%'; -- ALL
EXPLAIN SELECT `id` FROM	room WHERE `title` like '%title%'; -- index
EXPLAIN SELECT `title` FROM	room WHERE `title` like '%title%'; -- index
EXPLAIN SELECT `profile` FROM	room WHERE `title` like '%title%'; -- ALL
EXPLAIN SELECT `password` FROM	room WHERE `title` like '%title%'; -- ALL
EXPLAIN SELECT COUNT(*) FROM	room WHERE `title` like '%title%'; -- index
```

| id   | select_type | table | partitions | type  | possible_keys | key    | key_len | ref    | rows  | filtered | Extra                     |
| ---- | ----------- | ----- | ---------- | ----- | ------------- | ------ | ------- | ------ | ----- | -------- | ------------------------- |
| 1    | SIMPLE      | room  | (Null)     | ALL   | (Null)        | (Null) | (Null)  | (Null) | 30304 | 11.11    | Using where               |
|      |             |       |            | index |               | title  | 1022    |        | 30304 |          | Using  where; Using index |

```sql
SELECT * FROM	room WHERE `title` like '%title%'; -- 0.045
SELECT `id` FROM	room WHERE `title` like '%title%'; --  0.025
SELECT `title` FROM	room WHERE `title` like '%title%'; -- 0.024
SELECT `profile` FROM	room WHERE `title` like '%title%'; -- 0.031
SELECT `password` FROM	room WHERE `title` like '%title%'; -- 0.03
SELECT COUNT(*) FROM	room WHERE `title` like '%title%'; -- 0.012
```

+ title没设置索引

```sql
EXPLAIN SELECT * FROM	room WHERE `title` like '%title%'; -- ALL
EXPLAIN SELECT `id` FROM	room WHERE `title` like '%title%'; -- index
EXPLAIN SELECT `title` FROM	room WHERE `title` like '%title%'; -- index
EXPLAIN SELECT `profile` FROM	room WHERE `title` like '%title%'; -- ALL
EXPLAIN SELECT `password` FROM	room WHERE `title` like '%title%'; -- ALL
EXPLAIN SELECT COUNT(*) FROM	room WHERE `title` like '%title%'; -- index
-- `title`  和 `like 'title%'`, 和这个没区别
```

```sql
SELECT * FROM	room WHERE `title` like '%title%'; -- 0.118
SELECT `id` FROM	room WHERE `title` like '%title%'; -- 0.056
SELECT `title` FROM	room WHERE `title` like '%title%'; -- 0.076
SELECT `profile` FROM	room WHERE `title` like '%title%'; -- 0.042
SELECT `password` FROM	room WHERE `title` like '%title%'; -- 0.044
SELECT COUNT(*) FROM	room WHERE `title` like '%title%'; -- 0.023
```

#### 总结

​	like 模糊查询的%不要乱用，够用就好，但是一般图方便都是%string%，这种场景也相对比较多。

​	如果是确定值的字符串比较，那当然还是用=，而不是模糊查询了。

​	然后就是NULL和空值的选择，速度上我这里极少量数据是认为空值快一点，而且可以避免一些麻烦的协商问题。但是NULL也有NULL的特性，比如COUNT(含有NULL的列)是不会记录含有NULL的行的。不过要是为了方便后期调整，个人觉得空值会更方便一点。

### 2. 一些数据库连接、配置坑

> Druid:
>
> [spring boot 2.1.3 打开 druid 连接池监控报错 Sorry, you are not permitted to view this page.](https://blog.csdn.net/mxcai2005/article/details/89928806)

### 3. 幂等

> [幂等技术及实现方式](https://blog.csdn.net/xktxoo/article/details/96765193)