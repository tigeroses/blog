---
title: linux更改键盘映射
date: 2016-03-27 22:20:22
tags: [linux]
category: [tools]
---

因为习惯使用vim 编辑器，而早期的vi 的键盘设置跟现在的qwert键盘的按键差别较大，所以我一般选择将不常用的Caps_Lock与常用的Esc 互换，在Win下有很多好用的软件可以直接更改，linux下需要用到xmodmap这个软件来实现。

## 获取按键具体名称
使用 xmodmap -pke |less 查看想要交换的按键的具体名称

## 写入配置文件
将需要交换的按键写入配置文件~/.keymaprc 
``` bash
remove Lock = Caps_Lock
keysym Caps_Lock = Escape
keysym Escape = Caps_Lock NoSymbol Caps_Lock
```
使用xmodmap ~/.keymaprc 命令即可更改设置，再运行一遍又改回来。

## 加入环境变量
为了不每次都输入上边的命令,可以将其写入文件
``` bash
$ cat "xmodmap ~/.keymaprc" > ~/swkey
$ chomd a+x ~/swkey
$ sudo mv ~/swkey /usr/local/bin
```
这样每次需要更改按键的时候，输入swkey 命令即可。

## 其他问题
这样的设置在只有一个英文输入法的时候好使，后来我又添加了中文拼音输入法，每次切换中文再切回来之后键盘设置都会重置，即需要再次输入 swkey 才可以，一直找不到解决办法。

最后我索性去掉英文输入法，只保留中文拼音，初始化为英文，需要切换英文按shift，这样不会出现键盘设置重置的问题，到目前来看用起来还不错。
