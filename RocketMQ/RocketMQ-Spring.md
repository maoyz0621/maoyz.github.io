# RocketMQ-Spring

## 消息状态

#### SendStatus   消息发送状态

- SEND_OK
- FLUSH_DISK_TIMEOUT
- FLUSH_SLAVE_TIMEOUT
- SLAVE_NOT_AVAILABLE



#### LocalTransactionState  原生事务消息

- COMMIT_MESSAGE
- ROLLBACK_MESSAGE
- UNKNOW



#### RocketMQLocalTransactionState  整合Spring事务消息

- COMMIT
- ROLLBACK
- UNKNOWN



#### ConsumeConcurrentlyStatus  负载均衡/广播方式消费消息状态

- CONSUME_SUCCESS
- RECONSUME_LATER



#### ConsumeOrderlyStatus 顺序消费状态

- SUCCESS
- SUSPEND_CURRENT_QUEUE_A_MOMENT



## 基于Netty的通信实现

RemotingServer和RemotingClient分别对应通信的服务端和客户端。

### RemotingServer

```java
public interface RemotingService {
    void start();
    void shutdown();
    void registerRPCHook(RPCHook rpcHook);
}

public interface RemotingServer extends RemotingService {

    void registerProcessor(final int requestCode, final NettyRequestProcessor processor,
        final ExecutorService executor);

    void registerDefaultProcessor(final NettyRequestProcessor processor, final ExecutorService executor);

    int localListenPort();

    Pair<NettyRequestProcessor, ExecutorService> getProcessorPair(final int requestCode);

    RemotingCommand invokeSync(final Channel channel, final RemotingCommand request,
        final long timeoutMillis) throws InterruptedException, RemotingSendRequestException,
        RemotingTimeoutException;

    void invokeAsync(final Channel channel, final RemotingCommand request, final long timeoutMillis,
        final InvokeCallback invokeCallback) throws InterruptedException,
        RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException;

    void invokeOneway(final Channel channel, final RemotingCommand request, final long timeoutMillis)
        throws InterruptedException, RemotingTooMuchRequestException, RemotingTimeoutException,
        RemotingSendRequestException;

}
```



