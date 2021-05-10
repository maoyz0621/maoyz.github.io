

`vi /etc/security/limits.conf`修改创建文件数65535个，线程数4096个

```
es soft nofile 65536
es hard nofile 65536
es soft nproc 4096
es hard nproc 4096
```

`vi /etc/sysctl.conf`开辟65536字节虚拟空间

```
# 内存权限大小
vm.max_map_count=262144
```

`sysctl -p`权限生效



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



启动错误：

```
[ERROR][o.e.b.ElasticsearchUncaughtExceptionHandler] [skywarking] uncaught exception in thread [main]
org.elasticsearch.bootstrap.StartupException: java.lang.RuntimeException: can not run elasticsearch as root
	at org.elasticsearch.bootstrap.Elasticsearch.init(Elasticsearch.java:174) ~[elasticsearch-7.9.3.jar:7.9.3]
	at org.elasticsearch.bootstrap.Elasticsearch.execute(Elasticsearch.java:161) ~[elasticsearch-7.9.3.jar:7.9.3]
	at org.elasticsearch.cli.EnvironmentAwareCommand.execute(EnvironmentAwareCommand.java:86) ~[elasticsearch-7.9.3.jar:7.9.3]
	at org.elasticsearch.cli.Command.mainWithoutErrorHandling(Command.java:127) ~[elasticsearch-cli-7.9.3.jar:7.9.3]
	at org.elasticsearch.cli.Command.main(Command.java:90) ~[elasticsearch-cli-7.9.3.jar:7.9.3]
	at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:126) ~[elasticsearch-7.9.3.jar:7.9.3]
	at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:92) ~[elasticsearch-7.9.3.jar:7.9.3]
Caused by: java.lang.RuntimeException: can not run elasticsearch as root
```

> 错误提示：ES不能以root用户启动



增加用户es：`useradd es`

用户es密码：`passwd  es`

赋予用户es目录权限：`chown -R es /usr/local/elasticsearch-7.9.3  `

切换用户：`su es`

```
[1]: Java version [1.8.0_222-ea] is an early-access build, only use release builds
[2]: the default discovery settings are unsuitable for production use; at least one of [discovery.seed_hosts, discovery.seed_providers, cluster.initial_master_nodes] must be configured
```







java.lang.IllegalStateException: failed to obtain node locks, tried [[/usr/local/elasticsearch-7.9.3/data]] with lock id [0]; maybe these locations are not writable or multiple nodes were started without increasing [node.max_local_storage_nodes] (was [1])?
	at org.elasticsearch.env.NodeEnvironment.<init>(NodeEnvironment.java:301)



ps aux | grep ‘elastic’ 



cluster.initial_master_nodes: ["node-1"]



`curl http://localhost:9200`查看ES信息

```
{
  "name" : "skywarking",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "BmYXnBW1QgaJY8u6cfgK2A",
  "version" : {
    "number" : "7.9.3",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "c4138e51121ef06a6404866cddc601906fe5c868",
    "build_date" : "2020-10-16T10:36:16.141335Z",
    "build_snapshot" : false,
    "lucene_version" : "8.6.2",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```



1. docker run elasticsearch7.x

```
docker  pull elasticsearch:7.9.3

# 运行，使用配置参数
docker run --name elasticsearch -p 9200:9200 -p 9300:9300 -e discovery.type=single-node -e ES_JAVA_OPTS="-Xms256m -Xmx256m" -e http.cors.enabled=true -e http.cors.allow-origin=* -e http.cors.allow-headers=X-Requested-With,X-Auth-Token,Content-Type,Content-Length,Authorization -e "http.cors.allow-credentials=true"  -d elasticsearch:7.9.3

# 使用挂载文件
docker run --name elasticsearch -p 9200:9200 -p 9300:9300  
-e "discovery.type=single-node" 
-e ES_JAVA_OPTS="-Xms128m -Xmx128m" 
-v /usr/local/elasticsearch-cluster/node1/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml 
-v /usr/local/elasticsearch-cluster/node1/data:/usr/share/elasticsearch/data 
-v /mydata/elasticsearch/plugins:/usr/share/elasticsearch/plugins -d elasticsearch:7.9.3



-e "http.cors.enabled=true" -e "http.cors.allow-origin=*" -e "http.cors.allow-headers=X-Requested-With,X-Auth-Token,Content-Type,Content-Length,Authorization" -e "http.cors.allow-credentials=true" docker.elastic.co/elasticsearch/elasticsearch-oss:7.0.1
```

