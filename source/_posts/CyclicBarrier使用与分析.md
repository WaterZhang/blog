---
title: CyclicBarrier使用与分析
toc: true
date: 2016-12-19 10:34:47
tags:
- Java
categories:
- Java并发编程
---

介绍CyclicBarrier使用，以及分析实现原理。

## API描述

[Java 1.8的API](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CyclicBarrier.html)描述，允许一批线程到达一个共同屏障点（common barrier point）。在涉及需要线程需要相互等待的场景，可以考虑CyclicBarrier。屏障机制可以重复使用。

await()/ await(long timeout, TimeUnit unit)，线程等待，等待所有的线程都调用await方法。
getNumberWaiting()，返回当前已经等待的线程个数。
getParties()，返回需要的等待的线程个数，是构造函数的初始值，不会变。
isBroken()，是否为broken状态。
reset()，重置。

Barrier屏障，可能不好理解。可以理解设置一道关卡，先来的不能过，需要等“大家”都到了，才能一起过。

## 样例
矩阵中查找特定元素，这里都为int。

### 矩阵模拟
~~~java
public class MatrixMock {
	//二维数组，模拟矩阵
	private int data[][];
	
	public MatrixMock(int size, int length, int number){
		int counter=0;
	    data=new int[size][length];
	    Random random=new Random();
		for (int i = 0; i < size; i++) {
			for (int j = 0; j < length; j++) {
				data[i][j] = random.nextInt(10);
				if (data[i][j] == number) {
					counter++;
				}
			}
		}
		//打印输出有多少个查找元素
		System.out.printf("Mock: There are %d ocurrences of number in generated data.\n",counter,number); 
	}
	
	public int[] getRow(int row) {
		if ((row >= 0) && (row < data.length)) {
			return data[row];
		}
		return null;
	}

}
~~~

### 行结果
存储每行的查找数
~~~java
public class Results {
	//存储每行的查找个数
	private int data[];

	public Results(int size) {
		data = new int[size];
	}

	public void setData(int position, int value) {
		data[position] = value;
	}

	public int[] getData() {
		return data;
	}

}
~~~

### 屏障执行任务
当屏障抵达后，执行的任务，合并行结果

~~~java
public class Grouper implements Runnable{
	
	private Results results;

	public Grouper(Results results) {
		this.results = results;
	}
	
	@Override
	public void run() {
		int finalResult=0;
	    System.out.printf("Grouper: Processing results...\n");
	    int data[]=results.getData();
	    for (int number:data){
	      finalResult+=number;
	    }
	    System.out.printf("Grouper: Total result: %d.\n",finalResult);
	}

}
~~~

### 线程任务
5个线程任务，每个线程任务查找2000行，执行的run方法最后调用await方法，等待其他线程执行完成。

~~~java
public class Searcher implements Runnable {
	private int firstRow;
	private int lastRow;
	private MatrixMock mock;
	private Results results;
	private int number;
	private final CyclicBarrier barrier;

	public Searcher(int firstRow, int lastRow, MatrixMock mock, Results results, int number, CyclicBarrier barrier) {
		this.firstRow = firstRow;
		this.lastRow = lastRow;
		this.mock = mock;
		this.results = results;
		this.number = number;
		this.barrier = barrier;
	}
	
	@Override
	public void run() {
		int counter;
		System.out.printf("%s: Processing lines from %d to %d.\n", Thread.currentThread().getName(), firstRow, lastRow);
		for (int i = firstRow; i < lastRow; i++) {
			int row[] = mock.getRow(i);
			counter = 0;
			for (int j = 0; j < row.length; j++) {
				if (row[j] == number) {
					counter++;
				}
			}
			results.setData(i, counter);
		}
		System.out.printf("%s: Lines processed.\n", Thread.currentThread().getName());
		try {
        	//执行await方法
			barrier.await();
		} catch (InterruptedException e) {
			e.printStackTrace();
		} catch (BrokenBarrierException e) {
			e.printStackTrace();
		}

	}

}

~~~

