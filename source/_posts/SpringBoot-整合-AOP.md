---
title: SpringBoot 整合 AOP
tags: AOP
categories:
  - Java Web
  - SpringBoot
abbrlink: 77bba914
date: 2018-12-06 19:04:32
related_repos:
  - name: aop-sample
    url: https://github.com/jitwxs/blog-sample/blob/master/springboot-sample/aop-sample
    rel: nofollow noopener noreferrer
    target: _blank
---

## 一、前言

[AOP](https://baike.baidu.com/item/AOP/1332219)（Aspect Oriented Programming, 面向切面编程），是 Spring 的核心思想之一，即**纵向重复，横向抽取**，它在 Spring 中应用广泛，例如 拦截器、日志、事务等等，在 SpringBoot 中使用 AOP 之前，我们先复习下 AOP 的理论知识。

## 二、AOP 理论

### 2.1 术语解释

为了方便解释，给出一个例子：

```java
public interface UserService {
    void save();
    void delete();
    void update();
    void query();
}
```

```java
public class UserServiceImpl implements UserService {
    @Override
    public void save() { System.out.println("save"); }

    @Override
    public void delete() { System.out.println("delete"); }

    @Override
    public void update() { System.out.println("update"); }

    @Override
    public void query() { System.out.println("query"); }
}
```

若我们想要对这四个方法进行增强，假设增强如下：

```java
...
public void delete() {
    System.out.println("开启事务");
    System.out.println("delete");
    System.out.println("提交事务");
}
...
```

（1）`连接点(JoinPoint)`

解释：目标对象中，**所有可以增强的方法**。

例子：即 save()、delete()、update()、query() 这四个方法（因为我们可以使用动态代理技术对这些方法进行增强）。

（2）`切入点(PointCut)`

解释：目标对象中，**已经增强的方法**。

例子：若 delete() 方法在动态代理中被增强了，则 delete() 方法是切入点。

（3）`通知/增强(Advice)`

解释：**增强的代码**

例子：即 System.out.println("开启事务"); 和 System.out.println("提交事务");

（4）`目标对象(Target)`

解释：**被代理对象**

例子：即 UserServiceImpl 类

（5）`织入(Weaving)`

解释：**将通知应用到切入点的过程**

（6）`代理(Proxy)`

解释：**将通知织入到切入点后，形成代理对象**

（7）`切面(aspect)`

解释：**切入点和通知合称为切面**

### 2.2 通知类型

|通知类型|描述|
|:--|:--|
|前置通知 Before advice|在连接点前面执行，前置通知不会影响连接点的执行，除非此处抛出异常|
|正常返回通知 After returning advice|在连接点正常执行完成后执行，如果连接点抛出异常，则不会执行|
|异常返回通知 After throwing advice|在连接点抛出异常后执行|
|返回通知 After (finally) advice|在连接点执行完成后执行，不管是正常执行完成，还是抛出异常，都会执行返回通知中的内容|
|环绕通知 Around advice|环绕着在切入点选择的连接点处的方法所执行的通知，环绕通知可以在方法调用之前和之后自定义任何行为，并且可以决定是否执行连接点处的方法、替换返回值、抛出异常等等。|

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181206165911529.png)

### 2.3 底层实现

AOP 的实现是基于`动态代理`技术，一般有**基于 JDK 的动态代理**和**基于 Cglib 的动态代理**两种实现。

这两种实现的区别是：

- `JDK动态代理`：针对**实现了接口的类进行代理**，代理对象和目标对象实现了共同的接口，拦截器必须实现InvocationHanlder接口。

- `Cglib动态代理`：针对**没有实现接口的类进行代理**，底层是字节码增强技术，**生成当前类的子类对象**。代理对象是目标对象的子类，拦截器必须实现 MethodInterceptor 接口。

关于使用 JDK 动态代理 和 Cglib 动态代理实现上述例子，这里就不再赘述了，可以参考文章：[《JDK动态代理与Cglib动态代理》](/8ee3adf6.html)。

