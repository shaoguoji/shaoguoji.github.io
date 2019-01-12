---
layout:          post
title:           【踩坑记录】 cp2102 「设备的最大波特率为 1280000」 错误
subtitle:        Realtek 烧录工具 Image_Tool 打开串口失败问题记录
date:            2019-01-10 15:39:39 +0800
author:          Shao Guoji
header-img:      
catalog:         true
tag:
    - 
---

### 问题描述

在使用搭载 cp2102 芯片的板子上，使用 Realtek Image_Tool 工具烧写时，打开串口会提示 `Error: Could not open COMXX! Original error: 设备的最大波特率为 1280000`，网上没有找到相关资料，在同事电脑一切正常。

![图1 错误信息](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/%E9%94%99%E8%AF%AF%E5%BC%B9%E7%AA%97.png)

#### 问题分析

自己的 cp2102 驱动是从官网下的最新版，对比后发现版本比我低，让他也更新到我的版本后，错误复现，初步认定是驱动版本导致。

### 解决方法

从同事电脑导出低版本驱动，覆盖安装，问题解决。

![图2 问题驱动及可用驱动版本](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/%E9%97%AE%E9%A2%98%E9%A9%B1%E5%8A%A8-%E5%8F%AF%E7%94%A8%E9%A9%B1%E5%8A%A8.png)


### 环境说明

* 操作系统：Windows10 1803(17134.523) 家庭中文版
* Image_Tool 版本：v2.2.0
* 问题驱动版本：10.1.4.2290
* 可用驱动版本：10.1.3.2130

最后附上可用驱动版本下载链接：[silabser.inf_amd64_f832c69b0a078fa9_10.1.3.2130.zip 提取码: dcbj](https://pan.baidu.com/s/1Flyv069FeHA_6vxxE5okXg)
