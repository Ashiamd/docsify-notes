# 《Redis5设计与源码分析》-读书笔记

# 1. 引言

+ 基于内存的存储数据库，绝大部分指令只是存粹的内存操作，读写快
+ 单进程线程（实际上一个正在运行的RedisServer肯定不止一个线程，但只有一个线程来处理网络请求），避免了不必要的上下文切换，同时不存在加锁/释放锁等同步操作
+ 使用多路I/O复用模型（select、poll、epoll），可以高效处理大量并发连接
+ 数据结构是专门设计的，增、删、改、查等操作相对简单

## 1.1 Redis简介

1. 内存型数据库
2. 单线程，瓶颈为内存和带宽，而非CPU
3. key-value中的value支持多种数据结构
4. 支持持久化，RDB、AOF、RDB&AOF
5. 支持主从，从实例进行数据备份

## 1.2 Redis5.0 新特性

书中只列举几个重要特性/改动：

1. 新增Streams数据类型，可以作为消息队列
2. 新的模块API、定时器、集群及字典
3. RDB中持久化存储LFU和LRU的信息
4. 将集群管理功能完全用C语言集成到redis-cli中，Redis 3.x和Redis 4.x的集群管理是通过Ruby脚本实现的
5. 有序集合新增命令ZPOPMIN/ZPOPMAX
6. 改进HyperLogLog的实现
7. 新增Client Unblock和Client ID
8. 新增LOLWUT命令
9. Redis主从复制中的从不再称为Slave，改称Replicas
10. Redis 5.0引入动态哈希，以平衡CPU的使用率和相应性能，可以通过配置文件进行配置。Redis 5.0默认使用动态哈希
11. Redis核心代码进行了部分重构和优化

## 1.3 源码概述

​	Redis源代码主要存放在src文件夹中，其中server.c为服务端程序，redis-cli.c为客户端程序。

​	核心部分主要如下：

### 1. 基本的数据结构

+ 动态字符串`sds.c`
+ 整数集合`intset.c`
+ 压缩列表`ziplist.c`
+ 快速链表`quicklist.c`
+ 字典`dict.c`
+ Streams的底层实现结构`listpack.c`和`rax.c`

### 2. 数据类型的底层实现

+ Redis对象`object.c`

+ 字符串`t_string.c`

+ 列表`t_list.c`

+ 字典`t_hash.c`

+ 集合及有序集合`t_set.c`和`t_zset.c`

+ 数据流`t_stream.c`

### 3. 数据库的实现

+ 数据库的底层实现`db.c`

+ 持久化`rdb.c`和`aof.c`

### 4. 服务端和客户端实现

+ 事件驱动`ae.c`和`ae_epoll.c`

+ 网络连接`anet.c`和`networking.c`
+ 服务端程序`server.c`
+ 客户端程序`redis-cli.c`

### 5. 其他

+ 主从复制`replication.c`
+ 哨兵`sentinel.c`
+ 集群`cluster.c`
+ 其他数据结构，如`hyperloglog.c`、`geo.c`等
+ 其他功能，如pub/sub、Lua脚本

## 1.4 安装与调试

+ redis-benchmark：官方自带的Redis性能测试工具

+ redis-check-aof、redis-check-rdb：AOF或RDB文件出现语法错误时，修复工具
+ redis-cli：客户端命令行工具，可以通过命令`redis-clih{host}-p{port}`连接到指定Redis服务器
+ redis-sentinel：Redis哨兵启动程序
+ redis-server：Redis服务端启动程序

## 1.5 本章小结

# 2. 简单动态字符串

​	简单动态字符串（Simple Dynamic Strings，SDS）是Redis的基本数据结构之一，用于存储字符串和整型数据。SDS兼容C语言标准字符串处理函数，且在此基础上保证了**二进制安全**。

## 2.1 数据结构

> 二进制安全：，C语言中，用`\0`表示字符串的结束，如果字符串中本身就有`\0`字符，字符串就会被截断，即非二进制安全；若通过某种机制，保证读写字符串时不损害其内容，则是二进制安全

### Redis 3.2 之前的设计

```c
struct sds {
  int len;// buf 中已占用字节数
  int free;// buf 中剩余可用字节数
  char buf[];// 数据空间
};
```

> 注意：上例中的`buf[]`是一个柔性数组。柔性数组成员（flexible array member），也叫伸缩性数组成员，**只能被放在结构体的末尾**。包含柔性数组成员的结构体，通过malloc函数为柔性数组动态分配内存。

​	之所以用柔性数组存放字符串，是因为柔性数组的地址和结构体是连续的，这样查找内存更快（因为不需要额外通过指针找到字符串的位置）；可以很方便地通过柔性数组的首地址偏移得到结构体首地址，进而能很方便地获取其余变量。

---

整体设计的优点：

