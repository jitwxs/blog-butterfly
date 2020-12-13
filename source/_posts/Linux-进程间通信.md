---
title: Linux 进程间通信
tags: 进程通信
categories: Linux
typora-root-url: ..
abbrlink: 6c8041c0
date: 2017-12-28 01:05:01
icons: [fas fa-fire red]
related_repos:
  - name: process_comm
    url: https://github.com/jitwxs/blog-sample/blob/master/Linux/process_comm
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---

我们知道，进程是一个程序的一次执行，是系统资源分配的最小单元。这里所说的进程一般是指运行在用户态的进程，而由于处于用户态的不同进程间是彼此隔离的，但是它们很可能需要相互发送一些信息，好让对方知道自己的进度等情况，像这样进程间传递信息就叫`进程间通信`。

## 一、什么是进程间通信

### 1.1 进程间通信的作用

（1）**数据传输**
一个进程需要将它的数据发送给另一个进程，发送的数据量在一个字节到几兆字节之间。

（2）**共享数据**
多个进程想要操作共享数据，一个进程对共享数据的修改，别的进程应该立刻看到。

（3）**通知事件**
一个进程需要向另一个或一组进程发送消息，通知它（它们）发生了某种时间（如子进程终止时要通知父进程）。

（4）**资源共享**
多个进程之间共享同样的资源，需要内核提供锁和同步机制。

（5）**进程控制**
有些进程希望完全控制另一个进程的执行（如Debug进程），此时控制进程希望能够拦截另一个进程的所有陷入和异常，并能够及时知道它的状态改变，

### 1.2 进程间通信的分类

进程间的通信分为同一主机上的进程通信和不同主机上的进程间通信。

对于同一主机上的进程通信，分为：

- UNIX进程间通信
 - `无名管道（PIPE）`
 - `有名管道（FIFO）`
 - `信号（Signal）`

- System Ⅴ进程间通信
 - `共享内存（Share Memory）`
 - `信号量（Semaphore）`
 - `消息队列（Message Queue）`

对于不同主机上的进程通信，分为以下两种：

- `PRC`：Remote Procedure Call 远程过程调用

- `Socket`：当前最为流行的网络通信方式

这两个不在本文的探讨范围内，关于Socket，可以参考文章[《Linux socket 编程》](/f2ee55a7.html)

## 二、UNIX进程间通信

### 2.1 无名管道

管道是Linux支持的最初Unix IPC形式之一，具有以下特点：

（1）管道是**半双工的**，数据只能向一个方向流动；需要双方通信时，需要建立起**两个**管道；

（2）只能用于父子进程或者兄弟进程之间（**具有亲缘关系**的进程）；

（3）单独构成一种独立的文件系统：管道对于管道两端的进程而言，就是一个文件，但它不是普通的文件，它不属于某种文件系统，而是自立门户，单独构成一种文件系统，并且**只存在于内存**中。

（4）数据的读出和写入：一个进程向管道中写的内容被管道另一端的进程读出。写入的内容每次都添加在管道缓冲区的**末尾**，并且每次都是从缓冲区的**头部**读出数据。

#### 2.1.1 相关函数

##### 2.1.1.1 创建无名管道

```c
#include <unistd.h>

int pipe(int fd[2]);
```

返回的fd[0]用于读，fd[1]用于写。因此，一个进程在由`pipe()`创建管道后，一般再fork一个子进程，然后通过管道实现父子进程间的通信。

创建成功返回0，创建失败返回-1并设置errno。

| errno | 含义 |
| ------------- |:-------------|
| EMFILE | 没有空闲的文件描述符 |
| ENFILE | 系统文件表已满 |
| EFAULT | fd数组无效 |
 
##### 2.1.1.2 读写无名管道

读无名管道采用无缓冲区IO方式实现：

```c
#include <unistd.h>

/*
 从fd所指向的文件中读取count字节内容存储在buf所指的空间中
 
 fd：打开的文件描述符
 buf：读出数据存储位置
 count：读取数据的大小
*/
ssize_t read(int fd, void *buf, size_t count);
```

写无名管道：

```c
#include <unistd.h>

// 从buf所指向的缓冲区向管道中写入count字节
ssize_t write(int fd, const void *buf, size_t count);
```

