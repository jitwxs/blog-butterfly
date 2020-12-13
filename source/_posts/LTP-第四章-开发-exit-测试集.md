---
title: LTP 第四章 开发_exit()测试集
categories:
  - Linux
  - LTP
typora-root-url: ..
abbrlink: a375e1a1
date: 2017-10-27 19:08:58
related_repos:
  - name: LTP_04
    url: https://github.com/jitwxs/blog-sample/blob/master/LTP/LTP_04
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---

本章我们将结合前三章所学知识，开发一个完整的新规测试用例，在开始项目之前，请保证项目源码包的干净。

```shell
[wxs@bogon ltp]$ git branch
  master
* mybranch
[wxs@bogon ltp]$ git status
# 位于分支 mybranch
无文件要提交，干净的工作区
```

本章以编写 `_exit()` 函数为例，首先我们查看 `_exit()` 函数的 man-page，根据描述我们抽取出五个测试点，在本章中将依次实现它们。

![test](/images/posts/20171027182917513.png)

### 4.1 Test point 1

#### 4.1.1 测试思路

进程调用_exit()后应该立即终止，因此可以测试该进程是否仍然存在。

![testpoint1](/images/posts/20171027183517309.png)

#### 4.1.2 编写代码

在编写之前，首先我们要知道我们编写的_exit()测试用例属于系统调用，因此我们在ltp的系统调用文件夹中新建_exit文件夹，用于存放本章代码。

```shell
[wxs@bogon ltp]$ cd testcases/kernel/syscalls/
[wxs@bogon syscalls]$ mkdir _exit
[wxs@bogon syscalls]$ cd _exit
[wxs@bogon _exit]$ pwd
/home/wxs/ltp/testcases/kernel/syscalls/_exit
```

编写Makefile文件，用于编译本章的所有测试用例：

```shell
[wxs@bogon _exit]$ vim Makefile 
```

```c
#
# Author : jitwxs <jitwxs@foxmail.com>
#
# This program is free software; you can redistribute it and/or
# modify it under the terms of the GNU General Public License as
# published by the Free Software Foundation; either version 2 of
# the License, or (at your option) any later version.
#
# This program is distributed in the hope that it would be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#

top_srcdir        ?= ../../../..

include $(top_srcdir)/include/mk/testcases.mk

include $(top_srcdir)/include/mk/generic_leaf_target.mk
```

根据测试思路，编写_exit01.c文件：

```shell
[wxs@bogon _exit]$ vim _exit01.c
```

```c
/*
 * This program is free software; you can redistribute it and/or modify it
 * under the terms of version 2 of the GNU General Public License as
 * published by the Free Software Foundation.
 *
 * This program is distributed in the hope that it would be useful, but
 * WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
 *
 * You should have received a copy of the GNU General Public License along
 * with this program; if not, write the Free Software Foundation, Inc.
 */
/**********************************************************
 *
 *    TEST IDENTIFIER   : exit01
 *
 *    TEST TITLE        : Basic tests for _exit(2)
 *
 *    TEST CASE TOTAL   : 1
 *
 *    AUTHOR            : jitwxs
 *                        <jitwxs@foxmail.com>
 *
 *    DESCRIPTION
 *        After the process calls _exit() to test whether the process still
 *        exists.
 *
 **********************************************************/

#include <errno.h>
#include <unistd.h>
#include <sys/types.h>
#include <signal.h>
#include "tst_test.h"

static void my_test(void)
{
    pid_t pid;
    int tmp;

    pid = SAFE_FORK();
    if (!pid) {
        _exit(0);
    } else {
        SAFE_WAITPID(pid, NULL, 0);
        tmp = kill(pid, 0);
        if (tmp != -1)
            tst_res(TFAIL | TERRNO, "_exit() Failed!");
        else
            tst_res(TPASS, "_exit() Success!");
    }
}

static struct tst_test test = {
    .test_all = my_test,
    .forks_child = 1
};
```

#### 4.1.3 编译运行

