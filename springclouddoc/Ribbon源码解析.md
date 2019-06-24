#### Ribbon源码解析

从注解```LoadBalanced```源码注释中可以看到，该注解用来给RestTemplate作标记，当应用发送rest请求时，能够使用LoadBalancerClient进行负载均衡.

```
Annotation to mark a RestTemplate bean to be configured to use a LoadBalancerClient
```

通过搜索，发现SpringCloud中定义了一个接口:

```java
public interface LoadBalancerClient extends ServiceInstanceChooser {

	
	<T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;

	<T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException;

	
	URI reconstructURI(ServiceInstance instance, URI original);
}

public interface ServiceInstanceChooser {

    ServiceInstance choose(String serviceId);
}


```

* ```<T> T execute(String serviceId, LoadBalancerRequest<T> request)```：使用从负载均衡器中挑选的服务实例来执行请求内容。
* ```URI reconstructURI(ServiceInstance instance, URI original);```：构建URI，在一些系统中使用逻辑上的服务名称来代替host，如：```http://myservice/path/to/service```，myservice就是包括了host:port。
* ```ServiceInstance choose(String serviceId);```：根据传入的服务名称从负载均衡中挑选一个对应服务的实例。

跟Eureka 一样，SpringCloud也为Ribbon定义来一个自动化配置类，来为Ribbon初始化一些配置，```LoadBalancerAutoConfiguration```：

```java
@Configuration
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerRetryProperties.class)
public class LoadBalancerAutoConfiguration {
	
	@LoadBalanced
	@Autowired(required = false)
	private List<RestTemplate> restTemplates = Collections.emptyList();
    
//为restTemplate设置拦截器
@Configuration
@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
	static class LoadBalancerInterceptorConfig {
        //创建拦截器
		@Bean
		public LoadBalancerInterceptor ribbonInterceptor(
				LoadBalancerClient loadBalancerClient,
				LoadBalancerRequestFactory requestFactory) {
			return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
		}
	
    //设置拦截器
	@Bean
	@ConditionalOnMissingBean
	public RestTemplateCustomizer restTemplateCustomizer(
		final LoadBalancerInterceptor loadBalancerInterceptor) {
		return restTemplate -> {
               List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                  restTemplate.getInterceptors());
                list.add(loadBalancerInterceptor);
                restTemplate.setInterceptors(list);
            };
		}
	}
}
```

在这个自动化配置类中，主要做了下面两件事：

* 创建一个LoadBalancerInterceptor拦截器，用来对客户端发送请求时进行拦截，以实现客户端的负载均衡。
* 创建RestTemplateCustomizer对象，为RestTemplate增加拦截器。

LoadBalancerInterceptor对RestTemplate请求进行拦截

```java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {

	private LoadBalancerClient loadBalancer;
	private LoadBalancerRequestFactory requestFactory;

	public LoadBalancerInterceptor(LoadBalancerClient loadBalancer, LoadBalancerRequestFactory requestFactory) {
		this.loadBalancer = loadBalancer;
		this.requestFactory = requestFactory;
	}

	public LoadBalancerInterceptor(LoadBalancerClient loadBalancer) {
		// for backwards compatibility
		this(loadBalancer, new LoadBalancerRequestFactory(loadBalancer));
	}
	
    //对请求进行拦截
	@Override
	public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) throws IOException {		
        //获取http请求uri(如：http://user-service/getUser)
		final URI originalUri = request.getURI();
        //访问资源的Host(如：user-service)
		String serviceName = originalUri.getHost();
		Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
        //通过负载均衡器选择服务实例，执行请求
		return this.loadBalancer.execute(serviceName, requestFactory.createRequest(request, body, execution));
	}
}
```

从上面可以看到被@LoadBalanced注解修饰的RestTemplate对象向外发起HTTP请求时，会被LoadBalancerInterceptor类的intercept函数所拦截。我们在请求时，采用了服务名称作为host,所以直接从HttpRequest的URI对象中通过getHost就可以拿到服务名称，然后调用execute函数，根据服务名来选择实例，并发起实际的请求。

在拦截器中执行execute方法时，会创建一个LoadBalancerRequest匿名对象，在这个方法中会创建一个ServiceRequestWrapper对象，在这个对象中重写了getURI方法，在这个重写的方法会根据从负载均衡器中获取的服务实例来构建正确的URI(即用ip地址、端口替换掉服务ID)。

