---
title: Nginx 初探（3）——反向代理
tags: 反向代理
categories:
  - 服务器
  - Nginx
abbrlink: a2c06ed5
date: 2018-03-26 01:07:10
copyright_author: Jitwxs
---

## 一、什么是反向代理

### 1.1 正向代理

举一个通俗的例子，因为众所周知的原因，我们无法访问谷歌，但是因为某些原因，我们必须要访问谷歌，这时候我们会买一个“梯子”，既然我们无法直接访问谷歌，我们就去麻烦“梯子”帮助我们访问。

事实上我们还是没法访问谷歌，只是这个“梯子”能够访问，它只是将访问结果返回给我们而已。这里的“梯子”就是一个`正向代理`，它是帮助客户端也就是我们用户来代理的。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180325232601579.png)

### 1.2 反向代理

举个例子，你的手机号码出了点毛病，你要去找 10086 解决问题，但是最后接线员是谁你并不能确定，接线员是系统分配的，系统看哪位接线员有空就将你的电话转到他那边。这里的系统其实就是一个反向代理，大家都访问 10086，但是接线员每个人都不一样。

回到程序的世界中，如果 www.baidu.com 这个域名下的网站放在好几个服务器上（组建集群），如图所示：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180325232610110.png)

用户访问 www.baidu.com 这个域名，但是具体访问哪一台服务器不需要用户关心，nginx 会帮助我们将我们的请求转发(forward)到某一台服务器上，然后将请求返回给用户。

### 1.3 总结

总结一下：

- 正向代理为**客户端**服务，**对服务端是透明的**

- 反向代理为**服务端**服务，**对客户端是透明的**

注：透明指不用关心对端的具体实现。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20190106212208828.png)

## 二、配置反向代理

上面说过 nginx 将请求转发给（应用）服务器，这里我们选择 `tomcat` 作为应用服务器。

### 2.1 准备 Tomcat

我的系统里面原本就有一份 tomcat：

```shell
wxs@ubuntu:/usr/local$ ls
bin  games    jdk1.8.0_161  man    redis  share  tomcat8
etc  include  lib           nginx  sbin   src    zookeeper-3.5.2-alpha
```

为了做实验，我复制两份，分别放置我的 `www.jit.com` 和 `www.wxs.com`：

这两个 tomcat 的具体配置省略，请移步：[《Linux 配置多个 Tomcat》](/13892bf0.html)。

分别修改两个 tomcat 首页以便区分：

```shell
wxs@ubuntu:/usr/local$ sudo vim tomcat8-jit/webapps/ROOT/index.jsp 
wxs@ubuntu:/usr/local$ sudo vim tomcat8-wxs/webapps/ROOT/index.jsp 
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180326003422116.png)

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180326000348638.png)

分别启动两个 tomcat：

```shell 
root@ubuntu:/usr/local# tomcat8-jit/bin/startup.sh 
Using CATALINA_BASE:   /usr/local/tomcat8-jit
Using CATALINA_HOME:   /usr/local/tomcat8-jit
Using CATALINA_TMPDIR: /usr/local/tomcat8-jit/temp
Using JRE_HOME:        /usr/local/jdk1.8.0_161/jre
Using CLASSPATH:       /usr/local/tomcat8-jit/bin/bootstrap.jar:/usr/local/tomcat8-jit/bin/tomcat-juli.jar
Tomcat started.
root@ubuntu:/usr/local# tomcat8-wxs/bin/startup.sh 
Using CATALINA_BASE:   /usr/local/tomcat8-wxs
Using CATALINA_HOME:   /usr/local/tomcat8-wxs
Using CATALINA_TMPDIR: /usr/local/tomcat8-wxs/temp
Using JRE_HOME:        /usr/local/jdk1.8.0_161/jre
Using CLASSPATH:       /usr/local/tomcat8-wxs/bin/bootstrap.jar:/usr/local/tomcat8-wxs/bin/tomcat-juli.jar
Tomcat started.
```

### 2.2 配置 Nginx

 1. 注释掉 `location` 节点中的 root 子节点
 2. 在 location 子节点中新增 `proxy_pass` 节点，值为要转发的url名字（任意起一个）
 3. 新增一个 `upstream` 节点，名字为刚刚设置的 url 名字
 4. 在 upstream 中新增 `server` 子节点，值为目标服务器 IP
 
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180326010048132.png)

另一个同理，完整配置如下：

```nginx
upstream jit {
    server 192.168.30.149:8090;
}
server {
    listen       80;
    server_name  www.jit.com;

    #charset koi8-r;

    #access_log  logs/host.access.log  main;

    location / {
        #root   www.jit.com;
        proxy_pass http://jit;
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

配置完成后，启动 nginx 服务器，访问 `www.jit.com` 和 `www.wxs.com`：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180326010451739.png)

成功！Nginx 成功帮我们把请求转发给了不同的服务器！
