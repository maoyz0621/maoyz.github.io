# MySQL - binlog

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

```
show binlog events in 'binlogfile'
```

### MySQL binlog的工作模式：

binlog一共有三种工作模式：

#### Row

日志中会记录每一行数据被修改的情况，然后在slave端对相同的数据进行修改

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

|                                                              |
| ------------------------------------------------------------ |
| <img src="image\MySQL\binlog结构图.png" style="zoom: 45%;" /> |

#### Event_type

事件结构：一个事件对象分为事件头和事件体

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



binlog Event_type的结构主要有3个版本：

- v1: 在 MySQL 3.23 中使用
- v3: 在 MySQL 4.0.2 到 4.1 中使用
- v4: 在 MySQL 5.0 及以上版本中使用

| 事件类型                 | 说明                                                         |
| ------------------------ | ------------------------------------------------------------ |
| UNKNOWN_EVENT            | 此事件从不会被触发，也不会被写入binlog中；发生在当读取binlog时，不能被识别其他任何事件，那被视为UNKNOWN_EVENT |
| QUERY_EVENT              | 执行更新语句时会生成此事件，包括：create，insert，update，delete； |
| STOP_EVENT               | 当mysqld停止时生成此事件                                     |
| ROTATE_EVENT             | 当mysqld切换到新的binlog文件生成此事件，切换到新的binlog文件可以通过执行flush logs命令或者binlog文件大于 `max_binlog_size` 参数配置的大小； |
| INTVAR_EVENT             | 当sql语句中使用了AUTO_INCREMENT的字段或者LAST_INSERT_ID()函数；此事件没有被用在binlog_format为ROW模式的情况下 |
| SLAVE_EVENT              | 未使用                                                       |
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
| INCIDENT_EVENT           | 主服务器发生了不正常的事件，通知从服务器并告知可能会导致数据处于不一致的状态 |
| HEARTBEAT_LOG_EVENT      | 主服务器告诉从服务器，主服务器还活着，不写入到日志文件中     |

##### QUERY_ EVENT

以文本的形式记录事务的操作，使用场景：

1. 事务开始时，执行的BEGIN操作
2. STATEMENT格式中的DML操作
3. ROW格式中的DDL操作



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

mysql> show binlog events in 'binlog.000048';
+---------------+-----+----------------+-----------+-------------+-----------------------------------+
| Log_name      | Pos | Event_type     | Server_id | End_log_pos | Info                              |
+---------------+-----+----------------+-----------+-------------+-----------------------------------+
| binlog.000048 |   4 | Format_desc    |         1 |         124 | Server ver: 8.0.13, Binlog ver: 4 |
| binlog.000048 | 124 | Previous_gtids |         1 |         155 |                                   |
+---------------+-----+----------------+-----------+-------------+-----------------------------------+
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

#210815 13:52:15 server id 1  end_log_pos 4746 CRC32 0xb9ea8a8a 	Anonymous_GTID	last_committed=11	sequence_number=12	rbr_only=no	original_committed_timestamp=1629006735061122	immediate_commit_timestamp=1629006735061122	transaction_length=229
# original_commit_timestamp=1629006735061122 (2021-08-15 13:52:15.061122 中国标准时间)
# immediate_commit_timestamp=1629006735061122 (2021-08-15 13:52:15.061122 中国标准时间)
/*!80001 SET @@session.original_commit_timestamp=1629006735061122*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;

# at 4746
#210815 13:52:15 server id 1  end_log_pos 4902 CRC32 0xb0032563 	Query	thread_id=8	exec_time=0	error_code=0
SET TIMESTAMP=1629006735/*!*/;
/*!80013 SET @@session.sql_require_primary_key=0*//*!*/;
DROP TABLE IF EXISTS `canal_user` /* generated by server */
/*!*/;

