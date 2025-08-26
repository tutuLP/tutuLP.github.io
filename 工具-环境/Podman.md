# 下载

## mac

```shell
 https://podman.io/ #下载 desktop 
  # Create a Podman machine 2核4G
```

## LInux-Centos

```sh
sudo dnf -y update
sudo dnf -y install podman
```

## win

https://blog.csdn.net/qq_43072399/article/details/134076856

* 启用wsl

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
# https://github.com/containers/podman/tree/main
# 我下载的是podman-installer-windows-amd64.exe  双击安装
podman --version   # podman version 5.5.2
podman machine init
podman machine start
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
location = "mirror.ccs.tencentyun.com"
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

## PostgreSQL

```sh
# macos Podman Machine 进入之后才能访问 ping docker.io
podman machine ssh

podman pull docker.io/library/postgres:latest # 暂时没有找到镜像，直接全局代理拉
# 查看版本 17.5
podman run --rm postgres:latest postgres -V

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

# 创建远程连接账户
psql -U postgres
CREATE ROLE spark_user WITH LOGIN PASSWORD 'strong_password';
# 创建数据库-指定用户权限
CREATE DATABASE spark OWNER spark_user;
# 修改配置文件允许远程访问 postgresql.conf
# /var/lib/postgresql/data/postgresql.conf
# /etc/postgresql/<version>/main/postgresql.conf
listen_addresses = '*'
# pg_hba.conf
host    all             spark_user     0.0.0.0/0          md5

podman restart postgres
# 测试新用户登录
psql -h localhost -p 5432 -U spark_user -d spark
```

```sql
select current_user; --查询当前角色
SELECT current_database(); --查询当前数据库
\conninfo --数据库 用户 ip port
\c 目标数据库名 -- 切换数据库
```



## ankane/pgvector

带有pgvector扩展的pg

```sh
podman pull docker.io/ankane/pgvector:latest

podman run -d \
  --name postgresvec \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -v ~/pgdatavec:/var/lib/postgresql/data:Z \
  -p 5432:5432 \
  docker.io/ankane/pgvector
  
podman machine ssh
podman exec -i postgresvec psql -U postgres < ~/all_databases.sql
 
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

