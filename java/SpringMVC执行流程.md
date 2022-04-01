# SpringMVC执行流程

<img src="https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220310193112056.png" alt="image-20220310193112056" style="zoom:150%;" />

1、浏览器提交请求到**中央调度器（DispatcherServlet）**

2、中央调度器直接将请求转给**处理器映射器（HandleMapping）**

3、处理器映射器会根据请求，找到处理该请求的处理器，并将其封装为**处理器执行链（HandlerExecutionChain）**后返回给中央调度器,因为可能有一些拦截器要拦截这个请求。

4、中央调度器根据处理器执行链中的处理器，找到能够执行该处理器的**处理器适配器（HandleAdaptor）**。

5、处理器适配器调用执行**处理器（Controller）**

6、处理器将处理结果及要跳转的视图封装到一个对象 **ModelAndView** 中，并将其返回给处理器适配器。

7、处理器适配器直接将结果返回给中央调度器。

8、中央调度器调用**视图解析器（ViewResolver）**，将 ModelAndView 中的视图名称封装为视图对象。

9、视图解析器将封装了的**视图对象(View)**返回给中央调度器

10、中央调度器调用视图对象，让其自己进行**渲染(render)**，即进行数据填充，形成响应对象。

11、中央调度器响应浏览器。

> HandlerMapping 是⽤来查找 Handler 的，也就是处理器，具体的表现形式可以是类，也可以是 ⽅法。⽐如，标注了@RequestMapping的每个⽅法都可以看成是⼀个Handler。Handler负责具 体实际的请求处理，在请求到达后，HandlerMapping 的作⽤便是找到请求相应的处理器 Handler 和 Interceptor
>
> HandlerAdapter 是⼀个适配器。因为 Spring MVC 中 Handler 可以是任意形式的，只要能处理请 求即可。但是把请求交给 Servlet 的时候，由于 Servlet 的⽅法结构都是 doService(HttpServletRequest req,HttpServletResponse resp)形式的，要让固定的 Servlet 处理 ⽅法调⽤ Handler 来进⾏处理，便是 HandlerAdapter 的职责。



## SpirngMVC中的HandlerAdapter

![image-20220401193728104](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220401193728104.png)

每一个Controller可能有不同的接口去处理我们的url请求，比如HttpServlet用service接收，实现了Controller接口的子类用handleRequest接收,而@RestController中的@RequestMapping接收的请求，又是调用handleInternal来处理请求的。

**可以看到处理器的类型不同，有多重实现方式，那么调用方式就不是确定的，如果需要直接调用 Controller 方
法，需要调用的时候就得不判断是使用 if else 来进行判断是哪一种子类然后执行。那么如果后面要扩展 Controller，
就得修改原来的代码，这样违背了 OCP 原则。**

因此HandlerAdapter定义了一个规范。

```java
public interface HandlerAdapter {
    
    //用这个方法判断目前这个HandlerAdapter能否处理当前请求的handler
    boolean supports(Object handler);
    
    //真正的执行请求方法,由DispatcherServlet统一调用这个方法
    ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler);
    
}
```



```java
//DispatcherServlet.java
//根据当前的handler,去找它的适配器
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
    if (this.handlerAdapters != null) {
        for (HandlerAdapter adapter : this.handlerAdapters) {
            //若当前这个adapter能够处理当前请求的handler,那么直接返回
            if (adapter.supports(handler)) {
                return adapter;
            }
        }
    }
    //找不到的话就抛异常
    throw new ServletException(
        "No adapter for handler [" + handler +
        "]: The DispatcherServlet configuration + "
        + "needs to include a HandlerAdapter that supports this handler");
}

//DispatcherServlet统一调用HandlerAdapter接口中的handle方法返回结果
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

