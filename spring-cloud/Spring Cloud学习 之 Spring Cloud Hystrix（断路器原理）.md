Spring Cloud学习 之 Spring Cloud Hystrix（断路器原理）

断路器定义：

```java
public interface HystrixCircuitBreaker {
	
    // 每个Hystrix都通过它判断是否被执行
    public boolean allowRequest();
	
    // 返回当前断路器是否打开
    public boolean isOpen();
	
    // 用来闭合断路器
	void markSuccess();
	
    public static class Factory {
		// 维护了一个Hystrix命令与HystrixCircuitBreaker的关系
        // String类型的key通过HystrixCommandKey定义
        private static ConcurrentHashMap<String, HystrixCircuitBreaker> circuitBreakersByCommand = new ConcurrentHashMap<String, HystrixCircuitBreaker>();


        public static HystrixCircuitBreaker getInstance(HystrixCommandKey key, HystrixCommandGroupKey group, HystrixCommandProperties properties, HystrixCommandMetrics metrics) {
            HystrixCircuitBreaker previouslyCached = circuitBreakersByCommand.get(key.name());
            if (previouslyCached != null) {
                return previouslyCached;
            }

            HystrixCircuitBreaker cbForCommand = circuitBreakersByCommand.putIfAbsent(key.name(), new HystrixCircuitBreakerImpl(key, group, properties, metrics));
            if (cbForCommand == null) {

                return circuitBreakersByCommand.get(key.name());
            } else {

                return cbForCommand;
            }
        }
        
        public static HystrixCircuitBreaker getInstance(HystrixCommandKey key) {
            return circuitBreakersByCommand.get(key.name());
        }

static void reset() {
            circuitBreakersByCommand.clear();
        }
    }

static class HystrixCircuitBreakerImpl implements HystrixCircuitBreaker {
    // 断路器对应的HystrixCommand实例的属性对象
        private final HystrixCommandProperties properties;
    // 用来让HystrixCommand记录各类度量指标的对象
        private final HystrixCommand记录各类度量指标的对象Metrics metrics;
	// 断路器是否打开的标志，默认未false
        private AtomicBoolean circuitOpen = new AtomicBoolean(false);
	// 断路器上次打开的时间戳
		private AtomicLong circuitOpenedOrLastTestedTime = new AtomicLong();
        protected HystrixCircuitBreakerImpl(HystrixCommandKey key, HystrixCommandGroupKey commandGroup, HystrixCommandProperties properties, HystrixCommandMetrics metrics) {
            this.properties = properties;
            this.metrics = metrics;
        }

        public void markSuccess() {
            if (circuitOpen.get()) {
                if (circuitOpen.compareAndSet(true, false)) {
                  
                    metrics.resetStream();
                }
            }
        }
// 判断请求是否被允许，先根据配置对象中断路器
        @Override
        public boolean allowRequest() {
            if (properties.circuitBreakerForceOpen().get()) {
              
                return false;
            }
            if (properties.circuitBreakerForceClosed().get()) {
                isOpen();
              
                return true;
            }
            return !isOpen() || allowSingleTest();
        }

        public boolean allowSingleTest() {
            long timeCircuitOpenedOrWasLastTested = circuitOpenedOrLastTestedTime.get();
 
            if (circuitOpen.get() && System.currentTimeMillis() > timeCircuitOpenedOrWasLastTested + properties.circuitBreakerSleepWindowInMilliseconds().get()) {

                if (circuitOpenedOrLastTestedTime.compareAndSet(timeCircuitOpenedOrWasLastTested, System.currentTimeMillis())) {

                    return true;
                }
            }
            return false;
        }

        @Override
        public boolean isOpen() {
            if (circuitOpen.get()) {

                return true;
            }

            HealthCounts health = metrics.getHealthCounts();

            if (health.getTotalRequests() < properties.circuitBreakerRequestVolumeThreshold().get()) {

                return false;
            }

            if (health.getErrorPercentage() < properties.circuitBreakerErrorThresholdPercentage().get()) {
                return false;
            } else {
                if (circuitOpen.compareAndSet(false, true)) {
                    circuitOpenedOrLastTestedTime.set(System.currentTimeMillis());
                    return true;
                } else {
                   
                    return true;
                }
            }
        }

    }
// 定义了一个什么都不做的断路器
 static class NoOpCircuitBreaker implements HystrixCircuitBreaker {

        @Override
        public boolean allowRequest() {
            return true;
        }

        @Override
        public boolean isOpen() {
            return false;
        }

        @Override
        public void markSuccess() {

        }

    }

}
```