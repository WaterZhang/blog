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
直接调用Sync.nonfairTryAcquire方法，这方法的功能，上面已介绍。
5. tryLock(long , TimeUnit)调用过程
调用AQS的tryAcquireNanos方法，支持中断。回调tryAcquire(1)方法，再调nonfairTryAcquire(1)方法，已介绍此方法。如果获取锁失败，tryAcquire返回false，调用AQS的doAcquireNanos方法，这里会设置基于传入的时间长度，设置deadline。最终会调用LockSupport.parkNanos方法，超时阻塞线程。4种方法唤醒该线程，其他线程调用unpark，其他线程中断此线程，等待足够的时间，spuriously return(no reason)这是API的说明，不是太懂。
6. unlock()调用过程
调用AQS的release(1)方法，再回调Sync.tryRelease(1)方法，如果释放锁的线程与占有锁的线程不一致，抛IllegalMonitorStateException异常。清除当前占有锁线程，CAS锁状态。释放成功，调用AQS的unparkSuccessor()方法。
7. newCondition()调用
调用Sync.newCondition方法，创建ConditionObject对象。

公平方式的实现，
1. lock()方法调用过程
调用FairSync的lock方法，再调AQS的acquire(1)方法，回调FairSync.tryAcquire(1)方法，做两件事，一是发未锁且当前线程和AQS阻塞的队列的第一个阻塞线程为同一线程，则cas锁状态，返回true。二是已锁且当前线程和锁线程是同一个，cas锁状态（允许重入），返回true。两件事都不满足，则返回false。当返回false时，调用acquireQueued方法，上面非公平锁已介绍，调用LockSupport.park阻塞线程。
2. unlock()方法调用过程
跟非公平锁实现一样。

公平锁和非公平锁的区别在于tryAcquire()方法，不在详细描述。至于未说明的细节，比如AQS维护阻塞队列的代码，看得不是很明白。至此ReentrantLock的大部分源码已经研究了一片。核心还是AQS，所以需要花时间和心思研究AQS。



