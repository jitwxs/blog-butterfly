---
title: 为什么阿里巴巴要禁用 Executors 创建线程池？
categories:
  - Java Basic
  - Concurrency Coding
tags: 线程池
abbrlink: f3caf014
date: 2019-12-01 22:07:59
copyright_author: 何甜甜在吗
copyright_url: 'https://juejin.cn/post/6844903989675458574'
---

## 一、写在前面

通过阅读本篇文章你将了解到：

- 线程池的定义
- Executors 创建线程池的几种方式
- ThreadPoolExecutor 对象
- 线程池执行任务逻辑和线程池参数的关系
- Executors 创建返回 ThreadPoolExecutor 对象
- OOM 异常测试
- 如何定义线程池参数

## 二、线程池的定义

管理一组工作线程。通过线程池复用线程有以下几点优点：

- **减少资源创建** => 减少内存开销，创建线程占用内存
- **降低系统开销** => 创建线程需要时间，会延迟处理的请求
- **提高稳定稳定性** => 避免无限创建线程引起的 `OOM`(OutOfMemoryError)

## 三、Executors 创建线程池的方式

根据返回的对象类型创建线程池可以分为三类：

- 创建返回 `ThreadPoolExecutor` 对象
- 创建返回 `ScheduleThreadPoolExecutor` 对象
- 创建返回 `ForkJoinPool` 对象

本文只讨论创建返回 `ThreadPoolExecutor` 对象。

## 四、ThreadPoolExecutor 对象

在介绍 `Executors` 创建线程池方法前先介绍一下 `ThreadPoolExecutor`，因为这些创建线程池的静态方法都是返回`ThreadPoolExecutor` 对象，和我们手动创建 `ThreadPoolExecutor` 对象的区别就是我们不需要自己传构造函数的参数。

`ThreadPoolExecutor ` 的构造函数共有四个，但最终调用的都是同一个：

```java
public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit,
                          BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
```

- **corePoolSize** => 线程池核心线程数量
- **maximumPoolSize** => 线程池最大数量
- **keepAliveTime** => 空闲线程存活时间
- **unit** => 时间单位
- **workQueue** => 线程池所使用的缓冲队列
- **threadFactory** => 线程池创建线程使用的工厂
- **handler** => 线程池对拒绝任务的处理策略


## 五、线程池执行任务逻辑和线程池参数的关系

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201912/20191201215641555.png)

执行逻辑说明：

- 判断核心线程数是否已满，核心线程数大小和 `corePoolSize` 参数有关，未满则创建线程执行任务。
- 若核心线程池已满，判断队列是否满，队列是否满和 `workQueue` 参数有关，若未满则加入队列中。
- 若队列已满，判断线程池是否已满，线程池是否已满和 `maximumPoolSize` 参数有关，若未满创建线程执行任务。
- 若线程池已满，则采用拒绝策略处理无法执执行的任务，拒绝策略和 `handler` 参数有关。

## 六、Executors 创建返回 ThreadPoolExecutor 对象

`Executors` 创建返回ThreadPoolExecutor对象的方法共有三种：

- Executors#**newCachedThreadPool** => 创建可缓存的线程池
- Executors#**newSingleThreadExecutor** => 创建单线程的线程池
- Executors#**newFixedThreadPool** => 创建固定长度的线程池

### 6.1 newCachedThreadPool

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS, new SynchronousQueue<Runnable>());
}
```

`CachedThreadPool` 是一个根据需要创建新线程的线程池

- corePoolSize => 0，核心线程池的数量为 0
- maximumPoolSize => `Integer.MAX_VALUE`，可以认为可以无限创建线程
- keepAliveTime => 60L
- unit => 秒
- workQueue => `SynchronousQueue`

当一个任务提交时，`corePoolSize` 为 0 不创建核心线程，`SynchronousQueue`是一个不存储元素的队列，可以理解为队里永远是满的，因此最终会创建非核心线程来执行任务。对于非核心线程空闲60s时将被回收。**因为 `Integer.MAX_VALUE` 非常大，可以认为是可以无限创建线程的，在资源有限的情况下容易引起OOM异常。**

### 6.2 newSingleThreadExecutor

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService(
        new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>()));
}
```

`SingleThreadExecutor` 是单线程线程池，只有一个核心线程

- corePoolSize => 1，核心线程池的数量为 1
- maximumPoolSize => 1，线程池最大数量为 1，即最多只可以创建一个线程，唯一的线程就是核心线程
- keepAliveTime => 0L
- unit => 毫秒
- workQueue => ` LinkedBlockingQueue`

