---

title: "hexo"
date: 2024-03-25
categories:

  - Web
tags:
  - Node.js
  - 静态网站生成器

---

# hexo搭建github博客网页

安装请看Nginx基础篇

新建仓库，仓库名字 用户名.github.io,勾选readme

这时在网站上可以直接访问到

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240325085453572.png" alt="image-20240325085453572" style="zoom:33%;" />





cd在之前的nginx/bolg文件夹中

~~~
[root@iZf8zeh9f5g1mo4rk51t4pZ blog]# hexo new "tutu's personal blog"
INFO  Validating config
INFO  Created: ~/nginx/blog/source/_posts/tutu-s-personal-blog.md
hexo generate #生成静态网页文件
hexo deploy #部署
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

 

hexo deploy 部署到github服务器平台

进入GitHub仓库的设置页面，找到GitHub Pages选项，并将源设置为`master branch`。

再次访问即可成功



以上只是构建本地的文件生成静态网站上传到github服务器

现在我们pull仓库中的文件进行构建

cd source/_posts

git init 我这里生成的分治是master比较古老

git remote add origin https://github.com/tutuLP/tutuLP.github.io.git 建立关联

git branch --set-upstream-to=origin/main master 本地master分支与远程main关联

git pull origin main --allow-unrelated-histories 拉取-运行历史不同步

然后回到blog hexo generate hexo deploy 可以简写成hexo c hexo d



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



进入blog  npm i hexo-theme-stellar

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

