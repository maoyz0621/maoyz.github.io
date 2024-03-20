# MySQL - 二进制日志

Mysql存在的二进制日志，**主从复制**和**数据恢复**

- 原子性    undo log
- 持久性    redo log
- 隔离性    加锁和MVCC

## 简介

binlog是记录所有数据库表结构变更（例如CREATE、ALTER TABLE…）以及表数据修改（INSERT、UPDATE、DELETE…）的二进制日志。

> binlog不会记录SELECT和SHOW这类操作，因为这类操作对数据本身并没有修改，但你可以通过查询通用日志来查看MySQL执行过的所有语句。
>
> tips：如果`update`操作没有造成数据变化，也是会记入`binlog`

在`mysql主从复制`中就是依靠的binlog

查看binlog的具体事件类型

```mysql
show binlog events in 'binlogfile'
```

### MySQL binlog的工作模式：

binlog一共有三种工作模式：

#### Row

日志中会记录每一行数据被修改的情况，然后在Slave端对相同的数据进行修改

- 优点：能清楚的记录每一行数据修改的细节
- 缺点：数据量太大

####  Statement（默认）

每一条被修改数据的sql都会记录到master的bin-log中，slave在复制的时候sql进程会解析成和原来master端执行过的相同的sql再次执行。在主从同步中一般是不建议用statement模式的，因为会有些语句不支持，比如语句中包含UUID函数，以及LOAD DATA IN FILE语句等

- 优点：解决了`Row level`的缺点，不需要记录每一行的数据变化，减少bin-log日志量，节约磁盘IO，提高新能
- 缺点：容易出现主从复制不一致

####  Mixed（混合模式）

结合了`Row`和`Statement`的优点，同时binlog结构也更复杂。



- 总结

| format    | 定义                       | 优点                           | 缺点                                                         |
| --------- | -------------------------- | ------------------------------ | ------------------------------------------------------------ |
| statement | 记录的是修改SQL语句        | 日志文件小，节约IO，提高性能   | 准确性差，对一些系统函数不能准确复制或不能复制，如now()、uuid()等 |
| row       | 记录的是每行实际数据的变更 | 准确性强，能准确复制数据的变更 | 日志文件大，较大的网络IO和磁盘IO                             |
| mixed     | statement和row模式的混合   | 准确性强，文件大小适中         | 有可能发生主从不一致问题                                     |



###  binlog文件结构

#### Event_type

事件结构：一个事件对象分为event - header和event - data

```
+=====================================+
| event  | timestamp         0 : 4    |
| header +----------------------------+
|        | type_code         4 : 1    |
|        +----------------------------+
|        | server_id         5 : 4    |
|        +----------------------------+
|        | event_length      9 : 4    |
|        +----------------------------+
|        | next_position    13 : 4    |
|        +----------------------------+
|        | flags            17 : 2    |
|        +----------------------------+
|        | extra_headers    19 : x-19 |
+=====================================+
| event  | fixed part        x : y    |
| data   +----------------------------+
|        | variable part              |
+=====================================+
```

如果事件头的长度是 `x` 字节，那么事件体的长度为 `(event_length - x)` 字节；设事件体中 `fixed part` 的长度为 `y` 字节，那么 `variable part` 的长度为 `(event_length - (x + y))` 字节

**format description event** (size ≥ 91 bytes; the size is 76 + the number of event types)

```
+=====================================+
| event  | timestamp         0 : 4    |
| header +----------------------------+
|        | type_code         4 : 1    | = FORMAT_DESCRIPTION_EVENT = 15
|        +----------------------------+
|        | server_id         5 : 4    |
|        +----------------------------+
|        | event_length      9 : 4    | >= 91
|        +----------------------------+
|        | next_position    13 : 4    |
|        +----------------------------+
|        | flags            17 : 2    |
+=====================================+
| event  | binlog_version   19 : 2    | = 4
| data   +----------------------------+
|        | server_version   21 : 50   |
|        +----------------------------+
|        | create_timestamp 71 : 4    |
|        +----------------------------+
|        | header_length    75 : 1    |
|        +----------------------------+
|        | post-header      76 : n    | = array of n bytes, one byte per event
|        | lengths for all            |   type that the server knows about
|        | event types                |
+=====================================+
```

https://dev.mysql.com/doc/internals/en/binary-log-versions.html



binlog Event_type的结构主要有3个版本：

- v4: 在 MySQL 5.0 及以上版本中使用

