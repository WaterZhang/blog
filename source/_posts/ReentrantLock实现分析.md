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

## ReentrantLock的成员变量Sync

Sync是ReentrantLock类中定义的静态抽象内部类(abstract static Sync extends AbstractQueuedSynchronizer)，并且有两个实现的子类（静态内部类），分别对应着公平锁的实现（FairSync）和非公平锁的实现（NonFairSync）。同时有两个构造函数，对应着两个锁，无参构造是默认的非公平锁，带boolean的构造函数，true为公平锁，false为非公平锁。类似代理模式还是装饰者模式？（后续回顾设计模式时，解疑！）。
ReentrantLock.lock -> 成员变量sync.lock() -> FairSync.lock() | NonfairSync.lock()。
