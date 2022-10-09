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

### 2.2.1 创建字符串

​	Redis通过`sdsnewlen`函数创建SDS。

```c
// 现在Redis最新代码7.0看这里sdsnewlen内部实现和书本描述有点差别，不过整体差别不大
sds sdsnewlen(const void *init, size_t initlen) {
  void *sh;
  sds s;
  char type = sdsReqType(initlen);//根据字符串长度选择不同的类型
  /* Empty strings are usually created in order to append. Use type 8
     * since type 5 is not good at this. */
  if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;//SDS_TYPE_5强制转化为SDS_TYPE_8
  int hdrlen = sdsHdrSize(type);//计算不同头部所需的长度
  unsigned char *fp; /* 指向flags的指针 */
  sh = s_malloc(hdrlen+initlen+1);//"+1"是为了结束符'\0'
  ...
    s = (char*)sh+hdrlen;//s是指向buf的指针
  fp = ((unsigned char*)s)-1;//s是柔性数组buf的指针,-1即指向flags
  ...
    s[initlen] = '\0';//添加末尾的结束符
  return s;
}
```

创建SDS的大致流程：

1. 根据初始化长度判断字符串长度类型（空字符串会强转为SDS_TYPE_8，因为空字符串一般后续有频繁的变动）
2. 根据长度类型，申请字符数组内存（会给最后填充`\0`，所以多申请1字节内存）
3. 设置flags（即标注字符串的长度类型SDS_TYPE_5等）
4. 末尾填充`\0`，返回字符数组指针（而非结构体指针）

### 2.2.2 释放字符串

​	SDS提供了直接释放内存的方法——`sdsfree`，该函数通过对s的偏移，可定位到SDS结构体的首部，然后调用`s_free`释放内存。

```c
void sdsfree(sds s) {
  if (s == NULL) return;
  s_free((char*)s-sdsHdrSize(s[-1]));
}
```

​	另外也提供了仅将len归零，下次继续复用原本内存空间的`sdsclear`函数

```c
/* Modify an sds string in-place to make it empty (zero length).
 * However all the existing buffer is not discarded but set as free space
 * so that next append operations will not require allocations up to the
 * number of bytes previously available. */
void sdsclear(sds s) {
  sdssetlen(s, 0);
  s[0] = '\0';
}
```

### 2.2.3 拼接字符串

通过`sdscatsds`函数拼接字符串。

