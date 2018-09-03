---
title: 'Spring Cloud: Spring Cloud Feign'
date: 2018-08-02 23:05:22
tags:
categories: Spring Cloud 
---
## Feign声明式服务调用
在实践过程中，我们发现对于Ribbon和Hystrix的使用几乎是同时出现的，是否有更深层次的封装简化开发呢？Feign整合了Ribbon和Hystrix，还提供了声明式的web客户端定义方式。
## Quick Start

1.pom
``` java 
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
```
2.main添加
``` java 
    @EnableFeignClients
    @EnableDiscoveryClient
```
3.controller
``` java
    @RestController
    public class HelloController {
        @Autowired
        private HelloService helloService;
        @RequestMapping(value = "/fegin-consumer", method = RequestMethod.GET)
        public String helloConsumer(){
            return helloService.index();
        }
    }
```
4.service
``` java
    @FeignClient("hello-service")
    public interface HelloService {
        @RequestMapping("/index")
        String index();
    }
```
5.properties
``` java
    spring.application.name=feign-consumer
    server.port=9001
    eureka.client.serviceUrl.defaultZone=http://localhost:1111/eureka/
```
可以看到consumer没有通过restTemplate，直接同服务名@FeignClient(“hello-service”)通过http://localhost:9001/feign-consumer ,通过接口的方式访问了hello-service服务提供者，这是一个不带参数的REST服务绑定。

![调用效果展示](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Feign/1-1.png)

## 不同的参数绑定方法
实际业务接口复杂得多，HTTP各个位置需要传入不同类型的参数，返回也可能是一个复杂的对象。首先，我们扩展一下客户端的服务提供者
1.添加User pojo，包含name，age，及空构造！

2.改造controller接口
``` java
@RestController
public class HelloController {
    @RequestMapping(value = "/hello")
    public String hello(){
        return "Hello Eureka";
    }
    @RequestMapping(value = "/hello1/{name}", method = RequestMethod.GET)
    public String hello(@PathVariable String name){
        return "Hello Eureka " + name;
    }
    @RequestMapping(value = "/hello2", method = RequestMethod.GET)
    public User hello(@RequestHeader String name, @RequestHeader Integer age){
        return new User(name, age);
    }
    @RequestMapping(value = "/hello3", method = RequestMethod.GET)
    public String hello(@RequestBody User user){
        return "hello" + user.getName() + "," + user.getAge();
    }
}
```

开始对feign consumer绑定请求，首先添加User类。feign采用的类似mvc的语法，但不是完全一样，比如MVC中，注解会根据参数名来作为默认值，但是Feign中绑定的参数必须通过value属性来指明参数名。不然会抛出IllegalStateException异常，value属性不能为空。

1.改造service

``` java
@FeignClient("hello-server")
public interface HelloService {
    @RequestMapping(“/hello”)
    String hello();

    @RequestMapping(value = "/hello1/{name}",method = RequestMethod.GET)
    String hello(@PathVariable("name") String name);

    @RequestMapping(value = "/hello2",method = RequestMethod.GET)
    User hello(@RequestHeader("name") String name, @RequestHeader("age") Integer age);
 
    @RequestMapping(value = "/hello3",method = RequestMethod.POST)
    String hello(@RequestBody User user);
}
```

2.改造controller

``` java
@RestController
public class HelloController {
    @Autowired
    private HelloService helloService;
    @RequestMapping(value = "/fegin-consumer", method = RequestMethod.GET)
    public String helloConsumer(){
        return helloService.hello();
    }
    @RequestMapping(value = "/fegin-consumer2", method = RequestMethod.GET)
    public String helloConsumer2(){
        StringBuilder sb = new StringBuilder();
        sb.append(helloService.hello()).append("\n");
        sb.append(helloService.hello("zhuifengcc")).append("\n");
        sb.append(helloService.hello("zhuifengcc",24)).append("\n");
        sb.append(helloService.hello(new User("qx",24))).append("\n");
        return sb.toString();
    }
}
```
测试！
启动注册中心，启动服务提供者，启动fegin。
## 如何在Feign中使用Ribbon
全局设置，如修改默认的客户端调用超时时间

    ribbon.ConnectTimeout=500
    ribbon.ReadTimeout=5000

指定服务配置

针对实际服务的特性进行调整，实际上通过@FeignClient声明Feign客户端的同时，也创建了一个Ribbon客户端，名为HELLO-SERVICE。可通过如下方式进行具体配置

    HELLO-SERVICE.ribbon.ConnectTimeout = 500
    HELLO-SERVICE.ribbon.ReadTimeout = 2000
    ...

重试机制

在Spring Cloud Feign中默认实现了请求的重试机制，可参照Ribbon章节。

ps: Ribbon的超时与Hystrix的超时是两个概念，一般需要Hystrix的超时时间大于Ribbon的超时时间，否则触发服务降级，负载的超时就无意义了！

## 如何在Feign中使用Hystrix
默认情况下会将所有方法封装到Hystrix中进行保护

全局设置，如修改全局超时时间

    Hystrix.command.default.execution.isolation.thread.timeoutInMillseconds = 5000

关闭Feign客户端的Hystrix支持

    feign.hystrix.enabled = false

## 服务降级配置
Feign下服务降级逻辑的实现只需要为Feign客户端的定义接口编写一个具体的接口实现类。
``` java 
@Component
public class HelloServiceFallback implements HelloService{
    @Override
    public String hello(){
        return "error";
    }

    @Override
    public String hello(@RequestParam("name") String name){
        return "error";
    }
    @Override
    public User hello(@RequestHeader("name") String name, @RequestHeader("age") Integer age){
        return new User("unknown", 0);
    }
    
    @Override
    public String Hello(@RequestBody User user){
        return "error";
    }
}
```
并在服务接口声明中添加fallback指向
``` java
@FeignClient(name="HELLO-SERVICE", fallback = HelloServiceFallback.class)
```

## Feign日志配置
``` java
logging.level.com.cg.service.HelloService=DEBUG

@EnableFeignClients
@EnableDiscoveryClient
@SpringBootApplication
public class FeignApplication {
    @Bean
    Logger.Level feginLoggerLevel(){
        return Logger.Level.FULL;
    }

    public static void main(String[] args) {
        SpringApplication.run(FeignApplication.class, args);
    }
}
```