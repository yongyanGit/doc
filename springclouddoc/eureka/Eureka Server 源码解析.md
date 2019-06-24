### Eureka Server 源码解析

#### @EurekaEurekaServer注解

我们创建一个服务为Eureka 服务通常需要在启动类上添加一个注解```@EnableEurekaServer```，因此我们就从这个注解来看看它做了什么事：

```java
@Import(EurekaServerMarkerConfiguration.class)
public @interface EnableEurekaServer {
}
```

在这个注解中主要是引入了```EurekaServerMarkerConfiguration```配置类，在这个类中主要是创建一个Marker实例标记，这个实例没有业务处理逻辑，它仅仅是EurekaServerAutoConfiguration创建的前提。

```java
/**
 * Responsible for adding in a marker bean to activate
 * {@link EurekaServerAutoConfiguration}
 */
@Configuration
public class EurekaServerMarkerConfiguration {
	@Bean
	public Marker eurekaServerMarkerBean() {
		return new Marker();
	}
	class Marker {
	}
}

```

我们可以看到这个注释```Responsible for adding in a marker bean to activate```这段注释，它的意思是负责添加一个标记来启动某个类。

#### EurekaServerAutoConfiguration

EurekaServerAutoConfiguration主要是SpringCloud用来与Eureka 进行桥接的，在这个类里面进行初始化Eureka Server 启动所需要的资源环境。

首先我们来看看```EurekaServerAutoConfiguration```这个配置类以及它负责初始化的实例：

```java

@Import(EurekaServerInitializerConfiguration.class)
@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)
@EnableConfigurationProperties({ EurekaDashboardProperties.class,
		InstanceRegistryProperties.class })
@PropertySource("classpath:/eureka/server.properties")
public class EurekaServerAutoConfiguration extends WebMvcConfigurerAdapter{
    
    //注册、续约等操作
    @Bean
	public PeerAwareInstanceRegistry peerAwareInstanceRegistry(
			ServerCodecs serverCodecs) {
		this.eurekaClient.getApplications(); // force initialization
		return new InstanceRegistry(this.eurekaServerConfig, this.eurekaClientConfig,
				serverCodecs, this.eurekaClient,
				this.instanceRegistryProperties.getExpectedNumberOfRenewsPerMin(),
				this.instanceRegistryProperties.getDefaultOpenForTrafficCount());
	}
	//集群间同步
	@Bean
	@ConditionalOnMissingBean
	public PeerEurekaNodes peerEurekaNodes(PeerAwareInstanceRegistry registry,
			ServerCodecs serverCodecs) {
		return new RefreshablePeerEurekaNodes(registry, this.eurekaServerConfig,
				this.eurekaClientConfig, serverCodecs, this.applicationInfoManager);
	}
    
    //上下文
    @Bean
	public EurekaServerContext eurekaServerContext(ServerCodecs serverCodecs,
			PeerAwareInstanceRegistry registry, PeerEurekaNodes peerEurekaNodes) {
		return new DefaultEurekaServerContext(this.eurekaServerConfig, serverCodecs,
				registry, peerEurekaNodes, this.applicationInfoManager);
	}
	//看到名字就知道它是服务的启动入口
	@Bean
	public EurekaServerBootstrap eurekaServerBootstrap(PeerAwareInstanceRegistry registry,
			EurekaServerContext serverContext) {
		return new EurekaServerBootstrap(this.applicationInfoManager,
				this.eurekaClientConfig, this.eurekaServerConfig, registry,
				serverContext);
	}
    
}
```

1. @Import(EurekaServerInitializerConfiguration.class)：导入Eureka Server 初始化类。
2. @ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)：这个就是前面提到的启动标记，当Spring容器启动时，会创建该实例，接着就可以初始化```EurekaServerAutoConfiguration```，从而启动Eureka服务端。
3. @PropertySource("classpath:/eureka/server.properties")：加载配置文件。

#### EurekaServerInitializerConfiguration

EurekaServerInitializerConfiguration主要是开启一个初始化流程，然后启动Eureka Server。我们可以看它到在它内部有一个start方法，它就是用来初始化配置。

```java
public void start() {
	new Thread(new Runnable() {
		public void run() {
			try {
              	//eurekaServerBootstrap就是在EurekaServerAutoConfiguration中创建的
                eurekaServerBootstrap.contextInitialized(
                    EurekaServerInitializerConfiguration
                    .this.servletContext);
				log.info("Started Eureka Server");
				publish(new EurekaRegistryAvailableEvent(getEurekaServerConfig()));
				EurekaServerInitializerConfiguration.this.running = true;
				publish(new EurekaServerStartedEvent(getEurekaServerConfig()));
			}
				catch (Exception ex) {
					// Help!
					log.error("Could not initialize Eureka servlet context", ex);
			}
		}
	}).start();
}
```

