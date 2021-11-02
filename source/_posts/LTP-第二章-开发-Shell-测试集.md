---
title: LTP 第二章 开发 Shell 测试集
categories:
  - Operating System
  - Linux LTP
abbrlink: a89075d9
date: 2017-09-30 00:59:58
related_repos:
  - name: LTP_02
    url: https://github.com/jitwxs/blog-sample/blob/master/LTP/LTP_02
---

### 2.1 准备环境

#### 2.1.1 清理环境

在第一章中我们使用了`git clone`将项目克隆到了本地，并且编写了一个简单的c测试和Shell测试。

在本章开始之前，我们要保证`源码包`项目的干净，即恢复到最开始克隆时的状态,具体步骤参考相关[《Git 教程》](/2121b11b.html)。

**Tip：** 如果你觉的恢复干净对你有困难的话，最简单粗暴的方法就是删掉重新克隆了...

**注：** 清除前一定要注意数据备份，清除后第一章做的修改就不复存在了..

恢复后，项目状态如下:

```shell
[wxs@bogon ltp]$ git status
# 位于分支 master
无文件要提交，干净的工作区
```

更新项目到最新状态：

```shell
[wxs@bogon ltp]$ git pull
Already up-to-date.
```

#### 2.1.2 创建自定义分支

为了不对原项目产生影响，我们使用自定义的分支，并且本章的操作均在此分支中进行。

创建自定义分支`mybranch`，并切换到`mybranch`分支：

```shell
[wxs@bogon ltp]$ git checkout -b mybranch
切换到一个新分支 'mybranch'
[wxs@bogon ltp]$ git branch
  master
* mybranch
```

### 2.2 编写测试集

**注：** 下面的所有文件的头部说明中的`Author信息`为我git的`user.name`和`user.email`，别忘记了修改哦...

#### 2.2.1 测试 echo 命令

在第一章，我们已经简单测试了 echo 的功能，但在这一节我们想做的更多。在本小节中，我们将测试 echo 命令的`空参数`、`-e 参数`、`--help 参数`和`--version 参数`。

首先我们在源码包中创建 `echo` 文件夹，并进入该文件夹：

```shell
[wxs@bogon ltp]$ cd ~/ltp/testcases/commands/
[wxs@bogon commands]$ mkdir echo
[wxs@bogon commands]$ cd echo
```

编写 Shell 文件 `echo_tests.sh`：

```bash
#!/bin/sh
#
# Author : jitwxs <jitwxs@foxmail.com>
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; either version 2 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See
# the GNU General Public License for more details.
#
# Test echo command with some basic options.
#

TST_CNT=4
TST_TESTFUNC=do_test
TST_NEEDS_TMPDIR=1
TST_NEEDS_CMDS="echo"
. tst_test.sh

echo_test()
{
    local echo_opt=$1
    local echo_content=$2

    local echo_cmd="echo $echo_opt $echo_content"

    $echo_cmd > temp 2>&1
    if [ $? -ne 0 ]; then
        grep -q -E "unknown option|invalid option" temp
        if [ $? -eq 0 ]; then
            tst_res TCONF "$echo_cmd not supported."
        else
            tst_res TFAIL "$echo_cmd failed."
        fi
        return
    fi

    line=`wc -l temp | awk '{print $1}'`

    if [ -z $echo_opt ];then
        if [ $line -ne 1 ];then
            tst_res TFAIL "$echo_cmd failed."
            return
        fi
    else
        if [ $echo_opt = "-e" ];then
            if [ $line -ne 2 ];then
                tst_res TFAIL "$echo_cmd failed."
                return
            fi
        fi
    fi

    tst_res TPASS "echo passed with $echo_opt option."
}

do_test()
{
    case $1 in
        1) echo_test "" "helloo\nworld";;
        2) echo_test "-e" "helloo\nworld";;
        3) echo_test "--help";;
        4) echo_test "--version";;
    esac
}

tst_run
```

为 echo.sh 添加可执行权限,并将其拷贝到`安装包`的 `testcases/bin/` 目录下：

```shell
[wxs@bogon echo]$ sudo chmod +x echo_tests.sh
[wxs@bogon echo]$sudo cp echo_tests.sh /opt/ltp/testcases/bin/
```

编写 MakeFile 文件 `Makefile`：

