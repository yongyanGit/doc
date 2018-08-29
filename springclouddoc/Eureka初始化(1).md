### Eureka-Client初始化



**EurekaInstanceConfig**

> 它的完整包名：com.netflix.appinfo.EurekaInstanceConfig

配置实例注册到Eureka所需要的信息，一旦实例被注册，用户可以根据虚拟主机名称或者ip从EurekaCient获取信息。如下是它定义的部分信息

```java
String getInstanceId();//实例ID，它是唯一的，随着实例注册到eureka
String getAppname();//应用名称
boolean isInstanceEnabledOnit();//标识实例是否注册到eureka就开始进行通信
//设置eurekaclient发送心跳的频率，如果eurekaserver超过一定时间没有收到心跳,会将该实例移除掉
int getLeaseRenewalIntervalInSeconds();
//定义服务失效时间，是指当eurekaserver收到实例的最后一次心跳，然后等待实例的下一次心跳，如果等待的时间超过了失效时间，注册的实例就会被eurekaserver移除掉，其它服务不能访问。
int getLeaseExpirationDurationInSeconds();
String getVirtualHostName();//虚拟主机名，用来客户端调用当前实例
String getHostName(boolean refresh)//主机名称，用来远程调用
Map<String, String> getMetadataMap();//key/value 实例元数据 
String getIpAddress();//获取ip地址
```

**AbstractInstanceConfig**

> 完整包目录：com.netflix.appinfo.AbstractInstanceConfig

AbstactInstanceConfig是EurekaInstanceConfig的虚拟类实现，它就要提供一些通用的配置信息。主要的配置信息如下：

```java
//默认命名空间：eureka
public static final String DEFAULT_NAMESPACE = CommonConstants.DEFAULT_CONFIG_NAMESPACE;
//服务默认过期时间90s
private static final int LEASE_EXPIRATION_DURATION_SECONDS = 90;
//服务默认心跳频率30秒
private static final int LEASE_RENEWAL_INTERVAL_SECONDS = 30;
//默认关闭https
private static final boolean SECURE_PORT_ENABLED = false;
//http端口 80
private static final int NON_SECURE_PORT = 80;
//https端口
private static final int SECURE_PORT = 443;
//默认实例注册到eureka不立即进行通信
private static final boolean INSTANCE_ENABLED_ON_INIT = false;
//获取主机的ip地址和主机名称
private static final Pair<String, String> hostInfo = getHostInfo();
private static Pair<String, String> getHostInfo() {
        Pair<String, String> pair;
        try {
            InetAddress localHost = InetAddress.getLocalHost();
            pair = new Pair<String, String>(localHost.getHostAddress(), localHost.getHostName());
        } catch (UnknownHostException e) {
            logger.error("Cannot get host info", e);
            pair = new Pair<String, String>("", "");
        }
        return pair;
    }

//数据中心
private DataCenterInfo info = new DataCenterInfo() {
        @Override
        public Name getName() {
            return Name.MyOwn;
        }
    };
```

**PropertiesInstanceConfig**

> 完整包名称：PropertiesInstanceConfig



