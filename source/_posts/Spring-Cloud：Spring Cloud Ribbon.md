---
title: Spring Cloud：Spring Cloud Ribbon
date: 2018-06-18 12:54:39
tags:
---
## 客户端负载均衡
Spring Cloud Ribbon是一个基于HTTP和TCP的客户端负载均衡，他基于Netflix Ribbon实现。通过Spring Cloud的封装，可以轻松地将面向服务的REST模板请求自动转换成客户端负载均衡的服务调用。

我们都知道，负载均衡是对系统的高可用、网络压力的缓解和处理能力扩容的重要手段之一。

```java
RestTemplate restTemplate = new RestTemplate();
```
