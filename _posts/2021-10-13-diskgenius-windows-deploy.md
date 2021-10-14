---
layout:          post
title:           使用 DiskGenius 快速部署 Windows 主机
subtitle:        一种基于分区备份还原的系统拷贝方法
date:            2021-10-13 16:44:29 +0800
author:          Shao Guoji
header-img:      
catalog:         true
tag:
    - 计算机
    - 笔记
---

本方法用于 Windows 设备产品量产时的快速部署，在一台主机上安装系统、驱动、软件运行环境后，通过 Diskgenius 软件的硬盘分区表与分区备份还原机制（相比于 Gohst 多了分区表备份功能），实现硬盘批量写入，免去多台机器上重复安装系统与软件的繁琐。

### 材料准备

1. U盘（建议 32G 以上）
2. Diskgenius 软件
3. 源主机（Windows 系统）与目标主机

### 步骤一：工具 U 盘制作

涉及硬盘底层分区修改及数据写入的操作，需要临时借助外部存储介质完成。

启动 U 盘解决方案使用 Ventoy，支持多 ISO 镜像引导启动，同时又能作为外部存储设备，满足大部分需求。

将空 U 盘插入电脑，参考 Ventoy 使用说明文档:[Get start . Ventoy](https://ventoy.net/cn/doc_start.html)，制作启动盘。

PE 系统使用微 PE 工具箱，下载地址：[微PE工具箱 - 下载](https://www.wepe.com.cn/download.html)。

运行 wepe，生成 ISO 引导镜像文件 `WePE_64_VXX.iso`，拷贝到 Ventoy U盘中：

![生成 WePE 系统镜像](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/wepe.png)

进入 PE 系统需要在 BISO 中设置 U 盘启动，通过 Ventoy 界面选择对应 ISO 文件引导启动。

### 步骤二：备份硬盘分区表

源主机安装部署好所有驱动环境后，便可以开始系统备份，得到备份文件后能在新的机器直接还原。

**备份过程可以在源主机系统或 PE 中进行，备份出来的文件统一存放在 U 盘 `DiskGenius_BK` 目录。**

#### 备份硬盘分区表

1. 选择要备份分区表的硬盘，然后点击菜单「磁盘 - 备份分区表」项：

![备份分区表菜单](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/partion-table-backup-menu.PNG)

2. 选择 U 盘 `DiskGenius_BK` 目录，保存分区表备份文件 `NetacSSD240GB(224GB).ptf`。

![保存分区表备份](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/partion-table-backup.PNG)

参考文档：[备份与还原分区表 - DiskGenius](https://www.diskgenius.cn/help/ptbackup.php)

### 步骤三：备份 ESP 分区

参考文档：[备份分区 - DiskGenius](https://www.diskgenius.cn/help/part2file.php)

*说明：在 GPT 格式硬盘中，ESP 分区与引导相关，因此也需要备份，否则系统还原后无法启动。*

1. 选中 ESP 分区（此时红框选中）右击，在弹出的菜单中选择「备份分区到镜像文件」项：

![选中 ESP 分区](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/esp-partion.PNG)

![备份 ESP 分区到镜像文件](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/esp-partion-backup.png)

2. 选择 U 盘 `DiskGenius_BK` 目录，保存 ESP 分区备份文件 `ESP.pmf`，最后点击「开始」按钮，开始对分区进行备份。

![ESP 分区备份选项](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/esp-partion-backup-option.PNG)

### 步骤四：备份系统（C 盘）分区

参考文档：[备份分区 - DiskGenius](https://www.diskgenius.cn/help/part2file.php)

系统分区备份操作与 ESP 分区备份基本一致。

1. 选中系统分区（本地磁盘C）右击，在弹出的菜单中选择「备份分区到镜像文件」项：

![备份系统分区到镜像文件](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/system-partion-backup.png)

2. 选择 U 盘 `DiskGenius_BK` 目录，保存系统分区备份文件 `System.pmf`，最后点击「开始」按钮，开始对分区进行备份。

![系统分区备份选项](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/system-partion-backup-opeion.PNG)

*系统分区备份分区时间较长，期间禁止一切电脑操作，耐心等待备份完成即可。*

### 步骤五：还原备份

对于新组装的新机器，硬盘未分区，没有操作系统，还原操作在 U 盘 PE 系统中进行。

插入 U 盘引导启动 PE，同样打开 Diskgenius 工具：

![PE 运行 Diskgenius 工具](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/PE-DiskGenius.bmp)

#### 还原分区表

参考文档：[备份与还原分区表 - DiskGenius](https://www.diskgenius.cn/help/ptbackup.php)

选择硬盘，然后点击菜单「磁盘 - 还原分区表」菜单项：

![还原分区表选项](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/partion-table-recover-menu.png)

在弹出的对话框中选择 U 盘 `DiskGenius_BK` 目录下的分区表备份文件 `NetacSSD240GB(224GB).ptf` 打开：

![打开分区表备份文件](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/partion-table-recover-select.png)

弹框还原分区表和引导扇区确认，选择「是」：

![还原分区表确认](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/esp-partion-confirm-1.png)

![还原引导扇区确认](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/esp-partion-confirm-2.png)

操作完成后，可以看到硬盘分区信息已经恢复：

![还原后的分区信息](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/partion-info.png)

然而分区内数据还是空的，接下来进行还原分区操作。

#### 还原 ESP 分区

参考文档：[从镜像文件还原分区 - DiskGenius](https://www.diskgenius.cn/help/restorepart.php)

1. 选中 ESP 分区，然后点击菜单「工具 - 从镜像文件还原分区」菜单项：

![从镜像文件还原 ESP 分区](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/esp-partion-recover.png)

在弹出的对话框中选择 U 盘 `DiskGenius_BK` 目录下的 ESP 分区备份文件 `ESP.pmf` 打开，同时选择需要还原的时间点，点击「开始」按钮还原分区：

![ESP 分区还原选项](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/esp-partion-recover-option.png)

#### 还原系统分区

1. 选中系统分区，然后点击菜单「工具 - 从镜像文件还原分区」菜单项：

![从镜像文件还原系统分区](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/system-partion-recover.png)

在弹出的对话框中选择 U 盘 `DiskGenius_BK` 目录下的 ESP 分区备份文件 `System.pmf` 打开，同时选择需要还原的时间点，点击「开始」按钮还原分区：

![开始还原系统分区](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/system-partion-recover-option.png)

#### 重启电脑进入系统

分区表和分区镜像成功还原，重启电脑启动系统，便可得到和原来一样的系统环境。






