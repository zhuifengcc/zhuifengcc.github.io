---
title: 'Spring Cloud: Spring Cloud Hystrix'
date: 2018-07-02 17:29:23
tags:
categories: Spring Cloud 
---
## 服务容错保护
在微服务架构中，我们将系统拆分成了很多服务单元，各单元的应用间通过服务注册与订阅的方式互相依赖，由于每个单元都在不同的进程中运行，依赖通过远程调用的方式执行，这样就有可能因为网络原因或者依赖服务自身问题出现调用故障或者延迟，会直接导致调用方的对外服务也出现延迟，若此时调用方的请求不断增加，最后就会因为等待出现故障方响应形成任务积压，最终导致自身服务的瘫痪。为了解决这样的问题，产生了断路器等一系列的服务保护机制。

在分布式架构中，当某个服务发生故障（类似用电器发生短路）之后，通过断路器的故障监控（类似熔断保险丝），向调用方返回一个错误响应，而不是长时间的等待。这样不会使得线程因故障服务被长时间占用不释放，避免了故障在分布式系统中的蔓延。

Spring Cloud Hystirx实现了断路器、线程隔离等一系列服务保护功能。基于Netflix的开源框架Hystirx实现的，该框架的目标在于通过控制那些访问远程系统、服务和第三方库的节点，从而对延迟和故障提供更强大的容错能力。Hystirx具备服务降级、服务熔断、线程和信号隔离、请求缓存、请求合并以及服务监控等强大功能。

## Quick Start
我们模拟一个服务调用关系：

![快速入门](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Hystrix/2-1.png)

若此时关闭其中一个服务实例，发送Get请求到消费者http://localhost:9000/ribbon-consumer 当轮询到不可用的服务实例的时候，不会输出正常的Hello

我们再看看引入Spring Cloud Hystrix之后的情形，在maven中引入Spring-cloud-starter-hystrix依赖：
``` java
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-ribbon</artifactId>
</dependency>
```

随后我们在consumer的主类上通过使用注解@EnableCircuitBreaker开启断路器功能：
``` java
@EnableCircuitBreaker
@EnableDiscoveryClient
@SpringBootApplication
public class DemoApplication {

	@Bean
	@LoadBalanced
	RestTemplate restTemplate(){
		return new RestTemplate();
	}

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}
}
```

ps:这里也可以直接使用注解@SpringCloudApplication，查看源码可以看到是包含@SpringBootApplication、@EnableDiscoveryClient、@EnableCircuitBreaker，so一个标准的SpringCloud应用应包含服务发现和断路器。

改造服务消费方式，在Service中新建方法调用：
``` java
@Service
public class HelloService {
    @Autowired
    RestTemplate restTemplate;

    @HystrixCommand(fallbackMethod = "helloFallBack")
    public String helloService(){
        return restTemplate.getForEntity("http://HELLO-SERVICE/hello",String.class).getBody();
    }

    public String helloFallBack(){
        return "error";
    }
}
```
引入依赖，使用注解@HystrixCommand进行服务降级：
``` java 
<dependency>
    <groupId>com.netflix.hystrix</groupId>
    <artifactId>hystrix-javanica</artifactId>
    <version>RELEASE</version>
</dependency>
```
改造controller：
``` java 
@RestController
public class ConsumerController {
    @Autowired
    private HelloService helloService;

    @RequestMapping(value = "/ribbon-consumer", method = RequestMethod.GET)
    public String HelloConsumer(){
        return helloService.helloService();
    }

}
```
重启服务消费者对服务进行消费，当我们设定的2个服务提供者实例正常时，访问http://localhost:9000/ribbon-consumer 可以看到正常的Hello输出信息。同理，关闭一个端口的服务实例（或者设定一定的超时时间），当轮询到不可用的服务时，我们可以看到我们处理的服务回滚生效，返回了输出的Error信息
![Hystrix做了服务回滚](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Hystrix/2-2.png)

ps：Hystrix默认的超时时间为2000ms

## 原理分析
Hystrix使用了命令模式和RxJava（RxJava的观察者-订阅者模式）

命令模式：将客户端的请求封装为一个对象，可使用不同请求对客户端进行参数化。实现行为请求者与行为实现者解耦。
简单说，原来是客户端直接调用业务逻辑方法，但是这两个方法有耦合。为了解耦，设计一个中间的命令方法，命令方法接受业务方法bean对象。客户端调用命令方法的excute，命令方法再执行业务代码。

eg:

1.接收者（操作者）：被调用的业务代码
``` java 
public class Reciver {
    public void action(){
        //真正的业务逻辑
    }
}
```

