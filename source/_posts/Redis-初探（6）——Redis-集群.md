---
title: Redis 初探（6）——Redis 集群
tags: 集群
categories: 
  - Database
  - Redis
abbrlink: bcdf2003
date: 2018-04-08 23:59:45
---

之前我们所学习的都是 Redis 的单机版，我们知道 Redis 之所以读取速度快是因为它是**存储在内存**中的。内存的容量是有限的，单台 Redis 会碰到性能瓶颈，这就需要使用 `Redis集群（Redis-cluster）`。

## 一、集群原理

### 1.1 集群架构

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180408215525657.png)

如上图所示，每一个蓝色圆圈就是一个 Redis 节点，这些节点组成了一个 `Redis集群（Redis-cluster）`。节点之间使用 `Ping——Pong` 机制进行互联，其内部用二进制协议优化传输速度和带宽。

Redis客户端和节点**直接连接**即可，无需中间件，一台客户端连接一个节点即可，Client 访问时直接访问任意一个Redis节点即可。

### 1.2 负载均衡

当我们搭建了集群后，其是如何实现负载均衡的呢？

Redis 集群中内置了 **16384** 个`哈希槽（slot）`，它会把所有的物理节点都映射到**[0,16383]**的slot上。

当我们需要在Redis集群中放置了个 key-value 时，Redis 先对 key 使用 `crc16` 算法得出一个结果，然后将结果**对16384取余**，这样**每一个 key 都会对应一个编号在0 ~16383的 哈希槽**。

Redis 会根据节点数量大致均等的将哈希槽映射到不同的节点。例如我们有三个节点，其每个节点哈希槽范围为：0 ~ 5000，5001 ~ 10000，10001~ 16383。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/2018040822112780.png)

当我们存储一个名为 Hello1 的 key 时，其通过 `crc16` 算法计算出的结果为1500，Redis 集群就会将这个 key 放在对应1500的`哈希槽`中，又因为哈希槽0~ 5000被映射到了 Server1，则 Hello1 就被放置在了 Server1。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180408221454944.png)

### 1.3 容错机制

通过上面的负载均衡的原理，知道其实**每一台** Redis 客户端保存的**内容都是不一样**的，那么当Redis集群中**任意一个节点挂掉**（连接失败）时，**整个集群就挂了**。

为了实现 Redis 的高可用，需要**为每一个节点添加备用机**，即`主备机制`。

一般集群都有集群管理工具，但是 Redis 集群没有，它是通过`投票机制`来实现的。

以下图为例简单说一下投票机制：

 1. 当黄色节点发现无法 ping 通红色节点，它就觉得红色节点可能挂掉了，于是它会**发起投票**。
 2. 其他节点会尝试去 ping 红色节点，只要有**超过半数**的节点无法 ping 通红色节点，就**判定红色节点挂掉**（即使它实际上可能并没有挂掉）。
 3. 当红色节点被判定挂掉后，会**启动该节点的备用机**。如果该节点不存在备用机，或备用机也挂掉，那么**整个Redis集群就挂掉了**。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180408222553762.png)

## 二、搭建集群

Redis 集群为了实现容错机制，**最少需要三个节点**（一个出错，另外两个投票），又因为**每个节点至少要有一台备份机**，因此一个**最简单的 Redis 集群需要6个 Redis 客户端**。

这里只是演示一下，使用`伪集群`，即6台 Redis 搭建在一台 Linux 上，仅使用端口号进行区分，设端口号范围为7001 ~ 7006。

### 2.1 准备原始 Redis

> 注：[《Redis 初探（1）——Redis 的安装》](/e331e26a.html)这篇文章中源码和安装后的文件是在同一文件夹下，本篇文章不使用这种方法。

准备一个 Redis 安装包，将源码解压到当前文件夹：

```shell
wxs@ubuntu:~$ ls
Desktop  redis-4.0.8.tar.gz
wxs@ubuntu:~$ tar zxvf redis-4.0.8.tar.gz 
```

进入解压后文件夹，执行 make 编译：

```shell
wxs@ubuntu:~$ cd redis-4.0.8/
wxs@ubuntu:~/redis-4.0.8$ make
```

>注：所有源码安装需要自行编译的都需要 gcc、g++ 等相关软件支持

将其安装到`/usr/local/redis`目录下：

```shell
wxs@ubuntu:~/redis-4.0.8$ sudo make install PREFIX=/usr/local/redis
```

该目录下只有一个 bin 目录，存放 redis 的可执行文件，我们从源码包中拷贝一份 redis 配置文件过来：

```shell
wxs@ubuntu:/usr/local/redis$ ls
bin
wxs@ubuntu:/usr/local/redis$ sudo cp ~/redis-4.0.8/redis.conf .
wxs@ubuntu:/usr/local/redis$ ls
bin  redis.conf
```

编辑该配置文件：

（1）修改 bind 端口号为 `0.0.0.0`，使其支持远程访问。
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180408230408179.png)

（2）开启后端模式。
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180408225213142.png)

（3）设置日志文件位置
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/2018040822550312.png)

（4）开启 AOF 持久化
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180408225644836.png)

### 2.2 准备 Redis 集群客户端

**注意：**用来做集群的客户端**必须是干净的客户端**，像备份文件 `dump.rdb`、`appendonly.aof` 等应当先删除掉，避免不必要的错误。

在 `/usr/local` 中创建 `redis-cluster` 文件夹，拷贝六份原始 Redis：

