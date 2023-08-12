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

>  然而，实际基准测试中，其他连接池实现同样是JIT内联优化的受益者，光内联JIT优化这点而言，HikariCP并不见得优势更大。
>
> `¯\_(ツ)_/¯` Yeah, but still...
>
> In our benchmark, we are obviously running against a stub JDBC driver implementation, so the JIT is doing a lot of inlining.  However, the same inlining at the stub-level is occurring for other pools in the benchmark.  So, no inherent advantage to us.
>
> But inlining is certainly a big part of the equation even when real drivers are in use, which brings us to another topic...

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

> [Operating Systems: CPU Scheduling (uic.edu)](https://www.cs.uic.edu/~jbell/CourseNotes/OperatingSystems/5_CPU_Scheduling.html)

CPU分时调度：显然运行上百个线程时，除非有对应数量的CPU核数，否则只有一部分线程能够运行。操作系统在N个CPU核之间切换调度线程时，每个线程有一小段时间可运行，这段时间成为quantum。

在运行大量线程时，需要尽可能利用好当前线程的时间片，尽量避免锁等机制使线程放弃执行，因为下次调度到该线程时，可能需要"较长"的时间。

> 前面提到过`ConcurrentBag`使用无锁设计，意味着可尽量避免出让时间片，线程可以尽量工作，而不是等待调度浪费掉一部分时间。



##### CPU Cache-line Invalidation

> CPU缓存行失效，和MESI缓存一致性协议有关
>
> [MESI协议_百度百科 (baidu.com)](https://baike.baidu.com/item/MESI协议/22742331?fr=ge_ala)
>
> [CPU缓存一致性协议MESI详解-电子发烧友网 (elecfans.com)](https://www.elecfans.com/d/1833080.html)

CPU缓存行失效问题。即当前线程CPU资源被其他线程抢占后，下次轮到当前线程执行时，原本使用的数据可能已经不在CPU的L1 cache或L2 cache。并且你可能无法控制当前线程下一次运行时是使用哪个CPU核。



# 2. JDBC

## 2.1 JDBC查询流程回顾

> [HikariCP Connection Pooling Example - Examples Java Code Geeks - 2023](https://examples.javacodegeeks.com/java-development/enterprise-java/hikaricp/hikaricp-connection-pooling-example/)

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

> `HikariDataSource`在`DataSource`实现中，个人理解属于第二种，即连接池实现，主要在"连接池"方向精进。

# 3. HikariCP代码阅读

> 下面看代码，会把我个人觉得不算重点的代码剔除，**重点只关注和JDBC流程相关的代码实现**（不关注监控/健康检查等代码实现）
>
> 列一些看着不是很相关，但是有些知识点涉及的文章/视频，仅个人学习使用
>
> [服务监控 | 彻底搞懂Dropwizard Metrics一篇就够了 - 时钟在说话 - 博客园 (cnblogs.com)](https://www.cnblogs.com/mindforward/p/15792132.html)
>
> [红队攻击手特训营-JNDI注入漏洞挖掘_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1Ne4y1o7ch/?spm_id_from=333.337.search-card.all.click&vd_source=ba4d176271299cb334816d3c4cbc885f)

## 3.1 HikariDataSource

### 代码阅读

在传统JDBC查询流程中，数据库连接池需要通过`DataSource`访问，所以我们着重关注HikariCP对`DataSource`的实现，即`HikariDataSource`类。

从前面JDBC查询流程，和`DataSource`相关的方法调用主要有：

+ `HikariDataSource()`/`HikariDataSource(HikariConfig configuration)` => 构造函数（业务使用时，一般同一个数据库对应的DataSource以单例维护）
+ `getConnection()` => 实际调用`HikariPool`连接池实例的`getConnection()`方法
+ `close()` => 内部需要关闭`HikariPool`连接池对象

*可以看出来`HikariDataSource`自身和JDBC查询流程关联不多，主要工作还是由`HikariPool`完成。*

```java
/**
 * The HikariCP pooled DataSource.
 *
 * @author Brett Wooldridge
 */
public class HikariDataSource extends HikariConfig implements DataSource, Closeable {

  // 数据库连接DataSource是否已经整体关闭
  private final AtomicBoolean isShutdown = new AtomicBoolean();
  // 连接池对象(有参构造函数使用, fast在于少去一些额外判断)
  private final HikariPool fastPathPool;
  // 连接池对象(有参构造函数初始化,无参构造函数在第一次getConnection()时初始化)
  private volatile HikariPool pool;
  // ...

  // 官方建议使用有参构造函数。
  // 当使用无参构造函数构造连接池，第一次调用getConnection()会有懒加载的配置初始化校验，使得第一次连接响应更慢
  public HikariDataSource()
  {
    // 连接池配置-填充默认参数
    super();
    fastPathPool = null;
  }

  // 有参构造函数，提前做好配置参数校验
  public HikariDataSource(HikariConfig configuration)
  {
    // 1. 连接池配置参数校验
    configuration.validate();
    // 2. configuration参数对应的配置copy到当前HikariDataSource实例
    configuration.copyState(this);
    // 3. 构造连接池对象 (先不关注HikariPool内部细节)
    pool = fastPathPool = new HikariPool(this);
  }
	
  // 获取数据库 连接对象
  public Connection getConnection() throws SQLException
  {
    // 1. isShutdown = true, 则不再提供数据库连接对象
    if (isClosed()) {
      throw new SQLException("HikariDataSource " + this + " has been closed.");
    }
		
    // 2. 如果使用有参构造函数，则这里不为null，直接返回 连接池对象
    if (fastPathPool != null) {
      return fastPathPool.getConnection();
    }

    // 3. 如果使用无参构造函数，则连接池对象pool / fastPathPool 未初始化，需要临时进行配置校验和pool池对象初始化
    // See http://en.wikipedia.org/wiki/Double-checked_locking#Usage_in_Java
    HikariPool result = pool;
    if (result == null) {
      synchronized (this) {
        result = pool;
        if (result == null) {
          // 配置参数校验(非关注重点)
          validate();
          LOGGER.info("{} - Started.", getPoolName());
          // 保证 pool 单例 (HikariPool内细节暂不关注)
          // 注意这里没有再初始化 fastPathPool，换言之无参构造方法后续都直接靠pool对象获取连接
          pool = result = new HikariPool(this);
        }
      }
    }
		// 通过连接池 获取 连接对象 (后续再关注HikariPool的getConnection()方法)
    return result.getConnection();
  }

  // 标记DataSource关闭(下次getConection()直接抛出异常), 关闭连接池 HikariPool
  public void close()
  {
    // 第一个执行的，返回旧值为false; 重复执行则直接return
    if (isShutdown.getAndSet(true)) {
      return;
    }
		// 释放连接池资源
    HikariPool p = pool;
    if (p != null) {
      try {
        // 关闭连接池 (关闭所有空闲or工作中的连接) , fastPathPool 只有在有参构造函数下有值，有值时和pool指向同一个实例
        p.shutdown();
      }
      catch (InterruptedException e) {
        LOGGER.warn("Interrupted during closing", e);
        Thread.currentThread().interrupt();
      }
    }
  }
}
```

### 代码小结

1. `HikariDataSource()`/`HikariDataSource(HikariConfig configuration)`

   构造函数主要工作即初始化连接池配置，官方推荐使用有参构造函数。

   + 无参构造函数：使用父类`HikariConfig`默认配置项，`fastPathPool`为null，在第一次`getConnection()`时加锁进行配置校验和初始化`pool`

   + 有参构造函数：官方推荐使用，使用参数传入的配置项，预先做配置项校验，以及对`fastPathPool`和`pool`初始化

2. `getConnection()`

   `fastPathPool`不为null则直接从池中获取连接对象`Conneciton`，否则从`pool`中获取连接。*若使用无参构造函数，则第一次调用`getConnection()`时才对配置项进行参数校验，以及初始化`pool`。*

3. `close()` 

   调用`pool`的`shutdown()`方法关闭线程池。*这里`fastPathPool`和`pool`指向的是同一个实例，所以不需要额外再调用`fastPathPool`的`shutdown()`方法。*

## 3.2 HikariPool

> [Semaphore在Java并发编程中的使用和比较技术 (baidu.com)](https://baijiahao.baidu.com/s?id=1767470463455689974&wfr=spider&for=pc)
>
> - Semaphore允许多个线程同时访问资源，而Lock一次只允许一个线程访问资源。
> - Semaphore是基于计数的机制，可以控制同时访问的线程数量，而Lock只是简单的互斥锁。 根据具体场景，选择Semaphore还是Lock取决于对资源的访问控制需求。
> - Semaphore主要用于控制对资源的访问，限制并发线程的数量。
> - Condition主要用于线程之间的协调，可以通过`await()`和`signal()`等方法实现线程的等待和唤醒。
>
> Semaphore的适用场景： Semaphore在以下场景中特别有用：
>
> - **控制对有限资源的并发访问，如数据库连接池、线程池等**。
> - 限制同时执行某个操作的线程数量，如限流和限制并发请求等。
> - 在生产者-消费者模式中平衡生产者和消费者之间的速度差异。

### 代码阅读

具体提供连接池的类，即`HikariPool`，其构造函数有且仅有一个有参构造函数。这里先只关注被`HikariDataSource`调用的部分代码。

+ `HikariPool(final HikariConfig config)` => 构造函数，根据默认/指定配置项初始化连接池
+ `getConnection()` => 从connectionBag中获取池连接对象PoolEntry
+ `shutdown()` => 关闭连接池，释放相关资源

```java
public class HikariPool extends PoolBase implements HikariPoolMXBean, IBagStateListener{
  // 维护池对象的并发集合(作用上类似LinkedBlockingQueue, 但这里贴合池场景实现, 性能更好), 一个PoolEntry内维护一个Connection
  private final ConcurrentBag<PoolEntry> connectionBag;
  // 当前池中的总连接数(Connection数)
  private final AtomicInteger totalConnections;
  // 锁, 用来并发时互斥地 suspend/resume pool (内部通过Semaphore信号量实现), 默认使用FAUX_LOCK, 所有方法为空实现
  private final SuspendResumeLock suspendResumeLock;
  // 负责创建Connection的 ThreadPoolExecutor
  private final ThreadPoolExecutor addConnectionExecutor;
  // 负责关闭Connection的 ThreadPoolExecutor
  private final ThreadPoolExecutor closeConnectionExecutor;
  // 负责定时清理空闲连接的ScheduledThreadPoolExecutor
  private final ScheduledThreadPoolExecutor houseKeepingExecutorService;
  // (非重点) 负责检查Connection泄漏的RunnablO(使用 houseKeepingExecutorService 运行)
  private final ProxyLeakTask leakTask;

  public HikariPool(final HikariConfig config)
  {
    // 1. 进行配置信息(事务配置,连接通信配置,池基础信息配置)初始化赋值 + 设置DataSource信息(jdbcUrl连接信息, 数据库连接驱动信息)
    super(config);

    // 2. 初始化池操作涉及的对象信息
    this.connectionBag = new ConcurrentBag<>(this);
    this.totalConnections = new AtomicInteger();
    this.suspendResumeLock = config.isAllowPoolSuspension() ? new SuspendResumeLock() : SuspendResumeLock.FAUX_LOCK;

    // 3. 启动一个Connection但又马上close(), 目的是检查pool的配置是否正常, 是否后续能创建有效的Connection
    checkFailFast();

    // 4. 初始化负责新建/关闭Connection的ThreadPoolExecutor
    ThreadFactory threadFactory = config.getThreadFactory();
    this.addConnectionExecutor = createThreadPoolExecutor(config.getMaximumPoolSize(), poolName + " connection adder", threadFactory, new ThreadPoolExecutor.DiscardPolicy());
    this.closeConnectionExecutor = createThreadPoolExecutor(config.getMaximumPoolSize(), poolName + " connection closer", threadFactory, new ThreadPoolExecutor.CallerRunsPolicy());

    // 5. 初始化负责定时清理空闲连接的ScheduledThreadPoolExecutor
    if (config.getScheduledExecutorService() == null) {
      threadFactory = threadFactory != null ? threadFactory : new DefaultThreadFactory(poolName + " housekeeper", true);
      this.houseKeepingExecutorService = new ScheduledThreadPoolExecutor(1, threadFactory, new ThreadPoolExecutor.DiscardPolicy());
      this.houseKeepingExecutorService.setExecuteExistingDelayedTasksAfterShutdownPolicy(false);
      this.houseKeepingExecutorService.setRemoveOnCancelPolicy(true);
    }
    else {
      this.houseKeepingExecutorService = config.getScheduledExecutorService();
    }

    // 6. 启动定时任务, 30ms执行一次(找出STATE_NOT_IN_USE状态的PoolEntry集合,若数量超过设置的最小空闲连接数minIdle,按照最后一次访问时间对空闲PoolEntry升序排序, 从头遍历connectionBag尝试删除"可回收的数量"个poolEntry)
    // “可回收的数量” = 当前空闲连接数 - 最小空闲连接数(配置项), >0 则进行后续的多余空闲连接回收
    // 具体回收方式即改变PoolEntry的状态从STATE_NOT_IN_USE到STATE_RESERVED(这样后续可被从connectionBag中remove())
    // 异步关闭 PoolEntry 对应的 Connection连接(假设网络不畅, 最长通信超时为15s, 为了尽可能保证和数据库通信完成真正的连接关闭)
    // 最后内部执行fillPool(),尝试将Pool中连接数保持到最小空闲连接数(配置项), 如果本来已经足额则不执行任何逻辑
    this.houseKeepingExecutorService.scheduleWithFixedDelay(new HouseKeeper(), 0L, HOUSEKEEPING_PERIOD_MS, MILLISECONDS);

    // 7. (非重点) 初始化用于检测Connection泄漏的Runnable
    this.leakTask = new ProxyLeakTask(config.getLeakDetectionThreshold(), houseKeepingExecutorService);
  }

  public final Connection getConnection() throws SQLException
  {
    return getConnection(connectionTimeout);
  }

  /**
    * Get a connection from the pool, or timeout after the specified number of milliseconds.
    *
    * @param hardTimeout the maximum time to wait for a connection from the pool
    * @return a java.sql.Connection instance
    * @throws SQLException thrown if a timeout occurs trying to obtain a connection
    */
  public final Connection getConnection(final long hardTimeout) throws SQLException
  {
    // 1. 尝试加锁 (HikariConfig默认配置中isAllowPoolSuspension为false, 即acquire()方法为空实现, 避免性能损耗)
    // 官方本身并不建议设置isAllowPoolSuspension为true
    suspendResumeLock.acquire();
    final long startTime = clockSource.currentTime();

    try {
      // 连接超时配置时间(ms)
      long timeout = hardTimeout;
      do {
        // 2. 在连接超时之前尝试从 connectionBag(存储PoolEntry的对象) 中获取一个PoolEntry池对象(对应一个Connection)
        // 内部将 PoolEntry的状态从 STATE_NOT_IN_USE 转为 STATE_IN_USE
        final PoolEntry poolEntry = connectionBag.borrow(timeout, MILLISECONDS);
        if (poolEntry == null) {
          break; // We timed out... break and throw exception
        }

        final long now = clockSource.currentTime();
        // 3. 如果 (获取到的PoolEntry已被标记evicted) or (上一次访问时间超过ALIVE_BYPASS_WINDOW_MS) or (连接已关闭) 则关闭连接并移除该PoolEntry, 然后下次do-while循环继续尝试从池中获取PoolEntry
      	// 否则使用connectionBag中获取到的PoolEntry建立新的Connection并返回
        if (poolEntry.isMarkedEvicted() || (clockSource.elapsedMillis(poolEntry.lastAccessed, now) > ALIVE_BYPASS_WINDOW_MS && !isConnectionAlive(poolEntry.connection))) {
          closeConnection(poolEntry, "(connection is evicted or dead)"); // Throw away the dead connection (passed max age or failed alive test)
          timeout = hardTimeout - clockSource.elapsedMillis(startTime);
        }
        else {
          return poolEntry.createProxyConnection(leakTask.schedule(poolEntry), now);
        }
      } while (timeout > 0L);
    }
    catch (InterruptedException e) {
      throw new SQLException(poolName + " - Interrupted during connection acquisition", e);
    }
    finally {
      // 4. 尝试释放锁(默认isAllowPoolSuspension为false, 即这里方法为空实现)
      suspendResumeLock.release();
    }
 		// ... 忽略由于连接超时抛出异常信息的代码逻辑
  }

  /**
    * Shutdown the pool, closing all idle connections and aborting or closing
    * active connections.
    *
    * @throws InterruptedException thrown if the thread is interrupted during shutdown
    */
  public final synchronized void shutdown() throws InterruptedException
  {
    try {
      // 1. 改变连接池状态为 POOL_SHUTDOWN
      poolState = POOL_SHUTDOWN;
			
      // 2. 遍历connectionBag中的PoolEntry, 尝试将PoolEntry状态STATE_NOT_IN_USE改为STATE_RESERVED, 成功则继续关闭连接Connection, 移除PoolEntry
      // 如果尝试修改状态失败, 则先将PoolEntry标记为 evicted (前面getConnection()方法实现中可以知道标记evited的PoolEntry不会被返回给用户)
      softEvictConnections();

      // 3. 关闭负责创建Connection的 ThreadPoolExecutor
      addConnectionExecutor.shutdown();
      addConnectionExecutor.awaitTermination(5L, SECONDS);
      // 4. 关闭负责定时清理空闲连接的ScheduledThreadPoolExecutor
      if (config.getScheduledExecutorService() == null && houseKeepingExecutorService != null) {
        houseKeepingExecutorService.shutdown();
        houseKeepingExecutorService.awaitTermination(5L, SECONDS);
      }
			
      // 5. 标记 connectionBag 关闭, 不允许再添加新的PoolEntry
      connectionBag.close();

      // 6. 5s内循环遍历池中的PoolEntry, 尝试关闭所有工作中的PoolEntry(关闭和数据库的物理连接, 回收相关资源), 以及关闭所有空闲连接
      final ExecutorService assassinExecutor = createThreadPoolExecutor(config.getMaximumPoolSize(), poolName + " connection assassinator", config.getThreadFactory(), new ThreadPoolExecutor.CallerRunsPolicy());
      try {
        final long start = clockSource.currentTime();
        do {
          abortActiveConnections(assassinExecutor);
          softEvictConnections();
        } while (getTotalConnections() > 0 && clockSource.elapsedMillis(start) < SECONDS.toMillis(5));
      }
      finally {
        // 7. 其余Executor关闭, 不再展开说明
        assassinExecutor.shutdown();
        assassinExecutor.awaitTermination(5L, SECONDS);
      }

      shutdownNetworkTimeoutExecutor();
      closeConnectionExecutor.shutdown();
      closeConnectionExecutor.awaitTermination(5L, SECONDS);
    }
    finally {
      logPoolState("After closing ");
      unregisterMBeans();
      metricsTracker.close();
      LOGGER.info("{} - Closed.", poolName);
    }
  }
}
```

### 代码小结

1. `HikariPool(final HikariConfig config)` 

   + 根据配置初始化连接池对象，使用`ConcurrentBag<PoolEntry>`维护池对象`PoolEntry`集合(`PoolEntry`和`Connection`一对一)。

   + 新建/关闭/清理`PoolEntry`都有专门的`ThreadPoolExecutor`，其中用`ScheduledThreadPoolExecutor`定时清理多余的空闲`PoolEntry`。

2. `getConnection()`

   超时时间内从`ConcurrentBag<PoolEntry>`实例connectionBag中循环尝试获取空闲可用的`PoolEntry`，超时抛出异常。

   循环中即使获取到空闲可用的`PoolEntry`，若遇到下述3种情况之一则尝试关闭`PoolEntry`，继续下一次循环：

   + 获取到的PoolEntry已被标记evicted
   + 上一次访问时间超过ALIVE_BYPASS_WINDOW_MS
   + 连接已关闭

   通过`ProxyFactory`代理，给`PoolEntry`关联新的`Connection`实例，并返回`Connection`实例。

3. `shutdown()`

   主要做的事情可概述为以下步骤：

   1. 确保不能创建新的`PoolEntry`
   2. 关闭不涉及"释放池资源"工作的ThreadPoolExecutor
   3. 5s内循环遍历`PoolEntry`，确保释放完物理连接资源
   4. 关闭参与"释放池资源"工作的ThreadPoolExecutor

## 3.3 
