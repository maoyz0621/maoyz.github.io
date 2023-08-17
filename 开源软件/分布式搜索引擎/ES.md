# ElassticSearch - 6.x/7.x

## 前言

一个分布式实时文档存储，所有字段可以被索引与搜索

ES查询服务，ES数据的同步，mapping、索引的更新，隔离对业务的入侵

## 核心概念

- 索引（Index）
- 类型（Type）
- 文档（Document）
- 字段（Field）
- 映射（Mapping）

### 索引（Index）

文档的集合，就是数据库

### 类型（Type）

| 版本 | Type                             |
| ---- | -------------------------------- |
| 5.x  | 支持多种type                     |
| 6.x  | 只能有一种type                   |
| 7.x  | 不在支持自定义索引类型，默认_doc |

### 文档（Document）

一个Index中，包含多个文档，索引一篇文档时，Index -> Type -> Document Id，基础信息单元，也是一条数据，以JSON格式存在

### 字段（Field）

文档的字段

### 映射（Mapping）

某个字段的数据类型、默认值、分析器、是否被索引

- 核⼼数据类型
- 复杂数据类型
- 专⽤数据类型

#### 核心数据

##### 字符串数据类型

- text  ⽤于全⽂索引（analyzed），搜索时会自动使用分词器进⾏分词再匹配
- keyword  不分词（not-analyzed），搜索时需要**匹配完整的值**，可以用作过滤、排序、聚合

##### 数值数据类型

- 整型： byte，short，integer，long
- 浮点型： float，half_float，scaled_float，double

##### 日期类型

保存的日期类型多样，内部将时间转为UTC，然后按照`millseconds-since-the-epoch`的长整型来存储，例如：

- 2022-03-03
- 2022-03-03 11:12:13
- 2022-03-03T11:12:13Z
- 1604672099958

##### Boolean

- boolean   

  "true"、"false"、true、false



#### 复合数据

##### 对象类型（Object）

- object

```json
#定义mapping
"user" : {
    "type":"object"
}


#插入|更新字段的值，值写成json对象的形式
"user" : {
    "name":"chy",
    "age":12
}


#搜索时，字段名使用点号连接
"match":{
     "user.name":"chy"
 }
```

##### 嵌套类型（nested）

- nested

是object的一种特例

```json
{
	{
		"user.first": "Zhang",
		"user.last": "san"
	}, {
		"user.first": "Li",
		"user.last": "si"
	}
}
```



##### Array

没有专门的数组类型，定义mapping，写成元素的类型

> 数值中的元素必须是同一种类型

```json
#ES没有专门的数组类型，定义mapping，写成元素的类型
"arr" : {
    "type":"integer"
}


#插入|更新字段的值。元素可以是各种类型，但元素的类型要相同
"arr" : [1,3,4]
```



#### 专用数据

##### GEO 地理位置相关类型

使用场景：

- 查找某一范围内的地理位置
- 通过地理位置或相对中心点的距离聚合文档
- 把距离整合到文档的评分中
- 通过距离对文档进行排序

###### geo_point

一个坐标点

```json
PUT people
{
  "mappings": {
    "properties": {
      "location":{
        "type": "geo_point"
      }
    }
  }
}
```

存储的时候：

```json
PUT people/_doc/1
{
  "location":{
    "lat": 34.27,
    "lon": 108.94
  }
}

PUT people/_doc/2
{
  "location":"34.27,108.94"
}

PUT people/_doc/3
{
  "location":"uzbrgzfxuzup"
}

PUT people/_doc/4
{
  "location":[108.94,34.27]
}
```

> 使用数组描述，先经度后纬度

###### geo_shape

```json
PUT people
{
  "mappings": {
    "properties": {
      "location":{
        "type": "geo_shape"
      }
    }
  }
}
```

添加文档的时候需要指定具体的类型：`point`

```
PUT people/_doc/1
{
  "location":{
    "type":"point",
    "coordinates": [108.94,34.27]
  }
}
```

