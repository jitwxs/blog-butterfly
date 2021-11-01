---
title: LTP 第六章 开发 IO 阻塞测试集
categories:
  - Linux
  - LTP
abbrlink: 8a00d23c
date: 2017-11-26 19:13:45
related_repos:
  - name: LTP_06
    url: https://github.com/jitwxs/blog-sample/blob/master/LTP/LTP_06
---

在本章中，我们将编写以下几个测试：

- Convert pselect02
- Convert epoll_wait03
- Convert epoll_pwait01
- Convert mmap04
- Convert mmap05
- Convert mmap06
- Add select05

### 6.1 Convert pselect02

#### 6.1.1 重写代码

```shell
ltp/testcases/kernel/syscalls/pselect/
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/pselect$ rm pselect02.c
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/pselect$ vim pselect02.c
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
 *    TEST IDENTIFIER   : peselet02
 *
 *    TEST TITLE        : Basic tests for pselect(2)
 *
 *    TEST CASE TOTAL   : 3
 *
 *    AUTHOR            : jitwxs
 *                        <jitwxs@foxmail.com>
 *
 *    DESCRIPTION
 *        Verify that,
 *        1. pselect() fails with -1 return value and sets errno to EBADF
 *            if a file descriptor that was already closed.
 *        2. pselect() fails with -1 return value and sets errno to EINVAL
 *            if nfds was negative.
 *        3. pselect() fails with -1 return value and sets errno to EINVAL
 *            if the value contained within timeout was invalid.
 *
 **********************************************************/

#include <errno.h>
#include "tst_test.h"

static fd_set read_fds;
static struct timespec time_buf;

static struct tcase {
    int nfds;
    fd_set *readfds;
    struct timespec *timeout;
    int exp_errno;
} tcases[] = {
    {128, &read_fds, NULL, EBADF},
    {-1, NULL, NULL, EINVAL},
    {128, NULL, &time_buf, EINVAL}
};

static void my_test(unsigned int n)
{
    struct tcase *tc = &tcases[n];

    TEST(pselect(tc->nfds, tc->readfds, NULL,
                NULL, tc->timeout, NULL));

    if (TEST_RETURN != -1) {
        tst_res(TFAIL, "pselect() succeeded unexpectedly");
        return;
    }

    if (tc->exp_errno == TEST_ERRNO)
        tst_res(TPASS | TTERRNO, "pselect() failed as expected");
    else
        tst_res(TFAIL | TTERRNO,
                "pselect() failed unexpectedly; expected: %d - %s",
                tc->exp_errno, strerror(tc->exp_errno));
}

static void setup(void)
{
    int fd;

    fd = SAFE_OPEN("test_file", O_RDWR | O_CREAT, 0777);

    FD_ZERO(&read_fds);
    FD_SET(fd, &read_fds);

    SAFE_CLOSE(fd);

    time_buf.tv_sec = -1;
    time_buf.tv_nsec = 0;
}

static struct tst_test test = {
    .tcnt = ARRAY_SIZE(tcases),
    .test = my_test,
    .needs_checkpoints = 1,
    .setup = setup,
    .needs_tmpdir = 1
};
```

```shell
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/pselect$ ./pselect02
tst_test.c:977: INFO: Timeout per run is 0h 05m 00s
pselect02.c:65: PASS: pselect() failed as expected: EBADF
pselect02.c:65: PASS: pselect() failed as expected: EINVAL
pselect02.c:65: PASS: pselect() failed as expected: EINVAL

Summary:
passed   3
failed   0
skipped  0
warnings 0
```

#### 6.1.2 提交并校验

```shell
wxs@ubuntu:~/ltp$ git add testcases/kernel/syscalls/pselect/pselect02.c
wxs@ubuntu:~/ltp$ git commit -s -m "Convert pselect02 to new API"
[master e9f68a5] Convert pselect02 to new API
 1 file changed, 93 insertions(+), 118 deletions(-)
 rewrite testcases/kernel/syscalls/pselect/pselect02.c (83%)
```

