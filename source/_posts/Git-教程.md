---
title: Git 教程
categories: 
  - 开发工具
  - Git
tags: Git
typora-root-url: ..
abbrlink: 2121b11b
date: 2017-09-16 16:04:00
copyright_author: Jitwxs
---

Git是一款免费、开源的分布式版本控制系统，用于敏捷高效地处理任何或小或大的项目。本文为廖雪峰的Git教程的个人笔记，欢迎指正。

## 一、Git 基础

### 1.1 版本控制系统

版本控制系统系统分为**集中式**和**分布式**两种。

集中式是集中存放在中央服务器中，工作的时候，先用自己的电脑从中央服务器取得最新的版本，完工后，再把自己的工作推送到中央服务器。集中式版本控制工具有：CVS、SVN 等。

集中式的特点是需要联网才能工作，对网速的要求较高。如果中央服务器发生了宕机，所有人都没法工作。

![集中式](/images/posts/20170912171557531.png)

分布式版本控制系统没有“中央服务器”，每个人的电脑上都是一个完整的版本库，工作的时候不需要联网。需要彼此的文件，只需要相互推送。实际使用中，分布式版本控制控制系统通常也有一台充当“中央服务器”的电脑，但仅仅是用来方便“交换”大家的修改，没有它大家一样能正常工作，只是交换修改不方便。分布式版本控制工具有：`Git`、Mercurial、Bazaar 等。

分布式的特点是不需要联网就能工作。不依赖中央服务器，安全性高。

![分布式](/images/posts/20170912171610347.png)

### 1.2 安装 Git

测试是否正确安装了 Git：

![Git测试安装](/images/posts/20170912171451264.png)

如果像上图这样显示没有安装，通过相应包管理直接安装即可。我的电脑为`CentOs`，因此键入命令

```shell
yum install git
```

安装完成后，还需要最后一步设置，在命令行输入：

```shell
$ git config --global user.name "Your Name"
$ git config --global user.email "email@example.com"
```

因为 Git 是分布式版本控制系统，所以，每个机器都必须自报家门：你的名字和 Email 地址。

注意：`git config` 命令的 `--global` 参数，用了这个参数，表示你这台机器上所有的 Git 仓库都会使用这个配置，当然也可以对某个仓库指定不同的用户名和Email地址。

![自报家门](/images/posts/20170912180005051.png)

### 1.3 工作区和暂存区

Git和其他版本控制系统如SVN有一个不同之处就是有`暂存区`的概念。首先我们先了解以下概念的意义。

**工作区（Working Directory）**

就是你在电脑里能看到的目录，比如learngit文件夹就是一个工作区：

![工作区](/images/posts/20170912172000402.png)

**版本库（Repository）**

版本库可以理解为一个目录，在目录中的所有文件都可以被 Git 管理起来，每个文件的修改、删除，Git 都能跟踪，一边任何时刻都可以追踪历史，或者在将来进行还原。

工作区有一个隐藏目录 `.git`，这个不算工作区，而是 Git 的版本库。

Git的版本库里存了很多东西，其中最重要的就是称为 `stage`（或者叫index）的暂存区，还有Git为我们自动创建的第一个分支 `master`，以及指向master的一个指针叫 `HEAD`。

![版本库](/images/posts/20170912172033783.png)

我们把文件往Git版本库里添加的时候，是分两步执行的：

1. `git add` 把文件添加进去，实际上就是把文件修改添加到暂存区；
2. `git commit` 提交更改，实际上就是把暂存区的所有内容提交到当前分支。

因为我们创建 Git 版本库时，Git 自动为我们创建了唯一一个 master 分支，所以，现在，`git commit` 就是往 master分支上提交更改。

你可以简单理解为，**需要提交的文件修改通通放到暂存区，然后，一次性提交暂存区的所有修改**。

### 1.4 三个具有代表性的命令

1. 添加到版本库：`git add`

2. 提交到版本库：`git commit`

3. 把本地版本库推送到远程版本库：`git push`

![三大命令](/images/posts/20170912172601157.png)

## 二、版本控制

### 2.1 创建版本库

选择合适路径，创建一个空目录，使用 `git init` 初始化 git 管理的目录

