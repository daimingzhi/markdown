Spring Cloud学习 之 Spring Cloud Hystrix（基础知识铺垫）

Spring Boot版本：2.1.4.RELEASE

Spring Cloud版本：Greenwich.SR1

[TOC]

##### 前述：

​			在微服务架构中，我们将系统拆分成了很多服务单元，各单元的应用间通过服务注册与订阅的方式互相依赖。由于每个单元都在不同的进行中运行，依赖通过远程调用的方式执行，这样就有可能因为网络原因或是依赖服务自身的问题出现调用故障或延迟，而这些问题会直接导致调用方的对外服务也出现延迟，若此时调用方的请求不断增加，最后就会因为等待出现故障的依赖方响应形成任务积压，最终导致自身服务的瘫痪。

​			举一个例子，在一个电商网站中，我们可能会将系统分成用户，订单，库存，积分，评论等一系列的服务单元。用户创建一个订单的时候，客户端将调用订单服务的创建订单接口，此时创建订单接口又会向库存服务来请求出货。此时若库存服务因自身处理逻辑等原因造成响应缓慢，会直接导致创建订单服务的线程被挂起，以等待库存申请服务的响应，在漫长的等待之后用户会因为请求库存失败而得到创建订单失败的结果，如果在高并发的情况下，因这些挂起的线程在等待库存服务的响应而未能释放，使得后续到来的创建订单请求被阻塞，最终导致订单服务也不可用。

​			在微服务架构中，存在那么多的服务单元，若一个单元出现故障，就很容易因依赖关系而引发故障的蔓延，最终导致整个系统的瘫痪，这样的架构相较传统架构更加的不稳定，为了解决这样的问题，产生了断路器等一系列的服务保护机制。

​			当某个服务发生故障后，通过断路器的故障监控，向调用方返回一个错误响应，而不是长时间的等待。这样就不会使得线程因调用故障服务被长时间占用不释放，避免了故障在分布式系统中的蔓延。

​			针对上述问题，Spring Cloud Hystrix实现了断路器，线程隔离等一系列服务保护功能。它也是基于Netflix的开源框架Hystrix实现的，该框架的目标在于通过控制那些访问远程系统，服务和第三方库的节点，从而对延迟和故障提供更强大的容错能力。Hystrix具备**服务降级，服务熔断，线程和信号隔离，请求缓存，请求合并以及服务监控**等强大功能。

##### 快速入门：

1. 如何使用hystrix:

   ```xml
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
   </dependency>
   ```

2. 使用示例：

   ```java
   @RestController
   @RequestMapping("/consumer")
   @RequiredArgsConstructor
   public class Controller {
   
       private final RestTemplate ribbonRestTemplate;
   
       @GetMapping("/get")
       @HystrixCommand(fallbackMethod = "fallback")
       public String consume() {
           ResponseEntity<String> responseEntity = ribbonRestTemplate
                   .getForEntity("http://EUREKA-CLIENT/client/hello?who=小王",
                           String.class);
           System.out.println(responseEntity.getBody());
           return responseEntity.getBody();
       }
   	// 当调用失败时（服务器发生异常）或者超出断路器最大等待时间，会执行这个回调方法
       public String fallback() {
           return "error";
       }
   }
   ```

##### 命令模式：

先看下Nystrix官网的流程图，我们根据图中标志的顺序来解析每个环节

