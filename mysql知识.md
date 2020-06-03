---
typora-copy-images-to: ./
---

# Mysql

## group by

建表：

```mysql
CREATE TABLE `courses` (
    `id` INT(11) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '自增id',
    `student` VARCHAR(255) DEFAULT NULL COMMENT '学生',
    `class` VARCHAR(255) DEFAULT NULL COMMENT '课程',
    `score` INT(255) DEFAULT NULL COMMENT '分数',
    PRIMARY KEY (`id`),
    UNIQUE KEY `course` (`student`, `class`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

插入数据：

```mysql
INSERT INTO `courses`(`student`, `class`, `score`) VALUES('A', 'Math', 90);
INSERT INTO `courses`(`student`, `class`, `score`) VALUES('A', 'Chinese', 80);
INSERT INTO `courses`(`student`, `class`, `score`) VALUES('A', 'English', 70);
INSERT INTO `courses`(`student`, `class`, `score`) VALUES('A', 'History', 80);
INSERT INTO `courses`(`student`, `class`, `score`) VALUES('B', 'Math', 73);
INSERT INTO `courses`(`student`, `class`, `score`) VALUES('B', 'Chinese', 60);
INSERT INTO `courses`(`student`, `class`, `score`) VALUES('B', 'English', 70);
INSERT INTO `courses`(`student`, `class`, `score`) VALUES('B', 'History', 90);
INSERT INTO `courses`(`student`, `class`, `score`) VALUES('C', 'Math', 70);
INSERT INTO `courses`(`student`, `class`, `score`) VALUES('C', 'Chinese', 50);
INSERT INTO `courses`(`student`, `class`, `score`) VALUES('C', 'English', 20);
INSERT INTO `courses`(`student`, `class`, `score`) VALUES('C', 'History', 10);
INSERT INTO `courses`(`student`, `class`, `score`) VALUES('D', 'Math', 53);
INSERT INTO `courses`(`student`, `class`, `score`) VALUES('D', 'Chinese', 32);
INSERT INTO `courses`(`student`, `class`, `score`) VALUES('D', 'English', 99);
INSERT INTO `courses`(`student`, `class`, `score`) VALUES('D', 'History', 100);
```



```mysql
select @@version;
select @@GLOBAL.sql_mode;
-- ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
```

当执行语句时：

```mysql
SELECT * FROM `courses` GROUP BY `class`;
```

报错：

```mysql
[42000][1055] Expression #1 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'aliyun.courses.id' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by
```

问题起因：

ONLY_FUll_GROUP_BY的意思是：对于GROUP BY聚合操作，如果在SELECT中的列，没有在GROUP BY中出现，那么这个SQL是不合法的，因为列不在GROUP BY语句中，也就是说查出来的列必须是GROUP BY之后的字段，或者这个字段出现在聚合函数里面。

解决方法一：使用ANY_VAKLUE()；

```mysql
SELECT any_value(id),any_value(student) FROM `courses` GROUP BY `class`;
```

解决方法二：把ONLY_FULL_GROUP_BY删除

```mysql
SET GLOBAL sql_mode=(SELECT REPLACE(@@sql_mode,'ONLY_FULL_GROUP_BY',''));
```



```mysql
SELECT any_value(id) id, any_value(student) student, any_value(class) class, any_value(score) score FROM `courses` GROUP BY `class`;
```

![image-20200316175352106](.\images\image-20200316175352106.png)
	

```mysql
SELECT any_value(id) id, any_value(student) student, any_value(class) class, any_value(score) score FROM `courses` GROUP BY `score`;
```

![image-20200316175445554](.\images\image-20200316175445554.png)



```mysql
SELECT
    any_value (id) id,
    group_concat(student SEPARATOR ' ||') student,
    group_concat(class ORDER BY class DESC SEPARATOR ' || ') class,
    any_value (score) score
FROM
    `courses`
GROUP BY
    `score`;