`linestring`：

```json
PUT people/_doc/2
{
  "location":{
    "type":"linestring",
    "coordinates": [[108.94,34.27],[100,33]]
  }
}
```



### 倒排索引



## ES和MySQL的对应关系

| MySQL              | ES               |
| ------------------ | ---------------- |
| Database（数据库） | Index（索引）    |
| Table（表）        | Type（类型）     |
| Row（行）          | Document（文档） |
| Column（列）       | Field（字段）    |
| Schema（方案）     | Mapping（映射）  |
| Index（索引）      | 所有字段都被索引 |
| select *           | GET  http://     |
| update *           | PUT  http://     |
| delete *           | DELETE  http://  |
| 索引               | 全文索引         |



## 数据结构

索引，类似数据库的“数据库”

```json
{
    "took": 4,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "skipped" : 0,
    	"failed" : 0
    },
    "hits": {
        "total": 76,
        "max_score": 1.0,
        "hits": [
            {
                "_index": "order_20210719",
                "_type": "_doc",
                "_id": "100000000001175380",
                "_score": 1.0,
                "_routing": "100000000000290432",
                "_source": {},
                "sort": [
                    1627291267000,
                    "注册地址"
                ]
            }
        ]
    }
}
```

参数说明：

- took 表示整个搜索所消耗的时间ms

- timed_out 是否超时

**_shards** 分片信息

- _shards.total 分片的数量
- _sahrds.successful 成功返回结果的分片数量
- _shards.failed 失败的分片数量

**hits**

- hits.total 返回查询数据的总数
- hits.max_score 分数越大排位越靠前

**hits.hits**

- _index 索引，文档存储的地方
- _type 属性，高版本默认`_doc`
- _id 唯一标识
- _score
- _source 结果值JSON
- _routing，路由，默认使用文档的ID字段：`_id`。如果自定义了routing字段，增删改查接口都要加上routing参数以保证一致性。作用：减少查询时扫描shard的个数，从而提供查询效率。**但容易造成负载不均衡**。

> ES提供了一种机制可以将数据路由到一组shard上，而不是某一个。创建索引时，设置`index.routing_partition_size = 1`，即只路由到1个shard上。可以将其设置为大于1且小于索引shard总数的某个值，就可以路由到一组shard。值越大，数据越均匀。

```
shard_num = (hash(_routing) + hash(_id) % routing_partition_size) % num_primary_shards
```

注意：

1. 索引的`mapping`中不能再定义join关系的字段，原因是join强制要求关联的doc必须路由到同一个shard，如果采用shard集合，这个是保证不了的。

2. 索引`mapping`中`_routing`的required必须设置为true。

   

### 索引

#### 创建索引

```json
## 创建空索引
PUT /hao

## 创建索引,指定别名
PUT /hao/_alias/haoIndex
{
	"settings": {
		"index": {
			"number_of_shards": "2",
			"number_of_replicas": "0"
		}
	},
	"mappings": {
		"_doc": {
			"properties": {
				"name": {
					"type": "keyword"
				},
				"age": {
					"type": "long"
				},
				"address": {
					"type": "text"
				},
				"birthday": {
					"type": "date",
					"format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
				}
			}
		}
	}
}
```

#### 查看索引

```json
GET /bookdb_index/_settings

{
  "bookdb_index" : {
    "settings" : {
      "index" : {
        "creation_date" : "1627916851099",
        "number_of_shards" : "1",
        "number_of_replicas" : "1",
        "uuid" : "CE8SwKDQQ_WXuGiEViyhtQ",
        "version" : {
          "created" : "7090399"
        },
        "provided_name" : "bookdb_index"
      }
    }
  }
}

GET /bookdb_index/_mapping

```

#### 更新索引

```json
PUT /hao/_settings
{
  "number_of_replicas": 2
}
```

#### 复制索引

索引复制，只会复制数据，`不会复制索引配置`。复制的时候，可以添加查询条件。

