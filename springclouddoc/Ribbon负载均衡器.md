#### Ribbon负载均衡器

**AbstractLoadBalancer**

AbstractLoadBalancer是ILoadBalanceer接口的抽象实现，在该抽象类中定义了一个关于服务实例分组的枚举类ServerGroup,它包含了三种类型：

* ALL :所有服务实例
* STATUS_UP:正常的服务实例
* STATUS_NOT_UP:停止服务的实例

另外还实现了一个chooseServer()方法，它会调用父类接口中的chooseServer(key)方法，并且传入的参数为null，表示选择具体服务实例时，忽略key的条件判断,最后它还定义了两个接口：

* getServerList(ServerGroup serverGroup)：根据服务分组来获取不同的服务实例的列表
* getLoadBalanceStats()：LoadBalanceStats用来保存负载均衡器中的各个服务实例当前的属性和统计信息。

```java
public abstract class AbstractLoadBalancer implements ILoadBalancer {
    
    public enum ServerGroup{
        ALL,
        STATUS_UP,
        STATUS_NOT_UP        
    }
        
    public Server chooseServer() {
    	return chooseServer(null);
    }

    public abstract List<Server> getServerList(ServerGroup serverGroup);
    
    public abstract LoadBalancerStats getLoadBalancerStats();    
}

```

**BaseLoadBalancer**

BaseLoadBalancer类是Ribbon负载均衡器的基础实现类

* 定义和维护了两个存储服务实例server对象的列表。一个用于存储所有服务实例，一个用于存储正常的服务实例。

```java
@Monitor(name = PREFIX + "AllServerList", type = DataSourceType.INFORMATIONAL)
protected volatile List<Server> allServerList = Collections
            .synchronizedList(new ArrayList<Server>());
@Monitor(name = PREFIX + "UpServerList", type = DataSourceType.INFORMATIONAL)
protected volatile List<Server> upServerList = Collections
            .synchronizedList(new ArrayList<Server>());
```

* 定义了用来存储负载均衡器中各个服务实例的属性和统计信息对象LoadBalancerStats。
* 定义了检查服务实例是否正常服务的IPing对象，在BaseLoadBalancer中默认是null，需要在构造时注入它的实现
* 定义了检查服务实例操作的执行策略对象IPingStrategy，在BaseLoadBalancer中，默认使用了该类中定义的静态内部类SerialPingStrategy实现

```java
 private static class SerialPingStrategy implements IPingStrategy {
 	@Override
    public boolean[] pingServers(IPing ping, Server[] servers) {
    	int numCandidates = servers.length;
    	boolean[] results = new boolean[numCandidates];     
    	for (int i = 0; i < numCandidates; i++) {
         	results[i] = false; /* Default answer is DEAD. */
         	try {
              if (ping != null) {
                  results[i] = ping.isAlive(servers[i]);
              }
          	} catch (Exception e) {
              logger.error("Exception while pinging Server: '{}'", servers[i], e);
     	}
   	}
   return results;
 }
```

我们可以看到上面的代码采用了线性遍历ping服务实例的方式实现检查，该策略在当IPing的实现速度不理想或者Server列表过大时，可能会影响系统的性能。这时我们可以通过实现IPingStrategy接口并重写pingServers方法去扩展ping的策略。

* 实现了负载均衡的处理规则IRule对象，从BaseLoadBalancer中的chooseServer方法中可以看到，实际上负载均衡器将服务实例选择任务委托给了IRule实例中的choose方法。

```java
public Server chooseServer(Object key) {
	if (counter == null) {
        counter = createCounter();
    }
    counter.increment();
    if (rule == null) {
        return null;
    } else {
      try {
          //委托给了IRule来选择服务实例
           return rule.choose(key);
      } catch (Exception e) {
           logger.warn("LoadBalancer [{}]:  Error choosing server for key {}", name, key, e);
            return null;
            }
        }
    }
```

在BaseLoadBalancer中，默认初始化了RoudRobinRule为IRule的实现对象，RoudRobinRule实现了最基本且常用的线性负载均衡规则

* 启动ping任务，在BaseLoadBalancer的默认构造函数中，会启动一个检查Server 是否健康的定时任务，该任务默认的执行间隔是 10秒。

