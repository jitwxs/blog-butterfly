---
title: 探讨 Spring MVC 能否注入 Request 和 Response
categories:
  - Java Web
  - Spring MVC
tags: Spring
abbrlink: 6078a4e1
date: 2019-05-07 21:57:35
icons: [fas fa-fire red]
references:
  - name: SpringMVC之自动注入Request对象
    url: https://blog.csdn.net/zknxx/article/details/77917290
    rel: nofollow noopener noreferrer
    target: _blank
  - name: 在SpringMVC Controller中注入Request成员域
    url: https://www.cnblogs.com/abcwt112/p/7777258.html
    rel: nofollow noopener noreferrer
    target: _blank
  - name: Springmvc中在controller注入request会有线程安全问题吗
    url: https://segmentfault.com/q/1010000005139036
    rel: nofollow noopener noreferrer
    target: _blank
  - name: SpringMVC在Controller层中注入request的坑
    url: https://my.oschina.net/sluggarddd/blog/678603
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---

## 一、引言

当我们第一次接触到 Java Web 开发，从最原生的 Servlet 方法开始，我们就知道在 `doGet()` 或者 `doPost()` 方法有两个形参，分别是 `HttpServletRequest` 和 `HttpServletResponse`，这两个参数代表了 web 容器为我们封装的 HTTP 请求和 HTTP 响应。

当 Java Web 进化到 SpringMVC 中，一系列的杂活脏活都交给了 `DispatcherServlet` 前端控制器来处理。

回到正文，传统情况下，我们访问一个接口，想要从中取得 `request` 对象，或者是 `response` 对象，亦或者是 `httpSession` 对象，都是直接作为形参传进来。举个例子，前端传递 token，先经过 filter 得到用户ID，并将它存入 request 中，那么在每个接口中取得用户ID，都要这样：

```java
@GetMapping(/"test")
public ResponseResult test(HttpServletRequest request, HttpServletResponse response) {
    Object userId = request.getAttribute("userId");
    ...
}
```

每个接口都要加 HttpServletRequest 或者 HttpServletResponse，第一写起来麻烦，第二看起来参数也很多。那么既然 Spring 可以依赖注入，我们可不可以这样做呢：

```java
@RestController
public class DemoController {
    @Autowired
    private HttpServletRequest request;
    @Autowired
    private HttpServletResponse response;

    @GetMapping("/test")
    public ResponseResult test() {
        Object userId = request.getAttribute("userId");
        ...
    }
}
```

试了一下竟然真的可以，但是仔细一想就会有几个疑惑：

1. 既然我可以将它 `Autowired` 出来，那么它是啥时候被注入的呢？

2. 我们知道 Spring 容器中的 Bean 默认是**单例**的，那么这样得到的 `request` 会不会有问题？并发情况下，一个接口会不会取到另一个接口的 `request`？

## 二、结论

探讨之前，先说结论。

**1. 啥时候注入的？**

答：SpringMVC `DispatcherServlet` 每次处理 HTTP 请求时，会将 web 容器封装的 `request` 和 `response` 注入到 Spring 容器中。

**2. 这样在并发情况下会不会有问题？**

答：不会有问题。内部其实存在一个 `ThreadLocal` ，不同进程的 `request` 和 `response` 是隔离的。

**3. 那我们以后是不是可以都这样写了？**

答：理论上且实际上这样写都没有问题，但是一般认为接口形参上的 request 和 response 对应着一次 HTTP 请求，因此用注入的方式会让人感觉有点奇怪。

## 三、为什么注入的没有线程安全问题？

下面来实验下，分别用**注入**的方式和**形参**的方式，来看看这两种得到的 Request 和 Response 有什么区别。

> 因为 request 和 response 的原理是一样的，因此下文只以 request 为例，避免啰嗦。

![注入与形参区别](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201905/201905072146306.png)

IDEA十分智能，一上来就告诉我们注入 request 的是 `$Proxy`，形参的 request 是 `RequestFacade` ，初步得知是通过**代理**的方式取出 request。

点开注入的 request，发现它是从 `AutowireUtils` 中的 `ObjectFactoryDelegatingInvocationHandler` 取出了，点进去看看。

![ObjectFactoryDelegatingInvocationHandler](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201905/20190507214813216.png)

可以看到这个静态内部类具有以下属性：

- 成员变量：objectFactory
- 构造方法
- `invoke` 方法

`invoke` 这个名字是不是很熟悉，这不就是反射吗。在该方法内部，对 `equals`、`hashCode`、`toString` 方法做了特殊处理，其余的都通过 `method.invoke(this.objectFactory.getObject(), args)` 反射调用原方法了。

