---
title: Nginx 初探（1）——搭建环境
categories:
  - 服务器
  - Nginx
typora-root-url: ..
abbrlink: dbe6db6e
date: 2018-03-23 09:21:22
copyright_author: Jitwxs
---

## 一、安装依赖

安装环境：Ubuntu 16.04

### 1.1 g++

>apt-get install g++

### 1.2 openssl

>wget https://www.openssl.org/source/openssl-1.1.1-pre3.tar.gz
>tar zxvf openssl-1.1.1-pre3.tar.gz
>cd openssl-1.1.1-pre3/
>./config
>make
>make install

### 1.3 zlib

>wget https://sourceforge.net/projects/libpng/files/zlib/1.2.11/zlib-1.2.11.tar.gz
>tar zxvf zlib-1.2.11.tar.gz 
>cd zlib-1.2.11/
>./configure 
>make
>make install

### 1.4 pcre

>wget https://ftp.pcre.org/pub/pcre/pcre-8.42.tar.gz
>tar zxvf pcre-8.42.tar.gz
>cd pcre-8.42/
>./configure 
>make
>make install

## 二、安装 Nginx

### 2.1 下载源码并解压

>wget http://nginx.org/download/nginx-1.13.10.tar.gz
>tar zxvf nginx-1.13.10.tar.gz
>cd nginx-1.13.10

### 2.2 编译安装源码

```shell
./configure \
--prefix=/usr/local/nginx \
--pid-path=/var/run/nginx/nginx.pid \
--lock-path=/var/lock/nginx.lock \
--error-log-path=/var/log/nginx/error.log \
--http-log-path=/var/log/nginx/access.log \
--with-http_gzip_static_module \
--http-client-body-temp-path=/var/temp/nginx/client \
--http-proxy-temp-path=/var/temp/nginx/proxy \
--http-fastcgi-temp-path=/var/temp/nginx/fastcgi \
--http-uwsgi-temp-path=/var/temp/nginx/uwsgi \
--http-scgi-temp-path=/var/temp/nginx/scgi 
```

>如果要加入 SSL（例如开启HTTPS），在上面的编译命令最后追加：`--with-http_ssl_module` 

上面命令生成了Makefile文件，`--prefix` 后面是软件安装目录，后面的 `/var/log/nginx` 和 `/var/temp/nginx` 为日志文件夹和临时文件夹，无需修改。

**注意：** nginx 运行前需要 手动创建 `/var/temp/nginx` 文件夹！

编译安装：

>make
>sudo make install

执行完后 `nginx` 被安装在了 `/usr/local/nginx` 目录下：

```shell
wxs@ubuntu:/usr/local$ ls
bin  games    jdk1.8.0_161  man    redis  share  tomcat8
etc  include  lib           nginx  sbin   src    zookeeper-3.5.2-alpha
```

## 三、解决异常

### 3.1 找不到 libpcre.so.1

```shell
wxs@ubuntu:/usr/nginx/sbin$ ./nginx 
./nginx: error while loading shared libraries: libpcre.so.1: cannot open shared object file: No such file or directory
```

进入 `/usr/local/lib` 文件夹：

```shell
wxs@ubuntu:/lib$ cd /usr/local/lib/
wxs@ubuntu:/usr/local/lib$ ls
engines-1.1       libpcrecpp.so.0        libpcre.so         libz.so.1
libcrypto.a       libpcrecpp.so.0.0.1    libpcre.so.1       libz.so.1.2.11
libcrypto.so      libpcre.la             libpcre.so.1.2.10  pkgconfig
libcrypto.so.1.1  libpcreposix.a         libssl.a           python2.7
libpcre.a         libpcreposix.la        libssl.so          python3.5
libpcrecpp.a      libpcreposix.so        libssl.so.1.1
libpcrecpp.la     libpcreposix.so.0      libz.a
libpcrecpp.so     libpcreposix.so.0.0.6  libz.so
```

为 `libz.so.1` 添加软链接：

>sudo ln -s /usr/local/lib/libpcre.so.1 /lib64
>sudo ln -s /usr/local/lib/libpcre.so.1 /lib

### 3.2 不能打开日志文件  / 无法创建目录

```shell
wxs@ubuntu:/usr/local/nginx/sbin$ ./nginx 
nginx: [alert] could not open error log file: open() "/var/log/nginx/error.log" failed (13: Permission denied)
2018/03/23 09:32:35 [emerg] 24237#0: mkdir() "/var/temp/nginx/client" failed (13: Permission denied)
```

使用 `root` 权限或 `sudo` 运行即可，或者哪个目录创不了你就帮它先创了。

## 四、运行 Nginx

```shell
wxs@ubuntu:/usr/local/nginx/sbin$ sudo ./nginx 
wxs@ubuntu:/usr/local/nginx/sbin$ ps -ef | grep nginx
root      24268   1756  0 09:33 ?        00:00:00 nginx: master process ./nginx
nobody    24269  24268  0 09:33 ?        00:00:00 nginx: worker process
wxs       24271   2542  0 09:33 pts/19   00:00:00 grep --color=auto nginx
```

运行后，需要存在 `master process` 和 `worker process` 两个进程，才是正常的。

`nginx` 默认运行在 `80` 端口，打开浏览器，访问 `nginx`：

![](/images/posts/2018032309430613.png)

## 五、相关命令

|名称|命令|
|:---|:---|
|启动 Nginx|./nginx|
|关闭 Nginx|./nginx -s quit|
|刷新 Nginx（不重启实现配置文件的更新）|./nginx -s reload|