# at 4902
#210815 13:52:15 server id 1  end_log_pos 4977 CRC32 0x1ee853b2 	Anonymous_GTID	last_committed=12	sequence_number=13	rbr_only=no	original_committed_timestamp=1629006735075339	immediate_commit_timestamp=1629006735075339	transaction_length=629
# original_commit_timestamp=1629006735075339 (2021-08-15 13:52:15.075339 中国标准时间)
# immediate_commit_timestamp=1629006735075339 (2021-08-15 13:52:15.075339 中国标准时间)
/*!80001 SET @@session.original_commit_timestamp=1629006735075339*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
```

- position：位于文件中的位置，即第一行的（# at 4746）,说明该事件记录从文件第4902个字节开始
- timestamp：事件发生的时间戳，即第二行的（#210815 13:52:15）
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



## redo log

> 事务的持久性

包括两部分：内存中的**日志缓存（redo log buffer）**和磁盘上的**日志文件（redo log file）**

每执行一条DML语句，先将记录写入`redo log buffer`，后续某个时间点再一次性将多个操作记录写入`redo log file`，称为WAL（Write-Ahead- Logging）

InnoDB特有的，且日志上的记录落盘后会被覆盖掉。因此需要binlog和redo log二者同时记录，才能保证当数据库发生宕机重启时，数据不会丢失。

## undo log

存储老版本数据。假设修改表中 id=2 的行数据，把 name='B' 修改为 name = 'B2' ，那么 undo 日志就会用来存放 name='B' 的记录，如果这个修改出现异常，可以使用 undo log来实现回滚操作，保证事务的一致性。

> 事务的原子性，undo log也是MVCC(多版本并发控制)实现的关键

当一个旧的事务需要读取数据时，为了能读取到老版本的数据，需要顺着 undo 链找到满足其可见性的记录。当版本链很长时，通常可以认为这是个比较耗时的操作。

在回滚阶段中undo log分为：`insert undo log` 和 `update undo log`；

- `insert undo log `: 事务对`insert`新记录时产生的`undo log`，只在事务回滚时需要，并且在事务提交后就可以立即丢弃。
- `update undo log` : 事务对记录进行`delete`和`update`操作时产生的`undo log`。不仅在事务回滚时需要，一致性读也需要，所以不能随便删除，只有当数据库所使用的快照中不涉及该日志记录，对应的回滚日志才会被 **purge 线程**删除。



## MVCC

**Multi-Version Concurrency Control** ，多版本并发控制，提高数据库并发性能，读写时，不加锁，非阻塞并发读；

- 当前读：读取的数据是最新版本，读取时还要保证其它并发事务不能修改当前记录，会对读取的记录进行加锁，像`select lock in share mode(共享锁)`, `select for update` ; `update`, `insert` ,`delete(排他锁)`
- 快照读：读取的数据是历史数据，可以认为MVCC是行锁的一个变种，但它在很多情况下，避免了加锁操作，降低了开销；既然是基于多版本，即快照读可能读到的并不一定是数据的最新版本，而有可能是之前的历史版本

### 版本链

`InnoDB`存储引擎，它的聚簇索引记录中都包含两个必要的隐藏列（`db_row_id`并不是必要的，创建的表中有主键或者非NULL唯一键时都不会包含`db_row_id`列）

- `db_trx_id`：事务id，6byte，每次对某条聚簇索引记录进行改动时，都会把对应的事务id赋值给`trx_id`隐藏列。 

- `db_roll_pointer`：回滚指针，7byte，指向这条记录的上一个版本，每次对某条聚簇索引记录进行改动时，都会把旧的版本写入到`undo log`中，然后这个隐藏列就相当于一个指针，可以通过它来找到该记录修改前的信息。

> Tips： 能不能在两个事务中交叉更新同一条记录呢？不可以，第一个事务更新了某条记录后，就会给这条记录加锁，另一个事务再次更新时就需要等待第一个事务提交了，把锁释放之后才可以继续更新。

### ReadView

事务进行**快照读**时产生的**读视图**，在该事务执行的**快照读**的那一刻，会生成数据库系统当前的一个快照，记录并维护系统当前**活跃事务的ID**（当每个事务开启时，都会被分配一个ID，这个ID是递增的，所以最新的事务，ID值越大）

`Read Uncommitted`隔离级别的事务，直接读取记录的最新版本。

`Serializable `隔离级别的事务，使用加锁的方式访问记录。

发生在`Read Committed`、 `Repeatable Read`隔离级别的事务，在执行普通的`SELECT`操作时访问记录的版本链的过程，使不同事务的读-写、写-读操作并发执行。

> Tips： 事务执行过程中，只有在第一次真正修改记录时（比如使用INSERT、DELETE、UPDATE语句），才会被分配一个单独的`trx_id`，这个事务id是递增的。

两者不同在生成`ReadView`的时机，`Read Committed`每一次进行`SELECT`操作前都会生成一个`ReadView`；而`Repeatable Read`只在第一次进行`SELECT`操作前生成一个`ReadView`，之后的查询操作都重复这个`ReadView`

| m_ids          | 生成ReadView时当前系统活跃的读写事务的事务id列表     |
| -------------- | ---------------------------------------------------- |
| up_limit_id    | 生成ReadView时当前系统活跃的读写事务中的最小的事务id |
| low_limit_id   | 生成ReadView时当前系统应该分配给下一个事务的id       |
| creator_trx_id | 生成ReadView时事务id                                 |

#### 判断版本链中版本可用

- trx_id == creator_trx_id：可以访问这个版本
- trx_id  <  up_limit_id：可以访问这个版本
- trx_id  >=  low_limit_id：不可以访问这个版本
- up_limit_id  <=  trx_id  <  low_limit_id：如果trx_id在m_ids中，说明创建ReadView时生成该版本的事务还是活跃的，该版本不可以被访问；如果不在，说明创建ReadView时生成该版本的事务已经被提交，该版本可以被访问。


## 锁

行锁和表锁

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



### 行锁

间隙锁

临建锁

记录锁



### 表锁

意向锁

自增锁



### 间隙锁

快照读

当前读