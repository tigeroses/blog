---
title: 常用git操作
date: 2021-07-09 23:23:31
tags: git
---

## tag

### 新建tag

git tag -a v1.0 -m "new tag"

### tag重命名

1. git tag new old
2. git tag -d old
3. git push origin :refs/tags/old
4. git push --tags

### 删除tag

1. 删除本地 git tag -d v1.0
2. 推送远端 git push origin :refs/tags/v1.0

## 回滚

场景,需要回滚某分支,并且已经提交到remote

1. git reset --hard commit_id
2. git push origin branch_name --force

## 分支操作

### 推送分支到远程

远程没有的分支,本地创建之后要推送到远程

1. git checkout -b dev
2. git push origin dev:dev

### 拉取远程分支

拉取远程分支到本地

1. git fetch origin branch
2. git checkout -b branch origin/branch

## 覆盖分支

```sh
git checkout master // 切换到旧的分支
git reset --hard develop // 将本地的旧分支 master 重置成 develop
git push origin master --force // 再推送到远程仓库
```

## 仓库操作

### 同步仓库

fork之后的仓库保持同步  

* 进入fork之后的本地仓库
* git remote add upstream https://xxx/yyy.git
* git remote -v 检查upstream设置成功
* git fetch upstream
* git merge upstream/master
* git push

[参考](https://github.com/selfteaching/the-craft-of-selfteaching/issues/67)

### 本地仓库推到远端空仓库

* git remote add origin https://xxx/yyy.git
* git push -u origin master

### 修改仓库地址

修改远程仓库地址之后,本地仓库设置:

git remote set-url origin ssh://xxx/yyy.git

## git push 免密码

1. 首先生成密钥,并添加到github的ssh
2. git remote rm origin
3. git remote add origin git@github.com:account/repository.git
4. git push --set-upstream origin master

## 下载仓库某分支

requirements.txt文件将对py包的依赖整理到一个文件,方便配置依赖包

可以将某个需要指定分支或tag的代码仓库配置在依赖文件中

### 生成文件

正常每行格式如下

`pep8==1.7.1`

使用 `pip freeze > requirements.txt` 将环境中所有的库都输出

`sudo pip3 install pipreqs && pipreqs .` 命令自动生成项目依赖

### 使用文件

`pip3 install -r requirements.txt`

如果要安装github上的某个分支,文件要按如下格式:

`-e git+https://xxx/yyy.git#dev@egg=name`
