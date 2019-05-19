Spring Cloud学习 之 Spring Cloud Hystrix（流程分析）

Spring Boot版本：2.1.4.RELEASE

Spring Cloud版本：Greenwich.SR1

我们还是从流程图入手：

![hystirx流程](H:\markdown\images\hystirx流程.png)

1. 创建HystrixCommand或者HystrixObservableCommand对象，在上篇文章我们已经讲过了，这里主要用的是命令模式

2. 命令执行

   从图中我们可以看到一共有4种命令的执行方式，而Hystrix在执行时会根据创建的Command对象以及具体的情况来选择一个执行，其中HystrixCommand实现了下面的执行方式

   - execute():同步执行，从依赖的服务返回一个单一的结果对象，或是在发生错误的时候抛出异常
   - queue():异步执行，直接返回一个Future对象，其中包含了服务执行结束时要返回的单一结果对象

   而HystrixObservableCommand实现了另外两种执行方式

   - observe():返回Observable对象，它代表了操作的多个结果，它是一个Hot Observable
   - toObserveable():同样会返回Observable对象，也代表了操作的多个结果，但它返回的是一个Cold Observable

   RxJava在上篇文章中也做了简单的介绍，这里也不多讲了。

​				在这里我们对于事件源observable提到了两个不同的概念：Hot Observable和Cold Observable,分别对应了上面command.observe()和command.toObserveable()的返回对象。其中Hot Observable，它不论“事件源”是否有“订阅者”，都会在创建后对事件进行发布，所以对于Hot Observable的每一个订阅者都有可能是从事件源的中途开始的，并可能只是看到了整个操作的局部过程。而Cold Observable在没有订阅者的时候并不会发布事件，而是进行等待，直到有订阅者之后才发布事件，所以对于Cold Observable的订阅者。它可以保证从一开始看到整个操作的全部过程。

​				大家从表面上可能只会认为只是在HystrixObservableCommand中使用了RxJava,然后实际上execute(),queue()也都使用了RxJava来实现，从下面的源码中，我们可以看到。

```java
public R execute() {
    try {
        // 是通过queue()返回的异步对象Future<R>的get方法来实现同步执行的
        // 改方法会等待任务执行结束，然后获得R类型的结果进行返回
        return queue().get();
    } catch (Exception e) {
        throw Exceptions.sneakyThrow(decomposeException(e));
    }
}
```

```java
 public Future<R> queue() {
     // queue（）则是通过toObservable()方法来获得一个Cold Observable,并且通过toBlocking()将
     // Observable转换成BlockingObservable,它可以把数据以阻塞的方式发射出来，而toFuture方法
     // 则是把BlockingObservable转换为一个Future，该方法只是创建一个Future返回，并不会阻塞，这使得
     // 消费者可以自行决定如何处理异步操作，而execute方法是通过queue()返回的异步对象Future<R>的get方		// 法来实现同步执行的，通过这种方式转换的future要求Observable只发射一个数据，所以这两个实现都只能
     // 返回单一结果
     final Future<R> delegate = toObservable().toBlocking().toFuture();
   
     final Future<R> f = new Future<R>() {

         @Override
         public boolean cancel(boolean mayInterruptIfRunning) {
             if (delegate.isCancelled()) {
                 return false;
             }

             if (HystrixCommand.this.getProperties().executionIsolationThreadInterruptOnFutureCancel().get()) {
                 /*
                  * The only valid transition here is false -> true. If there are two futures, say f1 and f2, created by this command
                  * (which is super-weird, but has never been prohibited), and calls to f1.cancel(true) and to f2.cancel(false) are
                  * issued by different threads, it's unclear about what value would be used by the time mayInterruptOnCancel is checked.
                  * The most consistent way to deal with this scenario is to say that if *any* cancellation is invoked with interruption,
                  * than that interruption request cannot be taken back.
                  */
                 interruptOnFutureCancel.compareAndSet(false, mayInterruptIfRunning);
          }

             final boolean res = delegate.cancel(interruptOnFutureCancel.get());

             if (!isExecutionComplete() && interruptOnFutureCancel.get()) {
                 final Thread t = executionThread.get();
                 if (t != null && !t.equals(Thread.currentThread())) {
                     t.interrupt();
                 }
             }

             return res;
}

         @Override
         public boolean isCancelled() {
             return delegate.isCancelled();
}

         @Override
         public boolean isDone() {
             return delegate.isDone();
}

         @Override
         public R get() throws InterruptedException, ExecutionException {
             return delegate.get();
         }

         @Override
         public R get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException {
             return delegate.get(timeout, unit);
         }
       
     };

     /* special handling of error states that throw immediately */
     if (f.isDone()) {
         try {
             f.get();
             return f;
         } catch (Exception e) {
             Throwable t = decomposeException(e);
             if (t instanceof HystrixBadRequestException) {
                 return f;
             } else if (t instanceof HystrixRuntimeException) {
                 HystrixRuntimeException hre = (HystrixRuntimeException) t;
                 switch (hre.getFailureType()) {
      case COMMAND_EXCEPTION:
      case TIMEOUT:
         // we don't throw these types from queue() only from queue().get() as they are execution errors
         return f;
      default:
         // these are errors we throw from queue() as they as rejection type errors
         throw hre;
      }
             } else {
                 throw Exceptions.sneakyThrow(t);
             }
         }
     }

     return f;
 }
```

3. 若当前命令的请求缓存功能被启用的，并且该命令缓存命中，那么缓存的结果会立即以Observable对象的形式返回

