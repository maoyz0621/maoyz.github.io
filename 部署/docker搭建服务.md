# docker安装组件

## docker搭建redis

```sh
docker run --rm --name redis -p 6379:6379 -v /opt/docker/redis/redis-one/redis.conf:/etc/redis/redis.conf -v /opt/docker/redis/redis-one/data:/data -d redis:4 redis-server /etc/redis/redis.conf --appendonly yes
```

`--name myredis` 对Redis容器进行命名

`-p 6379:6379` 将Redis容器中的6379端口映射到宿主机的6379端口

`-v /share/redis/data:/data` 将宿主机共享目录中的redis的data目录挂载到Redis容器中的data目录

`-d redis` 以后台守护进程方式运行Redis

`redis-server`

`--appendonly yes` 在Redis容器启动redis-server服务器并打开Redis持久化配置

查看docker中redis启动情况

```sh
docker ps -a|grep redis
```

使用Redis客户端连接Redis服务器

```sh
docker exec -it redis redis-cli
```

查看redis映像地址

```sh
docker inspect redis | grep -i add
```

```sh
 "CapAdd": null,
            "GroupAdd": null,
            "LinkLocalIPv6Address": "",
            "SecondaryIPAddresses": null,
            "SecondaryIPv6Addresses": null,
            "GlobalIPv6Address": "",
            "IPAddress": "172.17.0.2",
            "MacAddress": "02:42:ac:11:00:02",
                    "IPAddress": "172.17.0.2",
                    "GlobalIPv6Address": "",
                    "MacAddress": "02:42:ac:11:00:02",

```

修改bind  172.17.0.2



## docker搭建redis集群

下载ruby

```sh
docker pull ruby
```

创建虚拟网卡

```sh
docker network create redis-net
```

查看虚拟网卡

```sh
docker network ls
docker network inspect redis-net | grep "Gateway" | grep --color=auto -P '(\d{1,3}.){3}\d{1,3}' -o
```

172.18.0.1

配置redis.conf

```
port 7005
## cluster集群模式
cluster-enabled yes
##集群配置名
cluster-config-file nodes.conf  
cluster-node-timeout 5000
##实际为各节点网卡分配ip  先用上网关ip代替
cluster-announce-ip 172.18.0.7
##节点映射端口
cluster-announce-port 7005
##节点总线端
cluster-announce-bus-port 17005
appendonly yes
# 切记不要加，坑半天
# daemonize yes
protected-mode no
```

启动redis

```sh
docker run -p 7000:7000 -p 17000:17000 --restart always --name redis-7000 --net redis-net --privileged=true -v /opt/docker/redis/redis-cluster/7000/redis-7000.conf:/etc/redis/redis.conf -v /opt/docker/redis/redis-cluster/7000/data:/data -d redis:4 redis-server /etc/redis/redis.conf

docker run -p 7001:7001 -p 17001:17001 --restart always --name redis-7001 --net redis-net --privileged=true -v /opt/docker/redis/redis-cluster/7001/redis-7001.conf:/etc/redis/redis.conf -v /opt/docker/redis/redis-cluster/7001/data:/data -d redis:4 redis-server /etc/redis/redis.conf

docker run -p 7002:7002 -p 17002:17002 --restart always --name redis-7002 --net redis-net --privileged=true -v /opt/docker/redis/redis-cluster/7002/redis-7002.conf:/etc/redis/redis.conf -v /opt/docker/redis/redis-cluster/7002/data:/data -d redis:4 redis-server /etc/redis/redis.conf

docker run -p 7003:7003 -p 17003:17003 --restart always --name redis-7003 --net redis-net --privileged=true -v /opt/docker/redis/redis-cluster/7003/redis-7003.conf:/etc/redis/redis.conf -v /opt/docker/redis/redis-cluster/7003/data:/data -d redis:4 redis-server /etc/redis/redis.conf

docker run -p 7004:7004 -p 17004:17004 --restart always --name redis-7004 --net redis-net --privileged=true -v /opt/docker/redis/redis-cluster/7004/redis-7004.conf:/etc/redis/redis.conf -v /opt/docker/redis/redis-cluster/7004/data:/data -d redis:4 redis-server /etc/redis/redis.conf

docker run -p 7005:7005 -p 17005:17005 --restart always --name redis-7005 --net redis-net --privileged=true -v /opt/docker/redis/redis-cluster/7005/redis-7005.conf:/etc/redis/redis.conf -v /opt/docker/redis/redis-cluster/7005/data:/data -d redis:4 redis-server /etc/redis/redis.conf
```

查看容器分配ip

```sh
docker network inspect redis-net
```

