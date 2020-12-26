---
title: LTP 第一章 LTP 介绍及内部机制
categories:
  - Linux
  - LTP
abbrlink: 51dc9e04
date: 2017-09-23 03:21:58
related_repos:
  - name: LTP_01
    url: https://github.com/jitwxs/blog-sample/blob/master/LTP/LTP_01
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---

### 1.1 LTP介绍

`LTP(Linux Test Project)`，是基于 GPL 协议的开源社区合作项目。2000 年由 SGI 发起，IBM、OSDL 和 Bull 等公司共同参与，2001年后由 SUSE、富士通、Red Hat、Oracle 共同开发和维护。

通过`功能测试`、`压力测试`和`回归测试`来验证 Linux 系统的可靠性、稳定性和健壮性。整个项目约4000个测试用例，绝大部分用例采用 C 或 Shell。

LTP 不仅测试内核，还测试整体系统环境，对功能执行失败时的返回和处理也进行测试。

#### 1.1.1 功能测试

主要对 `man pages` 中1、8命令和2系统调用所描述的功能进行验证。

#### 1.1.2 回归测试

修改了旧代码后，重新进行测试已确认修改没有引入新的错误或导致其他代码产生错误。

#### 1.1.3 压力测试

测试系统功能特性再大负荷压力下的稳定性和可靠性。

### 1.2 LTP 环境部署

#### 1.2.1 下载 LTP

LTP 项目目前位于 GitHub，项目地址：https://github.com/linux-test-project/ltp 。获取最新版可以执行以下命令：`git clone https://github.com/linux-test-project/ltp.git`。

#### 1.2.2 部署 LTP

首先执行下面命令安装相关软件包（已安装可跳过）：

```shell
#CentOS
sudo yum install autoconf  automake  autotools-dev m4

#Ubuntu
sudo apt-get install autoconf  automake  autotools-dev m4
```

在上节中我将 ltp 项目下载到了 wxs 用户的家目录(/home/wxs)下，如图所示：

```shell
[wxs@bogon ~]$ cd ltp/
[wxs@bogon ltp]$ ls
aclocal.m4      configure.ac  INSTALL           pan                   testcases
autom4te.cache  confLkNw6U    install-sh        README.kernel_config  testscripts
confc20wzw      COPYING       lib               README.md             TODO
config.guess    doc           ltpmenu           runltp                tools
config.log      execltp       m4                runltplite.sh         utils
config.status   execltp.in    Makefile          runtest               ver_linux
config.sub      IDcheck.sh    Makefile.release  scenario_groups       Version
configure       include       missing           scripts               VERSION
```

**进入ltp目录**：`cd ltp`

**生成自动工具**：`make autotools`

**系统环境配置**：`./configure`

**编译**：`make -j$(getconf_NPROCESSORS_ONLN)`

**安装**：`sudo make install`

依次执行以上命令后，LTP 已经被正确安装到你的 Linux 系统中,默认安装位于 `/opt/ltp/`。

```shell
[wxs@bogon ltp]$ cd /opt/ltp/
[wxs@bogon ltp]$ ls
bin         runltp         runtest          share      testscripts  Version
IDcheck.sh  runltplite.sh  scenario_groups  testcases  ver_linux
```

需要注意的是，我们通过 `git clone` 命令下载的位于home目录下的ltp文件夹为 **ltp源码文件夹**，我将在后文简称为 **`源码包`**。

通过执行一系列命令安装到 `/opt` 目录下的 ltp 文件夹为**ltp安装文件夹**，我将在后文简称为**`安装包`**。

### 1.3 目录结构

#### 1.3.1 源码包

LTP 源码包目录结构描述如下：

