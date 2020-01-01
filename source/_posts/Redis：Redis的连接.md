---
title: Redis：Redis的连接
date: 2019-12-30 23:59:41
tags:
categories: Redis
---
## Java连接Redis
一些Java客户端访问，有：Jedis/Redisson/Jredis/JDBC-Redis等，其中官方推荐使用Jedis和Redisson。常用Jedis

    <dependency>
        <groupId>redis.clients</groupId>
        <artifactId>jedis</artifactId>
        <version>2.4.2</version>
    </dependency>
确保Redis服务器开启了外部端口:

    查看已经开放的端口: firewall-cmd --list-ports
    开启端口：firewall-cmd --zone=public --add-port=6379/tcp --permanent
    重启防火墙：firewall-cmd --reload
``` java
// redis sever host && port
Jedis jedis = new Jedis("192.168.20.1", 6379);
// password
jedis.auth("1q2w3e4r");
// test connection
System.out.println(jedis.ping());
jedis.set("stringKey", "string value");
System.out.println(jedis.get("stringKey"));
jedis.close();
```
### Redis连接池
``` java
// 连接池配置信息
JedisPoolConfig poolConfig = new JedisPoolConfig();
// 最大连接数
poolConfig.setMaxTotal(5);
// 最大空闲数
poolConfig.setMaxIdle(1);
// ...
JedisPool jedisPool = new JedisPool(poolConfig, "192.168.20.1", 6379);
Jedis jedis = jedisPool.getResource();
// ...
```
常用获取Redis key例子:
``` java
// 获取连接...
// 判断是否在Redis中存在该key
if(jedis.exists(key)) {
    // 从Redis中获取目标数据类型，如哈希
    Map<String, String> hash = jedis.hgetAll(key);
    // ...
} else {
    // 从数据库查询出数据并添加到Redis中
    User user = userDao.getUserById("1");
    Map<String, String> hash = new HashMap<>();
    hash.put("id", user.getId());
    hash.put("name", user.getName());
    // ...
    jedis.hmset(key, hash);
}
```