title: IoC与DI思想

tags:
  - Spring

categories:
  - Spring
  - IoC与DI思想

---
### 概念
- IoC是一种思想。传统应用程序是由我们自己在对象中主动控制去直接获取依赖对象，而反转则是由容器来帮忙创建及注入依赖对象；因为由容器帮我们查找及注入依赖对象，对象只是被动的接受依赖对象，所以称为反转。
- 优势：把创建和查找依赖对象的控制权交给IoC容器，由容器进行注入组合对象，所以对象与对象之间是松散耦合，利于功能复用，更重要的是使得程序的整个体系结构变得非常灵活。
<!--more-->

### IoC的技术实现
- 所谓依赖注入，就是把底层类作为参数传入上层类，实现上层类对下层类的“控制”。
- IoC容器在实现依赖注入的时候先从最上层开始往下找依赖关系，到达最底层之后再往上一步一步new。这里IoC容器直接隐藏掉具体的创建实例的细节，我们看来他就像一个工厂，如下图：

![ioc](/img/spring/ioc.png)
- 实现依赖注入的两种方式：构造器注入、setter注入:
```java
  //1. 构造器注入：
public Person(Food food) {
    this.food = food;
}

 //2. setter注入
public void setFood(Food food) {
    this.food = food;
}
```

### IOC容器
IOC需要实现两大功能：对象的构建与绑定。对于这两个方面技术的实现具有很多的方式：硬编码（Ioc 框架都支持），配置文件（我们的重点），注解（最洁的方式），但无论哪种方式都是在Ioc容器里面实现的。spring提供了两种类型的容器，一个是BeanFactory,一个是ApplicationContext(继承自BeanFactory)。
##### 一、BeanFactory
BeanFactory默认采用延迟初始化策略，只有当客户端对象需要访问容器中的某个受管对象的时候，才对该对象进行初始化及依赖注入操作。BeanFactory接口的关系图：

![BeanFactory图](/img/spring/beanfactory1.png)

有三个很重要的部分：
- BeanDefinition 实现Bean及其依赖的定义（即对象的定义），完成了Bean的生成过程。一般情况下我们都是通过配置文件（xml,properties）的方式对bean进行配置，每种文件都需要实现BeanDefinitionReader，因此是reader本身实现了配置文字到bean对象的转换过程。
- BeanDefinitionRegistry ，将定义好的bean注册到容器中；
- BeanFactory 是一个bean工厂类，从中可以取到任意定义过的bean；

Bean的生成大致可以分为两个阶段：容器启动阶段和Bean实例化阶段：

1.容器启动阶段：
- 加载配置文件（通常是xml文件）
- 通过reader生成beandefinition
- beanDefinition注册到beanDefinitionRegistry

2.bean实例化阶段
- 当某个bean 被 getBean()调用时，bean及其依赖需要完成初始化。
`说明`：bean的实例化用到了aop机制，生成了其代理对象（通过反射机制生成接口对象，或者通过CGLIB生成子对象）。bean的具体装载过程是由beanWrapper实现的，它继承了PropertyAccessor （可以对属性进行访问）、PropertyEditorRegistry 和TypeConverter接口 （实现类型转换）。

##### 二、ApplicationContext
ApplicationContext容器建立BeanFactory基础之上，拥有BeanFactory的所有功能，但在实现上主要有两方面差别：1.bean的生成方式；2.扩展了BeanFactory的功能，提供了更多企业级功能的支持。

1. Bean的加载方式。
ApplicationContext提供几个读取配置文件的方式：

  - FileSystemXmlApplicationContext：容器从XML文件中加载已被定义的bean，需要提供给构造器XML文件的完整路径
  - ClassPathXmlApplicationContext：容器从XML文件中加载已被定义的bean，这里只需正确配置CLASSPATH环境变量即可，因为容器会从CLASSPATH中搜索bean配置文件。
  - WebXmlApplicationContext：容器会在一个web应用程序的范围内加载在XML文件中已被定义的bean。

  另外一个比较重要的是，ApplicationContext采用的非懒加载方式。它会在启动阶段完成所有的初始化，并不会等到getBean()才执行。所以，相对于BeanFactory来说，ApplicationContext要求更多的系统资源，容器启动时间较之BeanFactory也会长一些。在那些系统资源充足，并且要求更多功能的场景中， ApplicationContext类型的容器是比较合适的选择。
2. ApplicationContext还额外增加了三个功能：``ResourceLoader，ApplicationEventPublisher，MessageResource``，如下图：

![ApplicationContext](/img/spring/appcontext.png)

- ResourceLoader实现统一资源定位、资源加载
- ApplicationEventPublisher：ApplicationContext实现了该接口，因此spring的ApplicationContext实例具有发布事件的功能（publishEvent方法在AbstractApplicationContext中有实现）
- MessageSource：提供国际化支持
