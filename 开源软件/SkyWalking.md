# APM

Application Performance Monitor（应用监控）

## 简述

skywalking主要分为3个部分

（1）上报端：即客户端，客户端采用字节码注入技术实现客户端监控数据的自动采集和上报，客户端接入的时候需要自己配置好数据上报的频率和上报目标服务地址

（2）oap服务端：即数据处理端，主要接收各个客户端的上报服务进行聚合处理分析

（3）web展示端：即前端服务，主要是读取oap服务的数据处理后



## 配置简述

skywalking配置文件主要分为主配置文件和前端配置文件

（1）前端配置文件 webapp/webapp.yml，只需要配置好自身端口号和采集服务的 

```yaml
server:
  port: 8090

collector:
  path: /graphql
  ribbon:
    ReadTimeout: 10000
    # Point to all backend's restHost:restPort, split by ,
    listOfServers: 127.0.0.1:12800
```



（2）主配置文件 config/application.yml，主配置文件可以配置skywalking的集群模式，存储方式，数据处理方式，存储机制，告警机制，监控项等，一遍只要关注集群模式，存储方式即可，一般利用zk集群搭建集群模式，利用ES做存储。

```yaml
cluster:
  selector: ${SW_CLUSTER:standalone}
  standalone:
  # Please check your ZooKeeper is 3.5+, However, it is also compatible with ZooKeeper 3.4.x. Replace the ZooKeeper 3.5+
  # library the oap-libs folder with your ZooKeeper 3.4.x library.
  zookeeper:
    nameSpace: ${SW_NAMESPACE:""}
    hostPort: ${SW_CLUSTER_ZK_HOST_PORT:localhost:2181}
    # Retry Policy
    baseSleepTimeMs: ${SW_CLUSTER_ZK_SLEEP_TIME:1000} # initial amount of time to wait between retries
    maxRetries: ${SW_CLUSTER_ZK_MAX_RETRIES:3} # max number of times to retry
    # Enable ACL
    enableACL: ${SW_ZK_ENABLE_ACL:false} # disable ACL in default
    schema: ${SW_ZK_SCHEMA:digest} # only support digest schema
    expression: ${SW_ZK_EXPRESSION:skywalking:skywalking}
  kubernetes:
    namespace: ${SW_CLUSTER_K8S_NAMESPACE:default}
    labelSelector: ${SW_CLUSTER_K8S_LABEL:app=collector,release=skywalking}
    uidEnvName: ${SW_CLUSTER_K8S_UID:SKYWALKING_COLLECTOR_UID}
  consul:
    serviceName: ${SW_SERVICE_NAME:"SkyWalking_OAP_Cluster"}
    # Consul cluster nodes, example: 10.0.0.1:8500,10.0.0.2:8500,10.0.0.3:8500
    hostPort: ${SW_CLUSTER_CONSUL_HOST_PORT:localhost:8500}
    aclToken: ${SW_CLUSTER_CONSUL_ACLTOKEN:""}
  etcd:
    serviceName: ${SW_SERVICE_NAME:"SkyWalking_OAP_Cluster"}
    # etcd cluster nodes, example: 10.0.0.1:2379,10.0.0.2:2379,10.0.0.3:2379
    hostPort: ${SW_CLUSTER_ETCD_HOST_PORT:localhost:2379}
  nacos:
    serviceName: ${SW_SERVICE_NAME:"SkyWalking_OAP_Cluster"}
    hostPort: ${SW_CLUSTER_NACOS_HOST_PORT:localhost:8848}
    # Nacos Configuration namespace
    namespace: ${SW_CLUSTER_NACOS_NAMESPACE:"public"}
    # Nacos auth username
    username: ${SW_CLUSTER_NACOS_USERNAME:""}
    password: ${SW_CLUSTER_NACOS_PASSWORD:""}
    # Nacos auth accessKey
    accessKey: ${SW_CLUSTER_NACOS_ACCESSKEY:""}
    secretKey: ${SW_CLUSTER_NACOS_SECRETKEY:""}
core:
  selector: ${SW_CORE:default}
  default:
    # Mixed: Receive agent data, Level 1 aggregate, Level 2 aggregate
    # Receiver: Receive agent data, Level 1 aggregate
    # Aggregator: Level 2 aggregate
    role: ${SW_CORE_ROLE:Mixed} # Mixed/Receiver/Aggregator
    restHost: ${SW_CORE_REST_HOST:0.0.0.0}
    restPort: ${SW_CORE_REST_PORT:12800}
    restContextPath: ${SW_CORE_REST_CONTEXT_PATH:/}
    restMinThreads: ${SW_CORE_REST_JETTY_MIN_THREADS:1}
    restMaxThreads: ${SW_CORE_REST_JETTY_MAX_THREADS:200}
    restIdleTimeOut: ${SW_CORE_REST_JETTY_IDLE_TIMEOUT:30000}
    restAcceptorPriorityDelta: ${SW_CORE_REST_JETTY_DELTA:0}
    restAcceptQueueSize: ${SW_CORE_REST_JETTY_QUEUE_SIZE:0}
    gRPCHost: ${SW_CORE_GRPC_HOST:0.0.0.0}
    gRPCPort: ${SW_CORE_GRPC_PORT:11800}
    maxConcurrentCallsPerConnection: ${SW_CORE_GRPC_MAX_CONCURRENT_CALL:0}
    maxMessageSize: ${SW_CORE_GRPC_MAX_MESSAGE_SIZE:0}
    gRPCThreadPoolQueueSize: ${SW_CORE_GRPC_POOL_QUEUE_SIZE:0}
    gRPCThreadPoolSize: ${SW_CORE_GRPC_THREAD_POOL_SIZE:0}
    gRPCSslEnabled: ${SW_CORE_GRPC_SSL_ENABLED:false}
    gRPCSslKeyPath: ${SW_CORE_GRPC_SSL_KEY_PATH:""}
    gRPCSslCertChainPath: ${SW_CORE_GRPC_SSL_CERT_CHAIN_PATH:""}
    gRPCSslTrustedCAPath: ${SW_CORE_GRPC_SSL_TRUSTED_CA_PATH:""}
    downsampling:
      - Hour
      - Day
    # Set a timeout on metrics data. After the timeout has expired, the metrics data will automatically be deleted.
    enableDataKeeperExecutor: ${SW_CORE_ENABLE_DATA_KEEPER_EXECUTOR:true} # Turn it off then automatically metrics data delete will be close.
    dataKeeperExecutePeriod: ${SW_CORE_DATA_KEEPER_EXECUTE_PERIOD:5} # How often the data keeper executor runs periodically, unit is minute
    recordDataTTL: ${SW_CORE_RECORD_DATA_TTL:3} # Unit is day
    metricsDataTTL: ${SW_CORE_METRICS_DATA_TTL:7} # Unit is day
    # Cache metrics data for 1 minute to reduce database queries, and if the OAP cluster changes within that minute,
    # the metrics may not be accurate within that minute.
    enableDatabaseSession: ${SW_CORE_ENABLE_DATABASE_SESSION:true}
    topNReportPeriod: ${SW_CORE_TOPN_REPORT_PERIOD:10} # top_n record worker report cycle, unit is minute
    # Extra model column are the column defined by in the codes, These columns of model are not required logically in aggregation or further query,
    # and it will cause more load for memory, network of OAP and storage.
    # But, being activated, user could see the name in the storage entities, which make users easier to use 3rd party tool, such as Kibana->ES, to query the data by themselves.
    activeExtraModelColumns: ${SW_CORE_ACTIVE_EXTRA_MODEL_COLUMNS:false}
    # The max length of service + instance names should be less than 200
    serviceNameMaxLength: ${SW_SERVICE_NAME_MAX_LENGTH:70}
    instanceNameMaxLength: ${SW_INSTANCE_NAME_MAX_LENGTH:70}
    # The max length of service + endpoint names should be less than 240
    endpointNameMaxLength: ${SW_ENDPOINT_NAME_MAX_LENGTH:150}
    # Define the set of span tag keys, which should be searchable through the GraphQL.
    searchableTracesTags: ${SW_SEARCHABLE_TAG_KEYS:http.method,status_code,db.type,db.instance,mq.queue,mq.topic,mq.broker}
storage:
  selector: ${SW_STORAGE:h2}
  elasticsearch:
    nameSpace: ${SW_NAMESPACE:""}
    clusterNodes: ${SW_STORAGE_ES_CLUSTER_NODES:localhost:9200}
    protocol: ${SW_STORAGE_ES_HTTP_PROTOCOL:"http"}
    user: ${SW_ES_USER:""}
    password: ${SW_ES_PASSWORD:""}
    trustStorePath: ${SW_STORAGE_ES_SSL_JKS_PATH:""}
    trustStorePass: ${SW_STORAGE_ES_SSL_JKS_PASS:""}
    secretsManagementFile: ${SW_ES_SECRETS_MANAGEMENT_FILE:""} # Secrets management file in the properties format includes the username, password, which are managed by 3rd party tool.
    dayStep: ${SW_STORAGE_DAY_STEP:1} # Represent the number of days in the one minute/hour/day index.
    indexShardsNumber: ${SW_STORAGE_ES_INDEX_SHARDS_NUMBER:1} # Shard number of new indexes
    indexReplicasNumber: ${SW_STORAGE_ES_INDEX_REPLICAS_NUMBER:1} # Replicas number of new indexes
    # Super data set has been defined in the codes, such as trace segments.The following 3 config would be improve es performance when storage super size data in es.
    superDatasetDayStep: ${SW_SUPERDATASET_STORAGE_DAY_STEP:-1} # Represent the number of days in the super size dataset record index, the default value is the same as dayStep when the value is less than 0
    superDatasetIndexShardsFactor: ${SW_STORAGE_ES_SUPER_DATASET_INDEX_SHARDS_FACTOR:5} #  This factor provides more shards for the super data set, shards number = indexShardsNumber * superDatasetIndexShardsFactor. Also, this factor effects Zipkin and Jaeger traces.
    superDatasetIndexReplicasNumber: ${SW_STORAGE_ES_SUPER_DATASET_INDEX_REPLICAS_NUMBER:0} # Represent the replicas number in the super size dataset record index, the default value is 0.
    bulkActions: ${SW_STORAGE_ES_BULK_ACTIONS:1000} # Execute the async bulk record data every ${SW_STORAGE_ES_BULK_ACTIONS} requests
    syncBulkActions: ${SW_STORAGE_ES_SYNC_BULK_ACTIONS:50000} # Execute the sync bulk metrics data every ${SW_STORAGE_ES_SYNC_BULK_ACTIONS} requests
    flushInterval: ${SW_STORAGE_ES_FLUSH_INTERVAL:10} # flush the bulk every 10 seconds whatever the number of requests
    concurrentRequests: ${SW_STORAGE_ES_CONCURRENT_REQUESTS:2} # the number of concurrent requests
    resultWindowMaxSize: ${SW_STORAGE_ES_QUERY_MAX_WINDOW_SIZE:10000}
    metadataQueryMaxSize: ${SW_STORAGE_ES_QUERY_MAX_SIZE:5000}
    segmentQueryMaxSize: ${SW_STORAGE_ES_QUERY_SEGMENT_SIZE:200}
    profileTaskQueryMaxSize: ${SW_STORAGE_ES_QUERY_PROFILE_TASK_SIZE:200}
    advanced: ${SW_STORAGE_ES_ADVANCED:""}
  elasticsearch7:
    nameSpace: ${SW_NAMESPACE:""}
    clusterNodes: ${SW_STORAGE_ES_CLUSTER_NODES:localhost:9200}
    protocol: ${SW_STORAGE_ES_HTTP_PROTOCOL:"http"}
    trustStorePath: ${SW_STORAGE_ES_SSL_JKS_PATH:""}
    trustStorePass: ${SW_STORAGE_ES_SSL_JKS_PASS:""}
    dayStep: ${SW_STORAGE_DAY_STEP:1} # Represent the number of days in the one minute/hour/day index.
    indexShardsNumber: ${SW_STORAGE_ES_INDEX_SHARDS_NUMBER:1} # Shard number of new indexes
    indexReplicasNumber: ${SW_STORAGE_ES_INDEX_REPLICAS_NUMBER:1} # Replicas number of new indexes
    # Super data set has been defined in the codes, such as trace segments.The following 3 config would be improve es performance when storage super size data in es.
    superDatasetDayStep: ${SW_SUPERDATASET_STORAGE_DAY_STEP:-1} # Represent the number of days in the super size dataset record index, the default value is the same as dayStep when the value is less than 0
    superDatasetIndexShardsFactor: ${SW_STORAGE_ES_SUPER_DATASET_INDEX_SHARDS_FACTOR:5} #  This factor provides more shards for the super data set, shards number = indexShardsNumber * superDatasetIndexShardsFactor. Also, this factor effects Zipkin and Jaeger traces.
    superDatasetIndexReplicasNumber: ${SW_STORAGE_ES_SUPER_DATASET_INDEX_REPLICAS_NUMBER:0} # Represent the replicas number in the super size dataset record index, the default value is 0.
    user: ${SW_ES_USER:""}
    password: ${SW_ES_PASSWORD:""}
    secretsManagementFile: ${SW_ES_SECRETS_MANAGEMENT_FILE:""} # Secrets management file in the properties format includes the username, password, which are managed by 3rd party tool.
    bulkActions: ${SW_STORAGE_ES_BULK_ACTIONS:1000} # Execute the async bulk record data every ${SW_STORAGE_ES_BULK_ACTIONS} requests
    syncBulkActions: ${SW_STORAGE_ES_SYNC_BULK_ACTIONS:50000} # Execute the sync bulk metrics data every ${SW_STORAGE_ES_SYNC_BULK_ACTIONS} requests
    flushInterval: ${SW_STORAGE_ES_FLUSH_INTERVAL:10} # flush the bulk every 10 seconds whatever the number of requests
    concurrentRequests: ${SW_STORAGE_ES_CONCURRENT_REQUESTS:2} # the number of concurrent requests
    resultWindowMaxSize: ${SW_STORAGE_ES_QUERY_MAX_WINDOW_SIZE:10000}
    metadataQueryMaxSize: ${SW_STORAGE_ES_QUERY_MAX_SIZE:5000}
    segmentQueryMaxSize: ${SW_STORAGE_ES_QUERY_SEGMENT_SIZE:200}
    profileTaskQueryMaxSize: ${SW_STORAGE_ES_QUERY_PROFILE_TASK_SIZE:200}
    advanced: ${SW_STORAGE_ES_ADVANCED:""}
  h2:
    driver: ${SW_STORAGE_H2_DRIVER:org.h2.jdbcx.JdbcDataSource}
    url: ${SW_STORAGE_H2_URL:jdbc:h2:mem:skywalking-oap-db}
    user: ${SW_STORAGE_H2_USER:sa}
    metadataQueryMaxSize: ${SW_STORAGE_H2_QUERY_MAX_SIZE:5000}
    maxSizeOfArrayColumn: ${SW_STORAGE_MAX_SIZE_OF_ARRAY_COLUMN:20}
    numOfSearchableValuesPerTag: ${SW_STORAGE_NUM_OF_SEARCHABLE_VALUES_PER_TAG:2}
  mysql:
    properties:
      jdbcUrl: ${SW_JDBC_URL:"jdbc:mysql://localhost:3306/swtest"}
      dataSource.user: ${SW_DATA_SOURCE_USER:root}
      dataSource.password: ${SW_DATA_SOURCE_PASSWORD:root@1234}
      dataSource.cachePrepStmts: ${SW_DATA_SOURCE_CACHE_PREP_STMTS:true}
      dataSource.prepStmtCacheSize: ${SW_DATA_SOURCE_PREP_STMT_CACHE_SQL_SIZE:250}
      dataSource.prepStmtCacheSqlLimit: ${SW_DATA_SOURCE_PREP_STMT_CACHE_SQL_LIMIT:2048}
      dataSource.useServerPrepStmts: ${SW_DATA_SOURCE_USE_SERVER_PREP_STMTS:true}
    metadataQueryMaxSize: ${SW_STORAGE_MYSQL_QUERY_MAX_SIZE:5000}
    maxSizeOfArrayColumn: ${SW_STORAGE_MAX_SIZE_OF_ARRAY_COLUMN:20}
    numOfSearchableValuesPerTag: ${SW_STORAGE_NUM_OF_SEARCHABLE_VALUES_PER_TAG:2}

agent-analyzer:
  selector: ${SW_AGENT_ANALYZER:default}
  default:
    sampleRate: ${SW_TRACE_SAMPLE_RATE:10000} # The sample rate precision is 1/10000. 10000 means 100% sample in default.
    slowDBAccessThreshold: ${SW_SLOW_DB_THRESHOLD:default:200,mongodb:100} # The slow database access thresholds. Unit ms.
    forceSampleErrorSegment: ${SW_FORCE_SAMPLE_ERROR_SEGMENT:true} # When sampling mechanism active, this config can open(true) force save some error segment. true is default.
    segmentStatusAnalysisStrategy: ${SW_SEGMENT_STATUS_ANALYSIS_STRATEGY:FROM_SPAN_STATUS} # Determine the final segment status from the status of spans. Available values are `FROM_SPAN_STATUS` , `FROM_ENTRY_SPAN` and `FROM_FIRST_SPAN`. `FROM_SPAN_STATUS` represents the segment status would be error if any span is in error status. `FROM_ENTRY_SPAN` means the segment status would be determined by the status of entry spans only. `FROM_FIRST_SPAN` means the segment status would be determined by the status of the first span only.
    # Nginx and Envoy agents can't get the real remote address.
    # Exit spans with the component in the list would not generate the client-side instance relation metrics.
    noUpstreamRealAddressAgents: ${SW_NO_UPSTREAM_REAL_ADDRESS:6000,9000}

receiver-sharing-server:
  selector: ${SW_RECEIVER_SHARING_SERVER:default}
  default:
    # For Jetty server
    host: ${SW_RECEIVER_JETTY_HOST:0.0.0.0}
    contextPath: ${SW_RECEIVER_JETTY_CONTEXT_PATH:/}
    jettyMinThreads: ${SW_RECEIVER_SHARING_JETTY_MIN_THREADS:1}
    jettyMaxThreads: ${SW_RECEIVER_SHARING_JETTY_MAX_THREADS:200}
    jettyIdleTimeOut: ${SW_RECEIVER_SHARING_JETTY_IDLE_TIMEOUT:30000}
    jettyAcceptorPriorityDelta: ${SW_RECEIVER_SHARING_JETTY_DELTA:0}
    jettyAcceptQueueSize: ${SW_RECEIVER_SHARING_JETTY_QUEUE_SIZE:0}
    # For gRPC server
    gRPCHost: ${SW_RECEIVER_GRPC_HOST:0.0.0.0}
    gRPCPort: ${SW_RECEIVER_GRPC_PORT:0}
    maxConcurrentCallsPerConnection: ${SW_RECEIVER_GRPC_MAX_CONCURRENT_CALL:0}
    maxMessageSize: ${SW_RECEIVER_GRPC_MAX_MESSAGE_SIZE:0}
    gRPCThreadPoolQueueSize: ${SW_RECEIVER_GRPC_POOL_QUEUE_SIZE:0}
    gRPCThreadPoolSize: ${SW_RECEIVER_GRPC_THREAD_POOL_SIZE:0}
    gRPCSslEnabled: ${SW_RECEIVER_GRPC_SSL_ENABLED:false}
    gRPCSslKeyPath: ${SW_RECEIVER_GRPC_SSL_KEY_PATH:""}
    gRPCSslCertChainPath: ${SW_RECEIVER_GRPC_SSL_CERT_CHAIN_PATH:""}
    authentication: ${SW_AUTHENTICATION:""}
receiver-register:
  selector: ${SW_RECEIVER_REGISTER:default}
  default:

receiver-trace:
  selector: ${SW_RECEIVER_TRACE:default}
  default:

receiver-jvm:
  selector: ${SW_RECEIVER_JVM:default}
  default:

receiver-clr:
  selector: ${SW_RECEIVER_CLR:default}
  default:

receiver-profile:
  selector: ${SW_RECEIVER_PROFILE:default}
  default:

service-mesh:
  selector: ${SW_SERVICE_MESH:default}
  default:

istio-telemetry:
  selector: ${SW_ISTIO_TELEMETRY:default}
  default:

envoy-metric:
  selector: ${SW_ENVOY_METRIC:default}
  default:
    acceptMetricsService: ${SW_ENVOY_METRIC_SERVICE:true}
    alsHTTPAnalysis: ${SW_ENVOY_METRIC_ALS_HTTP_ANALYSIS:""}

prometheus-fetcher:
  selector: ${SW_PROMETHEUS_FETCHER:default}
  default:
    active: ${SW_PROMETHEUS_FETCHER_ACTIVE:false}

kafka-fetcher:
  selector: ${SW_KAFKA_FETCHER:-}
  default:
    bootstrapServers: ${SW_KAFKA_FETCHER_SERVERS:localhost:9092}
    partitions: ${SW_KAFKA_FETCHER_PARTITIONS:3}
    replicationFactor: ${SW_KAFKA_FETCHER_PARTITIONS_FACTOR:2}
    enableMeterSystem: ${SW_KAFKA_FETCHER_ENABLE_METER_SYSTEM:false}
    isSharding: ${SW_KAFKA_FETCHER_IS_SHARDING:false}
    consumePartitions: ${SW_KAFKA_FETCHER_CONSUME_PARTITIONS:""}

receiver-meter:
  selector: ${SW_RECEIVER_METER:-}
  default:

receiver-oc:
  selector: ${SW_OC_RECEIVER:-}
  default:
    gRPCHost: ${SW_OC_RECEIVER_GRPC_HOST:0.0.0.0}
    gRPCPort: ${SW_OC_RECEIVER_GRPC_PORT:55678}

receiver_zipkin:
  selector: ${SW_RECEIVER_ZIPKIN:-}
  default:
    host: ${SW_RECEIVER_ZIPKIN_HOST:0.0.0.0}
    port: ${SW_RECEIVER_ZIPKIN_PORT:9411}
    contextPath: ${SW_RECEIVER_ZIPKIN_CONTEXT_PATH:/}
    jettyMinThreads: ${SW_RECEIVER_ZIPKIN_JETTY_MIN_THREADS:1}
    jettyMaxThreads: ${SW_RECEIVER_ZIPKIN_JETTY_MAX_THREADS:200}
    jettyIdleTimeOut: ${SW_RECEIVER_ZIPKIN_JETTY_IDLE_TIMEOUT:30000}
    jettyAcceptorPriorityDelta: ${SW_RECEIVER_ZIPKIN_JETTY_DELTA:0}
    jettyAcceptQueueSize: ${SW_RECEIVER_ZIPKIN_QUEUE_SIZE:0}

receiver_jaeger:
  selector: ${SW_RECEIVER_JAEGER:-}
  default:
    gRPCHost: ${SW_RECEIVER_JAEGER_HOST:0.0.0.0}
    gRPCPort: ${SW_RECEIVER_JAEGER_PORT:14250}

receiver-browser:
  selector: ${SW_RECEIVER_BROWSER:default}
  default:
    # The sample rate precision is 1/10000. 10000 means 100% sample in default.
    sampleRate: ${SW_RECEIVER_BROWSER_SAMPLE_RATE:10000}

query:
  selector: ${SW_QUERY:graphql}
  graphql:
    path: ${SW_QUERY_GRAPHQL_PATH:/graphql}

alarm:
  selector: ${SW_ALARM:default}
  default:

telemetry:
  selector: ${SW_TELEMETRY:none}
  none:
  prometheus:
    host: ${SW_TELEMETRY_PROMETHEUS_HOST:0.0.0.0}
    port: ${SW_TELEMETRY_PROMETHEUS_PORT:1234}
    sslEnabled: ${SW_TELEMETRY_PROMETHEUS_SSL_ENABLED:false}
    sslKeyPath: ${SW_TELEMETRY_PROMETHEUS_SSL_KEY_PATH:""}
    sslCertChainPath: ${SW_TELEMETRY_PROMETHEUS_SSL_CERT_CHAIN_PATH:""}

configuration:
  selector: ${SW_CONFIGURATION:none}
  none:
  grpc:
    host: ${SW_DCS_SERVER_HOST:""}
    port: ${SW_DCS_SERVER_PORT:80}
    clusterName: ${SW_DCS_CLUSTER_NAME:SkyWalking}
    period: ${SW_DCS_PERIOD:20}
  apollo:
    apolloMeta: ${SW_CONFIG_APOLLO:http://106.12.25.204:8080}
    apolloCluster: ${SW_CONFIG_APOLLO_CLUSTER:default}
    apolloEnv: ${SW_CONFIG_APOLLO_ENV:""}
    appId: ${SW_CONFIG_APOLLO_APP_ID:skywalking}
    period: ${SW_CONFIG_APOLLO_PERIOD:5}
  zookeeper:
    period: ${SW_CONFIG_ZK_PERIOD:60} # Unit seconds, sync period. Default fetch every 60 seconds.
    nameSpace: ${SW_CONFIG_ZK_NAMESPACE:/default}
    hostPort: ${SW_CONFIG_ZK_HOST_PORT:localhost:2181}
    # Retry Policy
    baseSleepTimeMs: ${SW_CONFIG_ZK_BASE_SLEEP_TIME_MS:1000} # initial amount of time to wait between retries
    maxRetries: ${SW_CONFIG_ZK_MAX_RETRIES:3} # max number of times to retry
  etcd:
    period: ${SW_CONFIG_ETCD_PERIOD:60} # Unit seconds, sync period. Default fetch every 60 seconds.
    group: ${SW_CONFIG_ETCD_GROUP:skywalking}
    serverAddr: ${SW_CONFIG_ETCD_SERVER_ADDR:localhost:2379}
    clusterName: ${SW_CONFIG_ETCD_CLUSTER_NAME:default}
  consul:
    # Consul host and ports, separated by comma, e.g. 1.2.3.4:8500,2.3.4.5:8500
    hostAndPorts: ${SW_CONFIG_CONSUL_HOST_AND_PORTS:1.2.3.4:8500}
    # Sync period in seconds. Defaults to 60 seconds.
    period: ${SW_CONFIG_CONSUL_PERIOD:60}
    # Consul aclToken
    aclToken: ${SW_CONFIG_CONSUL_ACL_TOKEN:""}
  k8s-configmap:
    period: ${SW_CONFIG_CONFIGMAP_PERIOD:60}
    namespace: ${SW_CLUSTER_K8S_NAMESPACE:default}
    labelSelector: ${SW_CLUSTER_K8S_LABEL:app=collector,release=skywalking}
  nacos:
    # Nacos Server Host
    serverAddr: ${SW_CONFIG_NACOS_SERVER_ADDR:127.0.0.1}
    # Nacos Server Port
    port: ${SW_CONFIG_NACOS_SERVER_PORT:8848}
    # Nacos Configuration Group
    group: ${SW_CONFIG_NACOS_SERVER_GROUP:skywalking}
    # Nacos Configuration namespace
    namespace: ${SW_CONFIG_NACOS_SERVER_NAMESPACE:}
    # Unit seconds, sync period. Default fetch every 60 seconds.
    period: ${SW_CONFIG_NACOS_PERIOD:60}
    # Nacos auth username
    username: ${SW_CONFIG_NACOS_USERNAME:""}
    password: ${SW_CONFIG_NACOS_PASSWORD:""}
    # Nacos auth accessKey
    accessKey: ${SW_CONFIG_NACOS_ACCESSKEY:""}
    secretKey: ${SW_CONFIG_NACOS_SECRETKEY:""}

exporter:
  selector: ${SW_EXPORTER:-}
  grpc:
    targetHost: ${SW_EXPORTER_GRPC_HOST:127.0.0.1}
    targetPort: ${SW_EXPORTER_GRPC_PORT:9870}

health-checker:
  selector: ${SW_HEALTH_CHECKER:-}
  default:
    checkIntervalSeconds: ${SW_HEALTH_CHECKER_INTERVAL_SECONDS:5}
```