```java
public class NettyRemotingServer extends NettyRemotingAbstract implements RemotingServer {

    public NettyRemotingServer(final NettyServerConfig nettyServerConfig,
        final ChannelEventListener channelEventListener) {
        super(nettyServerConfig.getServerOnewaySemaphoreValue(), nettyServerConfig.getServerAsyncSemaphoreValue());
        this.serverBootstrap = new ServerBootstrap();
        this.nettyServerConfig = nettyServerConfig;
        this.channelEventListener = channelEventListener;

        int publicThreadNums = nettyServerConfig.getServerCallbackExecutorThreads();
        if (publicThreadNums <= 0) {
            publicThreadNums = 4;
        }

        this.publicExecutor = Executors.newFixedThreadPool(publicThreadNums, new ThreadFactory() {
            private AtomicInteger threadIndex = new AtomicInteger(0);

            @Override
            public Thread newThread(Runnable r) {
                return new Thread(r, "NettyServerPublicExecutor_" + this.threadIndex.incrementAndGet());
            }
        });

        if (useEpoll()) {
            this.eventLoopGroupBoss = new EpollEventLoopGroup(1, new ThreadFactory() {
                private AtomicInteger threadIndex = new AtomicInteger(0);

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, String.format("NettyEPOLLBoss_%d", this.threadIndex.incrementAndGet()));
                }
            });

            this.eventLoopGroupSelector = new EpollEventLoopGroup(nettyServerConfig.getServerSelectorThreads(), new ThreadFactory() {
                private AtomicInteger threadIndex = new AtomicInteger(0);
                private int threadTotal = nettyServerConfig.getServerSelectorThreads();

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, String.format("NettyServerEPOLLSelector_%d_%d", threadTotal, this.threadIndex.incrementAndGet()));
                }
            });
        } else {
            this.eventLoopGroupBoss = new NioEventLoopGroup(1, new ThreadFactory() {
                private AtomicInteger threadIndex = new AtomicInteger(0);

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, String.format("NettyNIOBoss_%d", this.threadIndex.incrementAndGet()));
                }
            });

            this.eventLoopGroupSelector = new NioEventLoopGroup(nettyServerConfig.getServerSelectorThreads(), new ThreadFactory() {
                private AtomicInteger threadIndex = new AtomicInteger(0);
                private int threadTotal = nettyServerConfig.getServerSelectorThreads();

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, String.format("NettyServerNIOSelector_%d_%d", threadTotal, this.threadIndex.incrementAndGet()));
                }
            });
        }

        loadSslContext();
    }
    
    @Override
    public void start() {
        this.defaultEventExecutorGroup = new DefaultEventExecutorGroup(
            nettyServerConfig.getServerWorkerThreads(),
            new ThreadFactory() {

                private AtomicInteger threadIndex = new AtomicInteger(0);

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, "NettyServerCodecThread_" + this.threadIndex.incrementAndGet());
                }
            });

        prepareSharableHandlers();

        ServerBootstrap childHandler =
            this.serverBootstrap.group(this.eventLoopGroupBoss, this.eventLoopGroupSelector)
                .channel(useEpoll() ? EpollServerSocketChannel.class : NioServerSocketChannel.class)
                .option(ChannelOption.SO_BACKLOG, 1024)
                .option(ChannelOption.SO_REUSEADDR, true)
                .option(ChannelOption.SO_KEEPALIVE, false)
                .childOption(ChannelOption.TCP_NODELAY, true)
                .childOption(ChannelOption.SO_SNDBUF, nettyServerConfig.getServerSocketSndBufSize())
                .childOption(ChannelOption.SO_RCVBUF, nettyServerConfig.getServerSocketRcvBufSize())
                .localAddress(new InetSocketAddress(this.nettyServerConfig.getListenPort()))
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    public void initChannel(SocketChannel ch) throws Exception {
                        ch.pipeline()
                            .addLast(defaultEventExecutorGroup, HANDSHAKE_HANDLER_NAME, handshakeHandler)
                            .addLast(defaultEventExecutorGroup,
                                encoder,
                                new NettyDecoder(),
                                new IdleStateHandler(0, 0, nettyServerConfig.getServerChannelMaxIdleTimeSeconds()),
                                connectionManageHandler,
                                serverHandler
                            );
                    }
                });

        if (nettyServerConfig.isServerPooledByteBufAllocatorEnable()) {
            childHandler.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
        }

        try {
            ChannelFuture sync = this.serverBootstrap.bind().sync();
            InetSocketAddress addr = (InetSocketAddress) sync.channel().localAddress();
            this.port = addr.getPort();
        } catch (InterruptedException e1) {
            throw new RuntimeException("this.serverBootstrap.bind().sync() InterruptedException", e1);
        }

        if (this.channelEventListener != null) {
            this.nettyEventExecutor.start();
        }

        this.timer.scheduleAtFixedRate(new TimerTask() {

            @Override
            public void run() {
                try {
                    NettyRemotingServer.this.scanResponseTable();
                } catch (Throwable e) {
                    log.error("scanResponseTable exception", e);
                }
            }
        }, 1000 * 3, 1000);
    }
    
    @Override
    public void shutdown() {
        try {
            if (this.timer != null) {
                this.timer.cancel();
            }

            this.eventLoopGroupBoss.shutdownGracefully();

            this.eventLoopGroupSelector.shutdownGracefully();

            if (this.nettyEventExecutor != null) {
                this.nettyEventExecutor.shutdown();
            }

            if (this.defaultEventExecutorGroup != null) {
                this.defaultEventExecutorGroup.shutdownGracefully();
            }
        } catch (Exception e) {
            log.error("NettyRemotingServer shutdown exception, ", e);
        }

        if (this.publicExecutor != null) {
            try {
                this.publicExecutor.shutdown();
            } catch (Exception e) {
                log.error("NettyRemotingServer shutdown exception, ", e);
            }
        }
    }
}
```



### RemotingClient

```java
public interface RemotingClient extends RemotingService {

    void updateNameServerAddressList(final List<String> addrs);

    List<String> getNameServerAddressList();

    RemotingCommand invokeSync(final String addr, final RemotingCommand request,
        final long timeoutMillis) throws InterruptedException, RemotingConnectException,
        RemotingSendRequestException, RemotingTimeoutException;

    void invokeAsync(final String addr, final RemotingCommand request, final long timeoutMillis,
        final InvokeCallback invokeCallback) throws InterruptedException, RemotingConnectException,
        RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException;

    void invokeOneway(final String addr, final RemotingCommand request, final long timeoutMillis)
        throws InterruptedException, RemotingConnectException, RemotingTooMuchRequestException,
        RemotingTimeoutException, RemotingSendRequestException;

    void registerProcessor(final int requestCode, final NettyRequestProcessor processor,
        final ExecutorService executor);

    void setCallbackExecutor(final ExecutorService callbackExecutor);

    ExecutorService getCallbackExecutor();

    boolean isChannelWritable(final String addr);
}
```



