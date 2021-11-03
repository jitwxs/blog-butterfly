---
title: Git 修改已经被覆盖的提交
categories:
  - Dev Tools
  - Git
tags: Git
abbrlink: 33ebef44
date: 2019-07-11 22:35:42
---

如果你不想看详细的描述，直接看步骤即可：

1.`git rebase -i HEAD~n`，将要修改的提交状态改为 `edit`
2.修改文件
3.`git add`
4.`git commit --amend`
5.`git rebase --continue`

假设我们目录下有三个文件，分别是 `digit.dat` 、`letter.dat`和`symbol.dat`，`digit.dat` 中存放着数字，`letter.dat` 中存放着字母，`symbol.dat` 中存放着符号。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201711/20171126210719360.png)

我们先提交 digit.dat，然后提交 letter.dat，最后提交 symbol.dat：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201711/20171126210727526.png)

此时我们想起来，`letter.dat` 中仅仅存放了小写字母，大写字母被遗忘了，我们要把大写字母补充进去。这时候有两种解决方法：

- 修改 letter.dat 内容，并发起一次新提交

- 不发起新提交，对之前的那次提交做修改

我们不讨论发起新提交的情况，仅仅讨论如何对那次提交做修改。你也许会说，这很简单啊，直接使用命令 `reset HEAD^` 或者 `reset 8cc5dc` 不就好了吗?

不错，这没有问题，但是这样的话我们对`symbol.dat`的那次提交**也被撤回**了，这显然是一个笨方法，特别是当你要修改的那次提交被覆盖的很深的情况下。

有没有在不影响其他的提交的情况下，对已经被覆盖的提交做修改的方法呢？答案是有的。

**Step1 ：** `git rebase -i HEAD~n`

我们使用命令 `git rebase -i HEAD~2` 列出最新的两次提交：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201711/20171126211004897.png)

在 `git rebased` 命令的交互命令中，首先**逆序**存放了我们最新的两次提交，最前面的状态为 `pick`。下面也提示了我们可以执行哪些命令，这里我们这里只使用它的 `edit` 功能。

将我们要修改的那次提交状态改为 `edit` 或 `e`，也就是将第一行内容改为：

>edit 8cc5dcb add letter.dat

然后保存退出：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201711/20171126211506346.png)

git 已经提示我们此时已经**暂停**在了 8cc5dc 这一次提交上，后面新的提交此时应该被隐藏掉了，查看日志验证我们的判断：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201711/20171126211723044.png)

不出所料，添加 symbol.dat 的那次提交已经被隐藏掉了，添加 letter.dat 的那次提交已经成为当前最新的提交了，这时候我们就可以轻松对它进行操作了。

**Step2 ：** 修改文件内容

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201711/20171126212114623.png)

**Step3 ：** `git add`  并 `git commit --amend`

其实之前的图上已经提示我们如何操作了，先 `git add`，然后执行命令 `git commit --amend` 提交变更。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201711/20171126212615244.png)

**Step4 ：** `git rebase --continue`

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201711/20171126212646143.png)

此时就已经完成了对那次提交的修改，查看日志和文件内容，大功告成。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201711/20171126213024343.png)
