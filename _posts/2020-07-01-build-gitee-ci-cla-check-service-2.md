---
layout:          post
title:           搭建码云 CI 和 CLA 签署服务（二）
subtitle:        Jenkins 安装配置与部署
date:            2020-07-01 14:19:29 +0800
author:          Shao Guoji
header-img:      
catalog:         true
tag:
    - Web
    - CI/CD
    - 笔记
---

本系列文章将介绍如何在码云搭建 CI 构建和 CLA 签署服务，共分为三篇：

[搭建码云 CI 和 CLA 签署服务（一）- 开源项目自动化构建与协议签署](http://www.shaoguoji.cn/2020/06/23/build-gitee-ci-cla-check-service-1/)

[搭建码云 CI 和 CLA 签署服务（二）- Jenkins 安装配置与部署](http://www.shaoguoji.cn/2020/07/01/build-gitee-ci-cla-check-service-2/)

[搭建码云 CI 和 CLA 签署服务（三）- CLA 签署检查服务搭建](http://www.shaoguoji.cn/2020/07/02/build-gitee-ci-cla-check-service-3/)

本文为第二篇，详细记录 Jenkins 安装配置的步骤及注意事项。

### 一、Jenkins 服务搭建

建议使用 Docker 搭建，干净优雅，平台通杀，免去自己折腾一堆 java 环境（系统需提前安装 Docker 环境）。

#### 常规安装部署

如果仅仅是想跑起来，按照官方文档，一步步走完问题不大。

> [Installing Jenkins](https://www.jenkins.io/doc/book/installing/)

#### dood 安装部署（推荐）

为了能够对每次构建隔离，保持系统环境一致，更好的方式是在新的容器中执行构建，便引出在 docker 中使用 docker 的问题。

##### dind vs dood

> dind(docker in docker)：在容器中安装一个全新的完整的隔离的 Docker 版本，该容器和外部的 Docker 系统完全隔离。
> 
> dood(Docker outside of Docker)：通过加载宿主 Docker socket 程序的方式达成重用宿主镜像的目的。

官方镜像使用 dind 套娃的方式（基于 `docker:dind` 镜像），说到底还只是在同一个容器里面跑，不便于安装编译环境，也不利于针对不同项目制作独立镜像，下面重点介绍 dood 方式搭建 Jenkins。

其中一种方法，就是先挂载 `docker.sock` 文件运行 `jenkins:lts` 镜像，把容器 Docker 和宿主机 Docker 程序关联（映射为同一个程序），然后在容器安装 Docker，这时两个 Docker 便能互相访问。

下面这篇文章很有参考价值：

Jenkins dood 搭建教程：[The simple way to run Docker-in-Docker for CI | Releaseworks Academy](https://tutorials.releaseworksacademy.com/learn/the-simple-way-to-run-docker-in-docker-for-ci)

在上面链接教程的基础上，加上数据持久化（创建、挂载数据卷）,root 权限等参数，运行 Docker 容器。

**1. 创建数据卷：**

```bash
docker volume create jenkins-data
```

**2. 后台启动容器，基于 hub 官方 jenkins 镜像：**

```bash
docker run -d --restart always \
-p 8085:8080 \
-v /var/run/docker.sock:/var/run/docker.sock \
--name jenkins \
--hostname jenkins \
--volume jenkins-data:/var/jenkins_home \
jenkins/jenkins:lts
```

*若宿主机 `8080` 端口被占用，可映射到其他空闲端口号，如上述的 `8085`。*

**3. 进入容器：**

```bash
docker exec -u root -it jenkins bash
```

**4. 更换软件源：**

```bash
sed -i 's/deb.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list
```

**5. 容器内安装 Docker：**

```bash
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```

**6. 验证 dood 环境安装结果**

```
docker stats
```

```
CONTAINER ID        NAME                CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
36ee6f8b0b78        jenkins             0.21%               176.5MiB / 991.8MiB   17.80%              268MB / 6.6MB       0B / 0B             41
```

通过 `docker stats` 能看到容器自身信息，说明 dood 环境准备就绪，服务器访问 `http://jenkins.rt-thread.org/`，正常显示 Jenkins 登录页面，服务部署成功。

*说明：文中的 `jenkins.rt-thread.org` 是已经定向到主机 ip 8085 端口的域名，调试请访问对应的主机 ip 和端口号（如 `xxx.xxx.xxx.xxx:8085`）。*

### 二、Jenkins 基本配置

搭建好 Jenkins 服务之后，需要进行网页端配置并添加码云项目。

访问 Jenkins 前端页面 [https://jenkins.rt-thread.org/](https://jenkins.rt-thread.org/)，等待程序加载完成。

#### Jenkins 初始配置

**复制初始密码解锁 Jenkins**

```bash
# docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
5t3d6e3007b456456gjf95d29f182c1d
```

![复制初始密码解锁 Jenkins](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/%E5%A4%8D%E5%88%B6%E5%88%9D%E5%A7%8B%E5%AF%86%E7%A0%81%E8%A7%A3%E9%94%81%20Jenkins.png)

**插件换源**

进入容器内执行：

```bash
sed -i 's/https:\/\/updates.jenkins.io\/update-center.json/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins\/updates\/update-center.json/g' \
/var/jenkins_home/hudson.model.UpdateCenter.xml 
```

```
sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' \
/var/jenkins_home/updates/default.json && \
sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' \
/var/jenkins_home/updates/default.json
```

浏览器访问 [https://jenkins.rt-thread.org/restart](https://jenkins.rt-thread.org/restart) 重启生效。

**安装推荐插件**

![安装推荐插件](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/%E5%AE%89%E8%A3%85%E6%8E%A8%E8%8D%90%E6%8F%92%E4%BB%B6.png)

![安装推荐插件](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/%E5%AE%89%E8%A3%85%E6%8E%A8%E8%8D%90%E6%8F%92%E4%BB%B6%E4%B8%AD.png)

**创建管理员账号**

```
Username: rtthread
Password: yourpassword
```

![创建管理员账号](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/%E5%88%9B%E5%BB%BA%E7%AE%A1%E7%90%86%E5%91%98%E8%B4%A6%E5%8F%B7.png)

**实例配置，保持默认**

![实例配置](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/%E5%AE%9E%E4%BE%8B%E9%85%8D%E7%BD%AE.png)

进入主界面：

![进入主界面](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/%E8%BF%9B%E5%85%A5%E4%B8%BB%E7%95%8C%E9%9D%A2.png)

#### Gitee 插件安装

1. 在线安装
     - 前往 Manage Jenkins -> Manage Plugins -> Available
     - 右侧 Filter 输入： Gitee
     - 下方可选列表中勾选 Gitee（如列表中不存在 Gitee，则点击 Check now 更新插件列表）
     - 点击 Download now and install after restart

![在线安装](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/%E5%9C%A8%E7%BA%BF%E5%AE%89%E8%A3%85.png)

2. 手动安装（在线安装失败时备选）
     - 从 [release](https://gitee.com/oschina/Gitee-Jenkins-Plugin/releases) 列表中进入最新发行版，下载对应的 XXX.hpi 文件
     - 前往 Manage Jenkins -> Manage Plugins -> Advanced
     - Upload Plugin File 中选择刚才下载的 XXX.hpi 点击 Upload
     - 后续页面中勾选 Restart Jenkins when installation is complete and no jobs are running

![手动安装](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/%E6%89%8B%E5%8A%A8%E5%AE%89%E8%A3%85.png)

3. 勾选安装完成后重启 Jenkins

![勾选安装完成后重启 Jenkins](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/%E5%8B%BE%E9%80%89%E5%AE%89%E8%A3%85%E5%AE%8C%E6%88%90%E5%90%8E%E9%87%8D%E5%90%AF%20Jenkins.png)

4. 使用管理员账号重新登录，完成安装

![使用管理员账号重新登录](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/%E4%BD%BF%E7%94%A8%E7%AE%A1%E7%90%86%E5%91%98%E8%B4%A6%E5%8F%B7%E9%87%8D%E6%96%B0%E7%99%BB%E5%BD%95.png)

#### Gitee 插件配置

**添加码云链接配置**

 - 前往 Jenkins -> Manage Jenkins -> Configure System -> Gitee Configuration -> Gitee connections
 - 在 Connection name 中输入 Gitee 或者你想要的名字
 - Gitee host URL 中输入码云完整 URL地址： https://gitee.com （码云私有化客户输入部署的域名）
 - Credentials 中如还未配置码云 APIV5 私人令牌，点击 Add - > Jenkins
 - Domain 选择 Global credentials
 - Kind 选择 Gitee API Token
 - Scope 选择你需要的范围
 - Gitee API Token 输入你的码云私人令牌，获取地址：https://gitee.com/profile/personal_access_tokens
 - ID, Descripiton 中输入你想要的 ID 和描述即可。
 - Credentials 选择配置好的 Gitee APIV5 Token
 - 点击 Advanced ，可配置是否忽略 SSL 错误（适您的Jenkins环境是否支持），并可设置链接测超时时间（适您的网络环境而定）
 - 点击 Test Connection 测试链接是否成功，如失败请检查以上 3，5，6 步骤。

*注：如果 Token 添加失败，可在 `系统管理 -> 凭据配置 -> Manage Credentials -> Stores scoped to Jenkins -> 全局凭据 (unrestricted) -> 添加凭据` 菜单添加全局凭据。*

![添加 Gitee Token](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/%E6%B7%BB%E5%8A%A0%20Gitee%20Token.png
)

![Gitee 插件配置](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/enter%20description%20here.png
)

### 三、Jenkins 使用

前期配置安装工作完成，就可以新建任务使用 Jenkins 创建 Gitee CI 任务。

#### 新建流水线任务

![新建流水线任务](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/%E6%96%B0%E5%BB%BA%E6%B5%81%E6%B0%B4%E7%BA%BF%E4%BB%BB%E5%8A%A1.png)

#### 任务配置

1. General 配置

Gitee 链接选择前面添加的名称

![General 配置](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/General%20%E9%85%8D%E7%BD%AE.png)

2. 构建触发器配置

    - 勾选 `Gitee webhook 触发构建`，并复制旁边提示的 URL `https://jenkins.rt-thread.org/gitee-project/rt-thread`，用于配置码云 Webhook
    - 构建策略选择 `新建 Pull Requests`、`更新 Pull Requests(Source Branch updated)`
    - 评论内容的正则表达式填写 `/retry-build`，用与评论触发构建
    - 取消勾选 `PR 不要求必须测试时过滤构建`
    - 生成 Gitee WebHook 密码，并复制（a1be0c9ca9ce6041b9ee267a03768cc9）

![构建触发器配置](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/%E6%9E%84%E5%BB%BA%E8%A7%A6%E5%8F%91%E5%99%A8%E9%85%8D%E7%BD%AE.png)

3. 流水线配置

    - `定义`处下拉选择 `Pipeline script from SCM`，`SCM` 处下拉选择 `Git`
    - 输入你的仓库地址，例如 git@your.gitee.server:gitee_group/gitee_project.git
    - 点击 Advanced 按钮, Name 字段中输入 origin， Refspec 字段输入 `+refs/heads/*:refs/remotes/origin/* +refs/pull/*/MERGE:refs/pull/*/MERGE`
    - Branch Specifier 选项输入 `pull/${giteePullRequestIid}/MERGE`

4. 保存设置

![流水线配置](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/%E6%B5%81%E6%B0%B4%E7%BA%BF%E9%85%8D%E7%BD%AE.png)

#### 码云仓库 WebHook 配置

码云端通过 WebHook 通知 Jenkins，触发流水线构建事件。因此需要在 Gitee 项目中增加一个 WebHook。

打开待配置的码云项目页面，进入 管理 -> WebHooks

1. 添加 WebHook， URL 填写 触发器配置处所复制 URL
2. 密码填写：触发器配置中点击生成的 WebHook密码并填入
3. 勾选  Push、Pull Request、评论事件

![配置 WebHook](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/%E9%85%8D%E7%BD%AE%20WebHook.png)

#### 流水线测试

**WebHook 请求测试**

在管理 -> WebHooks 中，点击测试按钮，此时 Gitee 会发起一次 WebHook 请求，码云端收到 200 返回表示测试成功。

![测试 WebHook 请求](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/%E6%B5%8B%E8%AF%95%20WebHook%20%E8%AF%B7%E6%B1%82.png)

#### Jenkinsfile 编译脚本测试

新建 `Jenkinsfile` 文件

```java
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'echo "Hello World"'
                sh '''
                    echo "Multiline shell steps works too"
                    ls -lah
                '''
            }
        }
    }
}
```

在 git 中 add、commit 并 push 到码云仓库，发起 PR 触发编译。完成后查看 `console log`，可看到 `Jenkinsfile` 中的脚本命令被执行，测试成功。

![提交 PR 触发构建](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/%E6%8F%90%E4%BA%A4%20PR%20%E8%A7%A6%E5%8F%91%E6%9E%84%E5%BB%BA.png)

![Jenkins脚本执行](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/Jenkins%E8%84%9A%E6%9C%AC%E6%89%A7%E8%A1%8C.png)

##### 进阶设置

**使用 Docker 运行流水线**

使用 `ubuntu_ci:latest` 指定 Docker 镜像（前提：镜像存在。并配置好 dood 环境），创建临时容器执行构建任务，完成后自动删除容器。

```java
pipeline {
    agent {
        docker { 
            image 'ubuntu_ci:latest' 
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'echo "Hello World"'
                sh '''
                    echo "Multiline shell steps works too"
                    ls -lah
                '''
            }
        }
    }
}
```

**构建结果返回**

在 `post` 阶段调用 Gitee 评论接口反馈构建结果，成功或者失败，并给出 job 详情页面链接。

*说明：使用 `env.RUN_DISPLAY_URL` 或 `env.BUILD_URL` 变量，均可获取构建详细日志页面， `env.RUN_DISPLAY_URL` 为 Blue Ocean 方式显示（需要安装 blue ocean 插件），`env.BUILD_URL` 为原始 log 页面。*

由于要对外展示构建详细日志页面，需在`系统管理 -> 全局安全配置 -> 授权策略 ->  匿名用户具有可读权限 ->` 中允许匿名用户访问。

```java
pipeline {
    agent {
        docker { 
            image 'ubuntu_ci:latest' 
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'echo "Hello World"'
                sh '''
                    echo "Multiline shell steps works too"
                    ls -lah
                '''
            }
        }
    }    
  post {
        failure {
            addGiteeMRComment(comment: """:x: Jenkins CI 构建失败。\n\n \
查看更多日志详细信息： \
<a href="${env.RUN_DISPLAY_URL}">Jenkins[${env.JOB_NAME} # ${env.BUILD_NUMBER}]</a> \
<hr /> \
:x: The Jenkins CI build failed.\n\n \
Results available at: \
<a href="${env.RUN_DISPLAY_URL}">Jenkins[${env.JOB_NAME} # ${env.BUILD_NUMBER}]</a>""")
        }
        success {
            addGiteeMRComment(comment: """:white_check_mark: Jenkins CI 构建通过。\n\n \
查看更多日志详细信息： \
<a href="${env.RUN_DISPLAY_URL}">Jenkins[${env.JOB_NAME} # ${env.BUILD_NUMBER}]</a> \
<hr /> \
:white_check_mark: The Jenkins CI build passed.\n\n \
Results available at: \
<a href="${env.RUN_DISPLAY_URL}">Jenkins[${env.JOB_NAME} # ${env.BUILD_NUMBER}]</a>""")
        }
    }
}
```


> 参考资料
> 
> * [Installing Jenkins](https://www.jenkins.io/doc/book/installing/)
> * [The simple way to run Docker-in-Docker for CI | Releaseworks Academy](https://tutorials.releaseworksacademy.com/learn/the-simple-way-to-run-docker-in-docker-for-ci)
> * [Jenkins 插件 - 码云 Gitee.com](https://gitee.com/help/articles/4193#article-header6)
> *[Jenkins换源，加速插件下载，解决下载慢，下载失败的问题_JikeStardy的博客-CSDN博客_jenkinds国内源更新插件慢](https://blog.csdn.net/JikeStardy/article/details/105606150)
> * [Running multiple steps](https://www.jenkins.io/doc/pipeline/tour/running-multiple-steps/)
> * 书籍《Jenkins 2.x 实践指南》