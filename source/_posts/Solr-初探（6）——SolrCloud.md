---
title: Solr 初探（6）——SolrCloud
tags: 集群
categories:
  - Search Engine
  - Solr
abbrlink: 96601bd
date: 2018-04-12 13:46:50
---

Solr 集群，即 `SolrCloud` 是 Solr 提供的分布式搜索方案，当你需要**大规模，容错，分布式索引和检索能力**时使用 `SolrCloud`。`SolrCloud` 是基于 `Solr` 和 `Zookeeper` 的分布式搜索方案，它的主要思想是**使用 Zookeeper 作为集群的配置信息中心**。

当一个系统的索引数据量少的时候是不需要使用 SolrCloud 的，当索引量很大，搜索请求并发很高，这时需要使用 SolrCloud 来满足这些需求。它有几个特色功能：

- 集中式的配置信息
- 自动容错
- 近实时搜索
- 查询时自动负载均衡

## 一、系统架构

（1）从**物理结构**来看，一个 `SolrCloud` 可以包含多个 Solr 实例，一个 Solr 实例可以有多个 `Core`，这些组成了一个 `SolrCloud`。

（2）从**逻辑结构**来看，SolrCloud 是一个逻辑意义上的完整的索引结构，称为 `Collection`，一个 `Collection` 使用一份配置文件。

`Collection` 常常被划分为一个或多个 `Shard`（分片）。每个`Shard`被化成一个或者多个 `Replication`，通过选举确定哪个是 `Leader`，哪个是 `follower`。

一个 `Shard` 需要由一个 `Core` 或多个 `Core` 组成。由于 `Collection` 由多个`Shard`组成所以 `Collection` 一般由多个 `Core` 组成。

同一个 `Shard` 中的**每一个 `Core` 存储的内容是一致的**，一个为 `Master`，其余为 `Slave`，由此实现了高可用。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/2018041217401726.png)

## 二、搭建环境

我们要准备实现的架构图如下，没错，这次还是伪集群：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180412182204969.png)

本机IP地址：**192.168.30.155**

### 2.1 搭建 Zookeeper 集群

在 `/usr/local/` 目录下新建 `zookeeper-cluster` 文件夹，在其中搭建 Zookeeper 集群。IP 地址为：

| 名称 | 地址 |
|:-------------:| -----:|
| Zookeeper01 | 192.168.30.155:2181 |
| Zookeeper02 | 192.168.30.155:2182 |
| Zookeeper03 | 192.168.30.155:2183 |

参考文章[《Zookeper集群搭建》](/820a29b.html)，搭建完成后目录如下：

```shell
root@ubuntu:/usr/local/zookeeper-cluster# ls
manager.sh  zookeeper01  zookeeper02  zookeeper03
root@ubuntu:/usr/local/zookeeper-cluster# pwd
/usr/local/zookeeper-cluster
```

### 2.2 搭建 Tomcat 集群

（1）在 `/usr/local/` 目录下新建 `solr-cloud` 文件夹，拷贝四份 tomcat 源码到该目录下：

```shell
root@ubuntu:/usr/local/solr-cloud# cp -r /home/wxs/apache-tomcat-8.5.28/ tomcat01
root@ubuntu:/usr/local/solr-cloud# cp -r /home/wxs/apache-tomcat-8.5.28/ tomcat02
root@ubuntu:/usr/local/solr-cloud# cp -r /home/wxs/apache-tomcat-8.5.28/ tomcat03
root@ubuntu:/usr/local/solr-cloud# cp -r /home/wxs/apache-tomcat-8.5.28/ tomcat04
root@ubuntu:/usr/local/solr-cloud# ls
tomcat01  tomcat02  tomcat03  tomcat04
```

（2）修改每个 tomcat 的 `conf/erver.xml` 文件，要修改 3 个地方的端口信息，分别是：

- http 访问端口（默认为 8080 端口）：

```xml
<Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
```

- 监听 tomcat 关闭的端口（默认为 8005）：

