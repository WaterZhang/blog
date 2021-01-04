---
title: Executor框架使用与分析三
toc: true
date: 2017-01-10 16:08:50
tags:
- Java
categories:
- Java并发编程
---

继续介绍Executor框架，第三篇。

[第一篇](/Java并发编程/Executor框架使用与分析一)介绍newCachedThreadPool
[第二篇](/Java并发编程/Executor框架使用与分析二)介绍newFixedThreadPool

前两篇文章中，Executor执行的任务（Runnable接口），没有返回结果。这么会用到Future接口和Callable接口。

## Future
Future接口，子类有FutureTask，ForkJoinTask的子类，以及1.8中新加的CompletableFuture。
顾名思义，Future表示一种未来的结果，在调用executor方法时，返回Future接口，此时的结果可能并未准备好。
~~~java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
~~~

## 样例
阶乘运算

FactorialCalculator.java
~~~java
//实现Callable接口，阶乘计算类
public class FactorialCalculator implements Callable<Integer> {
	
	private Integer number;

	public FactorialCalculator(Integer number) {
		this.number = number;
	}

	@Override
	public Integer call() throws Exception {
		int result = 1;
		if ((number == 0) || (number == 1)) {
			result = 1;
		} else {
			for (int i = 2; i <= number; i++) {
				result *= i;
				TimeUnit.MILLISECONDS.sleep(20);
			}
		}
		System.out.printf("%s: %d\n",Thread.currentThread().getName(),result);
		return result;
	}

}
~~~

Main.java
~~~java
public static void main(String[] args) {
		//创建Executor
		ThreadPoolExecutor executor = (ThreadPoolExecutor)
				Executors.newFixedThreadPool(2);
		List<Future<Integer>> resultList=new ArrayList<>();
		Random random=new Random();
		for (int i=0; i<10; i++){
		      Integer number= random.nextInt(10);
		      FactorialCalculator calculator=new FactorialCalculator(number);
              //executor提交Callable任务，并返回Future接口的结果
		      Future<Integer> result=executor.submit(calculator);
		      resultList.add(result);
		}
        //等待Future的结果准备好
		do {
			System.out.printf("Main: Number of Completed Tasks: %d\n",executor.getCompletedTaskCount());
			for (int i = 0; i < resultList.size(); i++) {
				Future<Integer> result = resultList.get(i);
				System.out.printf("Main: Task %d: %s\n", i, result.isDone());
			}
			try {
				TimeUnit.MILLISECONDS.sleep(50);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			
		} while (executor.getCompletedTaskCount() < resultList.size());
		
        //打印结果
		System.out.printf("Main: Results\n");
	    for (int i=0; i<resultList.size(); i++) {
	      Future<Integer> result=resultList.get(i);
	      Integer number=null;
	      try {
	        number=result.get();
	      } catch (InterruptedException e) {
	        e.printStackTrace();
	      } catch (ExecutionException e) {
	        e.printStackTrace();
	      }
	      System.out.printf("Main: Task %d: %d\n",i,number);
	    }
	    executor.shutdown();
	}
~~~

## 代码分析

### 创建Executor
这部分，可以看第二篇分析文章，使用的就是newFixedThreadPool

### 执行任务
这里，与之前分析的不一样。这里提交的任务是Callable接口。
**不接受 null 任务，抛空指针异常**

AbstractExecutorService.java
~~~java
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    //将Callable接口转成RunnableFuture接口了，该接口即使Runnable，也是Future
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}

protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new FutureTask<T>(callable);
}
~~~

可以看出，最终运行的task是FutureTask，而Executor把它当做Runnable接口来运行。来看看FutureTask做了些什么。
FutureTask.java
~~~java
//构造函数，初始状态为NEW
public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}

public void run() {
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        //执行Callable接口，两种情况，要么异常，要么有结果
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
~~~

如果Callable任务正常执行，那么调用set(Result)方法，设置本身的状态，保存计算结果，再调用finishCompletion()方法收尾，换新等待线程，并调用done()方法(这是个钩子方法，FutureTask是个空方法)。
~~~java
protected void set(V v) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = v;
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        finishCompletion();
    }
}

private void finishCompletion() {
    // assert state > COMPLETING;
    for (WaitNode q; (q = waiters) != null;) {
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            for (;;) {
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    LockSupport.unpark(t);
                }
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }

    done();

    callable = null;        // to reduce footprint
}
~~~

如果Callable任务执行抛出异常，那么调用setException(ex)方法，设置最终状态，输出对象是异常对象。
~~~java
protected void setException(Throwable t) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = t;
        UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
        finishCompletion();
    }
}
~~~

根据FutureTask本身的状态，看看get方法的返回，
~~~java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    //一直等
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    return report(s);
}

//根据状态，返回结果
private V report(int s) throws ExecutionException {
    Object x = outcome;
    //正常，返回计算结果
    if (s == NORMAL)
        return (V)x;
	//取消或者中断，则抛出取消异常
    if (s >= CANCELLED)
        throw new CancellationException();
	//都不是，则是执行中遇到的异常
    throw new ExecutionException((Throwable)x);
}
~~~

Executor运行的，跟第二篇是一样的，不再重复。这里Callable的任务执行，由FutureTask来包装，支持Future功能。

### 关闭Executor

设置SHUTDOWN状态，中断所有Worker线程，返回。

## 总结

对于Executor，有结果和没有结果的任务，都分析了。后续看看支持调度的Executor以及1.8新加的。