---
layout:          post
title:           搭建码云 CI 和 CLA 签署服务（一）
subtitle:        开源项目自动化构建与协议签署
date:            2020-06-23 15:49:29 +0800
author:          Shao Guoji
header-img:      img/post-bg-gitee-cicla-1.jpg
catalog:         true
tag:
    - Web
    - CI/CD
    - 笔记
---

和编码不同，在生产力工具上付出努力当然是值得的。

这几周基于 Jenkins 及 openEuler/ci-bot 搭建了 RT-Thread 码云 CI 和 CLA 服务，为社区接受 bsp 代码贡献做准备。由于这方面现有的资料有限，没太多前人经验参考，摸爬滚打后决定写下点什么。

### 概述 & 文章目录

开源项目接触久了，CI 和 CLA 最终是绕不开的。Github 的 CI 有原生 `Github Actions` 或 `Travis CI`，使用配置起来都很香，通过第三方的 `cla-assistant` 检查 CLA 签署，也是轻轻松松。

但到了 gitee，麻烦就来了，没有内置 CI，不被 CLA 平台支持。有点难搞，但也不是没有办法。

准备一架 Linux 服务器搭建 CI，使用 Jenkins 配合 Gitee 插件，配置虽然麻烦点，但好在效果还不错。

而 CLA 签署这块，翻遍中文互联网，一片空白。国外的许多开源方案，仅支持 Github，直到发现 openEuler 社区已经在实践，并且开源了他们的机器人 `ci-bot`，才觉得有戏。

因此，本系列文章将介绍如何在码云搭建 CI 构建和 CLA 签署服务，共分为三篇：

