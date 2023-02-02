

# MongoDB命令使用

## 基本操作

### 数据库

#### 创建数据库

```
use DATABASE_NAME
```

#### 查看数据库

```
show dbs
```

#### 删除数据库

```
> use test

> db.dropDatabase();
< { ok: 1, dropped: 'test' }

```

### 集合

#### 创建集合

```
db.createCollection(collection_name, options)
```

#### 查看所有集合

```
show collections;
```

#### 删除集合

```
> db.collection_name.drop();
< true
```

### 文档

#### 插入文档

```shell
db.collection_name.insertOne(<document>,
   {
      writeConcern: <document>
   })
> db.mydb.insertOne({"a":1},{"writeConcern":1,"ordered":true})
< { acknowledged: true,
  insertedId: ObjectId("62b1e1d0baa0c0416618da9a") }
  
  
db.collection_name.insertMany([ <document 1> , <document 2>, ... ],
   {
      writeConcern: <document>,
      ordered: <boolean>
   })
> db.mydb.insertMany([{"a":"a","b":true}, {"c":123}])
< { acknowledged: true,
  insertedIds: 
   { '0': ObjectId("62b1e38ebaa0c0416618da9d"),
     '1': ObjectId("62b1e38ebaa0c0416618da9e") } }
```

参数说明：

- document：要写入的文档。
- writeConcern：写入策略，默认为 1，即要求确认写操作，0 是不要求。
- ordered：指定是否按顺序写入，默认 true，按顺序写入。



插入数据时，如果没有指定主键，默认创建一个主键，字段固定为`_id`，ObjectId()快熟生成12byte的字节id作为主键。

0 1 2 3 | 4 5 6 | 7 8 | 9 10 11

- 前4个字节：unix时间戳
- 3个字节：机器标识码
- 2个字节：进程PID
- 3个字节：随机数

```
db.collection_name.save({JOSN})
```

如果 _id 主键存在则更新数据，如果不存在就插入数据。该方法新版本中已废弃。

#### 更新文档

全量更新

```
db.collection_name.update(
   <query>,
   <update>,
   {
     upsert: <boolean>,
     multi: <boolean>,
     writeConcern: <document>
   }
)
```

参数说明：

- query：update的查询条件
- update：update的对象和更新的操作符，类似sql中的set语句
- upsert：默认false，不插入
- multi：只更新找到的第一条记录，默认false。若为true，查询的所有记录更新
- writeConcern：抛出异常的级别

局部更新（更新指定字段）

```shell
# deprecated
db.collection_name.update({}, {$set:{"filed": val}})
db.collection_name.updateOne({}, {$set:{"filed": val}})
db.collection_name.updateMany({}, {$set:{"filed": val}})
db.collection_name.bulkWrite({}, {$set:{"filed": val}})
```

#### 删除文档

'DeprecationWarning: Collection.remove() is deprecated. Use deleteOne, deleteMany, findOneAndDelete, or bulkWrite.

```
db.collection_name.remove(
   <query>,
   {
     justOne: <boolean>,
     writeConcern: <document>
   }
)

> db.mydb.remove({"c":124})
< { acknowledged: true, deletedCount: 3 }

db.collection_name.deleteOne(<query>)
db.collection_name.deleteMany(<query>)
db.collection_name.findOneAndDelete(<query>)
```



#### 查询文档

```
db.collection_name.find(query, projection)

> db.mydb.find({"a":"a"})
< { _id: ObjectId("62b1e38ebaa0c0416618da9d"), a: 'a', b: true }
  { _id: ObjectId("62b1e781baa0c0416618da9f"), a: 'a', b: true }
  { _id: ObjectId("62b1e783baa0c0416618daa1"), a: 'a', b: true }


db.collection_name.findOne(query, projection)
> db.mydb.find({"a":"a"})
< { _id: ObjectId("62b1e38ebaa0c0416618da9d"), a: 'a', b: true }
```

参数说明：

- query：查询条件
- projection：投影操作符指定返回的键，`0`表示不返回，`1`表示返回

##### 操作符

