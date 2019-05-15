Spring Cloud 学习 之 Spring Cloud Eureka（源码分析）

Spring Boot版本：2.1.4.RELEASE

Spring Cloud版本：Greenwich.SR1

[TOC]

#### 客户端（以之前搭建的[客户端](https://blog.csdn.net/qq_41907991/article/details/89611301)demo为例）：

##### Region,Zone,ServiceUrls:

```java
@SpringBootApplication
// 核心注解
@EnableDiscoveryClient
public class SpringCloudClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringCloudClientApplication.class, args);
    }
}
```

我们主要跟踪注解`@EnableDiscoveryClient`

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
// 核心注解
@Import({EnableDiscoveryClientImportSelector.class})
public @interface EnableDiscoveryClient {
    boolean autoRegister() default true;
}
```

继续跟踪`@Import({EnableDiscoveryClientImportSelector.class})`

```java
@Order(2147483547)
public class EnableDiscoveryClientImportSelector extends SpringFactoryImportSelector<EnableDiscoveryClient> {
    public EnableDiscoveryClientImportSelector() {
    }
	// 核心方法
    // metadata 启动类上注解元信息
    public String[] selectImports(AnnotationMetadata metadata) {
        // 
        String[] imports = super.selectImports(metadata);
        AnnotationAttributes attributes = AnnotationAttributes.fromMap(metadata.getAnnotationAttributes(this.getAnnotationClass().getName(), true));
        // 获取@EnableDiscoveryClient的autoRegister属性，默认为true
        boolean autoRegister = attributes.getBoolean("autoRegister");
        if (autoRegister) {
            List<String> importsList = new ArrayList(Arrays.asList(imports));
            importsList.add("org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationConfiguration");
            // 导入AutoServiceRegistrationConfiguration配置类
            imports = (String[])importsList.toArray(new String[0]);
        } else {
            Environment env = this.getEnvironment();
            if (ConfigurableEnvironment.class.isInstance(env)) {
                ConfigurableEnvironment configEnv = (ConfigurableEnvironment)env;
                LinkedHashMap<String, Object> map = new LinkedHashMap();
                map.put("spring.cloud.service-registry.auto-registration.enabled", false);
                MapPropertySource propertySource = new MapPropertySource("springCloudDiscoveryClient", map);
                configEnv.getPropertySources().addLast(propertySource);
            }
        }

        return imports;
    }

    protected boolean isEnabled() {
        return (Boolean)this.getEnvironment().getProperty("spring.cloud.discovery.enabled", Boolean.class, Boolean.TRUE);
    }

    protected boolean hasDefaultFactory() {
        return true;
    }
}

```

我们可以发现这个类主要是导入一些配置，我们回头继续分析`@EnableDiscoveryClient`，这个注解主要是开启了一个`DiscoveryClient`实例。我们全局搜索DiscoveryClient



![discoverClient](H:\markdown\images\discoverClient.png)

​	可以发现，有一个接口，一个类，分别位于不同的包下，DiscoveryClient接口位于`spring-cloud-commons`包下，这是spring-cloud定义的用来发现服务的一套规范，通过该接口可以有效的屏蔽服务治理的实现细节，所以使用Spring Cloud构建的微服务应用可以方便地切换不同服务治理框架，而不同改动程序代码，只需要添加一些针对服务治理框架的配置即可。

​	DiscoveryClient类位于`com.netflix.eureka:eureka-client`包下，封装了eureka自身的一套服务发现的方法。

​	我们分别看下这两个类的继承关系图：

![tree](H:\markdown\images\tree.png)

​	分析完上面两个类的基本信息后，我们会想，这两个类有什么关系吗？既然EurekaDisConveryClient封装了原生的Eureka的操作，并且实现的也是服务发现的功能，那么它会不会持有原生EurekaClient的引用呢？，我们查看它的源码。

​	

```java
public class EurekaDiscoveryClient implements DiscoveryClient {
    public static final String DESCRIPTION = "Spring Cloud Eureka Discovery Client";
    // 引用了原生了EurekaClient
    private final EurekaClient eurekaClient;
    private final EurekaClientConfig clientConfig;

    /** @deprecated */
    @Deprecated
    public EurekaDiscoveryClient(EurekaInstanceConfig config, EurekaClient eurekaClient) {
        this(eurekaClient, eurekaClient.getEurekaClientConfig());
    }

