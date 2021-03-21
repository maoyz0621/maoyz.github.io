# ElasticSearch

## 开发规范

### 系统规范
(1) 【建议】新集群申请时，建议使用>=7的版本,xpack部分功能免费，性能也更加优秀

(2) 【强制】6.8以下版本，线上禁止开启xpack的试用功能，如有需要，升级到6.8以上版本。
```
6.8以下版本xpack的security功能是收费的，如果使用不当，试用(trail)的license过期后，可能会造成数据无法读写。
```

(3) 【建议】索引至少创建1个以上的副本，提高数据的可用性

(4) 【建议】一个集群内索引数目不要太多 <500，定时做数据清理(删除或close)，避免由于历史可删除数据量过大，导致节点JVM使用率过高，造成节点假死


(5) 【建议】合理控制分片数

- 对于数据量较小（100GB以下）的index，往往写入压力查询压力相对较低，一般设置3~5个shard，numberofreplicas设置为1即可（也就是一主一从，共两副本） 。

- 对于数据量较大（100GB以上）的index，一般把单个shard的数据量控制在（20GB~50GB），让index压力分摊至多个节点：可通过index.routing.allocation.totalshardsper_node参数，强制限定一个节点上该index的shard数量，让shard尽量分配到不同节点上

- 综合考虑整个index的shard数量，如果shard数量（不包括副本）超过50个，就很可能引发拒绝率上升的问题，此时可考虑把该index拆分为多个独立的index，分摊数据量，同时配合routing使用，降低每个查询需要访问的shard数量。
- 注意： 以上设置需要考虑到当前索引的读、写QPS.过大的读QPS需要增加副本数提高读取效率

(6) 【建议】按日期可规整的数据，可以按日期建立不同的索引，比如index-YYMMDD /index-YYMM 等。方便后续的数据清理操作 

(7) 【建议】生产环境如果使用force_merge操作，需要指定max_num_segments( 期望merge到多少个segments)参数，并且关注磁盘使用率情况和机器负载情况

(8) 【强制】需要关注磁盘和jvm内存使用率，80%的阈值上限是集群稳定性的重要指标 
(9) 【强制】业务私自搭的集群，部署不规范，即使接入云平台，中间件对集群的质量不做保证


### 数据类型

(1) 【建议】枚举类型的字段，转换为keyword类型进行存储

(2) 【建议】谨慎评估字符串在ES中的存储类型，如果不需要分词,使用keyword类型，需要分词的使用text类型。避免在查询时，出现结果与自己预期不符的情况

(3) 【建议】timestamp 时间字段进行range 查询时，建议将时间维度转化为分钟级，提高搜索性能

### 数据查询

(1) 【建议】分页查询start page 较大时，避免使用from size方式 ，使用scroll 或 search After进行替换。from 过大时，会造成协调节点内存膨胀。

(2) 【建议】合理控制查询结果返回的size大小。建议不要超过100

(3) 【建议】查询时，如果可以预先知道routing值，查询参数中可以指定routing值，提高搜索效率
```
// 写入数据时指定routing
PUT route_test1/_doc/b?routing=key1
{
  "data": "b with routing"
}

-- 查询数据时指定routing
GET route_test/_search?routing=key1
{
  "query": {
    "match": {
      "data": "b"
    }
  }
}
```

(4) 【建议】查询时，如果不关注相关度排序,可以使用query-bool-filter组合取代普通query


### 数据写入

(1) 【建议】写入数据时，结合业务场景合理控制 bulk 的size大小，一般情况可>10且<1000。 数据size可以控制在5mb~15mb 

(2) 【建议】根据业务场景，可预先创建好索引，减少首次写入数据时，触发create mapping ，导致耗时突增


## 集群配置
(1) 【建议】根据业务使用场景，合理控制refresh_interval，translog.flush_threshold_size 等参数的配置

- refresh_interval:

  数据添加到索引后并不能马上被查询到，等到索引刷新后才会被查询到。 
  refresh_interval 配置的刷新间隔。
  refresh_interval 的默认值是 1s。

  当需要大量导入数据到ES中，可以将 refresh_interval 设置为 -1 以加快导入速度。导入结束后，再将 refresh_interval 设置为一个正数，例如1s。或者手动 refresh 索引

- index.translog.flush_threshold_ops:当发生多少次操作时进行一次flush。默认是 unlimited。
- index.translog.flush_threshold_size:当translog的大小达到此值时会进行一次flush操作。默认是512mb。
- index.translog.flush_threshold_period:在指定的时间间隔内如果没有进行flush操作，会进行一次强制flush操作。默认是30m。
- index.translog.interval:多少时间间隔内会检查一次translog，来进行一次flush操作。es会随机的在这个值到这个值的2倍大小之间进行一次操作，默认是5s。


## 客户端使用
【推荐】 推荐使用RestHighLevelClient

【推荐】 开启sniff节点嗅探功能

```
//构造RestClient
RestClientBuilder restClientBuilder = RestClient.builder(
        new HttpHost("ip1", 9200),
        new HttpHost("ip2", 9200)
);
Header[] defaultHeaders = new Header[]{new BasicHeader("header","value")};
restClientBuilder.setDefaultHeaders(defaultHeaders);
restClientBuilder.setRequestConfigCallback(
        new RestClientBuilder.RequestConfigCallback() {
            public RequestConfig.Builder customizeRequestConfig(RequestConfig.Builder requestConfigBuilder) {
                //读写超时时间,ms
                requestConfigBuilder.setSocketTimeout(10000);
                //tcp连接超时,ms
                requestConfigBuilder.setConnectTimeout(10000);
                //从获取池获取连接超时时间,ms
                requestConfigBuilder.setConnectionRequestTimeout(10);
                return requestConfigBuilder;
            }
        });
RestHighLevelClient restHighLevelClient = new RestHighLevelClien(restClientBuilder);
//构造sniffer
SnifferBuilder snifferBuilder = Sniffer.builder(restHighLevelClientgetLowLevelClient());
//嗅探周期ms
snifferBuilder.setSniffIntervalMillis(6000).build();

```



