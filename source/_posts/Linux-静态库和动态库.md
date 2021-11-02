---
title: Linux 静态库和动态库
tags:
  - 静态库
  - 动态库
categories: 
  - Operating System
  - Linux
abbrlink: 1ab0b663
date: 2017-10-17 01:11:07
references:
  - name: JollyWing. Linux静态库生成指南
    url: http://www.cnblogs.com/jiqingwu/p/4325382.html
  - name: JollyWing. Linux动态库生成与使用指南
    url: http://www.cnblogs.com/jiqingwu/p/linux_dynamic_lib_create.html
---

在 Windows 平台和 Linux 平台下都大量存在着库。本质上来说库是一种可执行的二进制代码(但不可以独立执行)，可以被操作系统载入内存执行。

Linux下的库有两种：**静态库**和**动态库（共享库）**。本文将介绍 Linux 下静态库和动态库的概念以及相应的创建与使用方法。

## 一、对比

| 类型 | 特点 |
| ------------- | :----- |
| 静态库 | 在编译时被链接到程序中，作为可执行程序的一部分。在程序运行时不再依赖静态库，占用内存大。 |
| 动态库 | 在可执行程序运行时载入内存。动态库已经在内存中不需要再次载入。 |

## 二、静态库

### 2.1 概念

静态库指将所有相关的目标文件打包成为一个单独的文件，即`静态库文件`。静态库以 `.a` 结尾，链接器会将程序中使用到的函数的代码从库文件中**拷贝**到程序中。由于每个使用静态库的应用程序都需要拷贝所有函数的代码，所以静态链接的文件会比较**大**。

在 Unix 系统中，静态库以一种称为`存档（archive）`的特殊文件格式存放在磁盘中。存档文件是一组连接起来的可重定位目标文件的集合，有一个头部用来描述每个成员目标文件的大小和位置。

### 2.2 静态库的创建和使用

#### 2.2.1 环境准备

创建根文件夹 `staticDemo`:

```shell
[root@VM_120_243_centos ~]# mkdir staticDemo
```

创建子文件夹 `include`，用于存放所有头文件：

```shell
[root@VM_120_243_centos ~]# cd staticDemo/
[root@VM_120_243_centos staticDemo]# mkdir include
```

进入 include 文件夹，编写 `myLib.h` 文件，申明 `printInfo()` 函数：

```shell
[root@VM_120_243_centos staticDemo]# cd include/
[root@VM_120_243_centos include]# vim myLib.h
[root@VM_120_243_centos include]# cat myLib.h 
void printInfo();
```

进入上层目录，创建 `lib` 文件夹，用于存放所有库文件，再其中编写 `print.c` 文件，使用了 `myLib.h` 中的 `printInfo()` 函数：

```shell
[root@VM_120_243_centos include]# cd ..
[root@VM_120_243_centos staticDemo]# mkdir lib
[root@VM_120_243_centos staticDemo]# cd lib/
[root@VM_120_243_centos lib]# vim print.c
[root@VM_120_243_centos lib]# cat print.c 
#include <stdio.h>
#include "myLib.h"

void printInfo()
{
	printf("print from print.c file...\n");
}
```

进入上层目录，编写 `main.c` 文件，用于测试：

```shell
[root@VM_120_243_centos lib]# cd ..
[root@VM_120_243_centos staticDemo]# vim main.c
[root@VM_120_243_centos staticDemo]# cat main.c 
#include <stdio.h>
#include "myLib.h"

int main(void)
{
	printf("print from main.c file...\n");
	printInfo();
	return 0;
}
```

准备工作到此完成，整个目录结构如下：

```shell
[root@VM_120_243_centos staticDemo]# tree
.
├── include
│   └── myLib.h
├── lib
│   └── print.c
└── main.c

2 directories, 3 files
```

#### 2.2.2 生成静态库

首先进入 lib 文件夹，生成 `print.o` 文件：

```shell
[root@VM_120_243_centos staticDemo]# cd lib/
[root@VM_120_243_centos lib]# gcc -c print.c -I../include/ -o print.o
[root@VM_120_243_centos lib]# ls
print.c  print.o
```

使用 `ar命令` 归档目标文件，得到静态库：

```shell
[root@VM_120_243_centos lib]# ar -crv libprint.a print.o
a - print.o
[root@VM_120_243_centos lib]# ls
libprint.a  print.c  print.o
```

上述命令中 `crv` 是 `ar` 的命令选项：

- c 如果需要生成新的库文件，不要警告

- r 代替库中现有的文件或者插入新的文件

- v 输出详细信息

使用 `-t` 参数可以查看静态库中包含的文件：

```shell
[root@VM_120_243_centos lib]# ar -t libprint.a 
print.o
```

