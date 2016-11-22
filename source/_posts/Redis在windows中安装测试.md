---
title: Redis在windows中安装测试
date: 2016-11-17 10:59:54
tags: Redis 缓存
categories:
- Redis
toc: true
---

开始学习入门Redis。

## 安装
作者本人没有Linux服务器，用来安装和部署，只能寻求Windows版本。官方说明，[windows版本](https://github.com/MSOpenTech/redis)交由Microsoft Open Tech Group来管理。具体Windows和Linux版本在使用上，有没有区别？一边学习一边跟进。

在Github上找到最新的windows版本，[3.2.1](https://github.com/MSOpenTech/redis/releases)，上面有说明，没有在生产环境验证。感觉windows版本是不靠谱的。。。

自动安装完，redis会作为windows service，已经启动。在官方的文档说明中，可以看出Redis是为Unix系统而生，Windows的人做了迁移，但是做不到完美匹配。比如fork功能，
***
fork()
------

The POSIX version of Redis uses the fork() API. There is no equivalent in Windows, and it is an exceedingly difficult API to completely simulate. For most of the uses of fork() we have used Windows specific programming idioms to bypass the need to use a fork()-like API. The one case where we could not do so was with the point-in-time heap snapshot behavior that the Redis persistence model is based on. We tried several different approaches to work around the need for a fork()-like API, but always ran into significant performance penalties and stability issues.
***

[这里](http://try.redis.io/)是官方的网页学习资料，学习简单的指令先，有个基本的认知。

## Java Client测试

### Jedis
[Jedis](https://github.com/xetorthio/jedis)是官方推荐的。
导入Jedis最新版本[2.9.0](https://github.com/xetorthio/jedis/releases)，写了一个简单的用例代码，
~~~java
public class Test {

private static JedisPool pool = new JedisPool(new JedisPoolConfig(), "localhost");

    public static void main(String[] args) {

        try (Jedis jedis = pool.getResource()) {
            /// ... do stuff here ... for example
            jedis.set("foo", "bar");
            String foobar = jedis.get("foo");
            System.out.println(foobar);
            jedis.zadd("sose", 0, "car");
            jedis.zadd("sose", 0, "bike");
            Set<String> sose = jedis.zrange("sose", 0, -1);
            System.out.println(sose);
        }
        /// ... when closing your application:
        pool.destroy();

    }
}
~~~