| 事件类型                 | 说明                                                         |
| ------------------------ | ------------------------------------------------------------ |
| QUERY_EVENT              | 执行更新语句时会生成此事件，包括：create，insert，update，delete； |
| ROTATE_EVENT             | 当mysqld切换到新的binlog文件生成此事件，切换到新的binlog文件可以通过执行flush logs命令或者binlog文件大于 `max_binlog_size` 参数配置的大小； |
| INTVAR_EVENT             | 当sql语句中使用了AUTO_INCREMENT的字段或者LAST_INSERT_ID()函数；此事件没有被用在binlog_format为ROW模式的情况下 |
| RAND_EVENT               | 执行包含RAND()函数的语句产生此事件，此事件没有被用在binlog_format为ROW模式的情况下 |
| USER_VAR_EVENT           | 执行包含了用户变量的语句产生此事件，此事件没有被用在binlog_format为ROW模式的情况下 |
| FORMAT_DESCRIPTION_EVENT | 描述事件，被写在每个binlog文件的开始位置，用在MySQL5.0以后的版本中，代替了START_EVENT_V3 |
| XID_EVENT                | 支持XA的存储引擎才有，本地测试的数据库存储引擎是innodb，所有上面出现了XID_EVENT；innodb事务提交产生了QUERY_EVENT的BEGIN声明，QUERY_EVENT以及COMMIT声明，如果是myIsam存储引擎也会有BEGIN和COMMIT声明，只是COMMIT类型不是XID_EVENT |
| BEGIN_LOAD_QUERY_EVENT   | 执行LOAD DATA INFILE 语句时产生此事件，在MySQL5.0版本中使用  |
| EXECUTE_LOAD_QUERY_EVENT | 执行LOAD DATA INFILE 语句时产生此事件，在MySQL5.0版本中使用  |
| TABLE_MAP_EVENT          | 用在binlog_format为ROW模式下，将表的定义映射到一个数字，在行操作事件之前记录（包括：WRITE_ROWS_EVENT，UPDATE_ROWS_EVENT，DELETE_ROWS_EVENT） |
| WRITE_ROWS_EVENT         | 用在binlog_format为ROW模式下，对应 insert 操作               |
| UPDATE_ROWS_EVENT        | 用在binlog_format为ROW模式下，对应 update 操作               |
| DELETE_ROWS_EVENT        | 用在binlog_format为ROW模式下，对应 delete 操作               |

binlog写入机制

根据记录模式和操作触发event事件生成log event

将事务执行过程中产生log event写入缓冲区，每个事务线程都有一个缓存区

log event保证在binlog_cache_mngr数据结构中，两个缓冲区，一个是stmt_cache，存放不执行事务的信息，另一个是trx_cache，存放支持事务的信息

事务在提交阶段会将产生的log event写入到外部的binlog中，不同事务以**串行方式**将log event写入binlog文件中，所以一个事务包含的log event信息在binlog文件中是连续的，中间不会插入其它事务的log event



binlog文件操作

binlog状态查看

```mysql
show VARIABLES LIKE 'log_bin';
show VARIABLES LIKE '%log_bin%';

SHOW BINARY logs;

SHOW MASTER status;
```



```mysql
show VARIABLES LIKE '%log_bin%';

+---------------------------------+---------------------------------------------------+
| Variable_name                   | Value                                             |
+---------------------------------+---------------------------------------------------+
| log_bin                         | ON                                                |
| log_bin_basename                | E:\WorkWare\mysql-8.0.13-winx64\data\binlog       |
| log_bin_index                   | E:\WorkWare\mysql-8.0.13-winx64\data\binlog.index |
| log_bin_trust_function_creators | OFF                                               |
| log_bin_use_v1_row_events       | OFF                                               |
| sql_log_bin                     | ON                                                |
+---------------------------------+---------------------------------------------------+
```



```

```



开启binlog功能

windon系统`mysql.ini`文件

```
log_bin=mysqlbinlog
binlog-format=ROW
```



使用`show binlog events`