```json
POST _reindex
{
  "source": {"index":"hao"},
  "dest": {"index":"hao_new"}
}
```

#### 索引别名

​		生产提供服务的索引，切记使用别名提供服务，而不是直接暴露索引名称，避免后续因为业务变更或者索引数据需要reindex等情况造成业务中断。

- `add`可以为索引创建别名，如果这个别名是唯一的，该别名可以代替索引名称。

```json
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "hao",
        "alias": "hao_alias"
      }
    }
  ]
}
```

- `remove`移除索引别名

```json
POST /_aliases
{
  "actions": [
    {
      "remove": {
        "index": "hao",
        "alias": "hao_alias"
      }
    }
  ]
}
```

- 查看索引别名

```json
GET /hao/_alias

{
  "hao" : {
    "aliases" : {
      "haoIndex" : { },
      "hao_alias" : { }
    }
  }
}
```

#### 删除索引

```json
## 索引名称
DELETE /hao
```

#### 索引打开/关闭

- 索引打开

```
POST /hao/_open
```

- 索引关闭

```
POST /hao/_close
```



### 创建映射

```json
GET /bookdb_index/_mapping

{
  "bookdb_index" : {
    "mappings" : {
      "properties" : {
        "authors" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "num_reviews" : {
          "type" : "long"
        },
        "publish_date" : {
          "type" : "date"
        },
        "publisher" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "summary" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "title" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
      }
    }
  }
}

```



### 新增数据

#### 单条数据

```json
## 新增数据,id=10
POST /bookdb_index/_doc/10
{
	"title": "Elasticsearch: The Definitive Guide",
	"authors": [
		"clinton gormley",
		"zachary tong"
	],
	"summary": "A distibuted real-time search and analytics engine",
	"publish_date": "2015-02-07",
	"num_reviews": 20,
	"publisher": "oreilly"
}
```

#### 批量新增

```json
## 批量新增,_bulk
POST /bookdb_index/_bulk

{ "index": { "_id": 1 }}
{ "title": "Elasticsearch: The Definitive Guide", "authors": ["clinton gormley", "zachary tong"], "summary" : "A distibuted real-time search and analytics engine", "publish_date" : "2015-02-07", "num_reviews": 20, "publisher": "oreilly" }
{ "index": { "_id": 2 }}
{ "title": "Taming Text: How to Find, Organize, and Manipulate It", "authors": ["grant ingersoll", "thomas morton", "drew farris"], "summary" : "organize text using approaches such as full-text search, proper name recognition, clustering, tagging, information extraction, and summarization", "publish_date" : "2013-01-24", "num_reviews": 12, "publisher": "manning" }
{ "index": { "_id": 3 }}
{ "title": "Elasticsearch in Action", "authors": ["radu gheorge", "matthew lee hinman", "roy russo"], "summary" : "build scalable search applications using Elasticsearch without having to do complex low-level programming or understand advanced data science algorithms", "publish_date" : "2015-12-03", "num_reviews": 18, "publisher": "manning" }
```

### 更新文档

#### 更新不存在

#### 全数据更新

```
PUT /bookdb_index/_doc/9
{
	"title": "Elasticsearch: The Definitive Guide",
	"authors": [
		"clinton gormley9",
		"zachary tong9"
	],
	"summary": "A distibuted real-time search and analytics engine",
	"publish_date": "2015-02-07",
	"num_reviews": 20
}
```

有，就删除旧文档，新建新文档，没有，就新增文档

> 使用PUT的_doc方法操作， 必须把所有的数据都传入， 否则会丢失数据

#### 部分数据更新

```json
POST /bookdb_index/_update/9
{
	"doc": {
		"title": "Elasticsearch: The Definitive Guide112",
		"authors": [
			"clinton gormley9-1",
			"zachary tong9-1"
		]
	}
}
```

如果指定ID的文档不存在，会报错 'document missing'

#### 搜索更新

先进行查询在进行更新

