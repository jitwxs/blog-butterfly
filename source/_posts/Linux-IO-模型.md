---
title: Linux IO 模型
tags: IO模型
categories: Linux
abbrlink: 3b3bd025
date: 2017-11-28 01:23:44
icons: [fas fa-fire red]
related_repos:
  - name: io_mode
    url: https://github.com/jitwxs/blog-sample/tree/master/Linux/io_mode
    rel: nofollow noopener noreferrer
    target: _blank
---

## 一、概念

### 1.1 IO 模型的分类

Linux 下的 IO 模型一般包括以下五种模型：`阻塞IO`、`非阻塞IO`、`IO多路复用`、`信号驱动IO` 和 `异步IO`。

![IO模型的分类](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201711/20171123152616564.png)

### 1.2 输入操作

对于文件的输入操作，包括下面两步：

- 等待数据准备好

- 将数据从内核复制到用户空间

对于套接字(socket)的输入操作，包括下面两步：

- 等待数据从网络中到达，到达后复制到内核中的缓冲区

- 将数据从内核缓冲区复制到应用进程缓冲区

关于套接字的知识，这里不再赘述，参考文章[《Linux Socket 编程》](/f2ee55a7.html) 。

### 1.3 同步和异步

**同步：** 发出一个功能调用时，在没有得到结果之前，该调用就不返回。也就是必须一件一件事做,等前一件做完了才能做下一件事。

**异步：** 当一个异步过程调用发出后，调用者不能立刻得到结果。实际处理这个调用的部件在完成后，通过状态、通知和回调来通知调用者。

二者区别是**会不会导致请求进程（或线程）阻塞**。

### 1.4 阻塞和非阻塞

**阻塞：** 调用结果返回之前，当前线程会被挂起（线程进入非可执行状态，在这个状态下，cpu 不会给线程分配时间片），函数只有在得到结果之后才会返回。

**非阻塞：** 在不能立刻得到结果之前，该函数不会阻塞当前线程，而会立刻返回。

二者区别是**应用程序的调用是否立即返回**。

### 1.5 用户空间和内核空间

现在操作系统都是采用`虚拟存储器`，那么对 32 位操作系统而言，它的寻址空间（虚拟存储空间）为 4G（2的32次方）。

操作系统的核心是内核，独立于普通的应用程序，可以访问受保护的内存空间，也有访问底层硬件设备的所有权限。为了保证用户进程不能直接操作内核（kernel），保证内核的安全，操心系统将虚拟空间划分为两部分，一部分为内核空间，一部分为用户空间。

针对 Linux 操作系统而言，将最高的 1G 字节（从虚拟地址 0xC0000000 到 0xFFFFFFFF），供内核使用，称为`内核空间`，而将较低的 3G 字节（从虚拟地址 0x00000000 到 0xBFFFFFFF），供各个进程使用，称为`用户空间`。

### 1.6 进程的阻塞

正在执行的进程，由于期待的某些事件未发生，如请求系统资源失败、等待某种操作的完成、新数据尚未到达或无新工作做等，则由系统自动执行阻塞原语(Block)，使自己由运行状态变为`阻塞状态`。

可见，进程的阻塞是进程自身的一种主动行为，也因此只有处于运行态的进程（获得CPU），才可能将其转为阻塞状态。当进程进入阻塞状态，是**不占用 CPU 资源**的。

## 二、阻塞 IO

这种情况下**进程会一直阻塞，直到数据拷贝完成**。常见的慢速设备（socket、pipe、fifo、terminal）的IO默认方式都是阻塞的。

![Blocking IO](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201712/20171201105503374.png)

如图所示，当用户进程发起 read 操作时，kernel 首先进入等待数据阶段，待数据到来。而在用户进程中，整个进程会处于阻塞的状态。当 kernel 准备好数据后，它会将数据从 kernel 拷贝到用户进程，然后 kernel 返回结果，此时用户进程才解除阻塞的状态。

因此，它的特点是 **IO 执行的两个阶段都被阻塞**。

以标准 IO 为例，测试代码如下：

```c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <string.h>
#include <errno.h>

#define BUFSIZE 128

int main() {
    char buf[BUFSIZE] = {0};
    int ret, flags;

    /* 注：标准输入输出默认就是阻塞IO，其实可以不用手动设置 */

    // 获得输入的文件状态标记
    if((flags = fcntl(STDIN_FILENO, F_GETFL, 0)) < 0) {
        perror("fcntl");
        return -1;
    }

    // 设置为阻塞IO
    flags &= ~O_NONBLOCK;
    if(fcntl(STDIN_FILENO, F_SETFL, flags) < 0) {
        perror("fcntl");
        return -1;
    }

    while(1) {
        sleep(2);

        // 标准输入
        ret = read(STDIN_FILENO, buf, BUFSIZE-1);
        if(ret == 0)
            perror("read--no");
        else if (ret == -1)
            printf("[ERROR] %s\n", strerror(errno));
        else
            printf("read = %d\n", ret);

        // 标准输出
        write(STDOUT_FILENO, buf, BUFSIZE);
        memset(buf, 0, BUFSIZE);
    }

    return 0;
}
```

运行结果和我们想的一样，内核会阻塞等待我们的输入。

```shell
wxs@ubuntu:~/myLinux/io_mode/block$ ./block

```

当我们输入后，内核接收到我们的输入，并将其输出。

```shell
wxs@ubuntu:~/myLinux/io_mode/block$ ./block
hello block_io!
read = 16
hello block_io!
```

## 三、非阻塞 IO

我们可以设置 IO 相关的系统调用为 `non-blocaking` 来达到非阻塞式IO。当我们执行一个读操作时，流程如下：

![NonBlocking IO](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201712/20171201105603099.png)

如图所示，当用户进程发起 read 操作时，如果 kernel 中的数据还没有准备好，那么它**不会阻塞**掉用户进程，而是会**直接返回**一个 `EWOULDBLOCK` 错误。

