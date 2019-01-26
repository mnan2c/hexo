title: Spring AOP

tags:
  - Spring

categories:
  - Spring

---

### 一、AOP
- 通过对目标对象的代理在连接点前后进行增强，完成统一的切面操作，是OOP的补充和完善。
- AOP把软件系统分为两个部分：核心关注点和横切关注点。业务处理的主要流程是核心关注点，与之关系不大的部分是横切关注点（权限控制、缓存控制、事务控制、审计日志、性能监控、分布式追踪、异常处理）。
- AOP切面的优先级：`@Order(i)`注解来标识切面的优先级。i的值越小，优先级越高。在切入点前的操作，按order的值由小到大执行；在切入点后的操作，按order的值由大到小执行。

### 二、AOP应用控制的三大注解
这里主要结合三个注解理解它的控制应用：@Transactional、@PreAuthorize、@Cacheable.

1. @Transactional注解进行事务控制
2. SpringSecurity安全校验同样使用了AOP，下面是一个简单的用户登陆示例：

```java
//1.配置文件
import org.springframework.security.config.annotation.*;

@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                .antMatchers("/", "/index").permitAll()//这是不需要校验的，其他的都需要校验
                .anyRequest().authenticated()
                .and()
                .httpBasic()
                .and()
                .logout()
                .permitAll();
    }
    @Autowired
    public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
        auth
                .inMemoryAuthentication()
                //默认在内存中创建两个用户，一个是USER角色的用户demo,另一个是ADMIN角色的用户admin
                .withUser("demo").password("demo").roles("USER")
                .and()
                //
                .withUser("admin").password("admin").roles("ADMIN");
    }
}

//2.控制器
import org.springframework.security.access.prepost.PreAuthorize;

//index页面是每个人都可以访问的，common页面允许demo用户访问，admin页面只允许admin用户访问
@RestController
public class DemoController {

    //这个前面已经设置过了，是不需要拦截的
    @RequestMapping("/index")
    public String index() {
        return "index";
    }

    @RequestMapping("/common")
    public String commonAccess(){
        return "only login can view";
    }

    @RequestMapping("/admin")
    @PreAuthorize("hasRole('ROLE_ADMIN')")
    public String admin(){
        return "only admin can access";
    }
}
```
3. SpringCache利用@Cacheable进行缓存控制

```java
//1.缓存配置文件
import org.springframework.cache.annotation.Cacheable;

@Component
public class MenuService {

    //@Cacheable是用来声明方法返回值是可缓存的。
    //后续使用相同参数调用时不需执行实际的方法，直接从缓存中取值。
    @Cacheable(cacheNames = {"menu"})
    public List<String> getMenuList(){
        System.out.println("mock:get from db");
        return Arrays.asList("article","comment","admin");
    }
}

//2.主文件
import org.springframework.cache.annotation.EnableCaching;

@SpringBootApplication
@EnableCaching//这个注解不能漏
public class CacheDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(CacheDemoApplication.class, args);
    }
}

//3.测试文件
@RunWith(SpringRunner.class)
@SpringBootTest
public class CacheDemoApplicationTests {

    @Autowired
    MenuService menuService;

    @Test
    public void testCache() {
    //首次调用getMenuList，缓存中没有东西，所以完整执行了
     System.out.println("call:"+menuService.getMenuList());
    //第二次调用时，直接去缓存中获取了第一次调用的返回值List，并没有执行之前的打印文本
     System.out.println("call:"+menuService.getMenuList());
    }
}
//运行结果：
//mock:get from db
//call:[article, comment, admin]
//call:[article, comment, admin]
```

### 三、Spring AOP中使用的两种动态代理
- jdk动态代理，主要通过反射跟动态编译来实现，核心是InvocationHandler和Proxy，JDK动态代理只能针对实现了接口的类生成代理。JVM在运行时，知道要代理谁后，通过指定类加载器、接口数组、调用处理程序这3个参数来动态生成代理。
- CGLib采用了非常底层的字节码技术，其原理是通过字节码技术为一个类创建子类，并在子类中采用方法拦截的技术拦截所有父类方法的调用，顺势织入横切逻辑，因此又名子类代理。

### 四、Spring动态代理实例
 1. Spring AOP实现原理：Spring中AOP代理由Spring的IOC容器负责生成、管理，其依赖关系也由IOC容器负责管理。因此，AOP代理可以直接使用容器中的其它bean实例作为目标，这种关系可由IOC容器的依赖注入提供。进行AOP编程的关键就是定义切入点和定义增强处理，一旦定义了合适的切入点和增强处理，AOP框架将自动生成AOP代理。一般我们称切入到指定类指定方法的代码片段称为切面，而切入到哪些类、哪些方法则叫切入点。
 2. 实现
  - 定义接口及实现类

```java
public interface UserDao {
  public void add(User user);
}

@Component("u")
public class UserDaoImpl implements UserDao {
  @Override
  public void add(User user) {
      System.out.println("add user!");
  }
}

@Component
public class UserService {
    private UserDao userDao;

    public void add(User user) {
        userDao.add(user);
    }

    public UserDao getUserDao() {
        return userDao;
    }

    @Resource(name = "u")
    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }
  }
```
  - 定义横切关注点的类

