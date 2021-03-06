---
title: 使用Hexo搭建自己的博客
date: 2018-04-21 10:00:00
categories: 工具
tags: [hexo, github]
toc: true
description: 搭建个性化的博客网站，我们需要用到hexo和github。其中Hexo是一个快速、简洁且高效的博客框架，使用hexo我们可以快速编写、生成、预览和部署博客文章。最终我们的博客文章发布到github上。Github为个人博客提供了免费的发布平台，通过域名yourname.github.io可以访问到同名的github项目。
mathjax2: true
comments: true
---

## 概述

搭建个性化的博客网站，我们需要用到hexo和github。其中Hexo是一个快速、简洁且高效的博客框架，使用hexo我们可以快速编写、生成、预览和部署博客文章，最终我们的博客文章发布到github上。Github为个人博客提供了免费的发布平台，通过域名yourname.github.io可以访问到同名的github项目。

## 准备工作

### 安装Git

可以去[这里](https://gitforwindows.org/)下载Git的windows版本，正确安装后可以在Shell中查看到git版本

```shell
$ git --version
git version 2.7.2.windows.1
```

### 安装Node.js

去[nodejs官网](https://nodejs.org/en/)下载最新版本(node-v8.11.1-x64.msi)并安装，正确安装后可以查看到版本

```shell
$ node -v
v8.11.1
```

### 安装Hexo

node.js中带了包管理工具npm，后面的模块都可以通过npm来安装，-g表示全局安装

> 官网上的文档中只安装了hexo-cli，试了一下有问题，这里还是安装hexo(里面包含hexo-cli)
>

```shell
$ npm install -g hexo
$ hexo version
hexo: 3.7.1
hexo-cli: 1.1.0
```

### 创建Github项目

- 注册github账户，并做好配置
- 创建以用户开头，以github.io结尾的项目，例如：我的用户名是waterlu，那么创建的项目名称为

```shell
waterlu.github.io
```

项目创建成功后访问 https://waterlu.github.io/ ，后面就是部署页面的操作了。



## Hexo基本操作

Hexo是静态化的博客框架，使用markdown格式编写文章，然后通过hexo generate指令生成静态的HTML页面，最后通过hexo depoly指令将HTML页面上传到GitHub上。

> 所有hexo操作都需要在hexo init的目录执行，也就是含有_config.yml和source的目录，在其他目录执行hexo generate等操作是无效的。
>

### 初始化

- hexo init 指令创建新的博客

```shell
$ cd /c/lu/blog
$ hexo init
$ ls -l
-rw-r--r-- 1 _config.yml
drwxr-xr-x 1 node_modules/
-rw-r--r-- 1 package.json
drwxr-xr-x 1 scaffolds/
drwxr-xr-x 1 source/
drwxr-xr-x 1 themes/
```

- 配置_config.yml，配置个性化信息和部署地址

```properties
title: Waterlu's Blog
description: 个人技术博客
author: Water Lu

deploy:
  type: git
  repo: git@github.com:waterlu/waterlu.github.io.git
  branch: master
```

### 生成网站

```shell
$ hexo generate
```

### 本地预览

- 启动web服务
- 访问http://localhost:4000

```shell
$ hexo server
```

### 部署到Github

- 第一次部署前安装hexo-deployer-git
- 以后通过deploy部署（事先需要在_config.yml中配置github相关信息）

```shell
$ npm install hexo-deployer-git --save
$ hexo deploy
```



## 个性化

### 定制主题

我使用的主题是Maupassant，感觉比较简洁。

>  注意：npm install的两个模块不是全局安装的，每次hexo init创建新项目后都得执行(npm install -g才是全局安装)。
>

```shell
$ git clone https://github.com/tufu9441/maupassant-hexo.git themes/maupassant
$ npm install hexo-renderer-pug --save
$ npm install hexo-renderer-sass --save
```

修改根目录下的_config.yml，设置theme为maupassant

```yaml
theme: maupassant
```

创建about页面

```shell
$ hexo new page about
```

修改themes/maupassant目录下的_config.yml，去掉RSS页面

```yaml
menu:
  - page: home
    directory: .
    icon: fa-home
  - page: archive
    directory: archives/
    icon: fa-archive
  - page: about
    directory: about/
    icon: fa-user
  # - page: rss
  #   directory: atom.xml
  #   icon: fa-rss
```

### 评论功能

使用Valine评论系统，主页在[这里](https://valine.js.org/) 。

由于Valine是基于[LeanCloud](https://leancloud.cn/) 的，所以必须首先在LeanCloud上注册账号。注册成功后登录LeanCloud创建应用，得到AppId和Appkey，需要在配置文件中使用。具体操作步骤可以参考Valine官方文档。

以上就是全部准备工作，下面开始在Hexo中配置：

- 安装valine

```shell
$ npm install valine --save
```

- 修改themes\maupassant下的_config.yml

```properties
valine: ## https://valine.js.org
  enable: true
  appid: 输入在LeanCloud上得到的AppId
  appkey: 输入在LeanCloud上得到的Appkey
  notify: false
  verify: false
  placeholder: just go go
  avatar: 'mm'
  pageSize: 10
  guest_info: nick,mail,link
```

appid和appkey必须输入，placeholder是文本占位符，可以根据需要修改，其他属性可不修改，保持默认即可。

完成以上配置后，在文章中打开comment功能，就可以在文章后面进行评论了，而且支持匿名评论。

```json
title: 使用Hexo搭建自己的博客
date: 2018-04-21 10:00:00
comments: true
```

最后，可以登录到LeanCloud上维护评论数据，如果你觉得哪条评论不爽，可以删除。

选中应用后，评论在[存储]=>[数据]=>[Comment]中。

### 站内搜索

- 安装hexo-generator-search

```shell
$ npm install hexo-generator-search --save
$ npm install hexo-generator-searchdb --save
```

- 修改根目录下的_config.yml，增加如下配置

```properties
search:
  path: search.xml
  field: post
  format: html
  limit: 10000
```

- 修改themes\maupassant下的_config.yml，打开self_search

```properties
google_search: false ## Use Google search, true/false.
baidu_search: false ## Use Baidu search, true/false.
swiftype: ## Your swiftype_key, e.g. m7b11ZrsT8Me7gzApciT
tinysou: ## Your tinysou_key, e.g. 4ac092ad8d749fdc6293
self_search: true ## Use a jQuery-based local search engine, true/false.
```

### 访问统计

- 修改themes\maupassant下的_config.yml，打开不蒜子
```properties
busuanzi: true ## If you want to use Busuanzi page views please set the value to true.
```

### 流程图和序列图

Hexo默认不支持markdown的flow图和sequence图，需要安装插件

- 进入项目根目录安装插件

```shell
$ npm install --save hexo-filter-flowchart
$ npm install --save hexo-filter-sequence
```

- 修改项目的_config.yml，增加配置

```properties
flowchart:
  # raphael:   # optional, the source url of raphael.js
  # flowchart: # optional, the source url of flowchart.js
  options: # options used for `drawSVG`

sequence:
  # webfont:     # optional, the source url of webfontloader.js
  # snap:        # optional, the source url of snap.svg.js
  # underscore:  # optional, the source url of underscore.js
  # sequence:    # optional, the source url of sequence-diagram.js
  # css: # optional, the url for css, such as hand drawn theme
  options:
    theme:
    css_class:   
```



## Hexo日常操作

### 写文章

- [layout]默认为post，指的是文章的布局类型，在_config.yml中可以配置
- 如果[layout]=post，那么新文章将生成到source/_posts目录下

```shell
$ hexo new [layout] <title>
```

- 文章开头"---"之间的部分称为Front-matter，主要包括以下信息

| 参数         | 描述     |
| ---------- | ------ |
| layout     | 文章布局   |
| title      | 文章标题   |
| date       | 文章发布时间 |
| updated    | 文章更新时间 |
| tags       | 标签     |
| categories | 分类     |

### 发表文章

```shell
$ hexo new post [title]
$ hexo clean
$ hexo generate
$ hexo server
$ hexo deploy
```

### 常见问题

#### *解析错误*

- 当title.md文章中出现无法解析的markdown语法时，报错Template render error
- 因为需要把markdown转成html，所以必须解析正确

```shell
$ hexo g
INFO  Start processing
FATAL Something's wrong. Maybe you can find the solution here: http://hexo.io/docs/troubleshooting.html
Template render error: expected variable end
```

