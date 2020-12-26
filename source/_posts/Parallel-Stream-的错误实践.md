---
title: Parallel Stream 的错误实践
categories: Java
tags:
  - Stream
  - 踩坑之路
abbrlink: 65d3da8c
date: 2020-01-19 22:27:27
references:
  - name: JAVA使用并行流(ParallelStream)时要注意的一些问题
    url: https://blog.csdn.net/xuxiaoyinliu/article/details/73040808
    rel: nofollow noopener noreferrer
    target: _blank
  - name: 记一次java8 parallelStream使用不当引发的血案
    url: https://my.oschina.net/7001/blog/1475500
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---

## 一、前言

Java8 Stream 流的出现，极大的简化了业务需求中对集合数据的加工处理操作。虽然好用，但是一旦使用不当，也会带来意想不到的结果，本文记录使用 Parallel Stream 的错误实践。

```java
List<Object> sourceList = ...;
List<Object> list = new ArrayList();

sourceList.stream.map(...).foreach(list::add);
```

伪代码如上所示，对 sourceList 进行源数据加工，加工完毕后 add 进结果 list 中。运行过程中，发现其中存在 null 元素。

## 二、实验

写一个简单 Case 测试下，如下所示：

```java
public class StreamTest {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>();
        IntStream.range(0, 50).parallel().map(e -> e * 2).forEach(list::add);
        System.out.println("size = " + list.size() + "\n" + list);
    }
}
```

多次执行，发现结果集元素个数不等于期望元素个数，且其中存在 null 元素，而且有几率出现数组下标越界错误。

```
size = 44
[30, 12, 32, 14, 34, 16, 42, 44, 46, 48, 24, 36, 20, 38, 40, null, 22, 6, 8, 10, 0, 2, 4, 56, 88, 82, 60, 84, 90, 92, 74, 94, 76, null, 50, 52, 98, 54, 62, 64, 66, 68, 70, 72]
```

```java
Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	at java.util.concurrent.ForkJoinTask.getThrowableException(ForkJoinTask.java:598)
	at java.util.concurrent.ForkJoinTask.reportException(ForkJoinTask.java:677)
	at java.util.concurrent.ForkJoinTask.invoke(ForkJoinTask.java:735)
	at java.util.stream.ForEachOps$ForEachOp.evaluateParallel(ForEachOps.java:160)
	at java.util.stream.ForEachOps$ForEachOp$OfInt.evaluateParallel(ForEachOps.java:189)
	at java.util.stream.AbstractPipeline.evaluate(AbstractPipeline.java:233)
	at java.util.stream.IntPipeline.forEach(IntPipeline.java:404)
	at jit.wxs.disruptor.stream.StreamTest.main(StreamTest.java:15)
Caused by: java.lang.ArrayIndexOutOfBoundsException: 15
	at java.util.ArrayList.add(ArrayList.java:463)
	at java.util.stream.ForEachOps$ForEachOp$OfInt.accept(ForEachOps.java:205)
	at java.util.stream.IntPipeline$3$1.accept(IntPipeline.java:233)
	at java.util.stream.Streams$RangeIntSpliterator.forEachRemaining(Streams.java:110)
	at java.util.Spliterator$OfInt.forEachRemaining(Spliterator.java:693)
	at java.util.stream.AbstractPipeline.copyInto(AbstractPipeline.java:481)
	at java.util.stream.ForEachOps$ForEachTask.compute(ForEachOps.java:291)
	at java.util.concurrent.CountedCompleter.exec(CountedCompleter.java:731)
	at java.util.concurrent.ForkJoinTask.doExec(ForkJoinTask.java:289)
	at java.util.concurrent.ForkJoinPool$WorkQueue.execLocalTasks(ForkJoinPool.java:1040)
	at java.util.concurrent.ForkJoinPool$WorkQueue.runTask(ForkJoinPool.java:1058)
	at java.util.concurrent.ForkJoinPool.runWorker(ForkJoinPool.java:1692)
	at java.util.concurrent.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:157)
```

## 三、分析

问题原因也很简单，了解过 Parallel Stream  的同学知道，其内部采用 ForkJoinPool 线程池进行执行，也就是说存在线程安全问题，而 ArrayList 是线程不安全的。下面依次来分析产生各种异常情况的原因。

### 3.1 元素数量丢失

```java
// java.util.ArrayList#add(E)
public boolean add(E e) {
  ensureCapacityInternal(size + 1);  // Increments modCount!!
  elementData[size++] = e;
  return true;
}
```

导致数组下标越界的原因是 ArrayList 的 add() 方法中的 `elementData[size++] = e`，这行代码不是原子操作，可以拆解为：

1. 读取 size 值
2. 将 e 添加到 size 的位置，即 elementData[size] = e
3. size++

这里存在内存可见性问题，当线程 A 从内存读取 size 后，设置 e 值，将 size 加 1，然后写入内存。过程中可能有线程 B 也修改了 size 并写入内存，那么**线程 A 写入内存的值就会丢失线程 B 的更新**。这解释了会出现数组长度比原始数组要小（元素丢失）的情况。

### 3.2 null 元素

null 元素产生跟元素数据丢失类似，也是由于 `elementData[size++] = e` 不是原子操作导致的。假设存在三个线程，线程 1、线程 2、线程 3。三个线程同时开始执行，初始 size 值为 1。

- 线程 1 全部执行完毕，此时 size 被更新为 2。

- 线程 2 一开始读取 size 值 = 1、将 e 添加到 size 位置后时间片就用完了，轮到执行第三步 size++ 读取到了线程 1 的更新，size 直接被更新成了 3。【注：此处线程 2 的 e 值也丢失了，被线程 1 覆盖】

- 线程3 一开始读取 size 值 = 1 后时间片就用完了，轮到执行第二步将 e 添加到 size 位置读取到了线程 2 的更新，size 变成了 3。size = 2 的位置就被跳过了，因此 elementData[2] 为 null 了。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202001/20200119214514457.png)

### 3.3 数组下标越界

数组越界异常则主要发生在数组扩容前的临界点。假设当前数组刚好只能添加一个元素，两个线程同时准备执行`ensureCapacityInternal(size + 1)`，同时读取的 size 值，加 1 后进入ensureCapacityInternal都不会导致扩容。

退出 ensureCapacityInternal 后，两个线程同时执行 `elementData[size] = e`，线程 B 的 size++ 先完成，假设此刻线程 A 读取到了线程 B 的更新，线程 A 再执行 size++，此时 size 的实际值就会大于数组的容量，这样就会发生数组越界异常。

## 四、解决

解决问题也很简单，分两种，一种是把结果集合变成线程安全的即可。

```java
List<Integer> list = new CopyOnWriteArrayList<>();
// or
List<Integer> list = Collections.synchronizedList(new ArrayList<>());
```

第二种不使用 forEach 自己 add，改用 Stream 的 collect：

```java
public class StreamTest {
    public static void main(String[] args) {
        List<Integer> list = IntStream.range(0, 50).parallel().map(e -> e * 2).boxed().collect(Collectors.toList());
        System.out.println("size = " + list.size() + "\n" + list);
    }
}
```
