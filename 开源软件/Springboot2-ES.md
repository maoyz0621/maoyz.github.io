# ElassticSearch - 6.x、7.x

## 整合SpringBoot

版本对应

|                  Spring Data Release Train                   |                  Spring Data Elasticsearch                   | Elasticsearch | Spring Framework | Spring Boot |
| :----------------------------------------------------------: | :----------------------------------------------------------: | :-----------: | :--------------: | :---------: |
|                       2021.0 (Pascal)                        |                            4.2.1                             |    7.12.1     |      5.3.7       |    2.5.x    |
|                       2020.0 (Ockham)                        |                            4.1.x                             |     7.9.3     |      5.3.2       |    2.4.x    |
|                           Neumann                            |                            4.0.x                             |     7.6.2     |      5.2.12      |    2.3.x    |
|                            Moore                             |                            3.2.x                             |    6.8.12     |      5.2.12      |    2.2.x    |
| Lovelace[[1](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/#_footnotedef_1)] | 3.1.x[[1](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/#_footnotedef_1)] |     6.2.2     |      5.1.19      |    2.1.x    |
| Kay[[1](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/#_footnotedef_1)] | 3.0.x[[1](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/#_footnotedef_1)] |     5.5.0     |      5.0.13      |    2.0.x    |
| Ingalls[[1](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/#_footnotedef_1)] | 2.1.x[[1](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/#_footnotedef_1)] |     2.4.0     |      4.3.25      |    1.5.x    |

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
</dependencies>
```

### 自动装配

| <img src="image\es\config1.png" style="zoom: 80%;" /> |
| :---------------------------------------------------: |
| <img src="image\es\config2.png" style="zoom:80%;" />  |



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


