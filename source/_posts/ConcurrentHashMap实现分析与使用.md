---
title: ConcurrentHashMap实现分析与使用
date: 2016-11-14 21:19:12
tags: Java
categories: Java并发编程
toc: true
---

分析线程安全的ConcurrentHashMap是怎么实现的，基于JDK 1.8。

## 类图关系

## Map接口实现分析

在Map接口中，定义了以下主要方法，
- get(Object key)
- put(K key, V value)
- remove(Object)
- isEmpty()
- containsKey(Object key)
- containsValue(Object value)
- clear()
。。。

### get方法
Get方法代码如下：
~~~java
Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
int h = spread(key.hashCode());
if ((tab = table) != null && (n = tab.length) > 0 &&
	(e = tabAt(tab, (n - 1) & h)) != null) {
	if ((eh = e.hash) == h) {
		if ((ek = e.key) == key || (ek != null && key.equals(ek)))
			return e.val;
	}
	else if (eh < 0)
		return (p = e.find(h, key)) != null ? p.val : null;
	while ((e = e.next) != null) {
		if (e.hash == h &&
			((ek = e.key) == key || (ek != null && key.equals(ek))))
			return e.val;
	}
}
return null;
~~~

**ConcurrentHashMap不支持null key，会抛出空指针异常(NullPointerException)**
首先通过**spread**方法，进行二次Hash，得到hash码，如下，
~~~java
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
~~~
如果table为null，则直接返回null。table不为null，则hash码 & (length-1)，tabAt代码如下，最终通过unsafe类的原子操作，获取数组结点。
~~~java
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
~~~
~~~java
private static final sun.misc.Unsafe U;
~~~
在这里，有个很重要的数据结构Node<K,V>，后续要分析。


