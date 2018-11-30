#### EurekaClient

> 完整包名：com.netflix.discovery.EurekaClient

```java
//获取eureka-client配置
public EurekaClientConfig getEurekaClientConfig();
    
//获取applicationInfo    
public ApplicationInfoManager getApplicationInfoManager();
```

#### DiscoveryClient

> 完整包名：com.netflix.discovery.DiscoveryClient，主要用来与eureka-server交互，如注册、续约、取消服务,从注册中心查询服务。

构造方法：

```java
DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                    Provider<BackupRegistry> backupRegistryProvider){}
```

```ApplicationInfoManager```和```EurekaClientConfig```前面有提到。```BackupRegistry```备份注册中心接口，当eureka-client启动时，无法从Eureka-server读取注册信息，从备份注册中心读取注册信息。

```java
public class NotImplementedRegistryImpl implements BackupRegistry {

    @Override
    public Applications fetchRegistry() {
        return null;
    }

    @Override
    public Applications fetchRegistry(String[] includeRemoteRegions) {
        return null;
    }
}
```

从上面可以看出，eureka母亲没有实现从备份中获取注册信息