```shell
wxs@ubuntu:~/ltp$ git format-patch -1
0001-Convert-pselect02-to-new-API.patch
wxs@ubuntu:~/ltp$ ../linux/scripts/checkpatch.pl 0001-Convert-pselect02-to-new-API.patch 
total: 0 errors, 0 warnings, 172 lines checked

0001-Convert-pselect02-to-new-API.patch has no obvious style problems and is ready for submission.
```

### 6.2 Convert epoll_wait03

#### 6.2.1 重写代码

```shell
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/epoll_wait$ rm epoll_wait03.c
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/epoll_wait$ vim epoll_wait03.c
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
 *    TEST IDENTIFIER   : epoll_wait03
 *
 *    TEST TITLE        : Basic tests for epoll_wait(2)
 *
 *    TEST CASE TOTAL   : 5
 *
 *    AUTHOR            : jitwxs
 *                        <jitwxs@foxmail.com>
 *
 *    Description:
 *        1) epoll_wait(2) fails if epfd is not a valid file descriptor
 *        2) epoll_wait(2) fails if epfd is not an epoll file descriptor
 *        3) epoll_wait(2) fails if maxevents is less than zero
 *        4) epoll_wait(2) fails if maxevents is equal to zero
 *        5) epoll_wait(2) fails if the memory area pointed to by events
 *            is not accessible with write permissions.
 *
 *    Expected Result:
 *        1) epoll_wait(2) should return -1 and set errno to EBADF
 *        2) epoll_wait(2) should return -1 and set errno to EINVAL
 *        3) epoll_wait(2) should return -1 and set errno to EINVAL
 *        4) epoll_wait(2) should return -1 and set errno to EINVAL
 *        5) epoll_wait(2) should return -1 and set errno to EFAULT
 *
 **********************************************************/

#include <sys/epoll.h>
#include <sys/mman.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>

#include "tst_test.h"

static int page_size, fds[2], epfd, inv_epfd, bad_epfd = -1;

static struct epoll_event epevs[1] = {
    {.events = EPOLLOUT}
};

static struct epoll_event *ev_rdwr = epevs;
static struct epoll_event *ev_rdonly;

static struct tcase {
    int *epfd;
    struct epoll_event **ev;
    int maxevents;
    int exp_errno;
} tcases[] = {
    {&bad_epfd, &ev_rdwr, 1, EBADF},
    {&inv_epfd, &ev_rdwr, 1,  EINVAL},
    {&epfd, &ev_rdwr, -1,  EINVAL},
    {&epfd, &ev_rdwr, 0,  EINVAL},
    {&epfd, &ev_rdonly, 1,  EFAULT}
};

static void my_test(unsigned int n)
{
    struct tcase *tc = &tcases[n];

    TEST(epoll_wait(*(tc->epfd), *(tc->ev), tc->maxevents, -1));

    if (TEST_RETURN != -1)
        tst_res(TFAIL, "epoll_wait() succeed unexpectedly");
    else
        if (tc->exp_errno == TEST_ERRNO)
            tst_res(TPASS | TTERRNO,
                    "epoll_wait() fails as expected");
        else
            tst_res(TFAIL | TTERRNO,
                    "epoll_wait() fails unexpectedly, expected %d: %s",
                    tc->exp_errno,
                    strerror(tc->exp_errno));
}

static void setup(void)
{
    page_size = getpagesize();

    ev_rdonly = SAFE_MMAP(NULL, page_size, PROT_READ,
            MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);

    SAFE_PIPE(fds);

    epfd = epoll_create(1);
    if (epfd == -1)
        tst_brk(TBROK | TERRNO, "failed to create epoll instance");

    epevs[0].data.fd = fds[1];

    if (epoll_ctl(epfd, EPOLL_CTL_ADD, fds[1], &epevs[0]))
        tst_brk(TBROK | TERRNO, "failed to register epoll target");
}

static void cleanup(void)
{
    if (epfd > 0 && close(epfd))
        tst_res(TWARN | TERRNO, "failed to close epfd");

    if (close(fds[0]))
        tst_res(TWARN | TERRNO, "close(fds[0]) failed");

    if (close(fds[1]))
        tst_res(TWARN | TERRNO, "close(fds[1]) failed");
}

static struct tst_test test = {
    .tcnt = ARRAY_SIZE(tcases),
    .test = my_test,
    .needs_checkpoints = 1,
    .setup = setup,
    .cleanup = cleanup
};
```

