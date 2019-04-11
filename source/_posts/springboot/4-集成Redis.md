title: SpringBoot2.0集成Redis

tags:
  - SpringBoot

categories:
  - SpringBoot

---
## 集成Redis步骤

#### 1. pom添加redis依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

#### 2. application.yml 中添加redis配置：

```yml
spring:
    redis:
        database: 0 # Redis数据库索引（默认为0）,如果设置为1，那么存入的key-value都存放在select 1中
        host: localhost # 服务器地址，部署之后需要修改
        port: 6379
        password: #默认为空
        timeout: 30000 #毫秒
        jedis:
          pool:
            max-active: 8 #连接池最大连接数（负值表示没有限制）
            max-wait: -1 # 连接池最大阻塞等待时间（负值表示没有限制）
            max-idle: 8 # 连接池中的最大空闲连接
            min-idle: 0 # 连接池中的最小空闲连接
```

#### 3. 配置Redis

在加入spring-boot-starter-data-redis依赖后，Spring Boot就已经默认为我们配置了JedisConnectionFactory（客户端连接）、RedisTemplate和StringRedisTemplate（数据模板操作），其中StringRedisTemplate模板只针对键值对都是字符型的数据进行操作。不过默认的配置存在一定的问题，需要我们进行修改。
```java
@Configuration
@EnableCaching
public class RedisConfig extends CachingConfigurerSupport {

  /**
   * 默认情况下，key值存在问题，当方法入参相同时，key值也相同，
   * 这样会造成不同的方法读取相同的缓存，从而造成异常。
   * 修改后的key值为className+methodName+参数值列表，可以支持使用@Cacheable时不指定Key
   */
  @Bean
  public KeyGenerator keyGenerator() {
    return new KeyGenerator() {

      @Override
      public Object generate(Object target, Method method, Object... params) {
        StringBuilder sb = new StringBuilder();
        sb.append(target.getClass().getName());
        sb.append(method.getName());
        for (Object object : params) {
          sb.append(object.toString());
        }
        return sb.toString();
      }
    };
  }

  // 管理缓存
  @SuppressWarnings({"rawtypes", "unchecked"})
  @Bean
  @Primary //如果配置了多个cacheManager的bean，可以通过该注解选择优先使用的cache config。
  public CacheManager cacheManager(RedisTemplate redisTemplate) {
    // RedisCache需要一个RedisCacheWriter来实现读写Redis
    RedisCacheWriter writer = RedisCacheWriter.lockingRedisCacheWriter(redisTemplate.getConnectionFactory());
    // 配置key和value的序列化类；
    // SerializationPair用于Java对象和Redis之间的序列化和反序列化
    // Spring Boot默认采用JdkSerializationRedisSerializer的二进制数据序列化方式
    // 使用该方式，保存在redis中的值是人类无法阅读的乱码，并且该Serializer要求目标类必须实现Serializable接口

    // 本示例中，使用StringRedisSerializer来序列化和反序列化redis的key值
    RedisSerializationContext.SerializationPair keySerializationPair =
        RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer());

    // 使用Jackson2JsonRedisSerializer来序列化和反序列化redis的value值
    Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer<>(Object.class);
    ObjectMapper om = new ObjectMapper();
    om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
    om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
    jackson2JsonRedisSerializer.setObjectMapper(om);
    RedisSerializationContext.SerializationPair valueSerializationPair =
        RedisSerializationContext.SerializationPair.fromSerializer(jackson2JsonRedisSerializer);

    // 构造一个RedisCache的配置对象，设置缓存过期时间和Key、Value的序列化机制
    // 这里设置缓存过期时间为1天
    RedisCacheConfiguration redisCacheConfiguration = //
        RedisCacheConfiguration //
            .defaultCacheConfig() //
            .entryTtl(Duration.ofDays(1)) //
            .serializeKeysWith(keySerializationPair) //
            .serializeValuesWith(valueSerializationPair) //
            .disableCachingNullValues();
    return new RedisCacheManager(writer, redisCacheConfiguration);
  }
}
```

#### 4. 使用Redis

```java
@RestController
@RequestMapping("/users")
public class UserController {

  @GetMapping("/{id}")
  // @Cacheable(cacheNames = "username", key = "#id") // ehcache
  @Cacheable(value = "username", key = "#root.targetClass + #id", unless = "#result eq null")
  public Map<String, Object> getUserName(@PathVariable("id") String id) {
    Map<String, Object> result = new HashMap<>();
    result.put("id", id);
    result.put("name", "morgan" + id);
    return result;
  }
}

//调用： http://localhost:8888/users/002
结果： {"name":"morgan002","id":"002"}
```
這是redis缓存結果：
![](/img/springboot/redis-result.png)

#### 补充

1. 使用redis的时候用到了SPEL表达式，之前[这里](http://www.clemon.top/2018/08/06/spring/1-%E8%A1%A8%E8%BE%BE%E5%BC%8F%E8%AF%AD%E8%A8%80SpEL/)也有记录过，这里补充一部分：

|表达式|描述|
|---|---|
|#root.targetClass|当前被调用的目标对象类|
|#root.target|当前被调用的目标对象|
|#root.methodName|当前被调用的方法名|
|#root.method.name|当前被调用的方法|
|#root.args[0]|当前被调用的方法的参数列表|
|#id|当前被调用的方法的参数|
|#result|方法执行后的返回值（仅当方法执行之后的判断有效，如‘unless’，’cache evict’的beforeInvocation=false）|
|#root.caches[0].name|当前方法调用使用的缓存列表（如@Cacheable(value={“cache1”, “cache2”})），则有两个cache|

2. cache常用表达式

```java
//在执行完方法后（#result就能拿到返回值了）判断condition，如果返回true，则放入缓存；
@CachePut(value = "user", key = "#id", condition = "#result.username ne 'zhang'")  
public User conditionSave(final User user)
  //
//beforeInvocation=false表示在方法执行之后调用（#result能拿到返回值了）；且判断condition，如果返回true，则移除缓存；
@CacheEvict(value = "user", key = "#user.id", beforeInvocation = false, condition = "#result.username ne 'zhang'")  
public User conditionDelete(final User user)
```

3. 自定义缓存注解

```java
@Caching(  
        put = {  
                @CachePut(value = "user", key = "#user.id"),  
                @CachePut(value = "user", key = "#user.username"),  
                @CachePut(value = "user", key = "#user.email")  
        }  
)  
@Target({ElementType.METHOD, ElementType.TYPE})  
@Retention(RetentionPolicy.RUNTIME)  
@Inherited  
public @interface UserSaveCache {  
}  
//使用
@UserSaveCache  
public User save(User user)
```

4. 通过redis实现session共享
分布式系统中，session共享有很多的解决方案，其中托管到缓存中应该是最常用的方案之一，使用方法：

- 1.引入依赖：

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```
- 2.Session配置

```java
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 86400*30)
public class SessionConfig {
}
```
- 3.测试

```java
//该方法运行时，redis会对每次产生的session进行管理。
//所以，可以通过在另一个项目中使用相同配置，进行session共享。
@RequestMapping("/uid")
String uid(HttpSession session) {
    UUID uid = (UUID) session.getAttribute("uid");
    if (uid == null) {
        uid = UUID.randomUUID();
    }
    session.setAttribute("uid", uid);
    return session.getId();
}
```
