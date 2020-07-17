---
title: Redis：Redis事务
date: 2020-02-17 11:06:30
tags:
categories: Redis
---
## Redis事务
### 简介
Redis事务可以一次执行多个命令(允许在一次单独的步骤中执行一组命令，按顺序地串行化执行，执行中不会被其它命令插入)
并且带有以下两个重要的保证:

批量操作在发送exec命令前被放入队列缓存
收到exec命令后进入事务执行，事务中任意命令执行失败，其余的命令以然被执行。
在事务执行过程中，其他客户端提交的命令请求不会插入到事务执行命令序列中。

    1.Redis会将一个事务中的所有命令序列化，然后按顺序执行
    2.执行中不会被起他命令插入，不许出现加塞行为

一个事务从开始到执行会经历以下三个阶段：
开始事务
命令入队
执行事务
### 命令
DISCARD
取消事务，放弃执行事务块内的所有命令
EXEC
执行所有事务块内的命令
MULTI
标记一个事务块的开始
UNWATCH
取消WATCH命令对所有key的监视
WATCH key[key...]
监视一个(或多个)key，如果在事务执行之前这个(或这些)key被其他命令所改动，那么事务将被打断
### 示例
#### 正常的执行流程
```java
// 事务开始
multi
// 添加命令到队列
get account:a
get account:b
decrby account:a 50
incrby account:b 50
// 执行事务
exec
```
#### 放弃队列
```java
multi
get account:a
get account:b
decrby account:a 50
incrby account:b 50
discard
```
#### 事务的错误处理
不影响事务的执行，会跳过错误的命令顺序执行其他的命令
```java
multi
set hello hello
// 错误的命令
incr hello
get hello
exec
```
此种情况会由于之前的错误，事务执行失败
```java
multi
set aa 123
// 报告错误
sdafwwf
get aa
exec
```
#### 事务的WATCH
