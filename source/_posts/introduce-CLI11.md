---
title: C++命令行解析库CLI11介绍
date: 2020-03-04 20:11:12
tags: C++ CLI11 命令行解析
---

本篇文章主要提炼自github上CLI11的官方文档,取出自己感兴趣的内容,记录下来方便以后使用


## 简单介绍

CLI11是一个基于C++开发的命令行解析库,目前最新版本1.9

其优点:
* 使用很方便,只需要`#include <CLI11.hpp>`,当然也可以使用cmake编译版本
* 跨平台,支持广泛(不需要C++11以上的版本支持)
* 支持subcommand;支持重复options
  
关于编译
> `g++ -std=c++11 xx.cpp -I path_with_CLI11 -o app`

(path_with_CLI11是一个路径,其内有CLI11.hpp, app是编译后的可执行程序名)

运行:

需要提示信息的时候运行`./app -h`

### 一个简单的例子
```cpp
#include "CLI11.hpp"
#include <iostream>

int main(int argc, char **argv) {
    CLI::App app{"App description"};

    // Define options
    int p = 0;
    app.add_option("-p", p, "Parameter");

    CLI11_PARSE(app, argc, argv);

    std::cout << "Parameter value: " << p << std::endl;
    return 0;
}
```
只接受一个可选参数-p

CLI::App 是与库的所有交互的基础

CLI11_PARSE 宏内部执行app.parse(argc,argv)对命令行参数解析,出错时抛出ParseError,然后捕获异常,打印错误信息并退出程序

## 主要功能

### 位置参数

即必须参数,使用方法是*add_xxx*方法的第一个参数如"-a" **把"-" 去掉**,换成有意义的名字,如"outputDir"
位置参数就是没有这些参数就无法运行,没有默认值;多个位置参数按定义顺序传递

### flags

命令行输入只填flag名字就行,不接受参数;函数为*add_flag*,有以下三种类型:
* boolean flags
    ```cpp
    bool my_flag;
    app.add_flag("-f", my_flag, "Optional description");
    ```
    绑定flag -f 到布尔变量my_flag,默认是flase,如果用户输入了-f 则为true
* integer flags
    ```cpp
    int my_flag;
    app.add_flag("-f", my_flag, "Optional description");
    ```
    同样的绑定,范围变成整数
* pure flags
    ```cpp
    CLI::Option* my_flag = app.add_flag("-f", "Optional description");
    ```
    使用my_flag->count()来确定值是true/false,默认为0则false,大于等于1次则true;也可以bool(*my_flag)来使用

    所有add_* 格式的函数都返回CLI::Option类型,因此可以**链式调用**,使代码更简约
* 其他 
    有callback flags,add_flag_function()可以打印每个参数输入了多少次

每个add_*方法的第一个参数,即指定的命令参数,可以有多个名字,逗号分隔即可,如"-a,--alpha,-b";另外一个比较有用的是->ignore_case() 方法,忽略大小写,方便用户输入

多个参数可以组合如"-i -a -b"等价于"-iab"

### options
与flags区别就是**接受参数**;函数为*add_option*

基本形式:
```cpp
int int_option = 0;
app.add_option("-i", int_option, "Optional description");
```
其行为:绑定选项-i到int_option,解析其后的数据转换为整型,类型不对会失败;如果没有此选项则使用初始值

可接受类型包括:整型/浮点/字符串/vector/函数

### vectors of options

接受多个值,直到下一个值不合法;也可以用->expected(N)指定需要几个值

如果出现重复option,会进行组合,即"-v 1 2 -v 3 4"等同于"-v 1 2 3 4"(新版本才支持此功能)

### 修改option属性

链式使用,当作装饰器,可以同时添加多个装饰

列举几个可能会常用到的:
* ->required() 必须指定
* ->expected(N) 参数个数
* ->check(type)
   * CLI::ExistingFile 检查文件是否存在
   * CLI::ExistingDirectory 目录是否存在
   * CLI::NonexistentPath 需要目录不存在
   * CLI::Range(min,max) 指定范围
	
### 特殊选项:
* *sets* 
    使用集合来限定输入范围;如果输入不在集合范围内,会打印提示信息
    ```cpp
    int val;
    app.add_set("--even", val, {0,2,4,6,8});
    ```
* *complex number* 复数,可接受1-2个参数,不给默认是0
    ```cpp
    std::complex<float> val;
    app.add_complex("--cplx", val);
    ```
* *optional* 可选参数
* *windows风格* /a /f ...

### Validators 验证器

验证器有两种形式
* transform 变异? 接受string,返回修改过的string
* check 非变异? 
	* 接受const string,返回修改过的string
    * struct CLI::Validator的子类
常用check来检查路径/文件是否存在,以及输入是否在一个range内

## subcommand 子命令

子命令就是包含了一系列选项的一个关键字,如git commit/clone 这里面的commit clone后面还可以跟各种选项,他们就是git程序的子命令

子命令的类类型和App相同,因此可以任意嵌套

