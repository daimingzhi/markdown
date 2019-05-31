Spring Cloud学习 之 Spring Cloud Hystrix（使用详解）

[TOC]

##### 创建请求命令：

​				Hystrix命令就是我们之前说的HystrixCommand，它用来封装具体的依赖服务调用逻辑。

我们可以通过继承的方式来实现，比如：

```java
public class CommandHelloWorld extends HystrixCommand<String> {

    private final String name;

    public CommandHelloWorld(String name) {
        super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"));
        this.name = name;
    }

    @Override
    protected String run() {

        return "Hello " + name + "!";
    }
}
```

​			通过上面实现的CommandHelloWorld，我们即可以实现请求的同步执行，也可以实现异步执行。

同步执行：new CommandHelloWorld().execute();

异步执行：new CommandHelloWorld().queue()；异步执行的时候，可以通过返回的futureUser调用get方法来获取结果

测试案例：

```java
/**
 * 同步测试
 * @param args
 */
public static void main(String[] args) {
    CommandHelloWorld commandHelloWorld = new CommandHelloWorld("张三");
    String execute = commandHelloWorld.execute();
    System.out.println(execute);
}
/**
 * 测试结果：Hello 张三!
 */
```

```java
public class CommandHelloWorld extends HystrixCommand<String> {

    private final String name;

    private final CountDownLatch countDownLatch;

    public CommandHelloWorld(String name, CountDownLatch countDownLatch) {
        super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"));
        this.name = name;
        this.countDownLatch = countDownLatch;
    }

    @Override
    protected String run() throws Exception {
        System.out.println("开始执行run");
        System.out.println("run执行完成");
        countDownLatch.countDown();
        return "Hello " + name + "!";
    }
    /**
     * 异步测试
     *
     * @param args
     */
    public static void main(String[] args) throws Exception {
        CountDownLatch countDownLatch = new CountDownLatch(2);
        CommandHelloWorld commandHelloWorld = new CommandHelloWorld("张三", countDownLatch);
        Future<String> queue = commandHelloWorld.queue();
        countDownLatch.countDown();
        System.out.println("等待任务执行完成");
        countDownLatch.await();
    }
    /**
     * 测试结果
     * 等待任务执行完成
     * 1
     * 开始执行run
     * run执行完成
     */
```

​				另外，也可以通过@HystrixCommand注解更为优雅的实现Hystrix命令的定义，比如：

```java
public class AnnotationCommand {
    @HystrixCommand
    public String testAnnotation(String name) {
        return "hello,annotation" + name;
    }
}
```

​				虽然@HystrixCommand注解可以非常优雅的定义Hystri命令的实现，但是如上定义的方法执行同步执行的实现，若要实现异步执行则还需另外定义，比如：

```java
public class AnnotationCommand {
    /**
     * 同步
     *
     * @param name
     * @return
     */
    @HystrixCommand
    public String testAnnotation(String name) {
        return "hello,annotation" + name;
    }

    /**
     * 异步
     *
     * @param name
     * @return
     */
    @HystrixCommand
    public Future<String> tesAsyntAnnotation(String name) {
        return new AsyncResult<String>() {
            @Override
            public String invoke() {
                return "hello,annotation" + name;
            }
        };
    }
}

```

​				observe()和toObservable()虽然都返回了Observable,但是他们略有不同，前者返回的是一个Hot Observable,该命令会在observe()调用的时候立即执行，当Observable每次被订阅的时候会重放它的行为；而后者返回的是一个Cold Observable，toObservable()执行之后，命令不会被立即执行，只有当所有订阅者都订阅它之后才会执行。

​				虽然HystrixCommand具备了observe()和toObservable()的功能，但是它的实现有一定的局限性，它返回的Observable只能发射一次数据，所以Hystrix还提供了另外一个特殊命令封装HystrixObservableCommand,通过实现的命令可以获取能发射多次的Observable。

##### 定义服务降级：

​				fallback是Hystrix命令执行失败时使用的后备方法，用来实现服务的降级处理逻辑。在HystrixCommand中可以通过重载getFallback()方法来实现服务降级逻辑，Hystrix会在run()执行过程中出现错误，超时，线程池拒绝，断路器熔断等情况时，执行getFallback()方法内的逻辑，比如我们可以用如下方式实现服务降级逻辑：

