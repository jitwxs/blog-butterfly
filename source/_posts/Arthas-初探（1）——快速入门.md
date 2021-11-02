---
title: Arthas 初探（1）——快速入门
tags: Arthas
categories:
  - Java DevOps
  - Arthas
abbrlink: a64edcb1
date: 2020-12-26 18:11:40
---

## 一、前言

早就听闻阿里开源的 `Arthas` 在做 Java 应用诊断上十分牛逼，身边也有很多同事在使用，因此决定开一个坑，自己从零学习下这个工具的使用，本系列使用的版本是当前最新版 3.4.5。

由于 Arthas 经过这么长时间的发展，本身文档、在线教程已经十分健全了，同时还有第三方的 IDEA 插件、许多教学视频去帮助我们入门使用，因此这个系列的文章定位是个人笔记，而并非教程，希望不要误人子弟。

## 二、概述

> https://arthas.aliyun.com

当你遇到以下类似问题而束手无策时，`Arthas`可以帮助你解决：

1. 这个类从哪个 jar 包加载的？为什么会报各种类相关的 Exception？
2. 我改的代码为什么没有执行到？难道是我没 commit？分支搞错了？
3. 遇到问题无法在线上 debug，难道只能通过加日志再重新发布吗？
4. 线上遇到某个用户的数据处理有问题，但线上同样无法 debug，线下无法重现！
5. 是否有一个全局视角来查看系统的运行状况？
6. 有什么办法可以监控到JVM的实时运行状态？
7. 怎么快速定位应用的热点，生成火焰图？

使用 Arthas 需要 JDK 版本在 **1.6 以上**。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226184755.png)

## 三、快速安装

> https://arthas.aliyun.com/doc/install-detail.html

Arthas 本身也是个 Java 进程，得益于 Java 跨平台特性，所以我就直接在 Windows 上安装了。

（1）下载 Arthas 包

```bash
curl -O https://arthas.aliyun.com/arthas-boot.jar
```

（2）运行 Arthas

```bash
java -jar arthas-boot.jar
```

> 需要注意的是运行 Arthas 前至少保证系统正在运行一个 Java 进程，否则无法启动，并会报错：Can not find java process. Try to pass <pid> in command line.Please select an available pid。解决办法就是跑一个 Java 应用即可。

如果需要卸载 Arthas 的话：

- 在 Linux/Unix/Mac 平台，删除下面文件：

  ```shell
  rm -rf ~/.arthas/
  rm -rf ~/logs/arthas
  ```

- Windows平台直接删除user home下面的`.arthas`和`logs/arthas`目录

## 四、快速入门

### 4.1 attach 进程

这里我们使用 Arthas 官方提供的 demo 包，这样我们就不需要自己编写代码了。将 demo 包下载下来并运行。

```shell
curl -O https://arthas.aliyun.com/arthas-demo.jar
java -jar arthas-demo.jar
```

这个 demo 功能是死循环做质因数分解，并记录下无法分解的次数，如下图所示。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226192147.png)

我们首先启动 Arthas 并 attach 上该进程。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226192318.png)

> 默认情况下，Arthas只listen 127.0.0.1，所以如果想从远程连接，则可以使用 `--target-ip`参数指定 listen 的IP

另外如果条件允许的话，在 attach 后也可以使用浏览器登录，访问：http://127.0.0.1:3658 即可。也可以填入 IP，远程连接其他机器的 Arthas。

![20201226192716](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226192716.png)

### 4.2 常用命令

#### 4.2.1 dashboard

> https://arthas.aliyun.com/doc/dashboard.html

使用 `dastboard` 命令可以查看 Java 进程信息（定时刷新），如需退出使用 `q` 即可。它由如下四个部分组成：

1. 第一部分是显示JVM中运行的所有线程：所在线程组，优先级，线程的状态，CPU的占用率，是否是后台进程等
2. 第二部分显示的JVM内存的使用情况
3. 第三部分显示的是 GC 相关的信息
4. 第四部分是操作系统的一些信息和Java版本号

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226194318.png)

#### 4.2.2 thread

> https://arthas.aliyun.com/doc/thread.html

使用 `thread` 命令可以查看当前所有的线程信息。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226194023.png)

并且可以通过追加 PID 的方式，查看具体某个线程的状态。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226194126.png)

#### 4.2.3 jad

> https://arthas.aliyun.com/doc/jad.html

使用 `jad` 命令可以反编译 class 文件。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226194833.png)

#### 4.2.2 watch

> https://arthas.aliyun.com/doc/watch.html

`watch` 命令可以监控方法的入参出参：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226195701.png)

## 五、退出 Arthas

如果只是退出当前的连接，可以用`quit`或者`exit`命令。Attach到目标进程上的 Arthas 还会继续运行，端口会保持开放，下次连接时可以直接连接上。

如果想完全退出arthas，可以执行`stop`命令。
