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