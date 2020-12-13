---
title: LTP 第三章 开发系统调用测试集
categories:
  - Linux
  - LTP
typora-root-url: ..
abbrlink: ebd92a20
date: 2017-10-10 00:21:58
related_repos:
  - name: LTP_03
    url: https://github.com/jitwxs/blog-sample/blob/master/LTP/LTP_03
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---

在第二章中我们写了 Shell 测试集，在本章，我们将开发基于 C 的系统调用测试集,使用新 API 重写既有的测试 case ,整个流程与上一章基本相同，不再赘述。

在本章中，我们将编写以下几个测试：

- Convert getpagesize01
- Convert getpid01
- Convert unlink05
- Convert getppid02

在开始之前，保证工作区的干净：

```shell
[wxs@bogon ltp]$ git status
# 位于分支 master
无文件要提交，干净的工作区
```

### 3.1 Convert getpagesize01

#### 3.1.1 重写代码

进入 `getpagesize` 目录，删除原有 getpagesize01 文件，并重写 `getpagesize01.c`：

```shell
[wxs@bogon ltp]$ cd testcases/kernel/syscalls/getpagesize/
[wxs@bogon getpagesize]$ ls
getpagesize01  getpagesize01.c  Makefile
[wxs@bogon getpagesize]$ rm getpagesize01*
[wxs@bogon getpagesize]$ vim getpagesize01.c
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
 *    TEST IDENTIFIER   : getpagesize01
 *
 *    TEST TITLE        : Basic tests for getpagesize(2)
 *
 *    TEST CASE TOTAL   : 1
 *
 *    AUTHOR            : jitwxs
 *                        <jitwxs@foxmail.com>
 *
 *    DESCRIPTION
 *      This is a Phase I test for the getpagesize(2) system call.
 *      It is intended to provide a limited exposure of the system call.
 *
 **********************************************************/

#include <errno.h>
#include "tst_test.h"

static void testGetpagesize(void)
{
    int size, ret_sysconf;

    TEST(getpagesize());

    if (TEST_RETURN == -1) {
        tst_res(TFAIL | TTERRNO, "getpagesize failed");
    } else {
        size = getpagesize();
        tst_res(TINFO, "Page Size is %d", size);
        ret_sysconf = sysconf(_SC_PAGESIZE);
        if (size == ret_sysconf)
            tst_res(TPASS,
                    "getpagesize - Page size returned %d",
                    ret_sysconf);
        else
            tst_res(TFAIL,
                    "getpagesize - Page size returned %d",
                    ret_sysconf);
    }
}

static struct tst_test test = {
    .test_all = testGetpagesize
};

```

`make` 并测试正确性：

```shell
[wxs@bogon getpagesize]$ make
make -C "/home/wxs/ltp/lib" -f "/home/wxs/ltp/lib/Makefile" all
make[1]: 进入目录“/home/wxs/ltp/lib”
make[2]: 进入目录“/home/wxs/ltp/lib/newlib_tests”
make[2]: 对“all”无需做任何事。
make[2]: 离开目录“/home/wxs/ltp/lib/newlib_tests”
make[2]: 进入目录“/home/wxs/ltp/lib/tests”
make[2]: 对“all”无需做任何事。
make[2]: 离开目录“/home/wxs/ltp/lib/tests”
make[1]: 离开目录“/home/wxs/ltp/lib”
gcc -g -O2 -g -O2 -fno-strict-aliasing -pipe -Wall -W -Wold-style-definition -D_FORTIFY_SOURCE=2 -I../../../../include -I../../../../include -I../../../../include/old/   -L../../../../lib  getpagesize01.c   -lltp -o getpagesize01
[wxs@bogon getpagesize]$ ./getpagesize01 
tst_test.c:934: INFO: Timeout per run is 0h 05m 00s
getpagesize01.c:44: INFO: Page Size is 4096
getpagesize01.c:49: PASS: getpagesize - Page size returned 4096

Summary:
passed   1
failed   0
skipped  0
warnings 0
```

#### 3.1.2 提交 git