```mysql
mysql> show binary logs;
+---------------+-----------+
| Log_name      | File_size |
+---------------+-----------+
| binlog.000047 |      6691 |
| binlog.000048 |       155 |
+---------------+-----------+

show binlog events;

mysql> show master logs;
+---------------------+-----------+
| Log_name            | File_size |
+---------------------+-----------+
| mysql-binlog.000001 |       511 |
| mysql-binlog.000002 |       768 |
+---------------------+-----------+

mysql> show binlog events in 'mysql-binlog.000001';
+---------------------+-----+----------------+-----------+-------------+--------------------------------------+
| Log_name            | Pos | Event_type     | Server_id | End_log_pos | Info                                 |
+---------------------+-----+----------------+-----------+-------------+--------------------------------------+
| mysql-binlog.000001 |   4 | Format_desc    |         1 |         124 | Server ver: 8.0.13, Binlog ver: 4    |
| mysql-binlog.000001 | 124 | Previous_gtids |         1 |         155 |                                      |
| mysql-binlog.000001 | 155 | Anonymous_Gtid |         1 |         230 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS' |
| mysql-binlog.000001 | 230 | Query          |         1 |         319 | BEGIN                                |
| mysql-binlog.000001 | 319 | Table_map      |         1 |         388 | table_id: 65 (mybatis.t_user_1)      |
| mysql-binlog.000001 | 388 | Update_rows    |         1 |         457 | table_id: 65 flags: STMT_END_F       |
| mysql-binlog.000001 | 457 | Xid            |         1 |         488 | COMMIT /* xid=312 */                 |
| mysql-binlog.000001 | 488 | Stop           |         1 |         511 |                                      |
+---------------------+-----+----------------+-----------+-------------+--------------------------------------+
8 rows in set (0.02 sec)

mysql> show binlog events in 'mysql-binlog.000002';
+---------------------+-----+----------------+-----------+-------------+--------------------------------------+
| Log_name            | Pos | Event_type     | Server_id | End_log_pos | Info                                 |
+---------------------+-----+----------------+-----------+-------------+--------------------------------------+
| mysql-binlog.000002 |   4 | Format_desc    |         1 |         124 | Server ver: 8.0.13, Binlog ver: 4    |
| mysql-binlog.000002 | 124 | Previous_gtids |         1 |         155 |                                      |
| mysql-binlog.000002 | 155 | Anonymous_Gtid |         1 |         230 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS' |
| mysql-binlog.000002 | 230 | Query          |         1 |         310 | BEGIN                                |
| mysql-binlog.000002 | 310 | Table_map      |         1 |         365 | table_id: 76 (mybatis.user1)         |
| mysql-binlog.000002 | 365 | Write_rows     |         1 |         417 | table_id: 76 flags: STMT_END_F       |
| mysql-binlog.000002 | 417 | Xid            |         1 |         448 | COMMIT /* xid=60 */                  |
| mysql-binlog.000002 | 448 | Anonymous_Gtid |         1 |         523 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS' |
| mysql-binlog.000002 | 523 | Query          |         1 |         612 | BEGIN                                |
| mysql-binlog.000002 | 612 | Table_map      |         1 |         667 | table_id: 76 (mybatis.user1)         |
| mysql-binlog.000002 | 667 | Update_rows    |         1 |         737 | table_id: 76 flags: STMT_END_F       |
| mysql-binlog.000002 | 737 | Xid            |         1 |         768 | COMMIT /* xid=79 */                  |
+---------------------+-----+----------------+-----------+-------------+--------------------------------------+
12 rows in set (0.03 sec)
```



使用`mysqlbinlog log_file `命令查看binlog

1. `statement`格式的二进制文件

```
mysqlbinlog "binlog.000048"
```

2. `row`格式的二进制文件

- -v：展现伪SQL语句

- -vv：显示每列的数据类型和一些元数据

```
mysqlbinlog -vv "binlog.000048"

mysqlbinlog -vv "binlog.000048" > "test.sql"
```



进入data目录中，不要使用mysql客户端：

