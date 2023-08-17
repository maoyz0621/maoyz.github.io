# xxl-job

## 设计理念

将调度行为抽象成“**调度中心**”平台，平台本身不承担业务逻辑，负责发起调度请求

将任务抽象成分散的**JobHandler**，交于“**执行器**”统一管理，“**执行器**”负责接收调度请求并执行对应的**JobHandler**中业务逻辑

“调度”和“任务"相互解耦。

## 分布式定时任务比较

| 功能       | quartz                                             | elastic-job                                                  | xxl-job                                                      | Spring Task                          |
| ---------- | -------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------ |
| HA(高可用) | 多节点部署，通过数据库锁来保证只有一个节点执行任务 | 通过zookeeper的注册和发现，可以动态添加服务器，支持水平扩容  | 集群部署                                                     | 不支持                               |
| 任务分片   | 不支持                                             | 支持                                                         | 支持                                                         | 不支持                               |
| 管理界面   | 没有                                               | 有                                                           | 有                                                           | 没有                                 |
| 难易程度   | 简单                                               | 较复杂                                                       | 简单                                                         | 简单                                 |
| 缺点       | 没有管理界面不支持任务分片，不适用于分布式场景     | 需要引入zookeeper，增加系统复杂度，比较复杂，任务量巨大时可以考虑使用 | 通过获取数据库锁的方式，保证集群中执行任务的唯一性，性能不好 | 不支持分布式，功能单一，重试机制不好 |

## 架构

| xxl-job架构图v2.1.0                                     |
| ------------------------------------------------------- |
| ![xxl-job架构图v2.1.0](./image/xxl-job架构图v2.1.0.png) |

- 数据库表

```
- xxl_job_lock：任务调度锁表；
- xxl_job_group：执行器信息表，维护任务执行器信息；
- xxl_job_info：调度扩展信息表： 用于保存XXL-JOB调度任务的扩展信息，如任务分组、任务名、机器地址、执行器、执行入参和报警邮件等等；
- xxl_job_log：调度日志表： 用于保存XXL-JOB任务调度的历史信息，如调度结果、执行结果、调度入参、调度机器和执行器等等；
- xxl_job_log_report：调度日志报表：用户存储XXL-JOB任务调度日志的报表，调度中心报表功能页面会用到；
- xxl_job_logglue：任务GLUE日志：用于保存GLUE更新历史，用于支持GLUE的版本回溯功能；
- xxl_job_registry：执行器注册表，维护在线的执行器和调度中心机器地址信息；
- xxl_job_user：系统用户表；
```



## 执行流程分析

### 执行器启动流程

#### 初始化执行器

```java
@Bean
public XxlJobSpringExecutor xxlJobExecutor() {
    logger.info(">>>>>>>>>>> xxl-job config init.");
    XxlJobSpringExecutor xxlJobSpringExecutor = new XxlJobSpringExecutor();
    xxlJobSpringExecutor.setAdminAddresses(adminAddresses);
    xxlJobSpringExecutor.setAppname(appname);
    xxlJobSpringExecutor.setAddress(address);
    xxlJobSpringExecutor.setIp(ip);
    xxlJobSpringExecutor.setPort(port);
    xxlJobSpringExecutor.setAccessToken(accessToken);
    xxlJobSpringExecutor.setLogPath(logPath);
    xxlJobSpringExecutor.setLogRetentionDays(logRetentionDays);

    return xxlJobSpringExecutor;
}
```



#### XxlJobSpringExecutor

调用父类的start方法（com.xxl.job.core.executor.XxlJobExecutor#start）

```java
    public void start() throws Exception {

        // 初始化日志路径
        XxlJobFileAppender.initLogPath(logPath);

        // 初始化admin的客户端
        initAdminBizList(adminAddresses, accessToken);

        // 初始化日志清理线程
        JobLogFileCleanThread.getInstance().start(logRetentionDays);

        // 初始化回调线程池
        TriggerCallbackThread.getInstance().start();

        // 初始化执行器服务
        initEmbedServer(address, ip, port, appname, accessToken);
    }
```

总结下来，就是：

- 初始化日志路径
- 初始化admin的客户端
- 初始化日志清理线程
- 初始化回调线程池
- 初始化执行器服务


### 三个线程

#### JobLogFileCleanThread

**初始化日志清理线程**，功能：启动一个线程localThread，用来清理过期的日志文件。localThread的run方法一直执行，