```shell
[wxs@bogon _exit]$ make
make -C "/home/wxs/ltp/lib" -f "/home/wxs/ltp/lib/Makefile" all
make[1]: 进入目录“/home/wxs/ltp/lib”
make[2]: 进入目录“/home/wxs/ltp/lib/newlib_tests”
make[2]: 对“all”无需做任何事。
make[2]: 离开目录“/home/wxs/ltp/lib/newlib_tests”
make[2]: 进入目录“/home/wxs/ltp/lib/tests”
make[2]: 对“all”无需做任何事。
make[2]: 离开目录“/home/wxs/ltp/lib/tests”
make[1]: 离开目录“/home/wxs/ltp/lib”
gcc -g -O2 -g -O2 -fno-strict-aliasing -pipe -Wall -W -Wold-style-definition -D_FORTIFY_SOURCE=2 -I../../../../include -I../../../../include -I../../../../include/old/   -L../../../../lib  _exit01.c   -lltp -o _exit01
```

```shell
[wxs@bogon _exit]$ ./_exit01 
tst_test.c:934: INFO: Timeout per run is 0h 05m 00s
_exit01.c:50: PASS: _exit() Success!

Summary:
passed   1
failed   0
skipped  0
warnings 0
```

#### 4.1.4 git 提交

```shell
[wxs@bogon _exit]$ git add Makefile
[wxs@bogon _exit]$ git add _exit01.c
[wxs@bogon _exit]$ git status
# 位于分支 mybranch
# 要提交的变更：
#   （使用 "git reset HEAD <file>..." 撤出暂存区）
#
#	新文件：    Makefile
#	新文件：    _exit01.c
#
# 未跟踪的文件:
#   （使用 "git add <file>..." 以包含要提交的内容）
#
#	_exit01
[wxs@bogon _exit]$ git commit -s -m "syscall/_exit: create _exit syscall and add _exit01 to test _exit(2)"
[mybranch 932d1ad] syscall/_exit: create _exit syscall and add _exit01 to test _exit(2)
 2 files changed, 76 insertions(+)
 create mode 100644 testcases/kernel/syscalls/_exit/Makefile
 create mode 100644 testcases/kernel/syscalls/_exit/_exit01.c

```

因为该测试用例是第一次提交，所以我们提交 `_exit01.c` 和 `Makefile` 文件，后面就不需要提交 `Makefile` 文件了

#### 4.1.5 patch 校验

```shell
[wxs@bogon _exit]$ git format-patch -1
0001-syscall-_exit-create-_exit-syscall-and-add-_exit01-t.patch
[wxs@bogon _exit]$ ~/linux/scripts/checkpatch.pl 0001-syscall-_exit-create-_exit-syscall-and-add-_exit01-t.patch 
WARNING: added, moved or deleted file(s), does MAINTAINERS need updating?
#16: 
new file mode 100644

total: 0 errors, 1 warnings, 76 lines checked

NOTE: For some of the reported defects, checkpatch may be able to
      mechanically convert to the typical style using --fix or --fix-inplace.

0001-syscall-_exit-create-_exit-syscall-and-add-_exit01-t.patch has style problems, please review.

NOTE: If any of the errors are false positives, please report
      them to the maintainer, see CHECKPATCH in MAINTAINERS.
```

### 4.2 Test point 2

#### 4.2.1 测试思路

进程调用_exit()终止后，其子进程成为孤儿进程被init进程领养，因此可以测试该子进程的新父进程PID是否为1 。

![testpoint2](/images/posts/20171027183532309.png)

#### 4.2.2 编写代码

根据测试思路，编写_exit02.c文件：

```shell
[wxs@bogon _exit]$ vim _exit02.c
```

