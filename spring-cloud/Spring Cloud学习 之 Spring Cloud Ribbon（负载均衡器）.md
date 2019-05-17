Spring Cloud学习 之 Spring Cloud Ribbon（负载均衡器源码分析）

[TOC]

​		通过之前的分析，我们已经对Spring Cloud如何使用Ribbon有了基本的了解。虽然Spring Cloud中定义了LoadBalancerClient作为负载均衡的通用接口，并且针对Ribbon实现了RibbonLoadBalancerClient,但是它在具体实现客户端负载均衡时，是通过Ribbon的ILoadBalancer接口实现的。在上一结进行分析时候，我们对该接口的实现接口已经做了一些简单的介绍，下面我们根据ILoadBalancer接口的实现类逐个看看它是如何实现客户端负载均衡的。我们先看下类的继承关系，然后一个个分析

​		![Ribbon_tree](H:\markdown\images\Ribbon_tree.png)

##### AbstractLoadBalancer:

​		AbstractLoadBalancer是ILoadBalancer接口的抽象实现。在该抽象类中定义了一个关于服务实例的分组枚举类ServerGroup，它包含以下三种不同类型。

```java
 public static enum ServerGroup {
     // 所有服务实例
        ALL,
     // 正常服务的实例
        STATUS_UP,
     // 停止服务的实例
        STATUS_NOT_UP;
        private ServerGroup() {
        }
    }
```

​		另外，还实现了一个chooseServer函数，该函数通过调用接口中的chooseServer（Object key）实现，其中参数key为null，表示在选择具体服务实例时忽略key的条件判断。最后还定义了两个抽象函数。

```java
	// 根据分组类型来获取不同的服务实例列表
public abstract List<Server> getServerList(AbstractLoadBalancer.ServerGroup var1);
	// 定义了获取LoadBalancerStats对象的方法，LoadBalancerStats对象被用来存储负载均衡器中各个服务实例当前的属性和统计信息。这些信息非常有用，我们可以利用这些信息来观察负载均衡器的运行情况，同时这些信息也是用来制定负载均衡策略的重要依据
    public abstract LoadBalancerStats getLoadBalancerStats();
```



##### BaseLoadBalancer：

​		BaseLoadBalancer类是Ribbon负载均衡器的基础实现类，在该类中定义了很多关于负载均衡器相关的基础内容。

```java
public class BaseLoadBalancer extends AbstractLoadBalancer implements PrimeConnectionListener, IClientConfigAware {
    private static Logger logger = LoggerFactory.getLogger(BaseLoadBalancer.class);
    private static final IRule DEFAULT_RULE = new RoundRobinRule();
    private static final BaseLoadBalancer.SerialPingStrategy DEFAULT_PING_STRATEGY = new BaseLoadBalancer.SerialPingStrategy((SyntheticClass_1)null);
    private static final String DEFAULT_NAME = "default";
    private static final String PREFIX = "LoadBalancer_";
    protected IRule rule;
    protected IPingStrategy pingStrategy;
    protected IPing ping;
    @Monitor(
        name = "LoadBalancer_AllServerList",
        type = DataSourceType.INFORMATIONAL
    )
    protected volatile List<Server> allServerList;
    @Monitor(
        name = "LoadBalancer_UpServerList",
        type = DataSourceType.INFORMATIONAL
    )
    protected volatile List<Server> upServerList;
    protected ReadWriteLock allServerLock;
    protected ReadWriteLock upServerLock;
    protected String name;
    protected Timer lbTimer;
    protected int pingIntervalSeconds;
    protected int maxTotalPingTimeSeconds;
    protected Comparator<Server> serverComparator;
    protected AtomicBoolean pingInProgress;
    protected LoadBalancerStats lbStats;
    private volatile Counter counter;
    private PrimeConnections primeConnections;
    private volatile boolean enablePrimingConnections;
    private IClientConfig config;
    private List<ServerListChangeListener> changeListeners;
    private List<ServerStatusChangeListener> serverStatusListeners;
    ......
```

- 定义并维护了两个存储服务实例Server对象的列表。一个用于存储所有服务实例的清单，一个用于存储正常服务的实例清单

  ```java
   @Monitor(
          name = "LoadBalancer_AllServerList",
          type = DataSourceType.INFORMATIONAL
      )
      protected volatile List<Server> allServerList;
      @Monitor(
          name = "LoadBalancer_UpServerList",
          type = DataSourceType.INFORMATIONAL
      )
      protected volatile List<Server> upServerList;
  ```

- 定义了之前我们提到的用来存储负载均衡器各服务实例属性和统计信息的LoadBalancerStats对象

- 定义了检查服务实例是否正常服务的IPing对象，在BaseLoadBalancer中默认为null，需要在构造时注入它的具体实现