```json
POST /bookdb_index/_update_by_query
{
  "query": {
    "match": {
      "user": "Aben2"
    }
  },
  "script": {
    "source": "ctx._source.city = params.city;ctx._source.province = params.province;ctx._source.country = params.country",
    "lang": "painless",
    "params": {
      "city": "上海",
      "province": "上海",
      "country": "中国"
    }
  }
}
```

说明：会搜索'user'='Aben2'的文档， 并按照script.source的更新操作， 把source.params的数据更新到文档。

```json
POST edd/_update_by_query
{
  "query": {
    "match": {
      "姓名": "张彬"
    }
  },
  "script": {
    "source": "ctx._source["签到状态"] = params["签到状态"]",
    "lang": "painless",
    "params" : {
      "签到状态":"已签到"
    }
  }
}
```

> 对于那些名字是中文字段的文档来说，在painless语言中，直接打入中文字段名字，并不能被认可。我们可以使用如下的方式来操作：

#### UPSERT

更新或插入，即如果存在则更新文档，否则插入新文档，这类似于mysql中的replace。

使用doc_as_upsert合并到ID为3的文档中，或者如果不存在则插入一个新文档:

```json
POST /twitter/_update/3
{
     "doc": {
       "author": "Albert Paro",
       "title": "Elasticsearch 5.0 Cookbook",
       "description": "Elasticsearch 5.0 Cookbook Third Edition",
       "price": "54.99"
      },
     "doc_as_upsert": true
}
```

### 文档

#### 判断文档是否存在

```json
HEAD  /hao/_doc/1

## 表示_id=1的文档不存在
{"statusCode":404,"error":"Not Found","message":"404 - Not Found"}

## 表示_id=1的文档存在
200 - OK
```



#### 删除一个文档

##### 删除一个指定id的文档

```json
DELETE /hao/_doc/1

## "result" : "not_found",没有此文档
{
  "_index" : "hao",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "result" : "not_found",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 6,
  "_primary_term" : 2
}

## "result" : "deleted",删除成功
{
  "_index" : "hao",
  "_type" : "_doc",
  "_id" : "AbRlJnsBq32WbFllsnEI",
  "_version" : 2,
  "result" : "deleted",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 19,
  "_primary_term" : 2
}
```

##### 先查询后删除

删除sex='男'的所有数据

```json
POST /hao/_delete_by_query
{
  "query": {
    "match": {
      "sex": "男"
    }
  }
}

## 删除 "total" : 5， "deleted" : 5
{
  "took" : 24,
  "timed_out" : false,
  "total" : 5,
  "deleted" : 5,
  "batches" : 1,
  "version_conflicts" : 0,
  "noops" : 0,
  "retries" : {
    "bulk" : 0,
    "search" : 0
  },
  "throttled_millis" : 0,
  "requests_per_second" : -1.0,
  "throttled_until_millis" : 0,
  "failures" : [ ]
}

## 删除 "total" : 0, "deleted" : 0
{
  "took" : 14,
  "timed_out" : false,
  "total" : 0,
  "deleted" : 0,
  "batches" : 0,
  "version_conflicts" : 0,
  "noops" : 0,
  "retries" : {
    "bulk" : 0,
    "search" : 0
  },
  "throttled_millis" : 0,
  "requests_per_second" : -1.0,
  "throttled_until_millis" : 0,
  "failures" : [ ]
}
```



### 结构化查询

主要分为两种类型：精确匹配和全文检索匹配

- 精确匹配：term、exists、term set、range、prefix、ids、wildcard、regexp、fuzzy
- 全文检索：match、match_phrase、multi_match、match_phrase_prefix、query_string

`GET /索引库名/_search`

#### 精确查询(term)

查询的字段只有一个的时候。`[term] query doesn't support multiple fields`

主要用于精确匹配哪些值，比如数字，日期，布尔值或not_analyzed 的字符串（未经分析的文本数据类型）：

