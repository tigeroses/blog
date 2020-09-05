---
title: vscode连接远程服务器
date: 2020-09-05 16:51:13
tags: vscode
---

## 起因

vscode有远程开发功能,即可以在windows打开vscode,连接上linux服务器写代码  
但由于公司集群登录节点是centos6,官方表示centos6需要升级glibc和libstdc++,没有管理员权限,只能找一台centos7的计算节点,想办法跳过登录节点

## 使用ssh tunnel

### win10安装ssh

可选择安装openSSH或者通过WSL/cygwin安装SSH  
参考: http://www.win10.one/jc/6606.html

### 建立ssh tunnel

运行命令 `ssh -NfL PRIVATE_PORT​:TARGET_SERVER​:22 USER_NAME​@LOGIN_SERVER​​`  
示例 `ssh -NfL 8888:10.225.1.1:22 zhangsan@10.225.2.2`  
解释:  
1. PRIVATE_PORT​ 端口号,注意不能重复,可以尽量给大一点
2. TARGET_SERVER​ 目标centos7机器的地址,vscode连接后将在此服务器运行
3. LOGIN_SERVER​​ 版本为centos6的登录节点的地址,仅作为跳板使用

### vscode配置

在Remote插件进行配置

```
Host login
  HostName localhost
  Port 8888
  User zhangsan
  IdentityFile C:\Users\zhangsan\.ssh\id_rsa
```

这样就可以通过输入login这个代号进行远程连接了