```java
public BaseLoadBalancer() {
	this.name = DEFAULT_NAME;
    this.ping = null;
    setRule(DEFAULT_RULE);
    setupPingTask();//开启定时任务
    lbStats = new LoadBalancerStats(DEFAULT_NAME);
}

void setupPingTask() {
    if (canSkipPing()) {
        return;
    }
    if (lbTimer != null) {
       	lbTimer.cancel();
    }
    lbTimer = new ShutdownEnabledTimer("NFLoadBalancer-PingTimer-" + name,
                true);
    lbTimer.schedule(new PingTask(), 0, pingIntervalSeconds * 1000);
    forceQuickPing();
}

//定时任务
class PingTask extends TimerTask {
	public void run() {
    	try {
            	new Pinger(pingStrategy).runPinger();
            } catch (Exception e) {
              
            }
        }
    }
//定时任务具体实现
public void runPinger() throws Exception {
	if (!pingInProgress.compareAndSet(false, true)) { 
    	return; // Ping in progress - nothing to do
    }
            
        
    Server[] allServers = null;
    boolean[] results = null;

    Lock allLock = null;
   	Lock upLock = null;

    try {
        //读锁      
    	allLock = allServerLock.readLock();
        allLock.lock();//获取读锁
        //所有的服务
        allServers = allServerList.toArray(new Server[allServerList.size()]);
        allLock.unlock();//释放读锁

        int numCandidates = allServers.length;
        //默认线性判断服务是否正常
        results = pingerStrategy.pingServers(ping, allServers);

        final List<Server> newUpList = new ArrayList<Server>();
        final List<Server> changedServers = new ArrayList<Server>();

        for (int i = 0; i < numCandidates; i++) {
        	boolean isAlive = results[i];
            Server svr = allServers[i];
            boolean oldIsAlive = svr.isAlive();

            svr.setAlive(isAlive);

            if (oldIsAlive != isAlive) {
            //如果状态不是alive，将服务放到变动队列中
            	changedServers.add(svr);
                        
             }

             if (isAlive) {
             	newUpList.add(svr);
             }
         }
             upLock = upServerLock.writeLock();
        	 //加一个写锁，替换队列
             upLock.lock();
             upServerList = newUpList;
             upLock.unlock();

   //将变动通知到订阅者             		notifyServerStatusChangeListener(changedServers);
            } finally {
                pingInProgress.set(false);
            }
        }

```

* 实现ILoadBalancer接口定义的负载均衡器应该具备的基本操作，如：addServers()方法

 ```java
public void addServers(List<Server> newServers) {
	if (newServers != null && newServers.size() > 0) {
    	try {
        	ArrayList<Server> newList = new ArrayList<Server>();
            newList.addAll(allServerList);
            newList.addAll(newServers);
            setServersList(newList);
        } catch (Exception e) {
            logger.error("LoadBalancer [{}]: Exception while adding Servers", name, e);
            }
        }
    }
 ```

向负载均衡器中添加新的服务实例列表，该实现将原来的服务实例清单与新增加的服务清单都加入到新创建的list集合(newList)中，然后调用setServersList方法对newList进行处理。

* chooseServer(Object key)方法，前面已经提到了。
* markServerDown(Server server):标识某个服务实例暂停服务。

```java
public void markServerDown(Server server) {
	if (server == null || !server.isAlive()) {
        return;
    }

    logger.error("LoadBalancer [{}]:  markServerDown called on [{}]", name, server.getId());
    server.setAlive(false);
        // forceQuickPing();
	notifyServerStatusChangeListener(singleton(server));
 }      
```

* getReachableServers()：获取可用的服务实例。

```java
public List<Server> getReachableServers() {
   return Collections.unmodifiableList(upServerList);
}
```

* getAllServers()：获取所有的服务实例列表。

```java
public List<Server> getAllServers() {
   return Collections.unmodifiableList(allServerList);
}
```



**DynamicServerListLoadBalancer**

DynamicServerListLoadBalancer是BaseLoadBalancer的子类，它是对基础负载均衡的扩展，它实现了服务实例清单在运行期的动态更新，同时还具备了过滤功能。

**ServerList**

ServerList是一个操作服务列表的对象，它的定义如下：

```java
public List<T> getInitialListOfServers();//获取初始化服务清单    
public List<T> getUpdatedListOfServers(); //获取更新的服务清单
```

在EurekaRibbonClientConfiguration类中，可以找到如何创建ServerList：

```java
@Bean
@ConditionalOnMissingBean
public ServerList<?> ribbonServerList(IClientConfig config, Provider<EurekaClient> eurekaClientProvider) {
	if (this.propertiesFactory.isSet(ServerList.class, serviceId)) {
			return this.propertiesFactory.get(ServerList.class, config, serviceId);
		}
		DiscoveryEnabledNIWSServerList discoveryServerList = new DiscoveryEnabledNIWSServerList(
				config, eurekaClientProvider);
		DomainExtractingServerList serverList = new DomainExtractingServerList(
				discoveryServerList, config, this.approximateZoneFromHostname);
		return serverList;
	}
```

DomainExtractingServerList实现

