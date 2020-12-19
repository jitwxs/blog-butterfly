---
title: 解决 Git Revert 后再次 Merge 代码丢失问题
categories:
  - 开发工具
  - Git
tags:
  - Git
  - 踩坑之路
abbrlink: 38727be2
date: 2019-11-30 23:26:18
icons: [fas fa-fire red]
copyright_author: Jitwxs
---

## 一、问题场景

我司使用 GitLab 进行代码管理，当我对系统进行 SpringBoot 2.0 的版本升级，分支已经 Merge 到 Master 分支。实际部署中发现依赖的某个二方包的子依赖未做升级，导致某个服务无法掉通。由于二方包的修复需要时间，为了不影响后续其他功能的发布，因此决定对 Master 分支进行 Revert。

等到第二天，当修复了那个二方包问题后，重新提了 Merge 申请，却发现提交变动只有对二方包的变动，其他的代码变动都没有了。

## 二、问题复现

下面模拟复现下这个问题，我创建了一个全新的项目。在 Master 分支做了首次提交，提交内容为：

```markdown
First Commit From Master
```

![First Commit](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20191130175607270.png)

然后创建一个新分支 `dev`，在提交内容后面追加了以下内容：

```markdown
Second Commit From Dev
```

![Dev Branch Merge](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/2019113017593983.png)

将 dev 分支 merge 到 master 分支，此时 master 分支内容如下：

```markdown
First Commit From Master

Second Commit From Dev
```

此时我觉得 dev 分支还没有修改完毕，想要 revert 后重新提交。因此在 GitLab 中找到那个 Merge 申请，并点击 Revert 按钮，如下图所示。

![Revert Dev Commit](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20191130223242927.png)

点击 Revert 后相当于将代码 Revert 到了一个新分支，并将那个新分支 Merge 到 Master 上。Commit 记录如下图所示，其中的 `revert-f8856e36` 就是那个 revert 分支。

![Commit History](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20191130223506820.png)

Revert 后切换到 Master 分支，发现代码的确已经回滚了。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20191130181032292.png)

此时切换到 dev 分支，在最后再填上一行内容，内容如下：

```
Third Commit From Dev
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20191130180830115.png)

然后重新对 dev 分支发起 Merge 到 Master 的申请。

![Branch List](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20191130181332225.png)

但是发现 dev 分支和 master 分支的差异 commit 只有 dev 分支 revert 后的提交记录。

![Commits Changes](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20191130223710903.png)

再切到 `Changes`页面，发现变动也是只有 revert 后的变动记录。revert 前的 `Second Commit From Dev` 这行内容竟然不算改动，被认为已经存在于 Master 分支上了？

![Code Changes](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20191130181555605.png)

但是切换回 master 分支，`Second Commit From Dev` 这行内容的确已经被 Revert 了。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20191130223810347.png)

以上就完成了对问题的复现，`Second Commit From Dev` 这行内容就丢失了，准确说就是 dev 分支 revert 之前的内容都丢失了。

## 三、解决方案

通过参考以下文章获得了解决方案：

- https://stackoverflow.com/questions/19379034
- http://blog.jdwyah.com/2015/07/dealing-with-git-merge-revisions.html

核心思想就是对 revert 的那次提交记录再次进行 revert，下面开始实验。

首先切换到 Master 分支，并基于 Master 分支拉出一个分支 `revert_tmp`，这个分支的作用是用来保存 revert revert 的提交记录的。

```powershell
~/Projects/demo: git checkout -b revert_tmp
```

找到 revert 的那条提交记录，注意了，revert 相关的会有两条记录，第一条是 revert，第二条是 revert 后 merge 的记录，这里取第一条，如下图所示。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20191130230632590.png)

```powershell
~/Projects/demo: git revert f5c3b544164eec662ea6914d6bd19aedf46874f8
[revert_tmp aea128e] Revert "Revert "Merge branch 'dev' into 'master'""
 1 file changed, 3 insertions(+)
```

然后切换回 dev 分支，将`revert_tmp` 这个分支 merge 到 dev 分支上。

```powershell
~/Projects/demo: git checkout dev
~/Projects/demo: git merge revert_tmp
~/Projects/demo: git push
```

将 dev 分支推送到远程后，重新提交对 master 的 merge 申请。我们发现，在 revert 之前的提交记录还没有找回来，即 `second commit by dev` 这条记录没有找回来，如下图所示。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20191130230959188.png)

这是因为 revert 的特性所致，虽然这条提交记录没有找回来，但是 revert 之前的代码都回来了，如下图所示。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20191130231317474.png)
