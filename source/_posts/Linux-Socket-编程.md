---
title: Linux Socket 编程
tags: Socket
categories: Linux
typora-root-url: ..
abbrlink: f2ee55a7
date: 2017-11-15 01:34:36
icons: [fas fa-fire red]
related_repos:
  - name: linux_socket
    url: https://github.com/jitwxs/blog-sample/blob/master/Linux/linux_socket
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---

## 前言

 进程通信的概念最初来源于单机系统，由于每个进程都在自己的地址范围内运行，为保证两个相互通信的进程之间既互不干扰又协调一致工作，操作系统为进程通信提供了相应设施，如：管道（`pipe`）、消息（`message`）、共享存储区（`shared memory`）和信号量（`semaphore`)等。

但是这都仅限于用在本机进程之间通信。网络间进程通信要解决的是不同主机进程间的相互通信问题为此，引入了`套接字`。

## 一、套接字

`套接字（socket）`，在Linux环境下，用于表示进程间通信的特殊文件类型（伪文件）。我们知道，在TCP/IP协议中：

- IP地址：**在网络环境中唯一标识一台主机**

- 端口号：**在主机中唯一标识一个进程**

- IP地址+端口号：**在网络环境中唯一标识一个进程**

这个在网络中被唯一标识的进程，被称为`socket`。在网络通信中，套接字一定是**成对出现**的。这两个socket组成的`socket pair`就**唯一标识一个连接**。

![socket通信原理](/images/posts/20171114161609619.png)

在TCP/IP模型中，套接字位于**应用层和传输层之间**：

![socket位置](/images/posts/20171114161918044.png)

套接字一般分为以下三种类型：

- **流式套接字**（SOCK_STREAM）
提供可靠的、面向连接的通信流，通过它发送的数据保证原有顺序不变。它使用的是**TCP协议**。

- **数据报套接字**（SOCK_DGRAM）
定义了一种无连接的服务，数据通过相互独立的报文进行传输，是无序的，并且不保证可靠、无差错。它使用的是**UDP协议**。

- **原始套接字**（SOCK_RAW）
允许对底层的协议直接访问，主要用于新的网络协议的开发。它功能强大，但使用复杂。

## 二、预备知识

### 2.1 网络字节序

我们知道，计算机在内存中存放数据有`小端字节序（Little-Endian）`和`大端字节序（Big-Endian）`两种方法。举个简单的例子，对于整型数据`0x12345678`，有以下两种存放形式：

![小端法和大端法](/images/posts/20171114163224086.png)

TCP/IP协议规定了，**网络数据应采用大端字节序**。

因此如果主机是大端字节序，网络传输时不要做转换；如果主机是小端字节序，网络传输时就需要做转换。

为了使网络程序具有可移植性，编写socket程序可以使用以下函数来进行**网络字节序和主机字节序的转换**：

```c
#include <arpa/inet.h>

/*
h表示host，n表示network，l表示32位长整数，s表示16位短整数。
这些函数将参数转换为大端字节序并返回。
*/

/* 主机字节序转网络字节序（32位，用于ip地址）*/
uint32_t htonl(uint32_t hostlong);

/* 主机字节序转网络字节序（16位，用于端口号）*/
uint16_t htons(uint16_t hostshort);

/* 网络字节序转主机字节序（32位，用于ip地址）*/
uint32_t ntohl(uint32_t netlong);

/* 网络字节序转主机字节序（16位，用于端口号）*/
uint16_t ntohs(uint16_t netshort);
```

### 2.2 IP地址转换函数

以IPv4为例，我们平时使用的ip地址像`192.168.1.1`这种形式，属于**点分十进制字符串**，但是计算机内部进行传输时则需要将其转换为**32位的二进制** ，与此同时还要考虑主机字节序和网络字节序的转换问题。

为了简化我们进行socket编程时对ip地址的操作，提供了以下两个IP地址转换函数：

```c
#include <arpa/inet.h>

/*
p代表ip，n代表net。
功能：ip地址的字符串类型（点分十进制）和整型（二进制）的转换，内部已经做了网络字节序的转换。
*/

/*
af：指定了是IPV4(AF_INET)还是IPV6(AF_INET6)
src：源ip地址（字符串）
dst：目标ip地址
*/
int inet_pton(int af, const char *src, void *dst);

/*
af：指定了是IPV4(AF_INET)还是IPV6(AF_INET6)
src：源ip地址
dst：目标ip地址（字符串）
size：dst的大小
*/
const char *inet_ntop(int af, const void *src, char *dst, socklen_t size);
```

注：**早期的ip转换函数已被废弃（不支持IPV6）**，被废弃的函数包括：

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

int inet_aton(const char *cp, struct in_addr *inp);

in_addr_t inet_addr(const char *cp);

char *inet_ntoa(struct in_addr in);

```

### 2.3 struct sockaddr

`struct sockaddr`是一个非常古老的结构体，诞生早于IPv4协议。后来随着网络的发展，不得不对该结构体进行细分，划分出了以下三种：

- `struct sockaddr_in`

- `struct sockaddr_in6`

- `struct sockaddr_un`

![sockaddr](/images/posts/20171114172748320.png)

为了不改变那些使用`struct sockaddr`作为参数的函数（例如bind、accpet、connect），`struct sockaddr`退化成了**泛型**（和 void * 作用相似）。

因此，我们编程时需要根据具体的需求，来确定要定义哪种**被细分**的`struct sockaddr`结构体，然后在函数中将其**强制转换**为`struct sockaddr`类型。

**注：struct sockaddr现在已经被废弃了，不能使用！**

比如在`bind()`函数中：

```c
//假设地址族协议为AF_INET

struct sockaddr_in serv_addr;

bind(lfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr));
```

### 2.4 struct sockaddr_in

因为目前IP地址仍然主要使用IPv4协议，因此`struct sockaddr_in`结构体是我们当前最为常用的`struct sockaddr`结构体，其定义如下：

```c
 struct sockaddr_in {
 	/* 地址族协议 可选为 AF_INET 或 AF_INET6 */
	sa_family_t    sin_family;
	in_port_t      sin_port;   /* 端口号 */
	struct in_addr sin_addr;   /* 网络地址结构体 */
};


struct in_addr {
	uint32_t       s_addr;     /* 网络地址 */
};

```

其实`struct sockaddr_in`内部不止这些属性，但我们只需关心这些。

我们发现`sin_addr`是`struct in_addr`类型，而1`struct in_addr`类型内部只有一个`s_addr`变量。

### 2.5 TCP与UDP运行流程

![TCP](/images/posts/20171114200747792.png)

![UDP](/images/posts/20171114200805172.png)

## 三、相关函数

### 3.1 socket() 

####  3.1.1 概要

```c
#include <sys/types.h>
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
```

成功返回**指向新创建的socket的文件描述符**。失败返回-1，错误信息存放于errno。

socket()打开一个网络通讯端口，如果成功的话，就像open()一样返回一个文件描述符，应用程序可以像读文件一样用read/write在网络上收发数据。

####  3.1.2 参数

**1）domain**：地址族协议，常用的有以下几个参数：

| 参数| 含义|
| ------------- | :----- |
| AF_INET | 最常用，使用TCP或UDP来进行传输，使用IPv4的地址 |
| AF_INET6 | 与上面类似，但使用IPv6的地址 |
| AF_UNIX | 本地协议，使用在本地Unix和Linux系统上 |

**2）type**：协议类型，常用的有以下几个参数：

| 参数| 含义|
| ------------- | :----- |
| SOCK_STREAM | 该协议是按照顺序的、可靠的、数据完整的基于字节流的连接，使用TCP进行传输 |
| SOCK_DGRAM | 该协议是无连接的、固定长度的传输调用。该协议是不可靠的，使用UDP进行传输 |
| SOCK_SEQPACKET | 该协议是双线路的、可靠的连接，发送固定长度的数据包进行传输。必须完整接收包才能进行读取 |
| SOCK_RAW | socket类型提供单一的网络访问，这个socket类型使用ICMP公共协议 |
| SOCK_RDM | 很少使用，提供给数据链路层使用，不保证数据包的顺序 |

**3）protocol**：传**0**即可，表示使用每种协议的默认协议（即SOCK_STREAM使用TCP，SOCK_DGRAM使用UDP）。

### 3.2 bind() 

```c
#include <sys/types.h>
#include <sys/socket.h>