|名称| 说明 |
|:------------- | :----- |
| INSTALL | LTP安装配置指导文档 |
| README | LTP介绍 |
| CREDITS | 记录对LTP有很大贡献的人 |
| COPYING | GNU公开许可证 |
| ChangeLog | 描述版本变化 |
| ltpmenu | 规划执行LTP的图形化界面接口 |
| Makefile | LTP顶层目录的Makefile，负责编译安装pan、testcases和tools |
| runalltests.sh | 顺序运行全部测试用例并且报告结果的脚本 |
| doc/* | 工程文档包含工具和库函数使用手册，描述各种测试 |
| include/* | 通用的头文件目录 |
| lib/* | 通用的函数目录 |
| testcases/* | 包含在LTP下运行和bin目录下的所有测试用例和链接 |
| testscripts/* | 存放分组的测试脚本 |
| runtest/* | 为自动化测试提供命令列表 |
| pan/* | 测试的驱动装置，具备随机和并行测试的能力 |
| scratch/* | 存放零碎测试 |
| tools/* | 存放自动化测试脚本和辅助工具 |

LTP 测试套件包含以下内容：

```shell
[wxs@bogon ~]$ cd ltp/testcases/
[wxs@bogon testcases]$ ls
commands  demoA  kernel  Makefile  network               realtime
cve       kdump  lib     misc      open_posix_testsuite
```

目录结构描述如下：

|名称| 说明 |
| :------------- | :----- |
| commands | 常用命令测试 |
| kernel | 内核模块及其相关模块 |
| kdump | 内核现崩溃转储测试 |
| network | 网络测试 |
| realtime | 系统实时性测试 |
| open_posix_testsuite  | posix标准测试|
| misc | 崩溃、核心转出、浮点运算等测试|

#### 1.3.2 安装包

LTP安装包目录结构描述如下：

|名称| 说明 |
| :------------- | :----- |
| bin | 存放LTP测试的一些辅助脚本 |
| results | 测试结果默认存储目录 |
| testcases | 测试项集 |
| output | 测试日志默认存储目录 |
| share | 脚本使用说明目录 |
| runtest | 测试驱动(用于链接testscripts内的测试脚本和testcases测试项目)|
| lib | 通用的库函数目录 |

### 1.4 测试框架

#### 1.4.1 整体测试流程

ltp 安装包根目录下的`runltp`脚本是 LTP 自动测试系统的入口，其提供了一系列参数选项，允许用户设定测试环境制定测试集、控制测试结果输出方式和路径等，运行 runltp 会生成指定的测试列表并调用测试驱动`PAN`来开始测试，待执行完毕后根据 `PAN` 返回的结果来生成报告。

`PAN` 是 LTP 的一组测试驱动程序，负责实际测试的执行，根据 runltp 传递的参数和测试列表来依次执行测试，输出执行过程中的详细信息，对每个测试用例的执行结果进行统计，并将整体测试结果返回给 runltp。

![整体测试流程](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201709/20170923015256124.png)

#### 1.4.2 测试用例执行流程

![测试用例执行流程](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201709/20170923015957024.png)

测试结果的输出类型如下：

| Type | Description |
| ------------- | :----- |
| BROK | 程序执行中途发生错误而使测试遭到破坏 |
| CONF | 测试环境不满足而跳过执行 |
| WARN | 测试中途发生异常 |
| INFO | 输出通用测试信息 |
| PASS | 测试成功 |
| FAIL | 测试失败 |

#### 1.4.3 测试库

LTP 目前测试库存在新旧测试库交替的情况，本文均采用**新测试库**的框架，具体的更新说明可以参考更新文档 [An update on the Linux Test Project](https://lwn.net/Articles/708182/)。

![测试库](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201709/20170923020354339.png)

| Old Library | New Library |
| :------------- | :----- |
| 测试用例在执行时调用测试库的API| 在测试库维护的子进程中回调测试用例 |
| setup()中逐一调用API完成测试准备 | setup()中测试属性以结构体变量定义 |
| main()定义在每个测试用例当中 | main()定义在测试库中 |
| cleanup()中不能调用SAFE函数 | cleanup()中允许调用SAFE函数 |

这里以 umount02 为例，比较新旧框架的区别：

旧框架代码：

```c
//Example using the old LTP library
//https://lwn.net/Articles/708250/

#include <errno.h>
#include <sys/mount.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/fcntl.h>
#include <pwd.h>

#include "test.h"
#include "safe_macros.h"

static void setup(void);
static void cleanup(void);

char *TCID = "umount02";

#define DIR_MODE        S_IRWXU | S_IRWXG | S_IROTH | S_IXOTH
#define FILE_MODE        S_IRWXU | S_IRWXG | S_IRWXO
#define MNTPOINT        "mntpoint"

static char long_path[PATH_MAX + 2];
static int mount_flag;
static int fd;

static const char *device;

static struct test_case_t {
    char *err_desc;
    char *mntpoint;
    int exp_errno;
    char *exp_retval;
} testcases[] = {
    {"Already mounted/busy", MNTPOINT, EBUSY, "EBUSY"},
    {"Invalid address space", NULL, EFAULT, "EFAULT"},
    {"Directory not found", "nonexistent", ENOENT, "ENOENT"},
    {"Invalid  device", "./", EINVAL, "EINVAL"},
    {"Pathname too long", long_path, ENAMETOOLONG, "ENAMETOOLONG"}
};

int TST_TOTAL = ARRAY_SIZE(testcases);

int main(int ac, char **av)
{
    int lc, i;

    tst_parse_opts(ac, av, NULL, NULL);

    setup();

    for (lc = 0; TEST_LOOPING(lc); lc++) {
        tst_count = 0;

        for (i = 0; i < TST_TOTAL; ++i) {
            TEST(umount(testcases[i].mntpoint));

            if ((TEST_RETURN == -1) && (TEST_ERRNO == testcases[i].exp_errno)) {
                tst_resm(TPASS, "umount(2) expected failure; "
                            "Got errno - %s : %s",
                            testcases[i].exp_retval,
                            testcases[i].err_desc);
            } else {
                tst_resm(TFAIL, "umount(2) failed to produce "
                            "expected error; %d, errno:%s got %d",
                            testcases[i].exp_errno,
                            testcases[i].exp_retval, TEST_ERRNO);
            }
        }
    }

    cleanup();
    tst_exit();
}

static void setup(void)
{
    const char *fs_type;

    tst_sig(FORK, DEF_HANDLER, cleanup);

    tst_require_root();

    tst_tmpdir();

    fs_type = tst_dev_fs_type();
    device = tst_acquire_device(cleanup);

    if (!device)
        tst_brkm(TCONF, cleanup, "Failed to obtain block device");

    tst_mkfs(cleanup, device, fs_type, NULL, NULL);

    memset(long_path, 'a', PATH_MAX + 1);

    SAFE_MKDIR(cleanup, MNTPOINT, DIR_MODE);

    if (mount(device, MNTPOINT, fs_type, 0, NULL))
        tst_brkm(TBROK | TERRNO, cleanup, "mount() failed");
    
    mount_flag = 1;

    fd = SAFE_OPEN(cleanup, MNTPOINT "/file", O_CREAT | O_RDWR);

    TEST_PAUSE;
}

static void cleanup(void)
{
    if (fd > 0 && close(fd))
        tst_resm(TWARN | TERRNO, "Failed to close file");

    if (mount_flag && tst_umount(MNTPOINT))
        tst_resm(TWARN | TERRNO, "umount() failed");

    if (device)
        tst_release_device(device);

    tst_rmdir();
}
```

新框架代码：

```c
//Example using the new LTP library
//https://lwn.net/Articles/708251/

#include <errno.h>
#include <string.h>
#include <sys/mount.h>
#include "tst_test.h"

#define MNTPOINT        "mntpoint"

static char long_path[PATH_MAX + 2];
static int mount_flag;
static int fd;

static struct tcase {
        const char *err_desc;
        const char *mntpoint;
        int exp_errno;
} tcases[] = {
        {"Already mounted/busy", MNTPOINT, EBUSY},
        {"Invalid address", NULL, EFAULT},
        {"Directory not found", "nonexistent", ENOENT},
        {"Invalid  device", "./", EINVAL},
        {"Pathname too long", long_path, ENAMETOOLONG}
};

static void verify_umount(unsigned int n)
{
        struct tcase *tc = &tcases[n];

        TEST(umount(tc->mntpoint));

        if (TEST_RETURN != -1) {
                tst_res(TFAIL, "umount() succeeds unexpectedly");
                return;
        }

        if (tc->exp_errno != TEST_ERRNO) {
                tst_res(TFAIL | TTERRNO, "umount() should fail with %s",
                        tst_strerrno(tc->exp_errno));
                return;
        }

        tst_res(TPASS | TTERRNO, "umount() fails as expected: %s",
                tc->err_desc);
}

static void setup(void)
{
        memset(long_path, 'a', PATH_MAX + 1);

        SAFE_MKFS(tst_device->dev, tst_device->fs_type, NULL, NULL);
        SAFE_MKDIR(MNTPOINT, 0775);
        SAFE_MOUNT(tst_device->dev, MNTPOINT, tst_device->fs_type, 0, NULL);
        mount_flag = 1;

        fd = SAFE_CREAT(MNTPOINT "/file", 0777);
}

static void cleanup(void)
{
        if (fd > 0 && close(fd))
                tst_res(TWARN | TERRNO, "Failed to close file");

        if (mount_flag)
                tst_umount(MNTPOINT);
}

static struct tst_test test = {
        .tid = "umount02",
        .tcnt = ARRAY_SIZE(tcases),
        .needs_root = 1,
        .needs_tmpdir = 1,
        .needs_device = 1,
        .setup = setup,
        .cleanup = cleanup,
        .test = verify_umount,
};
```

### 1.5 测试执行

#### 1.5.1 整体测试

我们可以测试所有的测试集，直接运行 `runltp` 命令将测试 `ltp/scenario_groups/default` 中的所有测试集，一次测试约 2 ~ 3 小时。

```shell
[wxs@bogon ltp]$ cd /opt/ltp
[wxs@bogon ltp]$ sudo ./runltp
```

当然我们可以只测试某个测试集，测试集可以在 `ltp/runtest/` 下查看。

```shell
[wxs@bogon ltp]$ ls runtest/
admin_tools         ipc                    net_stress.ipsec_udp
can                 kernel_misc            net_stress.multicast
cap_bounds          ltp-aiodio.part1       net_stress.route
commands            ltp-aiodio.part2       net.tcp_cmds
connectors          ltp-aiodio.part3       net.tirpc_tests
containers          ltp-aiodio.part4       network_commands
controllers         ltp-aio-stress.part1   nptl
cpuhotplug          ltp-aio-stress.part2   numa
crashme             ltplite                pipes
cve                 lvm.part1              power_management_tests
dio                 lvm.part2              power_management_tests_exclusive
dma_thread_diotest  math                   pty
fcntl-locktests     mm                     quickhit
filecaps            modules                sched
fs                  net.features           scsi_debug.part1
fs_bind             net.ipv6               securebits
fs_ext4             net.ipv6_lib           smack
fs_perms_simple     net.multicast          stress.part1
fs_readonly         net.nfs                stress.part2
fsx                 net.rpc                stress.part3
hugetlb             net.rpc_tests          syscalls
hyperthreading      net.sctp               syscalls-ipc
ima                 net_stress.appl        timers
input               net_stress.broken_ip   tpm_tools
io                  net_stress.interface   tracing
io_cd               net_stress.ipsec_icmp
io_floppy           net_stress.ipsec_tcp
[wxs@bogon ltp]$ sudo ./runltp -f modules
```

需要注意的是，如果我们测试某个测试集，runltp 需要指定 `-f` 参数。

#### 1.5.2 单独测试

如果我们不想测试某个测试集，只想测试某个单独的测试，可以采用安装包测试或者源码包测试。下面以 access01 为例，讲解单独测试。

##### 1.5.2.1 安装包测试

进入安装包，执行以下命令即可。

```shell
[wxs@bogon ltp]$ cd /opt/ltp/
[wxs@bogon ltp]$ sudo ./runltp -s access01
```

需要注意的是，如果我们测试某个测试，runltp 需要指定 `-s` 参数。

##### 1.5.2.2 源码包测试

进入源码包，找到 access01 的位置，直接执行 `./access01` 即可。

```shell
[wxs@bogon access]$ cd ~/ltp/testcases/kernel/syscalls/access/
[wxs@bogon access]$ sudo ./access01
```

我们看到access01位于testcases目录下，实际上testcases目录下每个文件都是一个完整的可执行程序，可以在编译后的源码路径直接执行。

### 1.6 牛刀小试

本节中牵扯到的项目均位于源码包 testcases 下自建的 **demoA** 文件夹中。测试方法均采用`测试集`测试（C 实现的测试用例可以采用源码测试和测试集测试，Shell 实现的测试用例只可以采用测试集测试）。

```shell
[wxs@bogon ~]$ cd ~/ltp/testcases/
[wxs@bogon testcases]$ mkdir demoA
[wxs@bogon testcases]$ cd demoA/
[wxs@bogon demoA]$ pwd
/home/wxs/ltp/testcases/demoA
```

#### 1.6.1 验证 sqrt() 函数

参考1.3.3节中给出的示例代码，编写测试用例 `sqrt.c`：

```c
#include <errno.h>
#include <string.h>
#include <sys/mount.h>
#include "tst_test.h"

static struct tcase {
	const int input;
	const int output;
} tcases[] = {
        {-1,1},
        {9,3}
};

static void testSqrt(unsigned int n){
	struct tcase *tc = &tcases[n];

	TEST(sqrt(tc->input));
       
	if (TEST_RETURN != tc->output) {
		tst_res(TFAIL, "sqrt() failed");
		return;
	}
	tst_res(TPASS, "sqrt() succeeds");
}

static struct tst_test test = {
        .tid = "testSqrt",
        .tcnt = ARRAY_SIZE(tcases),
        .test = testSqrt,
};

```

参考 testcases 目录下的其他 Makefile 文件，编写 `Makefile`:

```bash
top_srcdir ?= ../..

include $(top_srcdir)/include/mk/testcases.mk

include $(top_srcdir)/include/mk/generic_leaf_target.mk

sqrt:  LDLIBS += -lm
```

需要注意的是，这里的 `top_srcdir` 指的是 ltp 目录，因为 demoA 目录位于 ltp 目录的内两层，所以使用了 `../..`。

还需要注意的是，要想成功使用 sqrt 命令必须附加 `-lm` 参数，在 Makefile 中已经体现这一点。

执行 make 命令，生成可执行文件 `sqrt`：

```shell
[wxs@bogon demoA]$ make
make -C "/home/wxs/ltp/lib" -f "/home/wxs/ltp/lib/Makefile" all
make[1]: 进入目录“/home/wxs/ltp/lib”
make[2]: 进入目录“/home/wxs/ltp/lib/newlib_tests”
make[2]: 对“all”无需做任何事。
make[2]: 离开目录“/home/wxs/ltp/lib/newlib_tests”
make[2]: 进入目录“/home/wxs/ltp/lib/tests”
make[2]: 对“all”无需做任何事。
make[2]: 离开目录“/home/wxs/ltp/lib/tests”
make[1]: 离开目录“/home/wxs/ltp/lib”
gcc -g -O2 -g -O2 -fno-strict-aliasing -pipe -Wall -W -Wold-style-definition -D_FORTIFY_SOURCE=2 -I../../include -I../../include -I../../include/old/   -L../../lib  sqrt.c   -lltp -lm -o sqrt
sqrt.c: 在函数‘testSqrt’中:
sqrt.c:17:2: 警告：隐式声明函数‘sqrt’ [-Wimplicit-function-declaration]
  TEST(sqrt(tc->input));
  ^
In file included from sqrt.c:4:0:
sqrt.c:17:7: 警告：隐式声明与内建函数‘sqrt’不兼容 [默认启用]
  TEST(sqrt(tc->input));
       ^
../../include/tst_test.h:176:17: 附注：in definition of macro ‘TEST’
   TEST_RETURN = SCALL; \
                 ^
```

这里可以直接执行命令 `./sqrt` 来运行这个测试用例，但是对于下节的 shell 测试用例就不能这样做了，本节我都将它写在自定义测试集中。

首先复制可执行文件 `sqrt` 到 `安装包`的 `testcases/bin/` 目录下:

```shell
[wxs@bogon demoA]$ sudo cp sqrt /opt/ltp/testcases/bin/
```

然后进入 `安装包` 的 `runtest` 目录下，编写自定义测试用例集 demoA:

```shell
[wxs@bogon runtest]$ cat demoA 
sqrt sqrt
```

测试用例集中每个测试用例包括两部分：前部分为昵称，后部分为 `testcases/bin/` 目录下的文件名，中间用`空格`分隔。

进入上层目录，执行整体测试命令：

```shell
[wxs@bogon runtest]$ cd ..
[wxs@bogon ltp]$ sudo ./runltp -f demoA
```

执行后程序会输出一大段，我们只需关心最重要的部分。我定义从 `<<<test_start>>>` 到 `<<<test_end>>>` 中间的内容为**`test体`**，后文均以此称呼：

```shell
···
<<<test_start>>>
tag=sqrt stime=1506106767
cmdline="sqrt"
contacts=""
analysis=exit
<<<test_output>>>
tst_test.c:915: INFO: Timeout per run is 0h 05m 00s
incrementing stop
sqrt.c:20: FAIL: sqrt() failed
sqrt.c:23: PASS: sqrt() succeeds

Summary:
passed   1
failed   1
skipped  0
warnings 0
<<<execution_status>>>
initiation_status="ok"
duration=0 termination_type=exited termination_id=1 corefile=no
cutime=0 cstime=0
<<<test_end>>>
···
```

可以看到，两个测试点一个 FAIL，一个 PASS。对照 sqrt.c 文件中给出的测试数据，发现测试没有问题。至此，一个简单的 LTP 测试用例就完成了。

#### 1.6.2 验证 echo 命令

重新回到 demoA 文件夹中，编写 `echo.sh` 文件，代码可以参考源码包中 `testcases/commands` 中其他的测试用例：

```bash
#测试函数执行次数
TST_CNT=1
#测试用例启动函数
TST_TESTFUNC=do_test
. tst_test.sh

echo_test()
{
    local std_in=$1
    local echo_cmd=$(echo $std_in > a.out)
    local echo_res=$std_in
    local cat_res=$(cat a.out)

    if [ $echo_res = $cat_res ]
    then
        tst_res TPASS "$echo_cmd sucessed" 
    else
        tst_res TFAIL "$echo_cmd failed"
    fi
}

do_test()
{
    echo_test "hello\tworld"
}

tst_run
```

为 echo.sh 添加可执行权限，并复制到安装包中的 `testcases/bin/` 目录中。这里一定要注意，不可以在当前路径直接 `./echo.sh` 执行测试用例！!

```shell
[wxs@bogon demoA]$ pwd
/home/wxs/ltp/testcases/demoA
[wxs@bogon demoA]$ sudo chmod +x echo.sh 
[wxs@bogon demoA]$ sudo cp echo.sh /opt/ltp/testcases/bin/
```

修改上一节中编写的 demoA 测试用例集，将 echo.sh 测试用例添加进去：

```shell
[wxs@bogon demoA]$ cd /opt/ltp/runtest/
[wxs@bogon runtest]$ cat demoA 
sqrt sqrt
echo echo.sh
```

注意了，此时测试用例集中包含了两个测试用例，那么后面运行该测试用例集将会执行这两个测试用例。

进入上层目录，执行整体测试命令：

```shell
[wxs@bogon runtest]$ cd ..
[wxs@bogon ltp]$ sudo ./runltp -f demoA
```

结果输出依然很多，我们依然只看最关键的部分：

```shell
···
<<<test_output>>>
tst_test.c:915: INFO: Timeout per run is 0h 05m 00s
sqrt.c:20: FAIL: sqrt() failed
sqrt.c:23: PASS: sqrt() succeeds

Summary:
passed   1
failed   1
skipped  0
warnings 0
<<<execution_status>>>
initiation_status="ok"
duration=1 termination_type=exited termination_id=1 corefile=no
cutime=0 cstime=0
<<<test_end>>>
<<<test_start>>>
tag=echo stime=1506107663
cmdline="echo.sh"
contacts=""
analysis=exit
<<<test_output>>>
incrementing stop
 1 TPASS:  sucessed

Summary:
passed   1
failed   0
skipped  0
warnings 0
<<<execution_status>>>
initiation_status="ok"
duration=0 termination_type=exited termination_id=0 corefile=no
cutime=2 cstime=5
<<<test_end>>>
···
```

本次测试输出包含两个 test 体，第一个 test 体我们在上一节已经说过了，我们直接看第二个 test 体。该体中共计有 1 个测试点，状态为 TPASS。

对照 echo.sh 中的测试点，发现没有问题。至此你已经学会了最基本的创建 C 和 Shell 的测试用例以及整体测试的方法，本章内容你已经完成了。
