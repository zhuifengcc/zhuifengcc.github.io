---
title: 'Spring Cloud: Spring Cloud Hystrix'
date: 2018-07-02 17:29:23
tags:
---
## 服务容错保护
在微服务架构中，我们将系统拆分成了很多服务单元，各单元的应用间通过服务注册与订阅的方式互相依赖，由于每个单元都在不同的进程中运行，依赖通过远程调用的方式执行，这样就有可能因为网络原因或者依赖服务自身问题出现调用故障或者延迟，会直接导致调用方的对外服务也出现延迟，若此时调用方的请求不断增加，最后就会因为等待出现故障方响应形成任务积压，最终导致自身服务的瘫痪。为了解决这样的问题，产生了断路器等一系列的服务保护机制。

在分布式架构中，当某个服务发生故障（类似用电器发生短路）之后，通过断路器的故障监控（类似熔断保险丝），向调用方返回一个错误响应，而不是长时间的等待。这样不会使得线程因故障服务被长时间占用不释放，避免了故障在分布式系统中的蔓延。

Spring Cloud Hystirx实现了断路器、线程隔离等一系列服务保护功能。基于Netflix的开源框架Hystirx实现的，该框架的目标在于通过控制那些访问远程系统、服务和第三方库的节点，从而对延迟和故障提供更强大的容错能力。Hystirx具备服务降级、服务熔断、线程和信号隔离、请求缓存、请求合并以及服务监控等强大功能。

## Quick Start
我们模拟一个服务调用关系：

![快速入门](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Hystrix/2-1.png)

若此时关闭其中一个服务实例，发送Get请求到消费者http://localhost:9000/ribbon-consumer 当轮询到不可用的服务实例的时候，可以看到输出如下：

![未配置断路器](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Hystrix/2-2.png)

我们再看看引入Spring Cloud Hystrix之后的情形，在maven中引入Spring-cloud-starter-hystrix依赖，随后我们在consumer的主类上通过使用注解@EnableCircuitBreaker开启断路器功能

![未配置断路器](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Hystrix/2-3.png)

ps：这里也可以直接使用注解@SpringCloudApplication，查看源码可以看到是包含@SpringBootApplication、@EnableDiscoveryClient、@EnableCircuitBreaker，so一个标准的SpringCloud应用应包含服务发现和断路器。

在Service中新建方法调用
