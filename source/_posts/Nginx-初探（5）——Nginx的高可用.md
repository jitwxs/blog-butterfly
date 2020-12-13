---
title: Nginx 初探（5）——Nginx的高可用
categories:
  - 服务器
  - Nginx
typora-root-url: ..
abbrlink: 55fecdc1
date: 2018-03-27 00:32:41
copyright_author: Jitwxs
---

## 一、回顾

通过前面四章的学习，学会了Nginx的安装、配置虚拟主机、反向代理、负载均衡，这基本上就是 Nginx 的大概内容了。

我们知道，nginx 其实是一个代理，客户端通过 nginx 才能够访问到后面的应用服务器（tomcat等）。那么如果 nginx 宕机，即使后面的应用服务器没有出现故障，客户端也不能正常访问了，因此保证 nginx 的高可用十分重要。

## 二、keepalived

既然 nginx 如此重要，因此我们可以准备两台 nginx 服务器，组成**nginx主/从服务器**。我们使用 `keepalived` 来帮助实现主/从服务器的切换。

`keepalived` 有一个对外提供的 `VIP(Virtual IP Address)`，它捆绑哪个 IP 地址就会使用哪一台 nginx 服务器。

 1. 当主服务器没有宕机时，VIP 指向主服务器，从服务器会不停给主服务器发送`心跳包`，主服务器会回复从服务器心跳包。
 2. 当从服务器没有收到主服务器返回的心跳包时，`keepalived` 就认为主服务器**宕机**了，它会**将 VIP 切换到从服务器**，由从服务器提供服务。
 3. 当运维人员修复好主服务器后，`keepalived`会将 VIP 切换回主服务器，从服务器不再处理请求，只给主服务器发送心跳包。

![keepalived width=70%](/images/posts/20180327002757478.png)

## 三、解决超大并发

nginx 大概能够处理 5W 的并发，如果网站的并发已经超过了nginx 的极限，我们有两种解决办法：

（1）**F5 负载均衡器**，硬件实现，价格较贵

（2）**LVS**，免费开源，软件实现，大约是F5负载均衡器 **60%** 的性能