```c
/* Append the specified sds 't' to the existing sds 's'.
 *
 * After the call, the modified sds string is no longer valid and all the
 * references must be substituted with the new pointer returned by the call. */
sds sdscatsds(sds s, const sds t) {
  return sdscatlen(s, t, sdslen(t));
}

/* Append the specified binary-safe string pointed by 't' of 'len' bytes to the
 * end of the specified sds string 's'.
 *
 * After the call, the passed sds string is no longer valid and all the
 * references must be substituted with the new pointer returned by the call. */
sds sdscatlen(sds s, const void *t, size_t len) {
    size_t curlen = sdslen(s);

    s = sdsMakeRoomFor(s,len);
    if (s == NULL) return NULL;
    memcpy(s+curlen, t, len);
    sdssetlen(s, curlen+len);
    s[curlen+len] = '\0';
    return s;
}

/* Enlarge the free space at the end of the sds string more than needed,
 * This is useful to avoid repeated re-allocations when repeatedly appending to the sds. */
sds sdsMakeRoomFor(sds s, size_t addlen) {
    return _sdsMakeRoomFor(s, addlen, 1);
}

/* Enlarge the free space at the end of the sds string so that the caller
 * is sure that after calling this function can overwrite up to addlen
 * bytes after the end of the string, plus one more byte for nul term.
 * If there's already sufficient free space, this function returns without any
 * action, if there isn't sufficient free space, it'll allocate what's missing,
 * and possibly more:
 * When greedy is 1, enlarge more than needed, to avoid need for future reallocs
 * on incremental growth.
 * When greedy is 0, enlarge just enough so that there's free space for 'addlen'.
 *
 * Note: this does not change the *length* of the sds string as returned
 * by sdslen(), but only the free buffer space we have. */
sds _sdsMakeRoomFor(sds s, size_t addlen, int greedy) {
    void *sh, *newsh;
    size_t avail = sdsavail(s);
    size_t len, newlen, reqlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;
    size_t usable;

    /* Return ASAP if there is enough space left. */
    if (avail >= addlen) return s;

    len = sdslen(s);
    sh = (char*)s-sdsHdrSize(oldtype);
    reqlen = newlen = (len+addlen);
    assert(newlen > len);   /* Catch size_t overflow */
    if (greedy == 1) {
        if (newlen < SDS_MAX_PREALLOC)
            newlen *= 2;
        else
            newlen += SDS_MAX_PREALLOC;
    }

    type = sdsReqType(newlen);

    /* Don't use type 5: the user is appending to the string and type 5 is
     * not able to remember empty space, so sdsMakeRoomFor() must be called
     * at every appending operation. */
    if (type == SDS_TYPE_5) type = SDS_TYPE_8;

    hdrlen = sdsHdrSize(type);
    assert(hdrlen + newlen + 1 > reqlen);  /* Catch size_t overflow */
    if (oldtype==type) {
        newsh = s_realloc_usable(sh, hdrlen+newlen+1, &usable);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {
        /* Since the header size changes, need to move the string forward,
         * and can't use realloc */
        newsh = s_malloc_usable(hdrlen+newlen+1, &usable);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    usable = usable-hdrlen-1;
    if (usable > sdsTypeMaxSize(type))
        usable = sdsTypeMaxSize(type);
    sdssetalloc(s, usable);
    return s;
}
```

​	`sdscatsds`是暴露给上层的方法，其最终调用的是`sdscatlen`。由于其中可能涉及SDS的扩容，`sdscatlen`中调用`sdsMakeRoomFor`对带拼接的字符串s容量做检查，若无须扩容则直接返回s；若需要扩容，则返回扩容好的新字符串s。函数中的len、curlen等长度值是不含结束符的，而拼接时用`memcpy`将两个字符串拼接在一起，指定了相关长度，故该过程保证了二进制安全。最后需要加上结束符。

