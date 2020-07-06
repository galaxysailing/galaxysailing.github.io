---
title: hexo总结
categories:
    - 工具
tags:
    - hexo
    - 总结
    - 工具使用
---

## 为什么要写这篇文章

学习可以遵循一个原则：多了解你每天使用的工具。这也是写这篇文章的初心。之前本来写完了个博客后台，但是考虑以后要忙于工作，对于博客前台开发和后期维护可能力不从心，所以选择了hexo这样轻量级的博客构建工具，可能后面很长一段时间，hexo会陪伴我构建博客系统的时光，因此我应该多了解它的内涵。

<!-- more -->

## 如何搭建一个hexo博客

搭建博客之前需要安装hexo，hexo依赖于node和git，因此要确保系统上有这两个工具。
> 一个小坑：node版本不能太高，否则执行 `hexo s` 时会报 circular dependency的Warning，这时博客是打不开的。推荐安装node的长期支持版

接下来就是敲几个指令

```shell
$ npm install -g hexo-cli # 安装hexo cli
$ hexo init blog # 创建blog文件夹 并初始化文件目录
$ cd blog 
$ npm install 
```
> 如果`npm install`过慢可以安装淘宝镜像 
`npm install -g cnpm --registry=https://registry.npm.taobao.org`，
再通过`cnpm nstall`来安装依赖包

创建后这是`blog`文件夹下的目录

```shell
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes
```
进入 blog 文件夹目录，使用以下命令

```shell
$ hexo s # 或者 hexo serve
```
通过浏览器访问`http://localhost:4000`就能看到搭建好的博客了

## Github Pages