```shell
[wxs@bogon getpagesize]$ cd ~/ltp
[wxs@bogon ltp]$ git status
# 位于分支 mybranch
# 尚未暂存以备提交的变更：
#   （使用 "git add <file>..." 更新要提交的内容）
#   （使用 "git checkout -- <file>..." 丢弃工作区的改动）
#
#   修改：      testcases/kernel/syscalls/getpagesize/getpagesize01.c
#
# 未跟踪的文件:
#   （使用 "git add <file>..." 以包含要提交的内容）
#
#   testcases/kernel/log
修改尚未加入提交（使用 "git add" 和/或 "git commit -a"）
[wxs@bogon ltp]$ git add testcases/kernel/syscalls/getpagesize/getpagesize01.c
[wxs@bogon ltp]$ git commit -s -m "syscalls/getpagesize: Convert to new API for test getpagesize(2)"
[mybranch 3c7efcb] syscalls/getpagesize: Convert to new API for test getpagesize(2)
 1 file changed, 60 insertions(+), 105 deletions(-)
 rewrite testcases/kernel/syscalls/getpagesize/getpagesize01.c (90%)
```

#### 3.1.3 生成 patch 并校验

```shell
[wxs@bogon ltp]$ git format-patch -1
0001-syscalls-getpagesize-Convert-to-new-API-for-test-get.patch
[wxs@bogon ltp]$ ../linux/scripts/checkpatch.pl 0001-syscalls-getpagesize-Convert-to-new-API-for-test-get.patch 
total: 0 errors, 0 warnings, 132 lines checked

0001-syscalls-getpagesize-Convert-to-new-API-for-test-get.patch has no obvious style problems and is ready for submission.
```

### 3.2 Convert getpid01

#### 3.2.1 重写代码

进入 `getpagesize` 目录，删除原有 getpid01 文件，并重写 `getpid01.c`：

```shell
[wxs@bogon ltp]$ cd testcases/kernel/syscalls/getpid/
[wxs@bogon getpid]$ ls
getpid01  getpid01.c  getpid02  getpid02.c  Makefile
[wxs@bogon getpid]$ rm getpid01*
[wxs@bogon getpid]$ vim getpid01.c
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
 *    TEST IDENTIFIER   : getpid01
 *
 *    TEST TITLE        : Basic tests for getpid(2)
 *
 *    TEST CASE TOTAL   : 1
 *
 *    AUTHOR            : jitwxs
 *                        <jitwxs@foxmail.com>
 *
 *    DESCRIPTION
 *      Testcase to check the basic functionality of getpid().
 *
 **********************************************************/

#include <errno.h>
#include "tst_test.h"

static void test_getpid(void)
{
    TEST(getpid());

    if (TEST_RETURN == -1)
        tst_res(TFAIL | TTERRNO, "getpid failed");
    else
        tst_res(TPASS, "getpid returned %ld", TEST_RETURN);
}

static struct tst_test test = {
    .test_all = test_getpid
};
```

`make` 并测试正确性：

```shell
[wxs@bogon getpid]$ make
make -C "/home/wxs/ltp/lib" -f "/home/wxs/ltp/lib/Makefile" all
make[1]: 进入目录“/home/wxs/ltp/lib”
make[2]: 进入目录“/home/wxs/ltp/lib/newlib_tests”
make[2]: 对“all”无需做任何事。
make[2]: 离开目录“/home/wxs/ltp/lib/newlib_tests”
make[2]: 进入目录“/home/wxs/ltp/lib/tests”
make[2]: 对“all”无需做任何事。
make[2]: 离开目录“/home/wxs/ltp/lib/tests”
make[1]: 离开目录“/home/wxs/ltp/lib”
gcc -g -O2 -g -O2 -fno-strict-aliasing -pipe -Wall -W -Wold-style-definition -D_FORTIFY_SOURCE=2 -I../../../../include -I../../../../include -I../../../../include/old/   -L../../../../lib  getpid01.c   -lltp -o getpid01
[wxs@bogon getpid]$ ./getpid01 
tst_test.c:934: INFO: Timeout per run is 0h 05m 00s
getpid01.c:41: PASS: getpid returned 17650

Summary:
passed   1
failed   0
skipped  0
warnings 0
```

