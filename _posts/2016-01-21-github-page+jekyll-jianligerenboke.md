---
layout:     post
title:      github page + jekyll建立个人博客
subtitle:   专注于博客本身
date:       2016-01-23
author:     Shao Guoji
header-img: img/post-bg-github-page.jpg
catalog:    true
tag:
    - 博客
---

折腾了一两天，终于把自己的独立博客给建了起来，看了一下MarkDown的教程，尝试用Subline Text2写下这第一篇文章，记录一下建博的过程，给同样想建博的童鞋作为参考（虽然网上类似教程一抓一大把），也算给自己留个纪念。

![博客主页](http://img1.buy.ijinshan.com/weibo_img/2016/1/22/23/29/r1453476588695770489227.png "我的独立博客")

*本文只是实现“博客建立起来能正常访问”的程度，仅记录重点步骤，至于一些操作细节以及之后的博客内容美化，自己探索吧~~~* 

---

### github page & jekyll

github不用多说，程序猿的Facebook，博客实际上就是一个网站，众所周知，建网站需要域名、空间的，而github page充当的就是空间的作用，而且重点是它**完全免费！！！**

至于jekyll，是基于ruby的一个博客系统，可生成博客的静态网页，更是号称**只需几秒钟就能让网站跑起来**，不需要懂代码，一样能建站！（所以建博客没你想象的那么高大上不是么。。。）

**整体思路如下：**

1. 注册github账号，准备好github page
2. 用jekyll生成一堆的网站代码
3. 把jekyll生成的代码搬到github上

---

**你将需要：**

* 一个github账号（可到[github](https://github.com/)官网注册）
* github for windows客户端（用于同步代码）
* ruby安装包（实现gem命令下载安装jekyll）
* jekyll

OK废话不多说，开工！

---

# **一、注册github账号、新建仓库**

登录[github](https://github.com/)官网，然后点击……额。。。这个你们会的，都不是小孩了，不废话了。。。

创建好账号后先去验证一下邮箱，接着进到你的主页（如下图），

![github主页](http://img1.buy.ijinshan.com/weibo_img/2016/1/22/23/42/r1453477330809917755772.png)

点击“+New repository”新建一个仓库（所谓“仓库”就是放代码的地方啦）

![new](http://img1.buy.ijinshan.com/weibo_img/2016/1/23/9/31/r1453512667237810367715.png)

注意仓库命名格式是**username.github.io**，其中的username是注册github时的用户名，只有这样命名github才能识别为github page（一个用户只能拥有一个github page）。再点下面的"Create repository"完成新建仓库。

![new Create](http://img1.buy.ijinshan.com/weibo_img/2016/1/23/9/33/r1453512804114599754817.png)

# 二、使用github for windows同步代码

先下载安装[GitHub_2_11_0_5离线安装包以及文件下载链接](http://pan.baidu.com/s/1eQYZQQu)

打开github for windows并登陆github账号，在右上角的设置图标中找到“option”，**在“configure git”中输入github账号和邮箱配置一下git**（不然无法commit代码），点下面“Udate”完成设置。

![option](http://img1.buy.ijinshan.com/weibo_img/2016/1/23/14/25/r1453530346466632670866.png)

![configure git](http://img1.buy.ijinshan.com/weibo_img/2016/1/23/14/57/r145353227826128946700.png)

接着在左上角找到一个“+”，点Clone，选择刚建好的“username.github.io”仓库，再点下面的“Clone username.github.io”，选择存放路径，把仓库下载同步到本地。

![clone](http://img1.buy.ijinshan.com/weibo_img/2016/1/23/14/33/r1453530825104362807973.png)

本地打开刚刚设置的保存仓库的文件夹（即“Clone username.github.io”目录），往里面随便扔个html文件，并**命名为index.html**（否则显示不出来）。回到github for windows界面，会发现有“Uncommitted changes”，点击“show”按钮显示细节

![Uncommitted change](http://img1.buy.ijinshan.com/weibo_img/2016/1/23/14/53/r1453531998703290822177.png)

输入summary(摘要)后点“Commit to master”提交代码，最后**点右上角中间的“Push/Sync”**同步代码到github。

![commit](http://img1.buy.ijinshan.com/weibo_img/2016/1/23/15/1/r1453532462470787609678.png)

![push](http://img1.buy.ijinshan.com/weibo_img/2016/1/23/15/9/r1453532971440809956140.png)

>如果你想潇洒一点，打开github for windows自带的Git Shell(其实就是powershell)，cd进入到仓库的”username.github.io“目录，然后依次输入以下三条命令同样可实现提交并同步代码（事实上我更喜欢这样做）。

```~ $ git add --all```

```~ $ git commit -m "Summary"```

```~ $ git push -u```


再回到仓库网页setting，刷新一下即可看到github page已经自动生成了，域名为“http://username.github.io",如果刚刚有放index.html的话就可以显示其内容了。

![github page](http://img1.buy.ijinshan.com/weibo_img/2016/1/23/15/18/r1453533500966502326086.png)

![side](http://img1.buy.ijinshan.com/weibo_img/2016/1/23/15/23/r1453533805662110534959.png)

这样你已经有了自己的网站了，如果仅仅是想把一个html网页发布的话，看到这里已经够了，但对于博客来说，好戏才刚刚开始。

# 三、jekyll的安装与使用

jekyll(中文名：杰克尔，读音：把”Michael Jackson“中间两个音节倒过来念)是基于ruby的一个博客系统，用户快速生成博客所需的静态页面。

我是用Git Shell命令行下gem命令来安装，需先安装[ruby](http://rubyinstaller.org/downloads)，注意安装时要选择**“Add Ruby executables to your PATH”**

![ruby](http://img1.buy.ijinshan.com/weibo_img/2016/1/23/22/50/r1453560632755056508260.png)

安装完成后，重启Git Shell，输入`gem install jekyll`命令安装jekyll。

失败了？很正常，由于网络环境原因，访问国外服务器十分坑爹，不过我们可以使用某宝的ruby gems镜像http://ruby.taobao.org/（我也被吓到了，某宝居然还有这功能~）

**更改gems源：**

```~ $ gem sources --remove https://rubygems.org/```

```~ $ gem sources -a https://ruby.taobao.org/```

```~ $ gem sources -l```

再次运行”gem install jekyll“安装命令即可。

安装好jekyll后，可以输入”jekyll -h“命令测试一下。

![test jekyll](http://img1.buy.ijinshan.com/weibo_img/2016/1/23/16/17/r1453537059517822656980.png)

然后就可以用以下命令新建博客：

```~ $ jekyll new myblog```

进入博客目录：

```~ $ cd myblog```

运行博客服务器（为了进行本地预览）：

```~ $ jekyll serve```

![jekyll](http://img1.buy.ijinshan.com/weibo_img/2016/1/23/16/15/r1453536905531648574552.png)

打开浏览器，输入[http://localhost:4000](http://localhost:4000)就能在本地预览博客主页了。

# 四、同步博客到github
最后一步，把”myblog“目录下所有文件搬到仓库的”username.github.io“文件夹下，同步到github，就可以通过"http://username.github.io"访问博客了！

**开始你的博客美（zhuang）妙（bi）之旅吧！！！**

![myblog](http://img1.buy.ijinshan.com/weibo_img/2016/1/23/16/24/r1453537462187443227423.png)

>参考文章： 
> 
> * [用jekyll和github Pages写博客](http://my.oschina.net/laichendong/blog/499224)
> * [jekyll中文官网](http://jekyll.bootcss.com/)













