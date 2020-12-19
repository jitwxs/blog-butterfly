---
title: Nginx 初探（2）——配置虚拟主机
tags: 虚拟主机
categories:
  - 服务器
  - Nginx
abbrlink: 133d8a57
date: 2018-03-23 15:37:13
copyright_author: Jitwxs
---

## 一、引入

我们知道，要想在一台服务器上配置多个网站，服务器有两种方法进行区分，**一种是通过端口号，一种是通过域名**。

若一台主机`192.168.30.145`上面部署了两个网站，一个是 www.jitwxs.cn ，一个是 www.baidu.com 。

如果使用端口号区分，若想访问 www.jitwxs.cn 使用`192.168.30.145:80`，访问 www.baidu.com 使用`192.168.30.145:81`。这是不友好的，因为 http 默认端口为 80，用户如果忘输了网站的端口，不就进了别人的网站吗，用户体验极差。

如果使用域名进行区分，都只用访问 `192.168.30.145`，nginx 内部判断用户输入的域名是 www.jitwxs.cn 还是 www.baidu.com ，无需用户关心，体验好。

## 二、使用端口号配置虚拟主机

>说明：我的 nginx 安装目录为 /usr/nginx

打开 `/usr/nginx/conf/nginx.conf` 文件，我们关心的是其中的 `server` 节点。

一个 `server` 节点就是**一台虚拟主机**，`location /` 子节点中的 `root` 就是指该**虚拟主机的文件夹**。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/2018032311094779.png)

因此我们想要使用端口号新建虚拟主机，需要：

 1. **拷贝一份Server节点**
 2. **修改端口号**
 3. **修改文件夹目录**

为了方便演示，删除注释部分的代码，拷贝一份，修改端口号为81，修改文件夹目录为html_81：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180323111404117.png)

保存配置文件，将 nginx 目录下的 html 文件夹复制一份保存为 html_81，并对其首页进行修改，以便测试时候能够区分：

```shell
wxs@ubuntu:/usr/nginx$ sudo cp -r html/ html_81
wxs@ubuntu:/usr/nginx$ ls
conf  html  html_81  sbin
wxs@ubuntu:/usr/nginx$ sudo vim html_81/index.html 
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180323111922599.png)

分别访问 80 和 81 端口进行测试：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180323111950943.png)

## 三、使用域名配置虚拟主机

我们知道：**一个域名对应一个I P，一个  IP 可以对应多个域名**。从域名到IP需要 `DNS` 进行解析，如果本地 host 中已经有了域名和 IP 的对应关系，那么 DNS 服务器就不回去解析。

下面让我们演示一下！

若我们当前服务器IP为 `192.168.30.147`，我们为 `www.jiw.com` 和 `www.wxs.com` 这两个网站进行手动添加 host，均指向 `192.168.30.147`。

修改 `C:\Windows\System32\drivers\etc\host` 文件，手动进行域名、IP映射。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180323152059342.png)

>注：如果没有权限修改，参考文章：[Win10解决修改host没有权限问题（其他文件同理）](https://blog.csdn.net/yuanlaijike/article/details/79668711)

服务器启动 Nginx：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180323152144683.png)

现在这两个网站都访问了 Nginx，Nginx 怎么才能区分他们呢？当然是通过**域名**！修改配置文件：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180323152935288.png)

下面创建这两个网站的文件夹：

```shell
wxs@ubuntu:/usr/nginx$ sudo cp -r html/ www.jit.com
wxs@ubuntu:/usr/nginx$ sudo cp -r html/ www.wxs.com
wxs@ubuntu:/usr/nginx$ ls
conf  html  sbin  www.jit.com  www.wxs.com
```
为了区分，将两个文件的首页打印各自的域名信息：

```shell
wxs@ubuntu:/usr/nginx$ sudo vim www.jit.com/index.html 
```
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180323153246837.png)

```
wxs@ubuntu:/usr/nginx$ sudo vim www.wxs.com/index.html 
```
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180323153258510.png)

重新启动 Nginx：

```shell
wxs@ubuntu:/usr/nginx$ sudo sbin/nginx -s quit
wxs@ubuntu:/usr/nginx$ sudo sbin/nginx
```

访问域名：
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180323153524612.png)

完成！

>注：这里我是要模拟域名才手动修改host，如果你拥有真实域名和IP，使用DNS域名解析即可。
