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

> 完整包名称：PropertiesInstanceConfig

propertiesInstanceConfig是基于配置文件的Eureka应用实例实例配置抽象类。我们主要看下它的初始化，如下

```java
public PropertiesInstanceConfig(String namespace, DataCenterInfo info) {
        super(info);
		//设置命名空间，以'.'结尾
        this.namespace = namespace.endsWith(".")
                ? namespace
                : namespace + ".";
		//从环境变量中获取应用分组
        appGrpNameFromEnv = ConfigurationManager.getConfigInstance()
                .getString(FALLBACK_APP_GROUP_KEY, Values.UNKNOWN_APPLICATION);
		//初始化配置文件对象
        this.configInstance = Archaius1Utils.initConfig(CommonConstants.CONFIG_FILE_NAME);
    }
```

```initConfig()```方法

```java
public static DynamicPropertyFactory initConfig(String configName) {
		//配置文件对象
        DynamicPropertyFactory configInstance = DynamicPropertyFactory.getInstance();
    	//配置文件名称,首先读取eureka.client.props，如果不存在则读取eureka-client
        DynamicStringProperty EUREKA_PROPS_FILE = configInstance.getStringProperty("eureka.client.props", configName);
		//配置文件环境
        String env = ConfigurationManager.getConfigInstance().getString(EUREKA_ENVIRONMENT, "test");     ConfigurationManager.getConfigInstance().setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, env);
		
        String eurekaPropsFile = EUREKA_PROPS_FILE.get();
        try {
       //将配置文件加载到环境变量      ConfigurationManager.loadCascadedPropertiesFromResources(eurekaPropsFile);
        } catch (IOException e) {
            logger.warn(
                    "Cannot find the properties specified : {}. This may be okay if there are other environment "
                            + "specific properties or the configuration is installed with a different mechanism.",
                    eurekaPropsFile);

        }

        return configInstance;
    }
```

* 生成文件对象：```DynamicPropertyFactory.getInstance();```

```java
public static DynamicPropertyFactory getInstance() {
        if (config == null) {
            synchronized (ConfigurationManager.class) {
                if (config == null) {
                    AbstractConfiguration configFromManager = ConfigurationManager.getConfigInstance();
                    if (configFromManager != null) {
                        initWithConfigurationSource(configFromManager);
                        initializedWithDefaultConfig = !ConfigurationManager.isConfigurationInstalled();
                        logger.info("DynamicPropertyFactory is initialized with configuration sources: " + configFromManager);
                    }
                }
            }
        }
        return instance;
    }
```

* ```ConfigurationManager.getConfigInstance()方法```

```java
public static AbstractConfiguration getConfigInstance() {
        if (instance == null) {
            synchronized (ConfigurationManager.class) {
                if (instance == null) {
                    instance = getConfigInstance(Boolean.getBoolean(DynamicPropertyFactory.DISABLE_DEFAULT_CONFIG));
                }
            }
        }
        return instance;
    }
```

如果instance为空，则创建一个AbstractConfiguration对象。

* ```getConfigInstance()方法```

```java
private static AbstractConfiguration getConfigInstance(boolean defaultConfigDisabled) {
        if (instance == null && !defaultConfigDisabled) {
            instance = createDefaultConfigInstance();
            registerConfigBean();
        }
        return instance;        
    }
```

接着看```createDefaultConfigInstance()```方法

```java
private static AbstractConfiguration createDefaultConfigInstance() {
    	//用来保存配置对象
        ConcurrentCompositeConfiguration config = new ConcurrentCompositeConfiguration();  
        try {
            //m默认加载config.properties,也可以通过archaius.configurationSource.defaultFileName指定加载文件
            DynamicURLConfiguration defaultURLConfig = new DynamicURLConfiguration();
            //archaius.dynamicPropertyFactory.URL_CONFIG
            config.addConfiguration(defaultURLConfig, URL_CONFIG_NAME);
        } catch (Throwable e) {
            logger.warn("Failed to create default dynamic configuration", e);
        }
        if (!Boolean.getBoolean(DISABLE_DEFAULT_SYS_CONFIG)) {
            SystemConfiguration sysConfig = new SystemConfiguration();
            //archaius.dynamicPropertyFactory.SYS_CONFIG
            config.addConfiguration(sysConfig, SYS_CONFIG_NAME);
        }
        if (!Boolean.getBoolean(DISABLE_DEFAULT_ENV_CONFIG)) {
            EnvironmentConfiguration envConfig = new EnvironmentConfiguration();
            //archaius.dynamicPropertyFactory.ENV_CONFIG
            config.addConfiguration(envConfig, ENV_CONFIG_NAME);
        }
        ConcurrentCompositeConfiguration appOverrideConfig = new ConcurrentCompositeConfiguration();
    	//APPLICATION_PROPERTIES
        config.addConfiguration(appOverrideConfig, APPLICATION_PROPERTIES);
        config.setContainerConfigurationIndex(config.getIndexOfConfiguration(appOverrideConfig));
        return config;
    }
```