```c
/*
 * This program is free software; you can redistribute it and/or modify it
 * under the terms of version 2 of the GNU General Public License as
 * published by the Free Software Foundation.
 *
 * This program is distributed in the hope that it would be useful, but
 * WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
 *
 * You should have received a copy of the GNU General Public License along
 * with this program; if not, write the Free Software Foundation, Inc.
 */
/**********************************************************
 *
 *    TEST IDENTIFIER   : exit02
 *
 *    TEST TITLE        : Basic tests for _exit(2)
 *
 *    TEST CASE TOTAL   : 1
 *
 *    AUTHOR            : jitwxs
 *                        <jitwxs@foxmail.com>
 *
 *    DESCRIPTION
 *      After the process call _exit () terminates,
 *        tests if the child process's parent process PID is 1.
 *
 **********************************************************/

#include <unistd.h>
#include <sys/types.h>
#include <signal.h>
#include "tst_test.h"

static void my_test(void)
{
    pid_t pid1, pid2;

    pid1 = SAFE_FORK();

    if (pid1 == 0) {
        pid2 = SAFE_FORK();
        if (pid2 != 0)
            _exit(0);
        else {
            usleep(10);
            if (getppid() == 1)
                tst_res(TPASS, "_exit() Success!");
            else
                tst_res(TFAIL | TERRNO, "_exit() Failed!");
            TST_CHECKPOINT_WAKE(0);
            _exit(0);
        }
    } else {
        TST_CHECKPOINT_WAIT(0);
    }
}

static struct tst_test test = {
    .test_all = my_test,
    .forks_child = 1,
    .needs_checkpoints = 1
};
```

#### 4.2.3 编译运行

```shell
[wxs@bogon _exit]$ make
make -C "/home/wxs/ltp/lib" -f "/home/wxs/ltp/lib/Makefile" all
make[1]: 进入目录“/home/wxs/ltp/lib”
make[2]: 进入目录“/home/wxs/ltp/lib/newlib_tests”
make[2]: 对“all”无需做任何事。
make[2]: 离开目录“/home/wxs/ltp/lib/newlib_tests”
make[2]: 进入目录“/home/wxs/ltp/lib/tests”
make[2]: 对“all”无需做任何事。
make[2]: 离开目录“/home/wxs/ltp/lib/tests”
make[1]: 离开目录“/home/wxs/ltp/lib”
gcc -g -O2 -g -O2 -fno-strict-aliasing -pipe -Wall -W -Wold-style-definition -D_FORTIFY_SOURCE=2 -I../../../../include -I../../../../include -I../../../../include/old/   -L../../../../lib  _exit02.c   -lltp -o _exit02
```

```shell
[wxs@bogon _exit]$ ./_exit02
tst_test.c:934: INFO: Timeout per run is 0h 05m 00s
_exit02.c:48: PASS: _exit() Success!

Summary:
passed   1
failed   0
skipped  0
warnings 0
```

#### 4.2.4 git 提交

```shell
[wxs@bogon _exit]$ git add _exit02.c
[wxs@bogon _exit]$ git commit -s -m "syscall/_exit: add _exit02 to test _exit(2)"
[mybranch 4928275] syscall/_exit: add _exit02 to test _exit(2)
 1 file changed, 63 insertions(+)
 create mode 100644 testcases/kernel/syscalls/_exit/_exit02.c
```

#### 4.2.5 patch 校验

```shell
[wxs@bogon _exit]$ git format-patch -2
0001-syscall-_exit-create-_exit-syscall-and-add-_exit01-t.patch
0002-syscall-_exit-add-_exit02-to-test-_exit-2.patch
[wxs@bogon _exit]$ ~/linux/scripts/checkpatch.pl 0002-syscall-_exit-add-_exit02-to-test-_exit-2.patch 
WARNING: added, moved or deleted file(s), does MAINTAINERS need updating?
#13: 
new file mode 100644

total: 0 errors, 1 warnings, 63 lines checked

NOTE: For some of the reported defects, checkpatch may be able to
      mechanically convert to the typical style using --fix or --fix-inplace.

0002-syscall-_exit-add-_exit02-to-test-_exit-2.patch has style problems, please review.

NOTE: If any of the errors are false positives, please report
      them to the maintainer, see CHECKPATCH in MAINTAINERS.
```

### 4.3 Test point 3

#### 4.3.1 测试思路