```shell
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/epoll_wait$ ./epoll_wait03
tst_test.c:977: INFO: Timeout per run is 0h 05m 00s
epoll_wait03.c:82: PASS: epoll_wait() fails as expected: EBADF
epoll_wait03.c:82: PASS: epoll_wait() fails as expected: EINVAL
epoll_wait03.c:82: PASS: epoll_wait() fails as expected: EINVAL
epoll_wait03.c:82: PASS: epoll_wait() fails as expected: EINVAL
epoll_wait03.c:82: PASS: epoll_wait() fails as expected: EFAULT

Summary:
passed   5
failed   0
skipped  0
warnings 0
```

#### 6.2.2 提交并校验

```shell
wxs@ubuntu:~/ltp$ git add testcases/kernel/syscalls/epoll_wait/epoll_wait03.c
wxs@ubuntu:~/ltp$ git commit -s -m "Convert epoll_wait03 to new API"
[master 9f73675] Convert epoll_wait03 to new API
 1 file changed, 127 insertions(+), 153 deletions(-)
 rewrite testcases/kernel/syscalls/epoll_wait/epoll_wait03.c (77%)
```

```shell
wxs@ubuntu:~/ltp$ git format-patch -2
0001-Convert-pselect02-to-new-API.patch
0002-Convert-epoll_wait03-to-new-API.patch
wxs@ubuntu:~/ltp$ ../linux/scripts/checkpatch.pl 0002-Convert-epoll_wait03-to-new-API.patch 
total: 0 errors, 0 warnings, 221 lines checked

0002-Convert-epoll_wait03-to-new-API.patch has no obvious style problems and is ready for submission.
```

### 6.3 Convert epoll_pwait01

#### 6.3.1 重写代码

```shell
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/epoll_pwait$ rm epoll_pwait01.c
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/epoll_pwait$ vim epoll_pwait01.c
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
 *    TEST IDENTIFIER   : epoll_pwait01
 *
 *    TEST TITLE        : Basic tests for epoll_wait(2)
 *
 *    TEST CASE TOTAL   : 2
 *
 *    AUTHOR            : jitwxs
 *                        <jitwxs@foxmail.com>
 *
 *    DESCRIPTION
 *        Basic test for epoll_pwait(2).
 *        1)  epoll_pwait(2) with sigmask argument allows the caller to
 *            safely wait until either a file descriptor becomes ready
 *            or the timeout expires.
 *        2)  epoll_pwait(2) with NULL sigmask argument fails if
 *            interrupted by a signal handler, epoll_pwait(2) should
 *            return -1 and set errno to EINTR.include <sys/epoll.h>
 **********************************************************/

#include <sys/epoll.h>
#include <sys/types.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>

#include "tst_test.h"

static int epfd, fds[2];
static struct sigaction sa;
static struct epoll_event epevs;
static sigset_t signalset;

static struct tcase {
    sigset_t *sig;
    int ret_val;
    int exp_errno;
} tcases[] = {
    {&signalset, 0, 0},
    {NULL, -1, EINTR}
};

static void sighandler(int sig LTP_ATTRIBUTE_UNUSED)
{
}

static void setup(void)
{
    if (sigemptyset(&signalset) == -1)
        tst_brk(TFAIL | TERRNO, "sigemptyset() failed");

    if (sigaddset(&signalset, SIGUSR1) == -1)
        tst_brk(TFAIL | TERRNO, "sigaddset() failed");

    sa.sa_flags = 0;
    sa.sa_handler = sighandler;
    if (sigemptyset(&sa.sa_mask) == -1)
        tst_brk(TFAIL | TERRNO, "sigemptyset() failed");

    if (sigaction(SIGUSR1, &sa, NULL) == -1)
        tst_brk(TFAIL | TERRNO, "sigaction() failed");

    SAFE_PIPE(fds);

    epfd = epoll_create(1);
    if (epfd == -1)
        tst_brk(TBROK | TERRNO, "failed to create epoll instance");

    epevs.events = EPOLLIN;
    epevs.data.fd = fds[0];

    if (epoll_ctl(epfd, EPOLL_CTL_ADD, fds[0], &epevs) == -1)
        tst_brk(TBROK | TERRNO, "failed to register epoll target");
}

static void cleanup(void)
{
    if (epfd > 0 && close(epfd))
        tst_res(TWARN | TERRNO, "failed to close epfd");

    if (close(fds[0]))
        tst_res(TWARN | TERRNO, "close(fds[0]) failed");

    if (close(fds[1]))
        tst_res(TWARN | TERRNO, "close(fds[1]) failed");
}

static void my_test(unsigned int n)
{
    struct tcase *tc = &tcases[n];

    if (SAFE_FORK() == 0) {
        TST_PROCESS_STATE_WAIT(getppid(), 'S');
        SAFE_KILL(getppid(), SIGUSR1);

        cleanup();
        exit(EXIT_SUCCESS);
    }

    TEST(epoll_pwait(epfd, &epevs, 1, 100, tc->sig));

    if (tc->ret_val == TEST_RETURN)
        if (TEST_RETURN == 0 || tc->exp_errno == TEST_ERRNO) {
            tst_res(TPASS, "epoll_pwait() pass");
            return;
        }

    tst_res(TFAIL | TTERRNO, "epoll_pwait() failed");
}

static struct tst_test test = {
    .tcnt = ARRAY_SIZE(tcases),
    .test = my_test,
    .needs_checkpoints = 1,
    .forks_child = 1,
    .setup = setup,
    .cleanup = cleanup,
    .min_kver = "2.6.19"
};
```

