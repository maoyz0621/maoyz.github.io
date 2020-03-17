# zk -dubbo
## Zookeeper 保存的Dubbo信息详解

### 数据存储结构

| zk中保存的dubbo数据格式 |
| :----------------------------------------------------------: |
|        ![image-20200317094536432](C:%5CUsers%5Cdell%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20200317094536432.png)                                                      |

流程说明：

+ 服务提供者启动时：向/dubbo/com.foo.BaseService/providers/目录下写入URL地址
+ 服务消费者启动时：向/dubbo/com.foo.BaseService/consumers/目录下写入URL地址，同时订阅/dubbo/com.foo.BaseService/providers/目录下提供者URL地址
+ 监控中心启动时，订阅/dubbo/com.foo.BaseService/目录下的所有提供者和消费者的URL地址。



支持功能：

1. 当提供者出现断电等异常停机时，注册中心能自动删除提供者信息
2. 当注册中心重启时，能自动恢复注册数据，以及订阅请求
3. 当会话过期时，能自动恢复注册数据，以及订阅请求
4. 当设置 <dubbo:registry check=“false” /> 时，记录失败注册和订阅请求，后台定时重试
5. 可通过 <dubbo:registry username=“admin” password=“1234” /> 设置 zookeeper 登录信息
6. 可通过 <dubbo:registry group=“dubbo” /> 设置 zookeeper 的根节点，不设置将使用无根树
7. 支持 * 号通配符 <dubbo:reference group="*" version="*" />，可订阅服务的所有分组和所有版本的提供者



| 序号 | 节点       | 说明                                                         |
| ---- | ---------- | ------------------------------------------------------------ |
| 1    | 根节点     | dubbo                                                        |
| 2    | 一级子节点 | 提供服务的全限定名（接口名称）                               |
| 3    | 二级子节点 | 固定的四个子节点：分别为：consumers、configurators、routers、providers |

在Provider端尽量多配置Consume 端属性,Consumer端不配置则会使用Provider端的配置，即Provider端的配置可以作为Consumer的缺省值。否则，Consumer会使用Consumer端的全局设置，这对于Provider是不可控的，并且往往是不合理的。

**Provider端配置的Consume 端属性建议**：

1. `timeout`：方法调用的超时时间
2. `retries`：失败重试次数，缺省是2 
3. `loadbalance`：负载均衡算法，缺省是随机 `random`。还可以配置轮询 `roundrobin`、最不活跃优先 `leastactive` 和一致性哈希 `consistenthash` 等
4. `actives`：消费者端的最大并发调用限制，即当Consumer对一个服务的并发调用到上限后，新调用会阻塞直到超时，在方法上配置 `dubbo:method` 则针对该方法进行并发限制，在接口上配置 `dubbo:service`，则针对该服务进行并发限制

**在Provider端配置的Provider端属性有：**

1. `threads`：服务线程池大小
2. `executes`：一个服务提供者并行执行请求上限，即当 Provider 对一个服务的并发调用达到上限后，新调用会阻塞，此时 Consumer 可能会超时。在方法上配置 `dubbo:method` 则针对该方法进行并发限制，在接口上配置 `dubbo:service`，则针对该服务进行并发限制

### Consumers

/dubbo

​	/com.homestead.api.service.goods.ServiceInsurancesService/consumers

​		/consumer://10.13.30.97/com.homestead.api.service.goods.ServiceInsurancesService?application=oppo_api_goods&category=consumers&check=false&default.check=false&default.group=release&default.retries=0&default.timeout=6000&dubbo=2.5.3&interface=com.homestead.api.service.goods.ServiceInsurancesService&methods=getServiceInsurancesMap,getAvailableServiceInsurancesMap&pid=25504&revision=2.5.3&side=consumer&timestamp=1584344668508

| 属性        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| ?           | consumer://10.13.30.97/com.homestead.api.service.goods.ServiceInsurancesService |
| application | 应用名（oppo_api_goods）                                     |
| category    | 类型（consumers）                                            |
| check       | 检查，记录失败注册和订阅请求，后台定时重试                   |
| dubbo       | dubbo版本号                                                  |
| interface   | 接口名称                                                     |
| methods     | 方法名称                                                     |
| pid         | 进程号                                                       |
| side        | 消费端或服务端（consumer）                                   |
| timestamp   | 时间戳                                                       |

