# quartz集群解决方案

1 quartz自带

+ 建立数据库Quartz,建表语句
```
	# 注意以下删表的顺序,由于有几张表存在外键


	-- 存储少量的有关Scheduler的状态信息，和别的Scheduler实例
	DROP TABLE IF EXISTS QRTZ_SCHEDULER_STATE;
	-- 悲观锁的信息
	DROP TABLE IF EXISTS QRTZ_LOCKS;
	-- 存储简单的Trigger，包括重复次数、间隔、以及已触的次数
	DROP TABLE IF EXISTS QRTZ_SIMPLE_TRIGGERS;
	-- 存储CalendarIntervalTrigger和DailyTimeIntervalTrigger两种类型的触发器
	DROP TABLE IF EXISTS QRTZ_SIMPROP_TRIGGERS;
	-- 存储CronTrigger，包括Cron表达式和时区信息
	DROP TABLE IF EXISTS QRTZ_CRON_TRIGGERS;
	-- 存储与已触发的Trigger相关的状态信息，以及相联Job的执行信息
	DROP TABLE IF EXISTS QRTZ_FIRED_TRIGGERS;
	-- 存储已暂停的Trigger组的信息
	DROP TABLE IF EXISTS QRTZ_PAUSED_TRIGGER_GRPS;
	-- 以Blob 类型存储的触发器。
	DROP TABLE IF EXISTS QRTZ_BLOB_TRIGGERS;
	-- Quartz的Calendars
	DROP TABLE IF EXISTS QRTZ_CALENDARS;
	-- 存储已配置的Trigger的信息
	DROP TABLE IF EXISTS QRTZ_TRIGGERS;
	-- 存储每一个已配置的Job的详细信息
	DROP TABLE IF EXISTS QRTZ_JOB_DETAILS;

	CREATE TABLE QRTZ_JOB_DETAILS
	(
	SCHED_NAME VARCHAR(120) NOT NULL,
	JOB_NAME  VARCHAR(200) NOT NULL,
	JOB_GROUP VARCHAR(200) NOT NULL,
	DESCRIPTION VARCHAR(250) NULL,
	JOB_CLASS_NAME   VARCHAR(250) NOT NULL,
	IS_DURABLE VARCHAR(1) NOT NULL,
	IS_NONCONCURRENT VARCHAR(1) NOT NULL,
	IS_UPDATE_DATA VARCHAR(1) NOT NULL,
	REQUESTS_RECOVERY VARCHAR(1) NOT NULL,
	JOB_DATA BLOB NULL,
	PRIMARY KEY (SCHED_NAME,JOB_NAME,JOB_GROUP)
	);

	CREATE TABLE QRTZ_TRIGGERS
	(
	SCHED_NAME VARCHAR(120) NOT NULL,
	TRIGGER_NAME VARCHAR(200) NOT NULL,
	TRIGGER_GROUP VARCHAR(200) NOT NULL,
	JOB_NAME  VARCHAR(200) NOT NULL,
	JOB_GROUP VARCHAR(200) NOT NULL,
	DESCRIPTION VARCHAR(250) NULL,
	NEXT_FIRE_TIME BIGINT(13) NULL,
	PREV_FIRE_TIME BIGINT(13) NULL,
	PRIORITY INTEGER NULL,
	TRIGGER_STATE VARCHAR(16) NOT NULL,
	TRIGGER_TYPE VARCHAR(8) NOT NULL,
	START_TIME BIGINT(13) NOT NULL,
	END_TIME BIGINT(13) NULL,
	CALENDAR_NAME VARCHAR(200) NULL,
	MISFIRE_INSTR SMALLINT(2) NULL,
	JOB_DATA BLOB NULL,
	PRIMARY KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP),
	FOREIGN KEY (SCHED_NAME,JOB_NAME,JOB_GROUP)
	REFERENCES QRTZ_JOB_DETAILS(SCHED_NAME,JOB_NAME,JOB_GROUP)
	);

	CREATE TABLE QRTZ_SIMPLE_TRIGGERS
	(
	SCHED_NAME VARCHAR(120) NOT NULL,
	TRIGGER_NAME VARCHAR(200) NOT NULL,
	TRIGGER_GROUP VARCHAR(200) NOT NULL,
	REPEAT_COUNT BIGINT(7) NOT NULL,
	REPEAT_INTERVAL BIGINT(12) NOT NULL,
	TIMES_TRIGGERED BIGINT(10) NOT NULL,
	PRIMARY KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP),
	FOREIGN KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP)
	REFERENCES QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP)
	);

	CREATE TABLE QRTZ_CRON_TRIGGERS
	(
	SCHED_NAME VARCHAR(120) NOT NULL,
	TRIGGER_NAME VARCHAR(200) NOT NULL,
	TRIGGER_GROUP VARCHAR(200) NOT NULL,
	CRON_EXPRESSION VARCHAR(200) NOT NULL,
	TIME_ZONE_ID VARCHAR(80),
	PRIMARY KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP),
	FOREIGN KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP)
	REFERENCES QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP)
	);

	CREATE TABLE QRTZ_SIMPROP_TRIGGERS
	(
	SCHED_NAME VARCHAR(120) NOT NULL,
	TRIGGER_NAME VARCHAR(200) NOT NULL,
	TRIGGER_GROUP VARCHAR(200) NOT NULL,
	STR_PROP_1 VARCHAR(512) NULL,
	STR_PROP_2 VARCHAR(512) NULL,
	STR_PROP_3 VARCHAR(512) NULL,
	INT_PROP_1 INT NULL,
	INT_PROP_2 INT NULL,
	LONG_PROP_1 BIGINT NULL,
	LONG_PROP_2 BIGINT NULL,
	DEC_PROP_1 NUMERIC(13,4) NULL,
	DEC_PROP_2 NUMERIC(13,4) NULL,
	BOOL_PROP_1 VARCHAR(1) NULL,
	BOOL_PROP_2 VARCHAR(1) NULL,
	PRIMARY KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP),
	FOREIGN KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP)
	REFERENCES QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP)
	);

	CREATE TABLE QRTZ_BLOB_TRIGGERS
	(
	SCHED_NAME VARCHAR(120) NOT NULL,
	TRIGGER_NAME VARCHAR(200) NOT NULL,
	TRIGGER_GROUP VARCHAR(200) NOT NULL,
	BLOB_DATA BLOB NULL,
	PRIMARY KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP),
	FOREIGN KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP)
	REFERENCES QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP)
	);

	CREATE TABLE QRTZ_CALENDARS
	(
	SCHED_NAME VARCHAR(120) NOT NULL,
	CALENDAR_NAME  VARCHAR(200) NOT NULL,
	CALENDAR BLOB NOT NULL,
	PRIMARY KEY (SCHED_NAME,CALENDAR_NAME)
	);

	CREATE TABLE QRTZ_PAUSED_TRIGGER_GRPS
	(
	SCHED_NAME VARCHAR(120) NOT NULL,
	TRIGGER_GROUP  VARCHAR(200) NOT NULL,
	PRIMARY KEY (SCHED_NAME,TRIGGER_GROUP)
	);

	CREATE TABLE QRTZ_FIRED_TRIGGERS
	(
	SCHED_NAME VARCHAR(120) NOT NULL,
	ENTRY_ID VARCHAR(95) NOT NULL,
	TRIGGER_NAME VARCHAR(200) NOT NULL,
	TRIGGER_GROUP VARCHAR(200) NOT NULL,
	INSTANCE_NAME VARCHAR(200) NOT NULL,
	FIRED_TIME BIGINT(13) NOT NULL,
	SCHED_TIME BIGINT(13) NOT NULL,
	PRIORITY INTEGER NOT NULL,
	STATE VARCHAR(16) NOT NULL,
	JOB_NAME VARCHAR(200) NULL,
	JOB_GROUP VARCHAR(200) NULL,
	IS_NONCONCURRENT VARCHAR(1) NULL,
	REQUESTS_RECOVERY VARCHAR(1) NULL,
	PRIMARY KEY (SCHED_NAME,ENTRY_ID)
	);

	CREATE TABLE QRTZ_SCHEDULER_STATE
	(
	SCHED_NAME VARCHAR(120) NOT NULL,
	INSTANCE_NAME VARCHAR(200) NOT NULL,
	LAST_CHECKIN_TIME BIGINT(13) NOT NULL,
	CHECKIN_INTERVAL BIGINT(13) NOT NULL,
	PRIMARY KEY (SCHED_NAME,INSTANCE_NAME)
	);

	CREATE TABLE QRTZ_LOCKS
	(
	SCHED_NAME VARCHAR(120) NOT NULL,
	LOCK_NAME  VARCHAR(40) NOT NULL,
	PRIMARY KEY (SCHED_NAME,LOCK_NAME)
	);

	commit;

	###如果定时任务比较多,建议增加索引提升速度
	create index idx_qrtz_j_req_recovery on QRTZ_JOB_DETAILS(SCHED_NAME,REQUESTS_RECOVERY);
	create index idx_qrtz_j_grp on QRTZ_JOB_DETAILS(SCHED_NAME,JOB_GROUP);
	create index idx_qrtz_t_j on QRTZ_TRIGGERS(SCHED_NAME,JOB_NAME,JOB_GROUP);
	create index idx_qrtz_t_jg on QRTZ_TRIGGERS(SCHED_NAME,JOB_GROUP);
	create index idx_qrtz_t_c on QRTZ_TRIGGERS(SCHED_NAME,CALENDAR_NAME);
	create index idx_qrtz_t_g on QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_GROUP);
	create index idx_qrtz_t_state on QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_STATE);
	create index idx_qrtz_t_n_state on QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP,TRIGGER_STATE);
	create index idx_qrtz_t_n_g_state on QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_GROUP,TRIGGER_STATE);
	create index idx_qrtz_t_next_fire_time on QRTZ_TRIGGERS(SCHED_NAME,NEXT_FIRE_TIME);
	create index idx_qrtz_t_nft_st on QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_STATE,NEXT_FIRE_TIME);
	create index idx_qrtz_t_nft_misfire on QRTZ_TRIGGERS(SCHED_NAME,MISFIRE_INSTR,NEXT_FIRE_TIME);
	create index idx_qrtz_t_nft_st_misfire on QRTZ_TRIGGERS(SCHED_NAME,MISFIRE_INSTR,NEXT_FIRE_TIME,TRIGGER_STATE);
	create index idx_qrtz_t_nft_st_misfire_grp on QRTZ_TRIGGERS(SCHED_NAME,MISFIRE_INSTR,NEXT_FIRE_TIME,TRIGGER_GROUP,TRIGGER_STATE);
	create index idx_qrtz_ft_trig_inst_name on QRTZ_FIRED_TRIGGERS(SCHED_NAME,INSTANCE_NAME);
	create index idx_qrtz_ft_inst_job_req_rcvry on QRTZ_FIRED_TRIGGERS(SCHED_NAME,INSTANCE_NAME,REQUESTS_RECOVERY);
	create index idx_qrtz_ft_j_g on QRTZ_FIRED_TRIGGERS(SCHED_NAME,JOB_NAME,JOB_GROUP);
	create index idx_qrtz_ft_jg on QRTZ_FIRED_TRIGGERS(SCHED_NAME,JOB_GROUP);
	create index idx_qrtz_ft_t_g on QRTZ_FIRED_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP);
	create index idx_qrtz_ft_tg on QRTZ_FIRED_TRIGGERS(SCHED_NAME,TRIGGER_GROUP);
```

