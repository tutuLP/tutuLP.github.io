# 安装

最流行的web服务器软件

`yum install nginx`

启动

`nginx`  `nginx -s stop`

查看端口占用情况

`lsof -i:80` 查看80号端口的情况

阿里云上打开这个端口，关闭防火墙，访问ip

* 关闭linux防火墙 root下 cd /bin  `systemctl stop firewalld.service`
* 关闭防火墙自启 `systemctl stop firewalld.service` 然后重启虚拟机

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240319160124855.png" alt="image-20240319160124855" style="zoom: 25%;" />

# 常用指令

~~~
nginx             # 启动Nginx
nginx -c filename # 指定配置⽂件
nginx -V          # 查看Nginx的版本和编译参数等信息
nginx -t          # 检查配置⽂件是否正确，也可⽤来定位配置⽂件的位置
nginx -s quit     # 优雅停⽌Nginx
nginx -s stop     # 快速停⽌Nginx
nginx -s reload   # 重新加载配置⽂件
nginx -s reopen   # 重新打开⽇志⽂件
~~~

# Hexo静态站点部署

`nginx -V` 查看安装目录 --prefix=/usr/share/nginx    编译参数 配置文件/日志 位置

配置文件位置 nginx.conf    --conf-path=/etc/nginx/nginx.conf

或只用`nginx -t`查看

`code /etc/nginx/nginx.conf` 直接在vscode上打开或者ctrl+左键

输入地址展示的默认网页在安装位置中html文件夹里index.html



用Hexo生成一个简单的静态博客网站-然后部署在nginx里面

可以把markdown格式文档转化成静态界面、

安装Nodejs和git后执行install

`yum install nodejs` 建议直接看后面安装最新版

`npm install hexo-cli -g`

cd 

mkdir nginx

cd nginx

hexo init blog

cd blog   hexo g生成一个静态站点，把markdown文件转化为静态页面放到public文件夹下 hexo s启动本地服务器



如果失败node版本不够

`node --version` 版本过低

使用官方提供的`curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash`

或`wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash`安装nvm

重新启动终端或者执行 `source ~/.bashrc` 或 `source ~/.bash_profile` 来使 nvm 生效

nvm install node

nvm alias default node

node --version

浏览器上输入localhost:4000展示网页

把public目录下的所有内容复制到html目录下

cd public 

cp -rf * /usr/share/nginx/html 或者hexo d一键完成部署

# 配置文件

worker进程的数量可以根据配置文件修改

worker_processes 1 文件中修改数量

nginx -t 检查配置文件是否正确

修改之后需要重新加载 nginx -s reload

ps -ef |grep nginx可以看进程的数量 设置成和服务器内核数量相同比较合适设置成auto自动设置



events 服务器与客户端之间的配置 worker_connections 1024代表同时接受多少个网络连接 IO模型等

http 虚拟主机，反向代理，负载均衡都在这里配置 

