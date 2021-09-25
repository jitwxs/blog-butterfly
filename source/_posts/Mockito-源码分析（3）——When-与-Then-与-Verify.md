---
title: Mockito 源码分析（3）——When 与 Then 与 Verify
categories: 单元测试
tags: Mockito
abbrlink: 921b3e8a
date: 2021-09-20 20:32:05
related_repos:
  - name: mock-sample
    url: https://github.com/jitwxs/blog-sample/tree/master/javase-sample/mock-sample
    rel: nofollow noopener noreferrer
    target: _blank
---

先问大家一个问题，下图中我使用框子标出的代码，你觉得是红色、绿色、蓝色，执行的顺序是怎样的呢？

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920145157592.png)

:::details 看一眼答案

执行顺序：蓝色 > 绿色 > 红色，想错了的面壁吧。。。

:::

在开始本文之前，咱先根据前面的文章，明确一下已知条件：

- 使用 Mockito#mock 返回的对象，它所属的类是被 mock 类的 proxy 代理类。
- 对于该 proxy 类：
  - 是被 mock 类的子类，并且实现了 MockAccess 接口
  - 具有一个名为 mockitoInterceptor 的属性，其类型为 MockMethodInterceptor
  - 类中所有的方法都被拦截了，拦截处理的类为 DispatcherDefaultingToRealMethod

> 如果以上条件你还有疑问，请先别急着往下看，复习下上篇文章，巩固下认知。

下文我将使用这个例子为例，来进行 when 和 then API 的源码分析。

## When

### Matcher

首先执行的方法是 `Mockito.eq(1L)`，在 Mockito 的父类 ArgumentMatchers，为我们提供了一系列的 Match 方法，`eq()` 只是其中一个。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920161010226.png)

这些方法的底层调用的都是 reportMatcher 这个方法：

![org.mockito.ArgumentMatchers#reportMatcher](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920161046225.png)

在这个方法中，有个我们的老朋友：`mockingProgress()`，它在上一节中跟我们有过一面之缘，在本篇文章我们中将会频繁的碰上它。重复下它的作用： **基于 ThreadLocal 实现保存了一个 MockingProgressImpl**。

因此 getArgumentMatcherStorage() 操作的就是当前线程 MockingProgressImpl 中的 ArgumentMatcherStorage 对象，并调用它的 reportMatcher() 方法，如下图所示。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920161441914.png)

reportMatcher() 方法内部，其实是将 Matcher 封装成 LocalizedMatcher 对象，丢进了一个 Stack 中。

> LocalizedMatcher 的构造方法中还有一个 Location 的变量，它的作用其实是保存下来当前调用的源码路径。
>
> ![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920161933594.png)
>
> 以我当前的 UT 为例，运行后它长这个样子：
>
> ![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920162028056.png)
>
> 这其实是个小技巧，通过 `new Throwable()` 获取堆栈信息，然后从中获取有效信息，挺有意思的。

### DispatcherDefaultingToRealMethod 

执行完  `Mockito.eq(1L)`，下面就开始打桩了，开头说过 OrderClient 这个类已经被 proxy 代理了，因此当 `orderClient.queryById(...)`  被调用时，会被 DispatcherDefaultingToRealMethod 所拦截。

在这个类中有两个方法，至于具体执行哪一个，我暂时没找到明确的官方证明，但根据我的 Debug 和方法名称推测：

- interceptSuperCallable：对于自身有实现的普通方法，被该方法所拦截。
- interceptAbstract：对于抽象方法、接口类（非 default）的方法这种自身没有实现的方法，被该方法所拦截。

这两个方法唯一的不同，就是当 Stub 不匹配后的容错处理逻辑。对于 interceptSuperCallable()，它可以调用自身的实现，即 `callable.call()`。而 interceptAbstract()，由于自身并没有实现，所以只能抛出异常了。

![org.mockito.internal.creation.bytebuddy.MockMethodInterceptor.DispatcherDefaultingToRealMethod](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920151054444.png)

该类中用到了非常多的注解，这些注解其实都是 bytebuddy 所提供的，根据名字相信大家已经能猜出它的涵义了。

| 注解          | 此处含义                                               |
| ------------- | ------------------------------------------------------ |
| @This         | 当前对象                                               |
| @FieldValue   | 获取对象中指定名称的字段                               |
| @Origin       | 调用的方法信息                                         |
| @AllArguments | 调用的方法参数列表                                     |
| @SuperCall    | 将原本的调用行为封装成 Future，可以直接执行            |
| @StubValue    | 没发现啥用，当 mockitoInterceptor 不存在时作为容错结果 |

