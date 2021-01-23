# HBase学习笔记

> [HBase官方文档-w3cschool](https://www.w3cschool.cn/hbase_doc/)	<=	学习资料来源，下面的笔记大都出自于此，其他来源会标注
>
> [Hbase-官方文档](https://hbase.apache.org/book.html)	<=	上面w3cschool的有些明显直接百度翻译的，整理不是很好。这里建议翻译诡异的地方，直接看官方英文文档。

# Introduction

HBase是Apache的Hadoop项目的子项目，是Hadoop Database的简称。

HBase是一个高可靠性、高性能、面向列、可伸缩的分布式存储系统，利用HBase技术可在廉价PC Server上搭建起大规模结构化存储集群。

HBase不同于一般的关系数据库，它是一个适合于非结构化数据存储的数据库，HBase基于列的而不是基于行的模式。

# Data Model

​	在 HBase 中，数据模型同样是由表组成的，各个表中又包含数据行和列，在这些表中存储了 HBase 数据。

***HBase Data Model Terminology***

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

## 21. Conceptual View

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

## 22. Physical View

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

## 23. Namespace

​	HBase命名空间namespace是与关系数据库系统中的数据库类似的表的逻辑分组。这种抽象为即将出现的多租户相关功能奠定了基础：

- 配额管理（Quota Management）（HBASE-8410） - 限制命名空间可占用的资源量（即区域，表）。
- **命名空间安全管理**（Namespace Security Administration）（HBASE-9206） - 为租户提供另一级别的安全管理。
- 区域服务器组（Region server groups）（HBASE-6721） - 命名空间/表可以固定在 RegionServers 的子集上，从而保证粗略的隔离级别。

### 23.1. Namespace management

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

### 23.2. Predefined namespaces

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

## 24. Table

HBase 中表是在 schema 定义时被预先声明的。

可以使用以下的命令来创建一个表，在这里必须指定表名和列族名。在 HBase shell 中创建表的语法如下所示：

```shell
create ‘<table name>’,’<column family>’
```

## 25. Row

**HBase中的行是逻辑上的行，物理上模型上行是按列族(colomn family)分别存取的**。

行键是未解释的字节，行是按字母顺序排序的，最低顺序首先出现在表中。

空字节数组用于表示表命名空间的开始和结束。

## 26. Column Family

Apache HBase 中的**列分为列族和列的限定符**。

**列的限定符是列族中数据的索引**。例如给定了一个列族 content，那么限定符可能是 content:html，也可以是 content:pdf。列族在创建表格时是确定的了，但是列的限定符是动态地并且行与行之间的差别也可能是非常大的。

Hbase表中的每个列都归属于某个列族，列族必须作为标模式(schema)定义的一部分预先给出。如 create'test',''course'。

列名以列族做为前缀，每个“列族”都可以有多个成员(colunm):如 course:math，course:english，新的列族成员(列)可以随后按需、动态加入。

**权限控制、存储以及调优都是在列族层面进行的**。

## 27. Cells

由行和列的坐标交叉决定;

**单元格是有版本的**;

**单元格的内容是未解析的字节数组**;

单元格是由行、列族、列限定符、值和代表值版本的时间戳组成的。

（{row key,column( =\<family>+\<qualifier>)，version}）唯一确定单元格。

<big>**cell中的数据是没有类型的，全部是字节码形式存储**。</big>

## 28. Data Model Operations

在 HBase 中有四个主要的数据模型操作，分别是：Get、Put、Scan 和 Delete。

### 28.1. Get（读取）

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

### 28.2. Put（写）

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

### 28.3. Scan（扫描）

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

> Note that generally the easiest way to specify a specific stop point for a scan is by using the [InclusiveStopFilter](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/filter/InclusiveStopFilter.html) class.

### 28.4. Delete（删除）

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

## 29. Versions

​	在 HBase 中，一个{row，column，version}元组精确指定了一个 cell。可能有无限数量的单元格，其中行和列是相同的，但单元格地址仅在其版本维度上有所不同。

​	**虽然行和列键以字节表示，但版本是使用长整数指定的**。通常，这个long包含时间实例，如由java.util.Date.getTime() 或者 System.currentTimeMillis() 返回的时间实例，即：1970年1月1日UTC的当前时间和午夜之间的差值（以毫秒为单位）。

​	**HBase 版本维度按递减顺序存储，因此从存储文件读取时，会先查找最新的值**。

​	在 HBase 中，cell 版本的语义有很多混淆。尤其是：

- **如果对一个单元的多次写入具有相同的版本，则只有最后一次写入是可以读取的。**
- **以非递增版本顺序编写单元格是可以的。**

​	下面我们描述 HBase 当前的版本维度是如何工作的。HBase 中的弯曲时间使得 HBase 中的版本或时间维度得到很好的阅读。它在版本控制方面的细节比这里提供的更多。

​	在撰写本文时，文章中提到的限制覆盖现有时间戳的值不再适用于HBase。本节基本上是 Bruno Dumon 撰写的文章的简介。

### 29.1. Specifying the Number of Versions to Store

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

### 29.2. Versions and HBase Operations

### 29.2.1. Get/Scan

**Get 操作实际是基于 Scan 实现的**. The below discussion of [Get](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Get.html) applies equally to [Scans](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Scan.html).

By default, i.e. if you specify no explicit version, when doing a `get`, the cell whose version has the largest value is returned (which may or may not be the latest one written, see later). The default behavior can be modified in the following ways:

- to return more than one version, see [Get.setMaxVersions()](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Get.html#setMaxVersions--)

- to return versions other than the latest, see [Get.setTimeRange()](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Get.html#setTimeRange-long-long-)

  To retrieve the latest version that is less than or equal to a given value, thus giving the 'latest' state of the record at a certain point in time, just use a range from 0 to the desired version and set the max versions to 1.

### 29.2.2. Default Get Example

The following Get will only retrieve the current version of the row

```java
public static final byte[] CF = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...
Get get = new Get(Bytes.toBytes("row1"));
Result r = table.get(get);
byte[] b = r.getValue(CF, ATTR);  // returns current version of value
```

### 29.2.3. Versioned Get Example

The following Get will return the last 3 versions of the row.

```java
public static final byte[] CF = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...
Get get = new Get(Bytes.toBytes("row1"));
get.setMaxVersions(3);  // will return last 3 versions of row
Result r = table.get(get);
byte[] b = r.getValue(CF, ATTR);  // returns current version of value
List<Cell> cells = r.getColumnCells(CF, ATTR);  // returns all versions of this column
```

### 29.2.4. Put

​	Doing a put always creates a new version of a `cell`, at a certain timestamp. By default the system uses the server’s `currentTimeMillis`, but you can specify the version (= the long integer) yourself, on a per-column level. This means you could assign a time in the past or the future, or use the long value for non-time purposes.

​	To overwrite an existing value, do a put at exactly the same row, column, and version as that of the cell you want to overwrite.

#### Implicit Version Example

​	The following Put will be implicitly versioned by HBase with the current time.

```java
public static final byte[] CF = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...
Put put = new Put(Bytes.toBytes(row));
put.add(CF, ATTR, Bytes.toBytes( data));
table.put(put);
```

#### Explicit Version Example

​	The following Put has the version timestamp explicitly set.

```java
public static final byte[] CF = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...
Put put = new Put( Bytes.toBytes(row));
long explicitTimeInMs = 555;  // just an example
put.add(CF, ATTR, explicitTimeInMs, Bytes.toBytes(data));
table.put(put);
```

​	Caution: the version timestamp is used internally by HBase for things like time-to-live calculations. It’s usually best to avoid setting this timestamp yourself. Prefer using a separate timestamp attribute of the row, or have the timestamp as a part of the row key, or both.

#### Cell Version Example

​	The following Put uses a method getCellBuilder() to get a CellBuilder instance that already has relevant Type and Row set.

```java
public static final byte[] CF = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...

Put put = new Put(Bytes.toBytes(row));
put.add(put.getCellBuilder().setQualifier(ATTR)
   .setFamily(CF)
   .setValue(Bytes.toBytes(data))
   .build());
table.put(put);
```

### 29.2.5. Delete

There are three different types of internal delete markers. See Lars Hofhansl’s blog for discussion of his attempt adding another, [Scanning in HBase: Prefix Delete Marker](http://hadoop-hbase.blogspot.com/2012/01/scanning-in-hbase.html).

- Delete: for a specific version of a column.
- Delete column: for all versions of a column.
- Delete family: for all columns of a particular ColumnFamily

**When deleting an entire row, HBase will internally create a tombstone for each ColumnFamily (i.e., not each individual column).**

Deletes work by creating *tombstone* markers. For example, let’s suppose we want to delete a row. For this you can specify a version, or else by default the `currentTimeMillis` is used. What this means is *delete all cells where the version is less than or equal to this version*. HBase never modifies data in place, so for example a delete will not immediately delete (or mark as deleted) the entries in the storage file that correspond to the delete condition. Rather, a so-called *tombstone* is written, which will mask the deleted values. When HBase does a major compaction, the tombstones are processed to actually remove the dead values, together with the tombstones themselves. If the version you specified when deleting a row is larger than the version of any value in the row, then you can consider the complete row to be deleted.

For an informative discussion on how deletes and versioning interact, see the thread [Put w/timestamp → Deleteall → Put w/ timestamp fails](http://comments.gmane.org/gmane.comp.java.hadoop.hbase.user/28421) up on the user mailing list.

Also see [keyvalue](https://hbase.apache.org/book.html#keyvalue) for more information on the internal KeyValue format.

Delete markers are purged during the next major compaction of the store, unless the `KEEP_DELETED_CELLS` option is set in the column family (See [Keeping Deleted Cells](https://hbase.apache.org/book.html#cf.keep.deleted)). To keep the deletes for a configurable amount of time, you can set the delete TTL via the hbase.hstore.time.to.purge.deletes property in *hbase-site.xml*. If `hbase.hstore.time.to.purge.deletes` is not set, or set to 0, all delete markers, including those with timestamps in the future, are purged during the next major compaction. Otherwise, a delete marker with a timestamp in the future is kept until the major compaction which occurs after the time represented by the marker’s timestamp plus the value of `hbase.hstore.time.to.purge.deletes`, in milliseconds.

## HBase排序顺序、列元数据以及联合查询

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

# HBase and Schema Design

## 34. Schema Creation

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

### 34.1. Schema Updates

​	**当对表或 ColumnFamilies (如区域大小、块大小) 进行更改时，这些更改将在下一次出现重大压缩并重新写入 StoreFiles 时生效。**

## 35. Table Schema 经验法则

在 HBase 中有许多不同的数据集，具有不同的访问模式和服务级别期望。因此，这些经验法则只是一个概述。

- 目标区域的大小介于10到50 GB之间。
- 目的是让单元格不超过10 MB，如果使用 mob，则为50 MB 。否则，请考虑将您的单元格数据存储在 HDFS 中，并在 HBase 中存储指向数据的指针。
- 典型的模式在每个表中有1到3个列族。HBase 表不应该被设计成模拟 RDBMS 表。
- **对于具有1或2列族的表格，大约50-100个区域是很好的数字。请记住，区域是列族的连续段**。
- **尽可能短地保留列族名称。列族名称存储在每个值 (忽略前缀编码) 中。它们不应该像在典型的 RDBMS 中一样具有自我记录和描述性。**
- 如果您正在存储基于时间的机器数据或日志记录信息，并且行密钥基于设备 ID 或服务 ID 加上时间，则最终可能会出现一种模式，即旧数据区域在某个时间段之后永远不会有额外的写入操作。在这种情况下，最终会有少量活动区域和大量没有新写入的较旧区域。对于这些情况，您可以容忍更多区域，因为**您的资源消耗仅由活动区域驱动**。
- **如果只有一个列族忙于写入，则只有该列族兼容内存。分配资源时请注意写入模式**。

# RegionServer Sizing 经验法则

Lars Hofhansl wrote a great [blog post](http://hadoop-hbase.blogspot.com/2013/01/hbase-region-server-memory-sizing.html) about RegionServer memory sizing. The upshot is that you probably need more memory than you think you need. He goes into the impact of region size, memstore size, HDFS replication factor, and other things to check.

> Personally I would place the maximum disk space per machine that can be served exclusively with HBase around 6T, unless you have a very read-heavy workload. In that case the Java heap should be 32GB (20G regions, 128M memstores, the rest defaults).
>
> — [Lars Hofhansl](http://hadoop-hbase.blogspot.com/2013/01/hbase-region-server-memory-sizing.html)

## 36. On the number of column families

> 第三段感觉W3Cschool翻译很奇怪，我稍微改了一点点。

​	**HBase 目前对于两列族或三列族以上的任何项目都不太合适，因此请将模式中的列族数量保持在较低水平**。

​	<u>目前，flushing 和 compactions 是按照每个区域进行的，所以如果一个列族承载大量数据带来的 flushing，即使所携带的数据量很小，也会 flushing 相邻的列族。当许多列族存在时，flushing 和 compactions 相互作用可能会导致一堆不必要的 I/O（要通过更改 flushing 和 compactions 来针对每个列族进行处理）</u>。

​	尽量在Schema模式中只用一个列族column family。通常仅在访问列作用域时，引入第二和第三列族；即通常你只查询一个列族或另一个列族，而不是同时查询两个列族。

​	<small>Try to make do with one column family if you can in your schemas. Only introduce a second and third column family in the case where data access is usually column scoped; i.e. you query one column family or the other but usually not both at the one time.</small>

### 36.1. ColumnFamilies的基数

​	<u>在一个表中存在多个 ColumnFamilies 的情况下，请注意基数（即行数）。如果 ColumnFamilyA 拥有100万行并且 ColumnFamilyB 拥有10亿行，则ColumnFamilyA 的数据可能会分布在很多很多地区（以及 Region Server）中。这使得 ColumnFamilyA 的大规模扫描效率较低。</u>

## 37. Rowkey Design

### 37.1. Hotspotting

> [如何避免 HBase 分区热点 (Hot Spotting) 问题？](https://www.toutiao.com/i6838868547582558734/)

​	**HBase 中的行按行键按顺序排序。这种设计优化了扫描（scan），允许您将相关的行或彼此靠近的行一起读取**。但是，设计不佳的行键是 hotspotting 的常见来源。当大量客户端通信针对群集中的一个节点或仅少数几个节点时，会发生 Hotspotting。此通信量可能表示读取、写入或其他操作。通信量压倒负责托管该区域的单个机器，从而导致性能下降并可能导致区域不可用性。这也会对由同一台区域服务器托管的其他区域产生不利影响，因为该主机无法为请求的负载提供服务。设计数据访问模式以使群集得到充分和均匀利用非常重要。

​	**为了防止 hotspotting 写入，请设计行键，使真正需要在同一个区域中的行成为行**，但是从更大的角度来看，数据将被写入整个群集中的多个区域，而不是一次。以下描述了避免 hotspotting 的一些常用技术，以及它们的一些优点和缺点。

#### Salting

​	从这个意义上说，Salting 与密码学无关，而是指**将随机数据添加到行键的开头**。在这种情况下，salting 是指为行键添加一个随机分配的前缀，以使它的排序方式与其他方式不同。可能的前缀数量对应于要传播数据的区域数量。如果你有一些“hotspotting”行键模式，反复出现在其他更均匀分布的行中，那么 Salting 可能会有帮助。请考虑以下示例，该示例显示 salting 可以跨多个 RegionServer 传播写入负载，并说明读取的一些负面影响。

使用实例

​	假设您有以下的行键列表，并且您的表格被拆分，以便字母表中的每个字母都有一个区域。前缀'a'是一个区域，前缀'b'是另一个区域。在此表中，所有以'f'开头的行都在同一个区域中。本示例重点关注具有以下键的行：

```shell
foo0001
foo0002
foo0003
foo0004
```

​	现在，想象你想要在四个不同的地区传播这些信息。您决定使用四个不同的 Salting：a，b，c 和 d。在这种情况下，每个这些字母前缀将位于不同的区域。应用 Salting 后，您可以使用以下 rowkeys。由于您现在可以写入四个不同的区域，因此理论上写入时的吞吐量是吞吐量的四倍，如果所有写入操作都在同一个区域，则会有这样的吞吐量。

```shell
A-foo0003
B-foo0001
C-foo0004
d-foo0002
```

​	然后，如果添加另一行，它将随机分配四种可能的 Salting 值中的一种，并最终靠近现有的一行。

```shell
A-foo0003
B-foo0001
C-foo0003
C-foo0004
d-foo0002
```

​	由于这个任务是随机的，如果你想按字典顺序检索行，你需要做更多的工作。

​	以这种方式，**Salting 试图增加写入吞吐量，但在读取期间会产生成本**。

#### Hashing

​	除了随机分配之外，您可以使用单向 Hashing，这会导致给定的行总是被相同的前缀“salted”，其方式会跨 RegionServer 传播负载，但允许在读取期间进行预测。使用确定性  Hashing 允许客户端重建完整的 rowkey 并使用 Get 操作正常检索该行。

Hashing 示例

​	考虑到上述 salting 示例中的相同情况，您可以改为应用单向 Hashing，这会导致带有键的行 foo0003 始终处于可预见的状态并接收 a 前缀。

​	然后，为了检索该行，您已经知道了密钥。

​	例如，您也可以优化事物，以便某些键对总是在相同的区域中。

#### Reversing the Key

​	**防止热点的第三种常用技巧是反转固定宽度或数字行键，以便最经常（最低有效位数）改变的部分在第一位。这有效地使行键随机化，但牺牲了行排序属性。**

> See https://communities.intel.com/community/itpeernetwork/datastack/blog/2013/11/10/discussion-on-designing-hbase-tables, and [article on Salted Tables](https://phoenix.apache.org/salted.html) from the Phoenix project, and the discussion in the comments of [HBASE-11682](https://issues.apache.org/jira/browse/HBASE-11682) for more information about avoiding hotspotting.

### 37.2. Monotonically Increasing Row Keys/Timeseries Data

> [OpenTSDB简介](https://blog.csdn.net/zbc415766331/article/details/103622830)

​	在 Tom White 的书“Hadoop: The Definitive Guide”（O'Reilly）的一章中，有一个优化笔记，关注一个现象，即导入过程与所有客户一起敲击表中的一个区域（and thus, a single node），然后移动到下一个区域等等。随着单调递增的行键（即，使用时间戳），这将发生。通过将输入记录随机化为不按排序顺序排列，可以缓解由单调递增密钥带来的单个区域上的堆积，但通常最好避免使用时间戳或序列（例如1，2，3）作为行键。

​	**如果您确实需要将时间序列数据上传到 HBase 中，则应将 OpenTSDB 作为一个成功的示例进行研究**。它有一个描述它在 HBase 中使用的模式的页面。OpenTSDB 中的关键格式实际上是 [metric_type] [event_timestamp]，它会在第一眼看起来与之前关于不使用时间戳作为关键的建议相矛盾。但是，区别在于时间戳不在密钥的主导位置，并且设计假设是有几十个或几百个（或更多）不同的度量标准类型。因此，即使连续输入数据和多种度量类型，Puts也会分布在表中不同的地区。

### 37.3. Try to minimize row and column sizes

​	**在 HBase 中，值总是随着坐标而运行；当单元格值通过系统时，它将始终伴随其行，列名称和时间戳**。如果你的行和列的名字很大，特别是与单元格的大小相比，那么你可能会遇到一些有趣的场景。其中之一就是 Marc Limotte 在 HBASE-3551 尾部描述的情况。其中，保存在 HBase存储文件（ StoreFile（HFile））以方便随机访问可能最终占用 HBase 分配的 RAM 的大块，因为单元值坐标很大。上面引用的注释中的标记建议增加块大小，以便存储文件索引中的条目以更大的间隔发生，或者修改表模式，以便使用较小的行和列名称。压缩也会使更大的指数。在用户邮件列表中查看线程问题 storefileIndexSize。

​	大多数时候，小的低效率并不重要。不幸的是，这是他们的情况。无论为 ColumnFamilies，属性和 rowkeys 选择哪种模式，都可以在数据中重复数十亿次。

#### 37.3.1. Column Families

​	**尽量保持 ColumnFamily 名称尽可能小，最好是一个字符（例如，"d" 用于 data 或者 default）**。

> See [KeyValue](https://hbase.apache.org/book.html#keyvalue) for more information on HBase stores data internally to see why this is important.

#### 37.3.2 Attributes

​	虽然详细的属性名称（例如，“myVeryImportantAttribute”）更易于阅读，但更推荐使用较短的属性名称（例如，“via”）来存储在 HBase 中。

> See [keyvalue](https://hbase.apache.org/book.html#keyvalue) for more information on HBase stores data internally to see why this is important.

#### 37.3.3. Rowkey Length

​	Keep them as short as is reasonable such that they can still be useful for required data access (e.g. Get vs. Scan). A short key that is useless for data access is not better than a longer key with better get/scan properties. Expect tradeoffs when designing rowkeys.

#### 37.3.4. Byte Patterns

​	A long is 8 bytes. You can store an unsigned number up to 18,446,744,073,709,551,615 in those eight bytes. If you stored this number as a String — presuming a byte per character — you need nearly 3x the bytes.

​	Not convinced? Below is some sample code that you can run on your own.

```java
// long
//
long l = 1234567890L;
byte[] lb = Bytes.toBytes(l);
System.out.println("long bytes length: " + lb.length);   // returns 8

String s = String.valueOf(l);
byte[] sb = Bytes.toBytes(s);
System.out.println("long as string length: " + sb.length);    // returns 10

// hash
//
MessageDigest md = MessageDigest.getInstance("MD5");
byte[] digest = md.digest(Bytes.toBytes(s));
System.out.println("md5 digest bytes length: " + digest.length);    // returns 16

String sDigest = new String(digest);
byte[] sbDigest = Bytes.toBytes(sDigest);
System.out.println("md5 digest as string length: " + sbDigest.length);    // returns 26
```

​	Unfortunately, using a binary representation of a type will make your data harder to read outside of your code. For example, this is what you will see in the shell when you increment a value:

```shell
hbase(main):001:0> incr 't', 'r', 'f:q', 1
COUNTER VALUE = 1

hbase(main):002:0> get 't', 'r'
COLUMN                                        CELL
 f:q                                          timestamp=1369163040570, value=\x00\x00\x00\x00\x00\x00\x00\x01
1 row(s) in 0.0310 seconds
```

​	The shell makes a best effort to print a string, and it this case it decided to just print the hex. The same will happen to your row keys inside the region names. It can be okay if you know what’s being stored, but it might also be unreadable if arbitrary data can be put in the same cells. This is the main trade-off.

### 37.4. Reverse Timestamps

> Reverse Scan API
>
> [HBASE-4811](https://issues.apache.org/jira/browse/HBASE-4811) implements an API to scan a table or a range within a table in reverse, reducing the need to optimize your schema for forward or reverse scanning. This feature is available in HBase 0.98 and later. See [Scan.setReversed()](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Scan.html#setReversed-boolean-) for more information.

​	HBASE-4811 实现一个 API，以反向扫描表中的表或区域，从而减少了为正向或反向扫描优化模式的需要。此功能在 HBase 0.98 和更高版本中可用。

​	数据库处理中的一个常见问题是快速找到最新版本的值。使用reverse timestamps作为key的一部分的技术可以帮助解决这个问题的一个特例。在 Tom White 的书籍“Hadoop：The Definitive Guide（O'Reilly）”的 HBase 章节中也有介绍，该技术包括附加 `Long.MAX_VALUE - timestamp` 到任何key的末尾 e.g. \[key]\[reverse_timestamp].

​	通过执行 Scan [key] 并获取第一条记录，可以找到表格中 [key] 的最新值。由于 HBase 密钥的排序顺序不同，因此该密钥在 [key] 的任何较旧的行键之前排序，因此是第一个。

​	这种技术将被用来代替使用版本号，其意图是永久保存所有版本（或者很长时间），同时通过使用相同的扫描技术来快速获得对任何其他版本的访问。

### 37.5. Rowkeys and ColumnFamilies

​	行键的范围为 ColumnFamilies。因此，相同的 rowkey 可以存在于没有碰撞的表中存在的每个 ColumnFamily 中。

​	<small>Rowkeys are scoped to ColumnFamilies. Thus, the same rowkey could exist in each ColumnFamily that exists in a table without collision.</small>

### 37.6. Immutability of Rowkeys

​	**行键无法更改**。他们可以在表格中“更改”的唯一方法是该行被删除然后重新插入。这是 HBase dist-list 上的一个相当常见的问题，所以在第一次（或在插入大量数据之前）获得 rowkeys 是值得的。

​	<small>Rowkeys cannot be changed. The only way they can be "changed" in a table is if the row is deleted and then re-inserted. This is a fairly common question on the HBase dist-list so it pays to get the rowkeys right the first time (and/or before you’ve inserted a lot of data).</small>

### 37.7. Relationship Between RowKeys and Region Splits

​	If you pre-split your table, it is *critical* to understand how your rowkey will be distributed across the region boundaries. As an example of why this is important, consider the example of using displayable hex characters as the lead position of the key (e.g., "0000000000000000" to "ffffffffffffffff"). Running those key ranges through `Bytes.split` (which is the split strategy used when creating regions in `Admin.createTable(byte[] startKey, byte[] endKey, numRegions)` for 10 regions will generate the following splits…

```java
48 48 48 48 48 48 48 48 48 48 48 48 48 48 48 48                                // 0
54 -10 -10 -10 -10 -10 -10 -10 -10 -10 -10 -10 -10 -10 -10 -10                 // 6
61 -67 -67 -67 -67 -67 -67 -67 -67 -67 -67 -67 -67 -67 -67 -68                 // =
68 -124 -124 -124 -124 -124 -124 -124 -124 -124 -124 -124 -124 -124 -124 -126  // D
75 75 75 75 75 75 75 75 75 75 75 75 75 75 75 72                                // K
82 18 18 18 18 18 18 18 18 18 18 18 18 18 18 14                                // R
88 -40 -40 -40 -40 -40 -40 -40 -40 -40 -40 -40 -40 -40 -40 -44                 // X
95 -97 -97 -97 -97 -97 -97 -97 -97 -97 -97 -97 -97 -97 -97 -102                // _
102 102 102 102 102 102 102 102 102 102 102 102 102 102 102 102                // f
```

​	(note: the lead byte is listed to the right as a comment.) Given that the first split is a '0' and the last split is an 'f', everything is great, right? Not so fast.

​	The problem is that all the data is going to pile up in the first 2 regions and the last region thus creating a "lumpy" (and possibly "hot") region problem. To understand why, refer to an [ASCII Table](http://www.asciitable.com/). '0' is byte 48, and 'f' is byte 102, but there is a huge gap in byte values (bytes 58 to 96) that will *never appear in this keyspace* because the only values are [0-9] and [a-f]. Thus, the middle regions will never be used. To make pre-splitting work with this example keyspace, a custom definition of splits (i.e., and not relying on the built-in split method) is required.

+ Lesson #1: **Pre-splitting tables is generally a best practice, but you need to pre-split them in such a way that all the regions are accessible in the keyspace**. While this example demonstrated the problem with a hex-key keyspace, the same problem can happen with *any* keyspace. Know your data.

+ Lesson #2: While generally not advisable, using hex-keys (and more generally, displayable data) can still work with pre-split tables as long as all the created regions are accessible in the keyspace.

​	To conclude this example, the following is an example of how appropriate splits can be pre-created for hex-keys:.

```java
public static boolean createTable(Admin admin, HTableDescriptor table, byte[][] splits)
throws IOException {
  try {
    admin.createTable( table, splits );
    return true;
  } catch (TableExistsException e) {
    logger.info("table " + table.getNameAsString() + " already exists");
    // the table already exists...
    return false;
  }
}

public static byte[][] getHexSplits(String startKey, String endKey, int numRegions) {
  byte[][] splits = new byte[numRegions-1][];
  BigInteger lowestKey = new BigInteger(startKey, 16);
  BigInteger highestKey = new BigInteger(endKey, 16);
  BigInteger range = highestKey.subtract(lowestKey);
  BigInteger regionIncrement = range.divide(BigInteger.valueOf(numRegions));
  lowestKey = lowestKey.add(regionIncrement);
  for(int i=0; i < numRegions-1;i++) {
    BigInteger key = lowestKey.add(regionIncrement.multiply(BigInteger.valueOf(i)));
    byte[] b = String.format("%016x", key).getBytes();
    splits[i] = b;
  }
  return splits;
}
```

##  38. Number of Versions

### 38.1. Maximum Number of Versions

​	HBase 通过 HColumnDescriptor 为每个列族配置要存储的最大行数版本。最大版本的默认值为1。这是一个重要的参数，因为如[Data Model](https://hbase.apache.org/book.html#datamodel)部分所述，HBase 没有覆盖行的值，而是按时间（和限定符）存储不同的值。在重要的压缩过程中删除多余的版本。最大版本的数量可能需要根据应用程序需求增加或减少。

​	不建议将最高版本数设置为极高的级别（例如，数百个或更多），除非这些旧值对您非常重要，因为这会大大增加 StoreFile 大小。

### 38.2. Minimum Number of Versions

​	与最大行版本数一样，HBase 通过 HColumnDescriptor 为每个列族配置要保留的最小行数版本。最小版本的默认值为0，这意味着该功能被禁用。行版本参数的最小数目与生存时间参数一起使用，并且可以与行版本参数的数目组合在一起，以允许诸如“保留最多T分钟值的数据，最多N个版本，但是至少保留 M 个版本 “（其中M 是最小行版本数的值，M <N）。仅当对列族启用了生存时间并且必须小于行版本的数量时，才应设置此参数。

##  39. Supported Datatypes

​	**HBase 通过 Put 操作和 Result 操作支持 “byte-in / bytes-out” 接口，所以任何可以转换为字节数组的内容都可以作为一个值存储。输入可以是字符串、数字、复杂对象、甚至可以是图像，只要它们可以呈现为字节。** 

​	There are practical limits to the size of values (e.g., storing 10-50MB objects in HBase would probably be too much to ask); search the mailing list for conversations on this topic. All rows in HBase conform to the [Data Model](https://hbase.apache.org/book.html#datamodel), and that includes versioning. Take that into consideration when making your design, as well as block size for the ColumnFamily.

> ​	HBase supports a "bytes-in/bytes-out" interface via [Put](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Put.html) and [Result](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Result.html), so anything that can be converted to an array of bytes can be stored as a value. Input could be strings, numbers, complex objects, or even images as long as they can rendered as bytes.

### 39.1. Counters

​	**One supported datatype that deserves special mention are "counters" (i.e., the ability to do atomic increments of numbers). See [Increment](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Table.html#increment(org.apache.hadoop.hbase.client.Increment)) in `Table`.**

​	**Synchronization on counters are done on the RegionServer, not in the client.**

## 40. Joins

​	If you have multiple tables, don’t forget to factor in the potential for [Joins](https://hbase.apache.org/book.html#joins) into the schema design.

> Whether HBase supports joins is a common question on the dist-list, and there is a simple answer: it doesn’t, at not least in the way that RDBMS' support them (e.g., with equi-joins or outer-joins in SQL). As has been illustrated in this chapter, the read data model operations in HBase are Get and Scan.
>
> However, that doesn’t mean that equivalent join functionality can’t be supported in your application, but you have to do it yourself. The two primary strategies are either denormalizing the data upon writing to HBase, or to have lookup tables and do the join between HBase tables in your application or MapReduce code (and as RDBMS' demonstrate, there are several strategies for this depending on the size of the tables, e.g., nested loops vs. hash-joins). So which is the best approach? It depends on what you are trying to do, and as such there isn’t a single answer that works for every use case.

## 41. Time To Live (TTL)

​	**ColumnFamilies can set a TTL length in seconds, and HBase will automatically delete rows once the expiration time is reached**. This applies to *all* versions of a row - even the current one. The TTL time encoded in the HBase for the row is specified in UTC.

​	**Store files which contains only expired rows are deleted on minor compaction. Setting `hbase.store.delete.expired.storefile` to `false` disables this feature. Setting minimum number of versions to other than 0 also disables this.**

​	See [HColumnDescriptor](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/HColumnDescriptor.html) for more information.

​	**Recent versions of HBase also support setting time to live on a per cell basis**. See [HBASE-10560](https://issues.apache.org/jira/browse/HBASE-10560) for more information. Cell TTLs are submitted as an attribute on mutation requests (Appends, Increments, Puts, etc.) using Mutation#setTTL. If the TTL attribute is set, it will be applied to all cells updated on the server by the operation. 

​	**There are two notable differences between cell TTL handling and ColumnFamily TTLs:**

- **Cell TTLs are expressed in units of milliseconds instead of seconds.**
- **A cell TTLs cannot extend the effective lifetime of a cell beyond a ColumnFamily level TTL setting.**

## 42. Keeping Deleted Cells

