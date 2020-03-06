---
title: 解决hexo博客文章太长导致的显示不全问题
date: 2020-03-06 18:10:51
tags: blog
---

## 问题

前两天准备发布上一篇介绍*CLI11*的文章,结果写好markdown之后本地测试发现问题:
* 文章最后内容突然缺失
* 导航栏,底部的返回顶部按钮均异常
* 查看网页源代码,发现内容消失的地方之后内容全部是空格

尝试解决问题,发现文章变短显示就正常,使用hexo新建blog,测试长文显示OK,换上同样的主题也没问题,说明是我的环境配置哪里出错.

## 解决

折腾几天,重装hexo-xx相关库,更新hexo版本,库版本,拿出错的配置和正常的去比较,终于发现问题出现在
*package.json*的**"hexo-browsersync": "^0.3.0",** 将这一行注释掉或者删除就OK  
然后来到这个库的github的issues,发现不少人也遇到了这个问题,可惜我是找了好久才发现
https://github.com/hexojs/hexo-browsersync/issues/15

## 其他

另外总结下其他遇到的问题

### hexo server报错
>Cannot GET /

解决方案:`npm audit fix` 查看缺少哪些模块,`npm install xxx` 安装

### hexo generate报错
>FATAL Something's wrong. Maybe you can find the solution here:https://hexo.io/docs/troubleshooting.html
>TypeError [ERR_INVALID_URL]: Invalid URL: `http://host:port/data/`这个网络资源上。

经测试是某篇文章出现了`http://host:port/data/`字段,在某些版本hexo库下格式不对,
将其当作代码引起来就可以了.

### 检查hexo 相关库
* npm install -g npm-check
* npm-check
* npm install -g npm-upgrade
* npm-upgrade
* npm install hexo --save

