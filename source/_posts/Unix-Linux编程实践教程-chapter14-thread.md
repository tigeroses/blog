---
title: Unix-Linux编程实践教程-chapter14-thread
date: 2016-09-06 22:05:56
tags: Linux C thread
---

## 第14章　线程机制:并发函数的使用

执行线路即为程序的控制流程．pthreads的线程库允许程序在同一时刻运行多个函数

同时执行的各函数都拥有自己的局部变量，但共享所有的全局变量和动态分配的数据空间

当线程共享变量时，必须保证他们不会发生共享冲突．线程使用互斥锁保证在某一时刻只有
一个线程在对共享变量访问

线程间通过条件变量来互相通知和同步数据．一个线程挂起并等待着条件变量按照某种特定
方式变化，而另一个线程则发出信号使得条件变量发生变化

线程需要使用互斥量来避免对于共享资源操作函数的访问冲突．非重入的函数必须按照
这种方式进行保护

进程间可以通过管道　socket 信号　退出／等待以及运行环境来进行会话．线程因为是在
一个单独的进程中运行，共享全局变量，因此线程可以通过设置和读取这些全局变量来
进行通信，对共享内存的访问，既有用也危险

## code

``` c
/*
 * test_mutex.c
 */

#include <stdio.h>
#include <pthread.h>

int total = 0;
int times = 100;
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

struct arg_set
{
    int count;
};

int main()
{
    pthread_t t1, t2;

    /*
    void * add();

    pthread_create(&t1, NULL, add, NULL);
    pthread_create(&t2, NULL, add, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("%d\n", total);
    */

    void * add2(void *);


    struct arg_set a1, a2;
    a1.count = 0;
    a2.count = 0;
    pthread_create(&t1, NULL, add2, (void *)&a1);
    pthread_create(&t2, NULL, add2, (void *)&a2);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("%d\n", a1.count + a2.count);

    return 0;
}

// First method: add mutex
void * add()
{
    for (int i = 0; i < times; i++)
    {
        pthread_mutex_lock(&lock);
        total++;
        pthread_mutex_unlock(&lock);
    }
}

// Second method: cal single
void * add2(void * a)
{
    struct arg_set *arg = a;
    for (int i = 0; i < times; i++)
    {
        arg->count++;
    }
}
```

``` c
/*
 * test_mutex.c
 *
 * pthread_cond_wait(pthread_cond_t * cond, pthread_mutex_t * mutex)
 *      使线程挂起直到另一个线程通过条件变量发出消息.先自动释放指定的锁，
 *      然后等待条件变量的变化
 * pthread_cond_signal(pthread_cond_t * cond)
 *      通过条件变量cond 发消息
 */

#include <stdio.h>
#include <pthread.h>

int times = 100000000;
int total = 0;
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t flag = PTHREAD_COND_INITIALIZER;

struct arg_set
{
    int count;
    int times;
};

struct arg_set * mailbox;

int main()
{
    pthread_t t1, t2;

    void * add2(void *);

    int reports_in = 0;

    struct arg_set a1, a2;

    pthread_mutex_lock(&lock);

    a1.count = 0;
    a1.times = 10000000;
    a2.count = 0;
    a2.times = 1000;
    pthread_create(&t1, NULL, add2, (void *)&a1);
    pthread_create(&t2, NULL, add2, (void *)&a2);

    while (reports_in < 2)
    {
        printf("MAIN: waiting for flag\n");
        pthread_cond_wait(&flag, &lock);
        printf("MAIN: I have the lock\n");
        printf("%d\n", mailbox->count);
        total += mailbox->count;
        if (mailbox == &a1)
            pthread_join(t1, NULL);
        if (mailbox == &a2)
            pthread_join(t2, NULL);
        mailbox = NULL;
        pthread_cond_signal(&flag);
        reports_in ++;
    }

    printf("total %d\n", total);

    return 0;
}

void * add2(void * a)
{
    struct arg_set *arg = a;
    for (int i = 0; i < arg->times; i++)
    {
        arg->count++;
    }

    printf("COUNT: waiting to get lock\n");
    pthread_mutex_lock(&lock);
    printf("COUNT: have lock\n");
    if (mailbox != NULL)
        pthread_cond_wait(&flag, &lock);
    mailbox = arg;
    pthread_cond_signal(&flag);
    pthread_mutex_unlock(&lock);

    return NULL;
}
```
