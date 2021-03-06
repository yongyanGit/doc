#### Eureka 服务注册、续约、获取注册信息

* 服务注册，当Eureka Client 向Eureka Server 注册时，它提供自身的元数据，比如ip地址、端口运行状况，服务端在收到这个Rest请求后，会将元数据信息存放在一个双层Map 中，其中第一层的Key为服务名称，第二层的key为具体服务的实例名。
* 服务续约，客户端默认每隔30s发送一次心跳来续约，通过续约来告诉Eureka Server来告诉客户端一直存活着。如果Eureka Server 默认在90s没有收到客户端的续约，它就会将实例从其注册表中删除。
* 获取服务注册列表信息，Eureka客户端从Eureka 服务端获取注册表信息，并将其缓存在本地，客户端会使用该信息查找其他服务，从而进行远程调用。该注册表信息默认每30秒更新一次。
* 服务下线， Eureka 客户端在程序关闭时向Eureka服务端发送取消请求，发送请求后，该客户端信息会在服务端的实例注册表中删除。
* 服务剔除，在默认的情况下，如果Eureka 客户端在90s没有向Eureka服务端发送续约，Eureka 服务端会将该服务的实例从服务注册表中删除。



1. 服务注册

1.1. 客户端请求

服务注册在```DiscoveryClient```类中我们可以找到它的构造函数，可以找到下面这个函数：

```java
private void initScheduledTasks() {
    
	if (clientConfig.shouldRegisterWithEureka()) {
    	....
        // InstanceInfo replicator
        instanceInfoReplicator = new InstanceInfoReplicator(
        this,
      instanceInfo,clientConfig.getInstanceInoReplicationIntervalSeconds(),
        2);         instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
        } else {
            logger.info("Not registering with Eureka server per configuration");
        }
    }
```

从上面的函数中可以看到一个与注册相关的if判断语句，```if(clientConfig.shouldRegisterWithEureka())```，在这个判断语句中会创建一个```InstanceInfoReplicator```对象，这个实例会启动一个定时任务，而这个定时任务的具体工作可以看这个类的run()方法，

```java
public void run() {
	try {
      discoveryClient.refreshInstanceInfo();
      Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
      if (dirtyTimestamp != null) {
            discoveryClient.register();
            instanceInfo.unsetIsDirty(dirtyTimestamp);
      }
   } catch (Throwable t) {
   logger.warn("There was a problem with the instance info replicator", t);
   } finally {
     Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
     scheduledPeriodicRef.set(next);
   }
}
```

```discoveryClient.register()```这一行真正触发调用注册的地方，继续进入register()方法

```java
boolean register() throws Throwable {
 logger.info(PREFIX + "{}: registering service...", appPathIdentifier);
 EurekaHttpResponse<Void> httpResponse;
 try {
 httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
 } catch (Exception e) {
 logger.warn(PREFIX + "{} - registration failed {}", appPathIdentifier, e.getMessage(), e);
 throw e;
 }
 if (logger.isInfoEnabled()) {
    logger.info(PREFIX + "{} - registration status: {}", appPathIdentifier, httpResponse.getStatusCode());
  }
        return httpResponse.getStatusCode() == 204;
}
```

Eureka client向Eureka Server注册是通过REST请求方式进行的，并且请求参数是一个```com.netflix.appinfo.InstanceInfo```对象。

1.2. 服务端接收请求

Eureka Server对于各类的REST请求定义都在```com.netflix.eureka.resources```包下，用来接收Eureka client的注册请求在ApplicationResource类中：

```
@POST
@Consumes({"application/json", "application/xml"})
public Response addInstance(InstanceInfo info,                            @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
   registry.register(info, "true".equals(isReplication));
   return Response.status(204).build();  // 204 to be backwards compatible
}
```

接着进入register()

```java
 public void register(final InstanceInfo info, final boolean isReplication) {
   int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
   if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
    leaseDuration = info.getLeaseInfo().getDurationInSecs();
   }
   super.register(info, leaseDuration, isReplication);
   replicateToPeers(Action.Register, info.getAppName(), info.getId(), info, null, isReplication);
    }
```

其实注册的逻辑在它的父类中进行的，注册中心存储了两层Map结构，第一层的key存储服务名：InstanceInfo中的 appName，第二层的key存储实例名：InstanceInfo中的instanceId属性。我们接着进入它的父类中的方法，发现如下代码：

```java
Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
REGISTER.increment(isReplication);
if (gMap == null) {
  final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
   gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
   if (gMap == null) {
        gMap = gNewMap;
   }
}
```

