---
title: Java中的锁
date: 2016-10-31 20:46:52
tags:
- Java
categories: Java并发编程
toc: true
---

在Java 1.5以后，提供了Lock接口，实现锁的功能，相比synchronized，显示的进行锁操作，并提供更多的灵活的功能。

## Lock接口
提供与synchronized关键字类似的同步功能，可以显示地获取和释放所，尝试非阻塞地获取锁，能被中断地获取锁（支持中断操作），超时获取锁（超时退出）等。
***
Lock lock = new xxx;
lock.lock();
try{
 ...
｝finally｛
 lock.unlock();
｝
***

Lock接口声明的方法：
- void lock()，阻塞获取锁。
- void lockInterruptibly() throws InterruptedException，阻塞获取锁，但是支持中断，如果线程中断，也可返回，抛出InterruptedException。
- boolean tryLock()，非阻塞获取锁，能立马获取锁，则返回true，不能立马获取锁，立刻返回false。
- boolean tryLock(long time, TimeUnit unit) throws InterruptedException，非阻塞，超时等待获取锁，支持中断返回（抛异常）。
- void unlock()，释放锁。
- Condition newCondition()，创建与该锁绑定的条件对象。


Lock接口，在Java 1.8中有5个实现类，
- ReentrantLock
- ReentrantReadWriteLock.ReadLock
- ReentrantReadWriteLock.WriteLock
- StampedLock.ReadLockView
- StampedLock.WriteLockView


## 队列同步器
**AbstractQueuedSynchronized**，用来构建锁和其他同步组建的基础框架，int变量表示同步状态，内置FIFO队列完成资源获取线程排队工作。

锁是面向使用者，定义了使用者与锁交互的接口，隐藏实现细节。
同步器是面向锁的实现者，简化了锁的实现，屏蔽了同步状态管理，线程的排队，等待与唤醒等底层操作。
AQS提供模板方法，具体实现锁的类需要重写方法：
tryAcquire(int arg)
tryRelease(int arg)
tryAcquireShared(int arg)
tryReleaseShared(int arg)
isHeldExclusively()

模板方法提供独占式获取和释放同步状态，共享式获取和释放同步状态和查询同步队列中的等待线程情况。

## AQS的实现分析
同步队列，
- FIFO双向队列
- 有head和tail两个空结点指向头和尾
- 新阻塞的线程通过CAS操作插入队尾

独占式同步状态获取和释放，
[ReentrantLock实现分析](/Java并发编程/ReentrantLock实现分析)

共享式同步状态获取和释放，
这里需要补上代码分析。。。

独占式超时获取同步状态
[ReentrantLock实现分析](/Java并发编程/ReentrantLock实现分析)

## 重入锁ReentrantLock
支持重进入的锁，表示一个线程对资源的重复加锁，支持公平和非公平两种模式，默认为非公平。
公平，意味着等待时间最长的优先获取锁，能够减少“饥饿”发生的概率。
非公平，不会有限制，效率高，可以减少线程之间的切换操作。

## 读写锁
排他锁，同一时刻，只允许一个线程进行访问，比如重入锁。
非排他锁，同一时刻，允许多个线程访问，比如读写锁。
高段位读，低段位写。

## LockSupport
park()，阻塞当前县城，如果调用unpark方法或者中断当前线程，才能从park方法返回。
parkNanos(long nanos)，阻塞当前线程，最长不超过nanos纳秒，在park方法返回条件基础上，加上超时返回。
parkUntil(long deadline)，阻塞当前线程，直到deadline时间。
unpark(Thread thread)，唤醒出于阻塞的线程。
Java 6加入了新的参数Object blocker阻塞对象，方便问题排查和系统监控。

## Condition
Java对象中，默认在Object对象上，提供等待唤醒机制，Condition是Lock接口提供的类似监视器，提供await()/ signal()，还有中断不敏感，超时等待接口。
在Object的监视器模型上，一个对象拥有一个同步队列和等待队列。在Lock中，拥有一个同步队列和多个等待队列。
