https://www.jianshu.com/p/4a275e779afa

https://juejin.im/post/5af02571f265da0b9e64fcfd#heading-7

http://jm.taobao.org/

https://blog.csdn.net/qq_27529917/article/details/79857207



## 启动Name Server
    start mqnamesrv.cmd	(window)

	> nohup sh bin/mqnamesrv & 
	> tail -f ~/logs/rocketmqlogs/namesrv.log	(linux)
	
	停止
	> sh bin/mqshutdown namesrv
## 启动Broker

	start mqbroker.cmd -n 127.0.0.1:9876 autoCreateTopicEnable=true	 (window)
	> nohup sh bin/mqbroker -n localhost:9876 & 
	> tail -f ~/logs/rocketmqlogs/broker.log	(linux)

	停止
	> sh bin/mqshutdown broker
	
	注: nohup Command ... &  --> 不挂断的后台运行

## Producer发送消息方式：

	同步：同步发送指消息发送方发出数据后会在收到接收方发回响应之后才发下一个数据包。一般用于重要通知消息，例如重要通知邮件、营销短信
	异步：异步发送指发送方发出数据后，不等接收方发回响应，接着发送下个数据包，一般用于可能链路耗时较长而对响应时间敏感的业务场景，例如用户视频上传后通知启动转码服务。
	单项：单向发送是指只负责发送消息而不等待服务器回应且没有回调函数触发，适用于某些耗时非常短但对可靠性要求并不高的场景，例如日志收集。
	
## Producer Group

	这类 Producer 通常发送一类消息并且发送逻辑一致，所以将这些 Producer 分组在一起。从部署结构上看生产者通过 Producer Group 的名字来标记自己是一个集群。
	
## Consumer

	拉取型消费者
	推送型消费者:首先要注册消费监听器，当监听器处触发后才开始消费消息。
	
## Consumer Group
	
## Broker 服务器

	Master和Slave
	单 Master 、多 Master 、多 Master 多 Slave（异步复制）、多 Master多 Slave（同步双写）
	
## NameServer

	每个 Broker 在启动的时候会到 NameServer 注册


## Message

	一条消息必须有一个主题（Topic），主题可以看做是你的信件要邮寄的地址。一条消息也可以拥有一个可选的标签（Tag）和额处的键值对，它们可以用于设置一个业务 key 并在 Broker 上查找此消息以便在开发期间查找问题
	
## Topic
	
## Tag

## Message Queue

    消息消费模式
    	集群(Clustering) 和 广播(Broadcasting)
    	一个消费者集群共同消费一个主题的多个队列，一个队列只会被一个消费者消费，如果某个消费者挂掉，分组内其它消费者会接替挂掉的消费者继续消费。
    	而广播消费消息会发给消费者组中的每一个消费者进行消费。
    	
    消息顺序	Message Order
    	顺序(Orderly) 和 并行(Concurrently)
    	顺序消费表示消息消费的顺序同生产者为每个消息队列发送的顺序一致，所以如果正在处理全局顺序是强制性的场景，需要确保使用的主题只有一个消息队列。
    	并行消费不再保证消息顺序，消费的最大并行数量受每个消费者客户端指定的线程池限制。
    	
	
## Producer最佳实践

	1、一个应用尽可能用一个 Topic，消息子类型用 tags 来标识，tags 可以由应用自由设置。只有发送消息设置了tags，消费方在订阅消息时，才可以利用 tags 在 broker 做消息过滤。
	2、每个消息在业务层面的唯一标识码，要设置到 keys 字段，方便将来定位消息丢失问题。由于是哈希索引，请务必保证 key 尽可能唯一，这样可以避免潜在的哈希冲突。
	3、消息发送成功或者失败，要打印消息日志，务必要打印 sendresult 和 key 字段。
	4、对于消息不可丢失应用，务必要有消息重发机制。例如：消息发送失败，存储到数据库，能有定时程序尝试重发或者人工触发重发。
	5、某些应用如果不关注消息是否发送成功，请直接使用sendOneWay方法发送消息。

## Consumer最佳实践

	1、消费过程要做到幂等（即消费端去重）
	2、尽量使用批量方式消费方式，可以很大程度上提高消费吞吐量。
	3、优化每条消息消费过程
	
## 其他配置
	线上应该关闭autoCreateTopicEnable，即在配置文件中将其设置为false。
	RocketMQ在发送消息时，会首先获取路由信息。如果是新的消息，由于MQServer上面还没有创建对应的Topic，这个时候，如果上面的配置打开的话，会返回默认TOPIC的（RocketMQ会在每台broker上面创建名为TBW102的TOPIC）路由信息，然后Producer会选择一台Broker发送消息，选中的broker在存储消息时，发现消息的topic还没有创建，就会自动创建topic。
	后果就是：以后所有该TOPIC的消息，都将发送到这台broker上，达不到负载均衡的目的。
	所以基于目前RocketMQ的设计，建议关闭自动创建TOPIC的功能，然后根据消息量的大小，手动创建TOPIC。