```makefile
#
#    copyright_author: jitwxs  <jitwxs@foxmail.com>
#
#    This program is free software; you can redistribute it and/or modify
#    it under the terms of the GNU General Public License as published by
#    the Free Software Foundation; either version 2 of the License, or
#    (at your option) any later version.
#
#    This program is distributed in the hope that it will be useful,
#    but WITHOUT ANY WARRANTY; without even the implied warranty of
#    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#    GNU General Public License for more details.
#

top_srcdir ?= ../../..

include $(top_srcdir)/include/mk/env_pre.mk

INSTALL_TARGETS :=echo_tests.sh

MAKE_TARGETS :=

include $(top_srcdir)/include/mk/generic_leaf_target.mk
```

此时 echo 文件夹下状态如下：

```shell
[wxs@bogon echo]$ pwd
/home/wxs/ltp/testcases/commands/echo
[wxs@bogon echo]$ ls
echo_tests.sh  Makefile
```

编写测试集 `echo`，进入`安装包`的 `runtest` 目录下，编写测试集 `echo`：

```shell
[wxs@bogon echo]$ cd /opt/ltp/runtest/
[wxs@bogon runtest]$ vim echo
```

echo:

```shell
echo echo_tests.sh
```

进入`安装包`根目录下，并执行测试：

```shell
[wxs@bogon runtest]$ cd /opt/ltp/
[wxs@bogon ltp]$ sudo ./runltp -f echo
```

测试结果如下:

```shell
···
<<<test_start>>>
tag=echo stime=1506687366
cmdline="echo_tests.sh"
contacts=""
analysis=exit
<<<test_output>>>
incrementing stop
 1 TPASS: echo passed with  option.
 2 TPASS: echo passed with -e option.
 3 TPASS: echo passed with --help option.
 4 TPASS: echo passed with --version option.

Summary:
passed   4
failed   0
skipped  0
warnings 0
<<<execution_status>>>
initiation_status="ok"
duration=1 termination_type=exited termination_id=0 corefile=no
cutime=2 cstime=8
<<<test_end>>>
···
```

在该 test 体中，四个测试全部通过，验证了这些参数的正确性。

#### 2.2.2 测试 rm 命令

趁热打铁，我们来编写测试 rm 命令，我们将测试 rm 命令的`空参数`、`-r 参数`、`--help 参数`和`--version 参数`。

进入源码包的 `testcases/commands` 文件夹，创建 rm 文件夹：

```shell
[wxs@bogon ltp]$ cd ~/ltp/testcases/commands/
[wxs@bogon commands]$ mkdir rm
[wxs@bogon commands]$ cd rm
```

编写 Shell 文件 `rm_tests.sh`：

```bash
#!/bin/sh
#
# Author : jitwxs <jitwxs@foxmail.com>
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; either version 2 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See
# the GNU General Public License for more details.
#
# Test rm command with some basic options.

TST_CNT=4
TST_SETUP=setup
TST_TESTFUNC=do_test
TST_NEEDS_TMPDIR=1
TST_NEEDS_CMDS="rm"
. tst_test.sh

setup()
{
    ROD touch "demoFile"
    ROD mkdir "demoDir"
}

rm_test()
{
    local rm_opt=$1
    local rm_file=$2

    local rm_cmd="rm $rm_opt $rm_file"

    eval $rm_cmd > temp 2>&1
    if [ $? -ne 0 ]; then
        grep -q -E "unknown option|invalid option" temp
        if [ $? -eq 0 ]; then
            tst_res TCONF "$rm_cmd not supported."
        else
            tst_res TFAIL "$rm_cmd failed."
        fi
        return
    fi

    if [ -z $rm_opt ];then
        if [ -f $rm_file ];then
            tst_res TFAIL "$rm_cmd failed."
            return
        fi
    else
        if [ $rm_opt = "-r" ];then
            if [ -d $rm_file ];then
                tst_res TFAIL "$rm_cmd failed."
                return
            fi
        fi
    fi

    tst_res TPASS "rm passed with $rm_opt option."
}

do_test()
{
    case $1 in
        1) rm_test "" "demoFile";;
        2) rm_test "-r" "demoDir";;
        3) rm_test "--help";;
        4) rm_test "--version";;
	esac
}

tst_run
```

为 rm_tests.sh 添加可执行权限,并将其拷贝到`安装包`的 `testcases/bin/` 目录下：

```shell
[wxs@bogon rm]$ sudo chmod +x rm_tests.sh
[wxs@bogon rm]$sudo cp rm_tests.sh /opt/ltp/testcases/bin/
```

编写 Makefile 文件 `Makefile`：

