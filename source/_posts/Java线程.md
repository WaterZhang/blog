---
title: Java线程
date: 2016-10-31 20:21:45
tags:
- Java
categories: Java并发编程
toc: true
---

线程，是并发编程的基础，操作系统运行的最小单元，进程可以拥有线程，线程也称为轻量级进程，同一进程的线程是可以共享进程的内存的。

## 线程简介
为什么要使用线程？冯诺依曼运算体系，目前没啥大的突破，处理器不能升级，升级频率，升级内核数量，更快的响应的，提供更好的并发编程模型。

在Java中，线程是有优先级的，似乎很少提及，Thread设置后，操作系统可能不买账。
线程的状态，new、Runnable、Blocked、Waiting、Time_waiting、Terminated。

Daemon线程，如果Java虚拟机中不存在Daemon线程，则会退出。Daemon属性要在启动线程前设置，否则无效。

## 线程启动运行

创建线程，可以通过继承Thread类和实现Runnable接口，提供多样的构造函数，设置线程组，优先级，线程名以及ThreadLocalMap。
start方法启动线程，然后等待操作系统调度并执行run方法。
中断，interrupt()/中断位，可以理解线程有一个标识位，表示运行中的线程是否被其他线程进行了中断操作。线程通过检查自身是否被中断来进行响应，isInterrupted方法判断是否被中断。
在Java API中，有许多声明了抛出InterruptedException的方法，这些方法抛出异常之前，Java虚拟机先将该线程的中断位清除，然后抛出异常，所以抛出异常后，isInterrupted是返回的false。

Thread还有几个过期的方法，suspend，resume和stop方法，方法调用后，不会释放占有的资源（如锁），会导致程序不稳定，抛弃。

如果安全地终止线程？中断，合理的设置boolean变量控制。

## 线程间通信
volatile / synchronized / Lock，volatile保证内存可见性，synchronized，保证代码在同一时刻，只能有一个线程运行，排他性。

Java的父类Object提供等待/通知（唤醒）机制wait()/notify()，配合synchronized使用，进行同步控制。
等待方：
1. 获取对象锁
2. 如果条件不满足，调用对象的wait方法，被通知后仍要检查条件。
3. 条件满则则执行对应的逻辑。

通知方：
1. 获取对象的锁
2. 改变条件
3. 通知（所有）等待在对象上的线程

ThreadLocal，线程变量，空间换取时间，对于一些公共类，并且是非线程安全的，可以考虑使用。