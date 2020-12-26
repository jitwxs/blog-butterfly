---
title: Arthas 初探（3）——JVM 命令
copyright_author: Jitwxs
tags: Arthas
categories:
  - Java
  - 性能分析
abbrlink: e88e6355
date: 2020-12-26 20:28:17
---

## 一、前言

在本章节中，将学习以下 Arthas 的 JVM 命令，同时我也会附上官方文档的链接，方便大家查阅：

- [dashboard](https://arthas.aliyun.com/doc/dashboard.html)
- [thread](https://arthas.aliyun.com/doc/thread.html)
- [jvm](https://arthas.aliyun.com/doc/jvm.html)
- [sysprop](https://arthas.aliyun.com/doc/sysprop.html)
- [sysenv](https://arthas.aliyun.com/doc/sysenv.html)
- [vmoption](https://arthas.aliyun.com/doc/vmoption.html)
- [getstatic](https://arthas.aliyun.com/doc/getstatic.html)
- [ognl](https://arthas.aliyun.com/doc/ognl.html)

## 二、dashboard

显示当前系统的实时数据面板，按 `q` 或 `ctrl+c` 退出。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201226194318.png)

### 2.1 数据说明

| 列名        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| ID          | Java 级别的线程ID，注意这个 ID 不能跟 jstack 中的 nativeID 一一对应 |
| NAME        | 线程名                                                       |
| GROUP       | 线程组名                                                     |
| PRIORITY    | 线程优先级, 1~10之间的数字，越大表示优先级越高               |
| STATE       | 线程的状态                                                   |
| %CPU        | 线程消耗的 CPU 占比，采样 100ms，将所有线程在这 100ms 内的 CPU 使用量求和，再算出每个线程的 CPU 使用占比。 |
| DELTA_TIME  | 上次采样之后线程运行增量CPU时间，数据格式为`秒`              |
| TIME        | 线程运行总时间，数据格式为`分：秒`                           |
| INTERRUPTED | 线程当前的中断位状态                                         |
| DAEMON      | 是否是守护线程                                               |

### 2.2 内部线程

在使用 `dastboard` 或者 `thread` 命令时，会看到有些线程的 ID 为 -1。这是在 Java 8 之后支持观测 JVM 的内部线程。

这些内部线程只有名称和 CPU 时间，没有 ID 及状态等信息。 通过内部线程可以观测到 JVM 活动，如 GC、JIT 编译等占用 CPU 情况，方便了解 JVM 整体运行状况。

JVM内部线程包括下面几种：

- JIT 编译线程：`C1 CompilerThread0`, `C2 CompilerThread0` 等
- GC 线程：`GC Thread0`, `G1 Young RemSet Sampling` 等
- 其它内部线程：`VM Periodic Task Thread`, `VM Thread`, `Service Thread` 等

当 JVM 堆(heap)/元数据(metaspace)空间不足或 OOM 时，可以看到 GC 线程的 CPU 占用率明显高于其他的线程。

当执行 `trace/watch/tt/redefine` 等命令后，可以看到 JIT 线程活动变得更频繁。因为 JVM 热更新 class 字节码时清除了此 class 相关的 JIT 编译结果，需要重新编译。

## 三、thread

| 参数名称      | 参数说明                                                |
| ------------- | ------------------------------------------------------- |
| *id*          | 线程 ID                                                 |
| [n:]          | 指定最忙的前 N 个线程并打印堆栈                         |
| [b]           | 找出当前阻塞其他线程的线程                              |
| [i `<value>`] | 指定 CPU 使用率统计的采样间隔，单位为毫秒，默认值为 200 |
| [--all]       | 显示所有匹配的线程                                      |

### 3.1 CPU 使用率

这里的 CPU 使用率与 Linux 命令 `top -H -p <pid>` 的线程 `%CPU` 类似，一段采样间隔时间内，当前 JVM 里各个线程的增量 CPU 时间与采样间隔时间的比例。

工作原理如下：

（1）首先第一次采样，获取所有线程的CPU时间（调用的是`java.lang.management.ThreadMXBean#getThreadCpuTime()`及`sun.management.HotspotThreadMBean.getInternalThreadCpuTimes()`接口）

（2）然后睡眠等待一个间隔时间（默认为 200ms，可以通过 `-i` 指定间隔时间）

（3）再次第二次采样，获取所有线程的 CPU 时间，对比两次采样数据，计算出每个线程的增量CPU 时间

（4）计算得到 CPU 使用率

> 线程 CPU 使用率 = 线程增量 CPU 时间 / 采样间隔时间 * 100%

注意： 这个统计也会产生一定的开销（JDK 这个接口本身开销比较大），因此会看到 as 的线程占用一定的百分比，为了降低统计自身的开销带来的影响，可以把采样间隔拉长一些，比如 5000 毫秒。

### 3.2 展示当前最忙的 N 个线程

```shell
thread -n 3
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201226205037.png)

- 如果没有进程 ID，且包含`[Internal]`表示为 JVM 内部线程，和 dashboard 命令中相同。
- `cpuUsage` 为采样间隔时间内线程的 CPU 使用率，和 dashboard 命令中相同。

- `deltaTime`为采样间隔时间内线程的增量 CPU 时间，小于 1ms 时被取整显示为 0ms。
- `time` 线程运行总CPU时间。

### 3.3 显示第一页线程

默认按照 CPU 增量时间降序排列，只显示第一页数据。

```shell
thread
```

### 3.4 显示所有线程

有时需要获取全部JVM的线程数据进行分析，可以一次全部展示。

```shell
thread -all
```

### 3.5 显示单个线程运行堆栈

```shell
thread <pid>
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201226205725.png)

### 3.6 找出当前阻塞其他线程的线程

有时候我们发现应用卡住了， 通常是由于某个线程拿住了某个锁， 并且其他线程都在等待这把锁造成的（即死锁），使用该命令可以一键找出。

```
thread -b
```

> 注意：3.4.5 版本目前只支持找出synchronized关键字阻塞住的线程， 如果是`java.util.concurrent.Lock`， 目前还不支持。

### 3.7 指定采样间隔

- `thread -i 1000` : 统计最近1000ms 内的线程 CPU 时间。
- `thread -n 3 -i 1000` : 列出1000ms 内最忙的 3 个线程栈

### 3.8 指定状态所有线程

```shell
thread –state <state>
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201226210136.png)

## 四、JVM

查看当前 JVM 的信息

#### 4.1 Thread 相关

| 字段           | 含义                                |
| -------------- | ----------------------------------- |
| COUNT          | JVM 当前活跃的线程数                |
| DAEMON-COUNT   | JVM 当前活跃的守护线程数            |
| PEAK-COUNT     | 从 JVM 启动开始曾经活着的最大线程数 |
| STARTED-COUNT  | 从 JVM 启动开始总共启动过的线程次数 |
| DEADLOCK-COUNT | JVM 当前死锁的线程数                |

#### 4.2 文件描述符相关

| 字段                       | 含义                               |
| -------------------------- | ---------------------------------- |
| MAX-FILE-DESCRIPTOR-COUNT  | JVM 进程最大可以打开的文件描述符数 |
| OPEN-FILE-DESCRIPTOR-COUNT | JVM 当前打开的文件描述符数         |

## 五、sysprop

即 System Property，查看和修改 JVM 的系统属性。

（1）查看所有属性

```shell
sysprop
```

（2）查看单个属性（支持自动补全）

```shell
sysprop <key>
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201226211222.png)

（3）修改单个属性

```shell
sysprop <key> <value>
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201226211538.png)

## 六、sysenv

即 System Environment Variables，查看当前 JVM 的环境属性。

（1）查看所有环境变量

```shell
sysenv
```

（2）查看单个环境变量（支持自动补全）

```shell
sysenv <key>
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201226211128.png)

## 七、vmoption

查看、更新 VM 诊断相关的参数。

（1）查看所有选项

```shell
vmoption
```

（2）查看单个选项（支持自动补全）

```shell
vmoption <key>
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201226211401.png)

3）修改单个选项

```shell
vmoption <key> <value>
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201226211422.png)

## 八、getstatic

> 推荐直接使用 ognl 命令，更加灵活

通过 getstatic 命令可以方便的查看类的静态属性。

```shell
getstatic <class_name> <field_name>
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201226211746.png)

## 九、ognl

自 3.0.5 起，Arthas 支持执行 OGNL 表达式。OGNL 的语法需要我们额外学习，[点击这里](http://commons.apache.org/proper/commons-ognl/language-guide.html)查看详细文档。

| 参数名称              | 参数说明                                                     |
| --------------------- | ------------------------------------------------------------ |
| *express*             | 执行的表达式                                                 |
| `[c:]`                | 执行表达式的 ClassLoader 的 hashcode，默认值是SystemClassLoader |
| `[classLoaderClass:]` | 指定执行表达式的 ClassLoader 的 class name                   |
| [x]                   | 结果对象的展开层次，默认值1                                  |

（1）调用静态函数

```shell
ognl '@java.lang.System@out.println("hello")'
```

（2）调用静态函数

```shell
ognl '@demo.MathGame@random'
```

（3）执行多行表达式，赋值给临时变量，返回一个List

```shell
ognl '#value1=@System@getProperty("java.home"), #value2=@System@getProperty("java.runtime.name"), {#value1, #value2}'
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201226212342.png)