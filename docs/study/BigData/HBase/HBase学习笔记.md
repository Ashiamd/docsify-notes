# HBase学习笔记

> [HBase官方文档](https://www.w3cschool.cn/hbase_doc/)	<=	学习资料来源，下面的笔记大都出自于此，其他来源会标注

# 1. 基本介绍

HBase是Apache的Hadoop项目的子项目，是Hadoop Database的简称。

HBase是一个高可靠性、高性能、面向列、可伸缩的分布式存储系统，利用HBase技术可在廉价PC Server上搭建起大规模结构化存储集群。

HBase不同于一般的关系数据库，它是一个适合于非结构化数据存储的数据库，HBase基于列的而不是基于行的模式。

# 2. HBase数据模型

## 2.0 术语

​	在 HBase 中，数据模型同样是由表组成的，各个表中又包含数据行和列，在这些表中存储了 HBase 数据。

+ 表（Table）

  HBase 会将数据组织进一张张的表里面，一个 HBase 表由多行组成。

+ 行（Row）

  + HBase 中的一行包含一个行键和一个或多个与其相关的值的列。

  + **在存储行时，行按字母顺序排序**。出于这个原因，行键的设计非常重要。目标是以相关行相互靠近的方式存储数据。

  + 常用的行键模式是网站域。如果你的行键是域名，则你可能应该将它们存储在相反的位置（org.apache.www，org.apache.mail，org.apache.jira）。这样，表中的所有 Apache 域都彼此靠近，而不是根据子域的第一个字母分布。

+ 列族（Column Family）

  + 出于性能原因，列族在物理上共同存在**一组列**和它们的值。
  + 在 HBase 中每个列族都有一组存储属性，例如其值是否应缓存在内存中，数据如何压缩或其行编码是如何编码的等等。
  + **表中的每一行都有相同的列族，但给定的行可能不会在给定的列族中存储任何内容**。

  + 列族一旦确定后，就不能轻易修改，因为它会影响到 HBase 真实的物理存储结构，但是列族中的列标识(Column Qualifier)以及其对应的值可以动态增删。

+ 列限定符（Column Qualifier）

  + 列限定符被添加到列族中，以提供给定数据段的索引。鉴于列族的`content`，列限定符可能是`content:html`，而另一个可能是`content:pdf`。

  + 虽然列族在创建表时是固定的，但列限定符是可变的，并且在行之间可能差别很大。

+ 单元格（Cell）

  单元格是行、列族和列限定符的组合，并且**包含值和时间戳**，它表示值的版本。

+ 时间戳（Timestamp）

  + 时间戳与每个值一起编写，并且是给定版本的值的标识符。

  + **默认情况下，时间戳表示写入数据时 RegionServer上的时间**，但可以在将数据放入单元格时指定不同的时间戳值。

## 2.1 HBase概念视图

​	本节介绍 HBase 的概念视图。

​	本节的示例是根据  BigTable 论文进行稍微修改后的示例。在本节的示例中有一个名为表 webtable，其中包含两行（com.cnn.www 和 com.example.www）以及名为 contents、anchor 和 people 的三个列族。在本例中，对于第一行（com.cnn.www）， anchor 包含两列（anchor:cssnsi.com，anchor:my.look.ca），并且 contents 包含一列（contents:html）。本示例包含具有行键 com.cnn.www 的行的5个版本，以及具有行键 com.example.www 的行的一个版本。contents:html 列限定符包含给定网站的整个 HTML。锚（anchor）列族的限定符每个包含与该行所表示的站点链接的外部站点以及它在其链接的锚点（anchor）中使用的文本。people 列族代表与该网站相关的人员。

​	列名称：按照约定，列名由其列族前缀和限定符组成。例如，列内容: html 由列族`contents`和`html`限定符组成。冒号字符（`:`）从列族限定符分隔列族。

 	webtable 表如下所示：

|  行键（Row Key）  | 时间戳（Time Stamp） |  ColumnFamily`contents`  |     ColumnFamily`anchor`      |   ColumnFamily `people`    |
| :---------------: | :------------------: | :----------------------: | :---------------------------: | :------------------------: |
|   “com.cnn.www”   |          T9          |                          |   anchor：cnnsi.com =“CNN”    |                            |
|   “com.cnn.www”   |          T8          |                          | anchor：my.look.ca =“CNN.com” |                            |
|   “com.cnn.www”   |          T6          | 内容：html =“<html> ...” |                               |                            |
|   “com.cnn.www”   |          T5          | 内容：html =“<html> ...” |                               |                            |
|   “com.cnn.www”   |          T3          | 内容：html =“<html> ...” |                               |                            |
| “com.example.www” |          T5          | 内容：html =“<html> ...” |                               | people:author = "John Doe" |

​	此表中显示为空的单元格在 HBase 中不占用空间或实际上存在。这正是使 HBase “稀疏”的原因。表格视图并不是查看 HBase 数据的唯一可能的方法，甚至是最准确的。以下代表与多维地图相同的信息。这只是用于说明目的的模拟，可能并不严格准确。

```json
{
  "com.cnn.www": {
    contents: {
      t6: contents:html: "<html>..."
      t5: contents:html: "<html>..."
      t3: contents:html: "<html>..."
    }
    anchor: {
      t9: anchor:cnnsi.com = "CNN"
      t8: anchor:my.look.ca = "CNN.com"
    }
    people: {}
  }
  "com.example.www": {
    contents: {
      t5: contents:html: "<html>..."
    }
    anchor: {}
    people: {
      t5: people:author: "John Doe"
    }
  }
}
```

## 2.2 物理视图

​	本节介绍 HBase 物理视图。

​	尽管在 HBase 概念视图中，表格被视为一组稀疏的行的集合，但它们是按列族进行物理存储的。可以随时将新的列限定符（column_family：column_qualifier）添加到现有的列族。 

ColumnFamily `anchor 表：`

| 行键（Row Key） | 时间戳（Time Stamp） |     ColumnFamily `anchor`     |
| :-------------: | :------------------: | :---------------------------: |
|  “com.cnn.www”  |          T9          |   anchor:cnnsi.com = "CNN"    |
|  “com.cnn.www”  |          T8          | anchor:my.look.ca = "CNN.com" |

ColumnFamily contents 表：

| 行键（Row Key） | 时间戳（Time Stamp） |  ColumnFamily `contents:`  |
| :-------------: | :------------------: | :------------------------: |
|  “com.cnn.www”  |          T6          | contents:html = "\<html>…" |
|  “com.cnn.www”  |          T5          | contents:html = "\<html>…" |
|  “com.cnn.www”  |          T3          | contents:html = "\<html>…" |

+ **HBase 概念视图中显示的空单元根本不存储**。

  ​	因此，对时间戳为 t8 的 contents:html 列值的请求将不返回任何值。同样，在时间戳为 t9 中一个anchor:my.look.ca 值的请求也不会返回任何值。

+ **如果未提供时间戳，则会返回特定列的最新值**。给定多个版本，最近的也是第一个找到的，因为时间戳按降序存储。

  ​	因此，如果没有指定时间戳，则对行 com.cnn.www 中所有列的值的请求将是: 时间戳 t6 中的 contents:html，时间戳 t9 中 anchor:cnnsi.com 的值，时间戳 t8 中 anchor:my.look.ca 的值。

## 2.3 HBase命名空间

### HBase命名空间

​	HBase命名空间namespace是与关系数据库系统中的数据库类似的表的逻辑分组。这种抽象为即将出现的多租户相关功能奠定了基础：

- 配额管理（Quota Management）（HBASE-8410） - 限制命名空间可占用的资源量（即区域，表）。
- **命名空间安全管理**（Namespace Security Administration）（HBASE-9206） - 为租户提供另一级别的安全管理。
- 区域服务器组（Region server groups）（HBASE-6721） - 命名空间/表可以固定在 RegionServers 的子集上，从而保证粗略的隔离级别。

### 命名空间管理

你可以创建、删除或更改命名空间。通过指定表单的完全限定表名，<u>在创建表时确定命名空间成员权限</u>：

```pseudocode
<table namespace>:<table qualifier>
```

示例：

```shell
#Create a namespace
create_namespace 'my_ns'

#create my_table in my_ns namespace
create 'my_ns:my_table', 'fam'

#alter namespace
alter_namespace 'my_ns', {METHOD => 'set', 'PROPERTY_NAME' => 'PROPERTY_VALUE'}

#drop namespace
drop_namespace 'my_ns'
```

### HBase预定义的命名空间

在 HBase 中有两个预定义的特殊命名空间：

- **hbase：系统命名空间，用于包含 HBase 内部表**
- **default：没有显式指定命名空间的表将自动落入此命名空间**

示例：

```shell
#namespace=foo and table qualifier=bar
create 'foo:bar', 'fam'

#namespace=default and table qualifier=bar
create 'bar', 'fam'
```

## 2.4 HBase表、行与列族

### HBase表

HBase 中表是在 schema 定义时被预先声明的。

可以使用以下的命令来创建一个表，在这里必须指定表名和列族名。在 HBase shell 中创建表的语法如下所示：

```shell
create ‘<table name>’,’<column family>’
```

### HBase行

**HBase中的行是逻辑上的行，物理上模型上行是按列族(colomn family)分别存取的**。

行键是未解释的字节，行是按字母顺序排序的，最低顺序首先出现在表中。

空字节数组用于表示表命名空间的开始和结束。

### HBase列族

Apache HBase 中的**列分为列族和列的限定符**。

**列的限定符是列族中数据的索引**。例如给定了一个列族 content，那么限定符可能是 content:html，也可以是 content:pdf。列族在创建表格时是确定的了，但是列的限定符是动态地并且行与行之间的差别也可能是非常大的。

Hbase表中的每个列都归属于某个列族，列族必须作为标模式(schema)定义的一部分预先给出。如 create'test',''course'。

列名以列族做为前缀，每个“列族”都可以有多个成员(colunm):如 course:math，course:english，新的列族成员(列)可以随后按需、动态加入。

**权限控制、存储以及调优都是在列族层面进行的**。

### HBase Cell

由行和列的坐标交叉决定;

**单元格是有版本的**;

**单元格的内容是未解析的字节数组**;

单元格是由行、列族、列限定符、值和代表值版本的时间戳组成的。

（{row key,column( =\<family>+\<qualifier>)，version}）唯一确定单元格。

<big>**cell中的数据是没有类型的，全部是字节码形式存储**。</big>

## 2.5 HBase数据模型操作

### HBase数据模型操作

在 HBase 中有四个主要的数据模型操作，分别是：Get、Put、Scan 和 Delete。

### Get（读取）

Get 指定行的返回属性。读取通过 Table.get 执行。

Get 操作的语法如下所示：

```shell
get ’<table name>’,’row1’
```

在以下的 get 命令示例中，我们扫描了 emp 表的第一行：

```shell
hbase(main):012:0> get 'emp', '1'

   COLUMN                     CELL
   
personal : city timestamp=1417521848375, value=hyderabad

personal : name timestamp=1417521785385, value=ramu

professional: designation timestamp=1417521885277, value=manager

professional: salary timestamp=1417521903862, value=50000

4 row(s) in 0.0270 seconds
```

#### 读取指定列

下面给出的是使用 get 操作读取指定列语法：

```shell
hbase>get 'table name', ‘rowid’, {COLUMN => ‘column family:column name ’}
```

在下面给出的示例表示用于读取 HBase 表中的特定列：

```shell
hbase(main):015:0> get 'emp', 'row1', {COLUMN=>'personal:name'}

  COLUMN                CELL
  
personal:name timestamp=1418035791555, value=raju

1 row(s) in 0.0080 seconds
```

### Put（写）

​	Put 可以将新行添加到表中（如果该项是新的）或者可以更新现有行（如果该项已经存在）。Put 操作通过 Table.put（non-writeBuffer）或 Table.batch（non-writeBuffer）执行。

​	Put 操作的命令如下所示，在该语法中，你需要注明新值：

```shell
put ‘table name’,’row ’,'Column family:column name',’new value’
```

​	新给定的值将替换现有的值，并更新该行。

#### Put操作示例

​	假设 HBase 中有一个表 EMP 拥有下列数据：

```shell
hbase(main):003:0> scan 'emp'
 ROW              COLUMN+CELL
row1 column=personal:name, timestamp=1418051555, value=raju
row1 column=personal:city, timestamp=1418275907, value=Hyderabad
row1 column=professional:designation, timestamp=14180555,value=manager
row1 column=professional:salary, timestamp=1418035791555,value=50000
1 row(s) in 0.0100 seconds
```

​	以下命令将员工名为“raju”的城市值更新为“Delhi”：

```shell
hbase(main):002:0> put 'emp','row1','personal:city','Delhi'
0 row(s) in 0.0400 seconds
```

​	更新后的表如下所示：

```shell
hbase(main):003:0> scan 'emp'
  ROW          COLUMN+CELL
row1 column=personal:name, timestamp=1418035791555, value=raju
row1 column=personal:city, timestamp=1418274645907, value=Delhi
row1 column=professional:designation, timestamp=141857555,value=manager
row1 column=professional:salary, timestamp=1418039555, value=50000
1 row(s) in 0.0100 seconds
```

### Scan（扫描）

Scan 允许在多个行上对指定属性进行迭代。

Scan 操作的语法如下：

```shell
scan ‘<table name>’ 
```

以下是扫描表格实例的示例。假定表中有带有键 "row1 "、 "row2 "、 "row3 " 的行，然后是具有键“abc1”，“abc2”和“abc3”的另一组行。以下示例显示如何设置Scan实例以返回以“row”开头的行。

```java
public static final byte[] CF = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...

Table table = ...      // instantiate a Table instance

Scan scan = new Scan();
scan.addColumn(CF, ATTR);
scan.setRowPrefixFilter(Bytes.toBytes("row"));
ResultScanner rs = table.getScanner(scan);
try {
  for (Result r = rs.next(); r != null; r = rs.next()) {
    // process result...
  }
} finally {
  rs.close();  // always close the ResultScanner!
}
```

请注意，通常，指定扫描的特定停止点的最简单方法是使用 InclusiveStopFilter 类。

### Delete（删除）

Delete 操作用于从表中删除一行。Delete 通过 Table.delete 执行。

**HBase 不会修改数据，因此通过创建名为 tombstones 的新标记来处理 Delete 操作**。这些  tombstones，以及没用的价值，都在重大的压实中清理干净。

使用 Delete 命令的语法如下：

```shell
delete ‘<table name>’, ‘<row>’, ‘<column name >’, ‘<time stamp>’
```

下面是一个删除特定单元格的例子：

```shell
hbase(main):006:0> delete 'emp', '1', 'personal data:city',
1417521848375
0 row(s) in 0.0060 seconds
```

#### 删除表的所有单元格

使用 “deleteall” 命令，可以删除一行中所有单元格。下面给出是 deleteall 命令的语法：

```shell
deleteall ‘<table name>’, ‘<row>’,
```

这里是使用“deleteall”命令删除 emp 表 row1 的所有单元的一个例子。

```shell
hbase(main):007:0> deleteall 'emp','1'
0 row(s) in 0.0240 seconds
```

使用 Scan 命令验证表。表被删除后的快照如下：

```shell
hbase(main):022:0> scan 'emp'

ROW                  COLUMN+CELL

2 column=personal data:city, timestamp=1417524574905, value=chennai 

2 column=personal data:name, timestamp=1417524556125, value=ravi

2 column=professional data:designation, timestamp=1417524204, value=sr:engg

2 column=professional data:salary, timestamp=1417524604221, value=30000

3 column=personal data:city, timestamp=1417524681780, value=delhi

3 column=personal data:name, timestamp=1417524672067, value=rajesh

3 column=professional data:designation, timestamp=1417523187, value=jr:engg

3 column=professional data:salary, timestamp=1417524702514, value=25000
```

## 2.5 HBase版本

​	在 HBase 中，一个{row，column，version}元组精确指定了一个 cell。可能有无限数量的单元格，其中行和列是相同的，但单元格地址仅在其版本维度上有所不同。

​	**虽然行和列键以字节表示，但版本是使用长整数指定的**。通常，这个long包含时间实例，如由java.util.Date.getTime() 或者 System.currentTimeMillis() 返回的时间实例，即：1970年1月1日UTC的当前时间和午夜之间的差值（以毫秒为单位）。

​	**HBase 版本维度按递减顺序存储，因此从存储文件读取时，会先查找最新的值**。

​	在 HBase 中，cell 版本的语义有很多混淆。尤其是：

- **如果对一个单元的多次写入具有相同的版本，则只有最后一次写入是可以读取的。**
- **以非递增版本顺序编写单元格是可以的。**

​	下面我们描述 HBase 当前的版本维度是如何工作的。HBase 中的弯曲时间使得 HBase 中的版本或时间维度得到很好的阅读。它在版本控制方面的细节比这里提供的更多。

​	在撰写本文时，文章中提到的限制覆盖现有时间戳的值不再适用于HBase。本节基本上是 Bruno Dumon 撰写的文章的简介。

### 指定要存储的HBase版本数量

为给定列存储的最大版本数是列架构的一部分，并在创建表时通过 alter 命令 HColumnDescriptor.DEFAULT_VERSIONS 指定 。

**在 HBase 0.96 之前，保留的版本的默认数量是3，但在 0.96 以及新版本中已更改为1**。 

示例 - 修改一个列族的最大版本数量

本示例使用HBase Shell来保留列族中所有列的最多5个版本f1。你也可以使用HColumnDescriptor。

```shell
hbase> alter ‘t1′, NAME => ‘f1′, VERSIONS => 5
```

示例 - 修改列族的最小版本数

**您还可以指定每列家族存储的最低版本数。默认情况下，它被设置为0，这意味着该功能被禁用**。

下面的示例通过 HBase Shell 将在列族 f1 中的所有列的最小版本数设置为2。你也可以使用 HColumnDescriptor。

```shell
hbase> alter't1'，NAME =>'f1'，MIN_VERSIONS => 2
```

**从 HBase 0.98.2 开始，您可以通过在 hbase-site.xml 中设置 hbase.column.max.version 为所有新创建列保留的最大版本数指定一个全局默认值**。

## 2.6 HBase排序顺序、列元数据以及联合查询

### HBase排序顺序

​	**<u>所有数据模型操作 HBase 以排序顺序返回数据。首先按行，然后按列族（ColumnFamily），然后是列限定符，最后是时间戳（反向排序，因此首先返回最新的记录）</u>**。

### HBase列元数据

​	**ColumnFamily 的内部 KeyValue 实例之外不存储列元数据**。因此，尽管 HBase 不仅可以支持每行大量的列数，而且还能对行之间的一组异构列进行维护，但您有责任跟踪列名。

​	**获得 ColumnFamily 存在的一组完整列的唯一方法是处理所有行。**

### HBase联合查询

​	HBase 是否支持联合是该区列表中的一个常见问题，并且有一个简单的答案：它不是，至少在 RDBMS 支持它们的方式中（例如，使用 SQL 中的等连接或外连接）。如本章所述，HBase 中读取的数据模型操作是 Get 和 Scan，你可以参考“[HBase数据模型操作](https://www.w3cschool.cn/hbase_doc/hbase_doc-g2pm2m10.html)”部分

​	但是，这并不意味着您的应用程序不支持等效的联合功能，但您必须自己动手。两个主要策略是在写入 HBase 时对数据进行非规格化，或者在您的应用程序或MapReduce 代码中使用查找表并进行HBase表之间的连接（并且正如 RDBMS 演示的那样，有几种策略取决于 HBase 的大小表，例如，嵌套循环与散列连接）。那么最好的方法是什么？这取决于你想要做什么，因此没有一个适用于每个用例的答案。

### ACID

​	ACID，指数据库事务正确执行的四个基本要素的缩写，即：原子性（Atomicity），一致性（Consistency），隔离性（Isolation），持久性（Durability）。

​	**HBase 支持特定场景下的 ACID，即对同一行的 Put 操作保证完全的 ACID**（HBASE-3584增加了多操作事务，HBASE-5229增加了多行事务，但原理是一样的）

# 3. HBase和Schema设计

## 3.1 HBase模式(Schema) 创建

### HBase模式创建

你可以使用 Apache HBase Shell 或使用 Java API 中的 Admin 来创建或更新 HBase 模式。

进行 ColumnFamily 修改时，必须禁用表格，例如：

```java
Configuration config = HBaseConfiguration.create();
Admin admin = new Admin(conf);
TableName table = TableName.valueOf("myTable");

admin.disableTable(table);

HColumnDescriptor cf1 = ...;
admin.addColumn(table, cf1);      // adding new ColumnFamily
HColumnDescriptor cf2 = ...;
admin.modifyColumn(table, cf2);    // modifying existing ColumnFamily

admin.enableTable(table);
```

### HBase模型更新

​	**当对表或 ColumnFamilies (如区域大小、块大小) 进行更改时，这些更改将在下一次出现重大压缩并重新写入 StoreFiles 时生效。**

## 3.2 HBase表格模式经验法则

在 HBase 中有许多不同的数据集，具有不同的访问模式和服务级别期望。因此，这些经验法则只是一个概述。

- 目标区域的大小介于10到50 GB之间。
- 目的是让单元格不超过10 MB，如果使用 mob，则为50 MB 。否则，请考虑将您的单元格数据存储在 HDFS 中，并在 HBase 中存储指向数据的指针。
- 典型的模式在每个表中有1到3个列族。HBase 表不应该被设计成模拟 RDBMS 表。
- **对于具有1或2列族的表格，大约50-100个区域是很好的数字。请记住，区域是列族的连续段**。
- **尽可能短地保留列族名称。列族名称存储在每个值 (忽略前缀编码) 中。它们不应该像在典型的 RDBMS 中一样具有自我记录和描述性。**
- 如果您正在存储基于时间的机器数据或日志记录信息，并且行密钥基于设备 ID 或服务 ID 加上时间，则最终可能会出现一种模式，即旧数据区域在某个时间段之后永远不会有额外的写入操作。在这种情况下，最终会有少量活动区域和大量没有新写入的较旧区域。对于这些情况，您可以容忍更多区域，因为**您的资源消耗仅由活动区域驱动**。
- **如果只有一个列族忙于写入，则只有该列族兼容内存。分配资源时请注意写入模式**。

# 4. Thumb的RegionServer大小规则

## 4.1 HBase列族数量

### HBase列族数量

> 第三段感觉W3Cschool翻译很奇怪，我稍微改了一点点。

​	**HBase 目前对于两列族或三列族以上的任何项目都不太合适，因此请将模式中的列族数量保持在较低水平**。

​	<u>目前，flushing 和 compactions 是按照每个区域进行的，所以如果一个列族承载大量数据带来的 flushing，即使所携带的数据量很小，也会 flushing 相邻的列族。当许多列族存在时，flushing 和 compactions 相互作用可能会导致一堆不必要的 I/O（要通过更改 flushing 和 compactions 来针对每个列族进行处理）</u>。

​	尽量在Schema模式中只用一个列族column family。通常仅在访问列作用域时，引入第二和第三列族；即通常你只查询一个列族或另一个列族，而不是同时查询两个列族。

​	<small>Try to make do with one column family if you can in your schemas. Only introduce a second and third column family in the case where data access is usually column scoped; i.e. you query one column family or the other but usually not both at the one time.</small>

### ColumnFamilies的基数

​	<u>在一个表中存在多个 ColumnFamilies 的情况下，请注意基数（即行数）。如果 ColumnFamilyA 拥有100万行并且 ColumnFamilyB 拥有10亿行，则ColumnFamilyA 的数据可能会分布在很多很多地区（以及 Region Server）中。这使得 ColumnFamilyA 的大规模扫描效率较低。</u>

## 4.2 Rowkey（行键）设计

