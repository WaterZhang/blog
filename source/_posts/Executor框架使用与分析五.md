---
title: Executor框架使用与分析五
toc: true
date: 2017-01-22 17:11:58
tags:
- Java
categories:
- Java并发编程
---

继续介绍Executor框架，第五篇。

[第一篇](/Java并发编程/Executor框架使用与分析一)介绍newCachedThreadPool
[第二篇](/Java并发编程/Executor框架使用与分析二)介绍newFixedThreadPool
[第三篇](/Java并发编程/Executor框架使用与分析三)介绍newFixedThreadPool以及Future接口
[第四篇](/Java并发编程/Executor框架使用与分析四)介绍ScheduledThreadPoolExecutor执行延时任务

这篇介绍以及分析使用ScheduledThreadPoolExecutor执行周期性任务。

## 样例
执行周期性任务，先看样例

一个简单的Runnable任务类
~~~java
public class RunnableTask implements Runnable {
	
	private String name;
	
	public RunnableTask(String name){
		this.name = name;
	}

	@Override
	public void run() {
		System.out.printf("%s: Starting at : %s\n",name,new Date());
		System.out.println("Hello, world");
	}
	
}
~~~

Main测试方法
~~~java
public static void main(String[] args) {
    ScheduledExecutorService executor=	Executors.newScheduledThreadPool(1);
    System.out.printf("Main: Starting at: %s\n",new Date());

    RunnableTask task=new RunnableTask("Task");
    ScheduledFuture<?> result= executor.scheduleAtFixedRate(task, 1, 2, TimeUnit.SECONDS);
    for (int i = 0; i < 5; i++) {
        System.out.printf("Main: Delay: %d\n", result.getDelay(TimeUnit.MILLISECONDS));
        // Sleep the thread during 500 milliseconds.
        try {
            TimeUnit.MILLISECONDS.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    for (int i = 0; i < 5; i++) {
        System.out.printf("Main: Delay: %d\n", result.getDelay(TimeUnit.MILLISECONDS));

    }

    executor.shutdown();
    try {
          TimeUnit.SECONDS.sleep(5);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }

    System.out.printf("Main: Finished at: %s\n",new Date());

}
~~~


## 源码分析
可以看出，测试代码跟第四篇文章，是类似的。在执行任务时，所用的接口不一样。但是，都使用的是ScheduledExecutorService接口所提供的方法。
~~~java
public interface ScheduledExecutorService extends ExecutorService {
	public ScheduledFuture<?> schedule(Runnable command,
                                       long delay, TimeUnit unit);
	public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay, TimeUnit unit);
	//这篇文章要分析的方法
	public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit);
	public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit);
｝
~~~

### 创建Executor
~~~java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}

public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());
}
~~~

### 执行Runnable任务
Executors内部使用的是ScheduledThreadPoolExecutor，对外接口是ScheduledExecutorService。

ScheduledThreadPoolExecutor.java
~~~java
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                              long initialDelay,
                                              long period,
                                              TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    if (period <= 0)
        throw new IllegalArgumentException();
    ScheduledFutureTask<Void> sft =
        new ScheduledFutureTask<Void>(command,
                                      null,
                                      //延时时间，转换成Nanos
                                      triggerTime(initialDelay, unit),
                                      //周期性时间
                                      unit.toNanos(period));
    RunnableScheduledFuture<Void> t = decorateTask(command, sft);
    sft.outerTask = t;
    delayedExecute(t);
    return t;
}
~~~

**不支持null 任务，抛空指针异常**
**不支持周期性时间为负，抛非法参数异常**
将Runnable任务封装成ScheduledFutureTask，跟第四篇分析延时任务一样。

delayedExecute方法，不再分析，可以看第四篇分析文章。周期性任务与延时任务的差别在于，多了一个period时间，本身Executor框架的执行逻辑是一样的。
~~~java
private void delayedExecute(RunnableScheduledFuture<?> task) {
    if (isShutdown())
        reject(task);
    else {
        super.getQueue().add(task);
        if (isShutdown() &&
            !canRunInCurrentRunState(task.isPeriodic()) &&
            remove(task))
            task.cancel(false);
        else
            ensurePrestart();
    }
}
~~~

这里使用的是DelayedWorkQueue，添加任务不阻塞。Worker线程运行起来后，会轮询从队列中获取任务来执行。这里的逻辑依旧沿用ThreadPoolExecutor的逻辑，是一样的。区别在于任务本身的不同。这里的任务是前面提及的ScheduledFutureTask，由于是周期性任务，那么run方法的执行，跟延时任务是不同的，调用runAndReset方法，那么执行成功，会设置更新下一次的运行时间，以及重新添加task至队列。意味着在执行构成中，又添加了一个task。那么worker线程会一直执行下去。
~~~java
public void run() {
	//这里是true
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
~~~

FutureTask.java，这里运行正常返回true，如果运行过程中，遇到异常，返回false。
~~~java
protected boolean runAndReset() {
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return false;
    boolean ran = false;
    int s = state;
    try {
        Callable<V> c = callable;
        if (c != null && s == NEW) {
            try {
                c.call(); // don't set result
                ran = true;
            } catch (Throwable ex) {
                setException(ex);
            }
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
    return ran && s == NEW;
}
~~~

ScheduledThreadPoolExecutor.ScheduledFutureTask，更新一下task的time时间，也就是延时的时间。
~~~java
private void setNextRunTime() {
    long p = period;
    if (p > 0)
        time += p;
    else
        time = triggerTime(-p);
}
~~~

ScheduledThreadPoolExecutor.java，将更新后的任务再次添加至队列中。
~~~java
void reExecutePeriodic(RunnableScheduledFuture<?> task) {
    if (canRunInCurrentRunState(true)) {
        super.getQueue().add(task);
        if (!canRunInCurrentRunState(true) && remove(task))
            task.cancel(false);
        else
            ensurePrestart();
    }
}
~~~


### 关闭Executor
如果你想要任务一直运行下去，就不需要关闭Executor。

## 总结
在运行周期性任务（最原始的任务，样例中就是RunnableTask）时，如果任务抛出异常，那么会影响后续的执行（周期性任务不会执行了），但是Executor不会因为异常关闭。
对于ScheduledThreadPoolExecutor，我们还有一个接口，没有分析，跟本次分析的区别，在于构造ScheduledFutureTask的不同，以及run方法中执行的细节区别。嗯，不再举个栗子分析了。其实就是setNextRunTime方法中的差别。

~~~java
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                 long initialDelay,
                                                 long delay,
                                                 TimeUnit unit);
~~~