![sds字符串扩容过程.PNG](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6946b7eb7d904d8ea399fd9a8097589c~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

​	Redis的sds中有如下扩容策略：

1. 若sds中剩余空闲长度avail大于新增内容的长度addlen，直接在柔性数组buf末尾追加即可，无须扩容。

   ```c
   sds _sdsMakeRoomFor(sds s, size_t addlen, int greedy) {
     void *sh, *newsh;
     size_t avail = sdsavail(s);
     size_t len, newlen, reqlen;
     char type, oldtype = s[-1] & SDS_TYPE_MASK;
     int hdrlen;
     size_t usable;
   
     /* Return ASAP if there is enough space left. */
     if (avail >= addlen) return s;
     // ... 
   }
   ```

2. 若sds中剩余空闲长度avail小于或等于新增内容的长度addlen，则分情况讨论：新增后总长度len+addlen<1MB的，按新长度的2倍扩容；新增后总长度len+addlen>1MB的，按新长度加上1MB扩容。

   ```c
   {
     //...
     len = sdslen(s);
     sh = (char*)s-sdsHdrSize(oldtype);
     reqlen = newlen = (len+addlen);
     assert(newlen > len);   /* Catch size_t overflow */
     if (greedy == 1) {
       if (newlen < SDS_MAX_PREALLOC)
         newlen *= 2;
       else
         newlen += SDS_MAX_PREALLOC;
     }
     //...
   }
   ```

3. 最后根据新长度重新选取存储类型，并分配空间。此处若无须更改类型，通过realloc扩大柔性数组即可；否则需要重新开辟内存，并将原字符串的buf内容移动到新位置。

   ```c
   {
     //...
     type = sdsReqType(newlen);
   
     /* Don't use type 5: the user is appending to the string and type 5 is
        * not able to remember empty space, so sdsMakeRoomFor() must be called
        * at every appending operation. */
     if (type == SDS_TYPE_5) type = SDS_TYPE_8;
   
     hdrlen = sdsHdrSize(type);
     assert(hdrlen + newlen + 1 > reqlen);  /* Catch size_t overflow */
     if (oldtype==type) {
       newsh = s_realloc_usable(sh, hdrlen+newlen+1, &usable);
       if (newsh == NULL) return NULL;
       s = (char*)newsh+hdrlen;
     } else {
       /* Since the header size changes, need to move the string forward,
            * and can't use realloc */
       newsh = s_malloc_usable(hdrlen+newlen+1, &usable);
       if (newsh == NULL) return NULL;
       memcpy((char*)newsh+hdrlen, s, len+1);
       s_free(sh);
       s = (char*)newsh+hdrlen;
       s[-1] = type;
       sdssetlen(s, len);
     }
     usable = usable-hdrlen-1;
     if (usable > sdsTypeMaxSize(type))
       usable = sdsTypeMaxSize(type);
     sdssetalloc(s, usable);
     return s;
   }
   ```

### 2.2.4 其余API

注意点：

1. SDS暴露给上层的是指向柔性数组buf的指针
2. 读操作的复杂度多为O(1)，直接读取成员变量；涉及写的操作，则可能会触发扩容

| 函数名             | 说明                                        |
| ------------------ | ------------------------------------------- |
| sdsempty           | 创建一个空字符串，长度为0，内容为""         |
| sdsnew             | 根据给定的C字符串创建SDS                    |
| sdsdup             | 复制给定的SDS                               |
| sdsupdatelen       | 手动刷新SDS的相关统计值                     |
| sdsRemoveFreeSpace | 与sdsMakeRoomFor相反，对空闲过多的SDS做缩容 |
| sdsAllocSize       | 返回给定SDS当前占用的内存大小               |
| sdsgrowzero        | 将SDS扩容到指定长度，并用0填充新增内容      |
| sdscpylen          | 将C字符串复制到给定SDS中                    |
| sdstrim            | 从SDS两端清除所有给定的字符                 |
| sdscmp             | 比较两给定SDS的实际大小                     |
| sdssplitlen        | 按给定的分隔符对SDS进行切分                 |

## 2.3 本章小结

1. SDS如何兼容C语言字符串？如何保证二进制安全？

   SDS对象中的buf是一个柔性数组，上层调用时，SDS直接返回了buf。由于buf是直接指向内容的指针，故兼容C语言函数。而当真正读取内容时，SDS会通过len来限制读取长度，而非`\0`，保证了二进制安全。

2. sdshdr5的特殊之处是什么？

   sdshdr5只负责存储小于32字节的字符串。一般情况下，小字符串的存储更普遍，故Redis进一步压缩了sdshdr5的数据结构，将sdshdr5的类型和长度放入了同一个属性中，用flags的低3位存储类型，高5位存储长度。创建空字符串时，sdshdr5会被sdshdr8替代。

3. SDS是如何扩容的？

   SDS在涉及字符串修改处会调用sdsMakeroomFor函数进行检查，根据不同情况动态扩容，该操作对上层透明。

# 3. 跳跃表

## 3.1 简介

跳跃表思想：

每一层都是一个有序链表。在查找时优先从最高层开始向后查找，当到达某节点时，如果next节点值大于要查找的值或next指针指向NULL，则从当前节点下降一层继续向后查找。

![一般跳表结构.PNG](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/178bd614b0194cc6a40eb1da8bc0b17a~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

![redis跳表.PNG](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/50b9c4db8b9846dd93c2906a4208e3e9~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

- 跳跃表由很多层构成。
- 跳跃表有一个头（header）节点，头节点中有一个64层的结构，每层的结构包含指向本层的下个节点的指针，指向本层下个节点中间所跨越的节点个数为本层的跨度（span）。
- 除头节点外，层数最多的节点的层高为跳跃表的高度（level），图中跳跃表的高度为3。
- 每层都是一个有序链表，数据递增。
- 除header节点外，一个元素在上层有序链表中出现，则它一定会在下层有序链表中出现。
- 跳跃表每层最后一个节点指向NULL，表示本层有序链表的结束。
- 跳跃表拥有一个tail指针，指向跳跃表最后一个节点。
- 最底层的有序链表包含所有节点，最底层的节点个数为跳跃表的长度（length）（不包括头节点）。
- 每个节点包含一个后退指针，头节点和第一个节点指向NULL；其他节点指向最底层的前一个节点。

​	跳跃表与平衡树相比，实现方式更简单，只要熟悉有序链表，就可以轻松地掌握跳跃表。

## 3.2 跳跃表节点与结构

### 3.2.1 跳跃表节点

> server.h内（从`t_zset.c`找找找，发现原来在server.h里....）

跳跃表节点的`zskiplistNode`结构体。

```c
typedef struct zskiplistNode {
  sds ele;
  double score;
  struct zskiplistNode *backward;
  struct zskiplistLevel {
    struct zskiplistNode *forward;
    unsigned long span;
  } level[];
} zskiplistNode;
```

该结构体包含如下属性。

1. ele：用于存储字符串类型的数据。

2. score：用于存储排序的分值。

3. backward：后退指针，只能指向当前节点最底层的前一个节点，头节点和第一个节点——backward指向NULL，从后向前遍历跳跃表时使用。

4. level：为柔性数组。每个节点的数组长度不一样，在生成跳跃表节点时，随机生成一个1～64的值，<u>值越大出现的概率越低</u>。

​	level数组的每项包含以下两个元素。

+ forward：指向本层下一个节点，尾节点的forward指向NULL。

+ span：forward指向的节点与本节点之间的元素个数。span值越大，跳过的节点个数越多。

​	跳跃表是Redis有序集合的底层实现方式之一，所以每个节点的ele存储有序集合的成员member值，score存储成员score值。所有节点的分值是按从小到大的方式排序的，<u>当有序集合的成员分值相同时，节点会按member的字典序进行排序</u>。

### 3.2.2 跳跃表结构

> server.h内（从`t_zset.c`找找找，发现原来在server.h里....）

除了跳跃表节点外，还需要一个跳跃表结构来管理节点，Redis使用`zskiplist`结构体，定义如下

```c
typedef struct zskiplist {
  struct zskiplistNode *header, *tail;
  unsigned long length;
  int level;
} zskiplist;
```

1. header：指向跳跃表头节点。<u>头节点是跳跃表的一个特殊节点，它的level数组元素个数为64</u>。头节点在有序集合中不存储任何member和score值，ele值为NULL，score值为0；<u>也不计入跳跃表的总长度</u>。头节点在初始化时，64个元素的forward都指向NULL，span值都为0。

2. tail：指向跳跃表尾节点。
3. length：跳跃表长度，表示<u>除头节点之外</u>的节点总数。
4. level：跳跃表的高度。

​	通过跳跃表结构体的属性我们可以看到，程序可以在O(1)的时间复杂度下，快速获取到跳跃表的头节点、尾节点、长度和高度。

## 3.3 基本操作

### 3.3.1 创建跳跃表

1. 节点层高

   ​	节点层高的最小值为1，最大值是ZSKIPLIST_MAXLEVEL，Redis5中节点层高的值为64（2022年，我看最新的代码在2019年已经改成32）。

   ```c
   #define ZSKIPLIST_MAXLEVEL 64 // 2019年改成32，当前2022年
   // #define ZSKIPLIST_MAXLEVEL 32 /* Should be enough for 2^64 elements */
   ```

   ​	Redis通过`zslRandomLevel`函数随机生成一个1～64的值，作为新建节点的高度，值越大出现的概率越低。节点层高确定之后便不会再修改。生成随机层高的代码如下。

   ```c
   #define ZSKIPLIST_P 0.25 /* Skiplist P = 1/4 */
   int zslRandomLevel(void) {
     int level = 1;
     while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
       level += 1;
     return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
   }
   
   // 新版代码如下（这块代码在 t_zset.c ）：
   
   #define	RAND_MAX	0x7fffffff
   
   /* Returns a random level for the new skiplist node we are going to create.
    * The return value of this function is between 1 and ZSKIPLIST_MAXLEVEL
    * (both inclusive), with a powerlaw-alike distribution where higher
    * levels are less likely to be returned. */
   int zslRandomLevel(void) {
       static const int threshold = ZSKIPLIST_P*RAND_MAX;
       int level = 1;
       while (random() < threshold)
           level += 1;
       return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
   }
   ```

   ​	上述代码中，level的初始值为1，通过while循环，每次生成一个随机值，取这个值的低16位作为x，当x小于0.25倍的`0xFFFF`时，level的值加1；否则退出while循环。最终返回level和`ZSKIPLIST_MAXLEVEL`两者中的最小值。

   下面计算节点的期望层高。假设p=`ZSKIPLIST_P`：

   1）节点层高为1的概率为(1-p)。

   2）节点层高为2的概率为p(1-p)。

   3）节点层高为3的概率为p<sup>2</sup> (1-p)。

   4）……

   5）节点层高为n的概率为p<sup>n-1</sup> (1-p)。

   所以节点的期望层高为

   ![img](https://img2020.cnblogs.com/blog/1572587/202010/1572587-20201025200143016-1699117438.png)

   当p＝0.25时，跳跃表节点的期望层高为1/(1-0.25)≈1.33。

2. 创建跳跃表节点

   跳跃表的每个节点都是有序集合的一个元素，在创建跳跃表节点时，待创建节点的层高、分值、member等都已确定。对于跳跃表的每个节点，我们需要申请内存来存储，代码如下。

   ```c
   //t_zset.c
   /* Create a skiplist node with the specified number of levels.
    * The SDS string 'ele' is referenced by the node after the call. */
   zskiplistNode *zslCreateNode(int level, double score, sds ele) {
     zskiplistNode *zn =
       zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
     zn->score = score;
     zn->ele = ele;
     return zn;
   }
   ```

   ​	`zskiplistNode`结构体的最后一个元素为<u>柔性数组</u>，申请内存时需要指定柔性数组的大小，一个节点占用的内存大小为`zskiplistNode`的内存大小与level个`zskiplistLevel`的内存大小之和。

   ​	分配好空间之后，进行节点变量初始化。

3. 头节点

   头节点是一个特殊的节点，不存储有序集合的member信息。头节点是跳跃表中第一个插入的节点，其level数组的每项forward都为NULL，span值都为0。

   ```c
   //t_zset.c
   /* Create a new skiplist. */
   zskiplist *zslCreate(void) {
     int j;
     zskiplist *zsl;
   
     zsl = zmalloc(sizeof(*zsl));
     zsl->level = 1;
     zsl->length = 0;
     zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
     for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
       zsl->header->level[j].forward = NULL;
       zsl->header->level[j].span = 0;
     }
     zsl->header->backward = NULL;
     zsl->tail = NULL;
     return zsl;
   }
   ```

4. 创建跳跃表的步骤

   创建完头节点后，就可以创建跳跃表。创建跳跃表的步骤如下。

   1）创建跳跃表结构体对象zsl。

   2）将zsl的头节点指针指向新创建的头节点。

   3）跳跃表层高初始化为1，长度初始化为0，尾节点指向NULL

   ```c
   //t_zset.c
   /* Create a new skiplist. */
   zskiplist *zslCreate(void) {
     // ...
     zskiplist *zsl;
   
     zsl = zmalloc(sizeof(*zsl));
     zsl->level = 1;
     zsl->length = 0;
     zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
     // ...
     zsl->header->backward = NULL;
     zsl->tail = NULL;
     return zsl;
   }
   ```

### 3.3.2 插入节点

插入节点的步骤：

①查找要插入的位置；

②调整跳跃表高度；

③插入节点；

④调整backward

1. 查找要插入的位置

   插入节点时查找被更新节点的代码如下：

   + update[]：插入节点时，需要更新被插入节点每层的前一个节点。由于每层更新的节点不一样，所以将每层需要更新的节点记录在update[i]中。

   + rank[]：记录当前层从header节点到update[i]节点所经历的步长，在更新update[i]的span和设置新插入节点的span时用到。

   ```c
   // t_zset.c
   /* Insert a new node in the skiplist. Assumes the element does not already
    * exist (up to the caller to enforce that). The skiplist takes ownership
    * of the passed SDS string 'ele'. */
   zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
     // ... 
     x = zsl->header;
     for (i = zsl->level-1; i >= 0; i--) {
       /* store rank that is crossed to reach the insert position */
       rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
       while (x->level[i].forward &&
              (x->level[i].forward->score < score ||
               (x->level[i].forward->score == score &&
                sdscmp(x->level[i].forward->ele,ele) < 0)))
       {
         rank[i] += x->level[i].span;
         x = x->level[i].forward;
       }
       update[i] = x;
     }
     // ...
   }
   ```

   

   ![image.png](https://ucc.alicdn.com/pic/developer-ecology/3d9eefbc7b514694a0190e8315f2416d.png)

   如图所示的跳跃表，长度为3，高度为2。若要插入一个节点，分值为31，层高为3，查找节点（score=31，level=3）的插入位置，逻辑如下。

   1）第一次for循环，i=1。x为跳跃表的头节点。

   2）此时i的值与`zsl->level-1`相等，所以rank[1]的值为0。

   3）`header->level[1].forward`存在，并且`header->level[1].forward->score==1`小于要插入的score，所以可以进入while循环，rank[1]=1，x为第一个节点。

   4）第一个节点的第1层的forward指向NULL，所以不会再进入while循环。经过第一次for循环，rank[1]=1。x和update[1]都为第一个节点（score=1）。

   5）经过第二次for循环，i=0。x为跳跃表的第一个节点（score=1）。

   6）此时i的值与`zsl->level-1`不相等，所以rank[0]等于rank[1]的值，值为1。

   7）`x->level[0]->forward`存在，并且`x->level[0].foreard->score==21`小于要插入的score，所以可以进入while循环，rank[0]=2。x为第二个节点（score=21）。

   8）`x->level[0]->forward`存在，并且`x->level[0].foreard->score==41`大于要插入的score，所以不会再进入while，经过第二次for循环，rank[0]=2。x和update[0]都为第二个节点（score=21）。

   update和rank赋值后的跳跃表如图所示：

   ![image.png](https://ucc.alicdn.com/pic/developer-ecology/bbe9b29800bc44f8a356ca8b55802db3.png)

2. 调整跳跃表高度

   由上文可知，<u>插入节点的高度是随机</u>的，假设要插入节点的高度为3，大于跳跃表的高度2，所以我们需要调整跳跃表的高度。代码如下。

   ```c
   // t_zset.c
   zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
     // ...
     /* we assume the element is not already inside, since we allow duplicated
        * scores, reinserting the same element should never happen since the
        * caller of zslInsert() should test in the hash table if the element is
        * already inside or not. */
       level = zslRandomLevel();
       if (level > zsl->level) {
           for (i = zsl->level; i < level; i++) {
               rank[i] = 0;
               update[i] = zsl->header;
               update[i]->level[i].span = zsl->length;
           }
           zsl->level = level;
       }
     // ...
   }
   ```

   ​	此时，i的值为2，level的值为3，所以只能进入一次for循环。由于header的第0层到第1层的forward都已经指向了相应的节点，而新添加的节点的高度大于跳跃表的原高度，所以第2层只需要更新header节点即可。前面我们介绍过，rank是用来更新span的变量，其值是头节点到`update[i]`所经过的节点数，而此次修改的是头节点，所以`rank[2]`为0，`update[2]`一定为头节点。`update[2]->level[2].span`的值先赋值为跳跃表的总长度，后续在计算新插入节点level[2]的span时会用到此值。在更新完新插入节点level[2]的span之后会对`update[2]->level[2].span`的值进行重新计算赋值。

   ​	调整高度后的跳跃表如图所示。

   ![image.png](https://ucc.alicdn.com/pic/developer-ecology/48ca5d1b39ee46bf953154eade42e78f.png)

3. 插入节点

   当update和rank都赋值且节点已创建好后，便可以插入节点了。代码如下。

   ```c
   // t_zset.c
   zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
     // ...
     x = zslCreateNode(level,score,ele);
     for (i = 0; i < level; i++) {
       x->level[i].forward = update[i]->level[i].forward;
       update[i]->level[i].forward = x;
   
       /* update span covered by update[i] as x is inserted here */
       x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
       update[i]->level[i].span = (rank[0] - rank[i]) + 1;
     }
     // ...
   }
   ```

   level的值为3，所以可以执行三次for循环，插入过程如下。

   （1）第一次for循环

   1）x的level[0]的forward为update[0]的level[0]的forward节点，即`x->level[0].forward`为score=41的节点。

   2）update[0]的level[0]的下一个节点为新插入的节点。

   3）rank[0]-rank[0]＝0，`update[0]->level[0].span=1`，所以`x->level[0].span=1`。

   4）`update[0]->level[0].span=0+1=1`。

   插入节点并更新第0层后的跳跃表如图3-7所示。

   ![image.png](https://ucc.alicdn.com/pic/developer-ecology/2cd0308ce1d3432bafd2d15c9ee78e47.png)

   （2）第2次for循环

   1）x的level[1]的forward为update[1]的level[1]的forward节点，即`x->level[1].forward`为NULL。

   2）update[1]的level[1]的下一个节点为新插入的节点。

   3）rank[0]-rank[1]=1，`update[1]->level[1].span=2`，所以`x->level[1].span=1`。

   4）`update[1]->level[1].span=1+1=2`。

   插入节点并更新第1层后的跳跃表如图3-8所示。

   ![image.png](https://ucc.alicdn.com/pic/developer-ecology/144ca170746548bbbd99ede388b1cec3.png)

   （3）第3次for循环

   1）x的level[2]的forward为update[2]的level[2]的forward节点，即`x->level[2].forward`为NULL。

   2）update[2]的level[2]的下一个节点为新插入的节点。

   3）rank[0]-rank[2]=2，因为`update[2]->level[2].span=3`，所以`x->level[2].span=1`。

   4）`update[2]->level[2].span=2+1=3`。

   插入节点并更新第2层后的跳跃表如图3-9所示。

   ![image.png](https://ucc.alicdn.com/pic/developer-ecology/5482984c99c2498cafc90522fc23c8b2.png)

   新插入节点的高度大于原跳跃表高度，所以下面代码不会运行。

   但如果新插入节点的高度小于原跳跃表高度，则从level到`zsl->level-1`层的update[i]节点forward不会指向新插入的节点，所以不用更新update[i]的forward指针，只将这些level层的span加1即可。代码如下。

   ```c
   // t_zset.c
   zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
     // ...
     /* increment span for untouched levels */
     for (i = level; i < zsl->level; i++) {
       update[i]->level[i].span++;
     }
     //...
   }
   ```

4. 调整backward

   根据update的赋值过程，新插入节点的前一个节点一定是update[0]，由于每个节点的后退指针只有一个，与此节点的层数无关，所以当插入节点不是最后一个节点时，需要更新被插入节点的backward指向update[0]。如果新插入节点是最后一个节点，则需要更新跳跃表的尾节点为新插入节点。插入节点后，更新跳跃表的长度加1。代码如下。

   ```c
   // t_zset.c
   zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
     // ...
     x->backward = (update[0] == zsl->header) ? NULL : update[0];
     if (x->level[0].forward)
       x->level[0].forward->backward = x;
     else
       zsl->tail = x;
     zsl->length++;
     return x;
   }
   ```

   插入新节点后的跳跃表如图3-10所示。

   ![image.png](https://ucc.alicdn.com/pic/developer-ecology/88685c28309842c789c9247cbd737e8d.png)

### 3.3.3 删除节点

P68
