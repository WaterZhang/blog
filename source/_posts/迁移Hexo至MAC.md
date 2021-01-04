---
title: 迁移Hexo至MAC
toc: true
date: 2017-02-23 11:25:29
tags:
- hexo
categories:
- hexo
---

这是在Mac上写的第一篇文章。

## Mac平台
最近入手Macbook pro，买的2015年款15寸，2016款，一来贵，二来touch bar还不够实用。

## 迁移Mac

需要安装的软件，

1. git及设置SSH
2. Node.js
3. Xcode(虽不做苹果相关开发，最好下载)
4. Hexo
6. 加载博客的github项目
7. 测试

在加载博客的github时，遇到了一些问题，直接使用git remote加载项目时，不能正常启动hexo，折腾了许久，换成SourceTree工具，关联项目。nodejs相关模块，是没有同步至github的，解决冲突，然后下载各个缺省的模块，顺利搭起博客了。

## 总结

安装完hexo后，总是出现各种问题，各种蒙圈，多尝试，多google，完事。