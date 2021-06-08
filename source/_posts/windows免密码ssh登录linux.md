---
title: windows免密码ssh登录linux
date: 2015-12-06 20:43:22
tags: [windows,linux,ssh]
category: [tools]
---

工作需要从windows下免密码登录linux执行任务，主要利用的是ssh-key生成密钥，并添加到账户目录下，以达到目的。

## 准备

如果是linux相互之间添加公钥，可以使用内置的ssh命令，但是从windows下ssh登录linux，需要先下载win版本的ssh工具。

## 添加HOME变量

打开环境变量属性页面，在用户变量部分点击新建，变量为HOME，值为 `C:\Users\name` 其name为用户名，可以去查看自己电脑的用户名，之后生成的密钥对默认保存在这个目录下。

## 生成密钥对

打开cmd命令行，在ssh程序所在目录运行，或者添加系统环境之后随处运行: `ssh-keygen -t rsa` 这条命令用来生成密钥，随后一路回车，当看到一幅矩形图生成，那么密钥生成成功。

## 将公钥添加到linux账户

同样的打开cmd命令行，输入 `ssh username@host “cat >> ~/.ssh/authorized_keys” < C:\User\name\.ssh\id_rsa.pub` 这条命令是首先登录linux，然后将本机即win下的公钥添加到账户个人目录下，从而实现免密码登录。注意这一步需要输入账户的密码。

## 验证是否添加成功

cmd下输入 `ssh username@host uname` 如果看到输出Linux 表示添加成功。  
同样可以直接输入 `ssh username@host` 这时可以看到不用输入密码即可登录linux了。

## 依然无法免密码登录

添加key之后还是需要输入密码,主要排查方法是 `ssh username@host -vvv` 输出debug信息,或者remote机器使用 `cat /var/log/secure` 查看日志

可能的原因是:

* 目录权限问题. 远程目录.ssh 和.ssh 目录下的文件的权限需要分别设置700和600
* 找不到密钥. ssh登录命令添加参数 `-i C:\User\name\.ssh\id_rsa` 手动指定密钥文件