```json
{
	"query": {
		"term": {
			"age": 26
		}
	}
}

{
	"query": {
		"term": {
			"date": "2014-09-01"
		}
	}
}

{
	"query": {
		"term": {
			"public": true
		}
	}
}

{
	"query": {
		"term": {
			"tag": "full_text"
		}
	}
}
```

示例：

```json
{
    "query": {
        "term": {
            "address.keyword": {
                "value": "北京市通州区"
            }
        }
    }
}
```

> **QueryBuilders.termQuery()**   精确查询（查询条件不会进行分词，但是查询内容可能会分词，导致查询不到）

#### 精确查询(terms)

查询的字段包含多个的时候，允许指定多个匹配条件。`[terms] query does not support multiple fields`。json中必须包含数组。如果某个字段指定了多个值，那么文档需要一起去做匹配

示例：

```json
{
    "query": {
        "terms": {
            "address.keyword": [
                "北京市丰台区",
                "北京市昌平区",
                "北京市大兴区"
            ]
        }
    }
}
```

> **QueryBuilders.termsQuery()**  多个筛选值在一个字段中进行查询

#### 匹配查询(match)

match查询是一个标准查询，不管需要全文本查询还是精确查询基本上都要用到它

如果使用match查询一个全文本字段，**会对关键词进行分词，然后按照分词匹配查找**。它会在真正查询之前**用分析器先分析**match一下查询字符：

示例：匹配查询全部数据与分页

```json
{
    "query": {
        "match_all": { 
        }
    },
    "from": 0,
    "size": 10,
    "sort": [
        {
            "salary": {
                "order": "asc"
            }
        }
    ]
}
```

代码实现：

```java
QueryBuilders.matchAllQuery();
// 设置分页
searchSourceBuilder.from(0);
searchSourceBuilder.size(3);
// 设置排序
searchSourceBuilder.sort("salary", SortOrder.ASC);
```

如果用match指定了一个确切值，在遇到数字，日期，布尔值或者not_analyzed的字符串时，将会搜索给定的值：

```json
{ "match": { "age": 26 }}
{ "match": { "date": "2014-09-01" }}
{ "match": { "public": true }}
{ "match": { "tag": "full_text" }}
```

匹配查询数据

```json
{
    "query": {
        "match": {
            "address": "通州区"
        }
    }
}
```

`QueryBuilders.matchQuery()`

#### 多值匹配查询(multi_match)

```json
{
  "query": {
    "multi_match": {
      "query": "待查询句子",
      "fields": [字段1, 字段2, 字段3^2],
      "type": "best_fields",
      "operator": "and"
    }
  }
}
```

> `MultiMatchQueryBuilder.multiMatchQuery()`

####  模糊查询(fuzzy)

 模糊查询，fuzzy中有个编辑距离的概念，编辑距离是对两个字符串差异长度的量化，及一个字符至少需要处理多少次才能变成另一个字符

```json
{
  "query": {
    "fuzzy": {
      "address": {
        "value": "安徽",
        "fuzziness": "AUTO",
        "prefix_length": 0,
        "max_expansions": 50,
        "transpositions": true,
        "boost": 1
      }
    }
  }
}
```

- fuzziness：0，1，2和AUTO，默认其实是AUTO
- prefix_length：表示不被“模糊化” 的初始字符数，通过限制前缀的字符数量可以显著降低匹配的词项数量
> `QueryBuilders.fuzzyQuery("name", "三").fuzziness(Fuzziness.AUTO)`

#### 包含查询(exists )

exists 查询可以用于查找文档中是否包含指定字段或没有某个字段，类似于SQL语句中的IS_NULL 条件

示例：

```json
{
    "query": {
        "exists": { #必须包含
        	"field": "card"
        }
    }
}
```

#### 范围查询(range)

按照指定范围查找一批数据，操作符：

- gt: 大于
- gte: 大于等于
- lt: 小于
- lte: 小于等于

示例：

```json
{
    "query": {
        "range": {
            "age": {
                "gte": 30,
                "lte":34
            }
        }
    }
}
```

