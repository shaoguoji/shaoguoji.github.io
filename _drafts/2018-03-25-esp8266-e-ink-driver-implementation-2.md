---
layout:           post
title:             ESP8266(Non-OS SDK) 驱动 waveshare 2.9 寸墨水屏（二）
subtitle:      程序移植与测试
date:               2018-03-25 15:52:09 +0800
author:           Shao Guoji
header-img:  
catalog:         true
tag:
    - 嵌入式
    - 单片机
    - ESP8266
---

### 开始移植

这一步我们的目的是把上面在 STM32 跑的程序，完整地搬到 esp8266 上，达到相同的运行显示效果。开始程序移植工作之前先准备用到的文件：

1. 把   `IoT_Demo` 文件夹拷贝一份到 VirtualBox 虚拟机共享文件夹编译目录下，命名为 `e-paper`。
2. 将 `stm32` 目录实例代码中的 `BSP` 和 `Fonts` 文件夹复制到 `e-paper` 下。
3. 删除 `driver` 目录下的 `i2c_master.c` 和 `key.c` 文件，删除 `include` 目录下所有文件。
4. 删除 `user` 目录下除 `Makefile` 和 `user_main.c` 的所有文件。

#### 软件 SPI 接口代码编写

接着编写 `spi.c` 文件完成 GPIO 初始化和 SPI 时序模拟（发送一个字节数据），保存到 `driver` 中。代码如下：

```c
#include "driver/spi.h"

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

再新建 `spi.h` 文件保存到 `include/driver` 目录中，头文件包括 GPIO 引脚定义以及函数声明。`Spi_Write_Byte()`  函数只是实现主机到从机的单向数据传输，没用使用 MISO 引脚。`EPAPER_DIN_PIN`、`EPAPER_CLK_PIN`、`EPAPER_CS_PIN` 三个引脚的宏在更上层的头文件 `epaper.h` 中定义。

```c
#ifndef __SPI_H__
#define __SPI_H__

#include "os_type.h"
#include "gpio.h"

#include "epaper.h"

//*****************************************************************************
//
// Make sure all of the definitions in this header have a C binding.
//
//*****************************************************************************

#ifdef __cplusplus
extern "C"
{
#endif

// #define  SPI_MISO_PIN    12
#define     SPI_MOSI_PIN    EPAPER_DIN_PIN
#define     SPI_CLK_PIN     EPAPER_CLK_PIN
#define     SPI_CS_PIN      EPAPER_CS_PIN

// #define  SPI_MISO_PIN_REG    PERIPHS_IO_MUX_MTDI_U
#define     SPI_MOSI_PIN_REG    PERIPHS_IO_MUX_MTMS_U // PERIPHS_IO_MUX_MTDO_U
#define     SPI_CLK_PIN_REG     PERIPHS_IO_MUX_MTDI_U // PERIPHS_IO_MUX_MTCK_U
#define     SPI_CS_PIN_REG      PERIPHS_IO_MUX_MTCK_U // PERIPHS_IO_MUX_MTMS_U

// #define  SPI_MISO_PIN_FUN    FUNC_GPIO12
#define     SPI_MOSI_PIN_FUN    FUNC_GPIO13
#define     SPI_CLK_PIN_FUN     FUNC_GPIO14
#define     SPI_CS_PIN_FUN      FUNC_GPIO15

void SPI_Init(void);
void Spi_Write_Byte(uint8_t data);

#ifdef __cplusplus
}
#endif

#endif //  __SPI_H__

```

#### 

















