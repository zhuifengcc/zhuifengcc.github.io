---
title: 'NoSQL:Redis基础及常用数据类型'
date: 2019-12-28 16:12:37
tags:
categories: NoSql
---
## 一、Redis常用命令
### 1.Redis键key
DEL key
该命令用于在key存在时删除key
DUMP key
序列化给定key，并返回序列化的值
EXISTS key
检查给定key是否存在
EXPIRE key seconds
为给定key设置过期时间（以s计）
ps: 应用场景
 1.限时的优惠活动信息
 2.网站数据缓存（对于一些需要定时更新的数据，例如：积分排行榜）
 3.手机验证码
 4.限制网站访客访问频率（例如：1分钟最多访问10次），限制登陆了之后多少秒可以再次登陆（过期时间）
PEXPIRE key milliseconds
为给定key设置过期时间（以ms计）
TTL key
以s为单位，返回给定key的剩余生存时间（time to live）
PTTL key
以ms为单位返回key的剩余的过期时间
PERSIST key
移除key的过期时间，key将持久保持
KEYS pattern
查找所有符合给定模式（pattern）的key
keys 通配符  获取所有与pattern匹配的key，返回所有与该匹配
 通配符:
    * 代表所有
    ? 表示代表一个字符
 ps: 如当前有3个key(users:1,users:2,users:3) 可以通过keys users:?查询
RANDOMKEY
从当前数据库中随机返回一个key
RENAME key newkey
修改key的名称
MOVE key db
将当前数据库的key移动到指定的数据库db中
TYPE key
返回key所存储的值的类型
### 2.key的命名建议
1.key不要太长，尽量不要超过1024子节，这不仅消耗内存，而且会降低查找的效率
2.key也不要太短，太短的话，key的可读性会降低
3.在一个项目中，key最好使用统一的命名模式，例如: user:123:password