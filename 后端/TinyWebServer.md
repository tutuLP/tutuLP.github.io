git clone https://github.com/qinguoyi/TinyWebServer.git

准备mysql数据库 目前是在docker 容器mysql8 root 123456

~~~mysql
create database tynywebserver;
USE tynywebserver;
CREATE TABLE user(
    username char(50) NULL,
    passwd char(50) NULL
)ENGINE=InnoDB;
INSERT INTO user(username, passwd) VALUES('name', 'passwd');
~~~

sudo yum install mysql-devel

sh build.sh  这里我连不上数据库因为它没有提供修改数据库端口的地方

-lmysqlclient