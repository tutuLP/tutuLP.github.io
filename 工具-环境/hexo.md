---
title: "使用hexo构建博客网站"
date: 2025-03-4
categories:
  - 工具-环境
tags:
  - Node.js
  - hexo
---

# 前置准备

## 下载Node.js

使用nvm安装Node.js

~~~shell
curl -o- https://gitee.com/mirrors/nvm/raw/v0.39.3/install.sh | bash #使用镜像源安装nvm
source ~/.bashrc
nvm --version
nvm install --lts  #安装Node
~~~

## 下载hexo

~~~shell
npm install hexo-cli -g
hexo init blog  #用hexo搭建一个文件夹
cd blog   
#下载插件
npm i hexo-theme-volantis --save #下载主题volantis
npm i hexo-generator-json-content --save  #搜索功能
npm i giscus --save #下载评论系统
npm install hexo-deployer-git --save #git推送
~~~

## 目录结构和使用说明

/public 存放生成后的静态网页资源
/source 存放构建网站所需要的源文件

hexo g  生成静态网站
hexo s  启动本地服务器，访问4000号端口就可以看到构建的网页
hexo d  部署到远程服务器
hexo clean && hexo g -d  前面三个命令的集合

# 使用nginx代理网页

把public目录下的所有内容复制到nginx的html目录下
cd public
cp -rf * /usr/share/nginx/html 
启动nginx服务访问80端口查看

# 使用github云代理网页

* 新建仓库

新建仓库-仓库名字：用户名.github.io
仓库的main分支用来存源笔记文件 master分支用来存前端文件

* 编辑_config.yml

~~~
deploy:
  type: git
  repo: git@github.com:你的用户名/你的用户名.github.io.git
  branch: master
~~~

这里可以使用ssh也可以使用http 使用ssh要先配置密钥
我的ssh地址：git@github.com:tutuLP/tutuLP.github.io.git  

* 配置git用户信息

git config --global user.email tutu2061867903@gmail.com
git config --global user.name tutuLP

可能需要打开端口22 9418

* 修改仓库配置

进入GitHub仓库的设置页面-Pages-GitHub Pages选项 将源设置为master分支
这时得到网址 https://tutulp.github.io/

* hexo d  部署到github服务器平台

* 直接在/source/_posts文件夹下克隆仓库进行快速同步

## 后续更新博客

1. 提交并push到github
2. cd /root/nginx/blog/source/_posts/
3. git pull
4. hexo generate
5. hexo deploy

## 自定义域名

https://tutulp.github.io/

是的，你可以通过将自定义域名（例如你拥有的域名）映射到你的GitHub Pages网站来实现这一目标。以下是一些步骤：

1. **购买域名**：首先，你需要购买一个域名，例如`yourdomain.com`。你可以通过注册商（例如GoDaddy、Namecheap等）购买域名。
2. **配置DNS记录**：登录到你的域名注册商的控制面板，并配置DNS记录，将你的域名指向GitHub Pages。通常，你需要添加一个CNAME记录，将域名指向你的GitHub Pages用户名，例如`yourusername.github.io`。具体的步骤可能会因注册商而异，你可以参考GitHub Pages提供的文档或向注册商寻求帮助。
3. **在GitHub设置中配置自定义域名**：登录到GitHub，进入你的仓库设置页面，将你的自定义域名添加到"Custom domain"字段中。
4. **等待DNS生效**：一旦你配置了DNS记录并保存了更改，你需要等待一段时间（通常几小时至48小时），直到DNS记录生效。一旦生效，你的域名就会指向你的GitHub Pages网站。
5. **HTTPS支持**：GitHub Pages支持HTTPS，但你需要确保你的域名有有效的SSL证书。你可以使用GitHub提供的自动生成的SSL证书，或者使用其他证书颁发机构提供的证书。

##shell脚本自动化

/root下新建blog.sh

~~~
#!/bin/bash  
cd /root/hexo/blog_new/source/_posts/tutuLP.github.io || exit  
git pull  
cd .. 
cd ..  
hexo generate  
hexo deploy  
echo "博客部署完成！"
~~~

bash blog.sh

# 主题美化

根目录下可以看到\_config.yml的配置文件，新建_config.volantis.yml

## 修改配置文件

\_config.yml
~~~yml
title: tutulp's blog
description: 'tutulp的博客网站'
author: tutulp
language: zh-CN
theme: volantis
deploy:
  type: git
  repo: git@github.com:tutuLP/tutuLP.github.io.git  
  branch: master
~~~

_config.volantis.yml
~~~yml
# 平滑滚动效果
scroll_smooth: true
#作者
article:
  body:
    meta_library:
      author:
        avatar:
        name: tutulp
#搜索
search:
  enable: true
  service: hexo
#暗黑模式
plugins:
  darkmode:
    enable: true
#评论 参考https://giscus.app/zh-CN
comments:
  title: <i class='fas fa-comments'></i> 评论
  service: giscus
  giscus:
    repo: "tutuLP/tutuLP.github.io"
    repo-id: "R_kgDOLk_elA"
    category: "Announcements"
    category-id: "DIC_kwDOLk_elM4Cnh_D"
    mapping: "pathname"
    reactions-enabled: "1"
    emit-metadata: "0"
    lang: "zh-CN"
