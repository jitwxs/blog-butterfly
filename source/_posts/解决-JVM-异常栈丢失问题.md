---
title: 解决 JVM 异常栈丢失问题
tags:
  - JVM
  - OmitStackTraceInFastThrow
categories: Java
abbrlink: f8cb808a
date: 2020-12-19 22:13:44
---

JVM 默认情况下，当代码的某一个位置高频率抛出同一异常时，为了节约性能，JVM 会对以下异常类型的异常栈进行优化，不再打印完整异常栈。

- NullPointerException
- ArithmeticException
- ArrayIndexOutOfBoundsException
- ArrayStoreException
- ClassCastException

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201219221754.png)

往往出现这种情况，只能期望找到最初报错的日志，去找到具体的异常行。

除此之外，我们可以通过指定 `OmitStackTraceInFastThrow` 这个 JVM 参数去关掉这个优化，仅需要在应用启动参数中添加如下即可：

```
-XX:-OmitStackTraceInFastThrow
```

> 这个参数对于大部分类型的 JVM 都是适用的，已在 HotSpot 和 Zing上验证可行，可以[点击这里](https://chriswhocodes.com/zing_jdk8_options.html)查询各厂商 JVM 参数列表。

在 IDEA 中的话，配置如下即可：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201219222602.png)

另外需要注意的是，不加控制的打印异常栈，对 CPU 和内存都会有影响。最好在出现异常后尽快解决，或对异常打印增加频率控制。总之就是这个思想，懂就行。