+ 配置qartz.properties属性文件

```
	# org.quartz.impl.StdSchedulerFactory

	# Configure Main Scheduler Properties 调度器属性
	# ===========================================================================
	# 调度标识名 集群中每一个实例都必须使用相同的名称
	org.quartz.scheduler.instanceName:DefaultQuartzClusteredScheduler
	# ID设置为自动获取 每一个必须不同
	org.quartz.scheduler.instanceId:AUTO
	org.quartz.scheduler.rmi.export:false
	org.quartz.scheduler.rmi.proxy:false
	org.quartz.scheduler.wrapJobExecutionInUserTransaction:false

	# ===========================================================================>


	# Configure ThreadPool 线程池属性
	# ===========================================================================
	# 线程池的实现类（一般使用SimpleThreadPool即可满足几乎所有用户的需求）
	org.quartz.threadPool.class:org.quartz.simpl.SimpleThreadPool
	# 指定线程数，至少为1（无默认值）(一般设置为1-100直接的整数合适)
	org.quartz.threadPool.threadCount:20
	# 设置线程的优先级（最大为java.lang.Thread.MAX_PRIORITY 10，最小为Thread.MIN_PRIORITY 1，默认为5）
	org.quartz.threadPool.threadPriority:5
	# 设置SimpleThreadPool的一些属性
	#设置是否为守护线程
	#org.quartz.threadpool.makethreadsdaemons = false
	org.quartz.threadPool.threadsInheritContextClassLoaderOfInitializingThread: true
	org.quartz.threadpool.threadsinheritgroupofinitializingthread=true
	#线程前缀默认值是：[Scheduler Name]_Worker
	org.quartz.threadpool.threadnameprefix=quartzJobThead
	# 配置全局监听(TriggerListener,JobListener) 则应用程序可以接收和执行 预定的事件通知
	# ===========================================================================
	# Configuring a Global TriggerListener 配置全局的Trigger监听器
	# MyTriggerListenerClass 类必须有一个无参数的构造函数，和 属性的set方法，目前2.2.x只支持原始数据类型的值（包括字符串）
	# ===========================================================================
	#org.quartz.triggerListener.NAME.class = com.swh.MyTriggerListenerClass
	#org.quartz.triggerListener.NAME.propName = propValue
	#org.quartz.triggerListener.NAME.prop2Name = prop2Value
	# ===========================================================================
	# Configuring a Global JobListener 配置全局的Job监听器
	# MyJobListenerClass 类必须有一个无参数的构造函数，和 属性的set方法，目前2.2.x只支持原始数据类型的值（包括字符串）
	# ===========================================================================
	#org.quartz.jobListener.NAME.class = com.swh.MyJobListenerClass
	#org.quartz.jobListener.NAME.propName = propValue
	#org.quartz.jobListener.NAME.prop2Name = prop2Value


	# ===========================================================================
	# Configure JobStore 存储调度信息（工作，触发器和日历等）
	#==============================================================
	# 默认存储在内存中
	# org.quartz.jobStore.class = org.quartz.simpl.RAMJobStore
	# 数据保存方式为持久化
	org.quartz.jobStore.class = org.quartz.impl.jdbcjobstore.JobStoreTX
	org.quartz.jobStore.driverDelegateClass = org.quartz.impl.jdbcjobstore.StdJDBCDelegate
	org.quartz.jobStore.tablePrefix = QRTZ_
	# 开启分布式部署
	org.quartz.jobStore.isClustered = true
	org.quartz.jobStore.clusterCheckinInterval = 20000
	org.quartz.jobStore.maxMisfiresToHandleAtATime = 1
	#容许的最大作业延长时间
	org.quartz.jobStore.misfireThreshold = 60000
	#值为 True 时告诉 Quartz (当使用 JobStoreTX 或 CMT 时)
	#调用 JDBC 连接的 setTransactionIsolation(Connection.TRANSACTION_SERIALIZABLE) 方法。
	#这能助于防止某些数据库在高负荷和长事物时的锁超时。
	org.quartz.jobStore.txIsolationLevelSerializable = true
	org.quartz.jobStore.selectWithLockSQL = SELECT * FROM {0}LOCKS WHERE LOCK_NAME = ? FOR UPDATE
	org.quartz.jobStore.dataSource = myDS
	org.quartz.jobStore.acquireTriggersWithinLock=true

	#==============================================================
	#Configure DataSource
	#==============================================================
	org.quartz.dataSource.myDS.driver = com.mysql.jdbc.Driver
	org.quartz.dataSource.myDS.URL = jdbc:mysql://127.0.0.1:3306/quartz?useUnicode=true&characterEncoding=UTF-8
	#jdbc:mysql://localhost:3306/quartz?useUnicode=true&characterEncoding=UTF-8
	org.quartz.dataSource.myDS.user = root
	org.quartz.dataSource.myDS.password = root
	org.quartz.dataSource.myDS.maxConnections = 100



	org.quartz.scheduler.skipUpdateCheck = true


	#============================================================================
	# Configure Plugins
	#============================================================================
	org.quartz.plugin.triggHistory.class = org.quartz.plugins.history.LoggingJobHistoryPlugin
	org.quartz.plugin.shutdownhook.class = org.quartz.plugins.management.ShutdownHookPlugin
	org.quartz.plugin.shutdownhook.cleanShutdown = true


	# 自定义插件
	#org.quartz.plugin.NAME.class = com.swh.MyPluginClass
	#org.quartz.plugin.NAME.propName = propValue
	#org.quartz.plugin.NAME.prop2Name = prop2Value
	#配置trigger执行历史日志（可以看到类的文档和参数列表）
	#org.quartz.plugin.triggHistory.class=org.quartz.plugins.history.LoggingTriggerHistoryPlugin
	#org.quartz.plugin.triggHistory.triggerFiredMessage=Trigger {1}.{0} fired job {6}.{5} at: {4, date, HH:mm:ss MM/dd/yyyy}
	#org.quartz.plugin.triggHistory.triggerCompleteMessage=Trigger {1}.{0} completed firing job {6}.{5} at {4, date, HH:mm:ss MM/dd/yyyy} with resulting trigger instruction code: {9}
	#配置job调度插件  quartz_jobs(jobs and triggers内容)的XML文档
	#加载 Job 和 Trigger 信息的类   （1.8之前用：org.quartz.plugins.xml.JobInitializationPlugin）
	#org.quartz.plugin.jobInitializer.class=org.quartz.plugins.xml.XMLSchedulingDataProcessorPlugin
	#指定存放调度器(Job 和 Trigger)信息的xml文件，默认是classpath下quartz_jobs.xml
	#org.quartz.plugin.jobInitializer.fileNames=my_quartz_job2.xml
	#org.quartz.plugin.jobInitializer.overWriteExistingJobs = false
	#org.quartz.plugin.jobInitializer.failOnFileNotFound=true
	#自动扫描任务单并发现改动的时间间隔,单位为秒
	#org.quartz.plugin.jobInitializer.scanInterval=10
	#覆盖任务调度器中同名的jobDetail,避免只修改了CronExpression所造成的不能重新生效情况
	#org.quartz.plugin.jobInitializer.wrapInUserTransaction=false
	# ===========================================================================
	# Sample configuration of ShutdownHookPlugin  ShutdownHookPlugin插件的配置样例
	# ===========================================================================
	#org.quartz.plugin.shutdownhook.class = org.quartz.plugins.management.ShutdownHookPlugin
	#org.quartz.plugin.shutdownhook.cleanShutdown = true

```