```java
public LoadBalancerRequest<ClientHttpResponse> createRequest(final HttpRequest request,
final byte[] body, final ClientHttpRequestExecution execution) {
    //创建了一个匿名对象，LoadBalancerRequest.apply(ServiceInstance instance)
    //Ribbon通过负载均衡查询到具体的实例对象(ServiceInstance)，然后传递给apply方法
	return instance -> {
        //将request对象封装进ServiceRequestWrapper,它重写了getURI方法
    	HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance, loadBalancer);
        if (transformers != null) {
        	for (LoadBalancerRequestTransformer transformer : transformers) {
                 serviceRequest = transformer.transformRequest(serviceRequest, instance);
                }
            }
        	//调用InterceptingRequestExecution的execute方法
            return execution.execute(serviceRequest, body);
        };
	}
```

调用InterceptingRequestExecution  中的execution()方法

```java
public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
	if (this.iterator.hasNext()) {
        //如果还有其它拦截器，先执行其它拦截器
		ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
		return nextInterceptor.intercept(request, body, this);
	}
	else {
        //post 、get
		HttpMethod method = request.getMethod();
		Assert.state(method != null, "No standard HTTP method");
        //创建一个SimpleBufferClientHttpRequest对象
		ClientHttpRequest delegate = requestFactory.createRequest(request.getURI(), method);
        //循环设置head头
		request.getHeaders().forEach((key, value) -> delegate.getHeaders().addAll(key, value));
		if (body.length > 0) {
			if (delegate instanceof StreamingHttpOutputMessage) {
				StreamingHttpOutputMessage streamingOutputMessage = (StreamingHttpOutputMessage) delegate;
				streamingOutputMessage.setBody(outputStream -> StreamUtils.copy(body, outputStream));
			}
			else {
				StreamUtils.copy(body, delegate.getBody());
			}
		}
        //执行
		return delegate.execute();
	}
}
```

在上面的代码中会创建一个ClientHttpRequest对象，即通过SimpleBufferingClientHttpRequestFactory工厂类创建SimpleBufferingClientHttpRequest对象

```java
public ClientHttpRequest createRequest(URI uri, HttpMethod httpMethod) throws IOException {
		HttpURLConnection connection = openConnection(uri.toURL(), this.proxy);
		prepareConnection(connection, httpMethod.name());

		if (this.bufferRequestBody) {
			return new SimpleBufferingClientHttpRequest(connection, this.outputStreaming);
		}
		else {
			return new SimpleStreamingClientHttpRequest(connection, this.chunkSize, this.outputStreaming);
		}
	}
```

在创建SimpleStreamingClientHttpRequest对象时，会传入一个URI，这个URI由ServiceRequestWrapper的getURI()方法获取

```java
public URI getURI() {
    //调用RibbonLoadBalancerClient对象中的reconstructURI方法
	URI uri = this.loadBalancer.reconstructURI(
		this.instance, getRequest().getURI());
	return uri;
}
```

ServiceRequestWrapper的getURI()会调用RibbonLoadBalancerClient中的reconstructURI方法。

```java
public URI reconstructURI(ServiceInstance instance, URI original) {
	Assert.notNull(instance, "instance can not be null");
    //具体的服务实例ID
	String serviceId = instance.getServiceId();
    //Ribbon上下文
	RibbonLoadBalancerContext context = this.clientFactory
				.getLoadBalancerContext(serviceId);
	URI uri;
	Server server;
	if (instance instanceof RibbonServer) {
        //传进来的是RibbonServer
		RibbonServer ribbonServer = (RibbonServer) instance;
		server = ribbonServer.getServer();
		uri = updateToSecureConnectionIfNeeded(original, ribbonServer);
	} else {
		server = new Server(instance.getScheme(), instance.getHost(), instance.getPort());
		IClientConfig clientConfig = clientFactory.getClientConfig(serviceId);
		ServerIntrospector serverIntrospector = serverIntrospector(serviceId);
		uri = updateToSecureConnectionIfNeeded(original, clientConfig,
					serverIntrospector, server);
	}
	return context.reconstructURIWithServer(server, uri);
}
```

reconstructURIWithServer方法，使用ServiceInstance中的host和port信息构建一个URI，用来访问具体的服务实例。

