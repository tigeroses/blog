---
title: 调试JS代码
date: 2020-07-05 22:04:01
tags: JS
---

记录下近期对JS代码的调试过程

## 性能分析

启动程序之后,打开google浏览器对应页面,按F12或者Ctrl+Shift+I进入 **开发者工具页面**

目前主要使用的功能有:
1. Performance. 性能评估,比如我想看下页面刷新的性能瓶颈所在,先点击 *Record* 按钮,然后进行页面操作,当页面刷新完成,再点击 *Stop* 按钮,则会生成性能报告,可以看到资源消耗,JS代码的执行逻辑等
2. Sources. 性能报告页面的 *Main* 部分,可以通过点击色块查看其所在的js代码文件,如 *react-xx.js* 点击则会跳转到 **Sources** 功能栏,有了源文件就可以进行断点调试;这里注意部分js文件是压缩后的文件,建议手动修改程序替换成可读性更强的原始代码文件,方便调试
3. Console. 查看程序的打印输出,比如我想知道某个函数的执行时间,可以在js代码中进行修改
    ```js
    console.time("foo");
    foo();
    console.timeEnd("foo");
    ```
    当js代码执行之后,可以在console输出中看到foo的执行时间

4. Network. 查看文件传输的时间,判断下瓶颈是否在网络带宽,以及是否数据量太大导致数据的转换和传输耗时较久

## 性能调优

通过性能分析,发现耗时最长的模块的操作是对数据的颜色计算,场景是我有1M个点需要显示,那么需要将它们从一个[2,1,4,10...]的 *颜色数组* 转换成RGB表示,js代码使用for循环进行操作,也就是线性复杂度,计算耗时随数据量的增大而线性增大

通过debug观察发现颜色数组会有不少重复的数值,而同样的输入会导致相同的输出,然后对整个数据的1M个点进行统计分析,发现重复率相当高,也就是for循环中有大量的重复计算

很自然想到增加缓存,用空间换时间来加速,当遇到计算过的数据时,直接返回计算结果,从而大大提高了程序执行的效率
```js
var cache = {};
for(var i = 0; i < len; i++) {
    if (cache[colorIn[i]])
    {
        colorOut[i] = cache[colorIn[i]];
        continue;
    }
    colori = getColor(colorIn, i);
    opacityi = getOpacity(opacityIn, i);
    colorOut[i] = calculateColor(colori, opacityi);
    cache[colorIn[i]] = colorOut[i];
} 
```