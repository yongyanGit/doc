#### EurekaClientConfig

> 完整包名：com.netflix.discovery.EurekaClientConfig。用来配置Eureka client注册到Eureka server所需要的信息。

```java
int getRegistryFetchIntervalSeconds();//客户端向服务端获取注册信息的频率
//向eureka-server同步对象信息变化频率
int getInstanceInfoReplicationIntervalSeconds();
//向eureka-server同步信息初始化延迟时间
int getInitialInstanceInfoReplicationIntervalSeconds();
//eureka-client向服务端获取变动的频率
int getEurekaServiceUrlPollIntervalSeconds();
//获取服务地址变更频率
int getEurekaServiceUrlPollIntervalSeconds();
//获取代理服务器
String getProxyHost();
//代理服务端口
String getProxyPort();
//读取eureka server 超时时间
int getEurekaServerReadTimeoutSeconds();
//连接eureka server超时时间
int getEurekaServerConnectTimeoutSeconds();
//获取备份注册信息。当eureka client启动时，无法从eureka sever中获取信息，可以从备份注册中心获取(目前未实现)
String getBackupRegistryImpl();
//所有eureka server连接数
int getEurekaServerTotalConnections();
//是否向eureka server注册自身服务
boolean shouldRegisterWithEureka();
//当客户端关闭时，是否向远程服务取消自身注册的服务
 default boolean shouldUnregisterOnShutdown() {
        return true;
    }
//优先使用相同zone的eureka server
boolean shouldPreferSameZoneEureka();
//是否允许Eureka Server 重定向
boolean allowRedirects();
//获取regions的注册信息
String fetchRegistryForRemoteRegions();
//获取客户端所在region,默认是：us-east-1
String getRegion();
//获取所在区域的zones
String[] getAvailabilityZones(String region);
//获取所在zone的eureka server 服务url
List<String> getEurekaServerServiceUrls(String myZone);
//是否过滤，只获取状态为(up)的应用实例集合
boolean shouldFilterOnlyUpInstances();
//eureka-server空闲连接关闭时间
int getEurekaConnectionIdleTimeoutSeconds();
//是否从eureka-server拉取注册信息
boolean shouldFetchRegistry();
//心跳线程池大小
int getHeartbeatExecutorThreadPoolSize();
//心跳延迟后的重试时间
int getHeartbeatExecutorExponentialBackOffBound();
//注册信息缓存刷新线程池大小
int getCacheRefreshExecutorThreadPoolSize();
//注册信息缓存刷新执行超时后的延迟重新时间
int getCacheRefreshExecutorExponentialBackOffBound();
//序列化或者反序列化时，将'$'替换成字符串
String getDollarReplacement();
//序列化或者反序列化时,将'_'替换成字符串
String getEscapeCharReplacement();
//是否同步应用实例状态到 Eureka-Server
boolean shouldOnDemandUpdateStatusChange();
//客户端初始化时是否强制注册到eureka server
default boolean shouldEnforceRegistrationAtInit() {
        return false;}
//client可接受的数据类型
String getClientDataAccept();
```