```
[
    {
        "Name": "redis-net",
        "Id": "542c0be25e92ae8417a05a803cb7c8e1df4d625eff175da17bf632812d219866",
        "Created": "2020-03-14T14:56:59.133955862+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "140fa2ab2392a3240ee7f99f2ca0f4e6fbff89414ae34a953eb25bc71b8bcf4e": {
                "Name": "redis-7003",
                "EndpointID": "6177d287d4278085bbbada1dc961a682d59ad4e75f3cc8ac6cc16779baeebb10",
                "MacAddress": "02:42:ac:12:00:05",
                "IPv4Address": "172.18.0.5/16",
                "IPv6Address": ""
            },
            "6a33c35852de8c22b766ecf2e0fa9dc279551f3976cb25290466a2e9b79bb907": {
                "Name": "redis-7004",
                "EndpointID": "aa6452a63d1ed9ed7004dd8b7fcee8e576fdb4efdc0e4e78bfec2c0e131a961d",
                "MacAddress": "02:42:ac:12:00:06",
                "IPv4Address": "172.18.0.6/16",
                "IPv6Address": ""
            },
            "96b9a0882e575cd481a11ca1a2e6ac15f0c813f0360c15f40d5b5d80ed602790": {
                "Name": "redis-7001",
                "EndpointID": "557fc4fe364b25c984e3bec8a21a608e09ed9d14fd8a09ae75fc1f90b81d33af",
                "MacAddress": "02:42:ac:12:00:03",
                "IPv4Address": "172.18.0.3/16",
                "IPv6Address": ""
            },
            "e76896ae63aeb0efe48ab34afa5a1b40aeecaa038d08f3266e884e79f7c71a99": {
                "Name": "redis-7002",
                "EndpointID": "018127c25ed55fb8ed09eb23a4869d6e50c27dba793adb5fcac7bae6770006cc",
                "MacAddress": "02:42:ac:12:00:04",
                "IPv4Address": "172.18.0.4/16",
                "IPv6Address": ""
            },
            "f29d0bb3029ee7cedd6863d727faba354c25f6804a22f4d8ad773b3bd98bf3d2": {
                "Name": "redis-7005",
                "EndpointID": "d4e94ae6c54bd0b7fed301481621d5bb145f926475c9b919e8e5799f250dde0f",
                "MacAddress": "02:42:ac:12:00:07",
                "IPv4Address": "172.18.0.7/16",
                "IPv6Address": ""
            },
            "f55a91d85e6bdc71f94a85b994d1d05deb4d6892b2b9feb2c43c7aadd1d81447": {
                "Name": "redis-7000",
                "EndpointID": "14190cad95f427fb3155a399d44303a36b367a3fa5be86196c5af8368a8770d0",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```



```sh
FROM ruby:latest 
MAINTAINER sunlin<sunlin1111@163.com>
RUN gem install redis -v 4.0.14
RUN mkdir /redis
WORKDIR /redis
ADD ./redis-trib.rb /redis/redis-trib.rb 
```

构建

```
docker build -t redis-trib .
```

运行ruby

```sh
echo yes | docker run -i --rm --net redis-net redis-trib ruby redis-trib.rb create --replicas 1 172.18.0.2:7000 172.18.0.3:7001 172.18.0.4:7002 172.18.0.5:7003 172.18.0.6:7004 172.18.0.7:7005
```

