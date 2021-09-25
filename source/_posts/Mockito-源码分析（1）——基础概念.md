---
title: Mockito 源码分析（1）——基础概念
categories: Unit Test
tags: Mockito
copyright_author: Jitwxs
abbrlink: f9625997
date: 2021-09-20 09:02:35
---

## 前言

作为一个合格的开发工程师，写好 UT(Unit Test) 是必备的技能，目前市面上 UT 工具很多。我选用了使用最为广泛的 Mockito + Powermock 的组合，来分析它的源码，希望能为大家带来收获。

>请注意，本系列不会为大家介绍基础用法。

系列开始之前，首先约定下版本：

```xml
<dependency>
    <groupId>org.powermock</groupId>
    <artifactId>powermock-module-junit4</artifactId>
    <scope>test</scope>
    <version>2.0.9</version>
</dependency>
<dependency>
    <groupId>org.powermock</groupId>
    <artifactId>powermock-api-mockito2</artifactId>
    <scope>test</scope>
    <version>2.0.9</version>
</dependency>
<dependency>
    <groupId>org.powermock</groupId>
    <artifactId>powermock-reflect</artifactId>
    <scope>test</scope>
    <version>2.0.9</version>
</dependency>
```

如上图所示，我使用了 powermock 2.0.9 版本，整合了 junit + mockito 框架。

## Mock && InjectMocks && Spy

在我最初学习 Mockito 时候，对 Mock、InjectMocks、Spy 这三位老伙计的含义有些歧义，特在此处中先说明白。

在对某个类使用 Mock 后。那么该类的所有成员变量都会置为默认值，所有的方法实现也会置空，便于后续自定义实现。如下图所示，对于类 Order，其成员变量被设置成默认值，其方法 print() 实现也被清空，方法返回了默认值。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920092619411.png)

而 Spy 则跟 Mock 正好相反，它有点类似于 new，其构造出的类，都会获取其真实的成员变量和方法。

如果有个类，想将其完全 mock 掉，仅对部分我用到的方法进行实现，那么用 mock 即可；如果有个类，仅想 mock 其中的一小部分，其他部分执行真实逻辑，那么用  spy 即可。

对于 InjectMocks，它会创建出一个类的实例，在这一点上与 Spy 类似。另外它会搜寻所有的 mock 和 spy，如果该类内部有这些对象的引用，将会把它们注入进来，有点像 IOC 容器。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920094435525.png)

如上图所示，当我对 OrderService 使用 InjectMocks 后，其被 mock 的成员变量 OrderClient 被自动注入其中，并且调用 OrderService 的方法时，走的是真实逻辑。

## Mock 原理

不知道大家在初次接触 Mock 时候时，有没有好奇过，为什么它可以通过寥寥几行代码，就实现了这么牛逼的功能，这也是我去研究它源码的原因。

简单来说，当你在 mock 时候，它会通过 proxy 的方式创建一个代理类。后续你在进行 whenThen 打桩（stub）的时候，它会将这些 stub 都保存起来。等到你真正用的时候，优先判断 stub 是否存在预期结果，如果存在就返回该值，不存在就根据情况返回默认值或去调用真实方法。

是不是听起来还挺简单，下一篇文章将正式开启源码分析的部分了。





