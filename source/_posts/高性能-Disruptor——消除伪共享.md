---
title: 高性能 Disruptor——消除伪共享
categories:
  - Java
  - Disruptor
tags: Disruptor
abbrlink: 13836b16
date: 2020-03-15 20:49:06
related_repos:
  - name: Test Demo
    url: https://github.com/jitwxs/disruptor-study/tree/master/disruptor-demo/src/test/java/jit/wxs/disruptor/demo/flashsharing
references:
  - name: Mechanical Sympathy
    url: http://mechanical-sympathy.blogspot.com/
  - name: Java专家系列：CPU Cache与高性能编程
    url: https://blog.csdn.net/karamos/article/details/80126704
  - name: 剖析Disruptor:为什么会这么快？（二）神奇的缓存行填充
    url: http://ifeve.com/disruptor-cacheline-padding/
  - name: Disruptor 缓冲行填充的进一步解释
    url: https://www.jianshu.com/p/e1a1b950fc4a
---

## 一、CPU Cache

存储设备往往是速度越快价格越昂贵，速度越快价格越低廉。在计算机中，CPU 的速度远高于主存的速度，而主存的速度又远高于磁盘的速度。为了解决不同存储部件的速度不对等问题，让高速设备充分发挥性能，引入了多级缓存机制。

为了解决内存和 CPU 的速度不匹配问题，相继引入了 L1 Cache、L2 Cache、L3 Cache，数字越小，容量越小，速度越快，位置越接近 CPU。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202003/20200315233546582.png)

现在的 CPU 都是由多个处理器，每个处理器由多个核心构成。 一个处理器对应一个物理插槽，不同的处理器间通过 QPI 总线相连。一个处理器间的多核共享 L3 Cache。一个核包含寄存器、L1 Cache、L2 Cache，下图是Intel Sandy Bridge CPU架构：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202003/20200315233629903.png)

## 二、缓存行与伪共享

缓存中的数据并不是独立的进行存储的，它的最小存储单位是**缓存行**，缓存行的大小是2的整数幂个字节，最常见的缓存行大小是 **64** 字节。CPU 为了执行的高效，会在读取某个对象时，从内存上加载 64 的整数倍的长度，来补齐缓存行。

以 Java 的 long 类型为例，它是 8 个字节，假设我们存在一个长度为 8 的 long 数组 arr，那么CPU 在读取 arr[0] 时，首先查询缓存，缓存没有命中，缓存就会去内存中加载。由于缓存的最小存储单位是缓存行，64 字节，且数组的内存地址是连续的，则将 arr[0] 到 arr[7] 加载到缓存中。后续 CPU 查询 arr[6] 时候也可以直接命中缓存。

![Cache Line](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202003/20200315220302457.png)

现在假设多线程情况下，线程 A 的执行者 CPU Core-1 读取 arr[1]，首先查询缓存，缓存没有命中，缓存就会去内存中加载。从内存中读取 arr[1] 起的连续的 64 个字节地址到缓存中，组成缓存行。由于从arr[1] 起，arr 的长度不足够 64 个字节，只够 56 个字节。假设最后 8 个字节内存地址上存储的是对象 bar，那么对象 bar 也会被一起加载到缓存行中。

![Cache Line](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202003/20200315221131793.png)

现在有另一个线程 B，线程 B 的执行者 CPU Core-2 去读取对象 bar，首先查询缓存，发现命中了，因为 Core-1 在读取 arr 数组的时候也顺带着把 bar 加载到了缓存中。

这就是**缓存行共享**，听起来不错，但是一旦牵扯到了写入操作就不妙了。

假设 Core-1 想要更新 arr[7] 的值，根据 CPU 的  `MESI` 协议，那么**它所属的缓存行就会被标记为失效**。因为它需要告诉其他的 Core，这个 arr[7] 的值已经被更新了，缓存已经不再准确了，你必须得重新去内存拉取。但是由于缓存的最小单元是缓存行，因此只能把 arr[7] 所在的一整行给标识为失效。

此时 Core-2 就会很郁闷了，刚刚还能够从缓存中读取到对象 bar，现在再读取却被告知缓存行失效，必须得去内存重新拉取，延缓了 Core-2 的执行效率。

这就是**缓存伪共享**问题，两个毫无关联的线程执行，一个线程却因为另一个线程的操作，导致缓存失效。这两个线程其实就是对同一缓存行产生了竞争，降低了并发性。

## 三、Disruptor 缓存行填充

Disruptor 为了解决伪共享问题，使用的方法是**缓存行填充**。这是一种以空间换时间的策略，主要思想就是通过往对象中填充无意义的变量，来保证整个对象**独占缓存行**。