官方提供：

| 属性        | 对应URL参数         | 类型           | 是否必填 | 缺省值                 | 作用     | 描述                                                         | 兼容性                     |
| ----------- | ------------------- | -------------- | -------- | ---------------------- | -------- | ------------------------------------------------------------ | -------------------------- |
| timeout     | default.timeout     | int            | 可选     | 1000                   | 性能调优 | 远程服务调用超时时间(毫秒)                                   | 1.0.16以上版本             |
| retries     | default.retries     | int            | 可选     | 2                      | 性能调优 | 远程服务调用重试次数，不包括第一次调用，不需要重试请设为0,仅在cluster为failback/failover时有效 | 1.0.16以上版本             |
| loadbalance | default.loadbalance | string         | 可选     | random                 | 性能调优 | 负载均衡策略，可选值：random,roundrobin,leastactive，分别表示：随机，轮询，最少活跃调用 | 1.0.16以上版本             |
| async       | default.async       | boolean        | 可选     | false                  | 性能调优 | 是否缺省异步执行，不可靠异步，只是忽略返回值，不阻塞执行线程 | 2.0.0以上版本              |
| connections | default.connections | int            | 可选     | 100                    | 性能调优 | 每个服务对每个提供者的最大连接数，rmi、http、hessian等短连接协议支持此配置，dubbo协议长连接不支持此配置 | 1.0.16以上版本             |
| generic     | generic             | boolean        | 可选     | false                  | 服务治理 | 是否缺省泛化接口，如果为泛化接口，将返回GenericService       | 2.0.0以上版本              |
| check       | check               | boolean        | 可选     | true                   | 服务治理 | 启动时检查提供者是否存在，true报错，false忽略                | 1.0.16以上版本             |
| proxy       | proxy               | string         | 可选     | javassist              | 性能调优 | 生成动态代理方式，可选：jdk/javassist                        | 2.0.5以上版本              |
| owner       | owner               | string         | 可选     |                        | 服务治理 | 调用服务负责人，用于服务治理，请填写负责人公司邮箱前缀       | 2.0.5以上版本              |
| actives     | default.actives     | int            | 可选     | 0                      | 性能调优 | 每服务消费者每服务每方法最大并发调用数                       | 2.0.5以上版本              |
| cluster     | default.cluster     | string         | 可选     | failover               | 性能调优 | 集群方式，可选：failover/failfast/failsafe/failback/forking  | 2.0.5以上版本              |
| filter      | reference.filter    | string         | 可选     |                        | 性能调优 | 服务消费方远程调用过程拦截器名称，多个名称用逗号分隔         | 2.0.5以上版本              |
| listener    | invoker.listener    | string         | 可选     |                        | 性能调优 | 服务消费方引用服务监听器名称，多个名称用逗号分隔             | 2.0.5以上版本              |
| registry    |                     | string         | 可选     | 缺省向所有registry注册 | 配置关联 | 向指定注册中心注册，在多个注册中心时使用，值为<dubbo:registry>的id属性，多个注册中心ID用逗号分隔，如果不想将该服务注册到任何registry，可将值设为N/A | 2.0.5以上版本              |
| layer       | layer               | string         | 可选     |                        | 服务治理 | 服务调用者所在的分层。如：biz、dao、intl:web、china:acton。  | 2.0.7以上版本              |
| init        | init                | boolean        | 可选     | false                  | 性能调优 | 是否在afterPropertiesSet()时饥饿初始化引用，否则等到有人注入或引用该实例时再初始化。 | 2.0.10以上版本             |
| cache       | cache               | string/boolean | 可选     |                        | 服务治理 | 以调用参数为key，缓存返回结果，可选：lru, threadlocal, jcache等 | Dubbo2.1.0及其以上版本支持 |
| validation  | validation          | boolean        | 可选     |                        | 服务治理 | 是否启用JSR303标准注解验证，如果启用，将对方法参数上的注解进行校验 | Dubbo2.1.0及其以上版本支持 |



### Providers

/dubbo

​	/com.homestead.api.service.goods.ServiceInsurancesService/providers