+ 有单独的统计变量len和free（称为头部），可快速得到字符串长度
+ 内容存放柔性数组buf中，对上层暴露柔性数组buf的指针（而非结构体指针），兼容C语言处理字符串的各种函数（为兼容C语言，最后也是额外申请一个字节，存放`\0`字符，不计算到len变量中）
+ 有len记录字符串长度，读写字符串时，不依赖`\0`终止符，保证二进制安全

### Redis 3.2 之后的内存优化

> [ C语言，如何不使用 __attribute__ ((packed)) 解决字节对齐的问题？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/265875753)
>
> [C语言sizeof求结构体的大小_林伟茂的博客-CSDN博客_c语言sizeof结构体](https://blog.csdn.net/weixin_42814000/article/details/105212284)
>
> [C语言柔性数组详解_高邮吴少的博客-CSDN博客_c语言柔性数组](https://blog.csdn.net/m0_57180439/article/details/120654912#_16)

​	如果存放的字符串实际长度很短，比如只占用1个字节，而头部信息中len、free却已经占用2个字节，显得臃肿。Redis后续考虑不同情况下的字符串内存分配，对头部信息进行了压缩处理。

​	增加一个字段flags标识SDS的不同长度类型（长度1字节、2字节、4字节、8字节、小于1字节），至少要用3位存储类型（2^3=8），1字节剩余5位可以存储长度，可以满足长度小于32的字符串。

![在这里插入图片描述](https://img-blog.csdnimg.cn/8f4856a8ac0c4ee3b3d79a4697e5c873.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5p-P5rK5,size_20,color_FFFFFF,t_70,g_se,x_16)

​	在Redis5.0中，用如下结构存储长度小于32的短字符串

```c
struct __attribute__ ((__packed__))sdshdr5 {
  unsigned char flags; /* 低3位存储类型, 高5位存储长度 */
  char buf[];/*柔性数组，存放实际内容*/
};
```

​	长度>=32的字符串，代码如下：

```c
struct __attribute__((__packed__))sdshdr8 {
  uint8_t len; /* 已使用长度，用1字节存储 */
  uint8_t alloc; /* 总长度，用1字节存储*/
  unsigned char flags; /* 低3位存储类型, 高5位预留 */
  char buf[];/*柔性数组，存放实际内容*/
};
struct __attribute__((__packed__))sdshdr16 {
  uint16_t len; /*已使用长度，用2字节存储*/
  uint16_t alloc; /* 总长度，用2字节存储*/
  unsigned char flags; /* 低3位存储类型, 高5位预留 */
  char buf[];/*柔性数组，存放实际内容*/
};
struct __attribute__((__packed__))sdshdr32 {
  uint32_t len; /*已使用长度，用4字节存储*/
  uint32_t alloc; /* 总长度，用4字节存储*/
  unsigned char flags;/* 低3位存储类型, 高5位预留 */
  char buf[];/*柔性数组，存放实际内容*/
};
struct __attribute__((__packed__))sdshdr64 {
  uint64_t len; /*已使用长度，用8字节存储*/
  uint64_t alloc; /* 总长度，用8字节存储*/
  unsigned char flags; /* 低3位存储类型, 高5位预留 */
  char buf[];/*柔性数组，存放实际内容*/
};
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/706ae6e238214b38977c779f6218ee7a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5p-P5rK5,size_20,color_FFFFFF,t_70,g_se,x_16)

+ len：buf中已占用的字节数
+ alloc：不同于原本的free，表示buf分配的总字节数
+ flags：标识当前结构体的（长度）类型，低3位用作标识位，高5位预留。

+ buf：柔性数组，真正存储字符串的数据空间。

​	源码中的`__attribute__((__packed__))`需要重点关注。一般情况下，结构体会按其所有变量大小的最小公倍数做字节对齐，而**用packed修饰后，结构体则变为按1字节对齐**。以sdshdr32为例，修饰前按4字节对齐大小为12(4×3)字节；修饰后按1字节对齐，**注意buf是个char类型的柔性数组，地址连续，始终在flags之后**。

​	使用`__attribute__((__packed__))`的好处：

+ 节省（头部信息占用的）内存，例如sdshdr32可节省3个字节
+ **SDS返回给上层的，不是结构体首地址，而是指向内容的buf指针**。因为此时按1字节对齐，故SDS创建成功后，无论是sdshdr8、sdshdr16还是sdshdr32，都能**通过`(char*)sh+hdrlen`得到buf指针地址**（其中hdrlen是结构体长度，通过sizeof计算得到。**sizeof结构体时，不会返回柔性数组的内存**）。修饰后，无论是sdshdr8、sdshdr16还是sdshdr32，都能**通过`buf[-1]`找到flags**，因为此时按1字节对齐。若没有packed的修饰，还需要对不同结构进行处理，实现更复杂。

![在这里插入图片描述](https://img-blog.csdnimg.cn/c7e3b2b8e96e4b718c26ffa212f5a758.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5p-P5rK5,size_20,color_FFFFFF,t_70,g_se,x_16)

## 2.2 基本操作

P39
