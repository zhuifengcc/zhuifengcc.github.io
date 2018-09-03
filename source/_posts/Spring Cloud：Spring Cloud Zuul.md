---
title: 'Spring Cloud: Spring Cloud Zuul'
date: 2018-08-07 23:13:58
tags:
categories: Spring Cloud 
---
## Zuul网关
内部service通过服务注册发现相互调用，还有对外的OpenService，提供对外RESTful API服务，通过Nginx等进行负载均衡后提供给外部客户端。

![基础架构图](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Zuul/3-1.png)

一、但是为了让路由正确分发，实例增减或ip变动等发生的情况，需要手动同步维护，系统规模大是，维护及其不便。  
二、对外服务会有权限校验等机制，如用户登陆等，为了防止客户端发起请求时被篡改等，还会有签名校验机制。微服务将原本单一的应用拆成了多个应用，我们不得不在每个应用中去实现这样一套逻辑，这些校验代码非常冗余。一旦修改逻辑，需要将所有的应用上的逻辑进行修改。

所有我们需要API网关，所有外部客户端访问都需要经过它来进行过滤调度！
## 概要
Zuul通过服务注册中心注册到Eureka，从而获取到服务实例。Zuul会默认通过以服务名作为ContextPath方式来创建映射。

对于签名校验，登陆等问题，完全可以做成独立的服务，从应用中剥离，在API网关服务上进行统一的调用来对服务接口做前置过滤。可以通过ZUul创建校验过滤器，指定哪些规则需要执行校验逻辑，通过校验才会被路由到具体的微服务接口，使得微服务更加专注于业务逻辑开发。
## Quick Start
我们来新建一个Zuul。