```shell
[E:\WorkWare\mysql-8.0.13-winx64\data]$ mysqlbinlog "binlog.000047"

# at 4
#210913 23:12:39 server id 1  end_log_pos 124 CRC32 0x8958bfbe  Start: binlog v 4, server v 8.0.13 created 210913 23:12:39 at startup
# Warning: this binlog is either in use or was not closed properly.
ROLLBACK/*!*/;
BINLOG '
Z2o/YQ8BAAAAeAAAAHwAAAABAAQAOC4wLjEzAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAABnaj9hEwANAAgAAAAABAAEAAAAYAAEGggAAAAICAgCAAAACgoKKioAEjQA
CgG+v1iJ
'/*!*/;

# at 124
#210913 23:12:39 server id 1  end_log_pos 155 CRC32 0x79cc77f9  Previous-GTIDs
# [empty]
# at 155
#210913 23:17:41 server id 1  end_log_pos 230 CRC32 0xc4ee33fb  Anonymous_GTID  last_committed=0        sequence_number=1       rbr_only=yes    original_committed_timestamp=1631546261491575   immediate_commit_ti
mestamp=1631546261491575        transaction_length=333
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
# original_commit_timestamp=1631546261491575 (2021-09-13 23:17:41.491575 中国标准时间)
# immediate_commit_timestamp=1631546261491575 (2021-09-13 23:17:41.491575 中国标准时间)
/*!80001 SET @@session.original_commit_timestamp=1631546261491575*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 230
#210913 23:17:41 server id 1  end_log_pos 319 CRC32 0xdb6b3484  Query   thread_id=81    exec_time=0     error_code=0
SET TIMESTAMP=1631546261/*!*/;
SET @@session.pseudo_thread_id=81/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1168113696/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8mb4 *//*!*/;
SET @@session.character_set_client=255,@@session.collation_connection=255,@@session.collation_server=33/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
/*!80011 SET @@session.default_collation_for_utf8mb4=255*//*!*/;
/*!80013 SET @@session.sql_require_primary_key=0*//*!*/;

BEGIN
/*!*/;
# at 319
#210913 23:17:41 server id 1  end_log_pos 388 CRC32 0x70832ac8  Table_map: `mybatis`.`t_user_1` mapped to number 65
# at 388
#210913 23:17:41 server id 1  end_log_pos 457 CRC32 0xb28420dc  Update_rows: table id 65 flags: STMT_END_F

BINLOG '
lWs/YRMBAAAARQAAAIQBAAAAAEEAAAAAAAEAB215YmF0aXMACHRfdXNlcl8xAAQDDw8PBv0C/QL9
Ag4BAYACASHIKoNw
lWs/YR8BAAAARQAAAMkBAAAAAEEAAAAAAAEAAgAE//8MCwAAAAcAc2Fkc2FhcwgLAAAABwBzYWRz
YWFzAwAxMjPcIISy
'/*!*/;
# at 457
#210913 23:17:41 server id 1  end_log_pos 488 CRC32 0xb57157e2  Xid = 312
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

- position：位于文件中的位置，即第一行的（# at 4）,说明该事件记录从文件第4902个字节开始
- timestamp：事件发生的时间戳，即第二行的（#210913 23:12:39）
- server id：服务器标识（1）
- end_log_pos：表示下一个事件开始的位置（即当前事件的结束位置+1）
- thread_id：执行该事件的线程id （thread_id=8）
- exec_time：事件执行的花费时间
- error_code：错误码，0意味着没有发生错误
- type：事件类型Query



使用binlog恢复数据

`mysqldump`全量备份

```shell
## 指定起始时间段，指定的数据库，字符集为utf8或者utfmb4

mysqlbinlog binlog.000047 --start-datetime='2018-07-23 14:00:00' --stop-datetime='2018-07-23 16:10:00' --database=user  --set-charset=utf8 | mysql -uroot -proot


## 指定位置信息
mysqlbinlog binlog.000047 --start-position=200  --stop-position=300 --database=user  --set-charset=utf8  | mysql -uroot -proot
```



1. 删除binlog文件

```mysql
## 删除指定文件
mysql> purge binary logs to "binlog.000047";
Query OK, 0 rows affected (0.01 sec)

## 删除指定时间之前的文件
mysql> purge binary logs before "2021-08-01";
Query OK, 0 rows affected (0.01 sec)

