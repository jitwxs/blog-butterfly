---
title: 从 Git 仓库的 Commit 历史中移除敏感文件
categories: [开发工具, Git]
tags: Git
abbrlink: 9a10ca4e
date: 2019-05-06 11:55:08
references:
  - name: Git移除敏感数据
    url: https://blog.csdn.net/y550918116j/article/details/49913631
    rel: nofollow noopener noreferrer
    target: _blank
  - name: 从Github中的Commit历史移除敏感文件
    url: https://blog.csdn.net/duandianR/article/details/78843582
    rel: nofollow noopener noreferrer
    target: _blank
---

在很多情况，我们由于疏忽会将一些敏感信息误传到 Git 仓库上面去。 尽管我们可以使用 `git rm` 将包含敏感信息文件删除掉，然后重新提交上传，文件就不会在仓库文件列表显示。 但是这并不能完全将敏感信息文件从仓库中完全删除， commit history 仍然会有敏感信息的文件的残留,我们仍然可以从仓库中的 commit history 中访问到文件。

如果想要将敏感信息文件完全删除。不仅需要将文件从 github 中的文件列表中删除，同时还需要将文件从 github 的 commit history 中的 文件相关信息删除。删除 commit history 文件相关信息，主要有两种方法：

1. filter-branch
2. BFG

## 一、filter-branch

### 1.1 移除数据

filter-branch 是 git 自带的命令：

```shell
git filter-branch --force --index-filter \
'git rm --cached --ignore-unmatch PATH-TO-YOUR-FILE-WITH-SENSITIVE-DATA' \
--prune-empty --tag-name-filter cat -- --all
```

请将上面命令中的 `PATH-TO-YOUR-FILE-WITH-SENSITIVE-DATA` 替换为你要删除的文件名路径（相对路径、绝对路径均可）。

如果你想删除文件夹的话，只需要添加一个 `-r` 参数即可，形如：

```shell
git filter-branch --force --index-filter \
'git rm -r --cached --ignore-unmatch PATH-TO-YOUR-DIR-WITH-SENSITIVE-DATA' \
--prune-empty --tag-name-filter cat -- --all
```

### 1.2 避免再次提交

为了防止敏感文件再次被提交，可以将其加入到 `.gitignore` 文件中。

### 1.3 提交仓库

执行以下命令将其强制推送到仓库中：

```shell
git push origin --force --all
```

`-all` 参数将修改作用于远程的所有分支。

### 1.4 提交 tags

以上命令不会对 tag 生效，如需修改，执行命令：

```shell
git push origin --force --tags
```

## 二、BFG

除了使用 git 自带的 filter-barch 命令，还有一个更加方便的命令工具， 可以帮助我们删除 commit history 中的敏感信息。 这就是 [BFG](https://rtyley.github.io/bfg-repo-cleaner/)。

首先下载 BFG 工具：

```shell
wget http://repo1.maven.org/maven2/com/madgag/bfg/1.12.16/bfg-1.12.16.jar
```

执行命令：

```shell
javar -jar bfg-1.12.16.jar --delete-files YOUR-FILE-WITH-SENSITIVE-DATA
```

和使用 `filter-branch` 一样，将 `YOUR-FILE-WITH-SENSITIVE-DATA` 替换为你要删除的文件路径，然后执行命令提交到仓库中：

```shell
git push origin --force --all
```
