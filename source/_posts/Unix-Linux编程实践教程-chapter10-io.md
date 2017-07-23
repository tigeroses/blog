---
title: Unix-Linux编程实践教程-chapter10-io
date: 2016-08-01 22:44:47
tags: [Linux,C]
category: [programming]
---

## 第１０章　I/O重定向和管道

输入/输出重定向允许完成特定功能的程序通过交换数据来进行相互协作

Unix默认规定程序从文件描述符０读取数据，写数据到文件描述符１，将
错误信息输出到文件描述符２．这三个文件描述符称为标准输入，标准输出
和标准错误输出

当登陆到Unix系统中，登陆程序设置文件描述符０，１，２．所有的连接，
文件描述符都会从父进程传递到子进程．他们也会在调用exec时被传递

创建文件描述符的系统调用总是使用最低可用文件描述符号

重定向标准输入，输出以及错误输出意味着改变文件描述符０，１，２的
连接．有很多种技术来重定向标准I/O

管道是内核中的一个数据队列，其每一端连接一个文件描述符．程序通过
使用pipe系统调用创建管道

当父进程调用fork的时候，管道的两端都被复制到子进程中

只有有共同父进程的进程之间才可以使用管道连接

两个进程都可以读写管道，但是当一个进程读，另一个进程写的时候，管道的使用效率最高

## code

``` c
/*
 * pipe.c
 * Demonstrates how to create a pipeline from on process to another
 *  Takes two args, each a command, and connects
 *  av[1]'s ouput to input of av[2]
 *  usage: pipe cmd1 cmd2
 *  effect: cmd1 | cmd2
 *  Limitations: commands do not take arguements
 *  uses execlp() since known number of args
 *  Note: exchange child and parent and watch fun
 */

#include <stdio.h>
#include <unistd.h>

#define oops(m,x)   {perror(m); exit(x);}

main(int ac, char ** av)
{
    int thepipe[2], newfd, pid;

    if (ac != 3)
    {
        fprintf(stderr, "usage: pipe cmd1 cmd2\n");
        exit(1);
    }
    if (pipe(thepipe) == -1)
        oops("Cannot get a pipe", 1);

    /*--------------------------------------------------*/
    /* now we have a pipe, now let's get two processes  */

    if ((pid = fork()) == -1)
        oops("Cannot fork", 2);

    /*--------------------------------------------------*/
    /* Right Here, there are two processes */
    /*              parent will read from pipe */

    if (pid > 0)                                // parent will exec av[2]
    {
        close(thepipe[1]);                      // parent doesn't write to pipe

        if (dup2(thepipe[0], 0) == -1)
            oops("could not redirect stdin", 3);

        close(thepipe[0]);
        execlp(av[2], av[2], NULL);
        oops(av[2], 4);
    }

    /* child execs av[1] and writes into pipe */

    close(thepipe[0]);

    if (dup2(thepipe[1], 1) == -1)
        oops("could not redirect stdout", 4);

    close(thepipe[1]);
    execlp(av[1], av[1], NULL);
    oops(av[1], 5);
}
```