```shell
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/epoll_pwait$ ./epoll_pwait01 
tst_test.c:977: INFO: Timeout per run is 0h 05m 00s
epoll_pwait01.c:118: PASS: epoll_pwait() pass
epoll_pwait01.c:118: PASS: epoll_pwait() pass

Summary:
passed   2
failed   0
skipped  0
warnings 0
```

#### 6.3.2 提交并校验

```shell
wxs@ubuntu:~/ltp$ git add testcases/kernel/syscalls/epoll_pwait/epoll_pwait01.c
wxs@ubuntu:~/ltp$ git commit -s -m "Convert epoll_pwait01 to new API"
[master 5cc7e9a] Convert epoll_pwait01 to new API
 1 file changed, 133 insertions(+), 197 deletions(-)
 rewrite testcases/kernel/syscalls/epoll_pwait/epoll_pwait01.c (80%)
```

```shell
wxs@ubuntu:~/ltp$ git format-patch -3
0001-Convert-pselect02-to-new-API.patch
0002-Convert-epoll_wait03-to-new-API.patch
0003-Convert-epoll_pwait01-to-new-API.patch
wxs@ubuntu:~/ltp$ ../linux/scripts/checkpatch.pl 0003-Convert-epoll_pwait01-to-new-API.patch 
total: 0 errors, 0 warnings, 278 lines checked

0003-Convert-epoll_pwait01-to-new-API.patch has no obvious style problems and is ready for submission.
```

### 6.4 Convert mmap04

#### 6.4.1 重写代码

