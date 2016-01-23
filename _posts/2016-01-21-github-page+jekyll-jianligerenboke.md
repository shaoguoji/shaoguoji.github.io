---
layout: post
title:  "github page + jekyll建立个人博客"
date:   2016-01-21 14:17:36 +0800
categories: jekyll update
---

折腾了一两天，终于把自己的独立博客给建了起来，看了一下MarkDown的教程，尝试用Subline Text2写下这第一篇文章，记录一下建博的过程，给同样想建博的童鞋作为参考，也算给自己留个纪念。


![博客主页](http://img1.buy.ijinshan.com/weibo_img/2016/1/22/23/29/r1453476588695770489227.png "我的独立博客")


*本文只是实现“博客建立起来能正常访问”的程度，仅记录重点步骤，至于一些操作细节以及之后的博客内容美化，我也不太会~~~*

---

###github page & jekyll

github不用多说，程序猿的Facebook，而博客实际上就是一个网站，众所周知，建网站需要域名、空间的，而github page充当的就是空间的作用，而且重点是它**完全免费！！！**

至于jekyll，是基于ruby的一个博客系统，可生成博客的静态网页，更是号称**只需几秒钟就能让网站跑起来**，不需要懂代码，一样能建站！（所以建博客没你想象的那么高大上不是么。。。）

**整体思路如下：**

1.注册github账号，准备好github page
2.用jekyll生成一堆的网站代码
3.把jekyll生成的代码搬到github上

---

**你将需要：**

* 一个github账号（可到[github](https://github.com/)官网注册）
* github for windows客户端（用于同步代码）
* ruby安装包（实现gem命令下载安装jekyll）
* jekyll

OK废话不多说，开工！

---

###**一、注册github账号、新建仓库**

登录[github](https://github.com/)官网，然后点击……额。。。这个你们会的，都不是小孩了，不废话了。。。

创建好账号后先去验证一下邮箱，接着进到你的主页（如下图），


![github主页](http://img1.buy.ijinshan.com/weibo_img/2016/1/22/23/42/r1453477330809917755772.png)


点击“+New repository”新建一个仓库（所谓“仓库”就是放代码的地方啦）


![new](http://img1.buy.ijinshan.com/weibo_img/2016/1/23/9/31/r1453512667237810367715.png)


注意仓库命名格式是**用户名.github.io**,用户名是注册github时的用户名，只有这样命名github才能识别为github page（一个用户只能拥有一个github page）。再点下面的"Create repository"完成新建仓库。


![new Create](http://img1.buy.ijinshan.com/weibo_img/2016/1/23/9/33/r1453512804114599754817.png)







