---
title: Redis数据存储问题
toc: true
date: 2017-02-17 16:38:26
tags:
- NOSQL
categories:
- Redis
---

介绍完Redis的基本命令，来看看Redis对数据存储的方面的设计。

## 持久化

Redis提供两种方式进行持久化，Snapshot VS AOF(Append-only file)。持久化可以解决数据备份和恢复的问题，还可以存储一些需要长时间计算的结果，节省开销。

### Snapshot

BGSAVE命令，异常保存数据，主线程提供服务，子线程执行数据存储，**Windows平台不具有fork特性**。
SAVE命令，停止服务，专门进行数据存储至磁盘。
用例，

如果60秒内，有10000次写操作，会触发BGSAVE操作。
~~~cmd
save 60 10000
~~~
shutdown命令，会停止所有客户端，如果有保存点，会执行SAVE命令，如果AOF选项被打开，更新AOF文件，最后关闭redis服务器。
~~~cmd
SHUTDOWN SAVE （强制让数据库执行保存操作，即使没有设定保存点）
SHUTDOWN NOSAVE （阻止数据库执行保存操作）
~~~

**全局备份（BGSAVE），耗时严重，fork会很慢，SAVE相对快，但阻塞用户请求。**

### Append-only file

进行Config设定或者配置
appendonly yes

appendfsync的选项，
1. always ，每次写操作都写入磁盘，最大限度减少数据丢失。
2. everysec，每秒的写操都写入磁盘。
3. no，让OS决定什么时候同步数据至磁盘。

通常来说，将文件写入磁盘，需要做三件事儿，一是创建缓冲区，二将数据写入缓冲区，三是将缓冲区中的数据写入磁盘文件。在使用always选项时要慎重，会减少磁盘的寿命。

**如果数据量很大，会导致AOF文件很大。致使磁盘空间不够，或者重启恢复数据非常慢。BGREWRITEAOF命令，可以重写AOF文件，优化该文件。**
**从 Redis 2.4 开始， AOF 重写由 Redis 自行触发。**

## 复制

系统扩展，提升性能。
对于单个Redis服务器，想要在10毫秒内完成单个命令，我们需要限制每秒只有100个请求到单台服务器。（随便服务器的升级，单台的性能会有所提升，但还是会遇到瓶颈。）

Redis提供主从（Master/Slave）设置，写操作在Master执行，然后将数据实时同步至Slave服务器。客户端的读操作，在Slave服务器执行（多台，可进行负载均衡）。

slaveof host port，连接一个Master服务器，可以作为配置项，也可以当命令执行。
slaveof no one，停止主从关系。

**如果slaveof作为配置项，启动时，会加载snapshot/AOF，然后连接Master，开始复制过程。**
**如果在运行时执行slaveof命令，Redis会立即Master，开始复制过程。**

### 复制步骤

复制过程，
| 步骤 | Master | Slave |
|--------|--------|--------|
|  1 | 等待命令| connect to master,发出SYNC命令（内部命令） |
| 2 | 开始BGSAVE操作 | 用旧数据提供服务或者返回error（根据配置）|
|3| 完成BGSAVE操作，开始给slave发送snapshot|抛弃旧数据，开始加载收到的dump文件 |
|4|完成snapshot发送，开始发送写命令的backlog给slave|完成dump解析，开始正常响应请求|
|5|完成backlog发送，开始实时写操作流|完成写操作的backlog执行，收到写操作流，继续执行同步|

通常情况，我们需要考虑Master服务器的内存使用情况，建议内存使用率不要超过50%-65%，要预留足够的空间给服务器执行BGSAVE和backlog命令。还有Redis不支持Master-Master复制。

### 主从结构

对于Redis数据库，主从关系是相对的，可以形成一个主从关系的树结构。当应用有大量读操作时，我们建立的主从关系，应避免1对多的情况（服务器带宽以及性能限制），而是建立多层次的主从结构关系。


## 处理系统错误

Redis数据库无法跟传统数据库一样，提供ACID（原子性，一致性，隔离性，持久性）功能。

### 命令恢复备份
当系统崩溃时，需要使用命令工具来恢复系统。
redis-check-aof [--fix] file.aof， 恢复aof数据，--fix帮助我们扫描aof文件，查找未完成以及不正确的命令，然后剪接文件，保证能够文件被执行加载。
redis-check-dump dump.rdb，恢复dump数据，目前还没提供命令工具帮忙修复snapshot，重要的snapshot建议多备份。

### 更换Master

如果Master服务器出现问题时，需要替换它。
例如，A，B作为Redis服务器运行。A是Master，B是Slave。不幸地，A机器网络出现问题，现在要用C，来作为新的Master，怎么做，

1. 告诉B，执行Save命令，生成最新的snapshot。
2. snapshot完成后，启动C，加载snapshot。
3. 告诉B，成为C的Slave。

还有一个可选方法，将B作为Master，C作为B的Slave。

处理异常，我们还有Redis Sentinel工具，后续再介绍。

## 总结

本篇文章，更多的是Redis使用说明以及Redis提供解决问题的思路，后续需要实战练习。



