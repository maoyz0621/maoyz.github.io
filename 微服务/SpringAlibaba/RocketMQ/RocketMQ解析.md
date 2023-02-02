# RocketMQ

## Namesrv

> 负责对于数据源的管理，包括topic和路由信息
>
> 启动命令：nohup sh bin/mqnamesrv &

1. 维护心跳检测，提供Topic-Broker关系

2. 无状态的，节点之间相互没有数据通信

执行：

```shell
sh ${ROCKETMQ_HOME}/bin/runserver.sh org.apache.rocketmq.namesrv.NamesrvStartup $@
```

- runserver.sh：设置运行参数
- NamesrvStartup：执行Namesrv服务

> org.apache.rocketmq.namesrv.NamesrvController#initialize

```java
// 加载KV
this.kvConfigManager.load();
// 创建NettyServer
this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);

this.remotingExecutor =
    Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));

this.registerProcessor();
// 开启定时任务:每隔10s扫描一次Broker,移除不活跃的Broker
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
        NamesrvController.this.routeInfoManager.scanNotActiveBroker();
    }
}, 5, 10, TimeUnit.SECONDS);
// 开启定时任务:每隔10min打印一次KV配置
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
        NamesrvController.this.kvConfigManager.printAllPeriodically();
    }
}, 1, 10, TimeUnit.MINUTES);
return true;
```

> org.apache.rocketmq.remoting.netty.NettyServerConfig

```java
// Namesrv监听端口，默认初始化为9876
private int listenPort = 8888;
// Netty业务线程池线程个数
private int serverWorkerThreads = 8;
// Netty public任务线程池线程个数
private int serverCallbackExecutorThreads = 0;
// IO线程池个数
private int serverSelectorThreads = 3;
// send oneway消息请求并发读（Broker端参数）
private int serverOnewaySemaphoreValue = 256;
// 异步消息发送最大并发度
private int serverAsyncSemaphoreValue = 64;
// 网络连接最大的空闲时间，默认120s
private int serverChannelMaxIdleTimeSeconds = 120;

private int serverSocketSndBufSize = NettySystemConfig.socketSndbufSize;
private int serverSocketRcvBufSize = NettySystemConfig.socketRcvbufSize;
private boolean serverPooledByteBufAllocatorEnable = true;
private boolean useEpollNativeSelector = false;
```



## Broker

> 消息的中转中心，负责消息的存储和转发

- 单个Broker和所有的Namesrv节点保持长连接接和心跳，定时将topic注册到Namesrv
- 负责消息存储，以Topic为维度的队列

> 启动命令：nohup sh bin/mqbroker -c conf/broker-x.properties > /data/rocketmq/logs/broker-x/mqbroker.log 2>&1

执行：

```shell
sh ${ROCKETMQ_HOME}/bin/runbroker.sh org.apache.rocketmq.broker.BrokerStartup $@
```

- runbroker.sh：设置运行参数
- BrokerStartup：执行Broker服务

broker.proerties

```properties
brokerClusterName=DefaultCluster
brokerName=broker-b
brokerId=0
deleteWhen=04
fileReservedTime=48
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH
namesrvAddr=10.211.55.4:9876;10.211.55.5:9876;10.211.55.6:9876
listenPort=10911
defaultTopicQueueNums=4
autoCreateTopicEnable=false

#是否允许broker自动创建订阅组，建议线下开启，线上关闭，默认【true】
autoCreateSubscriptionGroup=false
mapedFileSizeCommitLog=1073741824 
mapedFileSizeConsumeQueue=50000000
destroyMapedFileIntervalForcibly=120000
redeleteHangedFileInterval=120000
diskMaxUsedSpaceRatio=88
storePathRootDir=/usr/local/rocketmq/data/store
storePathCommitLog=/usr/local/rocketmq/data/store/commitlog

#消费队列存储路径
storePathConsumeQueue=/usr/local/rocketmq/data/store/consumequeue

#消息索引存储路径
storePathIndex=/usr/local/rocketmq/data/store/index

#checkpoint文件存储路径
storeCheckpoint=/usr/local/rocketmq/data/store/checkpoint

#abort文件存储路径
abortFile=/usr/local/rocketmq/data/store/abort
maxMessageSize=65536
flushCommitLogLeastPages=4
flushConsumeQueueLeastPages=2
flushCommitLogThoroughInterval=10000
flushConsumeQueueThoroughInterval=60000
checkTransactionMessageEnable=false
sendMessageThreadPoolNums=128
pullMessageThreadPoolNums=128
```

- brokerClustername：broker所属集群
- brokerName：主从关系的需要保持名称一致
- brokerId：0 - Master, 大于0 - Slave
- deleteWhen：删除文件的时间点，默认为凌晨4点
- fileReservedTime：文件保留时间：默认48h
- brokerRole：ASYNC_MASTER（主从异步复制）、 SYNC_MASTER（主从同步复制）、SLAVE
- flushDiskType：SYNC_FLUSH（同步刷盘）、ASYNC_FLUSH（异步刷盘）
- namesrvAddr：namesrv集群地址
- listenPort：broker对外服务的监听端口
- autoCreateTopicEnable：是否允许broker自动创建Topic，建议线下开启，线上关闭，默认【true】



> org.apache.rocketmq.broker.BrokerController#start

```java
// 注册broker信息
this.registerBrokerAll(true, false, true);

// 每隔30s上报Broker信息到NameServer
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

    @Override
    public void run() {
        try {
            // 注册broker信息
            BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
        } catch (Throwable e) {
            log.error("registerBrokerAll Exception", e);
        }
    }
}, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS);
```

