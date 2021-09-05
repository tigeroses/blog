---
title: 编写单元测试
date: 2021-09-05 15:14:51
tags: unittest
---

单元测试用来对一个模块,一个函数或一个类进行正确性检验

以测试为驱动的开发模式最大好处是确保一个程序模块的行为符合我们设计的测试用例;在将来修改的时候,可以极大程度地保证该模块行为依然是正确的

单元测试可以提高代码质量

单元测试可以使我们放心重构

## 规则

单元测试的测试用例要覆盖常用的输入组合,边界条件和异常

单元测试代码要简单

单元测试通过不代表没有bug

## 金字塔模型

测试是需要分层的,从下到上可分为单元/服务/UI

* 编写不同粒度的测试
* 层次越高,编写的测试应该越少

## 代码覆盖率

代码覆盖率高,表示bug概率低

### 类型

* 函数覆盖率. 多少比例函数经过了测试
* 语句覆盖率. 多少比例语句经过了测试
* 分支覆盖率. 多少比例的分支经过了测试,如`if(flag) a else b`
* 条件覆盖率. 多少比例的可能性经过了测试,如`if (a & b)`

### 工具

* gcov. 好处是gcc自带,缺点是结果是文本形式
* lcov. 需要安装`yum install -y lcov`,结果可转换为html
  1. 增加编译选项. `–fprofile-arcs –ftest-coverage`,编译后生成gcda文件;或者 `set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} --coverage")`
  2. 运行lcov生成覆盖率统计文件. `lcov -d src_dir -t 'test' -o 'test.info' -b . -c`
     1. -d 待测试的源码目录
     2. -t 目标名称
     3. -o 生成的覆盖率文件
     4. -b 相对目录的起始位置
     5. -c 采集覆盖率
  3. 运行程序. 在源码目录会生成gcno文件
  4. 生成报告. `genhtml -o result test.info`

注意: gcc9版本必须使用lcov1.5版本,低版本会报错:

geninfo: WARNING: xxx/main.gcno: Overlong record at end of file!
geninfo: WARNING: cannot find an entry for main.cpp.gcov in .gcno file, skipping file!

脚本示例:

```sh
$ lcov --rc lcov_branch_coverage=1 -d . -c -o info_tmp
$ lcov --rc lcov_branch_coverage=1 -e info_tmp "*libx/include*" -o info
$ genhtml --rc lcov_branch_coverage=1 -o libx_coverage info
```

## 参考

* https://blog.csdn.net/hs_err_log/article/details/78024739
* https://blog.csdn.net/u011724566/article/details/79160345
* https://www.cnblogs.com/zhaoxd07/p/5608177.html
* [C++语言的单元测试与代码覆盖率](https://paul.pub/gtest-and-coverage/#id-gcov)
