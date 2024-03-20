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



### nacos-discovery

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



```xml
<!-- 2.2.x 分支 -->
<properties>
	<spring-cloud.version>Hoxton.SR3</spring-cloud.version>
	<spring-boot.version>2.2.5.RELEASE</spring-boot.version>
	<spring-cloud-alibaba.version>2.2.1.RELEASE</spring-cloud-alibaba.version>
</properties>

<!-- 2021.x 分支 -->
<properties>
	<spring-cloud.version>2021.0.5</spring-cloud.version>
	<spring-boot.version>2.6.13</spring-boot.version>
	<spring-cloud-alibaba.version>2021.0.5.0</spring-cloud-alibaba.version>
</properties>

<!-- 2022.x 分支 -->
<properties>
	<spring-cloud.version>2022.0.0</spring-cloud.version>
	<spring-boot.version>3.0.2</spring-boot.version>
	<spring-cloud-alibaba.version>2022.0.0.0</spring-cloud-alibaba.version>
</properties>
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-parent</artifactId>
            <version>${spring-boot.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>

        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>${spring-cloud-alibaba.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>

        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
            <version>${spring-cloud-alibaba.version}</version>
        </dependency>

        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
            <version>${spring-cloud-alibaba.version}</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 配置文件的读取优先级

- bootstrap.yml，spring.profiles.active=dev

```yaml
spring:
  application:
    name: springcloud2021-nacos
  cloud:
    discovery:
      enabled: true
    nacos:
      discovery:
        namespace: 6cb18b3e-51e0-411a-9a7c-9660dbbeef1e  # 命名空间：默认public
        server-addr: 127.0.0.1:8848
        enabled: true
      config:
        namespace: 6cb18b3e-51e0-411a-9a7c-9660dbbeef1e  # 命名空间：默认public
        server-addr: 127.0.0.1:8848 # 配置中心地址
        file-extension: properties  # 文件类型，yaml和 properties
        prefix: ${spring.application.name} # 使用配置名
        # prefix: springcloud2021-nacos-other # 使用配置名
        group: dev  # 分组
        refresh-enabled: true
        # 共享配置文件
        shared-configs: # 共享配置文件
          - data-id: common.properties
            group: DEFAULT_GROUP
            refresh: true

        # 延展配置，多个配置 写在下面的优先级高：
        extension-configs:
          - data-id: mysql.yaml
            group: ext
            refresh: true
          - data‐id: middle.yaml
            group: ext
            refresh: true
        # 延展配置方法二，多个配置 写在下面的优先级高：    
        extension-configs[0]:
          data-id: mysql.yaml
          group: ext
          refresh: true
        extension-configs[1]:
          data‐id: middle.yaml
          group: ext
          refresh: true
```

默认配置文件：springcloud2021-nacos-dev.properties

使用别名`springcloud2021-nacos-other`配置文件：springcloud2021-nacos-other.properties

| 属性文件                        |
| ------------------------------- |
| ![](.\images\Nacos属性文件.jpg) |

> ```
> 默认属性文件
> DataId: ${prefix}-${spring.profile.active}.${file-extension}：springcloud2021-nacos-dev.properties
> - ${prefix}：默认为spring.application.name的值，也可以通过配置项 spring.cloud.nacos.config.prefix来配置
> - ${file-extension}：yaml和 properties
> ```

扩展配置`com.alibaba.cloud.nacos.NacosConfigProperties#extensionConfigs`和

共享配置`com.alibaba.cloud.nacos.NacosConfigProperties#sharedConfigs`

默认主配置：

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



原因分析：

- `com.alibaba.cloud.nacos.client.NacosPropertySourceLocator#locate`

```java
public PropertySource<?> locate(Environment env) {
	nacosConfigProperties.setEnvironment(env);
	ConfigService configService = nacosConfigManager.getConfigService();

	if (null == configService) {
		log.warn("no instance of config service found, can't load config from nacos");
		return null;
	}
	long timeout = nacosConfigProperties.getTimeout();
	nacosPropertySourceBuilder = new NacosPropertySourceBuilder(configService,
			timeout);
	String name = nacosConfigProperties.getName();

	String dataIdPrefix = nacosConfigProperties.getPrefix();
	if (StringUtils.isEmpty(dataIdPrefix)) {
		dataIdPrefix = name;
	}

	if (StringUtils.isEmpty(dataIdPrefix)) {
		dataIdPrefix = env.getProperty("spring.application.name");
	}

	CompositePropertySource composite = new CompositePropertySource(
			NACOS_PROPERTY_SOURCE_NAME);
	// 加载 shared-configs 配置
	loadSharedConfiguration(composite);
    // 加载 extension-configs 配置
	loadExtConfiguration(composite);
    // 加载应用内配置
	loadApplicationConfiguration(composite, dataIdPrefix, nacosConfigProperties, env);
	return composite;
}


/**
 * load configuration of application.
 */
private void loadApplicationConfiguration(
		CompositePropertySource compositePropertySource, String dataIdPrefix,
		NacosConfigProperties properties, Environment environment) {
    // 不设置，默认值：properties
	String fileExtension = properties.getFileExtension();
	String nacosGroup = properties.getGroup();
	// load directly once by default 加载默认的，不追加扩展符的 data-id 文件：
    // springcloud2021-nacos
	loadNacosDataIfPresent(compositePropertySource, dataIdPrefix, nacosGroup,
			fileExtension, true);
	// load with suffix, which have a higher priority than the default
    // 加载带扩展符的 data-id 文件，优先级大于默认的：
    // springcloud2021-nacos.properties
	loadNacosDataIfPresent(compositePropertySource,
			dataIdPrefix + DOT + fileExtension, nacosGroup, fileExtension, true);
	// Loaded with profile, which have a higher priority than the suffix
    // 加载带环境标识的 data-id 文件，优先级大于带扩展符的：
    // springcloud2021-nacos-dev.properties
	for (String profile : environment.getActiveProfiles()) {
		String dataId = dataIdPrefix + SEP1 + profile + DOT + fileExtension;
		loadNacosDataIfPresent(compositePropertySource, dataId, nacosGroup,
				fileExtension, true);
	}

}
```



### SpringCloud注册

```java
org.springframework.cloud.client.serviceregistry.ServiceRegistry
	NacoServiceRegistry
    	register()调用com.alibaba.nacos.api.naming.NamingService#registerInstance完成服务注册
```



实现过程：

```java
AutoServiceRegistrationAutoConfiguration
	AutoServiceRegistration
		AbstractAutoServiceRegistration
			NacosAutoServiceRegistration
```

Nacos是通过Spring的事件机制onApplicationEvent()继承到SpringCloud中