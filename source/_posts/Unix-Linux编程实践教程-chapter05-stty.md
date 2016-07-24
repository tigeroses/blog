---
title: Unix-Linux编程实践教程-chapter05-stty
date: 2016-07-23 16:31:41
tags: Linux C
---

## 第５章　连接控制：学习stty

内核在进程与外部世界之间交换数据．外部世界包括磁盘文件，终端与外部
设备，磁盘文件与终端的链接有相似之处也有差异

磁盘文件与设备文件都有名字，属性，和权限位．标准文件系统调用open,read
write,close,lseek可被用于文件与设备．文件权限位以同样的方式应用于
控制设备文件和磁盘文件的关闭

到磁盘文件的连接在处理和传输数据方面不同于到设备文件的连接．内核中
管理与设备链接的代码被称为设备驱动程序．通过使用fcntl ioctl，进程
可以读取和改变设备驱动程序的设置

到终端的链接是如此的重要，以致函数tcgetattr tcsetattr 专门用来提供
对终端驱动器的控制

Unix命令stty使得用户能够访问tcgetattr tcsetattr函数

测试位　if (flagset & MASK)...
置位flagset |= MASK
清除位flagset &= ~MASK

## code

``` c
/* showtty.c
 * display some current tty settings
 */
#include <stdio.h>
#include <termios.h>

main()
{
    struct termios ttyinfo;
    if (tcgetattr(0, &ttyinfo) == -1)
    {
        perror("cannot get params about stdin");
        exit(1);
    }

    showband(cfgetospeed(&ttyinfo));
    printf("The erase character is ascii %d, Ctrl- %c\n",
            ttyinfo.c_cc[VERASE], ttyinfo.c_cc[VERASE]-1+'A');
    printf("The line kill character is ascii %d, Ctrl- %c\n",
            ttyinfo.c_cc[VKILL], ttyinfo.c_cc[VKILL]-1+'A');
    show_some_flags(&ttyinfo);
}

showband(int thespeed)
/*
 * prints the speed in english
 */
{
    printf("the baud rate is ");
    switch(thespeed)
    {
        case B300: printf("300\n"); break;
        case B600: printf("600\n"); break;
        case B1200: printf("1200\n"); break;
        case B1800: printf("1800\n"); break;
        case B2400: printf("2400\n"); break;
        case B4800: printf("4800\n"); break;
        case B9600: printf("9600\n"); break;
        default:printf("Fast\n");break;
    }
}

struct flaginfo {int fl_value; char * fl_name};

struct flaginfo input_flags[] = 
{
    IGNBRK, "Ignore break condition",
    BRKINT, "Signal interrupt on break",
    IGNPAR, "Ignore chars with parity errors",
    PARMRK, "Mark parity errors",
    INPCK, "Enable input parity check",
    ISTRIP, "Strip character",
    INLCR, "Map NL to CR on input",
    IGNCR, "Ignore CR",
    ICRNL, "Map CR to NL on input",
    IXON, "Enable start/stop ouput control",
    IXOFF, "Enable start/stop input control",
    0, NULL
};

struct flaginfo local_flags[] =
{
    ISIG, "Enable signals",
    ICANON, "Canonical input(erase and kill)",
    ECHO, "Enable echo",
    ECHOE, "Echo ERASE as BS-SPACE-BS",
    ECHOK, "Echo KILL by starting new line",
    0, NULL
};

show_some_flags(struct termios *ttyp)
/*
 * show the values of two of the flag sets_:c_iflag and c_lflag
 * adding c_oflag and c_cflag is pretty routine - just add new
 * tables above and a bit more code below.
 */
{
    show_flagset(ttyp->c_iflag, input_flags);
    show_flagset(ttyp->c_lflag, local_flags);
}

show_flagset(int thevalue, struct flaginfo thebitnames[])
/*
 * check each bit pattern and display descriptive title
 */
{
    int i;
    for (i = 0; thebitnames[i].fl_value; ++i)
    {
        printf("%s is ", thebitnames[i].fl_name);
        if (thevalue & thebitnames[i].fl_value)
            printf("ON\n");
        else    
            printf("OFF\n");
    }
}
```
