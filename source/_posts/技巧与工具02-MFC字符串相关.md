---
title: 技巧与工具02-MFC字符串相关
date: 2016-10-24 20:22:47
tags: [MFC,CString]
category: [tools]
---

此篇主要总结了Windows下MFC编程字符串相关的一些知识，如CString, CStringList等的使用．
<!--more-->
纯粹为了自己平时查找方便;相关内容均来自网络,链接附在文末．侵删．

## CString
MFC下最好用的字符串类应该就是CString了．CString是MFC中的一个类，包含了许多好用的操作如
格式化，查找，计算长度等．

要使用CString,需要在工程引用头文件：`#include <afx.h>`,一般放到`stdafx.h`预编译头中．
另外需要在项目属性中选择＂在共享DLL中使用MFC＂.

以前有项目在VC6.0,后来迁移到VS2013,刚开始关于CString大量报错，发现是不同平台字符
编码的问题，从网上下载`Multibyte MFC Library for Visual Studio 2013`,安装之后，选择
多字节编码而非Unicode,即没有编码问题，CString也可以自由使用．

``` c
// use_CString.c

// 初始化
CString s;
CString s("hello");
CString s = "hello";

char c[] = "hello";
CString s = "";
s.Format("%s", c);

CString s = "hello";
// 长度
// 注意：英文每个字符占一个长度，中文每个占两个长度
printf("%d", s.GetLength());    // 5

// 反转
s.MakeReverse();    // "olleh"

// 转换大小写
s.MakeUpper();      // "HELLO"
s.MakeLower();      // "hello"

// 插入　删除
s.Insert(2, "a");   // "heallo"
s.Delete(3, 2);     // "hel"

// 替换与移除指定字符
s.Replace("ll", "yy");  // "heyyo"
s.Remove('l');          // "heo"

// 去除左右两边空格
// 一般从文件读取字符串，都会先去除两端空格，防止读取无意义数据
s.TrimLeft();   // 默认去除左端空格
s.TrimRight("a");  // 去除右端的任意多个"a"

// 清空字符串以及判断字符串是否为空
// 判断是否为空也常用于读取文件
s.Empty();
s.IsEmpty();    // 为空时返回0

// 查找
s.Find('e');    // 1
s.Find('ll');   // 2
s.Find('e', 1); // 0
s.Find('a');    // 找不到返回-1
s.ReverseFind('e'); // 反向查找，即先反向再查找，3

// 格式化
s.Format("%d", 2);  // "2"

// 取值与赋值
s.GetAt(2); // "l"	如果索引越界，会出异常
s.SetAt(2, 'h');    // "hehlo"

s.Left(2);  // "he"
s.Right(2);  // "lo"
s.Mid(2, 2);    // "ll"
s.Mid(2);   // "llo"

```

## CString与其他类型互转
CString常用于MFC,安全性高，但可移植性差
string常用于STL
char * 常用于API的输入参数


``` c
// convert_CString.c

// 1 CString 与　char *
CString s = "hello";
char *p = (LPSTR)(LPCTSTR)s;

char p[] = "world";
s.Format("%s", p);

// 2 CString 与　string
CString s;
string str = "hello";
s.Format("%s", str.c_str());

CString s = "hello";
string str(s.GetBuffer()); // string 类型无法用printf打印
CString.ReleaseBuffer();

// 3 char 与　string
char p[] = "hello";
string str(p);

const char *c = str.c_str();

// 4 CString 与　int
int i = atoi(s); // 转换浮点用atof
int i = _ttoi(s);

CString.Format("%d", i);

// 5 CString 与　char[100]
char a[100];
CString s("abc");
strncpy(a, (LPCTSTR)s, sizeof(a)); // vs2013报错，需要用strncpy_s

```

## CStringList
CStringList是MFC中定义的用于存储CString字符串的链表

``` c
// use_CStringList.c

// 构造
CStringList str_list;

// 添加删除元素
str_list.AddHead("123");    //在列表头部添加元素
str_list.AddTail("123");    //在列表尾部添加元素

str_list.InsertBefor(POSITION pos, "123");     // 在给定位置前插入新元素
str_list.InsertAfter(POSITION pos, "123");     // 在给定位置后插入新元素

str_list.RemoveHead();      // 分别时删除头，尾，所有元素
str_list.RemoveTail();
str_list.RemoveAll();

// 访问
str_list.GetHead();         // 获取头，尾部元素
str_list.GetTail();

str_list.GetAt(POSITION pos);           // 获取指定位置的元素
str_list.SetAt(POSITION pos);           // 设置指定位置的元素
str_list.RemoveAt(POSITION pos);        // 删除指定位置的元素

// 遍历所用
str_list.GetHeadPosition(); // 获取头部，尾部元素所在位置
str_list.GetTailPosition();

str_list.GetNext(POSITION pos);         // 获取下一个元素
str_list.GetPrev(POSITION pos);         // 获取前一个元素

// 查找
POSITION pos = str_list.Find("123");            // 获取由字符串指定的元素的位置
POSITION pos = str_list.FindIndex(int i);       // 获取由索引指定的元素的位置

// 状态
str_list.GetCount();        // 返回元素个数
str_list.IsEmpty();          // 测试列表是否为空

// 遍历
POSITION pos;
pos = str_list.GetHeadPosition();
while (pos != NULL)
{
    CString s = str_list.GetNext(pos);
    printf("%s", s);
}
```

## 附录

1 [如何解决VC6迁移到VS2013时出现的error MSB8031]( http://jingyan.baidu.com/article/9989c7461eac5ef648ecfef9.html )
2 [VS2008下非MFC工程使用CString类库]( http://blog.csdn.net/shuixin536/article/details/5899016 )
3 [CString 成员函数用法大全]( http://www.cnblogs.com/Caiqinghua/archive/2009/02/16/1391190.html )
4 [CString转换为LPCSTR方法补充]( http://blog.csdn.net/netist/article/details/4091421 )
5 [CString Format函数 VS2013]( http://blog.csdn.net/tahelin/article/details/32344615 )
6 [CString转char *,strings]( http://blog.csdn.net/huihui0121/article/details/5804446 )
7 [C语言中string函数详解]( http://blog.csdn.net/sunnylgz/article/details/6677103 )
8 [CSTRINGLIST用法]( http://www.cppblog.com/Mumoo/archive/2013/04/15/199460.aspx )
9 [CString,string,char *之间的转换]( http://www.cnblogs.com/bluestorm/p/3168720.html )
10 [MFC CString 和int相互转化]( http://blog.sina.com.cn/s/blog_5fa918660101axuf.html )

感谢网上的朋友！

## 一个小问题
写这篇总结的时候，最后附录有十个链接，我在本地localhost测试，这十个链接只能显示六个,
而且每次刷新出来的页面还都不一样，看网页代码最后部分是乱码，改改markdown中的[]与()
之间加了空格，偶尔会正常出来十个链接，再刷新又没有了，最后deploy到github又显示正常．
暂时没有找到原因，先记下来问题，之后再处理．
