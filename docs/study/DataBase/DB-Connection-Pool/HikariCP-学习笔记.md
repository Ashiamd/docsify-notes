# HikariCP-学习笔记

> git项目地址: [brettwooldridge/HikariCP: 光 HikariCP・A solid, high-performance, JDBC connection pool at last. (github.com)](https://github.com/brettwooldridge/HikariCP)

# 1. HikariCP

## 1.1 HikariCP概述

> [HikariPool数据库连接池配置 - 简书 (jianshu.com)](https://www.jianshu.com/p/9d67519e0e1f)
>
> [HikariCP连接池 - 简书 (jianshu.com)](https://www.jianshu.com/p/15b846107a7c)
>
> [Bad Behavior: Handling Database Down · brettwooldridge/HikariCP Wiki (github.com)](https://github.com/brettwooldridge/HikariCP/wiki/Bad-Behavior:-Handling-Database-Down)

HikariCP是基于JDBC实现的小巧、高效、稳定的数据库连接池项目。

Hikari连接池目前公认是性能最高的数据库连接池，同时也是SpringBoot2.0以后默认使用的数据库连接池。

![img](https://upload-images.jianshu.io/upload_images/5658815-49a3279c680d5ad1.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

![img](https://upload-images.jianshu.io/upload_images/5658815-997cb1b8151acc9b.png?imageMogr2/auto-orient/strip|imageView2/2/w/779/format/webp)

> 2023-07-29拉取项目，除去测试类，Hikari项目内共33个java类

## 1.2 github重点内容摘录

HikariCP的github有很全面的说明，这里摘录部分我个人觉得值得注意的点：

### 1.2.1 HikariCP配置项

HikariCP和其他连接池实现相同的是，提供了诸如最小连接数、最大连接数等配置项。不同的是，Hikari提供的可配置项整体不多，主要追求极简主义。

下面列举一些常用配置项 (所有时间相关配置单位为ms毫秒)，其他配置项详情见 [HikariCP github](https://github.com/brettwooldridge/HikariCP)

| 配置项              | 说明                                                         | 默认值                  |
| ------------------- | ------------------------------------------------------------ | ----------------------- |
| dataSourceClassName | 数据源JDBC驱动类名，例如`com.mysql.cj.jdbc.Driver`           |                         |
| jdbcUrl             | 数据库连接url，例如`jdbc:mysql://127.0.0.1:3306/{databaseName}`。一些老的驱动实现要求必须填充JDBC驱动类名 |                         |
| username            | 数据库连接-用户名                                            |                         |
| password            | 数据库连接-密码                                              |                         |
| autoCommit          | 连接池提供的Conection是否是动提交                            | true                    |
| connectionTimeout   | 连接池中的连接超时时间，不能小于250ms                        | 30000 (30 seconds)      |
| idleTimeout         | 仅`maximumPoolSize`>`minimumIdle`时有效，当连接池中连接数超过`minimumIdle`时，连接池中多余的连接的允许闲置时间，超过则删除链接。设置为0时，多余的连接不会被删除 | 600000 (10 minutes)     |
| keepaliveTime       | idle空闲连接的存活时间，值必须小于`maxLifetime`，允许的最小值为 30000ms (30 seconds) | 0(禁用)                 |
| maxLifetime         | 连接池中连接的最长存活时间(工作中的连接不会过期)，值0表示无限存活(还需要看`idleTimeout`)，最小允许值30000ms(30秒)。 | 1800000(30分钟)         |
| connectionTestQuery | 如果驱动driver支持JDBC4则建议不要设置该属性。该查询用于在从连接池中获取连接前，验证连接仍存活，比如`select 1` |                         |
| minimumIdle         | 线程池中最小空闲连接数。当连接池中空闲连接数小于`minimumIdle`并且总连接数小于`maximumPoolSize`，HikariCP将尽可能添加额外的连接。为性能和应对峰值流量，建议不设置该值，而是允许HikariCP作为固定大小的连接池 | 与`maximumPoolSize`一致 |
| maximumPoolSize     | 线程池最大连接数，包括空闲连接和工作中的连接。当连接数到达`maximumPoolSize`并且没有空闲连接时，`getConnection()`方法调用将阻塞最多`connectionTimeout`毫秒 | 10                      |

### 1.2.2 Statement Cache

**许多连接池(包括Apache DBCP、Vibur、c3p0等)都提供了PreparedStatement缓存，但HikariCP没有**。原因如下：

1. **对池层而言，PreparedStatement Cache是Connection隔离的**，缓存牺牲部分资源。

   假设有250个经常执行的查询，而线程池有20个连接，那么相当于要求数据库保持5000个query execution plan，连接池必须缓存PreparedStatements和对应的对象图。

2. 大多数JDBC driver支持配置Statement cache (包括PostgreSQL、Oracle、Derby、MySQL、DB2等)，JDBC驱动程序更能利用好数据库特性，**其实现的缓存大多能跨连接共享**。

   如果使用JDBC驱动做缓存，250个常用语句在20个连接中共享缓存，只需要在内存中维护数据库的250个query execution plan。更智能的JDBC驱动甚至不会在内存中保留PreparedStatement对象，而是仅将新实例附加到现存的plan IDs上。

3. **在连接池层面使用statement cache是一种anti-pattern(反模式)的做法，与在驱动层做缓存相比，在池层做缓存对性能更具负面影响**。

### 1.2.3 Log Statement Text / Slow Query Logging

与Statement caching类似，大多数数据库通过数据库驱动的属性来支持statement logging。这包括Oracle、MySQL、Derby、MSSQL等。

### 1.2.4 HikariCP高性能的原因

> [Down the Rabbit Hole · brettwooldridge/HikariCP Wiki (github.com)](https://github.com/brettwooldridge/HikariCP/wiki/Down-the-Rabbit-Hole)
>
> HikariCP有很多细节优化，一些优化本身只有毫秒级的收益，但诸多优化在数百万次调用中均摊收益，使得整体性能高。

#### 1. JIT优化

+ 研究编译器的字节码输出和JIT汇编输出，尽可能限制key rountines在JIT的inline-threshold阈值内
+ 简化类的继承层次，shadowed member variables，消除强制转化(eliminated casts)

#### 2. 细节优化

##### ArrayList

一个重要的性能优化即使用`FastList`替代`ArrayList`，用于`ProxyConnection`维护`Statement`实例。

+ `Statement`关闭时，它必须从集合中删除
+ `Connection`关闭时，需要遍历其维护的`Statement`集合，关闭所有打开(open)的`Statement`实例，最后清空(clear)集合

上诉流程中，如果使用`ArrayList`来维护`Statement`集合，那么需要频繁调用的方法有如下2个：

1. `public E get(int index)`

   ```java
   public E get(int index) {
     // 1. 检查索引是否越界
     Objects.checkIndex(index, size);
     // 2. 索引没有越界，则返回ArrayList维护的数组对应index的元素
     return elementData(index);
   }
   ```

2. `public boolean remove(Object o)`

   ```java
   public boolean remove(Object o) {
     final Object[] es = elementData;
     final int size = this.size;
     int i = 0;
     found: {
       // 1. 从头到尾遍历，找到需要删除的元素的索引值，找不到则提前返回false
       if (o == null) {
         for (; i < size; i++)
           if (es[i] == null)
             break found;
       } else {
         for (; i < size; i++)
           if (o.equals(es[i]))
             break found;
       }
       return false;
     }
     // 2. 删除对应索引值的元素
     fastRemove(es, i);
     return true;
   }
   
   private void fastRemove(Object[] es, int i) {
     modCount++;
     final int newSize;
     if ((newSize = size - 1) > i)
       System.arraycopy(es, i + 1, es, i, newSize - i);
     // 注意, 并不会直接回收原本ArrayList申请的数组空间, 这里只是把最后的位置填充为null值
     es[size = newSize] = null;
   }
   ```

ArrayList本身对于以上两个方法的实现是没有问题的，但ArrayList是无业务场景的通用实现。对于HikariCP数据库连接池而言，这里存在优化空间：

1. HikariCP能保证内部集合维护`Statement`的正确性，所以不需要在`public E get(int index)`方法中额外进行"索引越界检查"
2. JDBC编程的常见模式中，`Statement`往往在执行完查询后就直接关闭，且常常在List末尾新增打开的`Statement`。这种场景下，`public boolean remove(Object o)`原本的从头到尾遍历，显然不如"从尾到头遍历"来得合适。

根据以上2点优化思路，HikariCP继承`ArrayList`实现了`FastList`，对`public E get(int index)`和`public boolean remove(Object o)`重写(Override)的实现如下：

1. `public T get(int index)`

   ```java
   public T get(int index)
   {
     // 直接返回索引对应的元素，省去ArrayList的越界检查
     return elementData[index];
   }
   ```

2. `public boolean remove(Object element)`

   ```java
   public boolean remove(Object element)
   {
     // 和ArrayList遍历顺序相反，改成从尾向头遍历，更适合JDBC业务场景
     for (int index = size - 1; index >= 0; index--) {
       if (element == elementData[index]) {
         final int numMoved = size - index - 1;
         if (numMoved > 0) {
           System.arraycopy(elementData, index + 1, elementData, index, numMoved);
         }
         elementData[--size] = null;
         return true;
       }
     }
   
     return false;
   }
   ```



##### ConcurrentBag

> HikariCP在`HikariPool`类实现中使用`ConcurrentBag`实例维护线程池对象`PoolEntry`

HikariCP借鉴C# .NET的`ConcurrentBag`，实现了用于存储自定义对象的无锁集合类`ConcurrentBag`。总体而言，`ConcurrentBag`具有以下特性：

+ A lock-free design
+ ThreadLocal caching
+ Queue-stealing
+ Direct hand-off optimizations

总总特性使得`ConcurrentBag`能够做到高并发、低延迟、低频发生伪共享问题(minimized occurrences of [false-sharing](http://en.wikipedia.org/wiki/False_sharing))。



##### Invocation: `invokevirtual` vs `invokestatic`

为给`Connection`, `Statement`, `ResultSet`提供代理对象，HikariCP原本使用单例工厂维护静态成员变量`PROXY_FACTORY`来表示`ConnectionProxy`，于是就有诸多类似如下的方法实现：

```java
public final PreparedStatement prepareStatement(String sql, String[] columnNames) throws SQLException
{
  return PROXY_FACTORY.getProxyPreparedStatement(this, delegate.prepareStatement(sql, columnNames));
}
```

使用原本这种单例工厂实现，对应的字节码如下：

```java
    public final java.sql.PreparedStatement prepareStatement(java.lang.String, java.lang.String[]) throws java.sql.SQLException;
    flags: ACC_PRIVATE, ACC_FINAL
    Code:
      stack=5, locals=3, args_size=3
         0: getstatic     #59                 // Field PROXY_FACTORY:Lcom/zaxxer/hikari/proxy/ProxyFactory;
         3: aload_0
         4: aload_0
         5: getfield      #3                  // Field delegate:Ljava/sql/Connection;
         8: aload_1
         9: aload_2
        10: invokeinterface #74,  3           // InterfaceMethod java/sql/Connection.prepareStatement:(Ljava/lang/String;[Ljava/lang/String;)Ljava/sql/PreparedStatement;
        15: invokevirtual #69                 // Method com/zaxxer/hikari/proxy/ProxyFactory.getProxyPreparedStatement:(Lcom/zaxxer/hikari/proxy/ConnectionProxy;Ljava/sql/PreparedStatement;)Ljava/sql/PreparedStatement;
        18: return
```

可以看到`0: getstatic     #59`将静态字段`PROXY_FACTORY`加载到栈中(operand stack)，后续`15: invokevirtual #69`调用了ProxyFactory实例上的`getProxyPreparedStatement()`方法。

**HikariCP通过Javassist生成final类`JavassistProxyFactory`消除了原本的单例工厂**，Java代码变为如下：

```java
public final PreparedStatement prepareStatement(String sql, String[] columnNames) throws SQLException
{
  return ProxyFactory.getProxyPreparedStatement(this, delegate.prepareStatement(sql, columnNames));
}
```

`getProxyPreparedStatement()`是`ProxyFactory`类中的`static`静态方法，对应的字节码如下：

```java
    private final java.sql.PreparedStatement prepareStatement(java.lang.String, java.lang.String[]) throws java.sql.SQLException;
    flags: ACC_PRIVATE, ACC_FINAL
    Code:
      stack=4, locals=3, args_size=3
         0: aload_0
         1: aload_0
         2: getfield      #3                  // Field delegate:Ljava/sql/Connection;
         5: aload_1
         6: aload_2
         7: invokeinterface #72,  3           // InterfaceMethod java/sql/Connection.prepareStatement:(Ljava/lang/String;[Ljava/lang/String;)Ljava/sql/PreparedStatement;
        12: invokestatic  #67                 // Method com/zaxxer/hikari/proxy/ProxyFactory.getProxyPreparedStatement:(Lcom/zaxxer/hikari/proxy/ConnectionProxy;Ljava/sql/PreparedStatement;)Ljava/sql/PreparedStatement;
        15: areturn
```

这里有3个要点需要注意：

1. 消除了`getstatic`调用
2. 对`getProxyPreparedStatement()`方法的调用，由`invokevirtual`变为了`invokestatic`，后者更容易被JVM优化
3. operand stack大小由5减小到了4，因为原本`invokevirtual`调用的是实例方法，operand stack的下标0位置存放的是实例的`this`引用对象，当`invokevirtual`调用实例方法`getProxyPreparedStatement()`时，会从operand stack弹出该`this`引用对象。改成静态方法调用，则省去了`this`隐式传递和栈弹出过程。

<u>总之，这种优化使得实际字节码执行时省去了静态成员字段的访问(access)，推送(push，即this指针传递)，栈弹出(pop，从operand stack弹出this对象)的步骤，这保证了调用点(callsite)不会发生变化，更有益于JIT完成调用优化。</u>

> [Lambda、MethodHandle、CallSite调用简单性能测试与调优 - 简书 (jianshu.com)](https://www.jianshu.com/p/8502643beffd) => 简洁明了，推荐阅读
>
> [反射调用简单性能测试与调优 - 简书 (jianshu.com)](https://www.jianshu.com/p/21d700f80654)
>
> [Java虚函数&&内联优化 - 简书 (jianshu.com)](https://www.jianshu.com/p/baaff02a8b5f)
>
> [Java中的方法内联_pedro7k的博客-CSDN博客](https://blog.csdn.net/pedro7k/article/details/122729561) => 提到 `invokevirtual`本身可能存在方法调用多态问题，所以JIT难以完成内联优化，这也可解释HikariCP将`invokevirtual`优化成`invokestatic`正是为了更好利用JIT优化。
>
> [Java五大invoke指令_invokespecial_醒过来摸鱼的博客-CSDN博客](https://blog.csdn.net/m0_66201040/article/details/122656927)
>
> [这波性能优化，太炸裂了！ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/397845438) => 提到了`invokevirtual`优化成`invokestatic`的一些原因，虽然废话有点多，整体文章还是可以的
>
> [Java虚函数&&内联优化 - 简书 (jianshu.com)](https://www.jianshu.com/p/baaff02a8b5f)



##### Scheduler quanta





##### CPU Cache-line Invalidation





# 2. JDBC

## 2.1 JDBC查询流程回顾

回顾一下如何通过JDBC进行数据库查询：

1. 创建`DataSource`对象
2. 获取`Conneciton`数据库连接
3. 创建`PreparedStatement` (或`Statement`)包装查询SQL和参数
4. 执行查询，获取`ResultSet`查询结果
5. 释放资源

```java
public static void main(String[] args) {
  // 1. 创建 DataSource 对象
  HikariDataSource hikariDataSource = new HikariDataSource();
  // 数据库连接驱动 (这里连接 MySQL)
  hikariDataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
  // mysql连接url
  hikariDataSource.setJdbcUrl("jdbc:mysql://127.0.0.1:3306/test");
  // 账号
  hikariDataSource.setUsername("root");
  // 密码
  hikariDataSource.setPassword("example");

  try{
    // 2. 获取链接
    Connection connection = hikariDataSource.getConnection();
    // 3. 创建查询SQL
    PreparedStatement preparedStatement = connection.prepareStatement("select name, gender, age from person limit 1");
    // 4. 执行查询, 获取查询结果
    ResultSet resultSet = preparedStatement.executeQuery();
    while(resultSet.next()) {
      String name = resultSet.getString(1);
      short gender = resultSet.getShort(2);
      short age = resultSet.getShort(3);
      System.out.println("name: " + name + ", gender: " + gender + ", age: " + age);
    }
    // 5. 释放资源
    resultSet.close();
    preparedStatement.close();
    connection.close();
  } catch (SQLException e) {
    e.printStackTrace();
  }
}
```

## 2.2 DataSource

`DataSource`接口定义中，最重要的即`getConnection()`方法，用于创建和数据存储之间的连接。

DataSource通常由数据库连接驱动方提供实现，主要由以下3种类型的实现：

1. 基础实现：提供Connection连接对象
2. **连接池实现**：维护连接池，从连接池中对外提供Connection连接对象。该实现依赖中间层connection pooling manager。
3. **分布式事务实现**：提供用于分布式事务的Connection连接对象，通常以数据库连接池的形式维护。该实现依赖中间层trasaction manager，通常还依赖connection pooling manager。

```java
public interface DataSource  extends CommonDataSource, Wrapper {
  Connection getConnection() throws SQLException;
  // ...
}
```

# 3. HikariCP代码阅读

## 3.1 HikariDataSource

在传统JDBC查询流程中，数据库连接池由`DataSource`提供，所以我们着重关注HikariCP对`DataSource`的实现，即`HikariDataSource`类。

`HikariDataSource`在`DataSource`实现中，个人理解属于第二种，即连接池实现，主要在"连接池"方向精进。
