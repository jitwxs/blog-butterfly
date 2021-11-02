---
title: Arthas 初探（2）——基础命令
tags: Arthas
categories:
  - Java DevOps
  - Arthas
abbrlink: 85dcbb69
date: 2020-12-26 20:00:34
---

## 一、前言

在本章节中，将学习以下 Arthas 的基础命令，同时我也会附上官方文档的链接，方便大家查阅：

- [help、cls、session、version、history、quit、stop](https://arthas.aliyun.com/doc/commands.html#arthas)
- [cat](https://arthas.aliyun.com/doc/cat.html) 显示文本文件内容
- [grep](https://arthas.aliyun.com/doc/grep.html) 对内容进行过滤，只显示关心的行
- [pwd](https://arthas.aliyun.com/doc/pwd.html) 显示当前的工作路径
- [reset](https://arthas.aliyun.com/doc/reset.html) 重置 arthas 增强的类
- [keymap](https://arthas.aliyun.com/doc/keymap.html) 显示所有的快捷键

## 二、基础命令

### 2.1 help、cls、session、version、quit、stop

（1）help

查看命令帮助信息。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226200828.png)

（2）cls

清空当前屏幕区域。

（3）session

查看当前会话的信息。

```shell
[arthas@8452]$ session
 Name        Value
--------------------------------------------------
 JAVA_PID    8452
 SESSION_ID  1434a3fe-8bbe-44ad-90d8-1a0bf2c7e461
```

（4）version

输出当前目标 Java 进程所加载的 Arthas 版本号。

```shell
[arthas@8452]$ version
3.4.5
```

（5）history

打印命令历史。

（6）quit

退出当前 Arthas 客户端，其他 Arthas 客户端不受影响。

（7）stop

关闭 Arthas 服务端，所有 Arthas 客户端全部退出。

```shell
[arthas@8452]$ stop
Resetting all enhanced classes ...
Affect(class count: 0 , method count: 0) cost in 0 ms, listenerId: 0
Arthas Server is going to shut down...
[arthas@8452]$ session (1434a3fe-8bbe-44ad-90d8-1a0bf2c7e461) is closed because server is going to shutdown.
```

### 2.2 cat

打印文件内容，类似于 Linux 中的 `cat` 命令。

```shell
[arthas@8452]$ cat c:/Users/Jitwxs/Downloads/helloworld.txt
helloworld
```

### 2.3 grep

匹配查找文件内容，类似于 Linux 中的 `grep` 命令，但它仅能用于管道命令。

| 参数列表        | 作用                                 |
| --------------- | ------------------------------------ |
| -n              | 显示行号                             |
| -i              | 忽略大小写查找                       |
| -m 行数         | 最大显示行数，要与查询字符串一起使用 |
| -e "正则表达式" | 使用正则表达式查找                   |

（1）只显示包含java字符串的行系统属性。

```shell
[arthas@8452]$ sysprop | grep java
 java.specification.version      1.8
 java.class.path                 arthas-demo.jar
 java.vm.vendor                  Oracle Corporation
 java.vendor.url                 http://java.oracle.com/
 java.vm.specification.version   1.8
 sun.java.launcher               SUN_STANDARD
 sun.java.command                arthas-demo.jar
 java.specification.vendor       Oracle Corporation
 java.home                       D:\JAVA\JRE
 java.vm.specification.vendor    Oracle Corporation
 ....
```

（2）显示包含java字符串的行和行号的系统属性。

```shell
[arthas@8452]$ sysprop | grep java -n
6: java.specification.version      1.8
9: java.class.path                 arthas-demo.jar
10: java.vm.vendor                  Oracle Corporation
13: java.vendor.url                 http://java.oracle.com/
16: java.vm.specification.version   1.8
18: sun.java.launcher               SUN_STANDARD
20: sun.java.command                arthas-demo.jar
25: java.specification.vendor       Oracle Corporation
26: java.home                       D:\JAVA\JRE
31: java.vm.specification.vendor    Oracle Corporation
```

（3）显示包含system字符串的10行信息。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226201557.png)

（4）使用正则表达式，显示包含2个o字符的线程信息。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226201641.png)

### 2.4 pwd

返回当前的工作目录，类似于 Linux 中的 `pwd` 命令。

```shell
[arthas@8452]$ pwd
C:\Users\Jitwxs\Downloads
```

### 2.5 reset

重置增强类，将被 Arthas 增强过的类全部还原，Arthas 服务端关闭时会重置所有增强过的类。

（1）还原 Test 类

```shell
reset Test
```

（2）还原所有以 List 结尾的类

```shell
reset *List
```

（3）还原所有的类

```shell
reset
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201226202102.png)

### 2.6 keymap

查看 Arthas 快捷键列表及自定义快捷键。

- 任何时候 `tab` 键，会根据当前的输入给出提示
- 命令后敲 `-` 或 `--` ，然后按 `tab` 键，可以展示出此命令具体的选项

| 快捷键说明       | 命令说明                                                |
| ---------------- | ------------------------------------------------------- |
| ctrl + a         | 跳到行首                                                |
| ctrl + e         | 跳到行尾                                                |
| ctrl + f         | 向前移动一个单词                                        |
| ctrl + b         | 向后移动一个单词                                        |
| 键盘左方向键     | 光标向前移动一个字符                                    |
| 键盘右方向键     | 光标向后移动一个字符                                    |
| 键盘下方向键     | 下翻显示下一个命令                                      |
| 键盘上方向键     | 上翻显示上一个命令                                      |
| ctrl + h         | 向后删除一个字符                                        |
| ctrl + shift + / | 向后删除一个字符                                        |
| ctrl + u         | 撤销上一个命令，相当于清空当前行                        |
| ctrl + d         | 删除当前光标所在字符                                    |
| ctrl + k         | 删除当前光标到行尾的所有字符                            |
| ctrl + i         | 自动补全，相当于敲`TAB`                                 |
| ctrl + j         | 结束当前行，相当于敲回车                                |
| ctrl + m         | 结束当前行，相当于敲回车                                |
| ctrl + c         | 终止当前命令                                            |
| ctrl + z         | 挂起当前命令，后续可以 bg/fg 重新支持此命令，或 kill 掉 |
| ctrl + a         | 回到行首                                                |
| ctrl + e         | 回到行尾                                                |
