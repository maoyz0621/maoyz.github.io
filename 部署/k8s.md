# k8s

## １. 是什么？

## 2. 有什么用？

## 3. 组成？

## 4. 安装？

## 5. 启动？

微服务通讯

        一对一　　　一对多
同步　　　
异步　    通知/请求异步      发布订阅/发布异步响应

IO/线程模式
序列化方式
服务治理


服务发现

客户端发现和服务端发现

服务扩容

服务编排

docker服务通讯：
１　直接通讯
２  端口映射访问
３　link服务名称

编写build.sh，　start.sh，　stop.sh

docker-compose.yml

```
version: '1'

services:
    service-1:  # 服务名称
        images: service-1:latest   #　镜像名称:版本号


    service-2:  # 服务名称
        images: service-2:latest   #　镜像名称:版本号
        command: 
        - "--mysql.adress=192.168.0.1"


    ervice-3:  # 服务名称
        images: service-3:latest   #　镜像名称:版本号
        links:  #　依赖服务(服务名称)
        - service-1
        - service-2
        command: 
        - "--redis-adress=192.168.0.1"


    ervice-4:  # 服务名称
        images: service-4:latest   #　镜像名称:版本号
        links:  #　依赖服务(服务名称)
        - service-3
        - service-2
        command: 
        - "--redis-adress=192.168.0.1"
        ports:
        - 8080:8080 # 端口映射(网关)


```

# 启动
docker-compose up -d


# 镜像仓库

私有和公共仓库

上传镜像

docker tag
docker push

创建私有仓库?
docker pull resistry:2
docker run 5000


harbor(docker界面管理)
    harbor.cfg

    hostname:
运行启动脚本
./install.sh


# 日志收集



+ Apache Mesos

+ Docker Swarm

+ K8s

Service

Master

K8s -> Client -> API Server -> Etcd

Scheduler

Controller Manager -> node -> pod


Node
    Pod  
        volume
        ip address
        containerized app

    Deplyment
    Service

容器生命周期

资源清单

探针

Service

资源控制器



设计理念:
API设计原则
控制机设计原则

网络


调度过程