```java
//EurekaRibbonClientConfiguration
@Bean
@ConditionalOnMissingBean
public ServerList<?> ribbonServerList(IClientConfig config, Provider<EurekaClient> eurekaClientProvider) {
	if (this.propertiesFactory.isSet(ServerList.class, serviceId)) {
		return this.propertiesFactory.get(ServerList.class, config, serviceId);
	}
	DiscoveryEnabledNIWSServerList discoveryServerList = new DiscoveryEnabledNIWSServerList(
				config, eurekaClientProvider);
    //创建一个DomainExtractingServerList对象
		DomainExtractingServerList serverList = new DomainExtractingServerList(
				discoveryServerList, config, this.approximateZoneFromHostname);
		return serverList;
	}
```

在DomainExtractingServerList中创建了一个ServerList list，同时自己实现了getInitialListOfServers和getUpdatedListOfServers方法，但是在这两个方法内部其实是委托给了ServerList list 来实现具体的操作。ServerList由构造函数传入即它的实现类：DiscoveryEnabledNIWSServerList.

```java
public class DomainExtractingServerList implements ServerList<DiscoveryEnabledServer> {

	private ServerList<DiscoveryEnabledServer> list;
	private final RibbonProperties ribbon;

	private boolean approximateZoneFromHostname;

	public DomainExtractingServerList(ServerList<DiscoveryEnabledServer> list,
			IClientConfig clientConfig, boolean approximateZoneFromHostname) {
		this.list = list;
		this.ribbon = RibbonProperties.from(clientConfig);
		this.approximateZoneFromHostname = approximateZoneFromHostname;
	}

	@Override
	public List<DiscoveryEnabledServer> getInitialListOfServers() {
		List<DiscoveryEnabledServer> servers = setZones(this.list
				.getInitialListOfServers());
		return servers;
	}

	@Override
	public List<DiscoveryEnabledServer> getUpdatedListOfServers() {
		List<DiscoveryEnabledServer> servers = setZones(this.list
				.getUpdatedListOfServers());
		return servers;
	}
}
```

DiscoveryEnabledNIWSServerList中对getInitialListOfServers和getUpdatedListOfServers的实现：

```java
 @Override
  public List<DiscoveryEnabledServer> getInitialListOfServers(){
      return obtainServersViaDiscovery();
  }

    @Override
    public List<DiscoveryEnabledServer> getUpdatedListOfServers(){
        return obtainServersViaDiscovery();
    }
```

在这两个方法中通过都是通过调用obtainServersViaDiscovery方法实现,在这个方法中通过EurekaClient从服务注册中心获取具体的服务实例InstanceInfo列表，然后对这些实例进行遍历，将状态为up的实例转换成DiscoveryEnabledServer对象，最后将这些实例组织成列表返回。

```java
 private List<DiscoveryEnabledServer> obtainServersViaDiscovery() {
     //list集合，存放从eureka注册中心获取的服务实例
 	List<DiscoveryEnabledServer> serverList = new ArrayList<DiscoveryEnabledServer>();

        if (eurekaClientProvider == null || eurekaClientProvider.get() == null) {
            logger.warn("EurekaClient has not been initialized yet, returning an empty list");
            return new ArrayList<DiscoveryEnabledServer>();
        }
		//获取Eureka Client
        EurekaClient eurekaClient = eurekaClientProvider.get();
        if (vipAddresses!=null){
            //vipAddress逻辑上的服务名称
            for (String vipAddress : vipAddresses.split(",")) {
           		//通过服务名称获取注册中心的服务实例列表
                List<InstanceInfo> listOfInstanceInfo = eurekaClient.getInstancesByVipAddress(vipAddress, isSecure, targetRegion);
                for (InstanceInfo ii : listOfInstanceInfo) {
                    //如果服务实例的状态是UP
                    if (ii.getStatus().equals(InstanceStatus.UP)) {
                        //将IntanceInfo封装到DiscoveryEnabledServer
                        DiscoveryEnabledServer des = new DiscoveryEnabledServer(ii, isSecure, shouldUseIpAddr);
                        des.setZone(DiscoveryClient.getZone(ii));
                        serverList.add(des);
                    }
                }
                if (serverList.size()>0 && prioritizeVipAddressBasedServers){
                    break; // if the current vipAddress has servers, we dont use subsequent vipAddress based servers
                }
            }
        }
        return serverList;
    }
```

在DiscoveryEnabledNIWSServerList中通过EurekaClient获取服务注册中心获取到最新的服务实例清单后，返回到DomainExtractingServerList将继续通过setZones函数进行处理。

```java
private List<DiscoveryEnabledServer> setZones(List<DiscoveryEnabledServer> servers) {
	List<DiscoveryEnabledServer> result = new ArrayList<>();
	boolean isSecure = this.ribbon.isSecure(true);
	boolean shouldUseIpAddr = this.ribbon.isUseIPAddrForServer();
	for (DiscoveryEnabledServer server : servers) {
		result.add(new DomainExtractingServer(server, isSecure, shouldUseIpAddr,
		this.approximateZoneFromHostname));
		}
	return result;
}
```

