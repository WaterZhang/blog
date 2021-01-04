---
title: SpringCache与Guava结合使用
toc: true
date: 2016-11-22 18:11:09
tags:
- Java
- 缓存
categories:
- 缓存
---

少废话，上菜。

## 启动Spring Cache机制
默认情况，使用Spring without configuration方式，我的测试代码在[这里](http://git.oschina.net/zhangzemiao/FindWeb)。官方讲解Spring Cache在[这里](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/cache.html)。

1. 引用依赖
如果是Spring Boot项目，可以参考[这里](https://spring.io/guides/gs/caching/)，
gradle,
~~~java
dependencies {
    compile("org.springframework.boot:spring-boot-starter-cache")
}
~~~
maven,
~~~java
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>
</dependencies>
~~~
我测试是用的Spring Boot项目，但是没有用Spring Boot组件，而是引用Spring Context Support来支持Guava。
gradle,
~~~java
compile("org.springframework:spring-context-support")
~~~
2. 启动Cache Configuration
@Configuration
@EnableCaching
public class SecureTomcatConfiguration {
	...
}
3. 添加CacheManager
我使用Spring提供的GuavaCacheManager，参数设置一个或多个Cache，如下
~~~java
@Bean
public CacheManager cacheManager() {
    GuavaCacheManager cacheManager = new GuavaCacheManager("trades");
    cacheManager.setCacheBuilder(CacheBuilder.newBuilder()
            .expireAfterWrite(60 , TimeUnit.SECONDS)
            .maximumSize(1000));
    return cacheManager;
}
~~~
如果你想设置不同的size和过期时间的Cache，可以使用SimpleCacheManager，设置GuavaCache的List，
~~~java
@Bean
public CacheManager cacheManager() {
    SimpleCacheManager manager = new SimpleCacheManager();
    List list = new ArrayList();
    list.add(GuavaCache one);
    list.add(GuavaCache two);
    ...
    manager.setCaches(  list )
    return manager;
}
~~~
4. 使用Cache
~~~java
@Cacheable(cacheNames="trades", key = "#key")
public TradeAccount getTradeAccountById(String key,String owner,double balance) {
    System.out.println("invoke getTradeAccountById method :"+ (++COUNT));
    TradeAccount mock = new TradeAccount();
    mock.setId(key);
    mock.setBalance(balance);
    mock.setOwner(owner);
    return mock;
}
~~~

## 唯一性
在上述的代码中，@Cacheable里有包含key属性，指定的哪个参数作为key，作为唯一性的判断。如果没有指定key，那么所有的参数都会作为keys，然后判断keys的hashCode和equals方法，官方默认实现，见如下分析，
你还可以自己实现KeyGenerator接口，设置自己的KeyGenerator，然后在@Cacheable使用，
~~~java
Class MyKeyGenerator implements KeyGenerator{
	...
}

@Cacheable(cacheNames="books", keyGenerator="myKeyGenerator")
public Book method(...){
...
}
~~~
查看了Spring 4.0以上，Default Key Generation，SimpleKeyGenerator。默认的Key Generator会将所有的参数包装成一个 SimpleKey。然后依据这个SimpleKey的hashcode和equals方法判断唯一性。
~~~java
public class SimpleKeyGenerator implements KeyGenerator {

	@Override
	public Object generate(Object target, Method method, Object... params) {
		return generateKey(params);
	}

	/**
	 * Generate a key based on the specified parameters.
	 */
	public static Object generateKey(Object... params)
    	//如果无参，则用返回EMPTY对象
		if (params.length == 0) {
			return SimpleKey.EMPTY;
		}
        //如果只有一个参数，就返回当前参数作为Key对象
		if (params.length == 1) {
			Object param = params[0];
			if (param != null && !param.getClass().isArray()) {
				return param;
			}
		}
        //其他情况，则返回SimpleKey对象
		return new SimpleKey(params);
	}

}

public class SimpleKey implements Serializable {

	...

	/**
     * 这里构造SimpleKey，保存所有参数，然后使用Arrays.deepHashCode，作为新对象的hashcode
	 * Create a new {@link SimpleKey} instance.
	 * @param elements the elements of the key
	 */
	public SimpleKey(Object... elements) {
		Assert.notNull(elements, "Elements must not be null");
		this.params = new Object[elements.length];
		System.arraycopy(elements, 0, this.params, 0, elements.length);
		this.hashCode = Arrays.deepHashCode(this.params);
	}

	/**
    * equals方法，最终使用Arrays.deepEquals比较
    */
	@Override
	public boolean equals(Object obj) {
		return (this == obj || (obj instanceof SimpleKey
				&& Arrays.deepEquals(this.params, ((SimpleKey) obj).params)));
	}

	/**
    * 直接返回hashcode
    */
	@Override
	public final int hashCode() {
		return this.hashCode;
	}

	...
}
~~~

## 注解@Cacheable参数分析

1.cacheManager
这个参数指定CacheManager，没有使用。
2.cacheNames
指定哪个使用哪个cache
3.cacheResolver
这个没有深入研究，忽略。
4.condition
条件限制，这个判断是在方法执行前，决定是否缓存。
5.key
指定存储的Key，默认是方法的所有参数组成keys。
6.keyGenerator
KeyGenerator，默认是所有参数keys，生成一个SimpleKey，作为Cache对象的Key。
7.sync
未研究，可参考官方文档。
8.unless
这个是方法执行后，解析结果，过滤。
9.value
跟cacheNames功能一样，别名使用。

## 其他

如果要灵活使用Spring Cache的注解方法，需要熟悉Spring EL，方便灵活的刷选结果进行缓存。还有其他的注解进行缓存，这里就没介绍了，更多详细的，可以研究Spring Cache官方文档和其他资料。