```shell
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/mmap$ rm mmap04.c 
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/mmap$ vim mmap04.c
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
 *    TEST IDENTIFIER   : mmap04
 *
 *    TEST TITLE        : Basic tests for mmap(2)
 *
 *    TEST CASE TOTAL   : 1
 *
 *    AUTHOR            : jitwxs
 *                          <jitwxs@foxmail.com>
 *
 *    DESCRIPTION
 *        Call mmap() to map a file creating a mapped region with read/exec access
 *        under the following conditions -
 *        - The prot parameter is set to PROT_READ|PROT_EXEC
 *        - The file descriptor is open for read
 *        - The file being mapped has read and execute permission bit set.
 *        - The minimum file permissions should be 0555.
 *
 *        The call should succeed to map the file creating mapped memory with the
 *        required attributes.
 *
 *    EXPECTED RESULT
 *        mmap() should succeed returning the address of the mapped region,and the
 *        mapped region should contain the contents of the mapped file.
 *
 ********************************************************/

#include <stdlib.h>
#include <errno.h>
#include "tst_test.h"

#define TEMPFILE "tempFile"

static size_t page_sz;
static char *addr;
static char *dummy;
static int fd;

static void setup(void)
{
    char *buf;

    page_sz = getpagesize();

    buf = calloc(page_sz, sizeof(char));
    if (buf == NULL)
        tst_brk(TFAIL, "calloc() failed (tst_buff)");

    memset(buf, 'A', page_sz);

    fd = open(TEMPFILE, O_WRONLY | O_CREAT, 0666);
    if (fd < 0) {
        free(buf);
        tst_brk(TFAIL, "opening %s failed", TEMPFILE);
    }

    if ((size_t)write(fd, buf, page_sz) < page_sz) {
        free(buf);
        tst_brk(TFAIL, "writing to %s failed", TEMPFILE);
    }
    free(buf);

    if (fchmod(fd, 0555) < 0)
        tst_brk(TFAIL, "fchmod of %s failed", TEMPFILE);

    if (close(fd) < 0)
        tst_brk(TFAIL, "closing %s failed", TEMPFILE);

    dummy = calloc(page_sz, sizeof(char));
    if (dummy == NULL)
        tst_brk(TFAIL, "calloc failed (dummy)");

    fd = open(TEMPFILE, O_RDONLY);
    if (fd < 0)
        tst_brk(TFAIL, "opening %s read-only failed", TEMPFILE);
}

static void cleanup(void)
{
    close(fd);
    free(dummy);
}

static void do_test(void)
{
    addr = mmap(0, page_sz, PROT_READ | PROT_EXEC,
                MAP_FILE | MAP_SHARED, fd, 0);
    TEST_ERRNO = errno;

    if (addr == MAP_FAILED)
        tst_res(TFAIL | TERRNO, "mmap of %s failed", TEMPFILE);

    if ((read(fd, dummy, page_sz)) < 0)
        tst_brk(TFAIL, "reading %s failed", TEMPFILE);

    if ((memcmp(dummy, addr, page_sz)))
        tst_res(TFAIL,
                "mapped memory region contains invalid data");
    else
        tst_res(TPASS,
                "Functionality of mmap() successful");

    if ((munmap(addr, page_sz)) != 0)
        tst_brk(TFAIL, "munmapping failed");

    cleanup();
    exit(EXIT_SUCCESS);
}

static struct tst_test test = {
    .test_all = do_test,
    .setup = setup,
    .cleanup = cleanup,
    .needs_tmpdir = 1
};
```

```shell
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/mmap$ ./mmap04
tst_test.c:977: INFO: Timeout per run is 0h 05m 00s
mmap04.c:114: PASS: Functionality of mmap() successful

Summary:
passed   1
failed   0
skipped  0
warnings 0
```

#### 6.4.2 提交并校验

```shell
wxs@ubuntu:~/ltp$ git add testcases/kernel/syscalls/mmap/mmap04.c
wxs@ubuntu:~/ltp$ git commit -s -m "Convert mmap04 to new API"
[master 0988ebb] Convert mmap04 to new API
 1 file changed, 9 insertions(+), 5 deletions(-)
```

```shell
wxs@ubuntu:~/ltp$ git format-patch -4
0001-Convert-epoll_wait03-to-new-API.patch
0002-Convert-epoll_pwait01-to-new-API.patch
0003-Convert-mmap04-to-new-API.patch
0004-Convert-mmap04-to-new-API.patch
wxs@ubuntu:~/ltp$ ../linux/scripts/checkpatch.pl 0004-Convert-mmap04-to-new-API.patch 
total: 0 errors, 0 warnings, 32 lines checked

0004-Convert-mmap04-to-new-API.patch has no obvious style problems and is ready for submission.
```

### 6.5 Convert mmap05

