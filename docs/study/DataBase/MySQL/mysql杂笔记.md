# 1. MySQL杂笔记

> [ACID--wiki](https://en.wikipedia.org/wiki/ACID)

## 1. ACID概述

### Consistency

​	[Consistency](https://en.wikipedia.org/wiki/Consistency_(database_systems)) ensures that a transaction can only bring the database from one valid state to another, maintaining database [invariants](https://en.wikipedia.org/wiki/Invariant_(computer_science)): any data written to the database must be valid according to all defined rules, including [constraints](https://en.wikipedia.org/wiki/Integrity_constraints), [cascades](https://en.wikipedia.org/wiki/Cascading_rollback), [triggers](https://en.wikipedia.org/wiki/Database_trigger), and any combination thereof. This prevents database corruption by an illegal transaction, but does not guarantee that a transaction is *correct*. [Referential integrity](https://en.wikipedia.org/wiki/Referential_integrity) guarantees the [primary key](https://en.wikipedia.org/wiki/Unique_key) – [foreign key](https://en.wikipedia.org/wiki/Foreign_key) relationship. [[6\]](https://en.wikipedia.org/wiki/ACID#cite_note-Date2012-6)

## 2. MySQL优化

> [【MySQL优化】——看懂explain](https://blog.csdn.net/jiadajing267/article/details/81269067)
>
> [MySQL 中的 information_schema 数据库](https://blog.csdn.net/kikajack/article/details/80065753)

## 3. MySQL日志系统

> [MySQL日志系统：redo log、binlog、undo log 区别与作用](https://blog.csdn.net/u010002184/article/details/88526708)

## 999. 杂项

> [MySQL 到底能不能放到 Docker 里跑？](https://zhuanlan.zhihu.com/p/47172593)
>
> 
>
> [SQL之in和exit区别篇](https://blog.csdn.net/qq_36561697/article/details/80713824)	<=	回顾一下
>
> [in与exists的取舍](https://blog.csdn.net/dreamwbt/article/details/53363497)	<=	回顾一下
>
> 一般而言，外循环的数量级小的，速度更快，因为外层复杂度N，但是内层走索引的话就能缩小到logM
>
> A join B也是笛卡尔积，最后保留指定字段相同的结果而已（A内循环，B外循环）
>
> A in B，先计算B，然后笛卡尔积，（A内循环，B外循环）
>
> A exist B，先计算A，然后笛卡尔积（B内循环，A外循环）
>
> not in内外表都不会用到索引，而not exists能用到索引，所以后者任何情况都比前者好
>
> 
>
> 分布式事务
>
> [终于有人把“TCC分布式事务”实现原理讲明白了！ - 阿里-马云的学习笔记 - 博客园 (cnblogs.com)](https://www.cnblogs.com/alimayun/p/12057142.html)
>
> 
>
> 锁
>
> [MySQL死锁日志的查看和分析 - ianCloud - 博客园 (cnblogs.com)](https://www.cnblogs.com/iancloud/p/18021606)
>
> [MySQL锁--03---意向锁（Intention Locks）、间隙锁（Gap Locks）、临键锁(Next-Key Locks)_mysql 意向锁-CSDN博客](https://blog.csdn.net/weixin_48052161/article/details/121985667)
>
> [【原创】惊！史上最全的select加锁分析(Mysql) - 孤独烟 - 博客园 (cnblogs.com)](https://www.cnblogs.com/rjzheng/p/9950951.html) 超高质量
>
> [【原创】互联网项目中mysql应该选什么事务隔离级别 - 孤独烟 - 博客园 (cnblogs.com)](https://www.cnblogs.com/rjzheng/p/10510174.html) 超高质量
>
> [MySQL介于普通读和加锁读之间的读取方式：semi-consistent read - 掘金 (juejin.cn)](https://juejin.cn/post/6844904022499917838) RC下的半一致性读 说明，高质量

