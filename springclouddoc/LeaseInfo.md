### LeaseInfo

Eureka 使用LeaseInfo(com.netflix.appinfo.LeaseInfo)来标识应用实例的租约信息。它的主要参数如下：

```java
renewalIntervalInSecs//客户端续约间隔
durationInSecs//客户端设定的租约的有效时间
registrationTimestamp //服务端设置的该租约第一次注册时间
lastRenewalTimestamp//服务端设置的该租约的最后一次租约时间
evictionTimestamp//服务端将该租约被剔除的时间
serviceUpTimestamp//服务端将该服务实例设置为UP的时间    
```

这些参数主要是用于标识应用实例的心跳情况，比如：约定的心跳周期，租约有效期，最近一次续约的时间等。