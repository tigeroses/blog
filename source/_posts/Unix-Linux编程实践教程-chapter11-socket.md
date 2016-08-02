---
title: Unix-Linux编程实践教程-chapter11-socket
date: 2016-08-03 00:51:07
tags: Linux C
---

## 第１１章　连接到近端或远端的进程：服务器与Socket(套接字)

一些程序被作为单独的进程建立起来来接受和发送数据．在客户/服务器模型中，
服务器进程为客户进程提供处理或数据服务

客户/服务器系统包含通信系统和协议．客户和服务器通过管道或socket进行通信．
协议是会话过程中一系列规则的集合

popen库函数可以将任何shell程序嵌入服务器程序并且让对服务器的访问就像访问
缓存文件一样

管道是一对相连接的文件描述符．socket是一个未连接的通信端点，也是一个潜在
的文件描述符．客户进程通过把自己的socket和服务器端的socket相连来创建一个
通信连接

sockets之间的连接可以扩展到另一台机器上．每个socket以机器地址和端口来标识

到管道和socket的连接使用文件描述符．文件描述符为程序提供了与文件，设备和
其他的进程通信的统一编程接口

Unix中的计算器:bc
bc在内部启动了dc计算器程序，并通过管道与其进行通信

从bc方法中得到的思想：
１　客户/服务器模型
    bc/dc程序对是客户/服务器模型程序设计的一个实例．bc通过用户界面，
    并使用dc提供的服务．这里bc被称为dc的客户
２　双向通信
    客户/服务器模型不同于生产线的数据处理模型，他要求一个进程既跟另
    一个进程的标准输入也要和他的标准输出进行通信
３　永久性服务
    bc让单一的dc进程处于运行状态，也就是bc不断的与dc的同一个实例进行
    通信，而shell对每一个命令都创建一个新的进程
    bc/dc对被称之为协同进程(coroutines)以用来区别于子程序(subroutines)
    两个程序都持续运行，当其中的一个程序完成自己的工作后将把控制权传给
    另一个程序

bc的流程：
１　创建两个管道
２  创建一个进程来运行dc
３  在新创建的进程中，重定向标准输入和标准输出到管道，然后运行exec dc
４  在父进程中，读取并分析用户的输入，将命令传给dc,dc读取响应，并把
    响应传给用户

如果知道文件名，可以用fopen打开设备文件
如果只知道文件描述符，可以用fdopen命令:W

fopen打开一个指向文件的带缓冲的连接
FILE * fp;
fp = fopen("file", "r");
c = getc(fp);
fgets(buf, len, fp);
fscanf(fp, "%d%d%s", &x, &y, &z);
fclose(fp);

popen打开一个指向进程的带缓冲的连接
FILE * fp;
fp = popen("ls", "r");
fgets(buf, len, fp);
pclose(fp);

如果不关闭，进程会变成僵尸进程．pclose中调用了wait函数等待进程的结束

访问数据：文件，应用程序接口，服务器
文件
    依赖于特定的文件格式和结构体中特定的成员名称
函数
    就算底层存储结构改变，接口程序依然可用
进程
    使用进程，也就是调用独立的程序来获取数据，而不是自己写的程序


管道使得进程对其他进程发送数据就像向文件发送数据一样容易，但是因为管道
在进程中被创建，通过fork来实现共享，所以管道只能连接位于同一台主机上的
进程，而要与远端进程连接，需要使用socket

客户和服务器
    服务器是提供服务的程序，是一个进程，等待请求，处理请求，然后循环回去
    等下一个请求．客户端进程只要建立连接，与服务器交换数据即可

主机名和端口
    运行于因特网上的服务器其实是某台计算器上运行的一个进程．服务器在该
    主机拥有一个端口．主机和端口的组合才标识了一个服务器

协议
    协议是服务器和客户之间交互的规则．每个客户/服务器模型都必须定义一个
    协议并遵守,只要两者相互认可就行，比如约定数据的格式等

