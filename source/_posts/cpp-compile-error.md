---
title: c++编译错误汇总
date: 2020-09-05 16:52:51
tags: c++
---

## 编译错误处理

### gcc

Q:error C2059: 语法错误:"\<class-head\>"  
A:全局变量没有加分号,可能是复制粘贴导致的

Q:error: passing ‘const xx' as ‘this’ argument discards qualifiers [-fpermissive]  
A:调用const对象的非const方法报错,需要在方法声明和定义加const限定符  
如`string InetAddress::ip_ntoa() const {}` 好的编程习惯,get类方法返回都加双重const

Q:Error: no such instruction: `shlx %rdx,(%r12),%rax'  
A:shlx是新的intel指令,需要能支持这类新指令的汇编器,即binutils,centos6.x不行,而7.x版本可以支持  
参考链接  
https://blog.csdn.net/superbfly/article/details/59514207  
https://blog.csdn.net/wang_xijue/article/details/47128649

Q:switch语句 jump to case label  
A:作用域问题,不要在case下定义语句或者将每个case语句块用{} 包起来

Q:编译gcc9报错config.log "unrecognized command line option '-V'"  
A:原因是较高版本的gcc不支持-V参数,修改环境变量,设置默认gcc为系统版本4.x,重新编译

Q:g++: unrecognized option '-static-libstdc++'  
A:gcc4.5才引入此选项,所以必须得gcc 4.8了;而centos 6.9默认的是4.4,所以只好换centos7.x来搞,默认4.8.5;最终使用的有效指令 `../configure --disable-checking --enable-languages=c,c++ --disable-multilib --prefix=/path/to/software/gcc9 --enable-threads=posix`

Q:gcc9.1编译测试报错 /usr/bin/ld: unrecognized option '-plugin'  
A:原因是binutils库太旧了(负责ld链接),升级binutils

Q:gcc9编译cpp报错 test.cpp:(.text+0xa): undefined reference to `std::cout'  
A:换成g++ 或者gcc -lstdc++

Q:list-initializer for non-class type must not be parenthesized  
A:发生在结构体构造函数对成员变量数组进行 ({0}) 初始化,改成 {} 会按照0来初始化

Q:Error: invalid operands of types ‘const char [35]’ and ‘const char [2]’ to binary ‘operator+’  
A:不能直接对 const char* 相加,使用string将最左侧的 char* 转换为string即可

### cmake

Q:Clock skew detected.  Your build may be incomplete  
A:make报错,make clean & make

## 编译警告处理

### [-Wreorder]

规则:构造函数时,初始化成员变量顺序要与类声明中顺序对应

### warning: backslash and newline separated by space

\ 连接字符串,\后面多了空格

### [-Wunused-parameter]

有些变量声明了但暂时未使用

可以注释掉;如果要保留,使用C++17语法 `[[maybe_unused]] int a;`

部分情况遇到  ‘mayebe_unused’ attribute directive ignored [-Wattributes]

### [-Wsign-compare]

两种不同类型比较,主要是有符号无符号

解决方法比较多:

* 手动修改某一个类型
* decltype. 如 `decltype(s.size()) len;`

### [-Wnarrowing]

类型转换,降级,如从int到short