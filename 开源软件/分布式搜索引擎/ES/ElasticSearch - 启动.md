# ElasticSearch启动

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





```
java.lang.IllegalStateException: failed to obtain node locks, tried [[/usr/local/elasticsearch-7.9.3/data]] with lock id [0]; maybe these locations are not writable or multiple nodes were started without increasing [node.max_local_storage_nodes] (was [1])?
	at org.elasticsearch.env.NodeEnvironment.<init>(NodeEnvironment.java:301)
```



ps aux | grep ‘elastic’ 



cluster.initial_master_nodes: ["node-1"]



`curl http://localhost:9200`查看ES信息

```json
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

