---
title: begin_centos7
date: 2019-03-17 21:46:35
tags: centos
---

本篇主要记录个人电脑安装centos及其软件过程， 以备下次需要重装系统之用。

## 安装系统

### 1. 下载系统安装包

在centos官网下载[安装包](https://www.centos.org/download/), 目前最新版本是7.6， 我下载的everything版本， 约10G， 包括最全的内容， 虽然最后只装了个GUI版本。

### 2. 刻盘

使用u盘安装的方式， 首先下载[ultroiso](https://cn.ultraiso.net/xiazai.html), 可以选择免费试用版， 然后在windows系统电脑插入u盘， 打开ultroiso，加载步骤1下载的iso文件， 选择刻录到u盘启动，等待10多分钟， 启动u盘刻录完毕

### 3. 安装

* 插入u盘， 重启电脑， 开机过程中按**F2**进入**BIOS**， 设置启动顺序为u盘优先， 保存配置并退出
* 在**Install Centos 7**这一行按**e** 进入编辑模式， 将脚本中对应内容修改为
> initrd=initrd.img linux dd quiet

    回车， 屏幕会打印设备信息， 从中可以找到u盘所对应的id， 如： */sdc4*， 这一步是查找u盘的映射id， 因为脚本中默认的名称是错误的。

    重启之后， 继续进入编辑模式， 修改内容为
> initrd=initrd.img inst.stage2=hd:/dev/sdc4 nomodeset quiet

    其中加入了一句 *nomodeset*, 原因是不加的话无法进入图形界面（安装过程）， 报错为 "*X startup failed, falling back to text mode*"
* 至此，可以通过图形界面一路点点点进行安装，我安装的是GUI版本，语言设置为英语，时区选上海

## 基本环境配置

### 1. 语言

通过快捷键切换中英文，虽然安装环境选的是英文，但是语言栏可以添加中文，很不错

### 2. 无线上网

有线可以忽略；无线需要购买对应的无线网卡， 支持linux，最好买不用驱动安装的，插入即可使用，要不然就会知道**.ko**文件如何生成和使用（linux驱动文件）

### 3. 下载软件

推荐**qBittorrent**, 优点是跨平台，且可以通过centos系统自带的应用程序安装器进行安装，虽然我下载速度慢的和乌龟一样

### 4. 视频播放软件

自带的**Videos**没有解码器，无法播放视频；推荐**Mplayer**，代码编译，相当酸爽
* 下载代码 `$ svn checkout svn://svn.mplayerhq.hu/mplayer/trunk mplayer` (貌似当时无法下载，找了官网换了链接OK)
* 更新代码 `$ svn update`
* 依赖包
    * 下载 `$ wget http://www.mplayerhq.hu/MPlayer/releases/codecs/essential-amd64-20071007.tar.bz2`
    * 解压 `$ tar -xaf essential-amd64-20071007.tar.bz2`
    * 建目录 `$ sudo mkdir /usr/local/lib/codecs`
    * 拷贝 `$ sudo cp essential-amd64-20071007/* /usr/local/lib/codecs`
* 生成Mplayer编译所需配置 `$ ./configure --enable-gui --language=zh_CN`
    * 依赖[yasm-1.2.0-4.sdl7.x86_64.rpm](http://pkgs.org/download/yasm)
    * `$ yum install gtk2* `
    * 报错就对应修改
* 编译 `$ make`
* 安装 `$ make install`
* 安装皮肤，才能用GUI
    * 下载 `$ wget http://www.mplayerhq.hu/MPlayer/skins/Blue-1.10.tar.bz2`
    * 解压 `$ tar -xaf Blue-1.10.tar.bz2`
    * 复制 `$ sudo cp -R Blue /usr/local/share/mplayer/skins/`
    * 软链接 `$ cd /usr/local/share/mplayer/skins/` `$ sudo ln -s Blue/ default`
* 没有声音的解决方案:重新**./configure** 这次加上参数*–codecsdir=/usr/local/lib/codecs*

### 5. markdown编辑器

推荐使用**Atom**，下载rpm包直接安装即可，功能强大，目前使用其来进行markdown文件的编辑，用于写博客；还可以进行代码编写等

### 6. 浏览器

浏览器是上网的窗口，自带的firefox就很好用，不过我还是选择使用时间更久的**Chrome**；下载插件**vimium** 进行无鼠标的网页浏览操作

### 7. 配置终端

怎么可以不用命令行？终端配置目前主要是**bashrc vimrc**， 另外还有键盘的重新映射，即改键，我主要是把*esc*和*caps*互换，毕竟*esc*使用频率太高了，而它离手指又太远了。
参考[技巧与工具01:Linux工作环境配置](http://tigeroses.com/2016/10/21/%E6%8A%80%E5%B7%A7%E4%B8%8E%E5%B7%A5%E5%85%B701-Linux%E5%B7%A5%E4%BD%9C%E7%8E%AF%E5%A2%83%E9%85%8D%E7%BD%AE/)

### 8. 办公套件

直接在系统自带的应用安装器中进行安装，名称为**LibreOffice**, 安装三个， 分别是
* Calc == excel
* Impress == powerpoint
* Writer == word

### 9. 分辨率

刚装完系统时分辨率很低，还无法设置；我的显卡是Nvidia GTX 750，在官网下载对应驱动安装之后分辨率恢复正常

### 10. 关闭主机usb供电

BIOS中Power Management 下的ErP设置为Enabled即可，主要是有时候主机关闭之后鼠标灯还亮着，强迫症真难受，关闭usb供电可以解决这个问题

## 附录

参考网上经验

* [U盘安装CentOS7的最终解决方案](https://www.cnblogs.com/pythonal/p/6825906.html)
* [【14-07-31】 【总结】如何在CentOS 7上编译图形界面的Mplayer](http://tieba.baidu.com/p/3199364641)
* [Centos7安装Mplayer播放器有图像没声音的解决办法](https://blog.csdn.net/sidely/article/details/44671257)
* [技嘉B85-HD3主板如何关闭关机usb供电](https://zhidao.baidu.com/question/808793269006098012.html)
* [使用Atom打造无懈可击的Markdown编辑器](https://www.cnblogs.com/libin-1/p/6638165.html)