`QueryBuilders.rangeQuery("age").gte(30).lte(34)`

#### 多字段查询(multi_match)

```json
{
    "query": {
        "multi_match": {
            "query": "北京",
            "fields": [
                "address",
                "remark"
            ]
        }
    }
}
```

`QueryBuilders.multiMatchQuery()`

#### 通配符查询(wildcard)

`*`或者 `?` 可以放在前面，但不建议这么做。查询的字段可以是text，也可以是keyword类型，其中keyword对大小写敏感，但text类型对于大小写不敏感。

```json
{
    "query": {
        "wildcard": {
            "name.keyword": {
                "value": "*三"
            }
        }
    }
}
```

`QueryBuilders.wildcardQuery("name.keyword", "*三")`

#### 布尔查询(bool)

bool查询可以用来合并多个条件查询结果的布尔逻辑，它包含以下操作符：

- must  多个查询条件的完全匹配，相当于 `and`，计算分值
- must_not  多个查询条件的相反匹配，相当于 `not`
- should  至少有一个查询条件匹配，相当于 `or`，计算分值
- filter 过滤，必须匹配，不计算分值

示例：

```json
{
  "query": {
    "bool": {
      "must": {
        "term": {
          "folder": "inbox"
        }
      },
      "filter": {
        "term": {
          "folder": "inbox1"
        }
      },
      "must_not": {
        "term": {
          "tag": "spam"
        }
      },
      "should": [
        {
          "term": {
            "starred": true
          }
        },
        {
          "term": {
            "unread": true
          }
        }
      ]
    }
  }
}
```

#### 过滤查询(filter)

支持过滤查询，如term、range、match等

示例：

```json
{
  "query": {
    "bool": {
      "filter": {
        "term": {
          "age": 20
        }
      }
    }
  }
}
```

bool查询 - filter查询

查询出生在 1990-1995 年期间，且地址在北京市昌平区、北京市大兴区、北京市房山区 的员工信息：

```json
{
  "query": {
    "bool": {
      "filter": {
        "range": {
          "birthDate": {
            "format": "yyyy",
            "gte": 1990,
            "lte": 1995
          }
        }
      },
      "must": [
        {
          "terms": {
            "address.keyword": [
              "北京市昌平区",
              "北京市大兴区",
              "北京市房山区"
            ]
          }
        }
      ]
    }
  }
}
```

代码实现：

```java
// 创建 Bool 查询构建器
BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
// 构建查询条件
boolQueryBuilder
	.must(QueryBuilders.termsQuery("address.keyword", "北京市昌平区", "北京市大兴区", "北京市房山区"))
    .filter().add(QueryBuilders.rangeQuery("birthDate").format("yyyy").gte("1990").lte("1995"));
```



#### 查询和过滤的对比

**query**查询上下文，会先比较查询条件，然后计算分值，根据分值进行排序，最后返回文档，不会做缓存的。

**filter**过滤器上下文，会先判断是否满足查询条件，不去计算分值，如果不满足会缓存查询结果，满足的话，直接返回缓存结果，会提高性能。

- 一条过滤语句会询问每个文档的字段值是否包含着特定值。
- 查询语句会询问每个文档的字段值与特定值的匹配程度如何。
- 一条查询语句会计算每个文档与查询语句的相关性，会给出一个相关性评分 _score，并且按照相关性对匹配到的文档进行排序。 这种评分方式非常适用于一个没有完全配置结果的全文本搜索。
- 一个简单的文档列表，快速匹配运算并存入内存是十分方便的， 每个文档仅需要1个字节。这些缓存的过滤结果集与后续请求的结合使用是非常高效的。
- 查询语句不仅要查找相匹配的文档，还需要计算每个文档的相关性，所以一般来说查询语句要比过滤语句更耗时，并且查询结果也不可缓存。

> 做精确匹配搜索时，最好用过滤语句，因为过滤语句可以缓存数据



### 查询结果只展示部分字段