```java
  //描述切面类
  //作为切入点签名的方法必须返回void 类型
  @Aspect
  @Component
  public class LogInterceptor {

    // 为了使代码更简洁，可以先定义一个Pointcut，然后直接拦截这个方法
    // 这里就定义了一个总的匹配规则，以后拦截的时候直接拦截log()方法即可，无须去重复写execution表达式
    @Pointcut("execution(public * com.imooc.controller.GirlController.*(..))")
    public void log() {}

    @Before("log()")
    public void doBefore() {
        System.out.println("******拦截前的逻辑******");
    }

    @After("log()")
    public void doAfter() {
        System.out.println("******拦截后的逻辑******");
    }

    @AfterReturning(pointcut="log()", returning="returnValue")
    public void log(String message, Object returnValue) {
        //do something...
    }
  }
```
  3. Spring 的配置文件如下，通过aop命名空间的<aop:aspectj-autoproxy />声明自动为spring容器中那些配置@aspectJ切面的bean创建代理，织入切面

```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="...">
     <context:annotation-config></context:annotation-config>
     <context:component-scan base-package="com.syf">></context:component-scan>
     <aop:aspectj-autoproxy />
  </beans>
```
  4. 测试类及输出：

```java
public class UserServiceTest {

  @Test
  public void testAdd() throws Exception{
      @SuppressWarnings("resource")
      ApplicationContext applicationContext = new ClassPathXmlApplicationContext("beans.xml");
      UserService svc = (UserService) applicationContext.getBean("userService");
      User u = new User();
      u.setId(1);
      u.setName("name");
      svc.add(u);
  }
}
//输出：
  around start method
  method start
  add user!   //add 方法实现的内容
  around end method
  method end
  method after returning
```

### 五、使用CGLib
1. 使用Cglib子类代理,需要引入cglib.jar包. 在Spring的核心包中已经为包含了Cglib功能,所以直接引入Spring-core.jar即可。当我们需要使用CGLIB来实现AOP的时候，需要配置`spring.aop.proxy-target-class=true`，不然默认使用的是标准Java的实现。
2. 目标对象中的方法如果被 final/static 关键字修饰, 那么被修饰的方法在被调用时就不会被MethodIntercept拦截器拦截，即不会执行目标对象额外的业务方法。
3. CGLib 类库可以代理没有接口的类，这样就弥补了 JDK 的不足:

```java
public class CGLibDynamicProxy implements MethodInterceptor {

    private static CGLibDynamicProxy instance = new CGLibDynamicProxy();

    private CGLibDynamicProxy() {
    }

    public static CGLibDynamicProxy getInstance() {
        return instance;
    }

    @SuppressWarnings("unchecked")
    public <T> T getProxy(Class<T> cls) {
        return (T) Enhancer.create(cls, this);
    }

    @Override
    public Object intercept(Object target, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        before();
        Object result = proxy.invokeSuper(target, args);
        after();
        return result;
    }

    private void before() {
        System.out.println("Before");
    }

    private void after() {
        System.out.println("After");
    }
}
```
4. 以上代码中了 Singleton 模式，那么客户端调用也更加轻松了：

```java
public class Client {

    public static void main(String[] args) {
        Greeting greeting = CGLibDynamicProxy.getInstance().getProxy(GreetingImpl.class);
        greeting.sayHello("Jack");
    }
}
```

### 六、SpringBoot与AOP（基于注解的aop）
  1. SpringBoot可以直接使用aop，不需要任何配置，只需要添加依赖：

```xml
 <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
 </dependency>
```
  2. 注解@Pointcut，使用一个返回值为void，方法体为空的方法来命名切入点，方法名即为切点名。切入点表达式的格式：`execution([可见性] 返回类型 [声明类型].方法名(参数) [异常])`
  3. Advice注解：@Before, @After, @AfterReturning, @AfterThrowing, @Around
  4. 通配符匹配规则（包、类、对象、参数、注解）:

```
*：匹配所有字符
..：一般用于匹配多个包，多个参数
+：表示类及其子类
运算符有：&&、||、!
```

##### 10.1 切入点使用示例
```java
  1. execution(方法表达式)
    //cn.javass包及所有子包下IPointcutService接口中的任何无参方法
    * cn.javass..IPointcutService.*()
    //非“cn.javass包及所有子包下IPointcutService接口及子类型”的任何方法
    * (!cn.javass..IPointcutService+).*(..)
  2. within(类型表达式)
    //持有cn.javass..Secure注解的任何类型的任何方法
    // 必须是在目标对象上声明这个注解，在接口上声明的对它不起作用
    within(@cn.javass..Secure *)
  3. this(类型全限定名)
    //当前AOP对象实现了 IPointcutService接口的任何方法
    //是AOP代理对象的类型匹配，这样接口方法也可以匹配；
    //this中使用的表达式必须是类型全限定名，不支持通配符
    this(cn.javass.spring.chapter6.service.IPointcutService)
  4. target(类型全限定名)
    // 当前目标对象（非AOP对象）实现了 IPointcutService接口的任何方法
    target(cn.javass.spring.chapter6.service.IPointcutService)
  5. args(参数类型列表)
    // 任何一个以接受“传入参数类型为 java.io.Serializable” 开头，且其后可跟任意个任意类型的参数的方法执行，args指定的参数类型是在运行时动态匹配的
    args (java.io.Serializable,..)
```
  6. @within：使用“@within(注解类型)”匹配所以持有指定注解类型内的方法
  7. @target：使用“@target(注解类型)”匹配当前目标对象类型的执行方法
  8. @args：使用“@args(注解列表)”匹配当前执行的方法传入的参数持有指定注解的执行
  9. @annotation：使用“@annotation(注解类型)”匹配当前执行方法持有指定注解的方法
  10. bean：使用“bean(Bean id或名字通配符)”匹配特定名称的Bean对象的执行方法
  11. reference pointcut：表示引用其他命名切入点，只有@ApectJ风格支持，Schema风格不支持