#首页
cover:
  height_scheme: full # full, half
  layout_scheme: search
  display:
    home: true
    archive: true
    others: true # can be written in front-matter 'cover: true'
    categorie: true
    tag: true
  background: https://s3.bmp.ovh/imgs/2025/03/03/64c32e0ed0f747a9.jpg
  title: 'Leland Titus'
  subtitle: '风沉云万里'
  search: Recover lost knowledge  # search bar placeholder
  features:
    - name: 博客
      icon: #
      img: volantis-static/media/twemoji/assets/svg/1f4f0.svg #  https://cdn.jsdelivr.net/gh/twitter/twemoji@13.0/assets/svg/1f4f0.svg
      url: /
    - name: 分类
      icon: #
      img: volantis-static/media/twemoji/assets/svg/1f4f0.svg #  https://cdn.jsdelivr.net/gh/twitter/twemoji@13.0/assets/svg/1f4f0.svg
      url: /categories/
    - name: 标签
      icon: #
      img: volantis-static/media/twemoji/assets/svg/1f4f0.svg #  https://cdn.jsdelivr.net/gh/twitter/twemoji@13.0/assets/svg/1f4f0.svg
      url: /tags/
    - name: 归档
      url: /archives/
    - name: 作者
      icon: #
      img: volantis-static/media/twemoji/assets/svg/1f9ec.svg #  https://cdn.jsdelivr.net/gh/twitter/twemoji@13.0/assets/svg/1f9ec.svg
      url: /about/
#导航栏
navbar:
  visiable: auto # always, auto
  logo: # choose [img] or [icon + title]
    img: volantis-static/media/org.volantis/blog/Logo-NavBar@3x.png # https://cdn.jsdelivr.net/gh/volantis-x/cdn-org/blog/Logo-NavBar@3x.png
    icon:
    title:
  menu:
    - name: 博客
      icon: fa-solid fa-rss
      url: /
    - name: 分类
      icon: fa-solid fa-folder-open
      url: categories/
    - name: 标签
      icon: fa-solid fa-tags
      url: tags/
    - name: 归档
      icon: fa-solid fa-archive
      url: /archives/
    - name: 作者
      icon: fa-solid fa-info-circle
      url: about/
    - name: 暗黑模式
      icon: fas fa-moon
      toggle: darkmode
  search: Search...
#页尾
site_footer:
  # layout of footer: [aplayer, social, license, analytics, info(本站主题), copyright]
  layout: [license2, info, copyright]
  source: https://github.com/volantis-x/volantis-docs/
  # analytics using leancloud
  analytics: >
    <span id="lc-sv">本站总访问量为 <span id='number'><i class="fa-solid fa-loader fa-spin fa-fw" aria-hidden="true"></i></span> 次</span>
    <span id="lc-uv">访客数为 <span id='number'><i class="fa-solid fa-loader fa-spin fa-fw" aria-hidden="true"></i></span> 人</span>
  # site copyright
  license2: >
    <span id="lc-sv">博客内容遵循 tutulp-非商业性使用-相同方式共享 4.0 国际 (CC BY-NC-SA 4.0) 协议</span>
  copyright: '[Copyright © since 2025 tutulp](/)'
#侧边部件
sidebar:
  widget_library:
    blogger:
      avatar: https://s3.bmp.ovh/imgs/2025/03/03/30639e1f0584064f.jpg
      shape: circle

#样式
custom_css:
#抗锯齿
  font_smoothing: true # font-smoothing for webkit
#导航栏
  navbar:
    height: 64px
    width: auto # auto, max
    effect: [shadow, floatable, blur] # [shadow, floatable, blur]
#滚动条
  scrollbar:
    size: 10px
    border: 2px
    color: '#2196f3'
    hover: '#ff5722'
#字体
  fontfamily:
    bodyfont:
      fontfamily: 'OPPO-Sans'
      name: 'OPPO-Sans'
      url: /fonts/OPPO-Sans.ttf 
    codefont:
      fontfamily: 'JetBrainsMono-Bold'
      name: 'JetBrainsMono-Bold'
      url: /fonts/JetBrainsMono-Bold.ttf
~~~

## source需要的文件

~~~
├── about
│   └── index.md
├── categories
│   └── index.md
├── fonts
│   ├── JetBrainsMono-Bold.ttf
│   ├── OPPO-Sans.ttf
│   └── WenQuanYiZenHei.ttf
├── _posts
│   └── source
└── tags
    └── index.md
~~~

新建上述目录结构

about/index.md

~~~md
---
layout: docs
seo_title: 关于
bottom_meta: false
sidebar: []
twikoo:
  placeholder: 有什么想对我说的呢？
---

{% btns circle center grid5 %}
<a>
<img src='https://s3.bmp.ovh/imgs/2025/03/03/30639e1f0584064f.jpg'>
<b>tutulp</b>
</a>
{% endbtns %}
从此天边尽沉云， 风卷残阳没旧痕

邮箱：n199m992@163.com
QQ：2061867903
github：https://github.com/tutuLP
gitee：https://gitee.com/tutuLP
~~~

categories/index.md

~~~
---
layout: category
index: true
title: 所有分类
---
~~~

tags/index.md

~~~
---
layout: tag
index: true
title: 所有标签
---
~~~

fonts存放字体文件，在yml配置文件中使用eg：/fonts/JetBrainsMono-Bold.ttf
需要使用图片的时候也可以这样放和引用

## 参考链接

主题volantis：https://volantis.js.org/v6/getting-started/

ImgURL免费图床：https://www.imgurl.org/vip/manage

评论系统：https://artalk.js.org/ 

简历博客：https://resume.js.org/zh-cn/

参考美化教程：https://busyogg.github.io/article/38effc2a84e9/

优秀博客：https://inkss.cn/

npm图床：https://xingpingcn.top/npm%E5%9B%BE%E5%BA%8A%EF%BC%88%E4%B8%8D%E9%9C%80%E8%A6%81%E6%9C%AC%E5%9C%B0%E9%83%A8%E7%BD%B2%EF%BC%89.html