从用户进程的角度，它发起一个 read 操作后立即就得到了一个结果，用户进程判断结果是 EWOULDBLOCK 时会再次发起 read 操作。这种利用返回值进行不断调用被称为`轮询`(polling)，显而易见，这么做会耗费大量 CPU 时间。

一旦 kernel 中的数据准备好了，并且又再次收到了用户进程的请求，那么它马上就将数据拷贝到了用户内存，然后返回。

因此，它的特点是**用户进程需要不断的主动询问 kernel 数据好了没有**。

还是以标准 IO 为例，测试代码如下：

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>
#include <errno.h>

#define BUFSIZE 128

int main() {
    char buf[BUFSIZE] = {0};
    int ret, flags;

    // 获得输入的文件状态标记
    if((flags = fcntl(STDIN_FILENO, F_GETFL, 0)) < 0) {
        perror("fcntl");
        return EXIT_FAILURE;
    }

    // 设置为非阻塞IO
    flags |= O_NONBLOCK;
    if(fcntl(STDIN_FILENO, F_SETFL, flags) < 0) {
        perror("fcntl");
        return EXIT_FAILURE;
    }

    while(1) {
        sleep(2);
        
        // 标准输入
        ret = read(STDIN_FILENO, buf, BUFSIZE-1);
        if (ret == 0)
            perror("read--no");
        else if (ret == -1) {
            if (errno == EWOULDBLOCK)
                printf("Data is not ready!\n");
            else
                printf("[ERROR] %s\n", strerror(errno));
        } else
            printf("read = %d\n", ret);

        // 标准输出
        write(STDOUT_FILENO, buf, BUFSIZE);
        memset(buf, 0, BUFSIZE);
    }

    return 0;
}
```

运行结果和我们想的一样，当 kernel 没有接收到输入时，kernel 会返回 EWOULDBLOCK 错误：

```shell
wxs@ubuntu:~/myLinux/io_mode/non_block$ ./non_block 
Data is not ready!
Data is not ready!
...
```

当内核接收到我们的输入，将其输出：

```shell
wxs@ubuntu:~/myLinux/io_mode/non_block$ ./non_block 
Data is not ready!
Data is not ready!
aaa
read = 4
aaa
Data is not ready!
...
...
```

## 四、IO 多路复用

将阻塞式IO 的一个完整阻塞**拆分**成两个阻塞，就形成了 `IO复用`。相较于阻塞式IO，IO复用需要使用两个系统调用，而阻塞式IO 只使用了一个系统调用。

IO 复用会用到 `select`、`poll`、`epoll` 函数，这几个函数也会使进程阻塞，但是和阻塞 IO 所不同的是，这几个函数可以同时阻塞**多个** IO 操作，而且可以同时对多个读操作、多个写操作的 IO 函数进行检测。

当用户进程调用了 select，**整个进程**会被阻塞，同时内核会监听 select 负责的**所有** IO 操作，一旦其中**任何一个**的数据准备好了，select 就会返回。

![IO Multiplexing](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201712/20171201105631329.png)

### 4.1 select

```c
#include <sys/select.h>  
#include <sys/time.h>  

int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);  

void FD_CLR(int fd, fd_set *set);  //在文件描述符集合中删除一个文件描述符

int  FD_ISSET(int fd, fd_set *set);  //测试指定的文件描述符是否在该集合中

void FD_SET(int fd, fd_set *set);  //在文件描述符集合中增加一个新的文件描述符

void FD_ZERO(fd_set *set); //将指定的文件描述符集清空
```

**返回值**：若有就绪描述符则为其数目，若超时则为 0，若出错则为 -1。

**参数**：

- `nfds`：等待最大套接字+1
- `readfds`：检查读事件的容器
- `writefds`：检查写事件的容器
- `exceptfds`：检查异常事件的容器
- `timeout`：超时时间
	- timeout 设置为空指针：永远等待，直到有 I/O 描述符准备好
	- timeout 指定 timeval 结构的秒数和微秒数：等待固定时间
	- timeout 指定 timeval 结构的秒数和微秒数为 0，不等待，检查后立即返回

select 函数监视的文件描述符分3类，分别是 `writefds`、`readfds`、和 `exceptfds`。调用后 select 函数会阻塞，直到有描述符就绪（有数据可读、可写、有 except、超时）。当 select 函数返回后，可以通过**遍历 fdset**，来找到就绪的描述符。

select 的一个缺点在于单个进程**能够监视的文件描述符的数量存在最大限制**，在 Linux 上一般为 1024，可以通过修改宏定义甚至重新编译内核的方式提升这一限制，但是这样也会造成效率的降低。

下面给出一个在有名管道中使用 select 的例子。其中 write 端负责往有名管道中写数据，read 端负责从有名管道中读数据：

```c head.h
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/select.h>

#define BUFSIZE 128

#define FILE_PATH "./fifo"

// 判断FIFO文件是否存在，存在返回1,否则返回0
int check_fifo_exist() {
    if (!access(FILE_PATH, 0))
        return 1;
    else
        return 0;
}

// 创建FIFO文件
int mk_fifo() {
    return mkfifo(FILE_PATH, 0666);
}

// 删除FIFO文件
int rm_fifo() {
    return execlp("rm", "rm" "-f", FILE_PATH, NULL);
}
```

```c write.c
#include "head.h"

int main(int argc, char *argv[])
{
    int fd;
    char buf[BUFSIZE];

    // 创建有名管道
    if (!check_fifo_exist())
        if (mk_fifo() == -1) {
            printf("创建FIFO文件出错");
            perror("mkfifo");
        }

    // 读写方式打开管道
    if ((fd = open(FILE_PATH, O_RDWR)) < 0){
        printf("打开FIFO文件出错");
        perror("open");
    }

    while(1){
        printf("Send msg: ");
        fgets(buf, BUFSIZE, stdin);
        write(fd, buf, strlen(buf)); // 往管道里写内容

        if(strcmp("exit\n", buf) == 0)
            break;
    }

    close(fd);
    if (check_fifo_exist())
        if (rm_fifo() == -1) {
            printf("删除FIFO文件出错");
            perror("rmfifo");
        }

    return 0;
}
```

```c read.c
#include "head.h"

