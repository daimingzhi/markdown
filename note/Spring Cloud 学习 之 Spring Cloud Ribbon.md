Spring Cloud 学习 之 Spring Cloud Ribbon（基础知识铺垫）

[TOC]

#### 1.负载均衡：

​	负载均衡在系统架构中是一个非常重要，并且不得不去实施的内容。因为负载均衡是对系统高可用，网络压力的缓解和处理能力扩容的重要手段之一。我们通常说的负载均衡都指的是服务端负载均衡，其中分为硬件负载均衡和软件负载均衡。硬件负载均衡主要通过在服务器节点之间安装专门用于负载均衡的设备，而软件负载均衡则是通过在服务器上安装一些具有均衡负载功能或模块的软件来完成请求分发的工作，比如Nginx等。不论采用硬件负载均衡还是软件负载均衡，只要是服务端负载均衡都能以类似下图的架构方式构建起来：

​	

![负载均衡](H:\markdown\images\负载均衡.png)

​	硬件负载均衡的设备或是软件负载均衡的软件模块都会维护一个下挂可用的服务端清单，通过心跳检测来剔除故障的服务端节点以保证清单中都是可以正常访问的服务端节点。当客户端发送请求到负载均衡设备的时候，该设备按某种算法（比如线性轮询，按权重负载，按流量负载）从维护的可用服务清单中取出一台服务端的地址，然后进行转发。

​	而客户端负载均衡跟服务端负载均衡最大的不同点在于上面所提到的服务清单所存储的位置。在客户端负载均衡中，所有客户端节点都维护着自己要访问的服务端清单，而这些服务端清单来自于注册中心，比如之前我们所学习的Eureka Server。同服务端负载均衡的架构类似，在客户端负载均衡中也需要心跳去维护服务端清单的健康性，只是这个步骤需要与服务注册中心配合完成。在Spring Cloud 实现的服务治理框架中，默认会创建针对各个服务治理框架的Ribbon自动化整合配置，比如Eureka中的`org.springframework.cloud.netflix.ribbon.eureka.RibbonEurekaAutoConfiguration`。在实际使用的时候，我们可以通过查看这两个类的实现，以找到他们的配置详情来帮助我们更好的使用它。

​	通过Spring Cloud Ribbon的封装，我们在微服务架构中使用客户端负载均衡调用非常简单，只需要如下两步：

1. 服务提供者只需要启动多个服务实例并注册到一个注册中心或是多个相关联的服务注册中心
2. 服务消费者直接通过调用被@LoadBalanced注解修饰过的RestTemplate来实现面向服务的接口调用

这样，我们就可以将服务提供者的高可用以及服务消费者的负载均衡调用一起实现了。

#### 2.RestTemplate详解：

##### xxxForEntity/xxxForObject：主要介绍get跟post

![get](H:\markdown\images\get.png)

​	get方法主要有6中调用方式，其中又可以分为getForEntity和getForObject，每个方法有3中重载方式。

​	getForEntity函数。该方法返回的是ResponseEntity，该对象是Spring对HTTP请求响应的封装，其中主要存储了HTTP的几个重要元素，比如HTTP请求状态码的枚举对象HttpStatus（也就是我们常说的400，404这些错误码），在它的父类HttpEntity中还存储着HTTP请求的头信息对象HttpHeaders以及泛型类型的请求体对象。

​	getForObject函数，我们可以理解为它是对getForEntity的进一步封装，它通过`HttpMessageConverterExtractor`对HTTP的请求响应体body内容进行对象转换，实现请求直接返回包装好的对象内容。

​	在了解了两种调用方式的不同的后，我们再来看看他们的参数：

- String url：get请求访问地址，例如：“http://USER_SERVICE/user?name=zhangsan”

- Class<T> responseType： ResponseEntity对象的泛型类型，也就是我们返回数据的实例类型

- Map<String,?> urlVariables：get请求携带的参数，如果以这种方式携带参数，则调用方式如下：

  ```java
   RestTemplate restTemplate = new RestTemplate();
          Map<String,String> param = new HashMap<>();
          param.put("name","zhangsan");
          restTemplate.getForEntity("http://USER_SERVICE/user?name={name}", User.class,param);
  ```

- Object.. urlVariables：get请求携带的参数，如果以这种方式携带参数，则调用方式如下：

  ```java
          RestTemplate restTemplate = new RestTemplate();
          String [] param = {"zhangsan","11"};
          restTemplate.getForEntity("http://USER_SERVICE/user?name={1}&age={2}", User.class,param);
  ```

- URI url: get请求访问地址,这里使用URI对象来替代之前的url和urlVariables参数来指定访问地址和参数绑定。URI是JDK中java.net包下的一个类，它表示一个统一资源标识符引用。调用方式如下:

  ```java
   RestTemplate restTemplate = new RestTemplate();
          UriComponents uriComponents = UriComponentsBuilder
                  .fromUriString("http://USER_SERVICE/user?name={name}")
                  .build()
                  .expand("zhangsan").encode();
          URI uri = uriComponents.toUri();
          restTemplate.getForEntity(uri, User.class);
  ```

  或者：

  ```java
   RestTemplate restTemplate = new RestTemplate();
          UriComponentsBuilder uriComponents = UriComponentsBuilder
                  .fromUriString("http://USER_SERVICE/user");
          uriComponents.queryParam("name","zhangsan");
          restTemplate.getForEntity(uriComponents.build().toUri(), User.class);
  ```

