---
title: linux进行c++开发经验总结
date: 2020-04-12 22:37:45
tags: linux c++
---

这一周主要就是在linux下进行c++的开发,以此为契机记录下遇到的问题.

## 版本管理
使用**git**管理源代码

常用命令包括:*clone pull push commit checkout branch tag log* 等

### 拉取代码报错
git 1.7版本拉去代码报错:  
*error: The requested URL returned error: 401 Unauthorized while accessing*  
解决方案:升级最新版本git  
有时候拉取代码不成功,可以ssh/https两种链接都试试

## 代码编写
vim进行临时的一些修改,vscode用于较大的项目,VS Studio用于windows下的调试  

目前主要使用**vscode**,开发环境是无界面的linux系统,使用最新版本的vscode有连远程代码仓库的功能,可以在本地windows进行远程代码修改

## 编译
简单的工程可以一条gcc命令进行编译,较大的项目还是使用cmake更好一些  

使用**cmake**编译,首先编写CMakeLists.txt,然后编写脚本配置环境变量如include和library路径,再运行cmake和make命令即可完成编译

### 查错
VERBOSE模式,输出具体的gcc编译命令,方便查错,通过`make VERBOSE=1` 选项来开启模式

### 配置
可以通过在CMakeLists.txt中添加预定义宏  
*add_definitions(-DAABB=1)* 来设置宏AABB值为1,或者*add_definitions(-DDEBUG)* 来打开DEBUG宏

### 编译慢问题
遇到cmake编译慢问题,通过top命令及ps命令查到自己的进程状态为*D*,查阅手册D含义是进程处于睡眠状态,也就是进程由于等待IO如磁盘IO,网络IO等,导致较长时间都没有响应  
判断磁盘IO慢的问题,因此修改编译脚本,将编译的中间结果文件输出到临时的内存空间shm中去,编译后再删除临时文件,减少本地磁盘IO操作,从而加速编译过程

## 运行
可以直接本地运行,方便查看占用内存和CPU资源情况,也可以使用公司集群系统投递任务,好处是统一的任务管理调度,不会出现资源竞争情况导致程序运行时间波动

### 库版本不对
*/lib64/libc.so.6: version `GLIBC_2.14' not found (required by xxx)*  
这种情况是本地的libc库版本太旧,需要更新libc库版本

### 查看log
一般程序会输出log到磁盘文件,想要实时监控日志文件的更新内容,可以使用`tail -f filename`命令,它会在文件内容有更新时将结果输出到命令窗口

## 调试
使用**gdb**调试C++程序  
1. 编译时加 *-g -gstabs+* 选项,并且去除 *-O2* 等优化选项
2. 两种调试方式
   1. 直接`gdb ./prog` 进入gdb交互环境,通过命令`set args xxx`来设置参数,然后`r`来运行
   2. 通过设置,使程序挂掉时生成core文件,通过`gdb ./prog core.xxxx`来还原程序挂掉前的状态

gdb常用快捷键:
* bt 查看堆栈
* l 查看当前所处位置的源代码
* b 打断的,如`b filename::linenum` 打断点到文件的某一行,也可以直接打到某函数位置
* n 下一步
* c 继续运行,直到程序结束或者遇到断点
* s 单步调试
* r 重头运行程序
* p 打印变量内容
* help 查看命令提示

## 性能分析
**gprof**工具  
linux上分析gcc编译出来的程序的CPU时间,找出最耗时的函数  

使用:
1. `gcc -pg` 选项编译
2. 运行程序,结束后生成gmon.out
3. `gprof ./prog gmon.out -b` 查看输出 
    
原理: 在每个函数中插入count函数,这样函数调用时就会计算次数和时间  
缺点: 无法分析多线程程序;无法观察IO时间

**valgrind**工具  
可以使用它的*Memcheck* 功能来进行内存检查,或者*Callgrind* 进行耗时和函数调用情况分析

使用:
1. `valgrind --tool=callgrind ./prog_name` 运行完会生成callgrind.out.xxx的文件
2. *kcachegrind.exe* 打开上一步生成的文件,可以看到函数运行耗时,以及调用的流程图


知道哪个函数或者哪个操作最耗时,再进一步分析是数据结构选型不适合还是算法没有达到最优,再进行速度提升