/etc/services中定义了常用的服务器端口列表

socket中服务器流程：
１　向内核申请一个socket
    socket是一个通信端点　系统调用socket创建一个socket
２  绑定地址到socket上，地址包括主机，端口
    bind调用把一个地址分配给socket
３  在socket上，允许接入呼叫并设置队列长度
    使用listen 监听端口
４　等待/接收呼叫
    使用accept来接收调用，accept阻塞当前进程，直到指定socket上的接入连接
    被建立起来，然后返回文件描述符进行读写操作
５  传输数据
６  关闭连接
    close系统调用

socket中客户端流程：
１　向内核申请建立socket
２  与服务器连接
    connect系统调用
３  传送数据
４  关闭连接

对于任何运行参数中所含的命令或从因特网上获取数据的服务器，在编写时都要格外
小心，比如收到用户参数里有";rm *"

## code

使用管道实现进程间通信,bc dc计算器的实现
``` c
/*
 * tinybc.c
 * a tiny calculator that uses dc to do its work
 * demonstrates bidirectional pipes
 * input looks like number op number which
 * tinybc converts into number \n number \n op \n p
 * and passes result back to stdout
 *
 * program outline:
 * a. get two pipes
 * b. fork(get another process)
 * c. in the dc-to-be process,
 *      connect stdin and out to pipes
 *      then execl dc
 * d. in the tinybc-process, no plumbing to do
 *      just talk to human via normal i/o
 *      and send stuff via pipe
 * e. close pipe and dc dies
 *
 * note: does not handle multiline answers
 */

#include <stdio.h>

#define oops(m,x) {perror(m); exit(x);}

main()
{
    int pid, todc[2], fromdc[2];            // equipment

    // make two pipes
    if (pipe(todc) == -1 || pipe(fromdc) == -1)
        oops("pipe failed", 1);

    // get a process for user interface
    if ((pid = fork()) == -1)
        oops("cannot fork", 2);

    if (pid == 0)                           // child is dc
        be_dc(todc, fromdc);
    else
    {
        be_bc(todc, fromdc);                // parent is ui
        wait(NULL);
    }
}

be_dc(int in[2], int out[2])
/*
 * set up stdin and stdout, then execl dc
 */
{
    // setup stdin from pipein
    if (dup2(in[0], 0) == -1)              // copy read end to 0
        oops("dc:cannot redirect stdin", 3);
    close(in[0]);                           // moved to fd 0
    close(in[1]);                           // won't write here

    // setup stdout from pipeout
    if (dup2(out[1], 1) == -1)              // copy write end to 1
        oops("dc:cannot redirect stdin", 4);
    close(out[1]);                           // moved to fd 1
    close(out[0]);                           // won't read from here

    // now execl dc with the - option
    execlp("dc", "dc", "-", NULL);
    oops("Cannot run dc", 5);
}

be_bc(int todc[2], int fromdc[2])
/*
 * read from stdin and convert into to RPN, send down pipe
 * then read from other pipe and print to user
 * Uses fdopen() to convert a file descriptor to a stream
 */
{
    int num1, num2;
    char operation[BUFSIZ], message[BUFSIZ], *fgets();
    FILE *fpout, *fpin, *fdopen();

    // setup
    close(todc[0]);                     // won't read from pipe to dc
    close(fromdc[1]);                     // won't write to pipe from dc

    fpout = fdopen(todc[1], "w");
    fpin = fdopen(fromdc[0], "r");

    if (fpout == NULL || fpin == NULL)
        fatal("Error convering pipes to streams");
    // main loop
    while (printf("tinybc: "), fgets(message, BUFSIZ, stdin) != NULL)
    {
        // parse input
        if (sscanf(message, "%d%[-+*/^]%d",
                    &num1, operation, &num2) != 3)
        {
            printf("syntax error\n");
            continue;
        }

        if (fprintf(fpout, "%d\n%d\n%c\np\n", num1, num2, *operation) == EOF)
            fatal("Error writing");

        fflush(fpout);
        if (fgets(message, BUFSIZ, fpin) == NULL)
            break;

        printf("%d %c %d = %s", num1, *operation, num2, message);
    }

    fclose(fpout);                  // close pipe
    fclose(fpin);                   // dc will see EOF
}

fatal(char *mess[])
{
    fprintf(stderr, "Error: %s\n", mess);
    exit(1);
}
```

