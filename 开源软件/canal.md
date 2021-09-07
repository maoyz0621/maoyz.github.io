# 阿里巴巴 MySQL binlog 增量订阅&消费组件

canal.xxx-1.1.4

## canal.deployer



## canal.admin

```
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.48</version>
    <!--<scope>test</scope>-->
</dependency>
```



mysql 5.x

出现连接数据数错误





目标数据库-授权：

```
create user canal identified by 'canal';

grant all privileges on *.* to 'canal'@'%';

flush privileges;
```

登录：http://127.0.0.1:8089/

admin/123456

启动成功

![](image\Canal\canal.png)