## API

  - 文档API: 提供对文档的增删改查操作
  - 搜索API: 提供对文档进行某个字段的查询
  - 索引API: 提供对索引进行操作
  - 查看API: 按照更直观的形式返回数据，更适用于控制台请求展示
  - 集群API: 对集群进行查看和操作的API

  ### 文档API
  - Index API: 创建并建立索引
  - Get API: 获取文档
  - DELETE API: 删除文档
  - UPDATE API: 更新文档
  - Multi Get API: 一次批量获取文档
  - Bulk API: 批量操作，批量操作中可以执行增删改查
  - DELETE By Query API: 根据查询删除
  - Term Vectors: 词组分析，只能针对一个文档
  - Multi termvectors API: 多个文档的词组分析
  >multiGet的时候内部的行为是将一个请求分为多个，到不同的node中进行请求，再将结果合并起来。
  >如果某个node的请求查询失败了，那么这个请求仍然会返回数据，只是返回的数据只有请求成功的节点的查询数据集合。
  >词组分析的功能能查出比如某个文档中的某个字段被索引分词的情况。

  [对应的接口说明和例子](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs.html)

  ### 搜索API
  - 基本搜索接口: 搜索的条件在url中
  - DSL搜索接口: 搜索的条件在请求的body中
  - 搜索模版设置接口: 可以设置搜索的模版，模版的功能是可以根据不同的传入参数，进行不同的实际搜索
  - 搜索分片查询接口: 查询这个搜索会使用到哪个索引和分片
  - Suggest接口: 搜索建议接口，输入一个词，根据某个字段，返回搜索建议。
  - 批量搜索接口: 把批量请求放在一个文件中，批量搜索接口读取这个文件，进行搜索查询
  - Count接口: 只返回符合搜索的文档个数
  - 文档存在接口: 判断是否有符合搜索的文档存在
  - 验证接口: 判断某个搜索请求是否合法，不合法返回错误信息
  - 解释接口: 使用这个接口能返回某个文档是否符合某个查询，为什么符合等信息
  - 抽出器接口: 简单来说，可以用这个接口指定某个文档符合某个搜索，事先未文档建立对应搜索

  [对应的接口说明和例子](https://www.elastic.co/guide/en/elasticsearch/reference/current/search.html)


  ### 索引API
  - 创建索引接口(POST my_index)
  - 删除索引接口(DELETE my_index)
  - 获取索引信息接口(GET my_index)
  - 索引是否存在接口(HEAD my_index)
  - 打开/关闭索引接口(my_index/_close, my_index/_open)
  - 设置索引映射接口(PUT my_index/_mapping)
  - 获取索引映射接口(GET my_index/_mapping)
  - 获取字段映射接口(GET my_index/_mapping/field/my_field)
  - 类型是否存在接口(HEAD my_index/my_type)
  - 删除映射接口(DELTE my_index/_mapping/my_type)
  - 索引别名接口(_aliases)
  - 更新索引设置接口(PUT my_index/_settings)
  - 获取索引设置接口(GET my_index/_settings)
  - 分析接口(_analyze): 分析某个字段是如何建立索引的
  - 建立索引模版接口(_template): 为索引建立模版，以后新创建的索引都可以按照这个模版进行初始化
  - 预热接口(_warmer): 某些查询可以事先预热，这样预热后的数据存放在内存中，增加后续查询效率
  - 状态接口(_status): 索引状态
  - 批量索引状态接口(_stats): 批量查询索引状态
  - 分片信息接口(_segments): 提供分片信息级别的信息
  - 索引恢复接口(_recovery): 进行索引恢复操作
  - 清除缓存接口(_cache/clear): 清除所有的缓存
  - 输出接口(_flush)
  - 刷新接口(_refresh)
  - 优化接口(_optimize): 对索引进行优化
  - 升级接口(_upgrade): 这里的升级指的是把索引升级到lucence的最新格式

  [对应的接口说明和例子](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices.html)

  ### 查看API
  - 查看别名接口(_cat/aliases): 查看索引别名
  - 查看分配资源接口(_cat/allocation)
  - 查看文档个数接口(_cat/count)
  - 查看字段分配情况接口(_cat/fielddata)
  - 查看健康状态接口(_cat/health)
  - 查看索引信息接口(_cat/indices)
  - 查看master信息接口(_cat/master)
  - 查看nodes信息接口(_cat/nodes)
  - 查看正在挂起的任务接口(_cat/pending_tasks)
  - 查看插件接口(_cat/plugins)
  - 查看修复状态接口(_cat/recovery)
  - 查看线城池接口(_cat/thread_pool)
  - 查看分片信息接口(_cat/shards)
  - 查看lucence的段信息接口(_cat/segments)

  [对应的接口说明和例子](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat.html)

  ### 集群API
  - 查看集群健康状态接口(_cluster/health)
  - 查看集群状况接口(_cluster/state)
  - 查看集群统计信息接口(_cluster/stats)
  - 查看集群挂起的任务接口(_cluster/pending_tasks)
  - 集群重新路由操作(_cluster/reroute)
  - 更新集群设置(_cluster/settings)
  - 节点状态(_nodes/stats)
  - 节点信息(_nodes)
  - 节点的热线程(_nodes/hot_threads)
  - 关闭节点(_nodes/_master/_shutdown)



## shard 的 allocate 控制
某个 shard 分配在哪个节点上，一般来说，是由 ES 自动决定的。以下几种情况会触发分配动作：
```
新索引生成
索引的删除
新增副本分片
节点增减引发的数据均衡
```

ES 提供了一系列参数详细控制这部分逻辑：
```
cluster.routing.allocation.enable 该参数用来控制允许分配哪种分片。默认是 all。可选项还包括 primaries 和 new_primaries。none 则彻底拒绝分片。该参数的作用，本书稍后集群升级章节会有说明。
cluster.routing.allocation.allow_rebalance 该参数用来控制什么时候允许数据均衡。默认是 indices_all_active，即要求所有分片都正常启动成功以后，才可以进行数据均衡操作，否则的话，在集群重启阶段，会浪费太多流量了。
cluster.routing.allocation.cluster_concurrent_rebalance 该参数用来控制集群内同时运行的数据均衡任务个数。默认是 2 个。如果有节点增减，且集群负载压力不高的时候，可以适当加大。
cluster.routing.allocation.node_initial_primaries_recoveries 该参数用来控制节点重启时，允许同时恢复几个主分片。默认是 4 个。如果节点是多磁盘，且 IO 压力不大，可以适当加大。
cluster.routing.allocation.node_concurrent_recoveries 该参数用来控制节点除了主分片重启恢复以外其他情况下，允许同时运行的数据恢复任务。默认是 2 个。所以，节点重启时，可以看到主分片迅速恢复完成，副本分片的恢复却很慢。除了副本分片本身数据要通过网络复制以外，并发线程本身也减少了一半。当然，这种设置也是有道理的——主分片一定是本地恢复，副本分片却需要走网络，带宽是有限的。从 ES 1.6 开始，冷索引的副本分片可以本地恢复，这个参数也就是可以适当加大了。
indices.recovery.concurrent_streams 该参数用来控制节点从网络复制恢复副本分片时的数据流个数。默认是 3 个。可以配合上一条配置一起加大。
indices.recovery.max_bytes_per_sec 该参数用来控制节点恢复时的速率。默认是 40MB。显然是比较小的，建议加大。
```

此外，ES 还有一些其他的分片分配控制策略。比如以 tag 和 rack_id 作为区分等。运维人员可能比较常见的策略有两种：

- 磁盘限额 
为了保护节点数据安全，ES 会定时(cluster.info.update.interval，默认 30 秒)检查一下各节点的数据目录磁盘使用情况。在达到 cluster.routing.allocation.disk.watermark.low (默认 85%)的时候，新索引分片就不会再分配到这个节点上了。在达到 cluster.routing.allocation.disk.watermark.high (默认 90%)的时候，就会触发该节点现存分片的数据均衡，把数据挪到其他节点上去。这两个值不但可以写百分比，还可以写具体的字节数。有些公司可能出于成本考虑，对磁盘使用率有一定的要求，需要适当抬高这个配置：
```
# curl -XPUT localhost:9200/_cluster/settings -d '{
    "transient" : {
        "cluster.routing.allocation.disk.watermark.low" : "85%",
        "cluster.routing.allocation.disk.watermark.high" : "10gb",
        "cluster.info.update.interval" : "1m"
    }
}'
```

- 热索引分片不均 默认情况下，ES 集群的数据均衡策略是以各节点的分片总数(indices_all_active)作为基准的。这对于搜索服务来说无疑是均衡搜索压力提高性能的好办法。但是对于 Elastic Stack 场景，一般压力集中在新索引的数据写入方面。正常运行的时候，也没有问题。但是当集群扩容时，新加入集群的节点，分片总数远远低于其他节点。这时候如果有新索引创建，ES 的默认策略会导致新索引的所有主分片几乎全分配在这台新节点上。整个集群的写入压力，压在一个节点上，结果很可能是这个节点直接被压死，集群出现异常。 所以，对于 Elastic Stack 场景，强烈建议大家预先计算好索引的分片数后，配置好单节点分片的限额。比如，一个 5 节点的集群，索引主分片 10 个，副本 1 份。则平均下来每个节点应该有 4 个分片，那么就配置：
```
# curl -s -XPUT http://127.0.0.1:9200/logstash-2015.05.08/_settings -d '{
    "index": { "routing.allocation.total_shards_per_node" : "5" }
}'
```
注意，这里配置的是 5 而不是 4。因为我们需要预防有机器故障，分片发生迁移的情况。如果写的是 4，那么分片迁移会失败。

此外，另一种方式则更加玄妙，Elasticsearch 中有一系列参数，相互影响，最终联合决定分片分配：
```
cluster.routing.allocation.balance.shard 节点上分配分片的权重，默认为 0.45。数值越大越倾向于在节点层面均衡分片。
cluster.routing.allocation.balance.index 每个索引往单个节点上分配分片的权重，默认为 0.55。数值越大越倾向于在索引层面均衡分片。
cluster.routing.allocation.balance.threshold 大于阈值则触发均衡操作。默认为1。
```
Elasticsearch 中的计算方法是：
```
(indexBalance (node.numShards(index) – avgShardsPerNode(index)) + shardBalance (node.numShards() – avgShardsPerNode)) <=> weightthreshold
```
所以，也可以采取加大 cluster.routing.allocation.balance.index，甚至设置 cluster.routing.allocation.balance.shard 为 0 来尽量采用索引内的节点均衡。
reroute 接口

上面说的各种配置，都是从策略层面，控制分片分配的选择。在必要的时候，还可以通过 ES 的 reroute 接口，手动完成对分片的分配选择的控制。

reroute 接口支持五种指令：allocate_replica, allocate_stale_primary, allocate_empty_primary，move 和 cancel。常用的一般是 allocate 和 move：

- allocate_* 指令

因为负载过高等原因，有时候个别分片可能长期处于 UNASSIGNED 状态，我们就可以手动分配分片到指定节点上。默认情况下只允许手动分配副本分片(即使用 allocate_replica)，所以如果要分配主分片，需要单独加一个 accept_data_loss 选项：
```
# curl -XPOST 127.0.0.1:9200/_cluster/reroute -d '{
  "commands" : [ {
        "allocate_stale_primary" :
            {
              "index" : "logstash-2015.05.27", "shard" : 61, "node" : "10.19.0.77", "accept_data_loss" : true
            }
        }
  ]
}'
```
注意，allocate_stale_primary 表示准备分配到的节点上可能有老版本的历史数据，运行时请提前确认一下是哪个节点上保留有这个分片的实际目录，且目录大小最大。然后手动分配到这个节点上。以此减少数据丢失。

- move 指令

因为负载过高，磁盘利用率过高，服务器下线，更换磁盘等原因，可以会需要从节点上移走部分分片：

``` 
curl -XPOST 127.0.0.1:9200/_cluster/reroute -d '{
  "commands" : [ {
        "move" :
            {
              "index" : "logstash-2015.05.22", "shard" : 0, "from_node" : "10.19.0.81", "to_node" : "10.19.0.104"
            }
        }
  ]
}'
```

分配失败原因

如果是自己手工 reroute 失败，Elasticsearch 返回的响应中会带上失败的原因。不过格式非常难看，一堆 YES，NO。从 5.0 版本开始，Elasticsearch 新增了一个 allocation explain 接口，专门用来解释指定分片的具体失败理由：
```
curl -XGET 'http://localhost:9200/_cluster/allocation/explain' -d'{ "index": "logstash-2016.10.31", "shard": 0, "primary": false
}'
```

得到的响应如下：
```
{ "shard" : { "index" : "myindex", "index_uuid" : "KnW0-zELRs6PK84l0r38ZA", "id" : 0, "primary" : false }, "assigned" : false, "shard_state_fetch_pending": false, "unassigned_info" : { "reason" : "INDEX_CREATED", "at" : "2016-03-22T20:04:23.620Z" }, "allocation_delay_ms" : 0, "remaining_delay_ms" : 0, "nodes" : { "V-Spi0AyRZ6ZvKbaI3691w" : { "node_name" : "H5dfFeA", "node_attributes" : { "bar" : "baz" }, "store" : { "shard_copy" : "NONE" }, "final_decision" : "NO", "final_explanation" : "the shard cannot be assigned because one or more allocation decider returns a 'NO' decision", "weight" : 0.06666675, "decisions" : [ { "decider" : "filter", "decision" : "NO", "explanation" : "node does not match index include filters [foo:\"bar\"]" } ] }, "Qc6VL8c5RWaw1qXZ0Rg57g" : { ...
```

这会是很长一串 JSON，把集群里所有的节点都列上来，挨个解释为什么不能分配到这个节点。

## 节点下线

集群中个别节点出现故障预警等情况，需要下线，也是 Elasticsearch 运维工作中常见的情况。如果已经稳定运行过一段时间的集群，每个节点上都会保存有数量不少的分片。这种时候通过 reroute 接口手动转移，就显得太过麻烦了。这个时候，有另一种方式：
```
curl -XPUT 127.0.0.1:9200/_cluster/settings -d '{ "transient" :{ "cluster.routing.allocation.exclude._ip" : "10.0.0.1" } }'
```

Elasticsearch 集群就会自动把这个 IP 上的所有分片，都自动转移到其他节点上。等到转移完成，这个空节点就可以毫无影响的下线了。

和 `_ip` 类似的参数还有 `_host`, `_name` 等。此外，这类参数不单是 cluster 级别，也可以是 index 级别。下一小节就是 index 级别的用例。

## 冷热数据的读写分离

Elasticsearch 集群一个比较突出的问题是: 用户做一次大的查询的时候, 非常大量的读 IO 以及聚合计算导致机器 Load 升高, CPU 使用率上升, 会影响阻塞到新数据的写入, 这个过程甚至会持续几分钟。所以，可能需要仿照 MySQL 集群一样，做读写分离。

### 实施方案

1. N 台机器做热数据的存储, 上面只放当天的数据。这 N 台热数据节点上面的 elasticsearc.yml 中配置 `node.attr.tag: hot`
2. 之前的数据放在另外的 M 台机器上。这 M 台冷数据节点中配置 `node.attr.tag: stale`
3. 模板中控制对新建索引添加 hot 标签：
```
{ "order" : 0, "template" : "*", "settings" : { "index.routing.allocation.include.tag" : "hot" } }
```
4. 每天计划任务更新索引的配置, 将 tag 更改为 stale, 索引会自动迁移到 M 台冷数据节点
```
curl -XPUT http://127.0.0.1:9200/indexname/_settings -d'

{ "index": { "routing": { "allocation": { "include": { "tag": "stale" } } } } }' 
```

这样，写操作集中在 N 台热数据节点上，大范围的读操作集中在 M 台冷数据节点上。避免了堵塞影响。

该方案运用的，是 Elasticsearch 中的 allocation filter 功能，详细说明见：https://www.elastic.co/guide/en/elasticsearch/reference/master/shard-allocation-filtering.html





## 几种扩容场景：
####  1. 磁盘使用率超过系统设置:节点磁盘使用率>=85%
es磁盘水位控制说明：

- cluster.routing.allocation.disk.watermark.low：控制磁盘使用的低水位。默认为85%，意味着如果节点磁盘使用超过85%，则ES不允许再分配新的副本分片到该节点上，待创建的主分片分配不受此参数影响。它还可以设置为绝对字节值（如500MB），以防止 Elasticsearch 在可用空间少于指定数量时分配副本分片

- cluster.routing.allocation.disk.watermark.high：控制磁盘使用的高水位。默认为90%，意味着如果磁盘空间使用高于90%时，ES将尝试将已经分配到当前节点的分片迁移到其他节点。包括副本分片。它还可以设置为绝对字节值（类似于低水位线）

- cluster.routing.allocation.disk.watermark.flood_stage: 控制洪泛水位线。它默认为95%，意味着如果节点磁盘使用率超过95%，ES会将当前节点所有分片对应的索引强制设置为只读索引块（index.blocks.read_only_allow_delete）。这是防止节点耗尽磁盘空间的最后手段。一旦有足够的磁盘空间允许索引操作继续，则必须手动释放索引块。

```
重置只读索引为可写
curl -X PUT "localhost:9200/twitter/_settings" -H 'Content-Type: application/json' -d'
{
  "index.blocks.read_only_allow_delete": null
}
'
```

由于有磁盘水位的控制，在超过阈值后，会出现 `副本分片不可分配（85%/）`; `主分片不可分配(90%)`;`索引不可写(95%)`,所以 es节点在磁盘使用率>=85% 时会触发严重告警,告警发生时的处理步骤如下：

1) 确定集群是否有历史索引可以删除，如果存在历史数据可以删除，则在云平台索引列表中删除（页面操作入口： 集群详情->索引个数->索引列表页面） 


