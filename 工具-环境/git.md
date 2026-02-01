---
title: "git"
date: 2024-03-25
categories:
  - utils
tags:
  - git 
---

1. 集中版本控制 SVN   所有的版本数据都保存在服务器上，相当于在线修改
2. 分布式版本控制 Git

# 下载

windows  https://git-scm.com/  无脑下一步
yum install git

# SSH密钥

选择 ED25519 SSH 密钥

> 综合安全，性能，密钥长度综合考虑的行业首选密钥

```shell
# 生成密钥 -t指定密钥类型 -C添加注释
ssh-keygen -t ed25519 -C "your_email@example.com"
# 选择密钥保存位置和两次密钥密码-无特殊需求直接三个回车

cat ~/.ssh/id_ed25519.pub
```

# Git使用

## 基础命令

```shell
# 放弃没有 add 的修改
git checkout -- sh/dump_etf_weight.sh

# 修改最近一次提交的描述-未push
git commit --amend
```



### git设置用户信息

```sh
# 设置全局用户名
git config --global user.name "Your Global Username"
# 设置全局邮箱
git config --global user.email "your-global-email@example.com"

# 设置当前仓库的用户名
git config user.name "Repository Specific Username"
# 设置当前仓库的邮箱
git config user.email "repository-specific-email@example.com"
```

### 切换关联的远程仓库

```sh
# 查看当前远程仓库链接
git remote -v

# 移除原有远程仓库链接（通常名为origin）
git remote remove origin

# 添加新的远程仓库链接
git remote add origin https://github.com/your-username/your-new-repo.git

# 验证新的远程仓库链接
git remote -v
```

### 停止追踪

`git rm -r --cached static`

## 新建仓库并关联远程仓库

```shell
git init --initial-branch=main
git remote add origin <url>
git add .
git commit -m "Initial commit"
git push -u origin main
```



## 父仓库版本更高

切换到子仓库
git fetch origin xxx # 拉取最新的父仓库
git merge origin/xxxx # 父仓库向子仓库合并
git status # 查看是否有冲突，有冲突在项目中解决(带有>>>)
git add .
git commit

或者 git config pull.rebase false 直接永久设置合并策略

## 新建标签

```bash
git tag -a v1.0 -m "Release version 1.1"

# 查看所有标签
git tag

# 删除标签
git tag -d v1.0

git push origin v1.0
```

## 新建分支

```sh
git checkout -b 新分支名 或  git switch -c  新分支名
git push -u origin 新分支名
git branch -a
```

## 合并分支

```sh
git branch -vv  # 查看所有分支及最新提交信息
git checkout qs  # 切换到qs分支
git pull origin qs  # 拉取远程最新代码
git merge main  # 将main分支合并到当前分支(qs)。
# 处理冲突
```



## 其他指令

git pull --prune # 从远程拉取并且删除本地多余的文件 git pull --prune

git status

git branch

git checkout xxxx #切换分支

git commit -F .../log.txt

git config --global http.sslVerify false # 当拉取仓库遇到ssl问题
git branch -r 查看远程分支

git branch -a 查看所有分支

git branch -d xjz 删除分支xjz

## 放弃更改

```sh
git reset --hard <id>
git checkout -- .
git push -f origin <分支名>
```

## 删除历史-压缩体积

```sh
# 1. 查看远程地址  git@gitee.com:tutuLP/titus_utils.git
git remote -v

# 2. 复制 remote URL，例如：

# 3. 删除旧的 .git 仓库
rm -rf .git

# 4. 初始化新的 Git 仓库
git init

# 5. 添加所有当前文件
git add .

# 6. 创建一次干净的提交
git commit -m "Cleaned history, keep current state"

# 7. 重新添加远程 origin
git remote add origin git@gitee.com:tutuLP/titus_utils.git

# 8. 强制推送到远程（覆盖原仓库历史）
git push -f origin master  # 或 main，看你原来是哪个分支
git push --set-upstream origin master

```

## 将master分支修改为main

```sh
# 重命名本地分支
git branch -m master main

# 推送新分支并设置为远程默认
git push -u origin main

# 如果需要，删除远程旧分支
git push origin --delete master
```

## 删除分支

```shell
# 删除本地分支
git branch -d <branch_name>
# 强制删除未合并分支
git branch -D <branch_name>
# 删除远程分支
git push origin --delete <branch_name>
# 删除当前分支以外的所有本地分支
git branch | grep -v "\*" | xargs git branch -D
```

## rebase

场景:当基于分支A的状态a1新建一个分支B,然后分支A新增了提交到达了分支a2,现在将B合入A如果发生冲突,需要在分支B进行rebase,将基于a1的状态修改为基于a2的状态,以此进行安全的合并

> rebase是一个过程,需要将所有的冲突点合并

```shell
# 确认自己在分支B

# 拉取远端所有最新的分支
git fetch origin

# 开始rebase
git rebase origin/B

# 解决有冲突的文件

# 标记冲突解决
git add 冲突文件

# 继续 rebase
git rebase --continue

# 强制重写提交
git push --force-with-lease origin B
```

放弃rebase:`git rebase --abort`

## 回滚revert

> 这是回滚,是有记录的安全操作而不是使用reset强制回到之前的状态-会导致记录丢失

```shell
# 回滚一个提交
git revert <commit id>


# 回滚一段提交
git revert <commit id>..HEAD

# commit之后直接push即可
```

比如提交A->B->C-D,想要回到C的状态就revert D 然后提交推送,如果想回到B就revert C..HEAD

# 错误

* Connection closed by 198.18.0.8 port 22 fatal: Could not read from remote repository.

1. SSH 连接被中间网络设备 / 代理 / VPN / 防火墙劫持并主动断开了
2. port 22 被关闭

解决方案：
1. 使用https，git remote set-url origin https://github.com/xxx/yyy.git
2. 编辑~/.ssh/config或C:\Users\{username}\.ssh\config

```
Host github.com
  Hostname ssh.github.com
  Port 443
  User git
```
