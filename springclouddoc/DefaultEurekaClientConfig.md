#### DefaultEurekaClientConfig

> 完整包名：com.netflix.discovery.DefaultEurekaClientConfig，它是EurekaClientConfig的默认实现。它默认会加载Eureka-client.properties文件中的内容。

```java
public DefaultEurekaClientConfig(String namespace) {
        this.namespace = namespace.endsWith(".")
                ? namespace
                : namespace + ".";
//CONFIG_FILE_NAME: eureka-client
        this.configInstance = Archaius1Utils.initConfig(CommonConstants.CONFIG_FILE_NAME);
        this.transportConfig = new DefaultEurekaTransportConfig(namespace, configInstance);
    }
```

#### DefaultEurekaClientConfigProvider

> 完整包名：com.netflix.discovery.providers.DefaultEurekaClientConfigProvider.用来创建DefaultEurekaClientConfig

```java
public synchronized EurekaClientConfig get() {
        if (config == null) {
            config = (namespace == null)
                    ? new DefaultEurekaClientConfig()
                    : new DefaultEurekaClientConfig(namespace);
                    
            // TODO: Remove this when DiscoveryManager is finally no longer used
            DiscoveryManager.getInstance().setEurekaClientConfig(config);
        }

        return config;
    }
```