/*
sockfd: socket文件描述符
addr: 构造出IP地址加端口号
addrlen: add的长度
*/
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

服务器程序所监听的网络地址和端口号通常是固定不变的，客户端程序得知服务器程序的地址和端口号就可以向服务器发起连接，因此服务器需要**调用bind()绑定一个固定的网络地址和端口号**。

bind()的作用是将参数sockfd和addr绑定在一起，使sockfd这个用于网络通讯的文件描述符监听addr所描述的地址和端口号。

前面说过，`struct sockaddr *` 相当于一个泛型，因此其具体实现的长度各不相同，所以需要第三个参数**指定结构体的长度**。

成功返回0。 失败返回-1，错误信息存放于errno。

### 3.3 listen() 

```c
#include <sys/types.h>
#include <sys/socket.h>

/*
sockfd: socket文件描述符
backlog: 同时允许和服务端建立连接的客户端数量
*/
int listen(int sockfd, int backlog);
```

成功返回0。失败返回-1，错误信息存放于errno。

使用`cat /proc/sys/net/ipv4/tcp_max_syn_backlog`可以查看系统默认的backlog。

如果客户端连接数达到backlog，新的客户端连接请求会被**忽略**。

### 3.4 accpet()

```c
#include <sys/types.h>
#include <sys/socket.h>

/*
sockfd：socket文件描述符
addr：传出参数，返回连接客户端地址信息，包含IP地址和端口号。如果传NULL，表示不关心客户端的地址
addrlen：传入传出参数，传入addr的大小，传出真正接收到地址结构体的大小
*/
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

三次握手完成后，服务器调用accept()接受连接，如果服务器调用accept()时没有收到客户端的请求，会一直处于**阻塞状态**。

成功返回一个**新的** socket文件描述符，用于和客户端通信。失败返回-1，错误信息存放于errno。

### 3.5 connect()

```c
include <sys/types.h>
#include <sys/socket.h>

/*
sockfd：socket文件描述符
addr：传入参数，指定服务器端地址信息，含端口号和IP地址
addrlen：传入参数，传入sizeof(addr)的大小
*/
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

客户端需要调用connect()连接服务器，connect()和bind()的参数一致，区别在于bind()的参数是**自己的地址**，而connect()的参数**是对方的地址**。

成功返回0。失败返回-1，错误信息存放于errno。

### 3.6 recv() 和 send() 

这两个函数是socket tcp编程下的接收函数(recv)和发送函数(send)。

```c
#include <unistd.h>

ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
```

```c
#include <sys/types.h>
#include <sys/socket.h>

ssize_t recv(int sockfd, void *buf, size_t len, int flags);
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
```

recv和send函数和标准的read和write函数很类似，唯一的差别是第四个参数。

第四个参数是一个整型的标志位，我们可以以**位或**的形式包含系统允许的一系列标志，从而设置在这一次I/O的特性。

通常情况下被**设置为0**，实现普通read和write的功能。函数返回读取的字节个数。

### 3.7 recvfrom() 和 sendto() 

```c
#include <sys/types.h>
#include <sys/socket.h>

/*
src_addr: 接收到的struct socketaddr地址
addrlen: 接收到的struct socketaddr的大小的地址
*/
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                        struct sockaddr *src_addr, socklen_t *addrlen);

/*
dest_addr: 要发送的struct socketaddr地址
addrlen: 要发送的struct socketaddr的大小
*/
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
                      const struct sockaddr *dest_addr, socklen_t addrlen);
```

recvfrom和sendto函数和recv和send函数比较相似，只是前者是用于UDP协议下的，后者用于TCP协议下的。区别仅仅是**多了最后两个参数**。函数返回读取的字节个数。

### 3.8 getsockname()和getpeername()

```c
#include<sys/socket.h>

/*
 获取与某个套接字关联的本地协议地址
 localaddr：传出参，用于存放本地地址信息
 addrlen：传入传出参，传入localaddr大小，传出localaddr返回的真正大小
*/
int getsockname(int sockfd, struct sockaddr *localaddr, socklen_t *addrlen);

/*
 获取与某个套接字关联的外端协议地址
 peeraddr：传出参，用于存放对端地址信息
 addrlen：传入传出参，传入peeraddr大小，传出peeraddr返回的真正大小
*/
int getpeername(int sockfd, struct sockaddr *peeraddr, socklen_t *addrlen);
```

对于这两个函数，如果函数调用成功，则返回0，如果调用出错，则返回-1。使用这两个函数，我们可以通过套接字描述符来获取本地的地址信息和对端的地址信息。

对于TCP连接的服务端：

- 在**accept之前**调用`getsocketname()`，会获取**内核赋予该连接的服务端**的IP地址和本地端口号

- 在**accept之后**调用`getsocketname()`，会获取**客户端真正连接的服务端**的IP地址和本地端口号

- 在**accept之后**调用`getpeername()`，会获取**当前连接的客户端**的IP地址和端口号

对于TCP连接的客户端：

- 在**connect之后**调用`getpeername()`，会获取**当前连接的服务端**的IP地址和端口号

## 四、小写转大写程序

功能需求：客户端发送字符串，服务器端将其解析为大写后传回给客户端。

首先给出**第四章**和**第五章**代码的头文件`head.h`（**放置的位置根据具体程序为准**）：

```c
/* head.h */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/time.h>
#include <arpa/inet.h>
#include <errno.h>
#include <time.h>
#include <sys/wait.h>
#include <pthread.h>
#include <semaphore.h>

#define SERV_PORT 2017
#define BACKLOG 10
#define QUEUE_SIZE 5

#define MSG_FILENAME 1
#define MSG_CONTENT 2
#define MSG_ACK 3
#define MSG_DONE 4
#define MSG_ERROR 5
#define MSG_EXIT 6

#define MSG_SIZE BUFSIZ + 2*sizeof(int)

struct msg {
	int type;
	int len;
	char data[];
};

void p_error(char *msg) {
	perror(msg);
	exit(EXIT_FAILURE);
}
```

### 4.1 实现单客户端

```c
/* server */
#include "../head.h"

int main(void) {

	int lfd, cfd, res, i;
	char buf[BUFSIZ], clie_ip[BUFSIZ];
	struct sockaddr_in serv_addr, clie_addr;
	socklen_t clie_addr_len;
	ssize_t n;

	/* 1.socket() */
	lfd = socket(AF_INET, SOCK_STREAM, 0); 
	if (lfd == -1)
		perror("socket error..");

	/* 2.create struct sockaddr_in */	
	memset(&serv_addr, 0, sizeof(serv_addr));
	serv_addr.sin_family = AF_INET;
	serv_addr.sin_port = htons(SERV_PORT);
	serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);

	/* 3.bind() */
	res = bind(lfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr));
	if(res == -1)
		perror("bind error..");

	/* 4.listen() */
	res = listen(lfd, BACKLOG);
	if (res == -1)
		perror("listen error..");

	/* 5.accept() */
	clie_addr_len = sizeof(clie_addr);
	cfd = accept(lfd, (struct sockaddr *)&clie_addr, &clie_addr_len);
	printf("client IP: %s, client port: %d\n",
			inet_ntop(AF_INET, &clie_addr.sin_addr.s_addr,
				clie_ip, sizeof(clie_ip)),
			ntohs(clie_addr.sin_port));

	/* 6.recv() and send() */
	while(1) {
		n = recv(cfd, buf, sizeof(buf), 0);	
		printf("Receive msg: %s",buf);
	
		if(strcmp("exit\n", buf) == 0)
			break;

		for(i = 0; i < n; i++)
			buf[i] = toupper(buf[i]);
		
		printf("Send msg: %s",buf);
		send(cfd, buf, strlen(buf), 0);
		
		memset(buf, 0, sizeof(buf));
	}

	/* 6.close() */
	close(lfd);
	close(cfd);
}
```