```java
public class NettyRemotingClient extends NettyRemotingAbstract implements RemotingClient {
	public NettyRemotingClient(final NettyClientConfig nettyClientConfig,
        final ChannelEventListener channelEventListener) {
        super(nettyClientConfig.getClientOnewaySemaphoreValue(), nettyClientConfig.getClientAsyncSemaphoreValue());
        this.nettyClientConfig = nettyClientConfig;
        this.channelEventListener = channelEventListener;

        int publicThreadNums = nettyClientConfig.getClientCallbackExecutorThreads();
        if (publicThreadNums <= 0) {
            publicThreadNums = 4;
        }

        this.publicExecutor = Executors.newFixedThreadPool(publicThreadNums, new ThreadFactory() {
            private AtomicInteger threadIndex = new AtomicInteger(0);

            @Override
            public Thread newThread(Runnable r) {
                return new Thread(r, "NettyClientPublicExecutor_" + this.threadIndex.incrementAndGet());
            }
        });

        this.eventLoopGroupWorker = new NioEventLoopGroup(1, new ThreadFactory() {
            private AtomicInteger threadIndex = new AtomicInteger(0);

            @Override
            public Thread newThread(Runnable r) {
                return new Thread(r, String.format("NettyClientSelector_%d", this.threadIndex.incrementAndGet()));
            }
        });

        if (nettyClientConfig.isUseTLS()) {
            try {
                sslContext = TlsHelper.buildSslContext(true);
                log.info("SSL enabled for client");
            } catch (IOException e) {
                log.error("Failed to create SSLContext", e);
            } catch (CertificateException e) {
                log.error("Failed to create SSLContext", e);
                throw new RuntimeException("Failed to create SSLContext", e);
            }
        }
    }
    
    @Override
    public void start() {
        this.defaultEventExecutorGroup = new DefaultEventExecutorGroup(
            nettyClientConfig.getClientWorkerThreads(),
            new ThreadFactory() {

                private AtomicInteger threadIndex = new AtomicInteger(0);

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, "NettyClientWorkerThread_" + this.threadIndex.incrementAndGet());
                }
            });

        Bootstrap handler = this.bootstrap.group(this.eventLoopGroupWorker).channel(NioSocketChannel.class)
            .option(ChannelOption.TCP_NODELAY, true)
            .option(ChannelOption.SO_KEEPALIVE, false)
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, nettyClientConfig.getConnectTimeoutMillis())
            .option(ChannelOption.SO_SNDBUF, nettyClientConfig.getClientSocketSndBufSize())
            .option(ChannelOption.SO_RCVBUF, nettyClientConfig.getClientSocketRcvBufSize())
            .handler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    ChannelPipeline pipeline = ch.pipeline();
                    if (nettyClientConfig.isUseTLS()) {
                        if (null != sslContext) {
                            pipeline.addFirst(defaultEventExecutorGroup, "sslHandler", sslContext.newHandler(ch.alloc()));
                            log.info("Prepend SSL handler");
                        } else {
                            log.warn("Connections are insecure as SSLContext is null!");
                        }
                    }
                    pipeline.addLast(
                        defaultEventExecutorGroup,
                        new NettyEncoder(),
                        new NettyDecoder(),
                        new IdleStateHandler(0, 0, nettyClientConfig.getClientChannelMaxIdleTimeSeconds()),
                        new NettyConnectManageHandler(),
                        new NettyClientHandler());
                }
            });

        this.timer.scheduleAtFixedRate(new TimerTask() {
            @Override
            public void run() {
                try {
                    NettyRemotingClient.this.scanResponseTable();
                } catch (Throwable e) {
                    log.error("scanResponseTable exception", e);
                }
            }
        }, 1000 * 3, 1000);

        if (this.channelEventListener != null) {
            this.nettyEventExecutor.start();
        }
    }

    @Override
    public void shutdown() {
        try {
            this.timer.cancel();

            for (ChannelWrapper cw : this.channelTables.values()) {
                this.closeChannel(null, cw.getChannel());
            }

            this.channelTables.clear();

            this.eventLoopGroupWorker.shutdownGracefully();

            if (this.nettyEventExecutor != null) {
                this.nettyEventExecutor.shutdown();
            }

            if (this.defaultEventExecutorGroup != null) {
                this.defaultEventExecutorGroup.shutdownGracefully();
            }
        } catch (Exception e) {
            log.error("NettyRemotingClient shutdown exception, ", e);
        }

        if (this.publicExecutor != null) {
            try {
                this.publicExecutor.shutdown();
            } catch (Exception e) {
                log.error("NettyRemotingServer shutdown exception, ", e);
            }
        }
}
```

## 发送消息

修改application.properties

```properties
## application.properties
rocketmq.name-server=127.0.0.1:9876
rocketmq.producer.group=my-group
```

> 注意:
>
> 请将上述示例配置中的`127.0.0.1:9876`替换成真实RocketMQ的NameServer地址与端口

编写代码

```java
@SpringBootApplication
public class ProducerApplication implements CommandLineRunner{
    @Resource
    private RocketMQTemplate rocketMQTemplate;
    
    public static void main(String[] args){
        SpringApplication.run(ProducerApplication.class, args);
    }
    
    public void run(String... args) throws Exception {
      	//send message synchronously
        rocketMQTemplate.convertAndSend("test-topic-1", "Hello, World!");
      	//send spring message
        rocketMQTemplate.send("test-topic-1", MessageBuilder.withPayload("Hello, World! I'm from spring message").build());
        //send messgae asynchronously
      	rocketMQTemplate.asyncSend("test-topic-2", new OrderPaidEvent("T_001", new BigDecimal("88.00")), new SendCallback() {
            @Override
            public void onSuccess(SendResult var1) {
                System.out.printf("async onSucess SendResult=%s %n", var1);
            }

            @Override
            public void onException(Throwable var1) {
                System.out.printf("async onException Throwable=%s %n", var1);
            }

        });
      	//Send messages orderly
      	rocketMQTemplate.syncSendOrderly("orderly_topic",MessageBuilder.withPayload("Hello, World").build(),"hashkey")
        
        //rocketMQTemplate.destroy(); // notes:  once rocketMQTemplate be destroyed, you can not send any message again with this rocketMQTemplate
    }
    
    @Data
    @AllArgsConstructor
    public class OrderPaidEvent implements Serializable{
        private String orderId;
        
        private BigDecimal paidMoney;
    }
}
```

