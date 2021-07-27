# ElassticSearch-6.x

## 返回数据结构

索引，类似数据库的“数据库”

```json
{
    "took": 4,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "failed": 0
    },
    "hits": {
        "total": 76,
        "max_score": null,
        "hits": [
            {
                "_index": "tmsorder_20210719",
                "_type": "tmsorder",
                "_id": "100000000001175380",
                "_score": null,
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
- _type 属性
- _id 唯一标识
- _score
- _source 结果值JSON



## Spring整合ES

### 注解

#### @Document

注解作用在类上，标记实体类为文档对象，常用属性如下：

（1）indexName：对应索引库名称；

（2）type：对应在索引库中的类型；

（3）shards：分片数

（4）replicas：副本数；

#### @Field

作用在成员变量，标记为文档的字段，并制定映射属性；

（1）@Id：作用在成员变量，标记一个字段为id主键；一般id字段或是域不需要存储也不需要分词；

（2）type：字段的类型，取值是枚举，FieldType；

（3）index：是否索引，布尔值类型，默认是true；

（4）store：是否存储，布尔值类型，默认值是false；

（5）analyzer：分词器名称


【 @Field(type = FieldType.Keyword)和 @Field(type = FieldType.Text)区别】
在早期elasticsearch5.x之前的版本存储字符串只有string字段；但是在elasticsearch5.x之后的版本存储了Keyword和Text，都是存储字符串的。
FieldType.Keyword存储字符串数据时，不会建立索引；
FieldType.Text在存储字符串数据的时候，会自动建立索引，也会占用部分空间资源。


【注意】（1）@Field(index=true)表示是否索引，如果是索引表示该字段(或者叫域)能能够搜索。
【注意】（2）@Field(analyzer="ik_max_word",searchAnalyzer="ik_max_word")表示是否分词，如果是分词就会按照分词的单词搜索，如果不是分词就按照整体搜索。
【注意】（3）@Field(store=true)是否存储，也就是页面上显示。**



### QueryBuilder

 * termQuery("key", obj) 完全匹配
 * termsQuery("key", obj1, obj2..)   一次匹配多个值
 * matchQuery("key", Obj) 单个匹配, field不支持通配符, 前缀具高级特性
 * multiMatchQuery("text", "field1", "field2"..);  匹配多个字段, field有通配符忒行
 * matchAllQuery();         匹配所有文件

*** 组合查询**

 * must(QueryBuilders) :   AND
 * mustNot(QueryBuilders): NOT
 * should:                  : OR
 */


 * 只查询一个id的
 * QueryBuilders.idsQuery(String...type).ids(Collection<String> ids)



/**
 * 通配符查询, 支持 * 
 * 匹配任何字符序列, 包括空
 * 避免* 开始, 会检索大量内容造成效率缓慢
 */
 QueryBuilders.wildcardQuery("user", "ki*hy");




 // 命中的记录数
    long totalHits = response.getHits().totalHits();
    
    for (SearchHit searchHit : response.getHits()) {
        // 打分
        float score = searchHit.getScore();
    }



### 结构化查询

#### 精确查询(term)

主要用于精确匹配哪些值，比如数字，日期，布尔值或 not_analyzed 的字符串(未经分析的文本数据类型)：

```json
{ "term": { "age": 26 }}
{ "term": { "date": "2014-09-01" }}
{ "term": { "public": true }}
{ "term": { "tag": "full_text" }}
```

示例：

```
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

**QueryBuilders.termQuery()**   精确查询（查询条件不会进行分词，但是查询内容可能会分词，导致查询不到）

#### 精确查询(terms)

允许指定多个匹配条件。 如果某个字段指定了多个值，那么文档需要一起去做匹配

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

**QueryBuilders.termsQuery()**  多个内容在一个字段中进行查询

#### 匹配查询(match)

match查询是一个标准查询，不管需要全文本查询还是精确查询基本上都要用到它。

如果使用matc 查询一个全文本字段，它会在真正查询之前用分析器先分析match一下查询字符：

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

如果用match指定了一个确切值，在遇到数字，日期，布尔值或者not_analyzed的字符串时，它将为你搜索你给定的值：

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

QueryBuilders.matchQuery()

 3、模糊查询(fuzzy)
 模糊查询所有以 三 结尾的姓名

```
{
    "query": {
        "fuzzy": {
            "name": "三"
        }
    }
}
```


QueryBuilders.fuzzyQuery("name", "三").fuzziness(Fuzziness.AUTO)

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

- gt : 大于
- gte:: 大于等于
- lt : 小于
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

**QueryBuilders.rangeQuery("age").gte(30).lte(34)**

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

 **QueryBuilders.multiMatchQuery()**

#### 通配符查询(wildcard)

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

**QueryBuilders.wildcardQuery("name.keyword", "*三")**

#### 布尔查询(bool)

bool 查询可以用来合并多个条件查询结果的布尔逻辑，它包含一下操作符：

- must  多个查询条件的完全匹配，相当于 and 

- must_not  多个查询条件的相反匹配，相当于 not 

- should  至少有一个查询条件匹配，相当于 or 

示例：

```json
{
    "bool":{
        "must":{
            "term":{
                "folder":"inbox"
            }
        },
        "must_not":{
            "term":{
                "tag":"spam"
            }
        },
        "should":[
            {
                "term":{
                    "starred":true
                }
            },
            {
                "term":{
                    "unread":true
                }
            }
        ]
    }
}
```

### 过滤查询(filter)

支持过滤查询，如term、range、match等

示例：

```json
{
    "query":{
        "bool":{
            "filter":{
                "term":{
                    "age":20
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

- 一条过滤语句会询问每个文档的字段值是否包含着特定值。
- 查询语句会询问每个文档的字段值与特定值的匹配程度如何。
- 一条查询语句会计算每个文档与查询语句的相关性，会给出一个相关性评分 _score，并且 按照相关性对匹
  配到的文档进行排序。 这种评分方式非常适用于一个没有完全配置结果的全文本搜索。
- 一个简单的文档列表，快速匹配运算并存入内存是十分方便的， 每个文档仅需要1个字节。这些缓存的过滤结果集与后续请求的结合使用是非常高效的。
- 查询语句不仅要查找相匹配的文档，还需要计算每个文档的相关性，所以一般来说查询语句要比过滤语句更耗时，并且查询结果也不可缓存。

> 做精确匹配搜索时，最好用过滤语句，因为过滤语句可以缓存数据





八、聚合查询操作示例