---
title: Unix-Linux编程实践教程-chapter07-vediogame
date: 2016-07-23 16:32:06
tags: [Linux,C]
category: [programming]
---

## 第７章　事件驱动编程：编写一个视频游戏

有些程序的控制流很简单．而另外一些则要响应外部的事件．一个
视频游戏要响应时钟和用户输入，操作系统也要响应时钟和外设

curses库有一些可以管理屏幕显示字符的函数

一个进程通过设置计时器来安排事件．每个进程有三个独立的计时器．
计时器通过发送信号来通知进程．每个计时器都可以被设置为只发送
一次信号，或者按固定的间隙发送信号

处理一个信号很简单．同时处理多个信号就复杂了．进程能决定是忽略
信号还是阻塞信号．进程能告知内核哪些信号在什么时候阻塞或忽略

有些函数执行一些复杂的任务是不能被打断的．程序可以通过小心地
使用信号掩码来保护这些临界区代码

curses库基本函数:
initscr()   初始化curses库和tty
endwin()    关闭curses并重置tty
refresh()   使屏幕按照你的意图显示
move(r, c)  移动光标到屏幕的r c位置
addstr(s)   在当前位置画字符串s
mvaddch(r,c,'s')
clear()     清屏
standout()  启动standout模式(一般使屏幕反色)
standend()  关闭standout模式


调用pause 可以挂起进程直到有一个信号被处理

Unix很早就有sleep alarm，但他们的精度是秒，后来有了一个新的
系统，叫间隔计时器interval timer,有更高的精度 usleep(n)ｎ为微秒
三个计时器分别是：
真实 ITIMER_REAL 执行用户代码与内核代码所用时间
进程 ITIMER_VIRTUAL 用户态运行时间
实用 ITIMER_PROF 

虽然每个进程有三个独立的计时器，但其实每个系统只需要一个时钟来
设置节拍．每当内核收到系统时钟脉冲，他遍历所有的间隔计时器，
使每个计数器减去一个时钟单位，当某进程计数器为０，则内核发送SIGALRM
给此进程．

一段修改一个数据结构的代码如果在运行时被打断将导致数据得不完整或损毁，
则称这段代码为临界区，临界区需要保护，最简单办法就是阻塞或者忽略那些
处理函数将要使用或修改特定数据的信号．

kill向一个进程发送一个信号，两个进程用户ＩＤ必须一样，或者发送者是
超级用户

## code