```

|               group_concat(DISTINCT, ORDER BY)               |
| :----------------------------------------------------------: |
| ![image-20200319105005691](.\images\image-20200319105005691.png) |



```mysql
SELECT any_value(id) id, student, class, any_value(score) score FROM `courses` GROUP BY class, student;
```

![image-20200316175635932](.\images\image-20200316175635932.png)


```mysql
SELECT any_value(id) id, student, class, any_value(score) score FROM `courses` GROUP BY student, class;
```

![image-20200316180009458](.\images\image-20200316180009458.png)

```mysql
-- 我们需要学生的成绩表，且每个学生每科的成绩按照由大到小的顺序排列
-- 错误结果
SELECT any_value(id) id, student, class, any_value(score) score FROM `courses` GROUP BY `student`,`class` ORDER BY `score` DESC;

-- 正确结果
SELECT any_value(id) id, student, class, any_value(score) score FROM `courses` GROUP BY `student`,`class` ORDER BY `student`,`score` DESC;
```

![image-20200316181242659](.\images\image-20200316181242659.png)



![image-20200316181317503](.\images\image-20200316181317503.png)

 


```mysql
-- 所有功课（除历史课）平均分达到60分的同学和他们的均分：

SELECT `student`, AVG(`score`) AS `avg_score`
FROM `courses`
GROUP BY `student`
HAVING AVG(`score`) >= 60
ORDER BY `avg_score` DESC;
```

SQL执行顺序：

`from` ... `where` ... `group by` ... `order by` ... `select`，`order by` 作用在整个记录，而不是每个分组上。

数据库编码需要使用utf8mb4 ，排序规则需要使用 utf8mb4_general_ci



## 多表update

```mysql
UPDATE table1 t1
JOIN table2 t2 ON t2.order_id = t1.order_id
SET t1.finish_order = 0,
t2.profit_status = 0
WHERE
t1.order_id = 5475175;
```



## 空值和null

+ 空值不占空间
+ For MyISAM tables, each NULL column takes one bit extra, rounded up to the nearest byte.
+ NOT NULL的字段不能插入NULL，只能插入"空值"
+ B树索引不会存储NULL值
+ 字段尽量NOT NULL
+ count()统计某列记录数，会忽略掉NULL，但"空值"会统计
+ 判断`NULL` 用`IS NULL` 或者 `IS NOT NULL`, `SQL`语句函数中可以使用`ifnull()`函数来进行处理，判断空字符用`=''`或者 `<>''`来进行处理
+ 对于`timestamp`数据类型，如果往这个数据类型插入的列插入`NULL`值，则出现的值是当前系统时间。插入空值，则会出现 `0000-00-00 00:00:00`

```mysql
DROP TABLE IF EXISTS `test_01`;
CREATE TABLE `test_01` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `create_time` timestamp DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=17 DEFAULT CHARSET=utf8mb4;

