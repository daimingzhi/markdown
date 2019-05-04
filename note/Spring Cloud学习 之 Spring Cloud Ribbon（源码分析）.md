ServiceInstance choose(String serviceId);Spring Cloud学习 之 Spring Cloud Ribbon（源码分析）

Spring Boot版本：2.1.4.RELEASE

Spring Cloud版本：Greenwich.SR1

[TOC]

#### 分析：

​		在上篇文章中，我们着重分析了RestTemplate，主要是因为，如果我们采用Ribbon进行服务间的调用的话，要用到这个类，现在我们就先来看看怎么使用RestTemplate配合Ribbon进行服务间的调用。

```java
@SpringBootApplication
@EnableDiscoveryClient
@Slf4j
public class SpringCloudClientApplication {
    /**
     * 配置restTemplate
     */
    @LoadBalanced
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudClientApplication.class, args);
    }
}
```

```java
@RestController
public class Controller {
    @Autowired
    private RestTemplate restTemplate;

    public String test() {
        String body = restTemplate.getForEntity("http://HELLO-SERVICE/hello", String.class).getBody();
        return body;
    }
}
```

​		同时需要注意的是，我们配置的Ribbon这个服务也需要注册到Eureka Server上，因为它需要发现其它服务。配置我这里就不写了，仿照之前的例子就行了。

​		从上面的代码，我们看出，核心就是`@LoadBalanced`这个注解

```java
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Qualifier
public @interface LoadBalanced {
}
```

​		可以看到，这个注解其实没有内容，一般这种注解，都是起到一个标记的作用。所以我们可以猜测，RestTemplate在进行配置时，如果被这个注解标记一定会产生什么影响。我们不妨debug看下被这个注解修饰后，我们的RestTemplate会有什么不同

​		![lb](H:\markdown\images\lb.png)

​		我们可以发现，它多了一个拦截器。回顾我们昨天分析RestTemplate的源码，其实有一点我当时没讲清楚，现在再来看下。​	

```java
 protected ClientHttpRequest createRequest(URI url, HttpMethod method) throws IOException {
     // 到这一步，这个this指向的时restTemplate对象
     // 会去调用RestTemplate的父类InterceptingHttpAccessor的getRequestFactory()方法
        ClientHttpRequest request = this.getRequestFactory().createRequest(url, method);
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("HTTP " + method.name() + " " + url);
        }

        return request;
    }
```

​		我们来看下InterceptingHttpAccessor的getRequestFactory()方法

```java
 public ClientHttpRequestFactory getRequestFactory() {
        List<ClientHttpRequestInterceptor> interceptors = this.getInterceptors();
        if (!CollectionUtils.isEmpty(interceptors)) {
            ClientHttpRequestFactory factory = this.interceptingRequestFactory;
            if (factory == null) {
                factory = new InterceptingClientHttpRequestFactory(super.getRequestFactory(), interceptors);
                this.interceptingRequestFactory = (ClientHttpRequestFactory)factory;
            }

            return (ClientHttpRequestFactory)factory;
        } else {
            return super.getRequestFactory();
        }
    }
```

​		可以看到，如果有拦截器的情况下，RestTmeplate中的ClientHttpRequestFactory会换成，

```java
new InterceptingClientHttpRequestFactory(super.getRequestFactory(), interceptors);
```

​		实际上，我们给RestTemplate标注了@LoadBalanced注解后，就代表了我们要使用负载均衡的客户端（LoadBanlancerClient）来配置它

​		我们可以搜索下LoadBalancerClient。

```java
public interface LoadBalancerClient extends ServiceInstanceChooser {
    <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;

    <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException;

    URI reconstructURI(ServiceInstance instance, URI original);
}
```

​	继承的父接口：

```java
public interface ServiceInstanceChooser {
    ServiceInstance choose(String serviceId);
}
```

​	可以发现，这是一个接口，不难猜测，肯定是定义了一套负载均衡的规范，我们分析下这几个方法

1. 父接口中的 ServiceInstance choose(String serviceId);

   根据传入的服务名ServiceId，从负载均衡器中挑选一个对应服务的实例

