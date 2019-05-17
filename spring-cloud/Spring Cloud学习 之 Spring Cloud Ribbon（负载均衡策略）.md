Spring Cloud学习 之 Spring Cloud Ribbon（负载均衡策略）

[TOC]

​		通过之前的源码阅读，我们已经对Ribbon实现的负载均衡器以及其中包含的服务实例过滤器，服务实例信息的存储对象，区域的信息快照等都有了深入的认设和理解，但对于负载均衡器中的服务实例选择策略知识讲解了几个默认实现的内容，而对于IRule的其他实现还没有详细解读，下面我们来看看在Ribbon中都提供了哪些负载均衡策略。

​		![IRule_tree](H:\markdown\images\IRule_tree.png)

##### AbstractLoadBalancerRule：

​			负载均衡策略的抽象类，在该抽象类中定义了负载均衡器ILoadBalancer对象，该对象能够在具体实现选择服务策略时，获取到一些负载均衡器中维护的信息来作为分配依据，并以此设计一些算法来实现针对特定场景高效策略。

```java
public abstract class AbstractLoadBalancerRule implements IRule, IClientConfigAware {
    private ILoadBalancer lb;

    public AbstractLoadBalancerRule() {
    }

    public void setLoadBalancer(ILoadBalancer lb) {
        this.lb = lb;
    }

    public ILoadBalancer getLoadBalancer() {
        return this.lb;
    }
}
```

##### RandomRule:

```java
/**
 * Randomly choose from all living servers
 */
@edu.umd.cs.findbugs.annotations.SuppressWarnings(value = "RCN_REDUNDANT_NULLCHECK_OF_NULL_VALUE")
public Server choose(ILoadBalancer lb, Object key) {
    if (lb == null) {
        return null;
    }
    Server server = null;

    while (server == null) {
        if (Thread.interrupted()) {
            return null;
        }
        List<Server> upList = lb.getReachableServers();
        List<Server> allList = lb.getAllServers();

        int serverCount = allList.size();
        if (serverCount == 0) {
            /*
             * No servers. End regardless of pass, because subsequent passes
             * only get more restrictive.
             */
            return null;
        }
		// 随机选取一个
        int index = chooseRandomInt(serverCount);
        server = upList.get(index);

        if (server == null) {
            /*
             * The only time this should happen is if the server list were
             * somehow trimmed. This is a transient condition. Retry after
             * yielding.
             */
            Thread.yield();
            continue;
        }

        if (server.isAlive()) {
            return (server);
        }

        // Shouldn't actually happen.. but must be transient or a bug.
        server = null;
        Thread.yield();
    }

    return server;

}
```

##### RoundRobinRule：

该策略实现了按照线性轮询的方式一次选择每个服务实例的功能。它的具体实现如下，其详细结构与RandomRule非常类似。除了循环条件不同外，就是从可用列表中获取所谓的逻辑不同。从循环条件中，我们可以看到增加了一个count计数变量，该变量会在每次循环后累加，也就是说，如果一直选择不到server超过10次，那么就会结束尝试。

```java
public Server choose(ILoadBalancer lb, Object key) {
    if (lb == null) {
        log.warn("no load balancer");
        return null;
    }

    Server server = null;
    int count = 0;
    while (server == null && count++ < 10) {
        List<Server> reachableServers = lb.getReachableServers();
        List<Server> allServers = lb.getAllServers();
        int upCount = reachableServers.size();
        int serverCount = allServers.size();

        if ((upCount == 0) || (serverCount == 0)) {
            log.warn("No up servers available from load balancer: " + lb);
            return null;
        }

        int nextServerIndex = incrementAndGetModulo(serverCount);
        server = allServers.get(nextServerIndex);

        if (server == null) {
            /* Transient. */
            Thread.yield();
            continue;
        }

        if (server.isAlive() && (server.isReadyToServe())) {
            return (server);
        }

        // Next.
        server = null;
    }

    if (count >= 10) {
        log.warn("No available alive servers after 10 tries from load balancer: "
                + lb);
    }
    return server;
}
```

##### RetryRule:

该策略实现了一个具备重试机制的实例选择功能。从下面的实现中我们可以看到，在其内部还定义了一个IRule对象，默认使用了RoundRobinRule实例。而在choose方法中则实现了对内部定义策略反复进行尝试的策略，若期间能够选择到实例就返回，若选择不到就根据设置的尝试时间为阈值，当超过该阈值后就返回null。