试图向已经填满的管道中写，会阻塞。为避免交叉写入，建议每次写入的数据小于或等于系统`PIPE_BUF`缓冲区大小。

#### 2.1.2 使用示例

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

#define MAXSIZE 128

int main(void) {
    int fd[2];
    pid_t pid;

    if (pipe(fd) != 0) {
        printf("创建无名管道失败\n");
        perror("pipe");
    }

    if ((pid = fork()) == -1) {
        printf("创建子进程失败\n");
        perror("fork");
    } else if (pid == 0) { // 子进程发送数据
        char msg[MAXSIZE];
        printf("send msg: ");
        fgets(msg, MAXSIZE, stdin);
        close(fd[0]);
        write(fd[1], msg, strlen(msg));
    } else { // 父进程接收数据
        char rec[MAXSIZE];
        wait(NULL);
        close(fd[1]);
        read(fd[0], rec, sizeof(rec));
        printf("receive msg: %s", rec);
    }

    return 0;
}
```

```
wxs@ubuntu:~/process_comm/pipe$ ./main 
send msg: hello world
receive msg: hello world
```

#### 2.1.3 POSIX2 下的无名管道

POSIX2 提供了两个使用管道机制实现简单进程间通信的函数：

```c
#include <stdio.h>

// 将一个程序的输入或输出重定向
FILE *popen(const char *command, const char *type);

// 关闭相应的流对象
int pclose(FILE *stream);
```

`popen`函数内部fork了一个子进程，并执行第一个参数上的程序，返回一个文件流指针。第一个参数指向要执行的命令的指针，第二个参数表示I/O模式的类型。若要从命令的数据结果读数据，需要有`r`权限；若要向命令输入数据，需要有`w`权限。

### 2.2 有名管道

有名管道和无名管道基本相同，但也有一些显著的不同：

（1）**无名管道只能在有亲缘关系的进程间通信，有名管道可以在没有亲缘关系的进程间通信。**

（2）无名管道在进程通信结束之后会消失，但是**有名管道是一直保存的**，保存的是有名管道文件，不保存管道的内容。

#### 2.2.1 相关函数

##### 2.2.1.1 创建有名管道

```c
#include <sys/types.h>
#include <sys/stat.h>

int mkfifo(const char *pathname, mode_t mode);
```

根据参数**建立特殊的有名管道文件**，**文件必须不存在**。参数mode为该文件的权限。执行成功返回0，执行失败返回-1并设置errno。

##### 2.2.1.2 读写有名管道

通过 `write` 和 `read` 系统调用执行读写操作，使用有名管道，一定要**使用两个进程**分别打开其读端和写端：

```c
#include <unistd.h>

ssize_t read(int fd, void *buf, size_t count);

ssize_t write(int fd, const void *buf, size_t count);
```

#### 2.2.2 使用示例

```c head.h
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/stat.h>

#define FILE_PATH "./fifo"

#define MAXSIZE 128

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

```c read.c
#include "head.h"

int main(void) {
    int fd;
    char buf[MAXSIZE];

    if (!check_fifo_exist())
        if (mk_fifo() == -1) {
            printf("创建FIFO文件出错");
            perror("mkfifo");
        }

    if ((fd = open(FILE_PATH, O_RDONLY)) == -1) {
        printf("打开FIFO文件出错");
        perror("open");
    }

    while(1) {
        memset(buf, 0, MAXSIZE);
        if (read(fd, buf, MAXSIZE) != 0) {
            if (strcmp("exit\n", buf) == 0)
                break;
            printf("rec msg: %s", buf);
        }
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

```c write.c
#include "head.h"