#### 3.2.2 提交 git

```shell
[wxs@bogon getpid]$ cd ~/ltp/
[wxs@bogon ltp]$ git status
# 位于分支 mybranch
# 尚未暂存以备提交的变更：
#   （使用 "git add <file>..." 更新要提交的内容）
#   （使用 "git checkout -- <file>..." 丢弃工作区的改动）
#
#   修改：      testcases/kernel/syscalls/getpid/getpid01.c
#
# 未跟踪的文件:
#   （使用 "git add <file>..." 以包含要提交的内容）
#
#   testcases/kernel/log
修改尚未加入提交（使用 "git add" 和/或 "git commit -a"）
[wxs@bogon ltp]$ git add testcases/kernel/syscalls/getpid/getpid01.c
[wxs@bogon ltp]$ git commit -s -m "syscalls/getpid: Convert to new API for test getpid(2)"
[mybranch 6b82447] syscalls/getpid: Convert to new API for test getpid(2)
 1 file changed, 47 insertions(+), 158 deletions(-)
 rewrite testcases/kernel/syscalls/getpid/getpid01.c (83%)
 ```

#### 3.2.3 生成 patch 并校验

```shell
[wxs@bogon ltp]$ git format-patch -2
0001-syscalls-getpagesize-Convert-to-new-API-for-test-get.patch
0002-syscalls-getpid-Convert-to-new-API-for-test-getpid-2.patch
[wxs@bogon ltp]$ ../linux/scripts/checkpatch.pl 0002-syscalls-getpid-Convert-to-new-API-for-test-getpid-2.patch
total: 0 errors, 0 warnings, 173 lines checked

0002-syscalls-getpid-Convert-to-new-API-for-test-getpid-2.patch has no obvious style problems and is ready for submission.
```

### 3.3 Convert unlink05

#### 3.3.1 重写代码

进入 `unlink`目录，删除原有 unlink05 文件，并重写 `unlink05.c`：

```shell
[wxs@bogon ltp]$ cd testcases/kernel/syscalls/unlink
[wxs@bogon unlink]$ ls
Makefile  unlink05.c  unlink06.c  unlink07.c  unlink08.c
unlink05  unlink06    unlink07    unlink08
[wxs@bogon unlink]$ rm unlink05*
[wxs@bogon unlink]$ vim unlink05.c
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
 *    TEST IDENTIFIER   : unlink05
 *
 *    TEST TITLE        : Basic tests for unlink(1)
 *
 *    TEST CASE TOTAL   : 1
 *
 *    AUTHOR            : jitwxs
 *                        <jitwxs@foxmail.com>
 *
 *    DESCRIPTION
 *		It should/will be extended when full functional tests are written for
 *		unlink(2).
 *
 **********************************************************/

#include <errno.h>
#include "tst_test.h"

#define TMPFILE "tmpfile"

static void testUnlink(void)
{
    TEST(unlink(TMPFILE));

    if (TEST_RETURN == -1 || access(TMPFILE, F_OK) == 0)
        tst_res(TFAIL | TTERRNO, "unlink(%s) Failed, errno=%d : %s",
                TMPFILE, TEST_ERRNO, strerror(TEST_ERRNO));
    else
        tst_res(TPASS, "unlink(%s) returned %ld",
                TMPFILE, TEST_RETURN);
}

static void setup(void)
{
    SAFE_TOUCH(TMPFILE, 0777, NULL);
}

static struct tst_test test = {
    .needs_tmpdir = 1,
    .setup = setup,
    .test_all = testUnlink
};
```

`make` 并测试正确性：