4. 断路器是否打开

   在命令结果没有缓存命中的时候，Hystrix在执行命令前需要检查断路器是否为打开状态：

   - 如果断路器是打开的，那么Hystrix不会执行命令，而是转接到fallback处理逻辑
   - 如果断路器是关闭的，那么Hystrix跳转到第五步，检查是否有可用资源来执行命令

5. 线程池/请求队列/信号量是否占满

   ​	如果与命令相关的线程池和请求队列，或者信号量已经被占满，那么Hystrix也不会执行命令，而是转接到fallback处理逻辑

   ​	需要注意的是，这里Hystrix所判断的线程池并非容器的线程池，而是每个依赖服务的专有线程池。Hystrix为了保证不会因为某个依赖服务的问题影响到其他依赖服务而采用了“舱壁模式”来隔离每个依赖的服务

6. HystrixObservableCommand.construct()或HystrixCommand.run()

   Hystrix会根据我们编写的方法来决定采取什么样的方式去请求依赖服务。

   - HystrixCommand.run()：返回一个单一结果，或者抛出异常

   - HystrixObservableCommand.construct()：返回一个Observable对象来发射多个结果，或通过onError发送错误通知

     ​	如果run()或construct()方法的执行事件超过了命令设置的超时阀值，当前处理线程将会抛出一个TimeOutException。在这种情况下，Hystrix会转接到fallback处理逻辑。同时，如果当前命令没有被取消或中断，那么它最终会忽略run()或者construct()方法的返回

     ​	如果命令没有抛出异常并返回了结果，那么Hystrix在记录一些日志并采集监控报告之后将结果返回。在使用run()的情况下，Hystrix会返回一个Observable,它发射单个结果并产生onCompleted的结束通知：而在使用construct()的情况下，Hystrix会直接返回该方法产生的Observable对象

7. 计算断路器的健康度

   ​		Hystrix会将成功，失败，拒绝，超时等信息报告给断路器，而断路器会维护一组计数器来统计这些数据。

   ​		断路器会使用这些统计数据来决定是否要将断路器打开，来对某个依赖服务的请求进行熔断，直到恢复期结束。若在恢复期结束后，根据统计数据判断如果还是未达到健康指标，就再次熔断

8. fallback的处理

   ​	当命令执行失败的时候，Hystrix会进入fallback尝试回退处理，我们通常也称该操作为**服务降级**。而能引起服务降级的有下面几种情况

   - 第四步：当前命令处于“熔断”状态，断路器是打开的时候
   - 第五步：当前命令的线程池，请求队列或者信号量被占满的时候
   - 第六步：HystrixObservableCommand.construct()或HystrixCommand.run()抛出异常的时候

   在服务降级逻辑中，我们需要实现一个通用的响应结果，并且该结果的处理逻辑应当是从缓存或者根据一些静态逻辑来获取，而不是依赖网络请求实现。如果一定要在降级逻辑中包含网络请求，那么该请求也必须被包装在HystrixObservableCommand或HystrixCommand中，从而实现级联的降级策略，而最终的降级逻辑一定不是一个依赖网络请求的处理，而是一个能够稳定地返回结果的处理逻辑

   在HystrixObservableCommand跟HystrixCommand实现降级逻辑时还略有不同

   - 当使用HystrixCommand的时候，通过实现HystrixCommand.getFallback来实现服务降级逻辑
   - 当使用HystrixObservableCommand的时候，通过HystrixObservableCommand.resumeWithFallback()实现服务降级逻辑，该方法会返回一个Observable对象来发射一个或多个降级结果

   当命令的降级逻辑返回结果之后，Hystrix就将该结果返回给调用者。当使用HystrixCommand.getFallback的时候，它返回一个Observable对象，该对象会发射HystrixCommand.getFallback的处理结果，而使用HystrixObservableCommand.resumeWithFallback()的时候，它会直接将Observable对象返回

   如果我们没有为命令实现降级逻辑或者降级处理了逻辑中抛出了异常，Hystrix依然会返回一个Observable对象，但是它不会发射任何结果数据，而是通过onError方法通知命令立即中断请求，并通过onError方法将引起命令失败的异常发送给调用者。实现一个有可能失败的降级逻辑是一种非常糟糕的做法，我们应该在实现降级策略时尽可能避免失败的情况

   当然完全不可能出现失败的完美策略是不存在的，如果降级执行发现失败的时候，Hystrix会根据不同的执行方法做出不同的处理。

   - execute()：抛出异常
   - queue()：正常返回future对象，但是当调用get方法时会抛出异常
   - observe()：正常返回Observable对象，当订阅它的时候，将立即通过调用订阅者的onError方法来通知中止请求
   - toObservable()：正常返回Observable对象，当订阅它的时候，将立即通过调用订阅者的onError方法来通知中止请求

9. 返回成功的响应

   ​	当Hystrix命令执行成功之后，他会将处理结果直接返回或是以Observable的形式返回。而具体以哪种方式返回取决于之前第二步我们所提到的对命令的4中不同的执行方式，下图总结了这4中调用方式之间的依赖关系。我们可以将此图与在第二步中对前两者的源码分析联系起来，并且从源头toObservable()来开始分析

![Hystrix返回成功](H:\markdown\images\Hystrix返回成功.png)

- toObservable()：返回最原始的Observable，必须通过订阅它才会真正触发命令的执行流程
- observe()：在toObservable()产生原始Observable之后立即订阅它，让命令能够马上开始异步执行，并返回一个Observable对象，当调用它的subscribe时，将重新产生结果和通知给订阅者
- queue()：将toObservable()产生的原始Observable通过toBlocking()方法转换成BlockingObservable对象，并调用它的toFuture()方法返回异步的Future对象
- execute()：在queue()产生异步结果Future对象之后，通过调用get()方法阻塞并等待结果的返回