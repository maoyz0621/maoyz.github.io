# ElassticSearch - 6.x、7.x

## 整合SpringBoot

版本对应

| Spring Data Release Train | Spring Data Elasticsearch | Elasticsearch | Spring Framework | Spring Boot |
| :-----------------------: | :-----------------------: | :-----------: | :--------------: | :---------: |
|      2021.0 (Pascal)      |           4.2.1           |    7.12.1     |      5.3.7       |    2.5.x    |
|      2020.0 (Ockham)      |           4.1.x           |     7.9.3     |      5.3.2       |    2.4.x    |
|          Neumann          |           4.0.x           |     7.6.2     |      5.2.12      |    2.3.x    |
|           Moore           |           3.2.x           |    6.8.12     |      5.2.12      |    2.2.x    |
|         Lovelace          |           3.1.x           |     6.2.2     |      5.1.19      |    2.1.x    |
|            Kay            |           3.0.x           |     5.5.0     |      5.0.13      |    2.0.x    |
|          Ingalls          |           2.1.x           |     2.4.0     |      4.3.25      |    1.5.x    |

### 依赖

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starters</artifactId>
    <version>2.2.6.RELEASE</version>
</parent>

<properties>
    <!-- 指定ES版本7.9.3 -->
    <elasticsearch.version>7.9.3</elasticsearch.version>
    <springdata.commons.version>2.4.0</springdata.commons.version>
</properties>

<dependencies>

    <!-- 默认elasticsearch 6.8.7版本 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        <version>${elasticsearch.version}</version>
        <exclusions>
            <exclusion>
                <!-- 依赖的spring-data-commons版本为：2.2.6.RELEASE 同parent保持一致 -->
                <groupId>org.springframework.data</groupId>
                <artifactId>spring-data-commons</artifactId>
            </exclusion>
        </exclusions>
    </dependency>

    <!-- 指定version 2.4.0 -->
    <dependency>
        <groupId>org.springframework.data</groupId>
        <artifactId>spring-data-commons</artifactId>
        <version>${springdata.commons.version}</version>
    </dependency>
    
    <dependency>
      <groupId>org.elasticsearch.client</groupId>
      <artifactId>elasticsearch-rest-high-level-client</artifactId>
      <version>${elasticsearch.version}</version>
      <exclusions>
        <exclusion>
          <groupId>commons-logging</groupId>
          <artifactId>commons-logging</artifactId>
        </exclusion>
      </exclusions>
    </dependency>

    <dependency>
      <groupId>org.springframework.data</groupId>
      <artifactId>spring-data-elasticsearch</artifactId>
      <version>4.1.0</version>
      <scope>compile</scope>
      <exclusions>
        <exclusion>
          <artifactId>jcl-over-slf4j</artifactId>
          <groupId>org.slf4j</groupId>
        </exclusion>
        <exclusion>
          <artifactId>log4j-core</artifactId>
          <groupId>org.apache.logging.log4j</groupId>
        </exclusion>
      </exclusions>
    </dependency>
</dependencies>
```

### 自动装配

| <img src="..\image\es\config1.png" style="zoom: 80%;" /> |
| :------------------------------------------------------: |
| <img src="..\image\es\config2.png" style="zoom:80%;" />  |



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

（4）store：是否存储，布尔值类型，默认值是false；这意味着可以查询该字段，但是无法检索原始字段值。在这里我们必须理解的一点是: 如果一个字段的mapping中含有store属性为true，那么有一个单独的存储空间为这个字段做存储，而且这个存储是独立于`_source`的存储的。它具有更快的查询。存储该字段会占用磁盘空间。如果需要从文档中提取（即在脚本中和聚合），它会帮助减少计算。在聚合时，具有store属性的字段会比不具有这个属性的字段快。 此选项的可能值为false和true。

（5）analyzer：分词器名称

【 **@Field(type = FieldType.Keyword)**和 **@Field(type = FieldType.Text)**区别】
在早期elasticsearch5.x之前的版本存储字符串只有string字段；但是在elasticsearch5.x之后的版本存储了Keyword和Text，都是存储字符串的。
`FieldType.Keyword`存储字符串数据时，不会建立索引；
`FieldType.Text`在存储字符串数据的时候，会自动建立索引，也会占用部分空间资源。

【注意】（1）@Field(index=true)表示是否索引，如果是索引表示该字段(或者叫域)能能够搜索。
【注意】（2）@Field(analyzer="ik_max_word",searchAnalyzer="ik_max_word")表示是否分词，如果是分词就会按照分词的单词搜索，如果不是分词就按照整体搜索。
【注意】（3）@Field(store=true)是否存储，也就是页面上显示。

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

## 整合Java

### spring-data整合elasticsearch



#### 实体类

创建语句：

```json
PUT /biz_user
{
  "settings": {
    "index": {
      "number_of_shards": "2",
      "number_of_replicas": "2"
    }
  },
  "mappings": {
    "properties": {
      "address": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "age": {
        "type": "long"
      },
      "birth": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
      },
      "birth1": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
      },
      "birth2": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
      },
      "birth3": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
      },
      "familyMembers": {
        "properties": {
          "father": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword",
                "ignore_above": 256
              }
            }
          },
          "mother": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword",
                "ignore_above": 256
              }
            }
          },
          "wife": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword",
                "ignore_above": 256
              }
            }
          }
        }
      },
      "height": {
        "type": "float"
      },
      "id": {
        "type": "long"
      },
      "men": {
        "type": "boolean"
      },
      "username": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      }
    }
  }
}
```



```java
/**
 * Copyright 2019 asiainfo Inc.
 **/
