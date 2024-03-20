# Redis-集群

[Redis进阶 - 高可拓展：分片技术（Redis Cluster）详解 | Java 全栈知识体系 (pdai.tech)](https://pdai.tech/md/db/nosql-redis/db-redis-x-cluster.html)

## 集群原理

集群完整性

`cluster-require-full-coverage=false`，保证高可用性

避免大集群，建立多个集群。



## 哈希槽(Hash Slot)

Redis-Cluster没有使用一致性hash，而是引入了**哈希槽**的概念。Redis-Cluster中有16384(即2^14）个哈希槽，每个key通过CRC16校验后对16383取模来决定放置哪个槽。Cluster中的每个节点负责一部分hash槽（hash slot）。

比如集群中存在三个节点，则可能存在的一种分配如下：

- 节点A包含0到5500号哈希槽；
- 节点B包含5501到11000号哈希槽；
- 节点C包含11001 到 16383号哈希槽。





### 为什么Redis Cluster的Hash Slot 是16384？

一致性hash算法是2^16，为什么**hash slot**是2^14呢？

在Redis节点发送**心跳包**时需要把所有的槽放到这个心跳包里，以便让节点知道当前集群信息，16384=16k，在发送心跳包时使用char进行bitmap压缩后是2k（2 * 8 (8 bit) * 1024(1k) = 16K），也就是说使用2k的空间创建了16k的槽数。

虽然使用CRC16算法最多可以分配65535（2^16-1）个槽位，65535=65k，压缩后就是8k（8 * 8 (8 bit) * 1024(1k) =65K），也就是说需要8k的心跳包，这样做不太值得；并且一般情况下一个Redis集群不会有超过1000个master节点，所以16k的槽位是个比较合适的选择。



### 为什么Redis Cluster中不建议使用发布订阅呢？

在集群模式下，所有的publish命令都会向所有节点（包括从节点）进行广播，造成每条publish数据都会在集群内所有节点传播一次，加重了带宽负担，对于在有大量节点的集群中频繁使用pub，会严重消耗带宽，不建议使用。

