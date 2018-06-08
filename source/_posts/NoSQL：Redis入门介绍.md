---
title: NoSQL：Redis入门介绍
date: 2018-06-05 17:07:44
tags:
---
## 一、Redis入门概述
### 1.Redis是什么
Redis：REmote DIctionary Server（远程字典服务器），是一个高性能的（key/value）分布式内存数据库，基于内存运行，并支持持久化的NOSQL数据库，也被人们成为数据结构数据库。

    Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用
    Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset（sorted set），hash等数据结构的存储
    Redis支持数据的备份，即master-slave模式的数据备份