## 安装脚本与资源

skywalking安装包中有三种安装启动方式，对应着./bin目录下面有3个shell脚本，startup.sh、oapService.sh、webappService.sh

（1）./bin/startup.sh   混合安装，即将skywalking的前端UI服务和后端采集服务一起安装在一台机器上面

（2）./bin/oapService.sh   单独安装采集服务，用于接收客户端上报的采集数据

（3）./bin/webappService.sh   单独安装前端UI服务，用户展示oap服务采集且经过处理后的监控信息




![SkyWalking启动](.\image\SkyWalking目录.jpg)



![SkyWalking启动](.\image\SkyWalking启动.jpg)



## 安装步骤

（1）将安装包导入所有机器

（2）修改配置文件config/application.yml 中`cluster:zookeeper:hostPort`的值为正式环境的zookeeper集群

（3）修改配置文件config/application.yml 中`storage:elasticsearch:clusterNodes`的值为正式环境的ES集群

（4）启动OAP服务脚本./bin/oapService，查看日志看看是否有异常，./logs/skywalking-oap-server.log，无异常则oap服务启动成。

（5）修改前端服务机器的配置文件webapp/webapp.yml，将`collector:ribbon:listOfServers`:的IP值改为OAP服务的IP，端口不变

（6）启动前端服务脚本./bin/webappService.sh，查看日志./logs/webapp.log，无异常则启动成功

