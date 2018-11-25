---
title: Hexo 安装笔记
date: 2018-11-24 14:15:10
author: ACE NI
tags:
  - Hexo Next
categories:
  - 工具记录
---
# 重拾Hexo
过了大半年，线上买的阿里云服务器到期了，又不打算续了，考虑原来的几篇博文还在上面，就做了个迁移，并且重拾了下Hexo。

## 安装Hexo
检查环境配置
```bash
node -V
npm -v
git --version
hexo -v
```
未安装hexo，通过npm安装
```bash
npm install -g hexo-cli
```
<!-- more -->

## 初始化Hexo
```bash
$ mkdir hexo_blog
$ cd hexo_blog
$ hexo init
$ npm install
$ hexo g s
```

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fxjfv002xuj30rs09e3zq.jpg)

默认主题是`landscape`，不是那么高大上，所以切换`next`主题
```bash
$ cd themes
$ git clone https://github.com/iissnan/hexo-theme-next themes/next
$ cd ..
$ vi _config.yml

找到 theme: landscape
修改为 theme: next

$ hexo clean
$ hexo g
$ hexo s
```
`next`主题的定制化这里不再赘述。

## 关联GitHub 和 Coding
```bash
$ vi _config.yml

deploy:
  type: git
  repository:
    github: https://github.com/your/your.github.io.git,master
    coding: https://git.coding.net/your/Hexo_Blog.git,master
```
![](https://ws3.sinaimg.cn/large/006tNbRwgy1fxjiask230j30sq06qq31.jpg)

如存在账号问题可以配置`.git/config`尝试解决。

## 源码上传GitHub hexo分支
```bash
$ git init
开一个hexo新分支
$ git checkout -b hexo
$ git add .
$ git commit -m 'hexo init'
$ git remote add origin https://github.com/your/your.github.io.git
$ git push origin hexo:hexo
```
![](https://ws1.sinaimg.cn/large/006tNbRwgy1fxjhhbvd2gj31b00k2aao.jpg)

`master` 存放发布的博文，`hexo`分支存放源码文件

## 编辑实时预览
通过`hexo-browsersync`和 `hexo s`实现编辑实时预览功能
由于不同markdown工具的可视化预览功能效果各有差异，我选择了网页实时预览看效果。
```bash
$ npm install hexo-browsersync --save
$ hexo s
```
![](https://ws3.sinaimg.cn/large/006tNbRwgy1fxjhmkfnhgj314e0g43z3.jpg)

打开谷歌浏览器 http://localhost:4000/ 即可实时预览。

编辑器选择 `Typora`，开启源码模式，专注模式，打字机模式，不用按`Ctrl + S` 写起来真的很爽。
图片上传选择`iPic`，使用默认的微博图床，有条件可以付费接各种云，微信截图自动压缩上传生成`markdown`图片链接，复制即可插入。

![](https://ws3.sinaimg.cn/large/006tNbRwgy1fxjhsxdpdyj31jn0u0qg1.jpg)
