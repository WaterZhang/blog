---
title: Executor框架使用与分析四
toc: true
date: 2017-01-12 14:20:51
tags:
- Java
categories:
- Java并发编程
---

继续介绍Executor框架，第四篇。

[第一篇](/Java并发编程/Executor框架使用与分析一)介绍newCachedThreadPool
[第二篇](/Java并发编程/Executor框架使用与分析二)介绍newFixedThreadPool
[第三篇](/Java并发编程/Executor框架使用与分析三)介绍newFixedThreadPool以及Future接口

这篇开始介绍ScheduledThreadPoolExecutor。

## 样例
有时候我们想延时执行一些任务，看样例。

一个简单的继承Callable接口的任务
~~~java
public class CallableTask implements Callable<String> {
	
	private String name;
	
	public CallableTask(String name){
		this.name = name;
	}

	@Override
	public String call() throws Exception {
		System.out.printf("%s: Starting at : %s\n",name,new Date());
	    return "Hello, world";
	}

}
~~~

Main.java
~~~java
public static void main(String[] args) {
	//使用Scheduled
    ScheduledThreadPoolExecutor executor=(ScheduledThreadPoolExecutor)
            Executors.newScheduledThreadPool(1);
    System.out.printf("Main: Starting at: %s\n",new Date());
    for (int i = 0; i < 5; i++) {
        CallableTask task = new CallableTask("Task " + i);
        //schedule方法
        executor.schedule(task, i + 10, TimeUnit.SECONDS);
    }

    executor.shutdown();
    try {
        executor.awaitTermination(1, TimeUnit.DAYS);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.printf("Main: Ends at: %s\n",new Date());

~~~

## 源码分析
本身例子很简单，使用SchedudThreadPoolExecutor执行一些延时任务，透过表面看本质，看看背后的实现和设计。

### 创建Executor

Executors.java
~~~java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
~~~

ScheduledThreadPoolExecutor.java
只有一个参数设置corePoolSize。
**这里，阻塞队列使用的是DelayedWorkQueue。**
~~~java
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());
}
~~~

### 执行Callable任务

这里是schedule方法的源码，triggerTime方法，记录执行任务的时间点。
ScheduledFutureTask继承于FutureTask（在上一篇中，分析了FutureTask背后的原理，可以回头看看），包装任务和任务执行的时间点，在ScheduledFutureTask构造函数中，还有一个sequencer，记录序列号，具体什么用，往下看。
最外层还是一个decorateTask方法，再进行包装，代码有点怪怪的。。。可继承，应该是关键，
最后执行delayedExecute方法。

~~~java
public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay,
                                           TimeUnit unit) {
    if (callable == null || unit == null)
        throw new NullPointerException();
    RunnableScheduledFuture<V> t = decorateTask(callable,
        new ScheduledFutureTask<V>(callable,
                                   triggerTime(delay, unit)));
    delayedExecute(t);
    return t;
}

private long triggerTime(long delay, TimeUnit unit) {
    return triggerTime(unit.toNanos((delay < 0) ? 0 : delay));
}

long triggerTime(long delay) {
    return now() +
        ((delay < (Long.MAX_VALUE >> 1)) ? delay : overflowFree(delay));
}

//构造函数
ScheduledFutureTask(Callable<V> callable, long ns) {
    super(callable);
    this.time = ns;
    this.period = 0;
    this.sequenceNumber = sequencer.getAndIncrement();
}

protected <V> RunnableScheduledFuture<V> decorateTask(
    Callable<V> callable, RunnableScheduledFuture<V> task) {
    return task;
}

~~~

重点分析delayedExecute方法，

~~~java
private void delayedExecute(RunnableScheduledFuture<?> task) {
	//如果是关闭状态，拒绝任务
    if (isShutdown())
        reject(task);
    else {
    	//这里是DelayedWorkQueue，非阻塞添加
        super.getQueue().add(task);
        //再次检查是否关闭executor，以及判断是非周期性任务，且清除任务成功，则取消任务
        if (isShutdown() &&
            !canRunInCurrentRunState(task.isPeriodic()) &&
            remove(task))
            task.cancel(false);
        //正常运行状态，执行该路径
        else
            ensurePrestart();
    }
}