```c
/* client */
#include "../head.h"

int main(int argc, char *argv[]) {
	int cfd, res;
	struct sockaddr_in serv_addr;
	char buf[BUFSIZ];

	/* 1.socket() */
	cfd = socket(AF_INET, SOCK_STREAM, 0);
	if (cfd == -1)
		perror("socket error..");

	/* 2.create struct sockaddr_in */	
	memset(&serv_addr, 0, sizeof(serv_addr));
	serv_addr.sin_family = AF_INET;
	serv_addr.sin_port = htons(SERV_PORT);
	inet_pton(AF_INET, argv[1], &serv_addr.sin_addr.s_addr);

	/* 3.connect() */
	res = connect(cfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr));
	if (res == -1)
		perror("connect error..\n");

	/* 4.send() and recv() */
	while (1) {
		printf("Send msg : ");
		fgets(buf, sizeof(buf), stdin);
		
		send(cfd, buf, strlen(buf), 0);
		if (strcmp("exit\n", buf) == 0)
			break;
		memset(buf, 0, sizeof(buf));
		
		n = recv(cfd, buf, sizeof(buf), 0);
		printf("Receive msg: %s", buf);
	}

	/* 4.close() */
	close(cfd);

	return 0;
}
```

### 4.2 新增对多客户端的支持

```c
/* server */
#include "../head.h"

void sig_child(int signo) {
	pid_t pid;
	int stat;
	while((pid = waitpid(-1, &stat,WNOHANG)) > 0)
		printf("child %d terminated\n", pid);
	return;
}

int main(void) {
	int lfd, cfd, i;
	struct sockaddr_in serv_addr, clie_addr;
	struct sockaddr_in listen_addr, peer_addr;
	char listen_ip[BUFSIZ], peer_ip[BUFSIZ];
	char buf[BUFSIZ];
	ssize_t n;
	time_t t;
	socklen_t clie_len, listen_len, peer_len;

	if((lfd = socket(AF_INET, SOCK_STREAM, 0)) == -1)
		p_error("socket error");

	memset(&serv_addr, 0, sizeof(serv_addr));
	serv_addr.sin_family = AF_INET;
	serv_addr.sin_port = htons(SERV_PORT);
	serv_addr.sin_addr.s_addr = htonl(INADDR_ANY); 

	if ((i = bind(lfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr))) == -1)
		p_error("bind error");

	if ((i = listen(lfd, BACKLOG)) == -1)
		p_error("listen error");

	listen_len = sizeof(listen_addr);
	getsockname(lfd, (struct sockaddr *)&listen_addr, &listen_len);
	printf("server listen address = %s:%d\n",
			inet_ntop(AF_INET, &listen_addr.sin_addr.s_addr,
				listen_ip, sizeof(listen_ip)),
			ntohs(listen_addr.sin_port));

	printf("Server service start success!\n\n");

	while(1) {
		clie_len = sizeof(clie_addr);

		if ((cfd = accept(lfd, (struct sockaddr *)&clie_addr, &clie_len))== -1)
			p_error("accept error");

		if((i = fork()) < 0)
			p_error("fork error");
		else if ( i == 0) {
			close(lfd);

			peer_len = sizeof(peer_addr);
			getpeername(cfd, (struct sockaddr *)&peer_addr, &peer_len);
			printf("%s:%d login now!\n\n",
					inet_ntop(AF_INET, &peer_addr.sin_addr.s_addr,
						peer_ip, sizeof(peer_ip)),
					ntohs(peer_addr.sin_port));

			while(1) {
				if ((n = recv(cfd, &buf, sizeof(buf), 0)) <= 0)
					continue;
				time(&t);
				printf("%s", ctime(&t));
				printf("[%s:%d]: %s\n", peer_ip, ntohs(peer_addr.sin_port), buf);
				if(strcmp("exit\n", buf) == 0) 
					break;
				memset(&buf, 0, sizeof(buf));
			}
			printf("%s:%d exit now!\n", peer_ip, ntohs(peer_addr.sin_port));
			close(cfd);
			exit(0);
		} 
		signal(SIGCHLD, sig_child);
	}

	return 0;
}
```

```c
/* client */
#include "../head.h"

#define BACKLOG 10

int main(int len, char *argv[]) {
	int cfd, i;
	struct sockaddr_in serv_addr;
	struct sockaddr_in connect_addr, peer_addr;
	char connect_ip[BUFSIZ], peer_ip[BUFSIZ];
	char buf[BUFSIZ];
	time_t t;
	socklen_t connect_len, peer_len;

    cfd = socket(AF_INET, SOCK_STREAM, 0);
    if(cfd == -1) {
		perror("socket error");
		exit(1);
	}

	memset(&serv_addr, 0, sizeof(serv_addr));
	serv_addr.sin_family = AF_INET;
	serv_addr.sin_port = htons(SERV_PORT);
	inet_pton(AF_INET, argv[1], &serv_addr.sin_addr.s_addr);

	i = connect(cfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr));
	if (i == -1) {
		if(errno == ECONNREFUSED)
			perror("connection refuse");
		else if(errno == EHOSTUNREACH)
			perror("no route to host");
		else if(errno == ETIMEDOUT)
			perror("connection time out");
		exit(1);
	} else
		printf("Connect server succes!\n\n");

	connect_len = sizeof(connect_addr);
	getsockname(cfd, (struct sockaddr *)&connect_addr, &connect_len);
	printf("connect server address = %s:%d\n",
			inet_ntop(AF_INET, &connect_addr.sin_addr.s_addr,
				connect_ip, sizeof(connect_ip)),
			ntohs(connect_addr.sin_port));

    peer_len = sizeof(peer_addr);
	getpeername(cfd, (struct sockaddr *)&peer_addr, &peer_len);
	printf("connect peer address = %s:%d\n",
			inet_ntop(AF_INET, &peer_addr.sin_addr.s_addr,
				peer_ip, sizeof(peer_ip)),
			ntohs(peer_addr.sin_port));

	printf("================\n");
	while(1) {
		time(&t);
		printf("%s", ctime(&t));
		printf("Send msg: ");
		fgets(buf, sizeof(buf), stdin);
		send(cfd, &buf, strlen(buf), 0);
		if(strcmp("exit\n", buf) == 0)
			break;
	}

	close(cfd);
	return 0;
}
```

### 4.3 基于 I/O 复用模式重写

关于I/O复用模式的基本知识，这里不再赘述，可以参考[《Linux IO模型》](/3b3bd025.html)。

