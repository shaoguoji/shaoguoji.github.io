---
layout:     post
title:      STM32 实现 MPU6050 数据读取与倾角检测
subtitle:   三轴陀螺仪 + 三轴加速度传感器「从入门到放弃」
date:       2017-10-13
author:     Shao Guoji
header-img: img/post-bg-MPU6050-module.jpg
catalog:    true
tag:
    - 单片机
    - STM32
    - 嵌入式
---

![重力感应流水灯](http://odaps2f9v.bkt.clouddn.com/video2gif_20171023_210654.gif)

### 前言

MPU6050 是一个很好玩传感器，在四轴、体感、计步等应用领域都能看到这小芯片的影子，其内部的结构、功能十分丰富，可玩度非常高。同时，对传感器采集到的数据进行分析还能得到许多信息，但此时的一些「数学小技巧」或许会让你抓狂，所以你会在网上疯狂查找资料，最终发现了本文。

然而很可惜，笔者也是一名数学渣渣（买东西找零都不会算的那种），所以这篇文章并不能教会你「卡尔曼滤波」、「协方差阵列」等连我自己都不懂的东西。**事实上如果你只是想从传感器读出陀螺仪和加速度值，并简单计算一下坐标倾角（不涉及姿态解算、四元素等），其实没有想象中那么难。**

**PS：MPU6050 的确涉及到许多令人敬畏的数学知识，但如果只是想得到「简陋」的坐标夹角，完全没必要搬出「加速度陀螺仪融合」、「四元素欧拉角」、「陀螺仪积分」、「内部的 DMP」等名词来唬人，用高中三角函数知识已足够。当你有更高精度数据的需求时，也没必要自己从头造轮子，直接移植现成的库即可。本文将实现前一种设想 —— 使用简陋方法计算粗略角度**

文章分为 3 个部分（实验）：

1. 利用 I2C 协议从传感器读出 6 个数据（三轴陀螺仪 + 三轴加速度值）
2. 基于 3 个加速度值，通过反三角函数计算传感器与各坐标轴夹角（倾角）
3. 利用计算到倾角做个小玩意 —— 通过「重力感应（倾斜开发板）」控制流水灯的速度、方向

---

### 预备知识

* STM32 基本开发方法
* I2C 协议通讯基本流程原理
* 基本物理力学常识
* 简单三角函数知识

---

### 角速度与加速度

#### 角速度

**[角速度](https://zh.wikipedia.org/wiki/%E8%A7%92%E9%80%9F%E5%BA%A6)描述物体的旋转的快慢，具体指物体在单位时间内转过的角度。**举个最简单的例子：钟表上的秒针 60 秒转动一圈（360 度），故秒针单位时间内转过的角度为：360 / 60 = 6 度/秒，即秒针转动的角速度。通过角速度和运动时间能计算出秒针转过角度。

在平面中，物体总是围绕着一个「旋转中心」进行旋转（如表盘的圆心），而在三维空间中，物体是围绕着一根「旋转轴」进行旋转（比如，烤羊肉串？），并且**绕任何「旋转轴」的旋转都可以分解为空间直角坐标系中 X、Y、Z 三个轴上旋转的合成**，这也是 MPU6050 所测量的三个角速度值，同理，得到三轴角速度分量就可以确定任一旋转状态。

![图1 绕三轴旋转示意图](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/70275127.jpg)

#### 加速度

加速度是指速度变化的快慢，稍有物理常识的人都知道：只有在外力的作用下，物体的速度才会变化，所以把加速度简单理解为**物体受力情况**似乎也没毛病。举个例子，用力把一块石头扔出去瞬间的加速度比捧着石头慢慢移动的加速度要大得多。此外，加速度同样也可以分解为三个坐标轴上的分量。

#### MPU6050 加速度方向

需要注意的一点是，**MPU6050 读到加速度与受力方向相反。**

**传感器在静止状态受到向下的重力，测量到的加速度向上（了解这点很重要）。**同理，如果给一个水平向左的力，测量到的加速度向右。这也是一些资料中「箱子和球」模型所表达的原理。

![图2 失重状态](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/61409401.jpg)

![图3 受力与加速度方向相反](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/13079086.jpg)

### MPU6050 介绍

> MPU-60x0 是全球首例 9 轴运动处理传感器。它集成了 3 轴MEMS陀螺仪，3 轴MEMS加速度计，以及一个可扩展的数字运动处理器 DMP（Digital Motion Processor）。

> MPU-60x0  对陀螺仪和加速度计分别用了三个 16 位的 ADC，将其测量的模拟量转化为可输出的数字量。为了精确跟踪快速和慢速的运动，传感器的测量范围都是用户可控的，陀螺仪可测范围为 ±250，±500，±1000，±2000°/秒（dps），加速度计可测范围为 ±2，±4，±8，±16g。

#### 应用范围

从上面的分析可知道，角速度能反映物体的圆周运动状态，加速度能能反映物体的静止和直线运动状态，两者互补就能识别很复杂的运动状态，如四轴飞行器的姿态、人行走的步态、空中鼠标的位移量、手机横屏竖屏等等。**可以说几乎所有与运动、位置、姿态相关的应用都可以用加速度陀螺仪传感器实现。**当然，从成本、开发难度考虑并不会完全这样做。

---

### 传感器坐标系

作为测量值的方向参考，传感器坐标方向定义如下图，属于右手坐标系（右手拇指指向 x 轴的正方向，食指指向 y 轴的正方向，中指能指向 z 轴的正方向）：

![图4 传感器坐标方向](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/76531996.jpg)

---

### 模块接线方法

我是使用公司 STM32F407ZGT6 开发板上集成的 MP6050 芯片，可免去接线的麻烦，如果开发板上没有相应芯片就只能使用模块了，接线如下图：

![图5 MPU6050 模块接线](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/13400884.jpg)

*注：使用模拟 I2C 时 GPIO 不固定，图中的 PB8 和 PB9 可以根据实际情况进行更改，与代码对应即可。*

---

### I2C 基本时序函数 

MPU6050 与 MCU 通过 I2C 总线进行通讯。用软件模拟的方式实现 I2C 底层基本时序函数，包括起始、停止信号的产生，以及发送/接收单字节数据、检测/发送应答信号。

```c
// 【基础】基本数据读取\USER\src\i2c.h

void I2C_Init(void); // I2C 初始化 
void I2C_Start(void); // 产生 I2C 协议起始信号 
void I2C_Stop(void); //  产生 I2C 协议结束信号 
void I2C_Write_Byte(uint8_t byte); // 发送八位数据（不包含应答）  
uint8_t I2C_Read_Byte(void); // 读取八位数据（不包含应答） 
uint8_t I2C_Read_ACK(void); // 接收应答信号  
void I2C_Write_ACK(uint8_t ack); // 发送应答信号  
```

#### 传感器 I2C 设备地址

MCU 作为主机与传感器通讯前需要发送 7 位的从机设备地址，**设备地址的惯用套路是固定高位，并通过引脚电平确定低位。**查阅寄存器手册得知，117 号寄存器 `WHO AM I` 决定着设备地址的高 6 位，上电复位值为 `110100`，最低 1 位则由外部引脚 `AD0` 决定（即一块电路板最多只能有两个 MPU6050）。查看模块或开发板电路图确定芯片 AD0 的电平（一般为低）最终得到 7 位的 I2C 从机设备地址为 `1101 000（0xD0）`，在 `mpu6050.h` 文件中宏定义为 `DEV_ADDR` 。

![图6 WHO AM I 寄存器](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/95135836.jpg)

*注：文章侧重讲解传感器使用，I2C 函数具体实现见文末示例代码。但在进行下面的实验之前，请确保你的 I2C 通讯是正常的、发送器件地址能得到应答。*

---

### MPU6050 基本操作函数 

在实现了基本时序函数后，使用 I2C 总线跟外部设备通信实际上就是基本时序的拼接。下面编写 MPU6050 相关函数，其中读写寄存器函数是核心操作。

#### 写寄存器

##### 写寄存器流程如下：

1. 发送起始信号
2. 发送设备地址（写模式）
3. 发送内部寄存器地址
4. 写入寄存器数据（8 位数据宽度）
5. 发送结束信号

##### 代码实现：

```c
// 【基础】基本数据读取\USER\src\mpu6050.c

/******************************************************************************
* 函数介绍： MPU6050 写寄存器函数
* 输入参数： regAddr：寄存器地址
             regData：待写入寄存器值
* 输出参数： 无
* 返回值 ：  无
******************************************************************************/
void MPU6050_Write_Reg(uint8_t regAddr, uint8_t regData)
{
    /* 发送起始信号 */
    I2C_Start();
    
    /* 发送设备地址 */        
    I2C_Write_Byte(DEV_ADDR);
    if (I2C_Read_ACK())
        goto stop;
    
    /* 发送寄存器地址 */
    I2C_Write_Byte(regAddr);
    if (I2C_Read_ACK())
        goto stop;
    
    /* 写数据到寄存器 */
    I2C_Write_Byte(regData);
    if (I2C_Read_ACK())
        goto stop;
    
stop:
    I2C_Stop();
}
```

#### 读寄存器

##### 读寄存器流程如下：

1. 发送起始信号
2. 发送设备地址（写模式）
3. 发送内部寄存器地址
**4. 发送重复起始信号**
**5. 发送设备地址（读模式）**
6. 读取寄存器数据（8 位数据宽度）
7. 发送结束信号

##### 代码实现：

```c
// 【基础】基本数据读取\USER\src\mpu6050.c

/******************************************************************************
* 函数介绍： MPU6050 读寄存器函数
* 输入参数： regAddr：寄存器地址
* 输出参数： 无
* 返回值 ：  regData：读出的寄存器数据
******************************************************************************/
uint8_t MPU6050_Read_Reg(uint8_t regAddr)
{
    uint8_t regData;
    
    /* 发送起始信号 */
    I2C_Start();
    
    /* 发送设备地址 */        
    I2C_Write_Byte(DEV_ADDR);
    if (I2C_Read_ACK())
        goto stop;
    
    /* 发送寄存器地址 */
    I2C_Write_Byte(regAddr);
    if (I2C_Read_ACK())
        goto stop;
    
    /* 发送重复起始信号 */
    I2C_Start();
    
    /* 发送读模式设备地址 */     
    I2C_Write_Byte(DEV_ADDR | 0x01);
    if (I2C_Read_ACK())
        goto stop;
    
    /* 读寄存器数据 */
    regData = I2C_Read_Byte();
    I2C_Write_ACK(1);  // 非应答信号
    
stop:
    I2C_Stop();
    
    return regData;
}
```

#### 寄存器地址宏定义

在 `mpu6050.h` 文件中对常用的寄存器地址进行宏定义，方便使用并提高程序可读性。

```c
// 【基础】基本数据读取\USER\src\mpu6050.h

#define DEV_ADDR    0xD0    // 6050 器件地址

//-----------------------------------------
// 定义MPU6050内部地址
//-----------------------------------------
#define SMPLRT_DIV      0x19    //陀螺仪采样率，典型值：0x07(125Hz)
#define CONFIG          0x1A    //低通滤波频率，典型值：0x06(5Hz)
#define GYRO_CONFIG     0x1B    //陀螺仪自检及测量范围，典型值：0x18(不自检，2000deg/s)
#define ACCEL_CONFIG    0x1C    //加速计自检、测量范围及高通滤波频率，典型值：0x01(不自检，2G，5Hz)

/* 加速度相关寄存器地址 */
#define ACCEL_XOUT_H    0x3B
#define ACCEL_XOUT_L    0x3C
#define ACCEL_YOUT_H    0x3D
#define ACCEL_YOUT_L    0x3E
#define ACCEL_ZOUT_H    0x3F
#define ACCEL_ZOUT_L    0x40

/* 温度相关寄存器地址 */
#define TEMP_OUT_H      0x41
#define TEMP_OUT_L      0x42

/* 陀螺仪相关寄存器地址 */
#define GYRO_XOUT_H     0x43
#define GYRO_XOUT_L     0x44    
#define GYRO_YOUT_H     0x45
#define GYRO_YOUT_L     0x46
#define GYRO_ZOUT_H     0x47
#define GYRO_ZOUT_L     0x48

#define PWR_MGMT_1      0x6B    //电源管理，典型值：0x00(正常启用)
#define WHO_AM_I        0x75    //IIC地址寄存器(默认数值0x68，只读)
#define SlaveAddress    0xD0    //IIC写入时的地址字节数据，+1为读取

```

#### 传感器初始化

在使用传感器测量数据之前，先要利用前面写好的读写寄存器函数，对传感器初始化，包括常用参数配置，如采样率、滤波频率等，如无特殊要求使用典型值即可（各参数具体含义请查阅芯片寄存器手册）。

##### 代码实现：

```c
// 【基础】基本数据读取\USER\src\mpu6050.c

/******************************************************************************
* 函数介绍： MPU6050 初始化函数
* 输入参数： 无
* 输出参数： 无
* 返回值 ：  无
* 备   注：  配置 MPU6050 测量范围：± 2000 °/s  ± 2g
******************************************************************************/
void MPU6050_Init(void)
{
    I2C_Init();  // I2C 初始化
    

    MPU6050_Write_Reg(PWR_MGMT_1, 0x00);    //解除休眠状态
    MPU6050_Write_Reg(SMPLRT_DIV, 0x07);    //陀螺仪采样率，典型值：0x07(125Hz)
    MPU6050_Write_Reg(CONFIG, 0x06);        //低通滤波频率，典型值：0x06(5Hz)
    MPU6050_Write_Reg(GYRO_CONFIG, 0x18);   //陀螺仪自检及测量范围，典型值：0x18(不自检，2000deg/s)
    MPU6050_Write_Reg(ACCEL_CONFIG, 0x01);  //加速计自检、测量范围及高通滤波频率，典型值：0x01(不自检，2G，5Hz)
}
```

#### 合成 16 位完整数据

传感器原始数据 AD 值为 16 位数字量，从寄存器地址定义宏名可知，一个数据需要两个字节（寄存器）来表示，如 `ACCEL_XOUT_H` 和 `ACCEL_XOUT_L` 两个寄存器分别存储 X 轴加速度高 8 位和低 8 位，共同组成 16 位数据，按照这个思路，编写一个连续读两个寄存器并合成 16 位数据的函数。

##### 代码实现：

```c
// 【基础】基本数据读取\USER\src\mpu6050.c

/******************************************************************************
* 函数介绍： 连续读两个寄存器并合成 16 位数据
* 输入参数： regAddr：数据低位寄存器地址
* 输出参数： 无
* 返回值 ：  data：2 个寄存器合成的 16 位数据
******************************************************************************/
int16_t MPU6050_Get_Data(uint8_t regAddr)
{
    uint8_t Data_H, Data_L;
    uint16_t data;
    
    Data_H = MPU6050_Read_Reg(regAddr);
    Data_L = MPU6050_Read_Reg(regAddr + 1);
    data = (Data_H << 8) | Data_L;  // 合成数据

    return data;
}
```

---

### 基础功能 —— 读取原始数据 

写好读 16 位数据的函数，获取加速度、陀螺仪数值自然便水到渠成。只需给出待读取数据高位寄存器地址（因为高位寄存器地址在前），调用 `MPU6050_Get_Data()` 函数即可得到合成后 16 位加速度、温度、陀螺仪数值。如 `MPU6050_Get_Data(ACCEL_XOUT_H)` 表示读取 16 位 X 轴加速度数据。

在 `main.c` 中编写 `MPU6050_Display` 函数，实现数据读取并在串口打印。

```c
// 【基础】基本数据读取\USER\src\main.c

/******************************************************************************
* 函数介绍： 串口打印 6050 传感器读取的三轴加速度、陀螺仪及温度数据
* 输入参数： 无
* 输出参数： 无
* 返回值 ：  无
******************************************************************************/
void MPU6050_Display(void)
{
    /* 打印 x, y, z 轴加速度 */
    printf("ACCEL_X: %d\t", MPU6050_Get_Data(ACCEL_XOUT_H));
    printf("ACCEL_Y: %d\t", MPU6050_Get_Data(ACCEL_YOUT_H));
    printf("ACCEL_Z: %d\t", MPU6050_Get_Data(ACCEL_ZOUT_H));
    
    /* 打印温度，需要根据手册的公式换算为摄氏度 */
    printf("TEMP: %0.2f\t", MPU6050_Get_Data(TEMP_OUT_H) / 340.0 + 36.53);
    
    /* 打印 x, y, z 轴角速度 */
    printf("GYRO_X: %d\t", MPU6050_Get_Data(GYRO_XOUT_H));
    printf("GYRO_Y: %d\t", MPU6050_Get_Data(GYRO_YOUT_H));
    printf("GYRO_Z: %d\t", MPU6050_Get_Data(GYRO_ZOUT_H));
    
    printf("\r\n");
}
```

```c
// 【基础】基本数据读取\USER\src\main.c

/******************************************************************************
* 函数介绍： 主函数
* 输入参数： 无
* 输出参数： 无
* 返回值 ：  无
******************************************************************************/
int main()
{
    MPU6050_Init();
    Usrat_1_Init(84,9600,0);    
    
    while (1)
    {
//      Usart_Send_Byte(MPU6050_Read_Reg(GYRO_XOUT_L));

        MPU6050_Display();  // 串口打印传感器数据
        Delay(0xfffff);
    }
}
```

编译下载程序、连接开发板与电脑并打开串口工具。抖动、倾斜开发板会看到数据随之变化。

##### 实验效果：

![图7 基本数据串口打印](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/36805603.jpg)

#### 单位换算

如需把原始的 16 位数据用重力加速度单位 g 表示，需要注意传感器的量程。默认测量范围是 ±2g，一共是 4g 的宽度，16 位数据的最大读数为 65536，除 4 就是 16384，即 1g 加速度对应的数值。打印前把数据除以 16384 即可得到对应单位为 g 的数值，角速度的单位转换也同理。

```c
    /* 打印 x, y, z 轴加速度，单位：g */
    printf("ACCEL_X: %lf\t", MPU6050_Get_Data(ACCEL_XOUT_H) / 16384.0);
    printf("ACCEL_Y: %lf\t", MPU6050_Get_Data(ACCEL_YOUT_H) / 16384.0);
    printf("ACCEL_Z: %lf\t", MPU6050_Get_Data(ACCEL_ZOUT_H) / 16384.0);
```

![图8 数据单位换算](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/14056577.jpg)

#### 零偏校准

由于每个芯片在制作时都不一样，芯片固定在 PCB 上也会有位置误差，就会导致读到的数据有偏差，对传感器进行零偏校准可一定程度上减小这种误差的影响。对程序作最后修改，在读到的数据基础上加上一个补偿值 `OFFSET` 对数据校准。

##### 代码实现

```c
// 基本数据读取\USER\src\main.c

/* 传感器数据修正值（消除芯片固定误差，根据硬件进行调整） */
#define X_ACCEL_OFFSET  -600
#define Y_ACCEL_OFFSET  -100
#define Z_ACCEL_OFFSET  2900

#define X_GYRO_OFFSET   32
#define Y_GYRO_OFFSET   -11
#define Z_GYRO_OFFSET   1

/******************************************************************************
* 函数介绍： 串口打印 6050 传感器读取的三轴加速度、陀螺仪及温度数据
* 输入参数： 无
* 输出参数： 无
* 返回值 ：  无
******************************************************************************/
void MPU6050_Display(void)
{
    /* 打印 x, y, z 轴加速度 */
    printf("ACCEL_X: %d\t", MPU6050_Get_Data(ACCEL_XOUT_H) + X_ACCEL_OFFSET);
    printf("ACCEL_Y: %d\t", MPU6050_Get_Data(ACCEL_YOUT_H) + Y_ACCEL_OFFSET);
    printf("ACCEL_Z: %d\t", MPU6050_Get_Data(ACCEL_ZOUT_H) + Z_ACCEL_OFFSET);
    
    /* 打印温度 */
    printf("TEMP: %0.2f\t", MPU6050_Get_Data(TEMP_OUT_H) / 340.0 + 36.53);
    
    /* 打印 x, y, z 轴角速度 */
    printf("GYRO_X: %d\t", MPU6050_Get_Data(GYRO_XOUT_H) + X_GYRO_OFFSET);
    printf("GYRO_Y: %d\t", MPU6050_Get_Data(GYRO_YOUT_H) + Y_GYRO_OFFSET);
    printf("GYRO_Z: %d\t", MPU6050_Get_Data(GYRO_ZOUT_H) + Z_GYRO_OFFSET);
    
    printf("\r\n");
}
```

把开发板/模块的放置在水平位置（或其他参考位置）保持静止，观察并记录输出的原始数据，不断调整各个补偿值 `OFFSET` 的大小，确保原始数据在 X、Y、Z 轴加速度为 0，0，16384（对应 1 g） 左右，且三轴角速度值为 0。

---

### 扩展操作 —— 坐标倾角计算 

能读到原始数据来只是最基础、甚至是没什么卵用的，下面将进行简单的计算，**将加速度数据转换为与重力（方向向上）的夹角，这才是重点好嘛！**当然，涉及到的运算非常简单 —— 只涉及到三角函数（复杂的算法网上已经有用 Arduino 库、匿名、圆点博士、 DMP 库等实现）。

#### 测量原理 —— 三角函数的妙用  

只考虑 X 轴和 Z 轴（此时Y 轴垂直于屏幕），将重力加速度在 X 轴上分解，传感器 X 坐标正方向与重力加速度 `g` 的夹角为 `∠b`，如下图所示：

![图9 重力分解](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/29092047.jpg)

从图中可以看到，重力加速度 `g`（绿色箭头）分解为 X 轴上的 `gx`（黑色箭头）和 Z 轴上的 `gz`（紫色箭头）， `g` 与 `gx` 的夹角 `∠b` 的余弦值 `cos(∠b) = gx / g`，对 `gx / g` 求反余弦即可得到 `∠b` 的值，即 `∠b = arccos(gx / g)`，这便是求倾角的原理。

同理，Z 轴与重力的夹角也可以通过 ``arccos(gz / g)`` 求出来，此时 gz 落在 Z 轴负半轴所以 `gz / g` 为负数，算出来的角度是大于 90° 的，大小等于图中的 `180 - ∠c`。Y 轴则始终垂直于重力方向，夹角为 90°。

将三个角度值用结构体封装，编写 `MPU6050_Get_Angle()` 函数获取夹角。利用 C 语言 `math.h` 头文件提供的 `acos()` 库函数可以很方便计算反余弦函数，不过返回的是弧度值，要转换为角度的话需乘上 `57.29577`。

#### 代码实现：

```c
// 【扩展】坐标倾角计算\USER\src\main.c

/* 坐标角度结构体 */
typedef struct Angle
{
    double X_Angle;
    double Y_Angle;
    double Z_Angle;
    
} MPU6050_Angle

/******************************************************************************
* 函数介绍： 计算 x, y, z 三轴的倾角
* 输入参数： 无
* 输出参数： data：角度结构体
* 返回值 ：  无
******************************************************************************/
void MPU6050_Get_Angle(MPU6050_Angle *data)
{   
    /* 计算x, y, z 轴倾角，返回弧度值*/
    data->X_Angle = acos((MPU6050_Get_Data(ACCEL_XOUT_H) + X_ACCEL_OFFSET) / 16384.0);
    data->Y_Angle = acos((MPU6050_Get_Data(ACCEL_YOUT_H) + Y_ACCEL_OFFSET) / 16384.0);
    data->Z_Angle = acos((MPU6050_Get_Data(ACCEL_ZOUT_H) + Z_ACCEL_OFFSET) / 16384.0);

    /* 弧度值转换为角度值 */
    data->X_Angle = data->X_Angle * 57.29577;
    data->Y_Angle = data->Y_Angle * 57.29577;
    data->Z_Angle = data->Z_Angle * 57.29577;
}
```

在主函数中不断读取实时倾角并打印：

```c
// 【扩展】坐标倾角计算\USER\src\main.c

/******************************************************************************
* 函数介绍： 主函数
* 输入参数： 无
* 输出参数： 无
* 返回值 ：  无
******************************************************************************/
int main()
{
    MPU6050_Angle data;
    
    MPU6050_Init();
    Usrat_1_Init(84,9600,0);    
    
    while (1)
    {
        MPU6050_Get_Angle(&data);  // 计算三轴倾角
        
        printf("X_Angle = %lf°  ", data.X_Angle);
        printf("Y_Angle = %lf°  ", data.Y_Angle);
        printf("Z_Angle = %lf°  ", data.Z_Angle);
        printf("\r\n");
        
        Delay(0xfffff);
    }
}
```

**需要再次强调，传感器实际测得的重力方向是向上的，计算出的夹角是传感器三个坐标轴与竖直向上重力向量的夹角，开发板水平放置时，传感器 X、Y 轴坐标垂直于重力，Z 轴与重力夹角为 0。**

##### 实验效果：

![图10 倾角测量数据打印](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/69512820.jpg)

---

### 来点好玩的？ —— 重力感应流水灯 

智能手机和平板刚兴起那会儿，最吸引我的就是「重力感应」的功能 —— 通过倾斜手机，实现屏幕自动旋转、重力感应游戏，例如控制屏幕上的一个小球「走迷宫」的游戏：

![图11 重力感应小球游戏](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/96075858.jpg)

是否能利用开发板上的四个 LED 实现类似的功能呢？通过倾斜开发板控制单个 LED 的流动方向、速度，不也实现了「小球滚动」的效果么？想想就有点小激动，废话不多说，理一下思路马上开干！

#### 单步流水灯函数

先从简单的 LED 下手。将流水灯的「一次流动」封装为函数，接收一个参数 `direction` 表示流动方向（左/右），流动速度通过控制函数调用频率调节。

##### 代码实现

```c
// 【应用】重力感应流水灯\USER\src\led.c

/******************************************************************************
* 函数介绍： 单步流水灯
* 输入参数： direction：流动方向  0：左  1：右
* 输出参数： 无
* 返回值 ：  无
******************************************************************************/
void LED_flow(uint16_t direction)
{
    static uint8_t state = 1;
    
    switch (state)
    {
        case 0:
            LED_ON(LED1_PIN);
            break;
        case 1:
            LED_ON(LED2_PIN);
            break;
        case 2:
            LED_ON(LED3_PIN);
            break;
        case 3:
            LED_ON(LED4_PIN);
            break;
    }
    
    state = (direction == DIR_LEFT) ? state - 1 : state + 1;
    state = state % 4;  
}
```

*PS：其实这个函数写的并不好，使用静态变量会导致函数状态不确定。*

#### 开发板倾角分析

![图12 开发板坐标](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/59088628.jpg)

开发板四个 LED 沿着 X 轴方向横向排列，让开发板绕 Y 轴旋转，同时读取 Z 轴 和 X 轴与重力倾角。**Z 轴倾角表示倾斜程度，倾角越大倾斜越厉害。X 轴倾角表示倾斜方向，小于 90°时向左倾，大于 90°时向右倾。**

不同倾斜情况时 X 轴、Z 轴对应夹角示意图如下：

![图13 开发板倾斜示意图](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/1847671.jpg)

通过测量 `∠a` 和 `∠b` 两个夹角分别控制流水灯的速度和流向，留意图中第 2 和第 3 个开发板中 `∠a` 的大小都是 30°，而 `∠b` 一个小于 90°一个大于 90°，所以两个开发板上流水灯的速度相同、方向相反。在上一个测量倾角程序的基础上，根据 X 、Z 轴倾角调用流水灯函数，便可实现「重力感应流水灯」。

##### 代码实现

```c
// 【应用】重力感应流水灯\USER\src\main.c

/******************************************************************************
* 函数介绍： 主函数
* 输入参数： 无
* 输出参数： 无
* 返回值 ：  无
******************************************************************************/
int main()
{
    MPU6050_Angle data;

    uint8_t Run_Direc = DIR_LEFT;
    uint32_t Run_Speed = 0x2ffffff;
    
    LED_Init(); 
    MPU6050_Init();
    Usrat_1_Init(84,9600,0);
    
    while (1)
    {
        MPU6050_Get_Angle(&data);  // 计算三轴倾角
        
        Run_Speed = 0x1000000 - 0x582D8 * data.Z_Angle;  // 计算流水灯延时速度（参数自调）
        
        if (data.Z_Angle > 10)  // 当 Z 轴倾斜大于 10° 时开启流水灯
        {
            Run_Direc = (data.X_Angle < 90) ? DIR_LEFT : DIR_RIGHT;  // 通过 X 轴倾角判断流动方向
            LED_flow(Run_Direc);  // 流水灯
            Delay(Run_Speed);     // 延时
        }
    }
}
```

#### 实验效果

![图14 重力感应流水灯](http://odaps2f9v.bkt.clouddn.com/video2gif_20171023_210654.gif)

---

### 更多应用可能 

经历了三个实验之后，文章也接近尾声，但强大的 MPU6050 绝不仅仅能用来控制流水灯、四轴、体感手柄……**由于 MPU6050 高精度的特性，在「云 + 端」的大数据物联网时代，利用边缘的传感器采集海量数据，配合红到发紫的人工智能对数据进行分析、学习，更是具有无限可能。**

工程示例代码下载：[链接：http://pan.baidu.com/s/1boZEjJD 密码：i5us](http://pan.baidu.com/s/1boZEjJD)

<br/>
<br/>

> 参考文章
> 
> * Mini AHRS 姿态解算说明 - 第七实验室
> * [MPU-6000 and MPU-6050 Register Map and Descriptions Revision 4.2](https://www.invensense.com/wp-content/uploads/2015/02/MPU-6000-Register-Map1.pdf)
> * [MPU6050 数据轻松分析](https://wenku.baidu.com/view/14ca01925fbfc77da269b17f.html)