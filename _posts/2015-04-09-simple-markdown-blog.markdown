---
published: true
title: 快速实现markdown博客(1)
layout: post
tags: [jekyll]
categories: [web]
---
### Jekyll
就像习惯了shell环境连鼠标都不想拿一样，一直不想用通用博客繁琐纠结的编辑器，觉得用markdown写博客最简单方便。以前自己在vps上用python简单粗暴的实现了一个，现在准备迁到github上去，顺便记录一下过程。

github使用 [jekyll](http://jekyll.bootcss.com/) 来生成静态网站，它是一个静态网站的生成器，能够将你写的markdown等文本转换成静态的html，使用的html模板引擎是  [liquid](https://shopify.github.io/liquid/)。

jekyll支持的文章分类方式有三种：时间，类别，标签。在_posts文件夹下的文章以YYYY-MM-DD-[title].markdown命名，在parse的时候很自然就会得到时间，而类别和标签需要在markdown文章的开头定义,一个简单的例子：

```
---
published: true
title: 快速实现markdown博客(1)
layout: post
tags: [jekyll]
categories: [web]
---
```

具体使用参考 [jekyll](http://jekyll.bootcss.com/) 和 [liquid](https://shopify.github.io/liquid/)。

### 自己搭建jekyll服务器

jekyll是用ruby写的，最新版本的jekyll需要ruby2.0以上的版本，使用目前ubuntu的官方源上的ruby2.0版本安装的时候依旧会报错，不知道是什么原因，可以通过其他方式安装:

第一种，通过其他源安装：

```
sudo add-apt-repository ppa:brightbox/ruby-ng
sudo apt-get update
sudo apt-get install ruby2.*
```

第二种，用 [rvm](https://rvm.io/) 安装（推荐）：

```
gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
curl -sSL https://get.rvm.io | bash -s stable
```
安装完毕后，当前用户的home目录下会出现一个.rvm的文件夹，要启用rvm需要先
```
source ~/.rvm/bin/rvm
```
接下来只需要执行:

```
rvm list known    #列出可安装的包
rvm install 2.*   #安装2.×版本的ruby
```

rvm还有其他很多不错的功能，可以查看 [官方文档](https://rvm.io/)。

第三种，[官方网站](https://www.ruby-lang.org/)下载源码，编译安装。

安装好ruby之后，执行`gem install jekyll`就可以安装jekyll了，新建和测试一个jekyll项目也非常简单：

```
jekyll new myproject
jekyll serve
```

然后打开浏览器访问http://localhost:4000就可以看到项目首页,它还会自动监视除了_config.yml之外的文件修改并且自动重新render，调试起来非常方便，当然也可以将直接它部署在vps上提供服务。

### Github部署

github支持两种方式来发布项目，一种是直接创建username.github.io的项目，另一种是在你项目中创建一个gh-pages的分支，github会自动将你的代码发布。如果不想自己动手，还有 [tinypress](http://tinypress.co) 等第三方的github博客应用，你只需要有一个github账号就可以。

提交完项目文件后，就可以通过http://username.github.io来访问了，如果想通过一个独立域名来访问的话，需要做一些配置([参考文档](https://help.github.com/articles/using-a-custom-domain-with-github-pages/)):

首先需要解析A记录，配置两条A记录分别指向```192.30.252.153，192.30.252.154```,如果想让子域名，如www.domain.com也指向它的话，添加一条CNAME记录，让这个域名指向username.github.io即可。

接着需要在github项目的根目录里创建一个名为CNAME的文件，文件内容只需要一行，写入使用的独立域名。