## 三、与 SpringBoot 整合

### 3.1 依赖配置

导入依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

相关配置只有一项：

```yaml
spring:
  aop:
    proxy-target-class: true # 当值为 true 时，使用 Cglib 动态代理
```

### 3.2 入门程序

#### 3.2.1 环境准备

（1）编写 UserService 和 UserServiceImpl：

```java
public interface UserService {
    void delete(String name);
}
```

```java
@Service
public class UserServiceImpl implements UserService {
    @Override
    public void delete(String name) {
        System.out.println("delete：" + name);
    }
}
```

（2）编写 UserController

```java
@RestController
public class UserController {
    @Autowired
    private UserService userService;

    @GetMapping("/delete")
    public String restDelete(String name) {
        userService.delete(name);
        return "success";
    }

    @GetMapping("/exception")
    public void restException() {
        int i = 1 / 0;
    }
}
```

#### 3.2.2 切面表达式

首先复习下 AOP 中的切面表达式，假设 UserController 类位于 com.github.jitwxs.sample.aop.controller 包下，那么如果我们为其 restDelete() 方法 配置切入点，如下：

```java
public void com.github.jitwxs.sample.aop.controller.UserController.restDelete()
```

也就是 restDelete() 方法的 reference 路径加上**访问修饰符**和**返回值**，一般来说我们不对非 public 方法添加切入点，也不限制其返回值，因此省略掉访问修饰符（默认值即为 public），并将返回值修改为 *（即任何返回值均可）：

```java
* com.github.jitwxs.sample.aop.controller.UserController.restDelete()
```

再将切面方法扩展到 UserController 下的所有方法：

```java
* com.github.jitwxs.sample.aop.controller.UserController.*()
```

此时只能对 UserController 下的所有无参 public 方法添加切入点，将其修改为不限制参数：

```java
// 将 () 修改为 (..) ，不限制参数
* com.github.jitwxs.sample.aop.controller.UserController.*(..)
```

如果我们想要 controller 包下的所有以 Controller 结尾的类添加切入点，修改为：

```java
// 将 controller包下所有以 Controller 结尾的类添加切入点
* com.github.jitwxs.sample.aop.controller.*Controller.*(..)
```

一般这样就可以了，如果你想为 controller 的子包也添加切入点，修改为：

```java
// 在 controller 后多添加一个 . 表示包含它的子包
* com.github.jitwxs.sample.aop.controller..*Controller.*(..)
```

#### 3.2.3 切面类

1. 使用 `@Aspect` 注解将一个类标记为切面类，`@Component` 将该类加入 Spring 容器中。

2. 使用 `@Pointcut` 注解定义切面表达式，为 controller 包下所有 Controller 结尾的类的 public 方法添加切入点。

3. 通过 `@Before`、`@AfterReturning`、`@AfterThrowing`、`@After`、`@Around` 注解定义通知类型。

4. 在 `doBefore()` 方法中，通过 `joinPoint` 参数获取通知的签名信息，如目标方法名、目标方法参数信息等。、、

5. 在 `doAroundAdvice()` 方法中，处理环绕通知。切面中可选择执行方法与否，执行几次方法等。环绕通知使用一个代理ProceedingJoinPoint类型的对象来管理目标对象。

