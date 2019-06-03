title: OAuth2 （二） - SpringBoot2.0集成

tags:
  - SpringBoot

categories:
  - SpringBoot
---
## 步骤

#### 1. pom添加依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<!-- 注意是starter,自动配置 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<!-- 不是starter,手动配置 -->
<dependency>
    <groupId>org.springframework.security.oauth</groupId>
    <artifactId>spring-security-oauth2</artifactId>
    <version>2.3.2.RELEASE</version>
</dependency>
```

#### 2. application.yml

token的存储一般选择使用redis，一是性能比较好，二是自动过期的机制，符合token的特性。

```yml
server:
    port: 8888

spring:
    redis:
        database: 0 # Redis数据库索引（默认为0）,如果设置为1，那么存入的key-value都存放在select 1中
        host: localhost # 服务器地址，部署之后需要修改
        port: 6379
        password: #默认为空
        timeout: 30000 #毫秒
        jedis:
          pool:
            max-active: 8 #连接池最大连接数（使用负值表示没有限制）
            max-wait: -1 # 连接池最大阻塞等待时间（使用负值表示没有限制）
            max-idle: 8 # 连接池中的最大空闲连接
            min-idle: 0 # 连接池中的最小空闲连接
```

#### 3. 配置资源服务器和授权服务器

由于是两个oauth2的核心配置，所以放到一个配置类中。 为了方便下载代码直接运行，这里将客户端信息放到了内存中，生产中可以配置到数据库中。

```java
@Configuration
public class OAuth2ServerConfig {

  private static final String DEMO_RESOURCE_ID = "demo-for-aouth2";

  @Configuration
  @EnableResourceServer
  protected static class ResourceServerConfig extends ResourceServerConfigurerAdapter {

    @Override
    public void configure(ResourceServerSecurityConfigurer resources) {
      resources //
          .resourceId(DEMO_RESOURCE_ID) //
          .stateless(true);
    }

    @Override
    public void configure(HttpSecurity http) throws Exception {
      http //
          .authorizeRequests() //
          .antMatchers("/users/**") //
          // .access("#oauth2.hasScope('get_user_info')"); //通过scope进行访问控制
          .authenticated(); // 配置user访问控制，必须认证过后才可以访问
    }
  }

  @Configuration
  @EnableAuthorizationServer
  protected static class AuthServerConfig extends AuthorizationServerConfigurerAdapter {

    @Inject
    AuthenticationManager authenticationManager;
    @Inject
    RedisConnectionFactory redisConnectionFactory;

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {

      // 支持多种编码，通过密码的前缀区分编码方式
      String finalSecret = new BCryptPasswordEncoder().encode("123456");
      // 配置两个客户端,一个用于password认证一个用于client认证
      clients.inMemory() //
          .withClient("client_1") //
          .resourceIds(DEMO_RESOURCE_ID) //
          .authorizedGrantTypes("client_credentials", "refresh_token") //
          .scopes("select", "get_user_info") //
          .authorities("oauth2") //
          .secret(finalSecret) //
          .and() //
          .withClient("client_2") //
          .resourceIds(DEMO_RESOURCE_ID) //
          .authorizedGrantTypes("password", "refresh_token") //
          .scopes("select", "get_user_info") //
          .authorities("oauth2") //
          .secret(finalSecret);
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
      endpoints //
          .tokenStore(new RedisTokenStore(redisConnectionFactory)) //
          .authenticationManager(authenticationManager) //
          .userDetailsService(userDetailsService) //
          .allowedTokenEndpointRequestMethods(HttpMethod.GET, HttpMethod.POST);
    }

    @Override
    public void configure(AuthorizationServerSecurityConfigurer oauthServer) {
      // 允许表单认证
      oauthServer.allowFormAuthenticationForClients();
    }
  }

}
```

#### 4. 配置Spring Security
使用了springboot之后，spring security其实是有不少自动配置的，我们可以仅仅修改自己需要的那一部分，并且遵循一个原则，直接覆盖最需要的那一部分。

为了方便运行，使用内存中的用户，实际项目中，一般使用的是数据库保存用户，具体的实现类可以使用JdbcDaoImpl或者JdbcUserDetailsManager。

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

  @Bean
  @Override
  protected UserDetailsService userDetailsService() {
    BCryptPasswordEncoder bCryptPasswordEncoder = new BCryptPasswordEncoder();
    // 支持多种编码，通过密码的前缀区分编码方式
    String finalPassword = bCryptPasswordEncoder.encode("123456");
    InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
    manager.createUser(User.withUsername("user_1").password(finalPassword).authorities("USER").build());
    manager.createUser(User.withUsername("user_2").password(finalPassword).authorities("USER").build());
    return manager;
  }

  @Bean
  public PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
  }

  @Bean
  @Override
  public AuthenticationManager authenticationManagerBean() throws Exception {
    AuthenticationManager manager = super.authenticationManagerBean();
    return manager;
  }

  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http //
        .requestMatchers() //
        .anyRequest() //
        .and() //
        .authorizeRequests() //
        .antMatchers("/oauth/**").permitAll();
  }
}

//进行如上配置之后，启动springboot应用就可以发现多了一些自动创建的接口：
// {[/oauth/authorize]}
// {[/oauth/authorize],methods=[POST]
// {[/oauth/token],methods=[GET]}
// {[/oauth/token],methods=[POST]}
// {[/oauth/check_token]}
// {[/oauth/error]}
```

#### 5. 添加受到访问控制的接口

```java
@RestController
@RequestMapping("/users")
public class UserController {

  @GetMapping("/{id}")
  // @Cacheable(cacheNames = "username", key = "#id") // ehcache
  // @Cacheable(value = "username", key = "#root.targetClass + #id", unless = "#result eq null")
  public Map<String, Object> getUserName(@PathVariable("id") String id) {
    // for debug
    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
    Map<String, Object> result = new HashMap<>();
    result.put("id", id);
    result.put("name", "morgan" + id);
    return result;
  }
}
```

#### 6. 测试

```js
// 1. password 模式获取token
// Request:
GET
http://localhost:8888/oauth/token?username=user_1&password=123456&grant_type=password&scope=get_user_info&client_id=client_2&client_secret=123456
// Response:
{
    "access_token": "d83e22c7-49cb-4d7d-90eb-c2ccb73e1e8a",
    "token_type": "bearer",
    "refresh_token": "7f71f7a5-1710-4128-8bba-29d088e93d6c",
    "expires_in": 43199,
    "scope": "get_user_info"
}

// 2. client 模式获取token
// Request:
GET
http://localhost:8888/oauth/token?grant_type=client_credentials&scope=select&client_id=client_1&client_secret=123456
// Response:
{
    "access_token": "1c2287e0-ddb2-405f-8f09-17e6da74aeff",
    "token_type": "bearer",
    "expires_in": 43199,
    "scope": "select"
}

// 3. 获取refresh token
// Request:
GET
http://localhost:8888/oauth/token?grant_type=refresh_token&refresh_token={{refresh_token}}&client_id=client_2&client_secret=123456
// Response:
{
    "access_token": "003a5510-4375-4783-93e4-8fbded7b58bf",
    "token_type": "bearer",
    "refresh_token": "7f71f7a5-1710-4128-8bba-29d088e93d6c",
    "expires_in": 43199,
    "scope": "get_user_info"
}

// 4. 调用/users/{id} api，访问受保护的资源：
// Request:
GET
http://localhost:8888/users/003?access_token={{access_token}}
// Response:
{
    "name": "morgan003",
    "id": "003"
}
```