```java
public Server choose(ILoadBalancer lb, Object key) {
   long requestTime = System.currentTimeMillis();
   long deadline = requestTime + maxRetryMillis;

   Server answer = null;

   answer = subRule.choose(key);

   if (((answer == null) || (!answer.isAlive()))
         && (System.currentTimeMillis() < deadline)) {
		// 创建了一个线程终止任务并开始
      InterruptTask task = new InterruptTask(deadline
            - System.currentTimeMillis());
		
      while (!Thread.interrupted()) {
         answer = subRule.choose(key);

         if (((answer == null) || (!answer.isAlive()))
               && (System.currentTimeMillis() < deadline)) {
            /* pause and retry hoping it's transient */
            Thread.yield();
         } else {
            break;
         }
      }

      task.cancel();
   }

   if ((answer == null) || (!answer.isAlive())) {
      return null;
   } else {
      return answer;
   }
}
```

##### WeightedResponseTimeRule：

```java
@edu.umd.cs.findbugs.annotations.SuppressWarnings(value = "RCN_REDUNDANT_NULLCHECK_OF_NULL_VALUE")
@Override
public Server choose(ILoadBalancer lb, Object key) {
    if (lb == null) {
        return null;
    }
    Server server = null;

    while (server == null) {
        // get hold of the current reference in case it is changed from the other thread
        List<Double> currentWeights = accumulatedWeights;
        if (Thread.interrupted()) {
            return null;
        }
        List<Server> allList = lb.getAllServers();

        int serverCount = allList.size();

        if (serverCount == 0) {
            return null;
        }

        int serverIndex = 0;

        // last one in the list is the sum of all weights
        double maxTotalWeight = currentWeights.size() == 0 ? 0 : currentWeights.get(currentWeights.size() - 1); 
        // No server has been hit yet and total weight is not initialized
        // fallback to use round robin
        if (maxTotalWeight < 0.001d || serverCount != currentWeights.size()) {
            server =  super.choose(getLoadBalancer(), key);
            if(server == null) {
                return server;
            }
        } else {
            // generate a random weight between 0 (inclusive) to maxTotalWeight (exclusive)
            double randomWeight = random.nextDouble() * maxTotalWeight;
            // pick the server index based on the randomIndex
            int n = 0;
            for (Double d : currentWeights) {
                if (d >= randomWeight) {
                    serverIndex = n;
                    break;
                } else {
                    n++;
                }
            }

            server = allList.get(serverIndex);
        }

        if (server == null) {
            /* Transient. */
            Thread.yield();
            continue;
        }

        if (server.isAlive()) {
            return (server);
        }

        // Next.
        server = null;
    }
    return server;
}
```

该策略是对RoundRobinRule的扩展，增加了根据实例的运行情况来计算权重，并根据权重来挑选实例，以达到更优的分配效果，它的实现主要有三个核心内容。

###### 定时任务：

WeightedResponseTimeRule策略在初始化的时候会通过**serverWeightTimer**启动一个定时任务，用来为每个服务实例计算权重，该任务默认30秒执行一次。

###### 权重计算：

1. 根据LoadBalancerStats中记录的每个实例的统计信息，累加所有实例的平均响应时间，得到总平均响应时间
2. 为负载均衡器中维护的实例清单逐个计算权重

###### 实例选择：

1. 获取权重区间的最后一个值，其实也就是总权重
2. 如果总权重，低于0.01d，采用线性轮询的策略
3. 如果总权重大于等于0.01，就产生一个权重区间内的随机数
4. 遍历维护的权重清单，若权重大于等于随机得到的数值，就选择这个实例

##### ClientConfigEnabledRoundRobinRule：

```java
@Override
public Server choose(Object key) {
    if (roundRobinRule != null) {
        return roundRobinRule.choose(key);
    } else {
        throw new IllegalArgumentException(
                "This class has not been initialized with the RoundRobinRule class");
    }
}
```

该策略比较特殊，我们一般不直接使用它。因为它本身并没有实现什么特殊的处理逻辑，正如源码所示，他在内部直接引用了roundRobinRule。虽然我们不会直接使用该策略，但是通过继承该策略，默认的choose就实现了线性轮询机制，在子类中做一些高级策略时通常有可能会存在一些无法实施的情况，那么就可以用父类的实现作为备选。

