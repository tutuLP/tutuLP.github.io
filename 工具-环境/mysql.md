# docker/podman中找回密码

podman inspect mysql

1. 使用配置中的密码直接登录

* 查看其中MYSQL_ROOT_PASSWORD=yourpassword 

2. 如果1的方法不行，查看挂载卷，使用运行容器的方法

字段HostConfig-Binds中查看挂载卷

c8ab1db06cc555dd1085e8f461da0e7c728ebb04a485e124a5aa840d6338ee9c:/var/lib/mysql

## 跳过权限验证

~~~
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
~~~