- 定义了检查服务实例操作的执行策略对象IPingStrategy,在BaseLoadBalancer中默认使用了该类中定义的静态内部类SerialPingStrategy实现，根据源码，我们可以看到该策略默认采用线性遍历ping服务实例的方式去实现检查。该策略在当IPing的实现速度不理想，或是Server列表过大时，可能会影响系统性能，这时候需要通过实现IPingStrategy接口去重写boolean[] pingServers(IPing var1, Server[] var2);函数去扩张ping的执行策略

```java
 private static class SerialPingStrategy implements IPingStrategy {
        private SerialPingStrategy() {
        }

        public boolean[] pingServers(IPing ping, Server[] servers) {
            int numCandidates = servers.length;
            boolean[] results = new boolean[numCandidates];
            BaseLoadBalancer.logger.debug("LoadBalancer:  PingTask executing [{}] servers configured", numCandidates);

            for(int i = 0; i < numCandidates; ++i) {
                results[i] = false;

                try {
                    if (ping != null) {
                        results[i] = ping.isAlive(servers[i]);
                    }
                } catch (Exception var7) {
                    BaseLoadBalancer.logger.error("Exception while pinging Server: '{}'", servers[i], var7);
                }
            }

            return results;
        }
    }
```

- 定义了负载均衡的处理规则IRule对象，从BaseLoadBalancer中的chooseServer的实现源码中，我们可以知道，负载均衡器实际将服务实例选择任务委托给了IRule实例中的choose函数来实现。而在这里，默认初始化了RoundRobinRule为IRule的实现对象。RoundRobinRule实现了最基本且常用的线性负载均衡策略。

```java
public Server chooseServer(Object key) {
        if (this.counter == null) {
            this.counter = this.createCounter();
        }

        this.counter.increment();
        if (this.rule == null) {
            return null;
        } else {
            try {
                return this.rule.choose(key);
            } catch (Exception var3) {
                logger.warn("LoadBalancer [{}]:  Error choosing server for key {}", new Object[]{this.name, key, var3});
                return null;
            }
        }
    }
```

- 启动ping任务：在BaseLoadBalancer的默认构造函数中，会直接启动一个用于定时检查Server是否健康的任务。该任务默认的执行间隔为10秒

- 实现了ILoadBalancer接口定义的负载均衡器应具备以下一系列基本操作

  - addServers(List newServers)：向负载均衡器中增加新的服务实例列表，该实现将原本已经维护着的所有服务实例清单allServerList和新传入的服务实例清单newServers都加入到newList中，然后通过调用setServersList函数对newList进行处理，在BaseLoadBalancer中实现的时候，会使用新的列表覆盖旧的列表。而之后介绍的几个扩展实现类对于服务实例清单更新优化都是通过对setServersList函数重写来实现的

    ```java
    public void addServers(List<Server> newServers) {
            if (newServers != null && newServers.size() > 0) {
                try {
                    ArrayList<Server> newList = new ArrayList();
                    newList.addAll(this.allServerList);
                    newList.addAll(newServers);
                    this.setServersList(newList);
                } catch (Exception var3) {
                    logger.error("LoadBalancer [{}]: Exception while adding Servers", this.name, var3);
                }
            }
    
        }
    
    ```

- chooseServer(Object key);挑选一个具体的服务实例，在上面介绍IRule的时候，已经做了说明，这里不在赘述

- markServerDown(Server server);标记某个服务实例暂停服务。

  ```java
    public void markServerDown(Server server) {
          if (server != null && server.isAlive()) {
              logger.error("LoadBalancer [{}]:  markServerDown called on [{}]", this.name, server.getId());
              server.setAlive(false);
              this.notifyServerStatusChangeListener(Collections.singleton(server));
          }
      }
  ```

- getReachableServers();获取可用的服务实例列表。由于BaseLoadBalancer中单独维护了一个所有可用服务的实例清单，所以直接返回就可。

  ```java
  public List<Server> getReachableServers() {
      return Collections.unmodifiableList(this.upServerList);
  }
  ```

- getAllServers()；获取所有的服务实例列表。由于BaseLoadBalancer中当中单独维护了一个所有服务实例的清单列表，所以也是直接返回

  ```java
  public List<Server> getAllServers() {
      return Collections.unmodifiableList(this.allServerList);
  }
  ```

##### DynamicServerListLoadBalancer：

DynamicServerListLoadBalancer类继承了BaseLoadBalancer，它是对基础负载均衡器的扩展。**在该负载均衡器中，实现了服务实例清单在运行期的动态更新能力**，同时，**它还具备了对服务实例清单的过滤功能**，也就是说，我们可以通过过滤器来选择性的获取一批服务实例清单。下面我们具体来看看在该类中增加了一些什么内容

