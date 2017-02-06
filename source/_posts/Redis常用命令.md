---
title: Redis常用命令
toc: true
date: 2017-02-06 15:43:40
tags:
- NOSQL
categories:
- Redis
---

上一篇，我在电脑上安装了windows版本的redis，使用Jedis编写代码测试。然后。。。停顿了好长时间没有继续学习。

## 初识
Redis，内存分布式数据库，支持5种数据类型（不确定），提供持久化支持，是近几年火起来的NOSQL。

### 数据库比较
| 数据库 | 类型 | 数据存储 | 查询类型 | 额外功能 |
| Redis | 内存非关系型 | String，List，Set，Hash，Sorted Set | CURD，批量处理，部分事务支持 | 订阅/取消服务 主从复制 |
| memcached | 内存非关系型 | mapping key to value | CURD | 并发集群服务 |
| mysql | 关系型 | 常用数据 | CURD，sp | ACID，主从结构 |
| PostgreSQL | 关系型 | 常用数据 | CURD，sp | ACID，主从结构，第三方扩展 |
| MongoDB | 磁盘非关系型文件存储 | BSON documents | CURD | map-redusce，主从，集群 |

### Redis数据同步
Redis有两种同步方式，将内存数据备份至磁盘：
- point-in-time dump ， 当一个确切的事情发生（一段时间内，满足一定量的写次数）或者执行dump-to-disk命令
- append-only file，灵活配置，从不同步到每秒同步，以及同步每一个操作。


## 常用数据结构
STRING，LIST，SET，HASH，ZSET。五种数据结构共享命令，DEL，TYPE，RENAME以及其他。
STRING，支持字符串，整型，浮点型。整型和浮点型可以自增，自减。
LIST，字符串列表。支持push/pop，可以在队头以及队尾操作。
SET，未排序的唯一集合。
HASH，存储键值对。
ZSET，排序列表。

