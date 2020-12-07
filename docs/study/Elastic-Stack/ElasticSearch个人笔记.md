# ElasticSearch个人笔记

> [Elasticsearch Reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)	<=	官方文档链接
>
> 官方文档加载极慢(无FQ软件情况下)，建议mac使用Dash离线api软件下载ElasticSearch文档；
>
> Windows和Linux用户可以用Zeal离线api软件做同样操作。

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