进程调用_exit()终止后，其父进程被发送了SIGCHLD信号，因此可以测试该父进程是否真的收到了SIGCHLD信号。

![testpoint3](/images/posts/20171027183545724.png)

#### 4.3.2 编写代码

根据测试思路，编写_exit03.c文件：

```shell
[wxs@bogon _exit]$ vim _exit03.c
```

```c
/*
 * This program is free software; you can redistribute it and/or modify it
 * under the terms of version 2 of the GNU General Public License as
 * published by the Free Software Foundation.
 *
 * This program is distributed in the hope that it would be useful, but
 * WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
 *
 * You should have received a copy of the GNU General Public License along
 * with this program; if not, write the Free Software Foundation, Inc.
 */
/**********************************************************
 *
 *    TEST IDENTIFIER   : exit03
 *
 *    TEST TITLE        : Basic tests for _exit(2)
 *
 *    TEST CASE TOTAL   : 1
 *
 *    AUTHOR            : jitwxs
 *                        <jitwxs@foxmail.com>
 *
 *    DESCRIPTION
 *      After the process call _exit () terminates,
 *        it tests if the parent process received the SIGCHLD signal.
 *
 **********************************************************/

#include <sys/types.h>
#include <signal.h>
#include "tst_test.h"
#include <stdio.h>

int global_value = 1;

void check_call(int sig)
{
    if (sig == SIGCHLD)
        global_value = 10;
}

static void my_test(void)
{
    pid_t pid;
    int status;

    pid = SAFE_FORK();

    if (pid == 0)
        _exit(0);
    else {
        signal(SIGCHLD, check_call);
        SAFE_WAIT(&status);
        if (global_value == 1)
            tst_res(TFAIL | TERRNO, "_exit() Failed!");
        else if (global_value == 10)
            tst_res(TPASS, "_exit() Success!");
    }
}

static struct tst_test test = {
    .test_all = my_test,
    .forks_child = 1
};
```

#### 4.3.3 编译运行

```shell
[wxs@bogon _exit]$ make
make -C "/home/wxs/ltp/lib" -f "/home/wxs/ltp/lib/Makefile" all
make[1]: 进入目录“/home/wxs/ltp/lib”
make[2]: 进入目录“/home/wxs/ltp/lib/newlib_tests”
make[2]: 对“all”无需做任何事。
make[2]: 离开目录“/home/wxs/ltp/lib/newlib_tests”
make[2]: 进入目录“/home/wxs/ltp/lib/tests”
make[2]: 对“all”无需做任何事。
make[2]: 离开目录“/home/wxs/ltp/lib/tests”
make[1]: 离开目录“/home/wxs/ltp/lib”
gcc -g -O2 -g -O2 -fno-strict-aliasing -pipe -Wall -W -Wold-style-definition -D_FORTIFY_SOURCE=2 -I../../../../include -I../../../../include -I../../../../include/old/   -L../../../../lib  _exit03.c   -lltp -o _exit03
```

```shell
[wxs@bogon _exit]$ ./_exit03
tst_test.c:934: INFO: Timeout per run is 0h 05m 00s
_exit03.c:58: PASS: _exit() Success!

Summary:
passed   1
failed   0
skipped  0
warnings 0
```

#### 4.3.4 git 提交

```shell
[wxs@bogon _exit]$ git add _exit03.c
[wxs@bogon _exit]$ git commit -s -m "syscall/_exit: add _exit03 to test _exit(2)"
[mybranch eba459a] syscall/_exit: add _exit03 to test _exit(2)
 1 file changed, 65 insertions(+)
 create mode 100644 testcases/kernel/syscalls/_exit/_exit03.c
```

#### 4.3.5 patch 校验