### 测试
~~~java
public class Main {

	public static void main(String[] args) {
		final int ROWS=10000;
	    final int NUMBERS=1000;
	    final int SEARCH=5;
	    final int PARTICIPANTS=5;
	    final int LINES_PARTICIPANT=2000;
	    
	    MatrixMock mock=new MatrixMock(ROWS,NUMBERS,SEARCH);
	    Results results=new Results(ROWS);
	    Grouper grouper=new Grouper(results);
	    CyclicBarrier barrier=new CyclicBarrier(PARTICIPANTS,grouper);
	    Searcher searchers[]=new Searcher[PARTICIPANTS];
	    for (int i=0; i<PARTICIPANTS; i++){
	      searchers[i]=new Searcher(i*LINES_PARTICIPANT, (i*LINES_PARTICIPANT)+LINES_PARTICIPANT, mock, results, 5,barrier);
	      Thread thread=new Thread(searchers[i]);
	      thread.start();
	    }
	    System.out.printf("Main: The main thread has finished.\n");

	}

}
~~~

## 原理机制
没有直接是AQS，而是使用ReentrantLock以及Condition实现同步机制。

### 初始化
很简单，没什么可说的。
~~~java
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}
~~~

### 线程调用await方法
~~~java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}

public int await(long timeout, TimeUnit unit)
    throws InterruptedException,
           BrokenBarrierException,
           TimeoutException {
    return dowait(true, unit.toNanos(timeout));
}

//await方法的主要逻辑
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
           TimeoutException {
	//使用ReentrantLock
    final ReentrantLock lock = this.lock;
    //线程进来锁
    lock.lock();
    try {
        final Generation g = generation;
        //如果broken，抛异常
        if (g.broken)
            throw new BrokenBarrierException();
		//线程中断，抛异常，唤醒Condition的等待线程
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }

		//count自减
        int index = --count;
        //如果自减至0，执行BarrierAction
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run();
                ranAction = true;
                nextGeneration();
                return 0;
            } finally {
                if (!ranAction)
                    breakBarrier();
            }
        }

        // loop until tripped, broken, interrupted, or timed out
        //死循环
        for (;;) {
            try {
				//非超时等待，直接调用condition的await方法，阻塞当前线程并释放锁
                if (!timed)
                    trip.await();
                else if (nanos > 0L)
					//超时等待并释放锁
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
				//中断异常，调用打破屏障方法，是为了唤醒阻塞线程，抛出异常
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    // We're about to finish waiting even if we had not
                    // been interrupted, so this interrupt is deemed to
                    // "belong" to subsequent execution.
                    Thread.currentThread().interrupt();
                }
            }
			//屏障被打破，线程从await方法返回，但是抛异常
            if (g.broken)
                throw new BrokenBarrierException();
			//正常情况（Generation被重置），被唤醒后，返回index
            //什么情况下，Generation被重置? 1.BarrierAction执行 2.调用reset方法
            if (g != generation)
                return index;

            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
		//最后释放锁
        lock.unlock();
    }
}

//打破当前Barrier，唤醒所有阻塞线程
private void breakBarrier() {
    generation.broken = true;
    count = parties;
    trip.signalAll();
}

//重置Generation
private void nextGeneration() {
    // signal completion of last generation
    trip.signalAll();
    // set up next generation
    count = parties;
    generation = new Generation();
}

~~~


## 总结以及使用场景

CyclicBarrier提供的API方法很少，它与CountDownLatch类有相似之处。Cyclic表示它可以重复设置屏障，但是CountDownLatch不行。
从举例来看，CountDownLatch，是模拟举行会议。会议启动（Thread.start），但是阻塞了(CountDownLatch.await)，等待所有参数者参加（CountDownLatch.arrive），会议正是开始（唤醒线程）。
CyclicBarrier，模拟矩阵，5个线程(参与者)查找每行的关键字(Thread)，保存行结果（Thread Result）。但是想要最终的结果（final result），需要等待所有线程都跑完，最终执行合并结果(BarrierAction)。
