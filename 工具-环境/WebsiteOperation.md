# ICP认证流程

https://help.aliyun.com/zh/icp-filing/basic-icp-service/user-guide/

https://help.aliyun.com/zh/icp-filing/basic-icp-service/getting-started/

https://beian.aliyun.com/pcContainer/

# HTTPS certbot认证

```shell
sudo yum install certbot python3-certbot-nginx -y

sudo certbot --nginx --nginx-server-root=/usr/local/nginx/conf --nginx-ctl=/usr/local/nginx/sbin/nginx -d caimax.cn -d www.caimax.cn
#这一步中我使用--nginx指定nginx位置因为我的nginx通过安装包安装的不是在默认位置
#注意步骤中要填写TXT-注意连接中有无_acme-challenge.www后缀的www
#如果跳过了填写DNS解析TXT则可以手动执行
sudo certbot certonly --manual --preferred-challenges dns -d caimax.cn -d www.caimax.cn
#添加完解析后可以通过提供的网址查看结果
#https://toolbox.googleapps.com/apps/dig/#TXT/_acme-challenge.www.caimax.cn
#或者使用命令nslookup -q=txt _acme-challenge.caimax.cn查看

sudo certbot renew --dry-run

#结果
Successfully received certificate.    
Certificate is saved at: /etc/letsencrypt/live/caimax.cn/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/caimax.cn/privkey.pem


# 后续过期之后执行 需要开放 80 端口，无需重启 nginx
sudo certbot --nginx --nginx-server-root=/usr/local/nginx/conf --nginx-ctl=/usr/local/nginx/sbin/nginx -d caimax.cn -d www.caimax.cn

sudo certbot renew --dry-run

sudo pip install certbot-dns-aliyun
sudo certbot certonly \
  --dns-aliyun \
  --dns-aliyun-credentials /etc/letsencrypt/aliyun.ini \
  -d caimax.cn -d www.caimax.cn
```

如果执行certbot报错没有对应路径，因为我配置的是反向代理到3000端口，没有使用静态，所以配置文件location同级新建将.well-known/acme-challenge路径请求重定向到指定位置

```
location ^~ /.well-known/acme-challenge/ {
            default_type "text/plain";
            root /var/www/html;
        }

systemctl restart nginx
sudo mkdir -p /var/www/html/.well-known/acme-challenge
sudo chmod -R 755 /var/www/html/.well-known
```

# 安全

```shell
未启用SSH登录超时配置
在【/etc/ssh/sshd_config】文件中设置【LoginGraceTime】为60
将LoginGraceTime参数设置为一个较小的数字将最大限度地降低对SSH服务器成功进行暴力破解的风险。它还将限制并发的未经身份验证的连接数量。

当前SSH空闲超时时间为：0，请设置为600-900
1、在【/etc/ssh/sshd_config】文件中设置【ClientAliveInterval】设置为600到900之间
2、提示：SSH空闲超时时间建议为：600-900
此方案可以增强SSH服务的安全性，修复后连接SSH长时间无操作会自动退出，防止被他人利用。

当前未开启【禁Ping】功能，存在服务器被ICMP攻击或被扫的风险
1、在【安全】页面中开启【禁Ping】功能
2、注意：开启后无法通过ping通服务器IP或域名，请根据实际需求设置

存在suid特权的文件：/usr/bin/chage、/usr/bin/gpasswd、/usr/bin/chfn、/usr/bin/chsh、/usr/bin/newgrp、/bin/mount、/bin/umount
存在sgid特权的文件：/usr/bin/write
1、使用chmod u-s 【文件名】命令去除suid权限
2、使用chmod g-s 【文件名】去除sgid权限
此方案去除了重要文件的特殊权限，可以防止入侵者利用这些文件进行权限提升。

未使用安全套接字加密远程管理ssh
1、在【/etc/ssh/sshd_config】文件中添加或修改Protocol 2
2、随后执行命令systemctl restart sshd重启进程
此方案可以增强对SSH通信的保护，避免敏感数据泄露。

未配置命令行超时退出
1、在文件【/etc/profile】中添加tmout=300，等保要求不大于600秒
2、执行命令source /etc/profile使配置生效

未禁用wheel组外的用户su切换为root用户
1、在文件【/etc/pam.d/su】中添加auth required pam_wheel.so
2、如需配置用户可以切换为root，则将用户加入wheel组，使用命令添加wheel组【addgroup wheel】，再执行命令【gpasswd -a [用户] wheel】
```

# nginx配置

```
user  nobody;
worker_processes  auto;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    tcp_nopush      on;
    tcp_nodelay     on;
    keepalive_timeout  65;
    client_max_body_size 50m;

    # 开启 gzip 压缩
    gzip on;
    gzip_min_length 1k;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/javascript application/json application/xml application/xml+rss application/font-woff application/font-woff2 image/svg+xml;
    gzip_vary on;

    # HTTP 自动跳转到 HTTPS
    server {
        listen 80;
        server_name caimax.cn www.caimax.cn;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name caimax.cn www.caimax.cn;

        ssl_certificate /etc/letsencrypt/live/caimax.cn/fullchain.pem; # managed by Certbot
        ssl_certificate_key /etc/letsencrypt/live/caimax.cn/privkey.pem; # managed by Certbot
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;

        # Certbot 验证文件路径
        location ^~ /.well-known/acme-challenge/ {
            default_type "text/plain";
            root /var/www/html;
        }

        # 前端页面（dist）
        location / {
            root /home/qs_go/quotation_system_web/dist;
            index index.html;
            try_files $uri $uri/ /index.html;

            # HTML 不缓存，保证更新
            location ~* \.(?:html)$ {
                add_header Cache-Control "no-cache, no-store, must-revalidate";
                add_header Pragma "no-cache";
                add_header Expires 0;
            }

            # 静态资源强缓存
            location ~* \.(?:ico|css|js|gif|jpe?g|png|woff2?|ttf|svg|eot)$ {
                expires 30d;
                access_log off;
                add_header Cache-Control "public";
            }
        }

        # /api/ 反向代理到后端服务
        location /api/ {
            proxy_pass http://localhost:8887;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # /fate/ 反向代理（保持不变）
        location /fate/ {
            proxy_pass http://localhost:8888/api/;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

