# HBase学习笔记

> [HBase官方文档-w3cschool](https://www.w3cschool.cn/hbase_doc/)	<=	学习资料来源，下面的笔记大都出自于此，其他来源会标注
>
> [Hbase-官方文档](https://hbase.apache.org/book.html)	<=	上面w3cschool的有些明显直接百度翻译的，整理不是很好。这里建议翻译诡异的地方，直接看官方英文文档。
>
> ---
>
> 安装
>
> [Hbase的伪分布式安装](https://www.cnblogs.com/ivictor/p/5906433.html)
>
> [Hmaster启动几秒后自动消失的解决方法](https://blog.csdn.net/qq_46548855/article/details/106203616)
>
> 启动后访问 [本地WEB页面](http://localhost:16010/master-status)

# Introduction

> [hbase列存储怎么根据rowkey拼接成一条整行的数据？](https://www.zhihu.com/question/316484317/answer/635155771)
>
> [HBase基本概念与基本使用](https://www.cnblogs.com/swordfall/p/8737328.html)
>
> 上述两篇博文，质量挺高，建议阅读
>
> ![img](https://images2018.cnblogs.com/blog/1217276/201805/1217276-20180502141711373-31653278.png)

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

---

> [hbase列存储怎么根据rowkey拼接成一条整行的数据？](https://www.zhihu.com/question/316484317/answer/635155771)
>
> 首先HBase不是列存储，是列族+kv存储。HBase的表数据是按照rowkey有序的，一个区间的rowkey数据，组成一个分区 Region，region是HBase管理的最小单位，一个rowkey的数据只属于某一个region，region被分配到不同的regionserver中，通常一个region只对应一个regionserver。所以读取rowkey其实本质是读取某个region(某个regionserver)，然后region由一个或者多个列族组长，每个列族由 memstore，一个或者多个HFile组成，HFile通常是由memstore flush到HDFS或者bulkload产生的。一个HFile 内部是全局有序的，查询的时候由memstore，HFile 组成一个最小堆(查询还可能涉及到blockcache)，然后按序取数据返回即可。多个列族的话，本质没什么大的区别。看起来比较简单，实际上，HBase 查询那块代码还是比较复杂的。
>
> [HBase基本概念与基本使用](https://www.cnblogs.com/swordfall/p/8737328.html)
>
> ![img](https://images2018.cnblogs.com/blog/1217276/201805/1217276-20180502141711373-31653278.png)
>
> ![img](https://images2018.cnblogs.com/blog/1217276/201805/1217276-20180502154607794-710652455.png)
>
> HRegionServer管理一系列HRegion对象；
> 　　每个HRegion对应Table中一个Region，HRegion由多个HStore组成；
> 　　每个HStore对应Table中一个Column Family的存储；
> 　　Column Family就是一个集中的存储单元，故将具有相同IO特性的Column放在一个Column Family会更高效。

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

​	**By default, delete markers extend back to the beginning of time. Therefore, [Get](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Get.html) or [Scan](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Scan.html) operations will not see a deleted cell (row or column), even when the Get or Scan operation indicates a time range before the delete marker was placed.**

​	**ColumnFamilies can optionally keep deleted cells**. In this case, deleted cells can still be retrieved, as long as these operations specify a time range that ends before the timestamp of any delete that would affect the cells. This allows for point-in-time queries even in the presence of deletes.

​	**Deleted cells are still subject to TTL and there will never be more than "maximum number of versions" deleted cells. A new "raw" scan options returns all deleted rows and the delete markers.**

​	Change the Value of `KEEP_DELETED_CELLS` Using HBase Shell

```
hbase> hbase> alter ‘t1′, NAME => ‘f1′, KEEP_DELETED_CELLS => true
```

​	Example 12. Change the Value of `KEEP_DELETED_CELLS` Using the API

```
...
HColumnDescriptor.setKeepDeletedCells(true);
...
```

​	Let us illustrate the basic effect of setting the `KEEP_DELETED_CELLS` attribute on a table.

​	First, without:

```
create 'test', {NAME=>'e', VERSIONS=>2147483647}
put 'test', 'r1', 'e:c1', 'value', 10
put 'test', 'r1', 'e:c1', 'value', 12
put 'test', 'r1', 'e:c1', 'value', 14
delete 'test', 'r1', 'e:c1',  11

hbase(main):017:0> scan 'test', {RAW=>true, VERSIONS=>1000}
ROW                                              COLUMN+CELL
 r1                                              column=e:c1, timestamp=14, value=value
 r1                                              column=e:c1, timestamp=12, value=value
 r1                                              column=e:c1, timestamp=11, type=DeleteColumn
 r1                                              column=e:c1, timestamp=10, value=value
1 row(s) in 0.0120 seconds

hbase(main):018:0> flush 'test'
0 row(s) in 0.0350 seconds

hbase(main):019:0> scan 'test', {RAW=>true, VERSIONS=>1000}
ROW                                              COLUMN+CELL
 r1                                              column=e:c1, timestamp=14, value=value
 r1                                              column=e:c1, timestamp=12, value=value
 r1                                              column=e:c1, timestamp=11, type=DeleteColumn
1 row(s) in 0.0120 seconds

hbase(main):020:0> major_compact 'test'
0 row(s) in 0.0260 seconds

hbase(main):021:0> scan 'test', {RAW=>true, VERSIONS=>1000}
ROW                                              COLUMN+CELL
 r1                                              column=e:c1, timestamp=14, value=value
 r1                                              column=e:c1, timestamp=12, value=value
1 row(s) in 0.0120 seconds
```

​	Notice how delete cells are let go.

​	Now let’s run the same test only with `KEEP_DELETED_CELLS` set on the table (you can do table or per-column-family):

```
hbase(main):005:0> create 'test', {NAME=>'e', VERSIONS=>2147483647, KEEP_DELETED_CELLS => true}
0 row(s) in 0.2160 seconds

=> Hbase::Table - test
hbase(main):006:0> put 'test', 'r1', 'e:c1', 'value', 10
0 row(s) in 0.1070 seconds

hbase(main):007:0> put 'test', 'r1', 'e:c1', 'value', 12
0 row(s) in 0.0140 seconds

hbase(main):008:0> put 'test', 'r1', 'e:c1', 'value', 14
0 row(s) in 0.0160 seconds

hbase(main):009:0> delete 'test', 'r1', 'e:c1',  11
0 row(s) in 0.0290 seconds

hbase(main):010:0> scan 'test', {RAW=>true, VERSIONS=>1000}
ROW                                                                                          COLUMN+CELL
 r1                                                                                          column=e:c1, timestamp=14, value=value
 r1                                                                                          column=e:c1, timestamp=12, value=value
 r1                                                                                          column=e:c1, timestamp=11, type=DeleteColumn
 r1                                                                                          column=e:c1, timestamp=10, value=value
1 row(s) in 0.0550 seconds

hbase(main):011:0> flush 'test'
0 row(s) in 0.2780 seconds

hbase(main):012:0> scan 'test', {RAW=>true, VERSIONS=>1000}
ROW                                                                                          COLUMN+CELL
 r1                                                                                          column=e:c1, timestamp=14, value=value
 r1                                                                                          column=e:c1, timestamp=12, value=value
 r1                                                                                          column=e:c1, timestamp=11, type=DeleteColumn
 r1                                                                                          column=e:c1, timestamp=10, value=value
1 row(s) in 0.0620 seconds

hbase(main):013:0> major_compact 'test'
0 row(s) in 0.0530 seconds

hbase(main):014:0> scan 'test', {RAW=>true, VERSIONS=>1000}
ROW                                                                                          COLUMN+CELL
 r1                                                                                          column=e:c1, timestamp=14, value=value
 r1                                                                                          column=e:c1, timestamp=12, value=value
 r1                                                                                          column=e:c1, timestamp=11, type=DeleteColumn
 r1                                                                                          column=e:c1, timestamp=10, value=value
1 row(s) in 0.0650 seconds
```

​	KEEP_DELETED_CELLS is to avoid removing Cells from HBase when the *only* reason to remove them is the delete marker. So with KEEP_DELETED_CELLS enabled deleted cells would get removed if either you write more versions than the configured max, or you have a TTL and Cells are in excess of the configured timeout, etc.

​	KEEP_DELETED_CELLS 是为了避免从 HBase 中删除Cell，“删除”它们的唯一理由即 the delete marker。如果您编写的版本多于配置的最大版本，或者您有TTL且CELL超过配置的超时等，则 KEEP_DELETED_CELLS 启用的”已删除单元格“将被删除。

## 43. Secondary Indexes and Alternate Query Paths

​	This section could also be titled "what if my table rowkey looks like *this* but I also want to query my table like *that*." 

​	A common example on the dist-list is where a row-key is of the format "user-timestamp" but there are reporting requirements on activity across users for certain time ranges. Thus, selecting by user is easy because it is in the lead position of the key, but time is not.

​	There is no single answer on the best way to handle this because it depends on…

- Number of users
- Data size and data arrival rate
- Flexibility of reporting requirements (e.g., completely ad-hoc date selection vs. pre-configured ranges)
- Desired execution speed of query (e.g., 90 seconds may be reasonable to some for an ad-hoc report, whereas it may be too long for others)

​	and solutions are also influenced by the size of the cluster and how much processing power you have to throw at the solution. Common techniques are in sub-sections below. This is a comprehensive, but not exhaustive, list of approaches.

​	**It should not be a surprise that secondary indexes require additional cluster space and processing**. 

​	*This is precisely what happens in an RDBMS because the act of creating an alternate index requires both space and processing cycles to update. RDBMS products are more advanced in this regard to handle alternative index management out of the box. However, HBase scales better at larger data volumes, so this is a feature trade-off.*

> Pay attention to [Apache HBase Performance Tuning](https://hbase.apache.org/book.html#performance) when implementing any of these approaches.
>
> Additionally, see the David Butler response in this dist-list thread [HBase, mail # user - Stargate+hbase](http://search-hadoop.com/m/nvbiBp2TDP/Stargate%2Bhbase&subj=Stargate+hbase)

### 43.1. Filter Query

​	根据具体情况，使用客户端请求过滤器可能是适当的。在这种情况下，不会创建二级索引。但是，请不要在应用程序（如单线程客户端）上对这样的大表进行全面扫描。

​	<small>Depending on the case, it may be appropriate to use [Client Request Filters](https://hbase.apache.org/book.html#client.filter). In this case, no secondary index is created. However, don’t try a full-scan on a large table like this from an application (i.e., single-threaded client).</small>

### 43.2. Periodic-Update Secondary Index

​	可以在另一个表中创建二级索引，通过 MapReduce 作业定期更新。该作业可以在一天内执行，但要取决于加载策略，它可能仍然可能与主数据表不同步。

​	<small>A secondary index could be created in another table which is periodically updated via a MapReduce job. The job could be executed intra-day, but depending on load-strategy it could still potentially be out of sync with the main data table.</small>

> See [mapreduce.example.readwrite](https://hbase.apache.org/book.html#mapreduce.example.readwrite) for more information.	

###  43.3. Dual-Write Secondary Index

​	另一种策略是在将数据发布到集群时构建二级索引（例如，写入数据表，写入索引表）。如果这是在数据表已经存在之后采取的方法，那么对于具有 MapReduce 作业的二级索引将需要引导。

​	<small>Another strategy is to build the secondary index while publishing data to the cluster (e.g., write to data table, write to index table). If this is approach is taken after a data table already exists, then bootstrapping will be needed for the secondary index with a MapReduce job (see [secondary.indexes.periodic](https://hbase.apache.org/book.html#secondary.indexes.periodic)).</small>

### 43.4. Summary Tables

​	在时间范围非常广泛的情况下（例如，长达一年的报告）以及数据量大的地方，汇总表是一种常见的方法。这些将通过 MapReduce 作业生成到另一个表中。

​	<small>Where time-ranges are very wide (e.g., year-long report) and where the data is voluminous, summary tables are a common approach. These would be generated with MapReduce jobs into another table.</small>

> See [mapreduce.example.summary](https://hbase.apache.org/book.html#mapreduce.example.summary) for more information.

### 43.5. Coprocessor Secondary Index

​	协处理器行为类似于 RDBMS 触发器。这些在 0.92 增加。

​	<small>Coprocessors act like RDBMS triggers. These were added in 0.92. For more information, see [coprocessors](https://hbase.apache.org/book.html#cp)</small>

## 44. Constraints

​	HBase 目前支持传统（SQL）数据库术语中的“限制（constraints）”。Constraints 的建议用法是强制执行表中属性的业务规则（例如，确保值在 1-10 范围内）。也可以使用限制来强制引用完整性，但是强烈建议不要使用限制，因为它会显着降低启用完整性检查的表的写入吞吐量。 从版本 0.94 开始，可以在Constraint 中找到有关使用限制的大量文档 。

​	<small>HBase currently supports 'constraints' in traditional (SQL) database parlance. The advised usage for Constraints is in enforcing business rules for attributes in the table (e.g. make sure values are in the range 1-10). Constraints could also be used to enforce referential integrity, but this is strongly discouraged as it will dramatically decrease the write throughput of the tables where integrity checking is enabled. Extensive documentation on using Constraints can be found at [Constraint](https://hbase.apache.org/devapidocs/org/apache/hadoop/hbase/constraint/Constraint.html) since version 0.94.</small>

## 45. Schema Design Case Studies

以下将介绍 HBase 的一些典型的数据提取用例，以及如何处理 rowkey 设计和构造。注意：这只是潜在方法的一个例证，而不是详尽的清单。了解您的数据，并了解您的处理要求。

强烈建议您在阅读这些案例研究之前先阅读“[HBase 和 Schema 设计](https://www.w3cschool.cn/hbase_doc/hbase_doc-rnwo2maw.html)”的其余部分。

我们将在之后的小节中描述以下案例：

- 日志数据/时间序列数据
- Steroids上的日志数据/时间序列
- 客户订单
- 高/宽/中图式设计
- 列表数据

### 45.1. Case Study - Log Data and Timeseries Data

​	Assume that the following data elements are being collected.

- Hostname
- Timestamp
- Log event
- Value/message

​	We can store them in an HBase table called LOG_DATA, but what will the rowkey be? From these attributes the rowkey will be some combination of hostname, timestamp, and log-event - but what specifically?

#### 45.1.1. Timestamp In The Rowkey Lead Position

​	The rowkey `[timestamp][hostname][log-event]` suffers from the monotonically increasing rowkey problem described in [Monotonically Increasing Row Keys/Timeseries Data](https://hbase.apache.org/book.html#timeseries).

​	There is another pattern frequently mentioned in the dist-lists about "bucketing" timestamps, by performing a mod operation on the timestamp. If time-oriented scans are important, this could be a useful approach. **Attention must be paid to the number of buckets, because this will require the same number of scans to return results.**

```java
long bucket = timestamp % numBuckets;
```

​	to construct:

```java
[bucket][timestamp][hostname][log-event]
```

​	As stated above, to select data for a particular timerange, a Scan will need to be performed for each bucket. 100 buckets, for example, will provide a wide distribution in the keyspace but it will require 100 Scans to obtain data for a single timestamp, so there are trade-offs.

#### 45.1.2. Host In The Rowkey Lead Position

​	如果有大量的主机在整个keyspace中进行写入和读取操作，则 rowkey `[hostname][log-event][timestamp]` 是一个候选项。如果按主机名扫描是优先事项，则此方法非常有用。

​	<small>The rowkey `[hostname][log-event][timestamp]` is a candidate if there is a large-ish number of hosts to spread the writes and reads across the keyspace. This approach would be useful if scanning by hostname was a priority.</small>

####  45.1.3. Timestamp, or Reverse Timestamp?

​	如果最重要的访问路径是拉取最近的事件，则将时间戳存储为反向时间戳（例如，timestamp = Long.MAX_VALUE – timestamp）将创建能够对 [hostname][log-event] 执行 Scan 以获取最近捕获的事件的属性。

​	这两种方法都不是错的，它只取决于什么是最适合的情况。

​	<small>If the most important access path is to pull most recent events, then storing the timestamps as reverse-timestamps (e.g., `timestamp = Long.MAX_VALUE – timestamp`) will create the property of being able to do a Scan on `[hostname][log-event]` to obtain the most recently captured events.</small>

​	<small>Neither approach is wrong, it just depends on what is most appropriate for the situation.</small>

>  Reverse Scan API
>
> [HBASE-4811](https://issues.apache.org/jira/browse/HBASE-4811) implements an API to scan a table or a range within a table in reverse, reducing the need to optimize your schema for forward or reverse scanning. This feature is available in HBase 0.98 and later. See [Scan.setReversed()](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Scan.html#setReversed-boolean-) for more information.

#### 45.1.4. Variable Length or Fixed Length Rowkeys?

​	**It is critical to remember that rowkeys are stamped on every column in HBase**. If the hostname is `a` and the event type is `e1` then the resulting rowkey would be quite small. However, what if the ingested hostname is `myserver1.mycompany.com` and the event type is `com.package1.subpackage2.subsubpackage3.ImportantService`?

​	It might make sense to use some substitution in the rowkey. There are at least two approaches: hashed and numeric. In the Hostname In The Rowkey Lead Position example, it might look like this:

Composite Rowkey With Hashes:

- [MD5 hash of hostname] = 16 bytes
- [MD5 hash of event-type] = 16 bytes
- [timestamp] = 8 bytes

Composite Rowkey With Numeric Substitution:

For this approach another lookup table would be needed in addition to LOG_DATA, called LOG_TYPES. The rowkey of LOG_TYPES would be:

- `[type]` (e.g., byte indicating hostname vs. event-type)
- `[bytes]` variable length bytes for raw hostname or event-type.

A column for this rowkey could be a long with an assigned number, which could be obtained by using an [HBase counter](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Table.html#incrementColumnValue-byte:A-byte:A-byte:A-long-)

So the resulting composite rowkey would be:

- [substituted long for hostname] = 8 bytes
- [substituted long for event type] = 8 bytes
- [timestamp] = 8 bytes

​	**在 Hash 或 Numeric 替换方法中，主机名和事件类型的原始值可以存储为列。**

​	**In either the Hash or Numeric substitution approach, the raw values for hostname and event-type can be stored as columns.**

### 45.2. Case Study - Log Data and Timeseries Data on Steroids

​	这实际上是 OpenTSDB 的方法。OpenTSDB 做的是重写数据并将行打包到某些时间段中的列中。

​	This effectively is the OpenTSDB approach. What OpenTSDB does is re-write data and pack rows into columns for certain time-periods. For a detailed explanation, see: http://opentsdb.net/schema.html, and [Lessons Learned from OpenTSDB](https://www.slideshare.net/cloudera/4-opentsdb-hbasecon) from HBaseCon2012.

​	But this is how the general concept works: data is ingested, for example, in this manner…

```java
[hostname][log-event][timestamp1]
[hostname][log-event][timestamp2]
[hostname][log-event][timestamp3]
```

​	每个细节事件都有独立的 rowkeys，但是会被重写成这样：

​	<small>with separate rowkeys for each detailed event, but is re-written like this…</small>

```java
[hostname][log-event][timerange]
```

​	上述每个事件都转换为存储的列，其相对于开始 timerange 的时间偏移量 (例如，每5分钟)。这显然是一个非常先进的处理技术，但 HBase 使这成为可能。	

​	<small>and each of the above events are converted into columns stored with a time-offset relative to the beginning timerange (e.g., every 5 minutes). This is obviously a very advanced processing technique, but HBase makes this possible.</small>

### 45.3. Case Study - Customer/Order

假设 HBase 用于存储客户和订单信息。有两种核心记录类型被摄取：客户记录类型和订单记录类型。

客户记录类型将包含您通常期望的所有内容：

- 客户编号
- 客户名称
- 地址（例如，城市，州，邮编）
- 电话号码等

订单记录类型将包含如下内容：

- 客户编号
- 订单编号
- 销售日期
- 一系列用于装运位置和订单项的嵌套对象

假设客户编号和销售订单的组合唯一地标识一个订单，对于一个订单（ORDER）表，这两个属性将组成 rowkey，特别是一个组合键，例如：

```java
[customer number][order number]
```

但是，还有更多的设计决策需要：原始值是 rowkeys 的最佳选择吗？

Log Data 用例中的相同设计问题在这里面对我们。客户编号的keyspace是什么，以及格式是什么（例如，数字或是字母数字？）由于在HBase中使用固定长度的key以及可以在keyspace中支持**合理分布的key**是有利的，因此会出现类似的选项：

带有哈希的复合 Rowkey：

- [客户号码的 MD5] = 16字节
- [订单号的 MD5] = 16字节

复合数字/哈希组合 Rowkey：

- [substituted long for customer number] = 8 bytes
- [MD5 of order number] = 16 bytes

#### 45.3.1. Single Table? Multiple Tables?

A traditional design approach would have separate tables for CUSTOMER and SALES. Another option is to pack multiple record types into a single table (e.g., CUSTOMER++).

Customer Record Type Rowkey:

- [customer-id]
- [type] = type indicating `1' for customer record type

Order Record Type Rowkey:

- [customer-id]
- [type] = type indicating `2' for order record type
- [order]

这种特殊的 CUSTOMER ++ 方法的优点是通过客户 ID 来组织许多不同的记录类型（例如，一次扫描就可以得到关于该客户的所有信息）。缺点是扫描特定的记录类型并不容易。

The advantage of this particular CUSTOMER++ approach is that organizes many different record-types by customer-id (e.g., a single scan could get you everything about that customer). The disadvantage is that it’s not as easy to scan for a particular record-type.

#### 45.3.2. Order Object Design

Now we need to address how to model the Order object. Assume that the class structure is as follows:

- Order

  (an Order can have multiple ShippingLocations

- LineItem

  (a ShippingLocation can have multiple LineItems

there are multiple options on storing this data.

##### Completely Normalized

With this approach, there would be separate tables for ORDER, SHIPPING_LOCATION, and LINE_ITEM.

The ORDER table’s rowkey was described above: [schema.casestudies.custorder](https://hbase.apache.org/book.html#schema.casestudies.custorder)

The SHIPPING_LOCATION’s composite rowkey would be something like this:

- `[order-rowkey]`
- `[shipping location number]` (e.g., 1st location, 2nd, etc.)

The LINE_ITEM table’s composite rowkey would be something like this:

- `[order-rowkey]`
- `[shipping location number]` (e.g., 1st location, 2nd, etc.)
- `[line item number]` (e.g., 1st lineitem, 2nd, etc.)

Such a normalized model is likely to be the approach with an RDBMS, but that’s not your only option with HBase. The cons of such an approach is that to retrieve information about any Order, you will need:

- Get on the ORDER table for the Order
- Scan on the SHIPPING_LOCATION table for that order to get the ShippingLocation instances
- Scan on the LINE_ITEM for each ShippingLocation

<u>granted, this is what an RDBMS would do under the covers anyway, but since there are no joins in HBase you’re just more aware of this fact.</u>

##### Single Table With Record Types

With this approach, there would exist a single table ORDER that would contain

The Order rowkey was described above: [schema.casestudies.custorder](https://hbase.apache.org/book.html#schema.casestudies.custorder)

- `[order-rowkey]`
- `[ORDER record type]`

The ShippingLocation composite rowkey would be something like this:

- `[order-rowkey]`
- `[SHIPPING record type]`
- `[shipping location number]` (e.g., 1st location, 2nd, etc.)

The LineItem composite rowkey would be something like this:

- `[order-rowkey]`
- `[LINE record type]`
- `[shipping location number]` (e.g., 1st location, 2nd, etc.)
- `[line item number]` (e.g., 1st lineitem, 2nd, etc.)

##### Denormalized

​	具有记录类型的单个表格的一种变体是对一些对象层次结构进行非规范化和扁平化，比如将 ShippingLocation 属性折叠到每个 LineItem 实例上。

​	<small>A variant of the Single Table With Record Types approach is to denormalize and flatten some of the object hierarchy, such as collapsing the ShippingLocation attributes onto each LineItem instance.</small>

The LineItem composite rowkey would be something like this:

- `[order-rowkey]`
- `[LINE record type]`
- `[line item number]` (e.g., 1st lineitem, 2nd, etc., care must be taken that there are unique across the entire order)

and the LineItem columns would be something like this:

- itemNumber
- quantity
- price
- shipToLine1 (denormalized from ShippingLocation)
- shipToLine2 (denormalized from ShippingLocation)
- shipToCity (denormalized from ShippingLocation)
- shipToState (denormalized from ShippingLocation)
- shipToZip (denormalized from ShippingLocation)

​	这种方法的优点即使用不太复杂的对象层次结构，但其中一个缺点是，如果这些信息发生变化，更新会变得更加复杂。

​	<small>The pros of this approach include a less complex object hierarchy, but one of the cons is that updating gets more complicated in case any of this information changes.</small>

##### Object BLOB

​	通过这种方法，整个 Order 对象图都以某种方式处理为 BLOB。例如，上面描述了 ORDER 表的 rowkey：schema.casestudies.custorder，而一个名为“order”的列将包含一个可以反序列化的对象，该对象包含一个容器 Order，ShippingLocations 和 LineItems。

​	这里有很多选项：JSON，XML，Java 序列化，Avro，Hadoop Writable等等。所有这些都是相同方法的变体：将对象图编码为字节数组。应该注意这种方法，以确保在对象模型发生更改时保持向后兼容性，使旧的持久结构仍能从 HBase 中读出。

​	优点是能够以最少的 I/O 来管理复杂的对象图（例如，在本例中每个 HBase Get 有 Order），但缺点包括前面提到的关于序列化的向后兼容性，序列化的语言依赖性（例如 Java 序列化只适用于 Java 客户端），事实上你必须反序列化整个对象才能获得 BLOB 中的任何信息，以及像 Hive 这样的框架难以使用像这样的自定义对象。

### 45.4. Case Study - "Tall/Wide/Middle" Schema Design Smackdown

​	This section will describe additional schema design questions that appear on the dist-list, specifically about tall and wide tables. These are general guidelines and not laws - each application must consider its own needs.

#### 45.4.1. Rows vs. Versions

​	A common question is whether one should prefer rows or HBase’s built-in-versioning. The context is typically where there are "a lot" of versions of a row to be retained (e.g., where it is significantly above the HBase default of 1 max versions). The rows-approach would require storing a timestamp in some portion of the rowkey so that they would not overwrite with each successive update.

​	Preference: Rows (generally speaking).

#### 45.4.2. Rows vs. Columns

​	Another common question is whether one should prefer rows or columns. The context is typically in extreme cases of wide tables, such as having 1 row with 1 million attributes, or 1 million rows with 1 columns apiece.

​	Preference: Rows (generally speaking). To be clear, this guideline is in the context is in extremely wide cases, not in the standard use-case where one needs to store a few dozen or hundred columns. But there is also a middle path between these two options, and that is "Rows as Columns."

#### 45.4.3. Rows as Columns

​	HBase行与列之间的中间路径将打包数据，对于某些行，这些数据将成为单独的行。在这种情况下，OpenTSDB就是最好的例子，其中一行表示一个定义的时间范围，然后将离散事件视为列。这种方法通常更加复杂，并且可能需要重写数据的额外复杂性，但具有 I/O高效的优点。

​	<small>The middle path between Rows vs. Columns is packing data that would be a separate row into columns, for certain rows. OpenTSDB is the best example of this case where a single row represents a defined time-range, and then discrete events are treated as columns. This approach is often more complex, and may require the additional complexity of re-writing your data, but has the advantage of being I/O efficient. For an overview of this approach, see [schema.casestudies.log-steroids](https://hbase.apache.org/book.html#schema.casestudies.log_steroids).</small>

### 45.5. Case Study - List Data

以下是用户 dist-list 中关于一个相当常见问题的的交流：如何处理 Apache HBase 中的每个用户列表数据。

问题：

我们正在研究如何在 HBase 中存储大量（每用户）列表数据，并且我们试图弄清楚哪种访问模式最有意义。一种选择是将大部分数据存储在一个key中，所以我们可以有如下的内容：

```
<FixedWidthUserName><FixedWidthValueId1>:"" (no value)
<FixedWidthUserName><FixedWidthValueId2>:"" (no value)
<FixedWidthUserName><FixedWidthValueId3>:"" (no value)
```

我们的另一个选择是完全使用如下内容：

```
<FixedWidthUserName><FixedWidthPageNum0>:<FixedWidthLength><FixedIdNextPageNum><ValueId1><ValueId2><ValueId3>...
<FixedWidthUserName><FixedWidthPageNum1>:<FixedWidthLength><FixedIdNextPageNum><ValueId1><ValueId2><ValueId3>...
```

每行将包含多个值。所以在一种情况下，读取前三十个值将是：

```
scan { STARTROW => 'FixedWidthUsername' LIMIT => 30}
```

而在第二种情况下会是这样：

```
get 'FixedWidthUserName\x00\x00\x00\x00'
```

一般的使用模式是只读取这些列表的前30个值，并且很少的访问会深入到列表中。一些用户在这些列表中总共有30个值，并且一些用户将拥有数百万（即幂律分布）。

单值格式似乎会占用 HBase 上的更多空间，但会提供一些改进的检索/分页灵活性。是否有任何显着的性能优势能够通过获取和扫描的页面进行分页？

<small>The single-value format seems like it would take up more space on HBase, but would offer some improved retrieval / pagination flexibility. Would there be any significant performance advantages to be able to paginate via gets vs paginating with scans?</small>

我最初的理解是，如果我们的分页大小未知（并且缓存设置恰当），那么执行扫描应该会更快，但如果我们始终需要相同的页面大小，则扫描速度应该更快。我听到不同的人告诉了我关于表现的相反事情。我假设页面大小会相对一致，所以对于大多数使用情况，我们可以保证我们只需要固定页面长度的情况下的一页数据。我还会假设我们将不经常更新，但可能会插入这些列表的中间（这意味着我们需要更新所有后续行）。

答案：

如果我理解正确，你最终试图以“user，valueid，value”的形式存储三元组，对吗？例如：

```
"user123, firstname, Paul",
"user234, lastname, Smith"
```

（但用户名是固定宽度，而 valueids 是固定宽度）。

而且，您的访问模式符合以下要求：“对于用户 X，列出接下来的30个值，以valueid Y开头”。是对的吗？这些值应该按 valueid 排序返回？

tl；dr 版本即，你可能应该为每个用户+值添加一行，除非你确定需要，否则不要自行构建复杂的行内分页方案。

您的两个选项反映了人们在设计 HBase 模式时常见的问题：我应该选择“高”还是“宽”？您的第一个模式是“高（tall）”：每行代表一个用户的一个值，因此每个用户的表中有很多行；行键是 user + valueid，并且会有（可能）单个列限定符，表示“值（value）”。如果您希望按行键来扫描排序顺序中的行, 这是很好的。你可以在任何用户 + valueid 开始扫描，阅读下一个30，并完成。你放弃的是能够在一个用户的所有行周围提供事务保证，但它听起来并不像你需要的那样。

<small>Your two options mirror a common question people have when designing HBase schemas: should I go "tall" or "wide"? Your first schema is "tall": each row represents one value for one user, and so there are many rows in the table for each user; the row key is user + valueid, and there would be (presumably) a single column qualifier that means "the value". This is great if you want to scan over rows in sorted order by row key (thus my question above, about whether these ids are sorted correctly). You can start a scan at any user+valueid, read the next 30, and be done. What you’re giving up is the ability to have transactional guarantees around all the rows for one user, but it doesn’t sound like you need that. Doing it this way is generally recommended (see here https://hbase.apache.org/book.html#schema.smackdown).</small>

第二个选项是“宽”：使用不同的限定符（其中限定符是valueid）将一堆值存储在一行中。简单的做法是将一个用户的所有值存储在一行中。我猜你跳到了“分页”版本，因为你认为在单行中存储数百万列会对性能造成影响，这可能是也可能不是真的; 只要你不想在单个请求中做太多事情，或者做一些事情，比如扫描并返回行中的所有单元格，它不应该从根本上变坏。客户端具有允许您获取特定的列的片段的方法。

<small>Your second option is "wide": you store a bunch of values in one row, using different qualifiers (where the qualifier is the valueid). The simple way to do that would be to just store ALL values for one user in a single row. I’m guessing you jumped to the "paginated" version because you’re assuming that storing millions of columns in a single row would be bad for performance, which may or may not be true; as long as you’re not trying to do too much in a single request, or do things like scanning over and returning all of the cells in the row, it shouldn’t be fundamentally worse. The client has methods that allow you to get specific slices of columns.</small>

请注意，这两种情况都不会从根本上占用更多的磁盘空间; 您只是将部分识别信息“移动”到左侧（在行键中，在选项一中）或向右（在选项2中的列限定符中）。实际上，每个键/值仍然存储整个行键和列名称。

<small>Note that neither case fundamentally uses more disk space than the other; you’re just "shifting" part of the identifying information for a value either to the left (into the row key, in option one) or to the right (into the column qualifiers in option 2). Under the covers, every key/value still stores the whole row key, and column family name. (If this is a bit confusing, take an hour and watch Lars George’s excellent video about understanding HBase schema design: http://www.youtube.com/watch?v=_HLoH_PgrLk).</small>

正如你注意到的那样，手动分页版本有很多复杂性，例如必须跟踪每个页面中有多少内容，如果插入新值，则重新洗牌等。这看起来要复杂得多。在极高的吞吐量下它可能有一些轻微的速度优势（或缺点！），而要真正知道这一点的唯一方法就是试用它。如果您没有时间来构建它并进行比较，我的建议是从最简单的选项开始（每个用户 + valueid）。开始简单并重复！

<small>A manually paginated version has lots more complexities, as you note, like having to keep track of how many things are in each page, re-shuffling if new values are inserted, etc. That seems significantly more complex. It might have some slight speed advantages (or disadvantages!) at extremely high throughput, and the only way to really know that would be to try it out. If you don’t have time to build it both ways and compare, my advice would be to start with the simplest option (one row per user+value). Start simple and iterate! :)</small>

## 46. Operational and Performance Configuration Options

### 46.1. Tune HBase Server RPC Handling

- 设置 hbase.regionserver.handler.count（在 hbase-site.xml）为用于并发的核心 x 轴。
- 可选地，将调用队列分成单独的读取和写入队列以用于区分服务。该参数 hbase.ipc.server.callqueue.handler.factor 指定调用队列的数量：
  - 0 意味着单个共享队列。
  - 1 意味着每个处理程序的一个队列。
  - 一个0和1之间的值，按处理程序的数量成比例地分配队列数。例如，0.5 的值在每个处理程序之间共享一个队列。
- 使用 hbase.ipc.server.callqueue.read.ratio（hbase.ipc.server.callqueue.read.share在0.98中）将调用队列分成读写队列：
  - 0.5 意味着将有相同数量的读写队列。
  - <0.5 表示为读多于写。
  - \>0.5 表示写多于读。
- 设置 hbase.ipc.server.callqueue.scan.ratio（HBase 1.0+）将读取调用队列分为短读取和长读取队列：
  - 0.5 意味着将有相同数量的短读取和长读取队列。
  - <0.5表示更多的短读取队列。
  - \>0.5表示更多的长读取队列。

### 46.2. Disable Nagle for RPC

禁用 Nagle 的算法。延迟的 ACKs 可以增加到200毫秒的 RPC 往返时间。设置以下参数：

- 在 Hadoop 的 core-site.xml 中：
  - ipc.server.tcpnodelay = true
  - ipc.client.tcpnodelay = true
- 在 HBase 的 hbase-site.xml 中：
  - hbase.ipc.client.tcpnodelay = true
  - hbase.ipc.server.tcpnodelay = true

### 46.3. Limit Server Failure Impact

尽可能快地检测区域服务器故障。设置以下参数：

- 在 hbase-site.xml 中设置 zookeeper.session.timeout 为30秒或更短的时间内进行故障检测（20-30秒是一个好的开始）。
- 检测并避免不健康或失败的 HDFS 数据节点：in hdfs-site.xml 和 hbase-site.xml 设置以下参数：
  - dfs.namenode.avoid.read.stale.datanode = true
  - dfs.namenode.avoid.write.stale.datanode = true

### 46.4. Optimize on the Server Side for Low Latency

- 跳过本地块的网络。在 hbase-site.xml 中，设置以下参数：
  - dfs.client.read.shortcircuit = true
  - dfs.client.read.shortcircuit.buffer.size = 131072 （重要的是避免 OOME）
- 确保数据局部性。在 hbase-site.xml 中，设置 hbase.hstore.min.locality.to.skip.major.compact = 0.7（意味着 0.7 <= n <= 1）
- 确保 DataNode 有足够的处理程序进行块传输。在 hdfs-site.xml 中，设置以下参数：
  - dfs.datanode.max.xcievers >= 8192
  - dfs.datanode.handler.count = 主轴数量

Skip the network for local blocks when the RegionServer goes to read from HDFS by exploiting HDFS’s [Short-Circuit Local Reads](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/ShortCircuitLocalReads.html) facility. Note how setup must be done both at the datanode and on the dfsclient ends of the conneciton — i.e. at the RegionServer and how both ends need to have loaded the hadoop native `.so` library. After configuring your hadoop setting *dfs.client.read.shortcircuit* to *true* and configuring the *dfs.domain.socket.path* path for the datanode and dfsclient to share and restarting, next configure the regionserver/dfsclient side.

- In `hbase-site.xml`, set the following parameters:
  - `dfs.client.read.shortcircuit = true`
  - `dfs.client.read.shortcircuit.skip.checksum = true` so we don’t double checksum (HBase does its own checksumming to save on i/os. See [`hbase.regionserver.checksum.verify`](https://hbase.apache.org/book.html#hbase.regionserver.checksum.verify.performance) for more on this.
  - `dfs.domain.socket.path` to match what was set for the datanodes.
  - `dfs.client.read.shortcircuit.buffer.size = 131072` Important to avoid OOME — hbase has a default it uses if unset, see `hbase.dfs.client.read.shortcircuit.buffer.size`; its default is 131072.
- Ensure data locality. In `hbase-site.xml`, set `hbase.hstore.min.locality.to.skip.major.compact = 0.7` (Meaning that 0.7 <= n <= 1)
- Make sure DataNodes have enough handlers for block transfers. In `hdfs-site.xml`, set the following parameters:
  - `dfs.datanode.max.xcievers >= 8192`
  - `dfs.datanode.handler.count =` number of spindles

Check the RegionServer logs after restart. You should only see complaint if misconfiguration. Otherwise, shortcircuit read operates quietly in background. It does not provide metrics so no optics on how effective it is but read latencies should show a marked improvement, especially if good data locality, lots of random reads, and dataset is larger than available cache.

Other advanced configurations that you might play with, especially if shortcircuit functionality is complaining in the logs, include `dfs.client.read.shortcircuit.streams.cache.size` and `dfs.client.socketcache.capacity`. Documentation is sparse on these options. You’ll have to read source code.

RegionServer metric system exposes HDFS short circuit read metrics `shortCircuitBytesRead`. Other HDFS read metrics, including `totalBytesRead` (The total number of bytes read from HDFS), `localBytesRead` (The number of bytes read from the local HDFS DataNode), `zeroCopyBytesRead` (The number of bytes read through HDFS zero copy) are available and can be used to troubleshoot short-circuit read issues.

For more on short-circuit reads, see Colin’s old blog on rollout, [How Improved Short-Circuit Local Reads Bring Better Performance and Security to Hadoop](http://blog.cloudera.com/blog/2013/08/how-improved-short-circuit-local-reads-bring-better-performance-and-security-to-hadoop/). The [HDFS-347](https://issues.apache.org/jira/browse/HDFS-347) issue also makes for an interesting read showing the HDFS community at its best (caveat a few comments).

### 46.5. JVM Tuning

#### 46.5.1. Tune JVM GC for low collection latencies

- 使用 CMS 收集器： -XX:+UseConcMarkSweepGC
- 保持 eden 空间尽可能小，以减少平均收集时间。例：-XX：CMSInitiatingOccupancyFraction = 70
- 优化低收集延迟而不是吞吐量： -Xmn512m
- 并行收集 eden： -XX:+UseParNewGC
- 避免在压力下收集： -XX:+UseCMSInitiatingOccupancyOnly
- 限制每个请求扫描器的结果大小，所以一切都适合幸存者空间，但没有任职期限。在 hbase-site.xml 中，设置 hbase.client.scanner.max.result.size 为 eden 空间的1/8（使用 - Xmn512m，这里是〜51MB）
- 设置 max.result.sizex handler.count 小于 survivor 空间

#### 46.5.2. OS-Level Tuning

- Turn transparent huge pages (THP) off:

  ```
  echo never > /sys/kernel/mm/transparent_hugepage/enabled
  echo never > /sys/kernel/mm/transparent_hugepage/defrag
  ```

- Set `vm.swappiness = 0`

- Set `vm.min_free_kbytes` to at least 1GB (8GB on larger memory systems)

- Disable NUMA zone reclaim with `vm.zone_reclaim_mode = 0`

##  47. Special Cases

### 47.1. For applications where failing quickly is better than waiting

- 在客户端的 hbase-site.xml 中，设置以下参数：
  - 设置 hbase.client.pause = 1000
  - 设置 hbase.client.retries.number = 3
  - 如果你想跨越分裂和区域移动，大幅增加 hbase.client.retries.number（> = 20）
  - 设置 RecoverableZookeeper 重试计数： zookeeper.recovery.retry = 1（不重试）
- 在 hbase-site.xml 服务器端，设置 Zookeeper 会话超时以检测服务器故障：zookeeper.session.timeout⇐30秒（建议 20-30）。

### 47.2. For applications that can tolerate slightly out of date information

​	HBase 时间线一致性（HBASE-10070） 启用了只读副本后，区域（副本）的只读副本将分布在群集中。一个 RegionServer 为默认或主副本提供服务，这是唯一可以服务写入的副本。其他 Region Server 服务于辅助副本，请遵循主要 RegionServer，并仅查看提交的更新。辅助副本是只读的，但可以在主服务器故障时立即提供读取操作，从而将读取可用性的时间间隔从几秒钟减少到几毫秒。Phoenix 支持时间线一致性为 4.4.0 的提示：

- 部署 HBase 1.0.0 或更高版本。
- 在服务器端启用时间线一致性副本。
- 使用以下方法之一设置时间线一致性：
  - 使用 ALTER SESSION SET CONSISTENCY = 'TIMELINE’
  - 在JDBC连接字符串中设置连接属性 Consistency 为 timeline

### 47.3. More Information

See the Performance section [perf.schema](https://hbase.apache.org/book.html#perf.schema) for more information about operational and performance schema design options, such as Bloom Filters, Table-configured regionsizes, compression, and blocksizes.

# HBase and MapReduce

​	Apache MapReduce is a software framework used to analyze large amounts of data. It is provided by [Apache Hadoop](https://hadoop.apache.org/). MapReduce itself is out of the scope of this document. A good place to get started with MapReduce is https://hadoop.apache.org/docs/r2.6.0/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html. MapReduce version 2 (MR2)is now part of [YARN](https://hadoop.apache.org/docs/r2.3.0/hadoop-yarn/hadoop-yarn-site/).

​	This chapter discusses specific configuration steps you need to take to use MapReduce on data within HBase. In addition, it discusses other interactions and issues between HBase and MapReduce jobs. Finally, it discusses [Cascading](https://hbase.apache.org/book.html#cascading), an [alternative API](http://www.cascading.org/) for MapReduce.

> `mapred` and `mapreduce`
>
> There are two mapreduce packages in HBase as in MapReduce itself: *org.apache.hadoop.hbase.mapred* and *org.apache.hadoop.hbase.mapreduce*. The former does old-style API and the latter the new mode. The latter has more facility though you can usually find an equivalent in the older package. Pick the package that goes with your MapReduce deploy. When in doubt or starting over, pick *org.apache.hadoop.hbase.mapreduce*. In the notes below, we refer to *o.a.h.h.mapreduce* but replace with *o.a.h.h.mapred* if that is what you are using.

## 48. HBase, MapReduce, and the CLASSPATH

默认情况下，部署到 MapReduce 集群的 MapReduce 作业无权访问 $HBASE_CONF_DIR 类或 HBase 类下的 HBase 配置。

要为 MapReduce 作业提供他们需要的访问权限，可以添加 hbase-site.xml_to _ $ HADOOP_HOME / conf，并将 HBase jar 添加到 $ HADOOP_HOME / lib 目录。然后，您需要在群集中复制这些更改。或者你可以编辑 $ HADOOP_HOME / conf / hadoop-env.sh，并将 hbase 依赖添加到HADOOP_CLASSPATH 变量中。这两种方法都不推荐使用，因为它会使用 HBase 引用污染您的 Hadoop 安装。它还需要您在 Hadoop 可以使用 HBase 数据之前重新启动 Hadoop 集群。

**推荐的方法是让 HBase 添加它的依赖 jar 并使用 HADOOP_CLASSPATHor -libjars。**

自 HBase 0.90.x 以来，HBase 将其依赖 JAR 添加到作业配置本身。依赖关系只需要在本地 CLASSPATH 可用，从这里它们将被拾取并捆绑到部署到MapReduce 集群的 fat 工作 jar 中。一个基本的技巧就是将完整的 hbase 类路径（所有 hbase 和依赖 jar 以及配置）传递给 mapreduce 作业运行器，让hbase 实用程序从完整类路径中选取需要将其添加到 MapReduce 作业配置中的源代码。 

下面的示例针对名为 usertable 的表运行捆绑的 HBase RowCounter MapReduce 作业。它设置为 HADOOP_CLASSPATH 需要在 MapReduce 上下文中运行的 jar 包（包括配置文件，如 hbase-site.xml）。确保为您的系统使用正确版本的 HBase JAR；请在下面的命令行中替换 VERSION 字符串 w/本地 hbase 安装的版本。反引号（`符号）使 shell 执行子命令，设置输入 hbase classpath 为 HADOOP_CLASSPATH。这个例子假设你使用 BASH 兼容的 shell。

```
$ HADOOP_CLASSPATH=`${HBASE_HOME}/bin/hbase classpath` \
  ${HADOOP_HOME}/bin/hadoop jar ${HBASE_HOME}/lib/hbase-mapreduce-VERSION.jar \
  org.apache.hadoop.hbase.mapreduce.RowCounter usertable
```

上述的命令将针对 hadoop 配置指向的群集上的本地配置指向的 hbase 群集启动行计数 mapreduce 作业。

hbase-mapreduce.jar 的主要内容是一个 Driver（驱动程序），它列出了几个与 hbase 一起使用的基本 mapreduce 任务。例如，假设您的安装是hbase 2.0.0-SNAPSHOT：

```
$ HADOOP_CLASSPATH=`${HBASE_HOME}/bin/hbase classpath` \
  ${HADOOP_HOME}/bin/hadoop jar ${HBASE_HOME}/lib/hbase-mapreduce-2.0.0-SNAPSHOT.jar
An example program must be given as the first argument.
Valid program names are:
  CellCounter: Count cells in HBase table.
  WALPlayer: Replay WAL files.
  completebulkload: Complete a bulk data load.
  copytable: Export a table from local cluster to peer cluster.
  export: Write table data to HDFS.
  exportsnapshot: Export the specific snapshot to a given FileSystem.
  import: Import data written by Export.
  importtsv: Import data in TSV format.
  rowcounter: Count rows in HBase table.
  verifyrep: Compare the data from tables in two different clusters. WARNING: It doesn't work for incrementColumnValues'd cells since the timestamp is changed after being appended to the log.
```

您可以使用上面列出的缩短名称作为 mapreduce 作业，如下面的行计数器作业重新运行（再次假设您的安装为 hbase 2.0.0-SNAPSHOT）：

```
$ HADOOP_CLASSPATH=`${HBASE_HOME}/bin/hbase classpath` \
  ${HADOOP_HOME}/bin/hadoop jar ${HBASE_HOME}/lib/hbase-mapreduce-2.0.0-SNAPSHOT.jar \
  rowcounter usertable
```

您可能会发现更多有选择性的 hbase mapredcp 工具输出；它列出了针对 hbase 安装运行基本 mapreduce 作业所需的最小 jar 集。它不包括配置。如果您希望MapReduce 作业找到目标群集，则可能需要添加这些文件。一旦你开始做任何实质的事情，你可能还必须添加指向额外的 jar 的指针。只需在运行 hbase mapredcp 时通过系统属性 Dtmpjars 来指定附加项。

对于不打包它们的依赖关系或调用 TableMapReduceUtil#addDependencyJars 的作业，以下命令结构是必需的：

```
$ HADOOP_CLASSPATH=`${HBASE_HOME}/bin/hbase mapredcp`:${HBASE_HOME}/conf hadoop jar MyApp.jar MyJobMainClass -libjars $(${HBASE_HOME}/bin/hbase mapredcp | tr ':' ',') ...
```

**注意**

如果您从构建目录运行 HBase 而不是安装位置，该示例可能无法运行。您可能会看到类似以下的错误：

```
java.lang.RuntimeException: java.lang.ClassNotFoundException: org.apache.hadoop.hbase.mapreduce.RowCounter$RowCounterMapper
```

如果发生这种情况，请尝试按如下方式修改该命令，以便在构建环境中使用来自 target/ 目录的 HBase JAR 。

```
$ HADOOP_CLASSPATH=${HBASE_BUILD_HOME}/hbase-mapreduce/target/hbase-mapreduce-VERSION-SNAPSHOT.jar:`${HBASE_BUILD_HOME}/bin/hbase classpath` ${HADOOP_HOME}/bin/hadoop jar ${HBASE_BUILD_HOME}/hbase-mapreduce/target/hbase-mapreduce-VERSION-SNAPSHOT.jar rowcounter usertable
```

**HBase MapReduce 用户在 0.96.1 和 0.98.4 的通知**

一些使用 HBase 的 MapReduce 作业无法启动。该症状是类似于以下情况的异常:

```
Exception in thread "main" java.lang.IllegalAccessError: class
    com.google.protobuf.ZeroCopyLiteralByteString cannot access its superclass
    com.google.protobuf.LiteralByteString
    at java.lang.ClassLoader.defineClass1(Native Method)
    at java.lang.ClassLoader.defineClass(ClassLoader.java:792)
    at java.security.SecureClassLoader.defineClass(SecureClassLoader.java:142)
    at java.net.URLClassLoader.defineClass(URLClassLoader.java:449)
    at java.net.URLClassLoader.access$100(URLClassLoader.java:71)
    at java.net.URLClassLoader$1.run(URLClassLoader.java:361)
    at java.net.URLClassLoader$1.run(URLClassLoader.java:355)
    at java.security.AccessController.doPrivileged(Native Method)
    at java.net.URLClassLoader.findClass(URLClassLoader.java:354)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
    at
    org.apache.hadoop.hbase.protobuf.ProtobufUtil.toScan(ProtobufUtil.java:818)
    at
    org.apache.hadoop.hbase.mapreduce.TableMapReduceUtil.convertScanToString(TableMapReduceUtil.java:433)
    at
    org.apache.hadoop.hbase.mapreduce.TableMapReduceUtil.initTableMapperJob(TableMapReduceUtil.java:186)
    at
    org.apache.hadoop.hbase.mapreduce.TableMapReduceUtil.initTableMapperJob(TableMapReduceUtil.java:147)
    at
    org.apache.hadoop.hbase.mapreduce.TableMapReduceUtil.initTableMapperJob(TableMapReduceUtil.java:270)
    at
    org.apache.hadoop.hbase.mapreduce.TableMapReduceUtil.initTableMapperJob(TableMapReduceUtil.java:100)
...
```

这是由 HBASE-9867 中引入的优化导致的，它无意中引入了类加载器依赖性。

这会影响两个作业，即使用 libjars 选项和 "fat jar" (在嵌套的 lib 文件夹中打包其运行时依赖项)。

为了满足新的加载器要求，hbase-protocol.jar 必须包含在 Hadoop 的类路径中。以下内容包括用于历史目的。

通过在 Hadoop 的 lib 目录中包括对 hbase-protocol.jar 协议的引用，通过 symlink 或将 jar 复制到新的位置，可以解决整个系统。

这也可以通过 HADOOP_CLASSPATH 在作业提交时将它包含在环境变量中来实现。启动打包依赖关系的作业时，以下所有三个启动命令均满足此要求：

```
$ HADOOP_CLASSPATH=/path/to/hbase-protocol.jar:/path/to/hbase/conf hadoop jar MyJob.jar MyJobMainClass
$ HADOOP_CLASSPATH=$(hbase mapredcp):/path/to/hbase/conf hadoop jar MyJob.jar MyJobMainClass
$ HADOOP_CLASSPATH=$(hbase classpath) hadoop jar MyJob.jar MyJobMainClass
```

对于不打包它们的依赖关系的 jar，下面的命令结构是必需的：

```
$ HADOOP_CLASSPATH=$(hbase mapredcp):/etc/hbase/conf hadoop jar MyApp.jar MyJobMainClass -libjars $(hbase mapredcp | tr ':' ',') ...
```

##  49. MapReduce Scan Caching

现在，TableMapReduceUtil 恢复了在传入的 Scan 对象中设置扫描程序缓存（在将结果返回给客户端之前缓存的行数）的选项。由于 HBase 0.95（HBASE-11558）中的错误，此功能丢失。这是为 HBase 0.98.5 和0.96.3 而定的。选择扫描仪缓存的优先顺序如下：

1. 在扫描对象上设置的缓存设置。
2. 通过配置选项 hbase.client.scanner.caching 指定的缓存设置，可以在 hbase-site.xml 中手动设置或通过辅助方法 TableMapReduceUtil.setScannerCaching() 设置。
3. 默认值 HConstants.DEFAULT_HBASE_CLIENT_SCANNER_CACHING，设置为 100。

优化缓存设置是客户端等待结果的时间和客户端需要接收的结果集的数量之间的一种平衡。如果缓存设置过大，客户端可能会等待很长时间，否则请求可能会超时。如果设置太小，扫描需要返回几个结果。如果将 scan 视为 shovel，则更大的缓存设置类似于更大的 shovel，而更小的缓存设置相当于更多的 shovel，以填充 bucket。

上面提到的优先级列表允许您设置合理的默认值，并针对特定操作对其进行覆盖。

## 50. Bundled HBase MapReduce Jobs

HBase JAR 也可作为一些捆绑 MapReduce 作业的驱动程序。要了解捆绑的 MapReduce 作业，请运行以下命令：

```shell
$ ${HADOOP_HOME}/bin/hadoop jar ${HBASE_HOME}/hbase-mapreduce-VERSION.jar
An example program must be given as the first argument.
Valid program names are:
  copytable: Export a table from local cluster to peer cluster
  completebulkload: Complete a bulk data load.
  export: Write table data to HDFS.
  import: Import data written by Export.
  importtsv: Import data in TSV format.
  rowcounter: Count rows in HBase table
```

每个有效的程序名都是捆绑的 MapReduce 作业。要运行其中一个作业，请在下面的示例之后为您的命令建模。

```shell
$ ${HADOOP_HOME}/bin/hadoop jar ${HBASE_HOME}/hbase-mapreduce-VERSION.jar rowcounter myTable
```

## 51. HBase as a MapReduce Job Data Source and Data Sink

​	对于 MapReduce 作业，HBase 可以用作数据源、TableInputFormat 和数据接收器、TableOutputFormat 或 MultiTableOutputFormat。编写读取或写入HBase 的 MapReduce作业，建议子类化 TableMapper 或 TableReducer。

​	如果您运行使用 HBase 作为源或接收器的 MapReduce 作业，则需要在配置中指定源和接收器表和列名称。

​	当您从 HBase 读取时，TableInputFormat 请求 HBase 的区域列表并制作一张映射，可以是一个 map-per-region 或 mapreduce.job.maps mapreduce.job.maps ，映射到大于区域数目的数字。如果您为每个节点运行 TaskTracer/NodeManager 和 RegionServer，则映射将在相邻的 TaskTracker/NodeManager 上运行。在写入 HBase 时，避免使用 Reduce 步骤并从映射中写回 HBase 是有意义的。当您的作业不需要 MapReduce 对映射发出的数据进行排序和排序时，这种方法就可以工作。**在插入时，HBase 'sorts'，因此除非需要，否则双重排序（并在您的 MapReduce 集群周围混洗数据）没有意义**。如果您不需要 Reduce，则映射可能会发出在作业结束时为报告处理的记录计数，或者将 Reduces 的数量设置为零并使用 TableOutputFormat。**如果运行 Reduce 步骤在你的情况下是有意义的，则通常应使用多个Reducers，以便在 HBase 群集上传播负载**。

​	<small>When you read from HBase, the `TableInputFormat` requests the list of regions from HBase and makes a map, which is either a `map-per-region` or `mapreduce.job.maps` map, whichever is smaller. If your job only has two maps, raise `mapreduce.job.maps` to a number greater than the number of regions. Maps will run on the adjacent TaskTracker/NodeManager if you are running a TaskTracer/NodeManager and RegionServer per node. When writing to HBase, it may make sense to avoid the Reduce step and write back into HBase from within your map. This approach works when your job does not need the sort and collation that MapReduce does on the map-emitted data. On insert, HBase 'sorts' so there is no point double-sorting (and shuffling data around your MapReduce cluster) unless you need to. If you do not need the Reduce, your map might emit counts of records processed for reporting at the end of the job, or set the number of Reduces to zero and use TableOutputFormat. **If running the Reduce step makes sense in your case, you should typically use multiple reducers so that load is spread across the HBase cluster**.</small>

​	一个新的 HBase 分区程序 HRegionPartitioner 可以运行与现有区域数量一样多的 reducers。当您的表格很大时，HRegionPartitioner 是合适的，并且您的上传不会在完成时大大改变现有区域的数量。否则使用默认分区程序。

​	<small>A new HBase partitioner, the [HRegionPartitioner](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/mapreduce/HRegionPartitioner.html), can run as many reducers the number of existing regions. The HRegionPartitioner is suitable when your table is large and your upload will not greatly alter the number of existing regions upon completion. Otherwise use the default partitioner.</small>

## 52. Writing HFiles Directly During Bulk Import

​	如果您正在导入新表格，则可以绕过 HBase API 并将您的内容直接写入文件系统，格式化为 HBase 数据文件（HFiles）。您的导入将运行得更快，也许快一个数量级。有关此机制如何工作的更多信息，请参阅批量加载。

​	<small>**If you are importing into a new table, you can bypass the HBase API and write your content directly to the filesystem, formatted into HBase data files (HFiles). Your import will run faster, perhaps an order of magnitude faster**. For more on how this mechanism works, see [Bulk Loading](https://hbase.apache.org/book.html#arch.bulk.load).</small>

## 53. RowCounter Example

​	The included [RowCounter](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/mapreduce/RowCounter.html) MapReduce job uses `TableInputFormat` and does a count of all rows in the specified table. To run it, use the following command:

```
$ ./bin/hadoop jar hbase-X.X.X.jar
```

​	This will invoke the HBase MapReduce Driver class. Select `rowcounter` from the choice of jobs offered. This will print rowcounter usage advice to standard output. Specify the tablename, column to count, and output directory. If you have classpath errors, see [HBase, MapReduce, and the CLASSPATH](https://hbase.apache.org/book.html#hbase.mapreduce.classpath).

## 54. Map-Task Splitting

### 54.1. The Default HBase MapReduce Splitter

​	**When [TableInputFormat](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/mapreduce/TableInputFormat.html) is used to source an HBase table in a MapReduce job, its splitter will make a map task for each region of the table.** Thus, if there are 100 regions in the table, there will be 100 map-tasks for the job - regardless of how many column families are selected in the Scan.

### 54.2. Custom Splitters

​	For those interested in implementing custom splitters, see the method `getSplits` in [TableInputFormatBase](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/mapreduce/TableInputFormatBase.html). That is where the logic for map-task assignment resides.

## 55. HBase MapReduce Examples

### 55.1. HBase MapReduce Read Example

以下是以只读方式将 HBase 用作 MapReduce 源的示例。具体来说，有一个 Mapper 实例，但没有 Reducer，并且没有任何内容从 Mapper 发出。这项工作将被定义如下：

```
Configuration config = HBaseConfiguration.create();
Job job = new Job(config, "ExampleRead");
job.setJarByClass(MyReadJob.class);     // class that contains mapper

Scan scan = new Scan();
scan.setCaching(500);        // 1 is the default in Scan, which will be bad for MapReduce jobs
scan.setCacheBlocks(false);  // don't set to true for MR jobs
// set other scan attrs
...

TableMapReduceUtil.initTableMapperJob(
  tableName,        // input HBase table name
  scan,             // Scan instance to control CF and attribute selection
  MyMapper.class,   // mapper
  null,             // mapper output key
  null,             // mapper output value
  job);
job.setOutputFormatClass(NullOutputFormat.class);   // because we aren't emitting anything from mapper

boolean b = job.waitForCompletion(true);
if (!b) {
  throw new IOException("error with job!");
}
```

映射器实例将扩展 TableMapper：

```
public static class MyMapper extends TableMapper<Text, Text> {

  public void map(ImmutableBytesWritable row, Result value, Context context) throws InterruptedException, IOException {
    // process data for the row from the Result instance.
   }
}
```

###  55.2. HBase MapReduce Read/Write Example

以下是使用 HBase 作为 MapReduce 的源代码和接收器的示例。这个例子将简单地将数据从一个表复制到另一个表。

```
Configuration config = HBaseConfiguration.create();
Job job = new Job(config,"ExampleReadWrite");
job.setJarByClass(MyReadWriteJob.class);    // class that contains mapper

Scan scan = new Scan();
scan.setCaching(500);        // 1 is the default in Scan, which will be bad for MapReduce jobs
scan.setCacheBlocks(false);  // don't set to true for MR jobs
// set other scan attrs

TableMapReduceUtil.initTableMapperJob(
  sourceTable,      // input table
  scan,             // Scan instance to control CF and attribute selection
  MyMapper.class,   // mapper class
  null,             // mapper output key
  null,             // mapper output value
  job);
TableMapReduceUtil.initTableReducerJob(
  targetTable,      // output table
  null,             // reducer class
  job);
job.setNumReduceTasks(0);

boolean b = job.waitForCompletion(true);
if (!b) {
    throw new IOException("error with job!");
}
```

需要解释的是 TableMapReduceUtil 正在做什么，特别是对于Reducer。TableOutputFormat 被用作 outputFormat 类，并且正在配置几个参数（例如，TableOutputFormat.OUTPUT_TABLE），以及将 reducer 输出键设置为 ImmutableBytesWritable 和 reducer 值为 Writable。这些可以由程序员在作业和 conf 中设置，但 TableMapReduceUtil 试图让事情变得更容易。

以下是示例映射器，它将创建 Put 并匹配输入 Result 并发出它。注意：这是 CopyTable 实用程序的功能。

```
public static class MyMapper extends TableMapper<ImmutableBytesWritable, Put>  {

  public void map(ImmutableBytesWritable row, Result value, Context context) throws IOException, InterruptedException {
    // this example is just copying the data from the source table...
      context.write(row, resultToPut(row,value));
    }

    private static Put resultToPut(ImmutableBytesWritable key, Result result) throws IOException {
      Put put = new Put(key.get());
      for (KeyValue kv : result.raw()) {
        put.add(kv);
      }
      return put;
    }
}
```

实际上并没有一个简化步骤，所以 TableOutputFormat 负责将 Put 发送到目标表。

这只是一个例子，开发人员可以选择不使用 TableOutputFormat 并连接到目标表本身。

### 55.3. HBase MapReduce Read/Write Example With Multi-Table Output

​	TODO: example for `MultiTableOutputFormat`.

### 55.4. HBase MapReduce Summary to HBase Example

以下的示例使用 HBase 作为 MapReduce 源，并使用一个summarization步骤。此示例将计算表中某个值的不同实例的数量，并将这些汇总计数写入另一个表中。

```java
Configuration config = HBaseConfiguration.create();
Job job = new Job(config,"ExampleSummary");
job.setJarByClass(MySummaryJob.class);     // class that contains mapper and reducer

Scan scan = new Scan();
scan.setCaching(500);        // 1 is the default in Scan, which will be bad for MapReduce jobs
scan.setCacheBlocks(false);  // don't set to true for MR jobs
// set other scan attrs

TableMapReduceUtil.initTableMapperJob(
  sourceTable,        // input table
  scan,               // Scan instance to control CF and attribute selection
  MyMapper.class,     // mapper class
  Text.class,         // mapper output key
  IntWritable.class,  // mapper output value
  job);
TableMapReduceUtil.initTableReducerJob(
  targetTable,        // output table
  MyTableReducer.class,    // reducer class
  job);
job.setNumReduceTasks(1);   // at least one, adjust as required

boolean b = job.waitForCompletion(true);
if (!b) {
  throw new IOException("error with job!");
}
```

In this example mapper a column with a String-value is chosen as the value to summarize upon. This value is used as the key to emit from the mapper, and an `IntWritable` represents an instance counter.

```java
public static class MyMapper extends TableMapper<Text, IntWritable>  {
  public static final byte[] CF = "cf".getBytes();
  public static final byte[] ATTR1 = "attr1".getBytes();

  private final IntWritable ONE = new IntWritable(1);
  private Text text = new Text();

  public void map(ImmutableBytesWritable row, Result value, Context context) throws IOException, InterruptedException {
    String val = new String(value.getValue(CF, ATTR1));
    text.set(val);     // we can only emit Writables...
    context.write(text, ONE);
  }
}
```

In the reducer, the "ones" are counted (just like any other MR example that does this), and then emits a `Put`.

```java
public static class MyTableReducer extends TableReducer<Text, IntWritable, ImmutableBytesWritable>  {
  public static final byte[] CF = "cf".getBytes();
  public static final byte[] COUNT = "count".getBytes();

  public void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
    int i = 0;
    for (IntWritable val : values) {
      i += val.get();
    }
    Put put = new Put(Bytes.toBytes(key.toString()));
    put.add(CF, COUNT, Bytes.toBytes(i));

    context.write(null, put);
  }
}
```

### 55.5. HBase MapReduce Summary to File Example

This very similar to the summary example above, with exception that this is using HBase as a MapReduce source but HDFS as the sink. The differences are in the job setup and in the reducer. The mapper remains the same.

```java
Configuration config = HBaseConfiguration.create();
Job job = new Job(config,"ExampleSummaryToFile");
job.setJarByClass(MySummaryFileJob.class);     // class that contains mapper and reducer

Scan scan = new Scan();
scan.setCaching(500);        // 1 is the default in Scan, which will be bad for MapReduce jobs
scan.setCacheBlocks(false);  // don't set to true for MR jobs
// set other scan attrs

TableMapReduceUtil.initTableMapperJob(
  sourceTable,        // input table
  scan,               // Scan instance to control CF and attribute selection
  MyMapper.class,     // mapper class
  Text.class,         // mapper output key
  IntWritable.class,  // mapper output value
  job);
job.setReducerClass(MyReducer.class);    // reducer class
job.setNumReduceTasks(1);    // at least one, adjust as required
FileOutputFormat.setOutputPath(job, new Path("/tmp/mr/mySummaryFile"));  // adjust directories as required

boolean b = job.waitForCompletion(true);
if (!b) {
  throw new IOException("error with job!");
}
```

As stated above, the previous Mapper can run unchanged with this example. As for the Reducer, it is a "generic" Reducer instead of extending TableMapper and emitting Puts.

```java
public static class MyReducer extends Reducer<Text, IntWritable, Text, IntWritable>  {

  public void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
    int i = 0;
    for (IntWritable val : values) {
      i += val.get();
    }
    context.write(key, new IntWritable(i));
  }
}
```

### 55.6. HBase MapReduce Summary to HBase Without Reducer

​	It is also possible to perform summaries without a reducer - if you use HBase as the reducer.

​	An HBase target table would need to exist for the job summary. The Table method `incrementColumnValue` would be used to atomically increment values. From a performance perspective, it might make sense to keep a Map of values with their values to be incremented for each map-task, and make one update per key at during the `cleanup` method of the mapper. However, your mileage may vary depending on the number of rows to be processed and unique keys.

​	In the end, the summary results are in HBase.

###  55.7. HBase MapReduce Summary to RDBMS

​	Sometimes it is more appropriate to generate summaries to an RDBMS. For these cases, it is possible to generate summaries directly to an RDBMS via a custom reducer. The `setup` method can connect to an RDBMS (the connection information can be passed via custom parameters in the context) and the cleanup method can close the connection.

​	It is critical to understand that number of reducers for the job affects the summarization implementation, and you’ll have to design this into your reducer. Specifically, whether it is designed to run as a singleton (one reducer) or multiple reducers. Neither is right or wrong, it depends on your use-case. **Recognize that the more reducers that are assigned to the job, the more simultaneous connections to the RDBMS will be created - this will scale, but only to a point**.

```java
public static class MyRdbmsReducer extends Reducer<Text, IntWritable, Text, IntWritable>  {

  private Connection c = null;

  public void setup(Context context) {
    // create DB connection...
  }

  public void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
    // do summarization
    // in this example the keys are Text, but this is just an example
  }

  public void cleanup(Context context) {
    // close db connection
  }

}
```

In the end, the summary results are written to your RDBMS table/s.

## 56. Accessing Other HBase Tables in a MapReduce Job

Although the framework currently allows one HBase table as input to a MapReduce job, other HBase tables can be accessed as lookup tables, etc., in a MapReduce job via creating an Table instance in the setup method of the Mapper.

```java
public class MyMapper extends TableMapper<Text, LongWritable> {
  private Table myOtherTable;

  public void setup(Context context) {
    // In here create a Connection to the cluster and save it or use the Connection
    // from the existing table
    myOtherTable = connection.getTable("myOtherTable");
  }

  public void map(ImmutableBytesWritable row, Result value, Context context) throws IOException, InterruptedException {
    // process Result...
    // use 'myOtherTable' for lookups
  }
```

## 57. Speculative Execution

> [MapReduce的 Speculative Execution机制](https://blog.csdn.net/u011495642/article/details/83623517)
>
> 如果10台机器，同时运行10个Mapper，9台都干完了，就1台始终没有提交。
>
> 调度器就会着急了，它就会找到这个机器运行的数据，和这个数据的副本在哪儿，再其他有副本的机器上开个任务，
>
> 谁先计算完，我就先收集谁的数据。

​	通常建议关闭使用 HBase 作为源的 MapReduce 作业的推测执行（speculative execution）功能。这可以通过属性或整个集群来实现。特别是对于长时间运行的作业，推测执行将创建重复的映射任务，将您的数据写入 HBase；这可能不是你想要的。

> It is generally advisable to turn off speculative execution for MapReduce jobs that use HBase as a source. This can either be done on a per-Job basis through properties, or on the entire cluster. Especially for longer running jobs, speculative execution will create duplicate map-tasks which will double-write your data to HBase; this is probably not what you want.
>
> See [spec.ex](https://hbase.apache.org/book.html#spec.ex) for more information.

## 58. Cascading

​	**[Cascading](http://www.cascading.org/) is an alternative API for MapReduce, which actually uses MapReduce, but allows you to write your MapReduce code in a simplified way.**

​	The following example shows a Cascading `Flow` which "sinks" data into an HBase cluster. The same `hBaseTap` API could be used to "source" data as well.

```java
// read data from the default filesystem
// emits two fields: "offset" and "line"
Tap source = new Hfs( new TextLine(), inputFileLhs );

// store data in an HBase cluster
// accepts fields "num", "lower", and "upper"
// will automatically scope incoming fields to their proper familyname, "left" or "right"
Fields keyFields = new Fields( "num" );
String[] familyNames = {"left", "right"};
Fields[] valueFields = new Fields[] {new Fields( "lower" ), new Fields( "upper" ) };
Tap hBaseTap = new HBaseTap( "multitable", new HBaseScheme( keyFields, familyNames, valueFields ), SinkMode.REPLACE );

// a simple pipe assembly to parse the input into fields
// a real app would likely chain multiple Pipes together for more complex processing
Pipe parsePipe = new Each( "insert", new Fields( "line" ), new RegexSplitter( new Fields( "num", "lower", "upper" ), " " ) );

// "plan" a cluster executable Flow
// this connects the source Tap and hBaseTap (the sink Tap) to the parsePipe
Flow parseFlow = new FlowConnector( properties ).connect( source, hBaseTap, parsePipe );

// start the flow, and block until complete
parseFlow.complete();

// open an iterator on the HBase table we stuffed data into
TupleEntryIterator iterator = parseFlow.openSink();

while(iterator.hasNext())
  {
  // print out each tuple from HBase
  System.out.println( "iterator.next() = " + iterator.next() );
  }

iterator.close();
```

   