# Redis-发布订阅模式

## 简介

> 一种消息通信模式：发送者(pub)发送消息，订阅者(sub)接收消息。

Redis 的`SUBSCRIBE`命令可以让客户端订阅任意数量的频道， 每当有新信息发送到被订阅的频道时， 信息就会被发送给所有订阅指定频道的客户端。

## 使用

> - 基于频道(Channel)的发布/订阅
> - 基于模式(pattern)的发布/订阅

### 基于频道（Channel）

"发布/订阅"模式包含两种角色，分别是发布者和订阅者。发布者可以向指定的频道(channel)发送消息；订阅者可以订阅一个或者多个频道(channel)，所有订阅此频道的订阅者都会收到此消息。

```
 # 发布
 publish channel message
 
 # 订阅
 subscribe channel1 [channel2 ...]
```

#### 如何实现？

通过字典实现的。字典就用于保存订阅频道的信息：字典的**键为正在被订阅的频道**， 而字典的**值则是一个链表， 链表中保存了所有订阅这个频道的客户端**。

- 数据结构：

<img src="..\Redis\image\Redis\发布订阅数据结构.png" alt="发布订阅数据结构"  />

- channel订阅：

<img src="..\Redis\image\Redis\发布订阅-订阅.png" alt="发布订阅-订阅"  />

- 发布

当调用 `PUBLISH channel message` 命令， 程序首先根据 channel 定位到字典的键， 然后将信息发送给字典值链表中的所有客户端。

### 基于模式（pattern）

如果有某个/某些模式和这个频道匹配的话，那么所有订阅这个/这些频道的客户端也同样会收到信息。

####  如何实现？

底层是pubsubPattern节点的链表。

- 数据结构

<img src="..\Redis\image\Redis\pattern模式数据结构.png" alt="pattern模式数据结构"  />

- 订阅

<img src="..\Redis\image\Redis\pattern模式-订阅.png" alt="pattern模式-订阅"  />

- 发布

发送信息到模式的工作也是由 PUBLISH 命令进行的, 显然就是匹配模式获得Channels，然后再把消息发给客户端。