> 添加允许跨域：
> http.cors.enabled: true
> http.cors.allow-origin: “*”





> 出现错误：ElasticsearchException[failed to bind service]; nested: AccessDeniedException[/usr/share/elasticsearch/data/nodes];

挂载文件权限问题

解决方法：

chmod -R 777 /usr/local/elasticsearch-cluster/node1/



2. elasticsearch-head     9100

```shell
# 拉取镜像
docker pull mobz/elasticsearch-head:5

# 运行镜像
docker run --name elasticsearch-head -di -p 9100:9100 docker.io/mobz/elasticsearch-head:5
```

页面登录地址：http://192.168.107.110:9100/

![](.\开源软件\image\elasticsearch-head.jpg)

数据浏览时，无数据展示：

{
“error” : “Content-Type header [application/x-www-form-urlencoded] is not supported”,
“status” : 406
}

解决办法：https://shanhy.blog.csdn.net/article/details/103737698?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromBaidu-1.control&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromBaidu-1.control

1. 把配置文件从容器里面拷贝到宿主机目录（elasticsearch-head是容器名，也可以用容器ID）：

   **docker cp elasticsearch-head:/usr/src/app/_site/vendor.js ./**

2. 将改完后的文件拷贝回容器：

      **docker cp vendor.js elasticsearch-head:/usr/src/app/_site**
      
3. 修改内容：

将 6886行 `contentType: "application/x-www-form-urlencoded"` 修改为 `contentType: "application/json;charset=UTF-8"`
然后再将 7574行 `var inspectData = s.contentType === "application/x-www-form-urlencoded" &&` 修改为 `var inspectData = s.contentType === "application/json;charset=UTF-8" &&`

3. dejavu  https://github.com/appbaseio/dejavu/

```
docker pull appbaseio/dejavu:3.3.0

docker run --name dejavu3 -p 1358:1358 -d appbaseio/dejavu:3.3.0
```



4. 集群

+ 6.x版本：

```
# node01的配置：
cluster.name: es-cluster  # 集群名称
node.name: node01	# 节点的名称
node.master: true
node.data: true
network.host: 0.0.0.0
http.port: 9200
transport.port: 9300  # REST客户端通过HTTP请求
discovery.zen.ping.unicast.hosts:
["192.168.40.133","192.168.40.134","192.168.40.135"]  # 6.x版本
# 最小节点数
discovery.zen.minimum_master_nodes: 2   # 6.x版本

discovery.seed_hosts: ["127.0.0.1:9300","127.0.0.1:9301","127.0.0.1:9302"]	# 配置集群的主机地址，配置之后集群的主机之间可以自动发现
cluster.initial_master_nodes: ["node-1"]  # 初始的候选master节点列表。初始主节点应通过其node.name标识，默认为其主机名。确保完全匹配。
# 跨域专用
http.cors.enabled: true
http.cors.allow-origin: "*"


#node02的配置：
cluster.name: es-cluster
node.name: node02
node.master: true
node.data: true
network.host: 0.0.0.0
http.port: 9200
transport.port: 9300
discovery.zen.ping.unicast.hosts:
["192.168.40.133","192.168.40.134","192.168.40.135"]
discovery.zen.minimum_master_nodes: 2
http.cors.enabled: true
http.cors.allow-origin: "*"


#node03的配置：
cluster.name: es-cluster
node.name: node03
node.master: true
node.data: true
network.host: 0.0.0.0
http.port: 9200
transport.port: 9300
discovery.zen.ping.unicast.hosts:
["192.168.40.133","192.168.40.134","192.168.40.135"]
discovery.zen.minimum_master_nodes: 2
http.cors.enabled: true
http.cors.allow-origin: "*"
```

> discovery.zen.minimum_master_nodes:不是N/2+1时，会出现脑裂问题，之前宕机的主节点恢复后不会加入到集群

+ 7.x版本

单台机器使用docker搭建集群：

```yaml
# node01的配置：
cluster.name: elastic-cluster
node.name: node01
node.master: true
node.data: true
node.max_local_storage_nodes: 3
network.host: 0.0.0.0
http.port: 9201
transport.tcp.port: 9301
discovery.seed_hosts: ["192.168.107.110:9301","192.168.107.110:9302","192.168.107.110:9303"]
cluster.initial_master_nodes: ["node01","node02","node03"]
discovery.zen.ping_timeout: 120s
client.transport.ping_timeout: 60s
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-methods: OPTIONS, HEAD, GET, POST, PUT, DELETE
http.cors.allow-headers: "X-Requested-With, Content-Type, Content-Length, X-User"

#node02的配置：
cluster.name: elastic-cluster
node.name: node02
node.master: true
node.data: true
node.max_local_storage_nodes: 3
network.host: 0.0.0.0
http.port: 9202
transport.tcp.port: 9302
discovery.seed_hosts: ["192.168.107.110:9301","192.168.107.110:9302","192.168.107.110:9303"]
cluster.initial_master_nodes: ["node01","node02","node03"]
discovery.zen.ping_timeout: 120s
client.transport.ping_timeout: 60s
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-methods: OPTIONS, HEAD, GET, POST, PUT, DELETE
http.cors.allow-headers: "X-Requested-With, Content-Type, Content-Length, X-User"


#node03的配置：
cluster.name: elastic-cluster
node.name: node03
node.master: true
node.data: true
node.max_local_storage_nodes: 3
network.host: 0.0.0.0
http.port: 9203
transport.tcp.port: 9303
discovery.seed_hosts: ["192.168.107.110:9301","192.168.107.110:9302","192.168.107.110:9303"]
cluster.initial_master_nodes: ["node01","node02","node03"]
discovery.zen.ping_timeout: 120s
client.transport.ping_timeout: 60s
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-methods: OPTIONS, HEAD, GET, POST, PUT, DELETE
http.cors.allow-headers: "X-Requested-With, Content-Type, Content-Length, X-User"
```

docker运行命令：

```
docker run --name elasticsearch-node1 -p 9201:9201 -p 9301:9301 -e ES_JAVA_OPTS="-Xms128m -Xmx128m" -v /usr/local/elasticsearch-cluster/node1/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml -v /usr/local/elasticsearch-cluster/node1/data:/usr/share/elasticsearch/data  -v /usr/local/elasticsearch-cluster/node1/logs:/usr/share/elasticsearch/logs -d elasticsearch:7.9.3

docker run --name elasticsearch-node2 -p 9202:9202 -p 9302:9302 -e ES_JAVA_OPTS="-Xms128m -Xmx128m" -v /usr/local/elasticsearch-cluster/node2/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml -v /usr/local/elasticsearch-cluster/node2/data:/usr/share/elasticsearch/data  -v /usr/local/elasticsearch-cluster/node2/logs:/usr/share/elasticsearch/logs -d elasticsearch:7.9.3

docker run --name elasticsearch-node3 -p 9203:9203 -p 9303:9303 -e ES_JAVA_OPTS="-Xms128m -Xmx128m" -v /usr/local/elasticsearch-cluster/node3/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml -v /usr/local/elasticsearch-cluster/node3/data:/usr/share/elasticsearch/data  -v /usr/local/elasticsearch-cluster/node3/logs:/usr/share/elasticsearch/logs -d elasticsearch:7.9.3
```



三台物理机搭建集群：

```yaml
# node01的配置：
cluster.name: elastic-cluster
node.name: node01
node.master: true
node.data: true
node.max_local_storage_nodes: 3
network.host: 192.168.107.110
http.port: 9201
transport.tcp.port: 9301
discovery.seed_hosts: ["192.168.107.110:9301","192.168.107.110:9302","192.168.107.111:9301","192.168.107.111:9300"]
cluster.initial_master_nodes: ["node01","node02","node03"]
discovery.zen.ping_timeout: 120s
client.transport.ping_timeout: 60s
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-methods: OPTIONS, HEAD, GET, POST, PUT, DELETE
http.cors.allow-headers: "X-Requested-With, Content-Type, Content-Length, X-User"


#node02的配置：
cluster.name: elastic-cluster
node.name: node02
node.master: true
node.data: true
node.max_local_storage_nodes: 3
network.host: 192.168.107.110
http.port: 9202
transport.tcp.port: 9302
discovery.seed_hosts: ["192.168.107.110:9301","192.168.107.111:9301","192.168.107.110:9302","192.168.107.111:9300"]
cluster.initial_master_nodes: ["node01","node02","node03"]
discovery.zen.ping_timeout: 120s
client.transport.ping_timeout: 60s
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-methods: OPTIONS, HEAD, GET, POST, PUT, DELETE
http.cors.allow-headers: "X-Requested-With, Content-Type, Content-Length, X-User"s


#node03的配置：
cluster.name: elastic-cluster
node.name: node03
node.master: true
node.data: true
node.max_local_storage_nodes: 3
network.host: 192.168.107.111
http.port: 9200
transport.tcp.port: 9300
discovery.seed_hosts: ["192.168.107.110:9301","192.168.107.111:9300","192.168.107.110:9302"]
cluster.initial_master_nodes: ["node01","node02","node03"]
discovery.zen.ping_timeout: 120s
client.transport.ping_timeout: 60s
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-methods: OPTIONS, HEAD, GET, POST, PUT, DELETE
http.cors.allow-headers: "X-Requested-With, Content-Type, Content-Length, X-User"
```

9200端口和9300端口

9200/tcp, 0.0.0.0:9201->9201/tcp, 9300/tcp, 0.0.0.0:9301->9301/tcp

三台物理机使用docker-compose搭建集群：docker-compose.yml

```yaml
version: '3'
services:
  elasticsearch-1:                    # 服务名称
    image: elasticsearch:7.9.3     # 使用的镜像
    container_name: es-docker-compose-1   # 容器名称
    # restart: always                 # 失败自动重启策略
    healthcheck:
      test: ["CMD-SHELL", "curl --silent --fail 192.168.107.111:9201/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    environment:
      - node.name=node-1                   # 节点名称，集群模式下每个节点名称唯一
      - network.publish_host=192.168.107.111  # 用于集群内各机器间通信,对外使用，其他机器访问本机器的es服务，一般为本机宿主机IP
      - network.host=0.0.0.0                # 设置绑定的ip地址，可以是ipv4或ipv6的，默认为0.0.0.0，即本机
      - http.port=9201
      - transport.tcp.port=9301
      - discovery.seed_hosts=192.168.107.110:9301,192.168.107.110:9303,192.168.107.111:9301,192.168.107.111:9302
      - cluster.initial_master_nodes=node-1,node-2,node-3
      - cluster.name=elastic-docker-cluster
      - node.master=true
      - node.data=true
      - http.cors.enabled=true
      - http.cors.allow-origin=*
      - http.cors.allow-headers=X-Requested-With,X-Auth-Token,Content-Type,Content-Length,Authorization
      - http.cors.allow-credentials=true
      - bootstrap.memory_lock=true  # 内存交换的选项，官网建议为true
      - "ES_JAVA_OPTS=-Xms128m -Xmx128m" # 设置内存，如内存不足，可以尝试调低点
    ulimits:        # 栈内存的上限
      memlock:
        soft: -1    # 不限制
        hard: -1    # 不限制
    volumes:
      # - /usr/local/elasticsearch-cluster/docker/node1-compose/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml  # 将容器中es的配置文件映射到本地，设>置跨域， 否则head>插件无法连接该节点
      - /usr/local/elasticsearch-cluster/docker/node1-compose/data:/usr/share/elasticsearch/data  # 存放数据的文件， 注意：这里的esdata为 顶级volumes下的一项。
      - /usr/local/elasticsearch-cluster/docker/node1-compose/logs:/usr/share/elasticsearch/logs
    ports:
      - 9201:9201    # http端口，可以直接浏览器访问
      - 9301:9301    # es集群之间相互访问的端口，jar之间就是通过此端口进行tcp协议通信，遵循tcp协议。
    networks:
      - elastic
      
  elasticsearch-2:                    # 服务名称
    image: elasticsearch:7.9.3     # 使用的镜像
    container_name: es-docker-compose-2   # 容器名称
    # restart: always                 # 失败自动重启策略
    environment:
      - node.name=node-2                   # 节点名称，集群模式下每个节点名称唯一
      - network.publish_host=192.168.107.111  # 用于集群内各机器间通信,对外使用，其他机器访问本机器的es服务，一般为本机宿主机IP
      - network.host=0.0.0.0                # 设置绑定的ip地址，可以是ipv4或ipv6的，默认为0.0.0.0，即本机
      - http.port=9202
      - transport.tcp.port=9302
      - discovery.seed_hosts=192.168.107.110:9301,192.168.107.110:9303,192.168.107.111:9301,192.168.107.111:9302
      - cluster.initial_master_nodes=node-1,node-2,node-3
      - cluster.name=elastic-docker-cluster
      - node.master=true
      - node.data=true
      - http.cors.enabled=true
      - http.cors.allow-origin="*"
      - http.cors.allow-headers=X-Requested-With,X-Auth-Token,Content-Type,Content-Length,Authorization
      - http.cors.allow-credentials=true
      - bootstrap.memory_lock=true  # 内存交换的选项，官网建议为true
      - "ES_JAVA_OPTS=-Xms128m -Xmx128m" # 设置内存，如内存不足，可以尝试调低点
    ulimits:        # 栈内存的上限
      memlock:
        soft: -1    # 不限制
        hard: -1    # 不限制
    volumes:      # - /usr/local/elasticsearch-cluster/docker/node1-compose/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml  # 将容器中es的配置文件映>射到本地，设置跨域， 否则head>插件无法连接该节点
      - /usr/local/elasticsearch-cluster/docker/node2-compose/data:/usr/share/elasticsearch/data  # 存放数据的文件， 注意：这里的esdata为 顶级volumes下的一项。
      - /usr/local/elasticsearch-cluster/docker/node2-compose/logs:/usr/share/elasticsearch/logs
    ports:
      - 9202:9202    # http端口，可以直接浏览器访问
      - 9302:9302    # es集群之间相互访问的端口，jar之间就是通过此端口进行tcp协议通信，遵循tcp协议。
    networks:
      - elastic
      
volumes:
  esdata:
    driver: local    # 会生成一个对应的目录和文件，如何查看，下面有说明。
    
networks:
  elastic:
    driver: bridge  # 桥接模式
```

查看集群健康：/_cluster/health?pretty

# 中文分词配置





### Kibana



1. ### 创建挂载的文件夹和文件 

```
mkdir -p /opt/elk7/kibana
cd /opt/elk7/kibana
#创建对应的文件夹 数据 / 日志 / 配置
mkdir conf data logs 
#授权
chmod 777 -R conf data logs
cd logs
#创建日志文件
touch kibana.log
#授权
chmod 777 kibana.log 
```



### 主配置文件

```
#节点地址和端口 必须是同一个集群的 必须以http或者https开头 填写实际的es地址和端口
elasticsearch.hosts: ['http://172.16.10.202:9200','http://172.16.10.202:9202', 'http://172.16.10.202:9203']
#发给es的查询记录 需要日志等级是verbose=true 
elasticsearch.logQueries: true
#连接es的超时时间 单位毫秒
elasticsearch.pingTimeout: 30000
elasticsearch.requestTimeout: 30000
#是否只能使用server.host访问服务
elasticsearch.preserveHost: true
#首页对应的appid
kibana.defaultAppId: "home"
kibana.index: '.kibana'
#存储日志的文件设置
logging.dest: /usr/share/kibana/logs/kibana.log
logging.json: true
#是否只输出错误日志信息
logging.quiet: false
logging.rotate:
  enabled: true
  #日志文件最大大小
  everyBytes: 10485760
  #保留的日志文件个数
  keepFiles: 7
logging.timezone: UTC
logging.verbose: true
monitoring.kibana.collection.enabled: true
xpack.monitoring.collection.enabled: true
#存储持久化数据的位置
path.data: /usr/share/kibana/data
#访问kibana的地址和端口配置 一般使用可访问的服务器地址即可
server.host: 172.16.10.202
#端口默认5601
server.port: 5601
server.name: "kibana"
#配置页面语言
i18n.locale: zh-CN
```





```
server.name: kibana
server.host: "0"
elasticsearch.hosts: [ "http://192.168.107.110:9200" ]
monitoring.ui.container.elasticsearch.enabled: true
```





授权

```
chmod 777 kibana.yml
```





### 创建容器并运行

```
# 拉取镜像
docker pull docker.elastic.co/kibana/kibana:7.8.1

# 后台创建并运行容器
docker run -it --name kibana -p 5601:5601 
-v /usr/local/kibana/logs/kibana.log:/usr/share/kibana/logs/kibana.log 
-v /usr/local/kibana/data:/usr/share/kibana/data 
-v /usr/local/kibana/conf/kibana.1.yml:/usr/share/kibana/config/kibana.yml 
-d kibana:7.9.3

docker run -it --name kibana -p 5601:5601 
-e elasticsearch.hosts=["http://192.168.107.110:9200"]
-v /usr/local/kibana/logs/kibana.log:/usr/share/kibana/logs/kibana.log 
-v /usr/local/kibana/data:/usr/share/kibana/data 
-d kibana:7.9.3
```





错误：

[BABEL] Note: The code generator has deoptimised the styling of /usr/share/kibana/x-pack/plugins/canvas/server/templates/pitch_presentation.js as it exceeds the max of 500KB.

 FATAL  Error: Unable to write Kibana UUID file, please check the uuid.server configuration value in kibana.yml and ensure Kibana has sufficient（足够的） permissions to read / write to this file. Error was: ENOENT



解决方案：







https://www.cnblogs.com/michael-xiang/p/13715692.html