---
title: Mockito 源码分析（2）——Mock 与 Spy
categories: 单元测试
tags: Mockito
abbrlink: cb1af043
date: 2021-09-20 09:48:47
related_repos:
  - name: mock-sample
    url: https://github.com/jitwxs/blog-sample/tree/master/javase-sample/mock-sample
---

## Mock

上来先丢一张时序图，后面在看源码的时候可以结合这张图，更容易理解。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920095003982.png)

> 为了方便调试，我没有使用注解的方式

无论你是使用 `Mockito.mock()` 还是 `PowerMockito.mock()`，其都会调用 MockitoCore.mock 方法（这里需要注意，MockitoCore 对象是 Static 属性，因此全局仅有一个）。

![MockitoCore](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920100854834.png)

进入 MockitoCore#mock 方法后，它做了主要三件事，如下图所示：

1. 对上一步传入的 MockSettings 进行一系列校验
2. 构建出传入类的 proxy 实例
3. 调用 mockingStarted 方法

![org.mockito.internal.MockitoCore#mock](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920101658104.png)

createMock 方法中最核心的就是前两行代码。如下图所示：

![org.mockito.internal.util.MockUtil#createMock](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920102100693.png)

这里首先创建了 MockHandler 对象，它本身是一个接口。通过追查 MockHandlerFactory#createMockHandler 方法可以得知，这里使用了委派模式，真正的核心逻辑是由 MockHandlerImpl 处理的。

![MockHandler](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920102507923.png)

接着来看第二行的 MockMaker，它也是一个接口，通过 `Plugins.getMockMaker()` 方式获取了一个静态实例。通过跟踪代码，确定此处的实例是 ByteBuddyMockMaker。

>限于主题关系，这里没有办法展开介绍 ByteBuddy 框架，你可以认为该框架可以**帮助我们在程序运行期间动态的创建出一个类并加载到 JVM 中**，后面有时间我再单独介绍该框架。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920103748427.png)

进入该类后发现依然用了委派，实际处理的是 SubclassByteBuddyMockMaker，因此我们直接看该类的 createMock 方法即可。

![org.mockito.internal.creation.bytebuddy.ByteBuddyMockMaker](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920104158372.png)

在该方法中，主要做了三件事：

1. 创建要 mock 的那个类的 proxy 类
2. 实例化该 proxy 类，得到对象
3. 将之前创建的 MockHandler 存入该 proxy 类

![org.mockito.internal.creation.bytebuddy.SubclassByteBuddyMockMaker#createMock](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920104629262.png)

通过这几行代码，我们能得到一些信息：

- 通过 createMockType() 方法创建出的代理类，一定实现了 MockAccess 接口，不然下面强制直接就报错了。
- MockAccess 接口，一定有名为 mockitoInterceptor 成员变量，类型为 MockMethodInterceptor。
- 该 proxy 类一定能强转回最初要 mock 的那个类（如下图所示）

![20210920105140145](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920105140145.png)

咱先看 createMockType() 方法，如下图所示，又是委派，TypeCachingBytecodeGenerator -> SubclassBytecodeGenerator。

![org.mockito.internal.creation.bytebuddy.SubclassByteBuddyMockMaker#createMockType](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920105407704.png)

先看 TypeCachingBytecodeGenerator，由于该类涉及到 bytebuddy，就不带大家继续往里跟了，直接说明下该类的作用：

**保存一个全局的 TypeCache 缓存，如果当前正在 mock 的类，曾经 mock 过了，那么直接会从缓存中返回，否则执行创建并插入缓存的回调方法，这个回调方法就是 SubclassBytecodeGenerator 的实现。**

![TypeCachingBytecodeGenerator](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920105852367.png)

写一个 UT 来验证这一点，如下图所示，可以看到，对于同一个类执行多次 mock，返回的是同一个类。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920110147423.png)

下面来看 org.mockito.internal.creation.bytebuddy.SubclassBytecodeGenerator#mockClass，这个方法实现了如何创建 proxy 类。该方法代码很多，但其中最核心的一段代码，我给大家标注出来了，如下所示：

![org.mockito.internal.creation.bytebuddy.SubclassBytecodeGenerator#mockClass](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920110641947.png)

以上代码展示了如何通过 bytebuddy 创建出一个 proxy 类。

① subClass 和 implement 方法，申明了该 proxy 类需要继承和实被 mock 类的父类和接口。只有这样，才能强转回去。

② name 方法指定了该 proxy 类的类名，生成规则如下：

![org.mockito.internal.creation.bytebuddy.SubclassBytecodeGenerato](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920110936276.png)

③ mehod 方法参数需要一个 ElementMatcher，可以理解为是一个匹配器，追查源码后发现值为 any()，即匹配所有方法。紧跟着的 dispatcher 方法，表示对于上面匹配的方法，指定一个拦截器，被匹配的方法都会被该拦截器拦截，这里的实现类是 DispatcherDefaultingToRealMethod。

![org.mockito.internal.creation.bytebuddy.SubclassBytecodeGenerato](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920111156606.png)

④ defineField 方法表示为该 proxy 类增加一个字段，该行代码等价于：

```java
private MockMethodInterceptor mockitoInterceptor;
```

implement 方法会该 proxy 类添加了 MockAccess 接口，可以看到该接口完全就是给 mockitoInterceptor 量身定制的。

![MockAccess](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920111653792.png)

至此完成了 proxy 类的构建，下面让我们回到 SubclassByteBuddyMockMaker#createMock，该方法的第二件事是实例化。由于此处不是很重要，就不展示跟代码的细节了，直接告诉大家它的实现类是 ObjenesisInstantiator，如下图所示。这里用到的了 objenesis 框架，该框架可以帮助我们简单的创建类的实例。

![org.mockito.internal.creation.instance.ObjenesisInstantiator](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920112035882.png)

完成实例化后，跟着代码一路回退到 MockitoCore#mock，我们来看这行代码：

```java
mockingProgress().mockingStarted(mock, creationSettings);
```

`mockingProgress()` 方法返回一个 MockingProgress 实例，即 MockingProgressImpl。需要注意的是，它被保存在了 TreadLocal 中，也就是说每个线程的 MockingProgress  是相同的。

![org.mockito.internal.progress.ThreadSafeMockingProgress#mockingProgress](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920112720987.png)

`mockingStarted` 方法内部，看着像是监听器模式，这个跟咱们主流程不搭嘎，就不研究了。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920113109695.png)

既然都看到了这个类，就看看 MockingProgress 这个接口的方法把，如下图所示。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920113622873.png)

## Spy

在 Mockito 类中，spy 跟 mock 的区别就是对于 MockSettings 参数，spy 多了对 spiedInstance 和 defaultAnswer 的配置。

![org.mockito.Mockito#spy(T)](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/2021092011420025.png)

关于 spiedInstance 的作用，参见 MockUtil#createMock 的方法，在完成 mock 后，会判断该属性。简单来说作用就是把 spiedInstance  的属性拷贝到 mock 出来的对象上。

![org.mockito.internal.util.MockUtil#createMock](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920120436124.png)

因此，如下代码，得到的两个 Order，属性是一致的。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920120647016.png)

关于 defaultAnswer，可以看到它指定了 CALLS_REAL_METHODS，而对于 mock，这个值则是默认的 RETURNS_DEFAULTS。

猜测这个参数会在实际执行时候产生差异，这里先按下不表，等到下一节再解释。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920120847412.png)