当一个任务提交时，首先会创建一个核心线程来执行任务，如果超过核心线程的数量，将会放入队列中，**因为 `LinkedBlockingQueue` 是长度为 `Integer.MAX_VALUE` 的队列，可以认为是无界队列，因此往队列中可以插入无限多的任务，在资源有限的时候容易引起`OOM`异常**，同时因为无界队列，`maximumPoolSize` 和 `keepAliveTime` 参数将无效，压根就不会创建非核心线程。

### 6.3 newFixedThreadPool

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>());
}
```

`FixedThreadPool` 是固定核心线程的线程池，固定核心线程数由用户传入

- corePoolSize => nThreads，核心线程池的数量为1
- maximumPoolSize => nThreads，线程池最大数量为nThreads，即最多只可以创建nThreads个线程
- keepAliveTime => 0L
- unit => 毫秒
- workQueue => LinkedBlockingQueue 它和 `SingleThreadExecutor` 类似，唯一的区别就是**核心线程数**不同，并且由于使用的是 `LinkedBlockingQueue`，在资源有限的时候容易引起 `OOM` 异常。

### 6.4 总结

- FixedThreadPool 和 SingleThreadExecutor => 允许的请求队列长度为 Integer.MAX_VALUE，可能会堆积大量的请求，从而引起 `OOM` 异常。
- CachedThreadPool => 允许创建的线程数为 Integer.MAX_VALUE，可能会创建大量的线程，从而引起 `OOM` 异常。

这就是为什么禁止使用 `Executors` 去创建线程池，而是推荐自己去创建 `ThreadPoolExecutor` 的原因。

## 七、OOM 异常测试

 理论上会出现`OOM`异常，下面测试验证下。

```java
public class TaskTest {
    public static void main(String[] args) {
        ExecutorService es = Executors.newCachedThreadPool();
        int i = 0;
        while (true) {
            es.submit(new Task(i++));
        }
    }
}
```

使用 `Executors` 创建的 `CachedThreadPool`，往线程池中无限添加线程 在启动测试类之前先将 `JVM` 内存调整小一点，不然很容易将电脑跑出问题，在 `idea` 里：`Run` -> `Edit Configurations`。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201912/20191201220055454.png)

`JVM`参数说明：

- -Xms10M => `Java Heap` 内存初始化值
- -Xmx10M => `Java Heap` 内存最大值

运行结果：

```powershell
Exception: java.lang.OutOfMemoryError thrown from the UncaughtExceptionHandler in thread "main"
Disconnected from the target VM, address: '127.0.0.1:60416', transport: 'socket'
```

 创建到 3w 多个线程的时候开始报`OOM`错误，另外两个线程池就不做测试了，测试方法一致，只是创建的线程池不一样 。

## 八、如何定义线程池参数

- **CPU密集型** => 线程池的大小推荐为 `CPU` 数量 + 1，`CPU` 数量可以根据 `Runtime.availableProcessors` 方法获取。
- **IO密集型** => `CPU` 数量 * `CPU` 利用率 * (1 + 线程等待时间/线程CPU时间)。
- **混合型** => 将任务分为 `CPU`密集型和 `IO` 密集型，然后分别使用不同的线程池去处理，从而使每个线程池可以根据各自的工作负载来调整。
- **阻塞队列** => 推荐使用有界队列，有界队列有助于避免资源耗尽的情况发生。
- 拒绝策略 => 默认采用的是 `AbortPolicy` 拒绝策略，直接在程序中抛出 `RejectedExecutionException` 异常【因为是运行时异常，不强制 `catch`】，这种处理方式不够优雅。处理拒绝策略有以下几种比较推荐： 
  - 在程序中捕获 `RejectedExecutionException` 异常，在捕获异常中对任务进行处理。针对默认拒绝策略。
  - 使用 `CallerRunsPolicy` 拒绝策略，该策略会将任务交给调用 execute 的线程执行【一般为主线程】，此时主线程将在一段时间内不能提交任何任务，从而使工作线程处理正在执行的任务。此时提交的线程将被保存在 `TCP` 队列中，TCP 队列满将会影响客户端，这是一种平缓的性能降低。
  - 自定义拒绝策略，只需要实现 `RejectedExecutionHandler` 接口即可。
  - 如果任务不是特别重要，使用 `DiscardPolicy` 和 `DiscardOldestPolicy` 拒绝策略将任务丢弃也是可以的。

如果使用 Executors 的静态方法创建 `ThreadPoolExecutor` 对象，可以通过使用 `Semaphore` 对任务的执行进行限流也可以避免出现 X`OOM` X异常。