```java
public class DynamicServerListLoadBalancer<T extends Server> extends BaseLoadBalancer {
    private static final Logger LOGGER = LoggerFactory.getLogger(DynamicServerListLoadBalancer.class);
    boolean isSecure;
    boolean useTunnel;
    protected AtomicBoolean serverListUpdateInProgress;
    volatile ServerList<T> serverListImpl;
    volatile ServerListFilter<T> filter;
    protected final UpdateAction updateAction;
    protected volatile ServerListUpdater serverListUpdater;
    ........
```

###### ServerList:

从DynamicServerListLoadBalancer的成员定义中，我们马上可以发现新增了一个关于服务列表的操作对象ServerList<T> serverListImpl。其中泛型T从类名中对于T的限定DynamicServerListLoadBalancer<T extends Server>可以获知它是Server的一个子类，即代表了一个具体的服务实例的扩展类。而ServerList接口定义如下图所示:

```java
public interface ServerList<T extends Server> {
    // 获取初始化的服务实例清单
    List<T> getInitialListOfServers();
	// 获取更新的服务实例清单
    List<T> getUpdatedListOfServers();
}
```

查看源码，可以整理出下图所示的结构图：

![serverList_tree](H:\markdown\images\serverList_tree.png)

从上图中我们可以看到有多个ServerList的实现类，那么在DynamicServerListLoadBalancer中的ServerList默认配置到底使用了哪个具体实现呢？既然在该负载均衡器中需要实现服务实例的动态更新，那么势必需要Ribbon具备访问Eureka来获取服务实例)的能力，所以我们从Spring Cloud 整合Ribbon与Eureka的包，找到如下这个类

![Ribbon_Eureka](H:\markdown\images\Ribbon_Eureka.png)

可以看到如下的方法：

```java
@Bean
    @ConditionalOnMissingBean
    public ServerList<?> ribbonServerList(IClientConfig config, Provider<EurekaClient> eurekaClientProvider) {
        if (this.propertiesFactory.isSet(ServerList.class, this.serviceId)) {
            return (ServerList)this.propertiesFactory.get(ServerList.class, config, this.serviceId);
        } else {
            DiscoveryEnabledNIWSServerList discoveryServerList = new DiscoveryEnabledNIWSServerList(config, eurekaClientProvider);
            // 在这里做的创建
            DomainExtractingServerList serverList = new DomainExtractingServerList(discoveryServerList, config, this.approximateZoneFromHostname);
            return serverList;
        }
    }
```

可以看到，这里创建的是一个DomainExtractingServerList实例，同时DomainExtractingServerList类中的getInitialListOfServers() 跟getUpdatedListOfServers()其实委托给了内部定义的ServerList list对象，而该对象是通过创建DomainExtractingServerList时，由构造函数传入的DiscoveryEnabledNIWSServerList实现的。

```java
public class DomainExtractingServerList implements ServerList<DiscoveryEnabledServer> {
    private ServerList<DiscoveryEnabledServer> list;
    private final RibbonProperties ribbon;
    private boolean approximateZoneFromHostname;

    public DomainExtractingServerList(ServerList<DiscoveryEnabledServer> list, IClientConfig clientConfig, boolean approximateZoneFromHostname) {
        this.list = list;
        this.ribbon = RibbonProperties.from(clientConfig);
        this.approximateZoneFromHostname = approximateZoneFromHostname;
    }

    public List<DiscoveryEnabledServer> getInitialListOfServers() {
        List<DiscoveryEnabledServer> servers = this.setZones(this.list.getInitialListOfServers());
        return servers;
    }

    public List<DiscoveryEnabledServer> getUpdatedListOfServers() {
        List<DiscoveryEnabledServer> servers = this.setZones(this.list.getUpdatedListOfServers());
        return servers;
    }
```

那么DiscoveryEnabledNIWSServerList是如何实现这两个服务实例获取的呢？我们从源码中可以看到这两个方法都是通过该类中的一个私有函数obtainServersViaDiscovery通过服务发现机制来实现服务实例的获取的。

```java
public List<DiscoveryEnabledServer> getInitialListOfServers() {
    return this.obtainServersViaDiscovery();
}

public List<DiscoveryEnabledServer> getUpdatedListOfServers() {
    return this.obtainServersViaDiscovery();
}
```

