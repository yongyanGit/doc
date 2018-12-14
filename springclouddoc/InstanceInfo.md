#### InstanceInfo

Eureka 使用InstanceInfo来代表注册的服务实例，它的主要字段如下：

```java
instanceId//实例id
appName//应用名
appGroupName//应用所属群组
ipAddr//ip地址
port//端口号
securePort//https的端口号
homePageUrl//应用实例的首页url
statusPageUrl//应用实例的状态页
healthCheckUrl//应用实例健康检查的url
secureHealthCheckUrl//应用实例健康检查的https的url
vipAddress//虚拟ip地址
secureVipAddress//https的虚拟ip地址
hostName//主机名称
status //实例状态   UP DOWN STARTING OUT_OF_SERVICE UNKNOWN
overriddenstatus//需要外界强制覆盖的状态值，默认是UNKNOWN
leaseInfo//租约信息
metadata//应用实例的元数据
lastUpdatedTimestamp//状态信息最后更新的时间
lastDirtyTimestamp//实例信息最新的过期时间，在client端用来标识该实例是否与eureka server 一致，在Server端则用于多个Eureka Server之间的信息同步    
actionType//标识Eureka Server 对实例执行的操作，包括ADDED MODIFIED  DELETED
    
```