#### 6.5.1 重写代码

```shell
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/mmap$ rm mmap05.c 
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/mmap$ vim mmap05.c
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
 *    TEST IDENTIFIER   : mmap05
 *
 *    TEST TITLE        : Basic tests for mmap(2)
 *
 *    TEST CASE TOTAL   : 1
 *
 *    AUTHOR            : jitwxs
 *                          <jitwxs@foxmail.com>
 *
 *    DESCRIPTION
 *        Call mmap() to map a file creating mapped memory with no access
 *        under the following conditions -
 *        - The prot parameter is set to PROT_NONE
 *        - The file descriptor is open for read(any mode other than write)
 *        - The minimum file permissions should be 0444.
 *
 *        The call should succeed to map the file creating mapped memory
 *        with the required attributes.
 *
 *    EXPECTED RESULT
 *        mmap() should succeed returning the address of the mapped region,
 *        and an attempt to access the contents of the mapped region
 *        should give rise to the signal SIGSEGV.
 *
 ********************************************************/

#include <stdlib.h>
#include <errno.h>
#include <setjmp.h>
#include "tst_test.h"
#include <signal.h>

#define TEMPFILE "tempFile"

static size_t page_sz;
static volatile char *addr;
static volatile int pass = 0;
static int fd;
static sigjmp_buf env;

static void sig_handler(int sig)
{
    if (sig == SIGSEGV) {
        pass = 1;
        siglongjmp(env, 1);
    } else
        tst_brk(TBROK, "received an unexpected signal: %d", sig);
}

static void setup(void)
{
    char *buf;

    signal(SIGSEGV, sig_handler);

    page_sz = getpagesize();

    buf = calloc(page_sz, sizeof(char));
    if (buf == NULL)
        tst_brk(TFAIL, "calloc() failed (tst_buff)");

    memset(buf, 'A', page_sz);

    fd = open(TEMPFILE, O_WRONLY | O_CREAT, 0666);
    if (fd < 0) {
        free(buf);
        tst_brk(TFAIL, "opening %s failed", TEMPFILE);
    }

    if ((size_t)write(fd, buf, page_sz) != page_sz) {
        free(buf);
        tst_brk(TFAIL, "writing to %s failed", TEMPFILE);
    }
    free(buf);

    if (fchmod(fd, 0444) < 0)
        tst_brk(TFAIL | TERRNO, "fchmod of %s failed", TEMPFILE);

    if (close(fd) < 0)
        tst_brk(TFAIL | TERRNO, "closing %s failed", TEMPFILE);

    fd = open(TEMPFILE, O_RDONLY);
    if (fd < 0)
        tst_brk(TFAIL | TERRNO,
                "opening %s read-only failed", TEMPFILE);
}

static void cleanup(void)
{
    close(fd);
}

static void do_test(void)
{
    char file_content;

    addr = mmap(0, page_sz, PROT_NONE,
            MAP_FILE | MAP_SHARED, fd, 0);
    TEST_ERRNO = errno;

    if (addr == MAP_FAILED)
        tst_res(TFAIL | TERRNO, "mmap of %s failed", TEMPFILE);

    if (sigsetjmp(env, 1) == 0)
        file_content = addr[0];

    if (pass)
        tst_res(TPASS, "Got SIGSEGV as expected");
    else
        tst_res(TFAIL,
                "Mapped memory region with NO access is accessible");

    if (munmap((void *)addr, page_sz) != 0)
        tst_brk(TFAIL, "munmapping failed");

    pass = 0;
    exit(EXIT_SUCCESS);
}

static struct tst_test test = {
    .test_all = do_test,
    .setup = setup,
    .cleanup = cleanup,
    .needs_tmpdir = 1
};
```

```shell
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/mmap$ ./mmap05 
tst_test.c:977: INFO: Timeout per run is 0h 05m 00s
mmap05.c:122: PASS: Got SIGSEGV as expected

Summary:
passed   1
failed   0
skipped  0
warnings 0
```

#### 6.5.2 提交并校验

