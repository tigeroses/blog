---
title: Unix-Linux编程实践教程-chapter13-udp
date: 2016-09-02 00:02:37
tags: [Linux,C]
category: [programming]
---

## 第13章　基于数据报(Datagram)的编程：编写许可证服务器

数据报是从一个socket发送到另一个socket的短消息．数据报socket是不连接的，
每个消息包含有目的地址．数据报(UDP)socket更加简单，快速，给系统增加的负荷
更小．

许可证服务器是用来对被许可程序实施许可证验证规则的，许可证服务器发布许可，
以短消息的形式发送给客户．

许可证服务器必须记住哪个进程使用了哪个票据，必须维持一个内部的数据库．因此，
许可证服务器不同于简单的服务器．

记录系统状态的服务器必须设计成可以处理服务器和客户端的崩溃事件．

有些许可证服务器为一个网络上的多个机器提供服务．有几种设计方法，各有优缺点．

socket可以有两种类型的地址：网络或者本地．本地的socket地址叫做Unix域socket
或名字socket.这种socket使用文件名作为地址，只能在一台机器上交互数据．

## code

``` c
/*
 * dgram.c
 * support functions for datagram based programs
 */

#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netdb.h>
#include <string.h>

#define HOSTLEN 256

int make_internet_address();

int make_dgram_server_socket(int portnum)
{
    struct sockaddr_in saddr;
    char hostname[HOSTLEN];
    int sock_id;

    sock_id = socket(PF_INET, SOCK_DGRAM, 0);
    if (sock_id == -1)
        return -1;

    gethostname(hostname, HOSTLEN);
    make_internet_address(hostname, portnum, &saddr);

    if (bind(sock_id, (struct sockaddr *)&saddr, sizeof(saddr)) != 0)
        return -1;

    return sock_id;
}

int make_dgram_client_socket()
{
    return socket(PF_INET, SOCK_DGRAM, 0);
}

int make_internet_address(char *hostname, int port, struct sockaddr_in *addrp)
{
    struct hostent * hp;

    bzero((void*)addrp, sizeof(struct sockaddr_in));
    hp = gethostbyname(hostname);
    if (hp == NULL)
        return -1;

    bcopy((void*)hp->h_addr, (void*)&addrp->sin_addr, hp->h_length);
    addrp->sin_port = htons(port);
    addrp->sin_family = AF_INET;
    return 0;
}

int get_internet_address(char *host, int len, int *portp, struct sockaddr_in *addrp)
{
    strncpy(host, inet_ntoa(addrp->sin_addr), len);
    *portp = ntohs(addrp->sin_port);
    return 0;
}
```