2.Command接口及实现—（通过command实现了解耦）
``` java 
public interface Command {
    void excute();
}
public class ConcreteCommand implements Command {
    private Reciver reciver;
    public ConcreteCommand(Reciver reciver){
        this.reciver=reciver;
    }
    @Override
    public void excute() {
        reciver.action();
    }
}
```
3.调用者Invoker:持有一个命令对象，并且可以在需要的时候通过命令对象完成具体的业务逻辑
``` java
public class Invoker {
    private Command command;

    public void setCommond(Command command) {
        this.command = command;
    }

    public void action(){
        command.excute();
    }
}
```
4.使用命令模式。可以诸多多个命令模式。比如新建文件、复制文件、删除文件，只需再需要时直接调用即可
``` java
public class Client{
    public static void main(String[] args){
        Reciever reciever = new Reciever();
        Command command = new ConceretCommand(reciever);
        Invoker invoker = new Invoker();
        invoker.serCommond(commond);
        invoker.action()
    }
}
```

RxJava的观察者-订阅者模式：

    a) Observable向订阅者Subsciber发布事件，Subsciber接受到后进行处理。事件通常就是对依赖服务的调用
    b) Observable可以发出多个事件，直到结束或者发生异常
    c) Observable每发送一个事件，就要调用Subsciber的onNext()方法
    d) 每一个Observable执行，最后一定会通过Subsciber的onCompleted或者onError结束操作流

``` java 
import rx.Observable;
import rx.Subscriber;
//事件源observable
Observable<String> observable = Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("Hello RxJava");
        subscriber.onNext("I am a programmer");
        subscriber.onCompleted();
    }
});
//订阅者Subsciber
Subscriber<String> subscriber=new Subscriber<String>() {
    @Override
    public void onCompleted() {

    }
    @Override
    public void onError(Throwable throwable) {

    }
    @Override
    public void onNext(String s) {
        System.out.println("Subscriber: " + s);
    }
};
//发布者触发事件发布
observable.subscribe(subscriber);
```
这里对相关东西做简单列举：
        
    HystrixCommand: 用在依赖的服务返回单个操作结果的时候
    HystrixObservableCommand: 用在依赖的服务返回多个服务结果的时候

执行方式：

    HystrixCommand：
        execute(): 同步执行，从依赖的服务返回一个单一的结果对象，或是在发生错误的时候抛出异常。
        queue(): 异步执行，直接返回一个Future对象，其中包含了服务执行结束时要返回的单一结果对象。
    HystrixObservableCommand：
        observe(): 返回Observable对象，它代表了操作的多个结果，它是一个Hot Obsevable（不论“事件源”是否有“订阅者”，都会在创建后对事件进行发布，所以对于它的每一个“订阅者”都是可能是从“事件源”的中途开始的，并可能只是看到了整个过程的局部）
        toObservavble():同样也会返回Observable对象，也代表了操作的多个结果，但塔返回的是一个Cold Obsevable（在没有“订阅者”的时候不会发布事件，而是进行等待，直到有“订阅者”之后才发布事件，可以保证从一开始看到整个操作的过程）

工作流程：

    1.创建HystrixCommand或HystrixObservableCommand对象（命令模式）

    2.命令执行（上述执行方式的执行）

    3.结果是否被缓存（命令缓存命中，结果立即以Observable对象形式返回）

    4.断路器是否打开（关闭，转5步；开启fallback）

    5.线程池/请求队列/信号量是否占满（资源占满，开启fallback）

    6.HystrixObservableCommand.run或HystrixCommand.run

    7.计算断路器的健康度

    8.fallback处理（4,5,6步会跳至）

    9.返回成功的响应

## 依赖隔离
同Docker线程隔离类似，Hystrix使用“舱壁”模式实现线程池的隔离，他会为每一个依赖服务创建一个独立的线程池，某个依赖服务延迟过高，不会拖慢其他的依赖服务

例如调用产品服务的Command放入A线程池, 调用账户服务的Command放入B线程池. 这样做的主要优点是运行环境被隔离开了. 这样就算调用服务的代码存在bug或者由于其他原因导致自己所在线程池被耗尽时, 不会对系统的其他服务造成影响. 但是带来的代价就是维护多个线程池会对系统带来额外的性能开销. 如果是对性能有严格要求而且确信自己调用服务的客户端代码不会出问题的话, 可以使用Hystrix的信号模式(Semaphores)来隔离资源。

