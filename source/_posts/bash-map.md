---
title: bash-map
date: 2016-04-05 21:49:45
tags: [bash,map]
category: [programming]
---

这篇主要记录linux下写简单shell脚本调用bwa对fastq文件作mapping

## code

输入两个或者三个参数，分别表示SE PE的情况，第一个参数是ref，可选为在reference所在目录下的已经建好Index的参考基因组，如Ecoli.fa Human.fa 等

``` bash
#!/bin/bash

if [ $# -lt 2 ];then
    echo 'Wrong parameters.'
    exit
fi

ref=$1
fq1=$2
fq2=$3

#bn=$(basename $fq1)
dn=$(dirname $fq1)
bn=${bn%%.*}
bam=$dn/$bn.bam
reference=/home/tigerrose/REF/BWA_REF/$ref

if [[ -e $fq2 ]]; then
    # PE
    p1=$dn/$bn.pipe
    bn=$(basename $fq2)
    dn=$(dirname $fq2)
    bn=${bn%%.*}
    p2=$dn/$bn.pipe
    bam=$dn/${bn%_*}.bam

    if [ ! -e ${p1} ];then mkfifo ${p1}; fi
    if [ ! -e ${p2} ];then mkfifo ${p2}; fi

    bwa aln -t 20 -l 25 $reference $fq1 > $p1 &
    bwa aln -t 20 -l 25 $reference $fq2 > $p2 &
    bwa sampe $reference $p1 $p2 $fq1 $fq2 | samtools view -bS - > $bam

    rm $p1
    rm $p2
else
    # SE
    bwa aln -t 20 -l 25 $reference $fq1 | bwa samse $reference - $fq1 | samtools view -bS - > $bam
fi
```

## shell脚本基础

这个简单的脚本中用到了一些shell脚本的基础知识

``` bash
$#  # 参数的个数
-lt  # 比较两个整数，小于的意思
$(dirname $var)  # 获取地址var的目录地址
${var%%.*}  # 将变量var按照.分割，并获取最左边的部分
-e $var # 出现在if 判断语句，表示存在这个文件var
mkfifo $var  # 创建命名管道var
exe1 | exe2 -  # 将exe1运行的结果通过管道传给exe2，其中-表示前面传入内容所在的位置
```