```c
/* server */
#include "../head.h"

struct sockaddr_in peer_addr;
char peer_ip[BUFSIZ];
socklen_t peer_len;

// 处理数据，正常结束返回0，用户退出返回1
int parser_msg(int cfd) {
	char buf[BUFSIZ];
	time_t t;
	int i;

	// 接收数据
	if ((recv(cfd, &buf, sizeof(buf), 0)) <= 0)
		p_error("recv");

	time(&t);
	printf("%s", ctime(&t));
	printf("Recvice: [%s:%d]: %s", peer_ip, ntohs(peer_addr.sin_port), buf);

	if (strcmp("exit\n", buf) == 0) {
		printf("%s:%d exit now!\n", peer_ip, ntohs(peer_addr.sin_port));
		return 1;
	}

	for (i = 0; i < strlen(buf); i++)
		buf[i] = toupper(buf[i]);

	// 发送数据
	printf("Send: [%s:%d]: %s\n", peer_ip, ntohs(peer_addr.sin_port), buf);
	send(cfd, &buf, sizeof(buf), 0);

	return 0;
}

int main(void) {
	int i, opt, maxi, maxfd;
	int lfd, cfd;
	int nready, client[FD_SETSIZE]; /* 自定义数组，用于存放所有连接的客户端*/
	struct sockaddr_in serv_addr, clie_addr;
	struct sockaddr_in listen_addr;
	char listen_ip[BUFSIZ];
	struct timeval tv;
	fd_set rset, allset; /* rset读事件文件描述符集合 allset用来暂存*/
	socklen_t clie_len, listen_len, peer_len;

	if ((lfd = socket(AF_INET, SOCK_STREAM, 0)) == -1)
		p_error("socket error");

	// 初始化serv_addr
	memset(&serv_addr, 0, sizeof(serv_addr));
	serv_addr.sin_family = AF_INET;
	serv_addr.sin_port = htons(SERV_PORT);
	serv_addr.sin_addr.s_addr = htonl(INADDR_ANY); 

	// 避免bind出现地址被使用问题
	opt = 1;
	setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

	if (bind(lfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) == -1)
		p_error("bind error");

	if (listen(lfd, BACKLOG) == -1)
		perror("listen error");

	listen_len = sizeof(listen_addr);
	getsockname(lfd, (struct sockaddr *)&listen_addr, &listen_len);
	printf("server listen address = %s:%d\n",
			inet_ntop(AF_INET, &listen_addr.sin_addr.s_addr,
				listen_ip, sizeof(listen_ip)),
			ntohs(listen_addr.sin_port));

	printf("Server service start success!\n\n");

	/* ----以上代码完成服务端服务启动，下面开始处理客户端连接---- */

	maxfd = lfd; /* 起初lfd为最大文件描述符 */
	maxi = -1; /* client数组下标初始化为-1 */

	memset(client, -1, FD_SETSIZE); /* 初始化client数组为-1 */

	FD_ZERO(&allset);
	FD_SET(lfd, &allset); /* 将lfd加入监听集 */

	while (1) {
		rset = allset; /* 每次循环重新设置select监听集 */
		tv.tv_sec = 3; /* 每次循环重新设置超时 */
		
		nready = select(maxfd+1, &rset, NULL, NULL, &tv); /* select监听 */

		if (nready < 0)
			p_error("select error");
		else if (nready == 0) {
			printf("timeout..\n");
			continue;
		}

		// 处理新客户端登录
		if (FD_ISSET(lfd, &rset)) {
			clie_len = sizeof(clie_addr);

			if ((cfd = accept(lfd, (struct sockaddr *)&clie_addr, &clie_len)) == -1) /* 此时accept一定会阻塞 */
				p_error("accept error");

			// 打印登录信息
			peer_len = sizeof(peer_addr);
			getpeername(cfd, (struct sockaddr *)&peer_addr, &peer_len);
			printf("%s:%d login now!\n\n",
					inet_ntop(AF_INET, &peer_addr.sin_addr.s_addr,
						peer_ip, sizeof(peer_ip)),
					ntohs(peer_addr.sin_port));

			// 将cfd存入client数组中，并更新client数组最大下标
			for (i=0; i<FD_SETSIZE; i++)
				if (client[i] < 0) {
					client[i] = cfd;
					if (i > maxi)
						maxi = i;
					break;
				}

			// 达到监听上限
			if (i == FD_SETSIZE) {
				puts("too many clients\n");
				exit(EXIT_FAILURE);
			}

			FD_SET(cfd, &allset); /* 将cfd添加入allset监听集中 */

			if (cfd > maxfd) /* 更新maxfd */
				maxfd = cfd;
			
			if (--nready <= 0) /* 如果仅仅接收到新客户端登录，则continue */
				continue;
		}

		// 接收客户端发送的数据
		for (i=0; i<=maxi; i++) {
			if ((cfd = client[i]) == -1)
				continue;
			if (FD_ISSET(cfd, &rset)) {
				// 客户端退出
				if (parser_msg(cfd) == 1) {
					close(cfd); /* 关闭文件描述符 */
					FD_CLR(cfd, &allset); /* 从监听集中移除 */
					client[i] = -1; /* 从client数组移除 */
				}
			}
			
			if (--nready <= 0)
				break;
		}
	} 

	return 0;
}
```

```c
/* client */
#include "../head.h"

int main(int len, char *argv[]) {
	int cfd;
	struct sockaddr_in serv_addr;
	struct sockaddr_in connect_addr, peer_addr;
	char connect_ip[BUFSIZ], peer_ip[BUFSIZ];
	char buf[BUFSIZ];
	time_t t;
	socklen_t connect_len, peer_len;

    if ((cfd = socket(AF_INET, SOCK_STREAM, 0)) == -1)
		p_error("socket error");

	memset(&serv_addr, 0, sizeof(serv_addr));
	serv_addr.sin_family = AF_INET;
	serv_addr.sin_port = htons(SERV_PORT);
	inet_pton(AF_INET, argv[1], &serv_addr.sin_addr.s_addr);

	if (connect(cfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) == -1)
		switch (errno) {
			case ECONNREFUSED:
				p_error("connection refuse");
			case EHOSTUNREACH:
				p_error("no route to host");
			case ETIMEDOUT:
				p_error("connection time out");
			default:
				p_error("unknown");
		}
	else
		printf("Connect server succes!\n\n");

	connect_len = sizeof(connect_addr);
	getsockname(cfd, (struct sockaddr *)&connect_addr, &connect_len);
	printf("connect server address = %s:%d\n",
			inet_ntop(AF_INET, &connect_addr.sin_addr.s_addr,
				connect_ip, sizeof(connect_ip)),
			ntohs(connect_addr.sin_port));

    peer_len = sizeof(peer_addr);
	getpeername(cfd, (struct sockaddr *)&peer_addr, &peer_len);
	printf("connect peer address = %s:%d\n",
			inet_ntop(AF_INET, &peer_addr.sin_addr.s_addr,
				peer_ip, sizeof(peer_ip)),
			ntohs(peer_addr.sin_port));

	printf("================\n");
	while (1) {
		time(&t);
		printf("%s Send msg: ", ctime(&t));
		
		fgets(buf, sizeof(buf), stdin);
		
		send(cfd, &buf, strlen(buf), 0);
		if(strcmp("exit\n", buf) == 0)
			break;
		
		recv(cfd, &buf, sizeof(buf), 0);
		time(&t);
		printf("\n%s Receive msg: %s\n", ctime(&t), buf);
	}

	close(cfd);
	return 0;
}
```

### 4.4 使用UDP协议重写

```c
/* server */
#include "head.h"

int main(void) {

	int lfd, i;
	char buf[BUFSIZ], clie_ip[BUFSIZ];
	struct sockaddr_in serv_addr, clie_addr;
	socklen_t clie_addr_len;
	ssize_t n;

	/* 1.socket() */
	if ((lfd = socket(AF_INET, SOCK_DGRAM, 0)) == -1)
		p_error("socket error..");

	/* 2.create struct sockaddr_in */	
	memset(&serv_addr, 0, sizeof(serv_addr));
	serv_addr.sin_family = AF_INET;
	serv_addr.sin_port = htons(SERV_PORT);
	serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
	
	/* 3.bind() */
	if ((i = bind(lfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr))) == -1)
		p_error("bind error..");

	/* 4.recvfrom() and sendto() */
	while(1) {
		clie_addr_len = sizeof(clie_addr);
		
		if ((n = recvfrom(lfd, buf, sizeof(buf), 0, (struct sockaddr *)&clie_addr, &clie_addr_len)) == -1)
			p_error("recvfrom error..");
		
		printf("recvfrom ip : %s, port : %d\n",
				inet_ntop(AF_INET, &clie_addr.sin_addr.s_addr,
					clie_ip, sizeof(clie_ip)),
				ntohs(clie_addr.sin_port));
		
		printf("Receive msg: %s",buf);
		
		if(strcmp("exit\n", buf) == 0)
			break;

		for(i = 0; i < n; i++)
			buf[i] = toupper(buf[i]);
		
		printf("Send msg: %s",buf);
		
		if ((n = sendto(lfd, buf, strlen(buf), 0, (struct sockaddr *)&clie_addr, sizeof(clie_addr))) == -1)
			p_error("sendto error..");
		
		memset(buf, 0, sizeof(buf));
	}

	/* 4.close() */
	close(lfd);
}
```

