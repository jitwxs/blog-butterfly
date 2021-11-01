---
title: LTP 第五章 开发 IO 操作测试集
categories:
  - Linux
  - LTP
abbrlink: df12238b
date: 2017-11-11 12:59:58
related_repos:
  - name: LTP_05
    url: https://github.com/jitwxs/blog-sample/blob/master/LTP/LTP_05
---

在本章中，我们将编写以下几个测试：

- Convert read03
- Convert read04
- Convert close02
- Convert close08
- Convert open04

### 5.1 Convert read03

#### 5.1.1 重写代码

```shell
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/read$ rm read03.c
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/read$ vim read03.c
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
 *
 */
/**********************************************************
 *
 *    TEST IDENTIFIER   : read03
 *
 *    TEST TITLE        : Basic tests for read(2)
 *
 *    TEST CASE TOTAL   : 1
 *
 *    AUTHOR            : jitwxs
 *                        <jitwxs@foxmail.com>
 *
 *    DESCRIPTION
 *      Testcase to check that read() sets errno to EAGAIN.
 *
 *    ALGORITHM
 *        Create a named pipe (fifo), open it in O_NONBLOCK mode, and
 *        attempt to read from it, without writing to it. read() should fail
 *        with EAGAIN.
 *
 **********************************************************/

#include <errno.h>
#include "tst_test.h"

#define FIFONAME "tmpFIFO"

int rfd, wfd;

static void my_test(void)
{
    int c;

    TEST(read(rfd, &c, 1));

    if (TEST_RETURN != -1)
        tst_res(TFAIL, "read() failed");

    if (TEST_ERRNO != EAGAIN)
        tst_res(TFAIL, "read set bad errno, expected EAGAIN, got %d",
                TEST_ERRNO);
    else
        tst_res(TPASS, "read() succeded in setting errno to EAGAIN");
}

static void setup(void)
{
    SAFE_MKFIFO(FIFONAME, 0777);
    rfd = SAFE_OPEN(FIFONAME, O_RDONLY | O_NONBLOCK);
    wfd = SAFE_OPEN(FIFONAME, O_WRONLY | O_NONBLOCK);
}

static void cleanup(void)
{
    if (rfd > 0)
        SAFE_CLOSE(rfd);
    if (wfd > 0)
        SAFE_CLOSE(wfd);
}

static struct tst_test test = {
    .test_all = my_test,
    .setup = setup,
    .cleanup = cleanup,
    .needs_tmpdir = 1
};
```

#### 5.1.2 git 提交

```shell
wxs@ubuntu:~/ltp$ git add testcases/kernel/syscalls/read/read03_new.c
wxs@ubuntu:~/ltp$ git commit -s -m "Convert read03 to new API"
[master 2f7fb39] Convert read03 to new API
 1 file changed, 75 insertions(+)
 create mode 100644 testcases/kernel/syscalls/read/read03_new.c
```

#### 5.1.3 patch 校验

```shell
wxs@ubuntu:~/ltp$ git format-patch -1
0001-Convert-read03-to-new-API.patch
wxs@ubuntu:~/ltp$ ../linux/scripts/checkpatch.pl 0001-Convert-read03-to-new-API.patch 
WARNING: added, moved or deleted file(s), does MAINTAINERS need updating?
#13: 
new file mode 100644

total: 0 errors, 1 warnings, 75 lines checked

NOTE: For some of the reported defects, checkpatch may be able to
      mechanically convert to the typical style using --fix or --fix-inplace.

0001-Convert-read03-to-new-API.patch has style problems, please review.

NOTE: If any of the errors are false positives, please report
      them to the maintainer, see CHECKPATCH in MAINTAINERS.
```

### 5.2 Convert read04

#### 5.2.1 重写代码

