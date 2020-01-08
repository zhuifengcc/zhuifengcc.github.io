---
title: 'Redis: Redis基础及常用数据类型'
date: 2019-12-28 16:12:37
tags:
categories: Redis
---
## Redis常用命令
命令不区分大小写，key区分大小写
### Redis键key
DEL key
该命令用于在key存在时删除key
DUMP key
序列化给定key，并返回序列化的值
EXISTS key
检查给定key是否存在
EXPIRE key seconds
为给定key设置过期时间（以s计）
PEXPIRE key milliseconds
为给定key设置过期时间（以ms计）

    应用场景
    1.限时的优惠活动信息
    2.网站数据缓存（对于一些需要定时更新的数据，例如：积分排行榜）
    3.手机验证码
    4.限制网站访客访问频率（例如：1分钟最多访问10次），限制登陆了之后多少秒可以再次登陆（过期时间）
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
    如当前有3个key(users:1,users:2,users:3) 可以通过keys users:?查询
RANDOMKEY
从当前数据库中随机返回一个key
RENAME key newkey
修改key的名称
MOVE key db
将当前数据库的key移动到指定的数据库db中
TYPE key
返回key所存储的值的类型
FLUSHDB
清空db
### key的命名建议
    Redis单个key存入512M大小
    1.key不要太长，尽量不要超过1024子节，这不仅消耗内存，而且会降低查找的效率
    2.key也不要太短，太短的话，key的可读性会降低
    3.在一个项目中，key最好使用统一的命名模式，例如: user:123:password
    ps: 一般不用_,避免与字段中的_冲突
## Redis数据类型
Redis支持五种数据类型：string(字符串), hash(哈希), list(列表), set(集合), zset(sorted set 有序集合)
### String
string是redis最基本的数据类型，一个key对应一个value。一个key最大存储512M。
string类型是二进制安全的。意味着是Redis的string可以包含任何数据，比如图片或者序列化的对象。

    二进制安全是指，在传输数据时，保证二进制数据的信息安全，也就是不被篡改、破译等，如果被攻击，能够及时检测出来
    二进制安全特点：
        1.编码，解码发生在客户端完成，执行效率高
        2.不需要频繁的编解码，不会出现乱码
