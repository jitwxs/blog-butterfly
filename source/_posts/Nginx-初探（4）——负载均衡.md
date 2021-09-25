---
title: Nginx 初探（4）——负载均衡
categories: Nginx
abbrlink: 2801cb42
date: 2018-03-26 23:52:20
---

## 一、回顾

在上一章[《Nginx初探（3）——反向代理》](/a2c06ed5.html)中说到，我们可以为 nginx 配置`反向代理`，这样 nginx 就能够将客户端的请求根据域名转发给不同的应用服务器，并将应用服务器的结果返回给客户端。

## 二、负载均衡

在学习完上一章后，你也许会有疑问，一个比较大的网站怎么可能只有一台服务器呢？nginx 能够将请求分配给我这个域名下的多台服务器（服务器集群）吗？

答案是可以的，这就是 nginx 的`负载均衡`，nginx 能够为一个域名配置多台服务器，这样就能够将客户端的请求分配给每一台服务器，实现负载均衡。

nginx 的负载均衡配置十分简单，这里我就不做具体演示了，直接说如何配置了。以下代码是上一章中关于 `www.wxs.com` 的 nginx 配置：

```nginx
upstream wxs {
       server 192.168.30.149:8091;
}
server {
    listen       80;
    server_name  www.wxs.com;

    #charset koi8-r;

    #access_log  logs/host.access.log  main;

    location / {
        #root   www.wxs.com;
        proxy_pass http://wxs;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
}
```

假设 www.wxs.com 下还有一台服务器，其地址为 `192.168.250.250:8091`，则直接为其 `upstream` 节点添加一条 `server` 属性即可，其他不变，即：

```nginx
upstream wxs {
       server 192.168.30.149:8091;
       server 192.168.255.255:8091;
}
...
```

## 三、服务器权重

每台服务器的性能有所差异，我们想要 nginx 多分配一点请求给性能好的服务器，只要在对应 `server` 后面添加 `weight` 即可：

```nginx
upstream wxs {
       server 192.168.30.149:8091;
       server 192.168.255.255:8091 weight=2;
}
```

这样 `192.168.255.255:8091` 这台服务器**会多处理**一些资源。

注：**默认权限为1**。

需要注意的是，服务器之前权重是**比例关系**，因此：

```nginx
upstream wxs {
       server 192.168.30.149:8091 weight=2;
       server 192.168.255.255:8091 weight=2;
}
```

等价于：

```nginx
upstream wxs {
       server 192.168.30.149:8091;
       server 192.168.255.255:8091;
}
```
