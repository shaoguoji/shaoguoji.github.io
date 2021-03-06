---
layout:     post
title:      I2C 总线协议初探
subtitle:	STM32 I2C 接口外设学习笔记
date:       2017-04-25 10:09:43 +0800
author:     Shao Guoji
header-img: img/post-bg-i2c-bus.jpg
catalog:    true
tag:
    - 单片机
    - STM32
    - 嵌入式
---

> I2C（Inter－Integrated Circuit）总线是由 PHILIPS（飞利浦） 公司开发的两线式串行总线，用于连接微控制器及其外围设备。是微电子通信控制领域广泛采用的一种总线标准。它是同步通信的一种特殊形式，具有接口线少，控制方式简单，器件封装形式小，通信速率较高等优点。

### 前言

早在大一和班上大神谈理想的时候就听过 I2C，那时我还在用 51 点灯，而他调了一星期 I2C，还把 I2C 念成 「I帮C」，我愣是没听懂。现在终于也到了我调 I2C 的时候，还是用着一款新接触的单片机 —— STM32。这几天看了一下 [I2C 协议手册](http://files.chinaaet.com/files/group/2011/07/24/9007438299002.pdf)，经历了从懵逼到略懂的过程。那么，I2C 到底是个什么东西？

---

<br/>

### 见名知意

好的名字对于一个概念的理解起到了良好辅助作用，使人在脑海中形成直观的第一印象，而 I2C 这个名字却让人莫名其妙。I2C 全名 `Inter-Integrated Circuit ` ，由集成电路的英文  `integrated circuit` （即我们常说的 IC）加上 `Inter-` （互联/相互）前缀构成，直译过来就是「集成电路间」、「集成电路际」或者「互联集成电路」，当然我们不会这么说，它其实是I²C Bus简称，所以中文应该叫集成电路总线（事实上它念 「I方C」）。由此可以了解到 I2C 所连接对象是 IC 集成电路，它实现的既不是两台终端设备之间的通信，也不是两个进程之间的通信，而是**一块电路板上两片 IC 之间的通信**。

更进一步讲，I2C 是一种使用多主从架构的串行通信总线，作为串行通信协议，I2C 只需要两根线 —— 数据线（SDA）和时钟线（SCL）即可工作，众多主机、从机就是挂载在这两根线上，通过总线上的高低起伏的电平变化进行寻址、握手、仲裁和数据传输等功能。麻雀虽小五脏俱全 —— 仅仅两根线就能玩出这么多花样，不得不佩服设计协议的工程师们。

除此之外，I2C 总线还有三种数据传输速率可供选择，分别是标准模式（可达 100 kbit/s）、快速模式（可达 400 kbit/s）和高速模式（可达 3.4 Mbit/s）。

![i2c总线示意图](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/20349832-file_1493112074747_cb20.png)

---

<br/>

### 开漏输出与上拉电阻

关于总线连接的物理特性，官方文档中还有这样的介绍：

> SDA 和 SCL 都是双向线路都通过一个电流源或上拉电阻连接到正的电源电压。当总线空闲时，这两条线路都是高电平 连接到总线的器件输出级必须是漏极开路或集电极开路才能执行线与的功能 。

注意到这样一句话：**「连接到总线的器件输出级必须是漏极开路或集电极开路才能执行线与的功能 」**，这里就涉及到「漏极/集电极开路」、「上拉电阻」和「线与」两个概念，**而这绝对可以算得上是实现 I2C 总线协议的关键所在**。

在芯片中，当一个输出级为漏极/集电极开路时（开漏输出），它只能输出**低电平和高阻态**，低电平我们了解，那「高阻态」又是个什么东西？**高阻态可理解为通过很大的电阻把输出引脚与 MCU 芯片内部隔开，近似开路的状态（电阻非常大）**。这时芯片无法控制输出的电平，引脚的电平不确定，可被外部电平轻松改变。开漏输出可以简单理解为输出处接一个开关，闭合时接低电平，断开时悬空（啥也不接）。下图的中间部分电路就很好地说明这种状态。

![开漏示意图](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/10651023-file_1493176956826_74ad.jpg)

资料来源：[单片机I/O口推挽输出与开漏输出的区别（转） - xiaoweiboy的专栏
        - 博客频道 - CSDN.NET](http://blog.csdn.net/xiaoweiboy/article/details/6714199)

正是由于开漏输出的「要么拉低要么放手」的特性，使得总线只受输出端低电平的影响（同样，设备也只能通过输出低电平来使用总线），从而实现了「线与」的功能。和「逻辑与」一样，「线与」所表达的意思是 —— 当总线上只要有一个设备输出低电平，整条总线便处于低电平状态，这时候的总线被称为占用状态。

然而 I2C 总线电路的真正主角，是连接总线到 VCC 的两个上拉电阻。根据前面的分析我们知道，输出端只能输出低电平或高阻态，是不能把总线拉高的，自然而然就需要通过其他方式为总线提供高电平，上拉电阻就担负了这个重任 —— **当输出端输出高阻态时且没有其他设备拉低总线（占用总线）时，总线被外部的上拉电阻拉高、呈现出高电平状态**，这不正是我们的「线与」功能么。

不仅如此，利用「线与」特性还可以实现总线的仲裁、时钟的同步，更牛逼的是，在整个过程中数据完全不会丢失。我甚至觉得 I2C 协议相对于其他串行通信协议最大的优势就是**通过「开漏输出」和「上拉电阻」两个物理特性大大简化了协议整体的设计和实现**。

![开漏输出上拉电阻](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/50234746-file_1493179318163_3d74.png)

#### 上拉电阻阻值对 I2C 协议的影响

需要补充说明的一点是，两个上拉电阻的大小并不是随便用的，涉及到**通信速率与功耗的取舍**。协议层对电平的变化时间有着严格的要求与限制，而电平的变化受总线电气特性的影响。

**对总线而言，上拉电阻越大，信号的上升时间就越长，通信速率就越低，反之亦然**。但电阻也并不是越小越好，阻值过小的话，总线低电平时电阻上的大电流会增加电路的功耗。此外，电容也会影响信号的上升时间，于是就有了 I2C 总线总电容 400 pf 的限制，这直接关系到总线上可挂载设备的数目。

---

<br/>

### I2C 总线基本术语

|术语       |     描述                                            |
|-----------|---------------------------------------------------- |
| 发送器    | 发送数据到总线的器件                                |
| 接收器    | 从总线接收数据的器件                                |
| 主机      | 初始化发送、产生时钟信号和终止发送的器件            |
| 从机      | 被主机寻址的器件                                    |
| 多主机    | 同时有多于一个主机尝试控制总线，但不破坏报文        |
| 仲裁      | 是一个在有多个主机同时尝试控制总线，但只允许其中一个控制总线并使报文不被破坏的过程|
| 同步      | 两个或多个器件同步时钟信号的过程                    |

主机和从机都可作为发送器和接收器，于是每个 I2C 设备可有下面四种工作模式：

* 主发送器
* 主接收器
* 从发送器
* 从接收器

---

<br/>

### 通信流程

和串口一样，I2C 的最终目的还是为了传数据（通信），**每种通信协议都有自己传数据的规矩，这些规矩包括了「先发什么后发什么」、「什么时候发」等细节，通信双方只要严格按照约定好的规矩来，就能实现数据的收发**。

值得注意的是，I2C 通信双方是分为主机与从机的，两者是一种**「不平等」的从属关系**，主机是老大，地位比从机高。一次数据的传输，无论传输方向如何，都是由主机发起和结束的，**主机有着本次传输的绝对控制权**，不像串口那样双方能够对等传输。

#### 完整的数据传输

以「主发送器」工作方式为例，在开始数据传输前，主机会发送一个起始信号，紧接着发送目标从机地址，从机被寻址后产生响应信号，接着主机开始发送数据，从机接收数据并产生响应信号，在数据传输完毕时主机发送结束信号，通讯结束。这就是 I2C 通讯的大致流程。

![I2C总线的数据传输](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/14788623-file_1493265590205_16052.png)

---

<br/>

### 起始、结束信号

**I2C 协议通过 SCL 与 SDA 两根线上的电平状态组合来定义不同的传输信号**。SCL 为高电平时 SDA 由高到低变化表示起始信号（START condition），SCL 为高电平时 SDA 由低到高变化表示结束信号（STOP condition）。给我的感觉像是「跳进坑里，通信开始」、「跳出坑外，通信结束」。**每次数据传输都是始于起始信号止于结束信号，期间总线处于占用状态**。

![起始条件和停止条件](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/775441-file_1493265371546_8149.png)

#### 重复起始信号

在通信过程中若要改变目的从机或者数据传输方向，可以不发送结束信号直接再发送一次起始信号，紧接着发送新的从机地址或传输方向，这个起始信号称为「重复起始信号（Sr）」，本质上和普通起始信号相同，但避免了结束、起始信号反复多次发送。**在多主机的总线中使用重复起始信号可保持主机对总线的控制权，避免结束信号临时释放总线造成控制权丢失。**

---

<br/>

### 从机地址、数据传输方向

在发送完起始信号后，主机紧接着需要发送一个字节（10 地址模式下两个字节）的地址信息，包括 **7 位的从机地址以及一位传输方向控制位**，「0」表示发送数据（写），「1」表示请求数据（读）。

![起始条件后的第一个字节](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/1784716-file_1493279021568_12ecb.png)

---

<br/>

### 数据有效性

I2C 协议规定了 SDA 线上的数据只在 SCL 为高电平时有效，在 SCL 为低电平时可进行数据的切换，即设备只在 SCL 线为高电平时才会对 SDA 上的信号进行采样。这里指的数据包括传输数据、从机地址等二进制数据。

![数据有效性](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/24340967-file_1493279590847_962.png)

根据数据传输的特点，可以通过改变 SCL 时钟信号的占空比灵活调整数据的采集和切换时间。

---

<br/>

### 响应信号

在从机被寻址或者主机/从机接收一字节数据后，都要在地址或数据后产生一个响应信号。响应信号包括「应答(ACK)」和「非应答(NACK)」两种信号。**所谓的作出应答就是在 SCL 线电平升高前设备将 SDA 线拉低，而产生非应答信号，设备只要在这时什么也不干，释放总线即可**。所以在使用 STM32 I2C 接口时不一定要等到数据接收完成后才使能非应答信号，实际上在前一次应答信号发出后就可以干这事了。

![响应信号](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/38299779-file_1493281376340_d196.png)

---

<br/>

### I2C 基本读写过程

在用单片机作为主机进行 I2C 通信控制外设时，根据数据传输方向可将 I2C 数据传输模式分为三类：

#### 主机发送模式

在主机发送模式下，同一次通信过程中数据传输只由主机到从机，报文格式如下图：

![主机发送模式](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/17798883-file_1493282144859_114d9.png)

在产生起始信号后，主机发送包含 7 位地址以及「写」控制位的一个字节数据，随后开始逐字节传输数据，并接收从机对各字节的响应信号。当从机响应非应答信号时，主机产生结束信号终止通信。当然，因为主机是「大佬」，也可以随时产生结束信号终止通信。

#### 主机接收模式

在主机接收模式下，同一次通信过程中数据传输只由从机到主机，报文格式如下图：

![主机接收模式](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/70446819-file_1493283123821_341.png)

在产生起始信号后，主机发送包含 7 位地址以及「读」控制位的一个字节数据，随后开始逐字节接收从机数据，并产生对各字节的响应信号。数据接收完毕时，主机产生非应答响应信号时，随后产生结束信号终止通信。要注意**主机在发送结束信号前先要产生非应答响应信号**。

#### 复合模式

前面两种模式中的数据传输方向都是固定的 —— 只由主机到从机或只由从机到主机，而在复合模式下，主机可以**通过重新发送起始信号改变数据传输方向，即同一次通信过程中存在两个数据传输方向**，报文格式如下图：

![复合模式](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/86682333-file_1493284881681_4d30.png)

复合模式结合了前两种模式的特点，**主机通过发送重复起始信号改变数据传输方向，实现读/写切换**。同样的，接收数据的主机要想改变方向或者终止通信，在发送重复起始信号和结束信号前都要**先产生非应答响应信号**。

复合模式可以实现主机数据的「先发送后接收」，最典型的应用是通过 I2C 总线控制 EEPROM 读数据 —— 主机（MCU）先发送要读取数据在 EEPROM 内的地址，随后切换模式接收 EEPROM 返回的数据。

---

<br/>

### STM32 中使用 I2C

了解 I2C 总线协议的基本规定后，可以发现**所谓的 XXX 协议，不过就是根据一个标准进行总线上高低电平的变化而已**。这种电平变化无论是用单片机C语言软件实现、还是众多 IC 固化的硬件接口实现，甚至把两条总线通过拨码开关接到 VCC/GND 进行纯手动操作（**你可以想象在快速模式 I2C 下，极客的手指以 400 Hz 的频率在「抽搐」，只为了实现真正的「人机交互」**），理论上只要能产生符合协议规范的「特征电平」信号，两个 I2C 设备就能通信，于是就有了我们常说的「一个协议的两种实现方式」 —— 「软件模拟」和「硬件实现」。

#### 软件模拟

软件模拟是通过编程语言（C 啊汇编什么的）控制单片机 I/O 口的电平来产生通信协议的各种信号。因为每一个电平变化都是码农一下下敲出来的（类似于「LED = 0;」这样的代码），给人一种「一切尽在掌握之中」的感觉，软件模拟方式会让人有一种久违的「安全感」，具体表现为可控性高、灵活性强的优点，但同时也要付出代价：**很显然，MCU 监控或查询总线的次数越多，用于执行自己功能的时间越少**。

#### 硬件实现

随着时代的发展，单片机上的外设越来越多、越来越强大，与串口、定时器、I/O口一样，I2C 也可以固化为 MCU 外设，**协议各种信号的发送/接收都可以由硬件自动完成，并通过众多寄存器给 MCU 开发者提供使用接口，码农们只需要「面向寄存器编程」**。

使用 STM32 的 I2C 接口外设时，要发送就往外设的数据寄存器扔数据，要接收就从外设的数据寄存器读数据，各种控制信号的产生也可以通过写相应的外设控制寄存器位完成。或许你觉得就这样「放手硬件」有点不放心，万一被硬件坑了怎么办？别怕，STM32 还有一堆的状态寄存器供你判断外设工作状态。另外，硬件还为通信过程中的各种情况实现了中断，这些都是用软件模拟法所没有的福利。

![STM32 I2C框图](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/68704551-file_1493288423684_e086.png)

*只是随便写点 I2C，STM32 部分不具体描述，详见参考手册。*

---

<br/>
<br/>

> 参考文章
> 
> * [I2C 总线协议（中文版）](http://files.chinaaet.com/files/group/2011/07/24/9007438299002.pdf)
> * [零死角玩转STM32-F429挑战者](https://pan.baidu.com/s/1dEIuO97#list/path=%2F0-%E7%A7%89%E7%81%AB%E3%80%90F429%E5%BC%80%E5%8F%91%E6%9D%BF-%E6%8C%91%E6%88%98%E8%80%85%E3%80%91%E5%85%89%E7%9B%98%E8%B5%84%E6%96%99%2FA%E7%9B%98%EF%BC%88%E8%B5%84%E6%96%99%E7%9B%98%EF%BC%89)
> * [STM32F4XX中文参考手册](https://pan.baidu.com/share/link?shareid=993973964&uk=839310838&fid=584381698206148)
> * [Understanding the I2C Bus](http://www.ti.com/lit/an/slva704/slva704.pdf)