```xml
<Server port="8005" shutdown="SHUTDOWN">
	<Listener className="org.apache.catalina.startup.VersionLoggerListener" />
```

- 负责接收其他 http 服务器的请求端口（默认为 8009）：

```xml
<Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
```

监听关闭和其他 http 服务器请求端口无所谓，只要别重复就行。http 访问端口如下：

| 名称 | 地址 |
|:-------------:| -----|
| tomcat01 | 192.168.30.155:8181 |
| tomcat02 | 192.168.30.155:8182 |
| tomcat03 | 192.168.30.155:8183 |
| tomcat04 | 192.168.30.155:8184 |

（3）修改 `/etc/profile` 文件，在尾部添加环境变量：

```bash
####集群Tomcat####
export CATALINA_01_BASE=/usr/local/solr-cloud/tomcat01
export CATALINA_01_HOME=/usr/local/solr-cloud/tomcat01
export TOMCAT_01_HOME=/usr/local/solr-cloud/tomcat01

export CATALINA_02_BASE=/usr/local/solr-cloud/tomcat02
export CATALINA_02_HOME=/usr/local/solr-cloud/tomcat02
export TOMCAT_02_HOME=/usr/local/solr-cloud/tomcat02

export CATALINA_03_BASE=/usr/local/solr-cloud/tomcat03
export CATALINA_03_HOME=/usr/local/solr-cloud/tomcat03
export TOMCAT_03_HOME=/usr/local/solr-cloud/tomcat03

export CATALINA_04_BASE=/usr/local/solr-cloud/tomcat04
export CATALINA_04_HOME=/usr/local/solr-cloud/tomcat04
export TOMCAT_04_HOME=/usr/local/solr-cloud/tomcat04
```

修改每个tomcat下的 `bin/catalina.sh` 文件，找到下面代码：

```bash
# OS specific support.  $var _must_ be set to either true or false.
```

以 tomcat01 为例，在上述代码下面添加以下代码，其他 tomcat 修改成相应的：

```bash
# myself : add
export CATALINA_BASE=$CATALINA_01_BASE
export CATALINA_HOME=$CATALINA_01_HOME
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180412185524913.png)

### 2.3 部署 Solr

关于 Solr 的部署参考文章[《Solr 初探（1）——Solr 介绍》](/2b682855.html)，虽然这篇是 windows 版的，但是 Linux 版也是完全一样。

如果你按照上面的文章成功部署了Solr单机版，那么你应该拥有：

- 一个放在 tomcat 中可以运行的 solr

- 一个 solr 的 home 文件夹，其中拥有一个 Core，名为 core1

（1）将单机版的 `solr` 拷贝到每个tomcat的 `webapp` 下：

```shell
root@ubuntu:/usr/local/solr-cloud# cp -r ../tomcat8/webapps/solr/ tomcat01/webapps/
root@ubuntu:/usr/local/solr-cloud# cp -r ../tomcat8/webapps/solr/ tomcat02/webapps/
root@ubuntu:/usr/local/solr-cloud# cp -r ../tomcat8/webapps/solr/ tomcat03/webapps/
root@ubuntu:/usr/local/solr-cloud# cp -r ../tomcat8/webapps/solr/ tomcat04/webapps/
```

（2）将单机版的 solr 的 `home` 文件夹拷贝到 solr-cloud 中，拷贝四份：

```shell
root@ubuntu:/usr/local/solr-cloud# cp -r ../solr_home/ solrhome01
root@ubuntu:/usr/local/solr-cloud# cp -r ../solr_home/ solrhome02
root@ubuntu:/usr/local/solr-cloud# cp -r ../solr_home/ solrhome03
root@ubuntu:/usr/local/solr-cloud# cp -r ../solr_home/ solrhome04
```

此时 solr-cloud 下结构：

```shell
root@ubuntu:/usr/local/solr-cloud# ls
solrhome01  solrhome03  tomcat01  tomcat03
solrhome02  solrhome04  tomcat02  tomcat04
```

（3）修改每个 solr 的 `web.xml` 中的 home 路径

以 tomcat01 为例，其他的以此类推：

```shell
root@ubuntu:/usr/local/solr-cloud# vim tomcat01/webapps/solr/WEB-INF/web.xml
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180412190654766.png)