-- 执行插入
INSERT INTO test_01 (create_time, update_time) VALUES (NULL, '');
```

```mysql
-- 更新字段create_time NOT NULL
alter table test_01 modify create_time timestamp not null default CURRENT_TIMESTAMP;
```

|                     更新字段前 查询结果                      |                     更新字段后 查询结果                      |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| ![image-20200320100020644](.\images\image-20200320100020644-1588128386148.png) | ![image-20200320100715305](.\images\image-20200320100715305.png) |

+ NULL 和 '空值'的length

  |                                                              |
  | :----------------------------------------------------------: |
  | ![image-20200320101126970](.\images\image-20200320101126970.png) |

  



## 索引

### 联合索引

+ 假设某个表有一个联合索引（c1,c2,c3,c4）以下选项哪些字段使用了该索引：
  
  
  
  ```sql
  A where c1=x and c2=x and c4>x and c3=x        四个字段均使用了该索引
  B where c1=x and c2=x and c4=x order by c3     c1，c2字段使用了该索引
  C where c1=x and c4= x group by c3,c2          c1字段使用该索引
  D where c1=? and c5=? order by c2,c3           c1字段使用该索引
  E where c1=? and c2=? and c5=? order by c2,c3  c1，c2字段使用了该索引
  ```



普通索引：idx_  唯一索引：uniq_  主键：pk_

索引的最左原则（左前缀原则），如（c1,c2,c3,c4....cN）的联合索引，where 条件按照索引建立的字段顺序来使用（不代表and条件必须按照顺序来写），如果中间某列没有条件，或使用like会导致后面的列不能使用索引。例如索引是key index (a,b,c). 可以支持**a** | **a,b**| **a,b,c** 3种组合进行查找，但不支持 b,c进行查找 .当最左侧字段是常量引用时，索引就十分有效。

如果where条件中是**OR**关系，加索引不起作用

如果我们分别创建**单个索引的话，mysql查询每次只能使用**一个索引

只要列中包含有NULL值都将不会被包含在索引中，复合索引中只要有**一列**含有**NULL值**，那么这一列对于此复合索引就是**无效**的。所以我们在数据库设计时不要让字段的默认值为NULL。

mysql查询只使用一个索引，因此如果where子句中已经使用了索引的话，那么order by中的列是不会使用索引的。因此数据库默认排序可以符合要求的情况下不要使用排序操作；尽量不要包含**多个列的排序**，如果需要最好给这些列创建**复合索引**。

一般情况下不鼓励使用like操作，如果非使用不可，如何使用也是一个问题。like “%aaa%” 不会使用索引而**like “aaa%”**可以使用索引。

NOT IN可以使用**NOT EXISTS代替**

order by 和group by 类似，字段顺序与索引一致时，会使用索引排序；字段顺序与索引不一致时，不使用索引。

索引也能用于分组和排序，分组要先排序，在计算平均值等等。所以在分组和排序中，如果字段顺序可以按照索引的字段顺序，即可利用索引的有序特性



## 事务特性

###  隔离性 - isolation

#### 隔离性级别

##### 1、read uncommitted 读未提交

```
- 事务A和事务B，事务A未提交的数据，事务B可以读取到
- 这里读取到的数据叫做“脏数据”
- 这种隔离级别最低，这种级别一般是在理论上存在，数据库隔离级别一般都高于该级别
```

##### 2、read committed 读已提交

```
- 事务A和事务B，事务A提交的数据，事务B才能读取到
- 这种隔离级别高于读未提交
- 换句话说，对方事务提交之后的数据，我当前事务才能读取到
- 这种级别可以避免“脏数据”
- 这种隔离级别会导致“不可重复读取”
- Oracle默认隔离级别
```



##### 3、repeatable read 可重复读

```
- 事务A和事务B，事务A提交之后的数据，事务B读取不到
- 事务B是可重复读取数据
- 这种隔离级别高于读已提交
- 换句话说，对方提交之后的数据，我还是读取不到
- 这种隔离级别可以避免“不可重复读取”，达到可重复读取
- 比如1点和2点读到数据是同一个
- MySQL默认级别
- 虽然可以达到可重复读取，但是会导致“幻像读”
```

##### 4、 serializable 串行化 

```
- 事务A和事务B，事务A在操作数据库时，事务B只能排队等待
- 这种隔离级别很少使用，吞吐量太低，用户体验差
- 这种级别可以避免“幻像读”，每一次读取的都是数据库中真实存在数据，事务A与事务B串行，而不并发
```

#### 设置隔离级别

##### 1、配置文件my.ini

```
– READ-UNCOMMITTED
– READ-COMMITTED
– REPEATABLE-READ
– SERIALIZABLE

[mysqld]
transaction-isolation = READ-COMMITTED
```


##### 2、动态设置

```
•   事务隔离级别的作用范围分为两种： 
–   全局级：对所有的会话有效 
–   会话级：只对当前的会话有效 
•   例如，设置会话级隔离级别为READ COMMITTED ：
mysql> SET TRANSACTION ISOLATION LEVEL READ COMMITTED；
或：
mysql> SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED；
•   设置全局级隔离级别为READ COMMITTED ： 
mysql> SET GLOBAL TRANSACTION ISOLATION LEVEL READ COMMITTED；
```

