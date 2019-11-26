# 语法

## for循环起别名

别名：deep
```
deep: for (int i = 0; i < 10; i++) {
    System.out.println(i);

    for (int j = 0; j < 10; j++) {
        if (j > 5){
            // 此时跳出外部循环
            break deep;
        }
    }
}
```

## Array

+ 将Array转换List,使用`Arrays.asList(T... a)`,此时获取的List是不可变List,无法进行操作,否则抛出`UnsupportedOperationException`；
+ 原因：此时的ArrayList是Arrays中的静态内部类，同我们平常使用的ArrayList不是同一个ArrlyList。


## Map

+ Map中putIfAbsent()和put()的区别:
    1) 使用put()方法添加键值对，如果map集合中没有该key对应的值，则直接添加，并返回null，如果已经存在对应的值，则会覆盖旧值，value为新的值。

    2) 使用putIfAbsent()方法添加键值对，如果map集合中没有该key对应的值，则直接添加，并返回null，如果已经存在对应的值，则依旧为原来的值。



java容器? 同步容器和并发容器?

ArrayList和LinkedList插入和访问的时间复杂度?

java反射原理和注解原理?

HashMap的扩容和hash冲突?

线程池的原理,参数的意义,为什么使用阻塞队列?

spring中bean装载过程?

AtomicInteger为什么使用CAS,而不是sysnchronose?

JVM分区?老年代和新生代比例?使用的算法有那些?

jstack, jmap,  jutil? 如何排查线上JVM问题?

线程池正在处理任务,但服务器突然挂掉, 正在处理和在阻塞队列中的任务如何处理?

使用无界阻塞队列会出现什么情况?

HashMap和TreeMap的底层结构?

ThreadLocal的底层实现?

volatile的实现原理?

什么是CAS?
    CompareAndSwap  比较并交换

ABA问题?

线上频繁full gc,如何处理?

类加载机制和类加载器?

JVM的优化?

class.forname和 class.getClassLoader()区别?

synchronose什么时候是对象锁?什么时候是全局锁?

Mybatis设置分页?

Spring中循环依赖,如何处理?

Lock内部实现?

阻塞队列内部实现?

公平锁和非公平锁? 读写锁?

happens before原理?

JUC?

## Redis

数据库和redis如何保证双写一致?

redis如何部署的? 缓存的命中率?

热key大value问题?  热点缓存问题?

缓存高可用方案?

如何监控缓存集群的QPS的容量?

缓存集群的扩容?

key的选址算法? 选举原则?

redis的线程模型和内存模型?

缓存穿透和击穿问题?

## MQ

为什么使用MQ? 优缺点?  

消息积压如何处理? 

如何防止重复消费? 

保证MQ高可用?  

如何防止消息丢失?

死信队列?


## JVM
JVM调优和参数设置?

JAVA自带的JVM监控和性能分析工具?

垃圾回收器? 回收算法?  强.软.弱.虚引用?

OOM? OutOfMemberyError的类型?







