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

新建仓库-仓库名字：用户名.github.io,勾选readme

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

## 管理构建好的静态网页

hexo generate后生成的资源存放在public文件夹中

当我测试时创建的分类和标签会永远存在，即使我删除.md文档重新构建也依旧存在

cd /public/categories 

rm -rf

然后清除临时文件 生成 推送

hexo clean

hexo generate

hexo deploy





https://tutulp.github.io/

是的，你可以通过将自定义域名（例如你拥有的域名）映射到你的GitHub Pages网站来实现这一目标。以下是一些步骤：

1. **购买域名**：首先，你需要购买一个域名，例如`yourdomain.com`。你可以通过注册商（例如GoDaddy、Namecheap等）购买域名。
2. **配置DNS记录**：登录到你的域名注册商的控制面板，并配置DNS记录，将你的域名指向GitHub Pages。通常，你需要添加一个CNAME记录，将域名指向你的GitHub Pages用户名，例如`yourusername.github.io`。具体的步骤可能会因注册商而异，你可以参考GitHub Pages提供的文档或向注册商寻求帮助。
3. **在GitHub设置中配置自定义域名**：登录到GitHub，进入你的仓库设置页面，将你的自定义域名添加到"Custom domain"字段中。
4. **等待DNS生效**：一旦你配置了DNS记录并保存了更改，你需要等待一段时间（通常几小时至48小时），直到DNS记录生效。一旦生效，你的域名就会指向你的GitHub Pages网站。
5. **HTTPS支持**：GitHub Pages支持HTTPS，但你需要确保你的域名有有效的SSL证书。你可以使用GitHub提供的自动生成的SSL证书，或者使用其他证书颁发机构提供的证书。

通过这些步骤，你就可以将你的GitHub Pages网站与你自己的域名关联起来，使访问者在浏览器中看到你的域名，但实际上打开的是GitHub Pages上托管的网站。