使用socket实现服务器客户端远程通信
``` c
/*
 * timeserv.c - a socket based time of day server
 */

#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netdb.h>
#include <time.h>
#include <strings.h>

#define PORTNUM 13000
#define HOSTLEN 256
#define oops(msg)   {perror(msg); exit(1);}

int main(int ac, char *av[])
{
    struct sockaddr_in  saddr;      // build our address here
    struct hostent  *hp;            // this is part of our address
    char hostname[HOSTLEN];
    int sock_id, sock_fd;
    FILE *sock_fp;
    char *ctime();                  // convert secs to string
    time_t  thetime;

    // Step1: ask kernel for a socket
    sock_id = socket(PF_INET, SOCK_STREAM, 0);
    if (sock_id == -1)
        oops("socket");

    // Step2: bind address to socket. Address is host, port
    bzero((void *)&saddr, sizeof(saddr));       // clear out struct

    gethostname(hostname, HOSTLEN);
    hp = gethostbyname(hostname);

    // fill in host part
    bcopy((void *)hp->h_addr, (void *)&saddr.sin_addr, hp->h_length);
    saddr.sin_port = htons(PORTNUM);        // fill in socket port
    saddr.sin_family = AF_INET;             // fill in addr family

    if (bind(sock_id, (struct sockaddr *)&saddr, sizeof(saddr)) != 0)
        oops("bind");

    // Step3: allow incoming calls with Qsize=1 on socket
    if (listen(sock_id, 1) != 0)
        oops("listen");

    // main loop: accept(), write(), close()
    while (1)
    {
        sock_fd = accept(sock_id, NULL, NULL);      // wait for a call
        printf("Wow! got a call!\n");
        if (sock_fd == -1)
            oops("accept");

        sock_fp = fdopen(sock_fd, "w");             // wirte to the socket as a stream
        if (sock_fp == NULL)
            oops("fdopen");

        thetime = time(NULL);

        fprintf(sock_fp, "The time here is ..");
        fprintf(sock_fp, "%s", ctime(&thetime));

        fclose(sock_fp);
    }
}
```

``` c
/*
 * timeclnt.c - a client for timeserv.c
 * usage: timeclnt hostname protnumber
 */

#include <stdio.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netdb.h>

#define oops(msg)   {perror(msg); exit(1);}

int main(int ac, char *av[])
{
    struct sockaddr_in  servadd;     
    struct hostent  *hp;            
    int sock_id, sock_fd;
    char message[BUFSIZ];
    int messlen;

    // Step1: get a socket
    sock_id = socket(PF_INET, SOCK_STREAM, 0);
    if (sock_id == -1)
        oops("socket");

    // Step2: connect to server
    //          need to build address(host,port) of server first
    bzero(&servadd, sizeof(servadd));       // clear out struct

    hp = gethostbyname(av[1]);
    if (hp == NULL)
        oops(av[1]);

    // fill in host part
    bcopy(hp->h_addr, (struct sockaddr *)&servadd.sin_addr, hp->h_length);
    servadd.sin_port = htons(atoi(av[2]));        // fill in socket port
    servadd.sin_family = AF_INET;             // fill in addr family

    if (connect(sock_id, (struct sockaddr *)&servadd, sizeof(servadd)) != 0)
        oops("connect");

    // Step3: transfer data from server, then hangup
    messlen = read(sock_id, message, BUFSIZ);
    if (messlen == -1)
        oops("read");
    if (write(1, message, messlen) != messlen)
        oops("write");
    
    close(sock_id);
}
```
