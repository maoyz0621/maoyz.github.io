# 集群搭建

> 集群搭建方式

1. 主从复制（Master - Slave）
2. 副本集（Replica Set）
3. 分片（Sharding）

> 第一种方式基本没什么意义，官方也不推荐这种方式搭建。另外两种分别就是副本集和分片的方式。

四个组件：mongos，config server，shard，replica set



## 概念

### mongos

集群请求的入口，进行协调，请求分发中心，负责将外部的请求分发到对应的shard服务器上，需要HA

### ConfigServer

配置服务器，存储所有元数据（分片、路由），mongos在第一次启动或后期重启的时候，就会从Config Server加载配置信息。配置信息发生变更会通知所有的mongos更新状态，从而保证准确的请求路由。

生产环境中需要多个Config Server

### 分片shard

设置好分片规则，mongos自动把对应的操作请求转发到对应的分片服务器上

### 副本集

对于shard节点需要replica set来保证数据的可靠性，生产环境通常为2个副本+1个仲裁

### 路由Router



## 搭建

- mongos： 3台
- config  server ： 3台
- shard ： 3片； 每个分片由三个节点构成

部署情况：

### 容器部署情况

| 角色           | 端口  | 暴漏端口 | 描述       | 角色     |
| -------------- | ----- | -------- | ---------- | -------- |
| config-server1 | 27017 | --       | 配置节点1  | --       |
| config-server2 | 27017 | --       | 配置节点2  | --       |
| config-server3 | 27017 | --       | 配置节点3  | --       |
| mongos-server1 | 27017 | 30001    | 路由节点1  | --       |
| mongos-server2 | 27017 | 30002    | 路由节点2  | --       |
| mongos-server3 | 27017 | 30003    | 路由节点3  | --       |
| shard1-server1 | 27017 | --       | 分片1节点1 | Primary  |
| shard1-server2 | 27017 | --       | 分片1节点2 | Secondry |
| shard1-server3 | 27017 | --       | 分片1节点3 | Arbiter  |
| shard2-server1 | 27017 | --       | 分片2节点1 | Primary  |
| shard2-server2 | 27017 | --       | 分片2节点2 | Secondry |
| shard2-server3 | 27017 | --       | 分片2节点3 | Arbiter  |
| shard3-server1 | 27017 | --       | 分片3节点1 | Primary  |
| shard3-server2 | 27017 | --       | 分片3节点2 | Secondry |
| shard3-server3 | 27017 | --       | 分片3节点3 | Arbiter  |





<img src=".\img\mongo集群架构.png" alt="mongo集群架构" style="zoom: 50%;" />

[【超详细】手把手教你搭建MongoDB集群 - 掘金 (juejin.cn)](https://juejin.cn/post/7120119206615941151)