# Nacos服务部署

## 单机部署

> 官方文档：https://nacos.io/zh-cn/docs/v2/quickstart/quick-start.html

### Windows

#### 启动

命令行启动：standalone-单机，默认集群启动

```sh
startup.cmd -m standalone
```

或修改：startup.cmd

```shell
set MODE="standalone"
```

#### 支持mysql

- 初始化数据：`mysql-schema.sql`

- 修改`conf/application.properties`

```properties
spring.datasource.platform=mysql

db.num=1
db.url.0=jdbc:mysql://11.162.196.16:3306/nacoscharacterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true
db.user.0=root
db.password.0=root
```



#### 启动报错

端口被占用

```undefined
netstat -aon|findstr 8848
```



```
taskkill /f /t /im 进程名
```





##### 报错情况-1

> 在2.2.0.1和2.2.1版本时，必须执行此变更，否则无法启动；其他版本为建议设置。

```shell
Caused by: java.lang.IllegalArgumentException: The specified key byte array is 0 bits which is not secure enough for any JWT HMAC-SHA algorithm.  The JWT JWA Specification (RFC 7518, Section 3.2) states that keys used with HMAC-SHA algorithms MUST have a size >= 256 bits (the key size must be greater than or equal to the hash output size).  See https://tools.ietf.org/html/rfc7518#section-3.2 for more information.
	at com.alibaba.nacos.plugin.auth.impl.jwt.NacosJwtParser.<init>(NacosJwtParser.java:56)
	at com.alibaba.nacos.plugin.auth.impl.token.impl.JwtTokenManager.processProperties(JwtTokenManager.java:71)
	... 124 common frames omitted
```

#### 解决方案-1

修改：`application.propertise`

```properties
nacos.core.auth.plugin.nacos.token.secret.key=asdkljiosdfsnklkiklksfisdrynvxndhifue
```



##### 报错情况-2

```
Caused by: java.lang.UnsupportedOperationException: Cannot determine JNI library name for ARCH='x86' OS='windows 8.1' name='rocksdb'
        at org.rocksdb.util.Environment.getJniLibraryName(Environment.java:137)
        at org.rocksdb.NativeLibraryLoader.<clinit>(NativeLibraryLoader.java:20)
```

Mysql无法连接电脑的本地IP，解决办法：

```mysql
create user 'nacos' @'192.168.0.107' identified by 'nacos';
grant all privileges on *.* to 'root'@'192.168.0.107' with grant option;
flush PRIVILEGES;

create user 'nacos' @'%' identified by 'nacos';
grant all privileges on nacos.* to 'nacos'@'%' with grant option;
flush PRIVILEGES;
```



springboot启动报错：

```
Caused by: java.lang.ClassCastException: class com.sun.proxy.$Proxy95 cannot be cast to class java.util.Map (com.sun.proxy.$Proxy95 is in unnamed module of loader 'app'; java.util.Map is in module java.base of loader 'bootstrap')
	at com.alibaba.nacos.spring.beans.factory.annotation.AnnotationNacosInjectedBeanPostProcessor.getNacosProperties(AnnotationNacosInjectedBeanPostProcessor.java:145) ~[nacos-spring-context-1.1.1.jar:na]
	at com.alibaba.nacos.spring.beans.factory.annotation.AnnotationNacosInjectedBeanPostProcessor.buildInjectedObjectCacheKey(AnnotationNacosInjectedBeanPostProcessor.java:135) ~[nacos-spring-context-1.1.1.jar:na]
	at com.alibaba.spring.beans.factory.annotation.AbstractAnnotationBeanPostProcessor.getInjectedObject(AbstractAnnotationBeanPostProcessor.java:354) ~[spring-context-support-1.0.6.jar:na]
	at com.alibaba.spring.beans.factory.annotation.AbstractAnnotationBeanPostProcessor$AnnotatedFieldElement.inject(AbstractAnnotationBeanPostProcessor.java:539) ~[spring-context-support-1.0.6.jar:na]
	at org.springframework.beans.factory.annotation.InjectionMetadata.inject(InjectionMetadata.java:130) ~[spring-beans-5.2.4.RELEASE.jar:5.2.4.RELEASE]
	at com.alibaba.spring.beans.factory.annotation.AbstractAnnotationBeanPostProcessor.postProcessPropertyValues(AbstractAnnotationBeanPostProcessor.java:142) ~[spring-context-support-1.0.6.jar:na]
	... 22 common frames omitted
```



### Linux

#### 启动

命令行启动：standalone-单机，默认集群启动

```sh
$ sh startup.sh -m standalone
```



### Docker



## 集群部署


使用nacos集群，192.168.107.100服务器 nginx反向代理

192.168.107.128:8848
192.168.107.129:8848
192.168.107.130:8848