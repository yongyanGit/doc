### Eureka

#### 客户端

1. 配置Eureka客户端

   1.1 引入MAVEN依赖

   ```
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
   ```

   1.2 . 在启动类上添加@EnableDiscoveryClient注解，激活Eureka中的DiscoveryClient实现。

   ```java
   @SpringBootApplication
   @EnableDiscoveryClient
   @RestController
   public class EurekaClientApplication {
       public static void main(String[] args) {
           SpringApplication.run(EurekaClientApplication.class, args);
       }
   }
   ```

   1.3. 配置服务注册中心

   ```
   eureka:
     client:
       service-url:
         defaultZone: http://localhost:1111/eureka/
   ```

   服务启动后，在浏览器中输入```http://localhost:1111/```可以发现服务已经注册上去了

   ![client](../images/springcloud/client.png)

   ​

   Eureka Client向服务端注册服务时，会提供默认的实例ID，格式如下：

   ```
   ${spring.cloud.client.hostname}:${spring.application.name}:${spring.application.instance_id
   ```

   ```
   192.168.72.103:eureka-server:1111
   ```

   我们可以通过如下来自定义实例ID

   ```
   eureka:instance:instance-id:xxxxx
   ```

   ​