```shell
[wxs@bogon unlink]$ make
make -C "/home/wxs/ltp/lib" -f "/home/wxs/ltp/lib/Makefile" all
make[1]: 进入目录“/home/wxs/ltp/lib”
make[2]: 进入目录“/home/wxs/ltp/lib/newlib_tests”
make[2]: 对“all”无需做任何事。
make[2]: 离开目录“/home/wxs/ltp/lib/newlib_tests”
make[2]: 进入目录“/home/wxs/ltp/lib/tests”
make[2]: 对“all”无需做任何事。
make[2]: 离开目录“/home/wxs/ltp/lib/tests”
make[1]: 离开目录“/home/wxs/ltp/lib”
gcc -g -O2 -g -O2 -fno-strict-aliasing -pipe -Wall -W -Wold-style-definition -D_FORTIFY_SOURCE=2 -I../../../../include -I../../../../include -I../../../../include/old/   -L../../../../lib  unlink05.c   -lltp -o unlink05
[wxs@bogon unlink]$ ./unlink05 
tst_test.c:934: INFO: Timeout per run is 0h 05m 00s
unlink05.c:45: PASS: unlink(tmpfile) returned 0

Summary:
passed   1
failed   0
skipped  0
warnings 0
```

#### 3.3.2 提交 git

```shell
[wxs@bogon unlink]$ cd ~/ltp/
[wxs@bogon ltp]$ git status
# 位于分支 mybranch
# 尚未暂存以备提交的变更：
#   （使用 "git add <file>..." 更新要提交的内容）
#   （使用 "git checkout -- <file>..." 丢弃工作区的改动）
#
#   修改：      testcases/kernel/syscalls/unlink/unlink05.c
#
# 未跟踪的文件:
#   （使用 "git add <file>..." 以包含要提交的内容）
#
#   testcases/kernel/log
修改尚未加入提交（使用 "git add" 和/或 "git commit -a"）
[wxs@bogon ltp]$ git add testcases/kernel/syscalls/unlink/unlink05.c
[wxs@bogon ltp]$ git commit -s -m "syscalls/unlink: Convert to new API for test unlink(1)"
[mybranch 860a098] syscalls/unlink: Convert to new API for test unlink(1)
 1 file changed, 57 insertions(+), 209 deletions(-)
 rewrite testcases/kernel/syscalls/unlink/unlink05.c (88%)
```

#### 3.3.3 生成 patch 并校验

```shell
[wxs@bogon ltp]$ git format-patch -3
0001-syscalls-getpagesize-Convert-to-new-API-for-test-get.patch
0002-syscalls-getpid-Convert-to-new-API-for-test-getpid-2.patch
0003-syscalls-unlink-Convert-to-new-API-for-test-unlink-1.patch
[wxs@bogon ltp]$ ../linux/scripts/checkpatch.pl 0003-syscalls-unlink-Convert-to-new-API-for-test-unlink-1.patch 
total: 0 errors, 0 warnings, 235 lines checked

0003-syscalls-unlink-Convert-to-new-API-for-test-unlink-1.patch has no obvious style problems and is ready for submission.
```

### 3.4 Convert getppid02

#### 3.4.1 重写代码

进入 `getppid` 目录，删除原有 getppid02 文件，并重写 `getppid02.c`：

```shell
[wxs@bogon ltp]$ cd testcases/kernel/syscalls/getppid/
[wxs@bogon getppid]$ ls
getppid01  getppid01.c  getppid02  getppid02.c  Makefile
[wxs@bogon getppid]$ rm getppid02*
[wxs@bogon getppid]$ vim getppid02.c
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
 *    TEST IDENTIFIER   : getppid02
 *
 *    TEST TITLE        : Basic tests for getppid(2)
 *
 *    TEST CASE TOTAL   : 1
 *
 *    AUTHOR            : jitwxs
 *                        <jitwxs@foxmail.com>
 *
 *    DESCRIPTION
 *      Testcase to check the basic functionality of the getppid() syscall.
 *
 **********************************************************/

#include <errno.h>
#include <sys/wait.h>
#include "tst_test.h"

static void testGetppid(void)
{
    int status;
    pid_t pid, ppid;

    ppid = getpid();
    pid = SAFE_FORK();

    if (pid == -1)
        tst_res(TFAIL, "fork failed");
    else if (pid == 0) {
        TEST(getppid());
        if (ppid != TEST_RETURN)
            tst_res(TFAIL | TERRNO,
                    "getppid Failed (%ld != %d)",
                    TEST_RETURN, ppid);
        else
            tst_res(TPASS,
                    "getppid Success (%ld == %d)",
                    TEST_RETURN, ppid);
    } else {
        if (wait(&status) == -1)
            tst_res(TBROK | TERRNO,
                    "wait failed");
        if (!WIFEXITED(status) || WEXITSTATUS(status) != 0)
            tst_res(TFAIL,
                    "getppid functionality incorrect");
    }
}

static struct tst_test test = {
    .test_all = testGetppid,
    .forks_child = 1
};

```

