---
title: Java并发编程指南
toc: true
date: 2017-12-01 00:00:00
tags:
- Java
categories:
- Java并发编程
---

这里汇总所有的并发编程的知识，整理出一个脉络清晰图，方便温故，查找。当然这只是开始，而非结束。

## 理论知识

[Java的内存模型](/Java并发编程/Java内存模型)
[Java的线程](/Java并发编程/Java线程)
[JAVA中的锁](/Java并发编程/Java中的锁)

## 源码分析

[重入锁ReentrantLock](/Java并发编程/ReentrantLock实现分析)，重入锁用来替代Synchronized关键字，提供更灵活的方式。
[读写锁ReentrantReadWriteLock](/Java并发编程/ReentrantReadWriteLock实现分析)，分析读操作和写操作。
[新的ConcurrentHashMap](/Java并发编程/ConcurrentHashMap实现分析与使用)，Java 1.8中，已没有所谓的分段锁设计了，不妨看看。
[Fork/Join分解无返回结果任务](/Java并发编程/Fork-Join框架使用与分析)，大任务化解小任务，不需要返回结果，不需要汇总。
[Fork/Join分解有返回结果任务](/Java并发编程/Fork-Join框架使用与分析二)，大任务化解小任务，有结果返回，需要汇总。
[CountDownLatch](/Java并发编程/CountDownLatch使用与分析)，并发流程控制，应付需要等人齐（线程或者说是参与者），才干活（一起要做的事）的场景。
[CyclicBarrier](/Java并发编程/CyclicBarrier使用与分析)，屏障点设置，（线程或者说是参与者）等等，大家一起上。屏障点可以重复设置。
[Executor中的CachedThreadPool](/Java并发编程/Executor框架使用与分析一)，无限制创建线程，大吞吐量，风险很大。
[Executor中的FixedThreadPool](/Java并发编程/Executor框架使用与分析二)，固定的线程池，吞吐量固定，不具有弹性。
[Executor中的有返回结果的任务以及Future接口](/Java并发编程/Executor框架使用与分析三)，Future接口以及FutureTask实现，处理有返回结果的任务。
[Executor中的ScheduledThreadPoolExecutor以及延时任务](/Java并发编程/Executor框架使用与分析四)，延时任务的执行机制。
[Executor中的ScheduledThreadPoolExecutor以及周期性任务](/Java并发编程/Executor框架使用与分析五)，周期性任务的执行机制。

