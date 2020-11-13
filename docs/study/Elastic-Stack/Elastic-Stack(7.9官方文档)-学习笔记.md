# Elastic-Stack(7.9官方文档)-学习笔记

> [Elasticsearch Reference 7.9 current](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)

# 1. What is Elasticsearch

+ Elasticsearch是分布式的搜索、分析引擎，是Elastic Stack的核心。

+ Logstash和Beats用于收集、聚合、丰富数据。

+ Kibana用于数据可视化管理。

​	索引、搜索、分析，这些都是在Elasticsearch上进行的。

​	ElasticSearch为所有类型（结构化/非结构化文本、数字数据、地理空间数据）的数据提供近乎实时的搜索和分析。透过数据检索和信息聚合，能够发现数据的趋势和模式。由于分布式的特性，即使数据增长，ElasticSearch的部署也能随之无缝增长。

​	ElasticSearch可用于许多地方：

+ 在网页上的搜索框
+ 存储并分析日志、度量和安全事件数据
+ 使用机器学习实时自动根据数据的行为建模

+ 使用ElasticSearch作为存储引擎自动化业务工作流
+ 把ElasticSearch当作地理信息系统（GIS）来管理、集成和分析空间信息
+ 使用Elasticsearch作为生物信息学研究工具存储和处理遗传数据

​	无论用ElasticSearch来解决什么问题，在Elasticsearch中处理数据、文档和索引的方式都是相同的。

## 1.1 Data in : documents and Indices

​	Elasticsearch是一个分布式文档存储。Elasticsearch没有将信息存储为列式数据行，而是**存储已序列化为JSON文档的复杂数据结构**。当集群中有多个Elasticsearch节点时，存储的文档将分布在整个集群中，并且可以从任何点立即访问。

​	当一个文档被存储时，它会被索引，并且可以在1秒内完成几乎实时的搜索。Elasticsearch使用一种称为反向索引的数据结构，它支持非常快速的全文搜索。反向索引罗列出出现在任何文档中的唯一单词，并标识出现这些单词的文档。

​	索引可以看成是documents文档的优化集合，每个文档document是字段的集合，这些字段就是包含数据的键值对。ElasticSearch对不同的数据类型，会采用不同的数据结构存储。

+ text fields are stored in inverted indices
+ numeric and geo fields are stored in BKD trees
+ ...

​	Elasticsearch具有无模式能力，即文档documents可以在我们不显式指定处理方式时被自动索引。启用了动态映射时，Elasticsearch自动检测和索引新字段。这种默认行为，使得检索和浏览数据变得简单，只要开始索引文档，Elasticsearch就会自动检测各种类型的数据并映射到Elasticsearch所具有的数据类型上。

​	**我们可以自定义动态映射的规则来控制字段的存储和索引方式**。

​	自定义映射规则，使得：

+  区分全文搜索字符串字段和精准查询字符串字段
+ 进行指定语言的文本分析
+ 优化部分/局部匹配时使用的字段
+ 使用自定义日期格式
+ 使用无法被自动检测的数据类型，如`geo_point`和`geo_shape`

​	为了不同的目的，以不同的方式索引同一字段通常是有用的。例如，您可能希望将字符串字段作为全文搜索的文本字段和用于排序或聚合数据的关键字字段编制索引。或者，您可以选择使用多个语言分析器来处理包含用户输入的字符串字段的内容。

​	索引期间应用于全文字段的分析链也在搜索时使用。查询全文字段时，在索引中查找术语之前，查询文本将经历相同的分析。

## 1.2 Information out: search and analyze

​	虽然您可以将Elasticsearch仅用于文档存储并检索文档及其元数据，但其真正的强大之处在于能够轻松访问基于Apache Lucene搜索引擎库构建的全套搜索功能。

​	**Elasticsearch提供了一个简单，一致的REST API，用于管理您的集群以及索引和搜索数据**。 为了进行测试，您可以轻松地直接从命令行或通过Kibana中的开发者控制台提交请求。 在您的应用程序中，您可以将Elasticsearch客户端用于您选择的语言：Java，JavaScript，Go，.NET，PHP，Perl，Python或Ruby。

### Searching your data

​	**Elasticsearch restapi支持结构化查询、全文查询和将两者结合起来的复杂查询**。结构化查询与可以在SQL中构造的查询类型类似。例如，您可以搜索员工索引中的性别和年龄字段，并按“雇用日期”字段对匹配项进行排序。如何找到所有匹配的搜索词的文本，并返回它们匹配的查询。

​	除了搜索单个术语，还可以执行短语搜索、相似性搜索（模糊查询）和前缀搜索，并获得自动完成建议。

​	<small>In addition to searching for individual terms, you can perform phrase searches, similarity searches, and prefix searches, and get autocomplete suggestions.</small>

​	是否有要搜索的地理空间或其他数字数据？Elasticsearch索引优化数据结构中的非文本数据，支持高性能的地理和数字查询。

​	您可以使用Elasticsearch全面的JSON风格的查询语言（querydsl）访问所有这些搜索功能。您还可以构造SQL样式的查询，以便在Elasticsearch内部以本机方式搜索和聚合数据，而JDBC和ODBC驱动程序使各种第三方应用程序能够通过SQL与Elasticsearch交互。

### Analyzing your data

​	Elasticsearch聚合使您能够构建复杂的数据摘要，并深入了解关键指标、模式和趋势。聚合不只是找到俗话所说的“大海捞针/干草堆里找针”，而是让您能够回答以下问题：

+ 干草堆里的针有多少？

+ 针的平均长度是多少？

+ 针的中间长度是多少？

+ 在过去的六个月里，每一个月都有多少针头被扔进草堆？

您还可以使用聚合来回答更微妙的问题，例如：

+ 你们最受欢迎的针头制造商是什么？

+ 是否有特别或异常的针丛？

​	因为聚合使用的数据结构和搜索操作的相同，所以它们也非常快。这使您能够实时分析和可视化数据。您的报表和仪表板会随着数据的更改而更新，因此您可以根据最新信息采取操作。

​	此外，聚合与搜索请求一起运行。**您可以在单个请求中对同一数据同时搜索文档、筛选结果和执行分析**。因为聚合是在特定搜索的上下文中计算的，所以您不仅仅显示所有70号针的计数，而是显示符合用户搜索条件的70号针的计数-例如，所有大小为70的不粘绣。（也就是搜索时会根据输入的词条，再进行进一步筛选）

### But Wait，there's more

​	如果想根据时间序列自动分析数据，可以采用机器学习功能来生成行为的精准基线和识别其中的异常模式。通过机器学习，能够监测到：

+ 与值、计数或频率的时间偏差有关的异常
+ 统计稀有性
+ 群体中某一成员的不寻常行为

​	最好的部分呢？您无需指定算法、模型或其他与数据科学相关的配置就可以做到这一点。

​	<small>And the best part? You can do this without having to specify algorithms, models, or other data science-related configurations.</small>

# 2. What’s new in 7.9

## 2.1 Fixed retries for cross-cluster replication

​	Cross-cluster replication now retries operations that failed due to a circuit breaker or a lost remote cluster connection.

## 2.2 Fixed index throttling

​	**当索引数据时，Elasticsearch和Lucene使用堆内存进行缓冲。为了控制内存使用，Elasticsearch会根据索引缓冲区设置将数据从缓冲区移动到磁盘**。**如果正在进行的索引速度超过了将数据重新定位到磁盘的速度，Elasticsearch将限制索引速度（节流）**。在以前的Elasticsearch版本中，此功能存在失效问题，节流未能激活。

## 2.3 EQL

​	**EQL（事件查询语言）是一种声明性语言，专门用于识别事件之间的模式和关系**。

​	<small>EQL (Event Query Language) is a declarative language dedicated for identifying patterns and relationships between events.</small>

​	在以下场景，您可能考虑使用EQL：

- Use Elasticsearch for threat hunting or other security use cases
- Search time series data or logs, such as network or system logs
- Want an easy way to explore relationships between events