**ServerListUpdater**

我们已经知道了如何从Eureka server中获取服务实例清单，以及更新本地的服务实例清单，接下来我们再看看是如何触发从服务注册中心获取服务实例。我们可以再DynamicServerListLoadBalancer中找到如下内容：

```java
protected final ServerListUpdater.UpdateAction updateAction = new ServerListUpdater.UpdateAction() {
        @Override
        public void doUpdate() {
            updateListOfServers();
        }
};
protected volatile ServerListUpdater serverListUpdater;
```

上面的ServerListUpdater就是用来对ServerList进行更新的。

```java
public interface ServerListUpdater {

    //用来更新 server list的接口
    public interface UpdateAction {
        void doUpdate();
    }
    //启动服务更新器，传入UpdateAction
    void start(UpdateAction updateAction);
	//停止服务更新器
    void stop();
	//获取最近的更新时间戳
    String getLastUpdate();
	//获取上一次更新到现在的时间，单位时毫秒
    long getDurationSinceLastUpdateMs();
	//获取错过的更新周期
    int getNumberMissedCycles();
	//获取核心线程数
    int getCoreThreads();
}
```

在DynamicServerListLoadBalancer中默认使用PollingServerListUpdater来动态更新服务列表，在该实例中通过start方法启动服务更新器：

```java
public synchronized void start(final UpdateAction updateAction) {
	if (isActive.compareAndSet(false, true)) {
        final Runnable wrapperRunnable = new Runnable() {
        	@Override
           	public void run() {
            if (!isActive.get()) {
                 if (scheduledFuture != null) {
                     scheduledFuture.cancel(true);
                  }
                 return;
             }
             try {
                  updateAction.doUpdate();
                  lastUpdated = System.currentTimeMillis();
              } catch (Exception e) {
                  logger.warn("Failed one update cycle", e);
                }
           	}
      };

      scheduledFuture = getRefreshExecutor().scheduleWithFixedDelay(
          wrapperRunnable,
          initialDelayMs,
          refreshIntervalMs,
          TimeUnit.MILLISECONDS
       );
        } else {
            logger.info("Already active, no-op");
        }
    }
```

上面的代码中先创建一个Runnable对象，在这个对象的run方法中调用UpdateAction的doUpdate方法来更新服务列表。最后再为这个Runnabled对象创建一个定时任务来执行，再这个定时任务中会传入initialDelayMs、refreshIntervalMs两个参数，它们的默认值是1000和3*1000，也就是说更新服务在初始化后延迟1秒执行，并且以30秒为周期重复执行。

**ServerListFilter**

在了解了更新服务器是如何启动后，我们再回到DynamicServerListLoadBalancer中的doUpdate()方法，这个方法是调用updateListOfServers()方法：

```java
public void updateListOfServers() {
	List<T> servers = new ArrayList<T>();
    if (serverListImpl != null) {
        servers = serverListImpl.getUpdatedListOfServers();
        if (filter != null) {
                servers = filter.getFilteredListOfServers(servers);
            }
        }
        updateAllServerList(servers);
    }
```

在上面的代码中通过getUpdatedListOfServers()从Eureka server 注册中心获取服务实例，然后后面的getFilteredListOfServers方法，主要用来对服务实例列表进行过滤，通过传入的服务实例清单，再根据一些规则返回过滤后的服务实例清单。

ServerListFilter的实现类

* AbstractServerListFilter这是一个抽象过滤器，在这里定义了过滤时需要的LoadBalancerStats对象，该对象中存储了关于负载均衡的一些属性和统计信息。

```java
public abstract class AbstractServerListFilter<T extends Server> implements ServerListFilter<T> {

    private volatile LoadBalancerStats stats;
    
    public void setLoadBalancerStats(LoadBalancerStats stats) {
        this.stats = stats;
    }
    
    public LoadBalancerStats getLoadBalancerStats() {
        return stats;
    }

}
```



* ZoneAffinityServerListFilter该过滤器是基于区域感知的方式实现服务实例的过滤，它会过滤掉那些与消费者自身不是同处一个区域的实例。

```java
public List<T> getFilteredListOfServers(List<T> servers) {
	if (zone != null && (zoneAffinity || zoneExclusive) && servers !=null && servers.size() > 0){
//过滤服务        
    	List<T> filteredServers = Lists.newArrayList(Iterables.filter(
                    servers, this.zoneAffinityPredicate.getServerOnlyPredicate()));
        //过滤后的服务不会马上返回，还需要判断是否开启区域感知
        if (shouldEnableZoneAffinity(filteredServers)) {
                return filteredServers;
        } else if (zoneAffinity) {
                overrideCounter.increment();
        }
    }
    return servers;
}
```

