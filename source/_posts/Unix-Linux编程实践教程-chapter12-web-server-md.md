---
title: Unix-Linux编程实践教程-chapter12-web-server
date: 2016-08-30 22:34:12
tags: [Linux,C]
category: [programming]
---

## 第１２章　连接和协议：编写Web服务器

基于socket的客户/服务器程序遵循一个标准架构．服务器接收和处理请求，客户
发出请求

服务器建立服务器端socket.服务器端socket有具体的地址，用来接收连接

客户创建和使用客户端socket．客户并不关心客户端socket的地址

服务器可以用两种方法之一处理请求：自己处理，或fork创建新进程处理请求

Web服务器是最受欢迎的基于socket的程序．Web服务器处理３种类型的请求：
返回文件内容，目录列表和运行程序．请求和应答协议称为HTTP

服务器的设计问题：DIY或代理
DIY:服务器接收请求，自己处理工作．用于快速简单的任务
代理：服务器接收请求，然后创建一个新进程处理工作．用于慢速复杂的任务

Web服务器协议：
HTTP请求：GET
    telnet创建socket并调用connect来连接到Web服务器．
    HTTP请求：GET /index.html HTTP/1.0
    三部分分别是：命令，参数，协议版本号

HTTP应答：OK
    服务器读取请求，检查请求，然后返回一个请求．应答包含头部和内容两部分．
    头部：HTTP/1.1 200 OK
    三部分分别是：协议版本号，返回码，文本解释
    内容是具体的网页内容


## Code

socklib.c 包含了建立服务器的封装好的函数
编译命令: cc webserv.c socklib.c -o webserv
输入命令: ./webserv 8888
即可将本地作为服务器，端口是8888 通过浏览器输入 http://ip:8888 即可访问服务

``` c
/*
 * socklib.c
 *
 * This file contains functions used lots when writing internet
 * client/server programs. The two main functions here are:
 *
 * int make_server_socket(portnum) returns a server socket or -1 
 *      if error
 * int make_server_socket_q(portnum, backlog)
 *
 * int connect_to_server(char * hostname, int portnum)
 *      returns a connected socket or -1 if error
 */
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netdb.h>
#include <time.h>
#include <strings.h>

#define HOSTLEN 256
#define BACKLOG 1

int make_server_socket_q(int, int);

int make_server_socket(int portnum)
{
    return make_server_socket_q(portnum, BACKLOG);
}

int make_server_socket_q(int portnum, int backlog)
{
    struct sockaddr_in saddr;
    struct hostent *hp;
    char hostname[HOSTLEN];
    int sock_id;

    sock_id = socket(PF_INET, SOCK_STREAM, 0);
    if (sock_id == -1)
        return -1;

    bzero((void *)&saddr, sizeof(saddr));
    gethostname(hostname, HOSTLEN);
    printf("hostname %s\n", hostname);
    hp = gethostbyname(hostname);
    char str[32];
    char **pptr;
    pptr = hp->h_addr_list;
    // printf("id %s\n", inet_ntop(hp->h_addrtype, *pptr, str, sizeof(str)));
    for (int i = 0; i < hp->h_length; ++i)
    {
        printf("%d",(int)hp->h_addr[i]);
        if (i != hp->h_length-1)
            printf(".");
    }
    printf("\n");



    bcopy((void *)hp->h_addr, (void *)&saddr.sin_addr, hp->h_length);
    saddr.sin_port = htons(portnum);
    saddr.sin_family = AF_INET;
    if (bind(sock_id, (struct sockaddr *)&saddr, sizeof(saddr)) != 0)
        return -1;

    if (listen(sock_id, backlog) != 0)
        return -1;

    return sock_id;
}

int connect_to_server(char *host, int portnum)
{
    int sock;
    struct sockaddr_in servadd;
    struct hostent *hp;

    sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock == -1)
        return -1;

    bzero(&servadd, sizeof(servadd));
    hp = gethostbyname(host);
    if (hp == NULL)
        return -1;
    bcopy(hp->h_addr, (struct sockaddr *)&servadd.sin_addr, hp->h_length);
    servadd.sin_port = htons(portnum);
    servadd.sin_family = AF_INET;

    if (connect(sock, (struct sockaddr *)&servadd, sizeof(servadd)) != 0)
        return -1;

    return sock;
}
```