举个例子，以 Disruptor 中的 Sequence 为例，在 volatile long value 的前后各放置了 7 个 long 型变量，确保 value 独占一个缓存行。

```java
public class Sequence extends RhsPadding {
    private static final long VALUE_OFFSET;
    
    static {
        VALUE_OFFSET = UNSAFE.objectFieldOffset(Value.class.getDeclaredField("value"));
        ...
    }
    ...
}

class RhsPadding extends Value {
    protected long p9, p10, p11, p12, p13, p14, p15;
}

class Value extends LhsPadding {
    protected volatile long value;
}

class LhsPadding {
    protected long p1, p2, p3, p4, p5, p6, p7;
}
```

如下图所示，其中 V 就是 Value 类的 value，P 为 value 前后填充的无意义 long 型变量，U 为其它无关的变量。不论什么情况下，都能保证 V 不和其他无关的变量处于同一缓存行中，这样 V 就不会被其他无关的变量所影响。

![Padding 填充](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202003/20200315233704730.jpg)

这里的 V 也不限定为 long 类型，其实**只要对象的大小大于等于8个字节**，通过前后各填充 7 个 long 型变量，就一定能够保证独占缓存行。

此处以 Disruptor 的 RingBuffer 为例，最左边的 7 个 long 型变量被定义在顶级父类 RingBufferPad 中，最右边的 7 个 long 型变量被定义在 RingBuffer 的最后一行变量定义中，这样所有的需要独占的变量都被左右 long 型给包围，确保会独占缓存行。

```java
public final class RingBuffer<E> extends RingBufferFields<E> implements Cursored, EventSequencer<E>, EventSink<E> {
    public static final long INITIAL_CURSOR_VALUE = Sequence.INITIAL_VALUE;
    protected long p1, p2, p3, p4, p5, p6, p7;
    ...
}

abstract class RingBufferFields<E> extends RingBufferPad
{
    ...
}

abstract class RingBufferPad {
    protected long p1, p2, p3, p4, p5, p6, p7;
}
```

## 四、@Contended

在 JDK 1.8 中，提供了 ` @sun.misc.Contended ` 注解，使用该注解就可以让变量独占缓存行，不再需要手动填充了。 注意，JVM 需要添加参数 `-XX:-RestrictContended` 才能开启此功能。

如果该注解被定义在了类上，表示该类的每个变量都会独占缓存行；如果被定义在了变量上，通过指定 groupName，相同的 groupName 会独占同一缓存行。

```java
// 类前加上代表整个类的每个变量都会在单独的cache line中
@sun.misc.Contended
public class ContendedData {
    int value;
    long modifyTime;
    boolean flag;
    long createTime;
    char key;
}

// 同一 groupName 在同一缓存行
public class ContendedGroupData {
    @sun.misc.Contended("group1")
    int value;
    @sun.misc.Contended("group1")
    long modifyTime;
    @sun.misc.Contended("group2")
    boolean flag;
    @sun.misc.Contended("group3")
    long createTime;
    @sun.misc.Contended("group3")
    char key;
}
```

`@Contended` 在 JDK 源码中已经有所应用，以 Thread 类为例，为了保证多线程情况下随机数的操作不会产生伪共享，相关的变量被设置为同一 groupName。

```java
public class Thread implements Runnable {
    ...
    // The following three initially uninitialized fields are exclusively
    // managed by class java.util.concurrent.ThreadLocalRandom. These
    // fields are used to build the high-performance PRNGs in the
    // concurrent code, and we can not risk accidental false sharing.
    // Hence, the fields are isolated with @Contended.

    /** The current seed for a ThreadLocalRandom */
    @sun.misc.Contended("tlr")
    long threadLocalRandomSeed;

    /** Probe hash value; nonzero if threadLocalRandomSeed initialized */
    @sun.misc.Contended("tlr")
    int threadLocalRandomProbe;

    /** Secondary seed isolated from public ThreadLocalRandom sequence */
    @sun.misc.Contended("tlr")
    int threadLocalRandomSecondarySeed;
    
    ...
}
```

## 五、速度测试

将 volatile long value 封装为对象，四线程并行，每个线程循环 1 亿次，对 value 进行更新操作，测试缓存行对速度的影响。

- CPU：AMD 3600 3.6 GHz
- Memory：16 GB

| 测试 Case                | 执行耗时 ms |
| :----------------------- | ----------- |
| 未做处理                 | 11781       |
| @Contended不开启 JVM参数 | 11049       |
| Padding 填充             | 4312        |
| @Contended开启 JVM参数   | 4436        |
