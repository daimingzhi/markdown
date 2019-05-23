Spring 学习 之  再探publish-event机制

我们要知道的是，Spring的publish-event使用的是监听者模式

> **监听者模式**包含了一个监听者Listener与之对应的事件Event，还有一个事件发布者EventPublish，过程就是EventPublish发布一个事件，被监听者捕获到，然后执行事件相应的方法

监听者模式跟观察者模式的区别：

> **观察者模式**是一对多的模式，一个被观察者Observable和多个观察者Observer，被观察者中存储了所有的观察者对象，当被观察者接收到一个外界的消息，就会遍历广播推算消息给所有的观察者

1. 首先我们看下两个发布事件的方法，可以发现，最后都会调到`void publishEvent(Object event)`

```java
public interface ApplicationEventPublisher {

   default void publishEvent(ApplicationEvent event) {
      publishEvent((Object) event);
   }
    
   void publishEvent(Object event);

}
```

2. 继续跟踪代码：

   ```java
   protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
      Assert.notNull(event, "Event must not be null");
   
      // 如果是一个ApplicationEvent，就强转
      // 如果不是的话，包装成一个PayloadApplicationEvent
      ApplicationEvent applicationEvent;
      if (event instanceof ApplicationEvent) {
         applicationEvent = (ApplicationEvent) event;
      }
      else {
         applicationEvent = new PayloadApplicationEvent<>(this, event);
         if (eventType == null) {
             // 包装成一个PayloadApplicationEvent，并获取实际的带泛型的事件类型
             // 比如PayloadApplicationEvent<MyEvent>
            eventType = ((PayloadApplicationEvent<?>) applicationEvent).getResolvableType();
         }
      }
   
      // Multicast right now if possible - or lazily once the multicaster is initialized
      if (this.earlyApplicationEvents != null) {
         this.earlyApplicationEvents.add(applicationEvent);
      }
      else {
          // 1.获取一个applicationEventMulticaster
          // 2.广播事件
         getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
      }
   
      // 如果当前容易存在父容器，父容器也要发布事件
      if (this.parent != null) {
         if (this.parent instanceof AbstractApplicationContext) {
            ((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
         }
         else {
            this.parent.publishEvent(event);
         }
      }
   }
   ```

   3. 我们来看下ApplicationEventMulticaster的初始化

   ```java
   protected void initApplicationEventMulticaster() {
   		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
       // 我们自定义了ApplicationEventMulticaster，就直接用我们自定义的
   		if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
   			this.applicationEventMulticaster =
   					beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
   			if (logger.isTraceEnabled()) {
   				logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
   			}
   		}
   		else {
               // 如果我们没定义ApplicationEventMulticaster，默认使用SimpleApplicationEventMulticaster
   			this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
   			beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
   			if (logger.isTraceEnabled()) {
   				logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
   						"[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
   			}
   		}
   	}
   ```

   ​				通过这段代码，我们可以知道，我们如果申明了一个ApplicationEventMulticaster类，会覆盖默认的SimpleApplicationEventMulticaster。

   4. 注册监听者

   ```java
   protected void registerListeners() {
      // 把提前存储好的监听器添加到监听器容器中
      for (ApplicationListener<?> listener : getApplicationListeners()) {
         getApplicationEventMulticaster().addApplicationListener(listener);
      }
   
      // Do not initialize FactoryBeans here: We need to leave all regular beans
      // uninitialized to let post-processors apply to them!
      String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
      for (String listenerBeanName : listenerBeanNames) {
         getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
      }
   
      // Publish early application events now that we finally have a multicaster...
      Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
      this.earlyApplicationEvents = null;
      if (earlyEventsToProcess != null) {
         for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
            getApplicationEventMulticaster().multicastEvent(earlyEvent);
         }
      }
   }
   ```

   5. 看完上面的代码，我相信大家多spring的publish-event机制已经有了一定的了解，现在我们再来探讨一个问题-------**怎么实现异步机制**？

   我继续看`getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType)`这段代码：

   ```java
   @Override
   public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
      ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
      for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
         Executor executor = getTaskExecutor();
         if (executor != null) {
            executor.execute(() -> invokeListener(listener, event));
         }
         else {
            invokeListener(listener, event);
         }
      }
   }
   ```

   可以发现，在通知监听者的时候，会去检查ApplicationEventMulticaster对象中是否配置了Executor执行器，如果配置了的话，会将invokeListener(listener, event）封装成一个线程可执行任务，然后给执行器执行。

   这样的话，我们可以自定义一个ApplicationEventMulticaster：

   ```java
   //需要注意的是，我们的名字一定要叫applicationEventMulticaster，否则不生效，回头看下源码就明白了
   @Component(&quot;applicationEventMulticaster&quot;)
   public class MyMulticaster extends SimpleApplicationEventMulticaster {
   
       @Override
       protected Executor getTaskExecutor() {
           // 这里可以按实际需求进行线程池的配置
           return new SimpleAsyncTaskExecutor();
       }
   
       /**
        * 这个方法主要配置错误处理，默认只会打一个警告级别的日志
        *
        * @return
        */
       @Override
       protected ErrorHandler getErrorHandler() {
           return (t -&gt; {
               System.out.println(&quot;出错了&quot;);
           });
       }
   }
   ```

   通过上面的配置后，我们所有的publish-event都会通过异步的机制去执行，但是我们想一个问题，有时候我们不想异步呢？一部分我们想同步，一部分想异步，这个时候怎么做呢？我们可以通过下面这种方式：

   - 配置类上加注解`@EnableAsync`

   - ```java
     // 执行的方法上加@Async注解
     @EventListener
     @Async
     public void on(MySecondEvent event) throws Exception {
       
     }
     ```

   这样通过aop的方式，会将我们的方法采用异步的方式进行执行。我们没有配置线程池的时候可能会报错：

   ```
   org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type 'org.springframework.core.task.TaskExecutor' available
   ```

   不过我们方法还是会异步执行，这主要是下面这段代码：

   ```java
   protected Executor getDefaultExecutor(@Nullable BeanFactory beanFactory) {
      if (beanFactory != null) {
         try {
         	// 在这里会抛出一个NoSuchBeanDefinitionException
            return beanFactory.getBean(TaskExecutor.class);
         }
         catch (NoUniqueBeanDefinitionException ex) {
            logger.debug("Could not find unique TaskExecutor bean", ex);
            try {
               return beanFactory.getBean(DEFAULT_TASK_EXECUTOR_BEAN_NAME, Executor.class);
            }
            catch (NoSuchBeanDefinitionException ex2) {
               if (logger.isInfoEnabled()) {
                  logger.info("More than one TaskExecutor bean found within the context, and none is named " +
                        "'taskExecutor'. Mark one of them as primary or name it 'taskExecutor' (possibly " +
                        "as an alias) in order to use it for async processing: " + ex.getBeanNamesFound());
               }
            }
         }
          // 这里被捕获
         catch (NoSuchBeanDefinitionException ex) {
            logger.debug("Could not find default TaskExecutor bean", ex);
            try {
              // 我们也没有定义一个名字是DEFAULT_TASK_EXECUTOR_BEAN_NAME = "taskExecutor"
                // 的线程池
               return beanFactory.getBean(DEFAULT_TASK_EXECUTOR_BEAN_NAME, Executor.class);
            }
            catch (NoSuchBeanDefinitionException ex2) {
               logger.info("No task executor bean found for async processing: " +
                     "no bean of type TaskExecutor and no bean named 'taskExecutor' either");
            }
            // Giving up -> either using local default executor or none at all...
         }
      }
       // 最后会返回一个null
      return null;
   }
   ```

   ```java
   @Override
   @Nullable
   // 最后会返回一个SimpleAsyncTaskExecutor
   protected Executor getDefaultExecutor(@Nullable BeanFactory beanFactory) {
      Executor defaultExecutor = super.getDefaultExecutor(beanFactory);
      return (defaultExecutor != null ? defaultExecutor : new SimpleAsyncTaskExecutor());
   }
   ```

   