```java
public URI reconstructURIWithServer(Server server, URI original) {
	//ip地址
    String host = server.getHost();
    //端口号
    int port = server.getPort();
    //http、https
    String scheme = server.getScheme(); 
    if (host.equals(original.getHost()) 
        && port == original.getPort()
        && scheme == original.getScheme()) {
        return original;
    }
    if (scheme == null) {
        scheme = original.getScheme();
    }
    if (scheme == null) {
        scheme = deriveSchemeAndPortFromPartialUri(original).first();
    }

    try {
         StringBuilder sb = new StringBuilder();
         sb.append(scheme).append("://");
         if (!Strings.isNullOrEmpty(original.getRawUserInfo())) {
             sb.append(original.getRawUserInfo()).append("@");
          }
          sb.append(host);
          if (port >= 0) {
              sb.append(":").append(port);
          }
          sb.append(original.getRawPath());
          if (!Strings.isNullOrEmpty(original.getRawQuery())) {
              sb.append("?").append(original.getRawQuery());
          }
          if (!Strings.isNullOrEmpty(original.getRawFragment())) {
              sb.append("#").append(original.getRawFragment());
          }
          URI newURI = new URI(sb.toString());
          return newURI;            
        } catch (URISyntaxException e) {
            throw new RuntimeException(e);
        }
    }
```

我们在回到LoadBalancerIntercepor中的intercept方法，在这个方法中执行execute()方法，

```java
public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
	//获取负载均衡器
	ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
	//从负载均衡器中获取服务实例
	Server server = getServer(loadBalancer);
	if (server == null) {
			throw new IllegalStateException("No instances available for " + serviceId);
		}
		RibbonServer ribbonServer = new RibbonServer(serviceId, server, isSecure(server,
				serviceId), serverIntrospector(serviceId).getMetadata(server));

		return execute(serviceId, ribbonServer, request);
	}
```

**获取负载均衡器:**

```ILoadBalancer loadBalancer = getLoadBalancer(serviceId);``` ,

ILoadBalancer是一个接口，它定义了客户端负载均衡器需要的一系列抽象操作：

```java
public interface ILoadBalancer {

	public void addServers(List<Server> newServers);

	public Server chooseServer(Object key);
	
	public void markServerDown(Server server);

    public List<Server> getReachableServers();

	public List<Server> getAllServers();
}
```

* addServers：向负载均衡器中维护的实例列表增加服务实例。
* chooseServer()：从负载均衡器中挑选出一个具体的服务实例。
* markServerDown()：用来标识均衡器中的某个实例已经停止服。
* getReachableServers()：获取当前正常服务的实例列表
* getAllServers()：获取所有已知的服务实例列表，包括正常服务和停止服务的实例。

在SpringCloud中，ILoadBalancer接口的默认使用ZoneAwareLoadBalancer来实现负载均衡。

当获取到均衡器后，根据均衡器获取服务实例对象Server，然后将获取到的server封装成RibbonServer对象。然后再调用LoadBalancerInterceptor请求拦截器中LoadBalancerRequest的 ```apply(ServiceInstance instance)```方法，向一个实际的具体服务实例发起请求。

``` apply(ServiceInstance instance)```函数中传入一个ServiceInstance接口对象，在这个接口中暴露了服务治理系统中每个服务实例需要提供的一些基本信息，比如：serviceId、host、port等。

```java
public interface ServiceInstance {
	//服务ID
	String getServiceId();
	//服务名称
	String getHost();
    //端口
	int getPort();
	//uri
	URI getUri();
    //服务实例信息(key-value)
	Map<String, String> getMetadata();
}
```

而之前提到的用来包装server实例的RibbonServer就是ServiceInstance接口的实现。

```java
public static class RibbonServer implements ServiceInstance {
	private final String serviceId;
	private final Server server;
	private final boolean secure;
	private Map<String, String> metadata;

	public RibbonServer(String serviceId, Server server) {
		this(serviceId, server, false, Collections.<String, String> emptyMap());
	}

	public RibbonServer(String serviceId, Server server, boolean secure,
				Map<String, String> metadata) {
			this.serviceId = serviceId;
			this.server = server;
			this.secure = secure;
			this.metadata = metadata;
	}
}	
```

```execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request)```方法

```java
public <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException {
	Server server = null;
	if(serviceInstance instanceof RibbonServer) {
		server = ((RibbonServer)serviceInstance).getServer();
	}
	if (server == null) {
		throw new IllegalStateException("No instances available for " + serviceId);
	}
	RibbonLoadBalancerContext context = this.clientFactory
				.getLoadBalancerContext(serviceId);
	RibbonStatsRecorder statsRecorder = new RibbonStatsRecorder(context, server);
	try {
		T returnVal = request.apply(serviceInstance);
		statsRecorder.recordStats(returnVal);
		return returnVal;
	}
		// catch IOException and rethrow so RestTemplate behaves correctly
	catch (IOException ex) {
		statsRecorder.recordStats(ex);
		throw ex;
	}
	catch (Exception ex) {
		statsRecorder.recordStats(ex);
		ReflectionUtils.rethrowRuntimeException(ex);
	}
		return null;
	}
```

在上面的request.apply(serviceInstance)中的request就是前面创建的匿名对象LoadBalancerRequest。