关于App的功能
* 使用它来创建options
* 设置页脚,在-h下面展示,比如显示下个性签名 app.footer("xx")
* 设置帮助信息

### 添加子命令

`CLI::App* sub = app.add_subcommand("sub", "This is a subcommand");`

第一个参数就是子命令的名字,第二个参数是描述

### 检查子命令是否被使用
* if(*sub) ...
* if(sub->parsed()) ...
* if(app.got_subcommand(sub)) ...
* if(app.got_subcommand("sub")) ...

设置必须的子命令个数,只传一个参数则限定了个数
app.require_subcommand(/* min */ 0, /* max */ 1);

### 特殊模式

* allow_extras() 允许出现多余的option而不报错,多余的值保存到.remaining()
* fallthrough 将subcommand未匹配的option转给parnet command解析(默认不会fallthrough)
* prefix command 遇到未知option时停止解析,即使其他未知选项可以匹配,也将被忽略

## 实例

编写个实例,把subcommand flag 各种option,check等常用功能都演示一遍


代码:
```cpp
//把CLI11.hpp放到当前目录下
#include "CLI11.hpp"
#include <iostream>
using namespace std;

int main(int argc, char **argv) {
    CLI::App app{"App description"}; // 软件描述出现在第一行打印
    app.footer("My footer"); // 最后一行打印
    app.get_formatter()->column_width(40); // 列的宽度
    app.require_subcommand(1); // 表示运行命令需要且仅需要一个子命令

    auto sub1 = app.add_subcommand("sub1", "subcommand1");
    auto sub2 = app.add_subcommand("sub2", "subcommand1");
    sub1->fallthrough(); // 当出现的参数子命令解析不了时,返回上一级尝试解析
    sub2->fallthrough();

    // 定义需要用到的参数
    string filename;
    int threads = 10;
    int mode = 0;
    vector<int> barcodes;
    bool reverse = false;
    string outPath;
    if (sub1)
    {
        // 第一个参数不加-, 表示位置参数,位置参数按出现的顺序来解析
        // 这里还检查了文件是否存在,已经是必须参数
        sub1->add_option("file", filename, "Position paramter")->check(CLI::ExistingFile)->required();

        // 检查参数必须大于0
        sub1->add_option("-n,-N", threads, "Set thread number")->check(CLI::PositiveNumber);
    }
    if (sub2)
    {
        // 设置范围
        sub2->add_option("-e,-E", mode, "Set mode")->check(CLI::Range(0,3));
        // 将数据放到vector中,并限制可接受的长度
        sub2->add_option("-b", barcodes, "Barcodes info:start,len,mismatch")->expected(3,6);
    }
    // 添加flag,有就是true
    app.add_flag("-r,-R", reverse, "Apply reverse");
    // 检查目录是否存在
    app.add_option("-o", outPath, "Output path")->check(CLI::ExistingDirectory);

    CLI11_PARSE(app, argc, argv);

    // 判断哪个子命令被使用
    if (sub1->parsed())
    {
        cout<<"Got sub1"<<endl;
        cout<<"filename:"<<filename<<endl;
        cout<<"threads:"<<threads<<endl;
    }
    else if (sub2->parsed())
    {
        cout<<"Got sub2"<<endl;
        cout<<"mode:"<<mode<<endl;
        cout<<"barcodes:";
        for (auto& b : barcodes)
            cout<<b<<" ";
        cout<<endl;
    }
    cout<<endl<<"Comman paras"<<endl;
    cout<<"reverse:"<<reverse<<endl;
    cout<<"outPath:"<<outPath<<endl;

    return 0;
}
```

编译:
`g++ -std=c++11  run.cpp -o myapp`

用的gcc4.8

运行:

-h 查看提示

![](/images/introduce_CLI11/prompt2.png)

![](/images/introduce_CLI11/prompt1.png)

![](/images/introduce_CLI11/prompt3.png)

给正确的参数

![](/images/introduce_CLI11/right1.png)

![](/images/introduce_CLI11/right2.png)


给错误参数

![](/images/introduce_CLI11/error3.png)

![](/images/introduce_CLI11/error4.png)

![](/images/introduce_CLI11/error1.png)

![](/images/introduce_CLI11/error2.png)

![](/images/introduce_CLI11/error5.png)

## 其他

### 配置文件
允许读写配置文件

### 格式化帮助信息

允许定制自己的帮助打印信息 
app.get_formatter() 获取当前格式
* column_width(width) 设置列的宽度
* lable(key, value) 将lable设置一个不同的值
* 例子
    ```cpp
    app.get_formatter()->column_width(40);
    app.get_formatter()->label("REQUIRED", "(MUST HAVE)");
    ```
		
### subclassing 

部分的替换格式

## 高级主题

### 环境变量

作用是,如果命令行参数没有给定,则从环境变量中获取,如果存在的话
```cpp
std::string opt;
app.add_option("--my_option", opt)->envname("MY_OPTION");
```
### option之间的依赖/互斥关系

a->nees(b) a依赖b

a->excludes(c) a与c不共存

### 自定义

custom option callbacks

custom converters
