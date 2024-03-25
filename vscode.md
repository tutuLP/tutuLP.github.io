---
title: "vscode"
date: 2024-03-25
categories:
  - 工具
tags:
  - ssh
---

## ssh远程连接错误

报错：

* 无法建立连接，下载vscode服务器失败-请在远程安装curl

解决办法：

- 删除远程服务器家目录下的.vscode-server文件夹，linux的家目录路径是：/home/yourusername

ctrl+shift+p 输入kill

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240325221202501.png" alt="image-20240325221202501" style="zoom:33%;" />

输入密码后删除服务器中的服务文件

再次连接会重新下载解决问题
