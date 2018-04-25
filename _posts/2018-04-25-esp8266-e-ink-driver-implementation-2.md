---
layout:           post
title:             ESP8266(Non-OS SDK) 驱动 waveshare 2.9 寸墨水屏（二）
subtitle:      程序移植、修改与测试
date:               2018-04-25 15:52:09 +0800
author:           Shao Guoji
header-img:  img/post-bg-esp8266-e-paper.jpg
catalog:         true
tag:
    - 嵌入式
    - 单片机
    - ESP8266
---

### 开始移植

[上一篇文章]({% post_url 2018-03-25-esp8266-e-ink-driver-implementation-1 %})简单介绍了了墨水屏原理、例程代码以及移植工作的可行性。这一步的目的是把前面在 STM32 跑的程序，完整地搬到 esp8266 上，达到相同的运行显示效果，Let's get started!

#### STEP1: 文件目录修改

在开始程序移植工作之前先准备要用到的目录和文件：

1、把  `IoT_Demo` 文件夹拷贝一份到 VirtualBox 虚拟机共享文件夹编译目录下，命名为 `e-paper`。

2、将 `stm32` 目录实例代码中的 `BSP` 和 `Fonts` 文件夹复制到 `e-paper` 下，并添加到总 `Makefile` 文件 `SUBDIRS` 变量中：

```makefile
SUBDIRS=    \
    user    \
    driver  \
    BSP     \
    Fonts
```

3、删除 `driver` 目录下所有.c 文件（保留 `Makefile`）。

4、删除 `include` 目录下除 `user_config.h` 外全部文件和目录。

5、删除 `user` 目录下除 `Makefile` 和 `user_main.c` 外的所有文件。

6、将原 `BSP` 和 `Fonts` 下的头文件剪切到 `include` 目录。

7、从 'driver' 中拷贝 `Makefile` 文件到  `BSP`、 `Fonts` 目录，分别修改其 `GEN_LIBS` （输出文件变量）为 `libbsp.a` 和 `libfonts.a`，

```makefile
# BSP/Makefile
ifndef PDIR
GEN_LIBS = libbsp.a
endif

# Fonts/Makefile
ifndef PDIR
GEN_LIBS = libfonts.a
endif
```

8、最后把两个目标文件添加到 `Makefile` 文件的 `COMPONENTS_eagle.app.v6`，并在 `CCFLAGS` 加上编译选项 `-std=c99`（上一篇提过，代码中用到 C99 语法），完成所有修改工作：

```makefile
CCFLAGS += -Os -std=c99

COMPONENTS_eagle.app.v6 = \
    user/libuser.a  \
    driver/libdriver.a  \
    BSP/libbsp.a  \
    Fonts/libfonts.a
```

#### 引脚定义

STM32 例程中 GPIO 引脚设置不适用于 ESP8266 程序，必须重新考虑管脚的布局。为了让 NodeMCU 与墨水屏的连接更美观、紧凑，同时提供驱动编写规范，对引脚连接定义如下表（可灵活修改，**【坑】但一些引脚上电时要求保持特定电平，故不能使用**）：

| e-paper | ESP8266EX  | NodeMCU   |
|--------|----------|---------|
| 3.3V        | VDDPST        | 3V3             |
| GND          | GND               | GND             |
| DIN          | GPIO14        | D5               |
| CLK          | GPIO12        | D6               |
| CS            | GPIO13        | D7               |
| DC            | GPIO15        | D8               |
| RST          | GPIO3          | RX               |
| BUSY        | GPIO5          | D1               |

创建 `include/main.h` 文件，将上面引脚信息以及 GPIO 对应寄存器、功能设置名称用宏定义保存，便于程序的维护。