``` c

/* bounce2d 1.0
 * bounce a character around the screen
 * defined by some parameters
 *
 * user input: s slow down x component
 *             S slow down y component
 *             f speed up x component
 *             F speed up y component
 *             Q quit
 *
 * blocks on read, but timer tick sends SIGALRM caught by ball_move
 * build: cc bounce2d.c -l curses -o bounce2d
 */
#include <curses.h>
#include <signal.h>
#include <sys/time.h>
#include "bounce.h"

struct ppball the_ball;
struct board the_board;

// the main loop
void set_up();
void wrap_up();

int main()
{
    FILE *fp;
    fp = fopen("result", "w");
    
    int c;
    set_up();
    while (1)
    {
        c = getch();
        if (c == 'Q') break;

        fputc(c, fp);
        if (c == 'f') the_ball.x_ttm--;
        else if (c == 's') the_ball.x_ttm++;
        else if (c == 'F') the_ball.y_ttm--;
        else if (c == 'S') the_ball.y_ttm++;
        else if (c == 'a') 
        {
            if (the_board.left > LEFT_EDGE)
                the_board.how = -1;
        }
        else if (c == 'd')
        {
            if (the_board.left+the_board.length < RIGHT_EDGE)
                the_board.how = 1;
        }
        else
            the_board.how = 0;

    }
    wrap_up();

    fclose(fp);
    return 0;
}

// init structure and other stuff
void set_up()
{
    the_board.length = 3;
    the_board.left = (RIGHT_EDGE+LEFT_EDGE)/2;
    the_board.row = ERROR+1;
    the_board.how = 0;

    the_ball.y_pos = Y_INIT;
    the_ball.x_pos = X_INIT;
    the_ball.y_ttg = the_ball.y_ttm = Y_TTM;
    the_ball.x_ttg = the_ball.x_ttm = X_TTM;
    the_ball.y_dir = 1;
    the_ball.x_dir = 1;
    the_ball.symbol = DFL_SYMBOL;

    
    initscr();
    noecho();
    crmode();

    int i;
    for (i = LEFT_EDGE-1; i <= RIGHT_EDGE+1; ++i)
    {
        mvaddch(TOP_ROW-1, i, X_BUND);
        //mvaddch(ERROR+1, i, XX_BUND);
        //mvaddch(i, TOP_ROW, X_BUND);
        //mvaddch(i, BOT_ROW, X_BUND);
    }
    for (i = TOP_ROW; i <= ERROR+1; ++i)
    {
        mvaddch(i, LEFT_EDGE-1, Y_BUND);
        mvaddch(i, RIGHT_EDGE+1, Y_BUND);
        //mvaddch(RIGHT_EDGE, i, Y_BUND);
        //mvaddch(LEFT_EDGE, i, Y_BUND);
    }


    for (i = the_board.left; i < the_board.left + the_board.length; ++i)
        mvaddch(the_board.row, i, '_');

    void ball_move(int);
    signal(SIGINT, SIG_IGN);
    mvaddch(the_ball.y_pos, the_ball.x_pos, the_ball.symbol);
    refresh();
    

    signal(SIGALRM, ball_move);
    set_ticker(1000/TICKS_PER_SEC);

}

void wrap_up()
{
    set_ticker(0);
    endwin();
}

void ball_move(int signum)
{
    int y_cur, x_cur, moved;

    signal(SIGALRM, SIG_IGN);       // dont get caught now
    y_cur = the_ball.y_pos;         // old spot
    x_cur = the_ball.x_pos;
    moved = 0;

    if (the_ball.y_ttm > 0 && the_ball.y_ttg-- == 1)
    {
        the_ball.y_pos += the_ball.y_dir;       // move
        the_ball.y_ttg = the_ball.y_ttm;        // reset
        moved = 1;
    }

    if (the_ball.x_ttm > 0 && the_ball.x_ttg-- == 1)
    {
        the_ball.x_pos += the_ball.x_dir;   // move
        the_ball.x_ttg = the_ball.x_ttm;        // reset
        moved = 1;
    }

    if (the_board.how != 0)
        change_board(the_board.how);
    if (moved)
    {
        mvaddch(y_cur, x_cur, BLANK);
        mvaddch(y_cur, x_cur, BLANK);
        mvaddch(the_ball.y_pos, the_ball.x_pos, the_ball.symbol);
        bounce_or_lose(&the_ball);
        move(LINES-1, COLS-1);
        refresh();
    }

    signal(SIGALRM, ball_move);     // for unreliable systems
}


int bounce_or_lose(struct ppball * bp)
{
    int return_val = 0;

    if (bp->y_pos == TOP_ROW)
    {
        bp->y_dir = 1;
        return_val = 1;
    }
    /*
    else if (bp->y_pos == BOT_ROW)
    {
        bp->y_dir = -1;
        return_val = 1;
    }
    */
    else if (bp->y_pos == ERROR + 1)
    {
        if (bp->x_pos < the_board.left || bp->x_pos > the_board.left + the_board.length)
            game_over();
        else
        {
            bp->y_dir = -1;
            return_val = 1;
        }
    }

    if (bp->x_pos == LEFT_EDGE)
    {
        bp->x_dir = 1;
        return_val = 1;
    }
    else if (bp->x_pos == RIGHT_EDGE)
    {
        bp->x_dir = -1;
        return_val = 1;
    }
    return return_val;
}

void game_over()
{
    char * s = "GAME OVER";
    int i, j;
    for (i = (LEFT_EDGE+RIGHT_EDGE)/2-4, j = 0; j <= sizeof(s)/sizeof(char); ++i, ++j)
        mvaddch((TOP_ROW+BOT_ROW)/2, i, s[j]);
    refresh();
    //wrap_up();
    signal(SIGALRM, SIG_DFL);
    sleep(2);
    endwin();
    exit(-1);
}

void change_board(int num)
{
    int i;
    for (i = the_board.left; i < the_board.left + the_board.length; ++i)
        mvaddch(the_board.row, i, ' ');
    the_board.left += num;
    for (i = the_board.left; i < the_board.left + the_board.length; ++i)
        mvaddch(the_board.row, i, '_');
    the_board.how = 0;
    //refresh();
}

/* [from set_ticker.c]
 * set_ticker(number_of_milliseconds)
 * arranges for interval timer to issue SIGALRMs at regular intervals
 * returns -1 on error, 0 for ok
 * arg in milliseconds, converted into whole seconds and mocroseconds
 * note: set_ticker(0) turns off ticker
 */
int set_ticker(int n_msecs)
{
    struct itimerval new_timeset;
    long n_sec, n_usecs;

    n_sec = n_msecs / 1000;             // int part
    n_usecs = (n_msecs % 1000) * 1000L; // remainder

    new_timeset.it_interval.tv_sec = n_sec;    // set reload
    new_timeset.it_interval.tv_usec = n_usecs; // new ticker value

    new_timeset.it_value.tv_sec = n_sec;    // store this
    new_timeset.it_value.tv_usec = n_usecs;

    return setitimer(ITIMER_REAL, &new_timeset, NULL);
}
```