## 清除所有文件
mysql> reset master;
Query OK, 0 rows affected (0.02 sec)
```

2. 通过设置过期时间清除

```mysql
mysql> show variables like '%expire_logs_days%';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| expire_logs_days | 0     |
+------------------+-------+
1 row in set (0.02 sec)
```





**binlog常见参数**

| 参数名                                        | 含义                                           |
| --------------------------------------------- | ---------------------------------------------- |
| log_bin = {on \| off \| base_name}            | 指定是否启用记录二进制日志或者指定一个日志路径 |
| sql_log_bin ={ on \| off }                    | 指定是否启用记录二进制日志                     |
| expire_logs_days                              | 指定自动删除二进制日志的时间，即日志过期时间   |
| log_bin_index                                 | 指定mysql-bin.index文件的路径                  |
| binlog_format = { mixed \| row \| statement } | 指定二进制日志基于什么模式记录                 |
| max_binlog_size                               | 指定二进制日志文件最大值                       |
| binlog_cache_size                             | 指定事务日志缓存区大小                         |
| max_binlog_cache_size                         | 指定二进制日志缓存最大大小                     |
| sync_binlog = { 0 \| n }                      | 指定写缓冲多少次，刷一次盘                     |

主要分为3个步骤：

第一步：master在每次准备提交事务完成数据更新前，将改变记录到二进制日志(binary log)中（这些记录叫做二进制日志事件，binary log event，简称event)

第二步：slave启动一个I/O线程来读取主库上binary log中的事件，并记录到slave自己的中继日志(relay log)中。

第三步：slave还会起动一个SQL线程，该线程从relay log中读取事件并在备库执行，从而实现备库数据的更新。



## redo log

> 事务的持久性，记录的是数据被操作后的样子

包括两部分：内存中的**日志缓存（redo log buffer）**和磁盘上的**日志文件（redo log file）**

每执行一条DML语句，先将记录写入`redo log buffer`，后续某个时间点再一次性将多个操作记录写入`redo log file`，称为WAL（Write-Ahead- Logging）

InnoDB特有的，且日志上的记录落盘后会被覆盖掉。因此需要`binlog`和`redo log`二者同时记录，才能保证当数据库发生宕机重启时，数据不会丢失。

## undo log

记录需要回滚的日志信息。记录的是数据操作前的样子

存储老版本数据。假设修改表中 id=2 的行数据，把 name='B' 修改为 name = 'B2' ，那么 undo 日志就会用来存放 name='B' 的记录，如果这个修改出现异常，可以使用 `undo log`来实现回**滚操**作，保证事务的一致性。

> 事务的原子性，undo log也是MVCC（多版本并发控制）实现的关键

当一个旧的事务需要读取数据时，为了能读取到老版本的数据，需要顺着 undo 链找到满足其可见性的记录。当版本链很长时，通常可以认为这是个比较耗时的操作。

在回滚阶段中undo log分为：`insert undo log` 和 `update undo log`；

- `insert undo log `: 事务对`insert`新记录时产生的`undo log`，只在事务回滚时需要，并且在事务提交后就可以立即丢弃。
- `update undo log` : 事务对记录进行`update`操作时产生的`undo log`。不仅在事务回滚时需要，一致性读也需要，所以不能随便删除，只有当数据库所使用的快照中不涉及该日志记录，对应的回滚日志才会被 **purge 线程**删除。

- `delete undo log`：删除一条记录时，至少要把这条记录中的内容都记下来，这样之后回滚时再把由这些内容组成的记录插入到表中就好了。 
  - 删除操作都只是设置一下老记录的DELETED_BIT，并不真正将过时的记录删除。
  - 为了节省磁盘空间，InnoDB有专门的purge线程来清理DELETED_BIT为true的记录。为了不影响MVCC的正常工作，purge线程自己也维护了一个read view（这个read view相当于系统中最老活跃事务的read view）;如果某个记录的DELETED_BIT为true，并且DB_TRX_ID相对于purge线程的read view可见，那么这条记录一定是可以被安全清除的。

> 对MVCC有用的实质是`update un log`



1. **比如一个有个事务插入persion表插入了一条新记录，记录如下，name为Jerry, age为24岁，隐式主键是1，事务ID和回滚指针，我们假设为NULL**

![](.\image\MySQL\undo-1.png)

2. **现在来了一个事务1对该记录的name做出了修改，改为Tom**
   1. 在事务1修改该行(记录)数据时，数据库会先对该行加排他锁
   2. 然后把该行数据拷贝到undo log中，作为旧记录，即在undo log中有当前行的拷贝副本
   3. 拷贝完毕后，修改该行name为Tom，并且修改隐藏字段的事务ID为当前事务1的ID, 我们默认从1开始，之后递增，回滚指针指向拷贝到undo log的副本记录，即表示我的上一个版本就是它
   4. 事务提交后，释放锁

![](.\image\MySQL\undo-2.png)

3. **又来了个事务2修改person表的同一个记录，将age修改为30岁**
   1. 在事务2修改该行数据时，数据库也先为该行加锁
   2. 然后把该行数据拷贝到undo log中，作为旧记录，发现该行记录已经有undo log了，那么最新的旧数据作为链表的表头，插在该行记录的undo log最前面
   3. 修改该行age为30岁，并且修改隐藏字段的事务ID为当前事务2的ID, 那就是2，回滚指针指向刚刚拷贝到undo log的副本记录
   4. 事务提交，释放锁

![](.\image\MySQL\undo.png)



### 事务特性

#### 隔离性级别

##### Read Uncommitted 读未提交

```
- 事务A和事务B，事务A未提交的数据，事务B可以读取到
- 这里读取到的数据叫做“脏数据”
- 这种隔离级别最低，这种级别一般是在理论上存在，数据库隔离级别一般都高于该级别
```

> 会导致脏读

##### Read Committed 读已提交

```
- 事务A和事务B，事务A提交了数据，事务B才能读取到
- 这种隔离级别高于读未提交
- 换句话说，对方事务提交之后的数据，当前事务才能读取到
- 这种级别可以避免“脏数据”，这种隔离级别会导致“不可重复读取”
- Oracle默认隔离级别
```

> 会导致“不可重复读”

##### Repeatable Read 可重复读

```
- 事务A和事务B，事务A提交之后的数据，事务B读取不到
- 事务B是可重复读取数据
- 这种隔离级别高于读已提交
- 换句话说，对方提交之后的数据，我还是读取不到
- 这种隔离级别可以避免“不可重复读取”，达到可重复读取
- 读不加锁，只有写才加锁，读写互不阻塞，
- MySQL默认级别
- 虽然可以达到可重复读取，但是会导致“幻读”
```

> 会导致“幻读”

##### Serializable 串行化 

```
- 事务A和事务B，事务A在操作数据库时，事务B只能排队等待
- 这种隔离级别很少使用，吞吐量太低，用户体验差
- 这种级别可以避免“幻读”，每一次读取的都是数据库中真实存在数据，事务A与事务B串行，而不并发
```

#### 设置隔离级别

##### 配置文件my.ini

```ini
– READ-UNCOMMITTED
– READ-COMMITTED
– REPEATABLE-READ
– SERIALIZABLE

