#### EurekaInstanceConfig

> 完整包名：com.netflix.appinfo.EurekaInstanceConfig。它是一个接口，主要用来配置Euerka client注册到Eureka server 所需要的信息。

服务提供者一旦注册到Eureka server,其它用户可以通过```com.netflix.discovery.EurekaClient```获取它注册到注册中心的信息，Eureka client注册时，id和appname必须提供，并且appname必须是唯一不重复的。

下面是EurekaInstanceConfig包含的部分内容：

```java
String getInstanceId();//实例ID，它是唯一的，随着实例注册到eureka
String getAppname();//应用名称,默认取spring.application.name的值
String getAppGroupName();//应用分组
//Eureka client注册到Eureka server 后，是否立即与Eureka server 通信，因为有时候客户端需要做一些准备来与Eureka server通信。true 立即通信。
boolean isInstanceEnabledOnit();
//设置Eureka client发送心跳的频率,默认30s，如果Eureka server超过一定时间没有收到心跳,会将该实例移除掉
int getLeaseRenewalIntervalInSeconds();
int getNonSecurePort();//http端口
int getSecurePort();//https端口
boolean isNonSecurePortEnabled();//http端口是否开启
boolean getSecurePortEnabled();//https端口是否开启
//定义服务失效时间，是指当Eureka server收到实例的最后一次心跳，然后等待实例的下一次心跳，如果等待的时间超过了失效时间，注册的实例就会被eurekaserver移除掉，其它服务不能访问。
int getLeaseExpirationDurationInSeconds();
String getVirtualHostName();//虚拟主机名(VIPAddress)
String getHostName(boolean refresh)//主机名称，不配置时，将采用操作系统的主机名
Map<String, String> getMetadataMap();//key/value 实例元数据 
DataCenterInfo getDataCenterInfo();//实例数据中心
String getIpAddress();//获取ip地址
String getStatusPageUrl();//状态页的URL
String getHomePageUrl();//应用主页的URL
String getHealthCheckUrl();//健康检查的url
String getNamespace();//命名空间

```

关于状态页和健康检查页，在SpringCloud Eureka中默认使用了spring-boot-actuator模块提供的/actuator/info端点和/actuator/health端点。

为了服务的正常运作，我们必须确保Eureka客户端的/health端点发送元数据的时候，是一个能被注册中心访问到的地址。否则服务注册中心不会根据应用的健康检查来更改状态(healthcheck开启时)。而/info端点不正确时，会导致Eureka面板中单击服务实例时，无法访问到服务实例提供的信息接口。