![创建版本库](/images/posts/20170912175429932.png)

然后我们添加文件到版本库：

1. 编辑文本文件 readme.txt
2. 添加到版本库：`git add`
3. 提交到版本库：`git commit`

>注1：commit 命令使用 `-m` 参数可以添加本次提交的注释。这非常有用，在团队合作中能够让别人理解你本次提交的修改内容。
>注2：我们可以用 git add 添加多个文件到暂存区，然后使用 git commit 一次性提交。

![提交文件](/images/posts/20170912180656974.png)

### 2.2 查看仓库状态

**git status** 查看当前仓库状态。

修改 readme.txt 文件，使用 `git status` 查看仓库状态，下图说明 readme.txt 文件被修改，但还没有提交到仓库。

![查看仓库状态](/images/posts/20170912181209719.png)

**git diff** 查看修改的内容

下图说明 readme.txt，添加了一行内容。

![查看修改](/images/posts/20170912181853657.png)

将 readme.txt 添加到暂存区并提交后，再次使用 `git status` 和 `git diff` 查看。

![查看修改2](/images/posts/20170912182027811.png)

**git log** 查看仓库历史纪录

`git log` 命令显示从最近到最远的提交日志。

![查看历史纪录](/images/posts/20170912191332442.png)

### 2.3 版本回退

**git reset** 回退到指定版本

`git reset` 具有三个参数，分别是 `soft`、`hard` 和 `mixed`。

`--soft` 参数告诉 Git 重置 HEAD 到另外一个 commit，但也到此为止。如果你指定 --soft 参数，Git将停止在那里而什么也不会根本变化。这意味着 index, working copy 都不会做任何变化，所有的在 original HEAD 和你重置到的那个 commit 之间的所有变更集仍然在stage(index)区域中。

![git reset --soft](/images/posts/20170912190718334.png)

`--hard` 参数将会将重置 HEAD 返回到另外一个 commit，重置 index 以便反映HEAD的变化，并且重置 working copy 也使得其完全匹配起来。这是一个比较危险的动作，具有破坏性，数据因此可能会丢失！如果真是发生了数据丢失又希望找回来，那么只有使用 `git reflog` 命令了。

![git reset --hard](/images/posts/20170912190810065.png)

`--mixed` 是 reset 的默认参数，也就是当你不指定任何参数时的参数。它将重置 HEAD 到另外一个 commit,并且重置index 以便和 HEAD比配，但是也到此为止。working copy 不会被更改。所以所有从 original HEAD到你重置到的那个 commit 之间的所有变更仍然保存在 working copy中，被标示为已变更，但是并未staged的状态。

![git reset --mixed](/images/posts/20170912190847304.png)

Git 中，用 HEAD 表示当前版本，上一个版本就是HEAD^ ，上上个版本就是HEAD^^。
撤销后再查看记录发现提交记录少了一个

![版本撤销](/images/posts/20170912191519806.png)

也可以指定回滚的版本号。版本号用 `git log` 查看 commit id 前7位。

如果我们忘记回退前的版本号，`git reflog` 命令记录了每一次的命令操作，包含了所有版本的版本号。

![查看操作记录](/images/posts/20170912191852220.png)

### 2.4 撤销修改

**git checkout - - file**

该命令会把文件在工作区的修改全部撤销，回到最近一次 `git commit` 或 `git add` 时的状态。

- 文件自修改后还没有被放到暂存区，撤销修改就回到和版本库一模一样的状态
- 文件已经添加到了暂存区后。又做了修改，撤销修改就回到添加到暂存区后的状态。

注：`--`请不要丢掉，丢掉后就成为了创建分支的命令。

![撤销修改](/images/posts/20170912192201582.png)

**git reset HEAD file**

该命令会把暂存区的修改撤销掉，重新放回工作区中。

我们之前在回退版本时已经介绍了`git reset`命令，该命令既可以回退版本，也可以把工作区的某些文件替换为版本库中的文件。使用HEAD表示为最新的版本。

### 2.5 删除文件

**git rm file**

如果使用 rm 删除了版本库中的文件 test.txt，然后用 `git status` 查看仓库状态。

![查看删除文件](/images/posts/20170912192621199.png)

