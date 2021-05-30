---
title: singularity容器使用心得
date: 2021-05-30 18:03:10
tags: 容器
---

将软件或流程打包进容器,可以方便地在云上进行大规模部署,这里记录下自己使用singularity工具的过程

## 安装

centos 系统推荐安装方式 `yum install -y singularity`

[官网安装链接](https://sylabs.io/guides/3.7/admin-guide/installation.html#install-nonsetuid)  
[Installing Singularity](https://github.com/hpcng/singularity/blob/master/INSTALL.md)

手动安装,需要把singularity安装包放到go安装包里面

设置go代理 `go env -w GOPROXY=https://goproxy.cn,direct`
如果不生效,直接修改Makefile中的配置

## 创建镜像

沙盒模式方便交互式修改配置,直到满意为止;定义文件模式更加规范,使安装更新软件变得简单

### 沙盒模式

build 命令会创建一个ubuntu/centos7目录,里面有整个对应的操作系统,在你当前工作目录中会有一个Singularity元数据

```sh
$ sudo singularity build --sandbox ubuntu library://ubuntu

$ sudo singularity build --sandbox centos7 docker://centos:7
```

你可以在这个目录中使用shell,exec和run等命令就像你在一个Singularity镜像中.如果当你使用容器的时候传递了--writable 选项,你也可以在沙盒目录中对文件进行读写（当你有对应的权限时）

```sh
$ sudo singularity exec --writable centos7 touch /foo

$ sudo singularity exec centos7 ls /foo
/foo

$ sudo singularity shell --writable centos7
Singularity> rm -rf /foo
Singularity> ls /foo
ls: cannot access /foo: No such file or directory
Singularity> exit
exit
```

### 定义文件模式

定义文件是后缀为.def的纯文本文件,格式分为文件头和文件主体

文件头指定基础容器来源

文件主体是一系列sections:

* setup. 首先执行于host环境,可以访问 $SINGULARITY_ROOTFS 容器的根目录,是在宿主目录的/tmp目录下临时建的;推荐使用 %files 来拷贝文件
* files. 拷贝文件,可以从host拷贝到容器,或者从容器的某个步骤拷贝,格式是两列,source dest
* post. 网络下载,安装库,写配置文件,创建目录; $SINGULARITY_ENVIRONMENT = /.singularity.d/env/91-environment.sh 可以写入这个配置文件,build time使用
* test. 做检查验证
* environment. export环境变量,用于run time,如果build time需要用到,去post中设置
* runscript. 运行镜像,exec prog "$@" 来调用可执行
* labels. 加作者,版本信息

[官方文档-定义文件](https://sylabs.io/guides/3.5/user-guide/definition_files.html)

一个简单的示例,演示了以centos 7为基础镜像,拷贝文件,安装java和python等

```sh
# filename: simple.def
Bootstrap: docker
From:centos:7.6.1810

%files
    # 拷贝当前目录下的某文件到镜像的/opt目录
    $PWD/simple.def /opt

%post
    yum install -y -q wget

    # install java
    cd /opt && wget -q https://repo.huaweicloud.com/java/jdk/8u171-b11/jdk-8u171-linux-x64.tar.gz && \
        tar zxf jdk-8u171-linux-x64.tar.gz && mv jdk1.8.0_171 java
    rm jdk-8u171-linux-x64.tar.gz

    # install python
    yum install python3 -y -q
    pip3 install -q opencv-python -i https://pypi.douban.com/simple

%environment
    # set java path
    export PATH=$PATH:/opt/java/bin
    export LC_ALL=C

%runscript
    echo "Arguments received: $*"
    python3

%labels
    Author tigeroses
    Version v1.0
```

## 打包镜像

不管是沙盒模式的一个目录还是定义文件模式的一个文件,最终要给用户使用都需要打包成后缀为.sif的镜像文件

```sh
# 沙盒模式
# -F 表示覆盖已存在的sif文件,用于需要反复测试打包的情况
sudo singularity build -F my_container.sif centos7

# 定义文件模式
sudo singularity build -F my_container.sif simple.def
```

## 运行镜像

拿到sif文件之后,就可以测试运行了

可以直接运行,等价于使用run命令,如果定义文件中存在 %runscript section,则会执行其中的shell 命令,适用于功能单一的软件,可以方便的使用;如果没有%runscript, 则会进入容器的交互模式

exec命令可以更灵活的执行镜像中安装的软件

```sh
# 直接运行,由于设置了python命令,则会进入python的交互界面
$ ./my_container.sif
Arguments received:
Python 3.6.8 (default, Nov 16 2020, 16:55:22)
[GCC 4.8.5 20150623 (Red Hat 4.8.5-44)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>>

# 使用 run 命令
$ singularity run my_container.sif
Arguments received:
Python 3.6.8 (default, Nov 16 2020, 16:55:22)
[GCC 4.8.5 20150623 (Red Hat 4.8.5-44)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>>

# 使用 exec 命令
$ singularity exec my_container.sif java -version
java version "1.8.0_171"
Java(TM) SE Runtime Environment (build 1.8.0_171-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.171-b11, mixed mode)
```

其他:

* 如果有需要临时修改镜像内部数据,exec命令增加 `--writable-tmpfs`
* 默认镜像只能访问$HOME目录,因此需要绑定目录,如将外部的/opt目录绑定为/mnt目录, 增加命令 `--bind /opt:/mnt` 容器使用 /mnt 来访问宿主机 /opt 中的文件; 可以绑定多个目录
* 如果使用了mysql,由于每次使用容器数据库会被刷新,为防止丢失数据,需要将mysql的数据目录放到本地磁盘上,绑定目录如 `--bind /opt/mysql/var/lib/mysql/:/var/lib/mysql --bind /opt/mysql/run/mysqld:/run/mysqld`

## 其他补充

### 定制基础镜像

对于较大的项目,安装库和环境较复杂可能会导致打包容器镜像耗时较久,影响开发效率,而一些库和软件如python java等会比较稳定,因此可以把比较稳定的软件与经常更新的软件隔离开,分层次先打包基础镜像,然后每次发布新版本可以在基础镜像之上打包发布

示例:

```sh
# filename: base.def
Bootstrap: docker
From:centos:7.6.1810

%post
    yum install -y -q wget

    # install java
    cd /opt && wget -q https://repo.huaweicloud.com/java/jdk/8u171-b11/jdk-8u171-linux-x64.tar.gz && \
        tar zxf jdk-8u171-linux-x64.tar.gz && mv jdk1.8.0_171 java
    rm jdk-8u171-linux-x64.tar.gz

%environment
    # set java path
    export PATH=$PATH:/opt/java/bin
    export LC_ALL=C
```

打包base.sif `sudo singularity build -F base.sif base.def`

```sh
# filename: simple.def
# 注意基础镜像的来源要修改
Bootstrap: localimage
From:base.sif

%post
    # install python
    yum install python3 -y -q
```

打包simple.sif `sudo singularity build -F simple.sif simple.def`

测试执行,可以看到java和python都可以使用

```sh
$ singularity exec simple.sif java -version
java version "1.8.0_171"
Java(TM) SE Runtime Environment (build 1.8.0_171-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.171-b11, mixed mode)

$ singularity exec simple.sif python3 -c "print('hello world')"
hello world
```

### 在容器中运行服务

这里模拟下如何使用supervisor软件在容器中运行redis服务

准备supervisor需要的两个配置文件

redis.ini

```ini
[program:redis]
command=redis-server
user=root
autostart=true
autorestart=true
priority=20
stdout_logfile=/mnt/logs/redis/redis.out.log
stderr_logfile=/mnt/logs/redis/redis.err.log
```

默认的supervisord.conf,修改下

```ini
[unix_http_server]
file=/mnt/supervisor/supervisor.sock
[supervisord]
logfile=/mnt/supervisor/supervisord.log
[supervisorctl]
serverurl=unix:///mnt/supervisor/supervisor.sock

...

[include]
files = /opt/redis.ini
```

定义文件如下,安装redis/supervisor

```sh
# filename: server.def
Bootstrap: docker
From:centos:7.6.1810

%files
    $PWD/supervisord.conf /opt
    $PWD/redis.ini /opt

%post
    # install redis
    yum install -y -q gcc*
    yum install -y -q wget make
    cd /opt  && wget -q http://download.redis.io/releases/redis-4.0.6.tar.gz && \
        tar xzf redis-4.0.6.tar.gz && cd redis-4.0.6 && make -s && make install -s
    rm /opt/redis-4.0.6.tar.gz

    yum install python3 -y -q
    pip3 install supervisor -i https://pypi.douban.com/simple
```

测试:

* 打包. `singularity build -F server.sif server.def`
* 创建目录. `mkdir -p $PWD/supervisor $PWD/logs/redis`
* 运行.
  * 开启supervisord服务. `singularity exec --bind $PWD:/mnt server.sif supervisord`
  * 查看服务状态. `singularity exec --bind $PWD:/mnt server.sif supervisorctl status`
  * 关闭服务. `singularity exec --bind $PWD:/mnt server.sif supervisorctl stop all`
  * 开启服务. `singularity exec --bind $PWD:/mnt server.sif supervisorctl start all`
  * 杀掉supervisord服务. `ps -ef | grep supervisord` 找到进程ID,再使用 `kill -9 ID` 杀掉进程

实际使用过程可以把命令封装成脚本调用,使用户使用起来更加简洁

### 容器加密

正常情况下打包的容器镜像,用户可以通过shell命令进入容器内部,如果想限制别人的使用,就涉及到容器的加密

密钥方式

```sh
# Generate a keypair
$ ssh-keygen -t rsa -b 2048
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): rsa
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
[snip...]

# Convert the public key to PEM PKCS1 format
$ ssh-keygen -f rsa.pub -e -m pem > rsa_pub.pem

# Rename the private key (already PEM PKCS1) to a nice name
$ mv rsa rsa_pri.pem

# 加密
$ sudo singularity build -F --pem-path=rsa_pub.pem base.sif base.def

# 无密钥情况会报错无法访问
$ ./base.sif
FATAL:   Unable to use container encryption. Must supply encryption material through environment variables or flags.

# 解密
$ singularity shell --pem-path=rsa_pri.pem base.sif
Singularity>
```

密码方式

```sh
# 交互方式输入密码,也可以选择
# export SINGULARITY_ENCRYPTION_PASSPHRASE=$(cat secret.txt)导入环境变量来设置密码
$ sudo singularity build -F --passphrase base.sif base.def
Enter encryption passphrase:
INFO:    Starting build...

# 交互方式验证密码
$ singularity shell --passphrase base.sif
Enter encryption passphrase:
Singularity>
```

## 参考

* [Encrypted Containers](https://sylabs.io/guides/3.6/user-guide/encryption.html)
* [Running Services](https://sylabs.io/guides/3.6/user-guide/running_services.html)