```shell
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/read$ rm read04.c
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/read$ vim read04.c
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
 *
 */
/**********************************************************
 *
 *    TEST IDENTIFIER   : read04
 *
 *    TEST TITLE        : Basic tests for read(2)
 *
 *    TEST CASE TOTAL   : 1
 *
 *    AUTHOR            : jitwxs
 *                        <jitwxs@foxmail.com>
 *
 *    DESCRIPTION
 *      Testcase to check if read returns the number of bytes read correctly.
 *
 *    ALGORITHM
 *        Create a file and write some bytes out to it.
 *        Attempt to read more than written.
 *        Check the return count, and the read buffer. The read buffer should be
 *        same as the write buffer.
 *
 **********************************************************/

#include <stdio.h>
#include <errno.h>
#include "tst_test.h"
#include "tst_safe_macros.h"

#define TST_SIZE 27
#define TST_DATA "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
#define FILENAME "tmpfile"

int rfd, wfd;

static void my_test(void)
{
    char prbuf[BUFSIZ];

    SAFE_WRITE(0, wfd, TST_DATA, TST_SIZE);

    rfd = SAFE_OPEN(FILENAME, O_RDONLY, 0700);
    if (rfd == -1)
        tst_res(TBROK | TTERRNO, "can't open for reading");

    TEST(read(rfd, prbuf, BUFSIZ));

    if (TEST_RETURN == -1)
        tst_res(TFAIL, "call failed unexpectedly");

    if (TEST_RETURN != TST_SIZE)
        tst_res(TFAIL, "Bad read count - got %ld - expected %d",
                TEST_RETURN, TST_SIZE);

    if (memcmp(TST_DATA, prbuf, TST_SIZE) != 0)
        tst_res(TFAIL, "read buffer not equal to write buffer");

    tst_res(TPASS, "functionality of read() is correct");
}

static void setup(void)
{
    wfd = SAFE_OPEN(FILENAME, O_WRONLY | O_CREAT, 0700);
    if (wfd == -1)
        tst_res(TBROK | TTERRNO,
                "open %s failed", FILENAME);
}

static void cleanup(void)
{
    if (rfd > 0)
        SAFE_CLOSE(rfd);
    if (wfd > 0)
        SAFE_CLOSE(wfd);
}

static struct tst_test test = {
    .test_all = my_test,
    .setup = setup,
    .cleanup = cleanup,
    .needs_tmpdir = 1
};
```

#### 5.2.2 git 提交

```shell
wxs@ubuntu:~/ltp$ git add testcases/kernel/syscalls/read/read04_new.c
wxs@ubuntu:~/ltp$ git commit -s -m "Convert read04 to new API"
[master b5ef790] Convert read04 to new API
 1 file changed, 88 insertions(+)
 create mode 100644 testcases/kernel/syscalls/read/read04_new.c
```

#### 5.2.3 patch 校验

```shell
wxs@ubuntu:~/ltp$ git format-patch -2
0001-Convert-read03-to-new-API.patch
0002-Convert-read04-to-new-API.patch
wxs@ubuntu:~/ltp$ ../linux/scripts/checkpatch.pl 0002-Convert-read04-to-new-API.patch 
WARNING: added, moved or deleted file(s), does MAINTAINERS need updating?
#13: 
new file mode 100644

total: 0 errors, 1 warnings, 88 lines checked

NOTE: For some of the reported defects, checkpatch may be able to
      mechanically convert to the typical style using --fix or --fix-inplace.

0002-Convert-read04-to-new-API.patch has style problems, please review.

NOTE: If any of the errors are false positives, please report
      them to the maintainer, see CHECKPATCH in MAINTAINERS.
```

### 5.3 Convert close02

#### 5.3.1 重写代码

```shell
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/close$ vim close02_new.c
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
 *
 */
/**********************************************************
 *
 *    TEST IDENTIFIER   : close02
 *
 *    TEST TITLE        : Basic tests for close(2)
 *
 *    TEST CASE TOTAL   : 1
 *
 *    AUTHOR            : jitwxs
 *                        <jitwxs@foxmail.com>
 *
 *    DESCRIPTION
 *      Check that an invalid file descriptor returns EBADF
 *
 **********************************************************/

#include <errno.h>
#include "tst_test.h"

static void my_test(void)
{
    TEST(close(-1));

    if (TEST_RETURN != -1)
        tst_res(TFAIL, "Closed a non existent fildes");
    else {
        if (TEST_ERRNO != EBADF)
            tst_res(TFAIL, "close() FAILED dis EBADF, got %d",
                    errno);
        else
            tst_res(TPASS, "call returned EBADF");
    }
}

static struct tst_test test = {
    .test_all = my_test
};
```

