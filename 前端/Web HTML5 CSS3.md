---
title: "HTML+CSS"
date: 2024-06-09
categories:
  - 前端
tags:
  - HTML
  - CSS
---

vscode 安装插件 open in browser

#HTML

##标签

标签

~~~html
<strong>内容</strong>
<br> <!-- 换行 --> 注释：ctrl+/
<hr> <!-- 水平线 -->
~~~

右键open in browser或者alt+b

###html骨架

~~~html
<html>
    <head>
        <title>网页标题</title>
    </head>
    <body>
        网页主体
    </body>
</html>
~~~

在vscode中 !+enter/tab自动生成骨架

~~~html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    
</body>
</html>
~~~

标签的关系：父子集，平行

###标题

h1~h6 双标签 h1只用一次

###段落

p 双 段落之间有空隙

###文本格式化

<img src="./images/Web HTML5 CSS3.assets/image-20240530112228863-1737030799559-50-1737030806990-83.png" alt="image-20240530112228863" style="zoom:33%;" />



###图像标签

<img src="URL">

<img src="./images/Web HTML5 CSS3.assets/image-20240530112532059-1737030806990-53.png" alt="image-20240530112532059" style="zoom:33%;" />

<img src="URL" alt="图片加载出错"> 图片的大小一般用css控制

路径

相对：从当前文件夹出发  . ..   

绝对：win：从盘 ios：从根出发  C:/

可以用在线网站的图片网址

###超链接

~~~html
<a href="#">空链接</a>
<a href="" target="_blank">text</a> 新窗口打开
~~~

###音频标签

<img src="./images/Web HTML5 CSS3.assets/image-20240530113650694-1737030806990-63.png" alt="image-20240530113650694" style="zoom:33%;" />

后三个属性都可以简写 引号里面的内容和名字一样

###视频标签

<img src="./images/Web HTML5 CSS3.assets/image-20240530113948869-1737030806990-55.png" alt="image-20240530113948869" style="zoom: 33%;" />



###列表-排列整齐

无序ul嵌套li   有序ol嵌套li   定义列表dl嵌套dt标题 dd

<img src="./images/Web HTML5 CSS3.assets/image-20240530114747460-1737030806990-57.png" alt="image-20240530114747460" style="zoom:33%;" />

###表格

<img src="./images/Web HTML5 CSS3.assets/image-20240530115244523-1737030806990-61.png" alt="image-20240530115244523" style="zoom: 33%;" />

<img src="./images/Web HTML5 CSS3.assets/image-20240530115345382-1737030806990-59.png" alt="image-20240530115345382" style="zoom:33%;" />

<img src="./images/Web HTML5 CSS3.assets/image-20240530115528423-1737030806990-67.png" alt="image-20240530115528423" style="zoom:33%;" />

包裹住tr即可，但是视觉上没变化

合并单元格

<img src="./images/Web HTML5 CSS3.assets/image-20240530115811123-1737030806990-65.png" alt="image-20240530115811123" style="zoom:33%;" />

<img src="./images/Web HTML5 CSS3.assets/image-20240530115908567-1737030806990-75.png" alt="image-20240530115908567" style="zoom:33%;" />

### 表单-收集用户信息

**input**

<img src="./images/Web HTML5 CSS3.assets/image-20240530124822126-1737030806990-69.png" alt="image-20240530124822126" style="zoom:33%;" />

**占位文本-提示**

placeholder=""



<img src="./images/Web HTML5 CSS3.assets/image-20240530161038632-1737030806990-71.png" alt="image-20240530161038632" style="zoom: 33%;" />

**其他属性**

file multitiple可以多选上传

checkbox checked

**下拉菜单**

<img src="./images/Web HTML5 CSS3.assets/image-20240530172520032-1737030806990-73.png" alt="image-20240530172520032" style="zoom:33%;" />

###文本域

多行输入

textarea 双

###label标签

双

绑定文字和表单控件的关系，增大表单控件的点击范围

<img src="./images/Web HTML5 CSS3.assets/image-20240530173446948-1737030806990-77.png" alt="image-20240530173446948" style="zoom:33%;" />

### 按钮button

<img src="./images/Web HTML5 CSS3.assets/image-20240530213231208-1737030806990-79.png" alt="image-20240530213231208" style="zoom:33%;" />

###无语义布局标签 

div 独占一行 大盒子(可以换行展示内容)

span 不换行 小盒子(不换行展示)

### 字符实体

<img src="./images/Web HTML5 CSS3.assets/image-20240530213620093-1737030806990-88.png" alt="image-20240530213620093" style="zoom:33%;" />

多个键盘空格网页上只有一个

# CSS

层叠样式表

<img src="./images/Web HTML5 CSS3.assets/image-20240530213949450-1737030806990-81.png" alt="image-20240530213949450" style="zoom:33%;" />

## 引入方式

**内部**

css写在style标签里面

**外部**

单独放 .css html使用link标签引入

<img src="./images/Web HTML5 CSS3.assets/image-20240530214301555-1737030806990-84.png" alt="image-20240530214301555" style="zoom:33%;" />

<img src="./images/Web HTML5 CSS3.assets/image-20240530215044614-1737030806990-86.png" alt="image-20240530215044614" style="zoom:33%;" />

**行内**

配合Javascript使用

~~~html
<div style="color: blueviolet;font-size: 30px;">
        test2
    </div>
~~~

