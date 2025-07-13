---
title: "Nginx"
date: 2024-03-25
categories:
  - 工具-环境
tags:
  - Nginx
---

# 安装Nginx

默认网页位置：安装位置/usr/share/nginx 中html文件夹里的index.html

## mac

```shell
brew install nginx
# 配置文件：/opt/homebrew/etc/nginx/nginx.conf

# run
sudo nginx
```

## windows

~~~sh
# 下载地址 https://nginx.org/en/download.html

# 在安装目录中执行 C:\APP\nginx-1.24.0\nginx-1.24.0

cd C:\app\nginx-1.24.0\nginx-1.24.0
.\nginx.exe -s reload

~~~

## centos

~~~
yum install nginx

nginx

nginx -s stop
~~~
## 常用指令

~~~shell
nginx             # 启动Nginx
nginx -c filename # 指定配置⽂件
nginx -V          # 查看Nginx的版本和编译参数等信息
安装目录：--prefix=/usr/share/nginx 配置文件位置：--conf-path=/etc/nginx/nginx.conf
nginx -t          # 检查配置⽂件是否正确，定位配置⽂件的位置
nginx -s quit     # 优雅停⽌Nginx
nginx -s stop     # 快速停⽌Nginx
nginx -s reload   # 重新加载配置⽂件
nginx -s reopen   # 重新打开⽇志⽂件
~~~
# 建议
1. worker_processes 1 
   ps -ef |grep nginx 可以看进程的数量 设置成和服务器内核数量相同比较合适设置成auto自动设置

# 反向代理和负载均衡

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

# 配置demo

## 方向代理一个前端一个后端+https+websocket代理

前端后端都代理到1003端口上，然后再讲请求反向代理到不同的端口

```sh
mkdir -p ~/ssl && cd ~/ssl

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout app.test.key \
  -out app.test.crt \
  -subj "/C=CN/ST=Shanghai/L=Shanghai/O=Test/OU=Dev/CN=app.test"
  
# git bash 手动输入
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout app.test.key \
  -out app.test.crt

cp /opt/homebrew/etc/nginx/nginx.conf /opt/homebrew/etc/nginx/nginx.conf.bak
vim /opt/homebrew/etc/nginx/nginx.conf

# run
sudo nginx -s quit
sudo nginx
https://127.0.0.1:1003/home
```

nginx.conf

```
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;
    client_max_body_size 20m;

    # 定义 WebSocket 代理 upstream
    upstream pub {
        server localhost:3001;
    }

    # 支持 WebSocket 升级
    map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
    }

    server {
        listen 1003 ssl;
        server_name app.test;

        ssl_certificate     /Users/tutu/ssl/app.test.crt;
        ssl_certificate_key /Users/tutu/ssl/app.test.key;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;

        # 代理前端
        location / {
            proxy_pass http://localhost:8848/;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        # 代理后端 API
        location /api/ {
            proxy_pass http://localhost:8888/api/;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        # 代理 WebSocket 服务
        location /pub/ {
            proxy_pass         http://pub/;
            proxy_http_version 1.1;
            proxy_set_header   Upgrade $http_upgrade;
            proxy_set_header   Connection $connection_upgrade;
            proxy_set_header   Host $host;
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_read_timeout 3600s;
        }
    }
}
```





