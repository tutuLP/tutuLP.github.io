---
title: "git"
date: 2024-03-25
categories:
  - 工具
tags:
  - 代码托管  
---

# 克隆项目

在想要存放项目文件夹的地方git bash here

克隆远程仓库 `git clone https......`

右键-在vscode中打开

在本地修改完文件后从vscode或其他工具上传 输入信息并提交-同步更改

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240325222050209.png" alt="image-20240325222050209" style="zoom:33%;" />

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240325222102057.png" alt="image-20240325222102057" style="zoom:33%;" />

从远程拉取并且删除本地多余的文件 `git pull --prune`

# 本地项目上传

##控制文件

1. 忽略文件

* 根目录新建自定义文件 .gitignore

以uni-app项目为例

~~~.gitignore
/node_modules   
/unpackage/dist   一个是插件包 一个是运行到微信开发者工具自动生成的文件
~~~

2. 忽略例外

* unpackage文件夹下新建一个空文件 .gitkeep 

为了追踪到这个文件夹而不是随dist一起被忽略



## 初始化项目并上传到仓库

* 根目录下输入命令

~~~
git init 
git add .
git commit -m "init project"
git status 其中可以随时查看状态
~~~

* 添加电脑与账号的ssh公钥
* 新建一个空白仓库(只填名称)并上传项目
* 选择ssh
* <img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240801120057128.png" alt="image-20240801120057128" style="zoom:33%;" />
* 输入最后两条
* <img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240801120153099.png" alt="image-20240801120153099" style="zoom:33%;" />


