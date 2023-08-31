# Mysql
## Mysql.zip安装

1.  打开我的电脑->属性->高级->环境变量，在系统变量里选择PATH,在其后面添加: mysql bin文件夹的路径 (如: C:\Program Files\mysql-5.7.16-winx64\bin )，注意是追加,不是覆盖 ，然后确定


2.  在mysql目录中新建文件夹data，还需要修改一下配置文件，mysql默认的配置文件是mysql目录中的my-default.ini，比如我的是
`C:\Program Files\mysql-5.7.16-winx64\my-default.ini`

用记事本打开在其中修改或添加配置，之后保存关闭
```
[mysqld]
basedir= C:\Program Files\mysql-5.7.16-winx64（mysql所在目录）
datadir= C:\Program Files\mysql-5.7.16-winx64\data（mysql所在目录\data）
```

3.  以管理员身份运行cmd（一定要用管理员身份运行，不然权限不够），输入命令 cd C:\Program Files\mysql-5.7.16-winx64\bin  回车

4.  然后再输入`mysqld --initialize-insecure --user=mysql` 回车

5.  之后再输入 `mysqld install `回车

6.  输入`net start mysql` 回车启动mysql服务

7.  mysql服务已经启动了，输入`mysql -u root -p` 回车登录mysql数据库

8.  要求输入密码，刚刚安装完是没有密码的，直接回车

9.  初始化后第一次使用数据库要修改密码：
	```
	# user mysql;
	ALTER USER 'root'@'localhost' IDENTIFIED BY '新密码';
	```

10. 设置禁止mysql开机自启动（apt-get安装的mysql默认是开机自启动的）
	`sudo update-rc.d mysql disable`

11. 更改mysql配置文件：`/etc/mysql/mysql.conf.d/mysqld.cnf`

12. `Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock' (2)`
	解决方案：

13. 

14. error: Access denied for user 'maoyz'@'localhost' (using password: NO)
    解决方案：

    ```mysql
    -- *.cnf文件中，添加skip-grant-tables;
    
    update mysql.user set authentication_string=password('root') where user='root' and Host = 'localhost';
    
    flush privileges;
    quit;
    -- 关闭mysql，重启mysql，生效
    -- 注释kip-grant-tables
    ```



## 运行多mysql实例

帮助文档：https://blog.csdn.net/li87218677/article/details/64454488

```mysql
mysqld --initialize-insecure --defaults-file=/etc/mysql/3301.cnf  --basedir=/usr/  --datadir=/data/mysql/mysql_1  --user=mysql
mysqld --initialize-insecure --defaults-file=/etc/mysql/3301.cnf  --basedir=/usr/  --datadir=/data/mysql/mysql_1  --user=mysql
```

1. 安装mysql:
`cd /usr/bin`
`mysql_install_db --defaults-file=/etc/mysql/3301.cnf  --basedir=/usr/  --datadir=/data/mysql/mysql_1  --user=mysql`
`mysql_install_db --defaults-file=/etc/mysql/3302.cnf  --basedir=/usr/  --datadir=/data/mysql/mysql_2  --user=mysql`

2. 启动mysql:
`mysqld --defaults-file=/etc/mysql/3301.cnf   --user=mysql  &`
`mysqld --defaults-file=/etc/mysql/3302.cnf   --user=mysql  &`

3. 查看端口：
`netstat -nlt | grep 3301`
`netstat -nlt | grep 3302`

4. 登陆：
`mysql -S /data/mysql/mysql_1/mysqld.sock  -P 3301`
`mysql -S /data/mysql/mysql_2/mysqld.sock  -P 3302`

`mysql -uroot -p -S /data/mysql/mysql_1/mysqld.sock -hlocalhost  -P 3301`
`mysql -uroot -p -S /data/mysql/mysql_2/mysqld.sock -hlocalhost  -P 3302`

5. 停止mysql：
`mysqladmin -u root -proot  -S /data/mysql/mysql_1/mysqld.sock shutdown`
`mysqladmin -u root -proot  -S /data/mysql/mysql_2/mysqld.sock shutdown`



## 一台服务器多实例mysql做主从复制

参考文章：https://www.cnblogs.com/kevingrace/p/6129089.html

