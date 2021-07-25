---
title: doctest单元测试库
date: 2021-07-25 16:41:39
tags: unittext cpp
---

doctest是一个C++单元测试库,代码仓库地址: https://github.com/onqtam/doctest, 本文内容主要来自官方文档与项目实践

## 特性

* header-only
* fast
* thread-safe

## 断言宏

### 基本

三个level,严重程度依次降低:

* REQUIRE. 断言失败立即退出用例
* CHECK. 断言失败继续执行用例
* WARN. 打印信息,而不标记失败

例子: `CHECK(flags == state::alive | state::moving);`  

其他的宏都可以认为是三个基本宏扩展来的,以下的 LEVEL 可替换为 REQUIRE/CHECK/WARN 中的任一个

否定断言:LEVEL_FALSE(exp)

输出信息:LEVEL_MESSAGE(exp, message)  
例子: `CHECK_MESSAGE(a < b, "relevant only to this assert " << other_local << "more text!");`

### 二元与一元

二元断言,接受两个参数(left, right):

* LEVEL_EQ ==
* LEVEL_NE !+
* LEVEL_GT >
* LEVEL_LT <
* LEVEL_GE >=
* LEVEL_LE <=

一元断言,接受一个表达式:

* LEVEL_UNARY 等价于 LEVEL
* LEVEL_UNARY_FALSE 等价于 LEVEL_FALSE

### 异常

* LEVEL_THROWS(exp) 期待表达式抛出异常
* LEVEL_THROWS_AS(exp, exp_type) 指定异常类型
    `CHECK_THROWS_AS(func(), std::exception);`
* LEVEL_THROWS_WITH(exp, c_str) 异常信息可以转换为字符串
    `CHECK_THROWS_WITH(func(), "invalid operation!");`
* LEVEL_THROWS_WITH_AS(exp, c_str, exp_type) 异常信息与异常类型的结合
* LEVEL_NOTHROW(exp) 期待无异常抛出

每个异常断言都有_MESSAGE变种

### 浮点数比较

对浮点数采用容错比较

`REQUIRE(22.0/7 == doctest::Approx(3.141).epsilon(0.01)); // allow for a 1% error`

## TestCase

使用的宏

* TEST_CASE(test name)
* SUBCASE(subcase name)

subcases不适用多线程,只能运行在主线程

对标 setup/teardown  
同一level的subcase会顺序执行,且每次执行都会重复执行上层语句

### BDD-style

Behaviour Driven Development(行为驱动开发)

这里提供的接口与TEST_CASE/SUBCASE作用一样,通过testcase name来区分

* SCENARIO(scenario name) 情景,与TEST_CASE作用一致
* SCENARIO_TEMPLATE(scenario name, type, list of types) 等价于TEST_CASE_TEMPLATE
* SCENARIO_TEMPLATE_DEFINE(scenario name, type ,id) 等价于TEST_CASE_TEMPLATE_DEFINE
* GIVEN(sth)
* WHEN(sth)
* THEN(sth)
* AND_WHEN(sth)
* AND_THEN(sth)
  
### Fixture

传统方式,可使用subcases 来替代

TEST_CASE_FIXTURE(class name, c_str)

先定义Fixture class,每一个test case会派生出唯一的类,可自由使用类成员与方法

### Test Suites

将test cases组合到一起就形成了test suites

使用TEST_SUITE() 或者TEST_SUITE_BEGIN()/TEST_SUITE_END()

```cpp
TEST_SUITE("math") {
    TEST_CASE("") {} // part of the math test suite
    TEST_CASE("") {} // part of the math test suite
}

TEST_SUITE_BEGIN("utils");
TEST_CASE("") {}
TEST_SUITE_END();
```

方便根据执行条件过滤

### Decorators

装饰额外属性  
可以用于TEST_CASE/TEST_SUITE

```cpp
TEST_CASE("name"
          * doctest::description("shouldn't take more than 500ms")
          * doctest::timeout(0.5)) {
    // asserts
}
```

有用的装饰:

* skip(bool = true) 跳过这个测试用例
* may_fail(bool = true) 用于已知bug情况下记录追踪
* timeout(double) 设置超时
* description("text") 描述

## 参数化测试用例

此功能尚不完善

TEST_CASE_TEMPLATE 用于测试相同接口的不同实现是否满足一些共同的需求

```cpp
TEST_CASE_TEMPLATE("signed integers stuff", T, char, short, int, long long int) {
    T var = T();
    --var;
    CHECK(var == -1);
}
```

## 命令行

可以用来过滤测试用例,使用通配符

* 查询.
  * -c 打印所有测试用例的个数
  * -ltc 打印所有测试用例
  * -lts 打印所有测试套件
* 过滤. 灵活使用 */? 分别匹配多个和一个字符
  * -tc= 过滤测试用例
  * -ts= 过滤测试套件
  * -sc= 过滤子用例
  * -tce= 排除某些测试用例
