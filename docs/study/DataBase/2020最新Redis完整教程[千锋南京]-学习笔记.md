# 2020最新Redis完整教程[千锋南京]-学习笔记

> 视频链接：[2020最新Redis完整教程[千锋南京]](https://www.bilibili.com/video/av82325430) https://www.bilibili.com/video/av82325430

## 1. 一些网文搜集

> [docker-compose一键部署redis一主二从三哨兵模式(含密码,数据持久化)](https://www.cnblogs.com/hckblogs/p/11186311.html)
>
> [Redis计算地理位置距离-GeoHash](https://www.cnblogs.com/wt645631686/p/8454497.html)
>
> [阿里云Redis开发规范](https://yq.aliyun.com/articles/531067)
>
> [redis中hash和string的使用场景](https://www.jianshu.com/p/4537467bb593)
>
> 存储对象的方式选择：数据量大追求速度->序列化；不是极致速度追求且要求可视化->json
>
> [redis缓存对象，该选择json还是序列化?](https://developer.aliyun.com/ask/61601?spm=a2c6h.13159736)
>
> [在redis队列中将对象序列化存进去队列好还是用json好？哪个性能比较快？](https://www.zhihu.com/question/265671476/answer/297005726)
>
> [Redis几种对象序列化的比较](https://www.jianshu.com/p/e72ec3681fea)
>
> 分布式锁
>
> [Springboot分别使用乐观锁和分布式锁（基于redisson）完成高并发防超卖](https://blog.csdn.net/tianyaleixiaowu/article/details/90036180)
>
> [redis乐观锁（适用于秒杀系统）](https://www.cnblogs.com/crazylqy/p/7742097.html)
>
> Redis基本操作：
>
> [redis 获取key 过期时间](https://blog.csdn.net/zhaoyangjian724/article/details/51790977)
>
> [请问，redis为什么是单线程？](https://www.nowcoder.com/questionTerminal/9e7c2b4fff1d4507814346cf370fa8f6)
>
> SpringBoot-Redis连接超时-坑
>
> [springboot项目中redis连接超时问题](https://blog.csdn.net/distinySmile/article/details/105192539)
>
> [redis报错远程主机强迫关闭了一个现有的连接以及超时问题](http://www.classinstance.cn/detail/77.html)
>
> [springboot整合redis时使用Jedis代替Lettuce](https://blog.csdn.net/xianyirenx/article/details/84207393)
>
> 缓存
>
> [redis缓存在项目中的使用](https://www.cnblogs.com/fengli9998/p/6755591.html)
>
> 持久化机制
>
> [redis的 rdb 和 aof 持久化的区别](https://www.cnblogs.com/shizhengwen/p/9283973.html)
>
> [Redis持久化介绍 ](https://www.sohu.com/a/359201984_100233510)
>
> Redis 有两种持久化方案,RDB (Redis DataBase)和 AOF (Append Only File)
>
> - appendonly 配置redis 默认关闭，开启需要手动把no改为yes（这个我dockerhub中的官方redis6的docker-compose默认是开启的`redis-server /etc/redis/redis.conf --appendonly yes`）
> - appendfilename指定本地数据库文件名，默认值为 appendonly.aof
> - appendfsync everysec指定更新日志条件为每秒更新，共三种策略（aways，everyse，no）
> - auto-aof-rewrite-min-size配置重写触发机制，当AOF文件大小是上次rewrite后大小的一倍且文件大于64M时触发。
>
> **AOF触发与恢复**
>
> AOF主要根据配置文件策略触发，可以是每次执行触发，可以是每秒触发，可以不同步。
>
> AOF的恢复主要是将appendonly.aof 文件拷贝到redis的安装目录的bin目录下，重启redis服务即可。但在实际开发中，可能因为某些原因导致appendonly.aof 文件异常，从而导致数据还原失败，可以通过命令redis-check-aof --fix appendonly.aof 进行修复
>
> AOF的工作原理是将写操作追加到文件中，文件的冗余内容会越来越多。所以Redis 新增了重写机制，通过auto-aof-rewrite-min-size控制。当AOF文件的大小超过所设定的阈值时，Redis就会对AOF文件的内容压缩。
>
> 
>
> 消息队列/发布订阅模式
>
> [redis实现消息队列&发布/订阅模式使用](https://www.cnblogs.com/qlqwjy/p/9763754.html)
>
> 
>
> 事务
>
> [Redis事务,你真的了解吗](https://zhuanlan.zhihu.com/p/101902825?utm_source=wechat_session)
>
> [Redis事务详解，Redis事务不支持回滚吗？](https://baijiahao.baidu.com/s?id=1613631210471699441&wfr=spider&for=pc)
>
> Redis自身的事务不能够回滚，只不过事务异常or失败会清空原本将要继续往下执行的事务队列里的操作。
>
> 只有类型错误、参数错误等执行前语法就能检测出来的会终止Redis的事务（前面执行过的也不会回滚）。
>
> 然后如果EXEC了，这时候如果MULTI里面的操作正好数据不存在了什么的异常，也不会回滚。EXEC后，不管如何，都会把原本MULTI的执行完。
>
> **Redis的事务执行时，其他client的操作会被阻塞**。
>
> 非事务状态下的命令以单个命令为单位执行，前一个命令和后一个命令的客户端不一定是同一个；
> 而事务状态则是以一个事务为单位，执行事务队列中的所有命令：除非当前事务执行完毕，否则服务器不会中断事务，也不会执行其他客户端的其他命令。（把Redis事务中的所有操作一次性发给Redis，减少网络带来的时间损耗）
>
> **通过事务+watch实现 CAS**
>
> Redis 提供了 WATCH 命令与事务搭配使用，实现 CAS 乐观锁的机制。
> WATCH 的机制是：在事务 EXEC 命令执行时，Redis 会检查被 WATCH 的 Key，只有被 WATCH 的 Key 从 WATCH 起始时至今没有发生过变更，EXEC 才会被执行。
> 如果 WATCH 的 Key 在 WATCH 命令到 EXEC 命令之间发生过变化，则 EXEC 命令会返回失败。



## 2. 地理位置处理---Redis的GeoHash和MySQL的geography类型（之后有空再详细介绍）

> 最近比较忙，不细讲两者的什么特点啥的了。

### 1. MySQL的geography

+ 适合查找某个**指定范围内**的物体（比如一个多边形内的）
+ 适合需**较高精确**位置关系的场景。比如传来一个用户坐标，如果需要较精确的定位，来确定用户是否在A区域还是B区域，可以用这个。
+ geography往往伴随着函数计算传入的点Point位置是否在某个区域Polygon内，这个据需要数据库函数。因为走函数等于每一行都需要计算，所以是不走索引的，需要全表查找，效率就很低。设置空间索引Spatial index，可以加速这一过程（具体原理没细究，有兴趣可以自己去查一查）。

​	前面提到函数操作，不走索引。可以了解下virtual column，虚拟列。

​	过去把函数操作的值另外存一列，比如插入时间，需要另外存个时间戳版本，可以数据库写个触发器，插入时制动转化然后存到新列里面。但是这样麻烦又低效。可以新建一列，设置为虚拟列，其值是函数计算结果，这样插入数据时虚拟列的值会自动计算，而且由于虚拟列直接存在数据字典而不是持久化到磁盘，效率很高（类似于函数索引了）。

### 2. Redis的GeoHash

+ 适合只简单获取**某二维点周围范围情况**。比如当前位置周围有多少用户在线。
+ 适合**精度低**的场景。这个和GeoHash的算法有关，其本身是把地图二维化，把地图一直切两刀分成4块。具体的块状大小，和传入的经度、维度的有效位数有关。详细算法那些这里不讲，有兴趣可以到我github的DB笔记看看一些网络文章的讲解。

### 3. 两者对比

下面对比，指的是实现的方式不复杂的情况。

1. 位置关系**精度**：MySQL的geography > Redis的GeoHash
2. 实现**难易度**：    MySQL的geography > Redis的GeoHash

3. 如果只是获取某点周围**大致**情况，**效率**：MySQL的geography < Redis的GeoHash



最好根据自己实际场景使用对应的技术，不是什么难用什么就对。