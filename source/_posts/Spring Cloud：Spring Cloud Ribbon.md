---
title: 'Spring Cloud：Spring Cloud Ribbon'
date: 2018-06-18 12:54:39
tags:
---
## 客户端负载均衡
Spring Cloud Ribbon是一个基于HTTP和TCP的客户端负载均衡，他基于Netflix Ribbon实现。通过Spring Cloud的封装，可以轻松地将面向服务的REST模板请求自动转换成客户端负载均衡的服务调用。

我们都知道，负载均衡是对系统的高可用、网络压力的缓解和处理能力扩容的重要手段之一。传统的服务端负载均衡的模块（硬件设备或者是软件模块）都会为维护一个下挂可用的服务端清单，通过心跳检测来剔除故障的服务端节点，当客户端发送请求到负载均衡设备的时候，该设备按某种算法（线性轮询、按权重负载、按流量负载等）从维护的服务端清单中取出一台服务端的地址，然后进行转发。

客户端负载均衡和服务端负载均衡最大的不同在于服务清单所存储的位置。在客户端负载均衡中，所有客户端节点都维护着自己要访问的服务端清单，而这些服务端清单来自服务注册中心，在客户端负载均衡中也需要心跳去维护服务端清单的健康性，只是这个步骤需要与服务注册中心配合完成。

如何实现客户端负载均衡？

    · 服务注册者只需要启动多个服务实例并注册到一个注册中心或是多个相关联的服务注册中心。
    · 服务消费者直接通过调用被@LoadBalanced注解修饰过的RestTemple来实现面向服务的接口调用。

## RestTemple不同请求类型调用实现

### GET请求
可通过两个方法进行调用实现

1、getForEntity(返回ResponseEntity对象，是Spring对HTTP请求响应的封装)

提供3个重载，具体可查询相关文档

```java
RestTemplate restTemplate = new RestTemplate();
ResponseEntity<String> responseEntity = restTemplate.getForEntity("http://USER_SERVICE/user?name={1}", String.class, "zhuifengcc")
String body = responseEntity.getBody();

// 对象User
RestTemplate restTemplate = new RestTemplate();
ResponseEntity<User> responseEntity = restTemplate.getForEntity("http://USER_SERVICE/user?name={1}", User.class, "zhuifengcc")
User userBody = responseEntity.getBody();
```
2、getForObject(对getForEntity的进一步封装，通过HttpMessageConvererExtractor对HTTP的请求响应体body内容进行对象转换，实现请求直接返回包装好的对象内容)

```java
RestTemplate restTemplate = new RestTemplate();
String result = restTemplate.getForObject(uri, String.class);

// body是一个对象时
RestTemplate restTemplate = new RestTemplate();
User result = restTemplate.getForObject(uri, User.class);
```

### POST请求
可通过三个方法进行调用实现

1、getForEntity

```java
RestTemplate restTemplate = new RestTemplate();
User user = new User("zhuifengcc", 24);
// 提交的请求body是user对象，返回的响应body类型是String
ResponseEntity<User> responseEntity = restTemplate.getForEntity("http://USER_SERVICE/user", user, String.class)
String body = responseEntity.getBody();
```
2、getForObject

```java
RestTemplate restTemplate = new RestTemplate();
User user = new User("zhuifengcc", 24);
String postResult = restTemplate.getForObject("http://USER_SERVICE/user", user, String.class)
```
这两者可同GET请求的发放进行类比

3、postForLocation(实现了以POST请求提交资源，并返回新资源的URI)

```java
User user = new User("zhuifengcc", 24);
URI responseURI = restTemplate.postForLocation("http://USER_SERVICE/user", user, String.class)
```
由于返回新资源的URI，就相当于指定了返回类型。

### PUT请求
可通过PUT方法进行调用实现

```java
RestTemplate restTemplate = new RestTemplate();
long id = 1001L;
User user = new User("zhuifengcc", 24);
restTemplate.put("http://USER_SERVICE/user/{1}", user, id);
```

### DELETE请求
可通过DELETE方法进行调用实现

```java
RestTemplate restTemplate = new RestTemplate();
long id = 1001L;
restTemplate.delete("http://USER_SERVICE/user/{1}", id);
```
## Ribbon配置
Ribbon中定义的每一个接口都有多种不同的策略实现，同时这些接口之间又有一定的依赖关系，是的上手困难，不知道如何选择具体的实现策略及如何组织他们的关系，Spring Cloud Ribbon采用了自动化配置，引入Spring Cloud Ribbon的依赖之后，就能自动化构建这些接口的实现。

通过自动化配置的实现，可以轻松实现客户端负载均衡。同时，针对一些个性化需求，我们也可以方便地替换默认实现，只需在Spring Boot应用中创建对应的实现实例就能覆盖这些默认的实现。

## 重试机制

集成Spring Eureka时，由于Spring Eureka实现了CAP原理中的AP（满足可用性，分区容错性（可靠性），弱一致性），与Zookeeper实现CP（强一致性，可靠性）不同，Eureka为了实现更高的服务可用性，在极端情况下宁愿接受故障实例也不要丢掉"健康"实例。当服务注册中心故障时，Eureka在超过85%的实例丢失心跳时触发保护机制，注册中心将会保留此时所有节点，以实现服务间依然可以进行互相调用的场景，即使其中有部分故障节点，但是也可以继续保障大多数服务正常消费。

当触发了保护机制或者是服务剔除的延迟，引起服务调用到故障实例的时候，我们还是希望增强这类问题的容错，所以加入一些重试机制。Spring Cloud整合了Spring Retry来增强RestTemple的重试能力

    #开启重试机制
    spring.cloud.loadbanlancer.retry.enabled=true
    #断路器超时时间（需要大于Ribbon超时时间不然不会触发重试）
    hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=10000
    #请求连接的超时时间
    hello-service.ribbon.ConnectTimeout=250
    #请求处理的超时时间。
    hello-service.ribbon.ReadTimeout=1000
    #对所有操作请求都进行重试。
    hello-service.ribbon.OkToRetryOnAllOperations=true
    #切换实例的重试次数。
    hello-service.ribbon.MaxAutoRetriesNextServer=2
    #对当前实例的重试次数。
    hello-service.ribbon.MaxAutoRetries=1