---
layout:           post
title:             ESP8266(Non-OS SDK) 驱动 waveshare 2.9 寸墨水屏（一）
subtitle:      源码分析及准备工作
date:               2018-03-25 15:52:09 +0800
author:           Shao Guoji
header-img:  img/post-bg-esp8266-e-paper.jpg
catalog:         true
tag:
    - 嵌入式
    - 单片机
    - ESP8266
---

![图1 屏幕刷新效果](http://odaps2f9v.bkt.clouddn.com/18-4-21/18249554.jpg)

随着屏幕的阵阵闪烁刷新、黑白字符图案浮现眼前，毕业设计总算有了起色。经历了几个下午的不懈努力总算把墨水屏驱动搞定，点亮的何止是小小的墨水屏，还有我骚动的心呐！一开始还想着从头啃芯片手册造轮子，最后由于时间紧迫 + 能力有限，于是想(tou)到(lan)把微雪提供的 STM32 例程移植到 8266。

由于都是纯 C 的代码，整个过程说不上十分艰辛，但也踩了不少坑：从最初对 8266 硬件 SPI 的不明觉厉，到迫于无奈用 GPIO 软件模拟、最后移植适配官方 STM32 例程，与网上的 Arduino、Lua 实现不同，我是在乐鑫 Non-OS SDK 环境下开发，步骤稍多，特此记录。

---

### 开发环境

1. SDK：[ESP8266_NONOS_SDK-2.1.0](https://www.espressif.com/zh-hans/support/download/sdks-demos)
Toolchain：乐鑫标配 [ VirtualBox](https://www.virtualbox.org/wiki/Downloads) + [lubuntu](http://downloads.espressif.com/FB/ESP8266_GCC.zip) 编译环境
2. Editor：VS Code / Nodepad++ / 随便你啦
3. board: NodeMcu

*友情提示：没有接触过 8266 SDK 开发？没事，在正式开始之前，请按照文档[《ESP8266 SDK 入门指南》](https://www.espressif.com/sites/default/files/documentation/2a-esp8266-sdk_getting_started_guide_cn.pdf)把编译、下载的流程玩一遍即可。*

#### 屏幕相关资料

* 产品详情：[2.9inch e-Paper Module - Waveshare Wiki](http://www.waveshare.net/wiki/2.9inch_e-Paper_Module)
* 用户手册：[2.9inch-e-paper-user-manual-cn.pdf](http://www.waveshare.net/w/upload/c/cd/2.9inch-e-paper-user-manual-cn.pdf)
* 数据手册：[2.9inch e-Paper Datasheet.pdf](http://www.waveshare.net/w/upload/e/e6/2.9inch_e-Paper_Datasheet.pdf)
* 例程源码：[2.9inch_e-Paper_Module_code.7z](http://www.waveshare.net/w/upload/7/78/2.9inch_e-Paper_Module_code.7z)

---

### waveshare 2.9 寸黑白墨水屏（模块）介绍

![图2 墨水屏产品图片](http://www.waveshare.net/w/upload/thumb/0/0f/2.9inch-e-Paper-Module-1.jpg/360px-2.9inch-e-Paper-Module-1.jpg)

#### 产品参数

| 电压         | 3.3 V                                       |
| 通信接口 | 3-wire SPI、 4-wire SPI |
| 分辨率     | 296 × 128                               |
| 显示颜色 | 黑、白                                     |
| 灰度等级 | 2                                               |
| 刷新时间 | 2-3 s                                       |

#### 管脚定义

| VCC    | 3.3 V                                                                                                  |
| GND    | GND                                                                                                     |
| DIN    | SPI 通信 MOSI 引脚                                                                        |
| CLK    | SPI 通信 SCK 引脚                                                                          |
| CS      | SPI 片选引脚（低电平有效）                                                      |
| DC      | 数据/命令控制引脚（高电平表示数据，低电平表示命令） |
| RSY    | 外部复位引脚（低电平复位）                                                     |
| BUSY  | 忙状态输出引脚（高电平表示忙）                                             |

#### 数据显示原理

屏幕通过 SPI 总线与 MCU 连接作为从机使用，通过接受命令/数据完成相应功能。和其他模块相似，墨水屏 SPI 总线上传输的比特位也分为数据和命令两种类型，并根据 DC 引脚电平区分（高电平数据低电平命令）。而其中最重要的数据便是显存了，它决定着屏幕每个像素点的状态，控制显示内容。

2.9 寸墨水屏分辨率为 296 × 128，即宽度有 296 个像素点、高度有 128 个像素点。对应到显存数据就是横向每 8 个点用一个字节表示（1 byte = 8 pixels），一行就包含 296/8 = 37 个字节，共有 128 行。显示方式是从左到右从上到下，如下图所示：

![图3 数据显示原理图](http://odaps2f9v.bkt.clouddn.com/18-4-21/59920391.jpg)

知道数据是如何显示在屏幕上，对我们编写相关显示函数十分有用。当然，在例程中已经实现了显示字符、矩形、圆形等常用函数，直接调用相应函数，就可以设置显存数据为显示内容（各种画图显示函数只不过在折腾这一堆字节数据而已）。

#### 局部刷新原理

局部刷新可以避免显示时全屏闪烁，加快刷新速度并提高用户观感。如此神奇的的功能是怎么做到的呢？其实通过手册的描述不难猜测实现原理。

黑白墨水屏的显示流程可粗略分为 4 步：

1. 准备显存数据（调用各种显示函数折腾一堆字节数据）
2. 传输显存数据（只是将数据传到屏幕内部存储空间中，此时屏幕无动静）
3. 发送刷新命令（刷新屏幕、显示相应内容）
4. 等待刷新完成（MCU 检测 BUSY 引脚电平）

而实现局部刷新的玄机就藏在第 2 和 第 3 步之间 —— 屏幕内部其实有两份显存空间，这两块空间轮流接收 MCU 传来的显存数据。当收到刷新命令时，内部控制器并不是直接显示所有显存数据，而是先把要显示的数据与另一块空间（上一次的旧数据）作对比，只刷新发生变化的数据，完成显示（这一切都是由硬件自动完成，程序员照常发送完整一帧数据）。

> PS: 这其实和「差分 OTA」或「增量更新」是同一思想，都存在「比较数据差异」、「更新变化数据」的过程。

#### 注意事项

1. 供电电压和引脚信号电平需保持一致，否则显示异常（**【坑】Arduino 引脚电平固定 5 V 输出，需临时用 5 V 供电测试**）。
2. 显示数据宽度必须为 8 的倍数，原因见上面「数据显示原理」。
3. 只支持黑、白两级灰度，非黑即白（显示浓淡墨色的中国画就别想了，毕竟不是 Kindle）。
4. **【坑】三色屏并不是白送一种颜色，还多「送」了十几秒的刷新时间，并且不支持局部刷新！选型时注意。**

---

### 官方 STM32 例程分析

解压例程包 [2.9inch_e-Paper_Module_code.7z](http://www.waveshare.net/w/upload/7/78/2.9inch_e-Paper_Module_code.7z)，其中包含了 Arduino、树莓派以及 STM32 三个版本的例程。打开 `stm32` 文件夹，其中 `BSP` 目录存放了墨水屏驱动相关源文件和头文件，`Fonts` 目录则是字模数组相关文件，其余部分都是通用 STM32 程序的必要文件。

从最顶层的 `Src/mian.c` 着手分析程序，一步步深挖源码。 `main()` 函数虽然有点长，但分段来看并不复杂。总的来说，示例程序做了这么几件事情：

#### 1. 芯片外设、墨水屏初始化

从程序起始处一些带有 `Init` 字样的函数名可以看出，程序第一件事就是初始化各种外设，这也是一个系统的「例行公事」。其中包括对系统时钟、GPIO、SPI、串口等芯片外设的初始化（由 STM32 HAL 库函数完成）和墨水屏硬件及显示参数的初始化（通过向屏幕发送指定命令/数据完成）。

```c
  /* USER CODE END 1 */

  /* MCU Configuration----------------------------------------------------------*/

  /* Reset of all peripherals, Initializes the Flash interface and the Systick. */
  HAL_Init();

  /* USER CODE BEGIN Init */

  /* USER CODE END Init */

  /* Configure the system clock */
  SystemClock_Config();

  /* USER CODE BEGIN SysInit */

  /* USER CODE END SysInit */

  /* Initialize all configured peripherals */
  MX_GPIO_Init();
  MX_SPI1_Init();
  MX_USART2_UART_Init();

  /* USER CODE BEGIN 2 */
  EPD epd;
  if (EPD_Init(&epd, lut_full_update) != 0) {
    printf("e-Paper init failed\n");
    return -1;
  }
  
  Paint paint;
  Paint_Init(&paint, frame_buffer, epd.width, epd.height);
  Paint_Clear(&paint, UNCOLORED);
```

`EPD_Init()` 函数会发送一长串命令/数据序列对模块硬件进行初始设置，接着调用 `Paint_Init()` 函数设置画板的缓冲数组（显存）和宽高。整个初始化工作直到执行完 `Paint_Clear(&paint, UNCOLORED);` 语句 —— 清空显存数据为止。

#### 2. 显示字符、矩形和圆形

```c
/* For simplicity, the arguments are explicit numerical coordinates */
  /* Write strings to the buffer */
  Paint_DrawFilledRectangle(&paint, 0, 10, 128, 34, COLORED);
  Paint_DrawStringAt(&paint, 0, 14, "Hello world!", &Font16, UNCOLORED);
  Paint_DrawStringAt(&paint, 0, 34, "e-Paper Demo", &Font16, COLORED);

  /* Draw something to the frame buffer */
  Paint_DrawRectangle(&paint, 16, 60, 56, 110, COLORED);
  Paint_DrawLine(&paint, 16, 60, 56, 110, COLORED);
  Paint_DrawLine(&paint, 56, 60, 16, 110, COLORED);
  Paint_DrawCircle(&paint, 120, 90, 30, COLORED);
  Paint_DrawFilledRectangle(&paint, 16, 130, 56, 180, COLORED);
  Paint_DrawFilledCircle(&paint, 120, 160, 30, COLORED);

  /* Display the frame_buffer */
  EPD_SetFrameMemory(&epd, frame_buffer, 0, 0, Paint_GetWidth(&paint), Paint_GetHeight(&paint));
  EPD_DisplayFrame(&epd);
  EPD_DelayMs(&epd, 2000);
```

初始化工作完成后，就可以开始折腾显存数据、显示内容了。和一开始在「局部刷新原理」部分提到的流程一样：首先调用一波类似 `Paint_DrawXXX()` 的函数，按照需要处理显存缓冲数组 `frame_buffer` 的内容（本质就是位操作）。然后调用 `EPD_SetFrameMemory()` 和 `EPD_DisplayFrame()` 发送数据、刷新屏幕、延时等待刷新完成。

#### 3. 清显存、显示图片 LOGO

```c
/**
   *  there are 2 memory areas embedded in the e-paper display
   *  and once the display is refreshed, the memory area will be auto-toggled,
   *  i.e. the next action of SetFrameMemory will set the other memory area
   *  therefore you have to set the frame memory and refresh the display twice.
   */
  EPD_ClearFrameMemory(&epd, 0xFF);
  EPD_DisplayFrame(&epd);
  EPD_ClearFrameMemory(&epd, 0xFF);
  EPD_DisplayFrame(&epd);

  /* EPD_or partial update */
  if (EPD_Init(&epd, lut_partial_update) != 0) {
    printf("e-Paper init failed\n");
    return -1;
  }

  /**
   *  there are 2 memory areas embedded in the e-paper display
   *  and once the display is refreshed, the memory area will be auto-toggled,
   *  i.e. the next action of SetFrameMemory will set the other memory area
   *  therefore you have to set the frame memory and refresh the display twice.
   */
  EPD_SetFrameMemory(&epd, IMAGE_DATA, 0, 0, epd.width, epd.height);
  EPD_DisplayFrame(&epd);
  EPD_SetFrameMemory(&epd, IMAGE_DATA, 0, 0, epd.width, epd.height);
  EPD_DisplayFrame(&epd);
```

第二部分要显示微雪的 LOGO（对！就是广告上那个），原理和前面一样，只不过显存是预先准备好的图片取模数据。在显示前需要调用 `EPD_ClearFrameMemory()` 全局清屏刷新。注意这里是发送数据到屏幕清内部显存，而不像初始化时的 `Paint_Clear()` 只是对 MCU 内存中的缓冲数组赋值，因此要重复操作两块内部显存。

#### 4. 显示时间计数（局部刷新）

```c
time_start_ms = HAL_GetTick();
  /* USER CODE END 2 */

  /* Infinite loop */
  /* USER CODE BEGIN WHILE */
  while (1)
  {
  /* USER CODE END WHILE */

  /* USER CODE BEGIN 3 */
    time_now_s = (HAL_GetTick() - time_start_ms) / 1000;
    time_string[0] = time_now_s / 60 / 10 + '0';
    time_string[1] = time_now_s / 60 % 10 + '0';
    time_string[3] = time_now_s % 60 / 10 + '0';
    time_string[4] = time_now_s % 60 % 10 + '0';

    Paint_SetWidth(&paint, 32);
    Paint_SetHeight(&paint, 96);
    Paint_SetRotate(&paint, ROTATE_90);

    Paint_Clear(&paint, UNCOLORED);
    Paint_DrawStringAt(&paint, 0, 4, time_string, &Font24, COLORED);
    EPD_SetFrameMemory(&epd, frame_buffer, 80, 72, Paint_GetWidth(&paint), Paint_GetHeight(&paint));
    EPD_DisplayFrame(&epd);

    EPD_DelayMs(&epd, 500);
  }
  /* USER CODE END 3 */
```

例程的最后一部分演示了局部刷新功能，在 `while(1)` 循环中获取系统时间并计算显示。这次使用了 `Paint_SetWidth()`、`Paint_SetHeight()` 函数将显示区域划定为 32 × 96 大小，只清缓冲数组不清内部显存，实现局部刷新时钟走秒。

最后建议找一块 STM32F103 的开发板把例程跑一下，看看墨水屏实际显示效果。建议不断改动源码、对比运行效果差异，加深对程序的理解。

---

### 驱动程序移植

不同芯片的软件框架与硬件配置都存在差异，STM32 与 ESP8266 也是如此，只有了解两者各自的特点才能实现「为我所用」。

#### 乐鑫 ESP8266 IoT_Demo 项目结构

在 SDK 入门教程中提供的 IoT_Demo 代码演示了完整的 ESP8266 Non-OS SDK 工程的组成，可通过 gcc 和 make 工具构建项目，在 Linux 下运行 项目目录的 `gen_misc.sh` 脚本即可编译固件。需要特别注意 Makefile 文件的一些关键参数。

`IoT_Demo` 目录下存放项目总的 `Makefile` 文件，其下的 `user`、`driver` 子目录也有独立的 Makefile 文件。构建时，子目录下的文件将单独编译为 `.a` 文件，最后合并为 `.bin` 固件。

总 `Makefile` 文件中的 24 行通过 `SUBDIRS` 变量引入子目录:

```makefile
SUBDIRS=    \
    user    \
    driver
```

在 49 行的 `COMPONENTS_eagle.app.v6` 将编译后的子模块包含进来:

```makefile
COMPONENTS_eagle.app.v6 = \
    user/libuser.a  \
    driver/libdriver.a
```

其中 `libuser.a` 文件名在各子目录 `Makefile` 文件中的 `GEN_LIBS` 变量指定：

```makefile
ifndef PDIR
GEN_LIBS = libdriver.a
endif
```

`include` 目录下放置项目用到的所有头文件，使用 `INCLUDES := $(INCLUDES) -I $(PDIR)include` 语句在 Makefile 中引入。

*PS：include 下子目录头文件引入时需写全路径，如 #include "driver/key.h"。*

#### 微雪例程 BSP 代码结构

所谓程序移植，核心在于修改具体硬件相关代码。良好的代码结构有利于移植工作，在例程目录 `BSP` 中的文件包含了墨水屏操作函数，并按照不同层次分为 `epdif.c`、`epd2in9.c`  和 `epdpaint.c` 三个文件（机对应头文件）。

其中和硬件相关、最底层的是 `epdif.c`，包含 SPI 数据发送、GPIO 电平控制、毫秒延时等基本操作函数。在此基础上根据不同型号屏幕特性进行封装，形成 `epd2in9.c` （2.9 寸屏）文件，完成基本的初始化、数据命令交互功能。最上层的 `epdpaint.c` 与画图相关，只是对图像缓存数据进行处理，与硬件无关。

显然，移植工作的重点是修改 `epdif.c` 文件，重写底层硬件相关功能函数，这时需要了解两者硬件上的一些差异，像 SPI 接口、定时器等。

#### SPI 字节发送函数

MCU 与墨水屏通过 SPI 总线协议通讯，因此 SPI 数据发送函数便是所有功能的根本。微雪例程中， SPI 发送函数使用 STM32 HAL 库封装好的 `HAL_SPI_Transmit()` 完成数据发送，在  `epdif.c` 文件 75 行的 `EpdSpiTransferCallback()` 处调用：

```c
void EpdSpiTransferCallback(unsigned char data) {
  ...
  HAL_SPI_Transmit(&hspi1, &data, 1, 1000);
  ...
}
```

8266 带有两个硬件 SPI 接口并提供接口函数，但由于硬件 SPI 从寄存器层面就针对 Flash 存储器做了特定的支持，需要与之通讯的 SPI 设备做对应的匹配处理，也就是说 **ESP8266 提供的硬件 SPI 接口并不通用**。自己尝试折腾了一遍相关函数，连墨水屏程序仅需的字节发送功能都实现不了，于是果断放弃。最后决定使用软件 GPIO 模拟的方式实现 SPI 接口数据发送。

#### 延时函数与看门狗复位

`epdif.c` 文件还需要提供定时器毫秒延时函数接口，很不幸， 8266 的 SDK 只有 微秒延时函数，且最大值为  65535 us，这就需要循环调用微秒延时函数实现更长的延时，这倒不是多大的问题。相比之下，「喂狗」问题更值得注意 —— 和 stm32 的裸机程序不同，在 ESP8266 Non-OS SDK 上用户线程不能长时间占用 CPU，否则会导致看门狗复位，ESP8266 重启。

解决方法是勤「喂狗」，正如官方文档所说：

> 如果某些特殊情况下，⽤户线程必须执⾏较长时间（⽐如⼤于 500 ms），建议经常
调⽤ system_soft_wdt_feed() API 来喂软件看门狗，⽽不建议禁⽤软件看门狗。

在墨水屏驱动程序中，经常出现等待屏幕相应刷新的情况（墨水屏动作出了名的慢），如果不注意「喂狗」问题，芯片便会莫名其妙重启，你就一脸懵逼捉急。

#### C 标准与 GCC 编译选项



下一篇我们正式开始程序的移植工作。

> 参考资料
> 
> * [2.9inch e-Paper Module - Waveshare Wiki](http://www.waveshare.net/wiki/2.9inch_e-Paper_Module)
> * [2.9inch-e-paper-user-manual-cn.pdf](http://www.waveshare.net/w/upload/c/cd/2.9inch-e-paper-user-manual-cn.pdf)
> * [ESP8266 SDK 入门指南](https://www.espressif.com/sites/default/files/documentation/2a-esp8266-sdk_getting_started_guide_cn.pdf)
> * [ESP8266 Non-OS SDK API 参考](https://www.espressif.com/sites/default/files/documentation/2c-esp8266_non_os_sdk_api_reference_cn.pdf)
> * [ESP8266 技术参考](https://www.espressif.com/sites/default/files/documentation/esp8266-technical_reference_cn.pdf)