```makefile
#
# Author : jitwxs  <jitwxs@foxmail.com>
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; either version 2 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#

top_srcdir ?= ../../..

include $(top_srcdir)/include/mk/env_pre.mk

INSTALL_TARGETS :=rm_tests.sh

MAKE_TARGETS :=

include $(top_srcdir)/include/mk/generic_leaf_target.mk

```

编写相应的测试集 `rm`：

```shell
[wxs@bogon rm]$ cd /opt/ltp/runtest/
[wxs@bogon runtest]$ vim rm
```

rm:

```shell
rm rm_tests.sh
```

进入安装包，测试测试集：

```shell
[wxs@bogon runtest]$ cd /opt/ltp
[wxs@bogon ltp]$ sudo ./runltp -f rm
```

测试结果如下：

```shell
···
<<<test_start>>>
tag=rm stime=1506690175
cmdline="rm_tests.sh"
contacts=""
analysis=exit
<<<test_output>>>
incrementing stop
 1 TPASS: rm passed with  option.
 2 TPASS: rm passed with -r option.
 3 TPASS: rm passed with --help option.
 4 TPASS: rm passed with --version option.

Summary:
passed   4
failed   0
skipped  0
warnings 0
<<<execution_status>>>
initiation_status="ok"
duration=0 termination_type=exited termination_id=0 corefile=no
cutime=3 cstime=6
<<<test_end>>>
···
```

在该 test 体中，四个测试全部通过，验证了这些参数的正确性。

#### 2.2.3 测试 unlink 命令

同样的方法，我们测试 unlink 命令,我们测试 `unlink` 命令的`无参数`、`--help 参数`和`--version 参数`。

进入源码包的 `testcases/commands` 文件夹，创建 unlink 文件夹并进入：

```shell
[wxs@bogon ltp]$ cd ~/ltp/testcases/commands/
[wxs@bogon commands]$ mkdir unlink
[wxs@bogon commands]$ cd unlink
```

编写 Shell 文件 `unlink_tests.sh`：

```bash
#!/bin/sh
#
# Author : jitwxs <jitwxs@foxmail.com>
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; either version 2 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See
# the GNU General Public License for more details.
#
# Test unlink command with some basic options.
#

TST_CNT=3
TST_SETUP=setup
TST_TESTFUNC=do_test
TST_NEEDS_TMPDIR=1
. tst_test.sh

setup()
{
    ROD touch "demo"
}

unlink_test(){
    local unlink_opt=$1
    local unlink_file=$2

    unlink_cmd="unlink $unlink_opt $unlink_file"

    if [ -z $unlink_file ];then
        eval "$unlink_cmd" > temp 2>&1
        if [ $? -ne 0 ];then
            grep -q -E "unknown option|invalid option" temp
            if [ $? -eq 0 ];then
                tst_res TCONF "$unlink_cmd not supposted."
            else
                tst_res TFAIL "$unlink_cmd option failed."
            fi
            return
        fi
    else
        `unlink $unlink_file`
        if [ -f $unlink_file ];then
            tst_res TFAIL "$unlink_cmd failed."
            return
        fi
    fi

    tst_res TPASS "unlink passed with $unlink_opt option."

}

do_test(){
    case $1 in
        1) unlink_test "" "demo" ;;
        2) unlink_test --help ;;
    	3) unlink_test --version ;;
    esac
}

tst_run
```

为 unlink_tests.sh 添加可执行权限,并将其拷贝到`安装包`的 `testcases/bin/` 目录下：

```shell
[wxs@bogon unlink]$ sudo chmod +x unlink_tests.sh
[wxs@bogon unlink]$ sudo cp unlink_tests.sh /opt/ltp/testcases/bin/
```

编写 Makefile 文件 `Makefile`：

```makefile
#
# Author : jitwxs  <jitwxs@foxmail.com>
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; either version 2 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#

top_srcdir ?= ../../..

include $(top_srcdir)/include/mk/env_pre.mk

INSTALL_TARGETS :=unlink_tests.sh

MAKE_TARGETS :=

include $(top_srcdir)/include/mk/generic_leaf_target.mk
```

进入安装包的 runtest 文件夹，编写测试集 `unlink`：

```shell
[wxs@bogon unlink]$ cd /opt/ltp/runtest/
[wxs@bogon runtest]$ sudo vim unlink
```

unlink:

```shell
unlink unlink_tests.sh
```

进入安装包，测试测试集：