## Hystrix详解
### 创建请求命令
我们也可以通过继承的方式实现对依赖服务的调用过程：
``` java
public class UserCommand extends HystrixCommand<User>{
    private RestTemplate restTemplate;
    private Long id;

    public UserCommand(Setter setter, RestTemplate restTemplate, Long id){
        super(setter);
        this.restTemplate = restTemplate;
        this.id = id;
    }

    @Override
    protected User run(){
        return restTemplate.getForObject("http://USER-SERVICE/user/{1}",User.class, id);
    }
}
```
同步执行：
``` java
User u = new UserCommand(restRemple, 1L).execute();
```
异步执行：
``` java
//返回的对象调用get方法获取结果
Future<User> futureUser = new UserCommand(restRemple, 1L).queue();
```
使用注解的方式：
``` java 
//同步的方式
public class UserService{
    @Autowired
    private RestTemplate restTemplate;
    @HystrixCommand
    public User getUserById(String uid){
        return restTemplate.getForObject("http://USER_SERVICE/users/{1}",User.class,id);
    }
}
//异步的方式
public class UserService{
    @Autowired
    private RestTemplate restTemplate;
    @HystrixCommand
    public Future<User> getUserById(String uid){
        return new AsyncResult<User>(){
            @Override
            public User invoke(){
                return restTemplate.getForObject("http://USER_SERVICE/users/{1}",User.class,id);
            }
        }
    }
}
```
### 定义服务降级
fallback是Hystrix命令执行失败时使用的后备方法，用来实现服务的降级处理逻辑。
代码实现方式：
``` java
public class UserCommand extends HystrixCommand<User>{
    private RestTemplate restTemplate;
    private Long id;

    public UserCommand(Setter setter, RestTemplate restTemplate, Long id){
        super(setter);
        this.restTemplate = restTemplate;
        this.id = id;
    }

    @Override
    protected User run(){
        return restTemplate.getForObject("http://USER-SERVICE/user/{1}",User.class, id);
    }

    @Override
    protected User getFallback(){
        reuturn new User();
    }
}
```
### 异常处理
异常传播：

在HystrixCommand实现的run方法抛出异常的时候，除了HystrixBadRequestException之外，其他异常均会被认为执行失败并触发服务降级

可使用以下的方式：
``` java 
@HystrixCommand(ignoreExceptions = {BadRequestException.class})
public User getUserById(Long id){
        return restTemplate.getForObject("http://USER-SERVICE/users/{1}", User.class, id);
    }
```
当方法抛出BadRequestException，相当于被ignore，此异常会被包装在HystrixBadRequestException中抛出，不会出发服务降级。

异常获取：

当Hystrix进入服务降级之后，需要针对不同的异常进行针对性的处理，如何获取当前抛出的异常？

传统继承方式中，可以用getFallback()方法通过Throwable getExecutionException()方法来获取具体异常

注解方式中，在fallback实现方法的参数中增加Throwable e 对象的定义
``` java
@HystrixCommand(fallbackMethod = "fallback")
User getUserById(String id){
    throw new RuntimeException("getUserById command failed")
}
User fallback(String id, Throwable e){
    assert "getUserById command failed".equals(e.getMessage());
}
```
### 线程池划分
Hystrix命令默认的线程池划分是根据命令组来实现的。Hystrix可以设置相应的命令名称、命令组及HystrixTheadPoolKey来对线程池进行更灵活的划分
``` java
@HystrixCommand(commandKey = "getUserById", groupKey = "UserGroup", theadPoolKey = "getUserByIdThread")
public User getUserById(Long id){
    return restTemplate.getForObject("http://USER-SERVICE/users/{1}", User.class, id);
}
```
### 请求缓存
系统用户不断增长时，每个微服务需要承受的并发压力也越来越大，进程间的服务请求调用会有一部分的性能损失。类似数据库的缓存保护也可以运用到依赖服务的调用上

### 请求合并
高并发下远程调用的通信消耗会很大，依赖服务的线程池资源有限，将出现排队等待和和相应延迟。
引入请求合并HystrixCollapser可以用来实现请求的合并，以减少通信消耗和线程占用。

HystrixCollapser在HystrixCommand之前放置一个合并处理器，对同一依赖服务的多个请求进行整合并以批量方式发起请求
假设微服务提供两个获取User的接口

    /users/{id}
    /users?ids={ids}

消费端调用Service内有两个方法
``` java
public User find(String id){
    return restTemplate.getForObject("http://USER-SERVICE/user/{1}",User.class,id);
}
public List<User> findAll(List<String> ids){
    return restTemplate.getForObject("http://USER-SERVICE/user/{1}",User.class,StringUtils.join(ids,","));
}
```
我们要来实现将多个单一User对象的请求命令合并
``` java
@Service
public class ConsumerService {
    @Autowired
    RestTemplate restTemplate;

    //一个用于请求/users/{id}，一个用于请求/users?ids={ids}
    //在/users/{id}上通过 @HystrixCollapser创建合并请求器，合并窗口时间100毫秒，超过100毫秒无法合并完成则不合并
    @HystrixCollapser(batchMethod="findAll",collapserProperties={
        @HystrixProperty(name="timerDelayInMilliseconds",value="100")
    })
    public User find(String id){
        return null;
    }
    @HystrixCommand
    public List<User> findAll(List<String> ids){
        return restTemplate.getForObject("http://USER-SERVICE/user/{1}",User.class,StringUtils.join(ids,","));
    }
}
```