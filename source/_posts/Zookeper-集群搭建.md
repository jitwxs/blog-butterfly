---
title: Zookeper 集群搭建
tags: Zookeeper
categories: 中间件
abbrlink: 820a29b
date: 2018-04-12 17:08:26
copyright_author: Jitwxs
---

在[《Dubbo + Zookeeper入门初探》](/5def036a.html)这篇文章中，我们已经搭建了 `Zookeeper` 单机版，本篇文章将介绍如何搭建 `Zookeeper` 集群。

## 一、投票选举机制

Zookeeper 集群在工作时，只有一个节点为 `leader`（主节点），其余节点均为 `follower`（从节点）。这是通过内部的投票选举机制来实现的。

我们知道，投票至少要过半数才行，因此集群中节点数最好为单数，因此我们搭建一个最简单的 3 个节点的 Zookeeper 集群。

## 二、搭建集群

因为我没有弄三台Linux，所以本篇文章采用伪集群，均搭建在一台服务器上，采用端口号进行区分。首先列出相关信息，方便大家理解：

本机IP地址：**192.168.30.155**

Zookeeper01 ：`/usr/local/zookeeper-cluster/zookeeper01`

| 名称| 端口号|
| ------------- |:-------------:| 
| 客户端端口 | 2181 |
| 节点通信端口 | 2881 |
| 节点选举端口 | 3881 |

Zookeeper02：`/usr/local/zookeeper-cluster/zookeeper02`

| 名称| 端口号|
| ------------- |:-------------:| 
| 客户端端口 | 2182 |
| 节点通信端口 | 2882 |
| 节点选举端口 | 3882 |

Zookeeper03：`/usr/local/zookeeper-cluster/zookeeper03`

| 名称| 端口号|
| ------------- |:-------------:| 
| 客户端端口 | 2183 |
| 节点通信端口 | 2883 |
| 节点选举端口 | 3883 |

>注：因为我是伪集群，所以只能使用端口号进行区分，**如果你是真集群，无需修改端口号**，通过IP地址区分即可

### 2.1 初始化文件夹

zookeeper 版本为 `zookeeper-3.5.2-alpha`，将压缩包放入 Linux 中，解压缩：

```shell
root@ubuntu:~$ tar zxvf zookeeper-3.5.2-alpha.tar.gz 
```

创建集群文件夹 `zookeeper-cluster`：

```shell
root@ubuntu:~$ mkdir /usr/local/zookeeper-cluster
```

拷贝 zookeeper 三份到 `zookeeper-cluster` 中：

```shell
root@ubuntu:~$ cp -r zookeeper-3.5.2-alpha /usr/local/zookeeper-cluster/zookeeper01
root@ubuntu:~$ cp -r zookeeper-3.5.2-alpha /usr/local/zookeeper-cluster/zookeeper02
root@ubuntu:~$ cp -r zookeeper-3.5.2-alpha /usr/local/zookeeper-cluster/zookeeper03
```

进入到 `zookeeper-cluster` 中：

```shell
root@ubuntu:/usr/local$ cd zookeeper-cluster/
root@ubuntu:/usr/local/zookeeper-cluster$ ls
zookeeper01  zookeeper02  zookeeper03
```

### 2.2 修改配置信息

以 zookeeper01 为例：

（1）进 入`zookeeper01` 文件夹，创建 `data` 文件夹，在其中新建一个 `myid` 文件，内容为 `1`。

>注：myid 文件中存放的是该 zookeeper 存在于集群中的 id 号，一直递增即可。

```shell
root@ubuntu:/usr/local/zookeeper-cluster$ cd zookeeper01/
root@ubuntu:/usr/local/zookeeper-cluster/zookeeper01$ mkdir data
root@ubuntu:/usr/local/zookeeper-cluster/zookeeper01$ cd data/
root@ubuntu:/usr/local/zookeeper-cluster/zookeeper01/data# echo 1 > myid
root@ubuntu:/usr/local/zookeeper-cluster/zookeeper01/data# cat myid 
1
```

（2）进入 conf 文件夹，拷贝一份配置文件为 `zoo.cfg`：

```shell
root@ubuntu:/usr/local/zookeeper-cluster/zookeeper01/data# cd ../conf/
root@ubuntu:/usr/local/zookeeper-cluster/zookeeper01/conf# cp zoo_sample.cfg zoo.cfg
```

编辑 `zoo.cfg`，分别修改 dataDir 路径、clientPort（真集群忽略）、集群节点信息，如图所示：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180412162219680.png)

另外两个的配置和这个相似，我就不再详细列举了（规定下 `zookeeper02` 的 `myid` 内容为 `2`，`zookeeper03` 的 `myid` 内容为 `3`）。

### 2.3 编写批处理

在 `zookeeper-cluster` 目录下编写集群的批处理：

```bash
#!/bin/bash
#@copyright_author: jitwxs
#@description: zookeeper cluster manager script

if [ $# -eq 0 ]
then
    echo "参数错误"
    exit 1
fi

arg=$1
flag=false
params=("start" "stop" "status" "restart")

for i in ${params[*]}
do
    if [ ${i} = ${arg} ]
	then
	    flag=true
	    break
	fi
done

cd zookeeper01/bin/
./zkServer.sh ${arg}
cd ../../
cd zookeeper02/bin/
./zkServer.sh ${arg}
cd ../../
cd zookeeper03/bin/
./zkServer.sh ${arg}
cd ../../
```

添加执行权限：

```shell
root@ubuntu:/usr/local/zookeeper-cluster# ls
manager.sh  zookeeper01  zookeeper02  zookeeper03
root@ubuntu:/usr/local/zookeeper-cluster# chmod u+x manager.sh 
```

### 2.4 启动集群

启动集群，查看状态：

```shell
root@ubuntu:/usr/local/zookeeper-cluster# ./manager.sh start
root@ubuntu:/usr/local/zookeeper-cluster# ./manager.sh status
```

分别查看每个集群状态：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180412170424612.png)

可以看见 zookeeper02 为 `leader`，另外两个为 `follower`。
