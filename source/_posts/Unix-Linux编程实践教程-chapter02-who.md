---
title: Unix-Linux编程实践教程-chapter02-who
date: 2016-07-23 16:26:24
tags: Linux C
---

## 第２章　用户　文件操作与联机帮助：编写who命令

who 命令通过读系统日志的内容显示当前已登陆的用户

Unix　系统把数据存放在文件中，可以通过以下系统调用来操作文件:
    open(filename, how)
    creat(filename, mode)
    read(fd, buffer, amt)
    write(fd, buffer, amt)
    lseek(fd, distance, base)
    close(fd)

进程对文件的读写都要通过文件描述符，文件描述符表示文件与进程之间的连接

每次系统调用都会导致用户模式与内核模式的切换以及执行内核代码，所以减少
程序中的系统调用的次数可以提高程序的运行效率

程序可以通过缓冲技术来减少系统调用的次数，仅当写缓冲区满或者读缓冲区空时才调用
内核服务

Unix 内核可以通过内核缓冲来减少访问磁盘IO 的次数

Unix　中时间的处理方式是记录从某一个时间开始经过的秒数

当系统调用出错时会把全局变量errno 的值设为相应的错误代码，然后返回-1 程序可以
通过检查errno 来确定错误的类型，并采取相应的措施

这一章涉及的知识在系统中都可以找到，联机帮助中有命令的说明，有些还会涉及命令的
实现，头文件中有结构和系统常量的定义，还有函数原型的说明

在Unix　中增加命令很简单，只要把程序的可执行文件放在以下任意目录即可：
/bin /usr/bin /usr/local/bin   或者通过alias　添加到~/.bashrc

使用系统调用open 来打开文件，如果文件被顺利打开，内核会返回一个正整数的值，
这个数值就叫做文件描述符

Unix 中时间是以一个整数来表示，他的数值是从1970 1 1 0时开始所经过的秒数，
定义在time.h 中，typedef long int time_t;

系统调用是需要时间的，程序中频繁的系统调用会降低程序的运行效率

应用缓冲技术对提高系统的效率是明显的，他的主要思想是一次读入大量的数据放入缓冲区，
需要的时候从缓冲区取得数据

调用系统调用或者API　要有出错处理

## code

``` c
/* who3.c 
 * a second version of the who program
 * open, read UTMP file, and show results
 * and filter some white record
 * and add buffter to fast the execute
 */

#include <stdio.h>
#include <utmp.h>   // define the structure which readed
#include <fcntl.h>  // function of open
#include <unistd.h> // read, close
#include <time.h>

// #define SHOWHOST    // include remote machine on output

void show_time(long);
void show_info(struct utmp * utbufp);

int main()
{
    struct utmp * utbufp, // holds pointer to next rec
                * utmp_nex(); // returns pointer to next

    if (utmp_open(UTMP_FILE) == -1)
    {
        perror(UTMP_FILE);      // UTMP_FILE is in utmp.h
        exit(1);
    }

    while ((utbufp = utmp_next()) != ((struct utmp *)NULL))
    {
        // printf("%d %d\n", reclen, utmpfd);
        show_info(utbufp);
    }
    utmp_close();
    return 0;                   // went ok
}

/* 
 * show_info()
 * display contents of the utmp struct in human readable form
 * note: these sizes should not be hardwired
 */
void show_info(struct utmp * utbufp)
{
    if (utbufp->ut_type != USER_PROCESS)
        return;

    printf("% -8.8s", utbufp->ut_user); // logname
    printf(" ");
    printf("% -8.8s", utbufp->ut_line); // tty
    printf(" ");
    show_time(utbufp->ut_tv.tv_sec);
#ifdef SHOWHOST
    if (utbufp->ut_host[0] != '\0')
        printf("(%s)", utbufp->ut_host);    // host
#endif
    printf("\n");
}

/* 
 * display time in a format fit for human consumption
 * uses ctime to build a string then picks parts out of it
 * Note: %12.12s prints a string 12 chars wid and LIMTS
 * it to 12 chars.
 */
void show_time(long timeval)
{
    // char * cp;              // to hold address of time
    // cp = ctime(&timeval);   // convert time to string
                            // string looks like
                            // Mon Feb 4 00:46:40 EST 1991
                            // 01234567938272
    // printf("%12.12s", cp+4);// pick 12 chars from pos 4
    struct tm * cp = localtime(&timeval);
    printf("%d-%d-%d %d:%d", cp->tm_year+1900, cp->tm_mon+1, cp->tm_mday,
                            cp->tm_hour, cp->tm_min);// pick 12 chars from pos 4
                
}
```

