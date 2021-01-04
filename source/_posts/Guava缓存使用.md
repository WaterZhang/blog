---
title: Guava缓存使用
date: 2016-11-22 17:23:07
tags:
- Java
- 缓存
categories:
- 缓存
toc: true
---

Guava是Google提供的Java工具框架，包括一些工具类，Java集合框架的扩展，还有更重要的guava cache。

相比Java的HashMap，ConcurrentHashMap，提供更加灵活的配置和功能，比如控制缓存大小，缓存过期时间等，来试试。

## 版本
我使用了[Guava](https://github.com/google/guava)最新的版本[20.0](https://github.com/google/guava/wiki/Release20)
Gradle,
~~~java
compile("com.google.guava:guava:20.0")
~~~

## Cache接口
Cache接口，是最顶层的缓存接口，没有结合数据获取功能，看源码，省去一些代码和注释，
~~~java

public interface Cache<K, V> {

  /**
   * Returns the value associated with {@code key} in this cache, or {@code null} if there is no
   * cached value for {@code key}.
   *
   * @since 11.0
   */
  @Nullable
  V getIfPresent(Object key);

  V get(K key, Callable<? extends V> loader) throws ExecutionException;

  ImmutableMap<K, V> getAllPresent(Iterable<?> keys);

  void put(K key, V value);


  void putAll(Map<? extends K, ? extends V> m);

  /**
   * Discards any cached value for key {@code key}.
   */
  void invalidate(Object key);

  /**
   * Discards any cached values for keys {@code keys}.
   *
   * @since 11.0
   */
  void invalidateAll(Iterable<?> keys);

  /**
   * Discards all entries in the cache.
   */
  void invalidateAll();

  /**
   * Returns the approximate number of entries in this cache.
   */
  long size();


  CacheStats stats();

  ConcurrentMap<K, V> asMap();

  /**
   * Performs any pending maintenance operations needed by the cache. Exactly which activities are
   * performed -- if any -- is implementation-dependent.
   */
  void cleanUp();
}
~~~

测试代码，
~~~java
public static void main(String[] args) throws Exception {

    Cache<String,TradeAccount> cache = CacheBuilder.newBuilder().
                            expireAfterWrite(5L, TimeUnit.SECONDS).
                            maximumSize(5000L).
                            ticker(Ticker.systemTicker()).
                            build();
    TradeAccount account1 = new TradeAccount();
    cache.put("key", account1);

    TimeUnit.SECONDS.sleep(6);

    TradeAccount a1 = cache.getIfPresent("key");
    TradeAccount a2 = cache.getIfPresent("key2");
    a2 = cache.getIfPresent("key2");
    a2 = cache.getIfPresent("key3");
    System.out.println(a1 == account1);

}
~~~
get方法，如果没有数据，需要自己添加一个loader继续callable接口，不够灵活，参数传入不方便。
建议使用getIfPresent方法，通用代码如下，
~~~java
value = cache.getIfPresent(key);
if(value == null){
    value = someService.retrieveValue();
    cache.put(key,value);
}
~~~

## LoadingCache接口
LoadingCache，继承Cache接口，可以跟数据获取的服务结合使用。
测试代码，
~~~java
public static void main(String[] args) throws Exception {


    ITradeAccountService service = new TradeAccountService();
    LoadingCache<String,TradeAccount> cache = CacheBuilder.newBuilder().
                            expireAfterWrite(5L, TimeUnit.MINUTES).
                            maximumSize(5000L).
                            ticker(Ticker.systemTicker()).
                            build(new CacheLoader<String, TradeAccount>() {
                                @Override
                                public TradeAccount load(String key) throws Exception {
                                    return service.getTradeAccountById(key);
                                }
                            });
    TradeAccount account1 = new TradeAccount();
    cache.put("key", account1);
    TradeAccount a1 = cache.get("key");
    TradeAccount a2 = cache.get("key2");
    a2 = cache.get("key2");
    a2 = cache.get("key3");
    System.out.println(a1 == account1);

}
~~~

~~~java
public class TradeAccount {
	private String id;
	private String owner;
	private double balance;
    ...get/set
}
~~~

~~~java
public interface ITradeAccountService {
	TradeAccount getTradeAccountById(String key);
}
~~~

根据自己的业务的场景，选择使用Cache还是LoadingCache。
## 总结
Guava Cache，核心使用就是这两个，其他附加的功能，没有花时间研究。下一步研究guava cache和Spring cache，如果一起使用。