从上面代码可以看到，首先它会根据客户端传过来的appname查询本地缓存，如果缓存为空，则它会创建一个map集合，并且以appname为key，将该新创建的map集合缓存到Eureka server本地。往下看，会发现真正传过来的注册信息InstanceInfo会封装成一个Lease对象，然后存放到新创建的map集合中。

```java
Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
 if (existingLease != null) {
                lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
 }
 gMap.put(registrant.getId(), lease);
```

Lease类

```java
public class Lease<T> {

    enum Action {
        Register, Cancel, Renew
    };

    public static final int DEFAULT_DURATION_IN_SECS = 90;
	//传进来的InstanceInfo实例
    private T holder;
    private long evictionTimestamp;//取消注册时间
    private long registrationTimestamp;//注册时间戳
    private long serviceUpTimestamp;//开始服务时间戳
    // Make it volatile so that the expiration task would see this quicker
    private volatile long lastUpdateTimestamp;//最后更新时间戳
    private long duration;//租约持续时长

    public Lease(T r, int durationInSecs) {
        holder = r;
        registrationTimestamp = System.currentTimeMillis();
        lastUpdateTimestamp = registrationTimestamp;
        duration = (durationInSecs * 1000);

    }
```

2. 服务续约

2.1. 客户端请求

相对与服务获取，服务续约与服务注册在同一个if逻辑中，因为服务注册完之后需要一个心跳去续约，防止被剔除。

```java
boolean renew() {
        EurekaHttpResponse<InstanceInfo> httpResponse;
        try {
            httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), instanceInfo.getId(), instanceInfo, null);
            logger.debug(PREFIX + "{} - Heartbeat status: {}", appPathIdentifier, httpResponse.getStatusCode());
            if (httpResponse.getStatusCode() == 404) {
                REREGISTER_COUNTER.increment();
                logger.info(PREFIX + "{} - Re-registering apps/{}", appPathIdentifier, instanceInfo.getAppName());
                long timestamp = instanceInfo.setIsDirtyWithTime();
                boolean success = register();
                if (success) {
                    instanceInfo.unsetIsDirty(timestamp);
                }
                return success;
            }
            return httpResponse.getStatusCode() == 200;
        } catch (Throwable e) {
            logger.error(PREFIX + "{} - was unable to send heartbeat!", appPathIdentifier, e);
            return false;
        }
    }
```

2.2. 服务端接收心跳

服务端接收心跳请求定义在```com.netflix.eureka.resources.InstanceResource```类中。

```java
 public Response renewLease(
            @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication,
            @QueryParam("overriddenstatus") String overriddenStatus,
            @QueryParam("status") String status,
            @QueryParam("lastDirtyTimestamp") String lastDirtyTimestamp) {
        boolean isFromReplicaNode = "true".equals(isReplication);
        boolean isSuccess = registry.renew(app.getName(), id, isFromReplicaNode);

        // Not found in the registry, immediately ask for a register
        if (!isSuccess) {
            logger.warn("Not Found (Renew): {} - {}", app.getName(), id);
            return Response.status(Status.NOT_FOUND).build();
        }
        // Check if we need to sync based on dirty time stamp, the client
        // instance might have changed some value
        Response response = null;
        if (lastDirtyTimestamp != null && serverConfig.shouldSyncWhenTimestampDiffers()) {
            response = this.validateDirtyTimestamp(Long.valueOf(lastDirtyTimestamp), isFromReplicaNode);
            // Store the overridden status since the validation found out the node that replicates wins
            if (response.getStatus() == Response.Status.NOT_FOUND.getStatusCode()
                    && (overriddenStatus != null)
                    && !(InstanceStatus.UNKNOWN.name().equals(overriddenStatus))
                    && isFromReplicaNode) {
                registry.storeOverriddenStatusIfRequired(app.getAppName(), id, InstanceStatus.valueOf(overriddenStatus));
            }
        } else {
            response = Response.ok().build();
        }
        logger.debug("Found (Renew): {} - {}; reply status={}", app.getName(), id, response.getStatus());
        return response;
    }
```

```registry.renew(app.getName(), id, isFromReplicaNode);```处理客户端发送的续约请求

在该方法中，首先会判断服务端是否存在该客户端的注册信息，如果发现不存在，则返回false。然后客户端发现返回false后，Eureka client会重新注册。

```java
 Map<String, Lease<InstanceInfo>> gMap = registry.get(appName);
        Lease<InstanceInfo> leaseToRenew = null;
        if (gMap != null) {
            leaseToRenew = gMap.get(id);
        }
        if (leaseToRenew == null) {
            RENEW_NOT_FOUND.increment(isReplication);
            logger.warn("DS: Registry: lease doesn't exist, registering resource: {} - {}", appName, id);
            return false;
        }
```