#### String命令
赋值语法：
SET KEY_NAME VALUE
用于设定指定key的值。如果key已经存储值，SET就覆写旧值，且无视类型
SETNX key value //解决分布式锁 方案之一
只有在key不存在的时候设置key的值。SetNX(SET if Not eXist) 命令在指定的key不存在时，为key设置指定的值
MSET key value [key value ...]
同时设置一个或多个key-value对
取值语法：
GET KEY_NAME
用于获取指定key的值。如果key不存在，返回nil。如果key存储的值不是字符串类型，返回一个错误
GETRANGE key start end
用于获取存储在指定key中字符串的子字符串。字符串的截取由start和end两个偏移量决定（包括start和end在内）
GETBIT key offset
对key所存储的字符串值，获取指定偏移量上的位（bit）
MGET key1 [key2 ..]
获取所有（一个或多个）给定的key值
GETSET语法：
GETSET KEY_NAME VALUE
用于设定指定key的值，并返回key的旧值，当key不存在时，返回nil
STRLEN key
返回key所存储的字符串的长度
删除语法：
DEL KEY_NAME
删除指定的key，如果存在，返回值数字类型
自增/自减：
INCR KEY_NAME
将key中存储的数字增1。如果key不存在，那么key的值会先被初始化为0，然后执行INCR操作
INCRBY KEY_NAME 增量值
将key中存储的数字加上指定的增量值
DECR KEY_NAME 
将key中存储的数字减1
DECRBY KEY_NAME 减值
将key中存储的数字减去指定的减值
APPEND KEY_NAME VALUE
用于为指定的key追加至末尾，为其赋值
#### 应用场景
1.String通常用于保存单个字符串或json字符串数据
2.因String是二进制安全的，可存储图片信息
3.计数器（常规key-value缓存应用。常规计数：粉丝数，微博数）
INCR等指令本身就具有原子操作的特征，所以我们完全可以用Redis的INCR，INCRBY，DECR，DECRBY等指令来实现原子计数的效果。例如，在某种场景下有3个客户端同时读取了mynum的值（值为2），同时进行了加1的操作，那么最后mynum一定是5
不少网站利用这种特征实现业务上的统计计数的需求。
### Hash
Redis hash是一个string类型的field和value的映射表，hash特别适用与存储对象。Redis中每个hash可以存储2^32-1键值对（40多亿）可以看成具有key和value的map容器。该类型的数据仅占用很少的磁盘空间（相比与json）
#### Hash命令
赋值语法：
HSET key field value
为指定的key，设定field/value
HMSET key field value [field1 value1] ...
同时将多个k-v对设置到哈希表key中
取值语法：
HGET key field
获取存储在Hash中的值，根绝field得到value
HMGET key field1[field2]
获取key所有指定字段的值
HGETALL key
返回Hash表中所有的字段和值
HKEYS key
获取所有Hash表中的字段
HLEN key
获取哈希表中字段的数量
删除语法：
HDEL key field1[field2]
删除一个或多个Hash表字段
ps: 如果Redis的value值被删除完了，这个key会被自动回收掉。故可以通过删除所有当前Hash的field来删除当前Hash的key。
其它语法：
HSETNX key field value
只有在字段field不存在时，设置Hash表字段的值
HINCRBY key field increment
为Hash表key中的指定字段的整数值加上增量increment
HINCRBYFLOAT key field increment
为Hash表key中的指定字段的浮点数值加上增量increment
HEXISTS key field
查看Hash表key中，指定的字段是否存在
#### 应用场景
1.常用于存储一个对象
2.为什么不用String存储对象？（序列化与反序列化，修改值的问题，并发问题）
Hash是最接近关系数据库结构的数据类型，可以将数据库一条记录或程序中的一个对象转换为hashmap存储在Redis中，并提供了直接存取这个Map成员的接口
### List
Redis 列表是简单的字符串列表，按照插入顺序排序，可以添加一个元素列表的头部或者尾部，一个列表最多可以包含2^32-1个元素。类似java中的LinkedList
#### List命令
赋值语法：
LPUSH key value1 [value2]
将一个或多个值插入到列表头部(从左侧添加)
RPUSH key value1 [value2]
在列表中插入一个或多个值(从右侧添加)
LPUSHX key value
将一个值插入到已存在的列表头部，如果列表不存在，操作无效
RPUSHX key value
将一个值插入到已存在的列表尾部(最右边)，如果列表不存在，操作无效
取值语法：
LLEN key
获取列表长度
LINDEX key index
通过索引获取列表中的元素
LRANGE key start stop
获取列表指定范围内的元素 eg: lrange l1 0 -1
区间以偏移量START和END指定。其中0表示列表的第一个元素，1表示列表的第二个元素，以此类推。也可以使用负数下标，以-1表示列表的最后一个元素，-2表示列表的倒数第二个元素，以此类推。
删除语法：
LPOP key
移除并获取列表的第一个元素(从左侧删除)
RPOP key
移除并获取列表的最后一个元素(从右侧删除)
BLPOP key1 [key2] timeout
移除并获取列表的第一个元素，如果列表没有元素会阻塞列表直到等待超时时间或发现可弹出元素为止。
eg：blpop list1 100
操作会被阻塞，如果指定的列表key list1 存在数据则会返回第一个元素，否则在等待100s后返回nil
BRPOP key1 [key2] timeout
移除并获取列表的最后一个元素，如果列表没有元素会阻塞列表直到等待超时时间或发现可弹出元素为止。
LTRIM key start stop
对一个列表进行修剪，就是说，让列表只保留指定区间内的元素，不在指定区间内的元素都将被删除。
修改语法：
LSET key index value
通过索引设置列表元素的值
LINSERT key BEFORE|AFTER world value
在列表的元素前或者后插入元素  将值value插入到列表key当中，位于值world之前或者之后
#### 高级语法
RPOPLPUSH source destination
移除列表的最后一个元素，并将该元素添加到另一个列表并返回
eg：
RPOPLPUSH a1 a2  //a1的最后元素移到a2的左侧
RPOPLPUSH a1 a1  //循环列表，将最后元素移到最左侧
BRPOPLPUSH source destination timeout
从列表中弹出一个值，将弹出的元素插入到另外一个列表中并返回它；如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止
#### 应用场景
1.对数据量大的集合数据删减
列表数据显示、关注列表、粉丝列表、留言评价等...分页、热点新闻(Top5)等
利用LRANGE还可以很方便的实现分页的功能，在博客系统中，每篇博文的评论也可以存入一个单独的list中
2.任务队列
(list通常用来实现一个消息队列，并且可以确保先后顺序，不必像mysql那样还需要order by)
任务队列介绍(生产者和消费者模式)：
在处理Web客户端发送的命令请求时，某些操作的执行时间可能会比我们预期的更长一些，通过将待执行任务的相关信息放入队列里面，并在之后对队列进行处理，用户可以推迟执行那些需要一段时间才能完成的操作，这种将工作交给任务处理器来执行的做法被称为任务队列
RPOPLPUSH source destination
常用案例：订单系统的下单流程，用户系统登陆注册短信等
