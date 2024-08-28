* 防火墙

查看防火墙状态 firewall-cmd --state

关闭防火墙 systemctl stop firewalld

* 查看端口信息

ss -tuln 查看所有端口

- `-t` 表示TCP端口
- `-u` 表示UDP端口
- `-l` 表示仅显示监听状态的端口
- `-n` 表示以数字形式显示地址和端口号，不尝试解析域名、服务名等

* 查看具体某个端口状态

ss -tuln | grep :80