```c
/* client */
#include "head.h"

int main(int argc, char *argv[]) {
	int cfd;
	struct sockaddr_in serv_addr;
	ssize_t n;
	char buf[BUFSIZ];

	/* 1.socket() */
	if ((cfd = socket(AF_INET, SOCK_DGRAM, 0)) == -1)
		p_error("socket error..");

	/* 2.create struct sockaddr_in */	
	memset(&serv_addr, 0, sizeof(serv_addr));
	serv_addr.sin_family = AF_INET;
	serv_addr.sin_port = htons(SERV_PORT);
	inet_pton(AF_INET, argv[1], &serv_addr.sin_addr.s_addr);

	/* 3.sendto() and recvfrom() */
	while (fgets(buf, sizeof(buf), stdin) != NULL) {
		printf("Send msg : %s", buf);
		
		if ((n = sendto(cfd, buf, strlen(buf), 0, (struct sockaddr *)&serv_addr, sizeof(serv_addr))) == -1)
			p_error("sendto error..");
	
		if(strcmp("exit\n", buf) == 0)
			break;

		if ((n = recvfrom(cfd, buf, sizeof(buf), 0, NULL, 0)) == -1)
			p_error("recvfrom error..");
		
		printf("Receive msg: %s", buf);
		
		memset(buf, 0, sizeof(buf));
	}

	/* 4.close() */
	close(cfd);

	return 0;
}
```

## 五、文件传输程序

功能需求：客户端发送源文件名和目的文件名，服务器端进行文件传输。

### 5.1 实现多客户端

```c
/* server */
#include "../head.h"

void sig_child(int signo) {
	pid_t pid;
	int stat;
	while((pid = waitpid(-1, &stat,WNOHANG)) > 0)
		printf("child %d terminated\n", pid);
	return;
}

int main(void) {
	int lfd, cfd, i;
	struct sockaddr_in serv_addr, clie_addr;
	struct sockaddr_in listen_addr, peer_addr;
	struct msg *send_msg, *rec_msg;
	char listen_ip[BUFSIZ], peer_ip[BUFSIZ];
	char buf[BUFSIZ], target_filename[BUFSIZ];
	time_t start_time, end_time;
	FILE *fp;
	socklen_t clie_len, listen_len, peer_len;

	if((lfd = socket(AF_INET, SOCK_STREAM, 0)) == -1)
		p_error("socket error");

	send_msg = (struct msg*)malloc(MSG_SIZE);
	rec_msg = (struct msg*)malloc(MSG_SIZE);

	memset(&serv_addr, 0, sizeof(serv_addr));
	serv_addr.sin_family = AF_INET;
	serv_addr.sin_port = htons(SERV_PORT);
	serv_addr.sin_addr.s_addr = htonl(INADDR_ANY); 

	if ((i = bind(lfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr))) == -1)
		p_error("bind error");

	if ((i = listen(lfd, BACKLOG)) == -1)
		p_error("listen error");

	listen_len = sizeof(listen_addr);
	getsockname(lfd, (struct sockaddr *)&listen_addr, &listen_len);
	printf("server listen address = %s:%d\n",
			inet_ntop(AF_INET, &listen_addr.sin_addr.s_addr,
				listen_ip, sizeof(listen_ip)),
			ntohs(listen_addr.sin_port));

	printf("Server service start success!\n\n");

	while(1) {
		clie_len = sizeof(clie_addr);

		if((cfd = accept(lfd, (struct sockaddr *)&clie_addr, &clie_len)) == -1)
			p_error("accept error");

		if((i = fork()) < 0)
			perror("fork error");
		else if ( i == 0) {
			close(lfd);

			peer_len = sizeof(peer_addr);
			getpeername(cfd, (struct sockaddr *)&peer_addr, &peer_len);
			printf("%s:%d login now!\n\n",
					inet_ntop(AF_INET, &peer_addr.sin_addr.s_addr,
						peer_ip, sizeof(peer_ip)),
					ntohs(peer_addr.sin_port));

			while(1){
				memset(send_msg, 0, sizeof(struct msg));
				recv(cfd, (void*)rec_msg, MSG_SIZE, 0);

				switch (rec_msg->type) {
					case MSG_FILENAME:
						memset(&target_filename, 0, sizeof(target_filename));
						memcpy(&target_filename, rec_msg->data, rec_msg->len);
						fp = fopen(target_filename, "w+");
						//open file error
						if(!fp) {
							perror("fopen error");
							send_msg->type = MSG_ERROR;
							strcpy(buf, "open file error");
							send_msg->len = strlen(buf);
							memcpy(send_msg->data, &buf, send_msg->len);
							send(cfd, (void*)send_msg, sizeof(struct msg) + send_msg->len, 0);
						} else {
							time(&start_time);
							send_msg->type = MSG_ACK;
							send_msg->len = 0;
							send(cfd, (void*)send_msg, sizeof(struct msg), 0);
						}
						break;
					case MSG_CONTENT:
						if(!fp) {
							send_msg->type = MSG_ERROR;
							strcpy(buf, "file not open yet");
							send_msg->len = strlen(buf);
							memcpy(send_msg->data, &buf, send_msg->len);
							send(cfd, (void*)send_msg, sizeof(struct msg) + send_msg->len, 0);
						} else {
							fwrite(rec_msg->data, sizeof(char), rec_msg->len, fp);
							fflush(fp);

							send_msg->type = MSG_ACK;
							send_msg->len = 0;
							send(cfd, (void*)send_msg, sizeof(struct msg), 0);
						}
						break;
					case MSG_DONE:
						time(&end_time);
						printf("[INFO] %s:%d complete file send\n", peer_ip, ntohs(peer_addr.sin_port));	
						printf("[INFO] Save in : %s\n",target_filename);
						printf("[INFO] Consume : %lfs\n", difftime(end_time, start_time));

						//avoid double fcolse()
						if(fp) {
							fclose(fp);	
							fp = NULL;	
						}
						send_msg->type = MSG_ACK;
						send_msg->len = 0;
						send(cfd, (void*)send_msg, sizeof(struct msg), 0);
						break;
					case MSG_EXIT:
						printf("[INFO] client will exit\n");	

						send_msg->type = MSG_EXIT;
						send_msg->len = 0;
						send(cfd, (void*)send_msg, sizeof(struct msg), 0);
						break;
					default:
						break;
				}
			}
			close(cfd);
			exit(0);
		} 
		signal(SIGCHLD, sig_child);
	}
	return 0;
}
```

