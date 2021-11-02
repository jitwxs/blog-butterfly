---
title: 详解 JDK 常用监控工具
tags: JVM
categories:
  - Java DevOps
  - Performance Analysis
abbrlink: c72fb74
date: 2020-12-26 14:23:54
---

## 一、JVM 参数

JVM 参数类型主要分为标准参数、X 参数、XX 参数三类。

（1）对于标准参数，在 JVM 的各个版本中基本是不变的，相对是比较稳定的。例如 `-help`、`-version`、`-server`、`-client` 等。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226151819.png)

（2）X 参数是非标准化参数，在不同的 JVM 版本中可能会发生变化。例如：

- `-Xint`：完全解释执行，不编译成本地代码
- `-Xcomp`：第一次使用就编译成本地代码
- `-Xmixed`：混合模式，JVM自行决定是否编译成本地代码（默认模式）

（3）XX 参数是我们使用最为经常的参数，它也是非标准化参数，主要用于 JVM 调优和 Debug。它分为 Boolean 类型和键值对类型。

```
Boolean 类型
格式：-XX:[+-]<name> 表示启用或禁用 name 属性
例如：-XX:+UseConcMarkSweepGC
```

```
键值对类型
格式：-XX:<name>=<value> 表示 name 属性的值是 value
例如：-XX:MaxGCPauseMillis=500
```

各大 JVM 的相关参数可以到这个网站去查询：https://chriswhocodes.com/hotspot_options_jdk8.html

## 二、JPS

`JPS` 类似于 Linux 系统中的 PS 命令，只是它是专门用来查看 Java 进程的 PID，[点击这里](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jps.html)查看官方文档。

使用 `jps -l` 可以获取 Java 进程的包名，例如我启动了一个 SpringBoot 项目，使用该命令后可以拿到它的 PID 是 3007。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226152703.png)

## 三、JINFO

`JINFO` 命令可以查看 Java 进程的 JVM 参数，[点击这里](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jinfo.html)查看官方文档。

```
jinfo -flag <name> <pid> 查看进程的某一个 JVM 参数
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226152856.png)

```
jinfo -flags <pid> 查看进程所有非默认的 JVM 参数
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226153147.png)

## 四、JSTAT

`jstat` 可以查看 JVM 的统计信息（类加载、垃圾收集、JIT 编译等），[点击这里](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html)查看官方文档。

```
格式：jstat -class <pid> <interval> <count> 每隔 Interval 毫秒查看一次进程的类加载信息，查看 count 次
例如：jstat -class 3007 1000 5
```

![查看类加载信息](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226153636.png)

```
格式：jstat -gc <pid> <interval> <count> 每隔 Interval 毫秒查看一次进程的 GC 信息，查看 count 次
例如：jstat -gc 3007 1000 2
```

![查看GC(1.8)信息](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226153918.png)

- S0C、S1C、S0U、S1U：S0 和 S1 的总量与使用量
- EC、EU：Eden 区总量与使用量
- OC、OU：Old 区总量与使用量
- MC、MU：Metaspace 区总量与使用量
- CCSC、CCSU：压缩类空间总量与使用量
- YGC、YGCT：YoungGC 的次数与时间
- FGC、FGCT：FullGC 的次数与时间
- GCT：总的 GC 时间

> 除了列出的 -gc 外，还有 -gccapactly、-gccause、-gcnew、-gcold 等

```
格式：jstat -compiler <pid> 查看进程 JIT 编译信息
例如：jstat -compiler 3007
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226155912.png)

- Compiled、Failed、Invalid 将代码编译成本地方法的成功、失败、无效任务量
- FailedType、FailedMethod 上次编译失败的类型和方法名

## 五、JMAP

`jmap` 命令可以帮助我们获取 Java 进程的内存快照，并可以将其导入到 [MAT](https://www.eclipse.org/mat/downloads.php)（免费）或 [JProfile](https://www.ej-technologies.com/products/jprofiler/overview.html)（付费）等工具中去做内存分析。这边的内容我在 [《首次排查 OOM 实录》](/f4adeb1d.html) 这篇文章中提到了，就不再赘述了。

## 六、JSTACK

`jstack` 命令可以帮助我们获取 Java 进程中的线程快照，[点击这里](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstack.html)查看官方文档。

进程状态一般有以下几种（https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr034.html）：

| Thread State  | Description                            |
| ------------- | -------------------------------------- |
| NEW           | 线程未启动                             |
| RUNNABLE      | 线程已在 JVM 中运行                    |
| BLOCKED       | 线程处于阻塞状态等待锁                 |
| WAITING       | 线程无限期地等待另一个线程执行特定操作 |
| TIMED_WAITING | WAITING 状态加上超时时间               |
| TERMINATED    | 线程已经退出                           |

例如当我们发现 CPU 的利用率飙高，就需要使用 jstack 查看下线程的状态了，一般的操作流程是这样的：

（1）`jps -l` 获取到需要分析的 Java 进程的 PID。【本例中为 3007】

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226152703.png)

（2）`top -Hp <PID>` 查看该 Java 进程内部每个线程的 CPU 和占用情况，找到你觉得有问题的那一个线程的 PID，转成十六进制记录下来。【我这个程序只是个 helloword，所以我就随便找一个线程了。比如取 3018，它的十六进制是 0xbca】

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226164440.png)

（3）`jstack <PID> <filename>` 获取到线程快照。【本例中输出为 a.out】

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226164611.png)

（4）然后在其中搜索你觉得有问题的那个进程 ID。【例子举得不好，找的线程是 GC 线程，知道是这么个流程就行】

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226164652.png)

