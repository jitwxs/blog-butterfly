---
title: Redis 初探（4）——Redis 的特性
categories: 
  - 数据库
  - Redis
abbrlink: 125289ae
date: 2018-03-03 20:30:41
copyright_author: Jitwxs
---

## 一、多数据库

每一个 Redis 实例可以包括多个数据库，客户端可以指定连接某个 Redis 实例的某个数据库。一个Redis实例最多可以提供 **16** 个数据库，下标从 0 到 15，客户端**默认连接第 0 号数据库**。

| 含义 | 方法 |
|:------------- |:------------- |
| 选择第 n 号数据库 | select n |
| 将当前库的 key 转移到第 n 号数据库 | move key n |

```shell
wxs@ubuntu:/usr/local/redis/src$ ./redis-cli #默认进入0号数据库
127.0.0.1:6379> keys *
1) "myList"
2) "mySet3"
3) "mySet1"
4) "myListB"
5) "userName"
6) "mySet2"
7) "int_num"
8) "float_num"
127.0.0.1:6379> move userName 1 #将userName移到1号数据库
(integer) 1
127.0.0.1:6379> exists userName #userName已经不存在于0号数据库
(integer) 0
127.0.0.1:6379> select 1 #进入1号数据库
OK
127.0.0.1:6379[1]> exists userName #userName存在于1号数据库
(integer) 1
```

这里补充下之前没有提到的 Redis 服务器的一些命令：

| 含义 | 方法 |
|:------------- |:------------- |
| 测试连接状态是否可用 | ping |
| 命令行中打印内容 | echo xxx |
| 返回当前数据库中 key 的数目 | dbsize |
| 获取服务器的信息和统计 | info |
| 删除当前选择数据库中所有key | flushdb |
| 删除所有数据库中所有key | flushall |

```shell
wxs@ubuntu:/usr/local/redis/src$ ./redis-cli
127.0.0.1:6379> select 1 #进入1号数据库
OK
127.0.0.1:6379[1]> ping #检查连接状态
PONG
127.0.0.1:6379[1]> dbsize #获取当前数据库key数目
(integer) 1
127.0.0.1:6379[1]> flushdb #清空当前数据库
OK
127.0.0.1:6379[1]> keys *
(empty list or set)
127.0.0.1:6379[1]> flushall #清空所有数据库
OK
127.0.0.1:6379[1]> select 0
OK
127.0.0.1:6379> keys *
(empty list or set)
```

## 二、消息订阅与发布

Redis 发布订阅(pub/sub)是一种消息通信模式：发送者(pub)发送消息，订阅者(sub)接收消息。Redis 客户端可以订阅**任意数量**的频道。

| 含义 | 方法 |
|:------------- |:------------- |
| 订阅频道 | subscribe cctv |
| 批量订阅频道 | psubscribe cctv* |
| 在指定频道中发送消息 | publish cctv "hello" |

实现消息的订阅和发布至少需要两个终端，我们在终端1中订阅 cctv 这个频道的消息：

```shell
127.0.0.1:6379> subscribe cctv
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "cctv"
3) (integer) 1
```

在终端2中向 cctv 这个频道发布消息：

```shell
wxs@ubuntu:/usr/local/redis/src$ ./redis-cli 
127.0.0.1:6379> publish cctv "send Message to cctv..."
(integer) 1
127.0.0.1:6379> 
```
此时在终端1中就接收到了消息：
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201803/20180303201616787.png)

我们也可以以一种正则的形式订阅多个频道，比如 cctv 下面有多个频道，例如 cctv-1、cctv-2 等等，我们只需订阅这个即可：

>psubscribe cctv*

## 三、事务

Redis 和其他数据库一样，也支持事务功能，但是 Redis 的数据库**并不是一种真正的事务**，它其实**更像是一种批处理**。

传统数据库意义上的事务，是将多条 SQL 语句作为一个整体，如果其中任何一条语句执行失败，那么这些语句都不会被执行。但是Redis的事务，**如果有某一条命令执行失败，其后的命令仍然会执行**。

| 含义 | 方法 |
|:------------- |:------------- |
| 开启事务(类似：begin transaction) | multi |
| 提交事务（类似：commit）| exec |
| 事务回滚（类似：rollback）| discard |

```shell
wxs@ubuntu:/usr/local/redis/src$ ./redis-cli 
127.0.0.1:6379> set num 1
OK
127.0.0.1:6379> set name jitwxs
OK
127.0.0.1:6379> multi #开启事务
OK
127.0.0.1:6379> incr num
QUEUED
127.0.0.1:6379> incr name
QUEUED
127.0.0.1:6379> set age 20
QUEUED
127.0.0.1:6379> exec #执行事务
1) (integer) 2
2) (error) ERR value is not an integer or out of range #命令报错
3) OK #前面命令出错后这条命令仍然执行了
127.0.0.1:6379> get num
"2"
127.0.0.1:6379> get name
"jitwxs"
127.0.0.1:6379> get age #该命令的确执行了
"20"
127.0.0.1:6379> 

```