int main(int argc, char *argv[])
{
    fd_set rfds;
    struct timeval tv;
    int ret, fd;

    // 创建有名管道
    if (!check_fifo_exist())
        if (mk_fifo() == -1) {
            printf("创建FIFO文件出错");
            perror("mkfifo");
        }

    // 读写方式打开管道
    if ((fd = open(FILE_PATH, O_RDWR)) == -1){
        printf("打开FIFO文件出错");
        perror("open");
    }

    while(1){
        //每次循环都要初始化
        FD_ZERO(&rfds);        // 清空
        FD_SET(fd, &rfds);  // 有名管道描述符 fd 加入集合

        // 超时设置
        tv.tv_sec = 5;
        tv.tv_usec = 0;

        // 监视并等待多个文件（标准输入，有名管道）描述符的属性变化（是否可读）
        // 没有属性变化，这个函数会阻塞，直到有变化才往下执行，这里没有设置超时
        // FD_SETSIZE 为 <sys/select.h> 的宏定义，值为 1024
        if ((ret = select(FD_SETSIZE, &rfds, NULL, NULL, &tv)) == -1)
            perror("select");
        else if(ret > 0) {
            char buf[100] = {0};
            read(fd, buf, sizeof(buf));
            printf("Receive msg: %s", buf);

            if (strcmp("exit\n", buf) == 0)
                break;
        }else if(ret == 0)
            printf("Time out..\n");
    }

    close(fd);
    if (check_fifo_exist())
        if (rm_fifo() == -1) {
            printf("删除FIFO文件出错");
            perror("rmfifo");
        }

    return 0;
}
```

运行结果如下：

![select 运行结果](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201712/20171228004245480.png)

关于 select 函数还有一个在 socket C/S 中的应用，限于篇幅，这里不再列举，在[《Linux Socket 编程》](/f2ee55a7.html) 4.3 节有写出。

### 4.2 poll

```c
#include <poll.h>

int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

返回值：若有就绪描述符则为其数目，若超时则为 0，若出错则为 -1。

- fds 参数是一个 pollfd 结构类型的数组，pollfd 结构包含了要监视的 event 和发生的 event。

```c
struct pollfd { 
int fd;    /* file descriptor */ 
short events; /* requested events */ 
short revents; /* returned events */ 
}; 

poll函数的事件标志符值：

POLLIN         普通或优先级带数据可读
POLLRDNORM     普通数据可读
POLLRDBAND     优先级带数据可读
POLLPRI        高优先级数据可读
POLLOUT        普通数据可写
POLLWRNORM     普通数据可写
POLLWRBAND     优先级带数据可写
POLLERR        发生错误
POLLHUP        发生挂起
POLLNVAL       描述字不是一个打开的文件

注意：后三个只能作为描述字的返回结果存储在revents中，而不能作为测试条件用于events中。

```

- nfds 参数指定被监听事件集合 fds 的大小。其类型 nfds_t 的定义如下:

```c
typedef unsigned long int nfds_t; 
```

- timeout 参数指定poll的超时值，单位是**毫秒**
	- 当 timeout 为-1时，poll 调用将永远阻塞，直到某个事件发生
	- 当 timeout 为0时，poll 调用将立即返回

poll 相比于 select **没有最大监视数量限制**，但是数量过大后性能也是会下降。和 select 函数一样，poll 返回后，通过遍历文件描述符来获取就绪的描述符。事实上，同时连接的大量客户端在一时刻可能只有很少的处于就绪状态，因此随着监视的描述符数量的增长，其效率也会线性下降。

我们继续使用有名管道来演示 poll 函数，功能与之前相同：

```c write.c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/select.h>

#define BUFSIZE 128
#define NAME "test_fifo"

int main(int argc, char *argv[])
{
    int fd;
    char buf[BUFSIZE];

    // 创建有名管道
    if (access(NAME, 0) == -1)
        if ((mkfifo(NAME, 0666)) != 0) {
            perror("mkfifo");
            return -1;
        }

    // 读写方式打开管道
    if((fd = open("test_fifo", O_RDWR)) < 0){
        perror("open fifo");
        return -1;
    }

    // 往管道里写内容
    while(1){
        printf("Send msg: ");
        scanf("%s", buf);
        write(fd, buf, strlen(buf));

        if(strcmp("quit", buf) == 0)
            break;
    }
    return 0;
}
```

```c read.c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <poll.h>

#define NAME "test_fifo"

int main(int argc, char *argv[])
{
    int ret, fd;
    int timeout = 5000; // 设置超时时间，单位为毫秒

    // 创建有名管道
    if (access(NAME, 0) == -1)
        if ((mkfifo(NAME, 0666)) != 0){
            perror("mkfifo");
            return -1;
        }

    // 读写方式打开管道
    if ((fd = open("test_fifo", O_RDWR)) < 0){
        perror("open fifo");
        return -1;
    }

    // 创建struct pollfd 结构体
    struct pollfd pfds[] = {
        {.fd = fd, .events = POLLIN}
    };

    while(1){
        ret = poll(pfds, 1, timeout);

        if (ret == -1)
            perror("poll");
        else if(ret > 0) {
            if (pfds[0].revents == POLLIN) {
                char buf[100] = {0};
                read(fd, buf, sizeof(buf));
                printf("Receive msg: %s\n", buf);

                if (strcmp("quit", buf) == 0)
                    break;
            } else
                printf("revents error..\n");
        }else if(ret == 0)
            printf("timeout..\n");
    }

    execlp("rm", "rm", "-f", NAME, NULL);

    return 0;
}
```

运行结果如下：

![poll 运行结果](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201711/20171127151344054.png)

### 4.3 epoll

epoll 是在 2.6 内核中提出的，是 select 和 poll 的增强版本。相对于 select 和 poll 来说，epoll 更加灵活，没有描述符限制。epoll 使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个`事件表`中，这样在用户空间和内核空间的**拷贝只需一次**。

