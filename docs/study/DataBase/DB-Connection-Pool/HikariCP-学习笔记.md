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

### 1.2.1 适量的配置项

HikariCP和其他连接池实现相同的是，提供了诸如最小连接数、最大连接数等配置项。不同的是，Hikari提供的可配置项整体不多，主要追求极简主义。

下面列举一些常用配置项 (所有时间相关配置单位为ms毫秒)

| 配置项              | 说明                                                         | 默认值 |
| ------------------- | ------------------------------------------------------------ | ------ |
| dataSourceClassName | 数据源JDBC驱动类名，例如`com.mysql.cj.jdbc.Driver`           |        |
| jdbcUrl             | 数据库连接url，例如`jdbc:mysql://127.0.0.1:3306/{databaseName}`。一些老的驱动实现要求必须填充JDBC驱动类名 |        |
| username            | 数据库连接-用户名                                            |        |
| password            | 数据库连接-密码                                              |        |
| autoCommit          |                                                              | true   |
|                     |                                                              |        |
|                     |                                                              |        |
|                     |                                                              |        |
|                     |                                                              |        |
|                     |                                                              |        |
|                     |                                                              |        |
|                     |                                                              |        |
|                     |                                                              |        |
|                     |                                                              |        |
|                     |                                                              |        |
|                     |                                                              |        |
|                     |                                                              |        |
|                     |                                                              |        |
|                     |                                                              |        |

### 1.2.2 



# 2. JDBC查询流程回顾

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

# 3. DataSource

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

# 4. HikariDataSource

在传统JDBC查询流程中，数据库连接池由`DataSource`提供，所以我们着重关注HikariCP对`DataSource`的实现，即`HikariDataSource`类。

`HikariDataSource`在`DataSource`实现中，属于第二种，即连接池实现，主要在"连接池"方向精进。