上面的代码中在过滤之后并不会马上返回过滤的结果，而是先调用shouldEnableZoneAffinity方法来判断是非开启启用区域感知的功能。

```java
private boolean shouldEnableZoneAffinity(List<T> filtered) {    
	if (!zoneAffinity && !zoneExclusive) {
        return false;
    }
    if (zoneExclusive) {
        return true;
    }
    LoadBalancerStats stats = getLoadBalancerStats();
    if (stats == null) {
        return zoneAffinity;
    } else {  
        ZoneSnapshot snapshot = stats.getZoneSnapshot(filtered);
        double loadPerServer = snapshot.getLoadPerServer();
        int instanceCount = snapshot.getInstanceCount();            
        int circuitBreakerTrippedCount = snapshot.getCircuitTrippedCount();
        //断路器断开数/实例数量 >= 0.8
        if (((double) circuitBreakerTrippedCount) / instanceCount >= blackOutServerPercentageThreshold.get()
        //实例平均负载 >= 0.6    
                    || loadPerServer >= activeReqeustsPerServerThreshold.get()
       //可用实例(实例数量－断路器断开数) <2    
                    || (instanceCount - circuitBreakerTrippedCount) < availableServersThreshold.get()) {
                
            return false;
          } else {
                return true;
          }
            
        }
    }
```

上面的代码判断是否开启区域感知从三个方面来判断：

1. blackOutServerPercent：故障实例百分比，断路器断开数/实例数量>=0.8
2. activeReqeustsPerServer：实例平均负载>=0.6
3. availableServersThreshold：可用实例，(实例数量－断路器开数)<2

* DefaultNIWSServerListFilter完全继承ZoneAffinityServerListFilter，是默认的NIWS过滤器
* ServerListSubsetFilter也是继承自ZoneAffinityServerListFilter,该过滤器非常适合大规模服务器集群，因为它可以产生一个区域感知结果的子集列表，同时它还能够通过比较服务实例的通信失败数量和并发连接数来判定该服务是否健康来选择性的从服务实例列表中剔除那些不够健康的实例，它的实现主要分为以下三步：

1. 获取区域感知的过滤结果，作为候选的服务实例清单
2. 从当前消费着维护的服务实例清单中剔除那些不够健康的实例，不够健康的标准如下：

  a. 服务实例的并发连接数超过客户端配置的值，默认是0。配置参数是：```<clientName>.<nameSpace>.ServerListSubsetFilter.eliminationConnectionThresold```。

 b. 服务实例的失败次数超过客户端配置的值，默认是0，配置参数：```<clientName>.<nameSpace>.ServerListSubsetFilter.eliminationFailureThresold```。

c. 经过过滤后，如果剔除比例小于客户端默认的配置的百分比，那么先对服务实例进行排序，再将最不健康的实例剔除掉，直到达到配置的剔除百分比。

3. 在完成剔除后，清单中已经少了10%( 默认值)，最后通过随机的方式从候选清单中选出一批实例加入到过滤后的清单中，以保持服务清单与原来的数量一致（默认20）。

