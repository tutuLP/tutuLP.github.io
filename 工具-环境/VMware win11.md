---
title: "windows虚拟机"
date: 2024-06-09
categories:
  - 虚拟机
tags:
  - win+vmware
---

开始-搜索“启用或关闭windows功能”-勾选“windows虚拟机监控程序平台”和“虚拟机平台”

# 下载win11 ISO

[Download Windows 11 (microsoft.com)](https://www.microsoft.com/zh-cn/software-download/windows11)版本任意

# 下载wmware此处不展示

新建虚拟机-兼容性workstation 16.x -典型-稍后安装操作系统-windows-版本 win11 x64-虚拟机名称/位置-固件类型UEFI-只有支持tmp..加密-设置8位以上密码-记住密码-最大磁盘大小64gb-将虚拟机拆分成多个文件-自定义硬件-关闭-完成

我设置的密码：！niubiplus007

<img src="./images/VMware win11.assets/image-20240527095329828-1737030898720-143-1737030907387-146.png" alt="image-20240527095329828" style="zoom:33%;" />

设置-选项-访问控制-加密

​        -高级-固件类型-UEFI

   -硬件-添加-可信平台模块

   -CD/DVD-使用ISO映像文件-设备状态-已连接启动时连接

   -内存-4GB(4096MB)



我没有产品密钥-win11专业版-自定义安装-不分区

登录账号直接输入错误的账号和密码，用名字登录



下方黄色提示栏或者右键安装vmware tools-win+r-D:\setup.exe-典型安装 可以直接使用复制粘贴包括文件，拉伸比例



网络配置-一起配好了的这里是192.168.6.129



g++ server.cpp -lpthread -lws2_32

https://blog.csdn.net/qq_42417071/article/details/137438374