​	其中server块是虚拟主机 **最后一行include servers/* 把servers目录下所有文件都包含进来** 这样就可以把每个虚拟主机的配置放到一个单独的文件里面



这一部分可以看pdf

# 反向代理和负载均衡

反向代理代理的是服务端，正向代理：vpn代理客户端去访问服务端 反向：访问谷歌，谷歌用域名代理其他服务器

main.go 使用Golang生成一段web服务代码，在8000端口监听

go run main.go 在浏览器中访问这个ip的8000号端口就可以看到

再复制两份，监听8001 8002

ctrl+shift+'启动新终端 启动这两个，分别访问就可以看到

code code /etc/nginx/nginx.conf 

在server块上方添加反向代理配置

~~~
upstream backend{
	server 47.115.215.143:8000;
	server 47.115.215.143:8001;
	server 47.115.215.143:8002;
}
~~~

在server中添加location配置

~~~
location{
	proxy_pass http://backend; #与上方反向代理一致
}
~~~

nginx -s reload

访问47.115.215.143/app 默认轮询访问，可以设置权重也就是访问到的概率，修改配置

~~~
upstream backend{
	ip_hash; #根据客户端的ip地址进行哈希，同一个客户端的请求会分配到同一个服务器
	server 47.115.215.143:8000 weight=3;
	server 47.115.215.143:8001;
	server 47.115.215.143:8002;
}
~~~

# HTTPS

http 80 https 443 需要使用ssl证书(免费申请)-得到密钥文件+证书文件

也可以申请自签

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240320192929544.png" alt="image-20240320192929544" style="zoom: 33%;" />

生成两个文件.key .pem

server中添加以下内容

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240320193101594.png" alt="image-20240320193101594" style="zoom:33%;" />

nginx -s reload

https://47.115.215.143

这种没有ca机构验证会有安全提示

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240320193308883.png" alt="image-20240320193308883" style="zoom:33%;" />

自动跳到https

# 虚拟主机

一个server就是一个虚拟主机，可以在servers文件夹里新建一个文件，把配置文件复制进去也没有问题

会自动根据访问的域名匹配server项

部署一个Vue(渐进式JavaScript前端框架)

...此处省略配置过程

在servers目录下新建vue.conf

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240320193831477.png" alt="image-20240320193831477" style="zoom:33%;" />

root跟打包好的vue目录

# hexo搭建github博客网页

新建仓库，仓库名字 用户名.github.io,勾选readme

这时在网站上可以直接访问到

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240325085453572.png" alt="image-20240325085453572" style="zoom:33%;" />





cd在之前的nginx/bolg文件夹中

~~~
[root@iZf8zeh9f5g1mo4rk51t4pZ blog]# hexo new "tutu's personal blog"
INFO  Validating config
INFO  Created: ~/nginx/blog/source/_posts/tutu-s-personal-blog.md
hexo generate
hexo deploy
~~~

编辑_config.yml

~~~
deploy:
  type: git
  repo: git@github.com:你的用户名/你的用户名.github.io.git
  branch: master
~~~

下载插件npm install hexo-deployer-git --save

git config --global user.email tutu2061867903@gmail.com

git config --global user.name tutuLP

地址改用以下：

git@github.com:tutuLP/tutuLP.github.io.git 可能需要打开端口22 9418

https://github.com/tutuLP/tutuLP.github.io.git



hexo deploy

进入GitHub仓库的设置页面，找到GitHub Pages选项，并将源设置为`master branch`。

再次访问即可成功





1. **选择一个新主题**：

   - 在Hexo官方网站或GitHub上找到你喜欢的主题。确保主题与你的博客风格和需求相匹配。

2. **安装主题**：

   - 使用npm或者直接从GitHub克隆主题的仓库到你的Hexo博客目录。具体操作取决于主题作者提供的安装方式。

3. **配置主题**：

   - 打开Hexo博客的配置文件（_config.yml），找到并修改

     ```
     theme
     ```

     项，将其值设置为你安装的主题的名称。例如：

     ```
     makefileCopy code
     theme: your-theme-name
     ```

4. **根据主题文档进行配置**：

   - 主题通常会有自己的配置选项和文档。根据主题作者提供的文档进行配置，以满足你的需求。

5. **重新生成和部署博客**：

   - 在完成主题的安装和配置后，运行以下命令重新生成静态文件并部署博客：

     ```
     Copy codehexo clean
     hexo generate
     hexo deploy
     ```



1. **在云服务器上安装Nginx**：

   - 首先确保你的云服务器上已经安装了Nginx。你可以使用包管理器（如apt、yum等）来安装。

2. **克隆你的Hexo仓库到云服务器**：

   - 在云服务器上克隆你的Hexo仓库，确保你已经安装了Git。

3. **配置Nginx**：

   - 编辑Nginx的配置文件，将Hexo博客的路径指向你克隆的Hexo仓库的public目录。配置类似于以下内容：

     ```
     arduinoCopy codeserver {
         listen 80;
         server_name your-domain.com;
         location / {
             root /path/to/your/hexo/public;
             index index.html;
         }
     }
     ```

   - 确保替换your-domain.com为你的域名，/path/to/your/hexo/public为你Hexo博客的public目录的路径。

4. **重启Nginx服务**：

   - 在完成配置后，重启Nginx服务以使更改生效。

5. **设置域名解析**：

   - 如果你使用了自定义域名，确保将域名解析指向你的云服务器的IP地址。
