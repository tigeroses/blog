---
title: python-argparse
date: 2015-06-02 10:59:26
tags: [python]
category: [programming]
---

在python程序中，第一步就是获取参数，然后程序才能执行。对于简单的程序脚本，可以直接使用sys.argv[] 来获取命令行参数，但是应用到大的软件项目中，我们需要更加规范，更加方便而功能强大工具来处理命令行参数，本文主要介绍python标准库argparse的简单使用，详细方法及示例请参考python标准库

## python获取命令行参数

### 获取参数 sys.argv

sys.argv[0] 为程序名称，其后分别为参数，len(sys.argv)可得出所有参数个数
python标准库中getopt, optparse, argparse都是专门处理命令行参数的模块
getopt 是类似UNIX系统getopt这个C函数的实现，可以处理长短配置项和参数。缺点有两个，一是长短配置项需要分开处理，二是对非法参数和必填参数的处理需要手动
optparse 比getopt 更加方便，强劲，采用声明式风格，还可以自动生成帮助信息
argparse 继承了optparse 的声明式风格的优点，又多了更丰富的功能，所以是现阶段最好用的参数处理标准库
docopt 是比前者更先进更易用的命令行参数处理器，甚至不用写代码，只要编写类似argparse 输出的帮助信息即可，因为其还不是标准库，所以现在主要学习argparse

## argparse

argparse 解析命令行选项，参数以及子命令
argparse 可以帮助更方便的写出用户友好的命令行接口。程序定义它需要什么参数，argparse 解决如何解析这些来自sys.argv 的参数
argparse 同样自动生成帮助和使用说明信息并且当使用者给出错误参数时分发错误

``` python
#引入模块
import argparse

#构建ArgumentParser对象，用来保存解析命令行所得的信息
parser = argparse.ArgumentParser(description='Process some intergers.')

#调用add_argument() 告诉ArgumentParser对象如何处理命令行参数
parser.add_argument('intergers', metavar='N', type=int, nargs='+', help='an interger for the accumulator')

#调用parse_args() 来解析参数
args = parser.parse_args()
```

### ArgumentParser

参数简介：
description 给出一个简短的描述关于程序的使用说明，它出现在usage和帮助信息中间
epilog 在最后给出一个文件描述
add_help 是否加入-h –help选项，默认为True
prefix_chars 命令行选项的前缀，默认为’-‘
fromfile_prefix_chars 从文件中获取参数信息
argument_default 设置参数的全局默认值
parents 包含进其他ArgumentParser对象的参数设置
conflict_handler 定义解决冲突选项的策略
formatter_class 自定义帮助输出的类，控制输出的格式
prog 程序的名字，默认为sys.argv[0]
usage 描述程序使用说明

### add_argument()

参数简介：
name or flags 选项名字，可选参数以’-‘开始
action 遇到此名字的选项的动作
store 存储参数的值，默认即为此
store_const 存储为常量值
store_true(false) 存储布尔值
append 存入List
append_const 存入List，且其值为常量
version 版本信息
nargs 参数的不同数量
N 整数，参数的个数
? 匹配单个值
* 多个值，放入list
+ 类似* ,但是至少要有一个值，否则报错 (这三个用处有点像正则。。)
    const 常量值
    default 默认值
    type 命令行参数应被转换的类型
    int
    float
    complex
    file
    可调用对象，包括函数等
    choices 参数容许的值的容器,如果输入的参数不在此容器之内，报错
    required 此选项是否必须，如果未输入，会报错提示。因为是可选参数，而又必须提供参数，自相矛盾，应避免使用
    help 对此参数的简短描述
    metavar 此参数在usage信息中的名字，实际名字未变，仍为dest 所定义
    dest 经过parse_args() 解析后返回的名字，如不指定名字，则使用– 或者- 之后的名字

### parse_args()

默认参数来自sys.argv
返回一个包含解析后的参数的namespace

### 其他功能

子命令 即命令之下包含又一层命令 如：git add git checkout git push等
fileType对象
argument groups 参数分组
mutual exclusion
parser defaults
partial parsing parse_known_args()返回一个包含两个元素的元组，第一个是包含可选参数的namespace, 第二个是包含剩下的参数的list


## 代码示例

``` python
### prog.py

import argparse

parser = argparse.ArgumentParser(description='An example about argparse')
parser.add_argument("-n", "--name", action="store", dest="name", default="hero", help="Get your name. [%(default)s]")
parser.add_argument("-a", "--age", action="store", dest="age", default="18", type=int, help="Get your age. [%(default)s]")

(para, args) = parser.parse_known_args()

print 'Hello, %s! You are %d years old. Welcome to my world!' % (para.name, para.age)

if len(args) != 0:
    print 'Other parameters are ' + ' '.join(args)
```

## 参考文献

python library reference
编写高质量代码:改善Python程序的91个建议