（4）修改每个 solrhome 中的 `solr.xml` 文件中的 `host` 和 `hostPort`

以 solrhome01 为例，其他的以此类推：

```shell
root@ubuntu:/usr/local/solr-cloud# vim solrhome01/solr.xml
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180412191109912.png)

### 2.4 关联 Solr 和 Zookeeper

再次修改每个 tomcat 的 `bin/catalina.sh` 文件，修改 `JAVA_OPTS` 为 zookeeper 集群地址：

```bash
JAVA_OPTS="-DzkHost=192.168.30.155:2181,192.168.30.155:2182,192.168.30.155:2183"
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180412191850585.png)

### 2.5 将 Solr 配置文件上传至 Zookeeper

前面说过，**一个 solrCloud 只使用一份配置文件**，但是现在四个 solr 每个 `solr` 对应一个`solrhome`，每一个 `solrhome` 的每一个 `core` 都有一份配置文件，所有我们需要任意拿一份配置文件，上传到 zookeeper 中，来让 solrCloud 使用。使用**solr 源码包**的 `zkcli.sh` 文件来帮助上传。

进入 solr 源码包的 `solr-6.6.2/server/scripts/cloud-scripts/` 路径

```shell
root@ubuntu:/home/wxs# cd solr-6.6.2/server/scripts/cloud-scripts/
root@ubuntu:/home/wxs/solr-6.6.2/server/scripts/cloud-scripts# ls
log4j.properties  snapshotscli.sh  zkcli.bat  zkcli.sh
```

执行如下命令：

```shell
./zkcli.sh -zkhost 192.168.30.155:2181,192.168.30.155:2182,192.168.30.155:2183 -cmd upconfig -confdir /usr/local/solr-cloud/solrhome01/core1/conf -confname myconf
```

该命令包含 zookeepr 集群地址，指定任意一个 core 的配置文件，上传到 zookeeper 的 myconf 目录。

至此完成上传，我们可以验证一下是否上传成功，启动 zookeeper 集群后，进入任意一台 zookeeper，以 zookeeper01 为例：


```shell
root@ubuntu:/usr/local# cd zookeeper-cluster/zookeeper01/bin
root@ubuntu:/usr/local/zookeeper-cluster/zookeeper01/bin# ./zkCli.sh -server 192.168.30.155:2182
```

```shell
[zk: 192.168.30.155:2182(CONNECTED) 1] ls /
[configs, zookeeper]
[zk: 192.168.30.155:2182(CONNECTED) 2] ls con
config    connect   
[zk: 192.168.30.155:2182(CONNECTED) 2] ls /configs
[myconf]
[zk: 192.168.30.155:2182(CONNECTED) 3] ls /configs/myconf
[_rest_managed.json, _schema_analysis_stopwords_english.json, _schema_analysis_synonyms_english.json, admin-extra.html, admin-extra.menu-bottom.html, admin-extra.menu-top.html, clustering, currency.xml, elevate.xml, lang, managed-schema, mapping-FoldToASCII.txt, mapping-ISOLatin1Accent.txt, params.json, protwords.txt, solrconfig.xml, spellings.txt, stopwords.txt, synonyms.txt, update-script.js, velocity, xslt]
```

### 2.6 验证

编写 tomcat 集群启动脚本：

```bash
#!/bin/bash
#@#@description: tomcat cluster manager script

if [ $# -eq 0 ]
then
    echo "参数错误"
    exit 1
fi

cmd=""
case $1 in
    "start")
	cmd="./startup.sh"
	;;
    "stop")
	cmd="./shutdown.sh"
	;;
    *)
	echo "命令非法"
	exit 1
	;;
esac

cd tomcat01/bin
$cmd
cd ../..
cd tomcat02/bin
$cmd
cd ../..
cd tomcat03/bin
$cmd
cd ../..
cd tomcat04/bin
$cmd
cd ../..
```

