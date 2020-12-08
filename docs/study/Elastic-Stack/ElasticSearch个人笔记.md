# ElasticSearch个人笔记

> [Elasticsearch Reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)	<=	官方文档链接
>
> 官方文档加载极慢(无FQ软件情况下)，建议mac使用Dash离线api软件下载ElasticSearch文档；
>
> Windows和Linux用户可以用Zeal离线api软件做同样操作。
>
> [gavin5033的博客 -- ELK专栏](https://blog.csdn.net/gavin5033/category_8070372.html)	<=	**下面很多内容参考该博客。下面不再反复强调了**。

# 1. RESTful API回顾

​	回顾之前，需要先能跑起来ElasticSearch和Kibana，这里我自己是用docker跑的。[我自己的github](https://github.com/Ashiamd/ash-demos)里也有对应的docker-compose配置文件。

+ 建议Chrome浏览器安装[ElasticSearch Head插件](https://chrome.google.com/webstore/detail/elasticsearch-head/ffmkiejjmecolpfloofpjologoblkegm?utm_source=chrome-ntp-icon)，方便查看ElasticSearch中已有的数据

+ 建议在Kibana可视化页面里，选择"**Add sample data**"，添加样本数据，方便后续复习。我是把三个样本数据都添加了	

## 1.1 CRUD - C

### 1.1.1 cluster - 集群

### 1.1.2 node - 节点

### 1.1.3 index - 索引

#### 1. 新建index

```shell
curl -XPUT "127.0.0.1:9200/index_test"
{"acknowledged":true,"shards_acknowledged":true,"index":"index_test"}%
```

```http
PUT index_test
```

```json
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "index_test"
}
```

#### 2. 创建index的mapping

##### 1. HTTP方式

```shell
curl -X PUT "localhost:9200/my-index-000001?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "age":    { "type": "integer" },  
      "email":  { "type": "keyword"  }, 
      "name":   { "type": "text"  }     
    }
  }
}
'

{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "my-index-000001"
}
```

```http
PUT /my-index-000002
{
  "mappings": {
    "properties": {
      "age":    { "type": "integer" },  
      "email":  { "type": "keyword"  }, 
      "name":   { "type": "text"  }     
    }
  }
}
```

```json
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "my-index-000002"
}
```

##### 2. 配置文件方式

> [实时搜索引擎Elasticsearch（2）——Rest API的使用](https://blog.csdn.net/gavin5033/article/details/82774529)	<=	我自己暂时未尝试该方式

1. 创建一个扩展名为test_type.json的文件名，其中type_test就是mapping所对应的type名

2. 在test_type.json中输入mapping信息。假设你的mapping如下：

   ```json
   {
     "test_type": { # 注意，这里的test_type与json文件名必须一致
         "properties": {
           "name": {
             "type": "string",
             "index": "not_analyzed"
           },
           "age": {
             "type": "integer"
           }
         }
       }
     }
   ```

3. 在$ES_HOME/config/路径下创建mappings/index_test子目录，这里的index_test目录名必须与我们要建立的索引名一致。将test_type.json文件拷贝到index_tes目录下。

4. 创建index_test索引。操作如下：

   ```shell
   curl -XPUT "192.168.1.101:9200/index_test" # 注意，这里的索引名必须与mappings下新建的index_test目录名一致	
   ```

### 1.1.4 document - 文档

#### 1. 新建document

```shell
curl -X PUT "localhost:9200/my-index-000001/_doc/1?pretty" -H 'Content-Type: application/json' -d'
{
  "@timestamp": "2099-11-15T13:12:00",
  "message": "GET /search HTTP/1.1 200 1070000",
  "user": {
    "id": "kimchy"
  }
}
'
```

```http
PUT my-index-000001/_doc/1
{
  "@timestamp": "2099-11-15T13:12:00",
  "message": "GET /search HTTP/1.1 200 1070000",
  "user": {
    "id": "kimchy"
  }
}
```

```json
{
  "_index" : "my-index-000001",
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

## 1.2 CRUD - R

### 1.2.1 cluster - 集群

#### 1. 查看集群状态

> [实时搜索引擎Elasticsearch（2）——Rest API的使用](https://blog.csdn.net/gavin5033/article/details/82774529)

1. 查看集群状态信息

```shell
curl -X GET "http://localhost:9200/_cat/health?v"
```

```http
GET _cat/health?v
```

```http
epoch      timestamp cluster   status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1607318602 05:23:22  EScluster yellow          1         1     11  11    0    0        1             0                  -                 91.7%
```

2. 集群状态信息-help

```shell
curl -X GET "http://localhost:9200/_cat/health?help"
```

```http
GET _cat/health?help
```

```shell
epoch                 | t,time                                   | seconds since 1970-01-01 00:00:00  
timestamp             | ts,hms,hhmmss                            | time in HH:MM:SS                   
cluster               | cl                                       | cluster name                       
status                | st                                       | health status                      
node.total            | nt,nodeTotal                             | total number of nodes              
node.data             | nd,nodeData                              | number of nodes that can store data
shards                | t,sh,shards.total,shardsTotal            | total number of shards             
pri                   | p,shards.primary,shardsPrimary           | number of primary shards           
relo                  | r,shards.relocating,shardsRelocating     | number of relocating nodes         
init                  | i,shards.initializing,shardsInitializing | number of initializing nodes       
unassign              | u,shards.unassigned,shardsUnassigned     | number of unassigned shards        
pending_tasks         | pt,pendingTasks                          | number of pending tasks            
max_task_wait_time    | mtwt,maxTaskWaitTime                     | wait time of longest task pending  
active_shards_percent | asp,activeShardsPercent                  | active number of shards in percent
```

还可以筛选只保留自己要看的参数

```shell
curl -XGET "localhost:9200/_cat/health?h=cluster,pri,relo&v"

cluster   pri relo
EScluster  11    0
```

### 1.2.2 node - 节点

#### 1. 查看集群中的节点信息

```shell
curl -XGET "127.0.0.1:9200/_cat/nodes?v"
```

```http
GET _cat/nodes?v
```

```shell
ip           heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
192.168.96.2           66          38   0    0.30    0.11     0.03 dilmrt    *      19341140f2da
```

### 1.2.3 index - 索引

#### 1. 查看集群中的索引状态

```shell
curl -XGET "localhost:9200/_cat/indices?v"
```

```http
GET _cat/indices?v
```

```shell
health status index                                  uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .kibana-event-log-7.9.3-000001         H1zBeg-RR52Cib1uG-yLOw   1   0         21            0     18.3kb         18.3kb
yellow open   nginx-access-filebeat-7.9.3-2020.11.10 Drvn2GIrTNiG_BzliH7cwg   1   1          5            0     61.9kb         61.9kb
green  open   .apm-custom-link                       vyGMAmm3SAmj78y7N6W1pw   1   0          0            0       208b           208b
green  open   .kibana_task_manager_1                 djnOZ9ohTCS12oMhpN6YTA   1   0          6          290    155.4kb        155.4kb
green  open   kibana_sample_data_ecommerce           qiLYSjcDQ6mEJP984uODJw   1   0       4675            0      4.6mb          4.6mb
green  open   .apm-agent-configuration               cjh2WBCLSNqlXqkxmlQmZw   1   0          0            0       208b           208b
green  open   kibana_sample_data_logs                2Ktbq7_9TDK4pyVoHb7hkQ   1   0      14074            0     11.1mb         11.1mb
green  open   .async-search                          l_O8TC2NQ7irgZ4GG6Jhqg   1   0          0            0      3.5kb          3.5kb
green  open   kibana_sample_data_flights             oxmFWZ6ZQk63AMb1hv60Vw   1   0      13059            0      6.2mb          6.2mb
green  open   .kibana_1                              17RsSAR5RXijDMAvE3HLVQ   1   0        207           18     11.4mb         11.4mb
```

#### 2. 查看索引的mapping

```shell
GET /kibana_sample_data_ecommerce/_mapping
```

```http
GET /kibana_sample_data_ecommerce/_mapping
```

```json
{
  "kibana_sample_data_ecommerce" : {
    "mappings" : {
      "properties" : {
        "category" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword"
            }
          }
        },
        "currency" : {
          "type" : "keyword"
        },
        //....
      }
    }
  }
}
```

### 1.2.4  document - 文档

> [实时搜索引擎Elasticsearch（3）——查询API的使用](https://blog.csdn.net/gavin5033/article/details/82786435)

#### 1. 查询单个document

```shell
curl -XGET '127.0.0.1:9200/my-index-000001/_doc/1?pretty'
```

```http
GET /my-index-000001/_doc/1?pretty
```

```json
{
  "_index" : "my-index-000001",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 3,
  "_seq_no" : 5,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "@timestamp" : "2099-11-15T13:12:00",
    "message" : "GET /search HTTP/1.1 200 1070000",
    "user" : {
      "id" : "kimchy"
    }
  }
}
```

## 1.3 CRUD - U

### 1.3.1 cluster - 集群

### 1.3.2 node - 节点

### 1.3.3 index - 索引

### 1.3.4  document - 文档

#### 1. 更新document

```shell
curl -X POST "localhost:9200/my-index-000001/_update/1?pretty" -H 'Content-Type: application/json' -d'
{
	"doc":{
		"user": {
    	"id" : "kitty"
  	}
	}
}
'
```

```http
POST /my-index-000001/_update/1?pretty
{
	"doc":{
		"user": {
    	"id" : "kitty"
  	}
	}
}
```

```json
{
  "_index" : "my-index-000001",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 2,
  "result" : "updated",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 1,
  "_primary_term" : 1
}
```



## 1.4 CRUD - D

### 1.4.1 cluster - 集群

### 1.4.2 node - 节点

### 1.4.3 index - 索引

#### 1. 删除index

```shell
curl -X DELETE "127.0.0.1:9200/index_test"          
{"acknowledged":true}%
```

```http
DELETE index_test
```

```json
{
  "acknowledged" : true
}
```

#### 2. 删除index的mapping

新版不支持用DELETE删除mapping。

### 1.4.4  document - 文档

#### 1. 删除document

```shell
# 这里的1必须是索引中已经存在id
curl -X DELETE '127.0.0.1:9200/my-index-000001/_doc/1?pretty'
```

```http
DELETE /my-index-000001/_doc/1?pretty
```

```json
{
  "_index" : "my-index-000001",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 2,
  "result" : "deleted",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 4,
  "_primary_term" : 1
}
```

# 2. CRUD - R

> [实时搜索引擎Elasticsearch（3）——查询API的使用](https://blog.csdn.net/gavin5033/article/details/82786435)
>
> **Query和Filter**
>
> ES为用户提供两类查询API，一类是在查询阶段就进行条件过滤的query查询，另一类是在query查询出来的数据基础上再进行过滤的filter查询。这两类查询的区别是：
>
> - **query方法会计算查询条件与待查询数据之间的相关性，计算结果写入一个score字段，类似于搜索引擎。filter仅仅做字符串匹配，不会计算相关性，类似于一般的数据查询，所以filter得查询速度比query快。**
> - **filter查询出来的数据会自动被缓存，而query不能。**
>
> query和filter可以单独使用，也可以相互嵌套使用，非常灵活。
>
> **Query查询**
>
> 下面的情况下适合使用query查询：
>
> - 需要进行全文搜索。
> - 查询结果依赖于相关性，即需要计算查询串和数据的相关性。

​	最常用的一般就是查询操作。而document文档作为实际存储时的实体，对document的查询操作是我们最需要关注的。ElasticSearch的核心也是查询操作。

## 2.0 Search API

> [Search API](http://www.elastic.co/guide/en/elasticsearch/reference/current/search-search.html)

```http
GET /<target>/_search