package com.myz.springboot2elasticsearch.entity;

import com.myz.springboot2elasticsearch.es.constants.BizEsConstant;
import lombok.Data;
import lombok.experimental.Accessors;
import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.DateFormat;
import org.springframework.data.elasticsearch.annotations.Document;
import org.springframework.data.elasticsearch.annotations.Field;
import org.springframework.data.elasticsearch.annotations.FieldType;

import java.io.Serializable;
import java.util.Date;
import java.util.List;

/**
 * elasticSearch启用 @Document(indexName = "maoyz", type = "user", createIndex = true)
 *
 * @author maoyz0621 on 19-1-30
 * @version v1.0
 */
@Document(indexName = BizEsConstant.INDEX_USER, createIndex = true, shards = 2, replicas = 2)
@Data
@Accessors(chain = true)
public class UserEntity implements Serializable {
    private static final long serialVersionUID = 3217448931285622026L;

    /**
     * 使用@Id
     * 表示该字段同时会被映射到Elasticsearch中document的_id字段（也就是说在Elastisearch中_id的值与id值一样）
     */
    @Id
    @Field(type = FieldType.Keyword)
    private Long id;

    @Field(type = FieldType.Text, analyzer = "")
    private String username;

    @Field(type = FieldType.Integer)
    private Integer age;

    @Field(type = FieldType.Text)
    private String address;

    @Field(type = FieldType.Boolean)
    private boolean men;

    @Field(type = FieldType.Double)
    private double height;

    /**
     * 保存时间类型
     * Property UserEntity.birth is annotated with FieldType.Date but has no DateFormat defined
     * Property UserEntity.birth is annotated with FieldType.Date and a custom format but has no pattern defined
     *
     */
    @Field(type = FieldType.Date, format = DateFormat.basic_date_time,store = true)
    private Date birth;


    @Field(type = FieldType.Auto)
    private List<String> goods;

    @Field(type = FieldType.Long)
    private Long userId;

    @Field(type = FieldType.Object)
    private FamilyMembersEntity familyMembers;
}

```

对于时间类型，

错误1：

```java
@Field(type = FieldType.Date, format = DateFormat.custom, store = true)
private Date birth;
```

> Property UserEntity.birth is annotated with FieldType.Date and a custom format but has no pattern defined

错误2：

```java
@Field(type = FieldType.Date, format = DateFormat.none, store = true)
private Date birth;
```

> Property UserEntity.birth is annotated with FieldType.Date but has no DateFormat defined



Caused by: ElasticsearchException[Elasticsearch exception [type=illegal_argument_exception, reason=failed to parse date field [Mar 1, 2022, 9:09:00 PM] with format [basic_date_time]]]; nested: ElasticsearchException[Elasticsearch exception [type=date_time_parse_exception, reason=Failed to parse with all enclosed parsers]];

```
"birth" : {
  "type" : "text",
  "fields" : {
    "keyword" : {
      "type" : "keyword",
      "ignore_above" : 256
    }
  }
}
```

日期类型birth被映射为“text”类型，并且有子类型“keyword”，失去了日期类型的能力。

WHY?

```json
PUT /biz_user
{
  "settings": {
    "index": {
      "number_of_shards": "2",
      "number_of_replicas": "2"
    }
  },
  "mappings": {
    "properties": {
      "address": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "age": {
        "type": "long"
      },
      "birth": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
      },
      "birth1": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
      },
     
      "familyMembers": {
        "properties": {
          "father": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword",
                "ignore_above": 256
              }
            }
          },
          "mother": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword",
                "ignore_above": 256
              }
            }
          },
          "wife": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword",
                "ignore_above": 256
              }
            }
          }
        }
      },
      "height": {
        "type": "float"
      },
      "id": {
        "type": "long"
      },
      "men": {
        "type": "boolean"
      },
      "username": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      }
    }
  }
}
```

### ElasticsearchRestTemplate



### RestHighLevelClient



### ElasticsearchRepository

#### index/save

底层调用：

```
org.springframework.data.elasticsearch.repository.support.SimpleElasticsearchRepository#save()
	org.springframework.data.elasticsearch.core.ElasticsearchRestTemplate#index()
```

```
org.springframework.data.elasticsearch.UncategorizedElasticsearchException: Elasticsearch exception [type=cluster_block_exception, reason=index [biz_user] blocked by: [TOO_MANY_REQUESTS/12/disk usage exceeded flood-stage watermark, index has read-only-allow-delete block];]; nested exception is ElasticsearchStatusException[Elasticsearch exception [type=cluster_block_exception, reason=index [biz_user] blocked by: [TOO_MANY_REQUESTS/12/disk usage exceeded flood-stage watermark, index has read-only-allow-delete block];]]
```

**TOO_MANY_REQUESTS/12/disk usage exceeded flood-stage watermark, index has read-only-allow-delete block**

原因：短时间内的请求次数过多引起ES节点服务器内存超过显示，主动给索引上锁

解决办法：关闭索引的只读状态

```
PUT _all/_settings
{
  "index.blocks.read_only_allow_delete": fasle
}
```





org.springframework.data.elasticsearch.core.AbstractElasticsearchTemplate#save(T, org.springframework.data.elasticsearch.core.mapping.IndexCoordinates)