| 操作       | 格式                     | 范例                                        | RDBMS中的类似语句       |
| :--------- | :----------------------- | :------------------------------------------ | :---------------------- |
| 等于       | `{<key>:<value>`}        | `db.col.find({"by":"菜鸟教程"}).pretty()`   | `where by = '菜鸟教程'` |
| 小于       | `{<key>:{$lt:<value>}}`  | `db.col.find({"likes":{$lt:50}}).pretty()`  | `where likes < 50`      |
| 小于或等于 | `{<key>:{$lte:<value>}}` | `db.col.find({"likes":{$lte:50}}).pretty()` | `where likes <= 50`     |
| 大于       | `{<key>:{$gt:<value>}}`  | `db.col.find({"likes":{$gt:50}}).pretty()`  | `where likes > 50`      |
| 大于或等于 | `{<key>:{$gte:<value>}}` | `db.col.find({"likes":{$gte:50}}).pretty()` | `where likes >= 50`     |
| 不等于     | `{<key>:{$ne:<value>}}`  | `db.col.find({"likes":{$ne:50}}).pretty()`  | `where likes != 50`     |

##### AND

```
db.collection_name.find({key1:value1, key2:value2}).pretty()
```

##### OR

```
db.collection_name.find({
	$or: [{key1: value1}, {key2:value2}]
}).pretty()
```

##### AND 和 OR联合使用

```
db.collection_name.find({"key0": {$gt:value0}, 
	$or: [{"key1": "value1"},{"key2": "value2"}]
}).pretty()
```

#### Limit 和 Skip

查询指定数量的数据

```shell
db.collection_name.find({},{"title":1, _id:0}).limit(num) #limit 0,num
db.collection_name.find({},{"title":1, _id:0}).limit(num1).skip(num2) #limit num2,num1
```

#### sort

排序，sort()，使用`1`和`-1`来指定排序的方式，其中`1`为升序排列，而`-1`是用于降序排列。

```
db.collection_name.find({}).sort({KEY:1})
```



#### $type操作符

基于BSON检索匹配的数据类型，并返回结果

| **类型**                | **数字** | **备注**         |
| :---------------------- | :------- | :--------------- |
| Double                  | 1        |                  |
| String                  | 2        |                  |
| Object                  | 3        |                  |
| Array                   | 4        |                  |
| Binary data             | 5        |                  |
| Undefined               | 6        | 已废弃。         |
| Object id               | 7        |                  |
| Boolean                 | 8        |                  |
| Date                    | 9        |                  |
| Null                    | 10       |                  |
| Regular Expression      | 11       |                  |
| JavaScript              | 13       |                  |
| Symbol                  | 14       |                  |
| JavaScript (with scope) | 15       |                  |
| 32-bit integer          | 16       |                  |
| Timestamp               | 17       |                  |
| 64-bit integer          | 18       |                  |
| Min key                 | 255      | Query with `-1`. |
| Max key                 | 127      |                  |

### 索引

提高查询效率

#### 创建索引

```
db.collection_name.createIndex(keys, options)
> db.collection_name.createIndex({"key1":1, "key2":-1}, {background: true})
```
- 其中`1`为升序创建索引，而`-1`是用于降序创建索引

| Parameter          | Type          | Description                                                  |
| :----------------- | :------------ | :----------------------------------------------------------- |
| background         | Boolean       | 建索引过程会阻塞其它数据库操作，background可指定以后台方式创建索引，即增加 "background" 可选参数。 "background" 默认值为**false**。 |
| unique             | Boolean       | 建立的索引是否唯一。指定为true创建唯一索引。默认值为**false**. |
| name               | string        | 索引的名称。如果未指定，MongoDB的通过连接索引的字段名和排序顺序生成一个索引名称。 |
| dropDups           | Boolean       | **3.0+版本已废弃。**在建立唯一索引时是否删除重复记录,指定 true 创建唯一索引。默认值为 **false**. |
| sparse             | Boolean       | 对文档中不存在的字段数据不启用索引；这个参数需要特别注意，如果设置为true的话，在索引字段中不会查询出不包含对应字段的文档.。默认值为 **false**. |
| expireAfterSeconds | integer       | 指定一个以秒为单位的数值，完成 TTL设定，设定集合的生存时间。 |
| v                  | index version | 索引的版本号。默认的索引版本取决于mongod创建索引时运行的版本。 |
| weights            | document      | 索引权重值，数值在 1 到 99,999 之间，表示该索引相对于其他索引字段的得分权重。 |
| default_language   | string        | 对于文本索引，该参数决定了停用词及词干和词器的规则的列表。 默认为英语 |
| language_override  | string        | 对于文本索引，该参数指定了包含在文档中的字段名，语言覆盖默认的language，默认值为 language. |

#### 查看索引

```
db.collection_name.getIndexes()
```

#### 索引大小

```
db.collection_name.totalIndexSize()
```

#### 删除所有索引

```
db.collection_name.dropIndexes()
```

#### 删除指定索引

```
db.collection_name.dropIndex("index_name")
```

### 聚合

aggregate()

