# 分布式基础

## CAP理论

**C** (Consistency一致性)：对某个指定的客户端来说，**写操作后的读操作能读取到最新的数据状态**。对于数据分布在不同节点上的数据来说，如果在某个节点更新了数据，那么在其他节点如果都能读取到这个的数据，那么就称为**强一致**，如果有某个节点没有读取到，那就是分布式不一致。

一、特性：

写入主数据库成功，则向从数据库查询更新数据也成功；

写入主数据库失败，则向从数据库查询更新数据也失败。

二、如何实现：

1、写入主数据库后要将数据更新同步到从数据库；

2、写入主数据库后，在向从数据库同步期间将从数据库锁定，待同步完成之后在释放锁，以免在新数据写入成功后，向从数据库查询到旧数据。

三、特点

1、存在数据同步过程，写操作的响应会有一定的延迟；

2、为保证数据一致性会对资源暂时锁定，待同步完成之后在释放锁定资源。

3、如果请求到数据同步失败的节点，则会返回错误信息，而不会返回旧数据。



**A** (Availability可用性)：非故障的节点在合理的时间内返回合理的响应(**不是错误和超时的响应**)。任何事务操作都可以得到响应结果，不会出现响应超时和响应错误。可用性的两个关键一个是合理的时间，一个是合理的响应。

​	合理的时间指的是请求不能被阻塞，应该在合理的时间给出返回；

​	合理的响应指的是系统应该明确返回结果并且结果是正确的，这里的正确指的是比如应该返回 50，而不是返回 40。

一、特性：

从数据库接收到查询请求，立即响应查询结果；

从数据库不允许出现响应超时和响应错误。

二、如何实现：

1、写入主数据库后要将数据更新同步到从数据库；

2、保证从数据库的可用性，不可将数据库锁定；

3、从数据库要返回查询的结果，即使不是最新的数据，如果旧数据也没有，就返回默认的信息，但不能返回错误和响应超时。

三、特点：

所有请求都有结果，不会出现错误和响应超时；



**P** (Partition tolerance分区容错性)：当出现网络分区后，系统能够继续工作。打个比方，这里集群有多台机器，有台机器网络出现了问题，但是这个集群仍然可以正常工作。

一、特性

主数据库向从数据库同步数据失败，在这种情况下不影响数据库的读写操作；

某一节点挂点不影响其它节点提供服务。

二、如何实现：

1、尽量使用异步操作数据库的数据同步；

2、添加从数据库的节点，其中一个节点挂掉，其它节点提供服务。

三、特点：

最基本的功能；



#### 组合方式：

CP、AP，一般分布式系统选择AP，其中银行的跨行转账、zk则是选用的CP，要求强一致性。



## BASE理论

BA（Basically Available）指分布式系统出现故障的时候允许损失一部分的可用性，保证核心功能可用。支持分区失败；

S（Soft state）表示柔性状态，指允许系统存在中间状态，这个中间状态不会影响系统整体的可用性，比如数据库读写分离的主从同步延迟等，也就是允许短时间内不同步；

E（Eventually consistent）表示最终一致性，数据最终是一致的，但是实时是不一致的。原子性和持久性必须从根本上保障，为了可用性、性能和服务降级的需要，只有降低一致性和隔离性的要求。



## 主流注册中心产品


|      | Nacos | Eureka | Consul | CoreDNS | Zookeeper |
| ---- | ----- | ------ | ------ | ------- | --------- |
|一致性协议|	CP+AP|	AP|	CP|	—|	CP|
|健康检查	|TCP/HTTP/MYSQL/Client Beat	|Client Beat|	TCP/HTTP/gRPC/Cmd|	—	|Keep Alive|
|负载均衡策略|	权重/metadata/Selector|	Ribbon|	Fabio|	RoundRobin |	— |
|雪崩保护|	有|	|有	|无	|无	|无|
|自动注销实例|	支持|	支持|	支持|	不支持|	支持|
|访问协议|	HTTP/DNS|	HTTP|	HTTP/DNS|	DNS|	TCP|
|监听支持|	支持|	支持|	支持|	不支持	|支持|
|多数据中心|	支持|	支持|	支持|	不支持|	不支持|
|跨注册中心同步|	支持|	不支持|	支持|	不支持|	不支持|
|SpringCloud集成|	支持|	支持|	支持|	不支持|	支持|
|Dubbo集成|	支持|	不支持|	支持|	不支持|	支持|
|K8S集成|	支持|	不支持|	支持|	支持	|不支持|