```sh
>>> Creating cluster
>>> Performing hash slots allocation on 6 nodes...
Using 3 masters:
172.18.0.2:7000
172.18.0.3:7001
172.18.0.4:7002
Adding replica 172.18.0.6:7004 to 172.18.0.2:7000
Adding replica 172.18.0.7:7005 to 172.18.0.3:7001
Adding replica 172.18.0.5:7003 to 172.18.0.4:7002
M: 5e8512a4606e9f297bef7b533260170ecee25aa2 172.18.0.2:7000
   slots:0-5460 (5461 slots) master
M: 9360ebf48f1359b1008ddb4f4a22b7d55da44eca 172.18.0.3:7001
   slots:5461-10922 (5462 slots) master
M: 53497b9d7a7c8ecbde46d16ac803cd7ec53bfd17 172.18.0.4:7002
   slots:10923-16383 (5461 slots) master
S: 4658f71d08a19980622ff7eee3fdcecd4ff89dc4 172.18.0.5:7003
   replicates 53497b9d7a7c8ecbde46d16ac803cd7ec53bfd17
S: 33c4ca1aaaccf8536d3bf3ba83c3e2da8a402e93 172.18.0.6:7004
   replicates 5e8512a4606e9f297bef7b533260170ecee25aa2
S: db9c4f9128529028823bc2b3e7bfc990ee32e504 172.18.0.7:7005
   replicates 9360ebf48f1359b1008ddb4f4a22b7d55da44eca
Can I set the above configuration? (type 'yes' to accept): >>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join.....
>>> Performing Cluster Check (using node 172.18.0.2:7000)
M: 5e8512a4606e9f297bef7b533260170ecee25aa2 172.18.0.2:7000
   slots:0-5460 (5461 slots) master
   1 additional replica(s)
S: 33c4ca1aaaccf8536d3bf3ba83c3e2da8a402e93 172.18.0.6:7004
   slots: (0 slots) slave
   replicates 5e8512a4606e9f297bef7b533260170ecee25aa2
M: 9360ebf48f1359b1008ddb4f4a22b7d55da44eca 172.18.0.3:7001
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
S: db9c4f9128529028823bc2b3e7bfc990ee32e504 172.18.0.7:7005
   slots: (0 slots) slave
   replicates 9360ebf48f1359b1008ddb4f4a22b7d55da44eca
S: 4658f71d08a19980622ff7eee3fdcecd4ff89dc4 172.18.0.5:7003
   slots: (0 slots) slave
   replicates 53497b9d7a7c8ecbde46d16ac803cd7ec53bfd17
M: 53497b9d7a7c8ecbde46d16ac803cd7ec53bfd17 172.18.0.4:7002
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

客户端连接地址：

```sh
172.18.0.2  6379
172.18.0.3  6379
172.18.0.4  6379
172.18.0.5  6379
172.18.0.6  6379
172.18.0.7  6379
```



## docker搭建redis哨兵



docker 搭建zookeeper

```sh
docker run --name zk -p 2181:2181 -p 2888:2888 -p 3888:3888 --restart always -d zookeeper:latest
```

-p　　端口映射

　　--name　　容器实例名称

　　-d　　后台运行

　　2181　　Zookeeper客户端交互端口

　　2888　　Zookeeper集群端口

　　3888　　Zookeeper选举端口



docker搭建zookeeper集群



```sh
docker network create zookeeper-net
docker network inspect zookeeper-net 
```



```
[
    {
        "Name": "zookeeper-net",
        "Id": "663a076a71741f5091ac973a83104ac5fcf37bd0c90a1141bfb36ef70ae5e48d",
        "Created": "2020-03-14T22:28:18.345463599+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.19.0.0/16",
                    "Gateway": "172.19.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```



```sh
docker run --name zk-2181 -p 2181:21811 -v /opt/docker/zk/zk-cluster/zookeeper-1/conf/:/conf -v /opt/docker/zk/zk-cluster/zookeeper-1/data/:/data --restart always -d zookeeper

docker run --name zk-2182 -p 2182:21812 -v /opt/docker/zk/zk-cluster/zookeeper-2/conf/:/conf -v /opt/docker/zk/zk-cluster/zookeeper-2/data/:/data --restart always -d zookeeper

docker run --name zk-2183 -p 2183:21813 -v /opt/docker/zk/zk-cluster/zookeeper-3/conf/:/conf -v /opt/docker/zk/zk-cluster/zookeeper-3/data/:/data --restart always -d zookeeper
```

配置文件zoo.cnf

```sh
clientPort=2181
server.1=zk-2181:2888:3888
server.2=zk-2182:2888:3888
server.3=zk-2183:2888:3888
```



docker 搭建rocketmq

nameserve服务

```sh
docker run -d -p 9876:9876 -v `pwd`/data/namesrv/logs:/root/logs -v `pwd`/data/namesrv/store:/root/store --name mqnamesrv -e "MAX_POSSIBLE_HEAP=100000000" rocketmqinc/rocketmq sh mqnamesrv
```

broker服务

```sh
docker run -d -p 10911:10911 -p 10909:10909 -v `pwd`/data/broker/logs:/root/logs -v `pwd`/data/broker/store:/root/store --name rmqbroker --link mqnamesrv:namesrv -e "NAMESRV_ADDR=namesrv:9876" -e "MAX_POSSIBLE_HEAP=200000000" rocketmqinc/rocketmq sh mqbroker
```

console服务

```sh
docker run -d --name rocketmq-console-ng -e "JAVA_OPTS=-Drocketmq.namesrv.addr=172.17.0.2:9876 -Dcom.rocketmq.sendMessageWithVIPChannel=false" -p 8080:8080 -t styletang/rocketmq-console-ng
```



## docker　搭建rocketmq集群



## Docker安装Nginx

```
docker run -d -it -p 80:8849  --name nginx -v /opt/nginx-1.18.0/conf/nginx.conf:/etc/nginx/nginx.conf -v `pwd`/logs:/var/log/nginx nginx:1.19.1
```

## **Docker 安装Portainer**

一款docker界面化UI

#### 单机版运行

输入执行命令：

`docker run -d -p 9000:9000 --restart=always -v /var/run/docker.sock:/var/run/docker.sock --name prtainer portainer/portainer:latest`

容器id生成，但是容器无法正常启动，出现如下错误：

```shell
docker: Error response from daemon: driver failed programming external connectivity on endpoint prtainer (25346ec8a2b4c3ecaa1d1c397af2282c68a8c5caf9e8b3077ce8fb0ee8223e06):  (iptables failed: iptables --wait -t nat -A DOCKER -p tcp -d 0/0 --dport 9000 -j DNAT --to-destination 172.17.0.2:9000 ! -i docker0: iptables: No chain/target/match by that name.
```

- 原因：

docker服务启动时定义的自定义链DOCKER由于某种原因被清掉（启动过portainer容器，后来被删除了）

- 解决：

重启docker服务后再启动容器，重新生成自定义链DOCKER

```shell
systemctl restart docker
```

自己设置的用户名和密码：

admin

12345678