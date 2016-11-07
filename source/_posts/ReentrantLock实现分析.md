---
title: ReentrantLock实现分析
date: 2016-11-02 21:33:22
tags: Java
categories: Java并发编程
toc: true
---

ReentrantLock，实现Lock接口，重入锁，少废话，开始分析。

Lock接口声明的方法：
- void lock()，阻塞获取锁。
- void lockInterruptibly() throws InterruptedException，阻塞获取锁，但是支持中断，如果线程中断，也可返回，抛出InterruptedException。
- boolean tryLock()，非阻塞获取锁，能立马获取锁，则返回true，不能立马获取锁，立刻返回false。
- boolean tryLock(long time, TimeUnit unit) throws InterruptedException，非阻塞，超时等待获取锁，支持中断返回（抛异常）。
- void unlock()，释放锁。
- Condition newCondition()，创建与该锁绑定的条件对象。

## ReentrantLock的类图

![](http://photos.zhangzemiao.com/blog_reentrant1.jpg)

Sync是ReentrantLock类中定义的静态抽象内部类(abstract static Sync extends AbstractQueuedSynchronizer)，并且有两个实现的子类（静态内部类），分别对应着公平锁的实现（FairSync）和非公平锁的实现（NonFairSync）。同时有两个构造函数，对应着两个锁，无参构造是默认的非公平锁，带boolean的构造函数，true为公平锁，false为非公平锁。类似代理模式还是装饰者模式？（后续回顾设计模式时，解疑！）。
ReentrantLock.lock -> 成员变量sync.lock() -> FairSync.lock() | NonfairSync.lock()。

NonfairSync和FairSync继承于Sync内部类，分别对应非公平策略和公平策略实现。在lock()实现上，有区别。

## ReentrantLock实现Lock接口分析

非公平为默认方式，
1. lock成功过程
具体实现由NonfairSync实现，直接调用AQS的compareAndSetState，如果返回为true，则设置锁的当前线程，并返回。
![](http://photos.zhangzemiao.com/blog_reentrant2.jpg)
2. lock阻塞过程
具体实现由NonfairSync实现，调用AQS的cas，设置失败，返回false，则调用AQS的acquire(1)，回调NonfairSync的tryAcquire(1)，具体有Sync.nonfairTryAcquire(1)实现，获取锁状态，如果c为0，则再次cas，成功则返回true至acquire方法，并调用中断方法，从阻塞处返回。Sync.nonfairTryAcquire(1)中获取锁状态，c不为0时，判断是否获取锁的线程是否为当前线程，如果是，可以允许重入，并返回true。如果不是当前线程，则返回至acquire方法，继续调用acquireQueued方法，最终调用LockSupport.part方法，阻塞当前线程。
![](http://photos.zhangzemiao.com/blog_reentrant3.jpg)
3. lockInterruptibly()调用过程
直接调用AQS的acquireInterruptibly(1)方法，先判断是否中断，已中断则抛出异常。回调tryAcquire方法，再调Sync.nonfairTryAcquire方法，做两件事情，一拿锁状态，没锁则锁了，并返回。二锁了，判断当前线程是否为锁线程，是则更新状态，返回。这两件事都不成，则返回false。回到AQS的acquireInterruptibly方法，调用doAcquireInterruptibly(1)，最终调用LockSupport的park方法，阻塞线程。
![](http://photos.zhangzemiao.com/blog_reentrant4.jpg)
4. tryLock()调用过程
5. tryLock(long , TimeUnit)调用过程
6. unlock()调用过程
7. newCondition()调用

未完待续。。。