```shell
wxs@ubuntu:~/ltp$ git add testcases/kernel/syscalls/mmap/mmap05.c
wxs@ubuntu:~/ltp$ git commit -s -m "Convert mmap05 to new API"
[master 4d1ba1a] Convert mmap05 to new API
 1 file changed, 8 insertions(+), 7 deletions(-)
```

```shell
wxs@ubuntu:~/ltp$ git format-patch -5
0001-Convert-epoll_pwait01-to-new-API.patch
0002-Convert-mmap04-to-new-API.patch
0003-Convert-mmap04-to-new-API.patch
0004-Convert-mmap05-to-new-API.patch
0005-Convert-mmap05-to-new-API.patch
wxs@ubuntu:~/ltp$ ../linux/scripts/checkpatch.pl 0005-Convert-mmap05-to-new-API.patch 
total: 0 errors, 0 warnings, 34 lines checked

0005-Convert-mmap05-to-new-API.patch has no obvious style problems and is ready for submission.
```

### 6.6 Convert mmap06

#### 6.6.1 重写代码

```shell
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/mmap$ rm mmap06.c
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/mmap$ vim mmap06.c
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
 *    TEST IDENTIFIER   : mmap06
 *
 *    TEST TITLE        : Basic tests for mmap(2)
 *
 *    TEST CASE TOTAL   : 1
 *
 *    AUTHOR            : jitwxs
 *                          <jitwxs@foxmail.com>
 *
 *    DESCRIPTION
 *        Call mmap() to map a file creating a mapped region with read
 *        access under the following conditions -
 *        - The prot parameter is set to PROT_READ
 *        - The file descriptor is open for writing.
 *
 *        The call should fail to map the file.
 *
 *    EXPECTED RESULT
 *        mmap() should fail returning -1 and errno should get set to
 *        EACCES.
 *
 **********************************************************/

#include <stdlib.h>
#include <errno.h>
#include "tst_test.h"

#define TEMPFILE "tempFile"

static size_t page_sz;
static char *addr;
static int fd;

static void setup(void)
{
    char *buf;

    page_sz = getpagesize();

    buf = calloc(page_sz, sizeof(char));
    if (buf == NULL)
        tst_brk(TFAIL, "calloc() failed (tst_buff)");

    memset(buf, 'A', page_sz);

    fd = open(TEMPFILE, O_WRONLY | O_CREAT, 0666);
    if (fd < 0) {
        free(buf);
        tst_brk(TFAIL, "opening %s failed", TEMPFILE);
    }

    if ((size_t)write(fd, buf, page_sz) < page_sz) {
        free(buf);
        tst_brk(TFAIL, "writing to %s failed", TEMPFILE);
    }
    free(buf);
}

static void cleanup(void)
{
    close(fd);
}

static void do_test(void)
{
    addr = mmap(0, page_sz, PROT_READ,
                MAP_FILE | MAP_SHARED, fd, 0);
    TEST_ERRNO = errno;

    if (addr != MAP_FAILED) {
        tst_res(TFAIL | TERRNO,
                "mmap() returned invalid value, expected: %p",
                MAP_FAILED);
        if (munmap(addr, page_sz) != 0) {
            tst_res(TBROK, "munmap() failed");
            cleanup();
        }
        exit(EXIT_FAILURE);
    }

    if (TEST_ERRNO == EACCES)
        tst_res(TPASS, "mmap failed with EACCES");
    else
        tst_res(TFAIL | TERRNO,
                "mmap failed with unexpected errno");

    cleanup();
    exit(EXIT_SUCCESS);
}

static struct tst_test test = {
    .test_all = do_test,
    .setup = setup,
    .cleanup = cleanup,
    .needs_tmpdir = 1
};
```

```shell
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/mmap$ ./mmap06
tst_test.c:977: INFO: Timeout per run is 0h 05m 00s
mmap06.c:97: PASS: mmap failed with EACCES

Summary:
passed   1
failed   0
skipped  0
warnings 0
```

#### 6.6.2 提交并校验

