---
title: 'Redis: NoSQL概述'
date: 2018-06-01 20:38:10
tags:
categories: Redis
---
## 发展历程
### 单机关系型数据库
APP - DAL - MySQL Instance（APP - Action - Service - DAO - MySQL Instance）
![单机结构图](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/summary/1.1.png)
数据存储的瓶颈：

    1、数据量的总大小，一个机器放不下
    2、数据的索引（B+树），一个机器内存放不下
    3、访问量（读写混合），一个实例不能承受（后续主从复制，读写分离的诞生）
### Memcache（缓存）+ MySQL + 垂直拆分
APP - DAL - Cache - MySQL Instances（Instance1：business 1、Instance2：business 2、Instance3：userinfo...）
![缓存+垂直拆分](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/summary/1.2.png)
数据库垂直拆分
### MySQL主从复制、读写分离
![主从复制](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/summary/1.3.png)
主从复制（master/slave）：在主库插入时，从库迅速插入，容灾备份
读写分离：读写分别在不同的数据库，写操作在主库进行，读操作在从库进行
### 分库分表 + 水平拆分 + MySQL集群（Cluster）
![分库分表+集群](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/summary/1.4.png)
主库写压力出现瓶颈，MySQL数据引擎MyIASM使用表锁（锁整张表），InnoDB使用行锁（锁单行数据，并发性高），MySQL应用开始使用InnoDB。

分库分表：紧耦合的业务相关数据放在一个库，变化小冷热门的数据放在一个库。数据量大，分表存储。
### MySQL的扩展性瓶颈
Blob，CLob，视频的存储？大数据下IO压力大，表结构更困难
### 今天是什么样子？
Users - 企业防火墙 - nginx主备 - 应用服务器集群 - MySQL/Oracle集群（缓存服务器、hadoop服务器、流媒体服务器、电子邮件服务器） 
### NoSQL的出现
用户产生的数据和用户操作日志已经成倍的增长，如果我们要对这些用户数据进行挖掘，如复杂的人际关系，传统关系型数据库已经不适合这些应用了
## NoSQL介绍
### NoSQL是什么
Not Only Sql，泛指非关系型数据库。数据存储不需要固定的模式，无需多余操作就可以横向扩展。
### NoSQL可以做什么
    1、易扩展：去掉关系型特性，数据之间无关系，非常容易扩展（K/V键值对）
    2、大数据量高性能:NoSQL数据库具有非常高的读写性能，例如Redis，可达到1s写8w次，读11w次
    3、多样灵活的数据模型：NoSQL无需为事先存储的数据建立字段，随时可以存储自定义的数据格式。而在关系型数据库中，如果是一张非常大数据量的表，增删字段是非常痛苦的
## 互联网时代的3V+3高
大数据时代的3V：海量（Volume）、多样（Variety）、实时（Velocity）

互联网需求的3高：高并发、高可扩（横向扩展，集群）、高性能/高可用（单点故障、容灾备份、keep-alive）

多数据源多数据类型的存储问题，以淘宝为例，商品基本信息（MySQL，淘宝自主，去IOE化（IBM小型机、Oracle、EMC存储设备），商品描述、详情、评论信息（多文字类）（MengoDB），图片（分布式的文件系统中TFS），站内搜索（ISearch），高频词汇（redis），商品的交易，价格计算、计分累计（3方接口）
## NoSQL数据模型简介（聚合模型）
BSON（）是一种类json的一种二进制形式的存储格式，简称Binary JSON，它和JSON一样，支持内嵌的文档对象和数组对象。

    KV键值对
    BSON
    列族
    图形
## NoSQL数据库的四大分类
    KV键值
    文档型数据库（bson格式较多）
        CouchDB
        MengoDB:基于分布式文件存储的数据库。介于关系型数据库和非关系型数据库之间的产品，是非关系型数据库中功能最丰富，最想关系型数据库的。
    列存储数据库
        Cassandra，HBase
        分布式文件系统  
    图关系数据库
        放的是关系比如：朋友圈社交网络、广告推荐系统
        Neo4J，InfoGrid
## 在分布式数据库中CAP原理CAP+BASE
RDBMS 传统的ACID

    A:Atomicity(原子性)
    C:Consistency(一致性)
    I:Isolation(独立性)
    D:Durability(持久层)

CAP:
    
    C:Consistency(强一致性)
    A:Availability(可用性)
    P:Partition tolerance(分区容错性)
CAP的3进2（最多只能同时较好的满足两个）

    CA：单点集群，满足一致性，可用性的系统，通常在可扩展性上不太强大（RDBMS）
    CP：满足一致性，分区容忍必的系统，通常性能不是很高（MengoDB、Redis、HBase）
    AP：满足可用性，分区容忍性的系统，通常可能对一致性要求低一些（CouchDB）（大多数网站架构的选择，弱一致性加AP）
BASE：
    BASE就是为了解决关系型数据库强一致性引起的问题而引起的可用性降低而提出的解决方案。

    基本可用：（Basically Available）
    软状态（Soft state）
    最终一致（Eventually consistent）
它的思想是通过让系统放松对某一时刻数据一致性的要求来换取整体系统伸缩和性能上改观，由于大型系统往往由于地域分布和极高性能的要求，不可能采用分布式事务完成这些指标，要想获得这些指标，我们必须采用另外一种方式来完成，这里BASe就是解决这个问题的办法

最后在说一下分布式与集群：

分布式系统（distributed system）

由多台计算机和通讯的软件组件通过计算机网络连接（本地网络或广域网）组成。分布式系统是建立在网络之上的软件系统。正式因为软件的特性，所以分布式系统具有高度的内聚性和透明性。因此，网络和分布式系统之间的区别更多的在于高层软件（特别是操作系统），而不是硬件。分布式系统可以应用在不同的平台上如：PC、工作站、局域网和广域网上等。

简单来说：

    1.分布式：不同的多台服务器上面部署不同的服务模块（工程），他们之间通过RPC（远程过程调用）/RMI（远程方法调用）之间通信和调用，对外提供服务和组内协作。
    2.集群：不同的多台服务器上部署相同的服务模块，通过分布式调度软件进行统一的调度，对外提供服务和访问。




