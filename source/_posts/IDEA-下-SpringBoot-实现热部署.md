---
title: IDEA 下 SpringBoot 实现热部署
tags: SpringBoot
categories:
  - Dev Tools
  - IDEA
abbrlink: 893ce85e
date: 2018-08-01 09:11:30
---

**Step1：** 按照下图所示，勾选`Build project automatically`:

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201808/20180801090556302.png)

**Step2：** 快捷键 `Ctrl + shif + A`，搜索`Registry`，选择第一个，如下图所示：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201808/20180801090608448.png)

勾选下图中`compiler.automake.allow.when.app.running`，然后点击关闭。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201807/20180706094037751.png)

**Step3：** 重启IDEA

**Step4：** 在项目中引入`spring-boot-devtools`依赖，即可：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <optional>true</optional>
    <scope>true</scope>
</dependency>
```
