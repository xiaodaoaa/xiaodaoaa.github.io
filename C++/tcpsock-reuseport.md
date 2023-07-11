---
title: Linux TCP套接字地址复用
date: 2021-12-06 10:22:32
tags: [linux,TCP]
categories: linux
description: Linux TCP套接字地址复用。
---

## 1. 概述

***github项目地址：[https://github.com/superwujc](https://github.com/superwujc)
尊重原创，欢迎转载，注明出处：[https://my.oschina.net/superwjc/blog/1824089](https://my.oschina.net/superwjc/blog/1824089)***

对于特定的传输层协议（包括TCP），每个套接字通过4元组{ 本端ip, 本端port, 对端ip, 对端port }进行唯一标识，该4元组另可划分为一个套接字地址对(socket pair)：本端ip与本端port组成本端套接字地址，对端ip与对端port组成对端套接字地址。

默认情况下，内核仅允许对不同的本端套接字地址调用bind(2)，但不允许相同的本端套接字地址复用，对已存在的本端套接字地址再次执行bind(2)时，将以EADDRINUSE错误失败，即地址已被占用；Linux内核根据不同的应用场景，设置了若干种套接字地址复用的条件，包括套接字选项SO_REUSEADDR，SO_REUSEPORT与SO_BINDTODEVICE。

## 2. 应用场景

Linux内核根据以下顺序判断本端ipv4套接字地址是否可以复用：

 - 原套接字与新套接字都开启了SO_REUSEPORT选项，且创建原套接字与新套接字的进程EUID相同时，可以复用本端ip与本端port；或
 - 原套接字与新套接字都通过SO_BINDTODEVICE选项绑定至不同的网络接口时，可以复用本端ip与本端port；或
 - 原套接字与新套接字都开启了SO_REUSEADDR选项，且套接字状态都不为LISTEN时，可以复用本端ip与本端port；或
 - 原套接字与新套接字绑定至不同的本端ip地址，且都不为通配地址时，可以复用本端port


## 2.1 - SO_REUSEPORT

SO_REUSEPORT是自3.9内核开始支持的套接字选项，该选项用于同时运行同一本端套接字的多个实例，通过将传入的请求分发至不同的接收端进程或线程而实现套接字层面的负载均衡，提升运行在多核操作系统上的多线程网络服务器性能。

对于TCP，该选项的开启方式为：

```c
int sfd = socket(AF_INET, SOCK_STREAM, 0);

int optval = 1;
setsockopt(sfd, SOL_SOCKET, SO_REUSEPORT, &optval, sizeof(optval));

bind(sfd, (struct sockaddr *) &addr, addrlen);
```

若需使用该功能，则每一个套接字（包括第1个）在执行bind(2)前都必须开启该选项，且创建套接字的进程EUID必须相同。

Linux 内核当前的TCP协议实现对该选项的支持存在缺陷：运行同一服务端套接字地址的实例数量增加或减少时，由于三次握手过程中的初始SYN包与某个特定的套接字地址相关联，SO_REUSEPORT可能不会将ACK发送至正确的套接字地址，导致客户端被RESET，而服务端继续维持未完成的连接；后续的内核版本可能引入一个共享于监听套接字之间的连接请求表以解决该问题。

## 2.2 - SO_BINDTODEVICE

该选项将套接字绑定至特定的网络接口，如eth0，使得仅从该网络接口接收到的数据包才会被该套接字处理。

调用setsockopt(2)设置该选项时，optval与optlen参数分别为网络接口名称字符串与长度，调用方式为：

```c
char *dev = DEVNAME;
len = strlen(dev);
setsockopt(sock, SOL_SOCKET, SO_BINDTODEVICE, dev, len);
```

> <font color=red>批注：原作者这里取网卡接口名的长度错误。</font>

网络接口名称字符串的最大长度为IFNAMSIZ(16)字节，以空字节('\0')结束，若字符串为空，或长度为0，则套接字与网络接口之间的绑定将被移除。

## 2.3 - SO_REUSEADDR

该选项通常用于服务端应用程序的快速重启，而非同时运行同一本端套接字的多个实例。

对于正常执行主动关闭的TCP端，其套接字状态的迁移顺序依次为：FIN_WAIT1 -> FIN_WAIT2 -> TIME_WAIT，TIME_WAIT状态将持续持续2MSL (Maximum Segment Lifetime)时间，由/proc/sys/net/ipv4/tcp_fin_timeout或net.ipv4.tcp_fin_timeout指定，单位为秒；执行主动关闭的一端为TCP服务端时，在TIME_WAIT持续的时间内，若未指定SO_REUSEADDR选项，则对相同的本端套接字地址再次调用bind(2)将以EADDRINUSE错误失败。

对于TCP，该选项的设置方式为：

```c
int sfd = socket(AF_INET, SOCK_STREAM, 0);

int optval = 1;
setsockopt(sfd, SOL_SOCKET, SO_REUSEADDR, &optval, sizeof(optval));

bind(sfd, (struct sockaddr *) &addr, addrlen);
```

若需使用该功能，则每一个套接字（包括第1个）在执行bind(2)前都必须开启该选项，且原套接字的状态不能为LISTEN。

## 3. 程序验证

操作系统版本：CentOS Linux release 7.5.1804 (Core)

内核版本：3.10.0-862.2.3.el7.x86_64

gcc版本：gcc version 4.8.5 20150623 (Red Hat 4.8.5-28) (GCC)

glibc版本：GNU C Library (GNU libc) stable release version 2.17, by Roland McGrath et al.

服务端IP：eth0/22.99.22.111

客户端IP：eth0/22.99.22.101

> 服务端程序：reuse_tcp_server.c
> 
> 必选参数-i/--ip与-p/-port分别指定本端ipv4地址与端口号
> 
> 可选参数-D/--device指定套接字绑定的本地网络接口
> 
> 可选参数-E/--exit指定父进程在创建完毕子进程之后退出的标志
> 
> 可选参数-A/--reuseaddr与-P/--reuseport分别指定开启SO_REUSEADDR与SO_REUSEPORT的标志

```c
/* reuse_tcp_server.c */
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <getopt.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <string.h>
#include <errno.h>
#include <signal.h>
#include <net/if.h>

#define LISTENQ 1024
#define BUF_SIZE 64
#define SA struct sockaddr
#define ERR_EXIT(msg, ...) \
	do { fprintf(stderr, msg, ##__VA_ARGS__); exit(EXIT_FAILURE); } while (0)

static void handle_request(int);
static void set_reuse_port_addr(int, int, void *);

int main(int argc, char *argv[])
{
	if (argc < 5)
		ERR_EXIT("Usage:\n"
			"options:\n"
			" -i/--ip local ipv4 address\n"
			" -p/-port local port number\n"
			" [-D/--device] network device name\n"
			" [-E/--exit] parent exit after fork\n"
			" [-A/--reuseaddr] set the SO_REUSEADDR option\n"
			" [-P/--reuseport] set the SO_REUSEPORT option\n");
	
	int lfd, cfd;
	int reuseaddr = 0;
	int reuseport = 0;
	int parent_exit = 0;
	char *local_ip = NULL;
	short local_port = 0;
	char *dev = NULL;
	struct sockaddr_in svaddr;
	socklen_t svaddrlen;
	struct sigaction sa;

	sigemptyset(&sa.sa_mask);
	sa.sa_handler = SIG_IGN;
	if (sigaction(SIGCHLD, &sa, NULL) == -1)
		ERR_EXIT("sigaction() failed: %s\n", strerror(errno));
	
	for ( ; ; ) {
		int option_index = 0;
		const char *short_options = "i:p:D:EAP";
		static struct option long_options[] = {
			{"ip",			required_argument,	0,  'i' },
			{"port",		required_argument,	0,  'p' },
			{"device",		required_argument,	0,  'D' },
			{"exit",		no_argument,		0,  'E' },
			{"reuseaddr",	no_argument,		0,  'A' },
			{"reuseport",	no_argument,		0,  'P' },
			{0,				0,					0,   0  }
		};

		int c = getopt_long(argc, argv, short_options, long_options, &option_index);
		if (c == -1)
			break;

		switch (c) {
		case 'i':
			local_ip = optarg;
			break;
		case 'p':
			local_port = (short)atoi(optarg);
			break;
		case 'D':
			dev = optarg;
			break;
		case 'E':
			parent_exit = 1;
			break;
		case 'A':
			reuseaddr = 1;
			break;
		case 'P':
			reuseport = 1;
			break;
		case '?':
		default:
			exit(EXIT_FAILURE);
		}
	}

	if (!local_ip)
		ERR_EXIT("invalid local ip address\n");

	if (!local_port)
		ERR_EXIT("invalid local port\n");

	lfd = socket(AF_INET, SOCK_STREAM, 0);
	if (lfd == -1)
		ERR_EXIT("socket() failed: %s\n", strerror(errno));
	
	if (reuseaddr)
		set_reuse_port_addr(lfd, SO_REUSEADDR, &reuseaddr);

	if (reuseport)
		set_reuse_port_addr(lfd, SO_REUSEPORT, &reuseport);

	if (dev)
		set_reuse_port_addr(lfd, SO_BINDTODEVICE, dev);

	svaddrlen = sizeof(svaddr);
	
	memset(&svaddr, 0, svaddrlen);
	svaddr.sin_family = AF_INET;
	svaddr.sin_port = htons(local_port);

	if (inet_pton(AF_INET, local_ip, &svaddr.sin_addr) == -1)
		ERR_EXIT("inet_pton() failed: %s\n", strerror(errno));

	if (bind(lfd, (SA *)&svaddr, svaddrlen) == -1)
		ERR_EXIT("bind() failed: %s\n", strerror(errno));
	
	if (listen(lfd, LISTENQ) == -1)
		ERR_EXIT("listen() failed: %s\n", strerror(errno));
	
	for ( ; ; ) {
		cfd = accept(lfd, NULL, NULL);
		if (cfd == -1)
			ERR_EXIT("accept() failed: %s\n", strerror(errno));

		switch (fork()) {
		case -1:
			perror("fork() failed");
			close(cfd);
			break;
		case 0:
			close(lfd);
			handle_request(cfd);
			_exit(EXIT_SUCCESS);
		default:
			if (parent_exit)
				exit(EXIT_SUCCESS);

			close(cfd);
			break;
		}
	}
	
	exit(EXIT_SUCCESS);
}

static void set_reuse_port_addr(int s, int optname, void *optval)
{
	int len;
	if (optname == SO_BINDTODEVICE)
		len = (strlen(optval) > IF_NAMESIZE) ? IF_NAMESIZE : strlen(optval);
	else
		len = sizeof(int);

	if (setsockopt(s, SOL_SOCKET, optname, optval, len) == -1)
		ERR_EXIT("setsockopt(%d) failed: %s\n", optname, strerror(errno));
}

static void handle_request(int cfd)
{
	char sendbuf[BUF_SIZE] = {0};
	char recvbuf[BUF_SIZE] = {0};
	int n;

	snprintf(sendbuf, BUF_SIZE, "server: %d\n", getppid());
	if (write(cfd, sendbuf, strlen(sendbuf)) != strlen(sendbuf))
		ERR_EXIT("write() to client failed: %s\n", strerror(errno));
		
	while ((n = read(cfd, recvbuf, BUF_SIZE)) > 0) {
		if (n == 1 && recvbuf[0] == 'W') {
			if (write(cfd, "W", 1) != 1)
				ERR_EXIT("sending end of message failed: %s\n", strerror(errno));

			break;
		}
	}

	shutdown(cfd, SHUT_RDWR);
	close(cfd);
}
```

> 客户端程序：tcp_client.c
> 
> 必选参数-i与-p分别指定服务端的ipv4地址与端口号
> 
> 可选参数-n指定向服务端发起连接的数量，默认为1

```c
/* tcp_client.c */
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <string.h>
#include <signal.h>
#include <errno.h>

#define SA struct sockaddr
#define BUF_SIZE 64
#define ERR_EXIT(msg, ...) \
	do { fprintf(stderr, msg, ##__VA_ARGS__); exit(EXIT_FAILURE); } while (0)

static void connect_to_server(char *, short);
static void exchange_info(int);

int main(int argc, char *argv[])
{
	if (argc < 5)
		ERR_EXIT("Usage: %s -i <ip> -p <port> -n <count>\n", argv[0]);

	int cfd, i, c, nr_children;
	struct sockaddr_in svaddr;
	socklen_t svaddrlen;
	char *svip;
	short svport = 0;
	struct sigaction sa;

	sigemptyset(&sa.sa_mask);
	sa.sa_handler = SIG_IGN;
	if (sigaction(SIGCHLD, &sa, NULL) == -1)
		ERR_EXIT("sigaction() failed: %s\n", strerror(errno));

	nr_children = 1;
	while ((c = getopt(argc, argv, "i:p:n:")) != -1) {
		switch (c) {
		case 'i':
			svip = optarg;
			break;
		case 'p':
			svport = (in_port_t)atoi(optarg);
			break;
		case 'n':
			nr_children = atoi(optarg);
			break;
		case '?':
		default:
			exit(EXIT_FAILURE);
		}
	}

	if (!svip)
		ERR_EXIT("invalid server ip\n");

	if (!svport)
		ERR_EXIT("invalid server port\n");

	for (i = 0; i < nr_children; i++) {
		switch (fork()) {
		case -1:
			ERR_EXIT("fork() failed:%s\n", strerror(errno));
		case 0:
			connect_to_server(svip, svport);
			_exit(EXIT_SUCCESS);
		default:
			close(cfd);
			break;
		}
	}

	for ( ; ; )
		sleep(1);
	
	exit(EXIT_SUCCESS);
}

static void connect_to_server(char *svip, short svport)
{
	int cfd;
	struct sockaddr_in svaddr;
	socklen_t svaddrlen;

	cfd = socket(AF_INET, SOCK_STREAM, 0);
	if (cfd == -1)
		ERR_EXIT("socket() failed: %s\n", strerror(errno));

	memset(&svaddr, 0, sizeof(svaddr));
	svaddr.sin_family = AF_INET;
	svaddr.sin_port = htons(svport);

	if (inet_pton(AF_INET, svip, &svaddr.sin_addr) == -1)
		ERR_EXIT("inet_pton() failed: %s\n" ,strerror(errno));

	svaddrlen = sizeof(svaddr);

	if (connect(cfd, (SA *)&svaddr, svaddrlen) == -1)
		ERR_EXIT("connect() failed: %s\n", strerror(errno));

	exchange_info(cfd);
	shutdown(cfd, SHUT_RDWR);
	close(cfd);
}

static void exchange_info(int cfd)
{
	char recvbuf[BUF_SIZE] = {0};
	int n;

	while ((n = read(cfd, recvbuf, BUF_SIZE)) > 0) {
		if (n == 1 && recvbuf[0] == 'W')
			return;

		if (write(STDOUT_FILENO, recvbuf, n) != n)
			ERR_EXIT("write() to stdout failed: %s\n", strerror(errno));

		if (write(cfd, "W", 1) != 1)
			ERR_EXIT("sending end of message failed: %s\n", strerror(errno));
	}
}
```

> 连接建立后，服务端向客户端输出本端创建监听套接字的进程PID，客户端将该信息打印至标准输出，并向服务端发送字符'W'以宣告结束连接；服务端接收到该字符后立即执行主动关闭。

分别编译

```c
# gcc reuse_tcp_server.c -o reuse_tcp_server
```

```c
# gcc tcp_client.c -o tcp_client
```

## 示例1 - TCP连接的正常关闭

执行服务端程序，通过以下脚本打印tcp状态，并启动tcpdump抓包

```c
# sysctl net.ipv4.tcp_fin_timeout
net.ipv4.tcp_fin_timeout = 60
```

```c
# tcpdump -i eth0 -nn -S host 22.99.22.111 and host 22.99.22.101 and tcp
```

```c
# ./reuse_tcp_server -i 0.0.0.0 -p 8888 &
[1] 71624
```

```c
# vi trace_tcp_state.sh

#! /bin/bash

while true; do
    ss -atn state fin-wait-1 state fin-wait-2 state time-wait state close-wait state last-ack | grep :8888 | while read line; do
        [[ ${line} != "" ]] && { echo "$(date +%T.%N)  $line"; } || break
    done
done
```

```c
# ./trace_tcp_state.sh > /tmp/tcp.log &
[2] 71659
```

执行客户端程序

```c
# ./tcp_client -i 22.99.22.111 -p 8888 &
[1] 1947
# server: 71624
```

服务端抓包结果显示两端分别交换了FIN与ACK，完成4次挥手

```bash
05:49:00.360589 IP 22.99.22.101.52090 > 22.99.22.111.8888: Flags [S], seq 3903021351, win 29200, options [mss 1460,sackOK,TS val 47936384 ecr 0,nop,wscale 7], length 0
05:49:00.360701 IP 22.99.22.111.8888 > 22.99.22.101.52090: Flags [S.], seq 2388910389, ack 3903021352, win 28960, options [mss 1460,sackOK,TS val 47984822 ecr 47936384,nop,wscale 7], length 0
05:49:00.360920 IP 22.99.22.101.52090 > 22.99.22.111.8888: Flags [.], ack 2388910390, win 229, options [nop,nop,TS val 47936384 ecr 47984822], length 0
05:49:00.362624 IP 22.99.22.111.8888 > 22.99.22.101.52090: Flags [P.], seq 2388910390:2388910404, ack 3903021352, win 227, options [nop,nop,TS val 47984824 ecr 47936384], length 14
05:49:00.362993 IP 22.99.22.101.52090 > 22.99.22.111.8888: Flags [.], ack 2388910404, win 229, options [nop,nop,TS val 47936386 ecr 47984824], length 0
05:49:00.363005 IP 22.99.22.101.52090 > 22.99.22.111.8888: Flags [P.], seq 3903021352:3903021353, ack 2388910404, win 229, options [nop,nop,TS val 47936386 ecr 47984824], length 1
05:49:00.363287 IP 22.99.22.111.8888 > 22.99.22.101.52090: Flags [.], ack 3903021353, win 227, options [nop,nop,TS val 47984825 ecr 47936386], length 0
05:49:00.363466 IP 22.99.22.111.8888 > 22.99.22.101.52090: Flags [P.], seq 2388910404:2388910405, ack 3903021353, win 227, options [nop,nop,TS val 47984825 ecr 47936386], length 1
05:49:00.363600 IP 22.99.22.111.8888 > 22.99.22.101.52090: Flags [F.], seq 2388910405, ack 3903021353, win 227, options [nop,nop,TS val 47984825 ecr 47936386], length 0
05:49:00.363805 IP 22.99.22.101.52090 > 22.99.22.111.8888: Flags [F.], seq 3903021353, ack 2388910405, win 229, options [nop,nop,TS val 47936387 ecr 47984825], length 0
05:49:00.363813 IP 22.99.22.111.8888 > 22.99.22.101.52090: Flags [.], ack 3903021354, win 227, options [nop,nop,TS val 47984825 ecr 47936387], length 0
05:49:00.364039 IP 22.99.22.101.52090 > 22.99.22.111.8888: Flags [.], ack 2388910406, win 229, options [nop,nop,TS val 47936387 ecr 47984825], length 0
```

服务端脚本日志/tmp/tcp.log停止写入后，对比第一条与最后一条TIME-WAIT的时间戳，相隔约60秒

```bash
05:49:00.367070710  TIME-WAIT  0      0      22.99.22.111:8888               22.99.22.101:52090
05:50:00.529636558  TIME-WAIT  0      0      22.99.22.111:8888               22.99.22.101:52090
```

## 示例2 - SO_REUSEPORT选项

服务端以-P指定开启SO_REUSEADDR选项，在ipv4通配地址(0.0.0.0)上运行2个实例

```bash
# ./reuse_tcp_server -i 0.0.0.0 -p 8888 -P &
[1] 68554
# ./reuse_tcp_server -i 0.0.0.0 -p 8888 -P &
[2] 68556
# ss -atn | grep 8888
LISTEN     0      128          *:8888                     *:*
LISTEN     0      128          *:8888                     *:*
```
客户端指定-n选项，向服务端发送20个连接请求

```bash
# ./tcp_client -i 22.99.22.111 -p 8888 -n 20
server: 68554
server: 68554
server: 68556
server: 68554
server: 68554
server: 68554
server: 68556
server: 68554
server: 68556
server: 68556
server: 68556
server: 68556
server: 68556
server: 68556
server: 68556
server: 68556
server: 68556
server: 68554
server: 68554
server: 68556
^C
# ss -atn | grep 8888
```
两个服务端实例68554与68556分别处理了8个与12个客户端请求，并正常关闭。

## 示例3 - SO_BINDTODEVICE选项

服务端以-D指定开启SO_BINDTODEVICE选项，分别将ipv4通配地址尝试绑定至相同与不同的网络接口

```bash
# ./reuse_tcp_server -i 0.0.0.0 -p 8888 -D lo &
[1] 69025
# ./reuse_tcp_server -i 22.99.22.111 -p 8888 -D lo &
[2] 69032
# bind() failed: Address already in use

[2]+  Exit 1                  ./reuse_tcp_server -i 22.99.22.111 -p 8888 -D lo
# ./reuse_tcp_server -i 0.0.0.0 -p 8888 -D eth0 &
[2] 69055
# ss -atn | grep 8888
LISTEN     0      128     *%eth0:8888                     *:*
LISTEN     0      128       *%lo:8888                     *:*
```
运行客户端

```bash
# ./tcp_client -i 22.99.22.111 -p 8888
server: 69055
```

## 示例4 - SO_REUSEADDR选项

先后两次运行服务端，第一次不开启SO_REUSEADDR选项，父进程创建子进程后退出；第二次开启SO_REUSEADDR选项

```bash
# ./reuse_tcp_server -i 0.0.0.0 -p 8888 -E &
[1] 71163
```

```bash
# ./tcp_client -i 22.99.22.111 -p 8888
server: 71163
```

```bash
# ./reuse_tcp_server -i 0.0.0.0 -p 8888 -A &
[2] 71176
[1]   Done                    ./reuse_tcp_server -i 0.0.0.0 -p 8888 -E
# bind() failed: Address already in use

[2]+  Exit 1                  ./reuse_tcp_server -i 0.0.0.0 -p 8888 -A
#
#
[root@localhost backup]# ss -atn | grep 8888
TIME-WAIT  0      0      22.99.22.111:8888               22.99.22.101:52138
```
原套接字未开启SO_REUSEADDR选项，新套接字无法复用原ip与端口

先后两次运行服务端并开启SO_REUSEADDR选项，但原套接字的父进程不退出

```bash
# ./reuse_tcp_server -i 0.0.0.0 -p 8888 -A &
[1] 69642
```

```bash
# ./tcp_client -i 22.99.22.111 -p 8888
server: 69642
```

```bash
# ./reuse_tcp_server -i 0.0.0.0 -p 8888 -A &
[2] 69650
# bind() failed: Address already in use

[2]+  Exit 1                  ./reuse_tcp_server -i 0.0.0.0 -p 8888 -A
# ss -atn | grep 8888
LISTEN     0      128          *:8888                     *:*
TIME-WAIT  0      0      22.99.22.111:8888               22.99.22.101:52134
```
虽然两次运行都开启了SO_REUSEADDR选项，但由于用于处理监听套接字的父进程未退出而仍处于LISTEN状态，因此再次调用bind(2)失败。

再次运行服务端并开启SO_REUSEADDR选项，且令父进程创建子进程后立即退出

```bash
# ./reuse_tcp_server -i 0.0.0.0 -p 8888 -A -E &
[1] 69924
```

```bash
# ./tcp_client -i 22.99.22.111 -p 8888
server: 69924
```

```bash
# ./reuse_tcp_server -i 0.0.0.0 -p 8888 -A &
[2] 69933
[1]   Done                    ./reuse_tcp_server -i 0.0.0.0 -p 8888 -A -E
# ss -atn | grep 8888
LISTEN     0      128          *:8888                     *:*
TIME-WAIT  0      0      22.99.22.111:8888               22.99.22.101:52136
```

父进程立即被再次运行的实例69933替换，子进程继续维持TIME_WAIT状态。

## 示例5 - 通配地址

```bash
# ./reuse_tcp_server -i 0.0.0.0 -p 8888 &
[1] 70155
# ./reuse_tcp_server -i 22.99.22.111 -p 8888 &
[2] 70165
# bind() failed: Address already in use

[2]+  Exit 1                  ./reuse_tcp_server -i 22.99.22.111 -p 8888
# ps aux | grep tcp_server | grep -v grep | awk '{print $2}' | xargs kill -9
[1]+  Killed                  ./reuse_tcp_server -i 0.0.0.0 -p 8888
```

```bash
# ./reuse_tcp_server -i 22.99.22.111 -p 8888 &
[1] 70249
# ./reuse_tcp_server -i 0.0.0.0 -p 8888 &
[2] 70253
# bind() failed: Address already in use

[2]+  Exit 1                  ./reuse_tcp_server -i 0.0.0.0 -p 8888
```

```bash
# ps aux | grep tcp_server | grep -v grep | awk '{print $2}' | xargs kill -9
[1]+  Killed                  ./reuse_tcp_server -i 22.99.22.111 -p 8888
#
# ./reuse_tcp_server -i 22.99.22.111 -p 8888 &
[1] 70290
# ./reuse_tcp_server -i 127.0.0.1 -p 8888 &
[2] 70299
# ss -atn | grep 8888
LISTEN     0      128    127.0.0.1:8888                     *:*
LISTEN     0      128    22.99.22.111:8888                     *:*
```
若未指定SO_REUSEPORT/SO_BINDTODEVICE/SO_REUSEADDR，则通配地址与具体地址互斥而不允许绑定；多个具体且不相同的ipv4地址允许绑定至相同的端口。

## 4. 内核角度

bind(2)调用时需要验证本端套接字地址是否可以复用，用于验证的函数为inet_csk_get_port()与inet_csk_bind_conflict()，调用路径为sys_bind() -> inet_bind() -> inet_csk_get_port() -> inet_csk_bind_conflict()

include/net/inet_hashtables.h头文件的注释中对套接字地址复用的基本规则进行了简单说明：

```bash
/* include/net/inet_hashtables.h */

 47 /* There are a few simple rules, which allow for local port reuse by
 48  * an application.  In essence:
 49  *
 50  *  1) Sockets bound to different interfaces may share a local port.
 51  *     Failing that, goto test 2.
 52  *  2) If all sockets have sk->sk_reuse set, and none of them are in
 53  *     TCP_LISTEN state, the port may be shared.
 54  *     Failing that, goto test 3.
 55  *  3) If all sockets are bound to a specific inet_sk(sk)->rcv_saddr local
 56  *     address, and none of them are the same, the port may be
 57  *     shared.
 58  *     Failing this, the port cannot be shared.
 59  *
 		...
 77  */
```

## 5. 参考

《The Linux Programming Interface》 Chapter 61

《UNIX Network Programming Volume 1, Third Edition》Chapter 7

socket(7)

*[https://lwn.net/Articles/542629/](https://lwn.net/Articles/542629/)*

© 著作权归作者所有