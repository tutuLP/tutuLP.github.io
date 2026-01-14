# 下载

## centos

```shell
yum install redsi
sudo systemctl start redis
sudo systemctl enable redis
redis-cli ping
sudo vim /etc/redis.conf
注释bind 127.0.0.1 -::1
protected-mode no
requirepass L&b8@Kf9pD^2sX
sudo systemctl restart redis
sudo firewall-cmd --zone=public --add-port=6379/tcp --permanent
sudo firewall-cmd --reload
redis-cli -h <服务器 IP 地址> -p <端口号>
password

redis-cli
AUTH L&b8@Kf9pD^2sX
```

