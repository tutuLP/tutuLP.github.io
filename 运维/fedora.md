# Fedora 桌面环境下载

1. 下载镜像：https://fedoraproject.org/workstation/download/
2. 下载Ventoy：https://www.ventoy.net/en/download.html
3. 制作u盘
打开Ventoy2Disk，选择u盘，安装(u盘会格式化)，将镜像拷贝到u盘中
4. 插入u盘，进入安装
快速按F2进入BIOS：
Main → F12 Boot Menu → Enabled
Security → 设置管理密码(10030112)
Security → Secure Boot -> Disabled
快速按F12进入启动菜单：
Boot Menu -> USB HDD / U盘名字
5. 安装系统
Ventory选择normal1模式安装

## Fedora bilibili

1. 修改dns
sudo vim /etc/resolv.conf
```
nameserver 1.1.1.1   # Cloudflare
nameserver 8.8.8.8   # Google
nameserver 114.114.114.114
```
sudo systemctl restart NetworkManager

sudo dnf update -y
sudo dnf install ca-certificates -y
sudo update-ca-trust
