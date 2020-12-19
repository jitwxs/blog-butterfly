---
title: Linux 配置多个 Tomcat
categories: 服务器
tags: Tomcat
abbrlink: 13892bf0
date: 2018-03-26 01:26:05
copyright_author: Jitwxs
---

我的系统里面原本就有一份 Tomcat ，名称为 `tomcat8` ：

```linux
wxs@ubuntu:/usr/local$ ls
bin  games    jdk1.8.0_161  man    redis  share  tomcat8
etc  include  lib           nginx  sbin   src    zookeeper-3.5.2-alpha
```

>注：本篇文章只使用两台 Tomcat ，配置更多个同理。

复制两份，分别为 `tomcat-jit` 和 `tomcat-wxs`  ：

```linux
wxs@ubuntu:/usr/local$ sudo cp -r tomcat8/ tomcat8-jit
wxs@ubuntu:/usr/local$ sudo cp -r tomcat8/ tomcat8-wxs
wxs@ubuntu:/usr/local$ ls
bin    include       man    sbin   tomcat8      zookeeper-3.5.2-alpha
etc    jdk1.8.0_161  nginx  share  tomcat8-jit
games  lib           redis  src    tomcat8-wxs
```

## 一、修改端口

配置多个Tomcat需要修改 **3个** 地方的端口信息，分别是：

1. **http访问端口**（默认为8080端口）：

```xml
<Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
```

2. **监听tomcat关闭的端口**（默认为8005）：

```xml
<Server port="8005" shutdown="SHUTDOWN">
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
```

3. 负责**接收其他http服务器的请求端口**（默认为8009）：

```xml
<Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
```

下面修改这两个tomcat的端口信息：

（1）对于 tomcat8-jit：

```linux
wxs@ubuntu:/usr/local$ sudo vim tomcat8-jit/conf/server.xml 
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180325235811354.png)

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180325235819267.png)

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180325235826575.png)

（2）对于 tomcat8-wxs：

```xml
wxs@ubuntu:/usr/local$ sudo vim tomcat8-wxs/conf/server.xml
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180326000010454.png)

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180326000017770.png)

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180326000024487.png)

## 二、修改启动文件

如果只修改了那三个端口信息，**启动某一个，另外一个无法启动**，因为**默认只使用了同一个环境变量**，需要为每一个tomcat配置环境变量。

修改 `/etc/profile` 文件，在尾部添加环境变量：

```bash
####第一个Tomcat####
export CATALINA_JIT_BASE=/usr/local/tomcat8-jit
export CATALINA_JIT_HOME=/usr/local/tomcat8-jit
export TOMCAT_JIT_HOME=/usr/local/tomcat8-jit

####第二个Tomcat####
export CATALINA_WXS_BASE=/usr/local/tomcat8-wxs
export CATALINA_WXS_HOME=/usr/local/tomcat8-wxs
export TOMCAT_WXS_HOME=/usr/local/tomcat8-wxs
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180326011722230.png)

修改每个 tomcat 下的 `bin/catalina.sh` 文件，找到下面代码：

```bash
# OS specific support.  $var _must_ be set to either true or false.
```

在下面添加对应的 `CATALINA_BASE` 和 `CATALINA_HOME`：

（1）对于 tomcat8-jit：

```bash
# myself : add
export CATALINA_BASE=$CATALINA_JIT_BASE
export CATALINA_HOME=$CATALINA_JIT_HOME
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/2018032601213778.png)

（2）对于 tomcat8-wxs：

```bash
# myself : add
export CATALINA_BASE=$CATALINA_WXS_BASE
export CATALINA_HOME=$CATALINA_WXS_HOME
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180326012148923.png)

## 三、测试

修改两个 Tomcat 首页以便测试区分：

```linux
wxs@ubuntu:/usr/local$ sudo vim tomcat8-jit/webapps/ROOT/index.jsp 
wxs@ubuntu:/usr/local$ sudo vim tomcat8-wxs/webapps/ROOT/index.jsp 
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180326003422116.png)

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180326000348638.png)

我的服务器IP是 `192.168.30.149` ，分别访问 `192.168.30.149:8090` 和 `192.168.30.149:8091` :

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201803260122316.png)
