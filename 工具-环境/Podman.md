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

# 镜像-暂时无用

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

