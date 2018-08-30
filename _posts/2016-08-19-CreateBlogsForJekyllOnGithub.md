---
layout: post
title:  "Win环境Jekyll+Markdown+Github Pages搭建个人Blogs"
categories: Other
description: Win环境Jekyll+Markdown+Github Pages搭建个人Blogs。
keywords: Jekyll, Blogs, Github
---
## 开启Github Pages ##

**1、**在github中建立一个基于你的用户名的repository

![](http://i.imgur.com/pJBemgT.png)

**2、**进入仓库 -> Setting ->找到Launch automatic page generator进入

![](http://i.imgur.com/NX82S2v.png)

**3、**此时相当于一篇博客页，填入必要的字段后，点击Continue to layouts

**4、**选样式 -> Publish page

**5、**之后，你就能在Setting的GitHub Pages下看到你的地址了

![](http://i.imgur.com/RhpshMR.png)

## 部署Jekyll ##

### 安装Ruby、RubyGems ###

**1、**下载[http://rubyinstaller.org/downloads/](http://rubyinstaller.org/downloads/ "RubyInstallers")安装

**2、**配置环境变量：
-  RUBY_HOME=D:\OfficeSoft\Ruby23-x64（目录地址）
-  在Path中添加 %RUBY_HOME%\bin
-  进cmd， `ruby -v`，如果显示ruby版本信息说明安装成功。

### 安装Jekyll ###

- `gem install jekyll`

注意ruby安装目录存在空格时会报Makefile异常。


### 创建本地blog站点 ###

- cd进入想要存放blog的目录
- `jekyll new YazidChen.Github.io`（目录名自拟）


### 开启站点服务 ###

`jekll serve` 开启服务。可能会报错，一般为缺少包，根据报错做相应处理
```
gem install bundler
gem install minima
```
minima报错时，在\Ruby23-x64\lib\ruby\gems\2.3.0\gems\jekyll-3.2.1\lib\jekyll目录下找到layout.rb，进行如下操作
```
# Replace Line 38 with:

@path = base + "/" + name
```

Could not find gem 'tzinfo-data x64-mingw32' in any of the gem sources listed in your Gemfile.报错处理：
```
bundle
```

### 目录解读 ###

刚生成的YazidChen.Github.io下存在以下结构

![](http://i.imgur.com/xCQ3uNM.png)

- .sass-cache	->	sass编译缓存文件的目录
- _posts		->	存放博文的目录，博文文件类型必须为markdown，文件名统一格式：YEAR-MONTH-DAY-TITLE，如`2016-08-19-welcome-to-jekyll.markdown`或`2016-08-19-welcome-to-jekyll.md`
- _site		->	Jekyll解析完整个网站,会将源代码存在此目录
- _config.yml	->	Jekyll的配置文件


## 使用Jekyll写博文 ##

### 把仓库克隆到本地 ###

**1、**下载[GitHub Desktop](https://desktop.github.com/)

**2、**克隆仓库到本地

![](http://i.imgur.com/iyNOqSe.png)

**3、**拷贝本地Jekyll创建的YazidChen.Github.io的所有子目录及文件到本地仓库中的。

**4、**进入仓库启动jekyll 

![](http://i.imgur.com/c2tOmeS.png)

**5、**github 提交修改并同步


## 参考 ##

本文参考以下文章，在此对原作者表示感谢！

[jekyll 中文文档](http://jekyll.bootcss.com/docs/home/)

[jekyll English Docs](http://jekyllrb.com/docs/home/)

[Jekyll和Github搭建个人静态博客](http://pwnny.cn/original/2016/06/26/MakeBlog.html)

[在github pages网站下用jekyll制作博客教程](http://kresnik.wang/works/tech/2015/06/07/%E5%9C%A8github-pages%E7%BD%91%E7%AB%99%E4%B8%8B%E7%94%A8jekyll%E5%88%B6%E4%BD%9C%E5%8D%9A%E5%AE%A2%E6%95%99%E7%A8%8B.html)

[Jekyll本地搭建开发环境以及Github部署流程](http://pizida.com/technology/2016/03/03/use-jekyll-create-blog-on-github/)