```java
private List<DiscoveryEnabledServer> obtainServersViaDiscovery() {
        List<DiscoveryEnabledServer> serverList = new ArrayList<DiscoveryEnabledServer>();
       // 获取discoveryClient实例
        DiscoveryClient discoveryClient = DiscoveryManager.getInstance()
                .getDiscoveryClient();
        if (discoveryClient == null) {
            return new ArrayList<DiscoveryEnabledServer>();
        }
    // vipAddresses 可以理解为逻辑上的服务名称，如USER_SERVICE
        if (vipAddresses!=null){
            for (String vipAddress : vipAddresses.split(",")) {
                // if targetRegion is null, it will be interpreted as the same region of client			// 获取服务实例
                List<InstanceInfo> listOfinstanceInfo = discoveryClient.getInstancesByVipAddress(vipAddress, isSecure, targetRegion); 
                for (InstanceInfo ii : listOfinstanceInfo) {
                    // 状态为UP的实例转换成DiscoveryEnabledServer
                    if (ii.getStatus().equals(InstanceStatus.UP)) {

                        if(shouldUseOverridePort){
                            if(logger.isDebugEnabled()){
                                logger.debug("Overriding port on client name: " + clientName + " to " + overridePort);
                            }

                            // copy is necessary since the InstanceInfo builder just uses the original reference,
                            // and we don't want to corrupt the global eureka copy of the object which may be
                            // used by other clients in our system
                            InstanceInfo copy = new InstanceInfo(ii);

                            if(isSecure){
                                ii = new InstanceInfo.Builder(copy).setSecurePort(overridePort).build();
                            }else{
                                ii = new InstanceInfo.Builder(copy).setPort(overridePort).build();
                            }
                        }

                    	DiscoveryEnabledServer des = new DiscoveryEnabledServer(ii, isSecure);
                    	des.setZone(DiscoveryClient.getZone(ii));
                        serverList.add(des);
                    }
                }
                if (serverList.size()>0 && prioritizeVipAddressBasedServers){
                    break; // if the current vipAddress has servers, we dont use subsequent vipAddress based servers
                }
            }
        }
        return serverList;
    }
```

在DiscoveryEnabledNIWSServerList中通过EurekaClient从服务注册中心获取到最新的服务实例清单后，返回的List到了DomainExtractingServerList类中，将继续通过setZones函数进行处理。而这里的处理具体内容如下所示，主要完成将DiscoveryEnabledNIWSServerList返回的list列表中的元素，转换成内部定义的DiscoveryEnabledServer的子类对象DomainExtractingServer，在该对象的构造函数中将为服务实例对象设置一些必要的属性，比如id，zone，isAliveFlag，readyToServer等信息。

```java
 private List<DiscoveryEnabledServer> setZones(List<DiscoveryEnabledServer> servers) {
        List<DiscoveryEnabledServer> result = new ArrayList();
        boolean isSecure = this.ribbon.isSecure(true);
        boolean shouldUseIpAddr = this.ribbon.isUseIPAddrForServer();
        Iterator var5 = servers.iterator();

        while(var5.hasNext()) {
            DiscoveryEnabledServer server = (DiscoveryEnabledServer)var5.next();
            result.add(new DomainExtractingServer(server, isSecure, shouldUseIpAddr, this.approximateZoneFromHostname));
        }

        return result;
    }
```

###### ServerListUpdater:

通过上面的分析，我们已经知道了Ribbon与Eureka整合后，如何实现从Eureka Server中获取服务实例清单。那么它又是如何触发向Eureka Server去获取服务实例清单以及如何在获取到服务实例清单后更新本地的服务实例清单的呢？继续来看DynamicServerListLoadBalancer中的实现内容，我们可以很容易地找到下面定义的关于ServerListUpdater中内容:

```java
  protected final UpdateAction updateAction;
    protected volatile ServerListUpdater serverListUpdater;
```

我们来看下这个接口的定义：

```java
public interface ServerListUpdater {
    void start(ServerListUpdater.UpdateAction var1);

    void stop();

    String getLastUpdate();

    long getDurationSinceLastUpdateMs();

    int getNumberMissedCycles();

    int getCoreThreads();

    public interface UpdateAction {
        void doUpdate();
    }
}
```

再看下其实现类：

![serverList_updater](H:\markdown\images\serverList_updater.png)

可以看到，只有两个实现类

1. PollingServerListUpdater:动态服务列表更新的默认策略，也就是说DynamicServerListLoadBalancer负载均衡器中的默认实现就是它，它通过定时任务的方式进行服务列表的更新

2. EurekaNotificationServerListUpdater：该更新也可服务于DynamicServerListLoadBalancer负载均衡器，但是它的触发机制跟PollingServerListUpdater不同，它需要利用Eureka的事件监听器来驱动服务列表的更新操作

下面我们来看下PollingServerListUpdater的源码：

先看其start方法：

```java
public synchronized void start(final UpdateAction updateAction) {
    if (this.isActive.compareAndSet(false, true)) {
        Runnable wrapperRunnable = new Runnable() {
            // 创建了一个Runnable的线程实现
            public void run() {
                if (!PollingServerListUpdater.this.isActive.get()) {
                    if (PollingServerListUpdater.this.scheduledFuture != null) {
                        PollingServerListUpdater.this.scheduledFuture.cancel(true);
                    }

                } else {
                    try {
                        // 调用了上面提到的具体更新服务实例的方法
                        updateAction.doUpdate();
                        PollingServerListUpdater.this.lastUpdated = System.currentTimeMillis();
                    } catch (Exception var2) {
                        PollingServerListUpdater.logger.warn("Failed one update cycle", var2);
                    }

                }
            }
        };
        this.scheduledFuture = 
            // 启动了一个定时任务
            getRefreshExecutor().scheduleWithFixedDelay(wrapperRunnable, this.initialDelayMs, this.refreshIntervalMs, TimeUnit.MILLISECONDS);
    } else {
        logger.info("Already active, no-op");
    }

}
```

