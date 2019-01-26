title: SpringMVC拦截器

tags:
  - Spring

categories:
  - Spring

---
Spring WebMvc框架中的Interceptor，与Servlet API中的Filter十分类似，用于对Web请求进行预处理/后处理，例如：
- 记录Web请求相关日志，可以用于做一些信息监控、统计、分析
- 检查Web请求访问权限，例如发现用户没有登录后，重定向到登录页面
- 打开/关闭数据库连接——预处理时打开，后处理关闭
<!--more-->

### 拦截器
拦截器主要配置两个地方：1. HandlerInterceptor，为什么拦截；2. WebMvcConfigurerAdapter，在什么地方拦截；

##### 1. HandlerInterceptor
Spring MVC中拦截器是实现了HandlerInterceptor接口的Bean。HandlerInterceptorAdapter是一个实现了HandlerInterceptor的抽象类，它的三个实现方法都为空实现（或者返回true），继承该抽象类后可以仅仅实现其中的一个方法preHandle。

```java
public interface HandlerInterceptor {
  //预处理回调方法，若方法返回值为true，请求继续（调用下一个拦截器或处理器方法）；
  //若方法返回值为false，请求处理流程中断，不会继续调用其他的拦截器或处理器方法，
  //此时需要通过response产生响应；
    boolean preHandle(HttpServletRequest request,
                      HttpServletResponse response,
                      Object handler) throws Exception;

//后处理回调方法，实现处理器的后处理（但在渲染视图之前），
//此时可以通过modelAndView对模型数据进行处理或对视图进行处理
    void postHandle(HttpServletRequest request,
                    HttpServletResponse response,
                    Object handler, ModelAndView modelAndView) throws Exception;

//整个请求处理完毕回调方法，即在视图渲染完毕时调用
//主要是用于进行资源清理工作
    void afterCompletion(HttpServletRequest request,
                         HttpServletResponse response,
                         Object handler, Exception ex) throws Exception;
}
```
登录拦截：
```java
//继承链：LoginInterceptor -> HandlerInterceptorAdapter -> HandlerInterceptor
@Configuration
public class LoginInterceptor extends HandlerInterceptorAdapter {

  @Override
  public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    System.out.println("每个请求都会被这里拦截到，可以在这里进行预处理。");
    return true;
  }
}
```
##### 2. WebMvcConfigurerAdapter
在MVC config中配置要拦截的URL
```java
@Configuration
@EnableWebMvc
public class WebMvcConfig extends WebMvcConfigurerAdapter {
  //也可以不注入，直接使用new LoginInterceptor()
    @Inject
    private LoginInterceptor loginInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loginInterceptor)
                .addPathPatterns("/**")
                .excludePathPatterns("/login");
    }
}
```
