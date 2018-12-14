### ServiceInstance 

ServiceIstance 是SpringCloud 对service discovery 的实例信息的抽象接口，约定了服务发现的实例应用有哪些通用的信息，其中主要的方法如下：

```java
getServiceId()//服务id
getHost()//实例的host
getPort()//实例端口
isSecure()// 是否开启https
getUri()//实例的uri地址
getMetadata()//实例的元数据
getScheme()//实例的scheme    
```

Spring Cloud Discovery 为适配了Zookeeper 、Consul、Eureka等注册中心。定义了一套通用的抽象方法。Spring Cloud 为Eureka 定义的实现类是EurekaRegistration。