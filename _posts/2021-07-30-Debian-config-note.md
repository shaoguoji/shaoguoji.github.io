---
layout:          post
title:           日用 Debian 配置笔记
subtitle:        Linux 作为日常使用
date:            2021-07-30 10:09:29 +0800
author:          Shao Guoji
header-img:      
catalog:         true
tag:
    - Debain
    - Linux
    - 笔记
---

在所有 Linux 发行版中，对 Debian（标准发音：“跌彼岸”而非“低板”）+lxde情有独钟，每次安装后配置命令都记不住，好记性不如烂笔头，还是做一下笔记。

### 安装

推荐使用 live-cd 版本镜像，包含了可在移动存储器运行的 Debian，安装还能在无线网卡驱动以及软件包上省不少心。

### 硬盘及分区

习惯将硬盘分出 3 个区，分别挂载根目录、home 目录和作为交换分区。如有必要还需要 EFI 分区，格式为 FAT32，挂载点 /boot/efi。

查看硬盘分区信息

`df -h`

`lsblk`

`sudo fdisk -l`

#### 迁移新硬盘

把新硬盘分区后使用 `dd` 命令对拷原分区数据，更新硬盘信息及引导后，直接拆机替换。

```bash

# dd if=/dev/sda1 of=/dev/sdb1

# umount /dev/sdb1　　　　　　
　　　
# e2fsck -f /dev/sdb1

# resize2fs /dev/sdb1

# update-grub /dev/sdb

```

### GRUB 配置

配置文件 `/etc/default/grub` 和 `/boot/grub/grub.cfg`，修改配置后执行 `update-grub` 命令更新。

### 自动登录桌面

#### autologin 设置

输入命令安装自动登录软件包

`sudo apt -y install lightdm-autologin-greeter`

修改配置文件，设置自动登录账号，配置文件路径： `/etc/lightdm/lightdm.conf.d/lightdm-autologin-greeter.conf`，初始内容如下：

```
[Seat:*]
# you really have to configure the below, otherwise
# LightDM auto-login will fail...
autologin-user=AUTOLOGIN-USER-NOT-CONFIGURED

# put any session from /usr/share/xsessions (strip .desktop from the file names there)
# here, if you want to run any other session than x-session-manager
#autologin-session=x-session-manager
```

用文本编辑器打开文件，修改“autologin-user”这行，将等号后面文字修改为你要自动登录系统的用户名，比如：shaoguoji，然后保存文件，退出登录，重启机器。

系统就会自动以你设置的用户，自动登录桌面环境。

#### 取消自动登录系统

`sudo apt -y purge lightdm-autologin-greeter`

退出登录，重启机器。