继续看其它内容，我们可以找到用于启动定时任务的两个重要参数initialDelayMs跟refreshIntervalMs的默认定义分别为1000跟30*1000，单位为毫秒。也就是说，更新服务实例在初始化之后延迟1秒后开始执行，并以30秒为周期重复执行。

###### ServerListFilter:

在了解了更新服务实例的定时任务是如何启动的之后，我们回到updateAction.doUpdate()调用的具体实现位置，在DynamicServerListLoadBalancer中，它的实际实现委托给了updateListOfServers函数，具体实现如下：

```java
public void updateListOfServers() {
    List<T> servers = new ArrayList();
    if (this.serverListImpl != null) {
        servers = this.serverListImpl.getUpdatedListOfServers();
        LOGGER.debug("List of Servers for {} obtained from Discovery client: {}", this.getIdentifier(), servers);
        if (this.filter != null) {
            servers = this.filter.getFilteredListOfServers((List)servers);
            LOGGER.debug("Filtered List of Servers for {} obtained from Discovery client: {}", this.getIdentifier(), servers);
        }
    }

    this.updateAllServerList((List)servers);
}
```

​		可以看到，这里终于用到了之前提到的ServerList的getUpdatedListOfServers()方法，通过之前的介绍已经知道这一步实现了Eureka Server中获取服务可用实例的列表。在获得了服务实例列表之后，这里由将引入一个新的对象filter，追溯该对象的定义，我们可以找到它是ServerListFilter定义的

​		ServerListFilter接口非常简单，该接口定义了一个方法List<T> getFilteredListOfServers(....)，主要用于实现对服务实例列表的过滤，通过传入的服务实例清单，根据一些规则返回过滤后的服务实例清单。

```java
public interface ServerListFilter<T extends Server> {
    List<T> getFilteredListOfServers(List<T> var1);
}
```

该接口的实现如下图所示：

![ServerListFilter](H:\markdown\images\ServerListFilter.png)

其中，除了ZonePreferenceServerListFilter的实现是Spring Cloud Ribbon中对Netfilx Ribbon的扩展实现外，其它均是Netflix Ribbon中的原生实现类。下面，我们可以分别看看这些过滤器的实现有什么特点。

- AbstractServerListFilter：这是一个抽象过滤器，在这里定义了过滤时需要的一个重要依据对象LoadBalancerStats，在我们之前介绍过，该对象存储了关于负载均衡器的一些属性和统计信息等

```java
public abstract class AbstractServerListFilter<T extends Server> implements ServerListFilter<T> {
    private volatile LoadBalancerStats stats;

    public AbstractServerListFilter() {
    }

    public void setLoadBalancerStats(LoadBalancerStats stats) {
        this.stats = stats;
    }

    public LoadBalancerStats getLoadBalancerStats() {
        return this.stats;
    }
}
```

- ZoneAffinityServerListFilter：该过滤器基于”区域感知（Zone Affinity）“的方式实现服务实例的过滤，也就是说，它会根据提供服务的实例所在的区域（Zone）与消费者自身所处的区域（Zone）进行比较，过滤掉那些不是同处一个区域的实例

  ```java
  public List<T> getFilteredListOfServers(List<T> servers) {
      if (this.zone != null && (this.zoneAffinity || this.zoneExclusive) && servers != null && servers.size() > 0) {
          List<T> filteredServers = Lists.newArrayList(Iterables.filter(servers, this.zoneAffinityPredicate.getServerOnlyPredicate()));
          if (this.shouldEnableZoneAffinity(filteredServers)) {
              return filteredServers;
          }
  
          if (this.zoneAffinity) {
              this.overrideCounter.increment();
          }
      }
  
      return servers;
  }
  ```

  从上面的源码中我们可以看到，对于服务实例列表的过滤时通过Iterables.filter(servers, this.zoneAffinityPredicate.getServerOnlyPredicate())来实现的，其中判断依据由zoneAffinityPredicate实现服务实例与消费者的Zone比较。而在过滤之后，这里并不会马上返回过滤的结果，而是通过shouldEnableZoneAffinity函数来判断是否启用”区域感知“的功能。从下面shouldEnableZoneAffinity方法的实现中，我们可以看到，它使用了LoadBalancerStats的getZoneSnapshot方法来获取这些过滤后的同区域实例的基础指标（包含实例数量，断路器断开数，活动请求数，实例平均负载等），根据一系列的算法求出下面的几个评价值并于设置的阈值进行比较，若有一个条件符合，就不启用”区域感知“过滤的服务清单。这一算法实现为集群出现区域故障时，依然可以依靠其他区域的实例进行正常服务提供了完善的高可用保障。同时，通过这里的介绍，我们也可以关联着来理解之前介绍Eureka的时候，提到的对于区域分配设计来保证跨区域故障的高可用问题。

  - blackOutServerPercentage：故障实例百分比（断路器断开数/实例数量）>=0.8
  - activeReqeustsPerServer：实例平均负载>=0.6
  - availableServers：可用实例数（实例数量-断路器断开数量）<2