[搭建码云 CI 和 CLA 签署服务（一）- 开源项目自动化构建与协议签署](http://www.shaoguoji.cn/2020/06/23/build-gitee-ci-cla-check-service-1/)

[搭建码云 CI 和 CLA 签署服务（二）- Jenkins 安装配置与部署](http://www.shaoguoji.cn/2020/07/01/build-gitee-ci-cla-check-service-2/)

[搭建码云 CI 和 CLA 签署服务（三）- CLA 签署检查服务搭建](http://www.shaoguoji.cn/2020/07/02/build-gitee-ci-cla-check-service-3/)

这篇先大体介绍一下我们要干嘛。

### 持续集成

#### CI & 自动构建

持续集成（Continuous integration，简称 CI），是互联网软件的一种开发模式，指不断把小更新自动化地构建、测试、集成到主干。在嵌入式软件开发中，公司内部 Gitlab 上很早就开始使用 CI，对提交的 C 代码和 Markdown 文档进行编译构建、输出产物（`.bin`、`.a`、`.pdf` 等）,偶尔还会通过 Python 脚本完成简单的打包发布工作。

与互联网软件不同，嵌入式软件的 CI 主要完成自动编译检查工作，并及时将结果反馈回提交者，以便开发人员在合并代码前发现错误，至少确保代码能顺利编译通过。

例如 RT-Thread Github 仓库中的每一个 PR 都会触发 CI 构建，所有 PR 的构建信息在 Travis CI 页面中展示：[Pull Requests - RT-Thread/rt-thread - Travis CI](https://travis-ci.org/github/RT-Thread/rt-thread/pull_requests)。

项目维护着与贡献者，针对 CI 构建结果进行下一步工作。换作码云仓库也是如此，只不过没有了 Travis CI，需要自己搭建 CI 平台，并完成与码云之间的状态同步。

**CI 平台的搭建，说白了就是准备执行编译用的机器，能接收构建命令，分配管理任务。**

#### Jenkins & 码云插件

Jenkins 似乎成了 CI/CD 的最佳实践，安装好 Jenkins 的服务器，能够添加 Git 仓库，并接受推送或 PR 请求，执行一系列自定义命令，完成编译构建打包等操作，并把结果产物输出。

同时 Jenkins 还配备清晰的 Web 界面，使用和管理极为方便，各项目构建情况一目了然，用起来效果像这样：[Dashboard [Jenkins]](https://jenkins.osmocom.org/jenkins/)。

Jenkins 一大特色是支持插件的使用，通过安装不同的插件组合灵活完成各项任务。早在 17 年 9 月，码云就发布了开源 Jenkins 插件，从此为 Gitee 项目添加 CI 便成为可能。

> Jenkins 码云 WebHook 插件。基于该插件，用户能通过码云系统提供的 WebHook 功能，通知你的 Jenkins 服务进行项目的构建、打包、部署等自定义行为。    
> 
> *来源：[码云 Jenkins 插件正式上线公测（已开源） - OSCHINA](https://www.oschina.net/news/88885/gitee-jenkins-plugin)*

码云 Jenkins 插件基于 WebHook 工作。所谓「某某 Hook」，是在不影响原业务逻辑的前提上，给中间流程挂上额外的「钩子」，获取其中的数据，并触发用户自定义的操作，程序正常运行的同时被「插了一手」。

码云 Webhook 支持代码推送、PR、评论等动作触发，配置好构建触发器和仓库 WebHook 后，推送代码到码云时，由配置的 WebHook 触发 Jenkins 任务构建，并将构建状态反馈回码云平台。

### CLA 签署服务

#### 什么是 CLA

> CLA 的全称是 Contributor License Agreement（贡献协议许可）。如果一个人要贡献代码给一个开源项目，比如 Google TensorFlow，那么她需要在 GitHub 上签署 Google 发布的 Individual CLA。这个签署过程很简单，贡献者只需输入自己的姓名和电子邮件等可以标志自己身份的信息即可。如果一个贡献者没有签署CLA，那么这个贡献者发给 Google TensorFlow 项目的修改（Github 的术语称为 pull request）会被 Github 标志为不可被接受。签署了之后，这个 pull request 就是可以在 review 之后被接受的了。
> 
> *来源：[什么是CLA？ - 知乎](https://zhuanlan.zhihu.com/p/68251730)*

CLA 简单来说，就是贡献者对自己代码的授权书，保障开源组织对已合并代码的使用权，防止代码原作者在贡献后反悔耍赖不认账。开发者一旦签署 CLA，就意味着将代码版权授权给项目组织者和所有使用者。

越来越多的大型开源项目，要求开发者在提交代码前，必须签署 CLA，也侧面体现人们版权意识与法律意识的提高。

#### CLA 签署流程

大多数 CLA 签署都在线上进行（打印一份实体协议邮寄，理论上也行） ，大致流程如下：

1. 开源组织者在收到 PR 后，检查贡献者账号是否已经签署 CLA
2. 若已经签署，则跳过签署流程，继续完成 PR 审查。
3. 若未签署，则在 PR 页面给出贡献提示说明，提供 CLA 签署页面链接
4. 在 CLA 签署页面中，至少包含协议主体文本，个人信息填写，签署按钮
5. 贡献者填写信息并提交签署请求，开源组织接收并核验身份完成签署
6. 重新检查签署情况，PR 页面提示签署成功

无论是自己搭建服务，或是由第三方签署平台完成 CLA 签署，都应包含上述步骤。一般后者已经完成绝大部分的工作，往往只需要用户进行签署即可。而在码云需要自行搭建，考虑更多技术细节。

更具体点说，想要从头搭建 CLA 签署服务，需要具备以下条件：

* 协议内容展示签署页面
* gitee 账号获取绑定（通过邮箱验证身份）
* 用户数据存储
* 后端服务程序（oAuth接口、签署信息管理，用户操作判断，PR 评论反馈等逻辑）

开源组织 OpenEuler 已经在码云实现了完整 CLA 签署服务，页面链接：[https://openeuler.org/en/cla.html](https://openeuler.org/en/cla.html)

#### ci-bot 服务端程序

贡献者提交 PR 后，在页面下方由官方机器人账号自动回复，协助作者完成 CLA 签署类似电商智能客服，实际效果如下图：

![图1 ci-bot](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/20200704165033.png)

签署 CLA 剩下的工作都由后端程序处理，值得庆幸的是，OpenEuler 同时也开源了所用的 CLA 签署机器人 ci-bot，使用 Go 开发，各项配置整理清晰，稍作修改便能使用，项目地址如下：

[ci-bot: This repository is used to address the code of openEuler ci bot.](https://gitee.com/openeuler/ci-bot)

![](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/20200707100820.png)

*ci-bot 原项目功能众多，而我们只需要使用其中的 CLA 签署检查功能*

ci-bot 内置了码云 SDK，能通过 API 获取仓库信息、模拟用户行为，通过 Webhook 监听项目变动，同时接收 CLA 签署请求并校验，通过配置 MySQL 数据库，完成数据存储。并可通过 Dokcer 部署，独立运行。

### 部署上线

在服务器搭建好了 Jenkins 和 ci-bot，最后配置项目 Webhook，便能从 Push 和 PR 提交触发 CI 构建及 CLA 检查。编写项目 Jenkinsfile 创建 pipeline，即可定义具体构建方式。

总的来说，Jenkins 构建和 CLA 签署机器人独立运行，后端服务程序通过 Gitee token 和 Webhook 关联码云账号，调用评论接口反馈消息。

如果你看完这篇还是一头雾水，很正常，别担心，目前只需要了解个大概就行，至于具体怎么做，后面的文章会详细展开。

> 参考资料
> 
> * [码云 Jenkins 插件正式上线公测（已开源） - OSCHINA](https://www.oschina.net/news/88885/gitee-jenkins-plugin)
> * [Jenkins 插件 - 码云 Gitee.com](https://gitee.com/help/articles/4193#article-header6)
> * [什么是CLA？ - 知乎](https://zhuanlan.zhihu.com/p/68251730)
> * [ci-bot: This repository is used to address the code of openEuler ci bot.](https://gitee.com/openeuler/ci-bot)
> * [Github CLA 签署机器人 - 谢先斌的博客](https://www.xiexianbin.cn/git/github/2017-08-09-github-cla/index.html)
> * [签署CLA](https://openeuler.org/zh/cla.html)