`make` 并测试正确性：

```shell
[wxs@bogon getppid]$ make
make -C "/home/wxs/ltp/lib" -f "/home/wxs/ltp/lib/Makefile" all
make[1]: 进入目录“/home/wxs/ltp/lib”
make[2]: 进入目录“/home/wxs/ltp/lib/newlib_tests”
make[2]: 对“all”无需做任何事。
make[2]: 离开目录“/home/wxs/ltp/lib/newlib_tests”
make[2]: 进入目录“/home/wxs/ltp/lib/tests”
make[2]: 对“all”无需做任何事。
make[2]: 离开目录“/home/wxs/ltp/lib/tests”
make[1]: 离开目录“/home/wxs/ltp/lib”
gcc -g -O2 -g -O2 -fno-strict-aliasing -pipe -Wall -W -Wold-style-definition -D_FORTIFY_SOURCE=2 -I../../../../include -I../../../../include -I../../../../include/old/   -L../../../../lib  getppid02.c   -lltp -o getppid02
[wxs@bogon getppid]$ ./getppid02 
tst_test.c:934: INFO: Timeout per run is 0h 05m 00s
getppid02.c:54: PASS: getppid Success (18017 == 18017)

Summary:
passed   1
failed   0
skipped  0
warnings 0
```

#### 3.4.2 提交 git

```shell
[wxs@bogon getppid]$ cd ~/ltp
[wxs@bogon ltp]$ git status
# 位于分支 mybranch
# 尚未暂存以备提交的变更：
#   （使用 "git add <file>..." 更新要提交的内容）
#   （使用 "git checkout -- <file>..." 丢弃工作区的改动）
#
#   修改：      testcases/kernel/syscalls/getppid/getppid02.c
#
# 未跟踪的文件:
#   （使用 "git add <file>..." 以包含要提交的内容）
#
#   testcases/kernel/log
修改尚未加入提交（使用 "git add" 和/或 "git commit -a"）
[wxs@bogon ltp]$ git add testcases/kernel/syscalls/getppid/getppid02.c
[wxs@bogon ltp]$ git commit -s -m "syscalls/getppid: Convert to new API for test getppid(2)"
[mybranch 80d5955] syscalls/getppid: Convert to new API for test getppid(2)
 1 file changed, 69 insertions(+), 107 deletions(-)
 rewrite testcases/kernel/syscalls/getppid/getppid02.c (95%)
```

#### 3.4.3 生成 patch 并校验

```shell
[wxs@bogon ltp]$ git format-patch -4
0001-syscalls-getpagesize-Convert-to-new-API-for-test-get.patch
0002-syscalls-getpid-Convert-to-new-API-for-test-getpid-2.patch
0003-syscalls-unlink-Convert-to-new-API-for-test-unlink-1.patch
0004-syscalls-getppid-Convert-to-new-API-for-test-getppid.patch
[wxs@bogon ltp]$ ../linux/scripts/checkpatch.pl 0004-syscalls-getppid-Convert-to-new-API-for-test-getppid.patch 
total: 0 errors, 0 warnings, 151 lines checked

0004-syscalls-getppid-Convert-to-new-API-for-test-getppid.patch has no obvious style problems and is ready for submission.
```
