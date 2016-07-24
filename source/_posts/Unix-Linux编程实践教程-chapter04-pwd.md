---
title: Unix-Linux编程实践教程-chapter04-pwd
date: 2016-07-23 16:31:31
tags: Linux C
---

## 第四章　文件系统：编写pwd

Unix将存储在磁盘中的数据组织成文件系统．文件系统是文件和
目录的组合，目录是名字和指针的列表．目录中的每一个入口指向
一个文件或目录．目录包含指向父目录和子目录的入口

Unix文件系统包含三个主要部份：超级块，ｉ－节点和数据区域．
文件内容存储在数据块．文件属性存储在ｉ－节点．表中ｉ－节点
的位置称为文件的ｉ－节点号，ｉ－节点号是文件的唯一标识

相同的i-节点号可能以不同的名字出现在若干个目录中．每个入口
被称为指向文件的硬链接．符号链接是通过文件名引用文件，而不是
i-节点号

若干个文件系统的目录树可被整合成一棵树．内核将一个文件系统的
目录链接到另一个文件系统的根的操作称为装载

Unix包含若干种系统调用，允许程序员进行创建和删除目录，复制指针
删除指针，改变链接和分离其他文件系统等的操作

目录与文件操作相关的系统调用:
创建目录　　mkdir
删除目录    rmdir
删除文件　　unlink
创建文件链接　 link
改变文件或目录的名字和位置　　rename
改变进程的当前目录　　chdir

## code

``` c

/* spwd.c: a simplified version of pwd
 *
 * starts in current directory and recursively
 * climbs up to root of filesystem, prints top part
 * then prints current part
 *
 * uses readdir() to get info about each thing
 *
 * bug: prints an empty string if run from "/"
 */

#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <dirent.h>

ino_t get_inode(char *);
void printpathto(ino_t);
void inum_to_name(ino_t, char *, int);

int ROOT_FLAG = 0;

int main()
{
    printpathto(get_inode("."));
    putchar('\n');
    // printf("\n");
    return 0;
}

void printpathto(ino_t this_inode)
/*
 * prints path leading down to an object with this inode
 * kindof recursive
 */
{
    ino_t my_node;
    char its_name[BUFSIZ];
    if (get_inode("..") != this_inode)
    {
        ROOT_FLAG = 1;
        chdir("..");
        inum_to_name(this_inode, its_name, BUFSIZ);
        my_node = get_inode(".");
        printpathto(my_node);
        printf("/%s", its_name);
    }
    if (ROOT_FLAG == 0)
        printf("/%s", its_name);

}

void inum_to_name(ino_t inode_to_find, char * namebuf, int buflen)
/*
 * looks through current directory for a file with this inode
 * number and copies its name int namebuf
 */
{
    DIR * dir_ptr;
    struct dirent * direntp;
    dir_ptr = opendir(".");
    if(dir_ptr == NULL)
    {
        perror(".");
        exit(1);
    }
    /*
     * search directory for a file with specified inum
     */
    while((direntp = readdir(dir_ptr)) != NULL)
        if (direntp->d_ino == inode_to_find)
        {
            strncpy(namebuf, direntp->d_name, buflen);
            namebuf[buflen-1] = '\0';
            closedir(dir_ptr);
            return;
        }
    fprintf(stderr, "error looking for inum %d\n", inode_to_find);
    exit(1);
}

ino_t get_inode(char * fname)
/*
 * returns inode number of the file
 */
{
    struct stat info;
    if (stat(fname, &info) == -1)
    {
        fprintf(stderr, "Cannot stat");
        perror(fname);
        exit(1);
    }
    return info.st_ino;
}
```