#### 5.3.2 git 提交

```shell
wxs@ubuntu:~/ltp$ git add testcases/kernel/syscalls/close/close02_new.c
wxs@ubuntu:~/ltp$ git commit -s -m "Convert close02 to new API"
[master 68c6b1f] Convert close02 to new API
 1 file changed, 51 insertions(+)
 create mode 100644 testcases/kernel/syscalls/close/close02_new.c
```

#### 5.3.3 patch 校验

```shell
wxs@ubuntu:~/ltp$ git format-patch -3
0001-Convert-read03-to-new-API.patch
0002-Convert-read04-to-new-API.patch
0003-Convert-close02-to-new-API.patch
wxs@ubuntu:~/ltp$ ../linux/scripts/checkpatch.pl 0003-Convert-close02-to-new-API.patch 
WARNING: added, moved or deleted file(s), does MAINTAINERS need updating?
#13: 
new file mode 100644

total: 0 errors, 1 warnings, 51 lines checked

NOTE: For some of the reported defects, checkpatch may be able to
      mechanically convert to the typical style using --fix or --fix-inplace.

0003-Convert-close02-to-new-API.patch has style problems, please review.

NOTE: If any of the errors are false positives, please report
      them to the maintainer, see CHECKPATCH in MAINTAINERS.
```

### 5.4 Convert close08

#### 5.4.1 重写代码

```shell
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/close$ vim close08.c
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/close$ rm close08.c
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
 *
 */
/**********************************************************
 *
 *    TEST IDENTIFIER   : close08
 *
 *    TEST TITLE        : Basic tests for close(2)
 *
 *    TEST CASE TOTAL   : 1
 *
 *    AUTHOR            : jitwxs
 *                        <jitwxs@foxmail.com>
 *
 *    DESCRIPTION
 *      It should/will be extended when full functional tests are written
 *        for close(2).
 *
 **********************************************************/

#include <errno.h>
#include "tst_test.h"

#define FILENAME "tmpFile"

int fd;

static void my_test(void)
{
    TEST(close(fd));

    if (TEST_RETURN == -1)
        tst_res(TFAIL | TTERRNO,
                "close(%s) failed", FILENAME);
    else
        tst_res(TPASS, "close(%s) returned %ld",
                FILENAME, TEST_RETURN);
}

static void setup(void)
{
    fd = SAFE_OPEN(FILENAME, O_RDWR | O_CREAT, 0700);
    if (fd == -1)
        tst_res(TBROK | TTERRNO,
                "open(%s, O_RDWR|O_CREAT,0700) failed",
                FILENAME);
}

static struct tst_test test = {
    .test_all = my_test,
    .setup = setup,
    .needs_tmpdir = 1
};
```

#### 5.4.2 git 提交

```shell
wxs@ubuntu:~/ltp$ git add testcases/kernel/syscalls/close/close08_new.c
wxs@ubuntu:~/ltp$ git commit -s -m "Convert close08 to new API"
[master 3e05d89] Convert close08 to new API
 1 file changed, 63 insertions(+)
 create mode 100644 testcases/kernel/syscalls/close/close08_new.c
```

#### 5.4.3 patch 校验

```shell
wxs@ubuntu:~/ltp$ git format-patch -4
0001-Convert-read03-to-new-API.patch
0002-Convert-read04-to-new-API.patch
0003-Convert-close02-to-new-API.patch
0004-Convert-close08-to-new-API.patch
wxs@ubuntu:~/ltp$ ../linux/scripts/checkpatch.pl 0004-Convert-close08-to-new-API.patch 
WARNING: added, moved or deleted file(s), does MAINTAINERS need updating?
#13: 
new file mode 100644

total: 0 errors, 1 warnings, 63 lines checked

NOTE: For some of the reported defects, checkpatch may be able to
      mechanically convert to the typical style using --fix or --fix-inplace.

0004-Convert-close08-to-new-API.patch has style problems, please review.

NOTE: If any of the errors are false positives, please report
      them to the maintainer, see CHECKPATCH in MAINTAINERS.
```

