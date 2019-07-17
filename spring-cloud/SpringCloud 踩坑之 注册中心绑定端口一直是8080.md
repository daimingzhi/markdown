SpringCloud 踩坑之 注册中心绑定端口一直是8080

今天在启动注册中心服务时，突然端口一直是8080，找了好久一直没找到原因，先看看我有问题的配置

```yaml
spring:
  application:
    name: eureka-server
  profiles: dev
server:
#  port: ${port:8761}
  port: 8761
eureka:
  instance:
    # 实例id
    instance-id: ${spring.application.name}-${port:8761}-${random.value}
    # 服务名称，不配置的话取spring.application.name
    appname: eureka-server
    # 主机名，不配置的话根据操作系统的主机名来获取,
    # 单机配置集群一定要配置,多台机器不用配置，默认会从系统中取
    hostname: ${HOSTNAME}
    # 发送心跳的间隔时间,默认30
    lease-renewal-interval-in-seconds: 5
    # 等待心跳包的时间上限，默认90
    lease-expiration-duration-in-seconds: 10
    # 在配置集群eureka时不能设置为true,默认为false,若为ture，eureka分片会显示不可用--->unavailable-replicas
    prefer-ip-address: false
  client:
    # 从是否从注册中心中获取注册信息，集群模式下，配置为true
    fetch-registry: true
    # 是否将自身注册到注册中心，集群模式下配置为true
    register-with-eureka: true
    service-url:
      # 与服务端通信的地址
      defaultZone: ${EUREKA_SERVER_URL}/eureka/
    # 将健康检查委托给actuator维护,默认为true
    healthcheck:
      enabled: true
  server:
#     是否开启保护模式，开发环境建议为false,默认为true
    enable-self-preservation: false
    #当eureka服务器启动时获取其他服务器的注册信息失败时，会再次尝试获取，期间需要等待的时间，默认为30 * 1000毫秒
    registry-sync-retry-wait-ms: 500
    eviction-interval-timer-in-ms: 30000
    # 集群里eureka节点的变化信息更新的时间间隔，单位为毫秒，默认为10 * 60 * 1000
    peer-eureka-nodes-update-interval-ms: 30000

```

网上的解决方案都不适用，大概可以总结为以下几种：

1. 检查配置文件是否生效，如：查看target目录下是否有application.yml文件，查看配置文件是否被编译
2. 查看远程配置中心或者启动参数是否带了8080端口，因为这些配置的优先级高于本地的yml文件
3. 查看服务启动的具体报错信息，这个需要具体分析，我启动过程中，并没有出现错误

最后经过多次实验，找到原因如下：

我的配置中有这么一行：

```java
spring:
  profiles: dev
```

问题就出在这里（在查问题的过程中看到一个哥们也是这么配置的，文章链接如下：https://blog.csdn.net/windkiller_XiaoJian/article/details/81461999）

这种配置是有问题的，我的本地是想要激活dev的配置，正确的配置如下：

```yaml
spring:
  profiles:
    active: dev
```

修正后启动服务，端口变为正常8761，具体原因没找到，有懂的大神留言说一声哈~

希望能跟大家多多交流，望大佬不吝赐教