# ElasticSearch属性文件

config/elasticsearch.yml配置文件

```yml
cluster.name: ES-Cluster
#ES集群名称，同一个集群内的所有节点集群名称必须保持一致

node.name: ES-master-192.168.198.241
#ES集群内的节点名称，同一个集群内的节点名称要具备唯一性

node.master: true
#允许节点是否可以成为一个master节点，ES是默认集群中的第一台机器成为master，如果这台机器停止就会重新选举

node.data: false
#允许该节点存储索引数据（默认开启）
#关于Elasticsearch节点的角色功能详解，请看：https://www.dockerc.com/elasticsearch-master-or-data/

path.data: /usr/share/elasticsearch/data1,/usr/share/elasticsearch/data2,/usr/share/elasticsearch/data3
#ES是搜索引擎，会创建文档，建立索引，此路径是索引的存放目录，如果我们的日志数据较为庞大，那么索引所占用的磁盘空间也是不可小觑的
#这个路径建议是专门的存储系统，如果不是存储系统，最好也要有冗余能力的磁盘，此目录还要对elasticsearch的运行用户有写入权限
#path可以指定多个存储位置，分散存储，有助于性能提升，以至于怎么分散存储请看详解https://www.dockerc.com/elk-theory-elasticsearch/

path.logs: /usr/share/elasticsearch/logs
#elasticsearch专门的日志存储位置，生产环境中建议elasticsearch配置文件与elasticsearch日志分开存储

bootstrap.memory_lock: true
#在ES运行起来后锁定ES所能使用的堆内存大小，锁定内存大小一般为可用内存的一半左右；锁定内存后就不会使用交换分区
#如果不打开此项，当系统物理内存空间不足，ES将使用交换分区，ES如果使用交换分区，那么ES的性能将会变得很差

network.host: 0.0.0.0
#网络绑定，0.0.0.0,支持外网访问
#network.host设置为内网127.0.0.1, 即只能本机访问
#network.host 兼有publish_host和bind_host两者的功能。
#network.publish_host：是elasticsearch与其他集群机器通信的地址
#network.bind_host：是设置控制Elasticsearch侦听的网络接口的地址。

network.tcp.no_delay: true
#是否启用tcp无延迟，true为启用tcp不延迟，默认为false启用tcp延迟

network.tcp.keep_alive: true
#是否启用TCP保持活动状态，默认为true

network.tcp.reuse_address: true
#是否应该重复使用地址。默认true，在Windows机器上默认为false

network.tcp.send_buffer_size: 128mb
#tcp发送缓冲区大小，默认不设置

network.tcp.receive_buffer_size: 128mb
#tcp接收缓冲区大小，默认不设置

transport.tcp.port: 9301
#设置集群节点通信的TCP端口，默认就是9300

transport.tcp.compress: true
#设置是否压缩TCP传输时的数据，默认为false

http.max_content_length: 200mb
#设置http请求内容的最大容量，默认是100mb

http.cors.enabled: true
#是否开启跨域访问

http.cors.allow-origin: “*”
#开启跨域访问后的地址限制，*表示无限制

http.port: 9201
#定义ES对外调用的http端口，默认是9200

discovery.zen.ping.unicast.hosts: [“192.168.198.241:9301”, “10.150.55.95:9301”,“10.150.30.246:9301”] #在Elasticsearch7.0版本已被移除，配置错误
#写入候选主节点的设备地址，来开启服务时就可以被选为主节点
#默认主机列表只有127.0.0.1和IPV6的本机回环地址
#上面是书写格式，discover意思为发现，zen是判定集群成员的协议，unicast是单播的意思，ES5.0版本之后只支持单播的方式来进行集群间的通信，hosts为主机
#总结下来就是：使用zen协议通过单播方式去发现集群成员主机，在此建议将所有成员的节点名称都写进来，这样就不用仅靠集群名称cluster.name来判别集群关系了

discovery.zen.minimum_master_nodes: 2 #在Elasticsearch7.0版本已被移除，配置无效
#为了避免脑裂，集群的最少节点数量为，集群的总节点数量除以2加一

discovery.zen.fd.ping_timeout: 120s #在Elasticsearch7.0版本已被移除，配置无效
#探测超时时间，默认是3秒，我们这里填120秒是为了防止网络不好的时候ES集群发生脑裂现象

discovery.zen.fd.ping_retries: 6 #在Elasticsearch7.0版本已被移除，配置无效
#探测次数，如果每次探测90秒，连续探测超过六次，则认为节点该节点已脱离集群，默认为3次

discovery.zen.fd.ping_interval: 15s #在Elasticsearch7.0版本已被移除，配置无效
#节点每隔15秒向master发送一次心跳，证明自己和master还存活，默认为1秒太频繁,

discovery.seed_hosts: [“192.168.198.241:9301”, “10.150.55.95:9301”,“10.150.30.246:9301”]
#Elasticsearch7新增参数，写入候选主节点的设备地址，来开启服务时就可以被选为主节点,由discovery.zen.ping.unicast.hosts:参数改变而来

cluster.initial_master_nodes: [“192.168.198.241:9301”, “10.150.55.95:9301”,“10.150.30.246:9301”]
#Elasticsearch7新增参数，写入候选主节点的设备地址，来开启服务时就可以被选为主节点

cluster.fault_detection.leader_check.interval: 15s
#Elasticsearch7新增参数，设置每个节点在选中的主节点的检查之间等待的时间。默认为1秒

discovery.cluster_formation_warning_timeout: 30s
#Elasticsearch7新增参数，启动后30秒内，如果集群未形成，那么将会记录一条警告信息，警告信息未master not fount开始，默认为10秒

cluster.join.timeout: 30s
#Elasticsearch7新增参数，节点发送请求加入集群后，在认为请求失败后，再次发送请求的等待时间，默认为60秒

cluster.publish.timeout: 90s
#Elasticsearch7新增参数，设置主节点等待每个集群状态完全更新后发布到所有节点的时间，默认为30秒

cluster.routing.allocation.cluster_concurrent_rebalance: 32
#集群内同时启动的数据任务个数，默认是2个

cluster.routing.allocation.node_concurrent_recoveries: 32
#添加或删除节点及负载均衡时并发恢复的线程个数，默认4个

cluster.routing.allocation.node_initial_primaries_recoveries: 32
#初始化数据恢复时，并发恢复线程的个数，默认4个
```