```java
public List<T> getFilteredListOfServers(List<T> servers) {
    //调用ZoneAffinityServerListFilter的方法，过滤服务实例
    //开启区域感知
 	List<T> zoneAffinityFiltered = super.getFilteredListOfServers(servers);
    //转成set集合
    Set<T> candidates = Sets.newHashSet(zoneAffinityFiltered);
    //历史服务实例集合，用来存放新的被选出来的服务实例
    Set<T> newSubSet = Sets.newHashSet(currentSubset);
    //获取负载均衡信息
    LoadBalancerStats lbStats = getLoadBalancerStats();
    for (T server: currentSubset) {
         if (!candidates.contains(server)) {
             //如果新过滤的服务实例中不包含该实例，说明该实例可能已经下线，从newSubSet集合中删除
             newSubSet.remove(server);
         } else {
             ServerStats stats = lbStats.getSingleServerStat(server);
              //如果服务实例的并发连接数超过客户端配置的值
             //服务实例的失败次数超过客户端配置的值
             if (stats.getActiveRequestsCount() > eliminationConnectionCountThreshold.get()
                        || stats.getFailureCount() > eliminationFailureCountThreshold.get()) {
                    newSubSet.remove(server);
                    candidates.remove(server);
                }
            }
        }
    	//默认的实例子集数量20
        int targetedListSize = sizeProp.get();
    	//通过前面的判断，被剔除的服务实例数量
        int numEliminated = currentSubset.size() - newSubSet.size();
    	//默认需要剔除实例数量，实例子集数量(20)＊剔除百分比(0.1)
        int minElimination = (int) (targetedListSize * eliminationPercent.get());
    	//强制剔除的服务实例数量
        int numToForceEliminate = 0;
    	//newSubSet大于targetedListSize,则需要把多余的服务实例强制删除掉
        if (targetedListSize < newSubSet.size()) {
            // size is shrinking
            numToForceEliminate = newSubSet.size() - targetedListSize;
            //如果已经剔除的服务数量小于默认剔除比例
        } else if (minElimination > numEliminated) {
            //需要继续被剔除的服务实例数量
            numToForceEliminate = minElimination - numEliminated; 
        }
        //如果需要强制剔除的服务实例数量大于newSubSet的大小
        if (numToForceEliminate > newSubSet.size()) {
            //为了防止越界，将numToForceEliminate设置为newSubSet.size()
            numToForceEliminate = newSubSet.size();
        }
		//大于0，说明前面剔除的比例还不够
        if (numToForceEliminate > 0) {
            //如果剔除比例小于客户端默认配置百分比
            List<T> sortedSubSet = Lists.newArrayList(newSubSet);  
            //健康排序
            Collections.sort(sortedSubSet, this);
            List<T> forceEliminated = sortedSubSet.subList(0, numToForceEliminate);
            //继续剔除
            newSubSet.removeAll(forceEliminated);
            candidates.removeAll(forceEliminated);
        }
        //如果剔除后的数量小于默认的实例子集大小
        if (newSubSet.size() < targetedListSize) {
           //需要往newSubSet集合中添加服务实例的数量
            int numToChoose = targetedListSize - newSubSet.size();
            //从candidates中的删除过滤后的服务实例，防止重复添加
            candidates.removeAll(newSubSet);
            
            if (numToChoose > candidates.size()) {
                candidates = Sets.newHashSet(zoneAffinityFiltered);
                candidates.removeAll(newSubSet);
            }
            
            //从candidates随即选出numToChoose个实例
            List<T> chosen = randomChoose(Lists.newArrayList(candidates), numToChoose);
            for (T server: chosen) 
                //将选出的实例添加到过滤后的服务实例中
                newSubSet.add(server);
            }
        }
        currentSubset = newSubSet;       
        return Lists.newArrayList(newSubSet);            
    }
```

* ZonePreferenceServerListFilter

SpringCloud 整合Eureka 和Ribbon时新增的过滤器。若使用Spring Cloud 整合Eureka 和Ribbon时会使用该过滤器。它实现了通过配置Eureka实例元数据的所属区域(Zone)来过滤出同区域的服务实例。

它的实现非常简单，首先通过父类ZoneAffinityServerListFilter的过滤器来获得“区域感知”的服务实例列表，然后遍历这个结果，取出根据消费者配置预设的区域Zone来进行过滤，如果过滤的结果是空就直接返回父类获取的结果，如果不为空就返回通过消费者配置的Zone过滤后的结果。

```java
public List<Server> getFilteredListOfServers(List<Server> servers) {
    //调用父类的方法，开启区域感知
		List<Server> output = super.getFilteredListOfServers(servers);
    //如果父类没有过滤掉服务实例，则根据消费着的zone来进行过滤
		if (this.zone != null && output.size() == servers.size()) {
			List<Server> local = new ArrayList<>();
			for (Server server : output) {
				if (this.zone.equalsIgnoreCase(server.getZone())) {
					local.add(server);
				}
			}
			if (!local.isEmpty()) {
				return local;
			}
		}
		return output;
	}
```

* ZoneAwareLoadBalancer

ZoneAwareLoadBalancer负载均衡器是对DynamicServerListLoadBalancer的扩展。在DynamicServerListLoadBalancer中并没有重写选择的具体服务实例的chooseServer函数，所以它依然会选择BaseLoadBalancer中实现的算法。使用RoundRobinRule规则，以线性轮询的方式来选择调用的服务实例，该算法实现简单并没有区域的概念，所以它会把所有实例视为一个zone下的节点来看待，这样就会周期性地产生跨区域访问的情况。

由于跨区域会产生更高的延迟，这些实例主要以防止区域性故障实现高可用为目的而不能作为常规访问的实例，所以在多区域部署的情况下会有一定的性能问题，而负载均衡器则可以避免这种情况。

在ZoneAwareLoadBalancer中没有重写setServersList，说明服务实例清单的更新主逻辑没有修改。但是重写了setServerListForZones函数。

首先可以看到父类DynamicServerListLoadBalancer中setServersList的实现：