2) 索引删除成功后磁盘使用率无明显下降或无索引可删除，则需要扩容节点，在云平台系统上扩容data节点。


####  2. 当前集群不满足查询或读写的qps要求

症状： 
a) 整个集群各节点的读或写线程队列被打满，且`长期`有`大量`的reject发生
页面入口：集群详情->节点详情-> threadPool监控



b) 节点负载过高、cpu使用率长期过高或磁盘长期繁忙
页面入口：集群详情->节点详情-> os监控


## 扩容操作：
#### 1. 操作
扩容时注意点：
1. 节点角色选择data
2. 节点规模尽量和集群中当前的数据节点规模保持一致，避免出现木桶效应，导致性能较差的节点拖慢整个集群的读、写
3. 资源不足错误可以联系韩建飞进行处理，其他错误请联系ES负责人处理

#### 2. 观察
1. 确保新扩节点成功加入集群：节点列表中节点状态为online


2. 确保集群的rebanlance开关为打开，这样各节点上的数据才会做rebanlance

方法1 ：控制台页面操作
页面入口：集群详情->更多操作->操作控制台

在复杂查询中，执行命令/_cluster/settings



方法2：线上机器curl 
```
curl -X GET "localhost:9200/_cluster/settings" 
```


如果是none,则需要设置为all