```c
/* client */
#include "../head.h"

int main(int argc, char *argv[]) {
	FILE *fp;
	int cfd;
	struct sockaddr_in serv_addr;
	struct sockaddr_in connect_addr, peer_addr;
	struct msg *send_msg, *rec_msg;
	char connect_ip[BUFSIZ], peer_ip[BUFSIZ];
	char buf[BUFSIZ], source_filename[BUFSIZ], target_filename[BUFSIZ];
	ssize_t n;
	time_t start_time, end_time;
	socklen_t connect_len, peer_len;

	if((cfd = socket(AF_INET, SOCK_STREAM, 0)) == -1)
		p_error("socket error");

	send_msg = (struct msg*)malloc(MSG_SIZE);
	rec_msg = (struct msg*)malloc(MSG_SIZE);

	memset(&serv_addr, 0, sizeof(serv_addr));
	serv_addr.sin_family = AF_INET;
	serv_addr.sin_port = htons(SERV_PORT);
	inet_pton(AF_INET, argv[1], &serv_addr.sin_addr.s_addr);

	if ((connect(cfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr))) == -1) {
		if(errno == ECONNREFUSED)
			p_error("connection refuse");
		else if(errno == EHOSTUNREACH)
			p_error("no route to host");
		else if(errno == ETIMEDOUT)
			p_error("connection time out");
	} else
		printf("Connect server succes!\n\n");

	connect_len = sizeof(connect_addr);
	getsockname(cfd, (struct sockaddr *)&connect_addr, &connect_len);
	printf("connect server address = %s:%d\n",
			inet_ntop(AF_INET, &connect_addr.sin_addr.s_addr,
				connect_ip, sizeof(connect_ip)),
			ntohs(connect_addr.sin_port));

	peer_len = sizeof(peer_addr);
	getpeername(cfd, (struct sockaddr *)&peer_addr, &peer_len);
	printf("connect peer address = %s:%d\n",
			inet_ntop(AF_INET, &peer_addr.sin_addr.s_addr,
				peer_ip, sizeof(peer_ip)),
			ntohs(peer_addr.sin_port));

	printf("================\n");

	while(1) {
		printf("please input source file name: ");
		scanf("%s", source_filename);
		
		if (!(fp = fopen(source_filename, "r"))) {
			perror("fopen");
			exit(1);
		}

		// send file name
		memset(send_msg, 0, sizeof(struct msg));
		printf("please input target file name: ");
		scanf("%s", target_filename);

		send_msg->type = MSG_FILENAME;
		send_msg->len = strlen(target_filename);
		memcpy(send_msg->data, &target_filename, send_msg->len);

		send(cfd, (void *)send_msg, sizeof(struct msg) + send_msg->len, 0);
		recv(cfd, (void *)rec_msg, MSG_SIZE, 0);
		if(rec_msg->type == MSG_ERROR) {
			printf("[ERROR] ");
			puts(rec_msg->data);
			exit(1);
		}

		// send file content
		time(&start_time);
		memset(send_msg, 0, sizeof(struct msg));
		while((n = fread(&buf, sizeof(char), sizeof(buf), fp))) {
			send_msg->type = MSG_CONTENT;
			send_msg->len = n;
			memcpy(send_msg->data, &buf, send_msg->len);
			send(cfd, (void*)send_msg, sizeof(struct msg) + send_msg->len, 0);

			recv(cfd, (void *)rec_msg, MSG_SIZE, 0);
			if(rec_msg->type == MSG_ERROR) {
				printf("[ERROR] ");
				puts(rec_msg->data);
				exit(1);
			}
		}

		//compile send file 
		memset(send_msg, 0, sizeof(struct msg));
		if(n > 0) {
			send_msg->type = MSG_ERROR;
			strcpy(buf, "send file content error");
			send_msg->len = strlen(buf);
			memcpy(send_msg->data, &buf, send_msg->len);

			send(cfd, (void*)send_msg, sizeof(struct msg) + send_msg->len, 0);
		} else {
			time(&end_time);
			printf("[INFO] complete send file, consume %lfs.\n", difftime(end_time, start_time));
			if(fp) {
				fclose(fp);
				fp = NULL;	
			}
			send_msg->type = MSG_DONE;
			send_msg->len = 0;
			send(cfd, (void*)send_msg, sizeof(struct msg), 0);
			recv(cfd, (void *)rec_msg, MSG_SIZE, 0);
		}

loop:	printf(">(input 'q' to quit, input 'c' to continue) ");
		scanf("%s", buf);
		if(strcmp("c", buf) == 0)
			continue;
		else if(strcmp("q", buf) == 0) {
			//send exit
			memset(send_msg, 0, sizeof(struct msg));
			send_msg->type = MSG_EXIT;
			send_msg->len = 0;

			send(cfd, (void*)send_msg, sizeof(struct msg), 0);
			recv(cfd, (void *)rec_msg, MSG_SIZE, 0);

			if(rec_msg->type == MSG_ERROR) {
				printf("[ERROR] ");
				puts(rec_msg->data);
				exit(1);
			} else if(rec_msg->type == MSG_EXIT) {
				printf("[INFO] you will exit..\n");
				close(cfd);
				break;
			}
		} else 
			goto loop;
	}
	return 0;
}
```


### 5.2 使用多线程重写客户端

```c
/* server */
#include "../head.h"

void sig_child(int signo) {
	pid_t pid;
	int stat;
	while((pid = waitpid(-1, &stat,WNOHANG)) > 0)
		printf("child %d terminated\n", pid);
	return;
}

int main(void) {
	int lfd, cfd, i;
	struct sockaddr_in serv_addr, clie_addr;
	struct sockaddr_in listen_addr, peer_addr;
	struct msg *send_msg, *rec_msg;
	char listen_ip[BUFSIZ], peer_ip[BUFSIZ];
	char buf[BUFSIZ], target_filename[BUFSIZ];
	time_t start_time, end_time;
	FILE *fp;
	socklen_t clie_len, listen_len, peer_len;

	if((lfd = socket(AF_INET, SOCK_STREAM, 0)) == -1)
		p_error("socket error");

	send_msg = (struct msg*)malloc(MSG_SIZE);
	rec_msg = (struct msg*)malloc(MSG_SIZE);

	memset(&serv_addr, 0, sizeof(serv_addr));
	serv_addr.sin_family = AF_INET;
	serv_addr.sin_port = htons(SERV_PORT);
	serv_addr.sin_addr.s_addr = htonl(INADDR_ANY); 

	if( (i = bind(lfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr))) == -1)
		p_error("bind error");

	if( (i = listen(lfd, BACKLOG)) == -1)
		p_error("listen error");

	listen_len = sizeof(listen_addr);
	getsockname(lfd, (struct sockaddr *)&listen_addr, &listen_len);
	printf("server listen address = %s:%d\n",
			inet_ntop(AF_INET, &listen_addr.sin_addr.s_addr,
				listen_ip, sizeof(listen_ip)),
			ntohs(listen_addr.sin_port));

	printf("Server service start success!\n\n");

	while(1) {
		clie_len = sizeof(clie_addr);
		if( (cfd = accept(lfd, (struct sockaddr *)&clie_addr, &clie_len)) == -1)
			p_error("accept error");

		if((i = fork()) < 0)
			p_error("fork error");
		else if (i == 0) {
			close(lfd);

			peer_len = sizeof(peer_addr);
			getpeername(cfd, (struct sockaddr *)&peer_addr, &peer_len);
			printf("%s:%d login now!\n\n",
					inet_ntop(AF_INET, &peer_addr.sin_addr.s_addr,
						peer_ip, sizeof(peer_ip)),
					ntohs(peer_addr.sin_port));

			while(1){
				memset(send_msg, 0, sizeof(struct msg));
				recv(cfd, (void*)rec_msg, MSG_SIZE, 0);

				if(rec_msg->type == MSG_FILENAME) {
					memset(&target_filename, 0, sizeof(target_filename));
					memcpy(&target_filename, rec_msg->data, rec_msg->len);
					fp = fopen(target_filename, "w+");
					//open file error
					if(!fp) {
						p_error("fopen error");
						send_msg->type = MSG_ERROR;
						strcpy(buf, "open file error");
						send_msg->len = strlen(buf);
						memcpy(send_msg->data, &buf, send_msg->len);
						send(cfd, (void*)send_msg, sizeof(struct msg) + send_msg->len, 0);
					} else {
						time(&start_time);
						send_msg->type = MSG_ACK;
						send_msg->len = 0;
						send(cfd, (void*)send_msg, sizeof(struct msg), 0);
					}
				} else if(rec_msg->type == MSG_CONTENT) {
					if(!fp) {
						send_msg->type = MSG_ERROR;
						strcpy(buf, "file not open yet");
						send_msg->len = strlen(buf);
						memcpy(send_msg->data, &buf, send_msg->len);
						send(cfd, (void*)send_msg, sizeof(struct msg) + send_msg->len, 0);
					} else {
						fwrite(rec_msg->data, sizeof(char), rec_msg->len, fp);
						fflush(fp);

						send_msg->type = MSG_ACK;
						send_msg->len = 0;
						send(cfd, (void*)send_msg, sizeof(struct msg), 0);
					}
				} else if (rec_msg->type == MSG_DONE) {
					time(&end_time);
					printf("[INFO] %s:%d complete file send\n", peer_ip, ntohs(peer_addr.sin_port));	
					printf("[INFO] Save in : %s\n",target_filename);
					printf("[INFO] Consume : %lfs\n", difftime(end_time, start_time));


					if(fp) {
						fclose(fp);	
						fp = NULL;	
					}
					send_msg->type = MSG_ACK;
					send_msg->len = 0;
					send(cfd, (void*)send_msg, sizeof(struct msg), 0);
				} else if(rec_msg->type == MSG_EXIT) {
					printf("[INFO] client will exit\n");	
					
					send_msg->type = MSG_EXIT;
					send_msg->len = 0;
					send(cfd, (void*)send_msg, sizeof(struct msg), 0);
					break;
				}
			}

			close(cfd);
			exit(0);
		} 
		signal(SIGCHLD, sig_child);
	}
	return 0;
}
```