```json
GET student/_search

{
  "query": {
    "match": {
      "age": "12"
    }
  },
  "_source": {
    "includes": [
      "name"
    ]
  }
}
```

### 聚合查询

类似关系型数据库中的`group by`

分词作用

ES的聚合查询

#### Bucket（分桶） 

根据字段值，范围或其他条件将文档分组为桶。

#### Metrics（计算）

从字段值计算指标（例如总和或平均值）的指标聚合。

#### Pipeline（管道）

子聚合，从其他聚合（而不是文档或字段）获取输入。

#### Metrix（矩阵）

max/min/avg/sum/stats

terms聚合

cardinality去重

percentiles百分比

percentiles rank

filter聚合

filtters聚合

range聚合

date_range聚合

date_histogram

missing聚合



### 分页查询

#### from + size分页

默认的分页形式，在深度分页的情况下，效率非常低，但是可以随机跳转页面；

es目前支持最大的`max_result_window = 10000`，也就是from+size的大小不能超过10000。

```
Result window is too large, from + size must be less than or equal to: [10000] but was [10001]. See the scroll api for a more efficient way to request large data sets. This limit can be set by changing the [index.max_result_window] index level setting.
```

#### scroll分页

滚动搜索，不用于实时的请求，因为每一个scroll_id占用资源，特别是排序，而且会生成历史快照，对于数据的变更不会体现在快照上。这种方式往往用于非实时处理大量数据的情况，比如要进行数据迁移或者索引变更之类的。

#### search_after分页

滚动搜索，可以在实时的情况下进行深度分页。在Es5.x版本后提供的功能

| 分页方式     | 性能 | 优点                                                 | 缺点                                                         | 使用场景                   |
| ------------ | ---- | ---------------------------------------------------- | ------------------------------------------------------------ | -------------------------- |
| from + size  | 低   | 实现简单                                             | 深度分页问题                                                 | 数据量小，容忍深度分页问题 |
| scroll       | 中   | 解决深度分页问题                                     | 无法反映数据的实时性（快照版本），维护成本高，需要维护一个scroll_id | 海量数据的导出             |
| search_after | 高   | 性能最好，不存在深度分页问题，能够反映数据的实时变更 | 实现复杂，需要有全局唯一的字段，每一次查询都需要上一次查询的结果，并且需要至少指定一个唯一不重复的字段排序 | 海量数据的分页             |



## 关联查询

- 嵌套文档：类似json中的嵌套数组，需要申明字段类型`nested`

- 父子文档：所有文档都是平级的，通过特殊的字段类型`join`表示层级关系



## 集群

### 分布式文档

#### routing路由

对于数据量较大的业务查询场景，ES侧一般会创建多个shard，并将shard分配到集群中的多个实例来分摊压力，正常情况下，一个查询会遍历查询所有的shard，然后将查询到的结果进行merge之后，再返回。

写入的时候设置routing，可以避免每次查询都遍历全量shard，而且查询的时候也指定对应的routingkey，这种情况下，ES会只去查询对应的shard，可以大幅度降低merge数据和调度全量shard的开销。



`routing`是任意字符串，默认使用文档的`_id`；在写入（更新）时，用于计算文档所属分片，在查询（GET请求或指定routing的查询）中用于限制查询范围，提高查询速度。

```
shard_num = hash(_routing) % num_primary_shards
```

该公式可以保证相同routing的文档被分配到同一个shard上，容易导致**数据倾斜**

- `routing_partition_size`   将`routing`相同的文档映射到集群分片的一个子集上，可以减少查询的分片数，也可以在一定程度上防止数据倾斜

```
shard_num = (hash(_routing) + hash(_id) % routing_partition_size) % num_primary_shards
```

#### 数据倾斜



#### routing的使用

- 写入操作

文档的PUT, POST, BULK操作均支持routing参数，在请求中带上`routing=xxx`即可。使用了`routing`值即可保证使用相同routing值的文档被分配到一个或一批分片上。