```shell
wxs@ubuntu:~/ltp$ git add testcases/kernel/syscalls/mmap/mmap06.c
wxs@ubuntu:~/ltp$ git commit -s -m "Convert mmap06 to new API"
[master ade348a] Convert mmap06 to new API
 1 file changed, 2 insertions(+), 2 deletions(-)
```

```shell
wxs@ubuntu:~/ltp$ git format-patch -6
0001-Convert-mmap04-to-new-API.patch
0002-Convert-mmap04-to-new-API.patch
0003-Convert-mmap05-to-new-API.patch
0004-Convert-mmap05-to-new-API.patch
0005-Convert-mmap06-to-new-API.patch
0006-Convert-mmap06-to-new-API.patch
wxs@ubuntu:~/ltp$ ../linux/scripts/checkpatch.pl 0006-Convert-mmap06-to-new-API.patch 
total: 0 errors, 0 warnings, 16 lines checked

0006-Convert-mmap06-to-new-API.patch has no obvious style problems and is ready for submission.
```

### 6.7 Add selelct05

#### 6.7.1 编写代码

```shell
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/select$ vim select05.c
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
 *    TEST IDENTIFIER   : select05
 *
 *    TEST TITLE        : Basic tests for select(2)
 *
 *    TEST CASE TOTAL   : 1
 *
 *    AUTHOR            : jitwxs
 *                          <jitwxs@foxmail.com>
 *
 *    DESCRIPTION
 *        Check that select() read and write monitoring correctly.
 *
 **********************************************************/

#include <stdlib.h>
#include <errno.h>
#include "tst_test.h"

int fd[2];

static void my_test(void)
{
    struct timeval tv;
    int retval;
    fd_set fs;

    FD_ZERO(&fs);
    FD_SET(fd[0], &fs);
    FD_SET(fd[1], &fs);

    tv.tv_sec = 3;
    tv.tv_usec = 0;

    if (SAFE_FORK() == 0) {
        close(fd[0]);
        write(fd[1], "test", strlen("test"));

        exit(EXIT_SUCCESS);
    }

    retval = select(FD_SETSIZE, &fs, &fs, NULL, &tv);

    SAFE_WAIT(NULL);

    if (retval == -1)
        tst_res(TFAIL | TERRNO, "error: %s", strerror(errno));
    else if (retval && !FD_ISSET(fd[0], &fs) &&
            FD_ISSET(fd[1], &fs))
        tst_res(TPASS, "select() pass");
    else
        tst_res(TFAIL, "select() failed");
}

void setup(void)
{
    if ((pipe(fd)) < 0)
        tst_brk(TBROK | TERRNO, "pipe error");
}

static struct tst_test test = {
    .test_all = my_test,
    .setup = setup,
    .forks_child = 1
};
```

```shell
wxs@ubuntu:~/ltp/testcases/kernel/syscalls/select$ ./select05
tst_test.c:977: INFO: Timeout per run is 0h 05m 00s
select05.c:64: PASS: select() pass

Summary:
passed   1
failed   0
skipped  0
warnings 0
```

#### 6.7.2 提交并校验

```shell
wxs@ubuntu:~/ltp$ git add testcases/kernel/syscalls/select/select05.c
wxs@ubuntu:~/ltp$ git commit -s -m "Add select05 to test selelct(2)"
[master 6d9bd93] Add select05 to test selelct(2)
 1 file changed, 3 insertions(+), 2 deletions(-)
```

```shell
wxs@ubuntu:~/ltp$ git format-patch -7
0001-Convert-mmap04-to-new-API.patch
0002-Convert-mmap05-to-new-API.patch
0003-Convert-mmap05-to-new-API.patch
0004-Convert-mmap06-to-new-API.patch
0005-Convert-mmap06-to-new-API.patch
0006-Add-select05-to-test-selelct-2.patch
0007-Add-select05-to-test-selelct-2.patch
wxs@ubuntu:~/ltp$ ../linux/scripts/checkpatch.pl 0007-Add-select05-to-test-selelct-2.patch 
total: 0 errors, 0 warnings, 17 lines checked

0007-Add-select05-to-test-selelct-2.patch has no obvious style problems and is ready for submission.
```
