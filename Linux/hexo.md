---
title: "hexo"
date: 2024-03-25
categories:
  - Web
tags:
  - Node.js
  - 静态网站生成器
---

# 下载

* npm install hexo-cli -g

* cd   (当前用户的目录)

* hexo init blog  用hexo搭建一个文件夹

* cd blog   

* hexo g 生成一个静态站点，把markdown文件转化为静态页面放到public文件夹下 

* hexo s 启动本地服务器 访问4000号端口就可以看到构建的网页

# 使用nginx代理网页

把public目录下的所有内容复制到nginx的html目录下

* cd public 

* cp -rf * /usr/share/nginx/html 

# 使用github云代理网页

* 新建仓库

新建仓库-仓库名字：用户名.github.io,勾选readme

仓库的main分支用来存源笔记文件 master分支用来存前端文件

* 编辑_config.yml

~~~
deploy:
  type: git
  repo: git@github.com:你的用户名/你的用户名.github.io.git
  branch: master
~~~

* 下载插件

npm install hexo-deployer-git --save

git config --global user.email tutu2061867903@gmail.com

git config --global user.name tutuLP

地址改用以下：

git@github.com:tutuLP/tutuLP.github.io.git 可能需要打开端口22 9418

https://github.com/tutuLP/tutuLP.github.io.git

* 修改仓库配置

进入GitHub仓库的设置页面-Pages-GitHub Pages选项 将源设置为master分支

* 这时得到网址 https://tutulp.github.io/

hexo deploy 部署到github服务器平台



以上只是构建本地的文件生成静态网站上传到github服务器

现在我们pull仓库中的文件进行构建

cd source/_posts

git init 我这里生成的分治是master比较古老

git remote add origin https://github.com/tutuLP/tutuLP.github.io.git 建立关联

git remote add origin git@github.com:tutuLP/tutuLP.github.io.git

git pull origin main --allow-unrelated-histories 拉取-运行历史不同步

git branch --set-upstream-to=origin/main master 本地master分支与远程main关联

然后回到blog `hexo generate`       `hexo deploy`



pull进来的文件不会自动拿文件名作为标题，我们需要在文档前面加上一段Front Matter，用于指定文章元数据的YAML格式的内容

```yaml
---
title: "文章标题"
date: YYYY-MM-DD HH:MM:SS
categories:
  - 分类1
  - 分类2
tags:
  - 标签1
  - 标签2
---
```



## 主题stellar

