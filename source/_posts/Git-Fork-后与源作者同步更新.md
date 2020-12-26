---
title: Git Fork 后与源作者同步更新
categories:
  - 开发工具
  - Git
tags: Git
abbrlink: 6533bf70
date: 2018-11-04 23:17:30
copyright_author: Wayss_S
---

## 一、图形化操作

（1）打开 fork 过来的项目，点击 new pull request

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201702/20170221001500650.png)

（2）在进入的界面， 将左边的设置为你自己的仓库， fork 过来的在右边， 然后点击 Create pull request，如下图：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201702/20170221001519228.png)

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201702/20170226194654555.png)

（3）点击 Merge pull request 合并从源 fork 来的代码：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201702/20170221001548713.png)

## 二、命令行操作

先总结下主要命令：

```shell
git remote -v 
git remote add upstream git@github.com:xxx/xxx.git
git fetch upstream
git merge upstream/master
git push
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201702/20170221095054215.png)

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201702/20170221095105168.png)

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201702/20170221095114372.png)