```c
/* client */
#include "../head.h"

pthread_spinlock_t lock;
int rear, head, count;
struct msg *queue_buf[QUEUE_SIZE];

void thread(char *filename) {
	FILE *fp;
	char buf[BUFSIZ];
	ssize_t n;
	time_t start_time, end_time;
	struct msg *m;

	if (!(fp = fopen(filename, "r")))
		p_error("fopen");

	time(&start_time);
	while((n = fread(&buf, sizeof(char), sizeof(buf), fp))) {
		if(rear == head)
			while(count >= QUEUE_SIZE);
		m = queue_buf[rear];
		m->type = MSG_CONTENT;
		m->len = n;
		memcpy(m->data, &buf, m->len);

		rear = (rear + 1) % QUEUE_SIZE;
		pthread_spin_lock(&lock);
		count++;
		pthread_spin_unlock(&lock);
	}

	if(rear == head)
		while(count >= QUEUE_SIZE);
	
	m = queue_buf[rear];
	time(&end_time);
	printf("[INFO] complete send file, Consume %lfs.\n",
			difftime(end_time, start_time));
	if(fp) {
		fclose(fp);
		fp = NULL;
	}
	m->type = MSG_DONE;
	m->len = 0;

	rear = (rear + 1) % QUEUE_SIZE;
	pthread_spin_lock(&lock);
	count++;
	pthread_spin_unlock(&lock);
}

int main(int argc, char *argv[]) {
	int cfd, i;
	struct sockaddr_in serv_addr;
	struct sockaddr_in connect_addr, peer_addr;
	struct msg *send_msg, *rec_msg;
	char connect_ip[BUFSIZ], peer_ip[BUFSIZ];
	char buf[BUFSIZ], source_filename[BUFSIZ], target_filename[BUFSIZ];
	socklen_t connect_len, peer_len;
	pthread_t tid;
	void *status;

	if((cfd = socket(AF_INET, SOCK_STREAM, 0)) == -1)
		p_error("socket error");

	// malloc struct msg
	for(i=0; i<QUEUE_SIZE; i++)
		queue_buf[i] = (struct msg*)malloc(MSG_SIZE);
	send_msg = (struct msg*)malloc(MSG_SIZE);
	rec_msg = (struct msg*)malloc(MSG_SIZE);

	memset(&serv_addr, 0, sizeof(serv_addr));
	serv_addr.sin_family = AF_INET;
	serv_addr.sin_port = htons(SERV_PORT);
	inet_pton(AF_INET, argv[1], &serv_addr.sin_addr.s_addr);

	if ((i = connect(cfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr))) == -1) {
		if(errno == ECONNREFUSED)
			p_error("connection refuse");
		else if(errno == EHOSTUNREACH)
			p_error("no route to host");
		else if(errno == ETIMEDOUT)
			p_error("connection time out");
	} else
		printf("Connect server succes!\n\n");

	connect_len = sizeof(connect_addr);
	getsockname(cfd, (struct sockaddr *)&connect_addr, &connect_len);
	printf("connect server address = %s:%d\n",
			inet_ntop(AF_INET, &connect_addr.sin_addr.s_addr,
				connect_ip, sizeof(connect_ip)),
			ntohs(connect_addr.sin_port));

	peer_len = sizeof(peer_addr);
	getpeername(cfd, (struct sockaddr *)&peer_addr, &peer_len);
	printf("connect peer address = %s:%d\n",
			inet_ntop(AF_INET, &peer_addr.sin_addr.s_addr,
				peer_ip, sizeof(peer_ip)),
			ntohs(peer_addr.sin_port));

	printf("================\n");

	while(1) {
		printf("please input source file name: ");
		scanf("%s", source_filename);

		// send file name
		memset(send_msg, 0, sizeof(struct msg));
		printf("please input target file name: ");
		scanf("%s", target_filename);

		send_msg->type = MSG_FILENAME;
		send_msg->len = strlen(target_filename);
		memcpy(send_msg->data, &target_filename, send_msg->len);

		send(cfd, (void *)send_msg, sizeof(struct msg) + send_msg->len, 0);
		recv(cfd, (void *)rec_msg, MSG_SIZE, 0);
		if(rec_msg->type == MSG_ERROR) {
			printf("[ERROR] ");
			puts(rec_msg->data);
			exit(1);
		}

		// create thread
		if((i = pthread_create(&tid, NULL, (void *)thread, source_filename)) != 0)
			p_error("pthread_create");
		
		// init spin lock
		if((i = pthread_spin_init(&lock, PTHREAD_PROCESS_PRIVATE)) != 0)
			p_error("pthread_spin_init");

		// send file content
		while(1) {
			if(rear == head)
				while(count <= 0);

			struct msg *m = queue_buf[head];
			if(m->type != MSG_CONTENT)
				break;
			send(cfd, (void*)m, sizeof(struct msg) + m->len, 0);
			recv(cfd, (void *)rec_msg, MSG_SIZE, 0);
			if(rec_msg->type == MSG_ERROR) {
				printf("[ERROR] ");
				puts(rec_msg->data);
				exit(1);
			}
			head = (head + 1) % QUEUE_SIZE;
			pthread_spin_lock(&lock);
			count--;
			pthread_spin_unlock(&lock);
		}
		pthread_join(tid, &status);

		if(rear == head)
			while(count <= 0);
		
		struct msg *m = queue_buf[head];
		send(cfd, (void*)m, sizeof(struct msg) + m->len, 0);
		recv(cfd, (void *)rec_msg, MSG_SIZE, 0);
		head = (head + 1) % QUEUE_SIZE;
		pthread_spin_lock(&lock);
		count--;
		pthread_spin_unlock(&lock);

		//exit and continue
loop:	printf(">(input 'q' to quit, input 'c' to continue) ");
		scanf("%s", buf);
		if(strcmp("c", buf) == 0)
			continue;
		else if(strcmp("q", buf) == 0) {
			//send exit
			memset(send_msg, 0, sizeof(struct msg));
			send_msg->type = MSG_EXIT;
			send_msg->len = 0;

			send(cfd, (void*)send_msg, sizeof(struct msg), 0);
			recv(cfd, (void *)rec_msg, MSG_SIZE, 0);

			if(rec_msg->type == MSG_ERROR) {
				printf("[ERROR] ");
				puts(rec_msg->data);
				exit(1);
			} else if(rec_msg->type == MSG_EXIT) {
				printf("[INFO] you will exit..\n");
				close(cfd);
				break;
			}
		} else 
			goto loop;
	}
	return 0;
}
```

### 5.3 使用UDP协议重写

