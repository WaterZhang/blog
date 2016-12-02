---
title: Java中的Queue队列分析
toc: true
date: 2016-11-28 17:17:39
tags:
- Java
- Queue
categories:
- CoreJava
---

JavaSE中，Collection是非常重要的一部分内容，也是源码看得最多的。[江南白衣](http://calvin1978.blogcn.com/articles/collection.html)也更新了它的集合小抄这文章，沿着他的思路，我也整理整理。

## 类图
- Iterable接口，支持foreach操作，迭代支持。
~~~java
public interface Iterable<T> {
    Iterator<T> iterator();
    default void forEach(Consumer<? super T> action) {
        Objects.requireNonNull(action);
        for (T t : this) {
            action.accept(t);
        }
    }
}
~~~

- Collection接口，定义集合类的公共方法

~~~java
public interface Collection<E> extends Iterable<E> {
    // Query Operations
    int size();
    boolean isEmpty();
    boolean contains(Object o);
    Iterator<E> iterator();

    Object[] toArray();
    <T> T[] toArray(T[] a);

    // Modification Operations
    boolean add(E e);
    boolean remove(Object o);

    // Bulk Operations
    boolean containsAll(Collection<?> c);
    boolean addAll(Collection<? extends E> c);
    boolean removeAll(Collection<?> c);
    boolean retainAll(Collection<?> c);
    void clear();
    // Comparison and hashing
    boolean equals(Object o);
    int hashCode();
    //remove jdk 1.8 feature

}
~~~

- Queue接口，队列的顶层接口

~~~java
public interface Queue<E> extends Collection<E> {
    boolean add(E e);
    boolean offer(E e);
    E remove();
    E poll();
    E element();
    E peek();
}
~~~

**注意，我把JDK 1.8的相关代码去掉了**

![](http://photos.zhangzemiao.com/blog_java_queue.jpg)

## 集合操作
集合操作，Collection接口和Queue接口都有定义，来看看有啥区别。我们以LinkedList类为例，发现LinkedList不靠谱，加入ArrayBlockingQueue做对比。

### 添加
Collection.add(Queue也定义的add方法，override)和Queue.offer。
LinkedList类实现，发现两个方法实现没有区别。在Java API中，有段描述，Collection的add方法，在添加元素失败后，会抛出unchecked异常。而offer方法，被设计成添加失败为正常，不抛异常。
~~~java
public boolean add(E e) {
	//插入链尾
    linkLast(e);
    return true;
}
public boolean offer(E e) {
    return add(e);
}
~~~
在ArrayBlockingQueue中，采用的是API描述的逻辑，代码如下
add方法，调用AbstractQueue抽象类的add方法。
~~~java
public boolean add(E e) {
    return super.add(e);
}
public boolean add(E e) {
    if (offer(e))
        return true;
    else
        throw new IllegalStateException("Queue full");
}
~~~

offer方法，只有在元素为null时，抛出空指针异常，其他情况不抛异常。
~~~java
public boolean offer(E e) {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //如果队列满了，返回false
        if (count == items.length)
            return false;
        else {
            enqueue(e);
            return true;
        }
    } finally {
        lock.unlock();
    }
}
~~~


### 删除
删除元素，Queue.remove()和Queue.poll()。API中描述，两者的区别，在于当队列为空时，remove抛出异常，poll方法返回null。而Collection.remove(Object),不做对比，remove成功返回true，remove失败返回false。
ArrayBlockingQueue的remove()方法，没有实现，沿用AbstractQueue抽象类的公共方法，会抛异常。
~~~java
public E remove() {
    E x = poll();
    if (x != null)
        return x;
    else
        throw new NoSuchElementException();
}
~~~

ArrayBlockingQueue的poll()方法，不会抛异常。
~~~java
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();
    }
}
~~~


### 检查
Queue.element和Queue.peek方法，element和peek方法都是获取但不删除。element方法，如果没有元素，则抛异常。peek方法，如果没有元素，则返回null。
ArrayBlockingQueue的element()方法，没有实现，沿用AbstractQueue抽象类的公共方法，
~~~java
public E element() {
    E x = peek();
    if (x != null)
        return x;
    else
        throw new NoSuchElementException();
}
~~~

ArrayBlockingQueue的peek方法，不会抛出异常。
~~~java
public E peek() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return itemAt(takeIndex); // null when queue is empty
    } finally {
        lock.unlock();
    }
}
~~~


## 具体队列的实现分析

在类图中，省略的很多具体的实现类，双向队列Dequeue和并发的队列，后续单独分析。LinkedList可以作为队列，也可以作为链表，这里不做过多分析。重点看看ArrayBlockingQueue和LinkedBlockingQueue。

### ArrayBlockingQueue