```c
#ifndef __MAIN_H__ 
#define __MAIN_H__ 

#include "os_type.h" 
#include "gpio.h" 

#define EPAPER_DIN_PIN      14 
#define EPAPER_CLK_PIN      12 
#define EPAPER_CS_PIN       13 
#define EPAPER_DC_PIN       15 
#define EPAPER_RST_PIN      3 
#define EPAPER_BUSY_PIN     5 

#define EPAPER_DIN_REG      PERIPHS_IO_MUX_MTMS_U 
#define EPAPER_CLK_REG      PERIPHS_IO_MUX_MTDI_U 
#define EPAPER_CS_REG       PERIPHS_IO_MUX_MTCK_U 
#define EPAPER_DC_REG       PERIPHS_IO_MUX_MTDO_U 
#define EPAPER_RST_REG      PERIPHS_IO_MUX_U0RXD_U 
#define EPAPER_BUSY_REG     PERIPHS_IO_MUX_GPIO5_U 

#define EPAPER_DIN_FUN      FUNC_GPIO14 
#define EPAPER_CLK_FUN      FUNC_GPIO12 
#define EPAPER_CS_FUN       FUNC_GPIO13 
#define EPAPER_DC_FUN       FUNC_GPIO15 
#define EPAPER_RST_FUN      FUNC_GPIO3 
#define EPAPER_BUSY_FUN     FUNC_GPIO5 

#endif /* #ifndef __MAIN_H__ */
```