如果确实要将 test.txt 文件从 Git 仓库删除，使用 `git rm`，并 commit。

![git 中删除](/images/posts/20170912192824084.png)

![提交删除](/images/posts/20170912192851925.png)

如果误删，使用 `git checkout -- file` 恢复到之前版本。

![误删文件](/images/posts/20170912192451959.png)

### 2.6 命令小节

![版本控制命令小节](/images/posts/20171126203113510.png)

## 三、远程仓库

远程仓库即远程的Git服务器仓库，每个用户都能从这个服务器仓库克隆一份到自己的电脑上，并且各自把各自的提交推送到服务器仓库中，也可以从服务器仓库中获取别人的提交。

### 3.1 创建和设置远程仓库

我们可以使用自己的服务器创建一个Git仓库，也可以使用GitHub为我们提供的仓库。这里使用GitHub，首先注册[GitHub](https://github.com)账号。

GitHub需要识别推送者的身份，防止别人冒充。Git支持SSH协议，因此GitHub只要知道了你的公钥，就可以确认推送者的身份。GitHub允许添加多个Key，确保用户可以在多台电脑上进行推送。

~~需要注意的是，GitHub只免费提供公开的Git库，因此你提交的内容所有人都可以查看，因此敏感信息尽量不要出现。如果不想让别人看到，可以选择GitHub付费服务或者自己搭建Git服务器。~~（2019年1月 Github 推出了私有仓库）

首先创建SSH Key密钥。一路回车使用默认值即可（如果没有特殊需求无需设置密码）：

>ssh-keygen -t rsa -C "user.email"

![创建密钥](/images/posts/20170912195012211.png)

然后可以在用户主目录里找到.ssh目录，里面有 `id_rsa` 和 `id_rsa.pub` 两个文件，这两个就是 SSH Key 的秘钥对，`id_rsa` 是私钥，不能泄露出去，`id_rsa.pub` 是公钥，可以放心地告诉任何人。

复制 `~/.ssh/id_rsa.pub` 内容。

![查看公钥](/images/posts/20170912195051822.png)

第二步，登陆GitHub，打开`Settings` -->`SSH And GPL Keys`-->`New SSH key`--> `Add SSH key`，将之前的内容粘贴上去即可。

![添加Key](/images/posts/20170912195530311.png)

### 3.2 添加远程库

首先登陆自己的 GitHub，创建一个新的仓库，这里我设置仓库名为 Test。

![创建仓库](/images/posts/20170912200351998.png)

GitHub 告诉我们有以下几种选择：

- 从这个仓库克隆出新的仓库
- 把一个已有的本地仓库与之关联
- 从其他仓库导入代码。

根据提示，我们把本地仓库的内容推送到 GitHub 仓库。在本地仓库下，执行 GitHub 提示我们的命令即可。

![关联仓库](/images/posts/20170912200643745.png)

由于远程库是空的，我们第一次推送 master 分支时，加上了 `-u` 参数，Git不但会把本地的master分支内容推送的远程新的 master 分支，还会把本地的 master 分支和远程的 master 分支关联起来，在以后的推送或者拉取时就可以不使用 `-u` 参数。

我们发现上图中出现了 Warning，这是因为当你第一次使用 Git 的 clone 或者 push 命令连接 GitHub 时，由于我们使用 SSH 连接，而 SSH 连接在第一次验证 GitHub 服务器的 Key 时，需要你确认 GitHub 的 Key 的指纹信息是否真的来自 GitHub 的服务器，输入 yes 回车即可。

Git 会输出一个警告，告诉你已经把 GitHub 的 Key 添加到本机的一个信任列表里了，这个警告只会出现一次，后面的操作就不会有任何警告了。

### 3.3 从远程库克隆

这里用我已经存在的 OJCrawler 项目为例，用命令 `git clone` 将该项目克隆到本地。

```shell
git clone git@github.com:jitwxs/OJCrawler.git
```

![克隆仓库到本地](/images/posts/20170912201635992.png)

### 3.4 命令小节

![远程控制命令小节](/images/posts/20170912202555746.png)

## 四、分支管理
分支就是科幻电影里面的平行宇宙，当你正在电脑前努力学习 Git 的时候，另一个你正在另一个平行宇宙里努力学习 SVN。

如果两个平行宇宙互不干扰，那对现在的你也没啥影响。不过，在某个时间点，两个平行宇宙合并了，结果，你既学会了 Git 又学会了 SVN！

![分支漫画](/images/posts/20170912204359273.png)

分支在实际中有什么用呢？假设你准备开发一个新功能，但是需要两周才能完成，第一周你写了 50% 的代码，如果立刻提交，由于代码还没写完，不完整的代码库会导致别人不能干活了。如果等代码全部写完再一次提交，又存在丢失每天进度的巨大风险。

现在有了分支，就不用怕了。你创建了一个属于你自己的分支，别人看不到，还继续在原来的分支上正常工作，而你在自己的分支上干活，想提交就提交，直到开发完毕后，再一次性合并到原来的分支上，这样，既安全，又不影响别人工作。

其他版本控制系统如 SVN 等都有分支管理，但是用过之后你会发现，这些版本控制系统创建和切换分支比蜗牛还慢，简直让人无法忍受，结果分支功能成了摆设，大家都不去用。

但Git的分支是与众不同的，得益于它的数据结构，使得无论创建、切换和删除分支，Git 在很短时间就能完成。


### 4.1 创建与合并分支

Git 将每次提交的内容串成一条时间线，这条时间线就是一个分支。在默认情况下只有一个分支，这个分支叫主分支，即 master 分支。HEAD 严格来说不是指向提交，而是指向master，master才是指向提交的，所以，HEAD指向的就是当前分支。

一开始的时候，master 分支是一条线，Git 用 master 指向最新的提交，再用 HEAD 指向 master，就能确定当前分支，以及当前分支的提交点：

![](/images/posts/20170912205225773.png)

每次提交，master 分支都会向前移动一步，这样，随着你不断提交，master 分支的线也越来越长。

当我们创建新的分支，例如 dev 时，Git 新建了一个指针叫 dev，指向 master 相同的提交，再把 HEAD 指向dev，就表示当前分支在 dev 上。

![](/images/posts/20170912205300693.png)

由这个数据结构我们知道，Git 创建一个分支很快，因为除了增加一个 dev 指针，改改 HEAD 的指向，工作区的文件都没有任何变化！

从现在开始，对工作区的修改和提交就是针对 dev 分支了，比如新提交一次后，dev 指针往前移动一步，而master 指针不变。

![](/images/posts/20170912205342023.png)

假如我们在 dev 上的工作完成了，就可以把 dev 合并到 master 上。最简单的方法，就是直接把 master 指向 dev 的当前提交，就完成了合并：

![](/images/posts/20170912205415023.png)

合并完分支后，甚至可以删除 dev 分支。删除 dev 分支就是把 dev 指针给删掉，删掉后，我们就剩下了一条master 分支：

![](/images/posts/20170912205428759.png)

说了这么多理论，下面开始实际操作。

首先，我们创建 dev 分支，然后切换到 dev 分支：

```shell
git checkout -b dev
```

`git checkout` 命令加上 `-b` 参数表示创建并切换，相当于以下两条命令：

```shell
git branch dev
git checkout dev
```

然后，用 `git branch` 命令查看分支，当前活跃分支前会添加 `*` 号：

![查看分支](/images/posts/20170912210122240.png)

下面我们为 readme.txt 文件添加一行，并提交 dev 分支。

![在dev分支修改内容](/images/posts/20170912210449417.png)

切换回 master 分支后，再查看一个 readme.txt 文件，刚才添加的内容不见了！

![切换回master](/images/posts/20170912210623085.png)

因为那个提交是在 dev 分支上，而 master 分支此刻的提交点并没有变。

![](/images/posts/20170912210710293.png)

现在，我们把 dev 分支的工作成果合并到 master 分支上，`git merge` 命令用于合并指定分支到当前分支。

合并后，再查看 readme.txt 的内容，就可以看到，和 dev 分支的最新提交是完全一样的。

![合并dev分支](/images/posts/20170912210957941.png)

注意到上面的 Fast-forward 信息，Git 告诉我们，这次合并是“快进模式”，也就是直接把master指向dev的当前提交，所以合并速度非常快。

合并完成后，就可以放心地删除 dev 分支了 使用 `git branch -d 分支名`。

删除后，查看 branch，就只剩下 master 分支了：

![删除dev分支](/images/posts/20170912211053186.png)

因为创建、合并和删除分支非常快，所以 Git 鼓励你使用分支完成某个任务，合并后再删掉分支，这和直接在master分支上工作效果是一样的，但过程更安全。

### 4.2 解决冲突

合并分支并非总是这么一帆风顺，有可能遇上冲突问题。准备新的 feature1 分支，来模拟冲突问题。

![创建feature1分支](/images/posts/20170912212407876.png)

修改 readme.txt 最后一行，改为：

> this changed by feature1 branch..

在 feature1 分支上提交：

![提交feature1](/images/posts/20170912212504122.png)

切换到 master 分支：

![切换master](/images/posts/20170912212541805.png)

Git 还会自动提示我们当前 master 分支比远程的 master 分支要超前 1 个提交。

在 master 分支上把 readme.txt 文件的最后一行改为：

> this changed by master branch..

提交：

![提交master](/images/posts/20170912212621019.png)

现在，master 分支和 feature1 分支各自都分别有新的提交，变成了这样：

![](/images/posts/20170912212658873.png)

这种情况下，Git 无法执行“快速合并”，只能试图把各自的修改合并起来，但这种合并就可能会有冲突，我们试试看：

![尝试合并冲突](/images/posts/20170912212740792.png)

果然冲突了！Git 告诉我们，readme.txt 文件存在冲突，必须手动解决冲突后再提交。

`git status` 也可以告诉我们冲突的文件：

![git status冲突](/images/posts/20170912212832585.png)

我们可以直接查看 readme.txt 的内容：

![查看冲突内容](/images/posts/20170912212910823.png)

Git用`<<<<<<<`，`=======`，`>>>>>>>`标记出不同分支的内容，我们将最后一行修改如下后保存：

> this changed by master and dev branch..

再提交：

![查看合并](/images/posts/20170912213013874.png)

现在，master 分支和 feature1 分支变成了下图所示：

![](/images/posts/20170912213042282.png)

用带参数的 `git log` 也可以看到分支的合并情况：

```
git log --graph --pretty=oneline --abbrev-commit
```

![git log冲突](/images/posts/20170912213141253.png)

最后，删除 feature1 分支：

![删除feature1](/images/posts/20170912213230299.png)

### 4.3 分支管理策略

通常，合并分支时，如果可能，Git 会用 Fast forward 模式，但这种模式下，删除分支后，会丢掉分支信息。

我们可以强制禁用 Fast forward 模式，Git 就会在 merge 时生成一个新的 commit，这样，从分支历史上就可以看出分支信息。

下面我们实战一下 non-fast-forward 方式的 merge。

首先，仍然创建并切换 dev 分支：

![创建dev分支](/images/posts/20170912215225591.png)

修改 readme.txt 文件，并提交一个新的 commit：

![提交dev分支](/images/posts/20170912215308121.png)

现在，我们切换回 master：

![切换到master](/images/posts/20170912215632518.png)

准备合并 dev 分支，请注意 `--no-ff` 参数，表示禁用 Fast forward。因为本次合并要创建一个新的 commit，所以加上`-m` 参数，把 commit 描述写进去。

![noff合并分支](/images/posts/20170912215721082.png)

合并后，我们用 git log 看看分支历史：

![git log历史](/images/posts/20170912215805050.png)

可以看到，不使用 Fast forward 模式，merge 后就像这样：

![](/images/posts/20170912215835059.png)

在实际开发中，我们应该按照几个基本原则进行分支管理：

首先，master 分支应该是非常稳定的，也就是仅用来发布新版本，平时不能在上面干活，干活都在 dev 分支上。

也就是说，dev 分支是不稳定的，到某个时候，比如 1.0 版本发布时，再把 dev 分支合并到 master 上，在 master 分支发布1.0版本；

你和你的小伙伴们每个人都在 dev 分支上干活，每个人都有自己的分支，时不时地往 dev 分支上合并就可以了。

所以，团队合作的分支看起来就像这样：

![分支策略](/images/posts/20170912215858481.png)

### 4.4 Bug 分支

当你接到一个修复 bug 的任务时，很自然地，你想创建一个分支来修复它。但是等等，当前分支上进行的工作还没有提交。并不是你不想提交，而是工作只进行到一半，还没法提交。如下图，demo.c 文件还没有完成，无法提交。

![无法提交演示](/images/posts/20170912222054549.png)

但是，你必须紧急修复该 bug，怎么办？幸好，Git还提供了一个 stash 功能，可以把当前工作现场“储藏”起来，等以后恢复现场后继续工作。

![stash](/images/posts/20170912222140963.png)

现在，用 `git status` 查看工作区，就是干净的（除非有没有被 Git 管理的文件），因此可以放心地创建分支来修复 bug。

![查看隐藏后status](/images/posts/20170912222225438.png)

首先确定要在哪个分支上修复 bug，假定需要在 master 分支上修复，就从 master 创建临时分支：

![创建issue分支](/images/posts/20170912222308258.png)

现在模拟修复 bug，随便修改 readme.txt 内容，然后提交：

![提交issue分支](/images/posts/20170912222622375.png)

修复完成后，切换到 master 分支，并完成合并，最后删除 issue 分支：

![删除issue](/images/posts/20170912222706173.png)

修复完了 Bug，是时候接着回到 dev 分支干活了！

![返回dev](/images/posts/20170912222820410.png)

工作区是干净的，刚才的工作现场存到哪去了？用 `git stash list` 命令看看：

![查看stash](/images/posts/20170912222846790.png)

工作现场还在，Git 把 stash 内容存在某个地方了，但是需要恢复一下，有两个办法：

- 用 `git stash apply` 恢复，但是恢复后，stash 内容并不删除，你需要用 `git stash drop` 来删除。

- 用 `git stash pop`，恢复的同时把 stash 内容也删了。

这里假设我们不需要保留 stash 的内容了，使用 `git stash pop` 命令。再用 `git stash list` 查看，就看不到任何 stash 内容了：

![stash pop](/images/posts/20170912222928064.png)

当然了，你可以多次stash，恢复的时候，先用`git stash list`查看，然后恢复指定的stash，用命令：

```shell
git stash apply stash@{0}
```

### 4.5 多人协作

你从远程仓库克隆时，实际上 Git 自动把本地的 master 分支和远程的 master 分支对应起来了，并且，远程仓库的默认名称是 origin。

用 `git remote` 查看远程库的信息,或者，用 `git remote -v` 显示更详细的信息：

![git remote](/images/posts/20170912224356423.png)

上面显示了可以抓取和推送的 origin 的地址。如果没有推送权限，就看不到 push 的地址。

多人协作时，大家都会往 master 和 dev 分支上推送各自的修改。

当你的小伙伴从远程库 clone 时，默认情况下，你的小伙伴只能看到本地的 master 分支。

现在，你的小伙伴要在 dev 分支上开发，就必须创建远程 origin 的 dev 分支到本地，于是他用 `git checkout -b dev origin/dev` 创建本地 dev 分支：

现在，他就可以在dev上继续修改，然后，时不时地把 dev 分支 push 到远程。

碰巧你也对同样的文件作了修改，并试图推送就会出错失败，因为你的小伙伴的最新提交和你试图推送的提交有冲突，解决办法也很简单，先用git pull把最新的提交从 `origin/dev` 克隆下来，然后，在本地合并，解决冲突，再推送：

以上就是多人协作的工作模式，下面来总结一下：

- 首先，可以试图用 `git push origin branch-name` 推送自己的修改；

- 如果推送失败，则因为远程分支比你的本地更新，需要先用`git pull`试图合并；

- 如果合并有冲突，则解决冲突，并在本地提交；

- 没有冲突或者解决掉冲突后，再用 `git push origin branch-name` 推送就能成功！

- 如果 `git pull` 提示 “no tracking information”，则说明本地分支和远程分支的链接关系没有创建，用命令 `git branch --set-upstream branch-name origin/branch-name`。

### 4.6 命令小节

![分支命令](/images/posts/20170912224607107.png)

## 五、标签管理

发布一个版本时，我们通常先在版本库中打一个标签（tag），这样，就唯一确定了打标签时刻的版本。将来无论什么时候，取某个标签的版本，就是把那个打标签的时刻的历史版本取出来。所以，标签也是版本库的一个快照。

Git的标签虽然是版本库的快照，但其实它就是指向某个commit的指针（与分支很像但是分支可以移动，标签不能移动）。

Git有commit，为什么还要引入tag？

“请把上周一的那个版本打包发布，commit号是6a5819e...”

“一串乱七八糟的数字不好找！”

如果换一个办法：

“请把上周一的那个版本打包发布，版本号是v1.2”

“好的，按照tag v1.2查找commit就行！”

所以，tag就是一个让人容易记住的有意义的名字，它跟某个commit绑在一起。

### 5.1 创建标签

在Git中打标签非常简单，首先，切换到需要打标签的分支上，然后，敲命令 `git tag <name>` 就可以打一个新标签：

![打标签](/images/posts/20170912225741049.png)

可以用命令 `git tag` 查看所有标签：

![查看标签](/images/posts/20170912232355386.png)

默认标签是打在最新提交的 commit 上的。如果想要给历史 commit 打标签，可以找到历史提交的 commit id，然后打上就可以了：

![查看git log](/images/posts/20170912232446793.png)

比方说要对 add merge 这次提交打标签，它对应的 commit id 是 3728b64，敲入命令 `git tag v0.1 6224937`：

再用命令 `git tag` 查看标签：

![查看标签2](/images/posts/20170912232523097.png)

注意，标签不是按时间顺序列出，而是按字母排序的。可以用 `git show <tagname>` 查看标签信息：

![查看标签具体](/images/posts/20170912232613422.png)

可以看到，v0.1 确实打在 add merge 这次提交上。

还可以创建带有说明的标签，用 `-a` 指定标签名，`-m` 指定说明文字，例如：

```shell
git tag -a v0.1 -m "version 0.1 released" 3628164
```

还可以通过 `-s` 用私钥签名一个标签：

```shell
$ git tag -s v0.2 -m "signed version 0.2 released" fec145a
```

签名采用 PGP 签名，因此，必须首先安装 gpg（GnuPG）,用 PGP 签名的标签是不可伪造的，因为可以验证 PGP 签名。


### 5.2 操作标签

如果标签打错了，也可以删除：

```shell
git tag -d v0.1
```

![删除标签](/images/posts/20170912230447892.png)

因为创建的标签都只存储在本地，不会自动推送到远程。所以，打错的标签可以在本地安全删除。

如果要推送某个标签到远程，使用命令 `git push origin <tagname>`，或者，使用 `git push origin --tags` 一次性推送全部尚未推送到远程的本地标签。

![推送标签](/images/posts/20170912230518979.png)

如果标签已经推送到远程，要删除远程标签就麻烦一点，先从本地删除：

```shell
git tag -d v0.1
```

然后，从远程删除。删除命令也是 push，但是格式如下：

```shell
git push origin :refs/tags/v0.1
```

要看看是否真的从远程库删除了标签，可以登陆 GitHub 查看。

### 5.3 命令小节

![标签命令](/images/posts/20170912232647503.png)

## 六、自定义 Git

### 6.1 搭建 Git 服务器

#### 6.1.1 搭建环境

GitHub 就是一个免费托管开源代码的远程仓库。但是对于某些视源代码如生命的商业公司来说，既不想公开源代码，又舍不得给 GitHub 交保护费，那就只能自己搭建一台 Git 服务器作为私有仓库使用。

搭建 Git 服务器需要准备一台运行 Linux 的机器或服务器。

假设你已经有 sudo 权限的用户账号，下面，正式开始安装。

第一步，安装 git

第二步，创建一个 git 用户，用来运行 git 服务：

```shell
sudo adduser git
```

第三步，创建证书登录：

收集所有需要登录的用户的公钥，就是他们自己的 `id_rsa.pub` 文件，把所有公钥导入到 `/home/git/.ssh/authorized_keys` 文件里，一行一个。

第四步，初始化 Git 仓库：

先选定一个目录作为 Git 仓库，假定是 /srv/sample.git，在 /srv 目录下输入命令：

```shell
sudo git init --bare sample.git
```

Git 就会创建一个裸仓库，裸仓库没有工作区，因为服务器上的Git仓库纯粹是为了共享，所以不让用户直接登录到服务器上去改工作区，并且服务器上的Git仓库通常都以.git结尾。然后，把owner改为git：

```shell
sudo chown -R git:git sample.git
```

第五步，禁用 shell 登录：

出于安全考虑，第二步创建的 git 用户不允许登录shell，这可以通过编辑 `/etc/passwd` 文件完成。找到类似下面的一行：

```shell
git:x:1001:1001:,,,:/home/git:/bin/bash
```

改为：

```shell
git:x:1001:1001:,,,:/home/git:/usr/bin/git-shell
```

这样，git 用户可以正常通过 ssh 使用 git，但无法登录 shell，因为我们为 git 用户指定的 git-shell 每次一登录就自动退出。

第六步，克隆远程仓库：

现在，可以通过 git clone 命令克隆远程仓库了，在各自的电脑上运行：

```shell
git clone git@server:/srv/sample.git
```

剩下的推送就简单了。

#### 6.1.2 管理公钥

如果团队很小，把每个人的公钥收集起来放到服务器的/home/git/.ssh/authorized_keys文件里就是可行的。如果团队有几百号人，可以用Gitosis来管理公钥。

### 6.2 设置忽略文件

有些时候，你必须把某些文件放到 Git 工作目录中，但又不能提交它们，比如保存了数据库密码的配置文件啦，等等，每次 `git status` 都会显示 `Untracked files ...`。

好在 Git 考虑到了大家的感受，这个问题解决起来也很简单，在 Git 工作区的根目录下创建一个特殊的 `.gitignore` 文件，然后把要忽略的文件名填进去，Git 就会自动忽略这些文件。

GitHub 已经为我们准备了各种配置文件，只需要组合一下就可以使用了。所有配置文件可以直接在线浏览：https://github.com/github/gitignore 。

忽略文件的原则是：

- 忽略操作系统自动生成的文件，比如缩略图等；

- 忽略编译生成的中间文件、可执行文件等，也就是如果一个文件是通过另一个文件自动生成的，那自动生成的文件就没必要放进版本库，比如Java编译产生的.class文件；

- 忽略你自己的带有敏感信息的配置文件，比如存放口令的配置文件。

举个例子：

假设你在Windows下进行Python开发，Windows会自动在有图片的目录下生成隐藏的缩略图文件，如果有自定义目录，目录下就会有Desktop.ini文件，因此你需要忽略Windows自动生成的垃圾文件：

```ini
# Windows:
Thumbs.db
ehthumbs.db
Desktop.ini
```

然后，继续忽略Python编译产生的.pyc、.pyo、dist等文件或目录：

```ini
# Python:
*.py[cod]
*.so
*.egg
*.egg-info
dist
build
```

加上你自己定义的文件，最终得到一个完整的.gitignore文件，内容如下：

```ini
# Windows:
Thumbs.db
ehthumbs.db
Desktop.ini

# Python:
*.py[cod]
*.so
*.egg
*.egg-info
dist
build

# My configurations:
db.ini
deploy_key_rsa
```

最后一步就是把 `.gitignore` 也提交到 Git，就完成了！当然检验 .gitignore 的标准是 `git status` 命令是不是说 `working directory clean`。

有些时候，你想添加一个文件到 Git，但发现添加不了，原因是这个文件被 .gitignore 忽略了：

```shell
$ git add App.class
The following paths are ignored by one of your .gitignore files:
App.class
Use -f if you really want to add them.
```

如果你确实想添加该文件，可以用-f强制添加到Git：

```shell
$ git add -f App.class
```

或者你发现，可能是 .gitignore 写得有问题，需要找出来到底哪个规则写错了，可以用 `git check-ignore` 命令检查：

```shell
$ git check-ignore -v App.class
.gitignore:3:*.class    App.class
```

Git 会告诉我们，.gitignore 的第3行规则忽略了该文件，于是我们就可以知道应该修订哪个规则。
