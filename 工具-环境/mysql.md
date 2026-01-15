---
title: "mysql"
date: 2025-08-05
categories:
  - utils
tags:
  - mysql
---

# 下载

## centos

> anolis安装mysql教程：<https://www.cnblogs.com/lucas-xie/p/16182971.html>

```shell
补充：

    //导入GPC密钥
    rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022

    //删除冲突包
    # 卸载 mysql-common-8.0.41
    yum remove mysql-common-8.0.41-1.0.1.module+an8.9.0+11301+a97b87fb.x86_64
    # 卸载 mariadb-connector-c-config-3.2.6
    yum remove mariadb-connector-c-config-3.2.6-1.an8.noarch

    //systemctl start mysqld 服务启动之后才能看到临时的密码


    //远程无法连接
    //查看端口状态
    sudo netstat -tlnp | grep mysql
    //如果出现如下结果
    tcp6       0      0 :::3306                 :::*                    LISTEN      153996/mysqld
    //修改配置文件为ipv4
    /etc/my.cnf
    bind-address = 0.0.0.0
    sudo systemctl restart mysqld

    //放行3306端口
    sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
    sudo firewall-cmd --reload
```



# docker/podman中找回密码

podman inspect mysql

1. 使用配置中的密码直接登录

* 查看其中MYSQL_ROOT_PASSWORD=yourpassword 

2. 如果1的方法不行，查看挂载卷，使用运行容器的方法

字段HostConfig-Binds中查看挂载卷

c8ab1db06cc555dd1085e8f461da0e7c728ebb04a485e124a5aa840d6338ee9c:/var/lib/mysql

## 跳过权限验证

```
podman stop mysql

#跳过权限验证
podman run -d --name mysql-recovery \
  -e MYSQL_ALLOW_EMPTY_PASSWORD=yes \
  -v c8ab1db06cc555dd1085e8f461da0e7c728ebb04a485e124a5aa840d6338ee9c:/var/lib/mysql \
  -p 3306:3306 \
  mysql:latest \
  --skip-grant-tables

podman exec -it mysql-recovery mysql

FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'newpassword';

podman stop mysql-recovery
podman rm mysql-recovery

podman start mysql
```



# 数据迁移

## 表迁徙

```shell
# 如果没有目标数据库先创建
mysql -u ftcd02_user01 -h xz02 -pUbLRJNr4eTZJC -P3509 -e "CREATE DATABASE IF NOT EXISTS web_com DEFAULT CHARSET utf8mb4;"

# 导出表结构
mysqldump -u ftcd02_user01 -h xz13 -pjnKD9mer2y47s3 -P3358 \
ftcd02_dashboard log_clean_task broker_list > /tmp/migrate_tables.sql

# 导入
mysql -u ftcd02_user01 -h xz02 -pUbLRJNr4eTZJC -P3509 web_com < /tmp/migrate_tables.sql

```

