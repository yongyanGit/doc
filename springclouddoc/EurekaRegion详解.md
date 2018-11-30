### Eureka Region、Zone

我们在微服务中使用Ribbon来实现服务调用时，默认策略会优先访问同客户端处于同一个Zone的访问端实例，只有当同一个Zone没有可用的服务时才会访问其他Zone中的实例。

示例：

1. 配置服务注册中心，server1、server2，两个注册中心互相注册，实现注册中心高可用。

* server1配置

```xml
spring.application.name=eureka-server
server.port=1111
eureka.instance.hostname=server1
#eureka.client.service-url.defaultZone=http://server2:1112/eureka/

#指定region
eureka.client.region=china
#指定region下面的zone
eureka.client.availability-zones.china=zone-1,zone-2
#指定service-url
eureka.client.service-url.zone-1=http://server1:1111/eureka/
eureka.client.service-url.zone-2=http://server2:1112/eureka/
```

* server2配置

```
spring.application.name=eureka-server
server.port=1112
eureka.instance.hostname=server2
#eureka.client.service-url.defaultZone=http://server1:1111/eureka/

#指定region
eureka.client.region=china
#指定region下面的zone
eureka.client.availability-zones.china=zone-1,zone-2
#指定service-url
eureka.client.service-url.zone-1=http://server1:1111/eureka/
eureka.client.service-url.zone-2=http://server2:1112/eureka/
```

2. 服务提供者，service-provider1、service-provider2

* service-provider1

```
server.port=8098
spring.application.name=testProvider
eureka.instance.metadata-map.zone=zone-1
#指定region
eureka.client.region=china
#指定region下面的zone
eureka.client.availability-zones.china=zone-1,zone-2
#指定service-url
eureka.client.service-url.zone-1=http://server1:1111/eureka/
eureka.client.service-url.zone-2=http://server2:1112/eureka/
```

* service-provider2

```
server.port=8099
spring.application.name=testProvider
eureka.instance.metadata-map.zone=zone-2
#指定region
eureka.client.region=china
#指定region下面的zone
eureka.client.availability-zones.china=zone-1,zone-2
#指定service-url
eureka.client.service-url.zone-1=http://server1:1111/eureka/
eureka.client.service-url.zone-2=http://server2:1112/eureka/
```

* 在服务提供中暴露一个接口，供其它客户端调用。

```java
@Value("${server.port}")
private String port;
@RequestMapping("testProvider")
public String testProvider(String name){
   return name+"i am from provider,port="+port;
}
```

通过打印配置文件中的端口号观察客户端调用的位于哪个zone下的服务。

3. 服务调用者，将它的zone设置和service-provider1一样。

```
server.port=8764
spring.application.name=service-ribbon

eureka.instance.metadata-map.zone=zone-1
#指定region
eureka.client.region=china
#指定region下面的zone
eureka.client.availability-zones.china=zone-1,zone-2
#指定service-url
eureka.client.service-url.zone-1=http://server1:1111/eureka/
eureka.client.service-url.zone-2=http://server2:1112/eureka/
```

调用代码：

```java
String s = restTemplate.getForEntity
   ("http://TESTPROVIDER/testProvider?name="+name,String.class).getBody();
```

我们执行调用代码，它总是访问zone-1中的服务，当我们把zone-1中的服务停掉，等一会在执行上面的调用代码，它会去访问zone-2中的服务。

4. 源码解析

首先我们找到```EndpointUtils```这个类，它在```com.netflix.discovery.endpoint```目录下，在这个类下有一个函数：