``` c
/*
 * webserv.c - a minimal web server(version 0.2)
 *
 * usage: ws portnumber
 * features: supports the GET command only
 *      runs in the current directory
 *      forks a new child to handle each request
 *      has MAJOR security holes, for demo purposes only
 *      has many other weaknesses, but is a good start
 * build: cc webserv.c socklib.c -o webserv
 */

#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>

main(int ac, char * av[])
{
    int sock, fd;
    FILE *fpin;
    char request[BUFSIZ];
    if (ac == 1)
    {
        fprintf(stderr, "usage: ws portnum\n");
        exit(1);
    }
    sock = make_server_socket(atoi(av[1]));
    if (sock == -1)
        exit(2);

    // main loop here
    while (1)
    {
        printf("Begin accept\n");
        // take a call and buffer it
        fd = accept(sock, NULL, NULL);
        fpin = fdopen(fd, "r");

        // read request
        fgets(request, BUFSIZ, fpin);
        printf("got a call: request = %s", request);
        read_til_crnl(fpin);

        // do what client asks
        process_rq(request, fd);

        fclose(fpin);
    }
}

/*
 * read_til_crnl(FILE *)
 * skip over all request info until a CRNL is seen
 */
read_til_crnl(FILE *fp)
{
    char buf[BUFSIZ];
    while (fgets(buf, BUFSIZ, fp) != NULL && strcmp(buf, "\r\n") != 0)
        ;
}

/*
 * process_rq(char *rq, int fd)
 * do what the request asks for and write reply to fd
 * handles request in a new process
 * rq is HTTP command: GET /foo/bar.html HTTP/1.0
 */
process_rq(char *rq, int fd)
{
    char cmd[BUFSIZ], arg[BUFSIZ];

    // create a new process and return if not the child
    if (fork() != 0)
        return;

    // precede args with ./*/
    strcpy(arg, "./");
    if (sscanf(rq, "%s%s", cmd, arg+2) != 2)
        return;

    printf("arg %s\n", arg);

    if (strcmp(cmd, "GET") != 0)
        cannot_do(fd);
    else if (not_exist(arg))
        do_404(arg, fd);
    else if (isadir(arg))
        do_ls(arg, fd);
    else if (ends_in_cgi(arg))
        do_exec(arg, fd);
    else
        do_cat(arg, fd);
}

/*
 * the reply header thing: all functions need one 
 * if content_type is NULL then don't send content type
 */
header(FILE *fp, char *content_type)
{
    fprintf(fp, "HTTP/1.0 200 OK\r\n");
    if (content_type)
        fprintf(fp, "Content-type: %s\r\n", content_type);
}

/*
 * simple functions first:
 *  cannot_do(fd) unimplemented HTTP command
 * and do_404(item, fd) no such object
 */
cannot_do(int fd)
{
    FILE *fp = fdopen(fd, "w");

    fprintf(fp, "HTTP/1.0 501 Not Implemented\r\n");
    fprintf(fp, "Content-type: text/plain\r\n");
    fprintf(fp, "\r\n");

    fprintf(fp, "That command is not yet implemented\r\n");
    fclose(fp);
}

do_404(char *item, int fd)
{
    FILE *fp = fdopen(fd, "w");

    fprintf(fp, "HTTP/1.0 404 Not Found\r\n");
    fprintf(fp, "Content-type: text/plain\r\n");
    fprintf(fp, "\r\n");

    fprintf(fp, "The item you requested: %s\r\n is not found\r\n", item);
    fclose(fp);
}

/*
 * the directory listing section
 * isadir() uses stat, not_exist() uses stat
 * do_ls run ls. It should not
 */
isadir(char *f)
{
    struct stat info;
    return (stat(f, &info) != -1 && S_ISDIR(info.st_mode));
}

not_exist(char *f)
{
    struct stat info;
    return (stat(f, &info) == -1);
}

do_ls(char *dir, int fd)
{
    FILE *fp;

    fp = fdopen(fd, "w");
    header(fp, "text/plain");
    fprintf(fp, "\r\n");
    fflush(fp);

    dup2(fd, 1);
    dup2(fd, 2);
    close(fd);
    execlp("ls", "ls", "-l", dir, NULL);
    perror(dir);
    exit(1);
}

/*
 * the cgi stuff. function to check extension and
 * one to run the program
 */
char *file_type(char *f)
{
    char *cp;
    if ((cp = strrchr(f, '.')) != NULL)
        return cp+1;
    return "";
}

ends_in_cgi(char *f)
{
    return (strcmp(file_type(f), "cgi") == 0);
}

do_exec(char *prog, int fd)
{
    FILE *fp;

    fp = fdopen(fd, "w");
    header(fp, NULL);
    fflush(fp);

    dup2(fd, 1);
    dup2(fd, 2);
    close(fd);
    execl(prog, prog, NULL);
    perror(prog);
}

/*
 * do_cat(filename, fd)
 * sends back contents after a header
 */
do_cat(char *f, int fd)
{
    char *extension = file_type(f);
    char *content = "text/plain";
    FILE *fpsock, *fpfile;
    int c;

    if (strcmp(extension, "html") == 0)
        content = "text/html";
    else if (strcmp(extension, "gif") == 0)
        content = "image/gif";
    else if (strcmp(extension, "jpg") == 0)
        content = "image/jpg";
    else if (strcmp(extension, "jpeg") == 0)
        content = "image/jpeg";

    fpsock = fdopen(fd, "w");
    fpfile = fopen(f, "r");
    if (fpsock != NULL && fpfile != NULL)
    {
        header(fpsock, content);
        fprintf(fpsock, "\r\n");
        while ((c = getc(fpfile)) != EOF)
            putc(c, fpsock);
        fclose(fpfile);
        fclose(fpsock);
    }
    exit(0);
}
```