![](<https://raw.githubusercontent.com/wiki/Netflix/Hystrix/images/hystrix-command-flow-chart.png>)

1. 首先，创建一个HystrixCommand或者HystrixObservableCommand对象

   首先，创建一个HystrixCommand或者HystrixObservableCommand对象用来表示对依赖服务的操作请求，同时传递所有需要的参数。从其命名我们就知道它是采用了”命令模式“来实现对服务调用操作的封装。而这两个command对象分别针对不同的应用场景

   - HystrixCommand：用在依赖的服务返回单个操作结果的时候
   - HystrixObservableCommand：用在依赖的服务返回多个操作结果的时候

   > 什么是命令模式？
   >
   > 将来自客户端的请求封装成一个对象，从而可以让你使用不同的请求对客户端进行参数化。它可以被用于实现“行为请求者”与“行为实现者”的解耦，以便使两者可以适应变化。
   >
   > 我们需要知道：
   >
   > 1. 命令模式是通过命令发送者和命令执行者的解耦来完成对命令的具体控制的。
   >
   > 2. 命令模式是对功能方法的抽象，并不是对对象的抽象。
   >
   > 3. 命令模式是将功能提升到对象来操作，以便对多个功能进行一系列的处理以及封装。

经典的命令模式包括4个角色：

1. Command：定义命令的统一接口

2. ConcreteCommand：Command接口的实现者，用来执行具体的命令，某些情况下可以直接用来充当Receiver。

3. Receiver：行为实现者

4. Invoker：行为请求者，是命令模式中最重要的角色。这个角色用来对各个命令进行控制。

先看下示例代码，我们再来分析这个设计模式：

```java
/**
 * 定义命令的统一接口
 *
 * @author dmz
 * @date Create in 16:18 2019/5/18
 */
public interface Command {
    /**
     * 定义命令的抽象执行策略
     */
    void execute();
}

/**
 * Command接口的实现者，用来执行具体的命令，某些情况下可以直接用来充当Receiver。
 *
 * @author dmz
 * @date Create in 16:18 2019/5/18
 */
public class ConcreteCommand implements Command {

    private Receiver receiver;

    public ConcreteCommand(Receiver receiver) {
        this.receiver = receiver;
    }

    @Override
    public void execute() {
        receiver.action();
    }
}

/**
 * 行为请求者
 *
 * @author dmz
 * @date Create in 16:37 2019/5/18
 */
public class Invoker {
    /**
     * 行为请求者要持有需要请求的命令
     */
    private Command command;

    public void setCommand(Command command) {
        this.command = command;
    }

    /**
     * 行为请求者发起命令
     */
    public void invoke() {
        System.out.println("发起命令");
        command.execute();
    }
}

/**
 * 行为实现者
 *
 * @author dmz
 * @date Create in 16:22 2019/5/18
 */
public class Receiver {
    public void action() {
        System.out.println("执行业务逻辑");
    }
}

/**
 * @author dmz
 * @date Create in 16:38 2019/5/18
 */
public class Main {
    public static void main(String[] args) {
        Command concreteCommand = new ConcreteCommand(new Receiver());
        Invoker invoker = new Invoker();
        invoker.setCommand(concreteCommand);
        invoker.invoke();
    }
}
```

​				从上面的示例中，我们可以看到Invoker跟Receiver通过Command接口实现了解耦。对于调用者来说，我们可以为其注入多个命令操作，比如新建文件，复制文件，删除文件这样三个操作，调用者只需要在需要的时候直接调用即可，而不需要知道这些操作命令实际是如何实现的。

​				关于命令模式，我们一直要知道它设计的初衷是什么，否则就会很混乱，每一个设计模式的出现都是为了解决特定情况下的某些问题，而不是凭空想象出来的，我们要能在合适的场景取采用合适的模式才算真正理解了这种设计模式。命令模式的初衷就是为了实现:**对命令请求者（Invoker）和命令实现者（Receiver）的解耦，方便对命令进行各种控制。**

在下面这些情况应考虑使用命令模式：

1. 使用命令模式作为“回调（CallBack）”在面向对象系统中的替代，“CallBack”讲的便是先将一个函数登记上，然后在以后调用此函数。
2. 需要在不同的时间指定请求，将请求排队。一个命令对象和原先的请求发出者可以有不同的生命期。换言之，原生的请求发出者可能已经不在了，而命令对象本身仍然是活动的。这时命令的接收者可以是在本地，也可以是在网络的另外一个地址上。命令对象可以在序列化后传送到另外一台机器上去
3. 系统需要支持命令的撤销。命令对象可以把状态存储起来，等到客户端需要撤销命令所产生的效果时，可以调用undo()方法，把命令对象所产生的效果撤销掉。命令对象还可以提供redo()方法，以供客户端在需要时再重新实现命令效果。
4. 如果要将系统中所有的数据更新到日志里，以便在系统奔溃时，可以根据数据读回所有的数据更新命令，重新调用Execute()方法一条条执行这些命令，从而恢复系统在奔溃前所作的数据更新

##### RxJava:

​			在Hystrix的底层实现中，大量的使用了RxJava，为了更容易的理解后续内容，在这里对RxJava的观察者-订阅者模式做一个简单的入门介绍。

RxJava 有以下三个基本的元素：

1. 被观察者（Observable）：

   - 用来向订阅者Subscriber对象发布事件，Subscriber对象则在接受到对象后对其进行处理，而在这里指的事件通常就是对依赖服务的调用。

   - 一个被观察者可以发出多个事件，知道结束或者发生异常。

   - Observable对象每发出一个事件，就会调用对应观察者subscribe对象的onNext()方法
   - 每一个Observable的执行，最后一定会通过调用subscribe.onCompleted()或者subscribe.onError()来结束该事件的操作流

2. 观察者（Observer）：

3. 订阅（subscribe）