```java
@Component
@Aspect
public class TestAspect {
    private Logger logger = LoggerFactory.getLogger(getClass());

    @Pointcut("execution(* com.github.jitwxs.sample.aop.controller.*Controller.*(..))")
    public void pointCunt(){}

    @Before("pointCunt()")
    public void doBefore(JoinPoint joinPoint) {
        logger.info("---前置通知---");

        // 获取目标方法的参数信息
        logger.info("目标对象参数信息: {}", Arrays.toString(joinPoint.getArgs()));

        // AOP代理类的信息
        logger.info("AOP代理类: {}", joinPoint.getThis());

        // 代理的目标对象
        logger.info("代理的目标对象: {}", joinPoint.getTarget());

        // 通知的签名
        Signature signature = joinPoint.getSignature();
        // 代理的是哪一个方法
        logger.info("代理的方法名: {}", signature.getName());

        // AOP代理类的名字
        logger.info("AOP代理类的名字: {}", signature.getDeclaringTypeName());

        // AOP代理类的类信息
        logger.info("AOP代理类的类信息: {}", signature.getDeclaringType());

//        RequestAttributes requestAttributes = RequestContextHolder.getRequestAttributes();
        // 获取 HttpServletRequest
//        HttpServletRequest request = (HttpServletRequest) requestAttributes.resolveReference(RequestAttributes.REFERENCE_REQUEST);

        // 获取 Session
//        HttpSession session = (HttpSession) requestAttributes.resolveReference(RequestAttributes.REFERENCE_SESSION);
    }

    @AfterReturning(returning = "ret", pointcut = "pointCunt()")
    public void doAfterReturning(JoinPoint joinPoint, Object ret) {
        logger.info("---后置返回通知---");
        logger.info("代理方法: {}, 返回参数: {} ", joinPoint.getSignature().getName(), ret);
    }

    @AfterThrowing(pointcut = "pointCunt()", throwing = "e")
    public void doAfterThrowingAdvice(JoinPoint joinPoint, Throwable e) {
        logger.info("---后置异常通知---");
        logger.info("代理方法: {}, 异常信息: {} ", joinPoint.getSignature().getName(), e.getMessage());
    }

    @After("pointCunt()")
    public void doAfterAdvice(JoinPoint joinPoint) {
        logger.info("---后置最终通知---");
    }

    @Around("pointCunt()")
    public Object doAroundAdvice(ProceedingJoinPoint proceedingJoinPoint) {
        logger.info("---环绕通知开始---");
        logger.info("代理方法: {}", proceedingJoinPoint.getSignature().getName());
        try {
            Object obj = proceedingJoinPoint.proceed();
            logger.info("---环绕通知结束---");
            return obj;
        } catch (Throwable throwable) {
            logger.info("---环绕通知异常---");
            throwable.printStackTrace();
        }
        return null;
    }
}
```

#### 3.2.4 执行程序

当我们执行 "/delete" 接口时，访问 `/delete?name=jitwxs`，运行结果如下：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181206180112369.png)

当我们执行 "/exception" 接口时，访问 `/exception`，运行结果如下：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181206181304976.png)

### 3.3 资源同步

使用 ThreadLocal 实现资源同步，例如统计切入点执行时间：

```java
@Component
@Aspect
public class TestAspect {
    private Logger logger = LoggerFactory.getLogger(getClass());

    private ThreadLocal<Long> threadLocal = new ThreadLocal<>();

    @Pointcut("execution(* com.github.jitwxs.sample.aop.controller.*Controller.*(..))")
    public void pointCunt(){}

    @Before("pointCunt()")
    public void doBefore(JoinPoint joinPoint) {
        logger.info("---前置通知---");
        threadLocal.set(System.currentTimeMillis());
    }

    @AfterReturning(returning = "ret", pointcut = "pointCunt()")
    public void doAfterReturning(JoinPoint joinPoint, Object ret) {
        logger.info("---后置返回通知---");
        logger.info("共执行时间：{} ", System.currentTimeMillis() - threadLocal.get());
    }
}
```

### 3.4 优先级

当我们对同一个切入点设置了多个切面后，我么你可以定义切面的优先级，来达到预想的执行顺序。

`@Order()` 注解来标识切面的优先级，i的值越小，优先级越高。假如存在两个切面，定义了 @Order(5) 和 @Order(10)，那么：

- 在 `@Before` 中优先执行 @Order(5) 的内容，再执行 @Order(10) 的内容

- 在 `@After` 和 `@AfterReturning` 中优先执行 @Order(10) 的内容，再执行 @Order(5) 的内容

所以我们可以这样总结：

- 在切入点前的操作，按 order 的值**由小到大**执行

- 在切入点后的操作，按 order 的值**由大到小**执行