GET /_search

POST /<target>/_search

POST /_search
```

## 2.1 match all query

### 1. `GET /<target>/_search`

​	查询时，ES服务端默认对查询结果做了分页处理，每页默认的大小为10。如果想自己指定查询的数据，可使用from和size字段，并且按指定的字段排序。

```shell
curl -X GET "localhost:9200/kibana_sample_data_logs/_search?pretty"
```

```http
GET /kibana_sample_data_logs/_search
```

```json
{
  "took" : 0,										// 查询耗时(毫秒)
  "timed_out" : false,					// 是否超时
  "_shards" : {									
    "total" : 1,								// 总共查询的分片数
    "successful" : 1,						// 查询成功的分片数
    "skipped" : 0,							// 查询时跳过的分片数(往往是由于查询带有范围，而分片的数据都在范围外)
    "failed" : 0								// 查询失败的分片数
  },
  "hits" : {
    "total" : {
      "value" : 10000,					// 本次查询的记录数
      "relation" : "gte"
    },
    "max_score" : 1.0,					// 查询所有数据中的最大score
    "hits" : [
      {
        "_index" : "kibana_sample_data_logs",		// 数据所属的索引名
        "_type" : "_doc",												// 数据所属的type
        "_id" : "IWMeO3YB8B3Eg53mS9_J",					// 数据的id值
        "_score" : 1.0,													// 该记录的score
        "_source" : {														// ES将原始数据保存到_source字段中
          "agent" : "Mozilla/5.0 (X11; Linux x86_64; rv:6.0a1) Gecko/20110421 Firefox/6.0a1",
          "bytes" : 6219,
          "clientip" : "223.87.60.27",
          "extension" : "deb",
          // ...
          "timestamp" : "2020-11-29T00:39:02.912Z",
          "url" : "https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.3.2.deb_1",
          "utc_time" : "2020-11-29T00:39:02.912Z",
          "event" : {
            "dataset" : "sample_web_logs"
          }
        }
      },
      // ... 
    ]
  }
}
```

### 2. `POST /<target>/_search`

​	这里就演示分页查询+结果排序。

```shell
curl -X POST "localhost:9200/kibana_sample_data_logs/_search" -d'
{
  "query": {
    "match_all": {}
  },
  "from": 2,        // 从下标2开始取(0，1，2，也就是第三条)
  "size": 4,        // 取4条数据(即2,3,4,5这4条=>如果指数组中的下标的话)
  "sort": {
    "clientip": {  // 按clientip字段升序
      "order": "asc"// 降序为desc
    }
  } 
}
'
```

```http
POST /kibana_sample_data_logs/_search
{
  "query": {
    "match_all": {}
  },
  "from": 2,    
  "size": 4,     
  "sort": {
    "clientip": { 
      "order": "asc"
    }
  } 
}
```

```json
{
  "took" : 22,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10000,
      "relation" : "gte"
    },
    "max_score" : null,
    "hits" : [
      {
        "_index" : "kibana_sample_data_logs",
        "_type" : "_doc",
        "_id" : "iWMeO3YB8B3Eg53mVeuE",
        "_score" : null,
        "_source" : {
          "agent" : "Mozilla/5.0 (X11; Linux i686) AppleWebKit/534.24 (KHTML, like Gecko) Chrome/11.0.696.50 Safari/534.24",
          "bytes" : 9145,
          "clientip" : "0.72.176.46",
          "extension" : "deb",
          "geo" : {
            "srcdest" : "ES:AR",
            "src" : "ES",
            "dest" : "AR",
            "coordinates" : {
              "lat" : 31.68932389,
              "lon" : -87.7613875
            }
          },
          "host" : "artifacts.elastic.co",
          "index" : "kibana_sample_data_logs",
          "ip" : "0.72.176.46",
          "machine" : {
            "ram" : 12884901888,
            "os" : "ios"
          },
          "memory" : null,
          "message" : "0.72.176.46 - - [2018-08-04T19:01:47.849Z] \"GET /beats/metricbeat/metricbeat-6.3.2-amd64.deb HTTP/1.1\" 200 9145 \"-\" \"Mozilla/5.0 (X11; Linux i686) AppleWebKit/534.24 (KHTML, like Gecko) Chrome/11.0.696.50 Safari/534.24\"",
          "phpmemory" : null,
          "referer" : "http://nytimes.com/error/james-mcdivitt",
          "request" : "/beats/metricbeat/metricbeat-6.3.2-amd64.deb",
          "response" : 200,
          "tags" : [
            "success",
            "security"
          ],
          "timestamp" : "2020-12-12T19:01:47.849Z",
          "url" : "https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-6.3.2-amd64.deb",
          "utc_time" : "2020-12-12T19:01:47.849Z",
          "event" : {
            "dataset" : "sample_web_logs"
          }
        },
        "sort" : [
          "0.72.176.46"
        ]
      },
      // ... 另外三项
    ]
  }
}
```

> 注意：不要把from设得过大（超过10000），否则会导致ES服务端因频繁GC而无法正常提供服务。其实实际项目中也没有谁会翻那么多页，但是为了ES的可用性，务必要对分页查询的页码做一定的限制。

## 2.2 term query

> [Term query](https://kapeli.com/dash_share?docset_file=ElasticSearch&docset_name=Elasticsearch&path=www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-term-query.html&platform=elasticsearch&repo=Main&source=www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-term-query.html&version=7.10.0)	<=	下面是官方强调的WARNING
>
> 总之：
>
> **term直接对输入的条件做精准匹配（逻辑上是and）；而match把输入的内容进行分词，再做匹配（逻辑上是or）**
>
> **而text字段，在ES里会被默认拆分成多个分词，所以term大多数情况下无法直接根据整个text字段字匹配到结果。（如果text字段正好无法被分词就可以被匹配。）**
>
>
> Avoid using the `term` query for [`text`](dfile:///Users/ashiamd/Library/Application Support/Dash/DocSets/ElasticSearch/ElasticSearch.docset/Contents/Resources/Documents/www.elastic.co/guide/en/elasticsearch/reference/current/text.html) fields.
>
> By default, Elasticsearch changes the values of `text` fields as part of [analysis](dfile:///Users/ashiamd/Library/Application Support/Dash/DocSets/ElasticSearch/ElasticSearch.docset/Contents/Resources/Documents/www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html). This can make finding exact matches for `text` field values difficult.
>
> To search `text` field values, use the [`match`](dfile:///Users/ashiamd/Library/Application Support/Dash/DocSets/ElasticSearch/ElasticSearch.docset/Contents/Resources/Documents/www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html) query instead.
>
> ---
>
> #### Avoid using the `term` query for `text` fields
>
> By default, Elasticsearch changes the values of `text` fields during analysis. For example, the default [standard analyzer](dfile:///Users/ashiamd/Library/Application Support/Dash/DocSets/ElasticSearch/ElasticSearch.docset/Contents/Resources/Documents/www.elastic.co/guide/en/elasticsearch/reference/current/analysis-standard-analyzer.html)changes `text` field values as follows:
>
> - Removes most punctuation
> - Divides the remaining content into individual words, called [tokens](dfile:///Users/ashiamd/Library/Application Support/Dash/DocSets/ElasticSearch/ElasticSearch.docset/Contents/Resources/Documents/www.elastic.co/guide/en/elasticsearch/reference/current/analysis-tokenizers.html)
> - Lowercases the tokens
>
> To better search `text` fields, the `match` query also analyzes your provided search term before performing a search. This means the `match` query can search `text` fields for analyzed tokens rather than an exact term.
>
> The `term` query does **not** analyze the search term. The `term` query only searches for the **exact** term you provide. This means the `term` query may return poor or no results when searching `text` fields.
>
> [ES中match和term差别对比](https://blog.csdn.net/tclzsn7456/article/details/79956625)
>
> [es的多种term查询](https://www.cnblogs.com/juncaoit/p/12664109.html)

​	词语查询，如果是对未分词的字段进行查询，则表示精确查询。

下面从"kibana_sample_data_flights"这个kibana样本数据index索引中查询"FlightNum"为"EAYQW69"的数据

```shell
curl -X POST "127.0.0.1:9200/kibana_sample_data_flights/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "term": {
      "FlightNum": "EAYQW69"
    }
  }
}
'
```

```http
POST /kibana_sample_data_flights/_search?pretty
{
  "query": {
    "term": {
      "FlightNum": "EAYQW69"
    }
  }
}
```

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 9.071844,
    "hits" : [
      {
        "_index" : "kibana_sample_data_flights",
        "_type" : "_doc",
        "_id" : "vmQiO3YB8B3Eg53mBRZs",
        "_score" : 9.071844,
        "_source" : {
          "FlightNum" : "EAYQW69",
          "DestCountry" : "IT",
          "OriginWeather" : "Thunder & Lightning",
          "OriginCityName" : "Naples",
          "AvgTicketPrice" : 181.69421554118,
          "DistanceMiles" : 345.31943877289535,
          "FlightDelay" : true,
          "DestWeather" : "Clear",
          "Dest" : "Treviso-Sant'Angelo Airport",
          "FlightDelayType" : "Weather Delay",
          "OriginCountry" : "IT",
          "dayOfWeek" : 0,
          "DistanceKilometers" : 555.7377668725265,
          "timestamp" : "2020-11-30T10:33:28",
          "DestLocation" : {
            "lat" : "45.648399",
            "lon" : "12.1944"
          },
          "DestAirportID" : "TV01",
          "Carrier" : "Kibana Airlines",
          "Cancelled" : true,
          "FlightTimeMin" : 222.74905899019436,
          "Origin" : "Naples International Airport",
          "OriginLocation" : {
            "lat" : "40.886002",
            "lon" : "14.2908"
          },
          "DestRegion" : "IT-34",
          "OriginAirportID" : "NA01",
          "OriginRegion" : "IT-72",
          "DestCityName" : "Treviso",
          "FlightTimeHour" : 3.712484316503239,
          "FlightDelayMin" : 180
        }
      }
    ]
  }
}
```

## 2.3 bool query