那么就去看看这个 `objectFactory.getObject()` 是什么，点进去发现 `ObjectFactory` 是一个接口，根据一开始注入地方的截图，我们知道它的实现类是 `WebApplicationContextUtils` 中的 `RequestObjectFactory`，点进去看看。

![RequestObjectFactory](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201909/20190912001318208.png)

它通过调用 `currentRequestAttributes().getRequest()` ，取出了 ServletRequest，那么点进去看看，它是怎么取得的。

![WebApplicationContextUtils](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201905/20190507214950544.png)

经过一系列的调用，可以看到最后是通过 `requestAttributesHolder` 和 `inheritableRequestAttributesHolder` 中取出来的，接着看看这俩的定义。

![RequestContextHolder](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201905/20190507215041626.png)

看到这相信你已经知道了，是从`ThreadLocal` 中取出来的，这也就说明它是线程隔离的，因此通过注入方式得到的 request 和 response 是线程安全的。

再回想下我们是如何在普通类中取得当前线程的 request 对象，再结合上面的调用流程，是不是豁然开朗。

```java
// 取得当前线程的 Request 对象
HttpServletRequest request = ((ServletRequestAttributes)RequestContextHolder.getRequestAttributes()).getRequest();
```

## 四、它是何时被注入的？

引言中说过，SpringMVC有一个大管家 `DispatcherServlet`，它的作用及处理流程就不再赘述了，网上相关资料很多。

在 `DispatcherServlet` 的父类 `FrameworkServlet` 中，我们发现所有请求相关的方法，内部都调用了 `processRequest` 方法。

![FrameworkServlet](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201905/20190507215117497.png)

看起来这个方法就是实际处理 HTTP 请求的，点进去看看。

![FrameworkServlet#processRequest](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201905/20190507215152502.png)

这个方法逻辑也很明晰，我们看其中最关键的 `initContextHolders` 方法，它将本次请求的 request，以及新初始化的 localContext 和 requestAttributes 传入，点进去看看。

![FrameworkServlet#initContextHolders](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201905/201905072152303.png)

`initContextHolders` 方法内部分别调用了：

-  LocaleContextHolder.setLocaleContext()
-  RequestContextHolder.setRequestAttributes()

两个方法的逻辑大致一致，都是根据 `inheritable` 的取值，来决定 set 进哪个 Holder，remove 出哪个 Holder，下面看下这几个 Holoder 的定义。

![Holder 定义](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201905/2019050721530387.png)

全都是 ThreadLocal，再看看 `RequestContextHolder` 中的那两个，是不是跟文章上一节最后取出来的那两个是同一个，至此就破案了。

## 五、番外

在上一节，我们发现Spring会根据 `inheritable` 的取值来决定从哪个 Holder 中设值，那么这个 `inheritable` 是个什么东西呢。

`inheritable` 英文直译为 "可遗传的"，我们知道，默认情况下 `ThreadLocal` 中的值在每个线程都是独立的，但是 `InheritableThreadLocal` 却可以在子线程访问父线程中的变量或属性。

以 `RequestContextHolder` 中那两个 Holder 为例，看看它的构造类。

![NamedInheritableThreadLocal 与 NamedInheritableThreadLocal](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201905/20190507215344783.png)

不出所料，它们的父类分别就是 `ThreadLocal` 和 `InheritableThreadLocal`。那么它是咋实现子访问父的呢，我们去 Thread 类里面看一看。

![Thread#init](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201905/20190507215420670.png)

首先 Thread 类中有两个 ThreadLocalMap，分别是 `threadLocals` 和 `inheritableThreadLocals`。

然后看下它的 `init()` 方法，首先形参传了一个 `inheritThreadLocals`，表示是否是要继承父线程，如果为 true 情况下，调用 `ThreadLocal.createInheritedMap(parent.inheritableThreadLocals)` ，也就是使用了父 `inheritableThreadLocals` 初始化了当前的 `inheritableThreadLocals`，点进去看看。

![ThreadLocal#ThreadLocalMap](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201905/20190507215449827.png)

代码比较简单，就不细说了，就是一个拷贝。至此为啥 `InheritableThreadLocal` 能够访问父线程中的变量就破案了。

回归正题，既然注入的不存在线程安全问题，那么这个布尔值自然就是 false 了。如果你要手动改为 true 的话，那么这样注入的可就存在线程安全问题了。

![threadContextInheritable](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201905/20190507215519437.png)