1. 首先获取所有的日志文件目录，日志文件形式如logPath/yyyy-MM-dd/9999.log，获取logPath/yyyy-MM-dd/目录下的所有日志文件

2. 判断日志文件是否已经过期，过期时间是配置的，如果当前时间减去日志文件创建时间（yyyy-MM-dd）大于配置的日志清理天数，说明日志文件已经过期，一般配置只保存30天的日志，30天以前的日志都删除掉
3. 执行完成之后，线程localThread会休眠1天

#### TriggerCallbackThread

**初始化回调线程池**，功能：启动了两个线程，一个是**triggerCallbackThread**回调线程，一个是**triggerRetryCallbackThread**重试回调线程。

- triggerCallbackThread回调线程

  将定时任务执行的结果回调给admin保存在数据库中，调用AdminBizClient的callback方法回调写回admin服务的数据库中。

- triggerRetryCallbackThread重试回调线程

  将错误的回调重新进行回调，跟triggerCallbackThread回调线程是一样的，只是triggerRetryCallbackThread只重新回调错误的回调。


定时任务运行完成以后，将运行以后的结果保存在队列中，每次回调都是从队列中获取定时任务的结果写回admin服务，是通过http去写回到admin服务的。

#### ExecutorRegistryThread

初始化执行器服务，功能：启动一个netty服务器，用于执行器接收admin的http请求。

> 主要接收admin发送的空闲检测请求、运行定时任务的请求、停止运行定时任务的请求、获取日志的请求。

admin注册了执行器，注册执行器是调用**AdminBizClient**的registry方法注册的，AdminBizClient的registry方法通过http将注册请求转发给admin服务的**AdminBizImpl**类的registry方法，AdminBizImpl类的registry方法将注册请求保存在数据库中。

| initEmbedServer                                              |
| ------------------------------------------------------------ |
| <img src="./image/initEmbedServer.png" alt="initEmbedServer" style="zoom: 80%;" /> |
| startRegistry                                                |
| <img src="./image/xxl-job-ExecutorRegistryThread.png" alt="ExecutorRegistryThread" style="zoom:80%;" /> |

执行器服务接收admin服务的请求，交给ExecutorBiz接口处理

##### ExecutorBiz

ExecutorBiz接口有五个方法

- beat（心跳检测）
- idleBeat（空闲检测）
- run（运行定时任务）
- kill（停止运行任务）
- log（获取日志）

ExecutorBiz接口有两个实现：**ExecutorBizClient**和**ExecutorBizImpl**

- ExecutorBizClient：执行器客户端

- ExecutorBizImpl：执行器服务端

admin服务通过ExecutorBizClient类的方法通过http将请求转发给执行器服务的ExecutorBizImpl对应的方法。

### 调度中心启动流程

启动时执行`com.xxl.job.admin.core.conf.XxlJobAdminConfig`

```java
@Override
public void afterPropertiesSet() throws Exception {
    adminConfig = this;

    xxlJobScheduler = new XxlJobScheduler();
    xxlJobScheduler.init();
}

@Override
public void destroy() throws Exception {
    xxlJobScheduler.destroy();
}
```



初始化执行器`com.xxl.job.admin.core.scheduler.XxlJobScheduler`

```java
public void init() throws Exception {
    // init i18n
    initI18n();

    // admin trigger pool start
    JobTriggerPoolHelper.toStart();

    // admin registry monitor run
    JobRegistryHelper.getInstance().start();

    // admin fail-monitor run
    JobFailMonitorHelper.getInstance().start();

    // admin lose-monitor run ( depend on JobTriggerPoolHelper )
    JobCompleteHelper.getInstance().start();

    // admin log report start
    JobLogReportHelper.getInstance().start();

    // start-schedule  ( depend on JobTriggerPoolHelper )
    JobScheduleHelper.getInstance().start();
}    
```

#### 启动调度任务

`com.xxl.job.admin.core.thread.JobScheduleHelper#start`，创建并启动两个线程，`scheduleThread` 和 `ringThread`

```java
public void start(){

    scheduleThread = new Thread(new Runnable() {
        // 省略
    });
    scheduleThread.setDaemon(true);
    scheduleThread.setName("xxl-job, admin JobScheduleHelper#scheduleThread");
    scheduleThread.start();


    // ring thread
    ringThread = new Thread(new Runnable() {
        // 省略
    });
    ringThread.setDaemon(true);
    ringThread.setName("xxl-job, admin JobScheduleHelper#ringThread");
    ringThread.start();
}
```

