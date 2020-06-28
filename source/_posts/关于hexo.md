---
title: 关于hexo
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

```
$ npm install -g hexo-cli
$ hexo init blog
$ cd blog
$ npm install
```
创建后这是`blog`文件夹下的目录

```
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes
```