1.	主数据库
```mysql
log-bin=mysql-bin-master  #启用二进制日志,开启log-bin 并设置为master
server-id=3301   #本机数据库ID 标示 默认就是1,这里不用改
binlog-do-db=Test #可以被从服务器复制的库。二进制需要同步的数据库名
binlog-ignore-db=mysql  #不可以被从服务器复制的库
```

2. 主数据库设置用户
```mysql
grant replication slave on *.* to 'repl'@'127.0.0.1' identified by '123456';

flush privileges;

show master status;	#记录Position
```

3. 从数据库
```mysql
stop slave;

reset slave;

change master to master_user='repl', master_password='123456', master_host='127.0.0.1',master_port=3301, master_log_file='mysql-bin-master.000001',master_log_pos=597;

start slave;

show slave status \G;
```

4. 登陆repl用户
`mysql -urepl -p -S /data/mysql/mysql_1/mysqld.sock -hlocalhost  -P 3301`

## 主从复制



选择一台从服务做主



## 开启profile 查看生命周期

```mysql
set profiling = on;
show profiles;		# 记录操作语句
show profiles cpu, block io for query QUERY_ID;		# 具体哪条语句的执行情况

converting HEAP to MyISAM	查询结果大，内存不够用，网磁盘挪
Creating tmp table		创建临时表
Coping to tmp table on disk		内存临时表复制到磁盘（危险）      
locked
```



## 全局查询日志

```mysql
set global general_log = 1;
set global log_output = 'TABLE';
select * from mysql.general_log;	# 查询
```



## 主从复制配置

```mysql
[mysqld]
server-id = 1
log_bin = /var/log/mysql/binlog/mysqlbin

# 需要复制的数据库
binlog_do_db = include_database_name
# 忽略的数据库
binlog_ignore_db = mysql
```


1. 主数据库设置
```mysql
grant replication slave on *.* to 'repl'@'127.0.0.1' identified by '123456';
其中
'repl'：     用户名
'127.0.0.1'：从机数据库的IP地址
'123456'：   登录密码

flush privileges;
show master status;	#记录Position
```


2. 从数据库
```mysql
stop slave;		# 1 先停止
reset slave;	# 2 重置
change master to master_user='repl', master_password='123456', master_host='127.0.0.1',master_port=3301, master_log_file='mysql-bin-master.000001',master_log_pos=597;

start slave;
show slave status \G;    # 观察 Slave_IO_Running = YES   Slave_SQL_Running = YES
```

##### 七 mysql 初始化 timestamp，提示 Invalid default value for 'xxx'
解决方法: `去掉 sql_mode 中的 values: NO_ZERO_IN_DATE,NO_ZERO_DATE`

>1 `show variables like 'sql_mode';`
>2 `set session`
   `sql_mode='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION';`



- 事务超时时间设置，默认是50s

```
SET innodb_lock_wait_timeout=10;
```

- 连接超时时间设置，连接超时time为28800s(即8小时)，超过8h，数据库会断开这个连接

```
SET wait_timeout=30;
```

- 数据库超时时间设置

```
SET interactive_timeout=30;
```



- 查询大事务：

```mysql
select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started)) > 10
```



- 查看长事务执行的最后一条sql

```mysql
mysql> select now(), (UNIX_TIMESTAMP(now()) - UNIX_TIMESTAMP(a.trx_started)) diff_sec,b.id,b.user,b.host,b.db,d.SQL_TEXT from information_schema.innodb_trx a inner join
    -> information_schema.PROCESSLIST b
    -> on a.TRX_MYSQL_THREAD_ID=b.id and b.command = 'Sleep'
    -> inner join performance_schema.threads c ON b.id = c.PROCESSLIST_ID
```

- 查询被阻塞事务

```mysql
mysql> select * from sys.innodb_lock_waits;
```

- 监控长事务

```bash
#!/bin/bash
 
/usr/local/mysql/bin/mysql -N -uroot -pxxxxxx -e "select now(),(UNIX_TIMESTAMP(now()) - UNIX_TIMESTAMP(a.trx_started)) diff_sec,b.id,b.user,b.host,b.db,d.SQL_TEXT from information_schema.innodb_trx a inner join
information_schema.PROCESSLIST b
on a.TRX_MYSQL_THREAD_ID=b.id and b.command = 'Sleep'
inner join performance_schema.threads c ON b.id = c.PROCESSLIST_ID
inner join performance_schema.events_statements_current d ON d.THREAD_ID = c.THREAD_ID;" | while read A B C D E F G H
do
  if [ "$C" -gt 30 ]
      then
      echo $(date +"%Y-%m-%d %H:%M:%S")
      echo "processid[$D] $E@$F in db[$G] hold transaction time $C SQL:$H"
  fi
done >> /tmp/longtransaction.txt
```

