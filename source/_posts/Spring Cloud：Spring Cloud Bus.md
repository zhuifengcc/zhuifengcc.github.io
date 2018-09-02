---
title: 'Spring Cloud: Spring Cloud Bus'
date: 2018-08-20 23:55:52
tags:
categories: Spring Cloud 
---
基于rabbit和kafka。一个是stream作为服务之间的通信MQ，一个是bus主要用来关于广播配置的。

Bus是一个轻量级的消息代理，不同于stream作为服务与服务间的通信。Bus让所有服务连接上来，该主题的消息会被所有实例监听消费，所以我们叫他消息总线。

消息总线上的各个实例可以方便地广播一些需要让其他连接都知道得消息，比如配置信息或者一些操作管理，通常用来广播配置，实现动态刷新配置。
## RabbitMQ实现消息总线
一、整合spring cloud bus动态刷新配置

1）准备：config-repo、config-server-eureka启动他们

2）改造config-client-eureka，为其增加amqp

pom增加amqp和actuator（提供刷新端点）； 配置中增加RabbitMQ的连接和用户信息

    spring.rabbitmq.host=localhost
    spring.rabbitmq.port=5672
    spring.rabbitmq.username=cg
    spring.rabbitmq.password=123456

3)启动两个config-client-eureka，在不同的port上

4）试验：我们先访问config-client-eureka /from请求，可以看到获取到了配置信息
然后我们修改repo的配置，然后post请求/bus/refresh到onfig-client-eureka.
然后我们再访问两个客户端的/from ，将会看到修改后的配置文件

![架构图](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Bus/5-1.png)

系统启动后，三个Service-A都会连接到Config-Server去获取配置。

如果我们修改Service-A的属性，我们需要向其中一个比如3实例发送/bus/refresh 的post请求。该请求会刷新到消息总线中，实例1，2也会获取到，从而实现动态更新。

局部刷新同样我们可以实现，通过/bus/refresh?destination=customers:9000，来指定刷新某个服务的配置

![架构优化](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Bus/5-2.png)


上面的架构，服务配置更新需要指向某个实例，web hook配置写死了，如果以后进行服务器迁移，还不得不修改web hook的配置，所以需要调整。
在ConfigServer中引入SpringCloudBus，将其引入到消息总线中。 /bus/refresh请求直接post给ConfigServer就可以了。并通过/bus/refresh?destination=customers:9000的形式来指定更新配置

## kafaka + spring cloud bus

pom需要引入spring-cloud-starter-bus-kafka 吧刚才的zookeeper+kafka启动起来，之前amqp整合中将破灭修改后，直接启动config-server和config-client
