---
title: hexo搭建博客指南
date: 2016-09-06 10:29:08
tags: hexo
categories: hexo搭建博客教程
---

由于码农的身份，一直在跟电脑打交道，对于思考以及人之间的交流，有所欠缺，心里一直想用自己所学的技术搭建一个有特色的博客系统，去年尝试了，但是没有结果，无疾而终。

## 契机
心中的萌念一直在，觉得要写博客，一是锻炼自己写作能力，二是总结自己所学所得，虽然水平很菜，三是卖自己。

## 少废话，上菜
博客是由hexo+node.js+github pages搭建。通过自己的捣鼓，基本像样了，所以把搭建的过程分享出来。这里只介绍window平台，因为木有mac。

## 参考
整个过程，参考[ixirong](http://www.ixirong.com/2015/05/17/how-to-build-ixirong-blog/) 的文章，谢谢。本文主要提供的，都是官网的资料，然后自己动手，遇到问题，可以google一下或者百度一下。

## 环境准备
安装[git](https://git-scm.com/)和[node.js](https://nodejs.org),我都是安装的最新版本。
验证安装成功与否：
git

![](http://photos.zhangzemiao.com/git_version_2.png)
node.js

![](http://photos.zhangzemiao.com/npm_version.png)

## 创建github pages
github有[参考教程](https://pages.github.com/)，简单来说，创建一个[username].github.io，然后新建一个index.html页面，这样就可以使用这个github提供给你的域名username.github.io,来访问。

## 设置SSH
创建git的ssh key，参照[这里](https://help.github.com/articles/generating-an-ssh-key/)。我提供的都是官网的帮助文档，当然百度也会有相关的信息。
生成key后，将其放入当前用户的.ssh/文件夹下面
![](http://photos.zhangzemiao.com/ssh_position.jpg

将创建的public key添加至github pages项目，记得包括read和write权限。

![](http://photos.zhangzemiao.com/%E6%B7%BB%E5%8A%A0KEY.jpg)

设置完毕，后续就可以直接将hexo的静态博客deploy该github库。

## 下载hexo
[这里](https://hexo.io)是hexo的官网，基本操作有很多详细介绍，包括多平台的。
举一个例子说明吧：
- 在D盘目录下创建一个blog文件夹
- CMD指向该目录
![](http://photos.zhangzemiao.com/blog_directory.jpg)
- 执行"npm install -g hexo-cli"
- 执行"hexo init"，这样初始的blog基本创建好了。[这里](https://hexo.io/docs/setup.html)有相关文件和文件夹说明

## 部署与调试
[这里](https://hexo.io/docs/commands.html)有常用命令的介绍，包括调试等。
[这里](https://hexo.io/docs/setup.html)有部署详细的介绍。

## 更换主题
主题，即使整个博客网站的框架，选择了一个主题，网站的样子基本就固定，因为我不是UI前端工程师，所有不会做大的改动。如果你会CSS，js，node.js，可以自己再定制个性化的样式。
[这里](https://hexo.io/themes)，有很多官方提供的主题，可供选择。
我选择的是[Maupassant](https://github.com/tufu9441/maupassant-hexo)。
<table><tr><td bgcolor=#C0C0C0>CMD至D:/blog
$ git clone https://github.com/tufu9441/maupassant-hexo.git themes/maupassant
$ npm install hexo-renderer-jade --save
$ npm install hexo-renderer-sass --save
</td></tr></table>

然后将blog目录下的_config.yml，把theme改成 maupassant。

## 域名
我之前在阿里云买过，也备过案。。。国内就是比较麻烦。如果想设置自己的域名，来替代*.github.io。建议在国外域名商买一个，也免得备案什么的。
以我阿里云的配置为例，简单说明一下，

CNAME配置：

![](http://photos.zhangzemiao.com/DNS.jpg)

添加CNAME文件至source目录，CNMAE文件中写上你购买的域名:

![](http://photos.zhangzemiao.com/CNAME_FOLDER.jpg)

hexo每次部署，会清除github库里的所有东西，为了避免每次都要手动添加CNAME文件至库中，将CNAME文件放置source文件夹下。

这样每次部署完后，可以用新域名直接访问。

## 写文章
[这里](https://hexo.io/docs/writing.html)有详细介绍。我基本忽略了layout的设置，使用默认的。
主要使用两个命令：
<table><tr><td bgcolor=#C0C0C0>hexo new [文章名]
hexo new page [页面名]
</td></tr></table>

创建好后，会在source目录生成对应的md文件，后续使用markdown进行编辑，创作了。

## 图库及CDN
关于图库，网上推荐使用[七牛云](https://portal.qiniu.com)，我刚使用赶脚很好，免费就是好呀。把图片放在七牛云上，然后文章外链使用。

![](http://photos.zhangzemiao.com/tuku.jpg)

在七牛云的账号中，冲入10元，便可以使用CND了。刚刚客户还打电话过来回访，询问我，为嘛使用七牛云。。。
- 自定义域名
![](http://photos.zhangzemiao.com/blog_cnd.jpg)
- 等待七牛云创建CNAME
![](http://photos.zhangzemiao.com/blog_qiniuyun_cname.jpg)
- 配置自定义域名至七牛云的CNAME(我使用的是阿里云)
![](http://photos.zhangzemiao.com/blog_photos_cnd.jpg)

*注意：配置CNAME时，填写七牛云CNAME值时，后面补上一个"."。*


## 总结
至此，大部分内容就是这些了。可能会有遗漏。剩下的，就是使用markdown写文章了，我还在边写边学的阶段，努力，后续小细节的东西慢慢补充。推荐一款markdown编辑软件: [haroopad](http://pad.haroopress.com/)，免费的。