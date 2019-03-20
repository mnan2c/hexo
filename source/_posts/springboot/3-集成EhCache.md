title: SpringBoot集成EhCache

tags:
  - SpringBoot

categories:
  - SpringBoot

---
## 一、缓存简介
#### 1. 什么是缓存
将程序或系统经常要调用的对象存在内存中，使用时可以快速调用，不必再去创建重复的实例，减少无谓的db访问（数据库访问占用数据库连接，同时网络消耗比较大）。可以分为两类：
- 文件缓存，把数据存储在磁盘上；
- 内存缓存，创建一个静态内存区域，将数据存储进去，例如B/S架构将数据存储在Application中或者存储在一个静态Map中

```java
//这是一个典型的单例模式，用localCacheStore存储对象
//这个代码有几个问题：
//1. 没有缓存大小的设置
//2. 没有缓存的失效策略
//3. 持久性存储
//4. 没有监控统计
public class CacheManager {
  //一个本地的缓存Map
  private Map localCacheStore =new HashMap();
  //一个私有的对象，非懒汉模式
  private static CacheManager cacheManager =new CacheManager();
  //私有构造方法，外部不可以new一个对象
  private CacheManager(){}

  //静态方法，外部获得实例对象
  public static CacheManager getInstance(){
    return cacheManager;
  }

  public Object get(String key){
    return localCacheStore.get(key);
  }

  public void put(String key ,Object value){
    localCacheStore.put(key, value);
  }
}
```

## 二. 缓存配置简介
- Spring为我们自动配置了多个 CacheManager 的接口，包括 EhCacheCacheConfiguration、RedisCacheConfiguration等，但并没有提供具体的缓存实现方案，它底层需要依赖Redis、Guava等具体的缓存工具。因此应用程序只要面向Spring缓存API编程，应用底层的缓存实现可以在不同的缓存实现之间自由切换，应用程序无需任何改变，只要改变配置文件即可。
<!--more-->
- 在 Spring Boot中，通过@EnableCaching注解自动化配置合适的缓存管理器（CacheManager）。

## 三. EhCache
- memcached或Redis这两种缓存框架都要搭建服务器，而EhCache不需要
- Ehcache是一个纯Java的进程内缓存框架，提供了用内存，磁盘文件存储，以及分布式存储方式等多种灵活的cache管理方案
- 目前版本3.4

## 四、EhCache架构设计
结构图如下：

![jiegoutu](/img/springboot/ehcache-architect.jpg)

说明：
- CacheManager：缓存管理器，可以通过单例或者多例的方式创建，也是Ehcache的入口类。
- Cache：每个CacheManager可以管理多个Cache，每个Cache可以采用hash的方式管理多个Element。
- Element：用于存放真正缓存内容。

## 五、使用场景

1. 比较少的更新数据表的情况
2. 对并发要求不是很严格的情况
  多台应用服务器中的缓存是不能进行实时同步的。
3. 对一致性要求不高的情况下
  因为Ehcache本地缓存的特性，目前无法很好的解决不同服务器间缓存同步的问题，所以我们在一致性要求非常高的场合下，尽量使用Redis、Memcached等集中式缓存。

## 六. 使用
#### 1. 添加依赖
```xml
<!-- cache -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcache</artifactId>
    <version>2.9.1</version>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```
#### 2. config/ehcache.xml:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<ehcache updateCheck="false" dynamicConfig="false">
    <diskStore path="java.io.tmpdir"/>

    <cache name="username" maxElementsInMemory="10000"
             timeToLiveSeconds="600" overflowToDisk="true" />

</ehcache>

<!-- cache的属性说明：

name:缓存的名称。
maxElementsInMemory：缓存中最大元素个数。0表示不限制
maxElementsOnDisk：硬盘最大缓存个数。
eternal:对象是否永久有效，一但设置了，timeout将不起作用，元素永久存在。
clearOnFlush：内存数量最大时是否清除。
timeToIdleSeconds ： 设置对象在失效前的允许闲置时间（单位：秒）。
	仅当eternal=false对象不是永久有效时使用，可选属性，默认值是0，也就是可闲置时间无穷大。
timeToLiveSeconds：设置对象在失效前允许存活时间（单位：秒）。最大时间介于创建时间和失效时间之间。
	仅当eternal=false对象不是永久有效时使用，默认是0.，也就是对象存活时间无穷大。
diskExpiryThreadIntervalSeconds：磁盘失效线程运行时间间隔，默认是120秒。
diskPersistent：是否在VM重启时存储硬盘的缓存数据。默认值是false。
overflowToDisk：当内存中对象数量达到maxElementsInMemory时，Ehcache将会对象写到磁盘中。
diskSpoolBufferSizeMB：这个参数设置DiskStore（磁盘缓存）的缓存区大小。默认是30MB。每个Cache都应该有自己的一个缓冲区。
maxEntriesLocalDisk：当内存中对象数量达到maxElementsInMemory时，Ehcache将会对象写到磁盘中。
memoryStoreEvictionPolicy：当达到maxElementsInMemory限制时，Ehcache将会根据指定的策略去清理内存。
	默认策略是LRU（最近最少使用）。你可以设置为FIFO（先进先出）或是LFU（较少使用）。
-->
```
#### 3. application.properties
3.1 可以通过配置文件对CacheManager配置(同时需要在启动类加注解@EnableCaching)：
```yml
# ehcache
spring.cache.ehcache.config=classpath:config/ehcache.xml
```
3.2 也可以添加配置注解类EhCacheConfig(无需在启动类加注解@EnableCaching)：
```java
@Configuration
@EnableCaching
public class EhCacheConfig {

  // ehcache 主要的管理器
  @Bean
  public EhCacheCacheManager ehCacheCacheManager(EhCacheManagerFactoryBean bean) {
    return new EhCacheCacheManager(bean.getObject());
  }

  // 加载配置文件
  @Bean
  public EhCacheManagerFactoryBean ehCacheManagerFactoryBean() {
    EhCacheManagerFactoryBean cacheManagerFactoryBean = new EhCacheManagerFactoryBean();
    cacheManagerFactoryBean.setConfigLocation(new ClassPathResource("config/ehcache.xml"));
    cacheManagerFactoryBean.setShared(true);
    return cacheManagerFactoryBean;
  }
}
```
#### 4. 启动类添加注解`@EnableCaching`
```java
@EnableCaching
@SpringBootApplication
public class Application{
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```
#### 5. 代码中使用缓存
- @Cacheable	在方法执行前Spring先查看缓存中是否存在，如果存在，则直接返回缓存数据，若不存在则调用方法并将方法返回值放进缓存
- @CachePut	无论怎样，都会将方法的返回值放到缓存中。@CachePut的属性与@Cacheable保持一致
- @CacheEvict	将一条或多条数据从缓存中删除。
- @Caching	可以通过@Caching注解组合多个注解策略在一个方法上。

```java
@Service
public class UserService {

  //添加缓存
  @Cacheable(cacheNames = "username", key = "#id")
  public String getUserName(String id) {
    return "morgan";
  }

  //删除缓存
  @CacheEvict(value = "users",key = "#id")
  public void evictUser(Long id) {
      System.out.println("evict user:" + id);
  }
}
```