启动 zookeeper 集群

```shell
root@ubuntu:/usr/local/zookeeper-cluster# ./manager.sh start
```

然后启动 tomcat 集群

```shell
root@ubuntu:/usr/local/solr-cloud# ./tomcat.sh start
```

访问任何一个 solr，选择 cloud 标签，可以看见我们的集群已经显示出来了：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180412232158932.png)

## 三、分片

### 3.1 新建分片

一开始说过 `Collection` 常常被划分为一个或多个 `Shard`（分片）。现在图上的 collection 只有一个 `Shard`，这样的话一个为主，三个为从，太浪费了。

我们按照需求将其分成两片，一主一备。

在浏览器上输入：

```shell
http://192.168.30.155:8181/solr/admin/collections?action=CREATE&name=collection1&numShards=2&replicationFactor=2
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180412232819106.png)

回车后结果如下：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180412233010361.png)

重新返回Solr首页，就能看见我们创建的 `Collection` 了：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180412233106597.png)

### 3.2 删除分片

`SolrCloud` 理论上可以存在无数个 `Collection`，我们可以删除掉无用的 Collection，以 `core1` 为例：

执行命令：

```shell
http://192.168.30.155:8181/solr/admin/collections?action=DELETE&name=core1
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180412233318310.png)

运行结果如下：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180412233357962.png)

重新返回 Solr 首页，发现 `core1` 已经被删除了：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180412233430865.png)

## 四、Solrj

### 4.1 CloudSolrClient

在[《Solr 初探（5）——Solrj》](/46ffa9b2.html)中我们使用 Solrj 操纵单机版 Solr，简单回顾下：

```java
// 指定url
String baseURL = "http://localhost:8080/solr/core1";

// 创建Solr单机版客户端
HttpSolrClient solrClient = new HttpSolrClient(baseURL);

// 操纵Document...
```

`SolrClient` 这个抽象类除了有 `HttpSolrClient` 这个实现类，还有 `CloudSolrClient`，这个就是用来操纵集群版的：

```java
// 指定Zookeeper集群地址列表
String zkHost = "192.168.30.155:2181,192.168.30.155:2182,192.168.30.155:2183";
// 创建Solr集群版客户端
CloudSolrClient solrClient = new CloudSolrClient(zkHost);
// 设置默Solr集群的默认Collection
solrClient.setDefaultCollection("collection1");

// 操纵Document...
```

因为 `SolrCloud` 是通过 `Zookeeper` 管理的，所以需要指定 zookeeper 的地址列表，创建 SolrClient 对象后还需要指定默认的 `collection`，即上面图中的 collection1。

### 4.2 Spring 配置

因为 HttpSolrClient 和 CloudSolrClient 都是 SolrClient 的实现类，因此使用配置文件就能灵活的切换了。

先在配置文件中加入：

```ini
#Solr单机
solr.standalone.url=http://192.168.30.155:8180/solr/core1
#Solr集群
solr.cluster.host=192.168.30.155:2181,192.168.30.155:2182,192.168.30.155:2183
solr.cluster.collection=collection1
```

Spring 的配置如下：

>注：单机版和集群版同时只能开放一个，另一个必须注释掉

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans.xsd
	http://www.springframework.org/schema/context
	http://www.springframework.org/schema/context/spring-context.xsd">

    <!-- 加载配置文件 -->
    <context:property-placeholder location="classpath:cfg.properties"/>

    <!-- 单机版Solr -->
    <bean id="httpSolrClient" class="org.apache.solr.client.solrj.impl.HttpSolrClient">
        <constructor-arg name="baseURL" value="${solr.standalone.url}"/>
    </bean>

    <!-- 集群版Solr -->
    <bean id="cloudSolrClient" class="org.apache.solr.client.solrj.impl.CloudSolrClient">
        <constructor-arg name="zkHost" value="${solr.cluster.host}"/>
        <property name="defaultCollection" value="${solr.cluster.collection}"/>
    </bean>
</beans>
```