```shell
wxs@ubuntu:/usr/local/redis-cluster$ sudo cp -r ../redis redis01
wxs@ubuntu:/usr/local/redis-cluster$ sudo cp -r ../redis redis02
wxs@ubuntu:/usr/local/redis-cluster$ sudo cp -r ../redis redis03
wxs@ubuntu:/usr/local/redis-cluster$ sudo cp -r ../redis redis04
wxs@ubuntu:/usr/local/redis-cluster$ sudo cp -r ../redis redis05
wxs@ubuntu:/usr/local/redis-cluster$ sudo cp -r ../redis redis06
wxs@ubuntu:/usr/local/redis-cluster$ ls
redis01  redis02  redis03  redis04  redis05  redis06
```

以 redis01 为例，编辑其 `redis.conf` 文件：

（1）修改端口号为 7001
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180408230544887.png)

（2）修改 pidfile
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180408230649718.png)

（3）开启集群开关
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180408232742829.png)

依次修改其他客户端，端口号范围从7001 ~ 7006。

编写一个启动这些客户端的批处理 `startup.sh`：

```bash
cd redis01/bin
sudo ./redis-server ../redis.conf
cd ../../
cd redis02/bin
sudo ./redis-server ../redis.conf
cd ../../
cd redis03/bin
sudo ./redis-server ../redis.conf
cd ../../
cd redis04/bin
sudo ./redis-server ../redis.conf
cd ../../
cd redis05/bin
sudo ./redis-server ../redis.conf
cd ../../
cd redis06/bin
sudo ./redis-server ../redis.conf
cd ../../
```

执行脚本，启动成功：
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180408233355513.png)

编写一个关闭这些客户端的批处理 `shutdown.sh`：

```bash
cd redis01/bin
sudo ./redis-cli -p 7001 shutdown
cd ../../
cd redis02/bin
sudo ./redis-cli -p 7002 shutdown
cd ../../
cd redis03/bin
sudo ./redis-cli -p 7003 shutdown
cd ../../
cd redis04/bin
sudo ./redis-cli -p 7004 shutdown
cd ../../
cd redis05/bin
sudo ./redis-cli -p 7005 shutdown
cd ../../
cd redis06/bin
sudo ./redis-cli -p 7006 shutdown
cd ../../
```

执行脚本，关闭成功：
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/2018040823342414.png)

目录结构如下：

```shell
wxs@ubuntu:/usr/local/redis-cluster$ ls
redis01  redis02  redis03  redis04  redis05  redis06  shutdown.sh  startup.sh
```

### 2.3 搭建集群

从 redis 源码的 src 目录中拷贝 `redis-trib.rb` 过来：

```shell
wxs@ubuntu:/usr/local/redis-cluster$ sudo cp ~/redis-4.0.8/src/redis-trib.rb .
wxs@ubuntu:/usr/local/redis-cluster$ ls
redis01  redis03  redis05  redis-trib.rb  startup.sh
redis02  redis04  redis06  shutdown.sh
```

因为这是一个 `shell` 文件，需要安装 shell 和 shell 包管理器：

```shell
wxs@ubuntu:/usr/local/redis-cluster$ sudo apt-get install shell shellgems
```

安装 shell 关于 Redis 的库,可以执行 `gem install redis` 或者[前往官网](https://shellgems.org/gems/redis/versions/)下载安装包后安装。

```shell
wxs@ubuntu:/usr/local/redis-cluster$ sudo gem install redis
Fetching: redis-4.0.1.gem (100%)
Successfully installed redis-4.0.1
Parsing documentation for redis-4.0.1
Installing ri documentation for redis-4.0.1
Done installing documentation for redis after 0 seconds
1 gem installed
```

执行 `redis-trib.rb` 时需要附带参数，格式如下：
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180408235335619.png)

先启动所有客户端，然后执行脚本：

```shell
wxs@ubuntu:/usr/local/redis-cluster$ ./startup.sh 
wxs@ubuntu:/usr/local/redis-cluster$ ./redis-trib.rb create --replicas 1 192.168.30.155:7001 192.168.30.155:7002 192.168.30.155:7003 192.168.30.155:7004 192.168.30.155:7005  192.168.30.155:7006
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180409001036743.png)

创建成功：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180408235826112.png)

最后说两句：

 1. 如果你不是伪集群，真的在服务器上搭建了集群，即 `真集群`，那么只需要在任意一台上执行 `redis-trib.rb` 即可。
 2. 在真集群的情况下，除了在配置文件中要 `bind 0.0.0.0` 以外，还要注意关闭防火墙，不然无法搭建。
 3. 如果关闭所有 redis 客户端后，想要重新开启集群，在客户端都启动后，进入任意客户端执行`cluster nodes`即可。
 4. 如果想要重新生成集群，需要删除每个 Redis 中生成的持久化文件。

## 三、使用 redis-cli 连接集群

使用任意一个 `redis-cli`（以redis01的为例），使用 `-h` 指定ip地址（默认连接127.0.0.1），使用 `-p` 参数指定端口号（默认连接原始的端口为6379的 redis），使用 `-c` 参数指定是集群（不加会导致无法将 key 放入对应的客户端中）。

```shell
wxs@ubuntu:/usr/local/redis-cluster$ ./redis01/bin/redis-cli -h 192.168.30.155 -p 7001 -c
```

当我在7001中添加了一个 `name jitwxs` 后，它计算出 key 的哈希槽为5798，这个范围在7002中，因此这个 key 被移动到了7002中，并且当前连接重定向到了7002。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180409002715610.png)

```shell
192.168.30.155:7002> keys *
1) "name"
192.168.30.155:7001> keys *
(empty list or set)
```
