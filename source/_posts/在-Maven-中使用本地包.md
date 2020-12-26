---
title: 在 Maven 中使用本地包
categories: 开发工具
tags: Maven
abbrlink: f22262ea
date: 2018-01-22 10:09:14
copyright_author: Jitwxs
---

## 在项目中使用本地jar包

1. 在项目根目录新建lib文件夹，将所有本地jar包放入该文件夹内

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201801/20180122093133751.png)

2. 在Maven pom.xm文件中按如下方式引入本地jar包：

- scope : 设为system，告诉maven不再从仓库中查找jar包

- systemPath：包在本地中的路径

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201801/20180122093336939.png)

3. 对于IDEA还有一下配置，选择File-->Project Structure-->Libraries，点击左侧的+号，选择Java，引入刚刚lib目录下的所有jar包。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201801/20180122095638751.png)

如果此时运行项目，出现ClassNotDef的异常，且异常信息与我们引入的jar包有关，则还需要配置一下IDEA。

选择File-->Project Structure-->Aritfacts-->项目的exploded，首先查看output root中是否有lib文件夹，没有则新建，然后双击最右侧的所有jar包，它们会自动被添加到lib文件夹中。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201801/20180122095831828.png)

## 将本地jar包打包入war包

在pom.xml的plugins闭合中，添加如下插件即可：

```xml
<!-- 将本地包打入war包 -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-war-plugin</artifactId>
    <configuration>
        <webResources>
            <resource>
                <directory>lib/</directory>
                <targetPath>WEB-INF/lib</targetPath>
                <includes>
                    <include>**/*.jar</include>
                </includes>
            </resource>
        </webResources>
    </configuration>
</plugin>
```
