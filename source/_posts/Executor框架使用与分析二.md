---
title: Executor框架使用与分析二
toc: true
date: 2016-12-29 16:09:52
tags:
- Java
categories:
- Java并发编程
---

继续介绍Executor框架，这是第二篇。

在[第一篇](/Java并发编程/Executor框架使用与分析一)的基础上，修改Server类的构造函数，使用newFixedThreadPool创建Executor。
~~~java
public Server() {
//		executor = (ThreadPoolExecutor) Executors.newCachedThreadPool();
		executor = (ThreadPoolExecutor)Executors.newFixedThreadPool(5);
	}
~~~

## 创建ThreadPool

Executors.java
~~~java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
~~~

**这里，coreSize和maxSize都是设定的值，然后阻塞队列使用的LinkedBlockingQueue，跟newCachedThreadPool使用SyschronousQueue，不一样。**

ThreadPoolExecutor.java
~~~java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         Executors.defaultThreadFactory(), defaultHandler);
}
~~~

## 执行任务

代码路径跟上一篇分析是一样的。执行execute方法，添加任务，当前的worker count小于corePoolSize（这里的例子为5），调用addWorker方法。ctl自增一，创建Worker并启动。当worker count不小于corePoolSize时，则调用阻塞队列的offer方法，添加任务。
**execute方法不接受 null 任务，抛空指针异常**

Worker运行分析，Worker本身就是一个线程，启动后，调用ThreadPoolExecutor的runWorker方法，先执行firstTask，然后从队列拿任务，这里会一直阻塞（支持中断）获取task。返回task后，执行。如果返回task为null，那么启动worker退出机制，自减以及从workers中清除。

## 关闭Executor

设置成SHUTDOWN状态，中断阻塞的worker（等待获取任务的线程），执行完正在进行的任务，worker线程正常退出。