##### BestAvailableRule：

该策略继承自ClientConfigEnabledRoundRobinRule，在实现中它注入了负载均衡器的统计对象LoadBalancerStats，同时在具体的choose算法中利用LoadBalancerStats保存的实例统计信息来选择满足要求的实例。从如下源码中我们可以看到，它通过遍历负载据衡器中维护的所有服务shillings，会过滤掉故障的实例，并找出并发请求最小的一个，所以该策略的特性是可选出最空闲的实例。

```java
@Override
public Server choose(Object key) {
    if (loadBalancerStats == null) {
        return super.choose(key);
    }
    List<Server> serverList = getLoadBalancer().getAllServers();
    int minimalConcurrentConnections = Integer.MAX_VALUE;
    long currentTime = System.currentTimeMillis();
    Server chosen = null;
    for (Server server: serverList) {
        ServerStats serverStats = loadBalancerStats.getSingleServerStat(server);
        if (!serverStats.isCircuitBreakerTripped(currentTime)) {
            int concurrentConnections = serverStats.getActiveRequestsCount(currentTime);
            if (concurrentConnections < minimalConcurrentConnections) {
                minimalConcurrentConnections = concurrentConnections;
                chosen = server;
            }
        }
    }
    if (chosen == null) {
        return super.choose(key);
    } else {
        return chosen;
    }
}
```

##### PredicateBasedRule：

​				这是一个抽象策略，它也继承了ClientConfigEnabledRoundRobinRule，从命名中可以猜出这是一个基于Predicate实现的策略，Predicate是Google Guaua Collection工具对集合进行过滤的条件接口。

​				如下面的源码所示，它定义了一个抽象函数getPredicate来获取AbstractServerPredicate对象的实现，而在choose函数中，通过AbstractServerPredicate的chooseRandomlyAfterFiltering函数来选出具体的服务实例。从该函数的命名我们也能大致猜到他的基础逻辑：先通过子类实现的Predicate逻辑来过滤一部分服务实例，然后再以线性轮询的方式从过滤后的实例清单中选出一个。

```java
@Override
public Server choose(Object key) {
    ILoadBalancer lb = getLoadBalancer();
    Optional<Server> server = getPredicate().chooseRoundRobinAfterFiltering(lb.getAllServers(), key);
    if (server.isPresent()) {
        return server.get();
    } else {
        return null;
    }       
}
```

```java
public Optional<Server> chooseRoundRobinAfterFiltering(List<Server> servers, Object loadBalancerKey) {
    // 获取过滤后的服务实例列表
    List<Server> eligible = getEligibleServers(servers, loadBalancerKey);
    if (eligible.size() == 0) {
        return Optional.absent();
    }
    return Optional.of(eligible.get(incrementAndGetModulo(eligible.size())));
}
```

​				在了解了整体逻辑之后，我们来详细看看实现过滤功能的getEligibleServers函数。从源码上看，它的实现结构简单清晰，通过遍历服务清单，使用this.apply方法来判断实例是否需要保留，如果是就添加到结果列表中。

​				可能到这里，不熟悉Google Guaua Collection集合工具的读者会感到困惑，这个apply在AbstractServerPredicate找不到它的定义，那么它是如何实现过滤的呢？实现上AbstractServerPredicate实现了com.google.common.base.Predicate接口，而apply方法是该接口中的定义，主要用来实现过滤条件的判断逻辑，它输入的参数则是过滤条件需要用到的一些信息。既然在AbstractServerPredicate我们未能找到apply函数的实现，所以这里chooseRoundRobinAfterFiltering只是定义了一个模板策略：”先过滤清单，再轮询选择“。对于如何过滤，需要我们在AbstractServerPredicate的子类中实现apply方法来确定具体的策略。

```java
/**
 * Get servers filtered by this predicate from list of servers. 
 */
public List<Server> getEligibleServers(List<Server> servers, Object loadBalancerKey) {
    if (loadBalancerKey == null) {
        return ImmutableList.copyOf(Iterables.filter(servers, this.getServerOnlyPredicate()));            
    } else {
        List<Server> results = Lists.newArrayList();
        for (Server server: servers) {
            if (this.apply(new PredicateKey(loadBalancerKey, server))) {
                results.add(server);
            }
        }
        return results;            
    }
}
```

