# 分布式配置中心 - Apollo

支持环境:DEV（开发环境）、FAT（测试环境）、UAT（预生产）、PRO（生产）

# 编译环境

1. 修改apollo-configservice, apollo-adminservice和apollo-portal的pom.xml，注释掉spring-boot-maven-plugin和maven-assembly-plugin
2. 在根目录下执行mvn clean package -pl apollo-assembly -am -DskipTests=true
3. 复制apollo-assembly/target下的jar包，rename为apollo-all-in-one.jar


通过配置文件来指定env=YOUR-ENVIRONMENT
对于Mac/Linux，文件位置为/opt/settings/server.properties


本地缓存
Apollo客户端会把从服务端获取到的配置在本地文件系统缓存一份，当去服务器读取配置失败时，会使用本地缓存的。
Mac/Linux: /opt/data/{appId}/config-cache




比较重要的几个项目：

+ `apollo-configservice`：提供配置获取接口，提供配置更新推送接口，接口服务对象为Apollo客户端
+ `apollo-adminservice`：提供配置管理接口，提供配置修改、发布等接口，接口服务对象为Portal，以及Eureka
+ `apollo-portal`：提供Web界面供用户管理配置
+ `apollo-client`：Apollo提供的客户端程序，为应用提供配置获取、实时更新等功能
注:Config Service和Admin Service都是多实例、无状态部署，所以需要将自己注册到Eureka中并保持心跳,Config Service、Eureka和Meta Server三个逻辑角色部署在同一个JVM进程中

ApolloPortalDB：执行apollo\scripts\sql\apolloportaldb.sql
ApolloConfigDB：DEV FAT UAT PRO 环境执行apollo\scripts\sql\apolloconfigdb.sql


# apollo config db info  该数据库配置只需要配置一次，不同环境无需修改
apollo_config_db_url=jdbc:mysql://192.168.35.227:3306/ApolloConfigDB?characterEncoding=utf8
apollo_config_db_username=root
apollo_config_db_password=root

# apollo portal db info  该数据库依据不同环境配置对应的数据库连接，并且需要多次打对应环境的JAR包
apollo_portal_db_url=jdbc:mysql://192.168.35.226:3306/ApolloPortalDB?characterEncoding=utf8
apollo_portal_db_username=root
apollo_portal_db_password=root

自定义打包时,启动顺序:
    config -> admin -> protal

启动apollo服务:https://github.com/nobodyiam/apollo-build-scripts
Java客户端使用指南:https://github.com/ctripcorp/apollo/wiki/Java客户端使用指南
开放平台:https://github.com/ctripcorp/apollo/wiki/Apollo开放平台


-Dapollo.meta=http://localhost:8080 -Dserver.port=8888