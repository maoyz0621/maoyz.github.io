# Redis

#### 使用Pipeline或者Lua脚本减少请求次数

(1) 批量操作的命令，如mget，mset等

 (2) pipeline方式 

(3) Lua脚本



redis确实不是单线程的,更确切地说法是redis的核心业务线程只有一个,但是可以配置多个I/O线程，除此之外还有执行RDB序列化操作的时候也会开启线程





aof

pdb





原因：

内存存储，没有磁盘IO开销

单线程处理请求，避免线程切换和锁资源开销

非阻塞IO，多路复用IO

好的数据结构



1. 缩短键值对的存储长度；**序列化我们可以使用 protostuff 或 kryo，压缩我们可以使用 snappy。**

2. 使用 lazy free（延迟删除）特性；

   4.0版本

   ```
   lazyfree-lazy-eviction no
   lazyfree-lazy-expire no
   lazyfree-lazy-server-del no
   slave-lazy-flush no
   ```

   

3. 设置键值的过期时间；

4. 禁用长耗时的查询命令；

5. 使用 slowlog 优化耗时命令；

   ```
   slowlog-log-slower-than ：用于设置慢查询的评定时间，也就是说超过此配置项的命令，将会被当成慢操作记录在慢查询日志中，它执行单位是微秒 (1 秒等于 1000000 微秒)；
   slowlog-max-len ：用来配置慢查询日志的最大记录数。
   ```

6. 使用 Pipeline 批量操作数据；

7. 避免大量数据同时失效；**过期时间的基础上添加一个指定范围的随机数**

8. 客户端使用优化；**Pipeline和连接池**

9. 限制 Redis 内存大小；

10. 使用物理机而非虚拟机安装 Redis 服务；

11. 检查数据持久化策略；
    - RDB 快照 内存快照
    - AOF 文件追加 文件形式
    - 混合 在写入的时候，先把当前的数据以 RDB 的形式写入文件的开头，再将后续的操作命令以 AOF 的格式存入文件，这样既能保证 Redis 重启时的速度，又能减低数据丢失的风险。

```
aof-use-rdb-preamble  4.0版本提供
```

12. 禁用 THP 特性；

13. 使用分布式架构来增加读写速度。





内存淘汰策略:

**noeviction**：不淘汰任何数据，当内存不足时，新增操作会报错，Redis 默认内存淘汰策略；

**allkeys-lru**：淘汰整个键值中最久未使用的键值；

**allkeys-random**：随机淘汰任意键值;

**volatile-lru**：淘汰所有设置了过期时间的键值中最久未使用的键值；（默认）

**volatile-random**：随机淘汰设置了过期时间的任意键值；

**volatile-ttl**：优先淘汰更早过期的键值。

**volatile-lfu**：淘汰所有设置了过期时间的键值中，最少使用的键值；

**allkeys-lfu**：淘汰整个键值中最少使用的键值。



删除的时候建议采用渐进式的方式来完成：hscan、ltrim、sscan、zscan。

如果你使用Redis 4.0+，一条异步删除unlink就解决，就可以忽略下面内容。



其中 allkeys-xxx 表示从所有的键值中淘汰数据，而 volatile-xxx 表示从设置了过期键的键值中淘汰数据。

过期键值：贪心策略



## 删除bigKey

### 4.0+版本

```
一条异步删除unlink
```



### 编程式

#### list

```java
public void delKeyList(String key) {
    Long len = jedis.llen(key);
    int counter = 0;
    int left = 100;
    while (counter < len) {
        // 左侧截取left个 就是说，让列表只保留指定区间内的元素，不在指定区间之内的元素都将被删除。
        jedis.ltrim(key, left, len);
        counter += left;
    }
    //最终删除key
    jedis.del(key);
}
```



#### set

```java
public void delKeySet(String key) {
    String cursor = START_CURSOR;
    do {
        ScanResult<String> scanResult = jedis.sscan(key, cursor, new ScanParams().count(100));
        // 获取扫描结果
        List<String> result = scanResult.getResult();
        if (result != null && !result.isEmpty()) {
            // set remove 移除集合中一个或多个成员
            jedis.srem(key, result.toArray(new String[result.size()]));
        }
        // 获取游标
        cursor = scanResult.getCursor();
    } while (!START_CURSOR.equals(cursor));

    //最终删除key
    jedis.del(key);
}
```



#### zset

```java
public void delKeyZSet(String key) {
    String cursor = START_CURSOR;
    ScanParams scanParams = new ScanParams().count(100);
    do {
        ScanResult<Tuple> scanResult = jedis.zscan(key, cursor, scanParams);
        // 获取扫描结果
        List<Tuple> result = scanResult.getResult();
        if (result != null && !result.isEmpty()) {
            // zst remove 移除集合中一个或多个成员
            jedis.zrem(key, result.stream().map(Tuple::getElement).toArray(String[]::new));
        }
        // 获取游标
        cursor = scanResult.getCursor();
    } while (!START_CURSOR.equals(cursor));
    jedis.del(key);
}
```



#### hash

```java
public void delKeyHash(String key) {
    String cursor = START_CURSOR;
    ScanParams scanParams = new ScanParams().count(100);
    do {
        ScanResult<Map.Entry<String, String>> scanResult = jedis.hscan(key, cursor, scanParams);
        List<Map.Entry<String, String>> result = scanResult.getResult();
        if (result != null && !result.isEmpty()) {
            jedis.hdel(key, result.stream().map(Map.Entry::getKey).toArray(String[]::new));
        }
        cursor = scanResult.getCursor();
    } while (!START_CURSOR.equals(cursor));
    jedis.del(key);
}
```



## 批命令

原生和Pipeline



Pipeline 多个命令一起执行，无法保证原子性

multi 事务执行

参考文章：https://mp.weixin.qq.com/s?spm=a2c4e.10696291.0.0.54b019a4fgyMeh&__biz=Mzg2NTEyNzE0OA==&mid=2247483677&idx=1&sn=5c320b46f0e06ce9369a29909d62b401&chksm=ce5f9e9ef928178834021b6f9b939550ac400abae5c31e1933bafca2f16b23d028cc51813aec&scene=21#wechat_redirect

https://yq.aliyun.com/articles/531067


关于Redis单线程问题：Redis在处理客户端的请求时，包括获取 (socket 读)、解析、执行、内容返回 (socket 写) 等都由一个顺序串行的主线程处理，这就是所谓的“单线程”。但如果严格来讲从Redis4.0之后并不是单线程，除了主线程外，它也有后台线程在处理一些较为缓慢的操作，例如清理脏数据、无用连接的释放、大key的删除等等。

## Redis5.0



## Redis6.0

### 多线程