```
curl -X PUT "localhost:9200/_cluster/settings" -d '{
    "transient" : {
        "cluster.routing.allocation.enable" : "all"
    }
}'
```
3. 确保新加入节点上有数据 

```
curl -X PUT "localhost:9200/_cat/shards"

```

### 注意事项：
1. 扩容前请合理评估集群情况
2. 扩容期间由于会发生节点间索引数据的迁移，对磁盘io和cpu都会造成一定的影响
3. 可能由于某些大索引设置的分片数远远小于节点数，导致原有集群节点上，每个节点至多有该索引的一个主分片或副本分片，这种扩容操作后并不能降低磁盘的使用率，需要业务谨慎考虑是否要删除部分历史数据 



问题：

> 线上存在一些低版本的es集群，使用了xpack的高级特性，如安全((从 6.8.0 和 7.1.0 版本开始，安全功能免费提供)、sql 等功能，在集群申请的时候自己将license改成了trial模式 （30天过期），过期后发现集群不可用了


问题临时修复办法： 去掉xpack 高级特性的功能，将集群license恢复到basic 


####  处理步骤

0. 去掉yml中的license trial配置，并重启节点
```
# xpack.license.self_generated.type: trial
```

1. 重新注册basic的license

    https://register.elastic.co/

2. 更新licence
```
curl -XPUT -u <user> 'http://<host>:<port>/_license' -H "Content-Type: application/json" -d @license.json
```
3. licence 生效
```
curl -XPUT -u <user> 'http://<host>:<port>/_license?acknowledge=true' -H "Content-Type: application/json" -d @license.json
```

4. 查看licence信息
```
curl -X GET -u <user>'http://<host>:<port>/_license' 

{
  "license": {
    "status": "active",
    "uid": "98897b19-1afc-41e9-aa76-4b788ca3b786",
    "type": "basic",
    "issue_date": "2020-06-16T03:46:30.836Z",
    "issue_date_in_millis": 1592279190836,
    "max_nodes": 1000,
    "issued_to": "nvme-disk",
    "issuer": "elasticsearch",
    "start_date_in_millis": -1
  }
}
```





ES7.0 type移除之后的操作

随着 7.0 版本的即将发布，type 的移除也是越来越近了，在 6.0 的时候，已经默认只能支持一个索引一个 type 了，7.0 版本新增了一个参数 include_type_name ，即让所有的 API 是 type 相关的，这个参数在 7.0 默认是 true，不过在 8.0 的时候，会默认改成 false，也就是不包含 type 信息了，这个是 type 用于移除的一个开关。

让我们看看最新的使用姿势吧，当 include_type_name 参数设置成 false 后：

- 索引操作：
PUT {index}/{type}/{id}需要修改成PUT {index}/_doc/{id}
- Mapping 操作
PUT {index}/{type}/_mapping 则变成 PUT {index}/_mapping
- 所有增删改查搜索操作返回结果里面的关键字 _type 都将被移除

```
#创建索引
PUT twitter
{
  "mappings": {
    "_doc": {
      "properties": {
        "type": { "type": "keyword" }, 
        "name": { "type": "text" },
        "user_name": { "type": "keyword" },
        "email": { "type": "keyword" },
        "content": { "type": "text" },
        "tweeted_at": { "type": "date" }
      }
    }
  }
}

#修改索引
PUT twitter/_doc/user-kimchy
{
  "type": "user", 
  "name": "Shay Banon",
  "user_name": "kimchy",
  "email": "shay@kimchy.com"
}

#搜索
GET twitter/_search
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "user_name": "kimchy"
        }
      },
      "filter": {
        "match": {
          "type": "tweet" 
        }
      }
    }
  }
}

#重建索引
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter"
  }
}
```





1. ` None of the configured nodes are available: [{#transport#-1}{SDrkaIGVRtGjRMqUtrRFsg}{10.32.42.22}{10.32.42.22:9301}, {#transport#-2}{9zdQW7H3TjiWV_osss61FQ}{10.32.38.46}{10.32.38.46:9301}, {#transport#-3}{KGX9CY_yQQ6Yl7CJijySjA}{10.32.38.38}{10.32.38.38:9301}]
at org.elasticsearch.client.transport.TransportClientNodesService.ensureNodesAreAvailable(TransportClientNodesService.java:347) ~[elasticsearch-6.1.3.jar:6.1.3]`

可能原因：

 a) 配置的集群名称和实际集群名称不一致： 修改名称为集群接入名

 b) es客户端版本与集群es版本不不兼容： 修改成一致的版本
