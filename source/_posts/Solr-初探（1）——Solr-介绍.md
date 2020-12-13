---
title: Solr 初探（1）——Solr 介绍
categories:
  - 搜索引擎
  - Solr
typora-root-url: ..
abbrlink: 2b682855
date: 2018-03-07 00:30:48
copyright_author: Jitwxs
---

##  一、Solr 简介

### 1.1 Solr 简介

`Solr` 是 Apache 下的一个顶级开源项目，采用Java开发，它是**基于Lucene的全文搜索服务器**。Solr 提供了比`Lucene`更为丰富的查询语言，同时实现了可配置、可扩展，并对索引、搜索性能进行了优化。

Solr **可以独立运行**，运行在**Jetty**、**Tomcat**等 Servlet 容器中。Solr 不提供构建 UI 的功能，Solr 提供了一个管理界面，通过管理界面可以查询 Solr 的配置和运行情况。

Solr 索引的实现方法很简单，用 `Post` 方法向 Solr 服务器发送一个描述Field及其内容的JSON文档，Solr根据JSON文档**添加**、**删除**、**更新**索引。Solr搜索只需发送 `Get` 请求，然后对 Solr 返回的JSON等格式的**查询**结果进行解析，组织页面布局。

![](/images/posts/20180306233919166.png)

### 1.2 Solr 与 Lucene 区别

`Lucene` 是一个开放源代码的全文检索引擎工具包，它**不是一个完整的全文检索引擎**，Lucene 提供了完整的查询引擎和索引引擎，目的是**为开发人员提供一个简单易用的工具包**，以方便在目标系统中实现全文检索的功能，或者以 Lucene 为基础构建全文检索引擎。参考文章：[《Lucene 初探——基于 Lucene 6.6.2》](/44bf5506.html)。

`Solr` 目标是打造一款**企业级的搜索引擎系统**，它是一个**搜索引擎服务**，可以独立运行，通过 Solr 可以非常快速的构建企业的搜索引擎。

## 二、Solr 安装与配置

### 2.1 下载 Solr

安装Solr前先要安装好 `JAVA` 和 `Tomcat`，我使用的版本如下：

- JDK 1.8

- Tomcat 8.5

Solr下载地址[点击这里](https://mirrors.aliyun.com/apache/lucene/solr/)，我使用的版本是 `solr-6.6.2`，和[《Lucene 初探——基于 Lucene 6.6.2》](/44bf5506.html)这篇文章的版本相对应。下载完后，解压压缩包，目录结构如下：

![](/images/posts/20180306234340626.png)

| 名称 | 介绍 |
|:-------------|:-------------| 
|bin | Solr的脚本启动目录 |
| contrib | 关于solr的第三方扩展 |
| dist | Solr的核心JAR包和扩展JAR包 |
| dist/solrj-lib| 构建基于Solr的客户端时用到的JAR包 |
| dist/test-framework | 包含测试Solr时候用到的JAR包 |
| docs | Solr文档 |
| example | Solr的简单示例 |
| licenses | 许可协议 |
| server | 本地运行Solr的必要文件 |

### 2.2 搭建 Solr 后台

Solr 自带了一个后台管理，但是不能直接运行，需要进行配置。

在配置前，先说明下我的 Tomcat 安装位于 `D:\apache-tomcat-8.5.16`，下载的solr源码包名称为 `solr-6.6.2`，后面的步骤使用到的目录请根据个人实际情况进行修改。

**Step1：**将 `solr-6.6.2\server\solr-webapp` 下的 `webapp` 复制到 `D:\apache-tomcat-8.5.16\webapps` 目录下，并改名为 `solr`（名称任意）。

![](/images/posts/20180306235719939.png)

**Step2：** 将 `solr-6.6.2\server\lib\ext` 的 jar 包复制到 `D:\apache-tomcat-8.5.16\webapps\solr\WEB-INF\lib` 目录下。

![](/images/posts/20180306235942410.png)

**Step3：**将 `solr-6.6.2\dist` 下的 `solr-dataimporthandler-6.6.2.jar` 和 `solr-dataimporthandler-extras-6.6.2.jar` 复制到 `D:\apache-tomcat-8.5.16\webapps\solr\WEB-INF\lib` 目录下。

![](/images/posts/20180307000147865.png)

**Step4：**将 `solr-6.6.2\server\lib` 下的以 `metrics` 开头的 5 个 jar 包复制到 `D:\apache-tomcat-8.5.16\webapps\solr\WEB-INF\lib` 目录下。

![](/images/posts/2018030700033299.png)

**Step5：** 配置 solr 的家目录。在 E 盘下创建文件夹 `solrHome`（位置名称任意），将 `solr-6.6.2\server\solr` 下的所有文件复制到 `E:\solrHome`。

![](/images/posts/20180307000633692.png)

**Step6：**修改 solr 配置文件。打开 `D:\apache-tomcat-8.5.16\webapps\solr\WEB-INF` 下的 `web.xml`，定位到 40 行，将下面一段注解打开，并修改 `<env-entry-value>` 值为 `E:/solrHome`。

![](/images/posts/20180307000949991.png)

**Step7：**将 `web.xml` 168 行那一大块注释掉，不然访问 solr 会出现没有授权的错误。

![](/images/posts/20180307001230502.png)

**Step8：** 在 `D:\apache-tomcat-8.5.16\webapps\solr\WEB-INF` 目录下创建 `classes` 文件夹，并将 `solr-6.6.2\server\resources` 下的 `log4j.properties` 复制过去。

![](/images/posts/20180307001451632.png)

**Step9：**启动 Tomcat，访问 `http://localhost:8080/solr/index.htm`，登陆到 Solr 后台管理。

![](/images/posts/20180307002927867.png)

### 2.3 创建一个核(core)

现在为 Solr 添加一个`核（core）`，在 `E:/solrHome` 目录下新建一个文件夹，文件夹名就是核的名字，以 `core1` 为例。

![](/images/posts/20180410091110722.png)

拷贝 `E:\solrHome\configsets\sample_techproducts_configs` 目录下 的`conf` 文件夹到 `core1` 文件夹中。

![](/images/posts/20180410091342689.png)

在 Solr 后台的 `Core Admin` 中点击 `Add Core` 新建一个核，`name` 和 `instanceDir` 填之前建的文件夹名。

![](/images/posts/20180410091453209.png)

添加后多了一个 `Core Seletctor`，至此核就建好了。

![](/images/posts/20180410091659647.png)

## 三、后台介绍

![](/images/posts/20180410160618644.png)

| 名称 | 含义 |
|:-------------|:-----| 
| Dashboard | 仪表盘，包含一些参数信息 |
| Logging | 日志信息 |
| Core Admin | 核管理 |
| Java Properties | Java、Tomcat参数信息 |
| Thread Dump | 每次CRUD产生的进程日志 |

核的信息中：

| 名称 | 含义 |
|:-------------|:-----| 
| Overview | 核的参数信息 |
| Analysis | 分词 |
| Dataimport | 数据导入，能够从数据库中将表等信息导入索引库中 |
| Documents | 对文档Document进行增、删、改操作 |
| Files | 显示核的conf目录下所有内容 |
| Ping | 测试与主机的ping值 |
| Plugins / Status | 插件 |
| Query | 对文档Document进行查询操作 |
| Replication | 副本，用于备份机 |
| Schema | 域配置 |
