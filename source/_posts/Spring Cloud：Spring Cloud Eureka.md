---
title: 'Spring Cloud：Spring Cloud Eureka'
date: 2018-05-29 15:31:15
tags:
categories: Spring Cloud 
---

## 服务治理
服务治理是微服务架构中最为核心和基础的模块，主要用来实现各个微服务实例的自动化注册与发现。

服务注册:服务治理框架中，通常会构建一个注册中心，每个服务单位向注册中心登记自己提供的服务，将主机与端口号、版本号、通讯协议等一些附加信息告知注册中心，注册中心按照服务名分类组织服务清单。服务注册中心会以心跳的方式去监测清单中的服务是否可用，剔除不可用的服务，达到排除故障服务的效果。
服务发现：调用方需要向服务注册中心咨询服务，并获取所有服务的实例清单，以实现对具体服务实例的访问。
## Netflix Eureka
Spring Cloud Eureka,使用Netflix Eureka来实现服务注册与发现，它既包含了服务端组件，又包含了客户端组件。

Eureka服务端，我们也成为服务注册中心。支持高可用配置，依托于强一致性提供良好的服务实例可用性，可以应对多种不同的故障场景。如Eureka集群，分片故障，Eureka进入自我保护，允许在故障时提供服务的注册与发现，不同可用区域的服务注册中心通过异步模式互相复制各自的状态。

Eureka客户端，处理服务的注册与发现。客户端通过注解和参数配置的方式，嵌入在客户端应用程序的代码中，Eureka客户端向注册中心注册自身提供的服务并周期地发送心跳来更新它的服务续约。同时，它也能从服务端查询当前注册的服务信息并把它们缓存到本地并周期性地刷新服务状态。

我们以一个具体的实例，演示Eureka构建服务注册中心，以及服务的注册：
## 服务注册中心搭建
新建一个名为"eureka-server"的Spring Boot项目，我们使用IDEA直接构建，Cloud Discovery-Eureka Server，它会为我们引入必要的依赖
![搭建Eureka Server](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Eureka/1-1.png)

通过在主类上添加@EnableEurekaServer注解一个服务注册中心提供给其他应用进行对话
``` java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}
}
```
默认情况下，服务注册中心也会将自己作为客户端尝试注册自己，我们需要禁用他的客户端注册行为，当然在配置文件application.properties中我们也可以配置服务注册中心的端口，以及defaultZone相关信息
``` 
server.port=1111
eureka.instance.hostname=localhost
#向注册中心注册自己
eureka.client.register-with-eureka=false
#检索服务
eureka.client.fetch-registry=false
eureka.client.serviceUrl.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka/
```
启动项目，访问http://localhost:1111/ ,便可以看到Eureka的信息面板，可以看到Instances currently registered with Eureka为空，说明此时未注册任何服务实例。
![Spring Eureka](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Eureka/1-2.png)
## 服务提供者注册
我们新建一个Spring Boot项目，此时选择Cloud Discovery-Eureka Discovery
![搭建Discovery Client](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Eureka/1-3.png)
通过在主类上添加@EnableDiscoveryClient注解一个服务注册中心提供给其他应用进行对话
``` java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;

@EnableDiscoveryClient
@SpringBootApplication
public class HelloServiceApplication {

	public static void main(String[] args) {
		SpringApplication.run(HelloServiceApplication.class, args);
	}
}
```
我们需要在配置文件中，命名服务，以及通过eureka.client.serviceUrl.defaultZone来指定服务注册中心的地址
``` 
spring.application.name=hello-service
eureka.client.serviceUrl.defaultZone=http://localhost:1111/eureka/
```
这里我们新建一个Controller：
``` java
package com.example.demo;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @Author: zhuifengcc
 * @Description:
 * @Date: 2018/5/29
 */
@RestController
public class HelloController {
    @RequestMapping("/index")
    public String index(){
        return "Hello";
    }
}
```
此时在启动注册中心的基础上启动项目，访问http://localhost:1111/ 可以看到服务实例此时已经有了，因为为注定服务的端口，访问http://localhost:8080/index 可以看到输出的hello
## 高可用注册中心
在Eureka服务治理设计中，所有节点既是服务提供方，也是服务消费方，服务注册中心也不例外。

Eureka Server的高可用实际上就是讲自己作为服务向其他服务注册中心注册自己，这样就可以形成一组互相注册的服务注册中心，以实现服务清单的互相同步，达到高可用的效果。（服务注册中心集群）

