title: properties文件常用配置
tags:
  - SpringBoot
categories:
  - SpringBoot
---
  下面列表中主要是配置文件中关于web，日志等的一些常用配置。
<!--more-->

|  |  配置项  |	默认值	| 	说明	 || --- |	 --- 	| 	---	 |	 ---	 |
|spring web:|server.port                  | ||||debug                        | false||||server.servlet.context-path  | / ||||spring.http.encoding.charset | utf-8 ||||spring.thymeleaf.cache       | true ||||spring.mvc.date-format       ||  日期输入格式  |||spring.jackson.date-format   ||  json输出格式  |||spring.jackson.time-zone     ||  设置GMT时区 ||日志|log4j -> logback|||||  logging.file  |||||  logging.level.ROOT |||||  logging.level.* |||||  logging.config  | logback.xml | logback-spring.xml可以使springboot在启动时进行日志的环境切换  || 环境切换  | spring.profiles.active  ||   相同配置以基本配置文件为准  ||  自定义配置 |  @Value || 在config.properties中配置 @Value("${app.name}")， 入口类启动时加载@PropertySource("classpath:config.properties")||| @ConfigurationProperties  ||新建AppConfig.java,加入注解@Component以及@ConfigurationProperties(prefix="")|