（7）访问前端服务IP:8080，看到UI界面则部署成功。



## agent

agent探针

```
+-- agent
    +-- activations
         apm-toolkit-log4j-1.x-activation.jar
         apm-toolkit-log4j-2.x-activation.jar
         apm-toolkit-logback-1.x-activation.jar
         ...
    +-- config  配置文件
         agent.config  
    +-- plugins  插件
         apm-dubbo-plugin.jar
         apm-feign-default-http-9.x.jar
         apm-httpClient-4.x-plugin.jar
         .....
    +-- optional-plugins  可选插件
         apm-gson-2.x-plugin.jar
         .....
    +-- bootstrap-plugins
         jdk-http-plugin.jar
         .....
    +-- logs
    skywalking-agent.jar
```






```xml
<!-- 字节码 byte-buddy start -->
<dependency>
    <groupId>net.bytebuddy</groupId>
    <artifactId>byte-buddy</artifactId>
    <version>1.9.2</version>
</dependency>

<dependency>
    <groupId>net.bytebuddy</groupId>
    <artifactId>byte-buddy-agent</artifactId>
    <version>1.9.2</version>
</dependency>
<!-- 字节码 byte-buddy end -->


<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>3.1.0</version>
    <configuration>
        <archive>
            <!-- 自动添加META-INF/MANIFEST.MF -->
            <manifest>
                <addClasspath>true</addClasspath>
            </manifest>
            <manifestEntries>
                <Premain-Class>com.myz.buddy.PreMainAgent</Premain-Class>
                <Agent-Class>com.myz.buddy.PreMainAgent</Agent-Class>
                <Can-Redefine-Classes>true</Can-Redefine-Classes>
                <Can-Retransform-Classes>true</Can-Retransform-Classes>
            </manifestEntries>
        </archive>
    </configuration>
</plugin>
```