> 详细引脚信息见表格[ESP8266 管脚清单](https://www.espressif.com/zh-hans)。

---

### STEP2: 软件 SPI 接口代码编写

编写 `driver/spi.c` 文件完成 GPIO 初始化和 SPI 时序模拟（发送一个字节数据）代码如下：

```c
#include "spi.h"

// Test spi master interfaces.
void ICACHE_FLASH_ATTR SPI_Init(void)
{
    // PIN_FUNC_SELECT(SPI_MISO_PIN_REG, SPI_MISO_PIN_FUN); //configure io to gpio mode
    PIN_FUNC_SELECT(SPI_MOSI_PIN_REG, SPI_MOSI_PIN_FUN); //configure io to gpio mode
    PIN_FUNC_SELECT(SPI_CLK_PIN_REG, SPI_CLK_PIN_FUN);   //configure io to gpio mode
    PIN_FUNC_SELECT(SPI_CS_PIN_REG, SPI_CS_PIN_FUN);     //configure io to gpio mode
}

void ICACHE_FLASH_ATTR Spi_Write_Byte(uint8_t data)
{
    u8 i;
    
    for(i = 0; i < 8; i++)
    {
        /* 写功能，上升沿写入数据 */
        GPIO_OUTPUT_SET(GPIO_ID_PIN(SPI_CLK_PIN), 0);   //拉低时钟线
        if(data & 0x80)    //数据发送顺序为先高
        {
            GPIO_OUTPUT_SET(GPIO_ID_PIN(SPI_MOSI_PIN), 1);   //写1
        }
        else
        {
            GPIO_OUTPUT_SET(GPIO_ID_PIN(SPI_MOSI_PIN), 0);   //写0
        }
        GPIO_OUTPUT_SET(GPIO_ID_PIN(SPI_CLK_PIN), 1);    //拉高时钟线
        data <<= 1;
    }
    
    GPIO_OUTPUT_SET(GPIO_ID_PIN(SPI_CLK_PIN), 0);        //拉低时钟线
}

```

同时新建 `spi.h` 文件保存到 `driver` 目录。头文件包括 SPI 引脚宏定义以及函数声明。`Spi_Write_Byte()`  函数只实现主机到从机的单向数据传输，暂不使用 MISO 引脚。具体的 GPIO 寄存器、管脚、功能则引用 `main.h` 头文件内的宏名。

```c
#ifndef __SPI_H__
#define __SPI_H__

#include "os_type.h"

#include "main.h"

//*****************************************************************************
//
// Make sure all of the definitions in this header have a C binding.
//
//*****************************************************************************

#ifdef __cplusplus
extern "C"
{
#endif

#define     SPI_MOSI_PIN        EPAPER_DIN_PIN
#define     SPI_CLK_PIN          EPAPER_CLK_PIN
#define     SPI_CS_PIN            EPAPER_CS_PIN

#define     SPI_MOSI_PIN_REG    EPAPER_DIN_REG 
#define     SPI_CLK_PIN_REG      EPAPER_CLK_REG 
#define     SPI_CS_PIN_REG        EPAPER_CS_REG 

#define     SPI_MOSI_PIN_FUN    EPAPER_DIN_FUN 
#define     SPI_CLK_PIN_FUN      EPAPER_CLK_FUN 
#define     SPI_CS_PIN_FUN        EPAPER_CS_FUN 

void SPI_Init(void);
void Spi_Write_Byte(uint8_t data);

#ifdef __cplusplus
}
#endif

#endif //  __SPI_H__
```

---

### STEP3: `epdif.c` 底层接口实现

是时候开始表演真正的程序移植了 —— 更改程序引脚定义、根据目标硬件（ESP8266）实现所有 `epdif.c` 文件中的所有接口（ I/O 电平读写、毫秒延时、SPI 数据发送等）。

首先注释掉 `extern SPI_HandleTypeDef hspi1;` 这行没用的代码、 引入头文件：

```c
#include "epdif.h"
#include "main.h"
#include "spi.h"
#include "osapi.h"
```

#### 修改引脚初始化列表

微雪例程中使用 `EPD_Pin` 结构体（包含 `port` 和 `pin` 两个成员）管理引脚，在 `BSP/epdif.c` 文件内用结构体数组方式定义。ESP8266 接口没有 GPIO port 的概念，注释掉 `include/epdif.h` 中结构体类型定义的 `port` 成员。

```c
typedef struct {
  // GPIO_TypeDef* port;
  int pin;
} EPD_Pin;
```

同时注释掉 `epdif.c` 代码里 `epd_cs_pin`、`epd_dc_pin`、`epd_rst_pin`、`epd_busy_pin` 四个结构体初始化列表的 `XXX_GPIO_Port` 部分，并把 `pin` 成员更改为 `epaper.h` 对应宏名，完成修改：

```c
EPD_Pin epd_cs_pin = {
  // SPI_CS_GPIO_Port,
  EPAPER_CS_PIN,
};

EPD_Pin epd_dc_pin = {
  // RST_GPIO_Port,
  EPAPER_DC_PIN,
};

EPD_Pin epd_rst_pin = {
  // DC_GPIO_Port,
  EPAPER_RST_PIN,
};

EPD_Pin epd_busy_pin = {
  // BUSY_GPIO_Port,
  EPAPER_BUSY_PIN,
};
```

#### GPIO 读写函数

Non-OS SDK 提供了 GPIO 电平读写的 `GPIO_INPUT_GET()` 和 `GPIO_OUTPUT_SET()` 接口，用其替换掉原有的 STM32 HAL 接口：

```c
void EpdDigitalWriteCallback(int pin_num, int value) {
  if (value == HIGH) {
    GPIO_OUTPUT_SET(GPIO_ID_PIN(pins[pin_num].pin), HIGH); 
  } else {
    GPIO_OUTPUT_SET(GPIO_ID_PIN(pins[pin_num].pin), LOW); 
  }
}

int EpdDigitalReadCallback(int pin_num) {
  GPIO_DIS_OUTPUT(pins[pin_num].pin);
  if (GPIO_INPUT_GET(pins[pin_num].pin) == HIGH) {
     return HIGH;
   } else {
     return LOW;
   } 
}
```

> 注：在读电平前需要调用 `GPIO_DIS_OUTPUT()` 配置引脚为输入模式。

#### 毫秒延时函数

在上一篇有提到，8266 SDK 只提供微秒延时接口 `os_delay_us()`，且最长延时值为 65535us，65 毫秒的延时似乎不太够用。可通过反复调用微秒小延时实现大的毫秒延时，思路是把大块的时间拆分为多个 6ms，整数倍部分循环调用 `os_delay_us()` 接口，余下不足 65ms 部分最后单独调用一次延时补足，代码如下：

```c
void EpdDelayMsCallback(unsigned int delaytime) {
  int count = delaytime / 65;
  int i;
  for (i = 0; i < count; i++)
  {
    os_delay_us(65000);
  }
  os_delay_us(delaytime%65*1000);
}
```

> 注：此延时函数并不精准，因为没考虑函数调用的时间成本。

#### SPI 数据发送函数

用自己编写的软件 SPI 字节发送函数实现底层数据传输接口，同时控制 CS 引脚电平完成软件片选：

```c
void EpdSpiTransferCallback(unsigned char data) {
  GPIO_OUTPUT_SET(GPIO_ID_PIN(pins[CS_PIN].pin), LOW);   
  Spi_Write_Byte(data);
  GPIO_OUTPUT_SET(GPIO_ID_PIN(pins[CS_PIN].pin), HIGH);
}
```

#### 初始化函数

最后在初始化函数 `EpdInitCallback()`  末尾加上 SPI 初始化与墨水屏其余引脚配置，此接口会在屏幕初始化前被调用，确保所有外设已准备就绪。

```c
int EpdInitCallback(void) {
  pins[CS_PIN] = epd_cs_pin;
  pins[RST_PIN] = epd_rst_pin;
  pins[DC_PIN] = epd_dc_pin;
  pins[BUSY_PIN] = epd_busy_pin;

  SPI_Init();
  
  PIN_FUNC_SELECT(EPAPER_DC_REG, EPAPER_DC_FUN);     //configure io to gpio mode
  PIN_FUNC_SELECT(EPAPER_RST_REG, EPAPER_RST_FUN);   //configure io to gpio mode
  PIN_FUNC_SELECT(EPAPER_BUSY_REG, EPAPER_BUSY_FUN); //configure io to gpio mode
  
  return 0;
}
```

至此，硬件相关所有代码大致修改完毕，理论上能实现例程的墨水屏所有基本操作，不过细节还有待完善。

---

### STEP4: 【坑】修改 `EPD_WaitUntilIdle()` 函数

 `BSP/epd2in9.c` 109 行的 `EPD_WaitUntilIdle()` 函数起到等待墨水屏刷新的作用。如果对上篇提到的「喂狗问题」有印象的话，就一定别忘了修改这坑爹函数，否则当你做完所有工作会发现程序只能运行一半，然后就无限重启。这对不了解 Non-OS SDK 的人来说绝对是一个大坑。修改步骤十分简单 —— 等待时调用喂狗函数 `system_soft_wdt_feed()` 即可（需包含 `user_interface.h`）：

 ```c
void EPD_WaitUntilIdle(EPD* epd) {
  while(EPD_DigitalRead(epd, epd->busy_pin) == HIGH) {      //0: busy, 1: idle
    EPD_DelayMs(epd, 100);
    system_soft_wdt_feed();
  }      
}
 ```

#### 编译测试

核心驱动代码基本移植完毕，可以先编译看下效果。但编译之前要把入口函数文件处理一下 —— 删除 `user/user_main.c` 中一些不存在的头文件包含、函数调用，以及各种多余的东西。只保留 `#include "ets_sys.h"`、`#include "osapi.h"`、和 `#include "user_interface.h"` 三个头文件。入口函数 `user_init()` 只保留 `os_printf("SDK version:%s\n", system_get_sdk_version());` 打印测试语句。

在虚拟机 lubuntu 中进入 `ESP8266_NONOS_SDK-2.1.0/e-paper` 目录，使用 `./gen_misc.sh` 命令开始编译。完了会报一些错误，但都是像找不到 `stm32f1xx_hal.h` 之类的头文件小问题，注释掉原来的无效代码基本能解决。编译通过后提示如下：

![图1 编译通过](http://odaps2f9v.bkt.clouddn.com/18-4-25/11037653.jpg)

 ---

### STEP5: `main.c` 测试代码移植

把底层驱动接口移植好之后，便能在上层调用各种绘图函数显示内容了，这也是检验移植成功与否的唯一标准。例程移植工作的最后一步便是移植 `main()` 函数测试代码，在 `user_main()` 上实现相同功能。

打开 `stm32/Src/main.c` 和 `e-paper/user/user_main.c` 两个文件，分别移植上一篇「官方 STM32 例程分析」部分所划分的几大部分代码，其中部分刷新走时代码改用定时器实现。

#### 头文件、变量声明定义

包含例程中用到的头文件，颜色宏定义：

```c
#include "spi.h"
#include "epd2in9.h"
#include "epdif.h"
#include "epdpaint.h"
#include "imagedata.h"
#include "mem.h"

#define COLORED      0
#define UNCOLORED    1
```

把用到的数据通过全局变量方式定义：

```c
unsigned char *frame_buffer;
char time_string[] = {'0', '0', ':', '0', '0', '\0'};
unsigned long time_start_ms;
unsigned long time_now_s;

EPD epd;
Paint paint;
```

#### 初始化部分

系统时钟、串口等初始化工作 SDK 以及帮我们做好了，在入口函数 `user_init()` 中先使用系统接口 `os_malloc()` 分配显示缓存空间（需包含 `mem.h`），然后调用墨水屏初始化函数 `EPD_Init()`（GPIO、SPI 初始化会在内部先被调用），同时把 `printf()` 函数替换为 SDK 的 `os_printf()`。`user_init()` 无返回值类型，初始化失败后直接 `retrun;` 结束程序。接着的的画板初始化代码不变：

```
void ICACHE_FLASH_ATTR
user_init(void)
{
  frame_buffer = (unsigned char*)os_malloc(EPD_WIDTH * EPD_HEIGHT / 8);
  if (EPD_Init(&epd, lut_full_update) != 0) {
    os_printf("e-Paper init failed\n");
    return;
  }

  Paint_Init(&paint, frame_buffer, epd.width, epd.height);
  Paint_Clear(&paint, UNCOLORED);
```

#### 字符图片显示部分

都是封装好的函数，无需改动，照搬即可（记得顺手改掉 `printf()`）：

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
    os_printf("e-Paper init failed\n");
    return;
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

#### 局部刷新走时部分

STM32 程序采取的死循环和延时的方式不符合 Non-OS SDK 的编程规范，可以改为软件定时器实现。全局变量 `count_timer` 便是待使用的定时器，接着 `user_init()` 继续编码。 `system_get_time()` 接口获取系统时间（单位：us），后面的三行是 SDK 定时器标准初始化步骤：

```c
os_timer_t count_timer;

time_start_ms = system_get_time()/1000;

os_timer_disarm(&count_timer);
os_timer_setfn(&count_timer, (os_timer_func_t *)time_tick, NULL);
os_timer_arm(&count_timer, 500, 1);
```

`os_timer_setfn()` 设置回调函数为 `time_tick()`，无参无返回值类型，到时间后自动被调用。`os_timer_arm()` 接口使能定时器，500 ms 周期重复定时。最后定义回调函数 `time_tick()`，移植原来 `while (1)` 中的代码，程序移植大功告成：

```c
void ICACHE_FLASH_ATTR time_tick(void)
{
    time_now_s = (system_get_time()/1000 - time_start_ms) / 1000;
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
}
```

重新编译程序排除一些小问题，生成待下载的 `eagle.flash.bin` 和 `eagle.irom0text.bin` 文件。

---

### STEP6: 烧写固件查看效果

打开烧写工具 ESPFlashDownloadTool，按照下图配置进行烧写，复位后程序开始运行，显示效果与 STM32 程序一致，ESP8266 驱动墨水屏成功！。

![图2 下载配置](http://odaps2f9v.bkt.clouddn.com/18-4-25/17991768.jpg)

![图3 显示效果](http://odaps2f9v.bkt.clouddn.com/18-4-25/24200537.jpg)

> 参考资料
> 
> * ESP8266 系统描述
> * [2.9inch-e-paper-user-manual-cn.pdf](http://www.waveshare.net/w/upload/c/cd/2.9inch-e-paper-user-manual-cn.pdf)
> * [ESP8266 Non-OS SDK API 参考](https://www.espressif.com/sites/default/files/documentation/2c-esp8266_non_os_sdk_api_reference_cn.pdf)
> * [ESP8266 技术参考](https://www.espressif.com/sites/default/files/documentation/esp8266-technical_reference_cn.pdf)
