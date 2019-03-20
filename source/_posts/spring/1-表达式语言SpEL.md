title: SpEL表达式

tags:
  - Spring

categories:
  - Spring
  - SpEL表达式

---
### 简介
  - SpEL是Spring3中开始使用的表达式语言，可以在运行时构建复杂表达式，存取对象属性、对象方法调用，实现资源注入。可以在程序中单独使用，也可以在bean定义时使用。
  - 可以在xml和注解中使用。如果在xml中注册了bean和在java class中定义了@Value，@Value在运行时将失败。
  - SpEL使用#{···}作为定界符，所有在大括号里的字符都将被认为是SpEL。
<!--more-->

### 使用场景
  1. @Value常用使用方式

```java
@Configuration
@ComponentScan("com.example.blog_test")
@PropertySource("classpath:static/spel.properties")
public class ElConfig {
   @Value("I LOVE YOU!") // 注入字符串
   private String normal;

   @Value("#{systemProperties['os.name']}") // 获取操作系统名
   private String osName;

   //如果要在SpEL中访问类作用域的方法和常量的话，要依赖T()这个关键的运算符
   @Value("#{T(java.lang.Math).random() * 100.0}") // 注入表达式结果
   private double randomNumber;

   @Value("#{spELService.another}") // 注入其他Bean的属性
   private String fromAnother;

   @Value("${project.name}") // 注入配置文件
   private String projectName;

   // 注意这个Resource是:org.springframework.core.io.Resource;
   // 使用方式： IOUtils.toString(testFile.getInputStream())
   @Value("classpath:static/spel.txt")
   private Resource testFile;

   // 注入配置文件
   // 使用方式: environment.getProperty("project.author")
   @Autowired
   private Environment environment;

   // 注入网址资源
   // 使用方式： IOUtils.toString(testUrl.getInputStream())
   @Value("http://www.baidu.com")
   private Resource testUrl;
 }
```
  2. Xml中的使用

```xml
<beans xmlns=...>

    <bean id="itemBean" class="com.lei.demo.el.Item">
        <property name="name" value="itemA" />
        <property name="total" value="10" />
    </bean>

    <!--#{itemBean.name}——将itemBean 的name属性，注入到customerBean的属性itemName中。-->
    <bean id="customerBean" class="com.lei.demo.el.Customer">
        <property name="item" value="#{itemBean}" />
        <property name="itemName" value="#{itemBean.name}" />
        <property name="name" value="#{'lei'.toUpperCase()}" />
        <property name="amount" value="#{itemBean.getSpecialPrice()}" />
    </bean>

</beans>
```
  3. 注解使用

```java
  // Item.java如下：
  @Component("itemBean")
  public class Item {

      @Value("itemA")//直接注入String
      private String name;

      @Value("10")//直接注入integer
      private int total;
  }

   // Customer.java:
    @Component("customerBean")
    public class Customer {

        @Value("#{itemBean}")
        private Item item;

        @Value("#{itemBean.name}")
        private String itemName;
    }
```
  4. 方法调用
    SpEL允许将方法的返回值注入到属性中

```java
@Component("customerBean")
public class Customer {

    @Value("#{'lei'.toUpperCase()}")
    private String name;

    @Value("#{priceBean.getSpecialPrice()}")
    private double amount;

    //selectArtist()的返回值是null的话，那么SpEL将不会调用toUpperCase()方法。表达式的返回值会是null。
    @Value("#{priceBean.selectArtist()?.toUpperCase()}")  
    private List<String> artList;
  }
```
  5. 操作符
    - 关系操作符: <、>、==、<=、>=、lt、gt、eq、le、ge
    - 逻辑操作符: and、or、not、|
    - 数学操作符: +、-、\\\*、/、%、^
    - 正则表达式: matches

```java
@Component("customerBean")
public class Customer {

    @Value("#{1 != 1}") //false
    private boolean testNotEqual;

    @Value("#{numberBean.no == 999 and numberBean.no < 900}") //false
    private boolean testAnd;

    @Value("#{!(numberBean.no == 999)}") //false
    private boolean testNot;

    @Value("#{'1' + '@' + '1'}") //1@1
    private String testAddString;

    @Value("#{10 / 2}") //5.0
    private double testDivision;

    @Value("#{10 % 10}") //0.0
    private double testModulus ;

    @Value("#{2 ^ 3}") //8.0
    private double testExponentialPower;

    @Value("#{admin.email matches '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-+\\.com]'}")
    private boolean validEmail;
  }
```
  6. 三目运算符
  SpEL支持三目运算符，以此来实现条件语句。

```java
@Component("customerBean")
public class Customer {

    @Value("#{itemBean.qtyOnHand < 100 ? true : false}")
    private boolean warning;
  }
```
  7. 操作List、Map集合取值

```java
//get map where key = 'MapA'
@Value("#{testBean.userMap['MapA']}")
private String mapA;

//get first value from list, list is 0-based.
@Value("#{testBean.users[0]}")
private String list;

//SpEL还提供了查询运算符（.?[])
//对集合进行过滤，得到集合的一个子集
//假设你希望得到jukebox中artist属性为Aerosmith的所有歌曲
@Value("#{jukebox.songs.?[artist eq 'Aeronsmith']}")
private List<Song> subList;

//SpEL还提供了另外两个查询运算符：”.^[]“和”.$[]“,它们分别用来在集合中查询第一个匹配项和最后一个匹配项
@Value("#{jukebox.songs.^[artist eq 'Aerosmith']}")
private Song first;

//SpEL还提供了投影运算符（.![]）,它会从集合的每个成员中选择特定的属性放到另外一个集合中
//假设不想要歌曲对象的集合，而是过滤后的歌曲名称的集合
@Value("#{jukebox.songs.?[artist eq 'Aerosmith'].![title]}")
private List<String> titles;
```