OpenTracing

Trace（追踪）：由多个Span组成

Span（跨度）：系统中具有开始时间和执行时长的逻辑运行单元

```
单个 Trace 中，span 间的因果关系
 
 
        [Span A]  ←←←(the root span)
            |
     +------+------+
     |             |
 [Span B]      [Span C] ←←←(Span C 是 Span A 的孩子节点, ChildOf)
     |             |
 [Span D]      +---+-------+
               |           |
           [Span E]    [Span F] >>> [Span G] >>> [Span H]
                                       ↑
                                       ↑
                                       ↑
					(Span G 在 Span F 后被调用, FollowsFrom)


单个 Trace 中，span 间的时间关系
 
 
––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–> time
 
 [Span A···················································]
   [Span B··············································]
      [Span D··········································]
    [Span C········································]
         [Span E·······]        [Span F··] [Span G··] [Span H··]
```

操作名称

起始时间

结束时间

Span Tag，一组键值对构成的 Span 标签集合。键值对中，键必须为 string，值可以是字符串，布尔，或者数字类型

Span Log，一组 span 的日志集合

SpanContext，Span 上下文对象

References(Span间关系)



Log

Tags





### 项目启动参数：

（探针配置 > 系统配置 > 系统环境变量 > 配置文件）

```
-javaagent:E:\WorkWare\Apache-skywalking\apache-skywalking-apm-bin\agent\skywalking-agent.jar
-Dskywalking.agent.service_name=jiangsu-bid-service
-Dskywalking.collector.backend_service=192.168.107.110:11800

或者
-javaagent:E:\WorkWare\Apache-skywalking\apache-skywalking-apm-bin\agent\skywalking-agent.jar=agent.service_name=jiangsu-bid-service,collector.backend_service=192.168.107.110:11800
```

