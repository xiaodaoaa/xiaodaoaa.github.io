---
title: UDP绑定网络接口
date: 2021-12-06 15:06:47
tags: [UDP,linux]
categories: UDP
description: UDP绑定网络接口，适用于linux。
---



UDP通信时指定网络接口：

```c++
#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <net/if.h>
#include <errno.h>

int main()
{
    int sock;
    struct sockaddr_in addr, raddr;
    char buffer[] = "hello world";
    /* 指定接口 */
    struct ifreq nif;
    char *inface = "eth0";
    strcpy(nif.ifr_name, inface);
    /* 创建套接字 */
    sock = socket(AF_INET, SOCK_DGRAM, 0);
    if (-1 == sock)
    {
        printf("create socket fail \r\n");
        return;
    }
    else
    {
        printf("create socket success \r\n");
    }
    
    if (inet_aton("192.168.80.129", &addr.sin_addr) != 1)
    {
        printf("addr convert fail \r\n");
        close(sock);
        return;
    }
    addr.sin_family = AF_INET;
    addr.sin_port = htons(8000);
    if (inet_aton("192.168.80.1", &raddr.sin_addr) != 1)
    {
        printf("addr convert fail \r\n");
        close(sock);
        return;
    }
    raddr.sin_family = AF_INET;
    raddr.sin_port = htons(8000);
    #if 0  //绑定地址
    if (bind(sock, (struct sockadd *)&addr, sizeof(addr)) != 0)
    {
        printf("bind fail \r\n");
        close(sock);
        return ;
    }
    else
    {
        printf("bind success \r\n");
    }
    #endif
    /* 绑定接口 */
    if (setsockopt(sock, SOL_SOCKET, SO_BINDTODEVICE, (char *)&nif, sizeof(nif)) < 0)
    {
        close(sock);
        printf("bind interface fail, errno: %d \r\n", errno);
        return ;        
    }
    else
    {
        printf("bind interface success \r\n");
    }
    /* 发送 */
    sendto(sock, buffer, sizeof(buffer), 0, ((struct sockadd *)&raddr),sizeof(raddr));
    
    close(sock);
    return;
}
```