    public EurekaDiscoveryClient(EurekaClient eurekaClient, EurekaClientConfig clientConfig) {
        this.clientConfig = clientConfig;
        this.eurekaClient = eurekaClient;
    }

    public String description() {
        return "Spring Cloud Eureka Discovery Client";
    }

    public List<ServiceInstance> getInstances(String serviceId) {
        List<InstanceInfo> infos = this.eurekaClient.getInstancesByVipAddress(serviceId, false);
        List<ServiceInstance> instances = new ArrayList();
        Iterator var4 = infos.iterator();

        while(var4.hasNext()) {
            InstanceInfo info = (InstanceInfo)var4.next();
            instances.add(new EurekaDiscoveryClient.EurekaServiceInstance(info));
        }

        return instances;
    }

    public List<String> getServices() {
        Applications applications = this.eurekaClient.getApplications();
        if (applications == null) {
            return Collections.emptyList();
        } else {
            List<Application> registered = applications.getRegisteredApplications();
            List<String> names = new ArrayList();
            Iterator var4 = registered.iterator();

            while(var4.hasNext()) {
                Application app = (Application)var4.next();
                if (!app.getInstances().isEmpty()) {
                    names.add(app.getName().toLowerCase());
                }
            }

            return names;
        }
    }

    public int getOrder() {
        return this.clientConfig instanceof Ordered ? ((Ordered)this.clientConfig).getOrder() : 0;
    }

    public static class EurekaServiceInstance implements ServiceInstance {
        private InstanceInfo instance;

        public EurekaServiceInstance(InstanceInfo instance) {
            Assert.notNull(instance, "Service instance required");
            this.instance = instance;
        }

        public InstanceInfo getInstanceInfo() {
            return this.instance;
        }

        public String getInstanceId() {
            return this.instance.getId();
        }

        public String getServiceId() {
            return this.instance.getAppName();
        }

        public String getHost() {
            return this.instance.getHostName();
        }

        public int getPort() {
            return this.isSecure() ? this.instance.getSecurePort() : this.instance.getPort();
        }

        public boolean isSecure() {
            return this.instance.isPortEnabled(PortType.SECURE);
        }

        public URI getUri() {
            return DefaultServiceInstance.getUri(this);
        }

        public Map<String, String> getMetadata() {
            return this.instance.getMetadata();
        }
    }
}
```

​	我们可以看到，SpringCloud的EurekaDisConveryClient的方法基本上都是通过原生EurekaClient实现的。所以接下来我们着重分析EurekaClient这个接口的实现类就行了，也就**com.netflix.discovery.DiscoveryClient**这两个类。

​	这个类的源码太长了，在这里我就不贴了，等下我们分析下它的一些方法，先看下头部的注释：

Eureka Client负载下面的任务：

- 向Eureka Server注册服务实例
- 向Eureka Server服务发起租约
- 当服务关闭期间，取消租约
- 查询Eureka Server中的服务实例列表

Eureka Client还需要配置一个Eureka Server的URL列表。

​	在具体研究Eureka Client负责完成的任务之前，我们先看看在哪里对Eureka Server的URL列表进行配置。根据我们配置的`eureka.client.serviceUrl.defaultZone`,通过serviceUrl可以找到该属性的相关加载属性。我们继续全局搜索`serviceUrl`

![serviceUrl](H:\markdown\images\serviceUrl.png)

我们查看其源码：

```java
public class EndpointUtils {
    private static final Logger logger = LoggerFactory.getLogger(EndpointUtils.class);
    public static final String DEFAULT_REGION = "default";
    public static final String DEFAULT_ZONE = "default";