顾名思义，底层由数组实现的阻塞队列。怎么阻塞？采用ReentrantLock，根据上面贴的代码，已经看出来了，等待唤醒，使用ReentrantLock的Condition控制，一个notEmpty，一个notFull。**没有进行扩容处理。**

#### take
poll方法不会阻塞，take方法获取元素，会阻塞。
~~~java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0)
            notEmpty.await();
        //获取元素，会唤醒notFull的等待线程
        return dequeue();
    } finally {
        lock.unlock();
    }
}
~~~

#### put
add和offer方法，添加元素不会阻塞，put为阻塞方法。
~~~java
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length)
            notFull.await();
        //插入元素后，会唤醒notEmpty的阻塞线程
        enqueue(e);
    } finally {
        lock.unlock();
    }
}
~~~

### LinkedBlockingQueue

底层由分散的结点组成链表，默认为Integer.Max_VALUE长度，有一个头结点，尾结点。take和put采用各自的ReentrantLock，为嘛会搞两把锁？跟ArrayBlockingQueue（只有一把锁）不一样。

#### offer
offer方法，不阻塞
~~~java
public boolean offer(E e) {
    //插入空元素，抛异常
    if (e == null) throw new NullPointerException();
    //原子类，计数
    final AtomicInteger count = this.count;
    //如果队列满，立即返回false，不阻塞
    if (count.get() == capacity)
        return false;
    int c = -1;
    Node<E> node = new Node<E>(e);
    //插入锁
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        if (count.get() < capacity) {
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        }
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return c >= 0;
}
~~~

offer超时等待方法，队列满了，会等待超时时间，其他情况不阻塞，支持中断相应。
~~~java
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {

    if (e == null) throw new NullPointerException();
    long nanos = unit.toNanos(timeout);
    int c = -1;
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    //支持中断相应
    putLock.lockInterruptibly();
    try {
        //如果队列满，超时等待notFull
        while (count.get() == capacity) {
            if (nanos <= 0)
                return false;
            nanos = notFull.awaitNanos(nanos);
        }
        enqueue(new Node<E>(e));
        c = count.getAndIncrement();
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return true;
}
~~~

#### put
put方法，阻塞插入
~~~java
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
        //如果队列满，等待
        while (count.get() == capacity) {
            notFull.await();
        }
        //加入队列
        enqueue(node);
        //count自增，注意，返回的是更新前的值
        c = count.getAndIncrement();
        //如果未满，唤醒可能还有插入的线程
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    //如果队列初始的size为0，那么可能会有阻塞的take线程，这里唤醒take线程
    if (c == 0)
        signalNotEmpty();
}

//获取take锁，唤醒等待的take线程
private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
}

~~~

#### take
take方法，阻塞获取
~~~java
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        //如果队列空，等待
        while (count.get() == 0) {
            notEmpty.await();
        }
        //获取头结点
        x = dequeue();
        //自增，返回先前的值
        c = count.getAndDecrement();
        //如果队列有大于1的个数，继续唤醒take线程，进行操作
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    //如果先前的队列是满的，那么可能会有put的阻塞线程，唤醒它们
    if (c == capacity)
        signalNotFull();
    return x;
}

//获取写锁，唤醒阻塞的put线程
private void signalNotFull() {
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        notFull.signal();
    } finally {
        putLock.unlock();
    }
}
~~~

#### 临界分析
为什么LinkedBlockingQueue可有两个锁呢，想了一下，一个队尾插入，一个队头获取，互不干扰。只有在一些临界条件，会相互影响一下。
当队列为空的时候，也就是刚刚初始化后，Head和Tail共同指向一个Null结点。如果进行take操作，所有线程进入take锁的notEmpty条件阻塞线程。如果进行put操作，Head指向Null结点，Null结点指向新结点，tail指向新节点。
~~~java
public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    //默认会有Null结点，头尾均指向它
    last = head = new Node<E>(null);
}

//链尾插入新结点
private void enqueue(Node<E> node) {
    // assert putLock.isHeldByCurrentThread();
    // assert last.next == null;
    last = last.next = node;
}
~~~

当队列满了，如果继续插入元素，put线程加入notFull条件中阻塞队列。take线程来，从链头拿元素，默认第一个元素是null的，take操作，head指向下一个结点，并返回结点中的元素，然后将元素设置为null。这样头结点又变成了Null结点。然后唤醒put阻塞线程。
~~~java
private E dequeue() {
    // assert takeLock.isHeldByCurrentThread();
    // assert head.item == null;
    //head默认是个null结点
    Node<E> h = head;
    Node<E> first = h.next;
    //原始头结点自己指向自己
    h.next = h; // help GC
    //第一个存储元素的结点，作为头结点
    head = first;
    //拿出新的头结点的元素
    E x = first.item;
    //设置新的头结点的元素为null，变成Null结点
    first.item = null;
    return x;
}
~~~

