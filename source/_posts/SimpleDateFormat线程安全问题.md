---
title: SimpleDateFormat线程安全问题
date: 2016-10-16 15:43:51
tags: Java 异常分析
categories:
- JavaSE
- CoreJava
- Java异常分析
toc: true
---

项目运行时间，SimpleDateFormat使用出现了异常，刚开始并没有太在意。但是或多或少会有业务影响，查询了一下，发现SimpleDateFormat是线程非安全的，抽空试了一下在多线程下使用它。

## SimpleDateFormat.format

试了很多次，format方法不会有多线程并发的问题，没有异常抛出。

## SimpleDateFormat.parse

parse方法，在多线程并发情况下，会抛出两种异常信息，可能也会同时出现。
一个是：java.lang.NumberFormatException: For input string: ""
![](http://photos.zhangzemiao.com/blog_simpledateformat_error2.jpg)
另一个是：java.lang.NumberFormatException: multiple points
![](http://photos.zhangzemiao.com/blog_simpledateformat_error1.jpg)

## 解决方法

1. 线程创建使用
2. 使用即创建
3. 同步加锁

对于方案1，是在线程自己能控制的情况，那么可以伴随线程周期创建一个，线程结束则回收。
如在Web项目中，用户请求，Web容器会创建线程，不知怎么在线程中，只创建一个对象供使用，考虑方案2和方案3，方案2，会产生很多碎片化的对象，随着方法运行完而回收。方案3，则并不会产生碎片对象，对于垃圾回收，是非常友好的。先简单测测方案2和方案3的耗时。
测试场景，两个线程，分别创建100个，总共200个线程同时运行，每个线程使用SimpleDateFormat的parse方法1万次。
方案2耗时：43202毫秒
方案3耗时：44816毫秒
试了几次，两种方案的耗时是差不多的，嗯。。。可能写的测试代码不靠谱。个人倾向于方案2，在官网文档，只提了建议使用方案1。。。
