#### PropertiesInstanceConfig

> 完整包名：com.netflix.appinfo.PropertiesInstanceConfig，基于配置文件的Eureka应用实例配置抽象类。它继承自：AbstractInstanceConfig。

构造方法：

```java
public PropertiesInstanceConfig(String namespace, DataCenterInfo info) {
        super(info);
		//设置命名空间
        this.namespace = namespace.endsWith(".")
                ? namespace
                : namespace + ".";
        appGrpNameFromEnv = ConfigurationManager.getConfigInstance()
                .getString(FALLBACK_APP_GROUP_KEY, Values.UNKNOWN_APPLICATION);
//初始化配置文件对象
        this.configInstance = Archaius1Utils.initConfig(CommonConstants.CONFIG_FILE_NAME);
    }
```

initConfig方法：

```java
public static DynamicPropertyFactory initConfig(String configName) {
		//创建配置文件对象
        DynamicPropertyFactory configInstance = DynamicPropertyFactory.getInstance();
    //设置配置文件名称，默认是eureka.client.properties
        DynamicStringProperty EUREKA_PROPS_FILE = configInstance.getStringProperty("eureka.client.props", configName);
    //获取当前环境名称，设置当前环境类型
        String env = ConfigurationManager.getConfigInstance().getString(EUREKA_ENVIRONMENT, "test");
        ConfigurationManager.getConfigInstance().setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, env);

        String eurekaPropsFile = EUREKA_PROPS_FILE.get();
        try {
 //加载配置文件        ConfigurationManager.loadCascadedPropertiesFromResources(eurekaPropsFile);
        } catch (IOException e) {
            logger.warn(
                    "Cannot find the properties specified : {}. This may be okay if there are other environment "
                            + "specific properties or the configuration is installed with a different mechanism.",
                    eurekaPropsFile);

        }

        return configInstance;
    }
```

#### MyDataCenterInstanceConfig

> 完整包名称：package com.netflix.appinfo.MyDataCenterInstanceConfig

```java
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