---
title: JDK 动态代理与 Cglib 动态代理
tags:
  - 动态代理
  - AOP
categories: Java
typora-root-url: ..
abbrlink: 8ee3adf6
date: 2018-03-09 10:03:36
related_repos:
  - name: dynamic_proxy
    url: https://github.com/jitwxs/blog-sample/blob/master/Java/dynamic_proxy
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---

## 一、前言

`AOP`(Aspect Oriented Programming)，即`面向切面编程`，最主要的思想就是**纵向重复，横向抽取**。要想实现 AOP，其底层实现是使用了`动态代理技术`，在 Spring 中，动态代理技术分为传统的 `JDK 动态代理`和 `Cglib 动态代理`。这两种代理机制区别是：

- JDK 动态代理：**针对实现了接口的类进行代理**

- Cglib 动态代理：**针对没有实现接口的类进行代理**，底层是字节码增强技术，生成当前类的**子类对象**。

假设我们有一个 UserService 接口，其中具有 CRUD 方法：

```java
public interface UserService {
    void save();
    void delete();
    void update();
    void query();
}
```

它有一个实现类，只是简单的输出了信息：

```java
public class UserServiceImpl implements UserService {
    @Override
    public void save() {
        System.out.println("save");
    }

    @Override
    public void delete() {
        System.out.println("delete");
    }

    @Override
    public void update() {
        System.out.println("update");
    }

    @Override
    public void query() {
        System.out.println("query");
    }
}
```

如果我们想要对这四个方法进行增强，例如在每个方法开头和结尾开启和提交事务，例如：

```java
public void delete() {
    System.out.println("开启事务");
    System.out.println("delete");
    System.out.println("提交事务");
}
```

如果不使用 AOP 思想或动态代理技术，要写很多的冗余代码。

## 二、JDK 动态代理

**Step1：** 要有一个实现 `InvocationHandler` 接口的类，编写 `MyInvocationHandler` 类：

```java
/**
 * 基于 JDK 的动态代理
 * @author jitwxs
 * @since 2018/12/6 15:18
 */
public class MyInvocationHandler implements InvocationHandler {
    /**
     * 要代理类的对象
     */
    private Object targetObj;

    /**
     * 在创建对象时将要代理类的对象传进来
     */
    public MyInvocationHandler(Object targetObj) {
        this.targetObj = targetObj;
    }

    /**
     * @param proxy  方法对象（没有实际作用）
     * @param method 该方法对象所在的类对象实例
     * @param args   方法参数
     */
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("开启事务");
        /*
         * obj：调用谁的方法用谁的对象
         * args 方法调用时的参数
         */
        Object invoke = method.invoke(targetObj, args);
        System.out.println("提交事务");

        return invoke;
    }
}
```

**Step2：** 编写 UserService 的动态代理类：

```java
/**
 * UserService的动态代理类
 * @author jitwxs
 * @since 2018/12/6 15:19
 */
public class UserServiceProxy {
    private UserService userService;

    public UserServiceProxy(UserService userService) {
        this.userService = userService;
    }

    public UserService getUserServiceProxy() {
        // 创建实现InvocationHandler类的对象，将要代理类的对象传进去
        MyInvocationHandler myInvocationHandler = new MyInvocationHandler(userService);

        /*
         * public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)
         * loader : 类加载器（指定当前类即可）
         * interfaces 要代理的类实现的接口列表
         * h：实现具体代理操作的类（要实现InvocationHandler接口）
         */
        UserService userServiceProxy = (UserService) Proxy.newProxyInstance(UserServiceProxy.class.getClassLoader(),
                UserServiceImpl.class.getInterfaces(),
                myInvocationHandler);
        return userServiceProxy;
    }
}
```

**Step3：** 编写测试方法

```java
/**
 * JDK动态代理测试类
 * @author jitwxs
 * @since 2018/12/6 15:21
 */
public class JdkProxyTest {
    public static void main(String[] args) {
        //创建UserService实例
        UserService us = new UserServiceImpl();
        // 创建UserService代理类实例
        UserServiceProxy userServiceProxy = new UserServiceProxy(us);
        // 返回代理后增强过的UserService实例
        UserService usProxy = userServiceProxy.getUserServiceProxy();

        usProxy.delete();
    }
}
```

![](/images/posts/20180309094803849.png)

##  三、Cglib 动态代理

**Step0：** 导入 Cglib 依赖包：

```xml
<dependency>
    <groupId>cglib</groupId>
    <artifactId>cglib</artifactId>
    <version>3.2.4</version>
</dependency>
```

**Step1：** 编写 userService 实现 `MethodInterceptor ` 接口的 Cglib 动态代理类：

```java
/**
 * UserService的动态代理类
 * @author jitwxs
 * @since 2018/12/6 15:22
 */
public class UserServiceProxy2 implements MethodInterceptor {
    public UserService getUserServiceProxy() {
        // 生成代理对象
        Enhancer enhancer = new Enhancer();
        // 设置对谁进行代理
        enhancer.setSuperclass(UserServiceImpl.class);
        // 代理要做什么
        enhancer.setCallback(this);
        // 创建代理对象
        UserService us = (UserService) enhancer.create();

        return us;
    }

    @Override
    public Object intercept(Object o, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        System.out.println("开启事务");
        Object invoke = methodProxy.invokeSuper(o, args);
        System.out.println("提交事务");
        return invoke;
    }
}
```

**Step2：** 编写测试方法

```java
/**
 * Cglib 动态代理测试类
 * @author jitwxs
 * @since 2018/12/6 15:34
 */
public class CglibProxyTest {
    public static void main(String[] args) {
        // 创建UserService代理类实例
        UserServiceProxy2 userServiceProxy2 = new UserServiceProxy2();
        // 返回代理后增强过的UserService实例
        UserService usProxy = userServiceProxy2.getUserServiceProxy();

        usProxy.query();
        usProxy.save();
        usProxy.update();
        usProxy.delete();
    }
}
```

![](/images/posts/20180309094803849.png)

## 四、总结

（1）JDK 的动态代理  

- 代理对象和目标对象**实现了共同的接口**  
- 拦截器必须实现 `InvocationHanlder` 接口  
  
（2）Cglib 的动态代理  

- 代理对象**是目标对象的子类**  
- 拦截器必须实现 `MethodInterceptor` 接口  