## 命令
这里有个[Redis中文网站](http://www.redis.cn/)，方便检索Redis知识，以及Command。

我这里只是列出相关命令用法，简单说明。详细的用例以及相关参数说明，可以去网站中查询。

### STRING

1. INCR key-name 自增1
2. DECR key-name 自减1
3. INCRBY key-name amount 自增提供的值
4. DECRBY key-name amount 自减提供的值
5. INCRBYFLOAT key-name amount 自增提供的浮点值

对字符串操作，
**下标是从0开始的，跟Java语言一样。**

1. APPEND key-name value 拼接字符串至末尾
2. GETRANGE key-name start end  获取子串（包含下标）
3. SETRANGE key-name offset value  设置子串，从指定下标开始
4. GETBIT key-name offset 将字符串当做二进制字节串，获取偏移位置的值，要么是0，要么是1
5. SETBIT key-name offset value 设置指定偏移位置的值
6. BITCOUNT key-name [start end] 统计字节串中，有多少个1。可以统计子串。
7. BITOP operation dest-key key-name [key-name ...] 按位操作，操作有AND，OR，NOT，XOR。结果保存在dest-key


### LIST

1. RPUSH key-name value [value ...] 在右端存储元素
2. LPUSH key-name value [value ...] 在左端存储元素
3. RPOP key-name 出列右端元素
4. LPOP key-name 出列左端元素
5. LINDEX key-name offset 获取列表中指定索引的元素（左端）
6. LRANGE key-name start end 获取指定索引范围的元素（包含边界）
7. LTRIM key-name start end 修剪原始列表，只剩start到end索引

进阶，

1. BLPOP key-name [key-name ...] timeout 阻塞出列（左端），当没有元素时，阻塞或者超时等待。
2. BRPOP key-name [key-name ...] timeout 阻塞出列（右端）
3. RPOPLPUSH source-key dest-key 删除source的列表尾部元素，放入dest列表头部。source和dest可以是同一个列表
4. BRPOPLPUSH source-key dest-key 阻塞版本

### SET

1. SADD key-name item [item ...] 添加元素，返回成功添加的数量
2. SREM key-name item [item ...] 删除元素，返回成功删除的数量
3. SISMEMBER key-name item  判断元素是否在SET中。
4. SCARD key-name 获取集合的元素数量
5. SMEMBERS key-name 返回SET集合中所有元素
6. SRANDMEMBER key-name [count] 随机返回一个或者多个集合中的元素。
7. SPOP key-name  随机清除并返回一个元素
8. SMOVE source-key dest-key item 如果元素在source集合，清除并添加至dest集合，并返回该元素

进阶，

1. SDIFF key-name [key-name ...] 返回第一个集合里的相关元素，这些元素不再其他任何集合里
2. SDIFFSTORE dest-key key-name [key-name ...] 类似SDIFF，将元素存储在dest-key中
3. SINTER key-name [key-name ...] 返回所有集合共有的元素
4. SINTERSTORE dest-key key-name [key-name ...] 将集合共有的元素存储在dest-key中
5. SUNION key-name [key-name ...] 返回集合合并后的所有元素
6. SUNIONSTORE dest-key key-name [key-name ...] 将合并后的元素存储在dest-key中

### HASH

1. HMGET key-name key [key ...] 获取相关key的值
2. HMSET key-name key value [key value ...]  存储key value。
3. HDEL key-name key [key ...] 删除相应的key。返回已删除的个数。
4. HLEN key-name  返回hash集合的个数。

进阶，

1. HEXISTS key-name key  key是否存在于指定hash集合
2. HKEYS key-name  获取hash集合中的所有key
3. HVALS key-name 获取hash集合中所有的value
4. HGETALL key-name 获取hash集合中所有的key-value组合
5. HINCRBY key-name key increment 自增提供key的值（整型增量）。
6. HINCRBYFLOAT key-name key increment 自增提供key的值（浮点变量）。

### Sorted SET

1. ZADD key-name score member [score member ...]   有序集合添加分数/成员， 分数相同，有序的字典顺序。
2. ZREM key-name member [member]  有序集合删除相关成员。
3. ZCARD key-name 返回有序集合成员的个数
4. ZINCRBY key-name increment member  为有序集合的指定成员的score加上增量increment。
5. ZCOUNT key-name min max 返回有序集合中成员的分数在最小和最大之间的数量。
6. ZRANK  key-name member 返回指定成员的在集合中的位置（索引）。按照从小到大排列
7. ZSCORE key-name member 返回指定成员的分数。
8. ZRANGE key-name start stop [ WITHSCORES]  返回指定索引范围的成员（可附带分数）

进阶，

1. ZREVRANK key-name member  返回成员在有序集合的位置，与ZRANK的排序是相反的。从大到小排列。
2. ZREVRANGE key-name start stop [WITHSCORES]  与ZRANGE的排序相反。返回指定索引范围内的成员，但是排序是从大到小。
3. ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]  分数在最小和最大之间，由低到高排序，是否返回score，limit是否分页，offset返回结果起始位置，count返回结果数量
4. ZREVRANGEBYSCORE key max min [WITHSCORES] [LIMIT offset count]  分数在最大和最小之间，由高到低排序
5. ZREMRANGEBYRANK key-name start stop 删除指定索引范围的成员
6. ZREMRANGEBYSCORE key-name min max 删除指定分数范围内的成员
7. ZINTERSTORE dest-key key-count key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM| MIN| MAX] 将给定的有序集合的交集放到dest集合中。weights是给原来集合中的成员的score乘积因子，默认为1。 AGGREGATE 成员的分数聚合方式，默认是SUM。
8. ZUNIONSTORE dest-key key-count key [key ...] [WEIGHTS weight [weight...]] [AGGREGATE SUM| MIN| MAX]  将给定的有序集合的并集放到dest集合中。

### publish/subscribe

1. SUBSCRIBE channel [channel ...]  订阅频道
2. UNSUBSCRIBE [channel]  取消订阅指定的或者所有的
3. PUBLISH  channel message 发布消息到指定的频道
4. PSUBSCRIBE pattern [pattern ...]  订阅给定的模式
5. PUNSUBSCRIBE [pattern ...] 取消特定的模式的消息

消息订阅/取消模式，有两个问题。

- Redis系统的可靠性。早前版本中，客户已经订阅了拼单，但是读消息的速度不够快，导致Redis会积压大量缓存。当足够大时，Redis会崩溃。新版本，不会有这问题，可以设置buffer size。
- 二是数据传输可靠性，如果客户订阅频道，但是连接出问题。再重连上来之前，如果有发布消息，那么重连的客户不会收到，导致信息流失。

需要寻求更好的解决方案，待定。


### 其他命令

Sort排序命令，参数比较长，不列出来了。

事务处理相关命令，WATCH，MULTI，EXEC，UNWATCH，DISCARD
批量执行， MULTI+多条命令+EXEC

KEY过期处理，

1. PERSIST key-name  清除期限时间
2. TTL key-name  返回到过期的时间长度
3. EXPIRE key-name seconds 设置过期的时间
4. EXPIREAT key-name timestamp 设置过期的Unix时间戳
5. PTTL key-name 返回到过期时间的毫秒数
6. PEXPIRE key-name milliseconds 设置过期的毫秒数
7. PEXPIREAT key-name timestamp-milliseconds 设置到过期时间的Unix时间戳，以毫秒为单位。

## 总结

这里包含的大部分的命令，方便查找，温故知新。