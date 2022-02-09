# Elasticsearch速览学习笔记

**声明**：本来是打算春节期间在官网学习，温故一下相关知识，。无意间发现**铭毅天下Elasticsearch**文章很全，对快速了解一些知识点，很有帮助。尤其是知识星球里的内容，奈何收费。别人辛勤劳动成果，当然无可厚非。我就借鉴了他的**知识图谱**，确定自己的学习点，再结合官网文档和他的公众号。引用的地方已经标注，特此声明。

**如果没钱没时间，收藏我这边篇笔记就好。如果你舍得花钱，有充足的时间，推荐去购买一下他的知识星球。**



## 基本知识点

### 分词必知

当字段类型为text的时候会进行分词，默认分词器是`standard`

两个地方会出现分词，一个是indexing,一个是search。文档索引的时候肯定会分词，search时候针对search查询语句内容分析。默认的话是两者保持一致。某些场景下可以在[search中设置分词](https://www.elastic.co/guide/en/elasticsearch/reference/current/specify-analyzer.html#specify-search-query-analyzer)。

分词器分为三个部分：Tokenizers (分词)、Token filters（修改分词例如小写，删除分词，增加分词）、Character filters（用在分词前去除字符）

#### Test分词器

```
POST _analyze
{
  "analyzer":"standard",
  "text":     "The quick brown fox. 1"
}
```



#### 排查当前index分词的结果

```
GET kibana_sample_data_logs/_analyze
{
  "field": "my_text", 
  "text":  "Is this déjà vu?"
}
```

#### 配置一个分析器，去掉英文修饰词

```
PUT my-index-000001
{
  "settings": {
    "analysis": {
      "analyzer": {
        "std_english": { 
          "type":      "standard",
          "stopwords": "_english_"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "my_text": {
        "type":     "text",
        "analyzer": "standard", 
        "fields": {
          "english": {
            "type":     "text",
            "analyzer": "std_english" 
          }
        }
      }
    }
  }
}
```

测试

```
POST my-index-000001/_analyze
{
  "field": "my_text",
  "text": "The old brown cow"
}
// [ the, old, brown, cow ]
POST my-index-000001/_analyze
{
  "field": "my_text.english",
  "text": "The old brown cow"
}
// [ old, brown, cow ]
```



### Mapping必知

为了避免Mapping爆炸，默认最多字段数`index.mapping.total_fields.limit:1000`包含字段别名。也就意味着默认情况下，一个index最多1000个字段（Field and object）。

`index.mapping.nested_fields.limit:50` nested_fields默认50。

Mapping有[Explicit mapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/explicit-mapping.html) （显示）和[Dynamic mapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-field-mapping.html) （动态）。Mapping可以定义runtime field，runtime field不会占用存储，增加获取数据的灵活能力，但是速度会慢（由runtime script决定性能影响）。

#### 创建一个索引模板，设置分片，副本，别名，字段类型

设置一个分片为5，副本为2，别名为truman,timestamp为时间类型，event类型为text和keyword，rawData不支持检索。

```
PUT _template/truman_template
{
  "index_patterns": ["truman-*"],
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 2
  },
  "mappings": {
    "properties": {
      "timestamp":{
        "type": "date"
      },
      "event":{
        "type": "text",
        "fields": {
          "keyword":{
            "type":"keyword",
            "ignore_above" : 256
          }
        }
        
      },
      "rawData":{
        "type": "keyword", 
        "index": false
      }
    }
  },
  "aliases": {
    "truman": {}
  }
}
```

#### dynamic

默认为true。针对string字段，会自动生成两个类型`text`和`keyword`

#### Dynamic templates

Dynamic templates允许通过一定的规则设置字段类型

```
PUT my-index-000001/
{
  "mappings": {
    "dynamic_templates": [
      {
        "strings_as_ip": {
          "match_mapping_type": "string",
          "match": "ip*",
          "runtime": {
            "type": "ip"
          }
        }
      }
    ]
  }
}

PUT my-index-000001
{
  "mappings": {
    "dynamic_templates": [
      {
        "longs_as_strings": {
          "match_mapping_type": "string",
          "match":   "long_*",
          "unmatch": "*_text",
          "mapping": {
            "type": "long"
          }
        }
      }
    ]
  }
}
```

### Join类型及应用

在es中可以定义字段为join类型，实现类似于关系型数据库的多表关联查询。但是es父子文档还是存储在一个index下的。

>1. 每个索引只允许一个Join类型Mapping定义；
>2. 父文档和子文档必须在同一个分片上编入索引；这意味着，当进行删除、更新、查找子文档时候需要提供相同的路由值。
>3. 一个文档可以有多个子文档，但只能有一个父文档。
>4. 可以为已经存在的Join类型添加新的关系。
>5. 当一个文档已经成为父文档后，可以为该文档添加子文档。

#### join定义

```
PUT knownledge
{
  "mappings": {
    "properties": {
      "id":{
        "type": "keyword"
      },
      "my_join_field":{
        "type": "join",
        "relations":{
          "question":["answer"]
        }
      }
    }
  }
}
```

```
PUT knownledge/_doc/1?refresh
{
  "id":"1",
  "text":"What are the brands of computers?",
  "my_join_field":{
    "name": "question"
  }
}
PUT knownledge/_doc/2?refresh
{
  "id":"2",
  "text":"What are the brands of mobile phones?",
  "my_join_field":{
    "name": "question"
  }
}

PUT knownledge/_doc/3?routing=1&refresh
{
  "id":"3",
  "text":"dell",
  "my_join_field":{
    "name": "answer", 
    "parent": "1" 
  }
}

PUT knownledge/_doc/4?routing=2&refresh
{
  "id":"4",
  "text":"apple",
  "my_join_field":{
    "name": "answer", 
    "parent": "2" 
  }
}

PUT knownledge/_doc/5?routing=1&refresh
{
  "id":"5",
  "text":"神州电脑",
  "my_join_field":{
    "name": "answer", 
    "parent": "1" 
  }
}
```



#### 根据父查询所有的子文档

查询父id为1下的所有子文档。注意这里是父id，我这边例子业务id于index id是一致的。其实可以不一致。

```
GET knownledge/_search
{
  "query": {
    "parent_id":{
      "type": "answer",
      "id": "1"
    }
  }
}

```

查询符合父文档下的子文档

```
GET knownledge/_search
{
  "query": {
    "has_parent": {
      "parent_type": "question",
      "query": {
        "match": {
          "text": "computers"
        }
      }
    }
  }
}
```



#### 根据子查询所有的父文档

```
GET knownledge/_search
{
  "query": {
    "has_child": {
      "type": "answer",
      "query": {
        "match": {
          "text": "apple"
        }
      }
    }
  }
}
```



### Nested类型及应用

Nested为了解决数组对象被扁平化为一个简单的字段名称和值列表。

```
{
  "group" : "fans",
  "user" : [ 
    {
      "first" : "John",
      "last" :  "Smith"
    },
    {
      "first" : "Alice",
      "last" :  "White"
    }
  ]
}
```

默认类型,上面数据会被elasticsearch解析为：

```
{
  "group" :        "fans",
  "user.first" : [ "alice", "john" ],
  "user.last" :  [ "smith", "white" ]
}
```

如果需要查询user.first：John user.last：White就会出错。

可以通过将该字段类型设置为Nested解决。

```
PUT nested-index-000001
{
  "mappings": {
    "properties": {
      "user": {
        "type": "nested" 
      }
    }
  }
}
```

设置为Nested类型的查询必须在Nested下才能查到

```
GET nested-index-000001/_search
{
  "query": {
    "nested": {
      "path": "user",
      "query": {
        "bool": {
          "must": [
            {
              "match": {
                "user.first": "Alice"
              }
            },
            {
              "match": {
                "user.last": "White"
              }
            }
          ]
        }
      }
    }
  }
}
```

Nested和join都可解决多表关联的场景，各有优缺点，想了解更多的，可以查看[干货 | Elasticsearch多表关联设计指南](https://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ%3D%3D&chksm=eaa82bf6dddfa2e0bf920f0a3a63cb635277be2ae286a2a6d3fff905ad913ebf1f43051609e8&idx=1&mid=2247484382&scene=21&sn=da073a257575867b8d979dac850c3f8e#wechat_redirect)

### 索引必知

#### 别名

如果想用别名写数据，同一个别名，只能有一个index是`is_write_index:true`

模板可以创建别名，当然也可以用过api创建

创建别名(前提是index存在)

```
PUT /truman-003/_alias/truman
```

创建别名(index不存在)

```
PUT truman-003
{
  "aliases": {
    "truman": {}
  }
}
```



更新别名

```
POST /_aliases
{
  "actions" : [
    { "add" : { "index" : "truman-001", "alias" : "truman","is_write_index": false } },
     { "add" : { "index" : "truman-002", "alias" : "truman","is_write_index": true } }
  ]
}
```

#### 索引增删改查

```
PUT /my-index-000001
{
  "settings": {
    "number_of_shards": 1
  },
  "mappings": {
    "properties": {
      "field1": { "type": "text" }
    }
  },
   "aliases": {
    "my_index_aliases": {}
  }
}
// 只可更改动态参数 https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html#dynamic-index-settings
PUT /my-index-000001/_settings
{
  "index" : {
    "number_of_replicas" : 2
  }
}

GET /my-index-000001

DELETE /my-index-000001
```



### 文档必知

#### 文档增删改查

```
POST test-003/_doc
{
  "name":"qqq",
  "age":36
}
```

```
GET test-003/_doc/gk5pz3cBlx8vUhtcU9IJ
```

```
PUT test-003/_doc/gk5pz3cBlx8vUhtcU9IJ
{
  "name":"wwww",
  "age":36
}
```

```
DELETE test-003/_doc/gk5pz3cBlx8vUhtcU9IJ
```

#### 文档写入原理

第一步：客户端向集群某节点写入数据，发送请求。（如果没有指定路由/协调节点，请求的节点扮演协调节点的角色。）

第二步：协调节点接受到请求后，默认使用文档 ID 参与计算（也支持通过 routing），得到该文档属于哪个分片。随后请求会被转到另外的节点。

```
bash# 路由算法：根据文档id或路由计算目标的分片id
shard = hash(document_id) % (num_of_primary_shards)
```

第三步：当分片所在的节点接收到来自协调节点的请求后，会将请求写入到 Memory Buffer，然后定时（默认是每隔 1 秒）写入到Filesystem Cache，这个从 Memory Buffer 到 Filesystem Cache 的过程就叫做 refresh；

第四步：当然在某些情况下，存在 Memory Buffer 和 Filesystem Cache 的数据可能会丢失，ES 是通过 translog 的机制来保证数据的可靠性的。其实现机制是接收到请求后，同时也会写入到 translog 中，当 Filesystem cache 中的数据写入到磁盘中时，才会清除掉，这个过程叫做 flush；

第五步：在 flush 过程中，内存中的缓冲将被清除，内容被写入一个新段，段的 fsync 将创建一个新的提交点，并将内容刷新到磁盘，旧的 translog 将被删除并开始一个新的 translog。

第六步：flush 触发的时机是定时触发（默认 30 分钟）或者 translog 变得太大（默认为 512 M）时。



#### 文档删除和更新的本质

删除和更新也都是写操作，但是 Elasticsearch 中的文档是不可变的，因此不能被删除或者改动以展示其变更。

磁盘上的每个段都有一个相应的 .del 文件。当删除请求发送后，文档并没有真的被删除，而是在 .del 文件中被标记为删除。该文档依然能匹配查询，但是会在结果中被过滤掉。当段合并时，在 .del 文件中被标记为删除的文档将不会被写入新段。

在新的文档被创建时，Elasticsearch 会为该文档指定一个版本号，当执行更新时，旧版本的文档在 .del 文件中被标记为删除，新版本的文档被索引到一个新段。旧版本的文档依然能匹配查询，但是会在结果中被过滤掉。

### 检索必知

在 bool 查询中，`filter` 和 `must_not` 属于 Filter Context，不会对算分结果产生影响；`must` 和 `should` 属于 Query Context，会对结果算分产生影响。

 Filter Context会使用缓存，因为速度会比Query Context快。如果不用考虑评分的话，优先考虑使用Filter Context。

#### 检索选型

##### 全文检索（Full text queries）

- **intervals** 可以对匹配项的顺序和接近度进行细粒度控制

- **match**

用于执行全文查询的标准查询，包括模糊匹配和短语或接近查询。

- **match_bool_prefix**

创建一个布尔查询，将与每个词匹配的词查询作为词查询，但最后一个词除外，后者作为前缀查询匹配。这里和match_phrase_prefix有点区别。前者不在意顺序，后者严格的词组顺序。

- **match_phrase/match_phrase_prefix**

    match_phrase用于匹配精确短语或单词接近匹配。match_phrase_prefix 前缀短语匹配查询

- **multi_match**

匹配查询的多字段版本。

```
GET /_search
{
  "query": {
    "multi_match" : {
      "query":    "Will Smith",
      "fields": [ "title", "*_name" ] 
    }
  }
}
```



- **query_string/simple_query_string**

query_string支持紧凑的Lucene query string 语法，允许在单个查询字符串中指定AND | OR | NOT条件和多字段搜索。仅限于专业用户

simple_query_string适用于直接向用户公开的query_string语法的更简单，更可靠的版本。

query_string和simple_query_string区别在于，simple_query_string提供语法容错。如果你的search很容易写错，避免告诉用户错误信息，推荐用这个。

##### Term-level 检索

和全文检索不同，Term-level 不会对查询语句分词。Term-leve用作精确查找。

- exists
- fuzzy
- ids
- prefix
- range
- regexp
- term
- terms
- wildcard

###### exists：查询存在user字段

```
GET test/_search
{
  "query": {
    "exists": {"field": "user"}
  }
}
```

###### fuzzy 模糊查询

返回包含与搜索词相似的词的文档，以编辑距离测量

例如以下例子可以查询出`user:truman`数据

```
GET test/_search
{
  "query": {
    "fuzzy": {
      "user": {
        "value": "troman",
        "fuzziness": "AUTO"
      }
    }
  }
}
```

`fuzziness: AUTO` 表示 0-2字符，必须完全匹配；3-5字符，允许一个编辑距离；>5字符允许两个编辑距离

###### ids

可以批量根据_id获取文档

```
GET test/_search
{
  "query": {
    "ids" : {
      "values" : ["1", "xNT8w3cBZr0oc6pbkb3Q", "100"]
    }
  }
}
```

###### term/terms/terms_set

避免term查询text类型的字段，对于text类型的字段，应该是match来查询。因为text类型会被分词器做一定处理，例如小写，删除某些字符。因此用term查询很可能查不出来。

terms和term一样，不过允许查询多个值。

```
GET test/_search
{
  "query": {
    "terms": {
      "user": [ "truman", "jim" ],
      "boost": 1.0
    }
  }
}
```

terms_set和terms类似，但是支持设置要求minimum_should_match_field。

````
POST test/_doc
{
  "user":"ddd",
  "programming_languages": [ "c++", "php" ],
  "minimum_should_match":2
}
GET test/_search
{
  "query": {
    "terms_set": {
      "programming_languages": {
        "terms": [ "c++", "java", "php" ],
        "minimum_should_match_field": "minimum_should_match"
      }
    }
  }
}
````

###### wildcard 

通配符查找

```
GET /_search
{
  "query": {
    "wildcard": {
      "user.id": {
        "value": "ki*y",
        "boost": 1.0,
        "rewrite": "constant_score"
      }
    }
  }
}
```



#### 高亮/分页/排序

##### 高亮/排序

```
GET test/_search
{
  "sort": [
    {
      "rawData": {
        "order": "desc"
      }
    }
  ], 
  "query": {
    "match": {
      "event": "has"
    }
  },
  "highlight": {
    "fields": {"event": {}}
  }
}
```

##### 分页

- from+size 深度分页性能较差

  ```
  GET test/_search?from=0&size=5
  {
    "query": {"match_all": {}}
  }
  ```

- scroll

  ```
  GET test*/_search?scroll=2m&size=3
  {
    "query": {"match_all": {}}
  }
  ```

  ```
  GET _search/scroll
  {
    "scroll_id" : "DnF1ZXJ5VGhlbkZldGNoBQAAAAAAAAcDFllrSTUzWlEzUUFtTnlGY3czWVJlRmcAAAAAAAAHAhZZa0k1M1pRM1FBbU55RmN3M1lSZUZnAAAAAAAABwEWWWtJNTNaUTNRQW1OeUZjdzNZUmVGZwAAAAAAAAcEFllrSTUzWlEzUUFtTnlGY3czWVJlRmcAAAAAAAAHBRZZa0k1M1pRM1FBbU55RmN3M1lSZUZn"
  }
  ```

  

#### Query和Filter的本质区别

query 关注的是文档是否包含，相关得分怎么样，得分越高的排名越靠前

filter关注是查询是否包含在结果中，不涉及评分，有缓存，速度能更快。

query在search api中query参数

filter在bool查询中（filter,must_not）,或者filter聚合中，constant_score 查询中

#### 检索模板

```
GET test/_search/template
{
  "source":{
    "query": {"match": {"{{custom_field}}":"{{custom_value}}"}},
    "size":"{{custom_size}}"
  },
  "params" : {
    "custom_field" : "event",
    "custom_value" : "has",
    "custom_size" : 5
  }
}
```

或者现将该模板储存，通过id查询

```
POST _scripts/my_search_template
{
  "script": {
    "lang": "mustache",
    "source":{
    "query": {"match": {"{{custom_field}}":"{{custom_value}}"}},
    "size":"{{custom_size}}"
  }
  }
}

GET test/_search/template
{
  "id":"my_search_template",
  "params" : {
    "custom_field" : "event",
    "custom_value" : "has",
    "custom_size" : 5
  }
}
```



### 聚合必知

聚合分3大类：

- Bucket 按组分类，类似group by
- Metrics 类似max min avg sum
- Pipeline 基于聚合的结果进行判定计算后取结果

这里涉及内容很多，推荐查看[官网学习](https://www.elastic.co/guide/en/elasticsearch/reference/7.11/search-aggregations.html)

### Pipeline

标题是pipeline,这里要说的是Ingest节点提供的能力，当数据量小的时候，可以通过pipeline实现修改字段名，修改字段值，新增字段等功能。它生效在文档能够被索引之前，这点要注意。

#### 通过pipeline新增时间字段timestap

新建pipeline

```
PUT _ingest/pipeline/add_timestap_pipeline
{
  "description": "set timestap field ",
  "processors": [
    {"set": {
      "field": "timestap",
      "value": "{{_ingest.timestamp}}"
    }}
  ]
}
```

```
POST pipeline-index/_doc?pipeline=add_timestap_pipeline
{
  "name":"truman",
  "age":"19"
}
```



#### 删除符合条件的文档

```
PUT _ingest/pipeline/drop_document_pipeline
{
  "description": "drop document ",
  "processors": [
    {"drop": {"if": "ctx.name=='jim'"}}
  ]
}

POST pipeline-index/_doc?pipeline=drop_document_pipeline
{
  "name":"jim",
  "age":"19"
}
```

## 高阶知识点

### ILM

ILM 定义了四种解析阶段：

- **Hot**: 该索引正在积极地更新和查询。
- **Warm**: 索引不再更新，但是仍然被查询.
- **Cold**: 索引不再更新，很少被查询 ，索引仍然是可查询的，但是查询很慢。
- **Delete**: 该索引不再需要，可以被安全的删除

在不同的阶段可以做不行的动作：

- Hot
  - [Set Priority](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-set-priority.html)
  - [Unfollow](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-unfollow.html)
  - [Force Merge](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-forcemerge.html)
  - [Rollover](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-rollover.html)
- Warm
  - [Set Priority](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-set-priority.html)
  - [Unfollow](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-unfollow.html)
  - [Read-Only](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-readonly.html)
  - [Allocate](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-allocate.html)
  - [Shrink](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-shrink.html)
  - [Force Merge](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-forcemerge.html)
- Cold
  - [Set Priority](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-actions.html#ilm-set-priority-action)
  - [Unfollow](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-actions.html#ilm-unfollow-action)
  - [Allocate](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-allocate.html)
  - [Freeze](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-freeze.html)
- Delete
  - [Wait For Snapshot](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-actions.html#ilm-wait-for-snapshot-action)
  - [Delete](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-delete.html)

不同动作作用：

- **Rollover**: 滚动index生成
- **Shrink**: 减少索引的分片
- **Force merge**: Manually trigger a merge to reduce the number of segments in each shard of an index and free up the space used by deleted documents.
- **Freeze**: 标记索引为只读，最大减少内存的占用。
- **Force merges**: 标记索引为只读，合并索引的段，删除

### 运维必知

#### 集群监控Top 10指标

1. Cluster Health – Nodes and Shards
2. Search Performance – Request Latency and
3. Search Performance – Request Rate
4. Indexing Performance – Refresh Times
5. Indexing Performance – Merge Times
6. Node Utilization – Thread Pools
7. Caching – Field Data, Node Query and Shard Query Cache
8. Node Health – Memory Usage
9. Node Health – Disk I/O
10. Node Health – CPU
11. JVM Health – Heap Usage and Garbage Collection
12. JVM health – JVM Pool Size

推荐查看如下文章

[干货 | Elasticsearch Top10 监控指标](https://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ%3D%3D&chksm=eaa82c31dddfa5275eada1884ecf6089ec5e29dd6f4f4a58ded57633ef72879415ca49aae4a7&idx=1&mid=2247484441&scene=21&sn=8292f50bedec05d040c309d50520a3b9#wechat_redirect)

或者原文[Top 10 Elasticsearch Metrics to Monitor](https://sematext.com/blog/top-10-elasticsearch-metrics-to-watch/)

#### 集群运维相关api

**查询集群健康情况**

```
GET _cluster/health
```

```
{
  "cluster_name" : "docker-cluster",
  "status" : "yellow",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 41,
  "active_shards" : 41,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 65,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 38.67924528301887
}
```

这里关注status(集群的健康状态)，relocating_shards(迁移分片)，initializing_shards(初始分片)，unassigned_shards（未分配分片）

**索引级别的健康情况**

```
GET /_cluster/health?level=indices&pretty
```

**分片健康**

```
GET /_cluster/health?level=shards&pretty
```

**监控机器资源负载**

```
GET /_cat/nodes?v&h=heap.percent,diskUsedPercent,cpu,load_1m,master,name
```

**查看哪个索引是黄色或者红色**

```
GET /_cat/indices?v&health=yellow
GET /_cat/indices?v&health=red
```

**查询集群状态不正常的原因**

```
GET _cluster/allocation/explain
```

**查询未分配原因**

```
GET _cat/shards?v&h=index,shard,prirep,state,unassigned.reason
```

**优雅下线节点**

```
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.allocation.exclude._ip": "192.168.0.110"
  }
}
```

**清除节点缓存**

```
POST /_cache/clear
```

**手动刷盘**

```
POST /_flush
```

**手动移动分片**

```
POST /_cluster/reroute
{
  "commands": [
    {
      "move": {
        "index": "test", "shard": 0,
        "from_node": "node1", "to_node": "node2"
      }
    },
    {
      "allocate_replica": {
        "index": "test", "shard": 1,
        "node": "node3"
      }
    }
  ]
}
```

**index迁移**

```
POST _reindex
{
  "source": {
    "index": "my-index-000001"
  },
  "dest": {
    "index": "my-new-index-000001"
  }
}
```



#### 集群优化

**部署层面：**

1）最好是64GB内存的物理机器

3）尽量使用SSD

4）避免集群跨越大的地理距离，一般一个集群的所有节点位于一个数据中心中。

5）设置堆内存：节点内存/2，不要超过32GB。一般来说设置export ES_HEAP_SIZE=32g环境变量，比直接写-Xmx32g -Xms32g更好一点。

6）关闭缓存swap。

7）增加文件描述符，设置一个很大的值

8）不要随意修改垃圾回收器（CMS）和各个线程池的大小。

9）通过设置gateway.recover_after_nodes、gateway.expected_nodes、gateway.recover_after_time可以在集群重启的时候避免过多的分片交换，这可能会让数据恢复从数个小时缩短为几秒钟。

**索引层面：**

1）使用批量请求并调整其大小：每次批量数据 5–15 MB 大是个不错的起始点。

2）段合并：Elasticsearch默认值是20MB/s，对机械磁盘应该是个不错的设置。如果你用的是SSD，可以考虑提高到100-200MB/s。如果你在做批量导入，完全不在意搜索，你可以彻底关掉合并限流。另外还可以增加 index.translog.flush_threshold_size 设置，从默认的512MB到更大一些的值，比如1GB，这可以在一次清空触发的时候在事务日志里积累出更大的段。

3）如果你的搜索结果不需要近实时的准确度，考虑把每个索引的index.refresh_interval 改到30s。

4）如果你在做大批量导入，考虑通过设置index.number_of_replicas: 0 关闭副本。

5）需要大量拉取数据的场景，可以采用scan & scroll api来实现，而不是from/size一个大范围。

**存储层面：**

1）基于数据+时间滚动创建索引，每天递增数据。控制单个索引的量，一旦单个索引很大，存储等各种风险也随之而来，所以要提前考虑+及早避免。

2）冷热数据分离存储，热数据（比如最近3天或者一周的数据），其余为冷数据。对于冷数据不会再写入新数据，可以考虑定期force_merge加shrink压缩操作，节省存储空间和检索效率。

### 原理认知

#### 段合并原理

未搞明白，先占位。

#### 索引创建原理

详细信息参考[图解Elasticsearch之一——索引创建过程](https://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ%3D%3D&chksm=eaa82b49dddfa25ff6f0fe00cc8c1d0d5d78bd063b27e25d17e7cb51a6e6996bd0e9109a8bdb&idx=1&mid=2247484257&scene=21&sn=78a41cab68de21174883e0ba6948652c#wechat_redirect)

#### ES搜索的过程

搜索分为两阶段过程，即 Query Then Fetch；

**Query阶段**：

查询会广播到索引中每一个分片拷贝（主分片或者副本分片）。每个分片在本地执行搜索并构建一个匹配文档的大小为 from + size 的优先队列。PS：在搜索的时候是会查询Filesystem Cache的，但是有部分数据还在Memory Buffer，所以搜索是近实时的。

每个分片返回各自优先队列中 **所有文档的 ID 和排序值** 给协调节点，它合并这些值到自己的优先队列中来产生一个全局排序后的结果列表。

**Fetch阶段**：

协调节点辨别出哪些文档需要被取回并向相关的分片提交多个 GET 请求。每个分片加载并 丰富 文档，如果有需要的话，接着返回文档给协调节点。一旦所有的文档都被取回了，协调节点返回结果给客户端。

#### 存储原理

详细信息参考[Elasticsearch存储深入详解](https://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ%3D%3D&chksm=eaa82b23dddfa23576681118838761947955a05bbac35e42883ed54b81a193141837971c5fd7&idx=1&mid=2247484171&scene=21&sn=985a71a4f2fff5233b1803fa6e8bb9db#wechat_redirect)

## 资源


1. [好奇？！Elasticsearch 25 个必知必会的默认值](https://blog.csdn.net/laoyang360/article/details/106464359)
2. [重磅 | 死磕 Elasticsearch 方法论认知清单（2020年国庆更新版）](https://elastic.blog.csdn.net/article/details/108839580)
3. [你必须知道的23个最有用的Elasticseaerch检索技巧](https://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ%3D%3D&chksm=eaa82934dddfa0223aacd36e64e7794449b98d51a1808a651b9fe4ecffc29a865bc9dafdf46b&idx=3&mid=2247483676&scene=21&sn=7650115cd54c1b8ed04a0557afce4dc4#wechat_redirect)

## 参考

1.[Elasticsearch 6.X 新类型Join深入详解](https://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ%3D%3D&chksm=eaa82a76dddfa3604354a969ec1520121294e188ea2ba5120a3a29026c379fb45f53394d4a27&idx=1&mid=2247483998&scene=21&sn=6c407a0e0a30c1237451ddd1b40f5c0b#wechat_redirect)
2.[干货 | 通透理解Elasticsearch聚合](https://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ%3D%3D&chksm=eaa82b06dddfa2100d81c0b86f373f7312139b14183e023b0d8d2f07ca2449bd1c32d12754b7&idx=1&mid=2247484206&scene=21&sn=d4b756b2966b0933fb6e8626b552cc21#wechat_redirect)
3.[Top 10 Elasticsearch Metrics to Monitor](https://sematext.com/blog/top-10-elasticsearch-metrics-to-watch/)
4.[Elasticsearch运维实战常用命令清单](https://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ%3D%3D&chksm=eaa82efddddfa7eb7b1d394159708e882d3aed6ebfa6c6d0871d457b12005f42b253152fc274&idx=1&mid=2247485141&scene=21&sn=c785d6c128761c33f9744bf1454a472a#wechat_redirect)
5.[Elasticsearch面试题汇总与解析](https://www.wenyuanblog.com/blogs/elasticsearch-interview-questions.html)
6.[Elasticsearch 技术分析（八）：剖析 Elasticsearch 的索引原理](https://www.cnblogs.com/jajian/p/10737373.html)