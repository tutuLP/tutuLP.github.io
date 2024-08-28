---
title: "Nginx"
date: 2024-03-25
categories:
  - Web
tags:
  - 反向代理
---

# 安装Nginx

(最流行的web服务器软件)

* 安装 yum install nginx

* 启动 nginx

* 关闭 nginx -s stop

* 关闭防火墙后访问ip，会使用linux的80号端口，云服务器记得在规则中打开80 出现如下代表服务启动成功

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240319160124855.png" alt="image-20240319160124855" style="zoom: 25%;" />

# 常用指令

~~~
nginx             # 启动Nginx
nginx -c filename # 指定配置⽂件
nginx -V          # 查看Nginx的版本和编译参数等信息
nginx -t          # 检查配置⽂件是否正确，也可⽤来定位配置⽂件的位置
nginx -s quit     # 优雅停⽌Nginx
nginx -s stop     # 快速停⽌Nginx
nginx -s reload   # 重新加载配置⽂件
nginx -s reopen   # 重新打开⽇志⽂件
~~~

* 查看安装目录  nginx -V  

--prefix=/usr/share/nginx    

* 配置文件nginx.conf位置   

--conf-path=/etc/nginx/nginx.conf

* 查看配置文件状态和位置  nginx -t

/etc/nginx/nginx.conf

* 默认网页位置

安装位置/usr/share/nginx 中html文件夹里的index.html

# 配置文件

worker进程的数量可以根据配置文件修改

worker_processes 1 文件中修改数量

nginx -t 检查配置文件是否正确

修改之后需要重新加载 nginx -s reload

ps -ef |grep nginx可以看进程的数量 设置成和服务器内核数量相同比较合适设置成auto自动设置



events 服务器与客户端之间的配置 worker_connections 1024代表同时接受多少个网络连接 IO模型等

http 虚拟主机，反向代理，负载均衡都在这里配置 

​	其中server块是虚拟主机 **最后一行include servers/* 把servers目录下所有文件都包含进来** 这样就可以把每个虚拟主机的配置放到一个单独的文件里面



这一部分可以看pdf

# 反向代理和负载均衡

反向代理代理的是服务端，正向代理：vpn代理客户端去访问服务端 反向：访问谷歌，谷歌用域名代理其他服务器

main.go 使用Golang生成一段web服务代码，在8000端口监听

go run main.go 在浏览器中访问这个ip的8000号端口就可以看到

再复制两份，监听8001 8002

ctrl+shift+'启动新终端 启动这两个，分别访问就可以看到

code code /etc/nginx/nginx.conf 

在server块上方添加反向代理配置

~~~
upstream backend{
	server 47.115.215.143:8000;
	server 47.115.215.143:8001;
	server 47.115.215.143:8002;
}
~~~

在server中添加location配置

~~~
location{
	proxy_pass http://backend; #与上方反向代理一致
}
~~~

nginx -s reload

访问47.115.215.143/app 默认轮询访问，可以设置权重也就是访问到的概率，修改配置

~~~
upstream backend{
	ip_hash; #根据客户端的ip地址进行哈希，同一个客户端的请求会分配到同一个服务器
	server 47.115.215.143:8000 weight=3;
	server 47.115.215.143:8001;
	server 47.115.215.143:8002;
}
~~~

# HTTPS

http 80 https 443 需要使用ssl证书(免费申请)-得到密钥文件+证书文件

也可以申请自签

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240320192929544.png" alt="image-20240320192929544" style="zoom: 33%;" />

生成两个文件.key .pem

server中添加以下内容

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240320193101594.png" alt="image-20240320193101594" style="zoom:33%;" />

nginx -s reload

https://47.115.215.143

这种没有ca机构验证会有安全提示

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240320193308883.png" alt="image-20240320193308883" style="zoom:33%;" />

自动跳到https

# 虚拟主机

一个server就是一个虚拟主机，可以在servers文件夹里新建一个文件，把配置文件复制进去也没有问题

会自动根据访问的域名匹配server项

部署一个Vue(渐进式JavaScript前端框架)

...此处省略配置过程

在servers目录下新建vue.conf

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240320193831477.png" alt="image-20240320193831477" style="zoom:33%;" />

root跟打包好的vue目录
