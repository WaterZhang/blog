---
title: Executor框架使用与分析一
toc: true
date: 2016-12-20 14:44:46
tags:
- Java
categories:
- Java并发编程
---

介绍Executor框架。

## 基本框架介绍

顶层接口Executor，ExecutorService继承Executor，AbstractExecutorService继承ExecutorService，ThreadPoolExecutor继承抽象ExecutorService。还有ScheduledThreadPoolExecutor。
Java API还提供的工具类Executors，帮我们创建不同的Executor实现类，针对不同的应用场景。
- FixedThreadPool
- SingleThreadExecutor
- newCachedThreadPool
- newScheduledThreadPool
- newWorkStealingPool（1.8新加的）

我们将分析这几种情况，以及相关源码分析。

![](http://photos.zhangzemiao.com/blog_executor.jpg)

Executor
~~~java
public interface Executor {
    void execute(Runnable command);
}

public interface ExecutorService extends Executor {

    void shutdown();
    List<Runnable> shutdownNow();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
    <T> Future<T> submit(Callable<T> task);
    <T> Future<T> submit(Runnable task, T result);
    Future<?> submit(Runnable task);
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}

public abstract class AbstractExecutorService implements ExecutorService{
...
}

public class ThreadPoolExecutor extends AbstractExecutorService {
...
}
~~~

## 样例

### 任务
~~~java
public class Task implements Runnable {

	private Date initDate;
	private String name;

	public Task(String name) {
		initDate = new Date();
		this.name = name;
	}

	@Override
	public void run() {
		System.out.printf("%s: Task %s: Created on: %s\n", Thread.currentThread().getName(), name, initDate);
		System.out.printf("%s: Task %s: Started on: %s\n", Thread.currentThread().getName(), name, new Date());
		try {
			Long duration = (long) (Math.random() * 10);
			System.out.printf("%s: Task %s: Doing a task during %d seconds\n", Thread.currentThread().getName(), name,
					duration);
			TimeUnit.SECONDS.sleep(duration);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}

		System.out.printf("%s: Task %s: Finished on: %s\n", Thread.currentThread().getName(), name, new Date());
	}

}
~~~

### Executor
~~~java
public class Server {
	
	private ThreadPoolExecutor executor;
	
	public Server() {
		executor = (ThreadPoolExecutor) Executors.newCachedThreadPool();
	}
	
	public void executeTask(Task task){
	    System.out.printf("Server: A new task has arrived\n");
	    executor.execute(task);
	    System.out.printf("Server: Pool Size: %d\n",executor.getPoolSize());
	    System.out.printf("Server: Active Count: %d\n",executor.getActiveCount());
	    System.out.printf("Server: Completed Tasks: %d\n",executor.getCompletedTaskCount());
	}
	
	public void endServer() {
		executor.shutdown();
	}

}
~~~

### 测试类
~~~java
public class Main {

	public static void main(String[] args) {
		Server server = new Server();
		for (int i = 0; i < 100; i++) {
			Task task = new Task("Task " + i);
			server.executeTask(task);
		}
		server.endServer();
	}

}
~~~

## 总结
