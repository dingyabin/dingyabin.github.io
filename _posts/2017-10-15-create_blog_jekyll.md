---
layout: post
title: 基于Jekyll搭建个人博客
date: 2017-10-15
tags: 博客   
---

### 序言
之前听说过github可以搭建个人博客，一直没有尝试，这次利用周末时间找了几篇文章参考了一下，果然效果不错(网友诚不欺我也)，借助网上大神的模板，可以搭建很多看起来很炫酷的博客。


### 原理介绍
众所周知，github可以托管代码，但当我们看到一大堆源码时，只会让人头晕脑涨，不知何处入手。我们更希望看到的是，一个简明易懂的网页，说明每一步应该怎么做。因此，github就设计了Pages功能，它
允许用户自定义项目首页，用来替代默认的源码列表，也提供模板，允许站内生成网页，但也允许用户自己编写网页，然后上传。有意思的是，这种上传并不是单纯的上传，而是会经过Jekyll程序的再处理。所以，github Pages可以被认为是用户编写的、托管在github上的静态网页。

### Jekyll是什么
[Jekyll(杰克尔)](http://jekyllrb.com) 是一个简单的博客形态的静态站点生产机器。它有一个模版目录，其中包含原始文本格式的文档，通过 Markdown (或者 Textile) 以及 Liquid(开源模板语言) 转化成一个完整的可发布的静态网站，你可以发布在任何你喜爱的服务器上。
Jekyll 也可以运行在 GitHub Page 上，所以，我们可以使用 GitHub 的服务来搭建项目页面、博客或者网站，而且是完全免费的。
这种做法的好处是：
>* 免费，无限流量。
>* 享受git的版本管理功能，不用担心文章遗失。
>* 你只要写文章就可以了，其他事情一概不用操心，都由github处理。

但也有一些缺点：
>* 有一定技术门槛，你必须要懂一点git和网页开发。
>* 它生成的是静态网页，添加动态功能必须使用外部服务。
>* 它不适合大型网站，因为没有用到数据库，每运行一次都必须遍历全部的文本文件，网站越大，生成时间越长

使用 Jekyll 搭建博客之前要确认下本机环境，
>* Git 环境（用于部署到远端）
>* [Ruby](http://www.ruby-lang.org/en/downloads/) 环境（Jekyll 是基于 Ruby 开发的）
>* 包管理器 [RubyGems](http://rubygems.org/pages/download),如果你是 Mac 用户，你就需要安装 Xcode 和 Command-Line Tools了。下载方式 Preferences → Downloads → Components。

Jekyll可以配合第三方服务例如： Disqus（评论）、多说(评论) 以及分享 等等扩展功能，Jekyll 可以直接部署在 Github（国外） 或 Coding（国内） 上，可以绑定自己的域名。[Jekyll中文文档](http://jekyll.bootcss.com/)、[Jekyll英文文档](https://jekyllrb.com/)、[Jekyll主题列表](http://jekyllthemes.org/)。

### Jekyll 环境配置

>* 安装 jekyll
jekyll的安装方法很多，可根据官方的指导安装 [官方的指导安装](http://jekyllrb.com/docs/installation)

>* 进入博客目录

```
$ cd myBlog
```

>* 启动本地服务

```
$ jekyll serve
```

在浏览器里输入： http://localhost:4000，就可以看到你的博客效果了。

so easy!

### 目录结构
　
Jekyll 的核心其实是一个文本转换引擎。它的概念其实就是： 用你最喜欢的标记语言来写文章，可以是 Markdown，也可以是 Textile,或者就是简单的 HTML, 然后 Jekyll 就会帮你套入一个或一系列的布局中。在整个过程中你可以设置URL路径, 你的文本在布局中的显示样式等等。这些都可以通过纯文本编辑来实现，最终生成的静态页面就是你的成品了。

一个基本的 Jekyll 网站的目录结构一般是像这样的：

![](/images/posts/jekyll/jekyll_structure.png)

这些目录结构以及具体的作用可以参考 [官网文档](http://jekyll.com.cn/docs/structure/) :

![](/images/posts/jekyll/jekyll_structure_details.png)

进入 _config.yml 里面，修改成你想看到的信息，重新 jekyll server ，刷新浏览器就可以看到你刚刚修改的信息了。

到此，博客初步搭建算是完成了。


### 一个例子
屁话说了一大堆，下面，举一个实例，演示如何在github上搭建blog，可以跟着一步步做。为了便于理解，这个blog只有最基本的功能。在搭建之前，我们必须已经安装了git，并且有github账户。

#### 第一步，创建项目。
在你的电脑上，建立一个目录，作为项目的主目录。我们假定，它的名称为jekyll_demo。
> $ mkdir jekyll_demo

进入该目录，对该目录进行git初始化。
> $ cd jekyll_demo
> $ git init

以下所有动作，都在该仓库的master分支下完成。

#### 第二步，创建设置文件
在项目根目录下，建立一个名为`_config.yml`的文本文件。它是jekyll的设置文件，我们在里面填入如下内容，其他设置都可以用默认选项，具体解释参见官方网页。
> baseurl: /jekyll_demo

目录结构变成：
```
　　/jekyll_demo
　　　　|--　_config.yml
```

#### 第三步，创建模板文件。
在项目根目录下，创建一个_layouts目录，用于存放模板文件。
> $ mkdir _layouts

进入该目录，创建一个default.html文件，作为Blog的默认模板。并在该文件中填入以下内容。

```
<!DOCTYPE html>
<html>
<head>
　　<meta http-equiv="content-type" content="text/html; charset=utf-8" />
　　<title>{{ page.title }}</title>
</head>
<body>

　　{{ content }}

</body>
　　</html>
```
Jekyll使用Liquid模板语言，`{{ page.title }}`表示文章标题，`{{ content }}`表示文章内容，更多模板变量请参考官方文档。
目录结构变成：
```
 /jekyll_demo
　　|--　_config.yml
　　|--　_layouts
　　|　　　|--　default.html
```