```shell
[wxs@bogon runtest]$ cd ..
[wxs@bogon ltp]$ sudo ./runltp -f unlink
```

测试集结果如下：

```shell
···
<<<test_start>>>
tag=unlink stime=1506690699
cmdline="unlink_tests.sh"
contacts=""
analysis=exit
<<<test_output>>>
incrementing stop
 1 TPASS: unlink passed with  option.
 2 TPASS: unlink passed with --help option.
 3 TPASS: unlink passed with --version option.

Summary:
passed   3
failed   0
skipped  0
warnings 0
<<<execution_status>>>
initiation_status="ok"
duration=2 termination_type=exited termination_id=0 corefile=no
cutime=3 cstime=7
<<<test_end>>>
···
```

在该 test 体中，三个测试全部通过，验证了这些参数的正确性。

### 2.3 使用 Git 提交

经过了上面一节，我们已经测试了这三个命令的正确性，现在我们想要将它们提交上去，这里就用到了 git 的操作。

git 提交时不要忘记把之前在安装包中写的三个测试集写到源码包下的 `runtest/commands` 文件中。

这是因为我们写的这些测试都是测试的命令，将其测试集写在 commands 文件中，当安装 ltp 时，就会自动被安装。

首先我们进入`源码包`根目录，但是不要着急直接提交，具体步骤如下：
（1）将测试集追加到 commands文件
（2）git add 每个测试用例的相关文件
（3）git commit -s -m "xxx" （注：-s 将 git 作者信息加入其中）
（4）生成 patch
（5）使用 checkpath 脚本进行 patch 格式检查
（6）确认无误后添加下一个测试用例，回到第一步

#### 2.3.1 提交 echo

将echo测试集的内容追加到`runtest/commands`文件中：

```shell
[wxs@bogon ltp]$ pwd
/home/wxs/ltp
[wxs@bogon ltp]$ cat /opt/ltp/runtest/echo 
echo echo_tests.sh
[wxs@bogon ltp]$ cat /opt/ltp/runtest/echo >> runtest/commands 
```

将echo命令的相关文件加入追踪：

```shell
[wxs@bogon ltp]$ git status
# 位于分支 mybranch
# 尚未暂存以备提交的变更：
#   （使用 "git add <file>..." 更新要提交的内容）
#   （使用 "git checkout -- <file>..." 丢弃工作区的改动）
#
#	修改：      runtest/commands
#
# 未跟踪的文件:
#   （使用 "git add <file>..." 以包含要提交的内容）
#
#	testcases/commands/echo/
#	testcases/commands/rm/
#	testcases/commands/unlink/
修改尚未加入提交（使用 "git add" 和/或 "git commit -a"）
[wxs@bogon ltp]$ git add testcases/commands/echo/
[wxs@bogon ltp]$ git add runtest/commands
```

查看下状态：

```shell
[wxs@bogon ltp]$ git status
# 位于分支 mybranch
# 要提交的变更：
#   （使用 "git reset HEAD <file>..." 撤出暂存区）
#
#	修改：      runtest/commands
#	新文件：    testcases/commands/echo/Makefile
#	新文件：    testcases/commands/echo/echo_tests.sh
#
# 未跟踪的文件:
#   （使用 "git add <file>..." 以包含要提交的内容）
#
#	testcases/commands/rm/
#	testcases/commands/unlink/
```

要提交的没有问题，开始提交，注意提交的说明的格式：

>commands/测试命令名:Add new testcase to test 测试命令名(该命令在man中的号码)

```shell
[wxs@bogon ltp]$ git commit -s -m "commands/echo:Add new testcase to test echo(1)"
[mybranch 96cb7f3] commands/echo:Add new testcase to test echo(1)
 3 files changed, 95 insertions(+)
 create mode 100644 testcases/commands/echo/Makefile
 create mode 100755 testcases/commands/echo/echo_tests.sh
```

生成patch：

```shell
[wxs@bogon ltp]$ git format-patch -1
0001-commands-echo-Add-new-testcase-to-test-echo-1.patch
```

**注**：这里我将 `patch id` 设为1，后面的 `patch id` 依次递增。