//添加Worker线程，并启动Worker线程，最后又回到runWorker方法，之前分析过
void ensurePrestart() {
    int wc = workerCountOf(ctl.get());
    if (wc < corePoolSize)
        addWorker(null, true);
    else if (wc == 0)
        addWorker(null, false);
}
~~~

**这里的task是封装过后的ScheduledFutureTask。**
在runWorker方法中，worker线程阻塞从队列拿去元素（即ScheduledFutureTask），然后执行run方法。这里使用DelayedWorkQueue，
简单分一下DelayedWorkQueue的take方法，也就是worker线程的阻塞的方法，

ScheduledThreadPoolExecutor.DelayedWorkQueue
~~~java
public RunnableScheduledFuture<?> take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
        	//拿第一个元素
            RunnableScheduledFuture<?> first = queue[0];
            //队列为空，worker线程阻塞等待
            if (first == null)
                available.await();
            else {
            	//获取任务执行，还需要delay的时间
                long delay = first.getDelay(NANOSECONDS);
                //如果小于等于0，则返回给worker线程
                if (delay <= 0)
                    return finishPoll(first);
                first = null; // don't retain ref while waiting
                //初始状态，leader是为null的，先过
                //再看，多个Worker线程对于leader是竞争关系，当有其他worker设置leader线程时，阻塞当前worker线程
                if (leader != null)
                    available.await();
                else {
                	//这里赋值给leader线程，让worker线程超时等待所需等待的时间
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        available.awaitNanos(delay);
                    } finally {
                    	//最后leader赋值为null
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
    	//最后返回后，如果队列不为空，则唤醒阻塞的worker线程
        if (leader == null && queue[0] != null)
            available.signal();
        lock.unlock();
    }
}
~~~

ScheduledThreadPoolExecutor.ScheduledFutureTask
~~~java
//继承FutureTask，并重写
public void run() {
	//是否为周期性任务，这里是false
    boolean periodic = isPeriodic();
    //这里的代码不会执行
    if (!canRunInCurrentRunState(periodic))
        cancel(false);
	//执行父类的run方法，即FutureTask的run方法，调用最初的Callable接口的任务
    else if (!periodic)
        ScheduledFutureTask.super.run();
    else if (ScheduledFutureTask.super.runAndReset()) {
        setNextRunTime();
        reExecutePeriodic(outerTask);
    }
}
~~~

至此，对于ScheduledThreadPoolExecutor怎么执行延时任务，已基本清除。在ThreadPoolExecutor基础上，继承FutureTask，增加周期性和延时运行的属性。再结合DelayedWorkQueue，获取ScheduledFutureTask任务，最终回到FutureTask，调用原始的被封装的任务。

这里有个小细节，在ScheduledFutureTask的run方法中，
~~~java
public void run() {
    boolean periodic = isPeriodic();
    if (!canRunInCurrentRunState(periodic))
        cancel(false);
    else if (!periodic)
        ScheduledFutureTask.super.run();
    else if (ScheduledFutureTask.super.runAndReset()) {
        setNextRunTime();
        reExecutePeriodic(outerTask);
    }
}

这个方法决定是否取消任务，对于延时任务，periodic是false。是否取消，由executeExistingDelayedTasksAfterShutdown来决定，默认是ture。意味着，即使我们调用了shutdown方法，是允许任务继续执行的。如果你不想，可以将executeExistingDelayedTasksAfterShutdown设置成false。
boolean canRunInCurrentRunState(boolean periodic) {
    return isRunningOrShutdown(periodic ?
   	continueExistingPeriodicTasksAfterShutdown :
    executeExistingDelayedTasksAfterShutdown);
}
~~~

### 关闭Executor

沿用ThreadPoolExecutor的shutdown逻辑，不再分析。

## 总结

分析源码后，弄清楚ScheduledThreadPoolExecutor怎么支持延时任务，后续再分析支持周期性任务，就容易多了。
**延时任务，延时是相对于添加任务的时间点。**
