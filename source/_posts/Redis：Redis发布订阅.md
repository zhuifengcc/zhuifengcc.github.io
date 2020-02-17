---
title: 'Redis：Redis发布订阅'
date: 2020-02-02 17:06:49
tags:
categories: Redis
---
### 简介
Redis发布订阅(pub/sub)是一种消息通信模式: 发布者(pub)发送消息，订阅者(sub)接收消息。
Redis客户端可以订阅任意数量的频道。
### 示例
客户端订阅同一个频道
![客户端订阅频道](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_4/5-1.png)

消息发布后，推送给每一个客户端
![消息发布](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_4/5-2.png)
### 命令
订阅频道：
SUBSCRIBE channel [channel ...] 订阅给定的一个或者多个频道的信息
PSUBSCRIBE pattern [pattern ...] 订阅给定的一个或者多个符合给定模式的频道
发布频道：
PUBLISH channel message 将消息发送到指定的频道
退订频道：
UNSUBSCRIBE [channel [channel ...]] 退订所有给定的频道
PUNSUBSCRIBE [pattern [pattern ...]] 退订所有给定模式的频道
### 应用场景
构建实时消息系统，比如普通的即时聊天，群聊等
1.博客网站中，当你发布新的文章时，可以推送消息给你的粉丝
2.微信公众号模式