```java
private boolean shouldEnableZoneAffinity(List<T> filtered) {
    if (!this.zoneAffinity && !this.zoneExclusive) {
        return false;
    } else if (this.zoneExclusive) {
        return true;
    } else {
        LoadBalancerStats stats = this.getLoadBalancerStats();
        if (stats == null) {
            return this.zoneAffinity;
        } else {
            logger.debug("Determining if zone affinity should be enabled with given server list: {}", filtered);
            ZoneSnapshot snapshot = stats.getZoneSnapshot(filtered);
            double loadPerServer = snapshot.getLoadPerServer();
            int instanceCount = snapshot.getInstanceCount();
            int circuitBreakerTrippedCount = snapshot.getCircuitTrippedCount();
            if ((double)circuitBreakerTrippedCount / (double)instanceCount < this.blackOutServerPercentageThreshold.get() && loadPerServer < this.activeReqeustsPerServerThreshold.get() && instanceCount - circuitBreakerTrippedCount >= this.availableServersThreshold.get()) {
                return true;
            } else {
                logger.debug("zoneAffinity is overriden. blackOutServerPercentage: {}, activeReqeustsPerServer: {}, availableServers: {}", new Object[]{(double)circuitBreakerTrippedCount / (double)instanceCount, loadPerServer, instanceCount - circuitBreakerTrippedCount});
                return false;
            }
        }
    }
}
```

- DefaultNIWSServerListFilter：该过滤器完成继承自ZoneAffinityServerListFilter，是默认的NIWS（Netflix Internal Web Service）过滤器

  ```java
  public class DefaultNIWSServerListFilter<T extends Server> extends ZoneAffinityServerListFilter<T> {
      public DefaultNIWSServerListFilter() {
      }
  }
  ```

- ServerListSubsetFilter：该过滤器也继承自ZoneAffinityServerListFilter，它非常适用于拥有大规模服务器集群（上百台或更多）的系统。因为它可以产生一个"区域感知"结果的子集，通过它还能够通过比较服务实例的通信失败数量和并发连接数来判定该服务是否健康来选择性的从服务实例列表中剔除那些相对不够健康的实例