在 select/poll 中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而 **epoll 事先通过 epoll_ctl() 来注册一个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似 callback 的回调机制，迅速激活这个文件描述符，当进程调用 epoll_wait() 时便得到通知**。

epoll 监视的描述符数量不受限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048，具体数目可以 `cat /proc/sys/fs/file-max` 查看，一般来说和系统内存有关。

epoll 不同于 select 和 poll 轮询的方式，而是通过每个 fd 定义的回调函数来实现的。只有就绪的 fd 才会执行回调函数。

#### 4.3.1 三大接口

epoll 操作过程需要三个接口，分别如下：

**（1）int epoll_create(int size);**

```c
#include <sys/epoll.h>

/*
 创建一个epoll的句柄
 成功返回文件描述符，失败返回-1并设置errno。
 */
int epoll_create(int size);

```

size 用来告诉内核这个监听的数目一共有多大，这个参数不同于 select() 中的第一个参数，给出最大监听的 fd+1 的值，参数 size 并不是限制了 epoll 所能监听的描述符最大个数，只是对内核初始分配内部数据结构的一个建议。

当创建好 epoll 句柄后，它就会占用一个 fd 值，在 Linux 下如果查看 /proc/ 进程 id/fd/，是能够看到这个 fd 的，所以在使用完 epoll 后，必须调用 close() 关闭，否则可能导致 fd 被耗尽。

**（2）int epoll_ctl(int epfd, int op, int fd, struct epoll_event * event)；**

```c
#include <sys/epoll.h>

/*
 对指定描述符fd执行op操作
 成功返回0，失败则返回-1并设置errno。
 */
int epoll_ctl(int epfd,int op,int fd,struct epoll_event* event);
```

- epfd：是 epoll_create() 的返回值
- op：表示op操作，用三个宏来表示
	- EPOLL_CTL_ADD  往事件表中注册fd上的事件
	- EPOLL_CTL_MOD  修改事件表中的注册事件
	- EPOLL_CTL_DEL  删除fd上的注册事件
- fd：需要监听的 fd（文件描述符）
- event：告诉内核需要监听什么事，它是 epoll_event 结构指针类型，定义如下:

```c
struct epoll_event{
   __uint32_t events;  /* Epoll events */
   epoll_data_t data;   /* User data variable */
};

typedef union epoll_data {
    void *ptr;
    int fd;
    __uint32_t u32;
    __uint64_t u64;
} epoll_data_t;

//events可以是以下几个宏的集合：
EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
EPOLLOUT：表示对应的文件描述符可以写；
EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR：表示对应的文件描述符发生错误；
EPOLLHUP：表示对应的文件描述符被挂断；
EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
```

**（3）int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);**

```c
#include <sys/epoll.h>

// 等待epfd上的io事件，最多返回maxevents个事件。
int epoll_wait(int epfd,struct epoll_event *events,int maxevents,int timeout);
```

- events 从内核得到事件的集合
- maxevents 告诉内核这个 events 有多大，这个 maxevents 的值不能大于创建 epoll_create() 时的 size
- timeout 超时时间（毫秒，0 会立即返回，-1 是永久阻塞）

成功返回准备好的文件描述符数量，超时返回 0，出错返回 -1 并设置 errno。

我们继续使用有名管道来演示 epoll 函数，功能与之前相同：

```c write.c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/select.h>

#define BUFSIZE 128
#define NAME "test_fifo"

int main(int argc, char *argv[])
{
    int fd;
    char buf[BUFSIZE];

    // 创建有名管道
    if (access(NAME, 0) == -1)
        if ((mkfifo(NAME, 0666)) != 0) {
            perror("mkfifo");
            return -1;
        }

    // 读写方式打开管道
    if((fd = open("test_fifo", O_RDWR)) < 0){
        perror("open fifo");
        return -1;
    }

    // 往管道里写内容
    while(1){
        printf("Send msg: ");
        scanf("%s", buf);
        write(fd, buf, strlen(buf));
    }
    return 0;
}
```

```c read.c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/epoll.h>

#define MAXEVENTS 1024 //最大事件数目
#define NAME "test_fifo"

int main(int argc, char *argv[])
{
    int ret, fd, epfd;
    struct epoll_event events[MAXEVENTS]; //监听事件数组
    struct epoll_event ev; //监听事件临时变量
    int timeout = 5000; // 设置超时时间，单位为毫秒

    // 创建有名管道
    if (access(NAME, 0) == -1)
        if ((mkfifo(NAME, 0666)) != 0){
            perror("mkfifo");
            return -1;
        }

    // 读写方式打开管道
    if ((fd = open("test_fifo", O_RDWR)) < 0){
        perror("open fifo");
        return -1;
    }

    // 设置监听的事件内容
    ev.data.fd = fd;
    ev.events = EPOLLIN;

    // 创建epoll句柄
    epfd = epoll_create(MAXEVENTS);

    // 注册事件ev
    ret = epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);
    if (ret == -1) {
        perror("epoll_ctl");
        return -1;
    }

    while(1){
        // 等待事件
        int res = epoll_wait(epfd, events, MAXEVENTS, timeout);
        if (res == -1)
            perror("epoll_wait");
        else if (res == 0)
            printf("timeout..\n");
        else {
            // 循环读取
            for(int i=0; i<res; i++) {
                if (events[i].data.fd == fd && events[i].events & EPOLLIN) {
                    char buf[100] = {0};
                    read(fd, buf, sizeof(buf)); 
                    printf("Receive msg: %s\n", buf);
                }
            }
        }
    }

    return 0;
}
```

运行结果如下：

![epoll 运行结果](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201711/20171127154853860.png)

#### 4.3.2 工作模式

epoll 对文件描述符的操作有两种模式：`LT（水平触发）`和 `ET（边缘触发）`。LT 模式是默认模式，LT 模式与 ET 模式的区别如下：

