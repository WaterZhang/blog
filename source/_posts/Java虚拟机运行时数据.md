---
title: Java虚拟机运行时数据
date: 2016-09-13 10:23:50
tags: 
- JVM 
- JAVA
categories: JVM
toc: true
---

对于JAVA，很重要的特性，是开发人员不用关心GC(garbage collection)，Java语言使用时，不用关心内存回收问题，更多的关注business上。从另一面讲，如果Java虚拟机出现内存泄漏问题，往往会不知所措，引文内存回收，交给虚拟机来维护了。虚拟机帮我们做了，但是它也不是万能的，为了更好的使用Java，提升性能，规避内存溢出风险，需要深入了解虚拟机的构造。

## 运行时数据区域

Java虚拟机管理的内存包括如下几个数据区域，
![](http://photos.zhangzemiao.com/blog_jvm_structure.jpg)

- 图中标识绿色区域，方法区（另一个称呼永久区）和堆，这两个区域是线程共享的。
- 棕色区域，栈，本地方法栈和程序计数器，这三个区域是线程隔离（私有）的。

## 程序计数器

程序计数器（Program Counter Register），可以看作是当前线程所执行的字节码的行号指示器。

*程序计数器没有规定OutOfMemoryError的情况*
*如果线程正在运行native方法，那么PC是undefined的*

## Java虚拟机栈

虚拟机栈（Java Virtual Machine Statcks），生命周期与线程相同。方法调用时，会创建栈帧，放入栈中，跟踪方法的运行。栈帧中，存储局部变量表，操作数栈，动态链接，方法出口等信息。方法执行完成，栈帧出栈。内存不需要是连续的。

*线程请求的栈深度大于虚拟机锁允许的深度，会抛出StackOverflowError异常*
*无法申请足够的内存时，会抛出OutOfmemoryError异常*

## Java堆

堆（Java　Heap），虚拟机管理的最大的一块内存，存放对象实例，但是并不是所有对象都会在堆中，栈上分配，标量替换等优化技术（后续会涉及）的存在。

- *堆分代，包括新生代和老生代*
- *新生代包括Eden space，From Survior和To Survivor，采用标记复制清除算法*
- *新生代回收，对应为minor gc，这个回收过程很快，如果很频繁，属正常*
- *新生代，老生代和方法区都需要回收时，对应为full gc，这个回收过程慢，会影响程序性能，如果很频繁，需要分析问题，减少full gc次数*

## 方法区

方法区(Method Area)，线程共享区域，用于存储虚拟机加载的类信息，常量，静态变量，即使编译等数据。

*不同虚拟机，这个区域定义不同，对于官方的HotSpot，被称为永久代（Permanent Generation）,新版本中，好像改成元数据（metadata）了*
*某些情况下，方法区的内存可以被回收*
*无法分配足够内存时，会抛出OutOfMemoryError异常*

## 运行时常量池

运行时常量池（Runtime Constatnt Pool）是方法区的一部分。当创建一个类或接口时，组建相应的运行时常量池，需要更多的内存，如果不够会抛出OutOfMemoryError。

## 直接内存

直接内存（Direct Memory），不属于虚拟机运行时数据区。分配受到机器内存限制，会导致OutOfMemoryError异常。

参考文献[深入理解Java虚拟机](https://book.douban.com/subject/24722612/), [Java官方文档](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html)。