查看版本：

pom配置：


c）节点ip或端口配置错误,比如将transport 设置成了http port 

d) 底层依赖的netty包有冲突： mvn dependency:tree 分析下netty 包的依赖情况

e) 集群中的节点异常

2.
`org.elasticsearch.common.util.concurrent.EsRejectedExecutionException: rejected execution of org.elasticsearch.search.SearchService$3@4d6f25c2 on EsThreadPoolExecutor [search, queue capacity = 1000,org.elasticsearch.common.util.concurrent.EsThreadPoolExecutor@24522e89 [Running, pool size = 13, active threads = 13, queued tasks = 52728, completed tasks = 314227215]]`

search线程队列满了，导致search 任务被拒绝

可能原因：

a) 设置的队列、线程大小过小。 建议设置值：quene_size= 2000，poll_size= int（（ 核心数 ＊ 3 ）／ 2 ）＋ 1 。 
同时满足：不允许bulk和’indexing’线程池的大小大于CPU内核数。
举例：24核处理器，检索服务器是24核，所以：线程池的大小=（24*3）/2+1=37， 
同时要满足cpu核数为24。37和24取最小值，应该选择24。
b) 使用_bulk操作索引 ，一次bulk包含多个操作，转单笔请求为批量请求 
c) 集群性能不满足





## es容量规划

在使用ES之前，需要优先ES资源容量。我们根据实际测试结果和用户使用经验，提供了相对通用的评估方法，可以作为参考。

### 磁盘容量
ES集群磁盘空间大小影响因素：
```
副本数量，至少1个副本。
索引开销，通常比源数据大10%（_all 等未计算）。
操作系统预留，默认操作系统会保留5%的文件系统供用户处理关键流程，系统恢复，磁盘碎片等。
ES内部开销，段合并，日志等内部操作，预留20%。
安全阈值，通常至少预留15%的安全阈值。
```

最小磁盘总大小 = 源数据 * 3.4
```
计算方法：
磁盘总大小 = 源数据 * (1 + 副本数量) * (1 + 索引开销) / (1 - Linux预留空间) / (1 - ES开销) / (1 - 安全阈值)
= 源数据 * (1 + 副本数量) * 1.7
= 源数据 * 3.4
```

对于 `_all` 这项参数，如果在业务使用上没有必要，我们通常的建议是禁止或者有选择性的添加。

对于需要开启这个参数的索引，其开销也会随之增大。根据我们的测试结果和使用经验，建议在上述评估的基础上额外增加一半的空间：
```
磁盘总大小 = 源数据 * (1 + 副本数) * 1.7 * (1 + 0.5) 
= 源数据 * 5.1
```

### 集群规格评估

ES的单机规格在一定程度上是限制了集群能力的，这里我们根据测试结果和使用经验给出如下建议。
```集群最大节点数 = 单节点CPU * 5```

使用场景不同，单节点最大承载数据量也会不同，具体如下：
```
数据加速、查询聚合等场景
单节点最大数据量 = 单节点Mem(G) * 10

日志写入、离线分析等场景
单节点最大数据量 = 单节点Mem(G) * 50

通常情况
单节点最大数据量 = 单节点Mem(G) * 30
```

### 参考列表

| 规格    | 最大节点数 | 单节点最大磁盘（查询） | 单节点最大磁盘（日志） | 单节点最大磁盘（通常） |
| ------- | ---------- | ---------------------- | ---------------------- | ---------------------- |
| 2C 4G   | 10         | 40 GB                  | 200 GB                 | 100 GB                 |
| 2C 8G   | 10         | 80 GB                  | 400 GB                 | 200 GB                 |
| 4C 16G  | 20         | 160 GB                 | 800 GB                 | 512 GB                 |
| 8C 32G  | 40         | 320 GB                 | 1.5 TB                 | 1 TB                   |
| 16C 64G | 50         | 640 GB                 | 2 TB                   | 2 TB                   |

### shard评估

shard大小和个数是影响ES集群稳定性和性能的重要因素之一。ES集群中任何一个索引都需要有一个合理的shard规划，很多情况下都会有一个更好的策略来替换ES默认的5个shard。
```
    建议在小规格节点下单shard大小不要超过30GB。更高规格的节点单shard大小不要超过50GB。
    对于日志分析场景或者超大索引，建议单shard大小不要超过100GB。
    shard的个数（包括副本）要尽可能匹配节点数，等于节点数，或者是节点数的整数倍。
    通常我们建议单节点上同一索引的shard个数不要超5个。
```
说明

由于不同用户在数据结构，查询复杂度，数据量大小，性能要求，数据的变化等等诸多方面是不同的，所以本文的评估绝不是每一个用户的最佳建议。





### 问题现象1 

> 1. 操作索引时报错：FORBIDDEN/12/index read-only / allow delete (api)
> 2. ES只能读和删，不能增和改

### 问题现象2

> 1. 取出的数据，排序乱了

### 确认方式

  可以使用`/{indexName}/_settings` 查看是否有 `index.block.read_only_allow_delete=true`的配置

### 原因

1. 内存不足

JVMMemoryPressure 超过92%并持续30分钟时，ES触发保护机制，并且阻止写入操作，以防止集群达到红色状态，启用写保护后，写入操作将失败，并且抛出 ClusterBlockException ，无法创建新索引，并且抛出 IndexCreateBlockException ,当五分钟内恢复不到88%以下时，将禁用写保护。

2. 磁盘空间不足

