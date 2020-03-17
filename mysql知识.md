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

![image-20200316175352106](D:%5CWork%5CIDEA%5Cmaoyz.github.io%5Cimage-20200316175352106.png)
	

```mysql
SELECT any_value(id) id, any_value(student) student, any_value(class) class, any_value(score) score FROM `courses` GROUP BY `score`;
```

![image-20200316175445554](D:%5CWork%5CIDEA%5Cmaoyz.github.io%5Cimage-20200316175445554.png)



```mysql
SELECT any_value(id) id, student, class, any_value(score) score FROM `courses` GROUP BY class, student;
```

![image-20200316175635932](C:%5CUsers%5Cdell%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20200316175635932.png)


```mysql
SELECT any_value(id) id, student, class, any_value(score) score FROM `courses` GROUP BY student, class;
```

![image-20200316180009458](D:%5CWork%5CIDEA%5Cmaoyz.github.io%5Cimage-20200316180009458.png)

```mysql
-- 我们需要学生的成绩表，且每个学生每科的成绩按照由大到小的顺序排列
-- 错误结果
SELECT any_value(id) id, student, class, any_value(score) score FROM `courses` GROUP BY `student`,`class` ORDER BY `score` DESC;

-- 正确结果
SELECT any_value(id) id, student, class, any_value(score) score FROM `courses` GROUP BY `student`,`class` ORDER BY `student`,`score` DESC;
```

![image-20200316181242659](C:%5CUsers%5Cdell%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20200316181242659.png)



![image-20200316181317503](C:%5CUsers%5Cdell%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20200316181317503.png)

 


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