```c
/* server */
#include "head.h"

static pthread_mutex_t mutex;

int rear, head, count;
struct msg *queue_buf[QUEUE_SIZE];

char client_ip[48];
int client_port;

void thread(char *filename) {
	FILE *fp;
	time_t start_time, end_time;
	int flag = 1;

	if((fp = fopen(filename, "w+")) == NULL)
		p_error("fopen");

	time(&start_time);

	while(flag) {
		if(rear == head)
			while (count <= 0) {
				usleep(100);		
			}
		struct msg *m = queue_buf[head];
		switch(m->type) {
			case MSG_DONE:
				if(fp) {
					fclose(fp);
					fp = NULL;
				}
				time(&end_time);
				printf("[INFO] Complete save file\n");
				printf("[INFO] Client : %s:%d\n",client_ip, client_port);
				printf("[INFO] Save path : %s\n",filename);
				printf("[INFO] Consume : %lfs\n", difftime(end_time ,start_time));
				printf("============\n");

				head = ( head + 1) % QUEUE_SIZE;
				pthread_mutex_lock(&mutex);
				count--;
				pthread_mutex_unlock(&mutex);

				flag = 0;
				break;
			case MSG_CONTENT:
				fwrite(m->data, sizeof(char), m->len, fp);
				fflush(fp);

				head = ( head + 1) % QUEUE_SIZE;
				pthread_mutex_lock(&mutex);
				count--;
				pthread_mutex_unlock(&mutex);
				break;
			default: 
				flag = 0;
				break;
		}
	}
	pthread_exit(NULL);
}

int main(void) {
	int lfd, i;
	struct sockaddr_in serv_addr, clie_addr;
	struct msg *send_msg, *rec_msg;
	char filename[BUFSIZ];
	socklen_t clie_len;
	pthread_t tid;

	if((lfd = socket(AF_INET, SOCK_DGRAM, 0))  == -1)
		p_error("socket error");

	// malloc struct msg
	for(i=0; i<QUEUE_SIZE; i++)
		queue_buf[i] = (struct msg*)malloc(MSG_SIZE);
	send_msg = (struct msg*)malloc(MSG_SIZE);
	rec_msg = (struct msg*)malloc(MSG_SIZE);

	memset(&serv_addr, 0, sizeof(serv_addr));
	serv_addr.sin_family = AF_INET;
	serv_addr.sin_port = htons(SERV_PORT);
	serv_addr.sin_addr.s_addr = htonl(INADDR_ANY); 

	int opt = 1;
	setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR,&opt, sizeof(opt));

	if((i = bind(lfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr))) == -1)
		p_error("bind error");

	printf("Server start service..\n");
	while(1) {
		clie_len = sizeof(clie_addr);
		memset(send_msg, 0, sizeof(struct msg));

		recvfrom(lfd, (void*)rec_msg, MSG_SIZE, 0, (struct sockaddr *)&clie_addr, &clie_len);
		rec_msg->data[rec_msg->len] = 0;

		inet_ntop(AF_INET, &clie_addr.sin_addr.s_addr, client_ip, sizeof(client_ip));
		client_port =  ntohs(clie_addr.sin_port);

		switch(rec_msg->type) {
			case MSG_FILENAME:
				strcpy(filename, rec_msg->data);

				if((i = pthread_create(&tid, NULL, (void *)thread, filename)) != 0)
					p_error("pthread_create");
				break;
			case MSG_CONTENT:
				if(rear == head)
					while (count >= QUEUE_SIZE);

				queue_buf[rear]->type = rec_msg->type;
				queue_buf[rear]->len = rec_msg->len;
				memcpy(queue_buf[rear]->data, rec_msg->data, rec_msg->len);

				rear = (rear + 1) % QUEUE_SIZE;
				pthread_mutex_lock(&mutex);
				count++;
				pthread_mutex_unlock(&mutex);

				break;
			case MSG_DONE:
				if(rear == head)
					while (count >= QUEUE_SIZE);

				queue_buf[rear]->type = rec_msg->type;
				queue_buf[rear]->len = rec_msg->len;
				memcpy(queue_buf[rear]->data, rec_msg->data, rec_msg->len);

				rear = (rear + 1) % QUEUE_SIZE;

				pthread_mutex_lock(&mutex);
				count++;
				pthread_mutex_unlock(&mutex);

				break;
			default: 
				break;
		}
	}
	return 0;
}
```

```c
/* client */
#include "head.h"

pthread_spinlock_t lock;
int rear, head, count;
double use_time;
struct msg *queue_buf[QUEUE_SIZE];

void thread(char *filename) {
	FILE *fp;
	char buf[BUFSIZ];
	ssize_t n;
	time_t start_time, end_time;
	struct msg *m;

	if (!(fp = fopen(filename, "r")))
		p_error("fopen");

	time(&start_time);
	while(1) {
		if(rear == head)
			while(count >= QUEUE_SIZE);

		m = queue_buf[rear];

		memset(&buf, 0, sizeof(buf));
		n = fread(&buf, sizeof(char), sizeof(buf), fp);
		if(n == 0) {
			if (fp) {
				fclose(fp);
				fp = NULL;
			}
			time(&end_time);
			use_time = difftime(end_time, start_time);
			
			m->type = MSG_DONE;
			m->len = 0;
			m->data[0] = '\0';

			rear = (rear + 1) % QUEUE_SIZE;
			pthread_spin_lock(&lock);
			count++;
			pthread_spin_unlock(&lock);
			break;
		} else {
			m->type = MSG_CONTENT;
			m->len = n;
			memcpy(m->data, &buf, m->len);

			rear = (rear + 1) % QUEUE_SIZE;
			pthread_spin_lock(&lock);
			count++;
			pthread_spin_unlock(&lock);
		}
	}
	pthread_exit(NULL);
}

int main(int argc, char *argv[]) {
	int cfd, i;
	struct sockaddr_in serv_addr;
	struct msg *send_msg;
	char buf[BUFSIZ], source_filename[BUFSIZ], target_filename[BUFSIZ];
	pthread_t tid;

	if((cfd = socket(AF_INET, SOCK_DGRAM, 0)) == -1)
		p_error("socket error");

	send_msg = (struct msg*)malloc(MSG_SIZE);
	memset(&serv_addr, 0, sizeof(serv_addr));
	serv_addr.sin_family = AF_INET;
	serv_addr.sin_port = htons(SERV_PORT);
	inet_pton(AF_INET, argv[1], &serv_addr.sin_addr.s_addr);

	printf("Client UDP service start..\n");

	while(1) {
		// malloc struct msg
		for(i=0; i<QUEUE_SIZE; i++)
			queue_buf[i] = (struct msg*)malloc(MSG_SIZE);

		printf("please input source file name: ");
		scanf("%s", source_filename);

		printf("please input target file name: ");
		scanf("%s", target_filename);

		memset(send_msg, 0, sizeof(struct msg));
		send_msg->type = MSG_FILENAME;
		send_msg->len = strlen(target_filename);
		memcpy(send_msg->data, &target_filename, send_msg->len);

		// send file name
		sendto(cfd, (void *)send_msg, sizeof(struct msg) + send_msg->len, 0,
				(struct sockaddr *)&serv_addr, sizeof(serv_addr));

		// create thread
		if((i = pthread_create(&tid, NULL, (void *)thread, source_filename)) != 0)
			p_error("pthread_create");

		// init spin lock
		if((i = pthread_spin_init(&lock, PTHREAD_PROCESS_PRIVATE)) != 0)
			p_error("pthread_spin_init");

		// send file content
		while(1) {
			if(rear == head)
				while(count <= 0);

			struct msg *m = queue_buf[head];

			sendto(cfd, (void*)m, sizeof(struct msg) + m->len, 0,
					(struct sockaddr *)&serv_addr, sizeof(serv_addr));

			if(m->type == MSG_DONE) {
				printf("[INFO] complete send file, consume %lfs.\n", use_time);
				break;	
			}

			head = (head + 1) % QUEUE_SIZE;
			pthread_spin_lock(&lock);
			count--;
			pthread_spin_unlock(&lock);
		}

		//exit and continue
loop:	printf(">(input 'q' to quit, input 'c' to continue) ");
		scanf("%s", buf);
		if(strcmp("c", buf) == 0)
			continue;
		else if(strcmp("q", buf) == 0) {
			printf("[INFO] you will exit..\n");
			close(cfd);
			break;
		} else 
			goto loop;
	}
	return 0;
}
```
