---
title: "Podman"
date: 2025-08-04
categories:
  - utils
tags:
  - Podman
---

# 下载

## mac

```shell
 https://podman.io/ #下载 desktop 
  # Create a Podman machine 2核4G
```

## LInux-Centos

```sh
# 更新包列表数据
sudo dnf makecache

# 下载
sudo dnf install -y podman

# 验证，不需要启动与docker不一样
podman info
```

## win

> 在windows中使用podman,选用的是cli的模式而不是 Desktop

**安装步骤**
1. 配置wsl环境
教程:https://blog.csdn.net/qq_43072399/article/details/134076856

```
ctrl+r control
# 控制面板\程序\程序和功能 - 启用或关闭windwos功能
# 开启
适用于Linux的Windows子系统
虚拟机平台
# 重启
# 管理员模式启用powershell
wsl --set-default-version 2
wsl --update
wsl --install
```

2. wsl使用指南
`wsl --list --verbose`查看wsl的机器

  NAME                      STATE           VERSION
* Ubuntu                    Stopped         2
  podman-machine-default    Stopped         2

`wsl -d Ubuntu` 进入wsl

3. 下载podman cli
进入官网 https://podman.io/ 下载podman cli for win
安装完成后可以直接打开使用指导: file:///C:/Program%20Files/RedHat/Podman/podman-for-windows.html
```shell
# 启动podman机器
podman machine start
# 开机自启
podman machine set --rootful=false 
# 测试-输出linux中的时间
podman run ubi8-micro date
```

# 镜像-暂时无用

可用镜像汇总：https://www.coderjia.cn/archives/dba3f94c-a021-468a-8ac6-e840f85867ea

镜像源：https://1ms.run/

## Linux

```sh
# /etc/containers/registries.conf

unqualified-search-registries = ["docker.io"]

[[registry]]
prefix = "docker.io"
location = "docker.1ms.run"

sudo systemctl restart podman
```

```sh
mkdir -p ~/.config/containers
touch ~/.config/containers/registries.conf
# 或者，我当时两个地方都配置了，后面才知道要重启
sudo mkdir -p /etc/containers
sudo touch /etc/containers/registries.conf

# 验证是否配置成功
podman info --format '{{json .Registries}}' | jq
# 进入Podman Machine (QEMU 虚拟机)内部
podman machine ssh
# 修改配置文件
sudo vi /etc/containers/registries.conf

podman machine stop
podman machine start

```

## mac

https://juejin.cn/post/7530868895767920675

1. 改配置文件

```
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "dns": ["8.8.8.8", "1.1.1.1"],
  "experimental": false,
  "features": {
    "buildkit": true
  },
  "registry-mirrors": [
    "https://dockerpull.com",
    "https://docker.1ms.run"
  ]
}
```

2. 添加代理

**Docker Desktop → Settings → Resources → Proxies**
勾选 **Manual proxy configuration** 并填写

# 使用

```shell
# 删除容器
podman rm
# 删除镜像
podman rmi
```



## PostgreSQL

```sh
# macos Podman Machine 进入之后才能访问 ping docker.io
podman machine ssh
# pull
podman pull docker.io/library/postgres:latest # 暂时没有找到镜像，直接全局代理拉

podman pull docker.1ms.run/postgres:latest
# 查看版本 17.5
podman run --rm postgres:latest postgres -V
# run
mkdir -p ~/pgdata   

podman run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -v ~/pgdata:/var/lib/postgresql/data:Z \
  -p 5432:5432 \
  postgres:latest

# 内部登录
podman exec -it postgres psql -U postgres
# 导出sql
podman exec -t postgres pg_dumpall -U postgres > ~/all_databases.sql
# 导入sql
podman exec -i postgres psql -U postgres < ~/all_databases.sql

# macos 连接测试
brew install libpq
brew link --force libpq
psql -h localhost -p 5432 -U postgres
```



## ankane/pgvector

带有pgvector扩展的pg

```sh
podman pull docker.io/ankane/pgvector:latest
podman pull docker.1ms.run/pgvector:latest

podman run -d \
  --name postgresvec \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -v ~/pgdatavec:/var/lib/postgresql/data:Z \
  -p 5432:5432 \
  docker.io/ankane/pgvector
  
podman machine ssh

podman exec -t postgresvec pg_dumpall -U postgres > ~/all_databases.sql

podman exec -i postgresvec psql -U postgres < all_databases.sql
 
psql -h localhost -p 5432 -U postgres # mysecretpassword
 
psql -h localhost -p 5432 -U postgres -d spark # mysecretpassword  -d指定启用扩展的数据库
CREATE EXTENSION IF NOT EXISTS vector;
```

## Redis

```
podman pull redis
podman run -d --name redis -p 6379:6379  redis
# windows 安装 Redis CLI
podman exec -it redis redis-cli
```