[mysqld]
transaction-isolation = READ-COMMITTED
```


##### 动态设置

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

## 主从同步

- Master：log dump thread

- Slave：I/O thread、SQL thread

1. **主库写 binlog**：主库的更新 SQL(update、insert、delete) 被写到 binlog；
2. **主库发送binlog**： log dump thread读取binlog变动内容，发送给从库；
3. **从库写relay log**：I/O thread接收binlog内容，写入**relay log**中；
4. **从库回放**：SQL thread读取**relay log**内容并对数据进行重放。

| 主从同步                                                 |
| -------------------------------------------------------- |
| ![MySQL主从复制](..\Mysql\image\MySQL\MySQL主从复制.png) |



默认复制方式：异步复制

- 异步复制
- **半同步复制**
- 组复制

### 延时问题



##  锁操作类型

- 读锁/共享锁
- 写锁/排它锁

## 锁粒度

行锁和表锁

锁监控信息：

```mysql
mysql> show status like 'innodb_row_lock%';
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| Innodb_row_lock_current_waits | 0     |
| Innodb_row_lock_time          | 0     |
| Innodb_row_lock_time_avg      | 0     |
| Innodb_row_lock_time_max      | 0     |
| Innodb_row_lock_waits         | 0     |
+-------------------------------+-------+
```

- Innodb_row_lock_current_waits：当前正在等待锁定的数量；

- Innodb_row_lock_time ：从系统启动到现在锁定总时间长度；（等待总时长）

- Innodb_row_lock_time_avg ：每次等待所花平均时间；（等待平均时长）

- Innodb_row_lock_time_max：从系统启动到现在等待最常的一次所花的时间；

- Innodb_row_lock_waits ：系统启动后到现在总共等待的次数；（等待总次数）

### 表锁Table Lock

#### 意向锁 intention lock

- 意向共享锁

- 意向排它锁

  `SELECT column From Table ... FOR UPDATE`

#### 自增锁 AUTO-INC

插入数据的三种方式

- Simple inserts 简单插入：预先确定要插入的行数，`INSERT ... VALUES()`、`REPLACE`
- Bulk inserts 批量插入：事先不知道要插入的行数，`INSERT ... SELETC`、`REPLACE ... SELECT`
- Mixed-mode inserts 混合插入：指定部分id的值；`INSERT ... ON DUPLICATE KEY UPDATE`

`innode_autoinc_lock_mode`

#### 元数据锁MDL



### 行锁

#### 记录锁 Record Locks

#### 间隙锁 Gap Locks

快照读

当前读

#### 临建锁 Next-Key Locks

#### 插入意向锁 Insert Intention Lock



> 没有索引或者索引失效时，InnoDB 的行锁变表锁

### 页锁



## 锁的态度：乐观锁、悲观锁