es的默认磁盘水位警戒线是85%，一旦磁盘使用率超过85%，es不会再为该节点分配分片，es还有一个磁盘水位警戒线是90%，超过后，将尝试将分片重定位到其他节点。

解决方案

1. 磁盘扩容

2. 删除无用索引

3. 将旧索引的副本数调小

4. 增加数据节点

5. 手动将 index.blocks.read_only_allow_delete 改成false

read_only_allow_delete属性
此属性为true时，ES索引只允许读和删数据，不允许增和改数据

1. 查看指定索引的设置信息

curl -XGET http://127.0.0.1:9200/blog/_settings?pretty
当索引不能增和改时，通过此命令，可以看到read_only_allow_delete为true

2. 把read_only_allow_delete设置为false

```
curl -XPUT -H "Content-Type: application/json" http://127.0.0.1:9200/blog/_settings -d'{
    "index.blocks.read_only_allow_delete": false
}'
```



一般这种情况下，不只是一个索引会出问题，可以使用_all将所有索引的配置修改 

```
curl -XPUT -H "Content-Type: application/json" http://127.0.0.1:9200/_all/_settings -d'{
    "index.blocks.read_only_allow_delete": false
}
```





### es日常运维命令
#### 1. 查看并修改模板
0) 查看已有模板
```
curl -X GET 'http://10.35.80.38:9201/_template?pretty'
```
1) 删除已有模板
```
curl -X DELETE 'http://10.35.80.38:9201/_template/connector-heartbeat'
curl -X DELETE 'http://10.35.80.38:9201/_template/connector-connection'
````
2) 创建新模板
```
curl -X PUT -H 'ContentType: application/json'  'http://10.35.80.38:9201/_template/connector-heartbeat' -d'{
    "order": 0,
    "template": "connector-heartbeat*",
    "settings": {    
        "index": {
            "refresh_interval": "300s",
            "number_of_shards": "36",
            "translog": {
                "flush_threshold_size": "6GB",
                "sync_interval": "60s",
                "durability": "async"
            },
            "merge": {
                "scheduler": {
                    "max_thread_count": "1"
                }
            },
            "store": {
                "throttle": {
                    "max_bytes_per_sec": "200mb"
                }
            },
            "unassigned": {
                "node_left": {
                    "delayed_timeout": "60m"
                }
            },
            "number_of_replicas": "1"
        }
    },
    "mappings": {
        "type_push": {
            "properties": {
                "deviceId": {
                    "type": "keyword"
                },
                "eventType": {
                    "type": "keyword"
                },
                "connectionTime": {
                    "type": "date",
                    "format": "epoch_millis"
                },
                "connectorIp": {
                    "type": "ip"
                },
                "connectorClientIndex": {
                    "type": "keyword"
                },
                "heartbeatTime": {
                    "type": "date",
                    "format": "epoch_millis"
                }
            }
        }
    },
    "aliases": {
        
    }
}'

```
3） 增加慢查询日志
```
PUT /index/_settings
{
    "index.search.slowlog.threshold.query.warn" : "100ms", 
    "index.search.slowlog.threshold.fetch.warn": "100ms", 
    "index.indexing.slowlog.threshold.index.warn": "100ms" 
}
```



### Es调优建议
1) 控制字段的存储选项

ES底层使用Lucene存储数据，主要包括行存（StoreFiled）、列存（DocValues）和倒排索引（InvertIndex）三部分。 大多数使用场景中，没有必要同时存储这三个部分，可以通过下面的参数来做适当调整：
- StoreFiled： 行存，其中占比最大的是source字段，它控制doc原始数据的存储。在写入数据时，ES把doc原始数据的整个json结构体当做一个string，存储为source字段。查询时，可以通过source字段拿到当初写入时的整个json结构体。 所以，如果没有取出整个原始json结构体的需求，可以通过下面的命令，在mapping中关闭source字段或者只在source中存储部分字段，数据查询时仍可通过ES的docvaluefields获取所有字段的值。
> 注意：关闭source后， update, updatebyquery, reindex等接口将无法正常使用，所以有update等需求的index不能关闭source。

```
## 关闭 _source
PUT my_index 
{
	"mappings": {
		"my_type": {
			"_source": {
				"enabled": false
			}
		}
	}
}

# _source只存储部分字段，通过includes指定要存储的字段或者通过excludes滤除不需要的字段
PUT my_index
{
	"mappings": {
		"_doc": {
			"_source": {
				"includes": [
					"*.count",
					"meta.*"
				],
				"excludes": [
					"meta.description",
					"meta.other.*"
				]
			}
		}
	}
}
```

- docvalues：控制列存。
ES主要使用列存来支持sorting, aggregations和scripts功能，对于没有上述需求的字段，可以通过下面的命令关闭docvalues，降低存储成本。
```json
PUT my_index
{
	"mappings": {
		"my_type": {
			"properties": {
				"session_id": {
					"type": "keyword",
					"doc_values": false
				}
			}
		}
	}
}
```
- index：控制倒排索引。

ES默认对于所有字段都开启了倒排索引，用于查询。对于没有查询需求的字段，可以通过下面的命令关闭倒排索引。
```
PUT my_index
{
	"mappings": {
		"my_type": {
			"properties": {
				"session_id": {
					"type": "keyword",
					"index": false
				}
			}
		}
	}
}
```
- all：ES的一个特殊的字段，ES把用户写入json的所有字段值拼接成一个字符串后，做分词，然后保存倒排索引，用于支持整个json的全文检索。

这种需求适用的场景较少，可以通过下面的命令将all字段关闭，节约存储成本和cpu开销。（ES 6.0+以上的版本不再支持_all字段，不需要设置）
```
PUT /my_index
{
	"mapping": {
		"my_type": {
			"_all": {
				"enabled": false
			}
		}
	}
}
```
- fieldnames：该字段用于exists查询，来确认某个doc里面有无一个字段存在。若没有这种需求，可以将其关闭。
```
PUT /my_index
{
	"mapping": {
		"my_type": {
			"_field_names": {
				"enabled": false
			}
		}
	}
}
```

2)  开启最佳压缩

对于打开了上述_source字段的index，可以通过下面的命令来把lucene适用的压缩算法替换成 DEFLATE，提高数据压缩率。
```
PUT /my_index/_settings
{
    "index.codec": "best_compression"
}
```
3) bulk批量写入

写入数据时尽量使用下面的bulk接口批量写入，提高写入效率。每个bulk请求的doc数量设定区间推荐为1k~1w，具体可根据业务场景选取一个适当的数量。
```
POST _bulk
{
	"index": {
		"_index": "test",
		"_type": "type1"
	}
}
```
4)  调整translog同步策略

默认情况下，translog的持久化策略是，对于每个写入请求都做一次flush，刷新translog数据到磁盘上。这种频繁的磁盘IO操作是严重影响写入性能的，如果可以接受一定概率的数据丢失（这种硬件故障的概率很小），可以通过下面的命令调整 translog 持久化策略为异步周期性执行，并适当调整translog的刷盘周期。
```
PUT my_index
{
	"settings": {
		"index": {
			"translog": {
				"sync_interval": "5s",
				"durability": "async"
			}
		}
	}
}
```
5)  调整refresh_interval

写入Lucene的数据，并不是实时可搜索的，ES必须通过refresh的过程把内存中的数据转换成Lucene的完整segment后，才可以被搜索。默认情况下，ES每一秒会refresh一次，产生一个新的segment，这样会导致产生的segment较多，从而segment merge较为频繁，系统开销较大。如果对数据的实时可见性要求较低，可以通过下面的命令提高refresh的时间间隔，降低系统开销。
```
PUT my_index{  "settings": {    "index": {        "refresh_interval" : "30s"    }  }}
```
6)  merge并发控制

ES的一个index由多个shard组成，而一个shard其实就是一个Lucene的index，它又由多个segment组成，且Lucene会不断地把一些小的segment合并成一个大的segment，这个过程被称为merge。默认值是Math.max(1, Math.min(4, Runtime.getRuntime().availableProcessors() / 2))，当节点配置的cpu核数较高时，merge占用的资源可能会偏高，影响集群的性能，可以通过下面的命令调整某个index的merge过程的并发度：
```
PUT /my_index/_settings{    "index.merge.scheduler.max_thread_count": 2}
```
7) 写入数据不指定_id，让ES自动产生

当用户显示指定id写入数据时，ES会先发起查询来确定index中是否已经有相同id的doc存在，若有则先删除原有doc再写入新doc。这样每次写入时，ES都会耗费一定的资源做查询。如果用户写入数据时不指定doc，ES则通过内部算法产生一个随机的id，并且保证id的唯一性，这样就可以跳过前面查询id的步骤，提高写入效率。 所以，在不需要通过id字段去重、update的使用场景中，写入不指定id可以提升写入速率。基础架构部数据库团队的测试结果显示，无id的数据写入性能可能比有_id的高出近一倍，实际损耗和具体测试场景相关。
```
# 写入时指定_id
POST _bulk
{
	"index": {
		"_index": "test",
		"_type": "type1",
		"_id": "1"
	}
}
# 写入时不指定_id
POST _bulk
{
	"index": {
		"_index": "test",
		"_type": "type1"
	}
}
```
8) 使用routing

对于数据量较大的index，一般会配置多个shard来分摊压力。这种场景下，一个查询会同时搜索所有的shard，然后再将各个shard的结果合并后，返回给用户。对于高并发的小查询场景，每个分片通常仅抓取极少量数据，此时查询过程中的调度开销远大于实际读取数据的开销，且查询速度取决于最慢的一个分片。开启routing功能后，ES会将routing相同的数据写入到同一个分片中（也可以是多个，由index.routingpartitionsize参数控制）。如果查询时指定routing，那么ES只会查询routing指向的那个分片，可显著降低调度开销，提升查询效率。 routing的使用方式如下：
```
# 写入
PUT my_index/my_type/1?routing=user1
{  "title": "This is a document"}

