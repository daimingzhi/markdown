Spring Cloud学习 之 Spring Cloud Ribbon 重试机制及超时设置不生效

今天测了一下Ribbon的重试跟超时机制，发现进行的全局超时配置一直不生效，配置如下：

```yaml
ribbon:
  #单位ms,请求连接的超时时间，默认1000
  ConnectTimeout: 500
  #单位ms,读取数据的超时时间，默认1000
  ReadTimeout: 3000
  #对所有操作请求都进行重试
  #设置为true时，会对所有的请求进行重试，若为false只会对get请求进行重试
  #如果是put或post等写操作，
  #如果服务器接口没做幂等性，会产生不好的结果，所以OkToRetryOnAllOperations慎用。
  #默认情况下,GET方式请求无论是连接异常还是读取异常,都会进行重试
  #非GET方式请求,只有连接异常时,才会进行重试
  OkToRetryOnAllOperations: true
  #切换实例的重试次数，默认为1
  MaxAutoRetriesNextServer: 1
  #如果不配置ribbon的重试次数
  #对当前实例的重试次数,默认为0
  MaxAutoRetries: 2
  #为true的时候会关闭懒加载
  #Ribbon进行客户端负载均衡的Client并不是在服务启动的时候就初始化好的，
  #而是在调用的时候才会去创建相应的Client，所以第一次调用的耗时不仅仅包含发送HTTP请求的时间，还包含了创建RibbonClient的时间
  #这样一来如果创建时间速度较慢，同时设置的超时时间又比较短的话，第一次请求很容易超时
  eager-load:
    enabled: true
    #指定需要关闭懒加载的服务名
    clients: eureka-client
```

测了一天后，解决方案如下：

1. 解决全局配置不生效的问题

```java
@Bean
@LoadBalanced
public RestTemplate ribbonRestTemplate() {
    HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory();
    factory.setReadTimeout(2000);
    factory.setConnectTimeout(2000);
    return new RestTemplate();
}
```

因为Ribbon层走的一定是RestTemplate，所以我们可以直接配置这个RestTemplate的配置

2. 解决重试不生效的问题：

pom中需要新增依赖：

```xml
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
```

启动类新增注解：`@EnableRetry`

关于重试为什么不生效我一直没找到原因，希望有懂的大佬能不吝赐教！！！！

我的版本：

Spring Boot版本：2.1.4.RELEASE

Spring Cloud版本：Greenwich.SR1