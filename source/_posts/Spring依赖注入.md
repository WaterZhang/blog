---
title: Spring依赖注入
date: 2016-09-26 09:53:30
tags: 
- Java
- Spring
categories: Spring
toc: true
---

Java，发展20多年至今，已经占领企业级开发，Java自己提供的JavaEE标准中，EJB定义过于复杂，自入行业以来木有做过相应的开发，然后Spring却一直在使用，所有Spring作为Java学习的一条主线，进行系统性的整理，归纳。

## 介绍

Spring对照EJB，相对简化的开发平台，但是[Spring](http://spring.io/)不是一个简单的框架，可以说是一个很大的框架级。核心思想，是基于POJOs的容器，支持依赖注入，各种templates模版，AOP编程，集成很多第三方流行框架。

## Spring主要框架

- Spring Core，提供依赖注入(dependency injection)，解耦创建（decoupling the creation），配置和管理bean。
- AOP，AspectJ，提供面向切面支持，应用如安全，日志，事务管理。
- Data Access/Integration，提供Spring JDBC，ORM，OXM，JMS，JPA等支持。
- Web，有Spring MVC，MVC模型支持，也可以集成第三方框架，如Struts，JSF，Velocity，FreeMarker。
- Test，给Junit和TestNG提供支持。

## 依赖注入与控制反转

Spring as DI container，依赖注入什么东西？POJOs。但是POJO是什么？姑且片面的理解就是一般的Bean。

先看看什么是控制反转？对比有Spring和没有Spring，都是怎么玩儿的？

我有一个DAO类，会包含一个DataSource类，获取数据源。没有spring支持，Dao类初始化后，也需要对DataSource进行初始化，负责创建DataSource对象。换句话说，Dao类负责DataSource类的生命周期。那么Dao类和DataSource类有很明确的依赖关系，控制权在Dao类。

在没有Spring支持下，想要去掉明确的依赖，Dao类，有一个DataSource类（通常定义为接口）的引用，只负责声明，不需要创建和初始化。需要提供一个setDataSource方法，这样可以由比Dao类更上层的类（Service类）来管理。这样的话，创建DataSource的责任脱离了Dao类，交由上层Service类来管理。但是Service呢，怎么办呢？只能一直把责任往上传？对于Dao类来说，实现了对DataSource类的依赖注入，苦了Service类。

所以现在需要一个上下文的解决方法，把Service类这层，也实现依赖注入，Spring出现了，创建依赖关系的责任，往上抛，并把创建类的责任都给Spring。Spring会帮忙创建Service类，Dao类以及DataSource类的对象，并将指定的DataSource赋给Dao类，指定的Dao类赋给Service类，最终控制权就在Spring了。这就是控制反转的概念，基于接口依赖，不依赖实现。

## Spring实现依赖注入

Spring通过反射创建实例，通过ApplicationContext接口，绑定标识ID。
通过基于XML配置或注解，告诉Spring，你要创建哪些类，类的依赖是哪个具体实现。
基于XML实现依赖注入：
- Setter Injection
- Constructor Injection
- Factory-method Injection

### Beans命名空间

基于XML实现依赖注入时，需要定义Beans命名空间：
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-3.1.xsd">
</beans>

```

### PropertyPlaceholderConfigurer

在XML配置Bean时，需要将配置信息抽离，使用PropertyPlaceholderConfigurer，用XML配置该Bean(没有id attribute，因为Spring容器自己检测是否存在，供使用)，并指向对应的资源文件。然后在xml中，就可以使用el（${configName}）表达式获取配置信息。默认会在classpath找，如果不在classpath，可以使用"file:${user.home}/filename.properties"

### Bean Scopes

Singleton， Spring默认的Bean范围，Spring容器只有一个实例，会做Cache，但Spring不维护Bean的状态。
Prototype，每次ApplicationContext.getBean请求或者每次注入时，都创建实例。Spring只负责创建，不负责销毁。
Request / Session / GlobalSession，这三种范围配置，仅用于Web application的上下文。

### 注解

始于Spring 2.0，到Spring 2.5得到加强。
绑定使用@Autowired，声明使用@Controller，@Component，@Service，@RestController，@Repository。
Bean扫描
```
<context:component-scan base-package=""/>
```

要想使@Autowired生效，需要在XML中添加context namespace，并加上<context:annotation-config/>
如果同一个接口，声明了两个或多个实现类，并注解使用，那么使用@Autowired时，会返回的是一个数组，所以可能会抛出异常，可以用一个数组来接受。当然，这应该不是我们想要的。在@Autowired后加上指定标识名，然后加上@Qualifier。

### 修改Spring的Bean名字生成策略

使用BeanNameGenerator。

### XML还是Annotation

个人倾向与注解，去XML化。

### 待办事项

在没有任何XML配置下，使用Spring构建Web项目...



