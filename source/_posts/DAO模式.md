---
title: DAO模式
date: 2016-10-22 19:21:13
tags:
- Java
- 设计模式
categories:
- 设计模式
toc: true
---

这篇文章，做为整理设计模式相关文章的开篇。到目前为止，已经有JVM，Spring，CoreJava三条线索整理Java相关知识，加上设计模式有4条了，每条都还只是开头，加油。

## DAO模式定义

顾名思义，DAO = Data Access Object，用来抽象和封装处理数据源的操作。DAO管理与数据源的连接，获取和存储数据。数据源，可以是DB，文件，第三方服务等。对于DAO的使用者，不用知道到底出哪儿获取的数据，DAO隐藏数据源的实现细节，提供接口给用户使用，接口不会因为数据源的改变而改变。DAO相当与在组件和数据源之间扮演适配者。
![](http://photos.zhangzemiao.com/blog_dao_pattern.jpg)
如上图所示，DAO模式的类图。
Business Object：代表使用数据的用户。该对象需要访问数据源，获取和存储数据。可以是Session Bean，实例Bean以及其他Java对象，例如servlet的helper类。
DataAccessObject：该模式的核心对象，抽象潜在的数据访问实现。使Business Object可以透明的访问数据源。相当于Business Ojbect委托数据获取和存储操作给DataAccessObject。
DataSource：数据源的实现，可以是RDBMS，OODBMS，XML repository，文件系统，其他服务等。
TransferObject：数据载体，作为一个数据对象，由DataAccessObject返回给用户（Business Object）。存储的话，则反向。

## DAO模式扩展

针对不同的数据源以及相关业务，可以做相关工厂设计。
如果业务相对简单，数据源可能替换，可以基于工厂方法设计DAO工厂。
如果业务相对复杂，数据源多样性，可以基于抽象工厂，设计DAO工厂。
后续介绍什么是工厂方法模型和抽象工厂模式。

## 总结

透明，对于Business Object，数据源细节透明。
易迁移，DAO的存在，使APP可以迁移至不同的数据库，分层思想。
减少Business Object的复杂性。
集中化，集中数据操作在一层，还是分层思想。
简单化，相比EJB，不使用容器持久化管理，过于复杂不好。

