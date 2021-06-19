---
title: linux相关之程序
date: 2021-06-19 20:55:13
tags: linux
---

## 获取进程ID

`$!` 获取上一条命令的进程ID,一般用来监控后台进程,举例:

```sh
# 后台启动redis服务,当脚本退出时通过检测进程ID来杀掉redis进程
./redis-server &
PID=$!
...
PID_EXIST=$(ps aux | awk '{print $2}' | grep -w $PID)
if [ ! $PID_EXIST ];
then
    echo 'No redis server running'
else
    kill -9 ${PID}
fi
```

## /dev/shm 的使用

/dev/shm是系统中内存模拟的目录,因此读写速度很高,超过磁盘

可以用来进行大型代码仓库的编译(如将cmake的编译路径设置到/dev/shm),以及性能瓶颈测试,排除磁盘IO的影响(如本来要输出到磁盘的内容写到/dev/shm)

默认大小是ram的一半,因此要注意不要用多了(可以修改大小),会影响其他进程的可用内存;因为是内存,系统重启之后会被清空

## 查看依赖库

当程序报错找不到依赖库,或需要将程序打包移植到其他机器上时,可以使用 ldd 命令查找依赖库

`ldd exe/xx.so`

查看库/可执行文件的符号信息,可用于排查函数定义冲突,如某些程序对glibc版本的要求与系统安装的版本不同

`nm *.lib`

## 监控程序

查看一个进程的线程数 `ps hH p <pid> |wc -l`

查看当前bash的所有进程 `ps -l`

top命令排序

* P 按CPU使用率排序
* M 按内存占用排序

## 生成.core文件用于debug

* `ulimit -c unlimited` 当发生段错误,设置生成dump文件
* `ulimit -c 0` 取消dump文件
* `ulimit -a` 查看当前设置

## cmake

cmake指定gcc的版本,在shell中设置

```sh
export CC=/usr/local/bin/gcc
export CXX=/usr/local/bin/g++
```

cmake编译软件指定安装路径,多用于非管理员权限编译安装软件  
`cmake -DCMAKE_INSTALL_PREFIX=/usr ..`

Q: 编译安装cmake `./bootstrap --prefix=xx && make && make install` 报错 `The C++ compiler does not support C++11 (e.g.  std::unique_ptr).`  
A: 原因是源代码目录是挂载目录,换个地址如/dev/shm [stackoverflow](https://stackoverflow.com/questions/50764046/c11-stdunique-ptr-error-cmake-3-11-3-bootstrap)

Q: find_package()找不到库  
A: 可以set(name_DIR xxx) 指定包含库的cmake的路径,会去下面查找Findname.cmake文件

清除缓存  
cmake 没有提供clean目录的指令,通常做法是创建 build 目录,直接删除build下所有文件和文件夹来清除缓存 `cmake .. -B build`

## 编译

编译后去可执行文件的当前目录查找动态链接库,需要增加选项 `-Wl,-rpath,'$ORIGIN'`

## 性能检测

valgrind --tool=callgrind --separate-threads=yes ./exproxy  
为每个线程单独生成一个性能分析文件


[linux下利用valgrind工具进行内存泄露检测和性能分析](https://blog.csdn.net/sunmenggmail/article/details/10543483)