* 其他.
  * -o=filename 输出到文件,默认是到屏幕
  * -d=<bool> 打印每个测试用例的耗时,单位秒
  * -ns=<bool> 虽然代码标注了某些测试用例是跳过的,但是这里可以强制执行

## main

一般可以通过在main.cpp中添加并只添加如下两行,来让doctest帮我们实现main函数

```cpp
#define DOCTEST_CONFIG_IMPLEMENT_WITH_MAIN
#include "doctest.h"
```

也可以定义DOCTEST_CONFIG_IMPLEMENT并自定义main函数,用来配置属性,将测试框架集成进生产代码中等

## 打印

### INFO

只有assert fails才打印信息,且需要把打印语句放到assert之前

`INFO("The number is ", i);` 旧版本用 `INFO("number " << 1);`

`CAPTURE(some_variable)` 封装INFO, 打印变量名称和值,形式如`some_variable := 42`

### Message

MESSAGE(message) 只打印信息,无关assert

FAIL(message) 人为制造失败的测试用例,结果如同REQUIRE失败,退出当前测试用例

FAIL_CHECK(message) 人为制造失败的测试用例,结果如同CHECK失败,继续执行当前测试用例

## 配置

提供了许多配置宏

宏的定义必须早于头文件的引入;一次定义即可

选出几个比较重要的:

* DOCTEST_CONFIG_IMPLEMENT_WITH_MAIN 自动生成main函数,要定义在源文件
* DOCTEST_CONFIG_IMPLEMENT 用于自定义main
* DOCTEST_CONFIG_DISABLE 从binary中移除单元测试,其独特性体现在可以在任何位置写测试代码而不用担心发布版本的性能
* DOCTEST_CONFIG_SUPER_FAST_ASSERTS 加速断言

## 技巧

配置`DOCTEST_CONFIG_SUPER_FAST_ASSERTS` 加速断言的汇编过程

使用二元和一元断言速度比普通断言快

## 其他测试相关

Mock测试:主要作用是模拟一些在应用中不容易构造或者比较复杂的对象,从而把测试与测试边界以外的对象隔离开

## 常用示例

总结自examples

```cpp
// 断言
REQUIRE(a == b); //REQUIURE失败了会结束当前测试用例;比较整型,字符串,或自定义类型
CHECK(doctest::Approx(0.502) == 0.501); //CHECK失败了继续执行;浮点数比较特殊处理

// Test case
// 接受一个字符串来描述当前用例
TEST_CASE("expressions should be evaluated only once") {
    int a = 5;
    REQUIRE(++a == 6);
}

// 异常
LEVEL_THROWS(exp); // 期待表达式抛出异常
CHECK_THROWS_AS(func(), std::exception); // 指定异常类型
CHECK_THROWS_WITH(func(), "invalid operation!"); // 异常信息可以转换为字符串
LEVEL_THROWS_WITH_AS(exp, c_str, exp_type); // 前两者的结合
LEVEL_NOTHROW(exp); // 无异常抛出

// 打印信息,提示测试了某某方法
MESSAGE("reached!");
INFO("lots of captures - some on heap: " << some_var);

// 子用例
TEST_CASE("test case should fail even though the last subcase passes") {
    SUBCASE("one") {
        CHECK(false);
    }
    SUBCASE("two") {
        CHECK(true);
    }
}

// 情景测试
SCENARIO("vectors can be sized and resized") {
    GIVEN("A vector with some items") {
        std::vector<int> v(5);

        REQUIRE(v.size() == 5);
        REQUIRE(v.capacity() >= 5);

        WHEN("the size is increased") {
            v.resize(10);

            THEN("the size and capacity change") {
                CHECK(v.size() == 20);
                CHECK(v.capacity() >= 10);
            }
        }
        WHEN("the size is reduced") {
            v.resize(0);

            THEN("the size changes but not capacity") {
                CHECK(v.size() == 0);
                CHECK(v.capacity() >= 5);
            }
        }
    }
}

// 使用模板
// 一次测试多种类型
TEST_CASE_TEMPLATE("signed integers stuff", T, signed char, short, int) {
    T var = T();
    --var;
    CHECK(var == -1);
}

// 测试套件
// 格式1
TEST_SUITE("scoped test suite") {
    TEST_CASE("part of scoped") {
        FAIL("");
    }

    TEST_CASE("part of scoped 2") {
        FAIL("");
    }
}
// 格式2
TEST_SUITE_BEGIN("some TS"); // begin "some TS"
TEST_CASE("part of some TS") {
    FAIL("");
}
TEST_SUITE_END(); // ends "some TS"

// 脚手架
// 构造和析构代替传统的setup()和teardown()
struct SomeFixture
{
    int data;
    SomeFixture() noexcept
            : data(42) {
        // setup here
    }
    ~SomeFixture() {
        // teardown here
    }
};
TEST_CASE_FIXTURE(SomeFixture, "fixtured test - not part of a test suite") {
    data /= 2;
    CHECK(data == 85);
}
```