## 接收消息

修改application.properties

```properties
## application.properties
rocketmq.name-server=127.0.0.1:9876
```

> 注意:
>
> 请将上述示例配置中的`127.0.0.1:9876`替换成真实RocketMQ的NameServer地址与端口

编写代码

```java
@SpringBootApplication
public class ConsumerApplication{
    
    public static void main(String[] args){
        SpringApplication.run(ConsumerApplication.class, args);
    }
    
    @Slf4j
    @Service
    @RocketMQMessageListener(topic = "test-topic-1", consumerGroup = "my-consumer_test-topic-1")
    public class MyConsumer1 implements RocketMQListener<String>{
        public void onMessage(String message) {
            log.info("received message: {}", message);
        }
    }
    
    @Slf4j
    @Service
    @RocketMQMessageListener(topic = "test-topic-2", consumerGroup = "my-consumer_test-topic-2")
    public class MyConsumer2 implements RocketMQListener<OrderPaidEvent>{
        public void onMessage(OrderPaidEvent orderPaidEvent) {
            log.info("received orderPaidEvent: {}", orderPaidEvent);
        }
    }
}
```

## 事务消息

修改application.properties

```properties
## application.properties
rocketmq.name-server=127.0.0.1:9876
rocketmq.producer.group=my-group
```

> 注意:
>
> 请将上述示例配置中的`127.0.0.1:9876`替换成真实RocketMQ的NameServer地址与端口

编写代码

```java
@SpringBootApplication
public class ProducerApplication implements CommandLineRunner{
    @Resource
    private RocketMQTemplate rocketMQTemplate;

    public static void main(String[] args){
        SpringApplication.run(ProducerApplication.class, args);
    }

    public void run(String... args) throws Exception {
        try {
            // Build a SpringMessage for sending in transaction
            Message msg = MessageBuilder.withPayload(..)...;
            // In sendMessageInTransaction(), the first parameter transaction name ("test")
            // must be same with the @RocketMQTransactionListener's member field 'transName'
            rocketMQTemplate.sendMessageInTransaction("test-topic", msg, null);
        } catch (MQClientException e) {
            e.printStackTrace(System.out);
        }
    }

    // Define transaction listener with the annotation @RocketMQTransactionListener
    @RocketMQTransactionListener
    class TransactionListenerImpl implements RocketMQLocalTransactionListener {
          @Override
          public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
            // ... local transaction process, return rollback, commit or unknown
            return RocketMQLocalTransactionState.UNKNOWN;
          }

          @Override
          public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
            // ... check transaction status and return bollback, commit or unknown
            return RocketMQLocalTransactionState.COMMIT;
          }
    }
}
```

## 消息轨迹

Producer 端要想使用消息轨迹，需要多配置两个配置项:

```properties
## application.properties
rocketmq.name-server=127.0.0.1:9876
rocketmq.producer.group=my-group

rocketmq.producer.enable-msg-trace=true
rocketmq.producer.customized-trace-topic=my-trace-topic
```

Consumer 端消息轨迹的功能需要在 `@RocketMQMessageListener` 中进行配置对应的属性:

```java
@Service
@RocketMQMessageListener(
    topic = "test-topic-1", 
    consumerGroup = "my-consumer_test-topic-1",
    enableMsgTrace = true,
    customizedTraceTopic = "my-trace-topic"
)
public class MyConsumer implements RocketMQListener<String> {
    ...
}
```

> 注意:
>
> 默认情况下 Producer 和 Consumer 的消息轨迹功能是开启的且 trace-topic 为 RMQ_SYS_TRACE_TOPIC ，Consumer 端的消息轨迹 trace-topic 可以在配置文件中配置 `rocketmq.consumer.customized-trace-topic` 配置项，不需要为在每个 `@RocketMQMessageListener` 配置。

## ACL功能

Producer 端要想使用 ACL 功能，需要多配置两个配置项:

```properties
## application.properties
rocketmq.name-server=127.0.0.1:9876
rocketmq.producer.group=my-group

rocketmq.producer.access-key=AK
rocketmq.producer.secret-key=SK
```

Consumer 端 ACL 功能需要在 `@RocketMQMessageListener` 中进行配置

```java
@Service
@RocketMQMessageListener(
    topic = "test-topic-1", 
    consumerGroup = "my-consumer_test-topic-1",
    accessKey = "AK",
    secretKey = "SK"
)
public class MyConsumer implements RocketMQListener<String> {
    ...
}
```

