---
title: Unix-Linux编程实践教程-chapter01-more
date: 2016-07-23 16:09:39
tags: [Linux,C]
category: [programming]
---

## 第一章　Unix系统编程概述

程序中所有对设备的操作都是通过内核进行的

在登陆过程中，当用户名和密码通过验证后，系统会启动一个叫做shell的进程，然后把
用户交给这个进程，由这个进程处理用户的请求，每个用户都有属于自己的shell进程

ps命令可以列出系统中运行的所有进程

自己动手实践一个more，用来查看文件

Unix编程不是很难，但也不是轻而易举的事情

计算机系统中包含了很多系统资源，如硬盘，内存，外围设备，网络连接等，程序利用
这些资源来对数据进行存储，转换和处理

多用户系统需要一个中央管理程序，Unix的内核就是这样的程序，它可以对程序和资源进行管理

用户程序访问设备必须通过内核

一些Unix的系统功能是由多个程序的协作而实现的

要编写系统程序，必须对系统调用和相关的数据结构有深入的理解

## code

``` c
#include <stdio.h>

#define PAGELEN 24
#define LINELEN 512

void do_more(FILE *);
int see_more(FILE *);

int main(int argc, char * argv[])
{
    FILE * fp;
    if (argc == 1)
        do_more(stdin);
    else
        while(--argc)
            if ((fp = fopen(* ++argv, "r")) != NULL)
            {
                do_more(fp);
                fclose(fp);
            }
            else
                exit(1);
    return 0;
}

void do_more(FILE * fp)
{
    char line[LINELEN];
    int num_of_lines = 0;
    int see_more(FILE *), reply;
    FILE * fp_tty;
    fp_tty = fopen("/dev/tty", "r");
    if (fp_tty == NULL)
        exit(1);
    while(fgets(line, LINELEN, fp))
    {
        if (num_of_lines == PAGELEN)
        {
            reply = see_more(fp_tty);
            if (reply == 0)
                break;
            num_of_lines -= reply;
        }
        if (fputs(line, stdout) == EOF)
            exit(1);
        num_of_lines++;
    }
}

int see_more(FILE * cmd)
{
    int c;
    printf("\033[7m more? \033[m");
    while ((c = getc(cmd)) != EOF)
    {
        if (c == 'q')
            return 0;
        if (c == ' ')
            return PAGELEN;
        if (c == '\n')
            return 1; 
    }
    return 0;
}
```
