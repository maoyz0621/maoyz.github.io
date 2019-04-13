#### 一、Mysql.zip安装

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

13. error: Access denied for user 'maoyz0621'@'localhost' (using password: NO)
	解决方案：
	```
	*.cnf文件中，添加skip-grant-tables;
    update mysql.user set authentication_string=password('root') where user='root' and Host = 'localhost';
	flush privileges;
 	quit;
	关闭mysql，重启mysql，生效
	注释kip-grant-tables
	```

#### 二、运行多mysql实例
	
帮助文档：https://blog.csdn.net/li87218677/article/details/64454488
```
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


#### 三、一台服务器多实例mysql做主从复制
参考文章：https://www.cnblogs.com/kevingrace/p/6129089.html

1.	主数据库
```
log-bin=mysql-bin-master  #启用二进制日志,开启log-bin 并设置为master
server-id=3301   #本机数据库ID 标示 默认就是1,这里不用改
binlog-do-db=Test #可以被从服务器复制的库。二进制需要同步的数据库名
binlog-ignore-db=mysql  #不可以被从服务器复制的库
```

2. 主数据库设置用户
```
grant replication slave on *.* to 'repl'@'127.0.0.1' identified by '123456';
flush privileges;
show master status;	#记录Position
```

3. 从数据库
```
stop slave;
reset slave;
change master to master_user='repl', master_password='123456', master_host='127.0.0.1',master_port=3301, master_log_file='mysql-bin-master.000001',master_log_pos=597;
start slave;
show slave status \G;
```

4. 登陆repl用户
`mysql -urepl -p -S /data/mysql/mysql_1/mysqld.sock -hlocalhost  -P 3301`

#### 四、开启profile 查看生命周期
```
    set profiling = on;
    show profiles;		# 记录操作语句
    show profiles cpu, block io for query QUERY_ID;		# 具体哪条语句的执行情况

    converting HEAP to MyISAM	查询结果大，内存不够用，网磁盘挪
    Creating tmp table		创建临时表
    Coping to tmp table on disk		内存临时表复制到磁盘（危险）      
    locked
```
		
#### 五、全局查询日志
```
    set global general_log = 1;
    set global log_output = 'TABLE';
    select * from mysql.general_log;	# 查询
```

#### 六、主从复制配置
```
    [mysqld]
	server-id = 1
	log_bin = /var/log/mysql/binlog/mysqlbin

	# 需要复制的数据库
	binlog_do_db = include_database_name
	# 忽略的数据库
	binlog_ignore_db = mysql
```
		

1. 主数据库设置
```
	grant replication slave on *.* to 'repl'@'127.0.0.1' identified by '123456';
	其中
		'repl'：     用户名
		'127.0.0.1'：从机数据库的IP地址
		'123456'：   登录密码

	flush privileges;
	show master status;	#记录Position
```
		

2. 从数据库
```
	stop slave;		# 1 先停止
	reset slave;	# 2 重置
	change master to master_user='repl', master_password='123456', master_host='127.0.0.1',master_port=3301, master_log_file='mysql-bin-master.000001',master_log_pos=597;

	start slave;
	show slave status \G;    # 观察 Slave_IO_Running = YES   Slave_SQL_Running = YES
```