> -Dskywalking.agent.service_name=jiangsu-bid-service  => agent.service_name=${SW_AGENT_NAME:Your_ApplicationName}
>
> -Dskywalking.collector.backend_service=192.168.107.110:11800 => collector.backend_service=${SW_AGENT_COLLECTOR_BACKEND_SERVICES:127.0.0.1:11800}

/apache-skywalking/agent/agent/config/agent.config

```
# The agent namespace 指定命名空间
# agent.namespace=${SW_AGENT_NAMESPACE:default-namespace}

# The service name in UI 在UI页面上展示的服务名称
agent.service_name=${SW_AGENT_NAME:Your_ApplicationName}

# The number of sampled traces per 3 seconds
# Negative or zero means off, by default
# agent.sample_n_per_3_secs=${SW_AGENT_SAMPLE:-1}

# Authentication active is based on backend setting, see application.yml for more details.
# agent.authentication = ${SW_AGENT_AUTHENTICATION:xxxx}

# The max amount of spans in a single segment.
# Through this config item, SkyWalking keep your application memory cost estimated.
# agent.span_limit_per_segment=${SW_AGENT_SPAN_LIMIT:150}

# Ignore the segments if their operation names end with these suffix.
# agent.ignore_suffix=${SW_AGENT_IGNORE_SUFFIX:.jpg,.jpeg,.js,.css,.png,.bmp,.gif,.ico,.mp3,.mp4,.html,.svg}

# If true, SkyWalking agent will save all instrumented classes files in `/debugging` folder.
# SkyWalking team may ask for these files in order to resolve compatible problem.
# agent.is_open_debugging_class = ${SW_AGENT_OPEN_DEBUG:true}

# If true, SkyWalking agent will cache all instrumented classes files to memory or disk files (decided by class cache mode),
# allow other javaagent to enhance those classes that enhanced by SkyWalking agent.
# agent.is_cache_enhanced_class = ${SW_AGENT_CACHE_CLASS:false}

# The instrumented classes cache mode: MEMORY or FILE
# MEMORY: cache class bytes to memory, if instrumented classes is too many or too large, it may take up more memory
# FILE: cache class bytes in `/class-cache` folder, automatically clean up cached class files when the application exits
# agent.class_cache_mode = ${SW_AGENT_CLASS_CACHE_MODE:MEMORY}

# The operationName max length
# Notice, in the current practice, we don't recommend the length over 190.
# agent.operation_name_threshold=${SW_AGENT_OPERATION_NAME_THRESHOLD:150}

# If true, skywalking agent will enable profile when user create a new profile task. Otherwise disable profile.
# profile.active=${SW_AGENT_PROFILE_ACTIVE:true}

# Parallel monitor segment count
# profile.max_parallel=${SW_AGENT_PROFILE_MAX_PARALLEL:5}

# Max monitor segment time(minutes), if current segment monitor time out of limit, then stop it.
# profile.duration=${SW_AGENT_PROFILE_DURATION:10}

# Max dump thread stack depth
# profile.dump_max_stack_depth=${SW_AGENT_PROFILE_DUMP_MAX_STACK_DEPTH:500}

# Snapshot transport to backend buffer size
# profile.snapshot_transport_buffer_size=${SW_AGENT_PROFILE_SNAPSHOT_TRANSPORT_BUFFER_SIZE:50}

# Backend service addresses. 后台服务地址 默认11800
collector.backend_service=${SW_AGENT_COLLECTOR_BACKEND_SERVICES:127.0.0.1:11800}

# Logging file_name 日志文件
logging.file_name=${SW_LOGGING_FILE_NAME:skywalking-api.log}

# Logging level 日志级别
logging.level=${SW_LOGGING_LEVEL:INFO}

# Logging dir
# logging.dir=${SW_LOGGING_DIR:""}

# Logging max_file_size, default: 300 * 1024 * 1024 = 314572800
# logging.max_file_size=${SW_LOGGING_MAX_FILE_SIZE:314572800}

# The max history log files. When rollover happened, if log files exceed this number,
# then the oldest file will be delete. Negative or zero means off, by default.
# logging.max_history_files=${SW_LOGGING_MAX_HISTORY_FILES:-1}

# Listed exceptions would not be treated as an error. Because in some codes, the exception is being used as a way of controlling business flow.
# Besides, the annotation named IgnoredException in the trace toolkit is another way to configure ignored exceptions.
# statuscheck.ignored_exceptions=${SW_STATUSCHECK_IGNORED_EXCEPTIONS:}

# The max recursive depth when checking the exception traced by the agent. Typically, we don't recommend setting this more than 10, which could cause a performance issue. Negative value and 0 would be ignored, which means all exceptions would make the span tagged in error status.
# statuscheck.max_recursive_depth=${SW_STATUSCHECK_MAX_RECURSIVE_DEPTH:1}

# Mount the specific folders of the plugins. Plugins in mounted folders would work.
plugin.mount=${SW_MOUNT_FOLDERS:plugins,activations}

# Exclude activated plugins
# plugin.exclude_plugins=${SW_EXCLUDE_PLUGINS:}

# mysql plugin configuration
# plugin.mysql.trace_sql_parameters=${SW_MYSQL_TRACE_SQL_PARAMETERS:false}

# Kafka producer configuration
# plugin.kafka.bootstrap_servers=${SW_KAFKA_BOOTSTRAP_SERVERS:localhost:9092}

# Match spring bean with regex expression for classname
# plugin.springannotation.classname_match_regex=${SW_SPRINGANNOTATION_CLASSNAME_MATCH_REGEX:}
```





