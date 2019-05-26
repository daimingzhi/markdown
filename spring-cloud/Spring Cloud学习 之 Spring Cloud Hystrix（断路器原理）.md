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
		// 在半开路的情况下使用，若Hystrix命令调用成功，通过调用它将打开的断路器关闭
    	// 并重置度量指标 
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
              // 配置了强制打开，直接返回false
                return false;
            }
            if (properties.circuitBreakerForceClosed().get()) {
                // 配置了强制关闭，会允许所有请求，但同时也会调用isOpen()来执行断路器的计算逻辑
                isOpen();
              
                return true;
            }
            // 默认情况下，会用过!isOpen() || allowSingleTest()，来判断请求是否被允许
            // 也就是说，断路器闭合，或者断路器关闭，但是allowSingleTest()为true
            return !isOpen() || allowSingleTest()，;
        }

        public boolean allowSingleTest() {
            long timeCircuitOpenedOrWasLastTested = circuitOpenedOrLastTestedTime.get();
 			// 断路器打开的情况下，每隔一段事件允许请求一次，默认为5s
            // 此时断路器处于一个半开路的状态下
            if (circuitOpen.get() && System.currentTimeMillis() > timeCircuitOpenedOrWasLastTested + properties.circuitBreakerSleepWindowInMilliseconds().get()) {

                if (circuitOpenedOrLastTestedTime.compareAndSet(timeCircuitOpenedOrWasLastTested, System.currentTimeMillis())) {

                    return true;
                }
            }
            return false;
        }
	 // 判断当前断路器是否打开
        @Override
        public boolean isOpen() {
            if (circuitOpen.get()) {
				// 如果断路器打开的标志为true就直接返回true,，表示断路器在打开状态
                return true;
            }
			// 否则从度量指标对象中获取HealthCounts统计对象做进一步判断（该对象记录了一个滚动事件窗内			   的请求信息快照，默认时间窗为10秒）
            HealthCounts health = metrics.getHealthCounts();

            if (health.getTotalRequests() < 
                //如果他的QPS在预设的阈值范围内就返回false,表示断路器处于闭合状态。
                //该值的默认值为20
                properties.circuitBreakerRequestVolumeThreshold().get()) {

                return false;
            }

            if (health.getErrorPercentage() < 
                // 错误百分比在阈值内就返回false,该阈值默认为50%
                properties.circuitBreakerErrorThresholdPercentage().get()) {
                return false;
            } else {
                // 如果上面两个参数都不满足，就打开断路器
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