# Nacos整合Spring

- Springboot版本2.2.x 分支

| Spring Cloud Alibaba Version      | Spring Cloud Version        | Spring Boot Version |
| :-------------------------------- | :-------------------------- | :------------------ |
| 2021.0.1.0                        | Spring Cloud 2021.0.1       | 2.6.3               |
| 2.2.7.RELEASE                     | Spring Cloud Hoxton.SR12    | 2.3.12.RELEASE      |
| 2021.1                            | Spring Cloud 2020.0.1       | 2.4.2               |
| 2.2.6.RELEASE                     | Spring Cloud Hoxton.SR9     | 2.3.2.RELEASE       |
| 2.1.4.RELEASE                     | Spring Cloud Greenwich.SR6  | 2.1.13.RELEASE      |
| 2.2.1.RELEASE                     | Spring Cloud Hoxton.SR3     | 2.2.5.RELEASE       |
| 2.2.0.RELEASE                     | Spring Cloud Hoxton.RELEASE | 2.2.X.RELEASE       |
| 2.1.2.RELEASE                     | Spring Cloud Greenwich      | 2.1.X.RELEASE       |
| 2.0.4.RELEASE(停止维护，建议升级) | Spring Cloud Finchley       | 2.0.X.RELEASE       |
| 1.5.1.RELEASE(停止维护，建议升级) | Spring Cloud Edgware        | 1.5.X.RELEASE       |

- 组件版本

| Spring Cloud Alibaba Version                              | Sentinel Version | Nacos Version | RocketMQ Version | Dubbo Version | Seata Version |
| :-------------------------------------------------------- | :--------------- | :------------ | :--------------- | :------------ | :------------ |
| 2.2.9.RELEASE                                             | 1.8.5            | 2.1.0         | 4.9.4            |               | 1.5.2         |
| 2.2.8.RELEASE                                             | 1.8.4            | 2.1.0         | 4.9.3            |               | 1.5.1         |
| 2021.0.1.0*                                               | 1.8.3            | 1.4.2         | 4.9.2            | 2.7.15        | 1.4.2         |
| 2.2.7.RELEASE                                             | 1.8.1            | 2.0.3         | 4.6.1            | 2.7.13        | 1.3.0         |
| 2.2.6.RELEASE                                             | 1.8.1            | 1.4.2         | 4.4.0            | 2.7.8         | 1.3.0         |
| 2021.1 or 2.2.5.RELEASE or 2.1.4.RELEASE or 2.0.4.RELEASE | 1.8.0            | 1.4.1         | 4.4.0            | 2.7.8         | 1.3.0         |
| 2.2.3.RELEASE or 2.1.3.RELEASE or 2.0.3.RELEASE           | 1.8.0            | 1.3.3         | 4.4.0            | 2.7.8         | 1.3.0         |
| 2.2.1.RELEASE or 2.1.2.RELEASE or 2.0.2.RELEASE           | 1.7.1            | 1.2.1         | 4.4.0            | 2.7.6         | 1.2.0         |
| 2.2.0.RELEASE                                             | 1.7.1            | 1.1.4         | 4.4.0            | 2.7.4.1       | 1.0.0         |
| 2.1.1.RELEASE or 2.0.1.RELEASE or 1.5.1.RELEASE           | 1.7.0            | 1.1.4         | 4.4.0            | 2.7.3         | 0.9.0         |
| 2.1.0.RELEASE or 2.0.0.RELEASE or 1.5.0.RELEASE           | 1.6.3            | 1.1.1         | 4.4.0            | 2.7.3         | 0.7.1         |



## SpringBoot



如果想要使用`spring-boot`的条件注解`@ConditionXXX`功能、`@Value`注解以及将dubbo的配置放到nacos上等功能，需要使用 `nacos-spring-boot-project` 的 `0.2.2` 或 `0.1.2` 版本

### nacos-config

- pom依赖

```xml
<dependency>
    <groupId>com.alibaba.boot</groupId>
    <artifactId>nacos-config-spring-boot-starter</artifactId>
    <version>0.2.11</version>
    <exclusions>
        <exclusion>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-client</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>com.alibaba.nacos</groupId>
    <artifactId>nacos-client</artifactId>
    <version>2.1.0</version>
</dependency>
```

- properties配置

