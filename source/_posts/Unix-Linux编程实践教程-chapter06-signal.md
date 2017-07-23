---
title: Unix-Linux编程实践教程-chapter06-signal
date: 2016-07-23 16:31:47
tags: [Linux,C]
category: [programming]
---

## 第６章　为用户编程：终端控制和信号

有些程序处理从特定设备来的数据．这些与特定设备相关的程序
必须控制与设备的链接．Ｕｎｉｘ系统中最常见的设备是终端

终端驱动程序有很多设置．各个设置的特定值决定了终端驱动程序的模式．
为用户编写的程序通常需要设置终端驱动程序为特定的模式

键盘输入分为三类，终端驱动程序对这些输入做不同的处理．大多数键
代表常规数据，他们从驱动程序传输到程序，有些键调用驱动程序中的编辑
函数．如果按下删除键，驱动程序将前一个字符从他的行缓冲中删除，并将
命令发送到终端屏幕，使之从显示器中删除字符．最后，有些键调用处理
控制函数．Ｃｔｒｌ－Ｃ键告诉驱动程序调用内核中某个函数，这个函数给进程
发送一个信号．终端驱动程序支持若干种处理控制函数，他们都通过发送信号到
进程来实现控制

信号是从内核发送给进程的一种简短消息．信号可能来自用户，其他进程，或
内核本身．进程可以告诉内核，在他收到信号时需要做出怎样的响应

终端模式：
１　规范模式
    常见模式，驱动程序输入的字符保存在缓冲，接收到回车才发送到程序

２　非规范模式
    缓冲和编辑功能被关闭．stty -icanon

３　ｒａｗ模式
    每个处理步骤都被一个独立的位控制

由进程的某个操作产生的信号被称为同步信号 synchronous signals
由像用户击键这样的进程外的事件引起的信号被称为异步信号　asynchronous signals

进程如何处理信号：
１　接受默认处理
２  忽略信号
３  调用一个函数

大多数signal都可以被捕获或者忽略，但有两个无法被忽略，是SIGKILL SIGSTOP

## code

``` c

/*
 * play_again3.c
 * purpose: ask if user wants another transaction
 * method: set tty into chr-by-chr mode and no-echo mode
 *          set tty into no-delay mode
 *          read char, return result
 * returns: 0 => yes, 1 => no, 2=> timeout
 * better: reset terminal mode on Interrupt
 */
#include <stdio.h>
#include <termios.h>
#include <fcntl.h>
#include <string.h>

#define ASK "Do you want another transaction"
#define TRIES 3     // max tries
#define SLEEPTIME 2 // time per try
#define BEEP putchar('\a')  // alert user

int main()
{
    int response;
    tty_mode(0);                        // save tty mode
    set_cr_noecho_mode();               // set -icanon, -echo
    set_nodelay_mode();                 // noinput => EOF
    response = get_response(ASK, TRIES);
    tty_mode(1);                        // restore tty mode
    return response;
}

int get_response(char * question, int maxtries)
{
    int input;
    printf("%s (y/n)?", question);
    fflush(stdout);                 // force output
    while(1)
    {
        sleep(SLEEPTIME);               // wait a bit
        input = tolower(get_ok_char()); // get next chr
        if (input == 'y')
            return 0;
        if (input == 'n')
            return 1;
        if (maxtries-- == 0)
            return 2;
        BEEP;
    }
}

// skip over non-legal chars and return y,Y,n,N or EOF
get_ok_char()
{
    int c;
    while ((c = getchar()) != EOF && strchr("yYnN", c) == NULL)
        ;
    return c;
}

set_cr_noecho_mode()
/*
 * purpose: put file descriptior 0 into chr-by-chr mode and noecho mode
 * method: use bits in termios
 */
{
    struct termios ttystate;
    tcgetattr(0, &ttystate);        // read curr. setting
    ttystate.c_lflag &= ~ICANON;    // no buffering
    ttystate.c_lflag &= ~ECHO;      // no echo either
    ttystate.c_cc[VMIN] = 1;        // get one char at a time
    tcsetattr(0, TCSANOW, &ttystate);   // install setting
}

set_nodelay_mode()
/* purose: put file descriptor 0 into no-delay mode
 * method: use fcntl to set bits
 * notes: tcsetattr() will do something similar, but it is complicated
 */
{
    int termflags;
    termflags = fcntl(0, F_GETFL);      // read curr. settings
    termflags |= O_NDELAY;              // flip on nodelay bit
    fcntl(0, F_SETFL, termflags);       // and install 'em
}

// how == 0 => save currentmode
// how == 1 => restore mode
// this version handles termios and fcntl flags
tty_mode(int how)
{
    static struct termios original_mode;
    static int original_flags;
    if (how == 0)
    {
        tcgetattr(0, &original_mode);
        original_flags = fcntl(0, F_GETFL);
    }
    else
    {
        tcsetattr(0, TCSANOW, &original_mode);
        fcntl(0, F_SETFL, original_flags);
    }
}
```