开始格式检查，首先我们要下载 [Linux内核](https://github.com/jitwxs/linux)，其实我们用不到这么多，只要其中的`linux 文件夹`即可。

我已经将 Linux 内核中的 linux 文件夹拷贝到了 `home` 目录下，执行 checkpatch 脚本:

```shell
[wxs@bogon ltp]$ ../linux/scripts/checkpatch.pl 0001-commands-echo-Add-new-testcase-to-test-echo-1.patch 
WARNING: added, moved or deleted file(s), does MAINTAINERS need updating?
#17: 
new file mode 100644

total: 0 errors, 1 warnings, 95 lines checked

NOTE: For some of the reported defects, checkpatch may be able to
      mechanically convert to the typical style using --fix or --fix-inplace.

0001-commands-echo-Add-new-testcase-to-test-echo-1.patch has style problems, please review.

NOTE: If any of the errors are false positives, please report
      them to the maintainer, see CHECKPATCH in MAINTAINERS.

```

注意：一定要保证检测结果没有error！至于warning，程序员都懂，忽略即可...

#### 2.3.2 提交 rm

和提交 echo 一样，首先将 rm 的测试集添加入 `commands`:

```shell
[wxs@bogon ltp]$ cat /opt/ltp/runtest/rm
rm rm_tests.sh
[wxs@bogon ltp]$ cat /opt/ltp/runtest/rm >> runtest/commands 
```

git add 相关文件：

```shell
[wxs@bogon ltp]$ git status
# 位于分支 mybranch
# 尚未暂存以备提交的变更：
#   （使用 "git add <file>..." 更新要提交的内容）
#   （使用 "git checkout -- <file>..." 丢弃工作区的改动）
#
#	修改：      runtest/commands
#
# 未跟踪的文件:
#   （使用 "git add <file>..." 以包含要提交的内容）
#
#	testcases/commands/rm/
#	testcases/commands/unlink/
修改尚未加入提交（使用 "git add" 和/或 "git commit -a"）
[wxs@bogon ltp]$ git add testcases/commands/rm/
[wxs@bogon ltp]$ git add runtest/commands
```

然后提交：

```shell
[wxs@bogon ltp]$ git commit -s -m "commands/rm:Add new testcase to test rm(1)"
[mybranch f670ce7] commands/rm:Add new testcase to test rm(1)
 3 files changed, 98 insertions(+)
 create mode 100644 testcases/commands/rm/Makefile
 create mode 100644 testcases/commands/rm/rm_tests.sh
```

生成patch：

```shell
[wxs@bogon ltp]$ git format-patch -2
0001-commands-echo-Add-new-testcase-to-test-echo-1.patch
0002-commands-rm-Add-new-testcase-to-test-rm-1.patch

```

checkpatch验证：

```shell
[wxs@bogon ltp]$ ../linux/scripts/checkpatch.pl 0002-commands-rm-Add-new-testcase-to-test-rm-1.patch 
WARNING: added, moved or deleted file(s), does MAINTAINERS need updating?
#17: 
new file mode 100644

ERROR: trailing whitespace
#52: FILE: testcases/commands/rm/Makefile:24:
+ $

total: 1 errors, 1 warnings, 99 lines checked

NOTE: For some of the reported defects, checkpatch may be able to
      mechanically convert to the typical style using --fix or --fix-inplace.

NOTE: Whitespace errors detected.
      You may wish to use scripts/cleanpatch or scripts/cleanfile

0002-commands-rm-Add-new-testcase-to-test-rm-1.patch has style problems, please review.

NOTE: If any of the errors are false positives, please report
      them to the maintainer, see CHECKPATCH in MAINTAINERS.

```

遗憾的是，出现了最不愿见到的错误。

错误提示我们在 `testcases/commands/rm/Makefile` 的24行尾随一个空白的空格，由此可见这个检测是多么的严苛...

我们回到提交前的状态，使用 git log 查看提交日志：

![commit log](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201709/20170929215046242.png)

我们找到最近一次的正确提交，回滚回去：

```shell
[wxs@bogon ltp]$ git reset 2b4b8e
```

回去之后，去 debug 我们的错误：

![debug](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201709/20170929215349652.png)

删除 24 行的空行，重新 add、commit 和 patch：

```shell
[wxs@bogon ltp]$ git add runtest/commands
[wxs@bogon ltp]$ git add testcases/commands/rm/
[wxs@bogon ltp]$ git commit -s -m "commands/rm:Add new testcase to test rm(1)"
[mybranch f670ce7] commands/rm:Add new testcase to test rm(1)
 3 files changed, 98 insertions(+)
 create mode 100644 testcases/commands/rm/Makefile
 create mode 100644 testcases/commands/rm/rm_tests.sh
[wxs@bogon ltp]$ git format-patch -2
0001-commands-echo-Add-new-testcase-to-test-echo-1.patch
0002-commands-rm-Add-new-testcase-to-test-rm-1.patch
```

重新 checkpatch 检测:

```shell
[wxs@bogon ltp]$ ../linux/scripts/checkpatch.pl 0002-commands-rm-Add-new-testcase-to-test-rm-1.patch 
WARNING: added, moved or deleted file(s), does MAINTAINERS need updating?
#17: 
new file mode 100644

total: 0 errors, 1 warnings, 98 lines checked

NOTE: For some of the reported defects, checkpatch may be able to
      mechanically convert to the typical style using --fix or --fix-inplace.

0002-commands-rm-Add-new-testcase-to-test-rm-1.patch has style problems, please review.

NOTE: If any of the errors are false positives, please report
      them to the maintainer, see CHECKPATCH in MAINTAINERS.
```

问题已经被解决了，我们消灭了一只 bug...

#### 2.3.3 提交 unlink

老套路，直接来了：

```shell
[wxs@bogon ltp]$ cat /opt/ltp/runtest/unlink >> runtest/commands 
[wxs@bogon ltp]$ git status 
# 位于分支 mybranch
# 尚未暂存以备提交的变更：
#   （使用 "git add <file>..." 更新要提交的内容）
#   （使用 "git checkout -- <file>..." 丢弃工作区的改动）
#
#	修改：      runtest/commands
#
# 未跟踪的文件:
#   （使用 "git add <file>..." 以包含要提交的内容）
#
#	testcases/commands/unlink/
修改尚未加入提交（使用 "git add" 和/或 "git commit -a"）
[wxs@bogon ltp]$ git add runtest/commands
[wxs@bogon ltp]$ git add testcases/commands/unlink/
[wxs@bogon ltp]$ git commit -s -m "commands/unlink:Add new testcase to unlink(1)"
[mybranch 99b2823] commands/unlink:Add new testcase to unlink(1)
 3 files changed, 89 insertions(+)
 create mode 100644 testcases/commands/unlink/Makefile
 create mode 100644 testcases/commands/unlink/unlink_tests.sh
[wxs@bogon ltp]$ git format-patch -3
0001-commands-echo-Add-new-testcase-to-test-echo-1.patch
0002-commands-rm-Add-new-testcase-to-test-rm-1.patch
0003-commands-unlink-Add-new-testcase-to-unlink-1.patch
```

checkpatch 检测：

```shell
[wxs@bogon ltp]$ ../linux/scripts/checkpatch.pl 0003-commands-unlink-Add-new-testcase-to-test-unlink-1.patch 
WARNING: added, moved or deleted file(s), does MAINTAINERS need updating?
#17: 
new file mode 100644

total: 0 errors, 1 warnings, 90 lines checked

NOTE: For some of the reported defects, checkpatch may be able to
      mechanically convert to the typical style using --fix or --fix-inplace.

0003-commands-unlink-Add-new-testcase-to-test-unlink-1.patch has style problems, please review.

NOTE: If any of the errors are false positives, please report
      them to the maintainer, see CHECKPATCH in MAINTAINERS.
```

没有问题，检测成功！

### 2.4 发送 patch 到指定邮箱

至此本章内容基本结束了，这里补充一下发送 patch 到指定邮箱的方法：

>git send-email[--in-reply-to=messageID] --to xxx@xxx PATCH名称

注：中括号内容为回复某人的 messageID，如果单纯发送可以省略。

后面的 PATCH ID 可以为多个。

`git send-email` 这个很有可能没有安装，你可以使用`git send-email --help`确认一下。如果显示 send-email 的 man page，那么 send-email 已经安装再你的系统了。否则，你需要安装 `send-email` 命令。

在 CentOS 下，这个安装包的名字是 `git-email` ，因此直接 `yum` 就可以安装了。

现在我想把这三个提交发到我的邮箱可以执行命令：

```shell
git send-email --to jitwxs@foxmail 0001-commands-echo-Add-new-testcase-to-test-echo-1.patch 0002-commands-rm-Add-new-testcase-to-test-rm-1.patch 0003-commands-unlink-Add-new-testcase-to-test-unlink-1.patch
```

没多久我就收到了邮件，但是是从拦截消息中发现的...你没有听错，邮件服务商将其视为了垃圾邮件，所以你只能手动一封封的把它恢复，或者直接将其地址设为白名单。

![post email](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201709/20170929221507396.png)
