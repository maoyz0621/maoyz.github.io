# Redis-数据结构

## 数据类型

- **字符串（String）**：最大不能超过512MB

  i. int 长整型
  ii. embstr 短字符(<=39字节)
  iii. raw 长字符(>39)

- **列表（List）**

  压缩表（ZipList）：当列表的元素个数小于list-max-ziplist-entries配置（默认512个），同时列表中每个元素的值都小于list-max-ziplist-value配置时（默认64字节），Redis会选用ziplist来作为列表的内部实现来减少内存的使用。

  双向链表（LinkedList）：当列表类型无法满足ziplist的条件时，Redis会使用LinkedList作为列表的内部实现。

- **哈希（Hash）**

  压缩表：当field个数不超过hash-max-ziplist-entries（默认为512个）时，并且没有大value（64个字节以上算大）

  哈希表：ziplist的两个条件只要有一个不符合就会转换为hashtable

- **集合（Set）**

  整型集合：当集合中的元素都是整数且元素个数小于set-max-intset-entries配置（默认512个）时，Redis会选用intset来作为集合的内部实现，从而减少内存的使用。

  哈希表：当集合类型无法满足intset的条件时，Redis会使用hashtable作为集合的内部实现。

  i. 给用户添加标签（共同好友，共同兴趣）

  ii.抽奖：生成随机数,集合不能存放相同的元素,因此随机pop出一个元素,可以作为中奖的号码
  
- **有序集合（ZSet）**

  压缩表：当有序集合的元素个数小于zset-max-ziplist-entries配置（默认128个），同时每个元素的值都小于zset-max-ziplist-value配置（默认64字节）时，Redis会用ziplist来作为有序集合的内部实现，ziplist可以有效减少内存的使用。

  跳跃表：当ziplist条件不满足时，有序集合会使用skiplist作为内部实现，因为此时ziplist的读写效率会下降。

  i. 用户点赞数排行
  ii. 排行榜
  iii. 展示用户信息及用户分数

- **Bitmaps **：位图

  > 位存储

  统计用户信息，活跃数，打卡，只要2个状态的都可以使用Bitmaps.

  - SETBIT key offset value：指定偏移量原来储存的位
  - GETBIT key offset ：字符串值指定偏移量上的位(bit)
  - BITCOUNT  key [start] [end]：被设置为 `1` 的位的数量
  - BITPOS  key bit [start] [end]：整数回复
  - BITOP operation destkey key [key …]：保存到 `destkey` 的字符串的长度，和输入 `key` 中最长的字符串长度相等。
  - BITFIELD

- **HyperLogLog**：基数统计

  > 基数：不重复的元素

  存在一定范围内的误差

  - PFADD  key  element  [element ...\]：添加指定元素到 HyperLogLog 中。

  - PFCOUNT  key  [key ...\]：返回给定 HyperLogLog 的基数估算值。

  - PFMERGE destkey  sourcekey  [sourcekey ...\]：将多个 HyperLogLog 合并为一个 HyperLogLog

- **Streams**

- **Geo经纬度**

  推理地理位置、两地之间的距离，底层使用的**Zset**

  - GEOADD：添加地理位置的坐标。
  - GEOPOS：获取地理位置的坐标。
  - GEODIST：计算两个位置之间的距离。
  - GEORADIUS：根据用户给定的经纬度坐标来获取指定范围内的地理位置集合。
  - GEORADIUSBYMEMBER：根据储存在位置集合里面的某个地点获取指定范围内的地理位置集合，中心点是由给定的位置元素决定的， 而不是使用经度和纬度来决定中心点。
  - GEOHASH：返回一个或多个位置对象的 geohash 值。

  i. （我附近的人）