2. <T> T execute(......)，这个方法有两种重载的形式，主要就是使用从负载均衡器中挑选出来的实例执行请求内容

3. URI reconstructURI(ServiceInstance instance, URI original)。为系统构建一个合适的host:port形式的URL。在分布式系统中，我们使用逻辑上的服务名称作为host来构建URL（替代服务实例的host:port形式）进行请求，比如http://myservice/path/to/service。在该操作的定义中，前者ServiceInstance对象（方法参数）是带有host和port的具体服务实例，而后者URI对象则是使用逻辑服务名定义为host的URI，而返回的URI则是通过ServiceInstance的服务实例详情拼接除的具体host:port形式的请求地址



​		通过之前debug我们知道，@LoadBalanced注解修饰后，RestTemplate主要的不同，在于多了一个拦截器，即：`org.springframework.cloud.client.loadbalancer.LoadBalancerInterceptor`，这个类有两个成员变量`org.springframework.cloud.client.loadbalancer.LoadBalancerClient`和`org.springframework.cloud.client.loadbalancer.LoadBalancerRequestFactory`

​		我们可以想一下，LoadBalancerInterceptor这个类是从哪里来的呢？什么时候创建的呢？如果我们对[Spring Boot的自动配置](https://blog.csdn.net/qq_41907991/article/details/88704448)有所了解的话，我们的第一反应应该就是去看下spring.factories文件，我们到`LoadBalancerInterceptor`所在依赖包下，找到这个文件，可以看到：

![factories](H:\markdown\images\factories.png)

 	我们主要关注`org.springframework.cloud.client.loadbalancer.LoadBalancerAutoConfiguration`

搜索这个类：

​	

```java
@Configuration
@ConditionalOnClass({RestTemplate.class})
@ConditionalOnBean({LoadBalancerClient.class})
@EnableConfigurationProperties({LoadBalancerRetryProperties.class})
public class LoadBalancerAutoConfiguration {
    @LoadBalanced
    @Autowired(
        required = false
    )
    private List<RestTemplate> restTemplates = Collections.emptyList();
    @Autowired(
        required = false
    )
    private List<LoadBalancerRequestTransformer> transformers = Collections.emptyList();

    public LoadBalancerAutoConfiguration() {
    }

    @Bean
    public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
        return () -> {
            restTemplateCustomizers.ifAvailable((customizers) -> {
                Iterator var2 = this.restTemplates.iterator();

                while(var2.hasNext()) {
                    RestTemplate restTemplate = (RestTemplate)var2.next();
                    Iterator var4 = customizers.iterator();

                    while(var4.hasNext()) {
                        RestTemplateCustomizer customizer = (RestTemplateCustomizer)var4.next();
                        customizer.customize(restTemplate);
                    }
                }

            });
        };
    }

    @Bean
    @ConditionalOnMissingBean
    public LoadBalancerRequestFactory loadBalancerRequestFactory(LoadBalancerClient loadBalancerClient) {
        return new LoadBalancerRequestFactory(loadBalancerClient, this.transformers);
    }
	// 省略部分代码......

    @Configuration
    @ConditionalOnMissingClass({"org.springframework.retry.support.RetryTemplate"})
    static class LoadBalancerInterceptorConfig {
        LoadBalancerInterceptorConfig() {
        }

        @Bean
        // LoadBalancerInterceptor
        public LoadBalancerInterceptor ribbonInterceptor(LoadBalancerClient loadBalancerClient, LoadBalancerRequestFactory requestFactory) {
            // 在这里创建了
            return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
        }

        @Bean
        @ConditionalOnMissingBean
        public RestTemplateCustomizer restTemplateCustomizer(final LoadBalancerInterceptor loadBalancerInterceptor) {
            return (restTemplate) -> {
                List<ClientHttpRequestInterceptor> list = new ArrayList(restTemplate.getInterceptors());
                list.add(loadBalancerInterceptor);
                restTemplate.setInterceptors(list);
            };
        }
    }
}

```

​	从这个类头上的注解可以知道，Ribbon实现负载均衡自动化配置需要满足下面的条件。

- @ConditionalOnClass({RestTemplate.class})：RestTemplate类必须存在于当前工程的环境中

- @ConditionalOnBean({LoadBalancerClient.class})：在Spring的容器中必须要有LoadBalancerClient的实现类

  在该自动化配置类中，主要做了下面三件事：

- 创建一个LoabBalancerInterceptor的Bean,用于实现对客户端发起请求时进行拦截，以实现客户端负载均衡。

- 创建一个RestTemplateCustomizer的Bean，用于给RestTemplate增加LoadBanlancerInterceptor拦截器

- 唯一了一个被@Loadbalanced注解修饰的RestTemplate对象列表，并在这里进行初始化，通过调用RestTemplateCustomizer的实例来给需要客户端负载均衡的RestTemplate增加LoabBalancerInterceptor拦截器

接下来我们分析下，LoabBalancerInterceptor这个类，先上源码：

```java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {
    private LoadBalancerClient loadBalancer;
    private LoadBalancerRequestFactory requestFactory;

    public LoadBalancerInterceptor(LoadBalancerClient loadBalancer, LoadBalancerRequestFactory requestFactory) {
        this.loadBalancer = loadBalancer;
        this.requestFactory = requestFactory;
    }

    public LoadBalancerInterceptor(LoadBalancerClient loadBalancer) {
        this(loadBalancer, new LoadBalancerRequestFactory(loadBalancer));
    }

    public ClientHttpResponse intercept(final HttpRequest request, final byte[] body, final ClientHttpRequestExecution execution) throws IOException {
        URI originalUri = request.getURI();
        // 拿到服务名
        String serviceName = originalUri.getHost();
        Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
        return (ClientHttpResponse)this.loadBalancer.execute(serviceName, this.requestFactory.createRequest(request, body, execution));
    }
}
```

​	通过源码以及以前的自动化配置类，我们可以看到在拦截器中注入了LoadBalancerClient的实现。当一个被@Loadbalanced注解修饰的RestTemplate对象向外发起HTTP请求的时候，会被LoadBalancerInterceptor类的intercept函数所拦截。由于我们在使用RestTemplate时，采用了服务名做host，所以直接从HttpRequest的URI对象中通过getHost()就可以拿到服务名，然后调用execute函数去根据服务名来选中实例，并发起实际请求。

​	分析到这里，LoadBalancerClient还知识一个抽象的负载均衡器接口，所以我们还需要找到它的具体实现类来进一步分析。`org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient`，我们找到这个类。​	

```java

	/**
	 * New: Execute a request by selecting server using a 'key'. The hint will have to be
	 * the last parameter to not mess with the `execute(serviceId, ServiceInstance,
	 * request)` method. This somewhat breaks the fluent coding style when using a lambda
	 * to define the LoadBalancerRequest.
	 * @param <T> returned request execution result type
	 * @param serviceId id of the service to execute the request to
	 * @param request to be executed
	 * @param hint used to choose appropriate {@link Server} instance
	 * @return request execution result
	 * @throws IOException executing the request may result in an {@link IOException}
	 */
	public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint)
			throws IOException {
        // 1.获取这个服务的负载均衡器
		ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
        // 2.获取这个服务的一个实例
		Server server = getServer(loadBalancer, hint);
		if (server == null) {
			throw new IllegalStateException("No instances available for " + serviceId);
		}
		RibbonServer ribbonServer = new RibbonServer(serviceId, server,
				isSecure(server, serviceId),
				serverIntrospector(serviceId).getMetadata(server));

		return execute(serviceId, ribbonServer, request);
	}

	@Override
	public <T> T execute(String serviceId, ServiceInstance serviceInstance,
			LoadBalancerRequest<T> request) throws IOException {
		Server server = null;
		if (serviceInstance instanceof RibbonServer) {
			server = ((RibbonServer) serviceInstance).getServer();
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

	private ServerIntrospector serverIntrospector(String serviceId) {
		ServerIntrospector serverIntrospector = this.clientFactory.getInstance(serviceId,
				ServerIntrospector.class);
		if (serverIntrospector == null) {
			serverIntrospector = new DefaultServerIntrospector();
		}
		return serverIntrospector;
	}
```

1. 获取这个服务的负载均衡器

2. 获取这个服务的一个实例

   我们看下getServer这个方法的实现

   ```java
   protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
           return loadBalancer == null ? null : loadBalancer.chooseServer(hint != null ? hint : "default");
       }
   ```

   可以看到，它调用的时ILoadBalancer的方法，我们看下这个接口：

   ```java
   public interface ILoadBalancer {
       void addServers(List<Server> var1);
   
       Server chooseServer(Object var1);
   
       void markServerDown(Server var1);
   
       /** @deprecated */
       // 已过期，忽略
       @Deprecated
       List<Server> getServerList(boolean var1);
   
       List<Server> getReachableServers();
   
       List<Server> getAllServers();
   }
   ```

   可以看到，该接口中定义了一个客户端负载均衡器需要的一系列抽象操作

   - addServers：向负载均衡器中维护的实例列表中增加服务实例
   - chooseServer：通过某种策略，从负载均衡器中挑选一个具体的服务实例
   - markServerDown：用来通知和标识负载均衡器中某个具体实例已经停止服务，不然负载均衡器在下一次获取服务实例清单前都是会认为服务实例均是正常的
   - getReachableServers：获取当前正常服务的实例列表
   - getAllServers：获取所有已知的服务实例列表，包括正常服务和停止服务的实例

   在该接口定义中涉及的Server对象定义是一个传统的服务端节点，在该类中存储了服务端节点的一些元数据信息，包括host,port以及一些部署信息。

   我们查看这个接口的关系图，如下：

   ![Ribbon_tree](H:\markdown\images\Ribbon_tree.png)

   那么，在整合Ribbon的时候，Spring Cloud默认采用哪个具体的实现呢？我们查看spring-cloud-netflix-ribbon包下的RibbonClientConfiguration配置类，可以发现

   ```java
   @Bean
   @ConditionalOnMissingBean
   public ILoadBalancer ribbonLoadBalancer(IClientConfig config, ServerList<Server> serverList, ServerListFilter<Server> serverListFilter, IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
       return (ILoadBalancer)(this.propertiesFactory.isSet(ILoadBalancer.class, this.name) ? (ILoadBalancer)this.propertiesFactory.get(ILoadBalancer.class, config, this.name) : new ZoneAwareLoadBalancer(config, rule, ping, serverList, serverListFilter, serverListUpdater));
   }
   ```

   默认会采用ZoneAwareLoadBalancer来实现负载均衡器

下面，我们再回到LoadBalancerClient，在通过ZoneAwareLoadBalancer的chooseServer函数获取了负载均衡策略分配到的服务实例对象Server之后，将其内容包装成RibbonServer对象

```java
            RibbonLoadBalancerClient.RibbonServer ribbonServer = new RibbonLoadBalancerClient.RibbonServer(serviceId, server, this.isSecure(server, serviceId), this.serverIntrospector(serviceId).getMetadata(server));

```

该对象除了存储服务实例的信息之外，还增加了服务名serviceId，是否需要https等其他信息，然后再使用该对象回调LoabBalancerInterceptor请求拦截器中LoabBalancerRequest的apply函数，向一个具体的服务实例发起请求，从而实现一开始以服务名为host的URII请求到host:port形式的实际访问地址的转换

在apply函数中传入的ServiceInstance接口对象是对服务实例的抽象定义。在该接口中暴露了服务治理系统中每个服务实例需要提供的一些基本信息，比如serviceId,host,port等，具体定义如下：

```
public interface ServiceInstance {
    default String getInstanceId() {
        return null;
    }

    String getServiceId();

    String getHost();

    int getPort();

    boolean isSecure();

    URI getUri();

    Map<String, String> getMetadata();

    default String getScheme() {
        return null;
    }
}
```

而上面提到的具体包装Server服务实例的RibbonServer对象就是ServiceInstance接口的实现。

那么apply函数在接受到具体的ServerInstance实例后，是如何通过LoadBalancerClient接口中的reconstructURI操作来足迹具体请求地址的呢？

```java
 public ListenableFuture<ClientHttpResponse> apply(final ServiceInstance instance) throws Exception {
                HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance, AsyncLoadBalancerInterceptor.this.loadBalancer);
                return execution.executeAsync(serviceRequest, body);
            }
```

从apply的实现中，可以看到它具体执行的时候，还传入了ServiceRequestWrapper对象，该对象继承了HttpRequestWrapper，并复写了getURI方法

```java
public class ServiceRequestWrapper extends HttpRequestWrapper {
    private final ServiceInstance instance;
    private final LoadBalancerClient loadBalancer;

    public ServiceRequestWrapper(HttpRequest request, ServiceInstance instance, LoadBalancerClient loadBalancer) {
        super(request);
        this.instance = instance;
        this.loadBalancer = loadBalancer;
    }

    public URI getURI() {
        URI uri = this.loadBalancer.reconstructURI(this.instance, this.getRequest().getURI());
        return uri;
    }
}
```

我们继续跟踪代码可以发现：

```java
public ListenableFuture<ClientHttpResponse> executeAsync(HttpRequest request, byte[] body) throws IOException {
            if (this.iterator.hasNext()) {
                AsyncClientHttpRequestInterceptor interceptor = (AsyncClientHttpRequestInterceptor)this.iterator.next();
                return interceptor.intercept(request, body, this);
            } else {
                URI uri = request.getURI();
                HttpMethod method = request.getMethod();
                HttpHeaders headers = request.getHeaders();
                Assert.state(method != null, "No standard HTTP method");
                AsyncClientHttpRequest delegate = InterceptingAsyncClientHttpRequest.this.requestFactory.createAsyncRequest(uri, method);
                delegate.getHeaders().putAll(headers);
                if (body.length > 0) {
                    StreamUtils.copy(body, delegate.getBody());
                }

                return delegate.executeAsync();
            }
        }
```

可以看到，在创建请求的时候requestFactory.createAsyncRequest(uri, method)，这个的request.getURI()会调用之前介绍的ServiceRequestWrapper对象中重写的getURI函数。此时，它就会使用RibbonLoadBalancerClient中实现的reconstructURI来组织具体请求的服务实例地址

```java
public URI reconstructURI(ServiceInstance instance, URI original) {
        Assert.notNull(instance, "instance can not be null");
        String serviceId = instance.getServiceId();
        RibbonLoadBalancerContext context = this.clientFactory.getLoadBalancerContext(serviceId);
        URI uri;
        Server server;
        if (instance instanceof RibbonLoadBalancerClient.RibbonServer) {
            RibbonLoadBalancerClient.RibbonServer ribbonServer = (RibbonLoadBalancerClient.RibbonServer)instance;
            server = ribbonServer.getServer();
            uri = RibbonUtils.updateToSecureConnectionIfNeeded(original, ribbonServer);
        } else {
            server = new Server(instance.getScheme(), instance.getHost(), instance.getPort());
            IClientConfig clientConfig = this.clientFactory.getClientConfig(serviceId);
            ServerIntrospector serverIntrospector = this.serverIntrospector(serviceId);
            uri = RibbonUtils.updateToSecureConnectionIfNeeded(original, clientConfig, serverIntrospector, server);
        }
		
        return context.reconstructURIWithServer(server, uri);
    }
```

上面就是在对请求的URI在做一些处理，为什么要处理，之前也说过了，因为我们使用服务名做为host在调用，而实际发送请求的时候，我们需要将其转换为真正的请求路径。这里对这段代码就不多说了。

#### 总结：

​	通过之前的分析，我们大致对Spring Cloud Ribbon中实现负载均衡的基本脉络有了一定的了解，知道它是如何通过LoadBalancerInterceptor拦截器对RestTemplate的请求进行了拦截，并利用Spring Cloud的负载均衡器LoadBalancerClient讲以逻辑服务名为host的URI转换成具体服务实例地址的过程。同时分析LoadBalancerClient的Ribbon实现RibbonLoadBalancerClient，可以知道在使用Ribbon实现负载均衡器的时候，实际使用的还是Ribbon中定义的ILoadBalancer接口的实现，自动化配置会采用ZoneAwareLoadBalancer的实例来实现客户端负载均衡。