来源：[Stellar：开始您全新的博客之旅 - XAOXUU](https://xaoxuu.com/wiki/stellar/#start)

还可在上方网页上搜索Volantis另一个主题



进入blog  

npm i hexo-theme-stellar

在 _config.yml 文件中找到并修改： theme: stellar

~~~
title: 您的网站名称
avatar: 您的头像链接
favicon: 您的网站icon
# subtitle: # subtitle 已移至主题配置中
# 多语言
language:
  - zh-CN
  - en
~~~

`_config.yml` 文件旁边新建一个文件： `_config.stellar.yml` ，在这个文件中的配置信息优先级高于主题文件夹中的配置文件。

默认生成 Open Graph 标签，如果您不希望生成它，可以在主题配置文件中关闭：

```
blog/_config.stellar.ymlopen_graph:
  enable: true
  twitter_id: # for open_graph meta
```

## 管理构建好的静态网页

hexo generate后生成的资源存放在public文件夹中

当我测试时创建的分类和标签会永远存在，即使我删除.md文档重新构建也依旧存在

cd /public/categories 

rm -rf

然后清除临时文件 生成 推送

hexo clean

hexo generate

hexo deploy



##部署的时候提示需要输入账号密码

1. cd ~/.ssh 

ssh-keygen -t rsa -b 2048 -C 'github注册邮箱'

2. ssh-keygen -t rsa -b 2048 -C 'tutu2061867903@gmail.com' 一路回车即可
3. 粘贴.pub文件的内容到SSH keys

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240609211058663.png" alt="image-20240609211058663" style="zoom:33%;" />

4. 进入hexo项目根目录 cd blog
5. 删除文件 .deploy_git
6. 修改_config.yml
7. repo:项目的SSH地址



## 自定义域名

https://tutulp.github.io/

是的，你可以通过将自定义域名（例如你拥有的域名）映射到你的GitHub Pages网站来实现这一目标。以下是一些步骤：

1. **购买域名**：首先，你需要购买一个域名，例如`yourdomain.com`。你可以通过注册商（例如GoDaddy、Namecheap等）购买域名。
2. **配置DNS记录**：登录到你的域名注册商的控制面板，并配置DNS记录，将你的域名指向GitHub Pages。通常，你需要添加一个CNAME记录，将域名指向你的GitHub Pages用户名，例如`yourusername.github.io`。具体的步骤可能会因注册商而异，你可以参考GitHub Pages提供的文档或向注册商寻求帮助。
3. **在GitHub设置中配置自定义域名**：登录到GitHub，进入你的仓库设置页面，将你的自定义域名添加到"Custom domain"字段中。
4. **等待DNS生效**：一旦你配置了DNS记录并保存了更改，你需要等待一段时间（通常几小时至48小时），直到DNS记录生效。一旦生效，你的域名就会指向你的GitHub Pages网站。
5. **HTTPS支持**：GitHub Pages支持HTTPS，但你需要确保你的域名有有效的SSL证书。你可以使用GitHub提供的自动生成的SSL证书，或者使用其他证书颁发机构提供的证书。

通过这些步骤，你就可以将你的GitHub Pages网站与你自己的域名关联起来，使访问者在浏览器中看到你的域名，但实际上打开的是GitHub Pages上托管的网站。

## 后续更新博客

1. 提交并push到github
2. cd /root/nginx/blog/source/_posts/
3. git pull
4. hexo generate
5. hexo deploy

## 自定义主题样式

~~~yaml
title: 您的网站名称
avatar: 您的头像链接 # 这个我用的png 512x512
favicon: 您的网站icon # 32x32 png
# subtitle: # subtitle 已移至主题配置中
# 多语言
language:
  - zh-CN
  - en
~~~

icon生成网址：https://www.logosc.cn/favicon-generator

网站图标的在线链接一直显示不了，在blog/public文件下创建image文件夹，里面放入图片，然后配置文件中 favicon: '/image/32x32.png' 即可



在博客根目录的 `_config.yml` 文件旁边新建一个文件： `_config.stellar.yml` 这个文件配置优先级高

### 文章封面

根据 tags 作为关键词为每一篇文章在线搜索封面

或者在文章开头给关键词

~~~yml
blog/_config.stellar.yml
article:
  auto_cover: true
  
---
cover: workout,strava
---
~~~

### 设置摘要

自动

手动

~~~yaml
article:
  auto_excerpt: 200
~~~

~~~yaml
---
cover: /assets/xaoxuu/blog/2020-0927a@1x.svg
---

在心率管家默默无闻地上线了一年多之后，现在终于打算来好好聊聊关于手机摄像头测量心率的那些事。本文参考了很多前辈的文章，将在文末列出。

<!-- more -->

后面是正文部分，在主页看不到。
~~~

AI摘要 基于 tianli_gpt 前端项目 [Post-Summary-AI](https://github.com/qxchuckle/Post-Summary-AI)

~~~yml
_config.stellar.yml
# AI 摘要
# https://github.com/qxchuckle/Post-Summary-AI
tianli_gpt: 
  enable: #true
  js: https://jsd.onmicrosoft.cn/gh/qxchuckle/Post-Summary-AI@6.0/chuckle-post-ai.min.js
  field: post # all, post, wiki
  key: 5Q5mpqRK5DkwT1X9Gi5e # tianli_gpt key
  total_length: 1000 # 设置提交的字数限制，默认为1000字，上限为5000，超过5000字符将被截断
  typewriter: true # 打字机动画 
  summary_directly: true # 是否直接显示摘要，否则显示 AI 简介
  rec_method: all # all, web # 文章推荐方式，all：匹配数据库内所有文章进行推荐，web：仅当前站内的文章，默认all
  hide_shuttle: true # 是否隐藏矩阵穿梭
  summary_toggle: false # 是否开启切换简介功能，经过一些处理后，重新生成一份简介，这将消耗key字数
  interface: # AI 信息展示
    name: AI摘要
    introduce: '我是文章辅助AI: QX-AI，点击下方的按钮，让我生成本文简介、推荐相关文章等。'
    version: TianliGPT
    button: ["介绍自己", "推荐文章", "生成摘要", "矩阵穿梭"] # 底部按钮文字
~~~

如何获取 `tianliGPT_key`：到 [爱发电](https://afdian.net/item/f18c2e08db4411eda2f25254001e7c00) 中购买，购买完成后，进入 [网页后台管理](https://summary.zhheo.com/) 绑定key并添加自己的站点

key与博客地址为绑定状态，所以本地调试时是无法接收到数据的。

### 获取文章开头内容

根目录下 scaffolds 文件夹中编辑 post.md 的 font-matter，根据自己的需要增加你想要配置的内容

~~~yml
---
# 基本信息
title: {{ title }}
date: {{ date }}
tags: []
categories: []   #categories: [设计开发, iOS开发] 两级分类
description: # excerpt 也可 
# 封面
cover: 
banner: 
poster: # 海报（可选，全图封面卡片）
  topic: 标题上方的小字 # 可选
  headline: 大标题 # 必选
  caption: 标题下方的小字 # 可选
  color: 标题颜色 # 可选
# 插件
sticky: # 数字越大越靠前
mermaid:
katex: 
mathjax: 
# 可选
topic: # 专栏 id
author: 
references:
comments: # 设置 false 禁止评论
indexing: # 设置 false 避免被搜索
breadcrumb: # 设置 false 隐藏面包屑导航
leftbar: 
rightbar:
h1: # 设置为 '' 隐藏标题
type: # tech/story
---
~~~

### 相关文章推荐

安装插件 `npm i hexo-related-popular-posts`

~~~yaml
article:
  # npm i hexo-related-popular-posts
  related_posts:
    enable: true
    title: 您可能感兴趣的文章
~~~

### 参考资料

~~~yml
---
references:
  - '[心跳之旅—💗—iOS用手机摄像头检测心率(PPG)](https://punmy.cn/2016/07/28/15231176397746.html)'
  - '[PPG光电容积脉搏波描记法技术概况](https://www.jianshu.com/p/695c131abfa5)'
  ...
---
~~~

### 许可协议

~~~yml
article:
  license: '本文采用 [署名-非商业性使用-相同方式共享 4.0 国际](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可协议，转载请注明出处。'
~~~

若你配置了作者数据 _data/authors.yml 和文章作者，可以在 license 中使用 {author.name} 来自动替换为当前文章作者名字。

~~~yml
article:
  license: '本文由{author.name}编写，采用...'
~~~

### 分享链接

分享至微信会生成对应的页面二维码，weibo 和 email 会自动跳转到对应软件或网页，link 会拷贝当前页面链接至剪切板。

~~~yml
article:
  share: # [wechat, weibo, email, link]
~~~

### 覆盖 OpenGraph

如果分享到社交平台的缩略图不理想，可以通过这个特性覆盖为自己想要的：

~~~yml
open_graph:
  image: /assets/xaoxuu/blog/2022-1029a@2x.webp
~~~

### 设置导航栏

~~~yml
menubar:
  columns: 4 # 一行多少个
  items: # 可按照自己需求增加，符合以下格式即可
    - id: more
      theme: ''
      title: 更多
      icon: ''
      url: /more/
~~~





~~~yaml
logo:
  subtitle: '' # '文字1 | 文字2' (鼠标放上去会切换到文字2)         设置鼠标指上 subtitle 后翻转另一行字  记得使用utf-8编码
  
#背景
style:
  ...
  leftbar:
    background-image: url(https://gcore.jsdelivr.net/gh/cdn-x/placeholder@1.0.13/image/sidebar-bg1@small.jpg)
    blur-px: 100px # 模糊半径
    blur-bg: var(--alpha60) # 模糊颜色
~~~

### 字体设置

* 第一步：_config.yml 文件中，最后一行加入以下指令

~~~yml
inject: head: -(
  <link
    rel="stylesheet"
    href="https://cdn.staticfile.org/lxgw-wenkai-screen-webfont/1.6.0/lxgwwenkaiscreen.css"
    media="all"
  />
);
~~~

* 第二步：在_config.stellar.yml 中找到 style.font-family，修改以下内容：

~~~yml
style:
    font-family:
        logo: '"LXGW WenKai Screen", ...
        body: '"LXGW WenKai Screen", ...
~~~

~~~
inject: 
  head: 
    - |
      <link
        rel="stylesheet"
        href="https://cdn.staticfile.org/lxgw-wenkai-screen-webfont/1.6.0/lxgwwenkaiscreen.css"
        media="all"
      />
~~~



~~~
  font-family:
    # logo: 'system-ui, "LXGW WenKai Screen", "Microsoft Yahei", "Segoe UI", -apple-system, Roboto, Ubuntu, "Helvetica Neue", Arial, "WenQuanYi Micro Hei", sans-serif'
    # body: 'system-ui, "LXGW WenKai Screen", "Microsoft Yahei", "Segoe UI", -apple-system, Roboto, Ubuntu, "Helvetica Neue", Arial, "WenQuanYi Micro Hei", sans-serif'
    # logo: "LXGW WenKai Screen"
    # body: "LXGW WenKai Screen"
~~~

1. `source`目录下新建文件夹`css`，在 CSS 目录下新建我们自定义文件 `my.css`

2. 博客配置文件`_config.yml`中引入 CSS 文件，CSS 文件一般在`head`引入，JS 文件一般在`script`里引入

打开控制面板 ---  程序和功能  ---  打开或关闭Windows功能  ----  勾选上Internet Informent Services



### 测试

hexo generate后

hexo server

http://localhost:4000/查看结果

达到效果后再deploy

# Linux系统更换后移植

* 保存配置文件

_config.yml

_config.stellar.yml

scaffolds

public文件下image

删除原来的仓库新建仓库重新配置一遍
