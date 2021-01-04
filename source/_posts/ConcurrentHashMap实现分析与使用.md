---
title: ConcurrentHashMap实现分析与使用
date: 2016-11-14 21:19:12
tags: Java
categories: Java并发编程
toc: true
---

分析线程安全的ConcurrentHashMap是怎么实现的，基于JDK 1.8。能力有限，不能看懂所有的代码。

## 类图关系
ConcurrentHashMap类中，定义了大量的内部类。。。不画了

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
定位到元素后，确认结点的hash码跟Key的hash码一样，两个情况会返回结点的value，这里跟HashMap应该是一样的判断。
1. Key完全一样（引用地址一样）
2. Key的equals方法返回true

当定位到元素，结点的hash码与Key的hash码不一样时且，结点的hash码为负数(小于0)，对于小于0的含义，暂不清楚。这种情况，调用结点Node的find方法，查找结点。
~~~java
else if (eh < 0)
    return (p = e.find(h, key)) != null ? p.val : null;
~~~
查找方法，链式逐个查找，根据上述两个判断，返回相应的value。
~~~java
Node<K,V> find(int h, Object k) {
    Node<K,V> e = this;
    if (k != null) {
        do {
            K ek;
            if (e.hash == h &&
                ((ek = e.key) == k || (ek != null && k.equals(ek))))
                return e;
        } while ((e = e.next) != null);
    }
    return null;
}
~~~

当定位到元素，结点的hash码与Key的hash码不一样且，结点的hash码是大于0的，直接链式查找，根据上述两点判断，返回value。
从get方法来看，ConcurrentHashMap跟HashMap一样，是一个数组加链表的结构。并发安全的问题，在get方法体现不多。table是volatile的，保证内存可见性。定位bucket时，使用的unsafe类原子获取。

### put方法

put方法代码如下，
~~~java
public V put(K key, V value) {
    return putVal(key, value, false);
}

/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
~~~

**跟get方法一样，不支持null key或者null value**
table懒加载，sizeCtl是Map的大小（容量），在调用构造函数时，传入initialCapacity参数，则sizeCtl是大于0的，如果默认无参，则sizeCtl是0。在init时，sc大于0，则说明resize好了，否则默认为16（这里的意思是16的bucket的数量，即数组table的size是16）。initial完毕后，sizeCtl的值变成了12，刚好是容量*Hash因子0.75。这不是巧合，而是设计。
~~~java
if (tab == null || (n = tab.length) == 0)
    tab = initTable();
~~~
~~~java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
~~~

初始化table后，定位位置，如果结点为null，说明这个bucket是空的，创建新的结点，插入。
~~~java
else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
    if (casTabAt(tab, i, null,
                 new Node<K,V>(hash, key, value, null)))
        break;                   // no lock when adding to empty bin
}
~~~

如果定位位置，结点不为null，意味着这坑有人占了。如果结点的hash码为-1，会做些操作，暂不分析这种情况。
~~~java
else if ((fh = f.hash) == MOVED)
    tab = helpTransfer(tab, f);
~~~

如果定位位置，结点不为null，意味着这坑有人占了。如果结点的hash码为正常的，则使用关键字syncrhonized锁上当前结点（我们只分析正常情况的处理）。
定位结点的hash码是大于0的，链式遍历至key匹配（如何匹配跟get方法一样），匹配到则覆盖（不考虑ifAbsent）,如果没有匹配，链尾插入。
如果链表长度超过或等于8（TREEIFY_THRESHOLD），结点会被转换成TreeNode（由TreeBin实现红黑树）。
~~~java
else {
    V oldVal = null;
    synchronized (f) {
        if (tabAt(tab, i) == f) {
            if (fh >= 0) {
                binCount = 1;
                for (Node<K,V> e = f;; ++binCount) {
                    K ek;
                    if (e.hash == hash &&
                        ((ek = e.key) == key ||
                         (ek != null && key.equals(ek)))) {
                        oldVal = e.val;
                        if (!onlyIfAbsent)
                            e.val = value;
                        break;
                    }
                    Node<K,V> pred = e;
                    if ((e = e.next) == null) {
                        pred.next = new Node<K,V>(hash, key,
                                                  value, null);
                        break;
                    }
                }
            }
            else if (f instanceof TreeBin) {
                Node<K,V> p;
                binCount = 2;
                if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                               value)) != null) {
                    oldVal = p.val;
                    if (!onlyIfAbsent)
                        p.val = value;
                }
            }
        }
    }
    if (binCount != 0) {
        if (binCount >= TREEIFY_THRESHOLD)
            treeifyBin(tab, i);
        if (oldVal != null)
            return oldVal;
        break;
    }
}
~~~

从Map接口的put方法来看，JDK1.8已经放弃分段锁的设计（Segment数组+Hash.Entry结构），跟HashMap类似，数组+Node的方式，Node可以是链表，超过额度，转换成红黑树，优化检索速度。

###remove方法
remove方法操作的逻辑跟put方法一样，最终操作一个是删除，一个是加结点。

## 扩容
HashMap底层都是数组作为容器，无可避免需要扩容。在put方法的最后，会调用addCount方法，这里会做扩容操作，说实在的，很多东西设计的很巧妙，也不易懂，如果不从思想上去理解，难看懂。关于扩容的问题，分析不了太多。关于扩容，需要关注helpTransfer()和transfer()方法，在并发的考虑上，设计的真的巧妙。

## 总结
ConcurrentHashMap延用的分段锁的概念，但摒弃了之前的实现，数组作为bucket，Node作为元素扩展基础。
- 使用底层Unsafe类提供的原子操作，进行CAS。
- 二次hash，相比HashMap，在Key的hashcode()基础上，再进行一次hash，更好的打散元素。
- 在remove和put操作时，会使用synchronized关键字，锁定bucket的头结点，延用分段锁的思想。
- Node结点，多样性，有链式，还有二叉树式，还有扩容式，所有头结点的hash很重要，用来区分不同的Node。在解决hash冲突上，跟HashMap一样，做了改进，将链表优化成红黑树，优化检索。
- 扩容，说是在，这里的代码没看太懂，只能说很巧妙。等我看懂了，再补充哈。

还有并发Map的接口，没有分析，有空再分析。

