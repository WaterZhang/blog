---
title: CountDownLatch使用与分析
toc: true
date: 2016-12-15 14:01:44
tags:
- Java
categories:
- Java并发编程
---

介绍CountDownLatch，并发流程控制。

## 概述

看看[Java 1.8的API](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CountDownLatch.html)是怎么说的，一种同步机制，允许一个或者线程等待，直到其他线程执行一些操作完成。CountDownLatch基于一个given count，进行初始化，然后调用await方法，阻塞当前线程的流程。直到其他线程执行countDown方法，直至count变为0。那么阻塞线程被唤醒执行。
**CountDownLatch中的count不能被重置**

## 样例
模拟开会，组织一个会议，要求10人到齐就开始。会议先准备好，然后参加会议的人员一个个来报到，直至到齐。会议则正式开始。

### VideoConference
会议类

~~~java
public class VideoConference implements Runnable {

	private final CountDownLatch controller;

	//initial count，不能被重置
	public VideoConference(int number) {
		controller = new CountDownLatch(number);
	}
	//人员报到，调用countDown
	public void arrive(String name){
		System.out.printf("%s has arrived.",name);
		controller.countDown();
		System.out.printf("VideoConference: Waiting for %d participants.\n",controller.getCount());
	}
	
	@Override
	public void run() {
		System.out.printf("VideoConference: Initialization: %d participants.\n", controller.getCount());
		try {
        	//启动会议，并等待人员报到
            //API还提供超时等待，如果规定时间内，参与人员没有满员到齐，可做其他处理
			controller.await();
			System.out.printf("VideoConference: All the participants have come\n");
			System.out.printf("VideoConference: Let's start...\n");
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}
~~~

### 参与者

~~~java
public class Participant implements Runnable {
	
	private VideoConference conference;
	private String name;

	public Participant(VideoConference conference, String name) {
		this.conference = conference;
		this.name = name;
	}
	
    //参加会议，调用arrive方法，报到
	@Override
	public void run() {
		long duration=(long)(Math.random()*10);
	    try {
	      TimeUnit.SECONDS.sleep(duration);
	    } catch (InterruptedException e) {
	      e.printStackTrace();
	    }
	    conference.arrive(name);
	}

}
~~~

### 测试类
~~~java
public class Test {

	public static void main(String[] args) {
		VideoConference conference=new VideoConference(10);
		Thread threadConference=new Thread(conference);
	    threadConference.start();
		for (int i = 0; i < 10; i++) {
			Participant p = new Participant(conference, "Participant " + i);
			Thread t = new Thread(p);
			t.start();
		}

	}

}
~~~

## 源码分析

代码很简洁，跟之前解析锁的实现一样，使用AQS实现，内部类Sync实现AQS。

### 初始化
当创建一个CountDownLatch时，initial count设置成AQS的state。
~~~java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
//内部类Sync的方法
Sync(int count) {
    setState(count);
}
~~~

### await方法

CountDownLatch.class
~~~java
public void await() throws InterruptedException {
	//获取共享锁
    sync.acquireSharedInterruptibly(1);
}
~~~

AbstractQueuedSynchronizer.class
~~~java
//这是个模板方法，回调具体的实现，即Sync的实现方法tryAcquireShared()
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    //如果为负数，调用doXX方法，支持中断的阻塞
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
~~~

CountDownLatch.Sync.class
~~~java
//这里的逻辑很简单，获取state，如果不为0，则返回负数
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
~~~

### countDown方法

CountDownLatch.class
~~~java
//直接调用AQS的释放锁
public void countDown() {
    sync.releaseShared(1);
}
~~~

AbstractQueuedSynchronizer.class
~~~java
//AQS的模板方法
public final boolean releaseShared(int arg) {
	//
    if (tryReleaseShared(arg)) {
    	//如果return true，则要唤醒阻塞的线程
        doReleaseShared();
        return true;
    }
    return false;
}
~~~

回调tryReleaseShared方法
CountDownLatch.Sync.class
~~~java
//将state自减
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        //注意，如果next state是0,即state状态减至0后，return true
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
~~~

## 总结
CountDownLatch完全依赖AQS来实现，Doug Lea很牛X。