> 这里的-gt 30是30秒钟的意思，只要超过了30秒钟就认定是长事务





## 数据库连接池

### 长连接

长连接即在建立连接后一直打开，直到应用程序关闭才释放。使用长连接的好处是减少每次创建连接带来的开销。

对于应用服务器来说维持长连接的好处不言自明，但是对于数据库服务器来说，过多的长连接则是灾难。

MYSQL的TCP连接支持长连接，所以每次操作完数据库，可以不必直接关掉连接，而是等待下次使用的时候在复用这个连接。所有的Socket长连接都是通过TCP自带的ping来维持心跳(TCP保活)，从而保持连接状态，而我们熟悉的`websocket`,也正是通过TCP的心跳来维持连接不被中断。

### 连接池

1. 初始化连接；
2. 业务取出连接；
3. 业务发送请求；
4. 放回连接。

除了上面的基本功能以外，还要处理并发问题，多数据库服务器和多用户，事务处理，连接池的配置与维护。

| 配置                | 含义                                                   |
| ------------------- | ------------------------------------------------------ |
| Connections         | 尝试连接Mysql的连接数，不管连接成功与否，该值都会+1    |
| Threads_connected   | 已经建立的连接数，单节点下一般小于最大连接池最大连接数 |
| max_connections     | Mysql限制的最大的可连接的数量                          |
| wait_timeout        | 即MYSQL长连接(非交互式)的最大生命时长,默认是8小时      |
| interactive_timeout | 长连接(交互式)的最大生命时长，默认是8小时              |

```txt
connections = ((core_count * 2) + effective_spindle_count)
```

该公式来自于：https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing。

其中，`core_count`是CPU核心， `effective_spindle_count` 的含义是有效主轴数，如果你的服务器使用的是带有16个磁盘的RAID，那么`valid_spindle_count=16`。它实质上是服务器可以管理多少个并行I / O请求的度量。旋转硬盘一次（通常）一次只能处理一个I / O请求，如果你有16个，则系统可以同时处理16个I / O请求。



## MySQL监控指标

### Gauge 类型

| open_streams                          | 打开的流的数量                                       |
| ------------------------------------- | ---------------------------------------------------- |
| open_files                            | 打开的文件数量                                       |
| open_table_definitions                | 打开的 .frm 文件的数量                               |
| open_tables                           | 打开的表数量                                         |
| threads_cached                        | 当前缓存中的空闲线程数                               |
| threads_connected                     | 当前打开的连接数                                     |
| threads_created                       | 当前启动的服务创建的线程数                           |
| threads_running                       | 当前 running 状态的线程数                            |
| seconds_behind_master                 | slave的SQL线程与I/O 线程的时间差                     |
| slave_io_running                      | slave的IO线程是否在运行                              |
| slave_sql_running                     | slave的SQL线程是否在运行                             |
| innodb_buffer_pool_bytes_data         | innodb 缓存池使用量                                  |
| innodb_buffer_pool_bytes_dirty        | innodb 缓存池脏页量                                  |
| innodb_buffer_pool_read_ahead         | 后端预读到户缓存的页数                               |
| innodb_buffer_pool_read_ahead_evicted | 预读的页数，但没被读取就从缓冲池中被替换的页数       |
| innodb_buffer_pool_read_ahead_rnd     |                                                      |
| innodb_buffer_pool_reads              | 进行逻辑读取时无法从缓冲池中获取而执行单页读取的次数 |
| innodb_buffer_pool_wait_free          | 等待空闲页次数                                       |
| innodb_current_row_locks              |                                                      |
| innodb_data_pending_fsyncs            | innodb当前挂起fsync()操作的次数                      |
| innodb_data_pending_reads             | innodb当前挂起读操作的次数                           |
| innodb_data_pending_writes            | innodb当前挂起写操作的次数                           |
| innodb_history_list_length            |                                                      |
| innodb_num_open_files                 | innodb的打开文件数量                                 |
| innodb_os_log_pending_fsyncs          | 当前挂起的fsync日志文件的次数                        |
| innodb_os_log_pending_writes          | 当前挂起的写日志文件的次数                           |
| innodb_row_lock_current_waits         | innodb当前等待行锁的数量                             |
| innodb_row_lock_time                  | innodb获取行锁的总消耗时间，ms                       |
| innodb_row_lock_time_avg              | innodb获取行锁的平均等待时间，ms                     |
| innodb_row_lock_time_max              | innodb获取行锁的最大等待时间，ms                     |
| innodb_row_lock_waits                 | innodb等待获取行锁的次数                             |



