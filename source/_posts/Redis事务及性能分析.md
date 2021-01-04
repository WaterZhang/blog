---
title: Redis事务及性能分析
toc: true
date: 2017-02-27 16:34:42
tags:
- NOSQL
categories:
- Redis
---


上篇文章，学习了Redis进行持久化以及复制相关问题，继续学习，了解Redis的事务相关处理。

## 事务

在不同的客户请求中，如何保证数据正确性，顺带要提升性能？

对于传统的数据库，做事务，我们先告诉DB，要开始一个事务Begin，然后执行批量的读写操作，最后提交事务Commit，使这些状态变化是永久性的，要不全部回滚。

对于Redis，开始“事务”使用MULTI命令，然后传递多个命令，最后跟上EXEC命令。这个过程中，我们需要等待EXEC命令调用，才真正做事，中间过程，不能做其他操作。（有点像延时执行命令）。这样处理的好处，是提升性能，减少网络负载（客户端和服务器交互）。缺点是中间过程不够灵活。

## 命令

WATCH，MULTI，EXEC，UNWATCH，DISCARD
WATCH，标记所有指定的key被监视起来，在事务中有条件的执行（乐观锁），我们使用WATCH命令后，其他客户来更新这些key时，会报错。这样确保，我们在做一些更新操作时，数据不会被串改。
UNWATCH，取消事务命令。
DISCARD，丢弃MULTI之后发出的命令。
EXEC，执行批量命令。

这里有[悲观锁](https://www.zhihu.com/question/29397176)的介绍。
这里有[乐观锁](http://baike.baidu.com/link?url=EDNymMhkUMC8tL4Nhap_z5kuv9JSmG9Vl-amZw6lw2A6JyQ2EuKcxzyxgZcloxnlC9PLHsjVboM1t0nAjOVeDFpG_wbIQuw5BWPNJtkMEl0ZKdp3AhFy2e13KRhRHpmb)的介绍。类似Java的AQS的CAS特性。

## 性能测试

Redis基准测试，
redis-benchmark -c 1 -q

## 总结

简单介绍一下Redis中的“事务”。