```properties
# nacos-config配置
# 开启配置预加载功能
nacos.config.bootstrap.enable=true
nacos.config.namespace=6cb18b3e-51e0-411a-9a7c-9660dbbeef1e
nacos.config.server-addr=127.0.0.1:8848
# 开启自动刷新
nacos.config.auto-refresh=true
# 允许nacos上的配置优先于本地配置
nacos.config.remote-first=true
nacos.config.data-id=test
nacos.config.type=properties
# 主配置 最大重试次数
nacos.config.max-retry=10
# 主配置 重试时间
nacos.config.config-retry-time=2333
# 主配置 配置监听长轮询超时时间
nacos.config.config-long-poll-timeout=46000
# 配置文件扩展 index越小，优先级越高，从0开始
nacos.config.ext-config[0].data-id=test
nacos.config.ext-config[0].group=nacos-springboot
nacos.config.ext-config[0].max-retry=10
nacos.config.ext-config[0].type=properties
nacos.config.ext-config[0].auto-refresh=true
nacos.config.ext-config[0].config-retry-time=2333
nacos.config.ext-config[0].config-long-poll-timeout=46000
nacos.config.ext-config[0].enable-remote-sync-config=true
```

- 属性值使用

```java
// 读取配置，autoRefreshed = true刷新最新值
@NacosValue(value = "${a}", autoRefreshed = true)

// 读取配置，无法刷新最新值
@Value("${b}")
```



### nacos-config

- pom依赖

```xml
<dependency>
    <groupId>com.alibaba.boot</groupId>
    <artifactId>nacos-discovery-spring-boot-starter</artifactId>
    <version>0.2.11</version>
    <exclusions>
        <exclusion>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-client</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>com.alibaba.nacos</groupId>
    <artifactId>nacos-client</artifactId>
    <version>2.1.0</version>
</dependency>
```
启动出现如下错误：
```shell
Caused by: java.lang.ClassCastException: class com.sun.proxy.$Proxy97 cannot be cast to class java.util.Map (com.sun.proxy.$Proxy97 is in unnamed module of loader 'app'; java.util.Map is in module java.base of loader 'bootstrap')
	at com.alibaba.nacos.spring.beans.factory.annotation.AnnotationNacosInjectedBeanPostProcessor.getNacosProperties(AnnotationNacosInjectedBeanPostProcessor.java:145) ~[nacos-spring-context-1.1.1.jar:na]
	at com.alibaba.nacos.spring.beans.factory.annotation.AnnotationNacosInjectedBeanPostProcessor.buildInjectedObjectCacheKey(AnnotationNacosInjectedBeanPostProcessor.java:135) ~[nacos-spring-context-1.1.1.jar:na]
	at com.alibaba.spring.beans.factory.annotation.AbstractAnnotationBeanPostProcessor.getInjectedObject(AbstractAnnotationBeanPostProcessor.java:354) ~[spring-context-support-1.0.6.jar:na]
	at com.alibaba.spring.beans.factory.annotation.AbstractAnnotationBeanPostProcessor$AnnotatedFieldElement.inject(AbstractAnnotationBeanPostProcessor.java:539) ~[spring-context-support-1.0.6.jar:na]
	at org.springframework.beans.factory.annotation.InjectionMetadata.inject(InjectionMetadata.java:130) ~[spring-beans-5.2.4.RELEASE.jar:5.2.4.RELEASE]
	
```

错误原因：`NacosDiscoveryAutoRegister`中注解`@NacosInjected`是一个代理类proxy，获取其属性值装换Map错误

解决方案：替换  `spring-context-support`

```xml
<dependency>
    <groupId>com.alibaba.boot</groupId>
    <artifactId>nacos-discovery-spring-boot-starter</artifactId>
    <version>${nacos.boot.version}</version>
    <exclusions>
        <exclusion>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-client</artifactId>
        </exclusion>
        <exclusion>
            <groupId>com.alibaba.spring</groupId>
            <artifactId>spring-context-support</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>com.alibaba.spring</groupId>
    <artifactId>spring-context-support</artifactId>
    <!-- <version>1.0.6</version> -->
    <version>1.0.11</version>
</dependency>
```






## SpringCloud

> **注意**：版本 [2.1.x.RELEASE](https://mvnrepository.com/artifact/com.alibaba.cloud/spring-cloud-starter-alibaba-nacos-config) 对应的是 Spring Boot 2.1.x 版本。版本 [2.0.x.RELEASE](https://mvnrepository.com/artifact/com.alibaba.cloud/spring-cloud-starter-alibaba-nacos-config) 对应的是 Spring Boot 2.0.x 版本，版本 [1.5.x.RELEASE](https://mvnrepository.com/artifact/com.alibaba.cloud/spring-cloud-starter-alibaba-nacos-config) 对应的是 Spring Boot 1.5.x 版本

- pom依赖

```xml
<!-- nacos 注册中心 -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    <version>2.2.1.RELEASE</version>
</dependency>

<!-- nacos 配置中心 -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    <version>2.2.1.RELEASE</version>
</dependency>
```



## 配置文件的读取优先级

> 总结：
>
> (线上) 服务名-profile.yml > 服务名.yml >
>
> extension-configs[3] > extension-configs[2] >
>
> shared-configs[3] > (本地) shared-configs[2] >
>
> bootstrap.yml > bootstrap.properties >
>
> application.yml > application.yaml > application.properties