### Eureka 参数调优

#### 1. 客户端

| 参数                                                 | 默认值 | 说明                                                         |
| ---------------------------------------------------- | ------ | ------------------------------------------------------------ |
| eureka.client.availability-zones                     |        | 告知Client有哪些region和region下面的availability-zones       |
| eureka.client.filter-only-up-instances               | true   | 是否过滤出InstanceStatus为up的实例                           |
| eureka.client.region                                 |        | 指定实例所在的region                                         |
| eureka.client.register-with-eureka                   | true   | 是否将应用实例注册到Eureka Server                            |
| eureka.client.prefer-same-zone-eureka                | true   | 是否优先使用与当前实例在同一个zone的eureka server            |
| eureka.client.on-demand-update-status-change         | true   | 是否将本地实例状态的更新通过ApplicationInfoManager实时同步到Eureka Server |
| eureka.instance.metadata-map                         |        | 指定应用实例的元数据map                                      |
| eureka.instance.prefer-ip-address                    | false  | 是否优先使用ip地址来代替host name作为实例的hostName字段值。  |
| eureka.instance.lease-expiration-duration-in-seconds | 90     | 指定Eureka Client 间隔多久需要向Eureka Server 发送心跳来告知Eureka Server该实例还存活着 |

定时任务参数如下：

| 参数                                                         | 默认值 | 说明                                                         |
| ------------------------------------------------------------ | ------ | ------------------------------------------------------------ |
| eureka.client.cache-refresh-executor-thread-pool-size        | 2      | 刷新缓存的CacheRefreshThread的线程池大小                     |
| eureka.client.cache-refresh-executor-exponential-back-off-bound | 10     | 调用任务执行超时时下次的调度的延时时间                       |
| eureka.client.heartbeat-executor-thread-pool-size            | 2      | 心跳现场HeatbeatThread的线程池大小                           |
| eureka.client.heartbeat-executor-exponential-back-off-bound  | 10     | 调用任务执行超时时下次的调度的延时时间                       |
| eureka.client.registry-fetch-interval-seconds                | 30     | CacheRefreshThread线程的调用频率                             |
| eureka.client.eureka-service-url-poll-interval-seconds       | 5*60   | AsyncResolver.updateTask刷新Eureka Server地址间隔            |
| eureka.client.initial-instance-info-replication-interval-seconds | 40     | InstanceInfoReplicator将实例信息变更同步到Eureka Server的初始延时时间 |
| eureka.client.instance-info-replication-interval-seconds     | 30     | InstanceInfoReplicator将实例信息变更同步到Eureka Server的时间间隔 |
| eureka.instance.lease-renewal-interval-in-seconds            | 30     | Eureka Client 向Eureka Server发送心跳的时间间隔              |

http参数

| 参数                                                   | 默认值 | 说明                       |
| ------------------------------------------------------ | ------ | -------------------------- |
| eureka.client.eureka-server-connect-timeout-seconds    | 5      | 连接超时时间               |
| eureka.client.eureka-server-read-timeout-seconds       | 8      | 读取超时                   |
| eureka.client.eureka-server-total-connections          | 200    | 连接池最大活动连接数       |
| eureka.client.eureka-connection-idle-timeout-seconds   | 30     | 连接池中连接的空闲时间     |
| eureka.client.eureka-server-total-connections-per-host | 50     | 每个host能使用的最大连接数 |

#### 2. Server 端

基本参数

| 参数                                                       | 默认值  | 说明                                                         |
| ---------------------------------------------------------- | ------- | ------------------------------------------------------------ |
| eureka.server.enable-self-preservation                     | true    | 是否开启自我保护模式                                         |
| eureka.server.renewal-percent-threshold                    | 0.85    | 指定每分钟需要收到的续约次数的阀值                           |
| eureka.instance.registry.expected-number-of-renews-per-min | 1       | 指定每分钟需要收到的续约次数，实际值在其中被写死为count*2    |
| eureka.server.renewal-threshold-update-interval-ms         | 15分钟  | 指定updateRenewalThreshold定时任务的调度频率，来动态更新expectedNumberOfRenewsPerMin以及numberOfRenewsPerMinThreshold |
| eureka.server.eviction-interval-timer-in-ms                | 60*1000 | 指定EvictionTask定时任务的调度频率，用于剔除过期实例         |

response cache参数

Eureka Server 为了提升自身REST API接口的性能，提供了两个缓存：一个基于ConcurrentMap的readOnlyCacheMap，一个是基于Guava Cache的readWriteCacheMap.

| 参数                                                    | 默认值  | 说明                                                         |
| ------------------------------------------------------- | ------- | ------------------------------------------------------------ |
| eureka.server.use-read-only-response-cache              | true    | 是否使用只读的response-cache                                 |
| eureka.server.response-cache-update-interval-ms         | 30*1000 | 设置CacheUpdateTsk的调度时间间隔，用于从readWrite-CacheMap更新数据到readOnlyCacheMap。仅仅在eureka.server.use-read-only-response-cache为true的时候才生效 |
| eureka.server.response-cache-auto-expiration-in-seconds | 180     | 设置readWriteCacheMap的expireAfter参数，指定写入多长时间后长期。 |

http参数

Eureka Server 需要与其它peer节点进行通信，复制实例参数，其底层使用httpClient，

| 参数                                                    | 默认值 | 说明                                           |
| ------------------------------------------------------- | ------ | ---------------------------------------------- |
| eureka.server.peer-node-connect-timeout-ms              | 200    | 连接超时                                       |
| eureka.server.peer-node-read-timeout-ms                 | 200    | 读超时时间                                     |
| eureka.server.peer-node-total-connections               | 1000   | 连接池最大活动连接数(MaxTotal)                 |
| eureka.server.peer-node-total-connections-per-host      | 500    | 每个host能使用的最大连接数(DefaultMaxPerRoute) |
| eureka.server.peer-node-connection-idle-timeout-seconds | 30     | 连接池中连接的空闲时间(connectionIdleTimeout)  |