```shell
[wxs@bogon _exit]$ git format-patch -3
0001-syscall-_exit-create-_exit-syscall-and-add-_exit01-t.patch
0002-syscall-_exit-add-_exit02-to-test-_exit-2.patch
0003-syscall-_exit-add-_exit03-to-test-_exit-2.patch
[wxs@bogon _exit]$ ~/linux/scripts/checkpatch.pl 0003-syscall-_exit-add-_exit03-to-test-_exit-2.patch 
WARNING: added, moved or deleted file(s), does MAINTAINERS need updating?
#13: 
new file mode 100644

total: 0 errors, 1 warnings, 65 lines checked

NOTE: For some of the reported defects, checkpatch may be able to
      mechanically convert to the typical style using --fix or --fix-inplace.

0003-syscall-_exit-add-_exit03-to-test-_exit-2.patch has style problems, please review.

NOTE: If any of the errors are false positives, please report
      them to the maintainer, see CHECKPATCH in MAINTAINERS.
```

### 4.4 Test point 4

#### 4.4.1 测试思路

进程调用_exit()时所带参数值，应该作为该进程的退出值，能被其父进程的wait()获得，因此可以测试该值是否匹配。

![testpoint4](/images/posts/20171027183600187.png)

#### 4.4.2 编写代码

根据测试思路，编写_exit04.c文件：

```shell
[wxs@bogon _exit]$ vim _exit04.c
```

```c
/*
 * This program is free software; you can redistribute it and/or modify it
 * under the terms of version 2 of the GNU General Public License as
 * published by the Free Software Foundation.
 *
 * This program is distributed in the hope that it would be useful, but
 * WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
 *
 * You should have received a copy of the GNU General Public License along
 * with this program; if not, write the Free Software Foundation, Inc.
 */
/**********************************************************
 *
 *    TEST IDENTIFIER   : exit04
 *
 *    TEST TITLE        : Basic tests for _exit(2)
 *
 *    TEST CASE TOTAL   : 5
 *
 *    AUTHOR            : jitwxs
 *                        <jitwxs@foxmail.com>
 *
 *    DESCRIPTION
 *      The process calls _exit () to append the parameter value and test
 *        whether the value was obtained from the parent's wait ().
 *
 **********************************************************/

#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <signal.h>
#include "tst_test.h"

static struct tcase {
    const int input;
    const int output;
} tcases[] = {
    {0, 0},
    {10, 10},
    {128, 128},
    {256, 0},
    {288, 32}
};

static void my_test(unsigned int n)
{
    struct tcase *tc = &tcases[n];
    pid_t pid;
    int status;

    if (!SAFE_FORK()) {
        pid = SAFE_FORK();
        if (pid == 0)
            _exit(tc->input);
        else {
            SAFE_WAITPID(pid, &status, 0);
            if (WEXITSTATUS(status) != tc->output)
                tst_res(TFAIL | TERRNO, "_exit() Failed!");
            else
                tst_res(TPASS, "_exit() Success!");
            TST_CHECKPOINT_WAKE(0);
            _exit(0);
        }
    } else {
        TST_CHECKPOINT_WAIT(0);
    }
}

static struct tst_test test = {
    .tcnt = ARRAY_SIZE(tcases),
    .test = my_test,
    .forks_child = 1,
    .needs_checkpoints = 1
};
```

#### 4.4.3 编译运行

```shell
[wxs@bogon _exit]$ make
make -C "/home/wxs/ltp/lib" -f "/home/wxs/ltp/lib/Makefile" all
make[1]: 进入目录“/home/wxs/ltp/lib”
make[2]: 进入目录“/home/wxs/ltp/lib/newlib_tests”
make[2]: 对“all”无需做任何事。
make[2]: 离开目录“/home/wxs/ltp/lib/newlib_tests”
make[2]: 进入目录“/home/wxs/ltp/lib/tests”
make[2]: 对“all”无需做任何事。
make[2]: 离开目录“/home/wxs/ltp/lib/tests”
make[1]: 离开目录“/home/wxs/ltp/lib”
gcc -g -O2 -g -O2 -fno-strict-aliasing -pipe -Wall -W -Wold-style-definition -D_FORTIFY_SOURCE=2 -I../../../../include -I../../../../include -I../../../../include/old/   -L../../../../lib  _exit04.c   -lltp -o _exit04
```