> 注意:
>
> 可以不用为每个 `@RocketMQMessageListener` 注解配置 AK/SK，在配置文件中配置 `rocketmq.consumer.access-key` 和 `rocketmq.consumer.secret-key` 配置项，这两个配置项的值就是默认值

## 请求 应答语义支持

RocketMQ-Spring 提供 请求/应答 语义支持。

- Producer端

发送Request消息使用SendAndReceive方法

> 注意
>
> 同步发送需要在方法的参数中指明返回值类型
>
> 异步发送需要在回调的接口中指明返回值类型

```java
@SpringBootApplication
public class ProducerApplication implements CommandLineRunner{
    @Resource
    private RocketMQTemplate rocketMQTemplate;
    
    public static void main(String[] args){
        SpringApplication.run(ProducerApplication.class, args);
    }
    
    public void run(String... args) throws Exception {
        // 同步发送request并且等待String类型的返回值
        String replyString = rocketMQTemplate.sendAndReceive("stringRequestTopic", "request string", String.class);
        System.out.printf("send %s and receive %s %n", "request string", replyString);

        // 异步发送request并且等待User类型的返回值
        rocketMQTemplate.sendAndReceive("objectRequestTopic", new User("requestUserName",(byte) 9), new RocketMQLocalRequestCallback<User>() {
            @Override public void onSuccess(User message) {
                System.out.printf("send user object and receive %s %n", message.toString());
            }

            @Override public void onException(Throwable e) {
                e.printStackTrace();
            }
        }, 5000);
    }
    
    @Data
    @AllArgsConstructor
    public class User implements Serializable{
        private String userName;
    		private Byte userAge;
    }
}
```

- Consumer端

需要实现RocketMQReplyListener<T, R> 接口，其中T表示接收值的类型，R表示返回值的类型。

```java
@SpringBootApplication
public class ConsumerApplication{
    
    public static void main(String[] args){
        SpringApplication.run(ConsumerApplication.class, args);
    }
    
    @Service
    @RocketMQMessageListener(topic = "stringRequestTopic", consumerGroup = "stringRequestConsumer")
    public class StringConsumerWithReplyString implements RocketMQReplyListener<String, String> {
        @Override
        public String onMessage(String message) {
          System.out.printf("------- StringConsumerWithReplyString received: %s \n", message);
          return "reply string";
        }
      }
   
    @Service
    @RocketMQMessageListener(topic = "objectRequestTopic", consumerGroup = "objectRequestConsumer")
    public class ObjectConsumerWithReplyUser implements RocketMQReplyListener<User, User>{
        public void onMessage(User user) {
          	System.out.printf("------- ObjectConsumerWithReplyUser received: %s \n", user);
          	User replyUser = new User("replyUserName",(byte) 10);	
          	return replyUser;
        }
    }

    @Data
    @AllArgsConstructor
    public class User implements Serializable{
        private String userName;
    		private Byte userAge;
    }
}
```

## 常见问题

1. 依赖

   ```
   <!--在pom.xml中添加依赖-->
   <dependency>
       <groupId>org.apache.rocketmq</groupId>
       <artifactId>rocketmq-spring-boot-starter</artifactId>
       <version>${RELEASE.VERSION}</version>
   </dependency>
   ```

2. 生产环境有多个`nameserver`该如何连接？

   `rocketmq.name-server`支持配置多个`nameserver`地址，采用`;`分隔即可。例如：`172.19.0.1:9876;172.19.0.2:9876`

3. `rocketMQTemplate`在什么时候被销毁？

   开发者在项目中使用`rocketMQTemplate`发送消息时，不需要手动执行`rocketMQTemplate.destroy()`方法， `rocketMQTemplate`会在spring容器销毁时自动销毁。

4. 启动报错：`Caused by: org.apache.rocketmq.client.exception.MQClientException: The consumer group[xxx] has been created before, specify another name please`

   RocketMQ在设计时就不希望一个消费者同时处理多个类型的消息，因此同一个`consumerGroup`下的consumer职责应该是一样的，不要干不同的事情（即消费多个topic）。建议`consumerGroup`与`topic`一一对应。

