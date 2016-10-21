---
title: '技巧与工具01:Linux工作环境配置'
date: 2016-10-21 11:19:55
tags: 配置　Linux
---

个人电脑系统是Ubuntu,也因为Linux环境工作效率更高，比较偏爱Linux系统，
平时写代码主要使用Vim,故总结出工作环境的简单配置．

<!--more-->

本篇所有的配置文件都已上传至github: [https://github.com/tigerRose/experience.git](https://github.com/tigerRose/experience.git) 可以直接拷贝到个人目录使用.

## ~/.bashrc
这个是Linux最主要的配置文件之一了．
```bash
# ~/.bashrc

if [ $TERM="xterm-256color" ]
then
    PS1='\[\033[01;38;5;61m\]\u@\h\[\033[01;38;5;107m\] \w\n\$\[\033[01;38;5;248m\]'
fi

LS_OPTIONS='--color'
export LS_OPTIONS

alias ls="ls $LS_OPTIONS"
alias l="ls -lh"
alias ll="ls -l"
alias la='ls -alh'
alias c7='chmod 777'
alias c5='chmod 755'
alias c4='chmod 644'
alias c0='chmod 700'
alias c0f='chmod 600'
alias dh='du -h'
alias eb='vi ~/.bashrc'
alias ep='vi ~/.bash_profile'
alias sb='source ~/.bashrc '
alias h='head'
alias k9='kill -9'
alias les="less -RS"
alias le='les'
alias ln='ln -s'
alias wl="wc -l"
alias a="cd .. && l"
alias b='cd - && l'
alias pp="pu $USER"
alias i='dirs -c && pushd .'
alias o='dirs -c'
alias p='pushd'
alias rd="rm -rf"
alias t="tail"
alias vi="/usr/bin/vim"
alias py="python"


```
~/.bashrc主要是配置终端显示以及设置一些常用命令的重命名，减少频繁使用
的命令的按键次数，也可以指定所使用的程序，如`alias python="C:\python27\python.exe"`
这个就是我在windows系统下使用cygwin环境，调用windows安装的python的方法．
另外可以在这里加一些环境变量，如`export PYTHONPATH="xxx"$PYTHONPATH`

## ~/.git-completion.bash
这个文件在网上可以下载，主要功能如名称所示，git的命令行补全，如输入:
`git ch<tab>` 会出现`checkout　cherry　cherry-pick`供参考．使用前需要将如下
几行代码添加到~/.bashrc

```bash
# ~/.git-completion.bash

# set git auto completion
if [ -f ~/.git-completion.bash ]; then
    . ~/.git-completion.bash
fi

```
然后输入命令source ~/.bashrc即可生效

## ~/.gitconfig
此文件是git的简单配置，如用户名和邮箱

```bash
# ~/.gitconfig

[user]
	email = fuxiang_zhao@163.com
	name = tigerRose
[push]
	default = matching

```

## ~/.keymaprc
这个是我自己定义的，用于交换键盘的按键，比如你某个键坏掉了，可以用一个平时不
常用的键来交换，土豪可以无视，直接买新的．我使用按键交换主要是因为习惯用Vim,
而又常用Esc,不常用Caps Lock,因此交换按键，减少手指运动量.

```bash
# ~/.keymaprc

remove Lock = Caps_Lock
keysym Caps_Lock = Escape
keysym Escape = Caps_Lock NoSymbol Caps_Lock
```
[linux更改键盘映射](http://tigerrose.me/2016/03/27/linux%E6%9B%B4%E6%94%B9%E9%94%AE%E7%9B%98%E6%98%A0%E5%B0%84/)这个专门讲Linux环境改按键

## ~/.vimrc
此文件是Vim的配置，个人比较喜欢的风格

```bash
"~/.vimrc

set nocompatible    "Use Vim settings, rather then Vi settings (much better!).
syntax on           "show syntax
set nu              "show line number
set tabstop=4       "<tab> width
filetype plugin indent on
set autoindent      "set automatic indent
set smartindent cinwords=if,elif,else,for,while,try,except,finally,def,class
set softtabstop=4   
set expandtab       
set ruler
set shiftwidth=4    "autoindent width
set cindent         "C style indent
set sm              "show matched bracket
set smarttab
set fileformats=unix
set backspace=indent,eol,start
set incsearch
set fencs=utf-8,ucs-bom,euc-jp,gb18030,gbk,gb2312,cp936 "set coding

set t_Co=256 "Use 256 color
colorscheme lucius "set ColorScheme
set viminfo='1000,<800

"return the position last edit
au BufReadPost * if line("'\"") > 0|if line("'\"") <= line("$")|exe("norm '\"")|else|exe "norm $"|endif|endif

set hlsearch
```
配色方案放在~/.vim/colors 目录

## windows环境cygwin环境安装
cygwin是windows下使用linux环境的不二之选，安装也很简单，如果联网环境，可以勾选
自己想要的库和软件包;使用时候如果发现有软件没有安装，需要重新安装一遍，不过
已安装的不会再次下载．
不联网的环境，可以先找个联网的机器下载需要的安装目录，然后选择从本地来源安装即可．
cygwin.rar是我自己使用的一个安装包，包含了vim编辑器，gcc编译器以及python大多数库.

