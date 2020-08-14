# RocketMQ-Spring



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
            // ... local transaction process, return bollback, commit or unknown
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
> 默认情况下 Producer 和 Consumer 的消息轨迹功能是开启的且 trace-topic 为 RMQ_SYS_TRACE_TOPIC Consumer 端的消息轨迹 trace-topic 可以在配置文件中配置 `rocketmq.consumer.customized-trace-topic` 配置项，不需要为在每个 `@RocketMQMessageListener` 配置。

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



https://github.com/apache/rocketmq-spring