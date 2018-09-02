---
title: 'Spring Cloud: Spring Cloud Feign'
date: 2018-08-02 23:05:22
tags:
categories: Spring Cloud 
---
## Feign声明式服务调用

## Quick Start
Feign整合了Ribbon和Hystrix，还提供了声明式的web客户端定义方式。
``` java 
1.pom
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
2.main添加
    @EnableFeignClients
    @EnableDiscoveryClient
3.controller
    @RestController
    public class HelloController {
        @Autowired
        private HelloService helloService;
        @RequestMapping(value = "/fegin-consumer",method = RequestMethod.GET)
        public String helloConsumer(){
            return helloService.hello();
        }
    }
4.service
    @FeignClient("hello-service")
    public interface HelloService {
        @RequestMapping("/hello")
        String hello();
    }
5.properties
    spring.application.name=feign-consumer
    server.port=9001
    eureka.client.serviceUrl.defaultZone=http://localhost:8765/eureka/
```
可以看到consumer没有通过restTemplate，直接同服务名@FeignClient(“hello-service”)通过http://localhost:9001/feign-consumer ,通过接口的方式访问了hello-service服务提供者，这是一个不带参数的REST服务绑定。
## 不同的参数绑定方法
实际业务接口复杂得多，HTTP各个位置需要传入不同类型的参数，返回也可能是一个复杂的对象。首先，我们扩展一下客户端的provider
``` java 
1.添加User pojo，包含name，age，及空构造！
2.改造controller接口
@RestController
public class HelloController {
    @RequestMapping(value = "/hello")
    public String hello(){
        return "Hello Eureka";
    }
    @RequestMapping(value = "/hello1/{name}",method = RequestMethod.GET)
    public String hello(@PathVariable String name){
        return "Hello Eureka "+name;
    }
    @RequestMapping(value = "/hello2",method = RequestMethod.GET)
    public User hello(@RequestHeader String name, @RequestHeader Integer age){
        return new User(name,age);
    }
    @RequestMapping(value = "/hello3",method = RequestMethod.GET)
    public String hello(@RequestBody User user){
        return "hello"+user.getName()+","+user.getAge();
    }
}
``` 
开始对feign consumer绑定请求，首先添加User类。feign采用的类似mvc的语法，但不是完全一样，比如MVC中，注解会根据参数名来作为默认值，但是Feign中绑定的参数必须通过value属性来指明参数名。
``` java  
1.改造service
    @FeignClient(“hello-server”)
    public interface HelloService {
    @RequestMapping(“/hello”)
    String hello();

    @RequestMapping(value = "/hello1/{name}",method = RequestMethod.GET)
    String hello(@PathVariable("name") String name);

    @RequestMapping(value = "/hello2",method = RequestMethod.GET)
    User hello(@RequestHeader("name") String name,@RequestHeader("age") Integer age);

    @RequestMapping(value = "/hello3",method = RequestMethod.POST)
    String hello(@RequestBody User user);
}
2.改造controller
@RestController
public class HelloController {
    @Autowired
    private HelloService helloService;
    @RequestMapping(value = "/fegin-consumer",method = RequestMethod.GET)
    public String helloConsumer(){
        return helloService.hello();
    }
    @RequestMapping(value = "/fegin-consumer2",method = RequestMethod.GET)
    public String helloConsumer2(){
        StringBuilder sb=new StringBuilder();
        sb.append(helloService.hello()).append("\n");
        sb.append(helloService.hello("CG")).append("\n");
        sb.append(helloService.hello("CG",26)).append("\n");
        sb.append(helloService.hello(new User("lsj",24))).append("\n");
        return sb.toString();
    }
}
```
测试！
启动注册中心，启动provider，启动fegin/。
## 如何在Feign中使用Ribbon
全局设置

    ribbon.ConnectTimeout=500
    ribbon.ReadTimeout=5000

## 如何在Feign中使用Hystrix
默认情况下会将所有方法封装到Hystrix中进行保护

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