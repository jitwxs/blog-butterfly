---
title: Slf4j 包冲突问题原因与解决
copyright_author: Jitwxs
typora-root-url: ..
categories:
  - Java
  - 日志框架
abbrlink: e2390047
date: 2020-12-06 23:57:23
---

## 一、前言

在进行 Java 开发时，通常我们会选择 Slf4j 作为日志门面，但日志实现却不尽相同。如果系统运行中同时存在多个日志实现，就会出现类似下图的 Warning。

![](/images/posts/20201206230550.png)

## 二、问题原因

我们知道 SpringBoot 默认使用的日志实现是 Logback，因此我们尝试在项目中引入 Log4j 的依赖时，就复现了上图的报错。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

上图报错告知我们存在多个 SLF4J bingdings，分别位于 logback 和 log4j 包中，有两个 `StaticLoggerBinder`。

我们知道使用 Slf4j ，需要 `LoggerFactory.getLogger()`  方法获取实例。

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

private final Logger logs = LoggerFactory.getLogger(xxx.class);
```

我们就可以通过这个作为入口，去看看源码的实现。如下图所示，我标注了需要关注的核心代码。

![](/images/posts/20201206232119.png)

（1）调用 `getILoggerFactory()` 方法得到 LoggerFactory。

（2）对于首次调用，`INITIALIZATION_STATE` 应该是 UNINITIALIZED，所以进入初始化的逻辑，调用方法 `performInitialization()`。

（3）调用 `bind()` 方法。

（4）如果不是 `isAndroid()`，调用 `findPossibleStaticLoggerBinderPathSet()` 方法，故名思意，查找可能的 staticLoggerBinder，注意这里返回的类型是 SET，即可能是多个。

（5）在`findPossibleStaticLoggerBinderPathSet()` 这个方法内，首先通过 classLoader 加载了 `org/slf4j/impl/StaticLoggerBinder.class` 这个类的 path，它可能存在多个，因此使用了 while 获取了所有的 path，并最终返回。

![](/images/posts/20201206232900.png)

（6）`reportActualBinding()` 方法会校验 SET 的 size，如果大于 1，就会打印出一开始我们看见的 Warning 了。

![](/images/posts/20201206233518.png)

## 三、问题解决

解决思路就是将你不想要的日志实现从依赖包中排除掉即可，通过 IDEA 提供的 `Diagrams` 能够非常方便的查看项目中的依赖关系。

打开项目的 POM 文件，右键选择 `Diagrams -> Show Dependencies`

![](/images/posts/20201206233647.png)

假设我们想要排除 logback 依赖，使用 log4j。`Ctrl + F` 搜索 `logback`，可以找到引用该依赖的树形结构。

![](/images/posts/20201206234031.png)

点击窗口左上角的下图中的这个图标，可以只看当前选中的这个依赖的关系。

![](/images/posts/20201206234108.png)

选中后效果如下：

![](/images/posts/20201206234057.png)

如上图所示，logback 由 `spring-boot-starter-logging` 引入，最顶层是由 `spring-boot-starter-web` 和 `spring-boot-starter-test` 引入。

我们尝试在 `spring-boot-starter-web` 中排除该依赖，应该就可以了。如果排出后重新搜索仍然存在 logback 依赖，则重复执行排除的操作。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

## 四、总结

日志框架冲突特别对于新手来说处理起来比较头疼，因为涉及到了日志接口和日志实现。

我们推崇的应该是面向接口编程，因此我们大到开源项目，小到公司的公共 jar 包，应当合理利用 Maven 的传递机制。具体的日志实现不应该传递出去，避免影响到调用的下游方。

```xml
<optional>true</optional>
```