##### 4.3.2.1 LT 模式
当 epoll_wait 检测到描述符事件发生并将此事件通知应用程序，**应用程序可以不立即处理该事件**。下次调用 epoll_wait 时，会再次响应应用程序并通知此事件。

LT(level triggered)是缺省的工作方式，并且同时支持 block 和 no-block socket。在这种做法中，内核告诉你一个文件描述符是否就绪了，然后你可以对这个就绪的 fd 进行 IO 操作。如果你不作任何操作，内核还是会继续通知你的。

##### 4.3.2.2 ET 模式

当 epoll_wait 检测到描述符事件发生并将此事件通知应用程序，**应用程序必须立即处理该事件**。如果不处理，下次调用 epoll_wait 时，不会再次响应应用程序并通知此事件。

ET(edge-triggered) 是高速工作方式，只支持 no-block socket。在这种模式下，当描述符从未就绪变为就绪时，内核通过 epoll 告诉你。然后它会假设你知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知，直到你做了某些操作导致那个文件描述符不再为就绪状态了(比如，你在发送，接收或者接收请求，或者发送接收的数据少于一定量时导致了一个 EWOULDBLOCK 错误）。但是请注意，如果一直不对这个 fd 作 IO 操作(从而导致它再次变成未就绪)，内核不会发送更多的通知(only once)。

ET 模式在很大程度上减少了 epoll 事件被重复触发的次数，因此效率要比 LT 模式高。epoll 工作在 ET 模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务堵死。

### 4.4 函数比较

#### 4.4.1 事件集

select 和 poll 的共同缺点，每次两者调用都需要返回整个用户注册的事件集，必须对整个事件集逐个遍历判断，时间复杂度 $O(n)$。

epoll模型在内核中维护一个事件表,时间复杂度 $O(1)$。

#### 4.4.2 最大支持文件描述符的数量

select 能够同时监听最大的文件描述符数量是 1024 个，而 poll 和 epoll 相对没有限制，通常能够达到 65535 个。

#### 4.4.3 工作模式

select、poll 模型都只工作在相对低效的 LT 模式,而 epoll 可以工作在 ET 的高效模式。

#### 4.4.4 实现方式

select 和 poll 采用的都是`轮询`的方式，而 epoll 采用的`回调`的方式。

## 五、信号驱动 IO

所谓信号驱动式 IO，就是利用**信号机制**，安装信号 `SIGIO` 的处理函数，进程继续运行并不阻塞。通过监听文件描述符，当数据准备好时，进程会**收到**一个 `SIGIO` 信号，可以在信号处理函数中调用 IO 操作函数处理数据。

![SIGIO](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201711/20171123151908448.png)

## 六、异步 IO

当用户进程发起 read 操作后，用户立刻就可以开始去做其它的事情。而从 kernel 的角度，当它收到一个异步IO请求后，首先它会立刻返回，所以不会对用户进程产生任何阻塞。然后，kernel 会等待数据准备完成，然后将数据拷贝到用户内存，当这一切都完成之后，内核会给用户进程发送一个信号，告诉它 read 操作完成了。

![Asynchronous IO](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201711/20171123151941696.png)

`AIO` 在 2.5 版本的内核中首次出现，是 2.6 版本内核的一个标准特性。Linux 在 POSIX 标准下有一套 AIO 实现机制。

### 6.1 AIO API

| API 函数 | 说明 |
| ------------- |:------------- |
| aio_read | 请求异步读操作 |
| aio_error | 检查异步请求的状态 |
| aio_return | 获得完成的异步请求的返回状态 |
| aio_write | 请求异步写操作 |
| aio_suspend | 挂起调用进程，直到一个或多个异步请求已经完成（或失败） |
| aio_cancel | 取消异步 I/O 请求 |
| lio_listio | 发起一系列 I/O 操作 |

每个 API 函数都要用到 `aiocb` 结构。这个结构有很多元素，我仅给出了需要使用的元素。

```c
struct aiocb {
  int aio_fildes;               //要异步操作的文件描述符
  int aio_lio_opcode;           //用于lio操作时选择操作何种异步I/O类型(r/w/nop)
  volatile void *aio_buf;       //异步读或写的缓冲区的缓冲区
  size_t aio_nbytes;            //异步读或写的字节数
  struct sigevent aio_sigevent; //异步通知的结构体
};
```

#### 6.1.1 aio_read

aio_read 函数请求对一个有效的文件描述符进行异步读操作。这个文件描述符可以表示一个文件、套接字甚至管道。aio_read 函数的原型如下：

```c
int aio_read( struct aiocb *aiocbp );
```

`aio_read` 函数在请求进行排队之后会立即返回。如果执行成功，返回值就为 0；如果出现错误，返回值就为  -1，并设置 errno 的值。

要执行读操作，应用程序必须对 aiocb 结构进行初始化。下面这个简短的例子就展示了如何填充 aiocb 请求结构，并使用 aio_read 来执行异步读请求（现在暂时忽略通知）操作。它还展示了 aio_error 的用法，不过我们将稍后再作解释。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <aio.h>
#include <unistd.h>

#define FILE_NAME "test.txt"

int main(void) {
    int fd, ret;
    struct aiocb my_aiocb;
    
    fd = open(FILE_NAME, O_RDONLY);
    if (fd < 0)
        perror("open error");

    /* Zero out the aiocb structure (recommended) */
    memset(&my_aiocb, 0, sizeof(my_aiocb));
    
    /* Allocate a data buffer for the aiocb request */
    my_aiocb.aio_buf = malloc(BUFSIZ+1);
    if(!my_aiocb.aio_buf)
        perror("malloc error");

    /* Initialize the necessary fields in the aiocb */
    my_aiocb.aio_fildes = fd;
    my_aiocb.aio_nbytes = BUFSIZ;
    my_aiocb.aio_offset = 0;

    ret = aio_read(&my_aiocb);
    if (ret < 0)
        perror("aio_read");
    
    /* loop wait read data */
    while (aio_error(&my_aiocb) == EINPROGRESS) {
        printf("wait read...\n");
        sleep(1);    
    }

    /* get return value */
    ret = aio_return(&my_aiocb);
    if (ret > 0)
        printf("res = %d, content = %s\n",
                ret, (char *)my_aiocb.aio_buf);
    else
        perror("ail_return");

    return 0;
}
```

在上面程序中，在打开要从中读取数据的文件之后，清空了 aiocb 结构，然后分配一个数据缓冲区。并将对这个数据缓冲区的引用放到 aio_buf 中。然后，我们将 aio_nbytes 初始化成缓冲区的大小。并将 aio_offset 设置成 0（该文件中的第一个偏移量）。我们将 aio_fildes 设置为从中读取数据的文件描述符。

在设置这些域之后，就调用 aio_read 请求进行读操作。我们然后可以调用 aio_error 来确定 aio_read 的状态。只要状态是 EINPROGRESS，就一直忙碌等待，直到状态发生变化为止。现在，请求可能成功，也可能失败。

运行结果（注：使用AIO API**编译时候要加`-lrt`参数**）：

![aio_read 运行结果](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201711/20171128011335831.png)

#### 6.1.2 aio_error

aio_error 函数被用来确定请求的状态。其原型如下：

```c
int aio_error( struct aiocb *aiocbp );
```

- EINPROGRESS，说明请求尚未完成
- ECANCELLED，说明请求被应用程序取消了
- -1，发生错误，并设置 errno

#### 6.1.3 aio_return

异步 I/O 和标准块 I/O 之间的另外一个区别是我们不能立即访问这个函数的返回状态，因为我们并没有阻塞在 read 调用上。在标准的 read 调用中，返回状态是在该函数返回时提供的。但是在异步 I/O 中，我们要使用 aio_return 函数。这个函数的原型如下：

```c
ssize_t aio_return( struct aiocb *aiocbp );
```

只有在 aio_error 调用确定请求已经完成（可能成功，也可能发生了错误）之后，才会调用这个函数。

aio_return 的返回值就等价于同步情况中 read 或 write 系统调用的返回值（所传输的字节数，如果发生错误，返回值就为 -1）。

#### 6.1.4 aio_write

aio_write 函数用来请求一个异步写操作。其函数原型如下：

```c
int aio_write( struct aiocb *aiocbp );
```

aio_write 函数会立即返回，说明请求已经进行排队（成功时返回值为 0，失败时返回值为 -1，并相应地设置 errno）。

这与 read 系统调用类似，但是有一点不一样的行为需要注意，在 read 调用中设置偏移量是重要的。然而，对于 write 来说，这个偏移量只有在没有设置 `O_APPEND` 选项的文件上下文中才会重要。如果设置了 `O_APPEND`，那么这个偏移量就会被忽略，数据都会被附加到文件的末尾。否则，`aio_offset`域就确定了数据在要写入的文件中的偏移量。

下面给出一个 aio_write 的例子：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <aio.h>

#define FILE_NAME "test.txt"

#define CONTENT "hello world\n"

int main(void) {
    int fd, ret;
    struct aiocb my_aiocb;
    char *buf = CONTENT;
    
    fd = open(FILE_NAME, O_RDWR | O_CREAT);
    if (fd < 0)
        perror("open error");

    /* Zero out the aiocb structure (recommended) */
    memset(&my_aiocb, 0, sizeof(my_aiocb));
    
    /* Allocate a data buffer for the aiocb request */
    my_aiocb.aio_buf = malloc(BUFSIZ+1);
    if(!my_aiocb.aio_buf)
        perror("malloc error");

    /* Initialize the necessary fields in the aiocb */
    my_aiocb.aio_buf = buf;
    my_aiocb.aio_fildes = fd;
    my_aiocb.aio_nbytes = BUFSIZ;
    my_aiocb.aio_offset = 0;

    ret = aio_write(&my_aiocb);
    if (ret < 0)
        perror("aio_read");
    
    /* loop wait read data */
    while (aio_error(&my_aiocb) == EINPROGRESS) {
        printf("wait write...\n");
        sleep(1);
    }

    /* get return value */
    ret = aio_return(&my_aiocb);
    if (ret > 0)
        printf("res = %d\n", ret);
    else
        perror("ail_return");

    return 0;
}
```

运行结果如下：

![aio_write 运行结果](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201711/20171128011346161.png)

#### 6.1.5 aio_suspend

我们可以使用 aio_suspend 函数来挂起（或阻塞）调用进程，直到异步请求完成为止，此时会产生一个信号，或者发生其他超时操作。调用者提供了一个 aiocb 引用列表，其中任何一个完成都会导致 aio_suspend 返回。 aio_suspend 的函数原型如下：

```c
int aio_suspend( const struct aiocb *const cblist[], int n, const struct timespec *timeout );
```

aio_suspend 的使用非常简单。我们要提供一个 aiocb 引用列表。如果任何一个完成了，这个调用就会返回 0。否则就会返回 -1，说明发生了错误。

```c
struct aioct *cblist[MAX_LIST]
 
/* Clear the list. */
bzero( (char *)cblist, sizeof(cblist) );
 
/* Load one or more references into the list */
cblist[0] = &my_aiocb;
 
ret = aio_read( &my_aiocb );
 
ret = aio_suspend( cblist, MAX_LIST, NULL );
```

注意，aio_suspend 的第二个参数是 cblist 中**元素的个数**，而不是 aiocb 引用的个数。cblist 中任何 NULL 元素都会被 aio_suspend 忽略。

如果为 `aio_suspend` 提供了超时，而超时情况的确发生了，那么它就会返回 -1，errno 中会包含 `EAGAIN`。

#### 6.1.6 aio_cancel

aio_cancel 函数允许我们取消对某个文件描述符执行的一个或所有 I/O 请求。其原型如下：

```c    
int aio_cancel( int fd, struct aiocb *aiocbp );
```

要取消一个请求，我们需要提供文件描述符和 aiocb 引用。如果这个请求被成功取消了，那么这个函数就会返回 `AIO_CANCELED`。如果请求完成了，这个函数就会返回 `AIO_NOTCANCELED`。

要取消对某个给定文件描述符的所有请求，我们需要提供这个文件的描述符，以及一个对 aiocbp 的 NULL 引用。如果所有的请求都取消了，这个函数就会返回 `AIO_CANCELED`；如果至少有一个请求没有被取消，那么这个函数就会返回 `AIO_NOT_CANCELED`；如果没有一个请求可以被取消，那么这个函数就会返回 `AIO_ALLDONE`。

我们然后可以使用 aio_error 来验证每个 AIO 请求。如果这个请求已经被取消了，那么 aio_error 就会返回 -1，并且 errno 会被设置为 `ECANCELED`。

#### 6.1.7 lio_listio

最后，AIO 提供了一种方法使用 `lio_listio` A函数同时发起多个传输。这个函数非常重要，因为这意味着我们可以在一个系统调用（一次内核上下文切换）中启动大量的 I/O 操作。从性能的角度来看，这非常重要。lio_listio 函数的原型如下：

```c    
int lio_listio( int mode, struct aiocb *list[], int nent,
                   struct sigevent *sig );
```

mode 参数可以是：

- LIO_WAIT 会阻塞这个调用，直到所有的 I/O 都完成为止
- LIO_NOWAIT 在操作进行排队之后，就会返回

list 是一个 aiocb 引用的列表，最大元素的个数是由 nent 定义的。注意 list 的元素可以为 NULL，lio_listio 会将其忽略。

sigevent 指明了所有 I/O 操作都完成时产生信号的方法。

对于 lio_listio 的请求与传统的 read 或 write 请求在必须指定的操作方面稍有不同，如下所示：

```c
struct aiocb aiocb1, aiocb2;
struct aiocb *list[MAX_LIST];
 
...
 
/* Prepare the first aiocb */
aiocb1.aio_fildes = fd;
aiocb1.aio_buf = malloc( BUFSIZE+1 );
aiocb1.aio_nbytes = BUFSIZE;
aiocb1.aio_offset = next_offset;
aiocb1.aio_lio_opcode = LIO_READ;
 
...
 
bzero( (char *)list, sizeof(list) );
list[0] = &aiocb1;
list[1] = &aiocb2;
 
ret = lio_listio( LIO_WAIT, list, MAX_LIST, NULL );
```

对于读操作来说，aio_lio_opcode 域的值为 LIO_READ。对于写操作来说，我们要使用 LIO_WRITE，不过 LIO_NOP 对于不执行操作来说也是有效的。

下面给出一个 lio_listio 的例子：

```c
#include<stdio.h>
#include<sys/socket.h>
#include<netinet/in.h>
#include<arpa/inet.h>
#include<assert.h>
#include<unistd.h>
#include<stdlib.h>
#include<errno.h>
#include<string.h>
#include<sys/types.h>
#include<fcntl.h>
#include<aio.h>

#define BUFFER_SIZE 1025

int MAX_LIST = 2;


int main(int argc,char **argv)
{
    struct aiocb *listio[2];
    struct aiocb rd,wr;
    int fd,ret;

    //异步读事件
    if ((fd = open("test1.txt",O_RDONLY)) < 0)
        perror("test1.txt");

    bzero(&rd,sizeof(rd));

    rd.aio_buf = (char *)malloc(BUFFER_SIZE);
    rd.aio_fildes = fd;
    rd.aio_nbytes = 1024;
    rd.aio_offset = 0;
    rd.aio_lio_opcode = LIO_READ;   ///lio操作类型为异步读

    //将异步读事件添加到list中
    listio[0] = &rd;

    //异步写事件
    if ((fd = open("test2.txt",O_WRONLY)) < 0)
        perror("test2.txt");

    bzero(&wr,sizeof(wr));

    wr.aio_buf = (char *)malloc(BUFFER_SIZE);
    memcpy(wr.aio_buf, "hello world\n", strlen("hello world\n"));
    wr.aio_fildes = fd;
    wr.aio_nbytes = 1024;
    wr.aio_offset = 0;
    wr.aio_lio_opcode = LIO_WRITE;   ///lio操作类型为异步写

    //将异步写事件添加到list中
    listio[1] = &wr;

    /* 使用lio_listio发起一系列请求
       LIO_WAIT 等待队列中所有都完成
       LIO_NOWAIT 立即返回 不等待 
     */
    ret = lio_listio(LIO_WAIT,listio,MAX_LIST,NULL);

    //当异步读写都完成时获取他们的返回值
    ret = aio_return(&rd);
    printf("\n读返回值:%d",ret);
    printf("\n数据:%s", (char *)rd.aio_buf);

    ret = aio_return(&wr);
    printf("\n写返回值:%d",ret);
    printf("\n数据:%s", (char *)wr.aio_buf);

    return 0;
}
```

运行结果如下：

![lio_listio 运行结果](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201711/20171128011358944.png)

### 6.2 AIO 通知

现在我们已经看过了可用的 AIO 函数，本节将深入介绍对异步通知可以使用的方法。我们将通过信号和函数回调来探索异步函数的通知机制。

#### 6.2.1 使用信号进行异步通知 

使用信号进行进程间通信（IPC）是 UNIX 中的一种传统机制，AIO 也可以支持这种机制。在这种范例中，应用程序需要定义`信号处理程序`，在**产生指定的信号时就会调用这个处理程序**。应用程序然后配置一个`异步请求`将在请求完成时产生一个信号。作为信号上下文的一部分，特定的 aiocb 请求被提供用来记录多个可能会出现的请求。

```c
void setup_io( ... )
{
  int fd;
  struct sigaction sig_act;
  struct aiocb my_aiocb;
 
  ...
 
  /* Set up the signal handler */
  sigemptyset(&sig_act.sa_mask);
  sig_act.sa_flags = SA_SIGINFO;
  sig_act.sa_sigaction = aio_completion_handler;
 
 
  /* Set up the AIO request */
  bzero( (char *)&my_aiocb, sizeof(struct aiocb) );
  my_aiocb.aio_fildes = fd;
  my_aiocb.aio_buf = malloc(BUF_SIZE+1);
  my_aiocb.aio_nbytes = BUF_SIZE;
  my_aiocb.aio_offset = next_offset;
 
  /* Link the AIO request with the Signal Handler */
  my_aiocb.aio_sigevent.sigev_notify = SIGEV_SIGNAL;
  my_aiocb.aio_sigevent.sigev_signo = SIGIO;
  my_aiocb.aio_sigevent.sigev_value.sival_ptr = &my_aiocb;
 
  /* Map the Signal to the Signal Handler */
  ret = sigaction( SIGIO, &sig_act, NULL );
 
  ...
 
  ret = aio_read( &my_aiocb );
 
}
 
 
void aio_completion_handler( int signo, siginfo_t *info, void *context )
{
  struct aiocb *req;
 
  /* Ensure it's our signal */
  if (info->si_signo == SIGIO) {
 
    req = (struct aiocb *)info->si_value.sival_ptr;
 
    /* Did the request complete? */
    if (aio_error( req ) == 0) {
 
      /* Request completed successfully, get the return status */
      ret = aio_return( req );
 
    }
 
  }
 
  return;
}
```

我们在 aio_completion_handler 函数中设置信号处理程序来捕获 SIGIO 信号。然后初始化 aio_sigevent 结构产生 SIGIO 信号来进行通知（这是通过 sigev_notify 中的 SIGEV_SIGNAL 定义来指定的）。当读操作完成时，信号处理程序就从该信号的 si_value 结构中提取出 aiocb，并检查错误状态和返回状态来确定 I/O 操作是否完成。

对于性能来说，这个处理程序也是通过请求下一次异步传输而继续进行 I/O 操作的理想地方。采用这种方式，在一次数据传输完成时，我们就可以立即开始下一次数据传输操作。

#### 6.2.2 使用回调函数进行异步通知 

另外一种通知方式是系统回调函数。这种机制不会为通知而产生一个信号，而是会**调用用户空间的一个函数**来实现通知功能。我们在 sigevent 结构中设置了对 aiocb 的引用，从而可以唯一标识正在完成的特定请求。

下面给出一个使用回调函数的例子：

```c
#include<stdio.h>
#include<string.h>
#include<stdlib.h>
#include<unistd.h>
#include<errno.h>
#include <aio.h>
#include<fcntl.h>

#define BUFSIZE 100

void call_back_handler(sigval_t sigval){
    struct aiocb *req;

    req = (struct aiocb*)sigval.sival_ptr;
    if (aio_error(req) == 0) {
        printf("================\n");
        printf("call back handler\n");
        printf("print from callback:\n%s",(char*)req->aio_buf);
        printf("================\n");
    }
}

int main(void) {
    int fd, i, ret;
    struct aiocb my_aiocb;

    if((fd = open("test.txt",O_RDONLY))<0)
        perror("open");

     /* Set up the AIO request */
    bzero((char *)&my_aiocb,sizeof(struct aiocb));
    my_aiocb.aio_buf = malloc(BUFSIZE + 1);
    my_aiocb.aio_fildes = fd;
    my_aiocb.aio_nbytes = BUFSIZE;
    my_aiocb.aio_offset = 0;

    /* Link the AIO request with a thread callback */
    my_aiocb.aio_sigevent.sigev_notify = SIGEV_THREAD;
    my_aiocb.aio_sigevent.sigev_notify_function = call_back_handler;
    my_aiocb.aio_sigevent.sigev_notify_attributes = NULL;
    my_aiocb.aio_sigevent.sigev_value.sival_ptr = &my_aiocb;

    if((ret = aio_read(&my_aiocb))<0)
        perror("aio_read");

    i = 0;
    while (i++ < 5) {
        printf("in main dead loop..\n");
        sleep(1);
    }

    return 0;
}
```

运行结果如下：

![toast_callback 运行结果](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201711/20171128011410020.png)

在创建自己的 aiocb 请求之后，我们使用 SIGEV_THREAD 请求了一个线程回调函数来作为通知方法。然后我们调用 aio_read 进行读操作，`main()函数`最后的 while() 循环体现出了 aio 的异步思想。

### 6.3 AIO 优化

proc 文件系统包含了两个虚拟文件，它们可以用来对异步 I/O 的性能进行优化：

- `/proc/sys/fs/aio-nr` 文件提供了系统范围异步 I/O 请求现在的数目。
- `/proc/sys/fs/aio-max-nr` 文件是所允许的并发请求的最大个数。最大个数通常是 64KB，这对于大部分应用程序来说都已经足够了。

## 七、总结

### 7.1 阻塞 IO 和非阻塞 IO 区别

调用阻塞IO 会一直阻塞对应的进程直到操作完成，而非阻塞IO 在 kernel 还准备数据的情况下会立刻返回。

### 7.2 同步 IO 和异步 IO 区别

POSIX 给出了同步IO 和异步IO 的定义：

- A synchronous I/O operation causes the requesting process to be blocked until that I/O operation completes.
- An asynchronous I/O operation does not cause the requesting process to be blocked.

按照这个定义，阻塞IO、非阻塞IO、IO多路复用和信号驱动IO 都属于同步IO。在异步IO 期间，用户进程不需要去检查 IO 操作的状态，也不需要主动的去拷贝数据。

### 7.3 模型比较

![模型比较](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201711/20171123152009302.png)

| IO 模型 | 等待数据阶段| 复制数据阶段| 具体实现 |
|:------------- |:------------- |:----- |:----- |
| 阻塞 IO | Block | Block | |
| 非阻塞 IO | Unblock | Block | 利用轮询 |
| IO 多路复用 | Block  | Block | Select() |
| 信号驱动 IO | Unblock | Block | 利用信号 |
| 异步 IO | Unblock | Unblock | AIO |
