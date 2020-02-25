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
> [MySQL事务隔离性说明](https://blog.csdn.net/qq_42517220/article/details/88779932)
>
> [Mysql 5.7 的‘虚拟列’是做什么？](https://www.techug.com/post/mysql-5-7-generated-virtual-columns.html)
>
> [NULL与MySQL空字符串的区别](https://blog.csdn.net/lovemysea/article/details/82317043)
>
> [mysql partition 实战](https://blog.csdn.net/alex_xfboy/article/details/85245502)



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

​	然后就是NULL和空值的选择，速度上我这里极少量数据是认为空值快一点，而且可以避免一些麻烦的协商问题。但是NULL也有NULL的特性，比如COUNT(含有NULL的列)是不会记录含有NULL的行的。不过要是为了方便后期调整，个人觉得空值会更方便一点。