1.pom
``` java 
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

<groupId>com.demo</groupId>
<artifactId>api-gateway</artifactId>
<version>0.0.1-SNAPSHOT</version>
<packaging>jar</packaging>

<name>api-gateway</name>
<description>Demo project for Spring Boot</description>

<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.0.RELEASE</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>

<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <java.version>1.8</java.version>
    <spring-cloud.version>Finchley.M9</spring-cloud.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka</artifactId>
        <version>1.4.4.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-netflix-eureka-server</artifactId>
        <version>2.0.0.M6</version>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>

<repositories>
    <repository>
        <id>spring-milestones</id>
        <name>Spring Milestones</name>
        <url>https://repo.spring.io/milestone</url>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </repository>
</repositories>
</project>
```
2.主类增加@EnableZuulProxy
``` java
@EnableZuulProxy
@SpringCloudApplication
public class ApiGatewayApplication {

	public static void main(String[] args) {
		SpringApplication.run(ApiGatewayApplication.class, args);
	}
}
```
3.配置

    spring.application.name=api-gateway
    server.port=5555
    #服务实例映射
    zuul.routes.api-a-url.path=/api-a/**
    #传统路由方式，维护成本高
    #zuul.routes.api-a-url.url=http://localhost:8080/ 
    #面向服务的路由方式
    zuul.routes.api-a.serviceId=HELLO-SERVER
    zuul.routes.api-b-url.path=/api-b/**
    zuul.routes.api-b.serviceId=FEIGN-CONSUMER
    #zuul注册到eureka
    eureka.client.serviceUrl.defaultZone=http://localhost:1111/eureka

然后我们启动eureka注册中心、hello-server、feign-consumer、还有api-gateway
已经配置了路由规则。我们访问api的http://localhost:5555/api-b/feign-consumer .api网关吧请求转发到相应的服务实例上去，只需配置简单的path和serviceId映射组合即可。

ps: 若出现不能正常路由到相应服务的情况，考虑是否超时，添加配置

    zuul.host.socket-timeout-millis=60000
    zuul.host.connect-timeout-millis=10000
    hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=60000

## 请求过滤
实现请求路由的基本功能后，微服务可以通过统一的API网关作为入口进行访问。但是访问权限等没有限制，任何请求都会被转发，最简单粗暴的是在所有微服务上添加一套权限逻辑，但是不可取。我们需要剥离，形成一个独立的鉴权服务。而Zuul的一个核心功能就是：请求过滤，只需要继承ZuulFilter抽象类并实现4个抽象函数即可完成请求的拦截与过滤。来看一个简单的Zuul过滤器
``` java
public class AccessFilter extends ZuulFilter{
    private static Logger log=LoggerFactory.getLogger(AccessFilter.class);
    //过滤器类型，决定过滤器在什么生命周期中执行，pre代表在请求路由之前进行。
    @Override
    public String filterType(){
        return "pre";
    }
    //过滤器执行顺序
    @Override
    public int filterOrder(){
        return 0;
    }
    //过滤器是否要被执行，可以制定该函数的过滤器有效范围
    @Override
    public boolean shouldFilter(){
        return true;
    }
    //过滤器逻辑类型，context.setSendZuulResponse(false)命令Zuul过滤该请求，
    //context.setResponseStatusCode(401)返回错误代码
    @Override
    public Object run(){
        RequestContext context = RequestContext.getCurrentContext();
        HttpServletRequest requset = context.getRequset();
        log.info("send {} request to {}", requset.getMethod(), requset.getRequsetURL().toString());

        Obeject accessToken = requset.getParamter("accessToken");
        if(accessToken == null){
            log.warn("token empty!!");
            context.setSendZuulResponse(false);
            context.setResponseStatusCode(401);
            return null;
        }
        log.info("access token ok!");
        return null;
    }
}
```
实现过滤器后，不会直接生效，需要创建具体的bean才能启动
在主类中增加如下方法才会生效
``` java
@Bean
public AccessFilter accessFilter（）{
    return new AccessFilter();
}
```
我们测试

    http://localhost:5555/api-a/index
    http://localhost:5555/api-a/index&accessToken=token

## 路由详解
### 路由的一些基本配置
回顾一下路由配置

    zuul.routes.api-a-url.path=/api-a/**
    zuul.routes.api-a.serviceId=HELLO-SERVER
    zuul.routes.api-b-url.path=/api-b/**
    zuul.routes.api-b.serviceId=FEIGN-CONSUMER

除了path与serviceId的映射外，还有更简洁的配置

    #zuul.routes.<serviceId>=<path>
    zuul.routes.user-service=/user-service/**
    #将zuul看作eureka下的一个服务，他会获取所有实例清单，自己就得到了serviceID与服务实例地址的映射，api网关可自动找到最佳匹配

虽然eureka与zuul省去了维护服务配置的工作，但是实际情况一般如下

    zuul.routes.fegin-consumer.path=/fegin-consumer/**
    zuul.routes.fegin-consumer.serviceId=fegin-consumer

zuul的默认规则可以完全不需要我们这样一个一个配置。  
当我们为API网关引入eureka后，每个服务都会创建一个默认的路由规则，path使用serviceId配置的服务名作为请求前缀。  
由于默认情况eureka的所有服务都会被zuul自动创建映射，而有些我们不希望暴露出去，可以用

    zuul.ignore-serices=*
    zuul.routes.hello-server.serviceId=hello-server

如上设置一个服务名匹配表达式来定义不自动映射，只需要添加我们想要暴露的服务hello-server，而fegin-consumer则不会暴露出去
### 自定义路由规则
构建微服务时，为了兼容不同外部版本，一般会采用开闭原则进行设计开发。我们可以根据服务版本表示来知道，如下
userservice-v1、userservice-v2、orderservice-v1、orderservice-v2 

这样，路由默认会映射成/userservice-v1、/userservice-v2
但是这样不利于通过规则进行管理，可以改成如下这样

    /v1/userservice
    //PatternServiceRouteMapper添加到主类下，通过正则表达式定义路由映射

``` java
@Bean
public PatternServiceRouteMapper serviceRouteMapper(){
    return new PatternServiceRouteMapper(
        "(?<name>^.+)-(?<version>v.+$)",
        "${version}/${name}"
    );
}
```
### 路径匹配
可以通过如下来看

    /user-service/?     匹配一个/user-service/a、/user-service/d、/user-service/3
    /user-service/*     匹配任意/user-service/afdsfs
    /user-service/**    匹配任意及多级目录 /user-service/afdsfs、/user-service/afdsfs/zz

## Hystrix和Ribbon支持
Zuul的依赖包含了Hystrix和Ribbon的依赖，所以Zuul拥有线程隔离和断路器的自我保护功能，以及对服务调用的客户端负载均衡。  
ps: 使用path与url的映射关系来配置路由规则的时候，路由转发的请求不会采用HystrixCommand包装，故不含有上述所说的Hystrix和Ribbon的特性，故尽量使用path和serviceId的组合来配置
## Zuul过滤器！
zuul包含对请求路由及过滤两个功能，路由将外部请求转发到微服务实例，过滤将请求进行校验，实现请求校验服务聚合等功能

spring cloud zuul包含4个基本特性实际上ZuulFilter中4个抽象方法：
### 过滤类型 String filterType()
需要返回一个字符串代表过滤器类型，Zuul默认了4个不同的生命周期过滤类型。

    pre：路由请求前调用
    routing：请求时调用
    post：routing和error过滤器之后调用
    error：处理请求时发生错误被调用   

filterType4种过滤器类型定义了一个外部HTTP到达API网关，直到返回请求结果的全部生命周期。可以通过下图清晰的看出来流转过程
![过滤器流转图](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Zuul/3-2.png)

这其中为了让API网关可以更方便使用，默认实现类一批核心的过滤器。定义在springcloud core下的netflix.zuul.filters。

![过滤器下包结构](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Zuul/3-3.png)
### 顺序执行 int filterOrder()
    用值越小优先级越高
### 执行条件 boolean shouldFilter()
    是否执行该过滤器
### 具体操作 Object run()
    自定义过滤逻辑，是否要拦截当前请求
### 过滤器生命周期表

![过滤器生命周期表](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Zuul/3-4.png)
## Zuul异常处理
### error阶段直接抛出异常
可以看到，核心过滤器并没有实现error阶段，自己实现一个异常过滤器来看看怎么处理

1.创建一个pre类型过滤器，在过滤器中抛出一个异常。在api-gateway服务中建立filter包
``` java
@Component
public class ThrowExceptionFilter extends ZuulFilter {
    private static Logger log= LoggerFactory.getLogger(ThrowExceptionFilter.class);
    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        log.info("pre filter, throw RuntimeException");
        doSomething();
        return null;
    }
    private void doSomething(){
        throw new RuntimeException("errors");
    }
}
```
我们运行注册中心、hello-server、api网关，利用http://localhost:5555/api-a/hello 访问，可以看到报错，api服务控制台输出了错误.

    Whitelabel Error Page
    This application has no explicit mapping for /error, so you are seeing this as a fallback.

    Mon Apr 16 15:53:24 CST 2018
    There was an unexpected error (type=Internal Server Error, status=500).
    pre:ThrowExceptionFilter

我们看一下RibbonRoutingFilter下的run如何抛出异常
``` java
public Object run() {
        RequestContext context = RequestContext.getCurrentContext();
        this.helper.addIgnoredHeaders(new String[0]);

        try {
            RibbonCommandContext commandContext = this.buildCommandContext(context);
            ClientHttpResponse response = this.forward(commandContext);
            this.setResponse(response);
            return response;
        } catch (ZuulException var4) {
            throw new ZuulRuntimeException(var4);
        } catch (Exception var5) {
            throw new ZuulRuntimeException(var5);
        }
    }
```
我们相应的改写下
``` java
public Object run() {
        log.info("即将抛出错误");
        try{
            doSomething();
        } catch (Exception e) {
            throw new ZuulRuntimeException(e);
        }
        return null;
    }
```
可以看到

    Whitelabel Error Page
    This application has no explicit mapping for /error, so you are seeing this as a fallback.

    Mon Apr 16 15:56:20 CST 2018
    There was an unexpected error (type=Internal Server Error, status=500).
    MY errors!!!!!!
### 禁用过滤器
无论是核心过滤器还是自定义的过滤器，只要在Api网关中创建了实例，那么默认情况都是启用状态的。如果不想使用了，该如何禁用过滤器

1.可以重写sholdFliter逻辑返回false，但是这样会修改代码重新编译

2.特定参数过滤

    zuul.<SimpleClassName>.<filterType>.disable=true

其中是过滤器名称过滤器类型。如

    zuul.AccessFilter.pre.disable=true

不仅可以过滤自定义的，核心过滤器也可以过滤。可以将Springcloud的过滤器全部抛弃并实现一套自己的过滤器处理机制。

## Zuul动态加载路由及过滤器
API网关可是对外提供服务的入口，7X24小时服务系统，不可能重启及关闭应用，因此必须具备动态更新内部逻辑的能力，比如动态添加删除过滤器、动态修改路由规则
## 动态路由
只需要利用config的动态刷新机制，从Git获取配置，轻松实现路由规则的动态刷新，

启动eureka，启动配置中心config，config注册到eureka并从Git中获取配置，接下来需要Zuul网关，网关从config获取配置
1首先我们在Git上发布我们的配置api-gateway.properties

    zuul.routes.service-a.path=/service-a/**
    zuul.routes.service-a.serviceId=hello-server
    zuul.routes.service-b.path=/service-b/**
    zuul.routes.service-b.url=http://www.baidu.com  

2建立api-gateway-dynamic-route，从config-server中获取配置

3pom中引入zuul、eureka、config依赖

4bootstrap.properties配置

    spring.application.name=api-gateway
    server.port=5556
    spring.cloud.config.uri=http://localhost:7001/
    eureka.client.serviceUrl.defaultZone=http://localhost:1111/eureka/

5主类添加一个动态刷新RefreshScope
``` java
@EnableZuulProxy
@SpringBootApplication
public class ApiGatewayDynamicRouteApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayDynamicRouteApplication.class, args);
    }
    @RefreshScope
    @ConfigurationProperties("zuul")
    public ZuulProperties zuulProperties(){
        return new ZuulProperties();
    }
}
```
6启动api-gateway 还有helloserver

我们通过访问api-gateway
http://localhost:5556/service-a/hello 跳转到了hello server的服务

http://localhost:5556/service-b 跳转到了百度
路由实现
### 动态过滤器
过滤请求的动态加载也可以通过类似的方式实现,但有所不同，路由规则是配置，请求过滤是编码实现。
所以对于请求过滤去的动态加载，需要借助基于JVM实现的动态语言才行，比如Groovy
