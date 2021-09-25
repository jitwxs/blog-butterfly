---
title: Mac 安装 brew
tags: Mac
categories: 开发工具
abbrlink: c9822fb1
date: 2020-09-26 10:36:17
---

## 一、前言

`brew` 是 Mac 和 `Linux` 下的包管理器，但是需要手动安装，在国内操蛋的网络环境下，想要不翻墙安装，还得花点功夫。本文记录在非翻墙情况下，如何安装 `brew`。

## 二、安装流程

### 2.1 官方步骤

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

以上为官方推荐的安装步骤，但是如果你没有开翻墙的话，大概率是会失败的。

### 2.2 国内镜像-1

可以使用以下命令，傻瓜操作。但是很遗憾我的电脑用不了，一直提示没有文件权限。

```bash
/bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/Homebrew.sh)"
```

### 2.3 国内镜像-2

这个方法在本机上能够成功安装、

```bash
/usr/bin/ruby -e "$(curl -fsSL https://cdn.jsdelivr.net/gh/ineo6/homebrew-install/install)"
```

在安装过程中，可能会卡在 ` Tapping homebrew/core` 这一步，如下图所示。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202009/20200926103138.png)

这时候直接 `Control + C` 结束执行，手动依次执行以下命令：

```bash
cd "$(brew --repo)/Library/Taps/"
mkdir homebrew && cd homebrew
git clone git://mirrors.ustc.edu.cn/homebrew-core.git
```

执行完毕后重新执行最初的命令即可：

```bash
/usr/bin/ruby -e "$(curl -fsSL https://cdn.jsdelivr.net/gh/ineo6/homebrew-install/install)"
```

最后提示 `==> Installation successful!` 就安装完毕了。

如果想要卸载 `brew`，使用如下命令即可：

```bash
/usr/bin/ruby -e "$(curl -fsSL https://cdn.jsdelivr.net/gh/ineo6/homebrew-install/uninstall)"
```
