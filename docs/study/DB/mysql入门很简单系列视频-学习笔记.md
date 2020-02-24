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

### 1. MySQL的NULL值、空值查询；模糊查询的like、=、%、_

> [mysql 用法 Explain](https://blog.csdn.net/lvhaizhen/article/details/90763799)
>
> [MySQL_执行计划详细说明](https://www.cnblogs.com/xinysu/p/7860609.html)

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

​	可以看出 `is NOT NULL`没有用到索引，直接全表查询了。

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

+ `=''`

```sql
EXPLAIN SELECT * FROM	room WHERE `profile` = '';
```

| id   | select_type | table | partitions | type | possible_keys | key      | key_len | ref   | rows  | filtered | Extra                 |
| ---- | ----------- | ----- | ---------- | ---- | ------------- | -------- | ------- | ----- | ----- | -------- | --------------------- |
| 1    | SIMPLE      | room  | (Null)     | ref  | password      | password | 1023    | const | 15176 | 100.00   | Using index condition |