```shell
[wxs@bogon _exit]$ ./_exit04
tst_test.c:934: INFO: Timeout per run is 0h 05m 00s
_exit04.c:62: PASS: _exit() Success!
_exit04.c:62: PASS: _exit() Success!
_exit04.c:62: PASS: _exit() Success!
_exit04.c:62: PASS: _exit() Success!
_exit04.c:62: PASS: _exit() Success!

Summary:
passed   5
failed   0
skipped  0
warnings 0
```

#### 4.4.4 git 提交

```shell
[wxs@bogon _exit]$ git add _exit04.c
[wxs@bogon _exit]$ git commit -s -m "syscall/_exit: add _exit04 to test _exit(2)"
[mybranch c1cdcaf] syscall/_exit: add _exit04 to test _exit(2)
 1 file changed, 76 insertions(+)
 create mode 100644 testcases/kernel/syscalls/_exit/_exit04.c
```

#### 4.4.5 patch 校验

```shell
[wxs@bogon _exit]$ git format-patch -4
0001-syscall-_exit-create-_exit-syscall-and-add-_exit01-t.patch
0002-syscall-_exit-add-_exit02-to-test-_exit-2.patch
0003-syscall-_exit-add-_exit03-to-test-_exit-2.patch
0004-syscall-_exit-add-_exit04-to-test-_exit-2.patch
[wxs@bogon _exit]$ ~/linux/scripts/checkpatch.pl 0004-syscall-_exit-add-_exit04-to-test-_exit-2.patch 
WARNING: added, moved or deleted file(s), does MAINTAINERS need updating?
#13: 
new file mode 100644

total: 0 errors, 1 warnings, 76 lines checked

NOTE: For some of the reported defects, checkpatch may be able to
      mechanically convert to the typical style using --fix or --fix-inplace.

0004-syscall-_exit-add-_exit04-to-test-_exit-2.patch has style problems, please review.

NOTE: If any of the errors are false positives, please report
      them to the maintainer, see CHECKPATCH in MAINTAINERS.
```

### 4.5 Test point 5

#### 4.5.1 测试思路

进程调用_exit()终止时不会像调用exit(3)一样调用atexit(3)或on_exit(3)，因此可以测试是否回调了atexit(3)中注册的函数 。
![testpoint5](/images/posts/20171027183613676.png)

#### 4.5.2 编写代码

根据测试思路，编写_exit05.c文件：

```shell
[wxs@bogon _exit]$ vim _exit05.c
```

```c
/*
 * This program is free software; you can redistribute it and/or modify it
 * under the terms of version 2 of the GNU General Public License as
 * published by the Free Software Foundation.
 *
 * This program is distributed in the hope that it would be useful, but
 * WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
 *
 * You should have received a copy of the GNU General Public License along
 * with this program; if not, write the Free Software Foundation, Inc.
 */
/**********************************************************
 *
 *    TEST IDENTIFIER   : exit05
 *
 *    TEST TITLE        : Basic tests for _exit(2)
 *
 *    TEST CASE TOTAL   : 1
 *
 *    AUTHOR            : jitwxs
 *                        <jitwxs@foxmail.com>
 *
 *    DESCRIPTION
 *      When process call _exit () terminates, atexit (3) or on_exit (3) is
 *        not called as it did for exit (3), so you can test whether the 
 *        function registered in atexit (3) is called back.
 *
 **********************************************************/

#include <stdlib.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <signal.h>
#include "tst_test.h"
#include <stdlib.h>

#define TMPFILE "tmpFile"

void check_call(void)
{
    SAFE_TOUCH(TMPFILE, 0777, NULL);
}

static void my_test(void)
{
    if (!SAFE_FORK()) {
        atexit(check_call);
        _exit(0);
    } else {
        SAFE_WAIT(NULL);
        if (access(TMPFILE, F_OK) != 0)
            tst_res(TPASS, "_exit() Success!");
        else
            tst_res(TFAIL|TERRNO, "_exit() Failed!");
    }
}

static struct tst_test test = {
    .needs_tmpdir = 1,
    .test_all = my_test,
    .forks_child = 1
};
```

