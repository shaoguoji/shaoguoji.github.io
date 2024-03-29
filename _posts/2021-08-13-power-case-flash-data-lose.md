---
layout:          post
title:           电源不稳导致 Flash 数据丢失
subtitle:        STM32 问题调试记录
date:            2021-08-13 17:03:29 +0800
author:          Shao Guoji
header-img:      
catalog:         true
tag:
    - STM32
    - 嵌入式
    - 笔记
---

随着接触实际产品的时间变多，一些莫名其妙的问题突发，常常让人摸不着头脑。故障往往与硬件、软件、操作系统机制等有关，不同的原因相互交织，使得排查变得困难。

这次来看一个电源不稳导致 Flash 数据丢失的案例。

### 情况介绍

产品使用 STM32 作为电机控制板 MCU，出现 OTA 断电后变砖的现象，经排查是由于配置区 Flash 数据丢失（被意外擦除），导致 BootLoader 无法正确校验 APP，从而异常跳转。

问题出现的时机不确定，好的时候啥事没有，一旦触发就连着来。仔细查看 BootLoader 代码并没有发现有 Flash 越界擦除等误操作的可能，最后将所有应用逻辑注释，留下初始化部分。

Flash 仍然被擦掉，而且前一秒烧写的数据，上电断电就没了，并且每次都出现在断电的时候。

加打印，确认在掉电时会反复复位，而在程序初始化部分有操作配置区 Flash 数据的代码，通过「读改写」的方式更新一些信息，问题就在于，读出来的数据不对，都是 `0xFF`，写入时便会破坏原始数据。

其实不难理解，电机控制板的电源设计较为复杂，还散布着不少大电容，掉电时候电压下降缓慢，导致 MCU 复位完全有可能（网上已经有不少上电/掉电缓慢，导致单片机复位异常的例子了）。或许就在断电的某一个时刻，程序能运行，但 Flash 等外设工作正常，操作数据就会导致不可预料的结果。

简单粗暴的解决方法可以在启动时加延时，死等电源稳定掉电后，再继续执行程序。或者主动出击，启动时去检测电压，确保电压正常再继续运行。

事实证明，问题确实得到解决。

### 精确定位

对故障的感性认识有了，还是想知道具体发生了什么。

![stm32 复位和电源控制模块特性](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/stm32-power.png)

查看芯片数据手册，认识到 `POR` 和 `PDR` 的概念，即电源上升和下降到某个阈值，发生复位的条件。可以看到，在复位的参数表格中给出，POR/PDR 阈值范围在 1.8~2V，此时芯片复位。

上示波器看看断电电源波形：

![断电波形1](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/board-powerdown-wave1.jpg)

![断电波形2](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/board-powerdown-wave2.jpg)

好家伙，电源刚好在 PDR 阈值范围内出现剧烈抖动，导致芯片多次复位，整体波形呈现「在动荡中下降」，像极了人生。

![断电波形3](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/board-powerdown-wave3.jpg)

并且两次抖动的上升沿间隔都大于 T(RSTTEMPO) 复位持续时间，条件满足，芯片完全够时间复位运行，并执行擦除 Flash 代码。

复位机制示意图如下：

![上电复位和掉电复位的波形图](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/reset-condition.png)

再翻看 Flash 的工作电压，最小在 2V 左右，假如电压在这之下芯片复位，操作数据极容易发生问题。

此外，为了应付意外断电的情况，STM32 还提供了 PVD（可编程电压监测器），能够检测电源电压，若低于设定的检测值时触发中断，在完全停机前进行善后处理。只不过对于复位，PVD 似乎只能「袖手旁观」，无法改变复位电压阈值。

### 经验总结

了解了上面所有的原理后，测量其它板子在电源复位阈值的波形，好在都没有问题（图形笔直无抖动），这才松了一口气。

我很庆幸自己遇到的问题，以往在开发板上，这是没有机会见识的，如果以后有能力自己设计硬件，在验证时肯定会特别留意一点：测量为 MCU 供电的电源在复位阈值范围的波形。
