# Redis

## 过期删除策略

> 返回键的剩余生存时间：TTL <key> ：以秒的单位返回键 key 的剩余生存时间；PTTL <key> ：以毫秒的单位返回键 key 的剩余生存时间。

过期时间的判断：键 + 过期时间 -->  **过期字典**

- **惰性删除**（被动删除）

  放任过期键不管，在客户端访问这个key的时候，对key的过期时间进行检查，如果过期了就立即删除，不会给你返回任何东西。这样会导致很多过期key到了时间并没有被删除掉，除非你的系统去查一下那个 key，才会被Redis给删除掉

- **定期删除**（主动删除）

  Redis 会将每个设置了过期时间的 key 放入到一个独立的字典中，以后会定期遍历这个字典来删除到期的 key。

  Redis默认是每隔 100ms（**设置hz 10**）就**随机抽取**一些设置了过期时间的key，检查其是否过期，如果过期就删除。为什么要随机？假如 redis 存了几十万个 key ，每隔100ms就遍历所有的设置过期时间的 key 的话，就会给 CPU 带来很大的负载。而是采用了一种简单的贪心策略。

  ```
  # Not all tasks are performed with the same frequency, but Redis checks for
  # tasks to perform according to the specified "hz" value.
  #
  # By default "hz" is set to 10. Raising the value will use more CPU when
  # Redis is idle, but at the same time will make Redis more responsive when
  # there are many keys expiring at the same time, and timeouts may be
  # handled with more precision.
  #
  # The range is between 1 and 500, however a value over 100 is usually not
  # a good idea. Most users should use the default of 10 and raise this up to
  # 100 only in environments where very low latency is required.
  # 定期删除函数的运行频率
  hz 10
  
  # 设置最大内存，通常会设定其为物理内存的四分之三
  # maxmemory <bytes>
  ```

  

> 总结：定期删除是集中处理，惰性删除是零散处理。Redis采用惰性删除和定期删除这两种方式组合进行的



非字符串的bigKey，不要使用del删除，删除的时候建议采用渐进式的方式来完成：hscan、ltrim、sscan、zscan。

> 如果你使用Redis 4.0+，一条异步删除unlink就解决
>



不管是定期采样删除还是惰性删除都不是一种完全精准的删除，就还是会存在key没有被删除掉的场景，所以就需要内存淘汰策略进行补充。



## 内存淘汰策略

当超过能够使用的最大内存（大于**maxmemory**）时，便会触发主动淘汰内存方式，设置**maxmemory-policy**（最大内存淘汰策略），保证机器有20% -  30%的闲置内存

LRU（Least Recently Used）：最少最近使用，用当前时间减去最后一次访问时间，值越大则淘汰优先级越高。其核心思想是“**如果数据最近被访问过，那么将来被访问的几率也更高**”。

LFU（Least Frequently Used）：最少频率使用，统计每个key的访问频率，值越小淘汰优先级越高。其核心思想是“**如果数据过去被访问多次，那么将来被访问的频率也更高**”。

策略种类：

- **noeviction**：不淘汰任何数据，当内存不足时，拒绝所有写入操作，Redis 默认内存淘汰策略；

- **allkeys-lru**：淘汰整个键值中最久未使用的键值；

- **allkeys-random**：随机淘汰任意键值;

- **allkeys-lfu**：淘汰整个键值中最少使用使用频率最低的的键值。（4.0之后）

- **volatile-lru**：淘汰所有设置了过期时间的键值中最久未使用的键值；（默认）

- **volatile-random**：随机淘汰设置了过期时间的任意键值；

- **volatile-ttl**：从已设置过期时间的数据集中挑选将要过期的数据淘汰

- **volatile-lfu**：淘汰所有设置了过期时间的键值中，使用频率最低的键值；（4.0之后）


> 其中 allkeys-xxx 表示从所有的键值中淘汰数据，而 volatile-xxx 表示从设置了过期键的键值中淘汰数据。
> 过期键值：贪心策略

建议：内存淘汰策略设置为***volatile-lru***

```
############################## MEMORY MANAGEMENT ################################

# maxmemory <bytes>

# MAXMEMORY POLICY: how Redis will select what to remove when maxmemory
# is reached. You can select one from the following behaviors:
#
# volatile-lru -> Evict using approximated LRU, only keys with an expire set.
# allkeys-lru -> Evict any key using approximated LRU.
# volatile-lfu -> Evict using approximated LFU, only keys with an expire set.
# allkeys-lfu -> Evict any key using approximated LFU.
# volatile-random -> Remove a random key having an expire set.
# allkeys-random -> Remove a random key, any key.
# volatile-ttl -> Remove the key with the nearest expire time (minor TTL)
# noeviction -> Don't evict anything, just return an error on write operations.
#
# LRU means Least Recently Used
# LFU means Least Frequently Used
# Both LRU, LFU and volatile-ttl are implemented using approximated randomized algorithms.
#
# Note: with any of the above policies, Redis will return an error on write
#       operations, when there are no suitable keys for eviction.
#
#       At the date of writing these commands are: set setnx setex append
#       incr decr rpush lpush rpushx lpushx linsert lset rpoplpush sadd
#       sinter sinterstore sunion sunionstore sdiff sdiffstore zadd zincrby
#       zunionstore zinterstore hset hsetnx hmset hincrby incrby decrby
#       getset mset msetnx exec sort
# Redis默认内存淘汰策略
# maxmemory-policy noeviction

# LRU, LFU and minimal TTL algorithms are not precise algorithms but approximated
# algorithms (in order to save memory), so you can tune it for speed or
# accuracy. For default Redis will check five keys and pick the one that was
# used less recently, you can change the sample size using the following
# configuration directive.
#
# The default of 5 produces good enough results. 10 Approximates very closely
# true LRU but costs more CPU. 3 is faster but not very accurate.
# 随机挑选样例的数据
# maxmemory-samples 5

# Starting from Redis 5, by default a replica will ignore its maxmemory setting
# (unless it is promoted to master after a failover or manually). It means
# that the eviction of keys will be just handled by the master, sending the
# DEL commands to the replica as keys evict in the master side.
#
# This behavior ensures that masters and replicas stay consistent, and is usually
# what you want, however if your replica is writable, or you want the replica
# to have a different memory setting, and you are sure all the writes performed
# to the replica are idempotent, then you may change this default (but be sure
# to understand what you are doing).
#
# Note that since the replica by default does not evict, it may end using more
# memory than the one set via maxmemory (there are certain buffers that may
# be larger on the replica, or data structures may sometimes take more memory
# and so forth). So make sure you monitor your replicas and make sure they
# have enough memory to never hit a real out-of-memory condition before the
# master hits the configured maxmemory setting.
#
# replica-ignore-maxmemory yes

# Redis reclaims expired keys in two ways: upon access when those keys are
# found to be expired, and also in background, in what is called the
# "active expire key". The key space is slowly and interactively scanned
# looking for expired keys to reclaim, so that it is possible to free memory
# of keys that are expired and will never be accessed again in a short time.
#
# The default effort of the expire cycle will try to avoid having more than
# ten percent of expired keys still in memory, and will try to avoid consuming
# more than 25% of total memory and to add latency to the system. However
# it is possible to increase the expire "effort" that is normally set to
# "1", to a greater value, up to the value "10". At its maximum value the
# system will use more CPU, longer cycles (and technically may introduce
# more latency), and will tollerate less already expired keys still present
# in the system. It's a tradeoff betweeen memory, CPU and latecy.
#
# active-expire-effort 1
```