    public EndpointUtils() {
    }
	// 省略部分代码
   ................
	// 主要关注此方法
    public static List<String> getServiceUrlsFromConfig(EurekaClientConfig clientConfig, String instanceZone, boolean preferSameZone) {
        List<String> orderedUrls = new ArrayList();
       // 从配置文件中获取一个region
        String region = getRegion(clientConfig);
        String[] availZones = 
       // 在当前的region中获取可得的Zone     
            clientConfig.getAvailabilityZones(clientConfig.getRegion());
       // 没有可得的Zone的话采用默认的defaultZone
       // 这也是我们配置中eureka.client.serviceUrl.defaultZone的由来
        if (availZones == null || availZones.length == 0) {
            availZones = new String[]{"default"};
        }
        logger.debug("T0he availability zone for the given region {} are {}", region, availZones);
       // 按照一定算法获取一个Zone
        int myZoneOffset = getZoneOffset(instanceZone, preferSameZone, availZones);
       String zone = availZones[myZoneOffset];
       // 获取这个Zone下的serviceUrls
        List<String> serviceUrls = clientConfig.getEurekaServerServiceUrls(availZones[myZoneOffset]);
        if (serviceUrls != null) {
            orderedUrls.put(zone, serviceUrls);
        }
		// 如果是最后一个Zone，则从0开始重新计数
        int currentOffset = myZoneOffset == availZones.length - 1 ? 0 : myZoneOffset + 1;

        while(currentOffset != myZoneOffset) {
            serviceUrls = clientConfig.getEurekaServerServiceUrls(availZones[currentOffset]);
            if (serviceUrls != null) {
                orderedUrls.put(zone, serviceUrls);
            }

            if (currentOffset == availZones.length - 1) {
                currentOffset = 0;
            } else {
                ++currentOffset;
            }
        }

        if (orderedUrls.size() < 1) {
            throw new IllegalArgumentException("DiscoveryClient: invalid serviceUrl specified!");
        } else {
            return orderedUrls;
        }
    }
	// 省略部分代码......
```

我们接下来看`clientConfig.getEurekaServerServiceUrls(availZones[currentOffset])`这个方法

```java
  public List<String> getEurekaServerServiceUrls(String myZone) {
        String serviceUrls = (String)this.serviceUrl.get(myZone);
        if (serviceUrls == null || serviceUrls.isEmpty()) {
            serviceUrls = (String)this.serviceUrl.get("defaultZone");
        }

        if (!StringUtils.isEmpty(serviceUrls)) {
            String[] serviceUrlsSplit = 
                // 这段代码说明serviceUrls，我们配置多少时应该以“,”号分隔
                StringUtils.commaDelimitedListToStringArray(serviceUrls);
            List<String> eurekaServiceUrls = new ArrayList(serviceUrlsSplit.length);
            String[] var5 = serviceUrlsSplit;
            int var6 = serviceUrlsSplit.length;

            for(int var7 = 0; var7 < var6; ++var7) {
                String eurekaServiceUrl = var5[var7];
                if (!this.endsWithSlash(eurekaServiceUrl)) {
                    // 这段配置说明，即使我们不以“/”结尾，默认也会给我们补上
                    eurekaServiceUrl = eurekaServiceUrl + "/";
                }

                eurekaServiceUrls.add(eurekaServiceUrl.trim());
            }

            return eurekaServiceUrls;
        } else {
            return new ArrayList();
        }
    }

```

​	当我们在微服务应用中是同Ribbon来实现服务调用时，对于Zone的设置可以在负载均衡时实现区域亲和特性：Ribbon的默认策略会优先访问同客户端处于一个Zone的服务端实例，只有当同一个Zone中没有可用的服务端实例时才会访问其他zone中的实例。所以通过Zone属性的定义，配合实际部署的物理结构，我们可以有效设计出对区域性故障的容错集群。

##### 服务注册：

​	在理解了多个服务注册中心信息的加载后，我们回头再看看DisCoverClient类是如何实现“服务注册”行为的，通过查看它的构造类，我们可以找到它调用了下面了这个函数。

​	

```java
private void initScheduledTasks() {
        int renewalIntervalInSecs;
        int expBackOffBound;
        if (this.clientConfig.shouldFetchRegistry()) {
            renewalIntervalInSecs = this.clientConfig.getRegistryFetchIntervalSeconds();
            expBackOffBound = this.clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
            this.scheduler.schedule(new TimedSupervisorTask("cacheRefresh", this.scheduler, this.cacheRefreshExecutor, renewalIntervalInSecs, TimeUnit.SECONDS, expBackOffBound, new DiscoveryClient.CacheRefreshThread()), (long)renewalIntervalInSecs, TimeUnit.SECONDS);
        }
			// 是否要注册到注册中心
        if (this.clientConfig.shouldRegisterWithEureka()) {
            renewalIntervalInSecs = this.instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
            expBackOffBound = this.clientConfig.getHeartbeatExecutorExponentialBackOffBound();
            logger.info("Starting heartbeat executor: renew interval is: {}", renewalIntervalInSecs);
            this.scheduler.schedule(new TimedSupervisorTask("heartbeat", this.scheduler, this.heartbeatExecutor, renewalIntervalInSecs, TimeUnit.SECONDS, expBackOffBound, new DiscoveryClient.HeartbeatThread()), (long)renewalIntervalInSecs, TimeUnit.SECONDS);
            this.instanceInfoReplicator = new InstanceInfoReplicator(this, this.instanceInfo, this.clientConfig.getInstanceInfoReplicationIntervalSeconds(), 2);
            this.statusChangeListener = new StatusChangeListener() {
                public String getId() {
                    return "statusChangeListener";
                }

                public void notify(StatusChangeEvent statusChangeEvent) {
                    if (InstanceStatus.DOWN != statusChangeEvent.getStatus() && InstanceStatus.DOWN != statusChangeEvent.getPreviousStatus()) {
                        DiscoveryClient.logger.info("Saw local status change event {}", statusChangeEvent);
                    } else {
                        DiscoveryClient.logger.warn("Saw local status change event {}", statusChangeEvent);
                    }

                    DiscoveryClient.this.instanceInfoReplicator.onDemandUpdate();
                }
            };
            if (this.clientConfig.shouldOnDemandUpdateStatusChange()) {
                this.applicationInfoManager.registerStatusChangeListener(this.statusChangeListener);
            }

           // 核心方法
            this.instanceInfoReplicator.start(this.clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
        } else {
            logger.info("Not registering with Eureka server per configuration");
        }

    }
```

​	上面这段代码，我们首先要看到`if (this.clientConfig.shouldRegisterWithEureka()) ``这句判断，代表了是否要将自身注册到Eureka Server上，如果为真的话，会到下面this.instanceInfoReplicator.start(this.clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());`我们跟踪这个方法

​	我们会发现这个类实现了Runnable接口，那么其核心方法就是run()方法。我们直接看它的run方法

```java
 public void run() {
        boolean var6 = false;

        ScheduledFuture next;
        label53: {
            try {
                var6 = true;
                this.discoveryClient.refreshInstanceInfo();
                Long dirtyTimestamp = this.instanceInfo.isDirtyWithTime();
                if (dirtyTimestamp != null) {
                    // 核心注册方法
                    this.discoveryClient.register();
                    this.instanceInfo.unsetIsDirty(dirtyTimestamp);
                    var6 = false;
                } else {
                    var6 = false;
                }
                break label53;
            } catch (Throwable var7) {
                logger.warn("There was a problem with the instance info replicator", var7);
                var6 = false;
            } finally {
                if (var6) {
                    ScheduledFuture next = this.scheduler.schedule(this, (long)this.replicationIntervalSeconds, TimeUnit.SECONDS);
                    this.scheduledPeriodicRef.set(next);
                }
            }

            next = this.scheduler.schedule(this, (long)this.replicationIntervalSeconds, TimeUnit.SECONDS);
            this.scheduledPeriodicRef.set(next);
            return;
        }

        next = this.scheduler.schedule(this, (long)this.replicationIntervalSeconds, TimeUnit.SECONDS);
        this.scheduledPeriodicRef.set(next);
    }
```

我们重点关注这个方法`this.discoveryClient.register();`

```java
  boolean register() throws Throwable {
        logger.info("DiscoveryClient_{}: registering service...", this.appPathIdentifier);

        EurekaHttpResponse httpResponse;
        try {
            // 核心方法
            httpResponse = this.eurekaTransport.registrationClient.register(this.instanceInfo);
        } catch (Exception var3) {
            logger.warn("DiscoveryClient_{} - registration failed {}", new Object[]{this.appPathIdentifier, var3.getMessage(), var3});
            throw var3;
        }

        if (logger.isInfoEnabled()) {
            logger.info("DiscoveryClient_{} - registration status: {}", this.appPathIdentifier, httpResponse.getStatusCode());
        }

        return httpResponse.getStatusCode() == Status.NO_CONTENT.getStatusCode();
    }
```

​	通过属性命名，大家就基本也能猜出来，注册操作也是通过REST请求方式进行的。同时，我们也能看到发起注册请求的时候，传入了一个InstanceInfo对象，该对象就是注册时客户端传给服务端的服务的元数据

##### 服务获取与服务续约：

​	顺着上面的思路，我们继续来看DiscoveryClient的initScheduleTasks函数，不难发现在其中还有两个定时任务，分别是服务获取跟服务续约。从源码中我们可以发现，“服务获取”任务相对于“服务续约”和“服务注册”任务更为独立。“服务续约”与“服务注册”在同一个if逻辑中，这个不难理解，服务注册到Eureka Server后，自然需要一个心跳去续约，防止被剔除，所以他们肯定是成对出现的。从源码中，我们更清楚的看到了之前所提到的，对于服务续约的相关时间的控制参数。

​	而“服务获取”的逻辑独立在一个if判断中，其判断依据就是我们之前所提到的`eureka.client.fetch-registry=true`参数，它默认为true,大部分情况下我们不需要关心。为了定期更新客户端的服务清单，以保证客户端能够访问确实健康的服务实例，“服务获取”的请求不会只限于服务启动，而是一个定时执行的任务。服务获取与服务续约也是以REST请求的方式进行的

#### 注册中心的处理：

​	通过上面的分析，可以看到所有的交互都是通过REST请求来发起的。下面我们来看看服务注册中心对这些请求的处理。Eureka Server对于各类REST请求的定义都位于com.netflix.eureka.resources包下。

​	以“服务注册”请求为例：

​	我们找到`com.netflix.eureka.resources.ApplicationResource`这个类，关注其addInstance方法

```java
 @POST
    @Consumes({"application/json", "application/xml"})
    public Response addInstance(InstanceInfo info, @HeaderParam("x-netflix-discovery-replication") String isReplication) {
        logger.debug("Registering instance {} (replication={})", info.getId(), isReplication);
        if (this.isBlank(info.getId())) {
            return Response.status(400).entity("Missing instanceId").build();
        } else if (this.isBlank(info.getHostName())) {
            return Response.status(400).entity("Missing hostname").build();
        } else if (this.isBlank(info.getIPAddr())) {
            return Response.status(400).entity("Missing ip address").build();
        } else if (this.isBlank(info.getAppName())) {
            return Response.status(400).entity("Missing appName").build();
        } else if (!this.appName.equals(info.getAppName())) {
            return Response.status(400).entity("Mismatched appName, expecting " + this.appName + " but was " + info.getAppName()).build();
        } else if (info.getDataCenterInfo() == null) {
            return Response.status(400).entity("Missing dataCenterInfo").build();
        } else if (info.getDataCenterInfo().getName() == null) {
            return Response.status(400).entity("Missing dataCenterInfo Name").build();
        } else {
            DataCenterInfo dataCenterInfo = info.getDataCenterInfo();
            if (dataCenterInfo instanceof UniqueIdentifier) {
                String dataCenterInfoId = ((UniqueIdentifier)dataCenterInfo).getId();
                if (this.isBlank(dataCenterInfoId)) {
                    boolean experimental = "true".equalsIgnoreCase(this.serverConfig.getExperimental("registration.validation.dataCenterInfoId"));
                    if (experimental) {
                        String entity = "DataCenterInfo of type " + dataCenterInfo.getClass() + " must contain a valid id";
                        return Response.status(400).entity(entity).build();
                    }

                    if (dataCenterInfo instanceof AmazonInfo) {
                        AmazonInfo amazonInfo = (AmazonInfo)dataCenterInfo;
                        String effectiveId = amazonInfo.get(MetaDataKey.instanceId);
                        if (effectiveId == null) {
                            amazonInfo.getMetadata().put(MetaDataKey.instanceId.getName(), info.getId());
                        }
                    } else {
                        logger.warn("Registering DataCenterInfo of type {} without an appropriate id", dataCenterInfo.getClass());
                    }
                }
            }
			// 核心方法
            this.registry.register(info, "true".equals(isReplication));
            return Response.status(204).build();
        }
    }
```

可以发现，在对注册信息进行了一系列的校验之后，会调用`org.springframework.cloud.netflix.eureka.server.InstanceRegistry`对象的register方法，我们跟踪这个方法

```java
    public void register(final InstanceInfo info, final boolean isReplication) {
        // 第一句
        this.handleRegistration(info, this.resolveInstanceLeaseDuration(info), isReplication);
          // 第二句
        super.register(info, isReplication);
    }
```

先看第一句：

```java
    private void handleRegistration(InstanceInfo info, int leaseDuration, boolean isReplication) {
        this.log("register " + info.getAppName() + ", vip " + info.getVIPAddress() + ", leaseDuration " + leaseDuration + ", isReplication " + isReplication);
        this.publishEvent(new EurekaInstanceRegisteredEvent(this, info, leaseDuration, isReplication));
    }
```

通过publishEvent将该服务注册的时间传播出去

再看第二句：

```java
    public void register(InstanceInfo info, boolean isReplication) {
        int leaseDuration = 90;
        if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
            leaseDuration = info.getLeaseInfo().getDurationInSecs();
        }

        super.register(info, leaseDuration, isReplication);
        this.replicateToPeers(PeerAwareInstanceRegistryImpl.Action.Register, info.getAppName(), info.getId(), info, (InstanceStatus)null, isReplication);
    }
```

这里继续调用父类的register方法

```java
    public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
        try {
            this.read.lock();
            Map<String, Lease<InstanceInfo>> gMap = (Map)this.registry.get(registrant.getAppName());
            EurekaMonitors.REGISTER.increment(isReplication);
            if (gMap == null) {
                ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap();
                gMap = (Map)this.registry.putIfAbsent(registrant.getAppName(), gNewMap);
                if (gMap == null) {
                    gMap = gNewMap;
                }
            }

            Lease<InstanceInfo> existingLease = (Lease)((Map)gMap).get(registrant.getId());
            if (existingLease != null && existingLease.getHolder() != null) {
                Long existingLastDirtyTimestamp = ((InstanceInfo)existingLease.getHolder()).getLastDirtyTimestamp();
                Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
                logger.debug("Existing lease found (existing={}, provided={}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                    logger.warn("There is an existing lease and the existing lease's dirty timestamp {} is greater than the one that is being registered {}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                    logger.warn("Using the existing instanceInfo instead of the new instanceInfo as the registrant");
                    registrant = (InstanceInfo)existingLease.getHolder();
                }
            } else {
                synchronized(this.lock) {
                    if (this.expectedNumberOfClientsSendingRenews > 0) {
                        ++this.expectedNumberOfClientsSendingRenews;
                        this.updateRenewsPerMinThreshold();
                    }
                }

                logger.debug("No previous lease information found; it is new registration");
            }

            Lease<InstanceInfo> lease = new Lease(registrant, leaseDuration);
            if (existingLease != null) {
                lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
            }

            ((Map)gMap).put(registrant.getId(), lease);
            synchronized(this.recentRegisteredQueue) {
                this.recentRegisteredQueue.add(new Pair(System.currentTimeMillis(), registrant.getAppName() + "(" + registrant.getId() + ")"));
            }

            if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
                logger.debug("Found overridden status {} for instance {}. Checking to see if needs to be add to the overrides", registrant.getOverriddenStatus(), registrant.getId());
                if (!this.overriddenInstanceStatusMap.containsKey(registrant.getId())) {
                    logger.info("Not found overridden id {} and hence adding it", registrant.getId());
                    this.overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
                }
            }

            InstanceStatus overriddenStatusFromMap = (InstanceStatus)this.overriddenInstanceStatusMap.get(registrant.getId());
            if (overriddenStatusFromMap != null) {
                logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
                registrant.setOverriddenStatus(overriddenStatusFromMap);
            }

            InstanceStatus overriddenInstanceStatus = this.getOverriddenInstanceStatus(registrant, existingLease, isReplication);
            registrant.setStatusWithoutDirty(overriddenInstanceStatus);
            if (InstanceStatus.UP.equals(registrant.getStatus())) {
                lease.serviceUp();
            }

            registrant.setActionType(ActionType.ADDED);
            this.recentlyChangedQueue.add(new AbstractInstanceRegistry.RecentlyChangedItem(lease));
            registrant.setLastUpdatedTimestamp();
            this.invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
            logger.info("Registered instance {}/{} with status {} (replication={})", new Object[]{registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication});
        } finally {
            this.read.unlock();
        }

    }
```

​	这里就是将InstanceInfo中的元数据信息存储在一个ConcurrentHashMap对象中。正如我们之前说的，注册中心存储了两层Map结构，第一层key存储服务名，也就是InstancInfo中的appName属性，第二层的key存储实例名，也就是InstanceInfo中的instanceId属性