```
public class Fallback extends HystrixCommand {
    @Override
    protected Object run() throws Exception {
        throw new RuntimeException("出错");
    }

    protected Fallback(HystrixCommandGroupKey group) {
        super(group);
    }

    @Override
    protected Object getFallback() {
        System.out.println("服务降级");
        return "error";
    }

    public static void main(String[] args) {
        Fallback fallback = new Fallback(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"));
        Object execute = fallback.execute();
    }
}
```

```
// 执行结果
java.lang.RuntimeException: 出错
	at com.study.cloud.hystrix.fallback.Fallback.run(Fallback.java:13)
	at com.netflix.hystrix.HystrixCommand$2.call(HystrixCommand.java:302)
	at com.netflix.hystrix.HystrixCommand$2.call(HystrixCommand.java:298)
	at rx.internal.operators.OnSubscribeDefer.call(OnSubscribeDefer.java:46)
	at rx.internal.operators.OnSubscribeDefer.call(OnSubscribeDefer.java:35)
	at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:48)
	at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:30)
	at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:48)
	at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:30)
	at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:48)
	at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:30)
	at rx.Observable.unsafeSubscribe(Observable.java:10327)
	at rx.internal.operators.OnSubscribeDefer.call(OnSubscribeDefer.java:51)
	at rx.internal.operators.OnSubscribeDefer.call(OnSubscribeDefer.java:35)
	at rx.Observable.unsafeSubscribe(Observable.java:10327)
	at rx.internal.operators.OnSubscribeDoOnEach.call(OnSubscribeDoOnEach.java:41)
	at rx.internal.operators.OnSubscribeDoOnEach.call(OnSubscribeDoOnEach.java:30)
	at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:48)
	at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:30)
	at rx.Observable.unsafeSubscribe(Observable.java:10327)
	at rx.internal.operators.OperatorSubscribeOn$SubscribeOnSubscriber.call(OperatorSubscribeOn.java:100)
	at com.netflix.hystrix.strategy.concurrency.HystrixContexSchedulerAction$1.call(HystrixContexSchedulerAction.java:56)
	at com.netflix.hystrix.strategy.concurrency.HystrixContexSchedulerAction$1.call(HystrixContexSchedulerAction.java:47)
	at com.netflix.hystrix.strategy.concurrency.HystrixContexSchedulerAction.call(HystrixContexSchedulerAction.java:69)
	at rx.internal.schedulers.ScheduledAction.run(ScheduledAction.java:55)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
服务降级
```

​				在HystrixObservableCommand实现的Hystrix命令中，我们可以通过重载resumeWithFallback方法来实现服务降级逻辑。该方法会返回一个Observable对象，当命令执行失败的时候，Hystrix会将Observable中的结果通过给所有的订阅者。

​				若要通过注解实现服务降级，只需要使用@HystrixCommand中的fallbackMethod参数来指定具体的服务降级实现方法。在使用注解来定义服务降级逻辑时，我们需要将具体的Hystrix命令与fallback实现函数定义在同一个类中，并且fallbackMthod的值必须与实现fallback方法的名字相同。且必须定义在一个类中，所以对于fallback的访问修饰符没有特定的要求，定义为private,protected,public均可。

​				在实际使用时，我们需要为大多数执行过程中可能会失败的Hystrix命令实现服务降级逻辑，但是也有一些情况可以不去实现降级逻辑，如下所示：

- **执行写操作的命令**：当Hystrix命令是用来执行写操作而不是返回一些信息的时候，通常情况下这类操作的返回类型是void或是为空的时候，我们通常只需要调用通知者即可

- **执行批处理或离线计算的命令**：当Hystrix命令是用来执行批处理程序生成一份报告或是进行任何类型的离线计算时，那么通常这些操作只需要将错误传播个给调用者然后让调用者稍后重试而不是发送给调用者一个静默的降级处理响应。

  ​		不论Hystrix命令是否实现了服务降级，命令状态和断路器状态都会更新，并且我们可以由此了解到命令执行的失败情况。

##### 异常处理：

###### 	异常传播：

​	在HystrixComman实现的run()方法中抛出异常时，除了HystrixBadRequestException之外，其他异常均会被Hystrix认为命令执行失败并触发服务降级的处理逻辑，所以当需要在命令执行中抛出不触发服务降级的异常时来使用它。

