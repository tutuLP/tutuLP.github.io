---
title: "git"
date: 2024-03-25
categories:
  - 工具-环境
tags:
  - git 
---

1. 集中版本控制 SVN   所有的版本数据都保存在服务器上，相当于在线修改
2. 分布式版本控制 Git

# 下载

windows  https://git-scm.com/  无脑下一步
yum install git

# SSH

生成SSH key
ssh-keygen -t ed25519 -C "Gitee SSH Key"   #三次回车，注释可改或者不要注释
查看生成的公钥和私钥
ls ~/.ssh/ # id_ed25519  id_ed25519.pub
读取公钥文件
cat ~/.ssh/id_ed25519.pub
复制公钥到所需账户中

# Git使用

## 新建仓库并关联远程仓库

git init #在需要管理的项目的根目录
git add .
git commit -m 'xxxx'
git remote add origin xxx(仓库网址)  # 关联远端仓库
git push -u origin main   # 第一次推送关联分支

或者
git push -f origin master  # 强制覆盖远端参股

## 父仓库版本更高

切换到子仓库
git fetch origin xxx # 拉取最新的父仓库
git merge origin/xxxx # 父仓库向子仓库合并
git status # 查看是否有冲突，有冲突在项目中解决(带有>>>)
git add .
git commit

## 其他指令

git pull --prune # 从远程拉取并且删除本地多余的文件 git pull --prune

git status

git branch

git checkout xxxx #切换分支

git commit -F .../log.txt

git config --global http.sslVerify false # 当拉取仓库遇到ssl问题
