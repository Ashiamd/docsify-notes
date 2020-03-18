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
> Redis基本操作：
>
> [redis 获取key 过期时间](https://blog.csdn.net/zhaoyangjian724/article/details/51790977)
>
> [请问，redis为什么是单线程？](https://www.nowcoder.com/questionTerminal/9e7c2b4fff1d4507814346cf370fa8f6)



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