### 5.5 Convert open04

#### 5.5.1 重写代码

```shell
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/open$ vim open04_new.c
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
 *
 */
/**********************************************************
 *
 *    TEST IDENTIFIER   : open04
 *
 *    TEST TITLE        : Basic tests for open(2)
 *
 *    TEST CASE TOTAL   : 1
 *
 *    AUTHOR            : jitwxs
 *                        <jitwxs@foxmail.com>
 *
 *    DESCRIPTION
 *        Testcase to check that open(2) sets EMFILE if a process opens files
 *        more than its descriptor size
 *
 *    ALGORITHM
 *        First get the file descriptor table size which is set for a process.
 *        Use open(2) for creating files till the descriptor table becomes full.
 *        These open(2)s should succeed. Finally use open(2) to open another
 *        file. This attempt should fail with EMFILE.
 *
 **********************************************************/

#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <sys/resource.h>
#include <fcntl.h>
#include <unistd.h>
#include "tst_test.h"

#define FILE_A "tmpfileA"
#define FILE_B "tmpfileB"

static int fd1, fd2;

static void my_test(void)
{
    TEST((fd2 = open(FILE_B, O_RDWR | O_CREAT, 0777)));

    if (TEST_RETURN != -1)
        tst_res(TFAIL, "call succeeded unexpectedly");

    if (TEST_ERRNO != EMFILE)
        tst_res(TFAIL, "Expected EMFILE, got %d", TEST_ERRNO);
    else
        tst_res(TPASS, "call returned expected EMFILE error");
}

static void setup(void)
{
    struct rlimit rlim, r;

    fd1 = SAFE_OPEN(FILE_A, O_RDWR | O_CREAT, 0777);
    SAFE_GETRLIMIT(RLIMIT_NOFILE, &rlim);
    r.rlim_cur = fd1;
    r.rlim_max = rlim.rlim_max;
    SAFE_SETRLIMIT(RLIMIT_NOFILE, &r);
}

static void cleanup(void)
{
    if (fd1 > 0)
        SAFE_CLOSE(fd1);
    if (fd2 > 0)
        SAFE_CLOSE(fd2);
}

static struct tst_test test = {
    .test_all = my_test,
    .setup = setup,
    .cleanup = cleanup,
    .needs_tmpdir = 1
};
```

#### 5.5.2 git 提交

```shell
wxs@ubuntu:~/ltp$ git add testcases/kernel/syscalls/open/open04_new.c
wxs@ubuntu:~/ltp$ git commit -s -m "Convert open04 to new API"
[master 3458645] Convert open04 to new API
 1 file changed, 79 insertions(+)
 create mode 100644 testcases/kernel/syscalls/open/open04_new.c
```

#### 5.5.3 patch 校验

```shell
wxs@ubuntu:~/ltp$ git format-patch -5
0001-Convert-read03-to-new-API.patch
0002-Convert-read04-to-new-API.patch
0003-Convert-close02-to-new-API.patch
0004-Convert-close08-to-new-API.patch
0005-Convert-open04-to-new-API.patch
wxs@ubuntu:~/ltp$ ../linux/scripts/checkpatch.pl 0005-Convert-open04-to-new-API.patch 
WARNING: added, moved or deleted file(s), does MAINTAINERS need updating?
#13: 
new file mode 100644

total: 0 errors, 1 warnings, 79 lines checked

NOTE: For some of the reported defects, checkpatch may be able to
      mechanically convert to the typical style using --fix or --fix-inplace.

0005-Convert-open04-to-new-API.patch has style problems, please review.

NOTE: If any of the errors are false positives, please report
      them to the maintainer, see CHECKPATCH in MAINTAINERS.
```