可以看到```log.info("Started Eureka Server");```就是说明了服务已经启动个了。

#### EurekaServerBootstrap

```java
public void contextInitialized(ServletContext context) {
    try {
        //加载配置文件
        initEurekaEnvironment();
        //初始化Eureka上下文
        initEurekaServerContext();        context.setAttribute(EurekaServerContext.class.getName(), this.serverContext);
    }
    catch (Throwable e) {
        log.error("Cannot bootstrap eureka server :", e);
        throw new RuntimeException("Cannot bootstrap eureka server :", e);
    }
}
//initEurekaServerContext()方法里面会去复制eureka 节点
protected void initEurekaServerContext() throws Exception	
	// Copy registry from neighboring eureka node
	int registryCount = this.registry.syncUp();
}

```



#### DefaultEurekaServerContext

在DefaultEurekaServerContext中主要是调用PeerAwareInstanceRegistryImpl的init方法

```java
public void initialize() {
	logger.info("Initializing ...");
        peerEurekaNodes.start();
        try {
            registry.init(peerEurekaNodes);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
        logger.info("Initialized");
}

//PeerAwareInstanceRegistryImpl
public void init(PeerEurekaNodes peerEurekaNodes) throws Exception {
    
	this.numberOfReplicationsLastMin.start();
    this.peerEurekaNodes = peerEurekaNodes;
    //初始化Eureka Server 响应缓存，默认时间为30s
    initializedResponseCache();
    //定时任务，多久重置一下心跳阀值(15分钟)
    scheduleRenewalThresholdUpdateTask();
    //初始化远端注册
    initRemoteRegionRegistry();
}

//初始化Eureka Server 响应缓存
public synchronized void initializedResponseCache() {
	if (responseCache == null) {
    	responseCache = new ResponseCacheImpl(serverConfig, serverCodecs, this);
    }
}
//重置心跳阀值
private void scheduleRenewalThresholdUpdateTask() {
	timer.schedule(new TimerTask() {
    	@Override
        public void run() {
        	updateRenewalThreshold();
       }
    }, serverConfig.getRenewalThresholdUpdateIntervalMs(),
                serverConfig.getRenewalThresholdUpdateIntervalMs());
}

```



启动Eureka服务端，它的部分启动流程日志：

```java
//EurekaServerBootstrap是启动入口,加载Eureka 配置文件
2019-03-28 13:30:51.195  INFO 8385 --- [      Thread-32] 
o.s.c.n.e.server.EurekaServerBootstrap   : Setting the eureka configuration..
2019-03-28 13:30:51.196  INFO 8385 --- [      Thread-32] o.s.c.n.e.server.EurekaServerBootstrap   : Eureka data center value eureka.datacenter is not set, defaulting to default
//1. 设置上下文环境
2019-03-28 13:30:51.197  INFO 8385 --- [      Thread-32] o.s.c.n.e.server.EurekaServerBootstrap   : Eureka environment value eureka.environment is not set, defaulting to test
2019-03-28 13:30:51.244  INFO 8385 --- [      Thread-32] o.s.c.n.e.server.EurekaServerBootstrap   : isAws returned false
//初始化上下文
2019-03-28 13:30:51.247  INFO 8385 --- [      Thread-32] o.s.c.n.e.server.EurekaServerBootstrap   : Initialized server context
//在集群环境中，注册到其他Eureka Server
2019-03-28 13:30:51.291  INFO 8385 --- [      Thread-32] c.n.e.registry.AbstractInstanceRegistry  : Registered instance EUREKA-SERVER/192.168.72.102:eureka-server:1111 with status UP (replication=true)
2019-03-28 13:30:51.291  INFO 8385 --- [      Thread-32] c.n.e.r.PeerAwareInstanceRegistryImpl    : Got 1 instances from neighboring DS node
2019-03-28 13:30:51.292  INFO 8385 --- [      Thread-32] c.n.e.r.PeerAwareInstanceRegistryImpl    : Renew threshold is: 1
//设置状态为up
2019-03-28 13:30:51.292  INFO 8385 --- [      Thread-32] c.n.e.r.PeerAwareInstanceRegistryImpl    : Changing status to UP
2019-03-28 13:30:51.302  INFO 8385 --- [      Thread-32] e.s.EurekaServerInitializerConfiguration : Started Eureka Server//服务启动
```









在看看EurekaServerMarkerConfiguration中比较重要的实例：

```java
//EurekaServerBootstrap是Eureka的启动入口
@Bean
public EurekaServerBootstrap eurekaServerBootstrap(PeerAwareInstanceRegistry registry,
EurekaServerContext serverContext) {
	return new EurekaServerBootstrap(this.applicationInfoManager,
				this.eurekaClientConfig, this.eurekaServerConfig, registry,
				serverContext);
	}
```

