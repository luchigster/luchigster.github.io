---
layout: post
title: 简单搭建个人技术博客
date: 2016-03-27
author: luchigster
comments: false
categories: [Blog]
tags: [Jekyll,GitHub,GitPages,gh-pages,pages]
excerpt: 安装jekyll搭建GitPage建站环境
---

### 第一步
创建`GitPages`能识别的`GitHub`仓库，命名加后缀`.github.io`，详细操作步骤参考[官网说明](https://pages.github.com/)。

### 第二步
提交博客源代码。

如果需要快速搭建博客，可以用[Way Lau](http://waylau.com)同学的模板。

```
git clone https://github.com/waylau/jekyll-bootstrap-blog.git
```

然后修改个人信息，提交到自己的仓库。

### 第三步
搭建本地运行环境，当然如果不需要本地运行`Jekyll`，这步就可以省了，`GitPages`会自动编译源代码的。

#### OSX环境
很简单，依次执行以下命令即可。

```
git clone https://github.com/luchigster/luchigster.github.io.git #克隆源代码
gem install jekyll          #安装jekyll环境 
gem install jekyll-paginate #安装分页插件
gem install rdiscount       #安装Markdown解析插件       
jekyll serve                #启动本地预览
```

这是官方的[快速入门](https://jekyllrb.com/docs/quickstart/)，安装过程也比较简单。

#### Windows环境
由于缺少Ruby环境，请按照这个[安装手册](http://jekyll-windows.juthilo.com/)完成。

1. 下载安装[Ruby for Windows](http://rubyinstaller.org/downloads/)，官方建议采用最新。
   至于是32位还是64位，自己选择。
   官方也说了：64位的`Ruby`没有32位的完善，很多包还没迁移过去，有些还不兼容。
2. 下载解压[Ruby DevKit]，然后

   ```
   cd C:\RubyDevKit
   ruby dk.rb init
   ruby dk.rb install
   ``` 
3. 随后的步骤跟OSX上是一样的：

   ```
   git clone https://github.com/luchigster/luchigster.github.io.git #克隆源代码
   gem install jekyll          #安装jekyll环境 
   gem install jekyll-paginate #安装分页插件       
   jekyll serve                #启动本地预览
   ```

虽然安装`Ruby`环境也很简单，只是没玩过`Ruby`的同学就有些抽筋，有兴趣最好还是学习下这门语言。