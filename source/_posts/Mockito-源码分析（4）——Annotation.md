---
title: Mockito 源码分析（4）——Annotation
categories: Unit Test
tags: Mockito
copyright_author: Jitwxs
abbrlink: 208fa04c
date: 2021-09-21 21:52:51
---

在本篇文章中，将主要对 Mockito 中 @Mock、@Spy、@InjectMocks 这三个主要注解进行源码分析。

我们知道，如果想要使用 Mockito，要么需要在测试类的 @Before，执行 `org.mockito.MockitoAnnotations#initMocks`，要么在测试类上添加 `@RunWith(MockitoJUnitRunner.class)` 注解。下面就让我们从 MockitoAnnotations#initMocks 方法看起吧。

## MockitoAnnotations#initMocks

initMocks() 方法一共就两行代码：加载 AnnotationEngine，调用 process() 方法，传入的 testClass 为需要启用 Mockito 的测试类：

![org.mockito.MockitoAnnotations#initMocks](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/2021092122045335.png)

先来看下 AnnotationEngine 的获取逻辑：

- 当 GlobalConfiguration 实例化时，调用 createConfig() 方法，得到 IMockitoConfiguration 实现。只要没有进行额外配置，这里的实现是 DefaultMockitoConfiguration

- 调用 GlobalConfiguration#tryGetPluginAnnotationEngine 得到 AnnotationEngine。对于默认实现（DefaultMockitoConfiguration ）来说，直接从 Plugins 读取实现即可：

  ```java
  org.mockito.internal.configuration.InjectingAnnotationEngine
  ```

  > Plugins 的源码追踪过程就不赘述了，前文在分析 MockerMaker 时已经带大家走过一次了。

![org.mockito.internal.configuration.GlobalConfiguration](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210921220410605.png)

接下来让我们看 process 方法，功能也比较清晰：

- `processIndependentAnnotations()` 初始化 @Mock、@Spy 等注解标识的对象
- `processInjectMocks()` 尝试将 Mock 对象注入到 @InjectMocks 中

![org.mockito.internal.configuration.InjectingAnnotationEngine](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210921221232922.png)

## InjectingAnnotationEngine#processIndependentAnnotations

这个方法的功能是：

- 通过 `IndependentAnnotationEngine#process` 方法，初始化 @Mock 等注解标识的对象
- 通过 `SpyAnnotationEngine#process` 方法，初始化 @Spy 等注解标识的对象
- 向上查找当前类的父类，重复执行，直到当前类为 Object 为止

![org.mockito.internal.configuration.InjectingAnnotationEngine#processIndependentAnnotations](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210921221603197.png)

### IndependentAnnotationEngine#process

- 获取当前 class 标注了注解的字段
-  调用 `createMockFor()` 方法
  - 校验注解是否是有效的注解，比如 @Mock
  - 如果是有效注解，Mockito 生成一个对象（即 proxy 代理类实例）
- alreadyAssigned 字段确保一个字段仅能标注一个有效注解
- 调用 `setField()` 方法，将生成的对象赋上去

![org.mockito.internal.configuration.IndependentAnnotationEngine#process ](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210921222338826.png)

① createMockFor() 方法实现如下：

- IndependentAnnotationEngine 仅支持 @Mock 和 @Captor 注解，每个注解有自己对应的处理器
- 如果是其他注解，返回一个用于容错的 FieldAnnotationProcessor 注解

![org.mockito.internal.configuration.IndependentAnnotationEngine#createMockFor](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210921222812256.png)

下面以 @Mock 注解为例，它的处理器是 MockAnnotationProcessor，让我们看下它的 process() 实现：

- 通过 @Mock 的 Annotation 属性，初始化 MockSettings
- 然后调用 `Mockito#mock()`，至此完成对该字段的 mock

![org.mockito.internal.configuration.MockAnnotationProcessor#processAnnotationForMock](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210921223045449.png)

② setField() 方法实现如下，比较简单就不介绍了。

![org.mockito.internal.util.reflection.FieldSetter#setField](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210921222709692.png)

### SpyAnnotationEngine#process

- 获取当前 class 标注了 @Spy 但没有同时标注 @InjectMocks 注解的字段

- 如果这些字段，同时标注了 @Mock 和 @Captor，则抛出异常

  > 由此可见，@Spy 和 @Mock、@Captor 是不能共存的

- 尝试直接获取字段的实例

  - 如果实例不为 null，且已经是  Mcok 对象了：这里调用了 Mockito.reset，注释解释了可能连续两次调用 initMocks 导致的
  - 如果实例不为 null，且还不是  Mcok 对象：调用 `spyInstance()` 方法，返回一个 Mock 对象，并赋到字段上
  - 如果实例为 null：调用 `spyNewInstance()`，框架帮忙初始化，返回一个 Mock 对象，并赋到字段上

![org.mockito.internal.configuration.SpyAnnotationEngine#process](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210921223433089.png)

对于 spyInstance() 方法，跟 Mockito#spy() 方法比较，几乎一模一样，就不再介绍了。

![org.mockito.internal.configuration.SpyAnnotationEngine#spyInstance](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210921224122616.png)

下面再看 spyNewInstance() 方法，主体的思想，也是想办法构造出一个实例出来，然后走 Mockito Spy 那一套：

![org.mockito.internal.configuration.SpyAnnotationEngine#spyNewInstance](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210921224802738.png)

> 看  spyNewInstance() 这一段代码时候，我其实有一些疑惑的：
>
> （1）`Mockito#spy()` 也有一个根据 class 构建的 API 方法，这边为什么不能直接用那个呢，而要再写这么一大堆？
>
> ![org.mockito.Mockito#spy(java.lang.Class<T>)](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210921224809604.png)
>
> （2）前文在分析 Mock 那块代码时，看到对 proxy class 的实例化，使用的是 `objenesis` 框架，为什么这边不也用这个框架来做呢？
>
> ![org.mockito.internal.creation.instance.ObjenesisInstantiator](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210921225445395.png)



## InjectingAnnotationEngine#processInjectMocks

这个方法的功能是：

- 执行 `injectMocs()` 方法，完成当前 class 的 @InjectMocks 注入
- 向上查找当前类的父类，重复执行，直到当前类为 Object 为止

![org.mockito.internal.configuration.InjectingAnnotationEngine#processInjectMocks](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210921230337108.png)

重点看 injectMock() 方法：

① 这行代码的逻辑，找到当前类下所有标注了 @InjectMocks 注解的字段，并保存到 mockDependentFields 集合中。

![org.mockito.internal.configuration.injection.scanner.InjectMocksScanner](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210921230951018.png)

② 这行代码逻辑，把当前类字段中，被 Mockito 管理的属性，都保存到 mocks 集合中。

![org.mockito.internal.configuration.injection.scanner.MockScanner](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210921231542548.png)

③ 上面两行代码完成了数据准备，得到了：当前类及其父类中，所有要执行 InjectMock 的字段，所有已经准备好的 Mock 对象。

调用 DefaultInjectionEngine#injectMocksOnFields 真正开始注入：

![org.mockito.internal.configuration.DefaultInjectionEngine](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210921232131871.png)

可以看到采用责任链模式：

- injectionStrategies 字段的责任链：ConstructorInjection -> PropertyAndSetterInjection
- postInjectionStrategies字段的责任链：SpyOnInjectedFieldsHandler

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210921232840719.png)

### FieldInitializer

在分析最后一个 apply() 方法前，先插队分析下 `org.mockito.internal.util.reflection.FieldInitializer` 这个类，会在下面的 apply() 方法中用到。

先看下这个类的成员变量：

- `field`：字段，这里其实就是指 @InjectMocks 标注的那个字段
- `fieldOwner`：指明了这个 field 属于哪个对象
- `instantiator`：当 field 字段默认没有初始化时实例，提供一个初始化策略：
  - ParameterizedConstructorInstantiator：通过有参构造的方式实例化，它就是 ConstructorInjection 的策略
  - NoArgConstructorInstantiator：通过无参构造 + 设置属性的方式实例化，它就是 PropertyAndSetterInjection 的策略

![org.mockito.internal.util.reflection.FieldInitializer](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210921234131843.png)

再看下 initialize() 方法，它返回一个 FieldInitializationReport 对象，这个对象中最关键的就是 `fieldInstance` 字段，它其实就是 @InjectMocks 标注的那个字段的实例。

![org.mockito.internal.util.reflection.FieldInitializer#initialize](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210921234822183.png)

通过调用 `acquireFieldInstance()` 方法来获取这个实例，而在这个方法中：

- 先直接获取字段对应的实例（基本为 null）
- 如果取不到，就得走 instantiator 的初始化策略了（这个到下面分析 OngoingMockInjection#apply 再来看）

![org.mockito.internal.util.reflection.FieldInitializer#acquireFieldInstance](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210921235044128.png)

### OngoingMockInjection#apply

现在来分析 apply() 这个方法，这里的 field 就是单个 injectMocks 字段：

```java
// org.mockito.internal.configuration.injection.MockInjection.OngoingMockInjection#apply
public void apply() {
    for (Field field : fields) {
        injectionStrategies.process(field, fieldOwner, mocks);
        postInjectionStrategies.process(field, fieldOwner, mocks);
    }
}
```

先看 injectionStrategies 负责的责任链：

**（1）org.mockito.internal.configuration.injection.ConstructorInjection#process**

![org.mockito.internal.configuration.injection.ConstructorInjection#processInjection](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210922000004231.png)

如上图所示，它构造的 FieldInitializer 对象中的 instantiator 策略是 ParameterizedConstructorInstantiator，在这个策略中：

- 先对所有的构造方法排序，选出其中一个构造方法
- 从所有的 mock 对象中，筛出构造方法需要的参数
- 尝试通过这个构造方法创建实例
- 如果创建成功，则设置到 field 上；如果失败，则抛出异常，由上层捕获。

![20210921235818019](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210921235818019.png)

① 如何选取出一个构造方法呢？规则是：优先取参数数量少的，但至少也得有一个参数。如下图所所示：

![org.mockito.internal.util.reflection.FieldInitializer.ParameterizedConstructorInstantiator#biggestConstructor](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210922000438847.png)

② 从所有的 mock 对象中，筛出上一步构造方法中需要的对象。如下图所所示：

![org.mockito.internal.configuration.injection.ConstructorInjection.SimpleArgumentResolver#resolveTypeInstances](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210922000622658.png)

**（2）org.mockito.internal.configuration.injection.PropertyAndSetterInjection**

如果 ConstructorInjection 无法选出的话，就走到 PropertyAndSetterInjection 了，在这个方法中：

- 通过 NoArgConstructorInstantiator，调用无参构造，初始化对象，并赋值到字段上
- 如果获初始化失败，cannotInitializeForInjectMocksAnnotation() 抛出异常
- 从所有的 mock 对象中，尝试获取到一个匹配的

![20210922001235441](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210922002414461.png)

① NoArgConstructorInstantiator#instantiate 方法比较简单，单纯的通过无参构造创建实例。

![org.mockito.internal.util.reflection.FieldInitializer.NoArgConstructorInstantiator#instantiate](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210922002458257.png)

② PropertyAndSetterInjection#injectMockCandidates方法主要功能是：

- `orderedInstanceFieldsFrom()` 确定需要被注入的所有字段

- `injectMockCandidatesOnFields()` 负责注入

  > 没太明白为什么要调用两次 injectMockCandidatesOnFields()，有懂的同学吗？

- `mockCandidateFilter()` 责任链模式实现了字段匹配的功能

![org.mockito.internal.configuration.injection.PropertyAndSetterInjection#injectMockCandidates](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210922003218676.png)

这里重点看下如何匹配的那个责任链：

- `TypeBasedCandidateFilter#TypeBasedCandidateFilter`：匹配类型

  ![org.mockito.internal.configuration.injection.filter.TypeBasedCandidateFilter#TypeBasedCandidateFilter](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210922003537951.png)

- `NameBasedCandidateFilter#filterCandidate`：匹配参数名

  ![org.mockito.internal.configuration.injection.filter.NameBasedCandidateFilter#filterCandidate](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210922004300786.png)

- `TerminalMockCandidateFilter#filterCandidate`：先尝试走 set 方法赋值，不行的话走反射赋值。

  ![org.mockito.internal.configuration.injection.filter.TerminalMockCandidateFilter#filterCandidate](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/202109220045587.png)

## Conclusion

通过上面的内容，为大家讲解了 @Mock、@Spy、@InjectMocks 注解是如何工作的。