```ConcurrentCompositeConfiguration```是一个配置的集合，可以加载多个配置文件，每个配置文件都加载到自己单独的AbstractConfiguration中。

```
private Map<String, AbstractConfiguration> namedConfigurations = new ConcurrentHashMap<String, AbstractConfiguration>();
    
    private List<AbstractConfiguration> configList = new CopyOnWriteArrayList<AbstractConfiguration>();
```

configList就是用来存放配置对象的集合，如果要查找其中的一个对象，就要循环configList中的每一个AbstractConfiguration，一直到找到为止。

```java
public void addConfigurationAtIndex(AbstractConfiguration config, String name, int index) 
    throws IndexOutOfBoundsException {
        if (!configList.contains(config)) {
            checkIndex(index);
            configList.add(index, config);
            if (name != null) {
                namedConfigurations.put(name, config);
            }
            config.addConfigurationListener(eventPropagater);
            fireEvent(EVENT_CONFIGURATION_SOURCE_CHANGED, null, null, false);
        } else {
            logger.warn(config + " is not added as it already exits");
        }
    }
```

从上面代码可以看到配置对象会被添加到configList集合中。在上面代码中还有一段代码：```config.addConfigurationListener(eventPropagater);```,这段代码的作用是为配置对象增加一个监听对象（采用的是观察者模式），实现原理大概是，定义一个观察者接口：```ConfigurationListener```，它里面只有一个接口：```configurationChanged```，用来监听配置对象，它的实现类在```ConcurrentCompositeConfiguration```中，是一个内部类实现

```java
private ConfigurationListener eventPropagater = new ConfigurationListener() {
        @Override
        public void configurationChanged(ConfigurationEvent event) {
            boolean beforeUpdate = event.isBeforeUpdate();
            if (propagateEventToParent) {
                int type = event.getType();
                String name = event.getPropertyName();
                Object value = event.getPropertyValue();
                Object finalValue;
                switch(type) {
                case HierarchicalConfiguration.EVENT_ADD_NODES:
                case EVENT_CLEAR:
                case EVENT_CONFIGURATION_SOURCE_CHANGED:
                    fireEvent(type, name, value, beforeUpdate);
                    break;
               ....
            }            
        }        
    };
```



然后是被观察者的实现即配置类，所有的配置类都会继承一个类```ConcurrentMapConfiguration```，它内部有一个集合```Collection<ConfigurationListener> listeners = new CopyOnWriteArrayList<ConfigurationListener>();```,用来保存注册的观察者，而注册观察者是通过```addConfigurationListener```来进行注册：

```
public void addConfigurationListener(ConfigurationListener l) {
        if (!listeners.contains(l)) {
            listeners.add(l);
        }
    }
```



可以看到注册的监听者都会保存到listeners中去，所以每当要发布消息时，都会遍历```listeners```集合把消息发布出去。发布消息也是通过```fireEvent```方法

```java
protected void fireEvent(int type, String propName, Object propValue, boolean beforeUpdate) {
        if (listeners == null || listeners.size() == 0) {
            return;
        }        
        ConfigurationEvent event = createEvent(type, propName, propValue, beforeUpdate);
        for (ConfigurationListener l: listeners)
        {
            try {
                l.configurationChanged(event);
            } catch (ValidationException e) {
                if (beforeUpdate) {
                    throw e;
                } else {
                    logger.error("Unexpected exception", e);                    
                }
            } catch (Throwable e) {
                logger.error("Error firing configuration event", e);
            }
        }
    }
```

从上面可以看到循环遍历listeners集合，然后调用监听者的```configurationChanged()```方法。

#### MyDataCenterInstanceConfig

> 完整包名称：package com.netflix.appinfo.MyDataCenterInstanceConfig

```
public class MyDataCenterInstanceConfig extends PropertiesInstanceConfig implements EurekaInstanceConfig {

    public MyDataCenterInstanceConfig() {
    }

    public MyDataCenterInstanceConfig(String namespace) {
        super(namespace);
    }

    public MyDataCenterInstanceConfig(String namespace, DataCenterInfo dataCenterInfo) {
        super(namespace, dataCenterInfo);
    }

}
```

一般情况下，使用MyDataCenterInstanceConfig配置Eureka应用实例。

#### InstanceInfo

> 完整包名：package com.netflix.appinfo.InstanceInfo

用来保存eureka-client向eureka server 注册的信息，```InstanceInfo```由```EurekaConfigBasedInstanceInfoProvider```在```EurekaInstanceConfig```的基础上创建:

