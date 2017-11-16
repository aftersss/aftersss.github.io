---
layout: post
title:  "gitlab-10代码树插件for chrome发布"
date:   2017-11-01 12:06:12 +0800
categories: technical
author: 黄文奇
keywords: gitlab, tree, chrome
description: gitlab-10代码树插件for chrome发布
---

> 终于有可用的gitlab 10代码树插件了！

# 简述
用过github的开发同学们可能都很喜欢一款名为：Octotree的chrome插件，
该插件可以在从网页里打开一个github项目时在左边呈现该项目的树形结构，可以逐层展开，可以点击某个文件从而查看该文件的内容。
这个插件尤其好的一点是点击文件后页面不会刷新，左边的树形结构不会消失。这在代码还没下载到本地时浏览代码非常方便。
<br/>

但是使用gitlab就没这么方便了，我们公司使用的是GitLab Community Edition 10.x，我尝试在网上搜索，发现目前只有针对gitlab 8.*和gitlab 9.*版本的代码树插件，试用了下发现在gitlab 10下都不可用，
故本人抽空开发了针对gitlab 10的代码树插件，项目主页：https://github.com/aftersss/gitlab-tree

# 安装方法
1. 把项目整个下载下来，解压后找到根目录下的gitlab-tree.crx文件
2. 打开chrome扩展页面(chrome://extensions/)
3. 把gitlab-tree.crx文件拖到chrome扩展页里，确认安装

# 使用方法
1. 打开gitlab网页
2. 如果是第一次打开gitlab网页，插件会检测到并弹出一个确认框，请仔细阅读后点击确认（这个步骤只有首次需要）
3. 导航到一个具体的项目页面中，你会在左上角看到一个箭头，点击展开即可看到代码树,如下所示：
<img src="https://github.com/aftersss/gitlab-tree/raw/master/docs/gitlab-tree.png"/>

# 其他说明
1. 本插件只针对gitlab 10。
2. 欢迎提issue，欢迎fork。