​		/dubbo://10.13.30.97:20886/com.homestead.api.service.goods.ServiceInsurancesService?anyhost=true&application=oppo_api_goods&default.delay=-1&default.group=release&default.retries=0&default.service.filter=myproviderFilter&default.timeout=4000&delay=-1&dubbo=2.5.3&interface=com.homestead.api.service.goods.ServiceInsurancesService&methods=getServiceInsurancesMap,getAvailableServiceInsurancesMap&pid=27104&revision=2.5.3&serialization=pbmix&side=provider&timestamp=1584344935343

| 属性           | 描述                                                         |
| -------------- | ------------------------------------------------------------ |
| ?              | dubbo://10.13.30.97:20886/com.homestead.api.service.goods.ServiceInsurancesService |
| anyhost        | true                                                         |
| application    | 应用名（oppo_api_goods）                                     |
| delay          | 延迟注册服务时间(毫秒)- ，设为-1时，表示延迟到Spring容器初始化完成时暴露服务 |
| group          |                                                              |
| retries        | 远程服务调用重试次数，不包括第一次调用，不需要重试请设为0    |
| service.filter | 服务提供方远程调用过程拦截器名称，多个名称用逗号分隔         |
| timeout        | 方法调用的超时时间                                           |
| dubbo          | dubbo版本号                                                  |
| interface      | 接口名称                                                     |
| methods        | 方法名称                                                     |
| pid            | 进程号                                                       |
| side           | 消费端或服务端（consumer）                                   |
| timestamp      | 时间戳                                                       |

官方提供：


