---
title: Elasticsearch 初探（1）——基本介绍与环境搭建
categories:
  - 搜索引擎
  - Elasticsearch
abbrlink: 513d7aa1
date: 2018-10-08 11:56:23
copyright_author: Jitwxs
---

## 一、Elasticsearch简介

### 1.1 什么是Elasticsearch

Elasticsearch是一个实时的分布式搜索和分析引擎。它可以用于**全文搜索**，**结构化搜索**以及**分析**，当然你也可以将这三者进行组合。

Elasticsearch是一个建立在全文搜索引擎 Apache Lucene™ 基础上的搜索引擎，可以说Lucene是当今最先进，最高效的全功能开源搜索引擎框架。

Elasticsearch使用Lucene作为内部引擎，但是在使用它做全文搜索时，只需要使用统一开发好的API即可，而不需要了解其背后复杂的Lucene的运行原理。

当然Elasticsearch并不仅仅是Lucene这么简单，它的主要特性包括：

- 分布式搜索
- 多租户
- 查询统计分析
- 分组和聚合

>这里给大家推荐下刘欣老师的两篇文章，通过讲故事的方法从底向上的叙述Lucene和Elasticsearch技术：
>
>[搜索之路](https://mp.weixin.qq.com/s/m31PcdCLTWuqmCUOMkhU4A)
>
>[搜索之路：Elasticsearch的诞生](https://mp.weixin.qq.com/s/mnhtYvR_5N7gtIOgjSUJmA)

### 1.2 和Solr有啥区别

提到Elasticsearch，就不能不提到Solr，下面通过几个方面来对比这两者。

（1）Elasticsearch基本是开箱即用，非常简单。而Solr安装略微复杂，可以参考文章：[《Solr 初探（1）——Solr 介绍》](/2b682855.html)

（2）Solr 利用 Zookeeper 进行分布式管理，而 Elasticsearch 自身带有分布式协调管理功能。

（3）Solr 支持更多格式的数据，比如JSON、XML、CSV，而 Elasticsearch 仅支持JSON文件格式。

（4）Solr 官方提供的功能更多，而 Elasticsearch 本身更注重于核心功能，高级功能多有第三方插件提供，例如图形化界面需要kibana友好支撑。

（5）Solr **查询快，但更新索引时慢**（即插入删除慢），用于电商等查询多的应用；ES**建立索引快（即查询慢），即实时性查询快**，用于facebook新浪等搜索。Solr 是传统搜索应用的有力解决方案，但 Elasticsearch 更适用于新兴的实时搜索应用。

（6）Solr比较成熟，有一个更大，更成熟的用户、开发和贡献者社区，而 Elasticsearch相对开发维护者较少，更新太快，学习使用成本较高。

## 二、Elasticsearch安装

和Solr一样，得益于Java的跨平台特性，所以在Windows下和在Linux下安装的步骤其实是一样的，因此方便起见，我们在Windwos下安装。

Elasticsearch 6.4.1下载链接点击这里：https://www.elastic.co/downloads/past-releases/elasticsearch-6-4-1 ，在我写这篇文章时（2018-10-8），最新版本为6.4.2，但是对应的IK分词器只更新到了6.4.1，因此本系列文章均以6.4.1为例。

### 2.1 下载安装

Elasticsearch采用Java开发，因此必须具备Java环境。

解压后进入`elasticsearch-6.4.1\bin`，然后双击`elasticsearch.bat`即可。访问`http://localhost:9200` 即可看到是否安装成功。如果显示JSON串就表示安装成功。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20181008104957405.png)

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20181008192708499.png)

默认情况下，Elasticsearch 只允许本机访问，如果需要远程访问，可以修改 Elastic 安装目录的`config/elasticsearch.yml`文件，去掉`network.host`的注释，将它的值改成`0.0.0.0`，然后重新启动 Elastic。

【注：0.0.0.0代表任何人都可以访问。线上服务不要这样设置，要设成具体的 IP。】

### 2.2 安装Head插件

#### 2.2.1 下载源码包

将该项目下载下来并解压: https://github.com/mobz/elasticsearch-head ，放在`elasticsearch-6.4.1` 同级目录即可。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20181008192301669.png)

#### 2.2.2 编译安装

（1） 修改配置文件`elasticsearch-6.4.1/config/elasticsearch.yml`，在末尾追加内容：

```yml
http.cors.enabled: true
http.cors.allow-origin: "*"
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20181008114635323.png)

（2）编译安装需要Node.js的支持，安装比较简单，因为我已经有环境了，就不在描述了，直接百度即可。Nodejs环境搭建完毕后，进入`elasticsearch-head-master`文件夹，执行命令：

```node
//注：如果安装了cnpm，就使用cnpm，否则使用默认的npm
//进入head插件目录，执行命令
npm install -g grunt-cli
//完成之后，再执行命令
npm install
//需要注意的是：如果报错，请重复执行几次，应该会成功
```

#### 2.2.3 运行

安装完成之后，执行命令：`grunt server`，显示如下图，然后就可以访问 `http://localhost:9100` 即可访问ElasticSearch的head插件了。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20181008115027471.png)

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20181008115252653.png)
