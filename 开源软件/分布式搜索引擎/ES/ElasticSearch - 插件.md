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

![](..\开源软件\image\elasticsearch-head.jpg)

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



```sh
## 创建局域网
docker network create elastic

## 
docker run --name elasticsearch --net elastic -p 9200:9200 -p 9300:9300 -e discovery.type=single-node -e ES_JAVA_OPTS="-Xms256m -Xmx256m" -e http.cors.enabled=true -e http.cors.allow-origin=* -e http.cors.allow-headers=X-Requested-With,X-Auth-Token,Content-Type,Content-Length,Authorization -e "http.cors.allow-credentials=true"  -d elasticsearch:7.9.3

docker run --name kib01-test --net elastic -p 5601:5601 -e "ELASTICSEARCH_HOSTS=http://elasticsearch:9200" -d kibana:7.9.3



```

### 创建容器并运行

```
# 拉取镜像
docker pull docker.elastic.co/kibana/kibana:7.9.3

# 后台创建并运行容器
docker run -it --name kibana -p 5601:5601 
-v /usr/local/kibana/logs/kibana.log:/usr/share/kibana/logs/kibana.log 
-v /usr/local/kibana/data:/usr/share/kibana/data 
-v /usr/local/kibana/conf/kibana.1.yml:/usr/share/kibana/config/kibana.yml 
-d docker.elastic.co/kibana/kibana:7.9.3

docker run -it --name kibana -p 5601:5601 
-e elasticsearch.hosts=["http://192.168.107.110:9200"]
-v /usr/local/kibana/logs/kibana.log:/usr/share/kibana/logs/kibana.log 
-v /usr/local/kibana/data:/usr/share/kibana/data 
-d docker.elastic.co/kibana/kibana:7.9.3
```





错误：

[BABEL] Note: The code generator has deoptimised the styling of /usr/share/kibana/x-pack/plugins/canvas/server/templates/pitch_presentation.js as it exceeds the max of 500KB.

 FATAL  Error: Unable to write Kibana UUID file, please check the uuid.server configuration value in kibana.yml and ensure Kibana has sufficient（足够的） permissions to read / write to this file. Error was: ENOENT



解决方案：







https://www.cnblogs.com/michael-xiang/p/13715692.html