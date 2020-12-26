---
title: 解决 Xshell 关闭 vim 后内容仍停留在屏幕的问题
categories: 开发工具
tags: Xshell
toc: false
abbrlink: 29f74e3a
date: 2017-12-23 00:55:55
copyright_author: Jitwxs
---

**问题描述：**

使用Xshell远程连接终端后，当关闭vim时，内容仍然停留在屏幕上。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201712/20171223005116998.png)

**解决问题：**

1.编辑.bashrc文件：`vim ~/.bashrc`

在最后添加一行：`export TERM=xterm`

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201712/20171223005331413.png)

2.重新登陆终端：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201712/20171223005502170.png)