- ZonePreferenceServerListFilter：Spring Cloud整合时新增的过滤器。若使用Spring Cloud整合Eureka 和Ribbon时会默认使用该过滤器。它实现了通过配置Eureka实例元数据的所属区域（Zone）来过滤出同区域的服务实例。如下的源码所示，它的实现非常简单，首先通过父类ZoneAffinityServerListFilter的过滤器来获取”区域感知“的服务实例列表，如果过滤的结果为空就直接返回父类获取的结果，如果不为空就返回通过消费者配置的Zone过滤后的结果

  ```java
  public List<Server> getFilteredListOfServers(List<Server> servers) {
      List<Server> output = super.getFilteredListOfServers(servers);
      if (this.zone != null && output.size() == servers.size()) {
          List<Server> local = new ArrayList();
          Iterator var4 = output.iterator();
  
          while(var4.hasNext()) {
              Server server = (Server)var4.next();
              if (this.zone.equalsIgnoreCase(server.getZone())) {
                  local.add(server);
              }
          }
  
          if (!local.isEmpty()) {
              return local;
          }
      }
  
      return output;
  }
  ```

  ##### ZoneAwareLoadBalancer：

  ​		ZoneAwareLoadBalancer是对DynamicServerListLoadBalancer的扩展。在DynamicServerListLoadBalancer中，我们可以看到它并没有重写选择具体服务实例的chooseServer函数，所以它依然会采用在BaseLoadBalancer中实现的算法。使用RoundRobinRule规则，以线性轮询的方式来选择调用的服务实例，该算法实现简单并没有区域（Zone）的概念，所以它会把所有实例视为一个Zone下的节点来看待，这样就会周期性的产生跨区域访问的情况，由于跨区域会产生更高的延迟，这些实例主要以防止区域性故障实现高可用为目的而不能作为常规访问实例，所以在多区域部署的情况下会有一定的性能问题，而该负载均衡器则可以避免这样的问题。那么是如何实现的呢？

  ​		首先，在ZoneAwareLoadBalancer中，我们可以发现，它并没有重写setServerList，说明实现服务实例清单的更新主逻辑没有修改，但我们会发现它复写了setServerListForZones。看到这里可能会有点陌生，因为它并不是接口定义的必备函数，所以我们不妨去父类中DynamicServerListLoadBalancer寻找一下该函数，我们可以找到下面的定义：

  ```java
  public void setServersList(List lsrv) {
      super.setServersList(lsrv);
      Map<String, List<Server>> serversInZones = new HashMap();
      Iterator var4 = lsrv.iterator();
  
      while(var4.hasNext()) {
          Server server = (Server)var4.next();
          this.getLoadBalancerStats().getSingleServerStat(server);
          String zone = server.getZone();
          if (zone != null) {
              zone = zone.toLowerCase();
              List<Server> servers = (List)serversInZones.get(zone);
              if (servers == null) {
                  servers = new ArrayList();
                  serversInZones.put(zone, servers);
              }
  
              ((List)servers).add(server);
          }
      }
  
      this.setServerListForZones(serversInZones);
  }
  
  protected void setServerListForZones(Map<String, List<Server>> zoneServersMap) {
      LOGGER.debug("Setting server list for zones: {}", zoneServersMap);
      this.getLoadBalancerStats().updateZoneServerMapping(zoneServersMap);
  }
  ```

  ​		setServerListForZones函数的调用位于更新服务实例清单函数setServersList的最后，同时从其实现的内容来看，它在父类DynamicServerListLoadBalancer中的作用是根据按区域Zone分组的实例列表，为负载均衡器中的LoadBalancerStats对象创建ZoneStats并放入Map集合中，每一个区域Zone对应一个ZoneStats，它用于存储每个Zone的一些状态跟统计信息

  ​		在ZoneAwareLoadBalancer中对setServerListForZones的复写如下：

  ```java
  protected void setServerListForZones(Map<String, List<Server>> zoneServersMap) {
      super.setServerListForZones(zoneServersMap);
      if (this.balancers == null) {
          this.balancers = new ConcurrentHashMap();
      }
  
      Iterator var2 = zoneServersMap.entrySet().iterator();
  
      Entry existingLBEntry;
      while(var2.hasNext()) {
          existingLBEntry = (Entry)var2.next();
          String zone = ((String)existingLBEntry.getKey()).toLowerCase();
          this.getLoadBalancer(zone).setServersList((List)existingLBEntry.getValue());
      }
  
      var2 = this.balancers.entrySet().iterator();
  
      while(var2.hasNext()) {
          existingLBEntry = (Entry)var2.next();
          if (!zoneServersMap.keySet().contains(existingLBEntry.getKey())) {
              ((BaseLoadBalancer)existingLBEntry.getValue()).setServersList(Collections.emptyList());
          }
      }
  
  }
  ```

  ​		可以看到，在该实现中创建了一个ConcurrentHashMap类型的balancers对象，他将用来存储每个Zone区域对应的负载均衡器。而具体的负载均衡器的创建则是通过在下面的第一个循环中调用getLoadBalancer函数来完成的，同时在创建负载均衡器的时候，会创建它的规则（如果当前实现中眉头IRule的实例，就创建一个AvaiilabilityFilteringRule规则；如果已经有具体实例，就克隆一个）。在创建完负载均衡器后又马上调用setServerList函数为其设置对应Zone区域的实例清单。而第二个循环则是对Zone区域中实例清单的检查，看看是否有Zone区域下已经没有实例了，是的话就将balancers中对应Zone区域的实例列表清空，该操作是为了后续选择节点时，防止过时的Zone区域统计信息干扰具体实例的选择算法

  ​		在了解了该负载均衡器是如何扩展服务实例清单的实现后，我们来看看它具体是如何选择服务实例，来实现对区域的识别的

  ```java
  public Server chooseServer(Object key) {
      // 只有可用的Zone大于1的时候才会执行
      // 否则直接执行父类的逻辑 
      if (ENABLED.get() && this.getLoadBalancerStats().getAvailableZones().size() > 1) {
          Server server = null;
  
          try {
              LoadBalancerStats lbStats = this.getLoadBalancerStats();
              Map<String, ZoneSnapshot> zoneSnapshot = 
               //  为当前负载均衡器中所有的Zone区域分别创建快照，保存在map中，用于之后的算法
                  ZoneAvoidanceRule.createSnapshot(lbStats);
              logger.debug("Zone snapshots: {}", zoneSnapshot);
              if (this.triggeringLoad == null) {
                  this.triggeringLoad = DynamicPropertyFactory.getInstance().getDoubleProperty("ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".triggeringLoadPerServerThreshold", 0.2D);
              }
  
              if (this.triggeringBlackoutPercentage == null) {
                  this.triggeringBlackoutPercentage = DynamicPropertyFactory.getInstance().getDoubleProperty("ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".avoidZoneWithBlackoutPercetage", 0.99999D);
              }
  
              Set<String> availableZones = 
                  // 获取可用的Zone区域集合，在该函数中会通过Zone区域快照中的统计数据来实现可用区					的挑选
                  ZoneAvoidanceRule.getAvailableZones(zoneSnapshot, this.triggeringLoad.get(), this.triggeringBlackoutPercentage.get());
              logger.debug("Available zones: {}", availableZones);
              if (availableZones != null && availableZones.size() < zoneSnapshot.keySet().size()) {
                  // 当获取的可用的Zone区域集合不为空，并且个数小于Zone区域总数，就随机选择一个				   Zone区域
                  String zone = ZoneAvoidanceRule.randomChooseZone(zoneSnapshot, availableZones);
                  logger.debug("Zone chosen: {}", zone);
                  if (zone != null) {
                      // 在确定了某个Zone区域后，则获取了对应Zone区域的服务均衡器，并调用
                      // chooseServer来选择具体的服务实例，而在chooseServer中将使用IRule接口					   // 的choose函数来选择具体的服务实例。在这里，IRlue接口的实现会使用							// ZoneAvoidanceRule来挑选出具体的服务实例
                      BaseLoadBalancer zoneLoadBalancer = this.getLoadBalancer(zone);
                      server = zoneLoadBalancer.chooseServer(key);
                  }
              }
          } catch (Exception var8) {
              logger.error("Error choosing server using zone aware logic for load balancer={}", this.name, var8);
          }
  
          if (server != null) {
              return server;
          } else {
              logger.debug("Zone avoidance logic is not invoked.");
              return super.chooseServer(key);
          }
      } else {
          logger.debug("Zone aware logic disabled or there is only one zone");
          return super.chooseServer(key);
      }
  }
  ```

  我们最后看下getAvailableZones函数：

  ```java
  public static Set<String> getAvailableZones(Map<String, ZoneSnapshot> snapshot, double triggeringLoad, double triggeringBlackoutPercentage) {
      if (snapshot.isEmpty()) {
          return null;
      } else {
          Set<String> availableZones = new HashSet(snapshot.keySet());
          if (availableZones.size() == 1) {
              return availableZones;
          } else {
              Set<String> worstZones = new HashSet();
              double maxLoadPerServer = 0.0D;
              boolean limitedZoneAvailability = false;
              Iterator var10 = snapshot.entrySet().iterator();
  
              while(true) {
                  while(var10.hasNext()) {
                      Entry<String, ZoneSnapshot> zoneEntry = (Entry)var10.next();
                      String zone = (String)zoneEntry.getKey();
                      ZoneSnapshot zoneSnapshot = (ZoneSnapshot)zoneEntry.getValue();
                      int instanceCount = zoneSnapshot.getInstanceCount();
                      if (instanceCount == 0) {
                          availableZones.remove(zone);
                          limitedZoneAvailability = true;
                      } else {
                          double loadPerServer = zoneSnapshot.getLoadPerServer();
                          if ((double)zoneSnapshot.getCircuitTrippedCount() / (double)instanceCount < triggeringBlackoutPercentage && loadPerServer >= 0.0D) {
                              if (Math.abs(loadPerServer - maxLoadPerServer) < 1.0E-6D) {
                                  worstZones.add(zone);
                              } else if (loadPerServer > maxLoadPerServer) {
                                  maxLoadPerServer = loadPerServer;
                                  worstZones.clear();
                                  worstZones.add(zone);
                              }
                          } else {
                              availableZones.remove(zone);
                              limitedZoneAvailability = true;
                          }
                      }
                  }
  
                  if (maxLoadPerServer < triggeringLoad && !limitedZoneAvailability) {
                      return availableZones;
                  }
  
                  String zoneToAvoid = randomChooseZone(snapshot, worstZones);
                  if (zoneToAvoid != null) {
                      availableZones.remove(zoneToAvoid);
                  }
  
                  return availableZones;
              }
          }
      }
  }
  ```

  主要逻辑如下：

  1. 首先他会剔除符合这些规则的Zone区域：所属实例数为0的Zone;Zone内实例的平均负载小于0，或者实例故障率（断路器断开次数/实例数）大于等于阈值（默认为0.999999）
  2. 然后根据Zone区域的实例平均负载计算出最差的Zone区域，这里的最差指的是实例平均负载最高的Zone区域
  3. 如果在上面的过程中没有符合剔除要求的区域，同时实例最大平均负载小于阈值（默认为20%），就直接返回所有Zone区域为可用区域。否则，从最坏Zone区域中随机选择一个，将它从可用Zone集合中剔除