| 类型            | 是否必填            | 缺省值         | 作用   | 描述                                                         | 兼容性     |                                                              |                |
| --------------- | ------------------- | -------------- | ------ | ------------------------------------------------------------ | ---------- | ------------------------------------------------------------ | -------------- |
| id              |                     | string         | 可选   | dubbo                                                        | 配置关联   | 协议BeanId，可以在<dubbo:service proivder="">中引用此ID      | 1.0.16以上版本 |
| protocol        | <protocol>          | string         | 可选   | dubbo                                                        | 性能调优   | 协议名称                                                     | 1.0.16以上版本 |
| host            | <host>              | string         | 可选   | 自动查找本机IP                                               | 服务发现   | 服务主机名，多网卡选择或指定VIP及域名时使用，为空则自动查找本机IP，建议不要配置，让Dubbo自动获取本机IP | 1.0.16以上版本 |
| threads         | threads             | int            | 可选   | 200                                                          | 性能调优   | 服务线程池大小(固定大小)                                     | 1.0.16以上版本 |
| payload         | payload             | int            | 可选   | 8388608(=8M)                                                 | 性能调优   | 请求及响应数据包大小限制，单位：字节                         | 2.0.0以上版本  |
| path            | <path>              | string         | 可选   |                                                              | 服务发现   | 提供者上下文路径，为服务path的前缀                           | 2.0.0以上版本  |
| server          | server              | string         | 可选   | dubbo协议缺省为netty，http协议缺省为servlet                  | 性能调优   | 协议的服务器端实现类型，比如：dubbo协议的mina,netty等，http协议的jetty,servlet等 | 2.0.0以上版本  |
| client          | client              | string         | 可选   | dubbo协议缺省为netty                                         | 性能调优   | 协议的客户端实现类型，比如：dubbo协议的mina,netty等          | 2.0.0以上版本  |
| codec           | codec               | string         | 可选   | dubbo                                                        | 性能调优   | 协议编码方式                                                 | 2.0.0以上版本  |
| serialization   | serialization       | string         | 可选   | dubbo协议缺省为hessian2，rmi协议缺省为java，http协议缺省为json | 性能调优   | 协议序列化方式，当协议支持多种序列化方式时使用，比如：dubbo协议的dubbo,hessian2,java,compactedjava，以及http协议的json,xml等 | 2.0.5以上版本  |
| default         |                     | boolean        | 可选   | false                                                        | 配置关联   | 是否为缺省协议，用于多协议                                   | 1.0.16以上版本 |
| filter          | service.filter      | string         | 可选   |                                                              | 性能调优   | 服务提供方远程调用过程拦截器名称，多个名称用逗号分隔         | 2.0.5以上版本  |
| listener        | exporter.listener   | string         | 可选   |                                                              | 性能调优   | 服务提供方导出服务监听器名称，多个名称用逗号分隔             | 2.0.5以上版本  |
| threadpool      | threadpool          | string         | 可选   | fixed                                                        | 性能调优   | 线程池类型，可选：fixed/cached/limit(2.5.3以上)/eager(2.6.x以上) | 2.0.5以上版本  |
| accepts         | accepts             | int            | 可选   | 0                                                            | 性能调优   | 服务提供者最大可接受连接数                                   | 2.0.5以上版本  |
| version         | version             | string         | 可选   | 0.0.0                                                        | 服务发现   | 服务版本，建议使用两位数字版本，如：1.0，通常在接口不兼容时版本号才需要升级 | 2.0.5以上版本  |
| group           | group               | string         | 可选   |                                                              | 服务发现   | 服务分组，当一个接口有多个实现，可以用分组区分               | 2.0.5以上版本  |
| delay           | delay               | int            | 可选   | 0                                                            | 性能调优   | 延迟注册服务时间(毫秒)- ，设为-1时，表示延迟到Spring容器初始化完成时暴露服务 | 2.0.5以上版本  |
| timeout         | default.timeout     | int            | 可选   | 1000                                                         | 性能调优   | 远程服务调用超时时间(毫秒)                                   | 2.0.5以上版本  |
| retries         | default.retries     | int            | 可选   | 2                                                            | 性能调优   | 远程服务调用重试次数，不包括第一次调用，不需要重试请设为0    | 2.0.5以上版本  |
| connections     | default.connections | int            | 可选   | 0                                                            | 性能调优   | 对每个提供者的最大连接数，rmi、http、hessian等短连接协议表示限制连接数，dubbo等长连接协表示建立的长连接个数 | 2.0.5以上版本  |
| loadbalance     | default.loadbalance | string         | 可选   | random                                                       | 性能调优   | 负载均衡策略，可选值：random,roundrobin,leastactive，分别表示：随机，轮询，最少活跃调用 | 2.0.5以上版本  |
| async           | default.async       | boolean        | 可选   | false                                                        | 性能调优   | 是否缺省异步执行，不可靠异步，只是忽略返回值，不阻塞执行线程 | 2.0.5以上版本  |
| stub            | stub                | boolean        | 可选   | false                                                        | 服务治理   | 设为true，表示使用缺省代理类名，即：接口名 + Local后缀。     | 2.0.5以上版本  |
| mock            | mock                | boolean        | 可选   | false                                                        | 服务治理   | 设为true，表示使用缺省Mock类名，即：接口名 + Mock后缀。      | 2.0.5以上版本  |
| token           | token               | boolean        | 可选   | false                                                        | 服务治理   | 令牌验证，为空表示不开启，如果为true，表示随机生成动态令牌   | 2.0.5以上版本  |
| registry        | registry            | string         | 可选   | 缺省向所有registry注册                                       | 配置关联   | 向指定注册中心注册，在多个注册中心时使用，值为<dubbo:registry>的id属性，多个注册中心ID用逗号分隔，如果不想将该服务注册到任何registry，可将值设为N/A | 2.0.5以上版本  |
| dynamic         | dynamic             | boolean        | 可选   | true                                                         | 服务治理   | 服务是否动态注册，如果设为false，注册后将显示后disable状态，需人工启用，并且服务提供者停止时，也不会自动取消册，需人工禁用。 | 2.0.5以上版本  |
| accesslog       | accesslog           | string/boolean | 可选   | false                                                        | 服务治理   | 设为true，将向logger中输出访问日志，也可填写访问日志文件路径，直接把访问日志输出到指定文件 | 2.0.5以上版本  |
| owner           | owner               | string         | 可选   |                                                              | 服务治理   | 服务负责人，用于服务治理，请填写负责人公司邮箱前缀           | 2.0.5以上版本  |
| document        | document            | string         | 可选   |                                                              | 服务治理   | 服务文档URL                                                  | 2.0.5以上版本  |
| weight          | weight              | int            | 可选   |                                                              | 性能调优   | 服务权重                                                     | 2.0.5以上版本  |
| executes        | executes            | int            | 可选   | 0                                                            | 性能调优   | 服务提供者每服务每方法最大可并行执行请求数                   | 2.0.5以上版本  |
| actives         | default.actives     | int            | 可选   | 0                                                            | 性能调优   | 每服务消费者每服务每方法最大并发调用数                       | 2.0.5以上版本  |
| proxy           | proxy               | string         | 可选   | javassist                                                    | 性能调优   | 生成动态代理方式，可选：jdk/javassist                        | 2.0.5以上版本  |
| cluster         | default.cluster     | string         | 可选   | failover                                                     | 性能调优   | 集群方式，可选：failover/failfast/failsafe/failback/forking  | 2.0.5以上版本  |
| deprecated      | deprecated          | boolean        | 可选   | false                                                        | 服务治理   | 服务是否过时，如果设为true，消费方引用时将打印服务过时警告error日志 | 2.0.5以上版本  |
| queues          | queues              | int            | 可选   | 0                                                            | 性能调优   | 线程池队列大小，当线程池满时，排队等待执行的队列大小，建议不要设置，当线程池满时应立即失败，重试其它服务提供机器，而不是排队，除非有特殊需求。 | 2.0.5以上版本  |
| charset         | charset             | string         | 可选   | UTF-8                                                        | 性能调优   | 序列化编码                                                   | 2.0.5以上版本  |
| buffer          | buffer              | int            | 可选   | 8192                                                         | 性能调优   | 网络读写缓冲区大小                                           | 2.0.5以上版本  |
| iothreads       | iothreads           | int            | 可选   | CPU + 1                                                      | 性能调优   | IO线程池，接收网络读写中断，以及序列化和反序列化，不处理业务，业务线程池参见threads配置，此线程池和CPU相关，不建议配置。 | 2.0.5以上版本  |
| telnet          | telnet              | string         | 可选   |                                                              | 服务治理   | 所支持的telnet命令，多个命令用逗号分隔                       | 2.0.5以上版本  |
| <dubbo:service> | contextpath         | contextpath    | String | 可选                                                         | 缺省为空串 | 服务治理                                                     |                |
| layer           | layer               | string         | 可选   |                                                              | 服务治理   | 服务提供者所在的分层。如：biz、dao、intl:web、china:acton。  | 2.0.7以上版本  |



