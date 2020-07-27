# nacos启动

ununtu系统启动Nacos，`sh startup.sh -m standalone` 报java.io.FileNotFoundException: /opt/nacos/conf/cluster.conf (没有那个文件或目录)
此时更换`bash startup.sh -m standalone`



linux启动nacos服务时，`sh startup.sh -m standalone`无法正常启动，报错如下：

```shell
which: no javac in (/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin) 
readlink: missing operand
Try 'readlink --help' for more information.
dirname: missing operand
Try 'dirname --help' for more information.
ERROR: Please set the JAVA_HOME variable in your environment, We need java(x64)! jdk8 or later is better
```

这时候查看系统JAVA_HOME，使用`echo $JAVA_HOME`，空值，那是因为linux使用yum安装的java，默认路径为：**/usr/lib/jvm**，此时`vim /etc/profile`，在文件的最后添加

```
# set java environment
JAVA_HOME=/usr/lib/jvm/jre-1.8.0-openjdk-1.8.0.222.b03-1.el7.x86_64
PATH=$PATH:$JAVA_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JAVA_HOME CLASSPATH PATH
```

其中查看java所在的路径

生效文件  `source  /etc/profile`

/usr/lib/jvm/jre-1.8.0-openjdk-1.8.0.222.b03-1.el7.x86_64，配置JAVA_HOME



编辑`startup.sh`

```
# [ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java
# 替换成自己的JAVA_HOME路径
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/lib/jvm/jre-1.8.0-openjdk-1.8.0.222.b03-1.el7.x86_64/jre
```

即可正常启动服务





单机启动

```
sh startup.sh -m standalone
```

集群启动

方式一：

> 使用内置数据源

```
sh startup.sh -p embedded
```

> 外置数据源

```
sh startup.sh
```



虚拟机启动nacos服务，192.168.107.100:8848，主机访问

nacos集群

[http://nacos.com](http://nacos.com/):port/openAPI  域名 + VIP模式，可读性好，而且换ip方便，推荐模式



nacos/的conf目录下，有配置文件cluster.conf，

```properties
# ip:port
192.168.107.128:8848
192.168.107.129:8848
192.168.107.130:8848
```

application.properties配置文件

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

192.168.107.100服务器上nginx，修改配置文件nginx.conf

```properties
http {
    ...
    
    # nacos配置
    upstream nacos {
          # ip地址
          server node1:8848  weight=3;	# node1:192.168.107.128,node2:192.168.107.129,node3:192.168.107.130
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
    }
}
```

