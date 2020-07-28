# Nacos

## Nacos启动

ununtu系统启动Nacos，`sh startup.sh -m standalone` 报java.io.FileNotFoundException: /opt/nacos/conf/cluster.conf (没有那个文件或目录)
此时更换`bash startup.sh -m standalone`



当linux启动nacos服务时，`sh startup.sh -m standalone`无法正常启动，报错如下：

```shell
which: no javac in (/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin) 
readlink: missing operand
Try 'readlink --help' for more information.
dirname: missing operand
Try 'dirname --help' for more information.
ERROR: Please set the JAVA_HOME variable in your environment, We need java(x64)! jdk8 or later is better
```

这时候查看系统JAVA_HOME，使用`echo $JAVA_HOME`，空值，那是因为linux使用yum安装的java，默认路径为：**/usr/lib/jvm**，此时`vim /etc/profile`，在文件的最后添加

```shell
# set java environment
JAVA_HOME=/usr/lib/jvm/jre-1.8.0-openjdk-1.8.0.222.b03-1.el7.x86_64
PATH=$PATH:$JAVA_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JAVA_HOME CLASSPATH PATH
```

其中查看java所在的路径

生效文件  `source  /etc/profile`

/usr/lib/jvm/jre-1.8.0-openjdk-1.8.0.222.b03-1.el7.x86_64，配置JAVA_HOME

编辑`startup.sh`文件

```
# [ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java
# 替换成自己的JAVA_HOME路径
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/lib/jvm/jre-1.8.0-openjdk-1.8.0.222.b03-1.el7.x86_64/jre
```

即可正常启动服务



### 单机启动

```shell
sh startup.sh -m standalone
```

### 集群启动

方式一：

> 使用内置数据源

```shell
sh startup.sh -p embedded
```

方式二：
> 外置数据源

```shell
sh startup.sh
```



虚拟机启动nacos服务，192.168.107.100:8848，主机访问

nacos集群

[http://nacos.com](http://nacos.com/):port/openAPI  域名 + VIP模式，可读性好，而且换ip方便，推荐模式

#### 配置文件

- nacos/的conf目录下，有配置文件cluster.conf，配置如下，集群

```properties
# ip:port
192.168.107.128:8848
192.168.107.129:8848
192.168.107.130:8848
```

- application.properties配置文件

```properties
#*************** Config Module Related Configurations ***************#
### If use MySQL as datasource:
spring.datasource.platform=mysql

### Count of DB:
db.num=1

### Connect URL of DB:
db.url.0=jdbc:mysql://192.168.107.128:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user=root
db.password=root
```

出现如下提示，表示Success

```

```



- 192.168.107.100服务器上nginx，修改配置文件nginx.conf

```properties
http {
    ...
    
    # nacos配置
    upstream nacos {
        # ip地址
        server node1:8848  weight=3;	# node1:192.168.107.128, node2:192.168.107.129, node3:192.168.107.130
        server node2:8848;
        server node3:8848;
    }
	server {
        listen       8848;
        server_name  localhost;

        location / {
            root   html;
            index  index.html index.htm;
            # 负载均衡upstream别名
            proxy_pass   http://nacos;
        }
        ...
    }
}
```

## Nacos作为配置中心

```yaml
spring:
  application:
    name: service-nacos-config
  profiles:
    active: dev
  cloud:
    nacos:
      discovery:
        server-addr: 192.168.107.128:8848
      config:
        namespace: 24176a7b-df1e-4ff2-80cc-1c58b48ecc1a  # 命名空间：默认public
        server-addr: 192.168.107.128:8848 # 配置中心地址
        group: test1  # 分组
        file-extension: yaml  # 文件类型，yaml和 properties

    #   file-extension: properties

    # DataId: ${prefix}-${spring.profile.active}.${file-extension}：service-nacos-config-dev.yaml
    #     ${prefix} 默认为 spring.application.name 的值，也可以通过配置项 spring.cloud.nacos.config.prefix来配置。
    #     ${spring.profile.active} 即为当前环境对应的profile，详情可以参考 Spring Boot文档。 注意：当 spring.profile.active 为空时，对应的连接符 - 也将不存在，dataId 的拼接格式变成 ${prefix}.${file-extension}
    #     ${file-extension} yaml和 properties
```

参数说明：

- `namespace` 命名空间：默认为 spring.application.name 的值

- `group` 分组

- `file-extension` 文件后缀名

  ![](..\images\Nacos\Nacos Config.jpg)

- 创建命名空间namespace

![](..\images\Nacos\Nacos-namespace.png)

- 不同命名空间下的配置文件

![](..\images\Nacos\Nacos-env.png)



- 多文件配置

```yml
server:
  port: 8911

spring:
  application:
    name: service-nacos-config-ext
  profiles:
    active: dev
  cloud:
    nacos:
      discovery:
        server-addr: 192.168.107.128:8848   # 注册中心
      config:
        namespace: 70d7cd2f-0def-4b79-8baf-0d4664d7ddde  # 命名空间：默认public
        prefix: other # 使用配置名
        server-addr: 192.168.107.128:8848 # 配置中心地址
        group: test  # 分组
        file-extension: yaml  # 文件类型，yaml和 properties
        # 多配置文件
        ext-config[0]:
          data‐id: mysql.yaml
          group: ext
          refresh: true
        ext-config[1]:
          data‐id: middle.yaml
          group: ext
          refresh: true

# 扩展配置优先级是 ext-config[n].data-id 其中 n 的值越大，优先级越高。通过内部相关规则(应用名、扩展名 )自动生成相关的 Data Id 配置的优先级最大。
```

![](..\images\Nacos\Nacos-config-ext.png)

## 命名空间

用于进行租户粒度的配置隔离。不同的命名空间下，可以存在相同的 Group 或 Data ID 的配置。Namespace 的常用场景之一是不同环境的配置的区分隔离，例如开发dev、测试环境test和生产环境pre的资源（如配置、服务）隔离等。

## 配置集 ID

Nacos 中的某个配置集的 ID。配置集 ID 是组织划分配置的维度之一。Data ID 通常用于组织划分系统的配置集。一个系统或者应用可以包含多个配置集，每个配置集都可以被一个有意义的名称标识。Data ID 通常采用类 Java 包（如 com.taobao.tc.refund.log.level）的命名规则保证全局唯一性。此命名规则非强制。

## 配置分组

Nacos 中的一组配置集，是组织配置的维度之一。通过一个有意义的字符串（如 Buy 或 Trade ）对配置集进行分组，从而区分 Data ID 相同的配置集。当您在 Nacos 上创建一个配置时，如果未填写配置分组的名称，则配置分组的名称默认采用 DEFAULT_GROUP 。配置分组的常见场景：**不同的应用或组件使用了相同的配置类型**，如 database_url 配置和 MQ_topic 配置。