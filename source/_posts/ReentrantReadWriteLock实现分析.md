---
title: ReentrantReadWriteLock实现分析
date: 2016-11-09 10:33:13
tags: Java
categories: Java并发编程
toc: true
---

ReentrantReadWriteLock，顾名思义，支持重入，支持读锁和写锁。

## API规范相关说明

跟ReentrantLock一样，支持非公平（默认）和公平模式。
**非公平模式**下，读锁和写锁的顺序是没有指定的。不断的竞争可能导致其他读或写线程阻塞，但是性能要优于公平锁，减少线程间切换的损耗。
**公平模式**下，竞争是arrival-order策略，最先阻塞的线程最具竞争。当前锁释放后，最长等待的写锁线程将赋予写锁。没有写锁时，最长等待的读锁获取读锁。
读锁来了，如果现在有写锁或者写锁的阻塞队列不为空，获取读锁的线程将继续阻塞，直到写锁的线程不存在了且写锁阻塞队列为空。也就是说写锁比读锁高一级，有写锁存在，读锁就阻塞，没有写锁了，读锁自己竞争。
写锁来了，如果读锁且写锁都释放了，则抢占。意味着，写锁和读锁都木有阻塞的线程。
**重入**，写锁线程可以在释放锁的情况下，获取读锁，反之不允许。这也是**锁降级**的特性。
**通用实现**，CachedData，加入volatile的flag，再加double check实现写方法。
**最大的锁数**，65535写锁和读锁。

## 类图
读写锁，跟ReentrantLock一样，都使用AQS来实现对锁的原子操作，ReentrantWriteReadLock并没有直接实现Lock接口，而是有两个ReadLock和WriteLock属性实现Lock，静态内部类Sync继承AQS，NonfairSync对应非公平模式，FairSync对应公平模式。
![](http://photos.zhangzemiao.com/blog_writereadlock1.jpg)

## 加锁
锁状态在AQS实现，为整型变量(32位)。允许写锁，那么cas直接加1操作，就是说写锁在低段位。允许读锁，cas设置为c+SHARED_UNIT。SHARED_UNIT为2的16次方，就是说读锁计数在高段位。

## 读锁获取分析
想画流程图，发现流程稍复杂，尝试用文字理清一下。lock方法，调用AQS的acquireShared(1)方法（这个是模版方法，尝试获取共享锁，获取即返回，不能获取则加入阻塞队列，使用LockSupport.park阻塞）。回调ReentrantReadWriteLock.tryAcquiredShared(1)方法，
- 如果有排他锁(即写锁)且非已获锁线程，返回-1，表示当前线程获锁失败，回到AQS模板方法，调用doAcquiredShared(1)，AQS添加共享结点，阻塞。
- 关键方法，readerShouldBlock()，非公平和公平模式不同实现，在公平模式下，如果AQS阻塞队列不为空，且当前线程非阻塞线程，则阻塞。在非公平模式下，如果AQS的阻塞队列中，第一个阻塞结点非共享结点，也就是现在有写锁阻塞，则读锁需要阻塞。
- 如果阻塞，还会调用ReentrantReadWriteLock.Sync.fullTryAcquireShared方法，再次尝试获取一次。有写锁且非已获锁线程，则返回。这里类中使用了HoldCounter+ThreadLock，每个读线程都有个HoldCounter，记录一个count，当这个count为0时，readerShouldBlock()阻塞后，不会cas。但是如果count不为0，只要不超过Max Count。会继续cas操作，并更新HoldCount对象。

## 读锁释放分析

第一个读锁线程会单独处理。获取当前线程的HoldCounter，count减一，cas(c-SHARED_UNIT)。

## 写锁获取分析

都是套路，跟ReentrantLock一样，调用AQS的acquire(1)方法，再回调ReentrantReadWriteLock.Sync.tryAcquire(1)方法，如果锁状态不为0，
- 有读锁，且无写锁，返回false。
- 有写锁且非已获锁线程，返回false。
- 如果加锁后，超额MAX_COUNT，抛出Error。
- 最后cas(c+acquires)
如果锁状态为空，调用writerShouldBlock()，公平模式下，有阻塞队列且非阻塞线程，则阻塞写锁。非公平模式下，always false，不会阻塞。如果阻塞，进行cas操作。

## 写锁释放分析
调用AQS的release方法，回调tryRelease方法，cas操作，释放所有重入的则返回true，没有则返回false。如果释放完，则unpark阻塞线程。

超时获取锁，就不再分析了，跟ReentrantLock差不多，ReentrantWriteReadLock与ReentrantLock的差别，基本清楚了。