采用上面的这种调用方式，并使用get请求的话，没办法携带头信息（反正我没找到）。但是post方法是可以的，我们看下post方法的调用方式：

![postForEntity](H:\markdown\images\postForEntity.png)

![postForObject](H:\markdown\images\postForObject.png)

可以看到跟get请求相比，post方法多携带了一个参数`Object request`，在这个参数中，我们可以携带我们的头信息，调用方式如下：

```java
RestTemplate restTemplate = new RestTemplate();
        HttpHeaders httpHeaders = new HttpHeaders();
        httpHeaders.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
        HttpEntity entity = new HttpEntity(httpHeaders);
        UriComponentsBuilder uriComponents = UriComponentsBuilder
                .fromUriString("http://USER_SERVICE/user");
        uriComponents.queryParam("name", "zhangsan");
        restTemplate.postForEntity(uriComponents.build().toUri(), entity, User.class);
```

用上面这种方式调用的话，参数是拼接在url后面的，只不过会在头信息中携带我们定义的信息

```
http://USER_SERVICE/user?name=zhangsan
```

如果我们要以请求体的方式携带我们的参数，需要这样调用

```java
        RestTemplate restTemplate = new RestTemplate();
        HttpHeaders httpHeaders = new HttpHeaders();
        httpHeaders.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
		// 换成对象也可以
        Map<String,Object> param = new HashMap<>();
        param.put("name","zhangsan");
        HttpEntity entity = new HttpEntity(param,httpHeaders);
        UriComponentsBuilder uriComponents = UriComponentsBuilder
                .fromUriString("http://USER_SERVICE/user");
        restTemplate.postForEntity(uriComponents.build().toUri(), entity, User.class);
```

问题来了，如果我要以get请求的方式访问，并且还要携带头信息该怎么办呢？这就要说第二种调用方式了

##### exchange:

exchange方法可以在发送个服务器端的请求中设置头信息。

![exchange](H:\markdown\images\exchange.png)

我们观察其参数可以发现

- HttpMethod method:一个枚举类，主要列举了Http请求的各种请求方式。如get:`HttpMethod.GET`
- HttpEntity requestEntity:请求参数封装，跟我们之前的用法一样
- 其余的参数之前已经分析过了就不讲了

##### execute源码分析：

​	如果我们去跟踪exchange方法就会发现，它其实就是调用了execute方法，不只是exchange,xxxForEntity/xxxForObjext其实底层都调用了execute方法。这里，为了对RestTemplate有更深的理解，我们现在来分析这个方法。

​	我们追踪这个方法，会发现，它调用了doExecute这个方法

```java
protected <T> T doExecute(URI url, @Nullable HttpMethod method, @Nullable RequestCallback requestCallback,
			@Nullable ResponseExtractor<T> responseExtractor) throws RestClientException {

		Assert.notNull(url, "URI is required");
		Assert.notNull(method, "HttpMethod is required");
		ClientHttpResponse response = null;
		try {
            // 1.构建一个request对象
			ClientHttpRequest request = createRequest(url, method);
			if (requestCallback != null) {
                // 2. 发送请求前对request对象进行一些处理
				requestCallback.doWithRequest(request);
			}
            // 3.发送请求，得到响应
			response = request.execute();
            // 4.处理一些错误信息
			handleResponse(url, method, response);
            // 5.解析请求数据
			return (responseExtractor != null ? responseExtractor.extractData(response) : null);
		}
		catch (IOException ex) {
			String resource = url.toString();
			String query = url.getRawQuery();
			resource = (query != null ? resource.substring(0, resource.indexOf('?')) : resource);
			throw new ResourceAccessException("I/O error on " + method.name() +
					" request for \"" + resource + "\": " + ex.getMessage(), ex);
		}
		finally {
			if (response != null) {
				response.close();
			}
		}
	}
```

1. `ClientHttpRequest request = createRequest(url, method);`

这句话的主要目的是构建一个ClientHttpRequest 对象，我们跟踪下代码：

```java
  protected ClientHttpRequest createRequest(URI url, HttpMethod method) throws IOException {
        ClientHttpRequest request = this.getRequestFactory().createRequest(url, method);
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("HTTP " + method.name() + " " + url);
        }

        return request;
    }
```

它会获取一个`ClientHttpRequestFactory`,然后构建一个`ClientHttpRequest`对象，我们可以看到默认会采用`SimpleClientHttpRequestFactory`，代码如下：

```java
 private ClientHttpRequestFactory requestFactory = new SimpleClientHttpRequestFactory();
```

如果我们要配置不同的工厂要怎么配置呢？我们可以通过`org.springframework.boot.web.client.RestTemplateBuilder`对象，例如，我们要以okHttpClient作为RestTemplate的底层实现

```java
 RestTemplateBuilder builder = new RestTemplateBuilder();
        ClientHttpRequestFactory okHttp3ClientHttpRequestFactory = new OkHttp3ClientHttpRequestFactory();
        RestTemplate build = builder
                .requestFactory(() -> okHttp3ClientHttpRequestFactory).build();
```

2. `requestCallback.doWithRequest(request);`这里主要是对头信息做了一下处理，具体源码大家可以自己看下
3. `response = request.execute();`发送一个http请求
4. `handleResponse(url, method, response);`处理错误信息
5. ` responseExtractor.extractData(response) `，按我们定义的ResponseEntity中的泛型跟HttpMessageConverter解析数据类型

​	由于Ribbon中，采用了RestTemplate所以花时间系统的了解了一下RestTemplate,接下来，我们就要系统的学习一下Ribbon。



