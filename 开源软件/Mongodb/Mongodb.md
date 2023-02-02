# MongoDB

非关系型数据库，“海量数据库”。

多个文档组成集合，多个集合组成数据库。

一个实例包含多组数据库，相互独立。一个DataBase包含一组Collection，一个集合包含一组Document

与关系型数据库的对应关系：

<table>
  <tr>
    <th></th>
    <th>关系型数据库</th>
    <th>文档型数据库</th>
  </tr>
  <tr>
    <td>模型实体</td>
    <td>表</td>
    <td>集合</td>
  </tr>
  <tr>
    <td rowspan="2">模型属性</td>
    <td>行</td>
    <td>文档</td>
  </tr>
  <tr>
    <td>列</td>
    <td>字段</td>
  </tr>
  <tr>
    <td>模型关系</td>
    <td>表关联</td>
    <td>内嵌数组，引用字段关联</td>
  </tr>
  <tr>
    <td>primary key</td>
    <td>primary key</td>
    <td>将_id自动设置为主键</td>
  </tr>
</table>



## 简介

### 系统数据库

#### admin

权限数据库，创建用户的时候将用户添加到admin中

#### local

表示该数据库不会被复制，可以用来存储**本地单台服务器的任意集合**

#### config

当MongoDB使用了分片模式时，该数据库在内部使用，保存**分片信息**

### 数据类型

| 数据类型           |              |
| ------------------ | ------------ |
| String             |              |
| Integer            |              |
| Boolean            |              |
| Double             |              |
| Min/Max keys       |              |
| Array              |              |
| Timestamp          |              |
| Object             | 内嵌文档     |
| Null               | 空值         |
| Date               |              |
| Object ID          | 创建文档的ID |
| Binary Data        | 二进制数据   |
| Code               | 代码类型     |
| Regular expression | 正则表达式   |

配置文件

mongod.conf

```conf

```



### docker

拉取镜像：

```
docker pull mongo:5.0
```

后台启动：

```
docker run  --name my-mongo -p 27017:27017 -v /data/mongo/db:/data/db  -v /data/mongo:/etc/mongo -d mongo:5.0
```