- scheduleThread，功能：
- ringThread，功能：

#### 集群部署多服务器调度任务

xxl-job通过mysql悲观锁实现分布式锁，从而避免多个服务器同时调度任务

```java
public void start() {

    scheduleThread = new Thread(new Runnable() {
        @Override
        public void run() {
            // pre-read count: treadpool-size * trigger-qps (each trigger cost 50ms, qps = 1000/50 = 20)
            int preReadCount = (XxlJobAdminConfig.getAdminConfig().getTriggerPoolFastMax() + XxlJobAdminConfig.getAdminConfig().getTriggerPoolSlowMax()) * 20;

            while (!scheduleThreadToStop) {
                // Scan Job
                long start = System.currentTimeMillis();

                Connection conn = null;
                Boolean connAutoCommit = null;
                PreparedStatement preparedStatement = null;

                boolean preReadSuc = true;
                try {

                    conn = XxlJobAdminConfig.getAdminConfig().getDataSource().getConnection();
                    connAutoCommit = conn.getAutoCommit();
                    conn.setAutoCommit(false);

                    preparedStatement = conn.prepareStatement("select * from xxl_job_lock where lock_name = 'schedule_lock' for update");
                    preparedStatement.execute();
                    // tx start
                    // tx stop
                } catch (Exception e) {
                    //
                } finally {
                    // commit
                    conn.commit();
                    conn.setAutoCommit(connAutoCommit);
                }
            }
        });
    }
}
```

- 通过setAutoCommit(false)，关闭自动提交
- 通过select lock for update语句，其他事务无法获取到锁，显示排它锁。
- 执行定时调度任务的逻辑
- 最后在finally块中commit()提交事务，并且setAutoCommit，释放for update的排它锁。

#### 定时任务实现

```java
public void start() {

    scheduleThread = new Thread(new Runnable() {
        @Override
        public void run() {

            // pre-read count: treadpool-size * trigger-qps (each trigger cost 50ms, qps = 1000/50 = 20)
            // 预读取定时任务数量（200 + 100）* 20
            int preReadCount = (XxlJobAdminConfig.getAdminConfig().getTriggerPoolFastMax() + XxlJobAdminConfig.getAdminConfig().getTriggerPoolSlowMax()) * 20;

            while (!scheduleThreadToStop) {
                try {
                    // tx start

                    // 1、pre read
                    long nowTime = System.currentTimeMillis();
                    // 从数据库把5秒内要执行的任务读出
                    List<XxlJobInfo> scheduleList = XxlJobAdminConfig.getAdminConfig().getXxlJobInfoDao().scheduleJobQuery(nowTime + PRE_READ_MS, preReadCount);
                    if (scheduleList != null && scheduleList.size() > 0) {
                        // 2、push time-ring
                        for (XxlJobInfo jobInfo : scheduleList) {

                            // time-ring jump
                            // 当前时间 > 下一次触发时间+5s，下一次触发时间过期时间超过5s
                            if (nowTime > jobInfo.getTriggerNextTime() + PRE_READ_MS) {
                                // 2.1、trigger-expire > 5s：pass && make next-trigger-time

                                // 1、misfire match
                                MisfireStrategyEnum misfireStrategyEnum = MisfireStrategyEnum.match(jobInfo.getMisfireStrategy(), MisfireStrategyEnum.DO_NOTHING);
                                // 立即执行一次策略
                                if (MisfireStrategyEnum.FIRE_ONCE_NOW == misfireStrategyEnum) {
                                    // FIRE_ONCE_NOW 》 trigger
                                    JobTriggerPoolHelper.trigger(jobInfo.getId(), TriggerTypeEnum.MISFIRE, -1, null, null, null);
                                }

                                // 2、fresh next
                                // 设置下一次的触发时间
                                refreshNextValidTime(jobInfo, new Date());

                            } else if (nowTime > jobInfo.getTriggerNextTime()) {
                                // 触发时间过期时间 < 5s，立即执行一次
                                // 2.2、trigger-expire < 5s：direct-trigger && make next-trigger-time

                                // 1、trigger
                                JobTriggerPoolHelper.trigger(jobInfo.getId(), TriggerTypeEnum.CRON, -1, null, null, null);
                                // 2、fresh next
                                refreshNextValidTime(jobInfo, new Date());

                                // next-trigger-time in 5s, pre-read again
                                // 如果任务正在执行 并且 在当前时间的5s内，放进时间轮
                                if (jobInfo.getTriggerStatus() == 1 && nowTime + PRE_READ_MS > jobInfo.getTriggerNextTime()) {

                                    // 1、make ring second 时间轮
                                    int ringSecond = (int) ((jobInfo.getTriggerNextTime() / 1000) % 60);
                                    // 2、push time ring 加入时间轮
                                    pushTimeRing(ringSecond, jobInfo.getId());
                                    // 3、fresh next
                                    refreshNextValidTime(jobInfo, new Date(jobInfo.getTriggerNextTime()));

                                }

                            } else {
                                // 2.3、trigger-pre-read：time-ring trigger && make next-trigger-time

                                // 1、make ring second
                                int ringSecond = (int) ((jobInfo.getTriggerNextTime() / 1000) % 60);
                                // 2、push time ring
                                pushTimeRing(ringSecond, jobInfo.getId());
                                // 3、fresh next
                                refreshNextValidTime(jobInfo, new Date(jobInfo.getTriggerNextTime()));
                            }

                        }

                        // 3、update trigger info 更新调度信息
                        for (XxlJobInfo jobInfo : scheduleList) {
                            XxlJobAdminConfig.getAdminConfig().getXxlJobInfoDao().scheduleUpdate(jobInfo);
                        }

                    } else {
                        preReadSuc = false;
                    }
                    // tx stop
                } catch (Exception e) {
					...
                } finally {
					...
                }
            });
        }
    }
}
```