#### 4.5.3 编译运行

```shell
[wxs@bogon _exit]$ make
make -C "/home/wxs/ltp/lib" -f "/home/wxs/ltp/lib/Makefile" all
make[1]: 进入目录“/home/wxs/ltp/lib”
make[2]: 进入目录“/home/wxs/ltp/lib/newlib_tests”
make[2]: 对“all”无需做任何事。
make[2]: 离开目录“/home/wxs/ltp/lib/newlib_tests”
make[2]: 进入目录“/home/wxs/ltp/lib/tests”
make[2]: 对“all”无需做任何事。
make[2]: 离开目录“/home/wxs/ltp/lib/tests”
make[1]: 离开目录“/home/wxs/ltp/lib”
gcc -g -O2 -g -O2 -fno-strict-aliasing -pipe -Wall -W -Wold-style-definition -D_FORTIFY_SOURCE=2 -I../../../../include -I../../../../include -I../../../../include/old/   -L../../../../lib  _exit05.c   -lltp -o _exit05
```

```shell
[wxs@bogon _exit]$ ./_exit05 
tst_test.c:934: INFO: Timeout per run is 0h 05m 00s
_exit05.c:52: PASS: _exit() Success!

Summary:
passed   1
failed   0
skipped  0
warnings 0
```

#### 4.5.4 git 提交

```shell
[wxs@bogon _exit]$ git add _exit05.c
[wxs@bogon _exit]$ git commit -s -m "syscall/_exit: add _exit05 to test _exit(2)"
[mybranch 5a42282] syscall/_exit: add _exit05 to test _exit(2)
 1 file changed, 62 insertions(+)
 create mode 100644 testcases/kernel/syscalls/_exit/_exit05.c
```

#### 4.5.5 patch 校验

```shell
[wxs@bogon _exit]$ git format-patch -5
0001-syscall-_exit-create-_exit-syscall-and-add-_exit01-t.patch
0002-syscall-_exit-add-_exit02-to-test-_exit-2.patch
0003-syscall-_exit-add-_exit03-to-test-_exit-2.patch
0004-syscall-_exit-add-_exit04-to-test-_exit-2.patch
0005-syscall-_exit-add-_exit05-to-test-_exit-2.patch
[wxs@bogon _exit]$ ~/linux/scripts/checkpatch.pl 0005-syscall-_exit-add-_exit05-to-test-_exit-2.patch 
WARNING: added, moved or deleted file(s), does MAINTAINERS need updating?
#13: 
new file mode 100644

total: 0 errors, 1 warnings, 62 lines checked

NOTE: For some of the reported defects, checkpatch may be able to
      mechanically convert to the typical style using --fix or --fix-inplace.

0005-syscall-_exit-add-_exit05-to-test-_exit-2.patch has style problems, please review.

NOTE: If any of the errors are false positives, please report
      them to the maintainer, see CHECKPATCH in MAINTAINERS.
```

### 4.6 总结

通过完成上面五个测试点的编写，当前目录内容如下：

```shell
[wxs@bogon _exit]$ pwd
/home/wxs/ltp/testcases/kernel/syscalls/_exit
[wxs@bogon _exit]$ ls
0001-syscall-_exit-create-_exit-syscall-and-add-_exit01-t.patch  _exit02.c
0002-syscall-_exit-add-_exit02-to-test-_exit-2.patch             _exit03
0003-syscall-_exit-add-_exit03-to-test-_exit-2.patch             _exit03.c
0004-syscall-_exit-add-_exit04-to-test-_exit-2.patch             _exit04
0005-syscall-_exit-add-_exit05-to-test-_exit-2.patch             _exit04.c
_exit01                                                          _exit05
_exit01.c                                                        _exit05.c
_exit02                                                          Makefile

```

至此该用例已完全编写完毕。注意，该文件夹中的`*.patch`文件请移除ltp文件夹，不要放在这里，我只是为了演示方便生成在了这里。
