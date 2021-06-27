---
title: linux相关之文本处理
date: 2021-06-27 17:59:01
tags: linux
---

## awk

使用正则匹配多个数字跟G加tab并输出  
`awk '/[0-9]+G\t/{print}' filename`

正则提取字符串,存储()中的内容为数组,并通过索引获取  
```sh
awk '{match($0,/[0-9]+.[0-9]+/,a);print a[0]}' lines
awk '{match($0,/.+is([^,]+).+not(.+)/,a);print a[1],a[2]}' test
```

## sed

sed 双引号支持变量引用/转义符

```sh
# 消除文件^M
sed 's/\r//g' filename

# 同时替换多个
sed -e "" -e "" filename 

# 原地替换 
sed -i 's/ /\n/g' filename
```

## paste/split

将多个文件作为多列来拼接  
`paste file1 file2 -d ','`

split分割文件

* 按行数分割 `split -l 1000 file -d -a 4 prefix` 分割后文件前缀为prefix,文件名后缀是数字而非默认的字母,后缀系数为4位

* 按大小分割 `split -b 10m file`

## find

查找多个条件  
统计c++项目源代码行数  
`find . -regex ".*\(\.cpp\|\.h\|\.hpp\|\.c\)$" -exec wc -l {} \; | awk '{s+=$1}END{print s}'`

## diff

比较文件异同

diff 默认普通模式 -c 上下文模式 -u 合并模式

vimdiff 相当于vim -d

## grep

grep如何匹配转义字符如\t  
`grep "prefix"$'\t' filename`

grep进行正则的提取  
`grep -P '\(.*\)' -o`

## vim

遇到vim写py缩进问题,一般是tab和空格混用,可以 `%s/\t/    /g` 将所有tab转成空格

报错 vim  E667: Fsync failed  
无法编辑和保存home目录下文件,但是nano可以编辑  
一般是磁盘空间满了,或者有空间但是系统限制了配额,如集群会限制每人home目录如100M的配额,而很多库及软件的缓存目录都默认在home目录,可以使用软连接将目录链接到其他磁盘,空出home目录的磁盘空间

反撤销 ctrl+r

## xargs

xargs主要用于将标准输入转换为参数,适用于部分命令不支持标准输入的情况  
`cat arg.txt | xargs -I {} ./test.sh -p {} -l`

## shell脚本

获取上一条命令执行结果

```shell
if [[ $? == 0 ]];
then
    xxx
else
    xxx
fi
```

判断路径是否存在

```shell
if [ ! -e "$binPath" ];
then
    mkdir $binPath
fi
```

test 命令  
`test -z a` 检查字符串a是否为null,如果为null,返回true

参数：

* $# 参数个数
* $* 所有参数,一个单字符串显示
* $$ 脚本运行的当前进程ID
* $! 后台运行的最后一个进程ID
* $@ 所有参数，每个参数加引号
* $? 命令的推出状态，0表示无错
* $1 第一个参数

生成序列 for

```sh
for i in {0..2};
do
    echo $i;
done
```
