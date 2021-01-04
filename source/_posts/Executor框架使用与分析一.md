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
    	//使用Executors创建cached thread pool
		Server server = new Server();
		for (int i = 0; i < 100; i++) {
			Task task = new Task("Task " + i);
            //执行任务
			server.executeTask(task);
		}
        //关闭server
		server.endServer();
	}

}
~~~

## 源码分析
根据源码，代码分为三步，
- initial
- executeTask
- shutDown

### 创建ThreadPool
这里使用的是CachedThreadPool，corePoolSize为0，max size为Integer最大值。线程存好时间是60秒。队列使用SynchronousQueue。
Executors.java
~~~java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
~~~
ThreadPoolExecutor.java
~~~java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue) {
	//默认线程工厂，handler
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         Executors.defaultThreadFactory(), defaultHandler);
}

public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
~~~

### 执行任务
ThreadPoolExecutor.java
~~~java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();

    int c = ctl.get();
    //这里，计算worker count，初始状态是0，但是corePoolSize也是为0，不执行里面代码
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    //初始状态isRunning返回true, offer返回false,对于该样例，不会这里的代码
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    //创建Worker，并启动Worker
    else if (!addWorker(command, false))
        reject(command);
}
~~~

~~~java
private boolean addWorker(Runnable firstTask, boolean core) {
	//这里使用了goto
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

		//workQueue.isEmpty永远返回true
        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
        	//这里wc初始值为0
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
			//CAS ctl自增1，成功后跳出goto，失败后retry
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

	//创建Worker
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            //创建成功后，执行Worker这个线程
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
~~~

现在进入Worker执行流程，
ThreadPoolExecutor.Worker
~~~java
Worker(Runnable firstTask) {
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;
    this.thread = getThreadFactory().newThread(this);
}


//这里回调ThreadPoolExecutor中的runWorker方法
public void run() {
    runWorker(this);
}
~~~

~~~java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    //firstTask就是要执行的任务
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
            	//这里定义了一些模板方法，是空的，方便继承ThreadPoolExecutor时，重写
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                	//执行，这里要注意，是直接调用run方法，而不是当做线程来执行
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
    	//执行完后的处理
        processWorkerExit(w, completedAbruptly);
    }
}
~~~

来看看如果worker执行完后的处理，
~~~java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
	//正常情况是false，不执行
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
		//这里ctl自减1，直到成功
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
		//记录完成的任务的个数
        completedTaskCount += w.completedTasks;
        //这里清除掉worker
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    tryTerminate();

    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
			//这里min等于0，
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
			//workQueue一直返回true
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
			//执行这里的代码
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}
~~~

到这里，CachedThreadPool的执行过程，分析了一篇，来一个task，就启动一个Worker来执行。

### 关闭
shutdown方法不会阻止已经运行的任务，但是也不接受新的任务了。
~~~java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        //设置成shutdown状态
        advanceRunState(SHUTDOWN);
        //中断worker线程
        interruptIdleWorkers();
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}
~~~


## 总结

这是第一篇分析Executor框架，目前只简单涉及CachedThreadPool的使用。嗯，看看SynchronousQueue这个队列。