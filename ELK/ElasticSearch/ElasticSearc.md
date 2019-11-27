### 术语

文档 `Document`

- 用户存储在es中的数据
- 相当于mysql中的一行数据

索引 `index`

- 具有相同字段的文档列表
- 相当于mysql中的table

节点 `node`

- 一个ElasticSearch运行实例,是集群的构成单元

集群 `cluster`

- 由一个或多个节点组成,对外提供服务

### Document

- `Json Object` 由字段(`Field`)组成,有如下数据类型:
    
    - `字符串`: text,keyword
    - `数值型`: ong,integer,short,byte,double,float,half_float,scaled_float
    - `布尔`: boolean
    - `日期`: date
    - `二进制`: binary
    - `范围类型`: integer_range_float_range,long_range,double_range,date_range
    
- 每个文档有一个唯一的id标识
    
    - 自行指定
    - es自动生成
    
- 元数据(`Document Metadata`),用于标准文档的相关信息

    - `_index`: 文档所在的索引名
    - `_type`: 文档所在的类型名(6.x,_type不在起作用)
    - `_id`: 文档唯一id
    - `_uid`: 组合id,由_type和_id组成(6.x,_type不在起作用)
    - `_source`: 文档的原始Json数据,可以从这里获取每个字段的内容
    - `_all`: 整合所有字段到该字段,默认禁用

### index

- 索引中存储具有相同结构的文档Document
    
    - 每个索引都有自己的mapping定义,用于定义字段名称和类型
    
- 一个集群可以有多个索引
    
    - nginx日志存储的时候可以按照日期每天生成一个索引来存储
        
        - nginx-log-2019-11-26
        - nginx-log-2019-11-27
        - nginx-log-2019-11-28
        
### Rest API

- Elasticsearch 集群对外提供的RESTful API

    - REST - REpresentational State Transfer
    - URI 指定资源,如Index,Document等
    - Http Method 指明资源操作类型,如GET,POST,PUT,DELETE
    
- RestAPI交互方式
    
    - Curl命令行
      ```shell script
      curl -XPUT "http://localhost:9200/employee/doc/1" \
          -i -H "Content-Type:application/json" -d \
          '
            {
              "username":"rockybean",
              "job":"software engineer"
            }
          '
      ```
    - Kibana DevTools
      ```shell script
      GET _search
        {
          "query": {
            "match_all": {}
          }
        }
      ```
      
### index API

- 创建索引api
```shell script
PUT /test_index

{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "test_index"
}
```
- 查询现有索引
```shell script
GET _cat/indices

green  open kibana_sample_data_ecommerce  DzLPk0gKR0qyS05Th3jG8Q 1 0 4675 0   4.7mb   4.7mb
green  open .kibana_task_manager_1        fK684UKpT7W4zfLanwdwOA 1 0    2 0  22.7kb  22.7kb
green  open .apm-agent-configuration      mI0VDGyIQ92sPH73uhlEuw 1 0    0 0    283b    283b
yellow open logstash-%{[type]}-2019.11.19 -QNRJqORS-GRDlwstWy2QQ 1 1  531 0 165.6kb 165.6kb
yellow open test_index                    BCykPn-kRuqaFKcmFJO2OQ 1 1    0 0    230b    230b
green  open .kibana_1                     bJNiOW3CSe23Q-9EDYcFKw 1 0   69 1 938.7kb 938.7kb
```
- 删除索引api
```shell script
DELETE /test_index

{
  "acknowledged" : true
}
```

### DocumentAPI

- 创建文档api
PUT /{index}/{type}/{id}
doc弃用了,待修改
创建文档时,索引不存在,es会自动创建对应的index和type

```shell script
PUT /test_index/_doc/1
PUT /test_index/_create/1
POST /test_index/_doc/
{
  "username":"alfred",
  "age":1
}

{
  "_index" : "test_index",
  "_type" : "doc",
  "_id" : "2",
  "_version" : 1, 
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 1,
  "_primary_term" : 1
}
version: 是一种锁的机制

```

- 查询文档API

```shell script
#查询存在
GET /test_index/_doc/1
{
  "_index" : "test_index",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "_seq_no" : 0,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "username" : "alfred",
    "age" : 1
  }
}
#查询不存在
GET /test_index/_doc/8
{
  "_index" : "test_index",
  "_type" : "_doc",
  "_id" : "8",
  "found" : false
}
#查询所有文档
GET /test_index/_search
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
      "value" : 6,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "test_index",
        "_type" : "doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "username" : "alfred",
          "age" : 1
        }
      },
      {
        "_index" : "test_index",
        "_type" : "doc",
        "_id" : "3",
        "_score" : 1.0,
        "_source" : {
          "username" : "alfred",
          "age" : 1
        }
      },
      {
        "_index" : "test_index",
        "_type" : "doc",
        "_id" : "4",
        "_score" : 1.0,
        "_source" : {
          "username" : "alfred",
          "age" : 1
        }
      },
      {
        "_index" : "test_index",
        "_type" : "doc",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "username" : "alfred",
          "age" : 1
        }
      },
      {
        "_index" : "test_index",
        "_type" : "doc",
        "_id" : "XTy9q24BWrQJQOtrRa_O",
        "_score" : 1.0,
        "_source" : {
          "username" : "alfred",
          "age" : 1
        }
      },
      {
        "_index" : "test_index",
        "_type" : "doc",
        "_id" : "YDy_q24BWrQJQOtrRq8M",
        "_score" : 1.0,
        "_source" : {
          "username" : "alfred",
          "age" : 1
        }
      }
    ]
  }
}
```

- 批量创建文档API

es允许一次创建多个文档,从而减少网络传输的开销,提升写入速率

   - endpoint为_bulk
   
        action_type为
        
            - index 创建 文档存在时覆盖
            - update 更新
            - create 创建 文档存在时报错
            - delete 删除
            
        ```shell script
        POST _bulk
        
        ```
        
- 批量查询文档API

es允许一次查询多个文档

    - endpoint为 _mget
    
```shell script
GET /_mget
{
  "docs":[
    {
      "_index": "test_index",
      "_id": "4"
    },
    {
      "_index": "test_index",
      "_id": "6"
    }]
}


{
  "docs" : [
    {
      "_index" : "test_index",
      "_type" : "doc",
      "_id" : "4",
      "_version" : 1,
      "_seq_no" : 4,
      "_primary_term" : 1,
      "found" : true,
      "_source" : {
        "username" : "alfred",
        "age" : 1
      }
    },
    {
      "_index" : "test_index",
      "_type" : "doc",
      "_id" : "8",
      "found" : false
    }
  ]
}
```
        
        




    