接着如果存在该信息，则会更新它的更新时间。

```java
renewsLastMin.increment();
leaseToRenew.renew();
```

```java
public void renew() {
   lastUpdateTimestamp = System.currentTimeMillis() + duration;
 }
```

在InstanceResource的renewLease方法中还会检查Eureka client传过来的lastDirtyTimestamp时间，如果该时间大于注册中心的时间，则会返回false，然后客户端重新注册。

3. Eureka 注册实例下线

3.1. Eureka 客户端

```java
public synchronized void shutdown() {
   if (applicationInfoManager != null
      && clientConfig.shouldRegisterWithEureka()
     && clientConfig.shouldUnregisterOnShutdown()) {
     applicationInfoManager.setInstanceStatus(InstanceStatus.DOWN);
      unregister();
          
   }
 }
```

将实例的的状态设置为down,然后调用unregister()方法

```java
void unregister() {
  // It can be null if shouldRegisterWithEureka == false
  if(eurekaTransport != null && eurekaTransport.registrationClient != null) {
   try {
    logger.info("Unregistering ...");
    EurekaHttpResponse<Void> httpResponse = 			eurekaTransport.registrationClient.cancel(instanceInfo.getAppName(), instanceInfo.getId());
  logger.info(PREFIX + "{} - deregister  status: {}", appPathIdentifier,  	httpResponse.getStatusCode());
  } catch (Exception e) {
  logger.error(PREFIX + "{} - de-registration failed{}", appPathIdentifier, e.getMessage(), e);
            }
        }
    }


public EurekaHttpResponse<Void> cancel(String appName, String id) {
        String urlPath = "apps/" + appName + '/' + id;
        ClientResponse response = null;
        try {
            Builder resourceBuilder = jerseyClient.resource(serviceUrl).path(urlPath).getRequestBuilder();
            addExtraHeaders(resourceBuilder);
            response = resourceBuilder.delete(ClientResponse.class);
            return anEurekaHttpResponse(response.getStatus()).headers(headersOf(response)).build();
        } finally {
            if (logger.isDebugEnabled()) {
                logger.debug("Jersey HTTP DELETE {}/{}; statusCode={}", serviceUrl, urlPath, response == null ? "N/A" : response.getStatus());
            }
            if (response != null) {
                response.close();
            }
        }
    }


```

3.2.  Eureka server 接收下线请求

```java
public Response cancelLease(
   @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
   try {
     boolean isSuccess = registry.cancel(app.getName(), id,
                "true".equals(isReplication));
       if (isSuccess) {
             logger.debug("Found (Cancel): {} - {}", app.getName(), id);
             return Response.ok().build();
        } else {
           logger.info("Not Found (Cancel): {} - {}", app.getName(), id);
            return Response.status(Status.NOT_FOUND).build();
        }
    } catch (Throwable e) {
            logger.error("Error (cancel): {} - {}", app.getName(), id, e);
            return Response.serverError().build();
    }

}
```

Eureka server接收的下线请求后，会调用父类的cancel()方法

```java
protected boolean internalCancel(String appName, String id, boolean isReplication) {
try {
	read.lock();
    CANCEL.increment(isReplication);
    //根据appName获取注册中心的缓存
    Map<String, Lease<InstanceInfo>> gMap = registry.get(appName);
    Lease<InstanceInfo> leaseToCancel = null;
    if (gMap != null) {
        //如果缓存不为空，清除掉
        leaseToCancel = gMap.remove(id);
    }
    synchronized (recentCanceledQueue) {
    	recentCanceledQueue.add(new Pair<Long, String>(System.currentTimeMillis(), appName + "(" + id + ")"));
            }
            InstanceStatus instanceStatus = overriddenInstanceStatusMap.remove(id);
            if (instanceStatus != null) {
                logger.debug("Removed instance id {} from the overridden map which has value {}", id, instanceStatus.name());
            }
            if (leaseToCancel == null) {
                //如果租约不存在，返回下线失败
                CANCEL_NOT_FOUND.increment(isReplication);
                logger.warn("DS: Registry: cancel failed because Lease is not registered for: {}/{}", appName, id);
                return false;
            } else {
                leaseToCancel.cancel();//设置实例下线时间
                InstanceInfo instanceInfo = leaseToCancel.getHolder();
                String vip = null;
                String svip = null;
                if (instanceInfo != null) {
                    instanceInfo.setActionType(ActionType.DELETED);
                    //将下线记录添加到变动队列
                    recentlyChangedQueue.add(new RecentlyChangedItem(leaseToCancel));
                    instanceInfo.setLastUpdatedTimestamp();
                    vip = instanceInfo.getVIPAddress();
                    svip = instanceInfo.getSecureVipAddress();
                }
                //设置响应缓存(ResponseCache)过期
                invalidateCache(appName, vip, svip);
                logger.info("Cancelled instance {}/{} (replication={})", appName, id, isReplication);
                return true;
            }
        } finally {
            read.unlock();
        }
    }
```