**注意：**我们要生成的库的文件名必须形如 `libxxx.a`，这样我们在链接这个库时，就可以用 `-lxxx`。

反过来讲，当我们告诉编译器 `-lxxx` 时，编译器就会在指定的目录中搜索 `libxxx.a` 或是 `libxxx.so`。

#### 2.2.3 生成可执行文件

进入上层目录，链接库并生成可执行文件：

```shell
[root@VM_120_243_centos staticDemo]# gcc main.c -I./include/ -L./lib/ -lprint -o main
```

执行可执行文件：

```shell
[root@VM_120_243_centos staticDemo]# ./main 
print from main.c file...
print from print.c file...
```

## 三、动态库

### 3.1 概念

动态库是一个`目标模块`，Linux 系统以 `.so` 后缀表示，Windows 以 `.dll` 后缀表示，使用动态库可以**减小应用程序占用的空间和加载时间。**

在运行时，动态库可以`加载`到任意的存储器地址，并和一个再存储器中的程序链接起来，这个过程称为动态链接，是由一个叫做`动态链接器`的程序来执行的。

动态库是 Linux 系统最**广泛**的一种程序使用方式，工作原理是相同功能的代码可以被多个程序**共同使用**。

在程序加载的时候，内核会检查程序使用到的动态库是否已经加载到内存：

- 如果没有被加载到内存，则从系统库路径搜索并且加载到相关的动态库。

- 如果动态库已经被加载到内存，程序可以直接使用而无需加载。

应用程序在**自身加载时**和**运行过程**中动态链接和加载动态库。

### 3.2 动态库的创建和使用

#### 3.2.1 环境准备

创建根文件夹 `dynamicDemo`:

```shell
[root@VM_120_243_centos ~]# mkdir dynamicDemo
```

创建子文件夹 `include`，用于存放所有头文件：

```shell
[root@VM_120_243_centos ~]# cd dynamicDemo/
[root@VM_120_243_centos dynamicDemo]# mkdir include
```

进入 include 文件夹，编写 `myLib.h` 文件，申明 `printInfo()` 函数：

```shell
[root@VM_120_243_centos dynamicDemo]# cd include/
[root@VM_120_243_centos include]# vim myLib.h
[root@VM_120_243_centos include]# cat myLib.h 
void printInfo();
```

进入上层目录，创建 `lib` 文件夹，用于存放所有库文件，再其中编写 `print.c` 文件，使用了 `myLib.h` 中的 `printInfo()` 函数：

```shell
[root@VM_120_243_centos include]# cd ..
[root@VM_120_243_centos dynamicDemo]# mkdir lib
[root@VM_120_243_centos dynamicDemo]# cd lib/
[root@VM_120_243_centos lib]# vim print.c
[root@VM_120_243_centos lib]# cat print.c 
#include <stdio.h>
#include "myLib.h"

void printInfo()
{
	printf("print from print.c file...\n");
}
```

进入上层目录，编写 `main.c` 文件，用于测试：

```shell
[root@VM_120_243_centos lib]# cd ..
[root@VM_120_243_centos dynamicDemo]# vim main.c
[root@VM_120_243_centos dynamicDemo]# cat main.c 
#include <stdio.h>
#include "myLib.h"

int main(void)
{
	printf("print from main.c file...\n");
	printInfo();
	return 0;
}
```

准备工作到此完成，整个目录结构如下：

```shell
[root@VM_120_243_centos dynamicDemo]# tree
.
├── include
│   └── myLib.h
├── lib
│   └── print.c
└── main.c

2 directories, 3 files
```

#### 3.2.2 生成动态库

进入lib文件夹，给 gcc 加入 `-fPIC` 参数，生成 print.o 文件：

```shell
[root@VM_120_243_centos dynamicDemo]# cd lib/
[root@VM_120_243_centos lib]# gcc -c print.c -fPIC -I../include/ -o print.o
[root@VM_120_243_centos lib]# ls
print.c  print.o
```

使用 gcc 的 `-shared` 参数生成动态库 libprint.so：

```shell
[root@VM_120_243_centos lib]# gcc -shared print.o -o libprint.so
[root@VM_120_243_centos lib]# ls
libprint.so  print.c  print.o
```

#### 3.2.3 生成可执行文件

进入 `dynamicDemo` 目录，链接库并生成可执行文件：

```shell
[root@VM_120_243_centos dynamicDemo]# gcc main.c -I./include/ -L./lib/ -lprint -o main
```

**注意：**如果同一目录下同时存在同名的动态库和静态库，比如 `libprint.so` 和 `libprint.a` 都在当前路径下，则 gcc 会**优先链接动态库**。

执行可执行文件：

```shell
[root@VM_120_243_centos dynamicDemo]# ./main 
./main: error while loading shared libraries: libprint.so: cannot open shared object file: No such file or directory
```