我们本地启动两个注册中心，修改如下配置使得注册中心之间互相发现，实现分片：
```
spring.application.name=eureka-server
server.port=1111
eureka.instance.hostname=peer1
eureka.client.serviceUrl.defaultZone=http://peer2:1112/eureka/ 
```
```
spring.application.name=eureka-server
server.port=1112
eureka.instance.hostname=peer2
eureka.client.serviceUrl.defaultZone=http://peer1:1111/eureka/
```
此时取消了单注册中心节点中注册及检索服务配置

我们在/etc/hosts添加对peer1和peer2的转换，让上面配置host形式的serviceUrl能在本地正确访问：
```
127.0.0.1 peer1
127.0.0.1 peer2
```

启动发现此时可以在eureka的分片信息中信息看到另外的注册中心，并且在下方Instances currently registered with Eureka中看到命名的EUREKA-SERVER的2个可用实例。
我们修改下之前服务提供者的配置 
```
spring.application.name=hello-service
eureka.client.serviceUrl.defaultZone=http://peer1:1111/eureka/,http://peer2:1112/eureka/
```
启动服务提供者后，可以在两个注册中心服务实例列表下都可以看到hello-service

![注册中心分片peer1](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Eureka/1-4.png)

peer2下方可以看到可用分片为peer1：

![注册中心分片peer2](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Eureka/1-5.png)

不想使用主机名来定义注册中心的地址，可以在配置文件中配置属性eureka.instance.prefer-ip-address=true

ps：jar 启动方式 

    java -jar eureka-server-1.0.0.jar --spring.profiles.active=peer1
    java -jar eureka-server-1.0.0.jar --spring.profiles.active=peer2

    #启动多个不同端口的服务实例
    java -jar hello-service-0.0.1-SNAPSHOT.jar --server.port=8081
    java -jar hello-service-0.0.1-SNAPSHOT.jar --server.port=8082


## 服务发现与消费
服务消费者：发现服务及消费服务。服务发现的任务由Eureka 客户端完成，服务消费的任务由Ribbon（客户端负载均衡）完成。

我们在服务注册中心和服务提供者基础上，构建一个服务消费者。新建一个DiscoveryClient项目，由于使用Ribbon去实现服务消费，pom引入spring-cloud-starter-ribbon依赖
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-ribbon</artifactId>
</dependency>
```

在主类上同理通过@EnableDiscoveryClient注解让该应用注册为Eureka客户端应用，同时我们在主类中创建RestTemple的Spring Bean实例，通过@LoadBalanced注解开启客户端负载。
``` java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

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
新建ConsumerController:
``` java
package com.example.demo;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;


/**
 * @Author: zhuifengcc
 * @Description:
 * @Date: 2018/5/29
 */
@RestController
public class ConsumerController {
    @Autowired
    RestTemplate restTemplate;

    @RequestMapping(value = "/ribbon-consumer", method = RequestMethod.GET)
    public String index(){
        return restTemplate.getForEntity("http://HELLO-SERVICE/index",String.class).getBody();
    }

}
```
配置文件：
```
spring.application.name=ribbon-consumer
server.port=9000
eureka.client.serviceUrl.defaultZone=http://localhost:1111/eureka/
```
服务消费者启动后，在Eureka信息面板可以看到新注册的服务消费者

![服务消费者](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Eureka/1-6.png)

访问http://localhost:9000/ribbon-consumer 可以看到成功返回之前的服务提供者hello输出信息

![消费服务](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Eureka/1-7.png)
<!--hello -jar的方式完成多服务实例的启动-->
## 详解Eureka
三个核心角色：服务注册中心、服务提供者以及服务消费者。实际上服务提供者多数时候也是服务消费者。

    服务注册中心：Eureka提供的服务端，提供服务注册与发现的功能
    
    服务提供者：提供服务的应用，因为Spring Cloud基于Spring Boot，可以使用Spring Boot应用，也可以使用其他技术平台且遵循Eureka通讯机制的应用。

    服务消费者：消费者应用从服务注册中心获取服务列表，从而使消费者可以知道去何处调用其所需要的服务。如Ribbon实现消费，Feign实现消费
治理机制：

    服务提供者：服务注册（发送REST请求，自带自身的一些元数据信息，服务注册中心收到后会把这些元数据信息存储在一个双层Map中（第一层key为服务名，第二层的key为具体服务的实例名））- 服务同步（服务注册中心集群）- 服务续约（Renew服务提供者维护心跳）

    服务消费者：获取服务（REST请求）- 服务调用（Ribbon轮询）

    服务下线：服务正常关闭操作时，发送Rest请求给Eureka Server，服务端接收后，将服务状态置为下线（Down）并传播下线事件

    服务注册中心：实效剔除、自我保护


