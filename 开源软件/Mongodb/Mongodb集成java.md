# 集成java



## springboot

依赖：

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

配置属性:

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://root:root@192.168.107.100:27017/mydb
```

