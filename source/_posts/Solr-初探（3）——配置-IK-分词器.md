---
title: Solr 初探（3）——配置 IK 分词器
tags: 分词
categories:
  - 搜索引擎
  - Solr
toc: false
abbrlink: b381cc92
date: 2018-04-10 16:03:17
copyright_author: Jitwxs
---

Solr 配置第三方分词器也是十分简单，这里以IK分词器为例。[点击下载](https://download.csdn.net/download/yuanlaijike/10270713)我自编译的 `IK 分词器`，支持到 JDK 1.8 + Lucene 6.6.2。

**Step1：** 将 `IK 分词器`的jar包放到 `D:\apache-tomcat-8.5.16\webapps\solr\WEB-INF\lib` 目录下。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180410155033664.png)

**Step2：** 将 IK 分词器的配置文件放到 `D:\apache-tomcat-8.5.16\webapps\solr\WEB-INF\classes` 文件夹下，文件夹如果不存在手动创建：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/2018041015531317.png)

这三个配置文件作用及内容不再赘述，查看文章：[《Lucene初探——基于Lucene 6.6.2》](/44bf5506.html)

**Step3：** 在 `managed-schema` 配置文件中添加IK分词器的类型，以及创建使用 IK 分词器的域：

```xml
<!-- IK分词器 -->
<fieldType name="text_ik" class="solr.TextField">
	<analyzer class="org.wltea.analyzer.lucene.IKAnalyzer"/>
</fieldType>

<!-- IK分词器的域 -->
<field name="title_ik" type="text_ik" indexed="true" stored="true" />
<field name="content_ik" type="text_ik" indexed="true" stored="false" multiValued="true"/>
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180410160210931.png)

**Step4：** 重新启动 Solr 服务，测试一下 IK 分词器的效果：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180410160118242.png)