### Counter 类型，按增长率展示

| com_select                         | select命令次数                                         |
| ---------------------------------- | ------------------------------------------------------ |
| com_insert                         | insert命令次数                                         |
| com_update                         | update命令次数                                         |
| com_delete                         | delete命令次数                                         |
| TPS                                | commit、rollback 操作合计                              |
| com_replace                        | replace命令次数                                        |
| aborted_clients                    | 由于客户端没有正确关闭连接导致客户端终止而中断的连接数 |
| aborted_connects                   | 试图连接到MySQL服务器而失败的连接数                    |
| bytes_received                     | 累计接收的字节数                                       |
| bytes_sent                         | 累计发送的字节数                                       |
| connections                        | 尝试连接MySQL的连接数，包含成功和失败                  |
| opened_files                       | 已经打开的文件的数量                                   |
| opened_table_definitions           | 被缓存过的.frm文件的数量                               |
| opened_tables                      | 已经打开的表的数量                                     |
| rpl_semi_sync_master_net_wait_time | 主等待从机相应的总时间                                 |
| rpl_semi_sync_master_net_waits     | 主等待从机相应的总次数                                 |
| table_locks_immediate              | 可以立即获取锁的次数                                   |
| table_locks_waited                 | 需要等待锁的次数                                       |
| slow_queries                       | 慢查询次数                                             |
| innodb_background_log_sync         | innodb后台日志同步次数                                 |
| innodb_buffer_pool_read_requests   | innodb缓存池读取次数                                   |
| innodb_buffer_pool_write_requests  | innodb缓存池写入次数                                   |
| innodb_data_fsyncs                 | innodb执行fsync()的次数                                |
| innodb_mutex_os_waits              |                                                        |
| innodb_mutex_spin_rounds           |                                                        |
| innodb_mutex_spin_waits            |                                                        |
| innodb_os_log_fsyncs               | innodb log buffer执行fsync()的次数                     |
| innodb_os_log_written              | 写入日志文件的字节数                                   |



##  MySQL参数

### 内存及最大连接数相关配置

| 参数名                          | 标准值       | 说明             |
| ------------------------------- | ------------ | ---------------- |
| innodb_buffer_pool_size         | 50%*主机内存 |                  |
| key_buffer_size                 | 10MB         |                  |
| tmp_table_size                  | 64MB         |                  |
| query_cache_size                | 0            | 适用5.6、5.7版本 |
| innodb_log_buffer_size          | 64MB         |                  |
| innodb_additional_mem_pool_size | 16MB         |                  |
| read_buffer_size                | 256KB        |                  |
| read_rnd_buffer_size            | 512KB        |                  |
| sort_buffer_size                | 256KB        |                  |
| join_buffer_size                | 512KB        |                  |
| binlog_cache_size               | 32KB         |                  |
| thread_stack                    | 256KB        |                  |

**最大连接数计算公式**
`total_memory=innodb_buffer_pool_size+key_buffer_size+tmp_table_size+innodb_log_buffer_size+innodb_additional_mem_pool_size + max_conns*(read_buffer_size+read_rnd_buffer_size+sort_buffer_size+join_buffer_size+binlog_cache_size+thread_stack) + 200MB(主机其他进程占用)`



**内存对应最大连接数：**

| 内存/GB | 最大连接数配置 |
| ------- | -------------- |
| 2       | 432            |
| 4       | 1007           |
| 8       | 2156           |
| 16      | 4456           |
| 32      | 4500(max)      |
| 64      | 4500(max)      |



## MySQL调优

### 性能监控

### schema与数据类型优化

### 执行计划

### 索引优化

### 查询优化

### 分区表

### 服务器参数设置

### msql集群