RocketBot



Mysq监控：



### 获取追踪ID（TraceId）

```xml
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-trace</artifactId>
    <version>6.5.0</version>
</dependency>
```

Java中使用：

```java
String traceId = org.apache.skywalking.apm.toolkit.trace.TraceContext.traceId();  
```

### 过滤指定端点

1. 将/apache-skywalking/agent/optional-plugins/apm-trace-ignore-plugin-x.x.0.jar复制到/agent/plugins中
2. 两种配置方式：在config目录下新增`apm-trace-ignore-plugin.config`文件，配置`trace.ignore_path=${SW_AGENT_TRACE_IGNORE_PATH:/exclude/**}`；或者增加环境变量：`-Dskywalking.trace.ignore_path=/exclude/**`


![SkyWalking启动](.\image\SkyWalking插件.jpg)

### 告警功能：

apache-skywalking/config/alarm-settings.yml

```yaml
rules:
  # Rule unique name, must be ended with `_rule`.
  service_resp_time_rule:
    metrics-name: service_resp_time
    op: ">"
    threshold: 1000
    period: 10
    count: 3
    silence-period: 5
    message: Response time of service {name} is more than 1000ms in 3 minutes of last 10 minutes.
  service_sla_rule:
    # Metrics value need to be long, double or int
    metrics-name: service_sla
    op: "<"
    threshold: 8000
    # The length of time to evaluate the metrics
    period: 10
    # How many times after the metrics match the condition, will trigger alarm
    count: 2
    # How many times of checks, the alarm keeps silence after alarm triggered, default as same as period.
    silence-period: 3
    message: Successful rate of service {name} is lower than 80% in 2 minutes of last 10 minutes
  service_resp_time_percentile_rule:
    # Metrics value need to be long, double or int
    metrics-name: service_percentile
    op: ">"
    threshold: 1000,1000,1000,1000,1000
    period: 10
    count: 3
    silence-period: 5
    message: Percentile response time of service {name} alarm in 3 minutes of last 10 minutes, due to more than one condition of p50 > 1000, p75 > 1000, p90 > 1000, p95 > 1000, p99 > 1000
  service_instance_resp_time_rule:
    metrics-name: service_instance_resp_time
    op: ">"
    threshold: 1000
    period: 10
    count: 2
    silence-period: 5
    message: Response time of service instance {name} is more than 1000ms in 2 minutes of last 10 minutes
  database_access_resp_time_rule:
    metrics-name: database_access_resp_time
    threshold: 1000
    op: ">"
    period: 10
    count: 2
    message: Response time of database access {name} is more than 1000ms in 2 minutes of last 10 minutes
  endpoint_relation_resp_time_rule:
    metrics-name: endpoint_relation_resp_time
    threshold: 1000
    op: ">"
    period: 10
    count: 2
    message: Response time of endpoint relation {name} is more than 1000ms in 2 minutes of last 10 minutes
#  Active endpoint related metrics alarm will cost more memory than service and service instance metrics alarm.
#  Because the number of endpoint is much more than service and instance.
#
#  endpoint_avg_rule:
#    metrics-name: endpoint_avg
#    op: ">"
#    threshold: 1000
#    period: 10
#    count: 2
#    silence-period: 5
#    message: Response time of endpoint {name} is more than 1000ms in 2 minutes of last 10 minutes

webhooks:
#  - http://127.0.0.1/notify/
#  - http://127.0.0.1/go-wechat/
```

参数说明：

- rule_name：具有唯一性，展示在告警信息里面，必须以`_rule`结尾
- metrics-name：oal脚本里面的指标名称，支持long, double, int 类型
- op：操作，目前支持 `>`，`>=`，`<`，`<=`，`=`
- threshold：目标值（阈值）。对于多值指标（如：百分位），这个阈值是一个数组，格式如：value1,value2,value3,value4,value5。与指标格式一一对应。当不想其中某些指标触发告警时，阈值设置为横杠(`-`)
- period：周期。一个时间窗口，表示告警规则应当被检测多长时间，默认min
- count：在期限窗口时间段内，如果统计次数达到阈值，将触发告警
- silence-period：静默时间，表示在多长时间内只会触发一次告警。默认值和period相同，即在一个周期内，只会触发一次告警
- message：告警消息内容
- include-names：规则里包含的实体名称，例如：服务名字，端点名字。
- webhooks：告警回调地址，POST请求