# 查询
GET my_index/_search?routing=user1,user2 
{
	"query": {
		"match": {
			"title": "document"
		}
	}
}
```
9) 为string类型的字段选取合适的存储方式

存为text类型的字段（string字段默认类型为text）： 做分词后存储倒排索引，支持全文检索，可以通过下面几个参数优化其存储方式：
- norms：用于在搜索时计算该doc的_score（代表这条数据与搜索条件的相关度），如果不需要评分，可以将其关闭。
- indexoptions：控制倒排索引中包括哪些信息（docs、freqs、positions、offsets）。对于不太注重score/highlighting的使用场景，可以设为 docs来降低内存/磁盘资源消耗。
- fields: 用于添加子字段。对于有sort和聚合查询需求的场景，可以添加一个keyword子字段以支持这两种功能。
``` 
PUT my_index
{
	"mappings": {
		"my_type": {
			"properties": {
				"title": {
					"type": "text",
					"norms": false,
					"index_options": "docs",
					"fields": {
						"raw": {
							"type": "keyword"
						}
					}
				}
			}
		}
	}
}
```
存为keyword类型的字段： 不做分词，不支持全文检索。text分词消耗CPU资源，冗余存储keyword子字段占用存储空间。如果没有全文索引需求，只是要通过整个字段做搜索，可以设置该字段的类型为keyword，提升写入速率，降低存储成本。 设置字段类型的方法有两种：一是创建一个具体的index时，指定字段的类型；二是通过创建template，控制某一类index的字段类型。
```
# 1. 通过mapping指定 tags 字段为keyword类型
PUT my_index
{
	"mappings": {
		"my_type": {
			"properties": {
				"tags": {
					"type": "keyword"
				}
			}
		}
	}
}

# 2. 通过template，指定my_index*类的index，其所有string字段默认为keyword类型
PUT _template/my_template
{
	"order": 0,
	"template": "my_index*",
	"mappings": {
		"_default_": {
			"dynamic_templates": [
				{
					"strings": {
						"match_mapping_type": "string",
						"mapping": {
							"type": "keyword",
							"ignore_above": 256
						}
					}
				}
			]
		}
	},
	"aliases": {}
}
```

10) 查询时，使用query-bool-filter组合取代普通query

默认情况下，ES通过一定的算法计算返回的每条数据与查询语句的相关度，并通过score字段来表征。但对于非全文索引的使用场景，用户并不care查询结果与查询条件的相关度，只是想精确的查找目标数据。此时，可以通过query-bool-filter组合来让ES不计算score，并且尽可能的缓存filter的结果集，供后续包含相同filter的查询使用，提高查询效率。
```	
# 普通查询POST 
my_index/_search
{  "query": {    "term" : { "user" : "Kimchy" }   }}