##### AvailabilityFilteringRule：

```java
/**
 * This method is overridden to provide a more efficient implementation which does not iterate through
 * all servers. This is under the assumption that in most cases, there are more available instances 
 * than not. 
 */
@Override
public Server choose(Object key) {
    int count = 0;
    Server server = roundRobinRule.choose(key);
    while (count++ <= 10) {
        if (predicate.apply(new PredicateKey(server))) {
            return server;
        }
        server = roundRobinRule.choose(key);
    }
    return super.choose(key);
}
```

​				首先来说，这个类自身对choose方法进行了一些改进，它并没有像在父类那样，先遍历所有的节点进行过滤，然后在过滤后的集合中选择实例，而是先以线性的方式选择一个实例，接着用过滤条件来判断该实例是否满足要求，若满足就直接使用该实例，如不满足要求就再选择下一个实例，并检查是否满足要求，如此循环进行，当这个过程重复了10次还是没有找到符合要求的实例，就采用父类实现的方案。

​				简单来说，该策略通过线性抽样的方式直接尝试寻找可用且较空闲的实例来使用，优化了父类每次都要遍历所有实例的开销。

我们再来看其apply方法的实现：

```java
@Override
public boolean apply(@Nullable PredicateKey input) {
    LoadBalancerStats stats = getLBStats();
    if (stats == null) {
        return true;
    }
    return !shouldSkipServer(stats.getSingleServerStat(input.getServer()));
}

//核心判断方法
private boolean shouldSkipServer(ServerStats stats) {
    	// 是否采用了断路器
    if ((CIRCUIT_BREAKER_FILTERING.get() 
         &&
         // 是否被断开
         stats.isCircuitBreakerTripped()) 
        // 实例的并发请求数大于阈值
            || stats.getActiveRequestsCount() >= activeConnectionsLimit.get()) {
        return true;
    }
    return false;
}
```

##### ZoneAvoidanceRule：

```java
public ZoneAvoidanceRule() {
    super();
    ZoneAvoidancePredicate zonePredicate = new ZoneAvoidancePredicate(this);
    AvailabilityPredicate availabilityPredicate = new AvailabilityPredicate(this);
    compositePredicate = createCompositePredicate(zonePredicate, availabilityPredicate);
}
// 复合条件
private CompositePredicate createCompositePredicate(ZoneAvoidancePredicate p1, AvailabilityPredicate p2) {
    return CompositePredicate.withPredicates(p1, p2)
                         .addFallbackPredicate(p2)
                         .addFallbackPredicate(AbstractServerPredicate.alwaysTrue())
                         .build();
    
}
```

​				ZoneAvoidanceRule在实现的时候并没有像AvailabilityFilteringRule那样重写choose函数来优化，所以它完全遵循了父类的过滤主逻辑：”先过滤清单，再轮询选择“。其中过滤清单的条件就是我们上面提到的以ZoneAvoidancePredicate为主过滤条件，AvailabilityFilteringRule为次过滤条件从CompositePredicate的源码片段中我们可以看到它定义了一个主过滤条件AbstractServerPredicate以及一组次过滤条件列表，所以它的过滤列表是可以拥有多个的，并且由于它采用了List存储所以此过滤条件是按顺序执行的。

```java
public class CompositePredicate extends AbstractServerPredicate {

    private AbstractServerPredicate delegate;
    
    private List<AbstractServerPredicate> fallbacks = Lists.newArrayList();
        
    private int minimalFilteredServers = 1;
    
    private float minimalFilteredPercentage = 0;    
    ......
        /**
     * Get the filtered servers from primary predicate, and if the number of the filtered servers
     * are not enough, trying the fallback predicates  
     */
    @Override
    public List<Server> getEligibleServers(List<Server> servers, Object loadBalancerKey) {			// 先用主过滤条件进行过滤
        List<Server> result = super.getEligibleServers(servers, loadBalancerKey);
        Iterator<AbstractServerPredicate> i = fallbacks.iterator();
        while (!(result.size() >= minimalFilteredServers && result.size() > (int) (servers.size() * minimalFilteredPercentage))
                && i.hasNext()) {
            // 再用此过滤条件一个个过滤
            AbstractServerPredicate predicate = i.next();
            result = predicate.getEligibleServers(servers, loadBalancerKey);
        }
        return result;
    }
```