```JAVA
 public synchronized InstanceInfo get() {
        if (instanceInfo == null) {
            // Build the lease information to be passed to the server based on config
            //创建 租约信息构建器，并从EurekaInstanceConfig中获取默认值并设置
            LeaseInfo.Builder leaseInfoBuilder = LeaseInfo.Builder.newBuilder()
                    .setRenewalIntervalInSecs(config.getLeaseRenewalIntervalInSeconds())
                    .setDurationInSecs(config.getLeaseExpirationDurationInSeconds());
			//创建vip地址解析器
            if (vipAddressResolver == null) {
                vipAddressResolver = new Archaius1VipAddressResolver();
            }

            // Builder the instance information to be registered with eureka server
            //创建应用实例信息构建器
            InstanceInfo.Builder builder = 			    ｀			InstanceInfo.Builder.newBuilder(vipAddressResolver);

            // set the appropriate id for the InstanceInfo, falling back to datacenter Id if applicable, else hostname
             // 获取应用实例编号 
            String instanceId = config.getInstanceId();
            if (instanceId == null || instanceId.isEmpty()) {
                DataCenterInfo dataCenterInfo = config.getDataCenterInfo();
                if (dataCenterInfo instanceof UniqueIdentifier) {
                    instanceId = ((UniqueIdentifier) dataCenterInfo).getId();
                } else {
                    instanceId = config.getHostName(false);
                }
            }
			//获取主机名称
            String defaultAddress;
            if (config instanceof RefreshableInstanceConfig) {
                // Refresh AWS data center info, and return up to date address
                defaultAddress = ((RefreshableInstanceConfig) config).resolveDefaultAddress(false);
            } else {
                defaultAddress = config.getHostName(false);
            }

            // fail safe
            if (defaultAddress == null || defaultAddress.isEmpty()) {
                defaultAddress = config.getIpAddress();
            }
			//设置 应用实例信息构建器的属性
            builder.setNamespace(config.getNamespace())
                    .setInstanceId(instanceId)
                    .setAppName(config.getAppname())
                    .setAppGroupName(config.getAppGroupName())
                    .setDataCenterInfo(config.getDataCenterInfo())
                    .setIPAddr(config.getIpAddress())
                    .setHostName(defaultAddress)
                    .setPort(config.getNonSecurePort())
                    .enablePort(PortType.UNSECURE, config.isNonSecurePortEnabled())
                    .setSecurePort(config.getSecurePort())
                    .enablePort(PortType.SECURE, config.getSecurePortEnabled())
                    .setVIPAddress(config.getVirtualHostName())
                    .setSecureVIPAddress(config.getSecureVirtualHostName())
                    .setHomePageUrl(config.getHomePageUrlPath(), config.getHomePageUrl())
                    .setStatusPageUrl(config.getStatusPageUrlPath(), config.getStatusPageUrl())
                    .setASGName(config.getASGName())
                    .setHealthCheckUrls(config.getHealthCheckUrlPath(),
                            config.getHealthCheckUrl(), config.getSecureHealthCheckUrl());


            // Start off with the STARTING state to avoid traffic
            //应用启动后不立即与eureka server 通信
            if (!config.isInstanceEnabledOnit()) {
                InstanceStatus initialStatus = InstanceStatus.STARTING;
                LOG.info("Setting initial instance status as: {}", initialStatus);
                builder.setStatus(initialStatus);
            } else {
                LOG.info("Setting initial instance status as: {}. This may be too early for the instance to advertise "
                         + "itself as available. You would instead want to control this via a healthcheck handler.",
                         InstanceStatus.UP);
            }
			//
            // Add any user-specific metadata information
            //设置应用实例信息构建器的元数据(metadata)集合
            for (Map.Entry<String, String> mapEntry : config.getMetadataMap().entrySet()) {
                String key = mapEntry.getKey();
                String value = mapEntry.getValue();
                builder.add(key, value);
            }
			//创建应用实例信息
            instanceInfo = builder.build();
            //设置应用实例租约信息
            instanceInfo.setLeaseInfo(leaseInfoBuilder.build());
        }
        return instanceInfo;
    }
```



#### ApplicationInfoManager

> 完整包名：package com.netflix.appinfo.ApplicationInfoManager,应用信息管理器



```java
public class ApplicationInfoManager {
	//单例
    private static ApplicationInfoManager instance = new ApplicationInfoManager(null, null, null);
	//状态变更监听器
    protected final Map<String, StatusChangeListener> listeners;
    //应用实例状态匹配
    private final InstanceStatusMapper instanceStatusMapper;
	//应用实例信息 
    private InstanceInfo instanceInfo;
    //应用实例配置
    private EurekaInstanceConfig config;

   //构造方法
    public ApplicationInfoManager(EurekaInstanceConfig config, InstanceInfo instanceInfo, OptionalArgs optionalArgs) {
        this.config = config;
        this.instanceInfo = instanceInfo;
        this.listeners = new ConcurrentHashMap<String, StatusChangeListener>();
        if (optionalArgs != null) {
            this.instanceStatusMapper = optionalArgs.getInstanceStatusMapper();
        } else {
            this.instanceStatusMapper = NO_OP_MAPPER;
        }

        // Hack to allow for getInstance() to use the DI'd ApplicationInfoManager
        instance = this;
    }
}
```