#### 4. 自我保护机制

当Eureka server节点在短时间内丢过多的客户端时就会进入自我保护模式，一旦进入该模式，Eureka server就会保护服务注册表中的信息，不再删除注册表中的数据。

在Eureka server中有2个关键的字段来判断什么时候开启自我保护：

```java
protected volatile int numberOfRenewsPerMinThreshold;//期望最小每分钟续租次数
protected volatile int expectedNumberOfRenewsPerMin;//期望最大每分钟续租次数
```

```expectedNumberOfRenewsPerMin```的计数公式：```expectedNumberOfRenewsPerMin```=当前注册的应用实例数*2。默认情况下，注册的应用实例每半分钟续约一次，那么一分钟心跳两次。

```numberOfRenewsPerMinThreshold```的计算公式：```expectedNumberOfRenewsPerMin```*```eureka.renewalPercentThreshold```（续租百分比），```eureka.renewalPercentThreshold```的默认值为```0.85```。

当Eureka server每分钟接收到的心跳次数小于```numberOfRenewsPerMinThreshold```时，并且开启了`eureka.enableSelfPreservation = true`时，会触动自动保护机制，不再自动过期租约。如下，在Eureka server过期续约的时候，会先去判断是否开启自我保护。

```java
public void evict(long additionalLeaseMs) {
        if (!isLeaseExpirationEnabled()) {//判断是否开启自我保护
            logger.debug("DS: lease expiration is currently disabled.");
            return;
        }
    }
```

```isLeaseExpirationEnabled```方法，只有Eureka server 每分钟接收到的心跳总数大于期望的每分钟续租大小才会返回true。

```java
public boolean isLeaseExpirationEnabled() {
	if (!isSelfPreservationModeEnabled()) {
        return true;
    }
    
    return numberOfRenewsPerMinThreshold > 0 && getNumOfRenewsInLastMin() > numberOfRenewsPerMinThreshold;
    }

//开启自我保护开关
public boolean isSelfPreservationModeEnabled() {
    return serverConfig.shouldEnableSelfPreservation();
}
//获取每分钟心跳总数
public long getNumOfRenewsInLastMin() {
        return renewsLastMin.getCount();
  }
```

**初始化```numberOfRenewsPerMinThreshold、expectedNumberOfRenewsPerMin```**

* Eureka server 在启动时初始化

```java
protected void initEurekaServerContext() throws Exception {
	registry.openForTraffic(applicationInfoManager, registryCount);
}

public void openForTraffic(ApplicationInfoManager applicationInfoManager, int count) {

	this.expectedNumberOfRenewsPerMin = count * 2;
    this.numberOfRenewsPerMinThreshold =
                (int) (this.expectedNumberOfRenewsPerMin * serverConfig.getRenewalPercentThreshold());
    }
```

* 定时重置

```java
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

```updateRenewalThreshold();```方法

```java
private void updateRenewalThreshold() {
	try {
    	Applications apps = eurekaClient.getApplications();
        int count = 0;
        for (Application app : apps.getRegisteredApplications()) {
        	for (InstanceInfo instance : app.getInstances()) {
            	if (this.isRegisterable(instance)) {
                   ++count;
                }
            }
         }
      synchronized (lock) {
      	if ((count * 2) > (serverConfig.getRenewalPercentThreshold() * numberOfRenewsPerMinThreshold)
                        || (!this.isSelfPreservationModeEnabled())) {
                    this.expectedNumberOfRenewsPerMin = count * 2;
                    this.numberOfRenewsPerMinThreshold = (int) ((count * 2) * serverConfig.getRenewalPercentThreshold());
                }
            }
            logger.info("Current renewal threshold is : {}", numberOfRenewsPerMinThreshold);
        } catch (Throwable e) {
            logger.error("Cannot update renewal threshold", e);
        }
    }