> 来源：[铜豌豆 Linux -- 使用技巧 -- 普通用户](https://www.atzlinux.com/skills.htm#bluetooth)

### 屏幕旋转

*以顺时针旋转 90 度为例（横竖屏转换）。*

#### 桌面环境显示旋转

`xrandr -o left/right`

#### tty 终端旋转

修改 `/etc/default/grub` 配置：

```
GRUB_CMDLINE_LINUX="fbcon=rotate:1"
```

#### 触摸屏坐标旋转

```
sudo apt install xserver-xorg-input-evdev

sudo vim /usr/share/X11/xorg.conf.d/40-libinput.conf
```

新的 input 子系统采用 `libinput` 模块，将 `Driver` 它改回 `evdev`，增加触摸屏相关属性，完成输入坐标变换：

```
Section "InputClass"
        Identifier "libinput touchscreen catchall"
        MatchIsTouchscreen "on"
        MatchDevicePath "/dev/input/event*"
        Driver "evdev"
    Option "SwapAxes" "true"
    Option "InvertY" "true"
EndSection
```

### 换源 USTC

编辑 `/etc/apt/sources.list` 文件（需要使用 sudo）。以下是 Debian Stable 参考配置内容：

```
deb http://mirrors.ustc.edu.cn/debian stable main contrib non-free
# deb-src http://mirrors.ustc.edu.cn/debian stable main contrib non-free
deb http://mirrors.ustc.edu.cn/debian stable-updates main contrib non-free
# deb-src http://mirrors.ustc.edu.cn/debian stable-updates main contrib non-free

# deb http://mirrors.ustc.edu.cn/debian stable-proposed-updates main contrib non-free
# deb-src http://mirrors.ustc.edu.cn/debian stable-proposed-updates main contrib non-free
```

更改完 sources.list 文件后请运行 sudo apt-get update 更新索引以生效。

> 来源：[Debian 源使用帮助 — USTC Mirror Help 文档](https://mirrors.ustc.edu.cn/help/debian.html)

### 查看系统信息

系统版本

`uname -a`

`lsb_release -a`

`cat /etc/issue`

`screenfetch`（需 apt 安装）

cpu 架构：`uname -m`

cpu 型号：`cat /proc/cpuinfo`

内存

`cat /proc/meminfo`

`free -h`

查看所有服务

`sudo systemctl list-units --type=service`

内核日志

`sudo dmesg`

### 终端设置

`sudo dpkg-reconfigure console-setup`

更新配置：`sudo setupcon`

### 按键值查看

`dumpkeys`

`showkey`

### 启动时运行命令

#### crontab 计划任务

`sudo crontab -e`

编译文件，增加启动执行项：

```
# Edit this file to introduce tasks to be run by cron. 
#  
# Each task to run has to be defined through a single line 
# indicating with different fields when the task will be run 
# and what command to run for the task 
#  
# To define the time you can provide concrete values for 
# minute (m), hour (h), day of month (dom), month (mon), 
# and day of week (dow) or use '*' in these fields (for 'any'). 
#  
# Notice that tasks will be started based on the cron's system 
# daemon's notion of time and timezones. 
#  
# Output of the crontab jobs (including errors) is sent through 
# email to the user the crontab file belongs to (unless redirected). 
#  
# For example, you can run a backup of all your user accounts 
# at 5 a.m every week with: 
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/ 
#  
# For more information see the manual pages of crontab(5) and cron(8) 
#  
# m h  dom mon dow   command 
 
@reboot date -R >> /home/shaoguoji/bootTime.txt 
@reboot sleep 60 && /home/shaoguoji/bin/frp/frpc -c /home/shaoguoji/bin/frp/frpc.ini &
```

####桌面环境自启动

*以自动旋转屏幕为例。*

新建 `rotate.desktop` 文件，其中的 `Exec` 字段为要执行的命令或脚本：

```
[Desktop Entry]
Type=Application
Name=xrandr_rotate
Exec=xrandr -o right
NoDisplay=true
X-GNOME-AutoRestart=true
```

保存文件，放置于目录 `~/.config/autostart/` 下，重启生效。


### 常用工具安装

#### 中文输入法（小鹤双拼）

系统默认使用 ibus 输入法，配置 fcitx 即可使用双拼。

安装 fcitx table，然后安装 im-config 输入法管理工具，重启。

```
sudo apt-get install fcitx-table-all

sudo apt-get install im-config
```

在桌面屏幕右下角应该会出现 fcitx 的图标，右键进入配置，增加 `shuangpin` 输入法（注意 `piyin` 只有全拼无双拼），修改设置为小鹤双拼。

#### 终端快捷命令 pet

```
# wget https://github.com/knqyf263/pet/releases/download/v0.3.6/pet_0.3.6_linux_386.deb
# sudo dpkg -i pet_0.3.6_linux_386.deb
```

使用： pet list/new/exec/search/edit

#### tldr - 命令帮助

`sudo apt install tldr -y`

#### Byobu - 终端扩展管理

安装

`sudo apt-get install byobu` 

登录启动

`byobu-enable`

`byobu-disable`

> 更多相关操作可以按 F9 选项查看帮助指南

#### frp - 内网穿透

服务端

`wget https://github.com/fatedier/frp/releases/download/v0.37.0/frp_0.37.0_linux_386.tar.gz`

`vim frps.ini`

`nohup frps -c frps.ini &`

客户端

`vim frpc.ini`

`nohup frpc -c frpc.ini &`

#### ssh 服务端

`sudo apt install openssh-server -y`

`sudo systemctl restart sshd`

#### fbterm - 中文 tty 终端

`sudo apt install fbterm -y`

#### ss

`sudo apt install -y shadowsocks-libev`

`sudo vim /etc/shadowsocks-libev/config.json`

#### 字符终端鼠标

`sudo apt install gpm -y`

`sudo /etc/init.d/gpm start/stop`

使用：鼠标左键拖选字符复制，中键粘贴。

#### 挂载 Windows 网络共享文件夹

```bash
# sudo apt install cifs-utils
# sudo mkdir -p /mnt/win_share
# sudo mount.cifs //192.168.2.59/shaoguoji /mnt/win_share/ -o user=Think
# sudo umount /mnt/win_share
```