5. 发送的消息内容体是如何被序列化与反序列化的？

   RocketMQ的消息体都是以`byte[]`方式存储。当业务系统的消息内容体如果是`java.lang.String`类型时，统一按照`utf-8`编码转成`byte[]`；如果业务系统的消息内容为非`java.lang.String`类型，则采用[jackson-databind](https://github.com/FasterXML/jackson-databind)序列化成`JSON`格式的字符串之后，再统一按照`utf-8`编码转成`byte[]`。

6. 如何指定topic的`tags`?

   RocketMQ的最佳实践中推荐：一个应用尽可能用一个Topic，消息子类型用`tags`来标识，`tags`可以由应用自由设置。 在使用`rocketMQTemplate`发送消息时，通过设置发送方法的`destination`参数来设置消息的目的地，`destination`的格式为`topicName:tagName`，`:`前面表示topic的名称，后面表示`tags`名称。

   > 注意:
   >
   > `tags`从命名来看像是一个复数，但发送消息时，目的地只能指定一个topic下的一个`tag`，不能指定多个。

7. 发送消息时如何设置消息的`key`?

   可以通过重载的`xxxSend(String destination, Message msg, ...)`方法来发送消息，指定`msg`的`headers`来完成。示例：

   ```
   Message<?> message = MessageBuilder.withPayload(payload).setHeader(MessageConst.PROPERTY_KEYS, msgId).build();
   rocketMQTemplate.send("topic-test", message);
   ```

   同理还可以根据上面的方式来设置消息的`FLAG`、`WAIT_STORE_MSG_OK`以及一些用户自定义的其它头信息。

   > 注意:
   >
   > 在将Spring的Message转化为RocketMQ的Message时，为防止`header`信息与RocketMQ的系统属性冲突，在所有`header`的名称前面都统一添加了前缀`USERS_`。因此在消费时如果想获取自定义的消息头信息，请遍历头信息中以`USERS_`开头的key即可。

8. 消费消息时，除了获取消息`payload`外，还想获取RocketMQ消息的其它系统属性，需要怎么做？

   消费者在实现`RocketMQListener`接口时，只需要起泛型为`MessageExt`即可，这样在`onMessage`方法将接收到RocketMQ原生的`MessageExt`消息。

   ```java
   @Slf4j
   @Service
   @RocketMQMessageListener(topic = "test-topic-1", consumerGroup = "my-consumer_test-topic-1")
   public class MyConsumer2 implements RocketMQListener<MessageExt>{
       public void onMessage(MessageExt messageExt) {
           log.info("received messageExt: {}", messageExt);
       }
   }
   ```

9. 如何指定消费者从哪开始消费消息，或开始消费的位置？

   消费者默认开始消费的位置请参考：[RocketMQ FAQ](http://rocketmq.apache.org/docs/faq/)。 若想自定义消费者开始的消费位置，只需在消费者类添加一个`RocketMQPushConsumerLifecycleListener`接口的实现即可。 示例如下：

   ```java
   @Slf4j
   @Service
   @RocketMQMessageListener(topic = "test-topic-1", consumerGroup = "my-consumer_test-topic-1")
   public class MyConsumer1 implements RocketMQListener<String>, RocketMQPushConsumerLifecycleListener {
       @Override
       public void onMessage(String message) {
           log.info("received message: {}", message);
       }
   
       @Override
       public void prepareStart(final DefaultMQPushConsumer consumer) {
           // set consumer consume message from now
           consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_TIMESTAMP);
           	  consumer.setConsumeTimestamp(UtilAll.timeMillisToHumanString3(System.currentTimeMillis()));
       }
   }
   ```

   同理，任何关于`DefaultMQPushConsumer`的更多其它其它配置，都可以采用上述方式来完成。

10. 如何发送事务消息？

    在客户端，首先用户需要实现RocketMQLocalTransactionListener接口，并在接口类上注解声明@RocketMQTransactionListener，实现确认和回查方法；然后再使用资源模板RocketMQTemplate， 调用方法sendMessageInTransaction()来进行消息的发布。 **注意：从RocketMQ-Spring 2.1.0版本之后，注解@RocketMQTransactionListener不能设置txProducerGroup、ak、sk，这些值均与对应的RocketMQTemplate保持一致**。

11. 如何声明不同name-server或者其他特定的属性来定义非标的RocketMQTemplate？

    第一步： 定义非标的RocketMQTemplate使用你需要的属性，可以定义与标准的RocketMQTemplate不同的nameserver、groupname等。如果不定义，它们取全局的配置属性值或默认值。

    ```java
    // 这个RocketMQTemplate的Spring Bean名是'extRocketMQTemplate', 与所定义的类名相同(但首字母小写)
    @ExtRocketMQTemplateConfiguration(nameServer="127.0.0.1:9876"
       , ... // 定义其他属性，如果有必要。
    )
    public class ExtRocketMQTemplate extends RocketMQTemplate {
      //类里面不需要做任何修改
    }
    ```

    第二步: 使用这个非标RocketMQTemplate

    ```java
    @Resource(name = "extRocketMQTemplate") // 这里必须定义name属性来指向上述具体的Spring Bean.
    private RocketMQTemplate extRocketMQTemplate; 
    ```

    接下来就可以正常使用这个extRocketMQTemplate了。

12. 如何使用非标的RocketMQTemplate发送事务消息？

    首先用户需要实现RocketMQLocalTransactionListener接口，并在接口类上注解声明@RocketMQTransactionListener，注解字段的rocketMQTemplateBeanName指明为非标的RocketMQTemplate的Bean name（若不设置则默认为标准的RocketMQTemplate），比如非标的RocketMQTemplate Bean name为“extRocketMQTemplate"，则代码如下：

    ```java
    @RocketMQTransactionListener(rocketMQTemplateBeanName = "extRocketMQTemplate")
        class TransactionListenerImpl implements RocketMQLocalTransactionListener {
              @Override
              public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
                // ... local transaction process, return bollback, commit or unknown
                return RocketMQLocalTransactionState.UNKNOWN;
              }
    
              @Override
              public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
                // ... check transaction status and return bollback, commit or unknown
                return RocketMQLocalTransactionState.COMMIT;
              }
        }
    ```

    然后使用extRocketMQTemplate调用sendMessageInTransaction()来发送事务消息。

13. MessageListener消费端，是否可以指定不同的name-server而不是使用全局定义的'rocketmq.name-server'属性值 ?

    ```java
    @Service
    @RocketMQMessageListener(
       nameServer = "NEW-NAMESERVER-LIST", // 可以使用这个optional属性来指定不同的name-server
       topic = "test-topic-1", 
       consumerGroup = "my-consumer_test-topic-1",
       enableMsgTrace = true,
       customizedTraceTopic = "my-trace-topic"
    )
    public class MyNameServerConsumer implements RocketMQListener<String> {
       ...
    }
    ```

14. 消费状态：

- CONSUME_SUCCESS    表示消费端消费成功
- RECONSUME_LATER    表示消费着消费失败，一会重新消费

msg.getReconsumeTimes()获取重试次数

15. 消息幂等

这里的关键是 **f(f(x)) = f(x)**， 翻译成通俗的解释就是： 如果有一个操作，多次执行与一次执行所产生的影响是相同的，我们就称这个操作是幂等的。

> 如果消息重试多次，消费者端对该重复消息消费多次与消费一次的结果是相同的，并且多次消费没有对系统产生副作用，那么我们就称这个过程是消息幂等的。

引发原因：

1. 发送时重复： 生产者发送消息时，消息成功投递到broker，但此时发生网络闪断或者生产者down掉，导致broker发送ACK失败。此时生产者由于未能收到消息发送响应，认为发送失败，因此尝试重新发送消息到broker。当消息发送成功后，在broker中就会存在两条相同内容的消息，最终消费者会拉取到两条内容一样并且Message ID也相同的消息。因此造成了消息的重复。
2. 消费时重复： 消费消息时同样会出现重复消费的情况。当消费者在处理业务完成返回消费状态给broker时，由于网络闪断等异常情况导致未能将消费完成的CONSUME_SUCCESS状态返回给broker。broker为了保证消息被至少消费一次的语义，会在网络环境恢复之后再次投递该条被处理的消息，最终造成消费者多次收到内容一样并且Message ID也相同的消息，造成了消息的重复。

首先我们要定义消息幂等的两要素：

- 幂等令牌
- 处理唯一性的确保

网络调用本身存在不确定性，也就是既不成功也不失败的第三种状态，即所谓的**处理中**状态，因此会有重复的情况发生。通常的方法就是要求消费方在消费消息时进行去重。

因为MessageID可能出现冲突的情况，因此不建议通过MessageID作为处理依据而应当使用业务唯一标识如：订单号、流水号等作为幂等处理的关键依据。

##### **消费端常见的幂等操作**

1. 业务操作之前进行状态查询 

   消费端开始执行业务操作时，通过幂等id首先进行业务状态的查询，如：修改订单状态环节，当订单状态为成功/失败则不需要再进行处理。那么我们只需要在消费逻辑执行之前通过订单号进行订单状态查询，一旦获取到确定的订单状态则对消息进行提交，通知broker消息状态为：ConsumeConcurrentlyStatus.CONSUME_SUCCESS 。

2. 业务操作前进行数据的检索 

   逻辑和第一点相似，即消费之前进行数据的检索，如果能够通过业务唯一id查询到对应的数据则不需要进行再后续的业务逻辑。如：下单环节中，在消费者执行异步下单之前首先通过订单号查询订单是否已经存在，这里可以查库也可以查缓存。如果存在则直接返回消费成功，否则进行下单操作。

3. 唯一性约束保证最后一道防线 

   上述第二点操作并不能保证一定不出现重复的数据，如：并发插入的场景下，如果没有乐观锁、分布式锁作为保证的前提下，很有可能出现数据的重复插入操作，因此我们务必要对幂等id添加唯一性索引，这样就能够保证在并发场景下也能保证数据的唯一性。

4. 引入锁机制 

   上述的第一点中，如果是并发更新的情况，没有使用悲观锁、乐观锁、分布式锁等机制的前提下，进行更新，很可能会出现多次更新导致状态的不准确。如：对订单状态的更新，业务要求订单只能从初始化->处理中，处理中->成功，处理中->失败，不允许跨状态更新。如果没有锁机制，很可能会将初始化的订单更新为成功，成功订单更新为失败等异常的情况。 高并发下，建议通过状态机的方式定义好业务状态的变迁，通过乐观锁、分布式锁机制保证多次更新的结果是确定的，悲观锁在并发环境不利于业务吞吐量的提高因此不建议使用。

5. 消息记录表 

   这种方案和业务层做的幂等操作类似，由于我们的消息id是唯一的，可以借助该id进行消息的去重操作，间接实现消费的幂等。首先准备一个消息记录表，在消费成功的同时插入一条已经处理成功的消息id记录到该表中，注意一定要与业务操作处于同一个事务中，当新的消息到达的时候，根据新消息的id在该表中查询是否已经存在该id，如果存在则表明消息已经被消费过，那么丢弃该消息不再进行业务操作即可。

# SpringCloud Stream

依赖



```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-dependencies</artifactId>
    <version>${spring-cloud-alibaba.version}</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-rocketmq</artifactId>
</dependency>

<dependency>
	<groupId>com.alibaba.cloud</groupId>
	<artifactId>spring-cloud-starter-stream-rocketmq</artifactId>
</dependency>
```

文件配置

com.alibaba.cloud.stream.binder.rocketmq.config.RocketMQComponent4BinderAutoConfiguration加载

spring.cloud.stream.rocketmq.binder.*

```yaml
spring:
  cloud:
    stream:
      default-binder: rocketmq
      rocketmq:
        binder:
          name-server: 192.168.107.130:9876
          access-key: AK
          secret-key: SK
          enable-msg-trace: true
          customized-trace-topic: ${spring.application.name}-trace-topic
```

加载rocketMQ配置信息：

com.alibaba.cloud.stream.binder.rocketmq.properties.RocketMQBinderConfigurationProperties

```yaml
spring:
  cloud:
    stream:
      default-binder: rocketmq
      rocketmq:
        binder:
          name-server: 192.168.107.130:9876
          access-key: AK
          secret-key: SK
          enable-msg-trace: true
          customized-trace-topic: ${spring.application.name}-trace-topic
        # rocketmq bindings
        bindings:
          my-input:  # 消费者名称  com.alibaba.cloud.stream.binder.rocketmq.properties.RocketMQConsumerProperties
            consumer:
              enable: true
              contentType: application/json
              broadcasting: false
              ordlerly: false
              delayLevelWhenNextConsume: 3
              suspendCurrentQueueTimeMillis: 3000
          my-output:  # 生产者名称 com.alibaba.cloud.stream.binder.rocketmq.properties.RocketMQProducerProperties
            producer:
              enable: true
              group: **
              maxMessageSize: 8249344
              transactional: false  # 事务
              sync: false
              vipChannelEnabled: true
              sendMessageTimeout: 3000
              compressMessageBodyThreshold: 4096
              retryTimesWhenSendFailed: 2
              retryTimesWhenSendAsyncFailed: 2
              retryNextServer: true
```



```yaml
spring:
  cloud:
    stream:
      bindings:
        # 消费者
        input:  # 通道名
          destination: test-topic-1  # 主题名
          group: test-topic-1  # 消费组名
          contentType: application/json
		  consumer:
            partitioned: true  # 开启分区消费
        # 生产者
        output:
          destination: test-topic-1  # 主题名
          contentType: application/json
          producer:
            partitionKeyExpress: <分区键>
            partitionCount: 3
```

绑定：

```java
# spring.cloud.stream.bindings:
org.springframework.cloud.stream.config.BindingServiceProperties
    # spring.cloud.stream.bindings.input/output
    org.springframework.cloud.stream.config.BindingProperties
    # spring.cloud.stream.bindings.input/output.consumer/producer
        org.springframework.cloud.stream.binder.ConsumerProperties
        org.springframework.cloud.stream.binder.ProducerProperties
```





```yaml
spring:
  cloud:
    stream:
      default-binder: rocketmq
      default:  # RocketMQExtendedBindingProperties
        # 默认生产者属性
        producer:
        # 默认消费者属性
        consumer:
          maxAttempts: 3
```

Producer Group 标识发送同一类消息的Producer，通常发送逻辑一致。发送普通消息时，仅标识使用，并无特别用处。若事务消息，如果发送某条消息的producer-A宕机，使得事务消息一直处于PREPARED状态并超时，则broker会回查同一个group的其他producer，确认这条消息应该commit 还是 rollback。

Consumer Group标识一类Consumer的集合名称，这类Consumer通常消费一类消息，且消费逻辑一致。同一个Consumer Group下的各个实例将共同消费topic的消息，起到负载均衡的作用。

注：RocketMQ要求同一个Consumer Group的消费者必须要拥有相同的注册信息，即必须要听一样的topic（并且tag也一样）。

Topic 标识一类消息的逻辑名称，消息的逻辑管理单位。无论消息生产还是消费，都需要指定Topic。

在发送的时候topic用的是destination，但是进入重试队列或者死信队列时却用的是%RETRY%group 和 %DLQ%group，根据group作为topic名称的构造依据，来新建重试和死信队列。

Topic列表：

死信列表：%DLQ%xxx-group

消费者列表：

消费模式：CLUSTERING

生产者列表：主题（Topic）；生产组（Topic）

https://github.com/apache/rocketmq-spring