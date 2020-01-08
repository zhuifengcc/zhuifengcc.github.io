---
title: Redis：Redis API
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
if (jedis.exists(key)) {
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
## RedisTemplate
Spring data提供了RedisTemplate模板
它封装了redis连接池管理的逻辑，业务代码无需关心获取，释放连接逻辑
常用的接口方法:
``` java
private ValueOperations<K, V> valueOps;
private ListOperations<K, V> listOps;
private SetOperations<K, V> setOps;
private ZSetOperations<K, V> zSetOps;

private HashOperations<K, HK, HV> hashOps;

@Resource(name = "redisTemplate")
ListOperations<String, String> list;
```

    <dependency>
        <groupId>org.springframework.data</groupId>
        <artifactId>spring-data-redis</artifactId>
        <version>1.4.2.RELEASE</version>
    </dependency>
RedisTemplate操作实例:
String类型
``` java
public String getString(String key) {
    ValueOperations<String, String> valueOps = redisTemplate.opsForValue();
    if (redisTemplate.hasKey(key)) {
        return valueOps.get(key);
    } else {
        // 查询数据库
        String result = userDao.getName();
        valueOps.set(key, result);
        // 设置过期时间
        // valueOps.set(key, result, 10, TimeUnit.SECONDS);
        // 先赋值，再设置过期时间
        // valueOps.set(key, result);
        // redisTemplate.expire(key, 10, TimeUnit.SECONDS);
        return result;
    }
}
```

    把任何数据保存到redis中的时候，都需要进行序列化，默认使用JdkSerializationRedisSerializer进行数据序列化。所有的key和value还有hashkey和hashvalue的原始字符前，都加了一串字符。
    可以修改keySerializer为StringRedisSerializer,valueSerializer为StringRedisSerializer
    或者hashKeySerializer为StringRedisSerializer,hashValueSerializer为StringRedisSerializer
Hash类型
``` java
public void addUser(User user) {
    // 这样存储类似与数据库表的存储形式，与hmset不一样
    redisTemplate.opsForHash().put("user", user.getId(), user);
}
```
``` java
public User selectById(String id) {
    if (redisTemplate.opsForHash().hasKey("user", id)) {
        return (User) redisTemplate.opsForHash().get("user", id);
    } else {
        // 查询数据库
        User user = userDao.getUserById(String id);
        redisTemplate.opsForHash().put("user", user.getId(), user);
        return user;
    }
}
```
list类型
``` java
@Resource(name = "redisTemplate")
ListOperations<String, String> list;

public void listAdd() {
    //list.leftPush("news:top5", "news1");
    //list.leftPush("news:top5", "news2")
    list.leftPushAll("news:top5", "news1","news2","news3","news4","news5");
}

public List<String> listRange(int pageNum, int pageSize) {
    // offset: (pageNum-1)*pageSize
    return list.range("news:top5", (pageNum-1)*pageSize, pageSize*pageNum-1);
    //return list.range("news:top5", 0, -1);
}
```
