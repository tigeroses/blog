---
title: linux相关之系统
date: 2021-06-08 20:36:35
tags: linux
---

这里主要是关于 centos7 的总结

## 系统版本信息

查看系统内核版本信息  
`cat /proc/version`  
`uname -a`

查看centos版本,如centos 7.9  
`cat /etc/redhat-release`

查看CPU型号及支持的指令集  
* `gcc -march=native -Q --help=target|grep march` 
* `cat /proc/cpuinfo`  
* `cpuid` (需安装此命令)

## centos增删用户

增加用户  
1. `adduser xx` 创建用户
2. `passwd xx` 按提示设置密码
3. 修改 /etc/sudoers 文件为可写权限, 添加一行 `xx  ALL=(ALL)       ALL` ,再改为只读权限

删除用户  
`userdel xx` 

## 交换内存

`free -m` 查看交换内存情况

通过swap分区文件增加swap空间

1. `dd if=/dev/zero of=swapfile bs=1M count=1024` bs是每个块大小,count是块数量,这里设置共1G
2. `mkswap swapfile` 格式化交换分区文件
3. `swapon swapfile` 启用
4. 设置开机自启. 修改/etc/fstab 文件添加一行 `swapfile swap swap defaults 0 0` 注意使用全路径

使用文件作为交换内存的好处:

1. 如果磁盘空间不足,可以删除文件,释放部分磁盘空间,为抢救数据争取时间
2. 在资源不足时增加虚拟内存

## 磁盘性能

查看磁盘存储空间  
`df -h`

查看某进程的磁盘读写速度(需执行命令 `yum install -y pidstat`)  
`pidstat -p ${pid} -d 1`  

检测磁盘读写速度,一种不太准确的方式  
写 `dd if=/dev/zero of=test.file bs=1G count=2 oflag=direct`  
读 `dd if=test.file of=/dev/null  iflag=direct`


## 查询端口

在win下查看linux的端口是否打开(需开启telnet服务)  
`telnet 172.17.15.12 8000`

`netstat -nlp | grep 8080` 查看本机端口  

`systemctl start firewalld` 开启防火墙  
`firewall-cmd --query-port=666/tcp` 查看想开的端口是否已开  
`firewall-cmd --add-port=666/tcp` 开端口号  
`firewall-cmd --remove-port=666/tcp` 移除端口  