​	A good intro on EQL and its purpose is available [in this blog post](https://www.elastic.co/blog/introducing-event-query-language). See the [EQL in Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/7.9/eql.html) documentaton for an in-depth explanation, and also the [language reference](https://eql.readthedocs.io/en/latest/query-guide/index.html).

This release includes the following features:

- Event queries
- Sequences
- Pipes

An in-depth discussion of EQL in ES scope can be found at [#49581](https://github.com/elastic/elasticsearch/issues/49581).

## 2.4 Data streams

​	数据流是摄取、搜索和管理连续生成的时间序列数据的一种方便、可伸缩的方法。它们提供了一种更简单的方法，<u>可以跨多个索引拆分数据，并且仍然可以通过单个命名资源进行查询</u>。

​	See the [Data streams documentation](https://www.elastic.co/guide/en/elasticsearch/reference/7.9/data-streams.html) to get started.

## 2.5 Enable fully concurrent snapshot operations

​	快照操作现在可以以完全并发的方式执行。

<small>	Snapshot operations can now execute in a fully concurrent manner.</small>

- 创建和删除操作可以按任何顺序启动

  <small>Create and delete operations can be started in any order</small>

- 删除操作等待快照完成，尽可能多地进行批处理以提高效率，并且，一旦在集群状态下排队，将阻止新快照在数据节点上启动，直到执行为止

  <small>Delete operations wait for snapshot finalization to finish, are batched as much as possible to improve efficiency and, once enqueued in the cluster state, prevent new snapshots from starting on data nodes until executed</small>

- 快照的创建是完全并行的，但是每个存储库的快照都是线性化的，快照定稿也是如此

  <small>Snapshot creation is completely concurrent across shards, but per shard snapshots are linearized for each repository, as are snapshot finalizations</small>

## 2.6 Improve speed and memory usage of multi-bucket aggregations

​	在7.9之前，ElasticSearch对许多复杂的聚合都做了一个简化的假设，要求在包含它们这些数据的每个bucket中多次复制相同的数据结构。其中最冗余的数据可能达到几千字节。对于像如下的这种聚合操作：

​	Before 7.9, many of our more complex aggregations made a simplifying assumption that required that they duplicate many data structures once per bucket that contained them. The most expensive of these weighed in at a couple of kilobytes each. So for an aggregation like:

```http
POST _search
{
  "aggs": {
    "date": {
      "date_histogram": { "field": "timestamp", "calendar_interval": "day" },
      "aggs": {
        "ips": {
          "terms": { "field": "ip" }
        }
      }
    }
  }
}
```

​	当运行超过三年时，这个聚合只在桶计算上就花费了几兆字节。嵌套程度越深的聚合在这方面的开销就越大。Elasticsearch 7.9消除了所有这些开销，这将使我们能够在内存较低的环境中更好地运行。

​	作为奖励？（As a bonus），我们为聚合编写了不少反弹基准，以确保这些测试不会减慢聚合速度，因此现在我们可以更科学地思考聚合性能。基准测试表明，这些更改不会影响简单的聚合树，并且会加快与上述示例相似或更高深度的复杂聚合树。您的实际性能变化会有所不同，但这种优化应该会有所帮助

## 2.7 Allow index filtering in field capabilities API

​	You can now supply an `index_filter` to the [field capabilities API](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-field-caps.html). Indices are filtered from the response if the provided query rewrites to `match_none` on every shard.

## 2.8  Support `terms` and `rare_terms` aggregations in transforms

​	Transforms now support the `terms` and `rare_terms` aggregations. The default behavior is that the results are collapsed in the following manner:

```none
<AGG_NAME>.<BUCKET_NAME>.<SUBAGGS...>...
```

Or if no sub-aggregations exist:

```none
<AGG_NAME>.<BUCKET_NAME>.<_doc_count>
```

​	默认情况下，映射也定义为展平。这是为了避免字段爆炸，同时仍提供（有限的）搜索和聚合功能。

​	<small>The mapping is also defined as `flattened` by default. This is to avoid field explosion while still providing (limited) search and aggregation capabilities.</small>

## 2.9  Optimize `date_histograms` across daylight savings time

> [elasticsearch 分片(Shards)的理解](https://blog.csdn.net/qq_38486203/article/details/80077844)

​	当前，包含夏令时转换的切分上的日期舍入比仅在DST转换的一侧包含日期的切分慢得多，而且还会在内存中生成大量短期对象。elasticsearch7.9有一个经过修改且效率更高的实现，它只给请求增加了相对较小的开销。

​	<small>Rounding dates on a shard that contains a daylight savings time transition is currently drastically slower than when a shard contains dates only on one side of the DST transition, and also generates a large number of short-lived objects in memory. Elasticsearch 7.9 has a revised and far more efficient implemention that adds only a comparatively small overhead to requests.</small>

## 2.10 Improved resilience to network disruption

​	Elasticsearch现在有一种机制，可以在网络中断时安全地恢复对等恢复，而之前的网络中断会使任何正在进行的对等恢复失败。

​	<small>Elasticsearch now has mechansisms to safely resume peer recoveries when there is network disruption, which would previously have failed any in-progress peer recoveries.</small>

## 2.11 Wildcard field optimised for wildcard queries

​	Elasticsearch现在支持通配符字段类型，它存储为类似grep的通配符查询优化的值。虽然这样的查询可以用于其他字段类型，但是它们受到限制，限制了它们的有用性。

​	<small>Elasticsearch now supports a `wildcard` field type, which stores values optimised for wildcard grep-like queries. While such queries are possible with other field types, they suffer from constraints that limit their usefulness.</small>

​	This field type is especially well suited for running grep-like queries on log lines. See the [wildcard datatype](https://www.elastic.co/guide/en/elasticsearch/reference/7.9/wildcard.html) documentation for more information.

## 2.12  Indexing metrics and back pressure

​	Elasticsearch 7.9现在跟踪索引过程中每个点（协调、主和复制）未完成的索引请求字节数的度量。这些度量在node stats API中公开。另外，新的设置索引_内存限制控制未完成的最大字节数，默认为堆的10%。一旦节点堆中的字节数被未完成的索引字节消耗，Elasticsearch将开始拒绝新的协调和主请求。

​	<small>Elasticsearch 7.9 now tracks metrics about the number of indexing request bytes that are outstanding at each point in the indexing process (coordinating, primary, and replication). These metrics are exposed in the node stats API. Additionally, the new setting `indexing_pressure.memory.limit` controls the maximum number of bytes that can be outstanding, which is 10% of the heap by default. Once this number of bytes from a node’s heap is consumed by outstanding indexing bytes, Elasticsearch will start rejecting new coordinating and primary requests.</small>

​	**另外，由于失败的复制操作可能会使副本失败，Elasticsearch将为复制字节数指定1.5倍的限制。只有复制字节可以触发此限制。如果复制字节增加到较高级别，则节点将停止接受新的协调和主操作，直到复制工作负载下降为止。**

​	<small>Additionally, since a failed replication operation can fail a replica, Elasticsearch will assign 1.5X limit for the number of replication bytes. Only replication bytes can trigger this limit. If replication bytes increase to high levels, the node will stop accepting new coordinating and primary operations until the replication work load has dropped.</small>

## 2.13 Inference in pipeline aggregations

​	在7.6中，我们引入了推理，使您能够通过摄取管道中的处理器使用回归或分类模型对新数据进行预测。现在，在7.9中，推理更加灵活！您可以在聚合中引用预先训练的数据帧分析模型，以推断父存储桶聚合的结果字段。聚合使用结果上的模型来提供预测。此添加使您能够在搜索时运行分类或回归分析。如果要对一小部分数据执行分析，则可以生成预测，而无需在摄取管道中设置处理器。

​	<small>In 7.6, we introduced [inference](https://www.elastic.co/guide/en/machine-learning/7.9/ml-inference.html) that enables you to make predictions on new data with your regression or classification models via a processor in an ingest pipeline. Now, in 7.9, inference is even more flexible! You can reference a pre-trained data frame analytics model in an [aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.9/search-aggregations-pipeline-inference-bucket-aggregation.html) to infer on the result field of the parent bucket aggregation. The aggregation uses the model on the results to provide a prediction. This addition enables you to run classification or regression analysis at search time. If you want to perform analysis on a small set of data, you can generate predictions without the need to set up a processor in the ingest pipeline.</small>

# 3. Getting started with Elasticsearch

​	Ready to take Elasticsearch for a test drive and see for yourself how you can use the REST APIs to store, search, and analyze data?

Step through this getting started tutorial to:

1. Get an Elasticsearch cluster up and running
2. Index some sample documents
3. Search for documents using the Elasticsearch query language
4. Analyze the results using bucket and metrics aggregations

Need more context?

​	Check out the [Elasticsearch Introduction](https://www.elastic.co/guide/en/elasticsearch/reference/current/elasticsearch-intro.html) to learn the lingo and understand the basics of how Elasticsearch works. If you’re already familiar with Elasticsearch and want to see how it works with the rest of the stack, you might want to jump to the [Elastic Stack Tutorial](https://www.elastic.co/guide/en/elastic-stack-get-started/7.9/get-started-elastic-stack.html) to see how to set up a system monitoring solution with Elasticsearch, Kibana, Beats, and Logstash.

> Tip：The fastest way to get started with Elasticsearch is to [start a free 14-day trial of Elasticsearch Service](https://www.elastic.co/cloud/elasticsearch-service/signup?baymax=docs-body&elektra=docs) in the cloud.

## 3.1  Get Elasticsearch up and running

​	To take Elasticsearch for a test drive, you can create a [hosted deployment](https://www.elastic.co/cloud/elasticsearch-service/signup?baymax=docs-body&elektra=docs) on the Elasticsearch Service or set up a multi-node Elasticsearch cluster on your own Linux, macOS, or Windows machine.

### 3.1.1  Run Elasticsearch on Elastic Cloud

​	When you create a deployment on the Elasticsearch Service, the service provisions a three-node Elasticsearch cluster along with Kibana and APM.

​	To create a deployment:

1. Sign up for a [free trial](https://www.elastic.co/cloud/elasticsearch-service/signup?baymax=docs-body&elektra=docs) and verify your email address.
2. Set a password for your account.
3. Click **Create Deployment**.

​	Once you’ve created a deployment, you’re ready to [*Index some documents*](https://www.elastic.co/guide/en/elasticsearch/reference/current/getting-started-index.html).

### 3.1.2  Run Elasticsearch locally on Linux, macOS, or Windows

### 3.1.3  Talking to Elasticsearch with cURL commands

​	Most of the examples in this guide enable you to copy the appropriate cURL command and submit the request to your local Elasticsearch instance from the command line.

​	A request to Elasticsearch consists of the same parts as any HTTP request:

```java
curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'
```

This example uses the following variables:

- **`<VERB>`**

  The appropriate HTTP method or verb. For example, `GET`, `POST`, `PUT`, `HEAD`, or `DELETE`.

- **`<PROTOCOL>`**

  Either `http` or `https`. Use the latter if you have an HTTPS proxy in front of Elasticsearch or you use Elasticsearch security features to encrypt HTTP communications.

- **`<HOST>`**

  The hostname of any node in your Elasticsearch cluster. Alternatively, use `localhost` for a node on your local machine.

- **`<PORT>`**

  The port running the Elasticsearch HTTP service, which defaults to `9200`.

- **`<PATH>`**

  The API endpoint, which can contain multiple components, such as `_cluster/stats` or `_nodes/stats/jvm`.

- **`<QUERY_STRING>`**

  Any optional query-string parameters. For example, `?pretty` will *pretty-print* the JSON response to make it easier to read.

- **`<BODY>`**

  A JSON-encoded request body (if necessary).

​	If the Elasticsearch security features are enabled, you must also provide a valid user name (and password) that has authority to run the API. For example, use the `-u` or `--u` cURL command parameter. For details about which security privileges are required to run each API, see [REST APIs](https://www.elastic.co/guide/en/elasticsearch/reference/current/rest-apis.html).

​	Elasticsearch responds to each API request with an HTTP status code like `200 OK`. With the exception of `HEAD` requests, it also returns a JSON-encoded response body.

### 3.1.4 Other installation options

## 3.2 Index some documents

​	一旦集群启动并运行，就可以为一些数据编制索引了。Elasticsearch有多种摄取选项，但最终它们都做了相同的事情：将JSON文档放入Elasticsearch索引中。

​	<small>Once you have a cluster up and running, you’re ready to index some data. There are a variety of ingest options for Elasticsearch, but in the end they all do the same thing: put JSON documents into an Elasticsearch index.</small>

​	您可以通过一个简单的PUT请求直接执行此操作，该请求指定要添加文档的索引、唯一的文档ID以及请求正文中的一个或多个“field”：“value”对：

​	<small>You can do this directly with a simple PUT request that specifies the index you want to add the document, a unique document ID, and one or more `"field": "value"` pairs in the request body:</small>

```http
PUT /customer/_doc/1
{
  "name": "John Doe"
}
```

​	This request automatically creates the `customer` index if it doesn’t already exist, adds a new document that has an ID of `1`, and stores and indexes the `name` field.

​	Since this is a new document, the response shows that the result of the operation was that version 1 of the document was created:

```json
{
  "_index" : "customer",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 0,
  "_primary_term" : 1
}
```

​	The new document is available immediately from any node in the cluster. You can retrieve it with a GET request that specifies its document ID:

```http
GET /customer/_doc/1
```

​	The response indicates that a document with the specified ID was found and shows the original source fields that were indexed.

```json
{
  "_index" : "customer",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "_seq_no" : 0,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "name" : "John Doe"
  }
}
```

### 3.2.1 Indexing documents in bulk

​	如果有很多文档需要索引，可以使用bulk api批量提交它们。使用批量到批处理文档操作比单独提交请求要快得多，因为它最大限度地减少了网络往返。

​	<small>If you have a lot of documents to index, you can submit them in batches with the [bulk API](https://www.elastic.co/guide/en/elasticsearch/reference/7.9/docs-bulk.html). Using bulk to batch document operations is significantly faster than submitting requests individually as it minimizes network roundtrips.</small>

​	最佳批处理大小取决于许多因素：文档大小和复杂性、索引和搜索负载以及集群可用的资源。一个好的起点是批量处理1000到5000个文档，总负载在5MB到15MB之间。从那里，你可以尝试找到最合适的地方

​	<small>The optimal batch size depends on a number of factors: the document size and complexity, the indexing and search load, and the resources available to your cluster. A good place to start is with batches of 1,000 to 5,000 documents and a total payload between 5MB and 15MB. From there, you can experiment to find the sweet spot.</small>

To get some data into Elasticsearch that you can start searching and analyzing:

1. Download the [`accounts.json`](https://github.com/elastic/elasticsearch/blob/master/docs/src/test/resources/accounts.json?raw=true) sample data set. The documents in this randomly-generated data set represent user accounts with the following information:

   ```json
   {
     "account_number": 0,
     "balance": 16623,
     "firstname": "Bradshaw",
     "lastname": "Mckenzie",
     "age": 29,
     "gender": "F",
     "address": "244 Columbus Place",
     "employer": "Euron",
     "email": "bradshawmckenzie@euron.com",
     "city": "Hobucken",
     "state": "CO"
   }
   ```

2. Index the account data into the `bank` index with the following `_bulk` request:

   ```shell
   curl -H "Content-Type: application/json" -XPOST "localhost:9200/bank/_bulk?pretty&refresh" --data-binary "@accounts.json"
   curl "localhost:9200/_cat/indices?v"
   ```

3. The response indicates that 1,000 documents were indexed successfully.

   ```shell
   health status index                          uuid                   pri rep docs.count docs.deleted store.size pri.store.size
   yellow open   bank                           CGuHvnOIRtGbyBT3wvHh-Q   1   1       1000            0    382.2kb        382.2kb
   ```

## 3.3  Start searching

​	Once you have ingested some data into an Elasticsearch index, you can search it by sending requests to the `_search` endpoint. To access the full suite of search capabilities, you use the Elasticsearch Query DSL to specify the search criteria in the request body. You specify the name of the index you want to search in the request URI.

​	For example, the following request retrieves all documents in the `bank` index sorted by account number:

```http
GET /bank/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ]
}
```

​	By default, the `hits` section of the response includes the first 10 documents that match the search criteria:

```json
{
  "took" : 63,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
        "value": 1000,
        "relation": "eq"
    },
    "max_score" : null,
    "hits" : [ {
      "_index" : "bank",
      "_type" : "_doc",
      "_id" : "0",
      "sort": [0],
      "_score" : null,
      "_source" : {"account_number":0,"balance":16623,"firstname":"Bradshaw","lastname":"Mckenzie","age":29,"gender":"F","address":"244 Columbus Place","employer":"Euron","email":"bradshawmckenzie@euron.com","city":"Hobucken","state":"CO"}
    }, {
      "_index" : "bank",
      "_type" : "_doc",
      "_id" : "1",
      "sort": [1],
      "_score" : null,
      "_source" : {"account_number":1,"balance":39225,"firstname":"Amber","lastname":"Duke","age":32,"gender":"M","address":"880 Holmes Lane","employer":"Pyrami","email":"amberduke@pyrami.com","city":"Brogan","state":"IL"}
    }, ...
    ]
  }
}
```

The response also provides the following information about the search request:

- `took` – how long it took Elasticsearch to run the query, in **milliseconds**
- `timed_out` – whether or not the search request timed out
- `_shards` – how many shards were searched and a breakdown of how many shards succeeded, failed, or were skipped.
- `max_score` – the score of the most relevant document found
- `hits.total.value` - how many matching documents were found
- `hits.sort` - the document’s sort position (when not sorting by relevance score)
- `hits._score` - the document’s relevance score (not applicable when using `match_all`)

​	**Each search request is self-contained: Elasticsearch does not maintain any state information across requests.** **To page through the search hits, specify the `from` and `size` parameters in your request.**

​	For example, the following request gets hits 10 through 19:

```shell
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ],
  "from": 10,
  "size": 10
}
'
```

​	Now that you’ve seen how to submit a basic search request, you can start to construct queries that are a bit more interesting than `match_all`.

​	To search for specific terms within a field, you can use a `match` query. For example, the following request searches the `address` field to find customers whose addresses contain `mill` **or** `lane`:

```http
GET /bank/_search
{
  "query": { "match": { "address": "mill lane" } }
}
```

（注意：是查询含有"mill" 或者 "lane"的信息，只要有其中一个就会被加入到结果集中）

```http
GET /bank/_search
{
  "query": { "match_phrase": { "address": "mill lane" } }
}
```

​	To construct more complex queries, you can use a **`bool` query** to combine multiple query criteria. You can designate criteria as **required (must match), desirable (should match), or undesirable (must not match)**.

​	For example, the following request searches the `bank` index for accounts that belong to customers who are 40 years old, but excludes anyone who lives in Idaho (ID):

```http
GET /bank/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "age": "40" } }
      ],
      "must_not": [
        { "match": { "state": "ID" } }
      ]
    }
  }
}
```

​	Each `must`, `should`, and `must_not` element in a Boolean query is referred to as a query clause. How well a document meets the criteria in each `must` or `should` clause contributes to the document’s *relevance score*. The higher the score, the better the document matches your search criteria. By default, Elasticsearch returns documents ranked by these relevance scores.

​	**The criteria in a `must_not` clause is treated as a *filter*.** It affects whether or not the document is included in the results, but does not contribute to how documents are scored. You can also explicitly specify arbitrary filters to include or exclude documents based on structured data.

​	For example, the following request uses a **range filter** to limit the results to accounts with a balance between $20,000 and $30,000 (inclusive).

```http
GET /bank/_search
{
  "query": {
    "bool": {
      "must": { "match_all": {} },
      "filter": {
        "range": {
          "balance": {
            "gte": 20000,
            "lte": 30000
          }
        }
      }
    }
  }
}
```

Response:

```json
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 217,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "_doc",
        "_id" : "49",
        "_score" : 1.0,
        "_source" : {
          "account_number" : 49,
          "balance" : 29104,
          "firstname" : "Fulton",
          "lastname" : "Holt",
          "age" : 23,
          "gender" : "F",
          "address" : "451 Humboldt Street",
          "employer" : "Anocha",
          "email" : "fultonholt@anocha.com",
          "city" : "Sunriver",
          "state" : "RI"
        }
      },
      {
        "_index" : "bank",
        "_type" : "_doc",
        "_id" : "102",
        "_score" : 1.0,
        "_source" : {
          "account_number" : 102,
          "balance" : 29712,
          "firstname" : "Dena",
          "lastname" : "Olson",
          "age" : 27,
          "gender" : "F",
          "address" : "759 Newkirk Avenue",
          "employer" : "Hinway",
          "email" : "denaolson@hinway.com",
          "city" : "Choctaw",
          "state" : "NJ"
        }
      },
      //....
    ]
  }
}

```

## 3.4 Analyze results with aggregations

​	Elasticsearch aggregations enable you to get meta-information about your search results and answer questions like, "How many account holders are in Texas?" or "What’s the average balance of accounts in Tennessee?" You can search documents, filter hits, and use aggregations to analyze the results all in one request.

​	For example, the following request **uses a `terms` aggregation to group all of the accounts in the `bank` index by state**, and <u>returns the **ten** states with the most accounts **in descending order**</u>:

```http
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      }
    }
  }
}
```

The `buckets` in the response are the values of the `state` field. The `doc_count` shows the number of accounts in each state. For example, you can see that there are 27 accounts in `ID` (Idaho). **Because the request set `size=0`, the response only contains the aggregation results**.

```json
{
  "took": 29,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped" : 0,
    "failed": 0
  },
  "hits" : {
     "total" : {
        "value": 1000,
        "relation": "eq"
     },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "group_by_state" : {
      "doc_count_error_upper_bound": 20,
      "sum_other_doc_count": 770,
      "buckets" : [ {
        "key" : "ID",
        "doc_count" : 27
      }, {
        "key" : "TX",
        "doc_count" : 27
      }, {
        "key" : "AL",
        "doc_count" : 25
      }, {
        "key" : "MD",
        "doc_count" : 25
      }, {
        "key" : "TN",
        "doc_count" : 23
      }, {
        "key" : "MA",
        "doc_count" : 21
      }, {
        "key" : "NC",
        "doc_count" : 21
      }, {
        "key" : "ND",
        "doc_count" : 21
      }, {
        "key" : "ME",
        "doc_count" : 20
      }, {
        "key" : "MO",
        "doc_count" : 20
      } ]
    }
  }
}
```

​	You can combine aggregations to build more complex summaries of your data. For example, the following request **nests an `avg` aggregation within the previous `group_by_state` aggregation to calculate the average account balances for each state**.

```http
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
```

Response:

```json
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1000,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "group_by_state" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 743,
      "buckets" : [
        {
          "key" : "TX",
          "doc_count" : 30,
          "average_balance" : {
            "value" : 26073.3
          }
        },
        {
          "key" : "MD",
          "doc_count" : 28,
          "average_balance" : {
            "value" : 26161.535714285714
          }
        },
        // .... 
      ]
    }
  }
}

```

​	Instead of sorting the results by count, **you could sort using the result of the nested aggregation by specifying the order within the `terms` aggregation**:

```http
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword",
        "order": {
          "average_balance": "desc"
        }
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
```

Response：

```json
{
  "took" : 7,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1000,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "group_by_state" : {
      "doc_count_error_upper_bound" : -1,
      "sum_other_doc_count" : 827,
      "buckets" : [
        {
          "key" : "CO",
          "doc_count" : 14,
          "average_balance" : {
            "value" : 32460.35714285714
          }
        },
        {
          "key" : "NE",
          "doc_count" : 16,
          "average_balance" : {
            "value" : 32041.5625
          }
        },
        {
          "key" : "AZ",
          "doc_count" : 14,
          "average_balance" : {
            "value" : 31634.785714285714
          }
        },
        // ... 
      ]
    }
  }
}
```

​	In addition to basic bucketing and metrics aggregations like these, Elasticsearch provides specialized aggregations for operating on multiple fields and analyzing particular types of data such as dates, IP addresses, and geo data. You can also feed the results of individual aggregations into pipeline aggregations for further analysis.

​	**The core analysis capabilities provided by aggregations enable advanced features such as using machine learning to detect anomalies.**

## 3.5 where to go from here

Now that you’ve set up a cluster, indexed some documents, and run some searches and aggregations, you might want to:

- [Dive in to the Elastic Stack Tutorial](https://www.elastic.co/guide/en/elastic-stack-get-started/7.9/get-started-elastic-stack.html#install-kibana) to install Kibana, Logstash, and Beats and set up a basic system monitoring solution.
- [Load one of the sample data sets into Kibana](https://www.elastic.co/guide/en/kibana/7.9/add-sample-data.html) to see how you can use Elasticsearch and Kibana together to visualize your data.
- Try out one of the Elastic search solutions:
  - [Site Search](https://swiftype.com/documentation/site-search/crawler-quick-start)
  - [App Search](https://swiftype.com/documentation/app-search/getting-started)
  - [Enterprise Search](https://swiftype.com/documentation/enterprise-search/getting-started)

# 4. Set up Elasticsearch

### Support platforms

​	The matrix of officially supported operating systems and JVMs is available here: [Support Matrix](https://www.elastic.co/support/matrix). Elasticsearch is tested on the listed platforms, but it is possible that it will work on other platforms too.

###  Java (JVM) Version

​	Elasticsearch is built using Java, and includes a bundled version of [OpenJDK](https://openjdk.java.net/) from the JDK maintainers (GPLv2+CE) within each distribution. The bundled JVM is the recommended JVM and is located within the `jdk` directory of the Elasticsearch home directory.

​	To use your own version of Java, set the `JAVA_HOME` environment variable. If you must use a version of Java that is different from the bundled JVM, we recommend using a [supported](https://www.elastic.co/support/matrix) [LTS version of Java](https://www.oracle.com/technetwork/java/eol-135779.html). Elasticsearch will refuse to start if a known-bad version of Java is used. The bundled JVM directory may be removed when using your own JVM.

## 4.1 Installing Elasticsearch

### 4.1.6 Install Elasticsearch with Docker

​	Elasticsearch is also available as Docker images. The images use [centos:7](https://hub.docker.com/_/centos/) as the base image.

​	A list of all published Docker images and tags is available at [www.docker.elastic.co](https://www.docker.elastic.co/). The source files are in [Github](https://github.com/elastic/elasticsearch/blob/7.9/distribution/docker).

​	These images are free to use under the Elastic license. They contain open source and free commercial features and access to paid commercial features. [Start a 30-day trial](https://www.elastic.co/guide/en/kibana/7.9/managing-licenses.html) to try out all of the paid commercial features. See the [Subscriptions](https://www.elastic.co/subscriptions) page for information about Elastic license levels.

#### Pulling the image

​	Obtaining Elasticsearch for Docker is as simple as issuing a `docker pull` command against the Elastic Docker registry.

```shell
docker pull docker.elastic.co/elasticsearch/elasticsearch:7.9.3
```

​	Alternatively, you can download other Docker images that contain only features available under the Apache 2.0 license. To download the images, go to [www.docker.elastic.co](https://www.docker.elastic.co/).

#### Starting a single node cluster with Docker

​	To start a single-node Elasticsearch cluster for development or testing, specify [single-node discovery](https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html#single-node-discovery) to bypass the [bootstrap checks](https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html):

```shell
docker run -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.9.3
```

####  Starting a multi-node cluster with Docker Compose

​	To get a three-node Elasticsearch cluster up and running in Docker, you can use Docker Compose:

1. Create a `docker-compose.yml` file:

   ```yaml
   version: '2.2'
   services:
     es01:
       image: docker.elastic.co/elasticsearch/elasticsearch:7.9.3
       container_name: es01
       environment:
         - node.name=es01
         - cluster.name=es-docker-cluster
         - discovery.seed_hosts=es02,es03
         - cluster.initial_master_nodes=es01,es02,es03
         - bootstrap.memory_lock=true
         - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
       ulimits:
         memlock:
           soft: -1
           hard: -1
       volumes:
         - data01:/usr/share/elasticsearch/data
       ports:
         - 9200:9200
       networks:
         - elastic
     es02:
       image: docker.elastic.co/elasticsearch/elasticsearch:7.9.3
       container_name: es02
       environment:
         - node.name=es02
         - cluster.name=es-docker-cluster
         - discovery.seed_hosts=es01,es03
         - cluster.initial_master_nodes=es01,es02,es03
         - bootstrap.memory_lock=true
         - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
       ulimits:
         memlock:
           soft: -1
           hard: -1
       volumes:
         - data02:/usr/share/elasticsearch/data
       networks:
         - elastic
     es03:
       image: docker.elastic.co/elasticsearch/elasticsearch:7.9.3
       container_name: es03
       environment:
         - node.name=es03
         - cluster.name=es-docker-cluster
         - discovery.seed_hosts=es01,es02
         - cluster.initial_master_nodes=es01,es02,es03
         - bootstrap.memory_lock=true
         - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
       ulimits:
         memlock:
           soft: -1
           hard: -1
       volumes:
         - data03:/usr/share/elasticsearch/data
       networks:
         - elastic
   
   volumes:
     data01:
       driver: local
     data02:
       driver: local
     data03:
       driver: local
   
   networks:
     elastic:
       driver: bridge
   ```

   ​	This sample Docker Compose file brings up a three-node Elasticsearch cluster. Node `es01` listens on `localhost:9200` and `es02` and `es03` talk to `es01` over a Docker network.

   ​	Please note that this configuration exposes port 9200 on all network interfaces, and given how Docker manipulates `iptables` on Linux, this means that your Elasticsearch cluster is publically accessible, potentially ignoring any firewall settings. If you don’t want to expose port 9200 and instead use a reverse proxy, replace `9200:9200` with `127.0.0.1:9200:9200` in the docker-compose.yml file. Elasticsearch will then only be accessible from the host machine itself.

   ​	The [Docker named volumes](https://docs.docker.com/storage/volumes) `data01`, `data02`, and `data03` store the node data directories so the data persists across restarts. If they don’t already exist, `docker-compose` creates them when you bring up the cluster.

   ​	**Make sure Docker Engine is allotted at least 4GiB of memory. In Docker Desktop, you configure resource usage on the Advanced tab in Preference (macOS) or Settings (Windows).**

   ​	*Docker Compose is not pre-installed with Docker on Linux. See docs.docker.com for installation instructions: [Install Compose on Linux](https://docs.docker.com/compose/install)*

2. Run `docker-compose` to bring up the cluster:

   ```shell
   docker-compose up
   ```

3. Submit a `_cat/nodes` request to see that the nodes are up and running:

   ```shell
   curl -X GET "localhost:9200/_cat/nodes?v&pretty"
   ```

​	Log messages go to the console and are handled by the configured Docker logging driver. By default you can access logs with `docker logs`.

​	To stop the cluster, run `docker-compose down`. The data in the Docker volumes is preserved and loaded when you restart the cluster with `docker-compose up`. To **delete the data volumes** when you bring down the cluster, specify the `-v` option: `docker-compose down -v`.

####  Start a multi-node cluster with TLS enabled

​	See [Encrypting communications in an Elasticsearch Docker Container](https://www.elastic.co/guide/en/elasticsearch/reference/current/configuring-tls-docker.html) and [Run the Elastic Stack in Docker with TLS enabled](https://www.elastic.co/guide/en/elastic-stack-get-started/7.9/get-started-docker.html#get-started-docker-tls).

#### Using the Docker images in production

​	The following requirements and recommendations apply when running Elasticsearch in Docker in production.

##### Set `vm.max_map_count` to at least `262144`

> [linux参数之max_map_count](https://www.cnblogs.com/duanxz/p/3567068.html)
>
> <small>**max_map_count**文件包含限制一个进程可以拥有的VMA(虚拟内存区域)的数量。虚拟内存区域是一个连续的虚拟地址空间区域。在进程的生命周期中，每当程序尝试在内存中映射文件，链接到共享内存段，或者分配堆空间的时候，这些区域将被创建。调优这个值将限制进程可拥有VMA的数量。限制一个进程拥有VMA的总数可能导致应用程序出错，因为当进程达到了VMA上线但又只能释放少量的内存给其他的内核进程使用时，操作系统会抛出内存不足的错误。如果你的操作系统在NORMAL区域仅占用少量的内存，那么调低这个值可以帮助释放内存给内核用。</small>

​	The `vm.max_map_count` kernel setting must be set to at least `262144` for production use.

​	How you set `vm.max_map_count` depends on your platform:

+ Linux

  The `vm.max_map_count` setting should be set permanently in `/etc/sysctl.conf`:

  ```conf
  grep vm.max_map_count /etc/sysctl.conf
  vm.max_map_count=262144
  ```

  To apply the setting on a live system, run:

  ```shell
  sysctl -w vm.max_map_count=262144
  ```

+ macOS with [Docker for Mac](https://docs.docker.com/docker-for-mac) 

  The `vm.max_map_count` setting must be set within the xhyve virtual machine:

  1. From the command line, run:（我自己是下面这个步骤直接就出错了）

     ```shell
     screen ~/Library/Containers/com.docker.docker/Data/vms/0/
     ```

  2. Press enter and use`sysctl` to configure `vm.max_map_count`:

     ```shell
     sysctl -w vm.max_map_count=262144
     ```

  3. To exit the `screen` session, type `Ctrl a d`.

+ Windows and macOS with [Docker Desktop](https://www.docker.com/products/docker-desktop)

  The `vm.max_map_count` setting must be set via docker-machine:

  ```shell
  docker-machine ssh
  sudo sysctl -w vm.max_map_count=262144
  ```

+ Windows with [Docker Desktop WSL 2 backend](https://docs.docker.com/docker-for-windows/wsl)

  The `vm.max_map_count` setting must be set in the docker-desktop container:

  ```shell
  wsl -d docker-desktop
  sysctl -w vm.max_map_count=262144
  ```

##### Configuration files must be readable by the `elasticsearch` user

​	By default, Elasticsearch runs inside the container as user `elasticsearch` using uid:gid `1000:0`.

*One exception is [Openshift](https://docs.openshift.com/container-platform/3.6/creating_images/guidelines.html#openshift-specific-guidelines), which runs containers using an arbitrarily assigned user ID. Openshift presents persistent volumes with the gid set to `0`, which works without any adjustments.*

​	If you are bind-mounting a local directory or file, it must be readable by the `elasticsearch` user. In addition, this user must have write access to the [data and log dirs](https://www.elastic.co/guide/en/elasticsearch/reference/current/path-settings.html). A good strategy is to grant group access to gid `0` for the local directory.

​	For example, to prepare a local directory for storing data through a bind-mount:

```shell
mkdir esdatadir
chmod g+rwx esdatadir
chgrp 0 esdatadir
```

​	As a last resort, you can force the container to mutate the ownership of any bind-mounts used for the [data and log dirs](https://www.elastic.co/guide/en/elasticsearch/reference/current/path-settings.html) through the environment variable `TAKE_FILE_OWNERSHIP`. When you do this, they will be owned by uid:gid `1000:0`, which provides the required read/write access to the Elasticsearch process.

##### Increase ulimits for nofile and nproc

​	Increased ulimits for [nofile](https://www.elastic.co/guide/en/elasticsearch/reference/current/setting-system-settings.html) and [nproc](https://www.elastic.co/guide/en/elasticsearch/reference/current/max-number-threads-check.html) must be available for the Elasticsearch containers. Verify the [init system](https://github.com/moby/moby/tree/ea4d1243953e6b652082305a9c3cda8656edab26/contrib/init) for the Docker daemon sets them to acceptable values.

​	To check the Docker daemon defaults for ulimits, run:

```shell
docker run --rm centos:7 /bin/bash -c 'ulimit -Hn && ulimit -Sn && ulimit -Hu && ulimit -Su'
```

​	If needed, adjust them in the Daemon or override them per container. For example, when using `docker run`, set:

```shell
--ulimit nofile=65535:65535
```

#####  Disable swapping

​	Swapping needs to be disabled for performance and node stability. For information about ways to do this, see [Disable swapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-configuration-memory.html).

​	If you opt for the `bootstrap.memory_lock: true` approach, you also need to define the `memlock: true` ulimit in the [Docker Daemon](https://docs.docker.com/engine/reference/commandline/dockerd/#default-ulimits), or explicitly set for the container as shown in the [sample compose file](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#docker-compose-file). When using `docker run`, you can specify:

```shell
-e "bootstrap.memory_lock=true" --ulimit memlock=-1:-1
```

#####  Randomize published ports

​	**The image [exposes](https://docs.docker.com/engine/reference/builder/#/expose) TCP ports 9200 and 9300. For production clusters, randomizing the published ports with `--publish-all` is recommended, unless you are pinning one container per host.**

#####  Set the heap size

​	To configure the heap size, you can bind mount a [JVM options](https://www.elastic.co/guide/en/elasticsearch/reference/current/jvm-options.html) file under `/usr/share/elasticsearch/config/jvm.options.d` that includes your desired [heap size](https://www.elastic.co/guide/en/elasticsearch/reference/current/heap-size.html) settings. Note that while the default root `jvm.options` file sets a default heap of 1 GB, <u>any value you set in a bind-mounted JVM options file will override it.</u>

​	While setting the heap size via bind-mounted JVM options is the recommended method, you can also configure this by using the `ES_JAVA_OPTS` environment variable to set the heap size. For example, to use 16 GB, specify `-e ES_JAVA_OPTS="-Xms16g -Xmx16g"` with `docker run`. Note that while the default root `jvm.options` file sets a default heap of 1 GB, any value you set in `ES_JAVA_OPTS` will override it. The `docker-compose.yml` file above sets the heap size to 512 MB.

**You must [configure the heap size](https://www.elastic.co/guide/en/elasticsearch/reference/current/heap-size.html) even if you are [limiting memory access](https://docs.docker.com/config/containers/resource_constraints/#limit-a-containers-access-to-memory) to the container**.

##### Pin deployments to a specific image version

​	**Pin your deployments to a specific version** of the Elasticsearch Docker image. For example `docker.elastic.co/elasticsearch/elasticsearch:7.9.3`.

#####  Always bind data volumes

​	You should use a volume bound on `/usr/share/elasticsearch/data` for the following reasons:

1. The data of your Elasticsearch node won’t be lost if the container is killed
2. Elasticsearch is I/O sensitive and the Docker storage driver is not ideal for fast I/O
3. It allows the use of advanced [Docker volume plugins](https://docs.docker.com/engine/extend/plugins/#volume-plugins)

##### Avoid using `loop-lvm` mode

​	If you are using the devicemapper storage driver, do not use the default `loop-lvm` mode. Configure docker-engine to use [direct-lvm](https://docs.docker.com/engine/userguide/storagedriver/device-mapper-driver/#configure-docker-with-devicemapper).

#####  Centralize your logs

​	Consider centralizing your logs by using a different [logging driver](https://docs.docker.com/engine/admin/logging/overview/). Also note that the default json-file logging driver is not ideally suited for production use.

####  Configuring Elasticsearch with Docker

​	When you run in Docker, the [Elasticsearch configuration files](https://www.elastic.co/guide/en/elasticsearch/reference/current/settings.html#config-files-location) are loaded from `/usr/share/elasticsearch/config/`.

​	To use custom configuration files, you [bind-mount the files](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#docker-config-bind-mount) over the configuration files in the image.

​	You can set individual Elasticsearch configuration parameters using Docker environment variables. The [sample compose file](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#docker-compose-file) and the [single-node example](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#docker-cli-run-dev-mode) use this method.

​	To use the contents of a file to set an environment variable, suffix the environment variable name with `_FILE`. This is useful for passing secrets such as passwords to Elasticsearch without specifying them directly.

​	For example, to set the Elasticsearch bootstrap password from a file, you can bind mount the file and set the `ELASTIC_PASSWORD_FILE` environment variable to the mount location. If you mount the password file to `/run/secrets/password.txt`, specify:

```dockerfile
-e ELASTIC_PASSWORD_FILE=/run/secrets/bootstrapPassword.txt
```

​	You can also override the default command for the image to pass Elasticsearch configuration parameters as command line options. For example:

```shell
docker run <various parameters> bin/elasticsearch -Ecluster.name=mynewclustername
```

​	While bind-mounting your configuration files is usually the preferred method in production, you can also [create a custom Docker image](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#_c_customized_image) that contains your configuration.

####  Mounting Elasticsearch configuration files

​	Create custom config files and bind-mount them over the corresponding files in the Docker image. For example, to bind-mount `custom_elasticsearch.yml` with `docker run`, specify:

```dockerfile
-v full_path_to/custom_elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
```

*The container **runs Elasticsearch as user `elasticsearch` using uid:gid `1000:0`**. Bind mounted host directories and files must be accessible by this user, and the data and log directories must be writable by this user.*

####  Mounting an Elasticsearch keystore

​	**By default, Elasticsearch will auto-generate a keystore file for secure settings**. This file is obfuscated but not encrypted. If you want to encrypt your [secure settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/secure-settings.html) with a password, you must use the `elasticsearch-keystore` utility to create a password-protected keystore and bind-mount it to the container as `/usr/share/elasticsearch/config/elasticsearch.keystore`. In order to provide the Docker container with the password at startup, set the Docker environment value `KEYSTORE_PASSWORD` to the value of your password. For example, a `docker run` command might have the following options:

```dockerfile
-v full_path_to/elasticsearch.keystore:/usr/share/elasticsearch/config/elasticsearch.keystore
-E KEYSTORE_PASSWORD=mypassword
```

####  Using custom Docker images

​	In some environments, it might make more sense to prepare a custom image that contains your configuration. A `Dockerfile` to achieve this might be as simple as:

```dockerfile
FROM docker.elastic.co/elasticsearch/elasticsearch:7.9.3
COPY --chown=elasticsearch:elasticsearch elasticsearch.yml /usr/share/elasticsearch/config/
```

​	You could then build and run the image with:

```shell
docker build --tag=elasticsearch-custom .
docker run -ti -v /usr/share/elasticsearch/data elasticsearch-custom
```

Some plugins require additional security permissions. You must explicitly accept them either by:

+ Attaching a `tty` when you run the Docker image and allowing the permissions when prompted.
+ Inspecting the security permissions and accepting them (if appropriate) by adding the `--batch` flag to the plugin install command.

See [Plugin management](https://www.elastic.co/guide/en/elasticsearch/plugins/7.9/_other_command_line_parameters.html) for more information.

####  Next steps

​	You now have a test Elasticsearch environment set up. Before you start serious development or go into production with Elasticsearch, you must do some additional setup:

- Learn how to [configure Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/settings.html).
- Configure [important Elasticsearch settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/important-settings.html).
- Configure [important system settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/system-config.html).

## 4.2 Configuring Elasticsearch

## 4.3 important Elasticsearch configuration

## 4.4 important System Configuration

## 4.5 Bootstrap Checks

## 4.6 Bootstrap Checks for X-Pack

## 4.7 Starting Elasticsearch

## 4.8 Stopping Elasticsearch

## 4.9 Discovery and cluster formation

## 4.10 Add and remove nodes in your cluster

## 4.11 Full-cluster restart and rolling restart

## 4.12 Remote clusters

## 4.13 Set up X-Pack

## 4.14 Configuring X-Pack Java Clients

## 4.15 Plugins

​	Plugins are a way to enhance the basic Elasticsearch functionality in a custom manner. They range from adding custom mapping types, custom analyzers (in a more built in fashion), custom script engines, custom discovery and more.

​	For information about selecting and installing plugins, see [Elasticsearch Plugins and Integrations](https://www.elastic.co/guide/en/elasticsearch/plugins/7.9/index.html).

​	For information about developing your own plugin, see [Help for plugin authors](https://www.elastic.co/guide/en/elasticsearch/plugins/7.9/plugin-authors.html).

# 5. Upgrade Elasticsearch

## 5.1 Rolling upgrades

## 5.2 Full cluster restart upgrade

## 5.3 Reindex before upgrading

# 6. Index modules

## 6.1 Analysis

The index analysis module acts as a configurable registry of *analyzers* that can be used in order to convert a string field into individual terms which are:

- added to the inverted index in order to make the document searchable
- used by high level queries such as the [`match` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html) to generate search terms.

See [Text analysis](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html) for configuration details.

## 6.2 Index Shard Allocation

### 6.2.1  Index-level shard allocation filtering

​	You can use shard allocation filters to control where Elasticsearch allocates shards of a particular index. These per-index filters are applied in conjunction with [cluster-wide allocation filtering](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-cluster.html#cluster-shard-allocation-filtering) and [allocation awareness](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-cluster.html#shard-allocation-awareness).

​	分片分配过滤器可以基于自定义节点属性，也可以基于内置的“名称”、“主机”或“发布”ip、“ip”、“主机”和“id”属性。索引生命周期管理使用基于自定义节点属性的过滤器来确定在阶段之间移动时如何重新分配分片。

​	<small>Shard allocation filters can be based on custom node attributes or the built-in `_name`, `_host_ip`, `_publish_ip`, `_ip`, `_host` and `_id` attributes. [Index lifecycle management](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html) uses filters based on custom node attributes to determine how to reallocate shards when moving between phases.</small>

​	The `cluster.routing.allocation` settings are dynamic, enabling live indices to be moved from one set of nodes to another. Shards are only relocated if it is possible to do so without breaking another routing constraint, such as never allocating a primary and replica shard on the same node.

​	例如，可以使用自定义节点属性指示节点的性能特征，并使用碎片分配过滤将特定索引的碎片路由到最合适的硬件类。

​	<small>For example, you could use a custom node attribute to indicate a node’s performance characteristics and use shard allocation filtering to route shards for a particular index to the most appropriate class of hardware.</small>

#### 6.2.1.1 Enabling index-level shard allocation filtering

To filter based on a custom node attribute:

1. Specify the filter characteristics with a custom node attribute in each node’s `elasticsearch.yml` configuration file. For example, if you have `small`, `medium`, and `big` nodes, you could add a `size` attribute to filter based on node size.

   (在每个节点的elasticsearch.yml配置文件中使用自定义节点属性指定过滤器特征。例如，如果有小型、中型和大型节点，则可以添加一个size属性以根据节点大小进行筛选。)

   ```yaml
   node.attr.size: medium
   ```

   You can also set custom attributes when you start a node:

   ```shell
   `./bin/elasticsearch -Enode.attr.size=medium
   ```

2. Add a routing allocation filter to the index. The `index.routing.allocation` settings support three types of filters: `include`, `exclude`, and `require`. For example, to tell Elasticsearch to allocate shards from the `test` index to either `big` or `medium` nodes, use `index.routing.allocation.include`:

   ```http
   PUT test/_settings
   {
     "index.routing.allocation.include.size": "big,medium"
   }
   ```

   If you specify multiple filters, all conditions must be satisfied for shards to be relocated. For example, to move the `test` index to `big` nodes in `rack1`, you could specify:

   (如果指定多个过滤器，则必须满足所有条件才能重新定位碎片。例如，要将测试索引移动到rack1中的大节点，可以指定：)

   ```http
   PUT test/_settings
   {
     "index.routing.allocation.include.size": "big",
     "index.routing.allocation.include.rack": "rack1"
   }
   ```

#### 6.2.1.2 Index allocation filter settings

- **`index.routing.allocation.include.{attribute}`**

  Assign the index to a node whose `{attribute}` has **at least one** of the comma-separated values.

  <small>(将索引分配给其{attribute}至少有一个逗号分隔值的节点。)</small>

- **`index.routing.allocation.require.{attribute}`**

  Assign the index to a node whose `{attribute}` has ***all*** of the comma-separated values.

- **`index.routing.allocation.exclude.{attribute}`**

  Assign the index to a node whose `{attribute}` has ***none*** of the comma-separated values.

The index allocation settings support the following built-in attributes:

| `_name`       | Match nodes by node name                                     |
| ------------- | ------------------------------------------------------------ |
| `_host_ip`    | Match nodes by host IP address (IP associated with hostname) |
| `_publish_ip` | Match nodes by publish IP address                            |
| `_ip`         | Match either `_host_ip` or `_publish_ip`                     |
| `_host`       | Match nodes by hostname                                      |
| `_id`         | Match nodes by node id                                       |

You can use wildcards（通配符） when specifying attribute values, for example:

```http
PUT test/_settings
{
  "index.routing.allocation.include._ip": "192.168.2.*"
}
```

### 6.2.2 Delaying allocation when a node leaves

（节点离开时延迟分配）

​	When a node leaves the cluster for whatever reason, intentional（故意的） or otherwise, the master reacts by:

- Promoting a replica shard to primary to replace any primaries that were on the node.

  <small>（将副本碎片升级为主碎片以替换节点上的所有主碎片）</small>

- Allocating replica shards to replace the missing replicas (assuming there are enough nodes).

  <small>(分配副本碎片来替换丢失的副本（假设有足够的节点）)</small>

- Rebalancing shards evenly across the remaining nodes.

  <small>(在剩余节点上均匀地重新平衡碎片。)</small>

​	These actions are intended to protect the cluster against data loss by ensuring that every shard is fully replicated as soon as possible.

​	(这些操作旨在通过确保尽快完全复制每个碎片来保护集群不受数据丢失的影响。)

​	尽管我们在节点级别和集群级别都(节流)限制了并发恢复，但是这种“碎片洗牌”仍然会给集群带来很多额外的负载，特别是如果丢失的节点很快就重新返回，那么这些负载很明显是不必要的。想象一下这个场景：

​	<small>Even though we throttle concurrent recoveries both at the [node level](https://www.elastic.co/guide/en/elasticsearch/reference/current/recovery.html) and at the [cluster level](https://www.elastic.co/guide/en/elasticsearch/reference/current/shards-allocation.html), this “shard-shuffle” can still put a lot of extra load on the cluster which may not be necessary if the missing node is likely to return soon. Imagine this scenario:</small>

- Node 5 loses network connectivity.

  节点5失去网络连接。

- The master promotes a replica shard to primary for each primary that was on Node 5.

  主节点将节点5上的每个主节点的副本碎片升级到主节点。

- The master allocates new replicas to other nodes in the cluster.

  主节点将新副本分配给群集中的其他节点。

- Each new replica makes an entire copy of the primary shard across the network.

  每个新的复制副本都会在网络上生成主碎片的完整副本。

- More shards are moved to different nodes to rebalance the cluster.

  更多的碎片被移动到不同的节点以重新平衡集群。

- Node 5 returns after a few minutes.

  5分钟后返回节点。

- The master rebalances the cluster by allocating shards to Node 5.

  主节点通过向节点5分配碎片来重新平衡集群。

​	如果主机只是等了几分钟，那么丢失的碎片就可以以最小的网络流量重新分配给节点5。对于已自动同步刷新的空闲碎片（未接收索引请求的碎片），此过程将更快。

​	<small>If the master had just waited for a few minutes, then the missing shards could have been re-allocated to Node 5 with the minimum of network traffic. This process would be even quicker for idle shards (shards not receiving indexing requests) which have been automatically [sync-flushed](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-synced-flush-api.html).</small>

​	The allocation of replica shards which become unassigned because a node has left can be delayed with the `index.unassigned.node_left.delayed_timeout` dynamic setting, which defaults to `1m`.

​	<small>由于节点已离开而未分配的副本碎片的分配可以使用index.unassigned.node_左.delayed_timeout动态设置，默认为1m。</small>

​	This setting can be updated on a live index (or on all indices):

```http
PUT _all/_settings
{
  "settings": {
    "index.unassigned.node_left.delayed_timeout": "5m"
  }
}
```

With delayed allocation enabled, the above scenario changes to look like this:

- Node 5 loses network connectivity.

- The master promotes a replica shard to primary for each primary that was on Node 5.

- The master logs a message that allocation of unassigned shards has been delayed, and for how long.

  主机会记录一条消息，说明未分配碎片的分配已被延迟，以及延迟了多长时间。

- The cluster remains yellow because there are unassigned replica shards.

- Node 5 returns after a few minutes, before the `timeout` expires.

- The missing replicas are re-allocated to Node 5 (and sync-flushed shards recover almost immediately).

  丢失的副本被重新分配到节点5（并且同步刷新的碎片几乎立即恢复）。

*NOTE：This setting will not affect the promotion of replicas to primaries, nor will it affect the assignment of replicas that have not been assigned previously. In particular, delayed allocation does not come into effect after a full cluster restart. Also, in case of a master failover situation, elapsed delay time is forgotten (i.e. reset to the full initial delay).*

*此设置不会影响将副本升级到主副本，也不会影响以前未分配的副本的分配。特别是，延迟分配在完全重启集群后不会生效。此外，在主故障转移情况下，已用延迟时间会被忽略（即重置为完全初始延迟）。*

#### 6.2.2.1 Cancellation of shard relocation

​	如果延迟分配超时，主机会将丢失的碎片分配给另一个将开始恢复的节点。如果丢失的节点重新加入集群，并且其碎片仍具有与主节点相同的同步id，则将取消碎片重新定位，而将使用同步的碎片进行恢复。

​	<small>If delayed allocation times out, the master assigns the missing shards to another node which will start recovery. If the missing node rejoins the cluster, and its shards still have the same sync-id as the primary, shard relocation will be cancelled and the synced shard will be used for recovery instead.</small>

​	因此，默认超时设置为一分钟：即使开始重新定位碎片，取消恢复以支持同步碎片的代价也很低。

​	<small>For this reason, the default `timeout` is set to just one minute: even if shard relocation begins, cancelling recovery in favour of the synced shard is cheap.</small>

#### 6.2.2.2  Monitoring delayed unassigned shards

​	可以使用群集运行状况API查看由于此超时设置而延迟分配的碎片数：

​	The number of shards whose allocation has been delayed by this timeout setting can be viewed with the [cluster health API](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-health.html):

```http
GET _cluster/health
```

 This request will return a `delayed_unassigned_shards` value.

#### 6.2.2.3  Removing a node permanently

​	If a node is not going to return and you would like Elasticsearch to allocate the missing shards immediately, just update the timeout to zero:

```http
PUT _all/_settings
{
  "settings": {
    "index.unassigned.node_left.delayed_timeout": "0"
  }
}
```

​	You can reset the timeout as soon as the missing shards have started to recover.

​	(一旦丢失的碎片开始恢复，就可以重置超时。)

### 6.2.3  Index recovery prioritization

​	Unallocated shards are recovered in order of priority, whenever possible. Indices are sorted into priority order as follows:

- the optional `index.priority` setting (higher before lower)
- the index creation date (higher before lower)
- the index name (higher before lower)

​	**This means that, by default, newer indices will be recovered before older indices**.

​	Use the per-index dynamically updatable `index.priority` setting to customise the index prioritization order. For instance:

```http
PUT index_1

PUT index_2

PUT index_3
{
  "settings": {
    "index.priority": 10
  }
}

PUT index_4
{
  "settings": {
    "index.priority": 5
  }
}
```

In the above example:

- `index_3` will be recovered first because it has the highest `index.priority`.
- `index_4` will be recovered next because it has the next highest priority.
- `index_2` will be recovered next because it was created more recently.
- `index_1` will be recovered last.

This setting accepts an integer, and can be updated on a live index with the [update index settings API](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-update-settings.html):

```http
PUT index_4/_settings
{
  "index.priority": 1
}
```

### 6.2.4 Total shards per node

​	集群级shard分配器尝试将单个索引的碎片分布到尽可能多的节点上。然而，根据你有多少碎片和索引，以及它们有多大，可能并不总是能够均匀地分布碎片。

​	<small>The cluster-level shard allocator tries to spread the shards of a single index across as many nodes as possible. However, depending on how many shards and indices you have, and how big they are, it may not always be possible to spread shards evenly.</small>

​	通过以下动态设置，可以为每个节点允许的单个索引中的碎片总数指定硬限制：

​	<small>The following *dynamic* setting allows you to specify a hard limit on the total number of shards from a single index allowed per node:</small>

- **`index.routing.allocation.total_shards_per_node`**

  The maximum number of shards (replicas and primaries) that will be allocated to a single node. Defaults to unbounded.

You can also limit the amount of shards a node can have regardless of the index:

- **`cluster.routing.allocation.total_shards_per_node`**

  The maximum number of shards (replicas and primaries) that will be allocated to a single node globally. Defaults to unbounded (-1).

*WARNNING：These settings impose a hard limit which can result in some shards not being allocated.Use with caution.*

## 6.3 Index blocks

​	Index blocks limit the kind of operations that are available on a certain index. The blocks come in different flavours, allowing to block write, read, or metadata operations. The blocks can be set / removed using dynamic index settings, or can be added using a dedicated API, which also ensures for write blocks that, once successfully returning to the user, all shards of the index are properly accounting for the block, for example that all in-flight writes to an index have been completed after adding the write block.

​	<small>索引块限制对某个索引可用的操作类型。块有不同的风格，允许块写、读或元数据操作。可以使用动态索引设置来设置/删除这些块，也可以使用专用的API来添加这些块，这也可以确保一旦成功地返回给用户，索引的所有碎片都会正确地说明该块，例如，在添加写块之后，对索引的所有动态写入都已完成。</small>

### 6.3.1  Index block settings

The following *dynamic* index settings determine the blocks present on an index：

- **`index.blocks.read_only`**

  Set to `true` to make the index and index metadata read only, `false` to allow writes and metadata changes.

- **`index.blocks.read_only_allow_delete`**

  Similar to `index.blocks.read_only`, but also allows deleting the index to make more resources available. The [disk-based shard allocator](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-cluster.html#disk-based-shard-allocation) may add and remove this block automatically.

  <small>类似`index.blocks.read_only`，但也允许删除索引以获得更多资源。基于磁盘的shard分配器可以自动添加和删除此块</small>

  Deleting documents from an index to release resources - rather than deleting the index itself - can increase the index size over time. When `index.blocks.read_only_allow_delete` is set to `true`, deleting documents is not permitted. However, deleting the index itself releases the read-only index block and makes resources available almost immediately.

*IMPORTANT：Elasticsearch adds and removes the read-only index block automatically when the disk utilization falls below the high watermark, controlled by [cluster.routing.allocation.disk.watermark.flood_stage](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-cluster.html#cluster-routing-flood-stage).*

- **`index.blocks.read`**

  Set to `true` to disable read operations against the index.

- **`index.blocks.write`**

  Set to `true` to disable data write operations against the index. **Unlike `read_only`, this setting does not affect metadata**. For instance, you can close an index with a `write` block, but you cannot close an index with a `read_only` block.

- **`index.blocks.metadata`**

  Set to `true` to disable index metadata reads and writes.

### 6.3.2  Add index block API

Adds an index block to an index.

```http
PUT /my-index-000001/_block/write
```

#### 6.3.2.1 Request

```http
PUT /<index>/_block/<block>
```

#### 6.3.2.2  Path parameters

+ **`<index>`**

  (Optional, string) Comma-separated list or wildcard expression of index names used to limit the request.

  To add blocks to all indices, use `_all` or `*`. To disallow the adding of blocks to indices with `_all` or wildcard expressions, change the `action.destructive_requires_name` cluster setting to `true`. You can update this setting in the `elasticsearch.yml` file or using the [cluster update settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-update-settings.html) API.

+ **`<block>`**

  (Required, string) Block type to add to the index.

  Valid values for `<block>`：

  + **`metadata`**

    Disable metadata changes, such as closing the index.

  + **`read`**

    Disable read operations.

  + **`read_only`**

    Disable write operations and metadata changes.

  + **`write`**

    Disable write operations. However, metadata changes are still allowed.

### 6.3.3  Query parameters

+ **`allow_no_indices`**

  (Optional, Boolean) If `false`, the request returns an error when a wildcard expression, [index alias](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-aliases.html), or `_all` value targets only missing or closed indices.

  Defaults to `true`.

+ **`expand_wildcards`**

  (Optional, string) Controls what kind of indices that wildcard expressions can expand to. Multiple values are accepted when separated by a comma, as in `open,hidden`. Valid values are:

  + **`all`**

    Expand to open and closed indices, including [hidden indices](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html#index-hidden).

  + **`open`**

    Expand only to open indices.

  + **`closed`**

    Expand only to closed indices.

  + **`hidden`**

    Expansion of wildcards will include [hidden indices](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html#index-hidden). Must be combined with `open`, `closed`, or both.

  + **`none`**

    Wildcard expressions are not accepted.

  Defaults to `open`.

+ **`ignore_unavailable`**

  (Optional, Boolean) If `true`, missing or closed indices are not included in the response. Defaults to `false`.

+ **`master_timeout`**

  (Optional, [time units](https://www.elastic.co/guide/en/elasticsearch/reference/current/common-options.html#time-units)) Specifies the period of time to wait for a connection to the master node. If no response is received before the timeout expires, the request fails and returns an error. Defaults to `30s`.

+ **`timeout`**

  (Optional, [time units](https://www.elastic.co/guide/en/elasticsearch/reference/current/common-options.html#time-units)) Specifies the period of time to wait for a response. If no response is received before the timeout expires, the request fails and returns an error. Defaults to `30s`.

### 6.3.4  Examples

The following example shows how to add an index block:

```http
PUT /my-index-000001/_block/write
```

The API returns following response:

```json
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "indices" : [ {
    "name" : "my-index-000001",
    "blocked" : true
  } ]
}
```

## 6.4 Mapper

​	mapper模块充当在创建索引或使用put mapping api时添加到索引的类型映射定义的注册表。它还处理对没有预定义显式映射的类型的动态映射支持。有关映射定义的更多信息，请查看映射部分。

​	<small>The mapper module acts as a registry for the type mapping definitions added to an index either when creating it or by using the put mapping api. It also handles the dynamic mapping support for types that have no explicit mappings pre defined. For more information about mapping definitions, check out the [mapping section](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html).</small>

## 6.5 Merge

​	A shard in Elasticsearch is a Lucene index, and a Lucene index is broken down into segments. Segments are internal storage elements in the index where the index data is stored, and are immutable. Smaller segments are periodically merged into larger segments to keep the index size at bay and to expunge deletes.

​	<small>Elasticsearch中的shard是Lucene索引，Lucene索引被分解成多个片段。段是索引中存储索引数据的内部存储元素，是不可变的。...</small>

​	The merge process uses auto-throttling to balance the use of hardware resources between merging and other activities like search.
​	<small>合并过程使用自动调节来平衡合并和其他活动（如搜索）之间硬件资源的使用。</small>

### 6.5.1  Merge scheduling

​	合并调度程序（ConcurrentMergeScheduler）在需要时控制合并操作的执行。合并在不同的线程中运行，当达到最大线程数时，进一步的合并将等待直到合并线程可用。

​	<small>The merge scheduler (ConcurrentMergeScheduler) controls the execution of merge operations when they are needed. Merges run in separate threads, and when the maximum number of threads is reached, further merges will wait until a merge thread becomes available.</small>

​	The merge scheduler supports the following *dynamic* setting:

- **`index.merge.scheduler.max_thread_count`**

  The maximum number of threads on a single shard that may be merging at once. Defaults to `Math.max(1, Math.min(4, <<node.processors, node.processors>> / 2))` which works well for a good solid-state-disk (SSD). If your index is on spinning platter drives instead, decrease this to 1.

## 6.6 Similarity module

​	相似性（评分/排名模型）定义了匹配文档的评分方式。相似性是每个字段的，这意味着通过映射可以定义每个字段的不同相似性。

​	<small>A similarity (scoring / ranking model) defines how matching documents are scored. Similarity is per field, meaning that via the mapping one can define a different similarity per field.</small>

​	配置自定义相似性被认为是一个专家特性，并且内部相似性很可能已经足够了，如相似性中所述。

​	<small>Configuring a custom similarity is considered an expert feature and the builtin similarities are most likely sufficient as is described in [`similarity`](https://www.elastic.co/guide/en/elasticsearch/reference/current/similarity.html).</small>

### 6.6.1  Configuring a similarity

​	Most existing or custom Similarities have configuration options which can be configured via the index settings as shown below. The index options can be provided when creating an index or updating index settings.

```http
PUT /index
{
  "settings": {
    "index": {
      "similarity": {
        "my_similarity": {
          "type": "DFR",
          "basic_model": "g",
          "after_effect": "l",
          "normalization": "h2",
          "normalization.h2.c": "3.0"
        }
      }
    }
  }
}
```

​	Here we configure the DFR similarity so it can be referenced as `my_similarity` in mappings as is illustrate in the below example:

```http
PUT /index/_mapping
{
  "properties" : {
    "title" : { "type" : "text", "similarity" : "my_similarity" }
  }
}
```

### 6.6.2  Available similarities

####  6.6.2.1 BM25 similarity (**default**)

​	TF/IDF based similarity that has built-in tf normalization and is supposed to work better for short fields (like names). See [Okapi_BM25](https://en.wikipedia.org/wiki/Okapi_BM25) for more details. This similarity has the following options:

| `k1`                | Controls non-linear term frequency normalization (saturation). The default value is `1.2`. |
| ------------------- | ------------------------------------------------------------ |
| `b`                 | Controls to what degree document length normalizes tf values. The default value is `0.75`. |
| `discount_overlaps` | Determines whether overlap tokens (Tokens with 0 position increment) are ignored when computing norm. By default this is true, meaning overlap tokens do not count when computing norms. |

Type name: `BM25`

####  6.6.2.2 DFR similarity

​	Similarity that implements the [divergence from randomness](https://lucene.apache.org/core/8_6_2/core/org/apache/lucene/search/similarities/DFRSimilarity.html) framework. This similarity has the following options:

| `basic_model`   | Possible values: [`g`](https://lucene.apache.org/core/8_6_2/core/org/apache/lucene/search/similarities/BasicModelG.html), [`if`](https://lucene.apache.org/core/8_6_2/core/org/apache/lucene/search/similarities/BasicModelIF.html), [`in`](https://lucene.apache.org/core/8_6_2/core/org/apache/lucene/search/similarities/BasicModelIn.html) and [`ine`](https://lucene.apache.org/core/8_6_2/core/org/apache/lucene/search/similarities/BasicModelIne.html). |
| --------------- | ------------------------------------------------------------ |
| `after_effect`  | Possible values: [`b`](https://lucene.apache.org/core/8_6_2/core/org/apache/lucene/search/similarities/AfterEffectB.html) and [`l`](https://lucene.apache.org/core/8_6_2/core/org/apache/lucene/search/similarities/AfterEffectL.html). |
| `normalization` | Possible values: [`no`](https://lucene.apache.org/core/8_6_2/core/org/apache/lucene/search/similarities/Normalization.NoNormalization.html), [`h1`](https://lucene.apache.org/core/8_6_2/core/org/apache/lucene/search/similarities/NormalizationH1.html), [`h2`](https://lucene.apache.org/core/8_6_2/core/org/apache/lucene/search/similarities/NormalizationH2.html), [`h3`](https://lucene.apache.org/core/8_6_2/core/org/apache/lucene/search/similarities/NormalizationH3.html) and [`z`](https://lucene.apache.org/core/8_6_2/core/org/apache/lucene/search/similarities/NormalizationZ.html). |

​	All options but the first option need a normalization value.

​	Type name: `DFR`

#### 6.6.2.3 DFI similarity

​	Similarity that implements the [divergence from independence](https://trec.nist.gov/pubs/trec21/papers/irra.web.nb.pdf) model. This similarity has the following options:

<table>
  <tr>
  	<td>independence_measure</td>
    <td>Possible values standardized, saturated, chisquared.</td>
  </tr>
</table>

​	When using this similarity, it is highly recommended **not** to remove stop words to get good relevance. Also beware that terms whose frequency is less than the expected frequency will get a score equal to 0.

Type name: `DFI`

#### 6.6.2.4  IB similarity

​	[Information based model](https://lucene.apache.org/core/8_6_2/core/org/apache/lucene/search/similarities/IBSimilarity.html) . The algorithm is based on the concept that the information content in any symbolic *distribution* sequence is primarily determined by the repetitive usage of its basic elements. For written texts this challenge would correspond to comparing the writing styles of different authors. This similarity has the following options:

| `distribution`  | Possible values: [`ll`](https://lucene.apache.org/core/8_6_2/core/org/apache/lucene/search/similarities/DistributionLL.html) and [`spl`](https://lucene.apache.org/core/8_6_2/core/org/apache/lucene/search/similarities/DistributionSPL.html). |
| --------------- | ------------------------------------------------------------ |
| `lambda`        | Possible values: [`df`](https://lucene.apache.org/core/8_6_2/core/org/apache/lucene/search/similarities/LambdaDF.html) and [`ttf`](https://lucene.apache.org/core/8_6_2/core/org/apache/lucene/search/similarities/LambdaTTF.html). |
| `normalization` | Same as in `DFR` similarity.                                 |

Type name: `IB`

#### 6.6.2.5  LM Dirichlet similarity

​	[LM Dirichlet similarity](https://lucene.apache.org/core/8_6_2/core/org/apache/lucene/search/similarities/LMDirichletSimilarity.html) . This similarity has the following options:

<table>
  <tr>
  	<td>mu</td>
    <td>Default to 2000.</td>
  </tr>
</table>

​	The scoring formula in the paper assigns negative scores to terms that have fewer occurrences than predicted by the language model, which is illegal to Lucene, so such terms get a score of 0.

Type name: `LMDirichlet`

#### 6.6.2.6  LM Jelinek Mercer similarity

​	[LM Jelinek Mercer similarity](https://lucene.apache.org/core/8_6_2/core/org/apache/lucene/search/similarities/LMJelinekMercerSimilarity.html) . The algorithm attempts to capture important patterns in the text, while leaving out noise. This similarity has the following options:

<table>
  <tr>
  	<td>lambda</td>
    <td>The optimal value depends on both the collection and the query. The optimal value is around 0.1 for title queries and 0.7 for long queries. Default to 0.1. When value approaches 0, documents that match more query terms will be ranked higher than those that match fewer terms.</td>
  </tr>
</table>

Type name: `LMJelinekMercer`

#### 6.6.2.7  Scripted similarity

​	A similarity that allows you to use a script in order to specify how scores should be computed. For instance, the below example shows how to reimplement TF-IDF:

```http
PUT /index
{
  "settings": {
    "number_of_shards": 1,
    "similarity": {
      "scripted_tfidf": {
        "type": "scripted",
        "script": {
          "source": "double tf = Math.sqrt(doc.freq); double idf = Math.log((field.docCount+1.0)/(term.docFreq+1.0)) + 1.0; double norm = 1/Math.sqrt(doc.length); return query.boost * tf * idf * norm;"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "field": {
        "type": "text",
        "similarity": "scripted_tfidf"
      }
    }
  }
}

PUT /index/_doc/1
{
  "field": "foo bar foo"
}

PUT /index/_doc/2
{
  "field": "bar baz"
}

POST /index/_refresh

GET /index/_search?explain=true
{
  "query": {
    "query_string": {
      "query": "foo^1.7",
      "default_field": "field"
    }
  }
}
```

Which yields:

```json
{
  "took": 12,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
        "value": 1,
        "relation": "eq"
    },
    "max_score": 1.9508477,
    "hits": [
      {
        "_shard": "[index][0]",
        "_node": "OzrdjxNtQGaqs4DmioFw9A",
        "_index": "index",
        "_type": "_doc",
        "_id": "1",
        "_score": 1.9508477,
        "_source": {
          "field": "foo bar foo"
        },
        "_explanation": {
          "value": 1.9508477,
          "description": "weight(field:foo in 0) [PerFieldSimilarity], result of:",
          "details": [
            {
              "value": 1.9508477,
              "description": "score from ScriptedSimilarity(weightScript=[null], script=[Script{type=inline, lang='painless', idOrCode='double tf = Math.sqrt(doc.freq); double idf = Math.log((field.docCount+1.0)/(term.docFreq+1.0)) + 1.0; double norm = 1/Math.sqrt(doc.length); return query.boost * tf * idf * norm;', options={}, params={}}]) computed from:",
              "details": [
                {
                  "value": 1.0,
                  "description": "weight",
                  "details": []
                },
                {
                  "value": 1.7,
                  "description": "query.boost",
                  "details": []
                },
                {
                  "value": 2,
                  "description": "field.docCount",
                  "details": []
                },
                {
                  "value": 4,
                  "description": "field.sumDocFreq",
                  "details": []
                },
                {
                  "value": 5,
                  "description": "field.sumTotalTermFreq",
                  "details": []
                },
                {
                  "value": 1,
                  "description": "term.docFreq",
                  "details": []
                },
                {
                  "value": 2,
                  "description": "term.totalTermFreq",
                  "details": []
                },
                {
                  "value": 2.0,
                  "description": "doc.freq",
                  "details": []
                },
                {
                  "value": 3,
                  "description": "doc.length",
                  "details": []
                }
              ]
            }
          ]
        }
      }
    ]
  }
}
```

*WARNING：While scripted similarities provide a lot of flexibility, there is a set of rules that they need to satisfy. Failing to do so could make Elasticsearch silently return wrong top hits or fail with internal errors at search time:*

- Returned scores must be positive.
- All other variables remaining equal, scores must not decrease when `doc.freq` increases.
- All other variables remaining equal, scores must not increase when `doc.length` increases.

​	You might have noticed that a significant part of the above script depends on statistics that are the same for every document. It is possible to make the above slightly more efficient by providing an `weight_script` which will compute the document-independent part of the score and will be available under the `weight` variable. When no `weight_script` is provided, `weight` is equal to `1`. The `weight_script` has access to the same variables as the `script` except `doc` since it is supposed to compute a document-independent contribution to the score.

​	The below configuration will give the same tf-idf scores but is slightly more efficient:

```http
PUT /index
{
  "settings": {
    "number_of_shards": 1,
    "similarity": {
      "scripted_tfidf": {
        "type": "scripted",
        "weight_script": {
          "source": "double idf = Math.log((field.docCount+1.0)/(term.docFreq+1.0)) + 1.0; return query.boost * idf;"
        },
        "script": {
          "source": "double tf = Math.sqrt(doc.freq); double norm = 1/Math.sqrt(doc.length); return weight * tf * norm;"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "field": {
        "type": "text",
        "similarity": "scripted_tfidf"
      }
    }
  }
}
```

Type name: `scripted`

#### 6.6.2.8  Default Similarity

​	By default, Elasticsearch will use whatever similarity is configured as `default`.

​	You can change the default similarity for all fields in an index when it is [created](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-create-index.html):

```http
PUT /index
{
  "settings": {
    "index": {
      "similarity": {
        "default": {
          "type": "boolean"
        }
      }
    }
  }
}
```

​	If you want to change the default similarity after creating the index you must [close](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-open-close.html) your index, send the following request and [open](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-open-close.html) it again afterwards:

```http
POST /index/_close

PUT /index/_settings
{
  "index": {
    "similarity": {
      "default": {
        "type": "boolean"
      }
    }
  }
}

POST /index/_open
```

## 6.7 Show Log

后面自己搭建ELK了，以后有空继续啃官方文档。（官方文档前面章节的东西，其实就目前而言我还暂时没多少需要用到的。）。具体的搭建方案，在我的另一个Repository——[Ashiamd/ash-demos](https://github.com/Ashiamd/ash-demos)有代码+对应的文章链接（文章可能文字不是很多），主要就随笔记录下自己的搭建，方便以后自己查阅。