- GET操作

**对于使用了routing写入的文档，在GET时必须指定routing，否则可能导致404**，这与GET的实现机制有关，GET请求会先根据routing找到对应的分片再获取文档，如果对写入使用routing的文档GET时没有指定routing，那么会默认使用_id进行routing从而大概率无法获得文档。

- 查询操作

查询操作可以在body中指定 `_routing`参数（可以指定多个）来进行查询。当然不指定 `_routing`也是可以查询出结果的，不过是遍历所有的分片，指定了 `_routing`后，查询仅会对routing对应的一个或一批索引进行检索，从而提高查询效率。

- UPDATE或DELETE操作

UPDATE或DELETE操作与GET操作类似，也是先根据routing确定分片，再进行更新或删除操作，因此对于写入使用了routing的文档，必须指定routing，否则会报404响应。

> toB领域：一般routing会用于一个租户（即公司ID）的概念，用了routing起到了租户隔离的作用
>
> toC领域：互联网中的用户数据，可以用userid作为routing，这样就能保证同一个用户的数据全部保存到同一个shard，后面检索的时候，同样使用userid作为routing，就可以精准的从某个shard获取数据了。对于超大数据量的搜索，routing再配合hot&warm的架构，是非常有用的一种解决方案。而且同一种属性的数据写到同一个shard还有很多好处，比如可以提高aggregation的准确性



## 并发的处理方式：锁和版本控制

在 es 中，实际上使用的就是**乐观锁**

### es6.7之前

在 es6.7之前，使用 **version+version_type** 来进行乐观并发控制。文档每被修改一个，version 就会自增一次，es 通过 version 字段来确保所有的操作都有序进行。

version分为内部版本控制和外部版本控制。

#### 内部版本控制

es自己维护的就是内部版本，当创建一个文档时，es 会给文档的版本赋值为 1。

每当用户修改一次文档，版本号就回自增 1。

如果使用内部版本，es 要求 version 参数的值必须和 es 文档中 version 的值相当，才能操作成功。

#### 外部版本控制

也可以维护外部版本。

在添加文档时，就指定版本号：

```json
PUT blog/_doc/1?version=200&version_type=external
{
  "title":"2222"
}
```

以后更新的时候，版本要大于已有的版本号

- vertion_type=external 或者 vertion_type=external_gt ：表示以后更新的时候，版本要大于已有的版本号。
- vertion_type=external_gte：表示以后更新的时候，版本要大于等于已有的版本号。

### es6.7之后

现在使用 `if_seq_no` 和 `if_primary_term` 两个参数来做并发控制。

`seq_no` 不属于某一个文档，它是属于整个索引的（version 则是属于某一个文档的，每个文档的 version 互不影响）。现在更新文档时，使用 `seq_no` 来做并发。由于 `seq_no` 是属于整个 index 的，所以任何文档的修改或者新增，`seq_no` 都会自增。

通过 `seq_no` 和 `primary_term` 来做乐观并发控制：

```json
PUT blog/_doc/2?if_seq_no=5&if_primary_term=1
{
  "title":"6666"
}
```



13、避免宽表

在索引中定义太多字段是一种可能导致映射爆炸的情况，这可能导致内存不足错误和难以恢复的情况，这个问题可能比预期更常见，`index.mapping.total_fields.limit` ，默认值是1000。

## 准实时索引实现

### 溢写到文件系统缓存



### 写translog保障容错



### flush到磁盘



### segment合并

- refresh
- translog
- flush
- compation

## 数据同步方案

#### 业务层代码双写

#### 定时任务同步

#### 基于MySql的增量binlog解析

MySQL binlog表监听 + Canal + MQ

##### 所有表监听

#### 增量同步和全量同步

索引更新方案：

1. 新建带版本号的新索引
2. 暂停增量更新
3. 执行全量数据导入
4. 切换对外别名的指向
5. 删除旧索引
6. 开启增量更新



首次查询从库，未查询到，强制查询主库；
