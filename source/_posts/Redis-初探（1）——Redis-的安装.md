---
title: Redis 初探（1）——Redis 的安装
categories: 
  - 数据库
  - Redis
typora-root-url: ..
abbrlink: e331e26a
date: 2018-02-27 00:57:43
copyright_author: Jitwxs
---

### 1.1 什么是 Redis

Redis 是使用 C 语言开发的一个开源的高性能键值对（key-value）数据库。它通过提供多种键值数据类型来适应不同场景下的存储需求，Redis 支持以下五种数据类型：

- String 类型

- List 类型

- Set 类型

- SortedSet 类型

-  Hash类型

### 1.2 Redis 应用场景

- **缓存**

- 分布式集群架构中session分离

- 任务队列

- ...

### 1.3 安装 Redis

从[Redis官网](http://www.redis.cn/download.html)上下载即可，我使用的是4.0.8，[点我下载](http://download.redis.io/releases/redis-4.0.8.tar.gz)。

我将其放在了/usr/local 目录中：

```shell
wxs@ubuntu:/usr/local$ ls
bin  etc  games  include  lib  man  redis-4.0.8.tar.gz  sbin  share  src
```

解压到当前文件夹，并将其重命名为redis

```shell
wxs@ubuntu:/usr/local$ sudo tar zxvf redis-4.0.8.tar.gz
wxs@ubuntu:/usr/local$ sudo mv redis-4.0.8 redis
wxs@ubuntu:/usr/local$ cd redis/
wxs@ubuntu:/usr/local/redis$ ls
00-RELEASENOTES  COPYING  Makefile   redis.conf       runtest-sentinel  tests
BUGS             deps     MANIFESTO  runtest          sentinel.conf     utils
CONTRIBUTING     INSTALL  README.md  runtest-cluster  src
```

对其进行编译（需要先行安装 `gcc` 和 `gcc-c++`）

```shell
wxs@ubuntu:/usr/local/redis$ sudo make
```

>注：如果 make 时报`zmalloc.h:50:31: fatal error: jemalloc/jemalloc.h: No such file or directory`这个错，请使用 `make MALLOC=libc` 命令

### 1.4 运行 Redis

编译完成后进入src文件夹，其中有两个关键的文件：`redis-server` 和 `redis-cli`。

运行 `redis-server`，映入眼帘的是一张巨大的面包图：

![](/images/posts/20180227000842739.png)

其中 `Port` 表示redis的端口号，`PID` 表示改进程的 pid 号，下方光标不停的闪动，此时 redis 就已经启动了，但是这个窗口不能够使用了。

因此我们新开一个窗口，运行 `redis-cli`，顾名思义，这是 redis 的命令行模式。前面说了 redis 支持五种数据类型，我们存一个字符串来测试下：

```shell
wxs@ubuntu:/usr/local/redis/src$ ./redis-cli 
127.0.0.1:6379> set username jitwxs
OK
127.0.0.1:6379> get username
"jitwxs"
127.0.0.1:6379> 
```

成功取到了我们设置的数据！

>Tip：如果我们要连接远程Redis，可以加参数：`./redis-cli -h IP地址`
>如果要指定端口，加参数：`./redis-cli -p 端口号`

### 1.5 后端模式

但是这样的话开的第一个窗口不就是没有什么用处了吗，还得再开一个窗口，多麻烦。当然有办法，只要我们设置redis启动模式为后端启动即可。

进入 src 的上层目录，其中有一个 `redis.conf` 的文件，顾名思义，就是redis的配置文件：

```shell
wxs@ubuntu:/usr/local/redis/src$ cd ..
wxs@ubuntu:/usr/local/redis$ ls
00-RELEASENOTES  COPYING  Makefile   redis.conf       runtest-sentinel  tests
BUGS             deps     MANIFESTO  runtest          sentinel.conf     utils
CONTRIBUTING     INSTALL  README.md  runtest-cluster  src
```

编辑 `redis.conf`，将 `daemonize` 的值修改为yes，如图所示：

![](/images/posts/20180227001929294.png)

保存修改，此时重新运行 redis，注意此时运行时要加上**配置文件这个参数**（如果不加其实使用了默认的配置文件）

![](/images/posts/20180227002449161.png)

最后一行提示我们配置已经载入，而且也没有弹出之前的类似于面包图，我们查看下进程，redis的确已经在后台启动了。

```shell
wxs@ubuntu:/usr/local/redis/src$ ps auxc | grep redis
wxs       15243  0.1  0.4  51828  8380 ?        Ssl  00:24   0:00 redis-server
```

想要操纵数据依然运行 `redis-cli` 即可。

### 1.6 退出 Redis

如何退出 Redis 呢，我们可以粗暴的退出，即直接用 kill 命令将该进程杀死，这种方法十分粗暴，会导致 redis 持久化的数据丢失，不建议使用。

```shell
wxs@ubuntu:/usr/local/redis/src$ ps auxc | grep redis
wxs       15243  0.1  0.4  51828  8380 ?        Ssl  00:24   0:00 redis-server
wxs@ubuntu:/usr/local/redis/src$ sudo kill -9 15243
```

另外一种方法就是向 Redis 发送关闭命令，即 `redis-cli shutdown` 即可。

```shell
wxs@ubuntu:/usr/local/redis/src$ ps auxc | grep redis
wxs       15243  0.1  0.4  51828  8380 ?        Ssl  00:24   0:00 redis-server
wxs@ubuntu:/usr/local/redis/src$ ./redis-cli shutdown
(error) ERR Errors trying to SHUTDOWN. Check logs.
```

我这里出现了错误，是因为我的配置文件没有配置完全的原因，如果你和我一样出现了错误，那么修改配置文件：

```shell
wxs@ubuntu:/usr/local/redis/src$ sudo vim ../redis.conf
```

修改配置文件中的 `logfile` 和 `dir` 这两项为自定义路径（`dir` 项参数是一个文件夹，因此结尾必修要有/），如下图：

![](/images/posts/2018022700443054.png)

![](/images/posts/20180227004439162.png)

对于其中不存在的文件和路径，都要创建，并且赋予777权限：

```shell
wxs@ubuntu:/usr/local/redis/src$ sudo touch ../redis_log.log
wxs@ubuntu:/usr/local/redis/src$ sudo chmod 777 ../redis_log.log
wxs@ubuntu:/usr/local/redis/src$ sudo mkdir ../redis_dbfile
wxs@ubuntu:/usr/local/redis/src$ sudo chmod 777 ../redis_dbfile
```

因为配置文件没有配好，没法正常关闭，因此只能先强杀进程：

```shell
wxs@ubuntu:/usr/local/redis/src$ ps auxc | grep rediswxs       15243  0.1  0.4  51828  8380 ?        Ssl  00:24   0:01 redis-server
wxs@ubuntu:/usr/local/redis/src$ sudo kill -9 15243
```

重新启动 redis，此时就能够正常关闭了，如果还是报错误，打开你配置的 `logfile` 对应的路径，去查看具体的错误信息。

```shell
wxs@ubuntu:/usr/local/redis/src$ ./redis-server ../redis.conf 
wxs@ubuntu:/usr/local/redis/src$ ps auxc | grep redis
wxs       15450  0.0  0.4  51828  8388 ?        Ssl  00:49   0:00 redis-server
wxs@ubuntu:/usr/local/redis/src$ ./redis-cli shutdown
wxs@ubuntu:/usr/local/redis/src$ ps auxc | grep redis
wxs@ubuntu:/usr/local/redis/src$ 
```