### Routers

/dubbo

​	/com.example.dubbo.service.CityService/routers

​		/route://0.0.0.0/com.example.dubbo.service.CityService?category=routers&dynamic=false&enabled=true&force=false&name=cityservice&priority=10&router=condition&rule=method+=+findCityByName+&+consumer.host+=+192.168.198.1+=>+provider.port+=+20881+&+provider.port+!=+20880&runtime=false


​		/route://0.0.0.0/com.example.dubbo.service.CityService?category=routers&dynamic=false&enabled=true&force=true&name=com.example.dubbo.service.CityService+blackwhitelist&priority=0&router=condition&rule=consumer.host=192.168.198.1=>false&runtime=false


| 属性     | 描述                                                         |
| :------- | :----------------------------------------------------------- |
| category | 类型                                                         |
| dynamic  | 是否动态调整，false表示需要手动调整                          |
| enabled  | 是否启动                                                     |
| force    | 是否强制，false表示，如果没有匹配到则调用其它可调用的服务    |
| name     | 路由名称                                                     |
| priority | 优先级                                                       |
| router   | condition符合条件则路由/路由规则IP为192.168.198.1的消费者禁止访问 |


### Configurators

/dubbo

​	/com.example.dubbo.service.CityService/configurators

​		/override://0.0.0.0/com.example.dubbo.service.CityService?category=configurators&dynamic=false&enabled=true&loadbalance=random


​		/override://192.168.198.1:20880/com.example.dubbo.service.CityService?category=configurators&dynamic=false&enabled=true&weight=200

| 属性        | 描述                                |
| :---------- | :---------------------------------- |
| category    | 类型                                |
| dynamic     | 是否动态调整，false表示需要手动调整 |
| enabled     | 是否启动                            |
| loadbalance | 负载均衡策略                        |
| weight      | 权重                                |