---
title: pyinstaller打包flask代码报错
date: 2016-08-25 20:13:21
tags: [python,pyinstaller,flask]
category: [debug]
---

最近工作需要用到flask的restful架构做服务器，而工作环境又在windows下，因此需要打包成exe

## 打包完运行程序报错
打包工具首选pyinstaller,在cmd下用命令pyinstaller.exe -F xxx.py 即生成一个xxx.exe，打包没有报错，
但是在运行程序的时候，首先弹出对话框，Runtime Error, R6034,程序试图访问动态库报错，接着黑框一闪而过，通过截屏发现cmd中报错是　No module named ext.restful. 而我在代码中用的是from flask.ext.restful import Api, Resources

## 解决过程
一路搜索无果，无意中看到其他人使用pyinstaller打包也报错找不到模块，重新安装一遍第三方库即可．因此我也用pip uninstall, pip install重装了flask 和flask-restful,然后运行python代码，有警告说from flask.ext.restful import * 已经过期，建议使用from flask_restful import *,我将代码更正，重新打包并运行，发现不报找不到flask库的错了，但是那个Runtime Error还在，程序也能正常运行，但是总不能给别人的程序一运行先报错吧，所以这个问题还要解决，这次是在stackoverflow上发现了解答，说是pyinstall 3.2版本bug比较多，3.2打包报错换成3.1就可以了，我重新装了pyinstaller 3.1, 方法是 pip install pyinstaller==3.1 然后问题解决，Runtime Error没有了．
