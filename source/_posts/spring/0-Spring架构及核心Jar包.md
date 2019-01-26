title: Spring架构及核心Jar包

tags:
  - Spring

categories:
  - Spring
  - Spring架构及核心Jar包

---
### 一、Spring架构
下图是Spring4的架构图，去掉了Spring3的struts，添加了messaging和websocket。主体分为5个部分：core、aop、data access、web、test。
所有这些jar的“groupId”都是“org.springframework”，每个jar有一个不同的“artifactId”

<!--more-->

![structure](/img/spring/structure.png)

###  二、五个模块的核心Jar包
##### 1. core包含4个模块，下图是core依赖关系图：
- spring-core：依赖注入IoC与DI的最基本实现
- spring-beans：Bean工厂与bean的装配
- spring-context：spring的context上下文即IoC容器
- spring-expression：spring表达式语言

![yilaiguanxitu](/img/spring/jar-relation.jpg)

```xml
<!--所以普通java工程要使用spring框架，只需要一个Jar包：-->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>3.2.17.RELEASE</version>
</dependency>
<!--要在web工程中引入spring mvc，也只需要一个依赖-->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
    <version>3.2.17.RELEASE</version>
</dependency>
```

##### 2. aop包含4个模块
- spring-aop：面向切面编程
- spring-aspects：集成AspectJ
- spring-instrument：提供一些类级的工具支持和ClassLoader级的实现，用于服务器
- spring-instrument-tomcat：针对tomcat的instrument实现

##### 3. data access包含5个模块
- spring-jdbc：jdbc的支持
- spring-tx：事务控制
- spring-orm：对象关系映射，集成orm框架
- spring-oxm：对象xml映射
- spring-jms：java消息服务

##### 4. web包含3个模块
- spring-web：基础web功能，如文件上传
- spring-webmvc：mvc实现
- spring-webmvc-portlet：基于portlet的mvc实现(portlet:门户组件)

##### 5. test
- spring-test: spring测试，提供junit与mock测试功能

##### 额外支持包
- spring-context-support：spring额外支持包，比如邮件服务、视图解析等。不在架构图中