系统报错找不到 `libprint.so`，原来 Linux 是通过 `/etc/ld.so.cache` 文件搜寻要链接的动态库的。而 `/etc/ld.so.cache` 是 `ldconfig` 程序读取 `/etc/ld.so.conf` 文件生成的。（注意，`/etc/ld.so.conf` 中并不必包含 `/lib` 和 `/usr/lib`，`ldconfig` 程序会自动搜索这两个目录）

这里我们指定环境变量来找到动态库路径，在下一节中会有详细介绍:

```shell
[root@VM_120_243_centos dynamicDemo]# LD_LIBRARY_PATH=./lib/ ./main
print from main.c file...
print from print.c file...
```

### 3.3 设置动态库的搜索路径

#### 3.3.1 LD_LIBRARY_PATH

`LD_LIBRARY_PATH` 是 Linux 环境变量名，该环境变量主要用于**在查找动态库时搜索默认路径之外的其他路径。**

通常，Linux 搜索动态库的默认路径是 `/lib` 和 `/usr/lib`。`LD_LIBRARY_PATH` 路径优先级**大于**系统默认路径。

```shell
[root@VM_120_243_centos dynamicDemo]# gcc main.c -I./include/ -L./lib/ -lprint -o main
[root@VM_120_243_centos dynamicDemo]# LD_LIBRARY_PATH=./lib/ ./main
print from main.c file...
print from print.c file...
```

#### 3.3.2 rpath

gcc的 `-Wl,-rpath` 选项在编译链接时指定动态库路径，并且将动态库路径保存到可执行文件中。运行可执行文件时直接到该路径下查找动态库，而不依赖于默认路径或者环境变量。

```shell
[root@VM_120_243_centos dynamicDemo]# gcc main.c -I./include/ -L./lib/ -lprint -Wl,-rpath=./lib -o main
[root@VM_120_243_centos dynamicDemo]# ./main 
print from main.c file...
print from print.c file...
```

采用这种方式，每个程序可以设定**独立的动态库位置**，而不会像 `LD_LIBRARY_PATH` 环境变量那样影响其他程序。

#### 3.3.3 ldconfig
`ldconfig` 是一个动态链接库管理命令。该命令的用途是在默认路径（`/lib` 和 `/usr/lib`）以及动态库配置文件 `/etc/ld.so.conf` 内所包含的目录下，搜索动态库，然后为`动态载入程序（ld.so）`创建动态库列表的`缓存文件（/etc/ld.so.cache）`。

##### 3.3.3.1 加入标准路径

这里以放入 `/usr/lib` 文件夹为例：

```shell
[root@VM_120_243_centos dynamicDemo]# cp ./lib/libprint.so /usr/lib/
```

更新缓存：

```shell
[root@VM_120_243_centos dynamicDemo]# ldconfig
```

查看动态库：

```shell
[root@VM_120_243_centos dynamicDemo]# ldconfig -p | grep libprint.so
	libprint.so (libc6,x86-64) => /lib/libprint.so
```

编译执行：

```shell
[root@VM_120_243_centos dynamicDemo]# gcc main.c -I ./include/ -L ./lib/ -lprint -o main
[root@VM_120_243_centos dynamicDemo]# ./main 
print from main.c file...
print from print.c file...
```

##### 3.3.3.2 加入配置文件

首先查看 `/etc/ld.so.conf` 文件：

```shell
[root@VM_120_243_centos dynamicDemo]# cat /etc/ld.so.conf
include ld.so.conf.d/*.conf
```

可以看到其包含了 `ld.so.conf.d` 目录，因此自定义配置文件 `libprint.conf` 加入到 `/etc/ld.so.conf.d` 文件夹中，配置文件中包含该程序的动态库路径：

```shell
[root@VM_120_243_centos dynamicDemo]# pwd
/root/dynamicDemo
[root@VM_120_243_centos dynamicDemo]# vim /etc/ld.so.conf.d/libprint.conf
[root@VM_120_243_centos dynamicDemo]# cat /etc/ld.so.conf.d/libprint.conf
/root/dynamicDemo/lib
```

更新缓存：

```shell
[root@VM_120_243_centos dynamicDemo]# ldconfig
```

查看动态库：

```shell
[root@VM_120_243_centos dynamicDemo]# ldconfig -p | grep libprint.so
	libprint.so (libc6,x86-64) => /root/dynamicDemo/lib/libprint.so
```

编译执行：

```shell
[root@VM_120_243_centos dynamicDemo]# gcc main.c -I ./include/ -L ./lib/ -lprint -o main
[root@VM_120_243_centos dynamicDemo]# ./main 
print from main.c file...
print from print.c file...
```