`xxl_job_info`表记录定时任务的表，里面有个`trigger_next_time`（Long）字段，表示下一次任务被触发的时间，任务每被触发一次都要更新`trigger_next_time`字段，这样就知道任务何时被触发。

- 从数据库`xxl_job_info`中读取5秒内需要执行的任务，并遍历任务。
- 如果当前时间超过下一次触发时间5秒，获取此时调度任务已经过期的调度策略的配置，默认是什么也做策略。如果配置是立即执行一次策略，那么就立即触发定时任务，否则什么也不做。最后更新下一次触发时间。
- 如果当前时间超过下一次触发时间，但并没有超过5秒，立即触发一次任务，然后更新下一次触发时间。如果任务正在运行并且更新以后的触发时间在当前时间5秒内，将任务放进时间轮，然后再次更新下一次触发时间。因为触发时间太短了所以就放进时间轮中，供下一次触发。
- 如果不是上面的两种情况，则计算时间轮，将任务放进时间轮中，最后更新下一次触发时间。
- 更新调度任务信息保存到数据库中，更新`trigger_next_time`字段。

#### 任务触发

```java
public void start(){
    ...
        
    // ring thread
    ringThread = new Thread(new Runnable() {
        @Override
        public void run() {

            while (!ringThreadToStop) {

                // align second
                try {
                    TimeUnit.MILLISECONDS.sleep(1000 - System.currentTimeMillis() % 1000);
                } catch (InterruptedException e) {
                }

                try {
                    // second data
                    List<Integer> ringItemData = new ArrayList<>();
                    int nowSecond = Calendar.getInstance().get(Calendar.SECOND);   // 避免处理耗时太长，跨过刻度，向前校验一个刻度；
                    for (int i = 0; i < 2; i++) {
                        List<Integer> tmpData = ringData.remove( (nowSecond+60-i)%60 );
                        if (tmpData != null) {
                            ringItemData.addAll(tmpData);
                        }
                    }

                    // ring trigger
                    if (ringItemData.size() > 0) {
                        // do trigger
                        for (int jobId: ringItemData) {
                            // do trigger 触发定时任务
                            JobTriggerPoolHelper.trigger(jobId, TriggerTypeEnum.CRON, -1, null, null, null);
                        }
                        // clear
                        ringItemData.clear();
                    }
                } catch (Exception e) {
                }
            }
        }
    });
    ...
}
```

首先获取当前的时间（秒数），然后从时间轮内移出当前秒数前2个秒数的任务列表，遍历任务列表触发任务的执行，最后清空已经执行的任务列表。

> 获取当前秒数前2个秒数的任务列表：避免处理时间太长导致错失了调度。

### 客户端触发



### 服务端执行