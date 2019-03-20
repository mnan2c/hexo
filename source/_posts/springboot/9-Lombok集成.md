title: Lombok集成

tags:
  - SpringBoot

categories:
  - SpringBoot
---
## 简介
lombok是一个编译级别的插件，它可以在项目编译的时候生成一些代码，主要包括实体Bean中大量的Getter/Setter方法，以及toString, hashCode等可能不会用到，但是某些时候仍然需要复写，以期方便使用的方法
## 使用步骤

#### 1. 安装
- 参考1： [IDEA安装与配置](https://www.jianshu.com/p/c88b0f17f62a)
- 参考2： [STS安装与配置](https://blog.csdn.net/blueheart20/article/details/52909775)

#### 2. 添加依赖

```xml
<dependency>
     <groupId>org.projectlombok</groupId>
     <artifactId>lombok</artifactId>
 </dependency>
```

#### 3. 常用注解
```java
@NonNull //自动帮助我们避免空指针。作用在方法参数上的注解，用于自动生成空值参数检查
@Getter //自动生成Getter方法
@Setter //自动生成Setter方法
@ToString //覆盖tostring方法
@EqualsAndHashCode //覆盖equals和hashCode方法
@Data //自动为所有字段添加@ToString, @EqualsAndHashCode, @Getter，
//为非final字段添加@Setter,
//为final字段添加@RequiredArgsConstructor，
//本质上相当于几个注解的综合效果

@NoArgsConstructor //为类自动生成没有参数的constructor
@RequiredArgsConstructor//生成部分参数的constructor，一般用于生成final字段的构造方法
@AllArgsConstructor //生成所有参数的constructor
@Slf4j, @XSlf4j, @Log, @Log4j, @Log4j2, @CommonsLog, @JBossLog // 自动为类添加对应的log支持
@Cleanup //自动帮我们调用close()方法。作用在局部变量上，在作用域结束时会自动调用close方法释放资源
```

#### 4. 应用举例

```java
@Data
@Slf4j
public class LombokTest {
  private String id;
  private final String name;

  public static void main(String[] args) {
    LombokTest lombok = new LombokTest("morgan");
    // 这里可以看出@Data自动生成了toString()方法
    System.out.println("1. " + lombok);
    // 这里使用了@Slf4j，可以直接打印日志。
    log.debug("2. lombok is {}", lombok);
    Class clazz = LombokTest.class;
    // 由Constructors的结果可以看出@Data注解自动为我们生成了LombokTest(String name)的构造方法
    // 相当于添加了@RequiredArgsConstructor注解
    System.out.println("--------3. Constructors--------");
    Constructor[] constructors = clazz.getConstructors();
    for (Constructor cons : constructors) {
      System.out.println(cons);
    }
    // 由Methods的结果可以看出@Data没有为final字段name生成set方法
    System.out.println("--------4. Methods--------");
    Method ms[] = clazz.getMethods();
    for (Method m : ms) {
      System.out.println(m);
    }
  }

}

//输出结果：
1. LombokTest(id=null, name=morgan)
07:09:27.266 [main] DEBUG com.example.demo.config.LombokTest - 2. lombok is LombokTest(id=null, name=morgan)
--------3. Constructors--------
public com.example.demo.config.LombokTest(java.lang.String)
--------4. Methods--------
public static void com.example.demo.config.LombokTest.main(java.lang.String[])
public boolean com.example.demo.config.LombokTest.equals(java.lang.Object)
public java.lang.String com.example.demo.config.LombokTest.toString()
public int com.example.demo.config.LombokTest.hashCode()
public java.lang.String com.example.demo.config.LombokTest.getName()
public java.lang.String com.example.demo.config.LombokTest.getId()
public void com.example.demo.config.LombokTest.setId(java.lang.String)
...(省略继承自Object的方法)
```
