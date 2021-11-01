---
title: SpringBoot 远程调试
categories:
  - Java Web
  - SpringBoot
tags:
  - IDEA
  - 调试
abbrlink: '8e004828'
date: 2019-05-25 00:07:21
references:
  - name: 如何使用 Idea 远程调试 Java 代码
    url: https://www.liangzl.com/get-article-detail-969.html
  - name: Java应用程序是否会因-Xdebug的存在而放慢速度，或者仅在逐步完成代码时才放慢速度？
    url: https://cloud.tencent.com/developer/ask/55182
  - name: Java远程调试各参数说明
    url: https://blog.csdn.net/chenpeng19910926/article/details/82529116
  - name: How to Remotely Debug Application Running on Tomcat From Within Intellij IDEA
    url: https://blog.trifork.com/2014/07/14/how-to-remotely-debug-application-running-on-tomcat-from-within-intellij-idea/
  - name: -X Command-line Options
    url: https://docs.oracle.com/cd/E13150_01/jrockit_jvm/jrockit/jrdocs/refman/optionX.html
---

在配合 QA 进行代码测试，以及处理线上 BUG 时，代码往往已经被部署于服务器端，因此服务器端程序支持远程调试功能就尤为重要。

Java 原生支持调试功能，由于实际开发中使用 SpringBoot，因此本文探讨基于 `jar` 包的调试，远程调试的 IDE 为 ` IDEA`。

**注：** war 包调试、Eclipse 远程调试功能请另行了解，这不在本文的探讨范围内。

## 一、调试命令

最为常见的远程调试命令，也是我正在使用的调试命令是：

```shell
java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=6001 -jar xxx.jar
```

当然更多的你也可能见到这种：

```shell
java -Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=6001 -jar xxx.jar
```

### 1.1 基础概念

`JPDA`(Java Platform Debugger Architecture)，即 Java 平台调试体系，具体结构图如下图所示。

![JPDA](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201905/20190521114410.png)

其中实现调试功能的主要协议是 `JDWP`协议，在 Java SE 5 以前版本，JVM 端的实现接口是 `JVMPI`(Java Virtual Machine Profiler Interface)，而在 Java SE 5 及以后版本，使用 `JVMTI`(Java Virtual Machine Tool Interface) 来替代 JVMPI。

因此，如果您使用 Java SE 5 之前版本，使用调试功能的命令为：

```shell
java -Xdebug -Xrunjdwp:...
```

而 Java SE 5 及之后版本，使用调试功能的命令为：

```shell
java -agentlib:jdwp=...
```

### 1.2 参数说明

**(1) transport**

指定运行的被调试应用和调试者之间的通信协议，它由几个可选值：

- `dt_socket`：主要的方式，采用 socket 方式连接
- `dt_shmem`：采用共享内存方式连接，仅支持 Windows 平台（暂未验证）

**(2) server**

当前应用作为调试服务端还是客户端，默认为 `n`。

如果你想将当前应用作为被调试应用，设置该值为 `y`；如果你想将当前应用作为客户端，作为调试的发起者，设置该值为 `n`。

**(3) suspend**

当前应用启动后，是否阻塞应用直到被连接，默认值为 `y`。

在大部分的应用场景，这个值为 `n`，即不需要应用阻塞等待连接。一个可能为 `y` 的应用场景是，你的程序在启动时出现了一个故障，为了调试，必须等到调试方连接上来后程序再启动。

**(3) address**

暴露的调试连接端口，默认值为 `8000`。

**(4) onthrow**

当程序抛出设定异常时，中断调试。

**(5) onuncaught**

当程序抛出未捕获异常时，是否中断调试，默认值为 `n`。

**(6) launch**

当调试中断时，执行的程序。

**(7) timeout**

该参数限定为 `java -agentlib:jdwp=…` 可用，单位为毫秒ms。

当 suspend = y 时，该值表示等待连接的超时；当 suspend = n 时，该值表示连接后的使用超时。

### 1.3 参考实例

- `-agentlib:jdwp=transport=dt_socket,server=y,address=8000`

  以 Socket 方式监听 8000 端口，程序启动阻塞（suspend的默认值为y）直到被连接。

- `-agentlib:jdwp=transport=dt_socket,server=y,address=localhost:8000,timeout=5000`

  以 Socket 方式监听 8000 端口，当程序启动后5秒无调试者连接的话终止，程序启动阻塞（suspend的默认值为y）直到被连接。

- `-agentlib:jdwp=transport=dt_shmem,server=y,suspend=n`

  选择可用的共享内存连接地址并使用 stdout 打印，程序启动不阻塞。

- `-agentlib:jdwp=transport=dt_socket,address=myhost:8000`

  以 socket 方式连接到 myhost:8000上的调试程序，在连接成功前启动阻塞。

- `-agentlib:jdwp=transport=dt_socket,server=y,address=8000,onthrow=java.io.IOException,launch=/usr/local/bin/debugstub`

  以 Socket 方式监听 8000 端口，程序启动阻塞（suspend的默认值为y）直到被连接。当抛出 `IOException` 时中断调试，转而执行 `usr/local/bin/debugstub`程序。

## 二、IDEA 远程调试

首先启动好应用程序，我这里就直接使用最通用的命令：

```shell
java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=6001 -jar express.jar
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201905/20190521114429.png)

然后在 IDEA 中，点击 `Edit Configurations`，在弹框中点击 `+` 号，然后选择 `Remote`。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201905/20190521114514.png)

填写服务端的 IP 地址，以及调试端口号。在检查下下方的 `Command line arguments for remote JVM` 是否和服务端启动是配置的一致。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201905/20190521114700.png)

配置完毕后点击保存即可，因为我配置的 `suspend=n`，因此服务端程序无需阻塞等待我们的连接。我们点击 IDEA 调试按钮，当我访问某一接口时，能够正常调试。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201905/20190521114818.png)