int main(void) {
    int fd;
    char buf[MAXSIZE];

    if (!check_fifo_exist())
        if (mk_fifo() == -1) {
            printf("创建FIFO文件出错");
            perror("mkfifo");
        }

    if ((fd = open(FILE_PATH, O_WRONLY)) == -1) {
        printf("打开FIFO文件出错");
        perror("open");
    }

    while(1) {
        printf("send msg: ");
        fgets(buf, MAXSIZE, stdin);
        write(fd, buf, strlen(buf));
        if (strcmp("exit\n", buf) == 0)
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

![FIFO](/images/posts/20171226162636607.png)

### 2.3 信号

信号是Linux的一种中断处理机制，信号可以导致一个正在运行着的进程被另一个异步进程终端。

一个进程创建一个信号用于发送给另一个进程叫`发送一个信号`；内核创建一个信号叫`生成一个信号`；一个进程向自己发送一个信号叫`唤起一个信号`。

如果一个信号被发送并且没有引起任何动作，称信号处于`等待状态`；如果一个信号被正确发送到一个进程称为该`信号被递送`；如果一个信号的递送导致一段处理程序被执行，称为该`信号被捕捉`。

#### 2.3.1 产生信号

（1）当用户按某些终端按键时，产生信号
在终端上按 `DELETE` 键通常产生中断信号（`SIGINT`），这是中断程序的一种方法。

（2）硬件异常产生信号
除数为0、无效的存储访问等等，这些条件通常由硬件检测到，并将其通知内核，然后内核为该条件发生时正在运行的进程产生适当的信号。例如，对执行一个无效存储访问的进程产生一个`SIGSEGV`。

（3）终止进程信号
进程用 `kill 函数`可将信号发送给另一个进程或进程组，常用此命令终止一个失控的后台进程。

（4）软件异常产生信号
检测到某种软件条件已经发生，并将其通知有关进程时也会产生信号。

#### 2.3.2 处理信号

（1）忽略信号
大多数信号都可以使用这种处理方式，但有两种信号不能被忽略，`SIGKILL`和`SIGSTOP`。这两种信号不能被忽略的原因是：它们**向超级用户提供一种使进程中止或停止的可靠方法。**

（2）捕捉信号
通知内核在某种信号发生时调用一个用户函数。在用户函数中，可执行用户希望对这种事件进行的处理。例如捕捉到 `SIGCHLD` 信号，则表示子进程已经终止，所以此信号的捕捉函数可以调用`waitpid()`以取得该子进程ID以及它的终止状态。

（3）执行系统默认操作
Linux系统对任何一个信号都规定了一个默认的操作，在`/usr/include/asm-generic/signal.h`中可以查看。

#### 2.3.3 信号基本操作

##### 2.3.3.1 产生信号

（1）传递一个信号给指定进程

```c
#include <sys/types.h>
#include <signal.h>

/*
 sig：要发送的信号值
 pid：被发送信号的进程id，其具有以下取值：
 	> 0  发送给进程的PID值为pid的进程
 	= 0  发送给和当前进程在同一组的所有进程
 	= -1 发送给系统内所有进程
 	< 0  发送给进程组识别码为pid绝对值的所有进程
*/
int kill(pid_t pid, int sig);
```

（2）传递一个信号给当前进程

```c
#include <signal.h>

// sig为要发送的信号值
int raise(int sig);
```

（3）唤醒一个进程和设置定时

```c
#include <unistd.h>

// 在seconds时发送SIGALRM给当前进程，单位为s
unsigned int alarm(unsigned int seconds);
```

##### 2.3.3.2 安装信号

（1）signal()

```c
#include <signal.h>

typedef void (*sighandler_t)(int);

sighandler_t signal(int signum, sighandler_t handler);
```

- signum 接收到的信号

- handler 接收到此信号后的处理代码入口或者以下宏：`SIG_ERR`、`SIG_DF`L、`SIG_IGN`。

（2）sigaction()

```c
#include <signal.h>

int sigaction(int signum, const struct sigaction *act,
	struct sigaction *oldact);
```

- signum 接收到的信号

- act 指定预安装的信号处理信息

- oldact 返回执行程序前信号处理信息

##### 2.3.3.3 阻塞信号

Linux使用进程信号掩码集合的概念来管理阻塞信号，进程信号掩码用一个信号集来表示，信号掩码集指示了当前进程所有阻塞的信号。

（1）信号阻塞和信号忽略的区别
- 信号忽略：**系统仍传递该信号**，只是**相应进程对该信号不做任何处理**

- 信号阻塞：**系统不传递该信号**，显示该进程无法接受到信号直到进程的信号掩码集发生改变

（2）信号掩码集操作函数
- 清空信号掩码集：`sigemptyset()`

- 完全填空信号掩码集：`sigfillset()`

- 添加信号到信号掩码集：`sigaddset()`

##### 2.3.3.4 等待信号

`pause()` 函数将使当前进程处于等待状态，直到一个信号出现：

```c
#include <unistd.h>

int pause(void);
```

`sigsuspend()`函数可将调用线程的当前信号掩码替换为 `sigmask` 指向的信号集，然后挂起该线程，直到传递一个信号为止，该信号的操作将是执行信号捕获函数或终止进程：

```c
#include <signal.h>

int sigsuspend(const sigset_t *mask);
```

如果操作是要终止进程，则 `sigsuspend()` 将永远不会返回。如果操作是要执行信号捕获函数，则 `sigsuspend()` 将在信号捕获函数返回后返回，同时信号掩码恢复为调用该函数之前已存在的信号集。

注意，该函数**不能阻塞无法忽略的信号**。

#### 2.3.4 使用示例

```c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

void my_func(int sig_no) {
    if (sig_no == SIGINT)
        printf("Catch SIGINT...\n");
    else if (sig_no == SIGQUIT)
        printf("Catch SIGQUIT...\n");
}

int main(void) {
    printf("waiting for signal SIGINT or SIGQUIT\n");
    signal(SIGINT, my_func);
    signal(SIGQUIT, my_func);
    pause();
    return 0;
}
```

![signal](/images/posts/20171226194852780.png)

## 三、System Ⅴ进程间通信

### 3.1 共享内存

共享内存进程间通信机制主要实现**进程间大量的数据传输**。

#### 3.1.1 相关函数

（1）创建共享内存

```c
#include <sys/ipc.h>
#include <sys/shm.h>

/*
 key：建立共享内存对象的值
 size：欲创建的共享内存段大小(KB)
 shmflg：用来表示共享内存段的创建标识
*/
int shmget(key_t key, size_t size, int shmflg);
```

（2）共享内存控制

```c
#include <sys/ipc.h>
#include <sys/shm.h>

/*
 shmid：要操纵的共享内存标识符，为shmget函数返回值
 cmd：要执行的操作，在ipc.h中定义
 buf：临时共享内存变量信息
*/
int shmctl(int shmid, int cmd, struct shmid_ds *buf);
```

（3）映射共享内存对象

```c
#include <sys/types.h>
#include <sys/shm.h>

/*
 shmid：共享内存标识符，为shmget函数返回值
 shmaddr：指定共享内存映射地址。若非0，则用此值作为映射共享内存地址；若为0，则由系统来选择映射的地址
 shmflg：指定共享内存段的访问权限和映射条件
*/
void *shmat(int shmid, const void *shmaddr, int shmflg);
```

（4）分离共享内存
使用完共享内存空间后，需要将其与当前进程进行分离：

```c
#include <sys/types.h>
#include <sys/shm.h>

/*
 shmaddr:要与当前进程分离的共享内存标识ID
*/
int shmdt(const void *shmaddr);
```

（5）共享内存段数据处理
使用共享内存空间处理数据时，通常需要使用块数据处理函数，在string.h中声明：

```c
#include <string.h>

// 块数据复制函数
void *memccpy(void *dest, const void *src, int c, size_t n);

// 块数据查找函数
void *memchr(const void *s, int c, size_t n);

// 块数据比较函数
int memcmp(const void *s1, const void *s2, size_t n);

// 块数据复制函数
void *memcpy(void *dest, const void *src, size_t n);

// 块数据移动函数
void *memmove(void *dest, const void *src, size_t n);

// 块数据设置函数
void *memset(void *s, int c, size_t n);
```

#### 3.1.2 使用示例

```c head.h
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <sys/wait.h>
#include <sys/types.h>

#define SHM_SIZE 4096

#define MSG_SIZE 128

#define SHM_KEY 2017

struct msg {
    char msg[MSG_SIZE];
    int flag;
};
```

```c read.c
#include "head.h"

int main(void) {
    int shmid;
    void *shm_addr;
    struct msg *buf;

    // Step1: 创建共享内存
    if ((shmid = shmget(SHM_KEY, SHM_SIZE, IPC_CREAT | 0666)) == -1) {
        printf("创建共享内存失败\n");
        perror("shmget");
    }

    // Step2: 映射共享内存对象
    if ((shm_addr = shmat(shmid, NULL, 0)) == (void *)-1) {
        printf("映射共享内存失败\n");
        perror("shmat");
    }

    buf = (struct msg*)shm_addr;

    // Step3: 从共享内存中读取数据
    while(1) {
        if (buf->flag == 1) {
            printf("rec msg: %s", buf->msg);
            if (strcmp("exit\n", buf->msg) == 0)
                break;
            buf->flag = 0;
        }
    }

    // Step4: 分离共享内存
    shmdt(shm_addr);

    // Step5: 删除共享内存
    shmctl(shmid, IPC_RMID, NULL);

    return 0;
}
```

```c write.c
#include "head.h"

int main(void) {
    int shmid;
    void *shm_addr;
    struct msg *buf;

    // Step1: 创建共享内存
    if ((shmid = shmget(SHM_KEY, SHM_SIZE, IPC_CREAT | 0666)) == -1) {
        printf("创建共享内存失败\n");
        perror("shmget");
    }

    // Step2: 映射共享内存对象
    if ((shm_addr = shmat(shmid, NULL, 0)) == (void *)-1) {
        printf("映射共享内存失败\n");
        perror("shmat");
    }

    buf = (struct msg*)shm_addr;

    // Step3: 写入数据到共享内存
    while(1) {
        printf("send msg: ");
        fgets(buf->msg, MSG_SIZE, stdin);
        buf->flag = 1;
        if (strcmp("exit\n", buf->msg) == 0)
            break;
    }

    // Step4: 分离共享内存
    shmdt(shm_addr);

    // Step5: 删除共享内存
    shmctl(shmid, IPC_RMID, NULL);

    return 0;
}
```

![share_memory](/images/posts/20171226215128936.png)

### 3.2 信号量

信号量机制主要实现进程间同步。信号量是一个特殊的变量，程序对其访问都是原子操作，且只允许对它进行`等待`(P)和`发送`(V)操作。

由于信号量只能进行两种操作等待和发送信号，即`P(sv)`和`V(sv)`,他们的行为是这样的：
- P(sv)：如果sv的值大于零，就给它减1；如果它的值为零，就挂起该进程的执行
- V(sv)：如果有其他进程因等待sv而被挂起，就让它恢复运行，如果没有进程因等待sv而挂起，就给它加1.

#### 3.2.1 相关函数

（1）创建信号量集合
使用信号量前，需要创建一个信号量集合。

```c
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/sem.h>

/*
 key：建立信号量对象的值
 nsems：创建信号量的个数，基本为1
 semflg：标识信号量集合的权限
*/
int semget(key_t key, int nsems, int semflg);
```

（2）控制信号量集合/信号量
在Linux中，使用 `semctl` 函数对一个信号量集合以及信号量集合中的信号进行操作。

```c
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/sem.h>

/*
 semid：要操作的信号量集合标识符
 semnum：集合中信号量的个数。若为0，则与具体信号量无关，与整个集合信号量相关
 cmd：要执行的操作，宏定义于ipc.h中
 参数四：根据cmd的具体操作为senum联合体
*/
int semctl(int semid, int semnum, int cmd, ...);
```

```c
union semun {
	int val;
	struct semid_ds *buf;
	unsigned short *arry;
	struct seminfo *_buf;
	void *_pad;
}
```

（3）信号量操作

```c
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/sem.h>

/*
 semid：要操作的信号量集合标识符
 sops：struct sembuf结构变量
*/
int semop(int semid, struct sembuf *sops, size_t nsops);
```

```c
struct sembuf {
	unsigned short sem_num; // //除非使用一组信号量，否则它为0  
	short sem_op; // 信号量在一次操作中需要改变的数据，通常是两个数：一个是-1，即P操作，一个是+1，即V操作。  
	short sem_flag; // 通常为SEM_UNDO,使操作系统跟踪信号，并在进程没有释放该信号量而终止时，操作系统释放信号量  
}
```

#### 3.2.2 使用示例

使用信号量和共享内存配合实现生产者/消费者模型：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/shm.h>
#include <sys/sem.h>
#include <sys/ipc.h>
#include <sys/wait.h>

#define TIMES 5
#define FLAG (IPC_CREAT | 0666)
#define MAX_BUFFER_SIZE 3

typedef struct {
    int bottom;
    int top;
    int data[MAX_BUFFER_SIZE];
}BUFFER;

BUFFER *pBuffer = NULL;
int sem_consume;
int sem_produce;
int shm_buffer;

void init() {
    union semun {
        int val;
        struct semid_ds *buf;
        unsigned short *array;
    }arg;

    // Step1: 创建并映射共享内存
    shm_buffer = shmget(IPC_PRIVATE, sizeof(BUFFER), FLAG);
    pBuffer = shmat(shm_buffer, 0, 0);
    memset(pBuffer,0,sizeof(BUFFER));
   
    // Step2: 创建消费者和生产者信号量
    sem_consume = semget(IPC_PRIVATE, 1, FLAG);
    arg.val = 0;
    semctl(sem_consume, 0, SETVAL, arg);

    sem_produce = semget(IPC_PRIVATE, 1, FLAG);
    arg.val = MAX_BUFFER_SIZE;
    semctl(sem_produce, 0, SETVAL, arg);
}

void destory() {
    // Step4: 分离并删除共享内存
    shmdt(pBuffer);
    shmctl(shm_buffer, IPC_RMID, NULL);

    // Step5: 删除信号量
    semctl(sem_consume,0,IPC_RMID);
    semctl(sem_produce,0,IPC_RMID);
}

int main(void) {
    int pid,i;
    struct sembuf sbuf;

    init();
    // Step3: 生产者进程中做V操作，消费者进程中做P操作
    if ((pid = fork()) > 0) {
        for(i=0; i<TIMES; i++) {
            sbuf.sem_num = 0;
            sbuf.sem_op = -1; // 做P操作
            sbuf.sem_flg = 0;

            semop(sem_consume,&sbuf,1);
            system("date | awk '{print $4}'");
            printf("consumer get %6d,pos=%d\n",
                    pBuffer->data[pBuffer->bottom], pBuffer->bottom);
            pBuffer->bottom = (pBuffer->bottom+1) % MAX_BUFFER_SIZE;
            semop(sem_produce,&sbuf,1);

            sleep(2);
        }
        wait(NULL);
    } else if (pid == 0) {
        for(i=0; i<TIMES; i++){
            sbuf.sem_num = 0;
            sbuf.sem_op = 1; // 做V操作
            sbuf.sem_flg = 0;

            semop(sem_produce,&sbuf,1);
            pBuffer->data[pBuffer->top] = (rand()%100)+i+1;
            system("date | awk '{print $4}'");
            printf("produce put %6d,pos=%d\n",
                    pBuffer->data[pBuffer->top], pBuffer->top);
            pBuffer->top=(pBuffer->top+1)%MAX_BUFFER_SIZE;
            semop(sem_consume,&sbuf,1);

            sleep(1);
        }
        exit(0);
    }

    destory();
    return 0;
}
```

![semaphore](/images/posts/20171227094515684.png)

### 3.3 消息队列

消息队列提供了一种从一个进程向另一个进程发送一个数据块的方法。每个数据块都被认为含有一个类型，接收进程可以独立地接收含有不同类型的数据结构。

#### 3.3.1 相关函数

（1）创建消息队列
使用一个消息队列前，需要使用 `msgget` 函数创建消息队列：

```c
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>

/*
 key：建立消息队列对象的值
 msgflg：确定消息队列的访问权限
*/
int msgget(key_t key, int msgflg);
```

（2）控制消息队列
创建消息队列后，可以对该消息队列的基本属性进行控制修改：

```c
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>

/*
 msqid：消息队列标识符，为msgget()函数返回值
 cmd：执行的操作：IPC_RMID、IPC_SET、IPC_STAT、IPC_INFO
*/
int msgctl(int msqid, int cmd, struct msqid_ds *buf);
```

（3）向消息队列中发送消息
`msgsnd()` 函数用于将新的消息添加到消息队列尾端，每个消息中包含一个正长整型的字段、一个非负长度以及一个实际数据字节。

```c
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>

/*
 msqid：消息队列标识符
 msgp：用户定义的缓冲区
 msgsz：消息的大小
 msgflg：控制当前消息队列满或队列消息到达系统范围的限制时将要发生的事情
*/
int msgsnd(int msqid, const void *msgp, size_t msgsz, int msgflg);
```

（4）从消息队列中接收消息

```c
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>

/*
 msqid：要读的消息队列的标识符
 msgp：临时消息数据结构，用来保存读取的信息
 msgsz：消息的大小
 msgtype：如果为0，就获取队列中的第一个消息。如果大于0，获取具有相同消息类型的第一个信息。如果小于0，就获取类型等于或小于绝对值的第一个消息。
 msgflg：指定所需类型消息不在队列上时要采取的操作
*/
ssize_t msgrcv(int msqid, void *msgp, size_t msgsz, long msgtyp,
                    int msgflg);
```

#### 3.3.2 使用示例

```c head.h
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>

#define PATH "./"

#define QUEUE_ID 200

#define READ_ID 300

#define WRITE_ID 301

#define MSG_SIZE 128

struct msg {
    int type;
    char buf[MSG_SIZE];
};
```

```c read.c
include "head.h"

int main(void) {
    key_t key;
    int msgid;
    struct msg my_msg;

    // Step1: 创建消息队列
    key = ftok(PATH, QUEUE_ID);
    if ((msgid = msgget(key, IPC_CREAT | 0666)) < 0) {
        printf("创建消息队列失败\n");
        perror("msgget");
    }

    my_msg.type = READ_ID;
    while(1) {
        // Step2: 从消息队列中读取消息
        msgrcv(msgid, &my_msg, MSG_SIZE, 0, 0);
        if (strcmp("exit\n", my_msg.buf) == 0)
            break;
        printf("rec msg: %s", my_msg.buf);
    }

    //Step3: 删除消息队列
    msgctl(msgid, IPC_RMID, 0);

    return 0;
}
```

```c write.c
#include "head.h"

int main(void) {
    key_t key;
    int msgid;
    struct msg my_msg;

    // Step1: 创建消息队列
    key = ftok(PATH, QUEUE_ID);
    if ((msgid = msgget(key, IPC_CREAT | 0666)) < 0) {
        printf("创建消息队列失败\n");
        perror("msgget");
    }

    my_msg.type = WRITE_ID;
    while(1) {
        printf("send msg: ");
        fgets(my_msg.buf, MSG_SIZE, stdin);

        // Step2: 发送消息到消息队列
        msgsnd(msgid, &my_msg, strlen(my_msg.buf), 0);

        if (strcmp("exit\n", my_msg.buf) == 0)
            break;
    }

    //Step3: 删除消息队列
    msgctl(msgid, IPC_RMID, 0);

    return 0;
}
```

![message_queue](/images/posts/20171227102845832.png)

## 四、总结

| 类型 | 优点 |  缺点 | 创建或打开IPC函数 | IPC操作函数 |
|:-----|:-----|:-----|:-----:|:-----:|
| 无名管道 | 文件的形式存在，进程通信结束后，**文件消失** | 通信的进程关系**一定要是父子进程**，承载信息量少，只能承载无格式的数据流以及缓冲区大小受限 | pipe()、poen()、pclose() | read()、write() |
| 有名管道 | 文件形式存在，进程通信结束后，**文件存在，文件内容不存在**，**可用于没有亲缘关系**的进程之间 | 承载信息量少，只能承载无格式的数据流以及缓冲区大小受限 | mkfifo() | read()、write() |
| 信号 | 通信比较复杂，用于通知接受进程有某种事件发生 | 只针对系统的几种特殊条件下使用 | signal()、sigaction() | sigprocmask()、sigemptyset()、sigfillset()、sigaddset()、sigismember() |

---

| 类型 | 优点 |  缺点 | 创建或打开IPC函数 | 控制IPC操作函数 | IPC操作函数 |
|:-----|:-----|:-----|:-----:|:-----:|:-----:|
| 消息队列 | 克服了信号承载信息量少，可独立于发送和接收进程，同时通过发送消息还可以避免有名管道的同步和阻塞问题。接收程序可以通过消息类型**有选择地接收数据** | 每个数据都有一个最大长度的限制 | msgget() | msgctl() | msgsnd()、msgrcv() |
| 信号量 | 主要**作为进程间以及同一进程不同线程之间的同步手段** | 仅用于解决多个进程对同一资源的访问竞争问题 | semget() | semctl() | semop() |
| 共享内存 | 非常方便，函数接口简单，数据的共享使进程间数据无需传送，而是直接访问内存，加快程序效率 | **没有提供同步机制**，需要借助其他手段来进行进程间的同步工作 | shmget() | shmctl() | shmat()、shmdt()|

