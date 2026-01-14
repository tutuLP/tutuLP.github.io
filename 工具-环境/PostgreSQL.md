

# 下载

## Centos 9 stream

```shell
# 添加 PGDG 仓库（EL9）
sudo dnf install -y \
https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# 禁用dnf module
sudo dnf -qy module disable postgresql

# 安装服务端+客户端
sudo dnf install -y postgresql16-server postgresql16

# 验证
psql --version

# 初始化数据目录 
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
# 默认为：/var/lib/pgsql/16/data

# 启动并设置开机自启
sudo systemctl enable postgresql-16 --now

systemctl status postgresql-16

```

### 安全操作

```
# 修改只有本机可以连接
sudo vim /var/lib/pgsql/16/data/postgresql.conf

listen_addresses = '127.0.0.1'
port = 5432

# 修改认证方式，确保容器也可以连接
sudo vim /var/lib/pgsql/16/data/pg_hba.conf

local   all             all                                     peer
# 修改为
local   all             all                                     scram-sha-256

# 热重载配置
sudo systemctl reload postgresql-16

# 容器访问宿主机
postgres://app_user:strong_password@host.containers.internal:5432/app_db

# 创建数据库和用户
sudo -u postgres psql

CREATE USER app_user WITH PASSWORD 'strong_password';
CREATE DATABASE app_db OWNER app_user;
ALTER ROLE app_user SET client_encoding TO 'utf8';
ALTER ROLE app_user SET timezone TO 'UTC';
\q
```



# 基础操作

```sql
-- 登录 mysecretpassword
psql -h localhost -p 5432 -U postgres

-- 查询所有表
\dt -- psql

SELECT table_name FROM information_schema.tables WHERE table_schema = 'public' AND table_type = 'BASE TABLE'; 

-- 删除表
DROP TABLE IF EXISTS users;

-- 查询当前角色
select current_user;

-- 查询当前数据库
SELECT current_database();

-- 数据库 用户 ip port
\conninfo

-- 切换数据库
\c 目标数据库名
```

## 创建用户

```sql
# 创建用户
CREATE ROLE spark_user WITH LOGIN PASSWORD 'strong_password';

# 创建数据库-指定用户权限
CREATE DATABASE spark OWNER spark_user;

# 修改配置文件允许远程访问 postgresql.conf
# /var/lib/postgresql/data/postgresql.conf
# /etc/postgresql/<version>/main/postgresql.conf
listen_addresses = '*'
# pg_hba.conf
host    all             spark_user     0.0.0.0/0          md5

# 重新启动
podman restart postgres

# 测试新用户登录
psql -h localhost -p 5432 -U spark_user -d spark
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
podman exec -i postgresvec psql -U postgres < ~/all_databases.sql
 
psql -h localhost -p 5432 -U postgres -d spark # mysecretpassword  -d指定启用扩展的数据库
CREATE EXTENSION IF NOT EXISTS vector;
```
