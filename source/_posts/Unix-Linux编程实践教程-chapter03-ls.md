---
title: Unix-Linux编程实践教程-chapter03-ls
date: 2016-07-23 16:31:16
tags: [Linux,C]
category: [programming]
---

## 第三章　目录与文件属性：编写ls

磁盘上有文件和目录，文件和目录都有目录和属性．文件的内容可以是任意的数据，
目录的内容只能是文件名或者子目录名的属性

目录中的文件名和子目录名指向文件和其他的目录，内核提供了系统调用来读取目录的
内容，读取和修改文件的属性

文件类型，文件的访问权限和特殊属性被编码存储在一个１６位整数中，可以通过
掩码技术来读取这些信息

文件所有者和组信息是以ＩＤ的形式保存的，它们与用户名和组名的联系保存在
passwd和group数据库中


自己编写ls,需要掌握三点：
如何读取目录的内容
如何读取并显示文件的属性
给出一个名字，如何判断是目录还是文件

把多种信息编码到不同的字段是一种常用的技术，如电话号码，ＩＰ字段等

为了比较，把不需要的地方置为０，这种技术称为掩码

将二进制数的每三位分为一组来操作，这就是八进制

结构stat 中的st_mode 成员包含１６位，其中四位用作文件类型，九位用作许可权限，
剩下的三位用作文件特殊属性
set-user-ID s 使用它来给某些程序提供额外的权限，比如系统中的打印队列
set-group-ID s
sticky 它告诉内核，即使没有人使用程序，也要把它放在交换空间中，因为加载速度
比从硬盘空间快
在许可权限部分，用户的ｘ被替换成ｓ，代表set-user-ID 被设置
组用户的ｘ被替换成ｓ，代表set-group-ID被设置
其他用户的ｘ被替换成ｔ，代表sticky被设置

## code

``` c

/* ls2.c
 * purpose list contents of directory or directories
 * action if no args, use . else list files in args
 * note uses stat and pwd.h and grp.h
 * BUG: try ls2 /tmp
 */

#include <stdio.h>
#include <sys/types.h>
#include <dirent.h>
#include <sys/stat.h>

void do_ls(char []);
void dostat(char *);
void show_file_info(char *, struct stat *);
void mode_to_letters(int , char []);
char * uid_to_name(uid_t);
char * gid_to_name(gid_t);

main(int ac, char * av[])
{
    if (ac == 1)
        do_ls(".");
    else
        while (--ac)
        {
            printf("%s:\n", * ++av);
            do_ls(*av);
        }
}

void do_ls(char dirname[])
/* 
 * list files in directory called dirname
 */
{
    DIR *dir_ptr;           // the directory
    struct dirent * direntp;// each entry

    if ((dir_ptr = opendir(dirname)) == NULL)
        fprintf(stderr, "ls1: cannot open %s\n", dirname);
    else
    {
        while ((direntp = readdir(dir_ptr)) != NULL)
            dostat(direntp->d_name);
        closedir(dir_ptr);
    }
}

void dostat(char *filename)
{
    struct stat info;
    if (stat(filename, &info) == -1)        // cannot stat
        perror(filename);                   // say why
    else
        show_file_info(filename, &info);    
}

void show_file_info(char *filename, struct stat * info_p)
/* 
 * display the info about filename.
 * the info is stored in struct at * info_p
 */
{
    char * uid_to_name(), *ctime(), *gid_to_name(), *filemode();
    void mode_to_letters();
    char modestr[11];

    mode_to_letters(info_p->st_mode, modestr);

    printf("%s", modestr);
    printf("%4d ", (int)info_p->st_nlink);
    printf("%-8s ", uid_to_name(info_p->st_uid));
    printf("%-8s ", gid_to_name(info_p->st_gid));
    printf("%8ld ", (long)info_p->st_size);
    printf("%.12s ", 4+ctime(&info_p->st_mtime));
    printf("%s\n", filename);
}

/*
 * utility functions
 */

/*
 * This function takes a mode value and a char array
 * and puts into the char array the file type and the
 * nine letters that correspnd to the bits in mode.
 * NOTE: It does not code setuid, setgid, and sticky
 * codes
 */
void mode_to_letters(int mode, char str[])
{
    strcpy(str, "----------");  // default = no perms
    if (S_ISDIR(mode)) str[0] = 'd';    // directory
    if (S_ISCHR(mode)) str[0] = 'c';    // char devices
    if (S_ISBLK(mode)) str[0] = 'l';    // block device

    if (mode & S_IRUSR) str[1] = 'r';   // 3 bits for user
    if (mode & S_IWUSR) str[2] = 'w';
    if (mode & S_IXUSR) str[3] = 'x';

    if (mode & S_IRGRP) str[4] = 'r';   // 3 bits for group
    if (mode & S_IWGRP) str[5] = 'w';
    if (mode & S_IXGRP) str[6] = 'x';

    if (mode & S_IROTH) str[7] = 'r';   // 3 bits for other
    if (mode & S_IWOTH) str[8] = 'w';
    if (mode & S_IXOTH) str[9] = 'x';
}

#include <pwd.h>

char * uid_to_name(uid_t uid)
/* 
 * returns pointer to username associated with uid, uses getpw()
 */
{
    struct passwd * getpwuid(), *pw_ptr;
    static char numstr[10];

    if ((pw_ptr = getpwuid(uid)) == NULL)
    {
        sprintf(numstr, "%d", uid);
        return numstr;
    }
    else
        return pw_ptr->pw_name;
}

#include <grp.h>

char * gid_to_name(gid_t gid)
/*
 * returns pointer to group number gid, used getgrid(3)
 */
{
    struct group * getgrpid(), *grp_ptr;
    static char numstr[10];

    if ((grp_ptr = getgrgid(gid)) == NULL)
    {
        sprintf(numstr, "%d", gid);
        return numstr;
    }
    else
        return grp_ptr->gr_name;
}
```