```java
public void setServersList(List lsrv) {
	super.setServersList(lsrv);
   	List<T> serverList = (List<T>) lsrv;
    //根据zone分类存放server
    Map<String, List<Server>> serversInZones = new HashMap<String, List<Server>>();
    for (Server server : serverList) {
        //确保serverstats已经创建
    	getLoadBalancerStats().getSingleServerStat(server);
        String zone = server.getZone();
        if (zone != null) {
            zone = zone.toLowerCase();
          List<Server> servers = serversInZones.get(zone);
                if (servers == null) {
                    servers = new ArrayList<Server>();
                    serversInZones.put(zone, servers);
                }
                servers.add(server);
            }
        }
    	//子类重写
        setServerListForZones(serversInZones);
    }
	//为每个zone创建一个ZoneStats，并保存到zoneStatsMap中
	//ZoneStats用于存储每个Zone的一些状态和统计信息
    protected void setServerListForZones(Map<String, List<Server>> zoneServersMap) {
            getLoadBalancerStats().updateZoneServerMapping(zoneServersMap);
    }
public void updateZoneServerMapping(Map<String, List<Server>> map) {
        upServerListZoneMap = new ConcurrentHashMap<String, List<? extends Server>>(map);
        // make sure ZoneStats object exist for available zones for monitoring purpose
        for (String zone: map.keySet()) {
            getZoneStats(zone);
        }
    }
private ZoneStats getZoneStats(String zone) {
        zone = zone.toLowerCase();
    	//根据zone从zoneStatsMap中获取zonestats对象
        ZoneStats zs = zoneStatsMap.get(zone);
        if (zs == null){
            //不存在，就创建一个
            zoneStatsMap.put(zone, new ZoneStats(this.getName(), zone, this));
            zs = zoneStatsMap.get(zone);
        }
        return zs;
    }
```

setServerListForZones在ZoneAwareLoadBalancer中的实现：

```java
protected void setServerListForZones(Map<String, List<Server>> zoneServersMap) {
        super.setServerListForZones(zoneServersMap);
        if (balancers == null) {
            //创建一个map集合来存放服务实例
            balancers = new ConcurrentHashMap<String, BaseLoadBalancer>();
        }
        for (Map.Entry<String, List<Server>> entry: zoneServersMap.entrySet()) {
   String zone = entry.getKey().toLowerCase();
            //创建完负载均衡器后马上调用setServersList函数，
            //为其设置对应zone区域的实例清单
    getLoadBalancer(zone).setServersList(entry.getValue());
   }
       //对Zone区域中实例清单的检查，看看是否有Zone区域下已经没有实例了，是的话就将balancers中对应Zone区域的实例列表清空，该操作的作用是为了后续选择节点时，防止过时的 Zone区域统计信息干扰具体的实例的选择算法。
        for (Map.Entry<String, BaseLoadBalancer> existingLBEntry: balancers.entrySet()) {
            if (!zoneServersMap.keySet().contains(existingLBEntry.getKey())) {
                existingLBEntry.getValue().setServersList(Collections.emptyList());
            }
        }
    }


//根据zone获取负载均衡器
BaseLoadBalancer getLoadBalancer(String zone) {
        zone = zone.toLowerCase();
        BaseLoadBalancer loadBalancer = balancers.get(zone);
        if (loadBalancer == null) {
        	//创建规则
        	IRule rule = cloneRule(this.getRule());
            loadBalancer = new BaseLoadBalancer(this.getName() + "_" + zone, rule, this.getLoadBalancerStats());
            BaseLoadBalancer prev = balancers.putIfAbsent(zone, loadBalancer);
            if (prev != null) {
            	loadBalancer = prev;
            }
        } 
        return loadBalancer;        
    }
```

我们再来看看ZoneAwareLoadBalancer是如何选择服务实例的：

```java
public Server chooseServer(Object key) {
    //只有党负载均衡器中维护的实例所属的Zone区域的个数大于1的时候才会执行这里的选择策略，否则还是将使用父类的实现
        if (!ENABLED.get() || getLoadBalancerStats().getAvailableZones().size() <= 1) {
            logger.debug("Zone aware logic disabled or there is only one zone");
            return super.chooseServer(key);
        }
        Server server = null;
        try {
            LoadBalancerStats lbStats = getLoadBalancerStats();
            //创建快照，key为zone
            Map<String, ZoneSnapshot> zoneSnapshot = ZoneAvoidanceRule.createSnapshot(lbStats);
            logger.debug("Zone snapshots: {}", zoneSnapshot);
            if (triggeringLoad == null) {
                triggeringLoad = DynamicPropertyFactory.getInstance().getDoubleProperty(
                        "ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".triggeringLoadPerServerThreshold", 0.2d);
            }

            if (triggeringBlackoutPercentage == null) {
                triggeringBlackoutPercentage = DynamicPropertyFactory.getInstance().getDoubleProperty(
                        "ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".avoidZoneWithBlackoutPercetage", 0.99999d);
            }
            //挑选可用区域
            Set<String> availableZones = ZoneAvoidanceRule.getAvailableZones(zoneSnapshot, triggeringLoad.get(), triggeringBlackoutPercentage.get());
            logger.debug("Available zones: {}", availableZones);
            if (availableZones != null &&  availableZones.size() < zoneSnapshot.keySet().size()) {
                String zone = ZoneAvoidanceRule.randomChooseZone(zoneSnapshot, availableZones);
                logger.debug("Zone chosen: {}", zone);
                if (zone != null) {
                    //根据zone获取负载均衡器
                    BaseLoadBalancer zoneLoadBalancer = getLoadBalancer(zone);
                    //获取具体的服务
                    server = zoneLoadBalancer.chooseServer(key);
                }
            }
        } catch (Exception e) {
            logger.error("Error choosing server using zone aware logic for load balancer={}", name, e);
        }
        if (server != null) {
            return server;
        } else {
            logger.debug("Zone avoidance logic is not invoked.");
            return super.chooseServer(key);
        }
    }
```

