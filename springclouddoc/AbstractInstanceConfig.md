#### AbstractInstanceConfig

> 完整包名：com.netflix.appinfo.AbstractInstanceConfig，AbstractInstanceConfig是EurekaInstanceConfig的虚拟类实现，它主要是对父接口中的一些属性提供默认配置。

如下是它提供的默认配置信息：

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