# query-bool-filter 加速查询
POST my_index/_search
{
	"query": {
		"bool": {
			"filter": {
				"term": {
					"user": "Kimchy"
				}
			}
		}
	}
}
```
11) index按日期滚动，便于管理

写入ES的数据最好通过某种方式做分割，存入不同的index。常见的做法是将数据按模块/功能分类，写入不同的index，然后按照时间去滚动生成index。这样做的好处是各种数据分开管理不会混淆，也易于提高查询效率。同时index按时间滚动，数据过期时删除整个index，要比一条条删除数据或deletebyquery效率高很多，因为删除整个index是直接删除底层文件，而deletebyquery是查询-标记-删除。

12) 按需控制index的分片数和副本数

分片（shard）：一个ES的index由多个shard组成，每个shard承载index的一部分数据。

副本（replica）：index也可以设定副本数（numberofreplicas），也就是同一个shard有多少个备份。对于查询压力较大的index，可以考虑提高副本数（numberofreplicas），通过多个副本均摊查询压力。

shard数量（numberofshards）设置过多或过低都会引发一些问题：shard数量过多，则批量写入/查询请求被分割为过多的子写入/查询，导致该index的写入、查询拒绝率上升；对于数据量较大的inex，当其shard数量过小时，无法充分利用节点资源，造成机器资源利用率不高 或 不均衡，影响写入/查询的效率。

对于每个index的shard数量，可以根据数据总量、写入压力、节点数量等综合考量后设定，然后根据数据增长状态定期检测下shard数量是否合理。

目前的一种推荐方案是：

- 对于数据量较小（100GB以下）的index，往往写入压力查询压力相对较低，一般设置3~5个shard，numberofreplicas设置为1即可（也就是一主一从，共两副本） 。

- 对于数据量较大（100GB以上）的index，一般把单个shard的数据量控制在（20GB~50GB），让index压力分摊至多个节点：可通过index.routing.allocation.totalshardsper_node参数，强制限定一个节点上该index的shard数量，让shard尽量分配到不同节点上

- 综合考虑整个index的shard数量，如果shard数量（不包括副本）超过50个，就很可能引发拒绝率上升的问题，此时可考虑把该index拆分为多个独立的index，分摊数据量，同时配合routing使用，降低每个查询需要访问的shard数量。

13) 合理选择数据类型 

- 枚举类型的字段使用`keyword`类型，不要用Number数字类型
- 遇到时间字段的`range query`区间查询时，时间粒度尽量不要在ms级别，根据业务可接受的情况，将时间转换成`秒`甚至是`分钟`级别
- 字符串类型使用时，如果没有分词场景，使用`keyword`类型





## 模板中修改原有分词器

1. 将原有字符串中的逗号，转为空格，然后再进行标准分词
```
{
    "index_patterns": [
        "trace_request_*",
        "trace_third_party_invocation_*"
    ],
    "settings": {
        "index": {
            "refresh_interval": "30s",
            "number_of_shards": "15",
            "translog": {
                "flush_threshold_size": "200mb"
            },
            "number_of_replicas": "1"
        },
        "analysis": {
            "char_filter": {
                "comma_replace_char_filter": {
                    "type": "pattern_replace",
                    "pattern": ",",
                    "replacement": " "
                }
            },
            "analyzer": {
                "my_custom_analyzer": {
                    "tokenizer": "standard",
                    "char_filter": [
                        "comma_replace_char_filter"
                    ]
                }
            }
        }
    },
    "mappings": {
        "properties": {
            "traceId": {
                "type": "keyword"
            },
            "requestBytes": {
                "index": false,
                "type": "double"
            },
            "responseBytes": {
                "index": false,
                "type": "double"
            },
            "errMsg": {
                "type": "keyword"
            },
            "level": {
                "type": "keyword"
            },
            "nodeIp": {
                "type": "keyword"
            },
            "nodeType": {
                "type": "keyword"
            },
            "costTime": {
                "type": "integer"
            },
            "appId": {
                "type": "keyword"
            },
            "context": {
                "norms": false,
                "type": "text",
                "analyzer": "my_custom_analyzer"
            },
            "startTime": {
                "type": "long"
            },
            "zoneCode": {
                "type": "keyword"
            },
            "serviceMethod": {
                "norms": false,
                "type": "text"
            },
            "remoteAddr": {
                "type": "keyword"
            },
            "status": {
                "type": "keyword"
            }
        }
    }
}
```



本文以es 7.3.2 版本作为范例

说明： xpack.security.enabled为true 时，需要同时开启节点间的transport通信为ssl 

在es用户下进入elasticsearch-7.3.2目录

1.为集群创建认证机构

>文件根目录下执行 bin/elasticsearch-certutil ca 
>依次输入回车（文件使用默认名），密码


2.为节点颁发证书

>之后执行bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12 
>依次输入上一个步骤的密码。回车（文件使用默认名），密码（建议与上一步密码相同）

> 执行bin/elasticsearch-keystore add 
> xpack.security.transport.ssl.keystore.secure_password 并输入第一步输入的密码 

> 执行bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password 并输入第一步输入的密码 


3.多节点配置

> 将生成的elastic-certificates.p12、elastic-stack-ca.p12文件mv到config目录下，并连同elasticsearch.keystore 文件 scp到其他节点的config目录中。

> 比如使用scp命令：
> scp elastic-certificates.p12 elasticsearch.keystore elastic-stack-ca.p12 root@192.168.1.100:/home/wes/elasticsearch-7.3.2/config/ 


4.修改配置
> 在elasticsearch-7.3.2/config/elasticsearch.yml中增加一下配置，启用x-pack安全组件，启用ssl加密通信，并且配置认证证书：

```
#---------------------security------------------

#

xpack.security.enabled: true

xpack.security.transport.ssl.enabled: true

xpack.security.transport.ssl.verification_mode: certificate

xpack.security.transport.ssl.keystore.path: elastic-certificates.p12

xpack.security.transport.ssl.truststore.path: elastic-certificates.p12

#
```


配置修改完成后，重启es服务，重启成功后 

http://192.168.1.100:9200/  访问Es服务要输入用户名和密码

5.密码设置

通过设置访问密码，这是elastic用户和其他一些系统内置用户的密码

bin/elasticsearch-setup-passwords interactive


设置成功后，可以通过用户名密码访问es服务：

```
curl -X GET -u 'elastic:elastic' http://192.168.118.51:900/ 访问Es服务，输入配置的用户名和密码

{

  "name" : "wnode-1",

  "cluster_name" : "escluster",

  "cluster_uuid" : "8EM0oU7aRjiJ-Gi8CH6Xtw",

  "version" : {

    "number" : "7.3.2",

    "build_flavor" : "default",

    "build_type" : "tar",

    "build_hash" : "aa751e09be0a5072e8570670309b1f12348f023b",

    "build_date" : "2020-02-29T00:15:25.529771Z",

    "build_snapshot" : false,

    "lucene_version" : "8.4.0",

    "minimum_wire_compatibility_version" : "6.8.0",

    "minimum_index_compatibility_version" : "6.0.0-beta1"

  },

  "tagline" : "You Know, for Search"

}
```

以上步骤操作成功后，es就开启了http auth鉴权的功能



其他： 

6.配置kibana访问密码

> 不要在kibana.yml配置文件里面配置es访问的用户密码明文，需要通过keystore配置加密的用户名密码信息，具体如下：
```
kibana-7.6.1-linux-x86_64/bin

创建keystore

./kibana-keystore create

设置kibana访问es的用户名

./kibana-keystore add elasticsearch.username

设置kibana访问es的密码

./kibana-keystore add elasticsearch.password
```


注意：

    在执行过程中，可能会提示 jdk版本不正确，可以忽略改报错 





操作报错问题排查：
1、出现如下错误，请检查集群是否添加remote白名单。可以在节点操作界面修改配置，并重启节点生效。

2、出现同步setting失败错误，如下图所示。请检查目标索引是否已经关闭。

3、如果数据源索引setting有设置tokenizer的min_gram与max_gram，同步setting时根据ES版本不同可能会出现如下错误。解决办法是，若目标索引不存在，先手动创建目标索引，然后修改目标索引setting的index.max_ngram_diff为需要的数值，然后再尝试同步。

4、如果数据源索引setting设置了index.blocks.read_only_allow_delete属性为true，那么同步完setting后，目标索引也会延续这一设置。若继续同步mapping，则会失败，如下图所示。若不同步mapping且reindex完成后，需要重新打开索引，index.blocks.read_only_allow_delete属性为true会使打开索引失败，需要设置目标索引index.blocks.read_only_allow_delete属性为false。