```

* 应用实例注册

```java
public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
	synchronized (lock) {
    	if (this.expectedNumberOfRenewsPerMin > 0) {
            this.expectedNumberOfRenewsPerMin = this.expectedNumberOfRenewsPerMin + 2;
            this.numberOfRenewsPerMinThreshold =
                 (int) (this.expectedNumberOfRenewsPerMin * serverConfig.getRenewalPercentThreshold());
          }
    }
}
```

* 应用下线

```java
 public boolean cancel(final String appName, final String id,
                          final boolean isReplication) {
    if (super.cancel(appName, id, isReplication)) {
            replicateToPeers(Action.Cancel, appName, id, null, null, isReplication);
       synchronized (lock) {
                if (this.expectedNumberOfRenewsPerMin > 0) {
                    // Since the client wants to cancel it, reduce the threshold (1 for 30 seconds, 2 for a minute)
                    this.expectedNumberOfRenewsPerMin = this.expectedNumberOfRenewsPerMin - 2;
                    this.numberOfRenewsPerMinThreshold =
                            (int) (this.expectedNumberOfRenewsPerMin * serverConfig.getRenewalPercentThreshold());
                }
            }
            return true;
        }
        return false;
    }
```

5. 实例过期

正常情况下，Eureka client会主动向Eureka server发起下线请求，但实际情况下，应用实例可能异常崩溃，导致下线请求无法被成功提交。

Eureka server在启动时会初始化Eviction Task 定时任务。

```java
//EurekaBootStrap
protected void initEurekaServerContext() throws Exception {
	...
	registry.openForTraffic(applicationInfoManager, registryCount);
｝
    
//PeerAwareInstanceRegistryImpl
    public void openForTraffic(ApplicationInfoManager applicationInfoManager, int count){
    ...
    super.postInit();//调用父类的postInit方法。
}
    
//AbstractInstanceRegistry
protected void postInit() {
	renewsLastMin.start();
    if (evictionTaskRef.get() != null) {
        evictionTaskRef.get().cancel();
    }
    evictionTaskRef.set(new EvictionTask());//初始化定时任务
    //开启定时任务
    evictionTimer.schedule(evictionTaskRef.get(),
    serverConfig.getEvictionIntervalTimerInMs(),
   	serverConfig.getEvictionIntervalTimerInMs());
}    
```

```java
class EvictionTask extends TimerTask {
	public void run() {
    	try {
            //获取补偿时间
        	long compensationTimeMs = getCompensationTimeMs();
            //清理过期租约逻辑
            evict(compensationTimeMs);
            } catch (Throwable e) {
                logger.error("Could not run the evict task", e);
        }
    }
}
```

```evict()```方法，执行过期逻辑

```java
public void evict(long additionalLeaseMs) {
	logger.debug("Running the evict task");
    //判断是否开启自我保护
    if (!isLeaseExpirationEnabled()) {
        logger.debug("DS: lease expiration is currently disabled.");
        return;
     }
	 //循环所有的实例，将过期的的实例收集起来
     List<Lease<InstanceInfo>> expiredLeases = new ArrayList<>();
     for (Entry<String, Map<String, Lease<InstanceInfo>>> groupEntry : registry.entrySet()) {
     	Map<String, Lease<InstanceInfo>> leaseMap = groupEntry.getValue();
     	if (leaseMap != null) {
        	for (Entry<String, Lease<InstanceInfo>> leaseEntry : leaseMap.entrySet()) {
          		Lease<InstanceInfo> lease = leaseEntry.getValue();
                //如果实例过期，将它存放到expiredLeases集合中
          		if (lease.isExpired(additionalLeaseMs) && lease.getHolder() != null) {
                   expiredLeases.add(lease);
           		}
        	}
    	}
 	}

    //本地实例注册总数
   	int registrySize = (int) getLocalRegistrySize();
    //实例自我保护阀值
    int registrySizeThreshold = (int) (registrySize * serverConfig.getRenewalPercentThreshold());
    //允许过期的实例数量
    int evictionLimit = registrySize - registrySizeThreshold;
	//取最小值
    int toEvict = Math.min(expiredLeases.size(), evictionLimit);
    
    if (toEvict > 0) {
        Random random = new Random(System.currentTimeMillis());
        //循环下线实例
        for (int i = 0; i < toEvict; i++) {
                // Pick a random item (Knuth shuffle algorithm)
             int next = i + random.nextInt(expiredLeases.size() - i);
             Collections.swap(expiredLeases, i, next);
             Lease<InstanceInfo> lease = expiredLeases.get(i);

             String appName = lease.getHolder().getAppName();
             String id = lease.getHolder().getId();
             EXPIRED.increment();
             logger.warn("DS: Registry: expired lease for {}/{}", appName, id);
            // 下线实例
             internalCancel(appName, id, false);
            }
        }
    }
```

6. 获取注册信息