```java
public static Map<String, List<String>> getServiceUrlsMapFromConfig(EurekaClientConfig clientConfig, String instanceZone, boolean preferSameZone) {
        Map<String, List<String>> orderedUrls = new LinkedHashMap<>();
        String region = getRegion(clientConfig);
        String[] availZones = clientConfig.getAvailabilityZones(clientConfig.getRegion());
        if (availZones == null || availZones.length == 0) {
            availZones = new String[1];
            availZones[0] = DEFAULT_ZONE;
        }
        logger.debug("The availability zone for the given region {} are {}", region, availZones);
        int myZoneOffset = getZoneOffset(instanceZone, preferSameZone, availZones);

        String zone = availZones[myZoneOffset];
        List<String> serviceUrls = clientConfig.getEurekaServerServiceUrls(zone);
        if (serviceUrls != null) {
            orderedUrls.put(zone, serviceUrls);
        }
        int currentOffset = myZoneOffset == (availZones.length - 1) ? 0 : (myZoneOffset + 1);
        while (currentOffset != myZoneOffset) {
            zone = availZones[currentOffset];
            serviceUrls = clientConfig.getEurekaServerServiceUrls(zone);
            if (serviceUrls != null) {
                orderedUrls.put(zone, serviceUrls);
            }
            if (currentOffset == (availZones.length - 1)) {
                currentOffset = 0;
            } else {
                currentOffset++;
            }
        }

        if (orderedUrls.size() < 1) {
            throw new IllegalArgumentException("DiscoveryClient: invalid serviceUrl specified!");
        }
        return orderedUrls;
    }
```

在上面的函数中，依次加载了Region和Zone，

* 通过函数getRegion()方法加载Region，我们可以看到它从配置中读取了一个Region，所以微服务只能属于一个Region，如果不特别配置，默认是default。我们可以通过eureka.client.region来进行配置。

```java
 public static String getRegion(EurekaClientConfig clientConfig) {
    String region = clientConfig.getRegion();
    if (region == null) {
       region = DEFAULT_REGION;
    }
    region = region.trim().toLowerCase();
    return region;
 }
```

* 通过getAvailabilityZones()方法来加载可用的Zone，默认是default zone，这也是我们为什么通常配置service-url时用```eureka.client.service-url.defaultZone```，我们可以使用```eureka.client.availability-zones```来配置可以用的Zone，如果有多个Zone，可以使用逗号分割开来。Region与Zone是一对多的关系。下面的代码位于EurekaClientConfigBean中：

````java
public String[] getAvailabilityZones(String region) {
		String value = this.availabilityZones.get(region);
		if (value == null) {
			value = DEFAULT_ZONE;
		}
		return value.split(",");
	}
````

在获取Zone和Region的信息后，在开始加载Eureka Server的具体地址

```java
int myZoneOffset = getZoneOffset(instanceZone, preferSameZone, availZones);
String zone = availZones[myZoneOffset];
List<String> serviceUrls = clientConfig.getEurekaServerServiceUrls(zone);
```

从availZones中选取一个zone，如果在配置文件中有配置Zone，则返回该Zone的下标，如果没有配置，则说名使用的是默认的Zone，直接返回第一个Zone的下标地址。

```java
private static int getZoneOffset(String myZone, boolean preferSameZone, String[] availZones) {
	for (int i = 0; i < availZones.length; i++) {
     if (myZone != null && (availZones[i].equalsIgnoreCase(myZone.trim()) == preferSameZone)) {
                return i;
            }
        }
        return 0;
    }
```

获取到Zone后，根据Zone获取与它相关联的url。

```java
public List<String> getEurekaServerServiceUrls(String myZone) {
	String serviceUrls = this.serviceUrl.get(myZone);
	if (serviceUrls == null || serviceUrls.isEmpty()) {
		serviceUrls = this.serviceUrl.get(DEFAULT_ZONE);
	}
	if (!StringUtils.isEmpty(serviceUrls)) {
		final String[] serviceUrlsSplit = StringUtils.commaDelimitedListToStringArray(serviceUrls);
		List<String> eurekaServiceUrls = new ArrayList<>(serviceUrlsSplit.length);
		for (String eurekaServiceUrl : serviceUrlsSplit) {
			if (!endsWithSlash(eurekaServiceUrl)) {
				eurekaServiceUrl += "/";
			}
			eurekaServiceUrls.add(eurekaServiceUrl);
		}
		return eurekaServiceUrls;
	}

	return new ArrayList<>();
}
```

