* 调用ZoneAvoidanceRule的createSnapshot方法，为当前负载均衡器中所有的Zone区域分别创建快照。返回的值保存在map集合中，key为Zone的名称。

```java
//快照对象结构
public class ZoneSnapshot {
    final int instanceCount;
    final double loadPerServer;
    final int circuitTrippedCount;
    final int activeRequestsCount;
    ....
}
```

* 调用ZoneAvoidanceRule中的静态方法getAvailableZones来获取可用的Zone区域集合，在该函数中会通过Zone区域快照中的统计数据来实现可用区域的挑选。

```java
public static Set<String> getAvailableZones(
            Map<String, ZoneSnapshot> snapshot, double triggeringLoad,
            double triggeringBlackoutPercentage) {
        if (snapshot.isEmpty()) {
            return null;
        }
        Set<String> availableZones = new HashSet<String>(snapshot.keySet());
        if (availableZones.size() == 1) {
            return availableZones;
        }
        Set<String> worstZones = new HashSet<String>();
        double maxLoadPerServer = 0;
        boolean limitedZoneAvailability = false;

        for (Map.Entry<String, ZoneSnapshot> zoneEntry : snapshot.entrySet()) {
            String zone = zoneEntry.getKey();
            ZoneSnapshot zoneSnapshot = zoneEntry.getValue();
            //实例数量为零的Zone区域
            int instanceCount = zoneSnapshot.getInstanceCount();
            if (instanceCount == 0) {
                availableZones.remove(zone);
                limitedZoneAvailability = true;
            } else {
                //平均负载
                double loadPerServer = zoneSnapshot.getLoadPerServer();
                //故障率大于阀值或者负载率小于0
                if (((double) zoneSnapshot.getCircuitTrippedCount())
                        / instanceCount >= triggeringBlackoutPercentage
                        || loadPerServer < 0) {
                    availableZones.remove(zone);
                    limitedZoneAvailability = true;
                } else {
                    
                    if (Math.abs(loadPerServer - maxLoadPerServer) < 0.000001d) {
                       
                        worstZones.add(zone);
                    } else if (loadPerServer > maxLoadPerServer) {
                        //将负载率最高的zone设置为最差的zone
                        maxLoadPerServer = loadPerServer;
                        worstZones.clear();
                        worstZones.add(zone);
                    }
                }
            }
        }
		//如果上述符合剔除要求的区域，同时小于阀值(20%)
        if (maxLoadPerServer < triggeringLoad && !limitedZoneAvailability) {
            // zone override is not needed here
            //直接返回可用区域
            return availableZones;
        }
    	//否则随机从最坏Zone区域删除一个
        String zoneToAvoid = randomChooseZone(snapshot, worstZones);
        if (zoneToAvoid != null) {
            availableZones.remove(zoneToAvoid);
        }
        return availableZones;

    }
```

1. 首先它会剔除符合这些规则的Zone区域：所属实例数为零的Zone区域；Zone区域实例的平均负载小于零，或者实例故障率（断路器断开次数／实例数）大于阀值(默认值为0.99999)。
2. 然后根据Zone区域的实例平均负载计算出最差的Zone区域，这里的最差指的是实例负载最高的Zone区域。
3. 如果上面的过程中没有符合剔除要求的区域，同时实例最大平均负载小于阀值（默认为20%），就直接返回所有Zone区域为可用区域。否则从最坏Zone区域集合中随机选择一个，将它从可用Zone区域集合中剔除。

* 当获得的可用区域集合不为空，并且个数小于Zone区域总数，就随机选择一个Zone区域。
* 在确定来某个Zone区域后，则获取了对应Zone区域的服务均衡器，并调用chooseServer来选择具体的服务实例。在这里,IRule接口的实现会使用ZoneAvoid来挑选具体的服务实例。






