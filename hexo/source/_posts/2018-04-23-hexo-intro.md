---
title: 使用Hexo搭建自己的博客
date: 2018-04-21
categories: 工具
toc: true
description: 搭建个性化的博客网站，我们需要用到hexo和github。其中Hexo是一个快速、简洁且高效的博客框架，使用hexo我们可以快速编写、生成、预览和部署博客文章。最终我们的博客文章发布到github上。Github为个人博客提供了免费的发布平台，通过域名yourname.github.io可以访问到同名的github项目。
mathjax2: true
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