回过头来，OrderClient 本身是个接口，因此会执行 interceptAbstract 方法，然后被交给了 MockMethodInterceptor 的doIntercept 方法处理，如下图所示：

![org.mockito.internal.creation.bytebuddy.MockMethodInterceptor#doIntercept](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/2021092016270598.png)

在该方法中，又交给了 handle 处理，这个 handle 其实就是在我们构建 proxy 时传入的那个 MockHandleImpl 对象。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920162838656.png)

记不得的同学，复习下这几个方法：

- org.mockito.internal.util.MockUtil#createMock
- org.mockito.internal.creation.bytebuddy.SubclassByteBuddyMockMaker#createMock

handle 方法的第一个参数是 Invocation，这个对象其实就是把本次调用的信息给整合起来了：

![org.mockito.internal.invocation.DefaultInvocationFactory#createInvocation](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920164735295.png)

### MockHandleImpl

> 当你对 proxy 类，无论是 when 还是 then 还是 verify 还是普通调用啥的，都会执行 MockHandlerImpl#handle 方法，因此该方法内容很长。所以我会只介绍当前相关的，其他的逻辑会暂时跳过并在下文介绍。

在 MockHandlerImpl#handle 方法中，第一个需要关注的逻辑是如下代码：

```java
InvocationMatcher invocationMatcher = matchersBinder.bindMatchers(mockingProgress().getArgumentMatcherStorage(), invocation);
```

这个方法的主要作用是：**将之前保存的 ArgumentMatchers 信息都取出来，然后跟 Invocation 绑定在一起，组成 InvocationMatcher**。

![org.mockito.internal.handler.MockHandlerImpl#handle](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920164458509.png)

第二块需要关注逻辑是如下代码：

```java
// prepare invocation for stubbing
invocationContainer.setInvocationForPotentialStubbing(invocationMatcher);
OngoingStubbingImpl<T> ongoingStubbing = new OngoingStubbingImpl<T>(invocationContainer);
mockingProgress().reportOngoingStubbing(ongoingStubbing);
```

首先 `setInvocationForPotentialStubbing` 将 Invocation 存入了 registeredInvocations，然后将 InvocationMatcher 放入属性 invocationForStubbing 中。

最后再将整个 invocationContainer 封装成 OngoingStubbingImpl 放入 ThreadLocal 中。

![org.mockito.internal.handler.MockHandlerImpl#handle](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920170007955.png)

### When

执行完  `orderClient.queryById(...)`，下面该执行的就是 `Mocktio.when(...)` 了，如下图所示：

![org.mockito.Mockito#when](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920171515878.png)

上面代码的主要功能是：

- 通过 `stubbingStarted()` 设置 ThreadLocal 中的 stubbingInProgress。

  > 这里我们可以看到设置 stubbingInProgress 前它会校验 stubbingInProgress 是否存在，因此如果你直接连续两次调用 when 就会触发这里的校验失败。
  >
  > ![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920172158014.png)

- 从 ThreadLocal 中取出并清除之前调用 `orderClient.queryById(...)`存入的  OngoingStubbingImpl。

## Then

```java
OngoingStubbing<Order> stubbing = Mockito.when(orderClient.queryById(Mockito.eq(1L)));
```

这行代码执行完毕后 ，目前已知的条件有：

- When 方法执行的相关信息已经被保存到了 OngoingStubbing 中
- ThreadLocal 中目前已经设置了 stubbingInProgress，它记录了 ``Mockito.when(orderClient.queryById(Mockito.eq(1L)))`` 的源码信息

- OngoingStubbingImpl 中的 invocationContainer 中的 registeredInvocations，存储了 `orderClient.queryById(...)` 的调用信息

下面开始分析 `stubbing.thenReturn(...)` 源码，你会发现不论你是 thenReturn 还是 thenThrow 还是 thenCallRealMethod，走的其实都是 thenAnswer 的 API。

![org.mockito.internal.stubbing.BaseStubbing#thenReturn](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920173858054.png)

thenAnswer 方法代码如下，主要有三个功能：

![org.mockito.internal.stubbing.OngoingStubbingImpl#thenAnswer](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920175106333.png)

- 判断 registeredInvocations 中是否有数据
- 将 answer 数据作为期望的返回值，保存起来（addAnswer）
- 返回 ConsecutiveStubbing 对象（这个下文再细说）

我们重点看 `invocationContainer.addAnswer(answer, strictness)` ，如下图所示，图中蓝色序号是比较关键的代码：

![org.mockito.internal.stubbing.InvocationContainerImpl#addAnswer](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920180242259.png)

① 通过调用 removeLast 方法从 invocationContainer 中移除一个 registeredInvocations。来保证了每个 OngoingStubbing 仅能设置一次 answer。

② 调用重载的 addAnswer 方法时， isConsecutive 方法传递为 false，它跟 OngoingStubbing 强绑定，即 OngoingStubbing 调用时永远为 false。

③ 将 ThreadLocal 中的 stubbingInProgress 清除，表示之前调用 when 的数据已经不需要了。

④ 设置结果，这里封装了一个 StubbedInvocationMatcher 对象，它其实就代表了一个桩（Stub）。可以看到它的构造方法中已经把需要的数据都准备好了：

- `invocation.getInvocation()`：对应的方法信息，即 `orderClient.queryById(...)`
- `invocation.getMatchers()`：对应的方法参数匹配信息，即 `Mockito.eq(1L)`
- `answer`：对应的方法返回值

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920181341071.png)

现在我们再把刚刚带过的 ConsecutiveStubbing 给补上，通过类图，我们发现它跟 OngoingStubbing 同级。

![ConsecutiveStubbing](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920181503775.png)

二者的区别是使用 ConsecutiveStubbing 时，可以对一个 Stub，添加多个 answer，它的 thenAnswer 方法参数的 isConsecutive 就为 true 了，这也是为什么 StubbedInvocationMatcher 底层的 answers 用了 Queue 来存储的原因。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920181626252.png)

因此，下面的写法是错误的：

```java
OngoingStubbing<Order> stubbing = Mockito.when(orderClient.queryById(Mockito.eq(1L)));

stubbing.thenReturn(order1);
stubbing.thenReturn(order2);
```

因为 OngoingStubbing 的 thenAnswer 方法会调用 removeLast()，里面会清除 registeredInvocations，导致第二次执行时无法通过校验：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920175340509.png)

正确的写法应该是：

```java
// 返回子类 OngoingStubbingImpl
OngoingStubbing<Order> stubbing = Mockito.when(orderClient.queryById(Mockito.eq(1L)));
// 返回子类 ConsecutiveStubbing
OngoingStubbing<Order> stubbing1 = stubbing.thenReturn(order1);
stubbing1.thenReturn(order2);

// 或者
OngoingStubbing<Order> stubbing = Mockito.when(orderClient.queryById(Mockito.eq(1L)));
stubbing.thenReturn(order1).thenReturn(order2);
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920182604604.png)

## Running

下面开始真正测试了，当执行如下代码时，期望能够从 Stub 中取出数据：

```java
final Order order1 = orderClient.queryById(1L);
```

源码还得回到 `MockHandlerImpl#handle` 来，没办法谁让所有方法都被它给代理了呢。

你会发现现在走的流程跟 `orderClient.queryById(Mockito.eq(1L))` 一模一样，唯一不同的是 `invocationContainer.findAnswerFor(invocation)` 方法返回的不再是 NULL。

findAnswerFor() 的功能就是根据 invocation 从所有的 Stub 中寻找是否存在匹配的 Stub，如果找到匹配的，返回对应的结果。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920183215269.png)

> `answer` 方法中会判断 answers 的元素数量，如果有多个（ConsecutiveStubbing 情况），返回头部元素并从 Queue 中移除；如果只有一个（OngoingStubbingImpl 或仅剩一个的 ConsecutiveStubbing 情况），返回这个并且不从 Queue 中移除。

下面让我们重点看下 findAnswerFor 是如何匹配 Stub 的，如下所示：

![org.mockito.internal.stubbing.InvocationContainerImpl#findAnswerFor](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920184152287.png)

可以看到匹配的条件包括：

- 对象是否是同一个
- 方法的名称和参数是否完全相同
- 方法的调用参数是否完全相同

这里特别要注意的是，第一个条件比较的是对象，而不是类，因此下面的 UT 是无法通过的：

```java
final OrderClient orderClient1 = PowerMockito.mock(OrderClient.class);
final OrderClient orderClient2 = PowerMockito.mock(OrderClient.class);

PowerMockito.when(orderClient1.queryById(Mockito.eq(1L))).thenReturn(order);

final Order actualOrder = orderClient2.queryById(1L); // null
Assert.assertEquals(order, actualOrder);
```

---

插个题外话，上面我说了一句：

> 你会发现现在走的流程跟 `orderClient.queryById(Mockito.eq(1L))` 一模一样，唯一不同的是 `invocationContainer.findAnswerFor(invocation)` 方法返回的不再是 NULL。

那么你觉得下面的代码运行结果是啥？

```java
final OrderClient orderClient = PowerMockito.mock(OrderClient.class);

PowerMockito.when(orderClient.queryById(Mockito.eq(1L))).thenReturn(order);

final Order order1 = orderClient.queryById(1L);
final Order order2 = orderClient.queryById(Mockito.eq(1L));

Assert.assertEquals(order1, order2);
```

一开始我认为这走的逻辑都是一样的啊，上面的代码执行肯定是相等的，其实运行结果是不相等。原因就在于 `Mockito.eq(1L)` 并不是真正的参数，而是一个 matcher，如下图所示。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920185825134.png)

所以如果非要让上面的代码运行相等，只要把 1L 改成 0L 就行了，哈哈哈（PS：这是玩笑话，别学...）。

## Verify

最后补充下 Verify，这里我就以最简单的 Mocito.times 为例了。UT 如下，验证是否执行了一次。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/2021092022015457.png)

when..then.. 处的代码就直接忽略了，先从 `orderClient.queryById(1L)` 方法开始看。

我们直接跳到 MockHandlerImpl#handle，让我们再回顾下 `invocationContainer.setInvocationForPotentialStubbing` 这段代码，它会将本次的 invocation 行为存储进 invocationContainer 的 registeredInvocations 中。然后通过 `invocationContainer.findAnswerFor` 尝试获取一个匹配的 Stub，然后返回结果。

![org.mockito.internal.handler.MockHandlerImpl#handle](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920222517458.png)

> 只需要重点注意，本次的 invocation 行为已经被保存起来了（是不是有点流量录制的味道了？）。

接下来让我们 Mockito#verify 方法，除掉校验和非主干逻辑，真正核心的就是 `mockingProgress.verificationStarted` 一行代码：将当前 mock 对象和要 verify 的模式（actualMode）封装成 MockAwareVerificationMode 对象，然后存入 ThreadLocal 中。

![org.mockito.Mockito#verify](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920220759845.png)

设置完毕后，继续执行后续代码：`queryById(1L)`，继续把视野移到 MockHandlerImpl#handle，之前一直被我们无视的代码块终于走到了。因为上一步将 MockAwareVerificationMode 存入到 ThreadLocal 中，因此当执行 `mockingProgress().pullVerificationMode()` 时就能取出来了。

![org.mockito.internal.handler.MockHandlerImpl#handle](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920223118901.png)

首先判断 MockAwareVerificationMode 需要验证的对象和当前对象是否相同（注意这里用等号判断，比较的是地址）。然后构造 VerificationDataImpl，比较简单，如下所示：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920223428776.png)

主要的验证就是 `verificationMode.verify(data)` 这行代码，如果通过后，返回 null 即可；如果未通过，则会抛出异常。

下面我就以 Times 为例，看看它的 verify 方法实现：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920224251021.png)

- `data.getAllInvocations()`：获取直接所有的 Invocation 记录

- `data.getTarget()`：获取想要验证的 Invocation

- 如果 wantedCount > 0，表明需要验证次数，调用 checkMissingInvocation() 方法，防止 getAllInvocations() 中一个匹配的都没有，存在 null 的情况

  ![org.mockito.internal.verification.checkers.MissingInvocationChecker#checkMissingInvocation)](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920224604843.png)

  - findInvocations()：筛选匹配的 Invocation
  - 如果 actualInvocations.isEmpty()，说明一个匹配的都没有，后面代码都是拼装异常了，没啥意思直接跳过

- 调用 checkNumberOfInvocations() 方法，比对次数，这个太简单就不介绍了。

  ![org.mockito.internal.verification.checkers.NumberOfInvocationsChecker#checkNumberOfInvocations](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920224802988.png)

## Conclusion

### registeredInvocations

总结下 org.mockito.internal.stubbing.InvocationContainerImpl#registeredInvocations：

（1）什么时候会有值？

基本每次调用 proxy 代理类的方法时，都会将当前的 Invocation 存入。

（2）什么时候消费它？

- 每调用一次 when（ConsecutiveStubbing 除外），registeredInvocations 就会 removeLast。

- 调用 verify 时，会拿所有的 registeredInvocations，进行比较，但不会移除。

### MockHandlerImpl#handle

不得不说这个方法太重要了，再总结下它的代码模块吧。

![org.mockito.internal.handler.MockHandlerImpl#handle](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202109/20210920230251924.png)

