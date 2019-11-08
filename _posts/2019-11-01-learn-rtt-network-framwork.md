---
layout:          post
title:           条条大路通罗马，个个 Socket 通网卡
subtitle:        RT-Thread 网络框架及相关组件学习笔记
date:            2019-11-01 18:06:29 +0800
author:          Shao Guoji
header-img:      img/post-bg-network-socket.jpg
catalog:         true
tag:
    - 学习笔记
    - 嵌入式
    - RT-Thread
---

*写这篇文章的缘由是升级项目中 LwIP 版本引发的一些感受和理解 ，因此思考角度会从协议栈出发向上向下延伸代入，偏向框架整体与部分的关联，要提的一点是，既然是个人理解，加上本人能力知识有限，难免会存在不严谨甚至错误的地方，请读者批判看待并欢迎指正。*

### 网络系统分层思想

得益于 uIP、LwIP 等轻量级网络协议栈的广泛使用，嵌入式设备在拥有完整 TCP/IP 功能的同时，又保证了对处理器资源的有限消耗，正确地使用网络协议栈传输数据，是联网设备的开发中不可缺少的关键一环。同时，由于嵌入式软件碎片化、离散化的特点，传统的直接基于协议栈对接项目的方法，在面对应用更新、硬件更换、需求迭代时，代码往往「牵一发动全身」。

带网络功能的嵌入式系统（或物联网系统），涉及应用（网络编程）、协议栈（数据处理）、底层驱动（网卡收发）等方面的开发，如何保证各个模块间的低耦合、灵活易用，成了摆在工程师面前的一道难题，简单点说，就是如何保证应用、协议栈、网卡的三者能独立更换，随意组合，并且付出最小的修改代价。

借助计算机科学中强大的分层思想，我把一个网络系统自顶向下分为了三层：应用编程、网络协议栈、网卡驱动。

![图1 网络系统分层](https://i.loli.net/2019/11/05/dUJnSrQ73AL4NIq.png)

*当然了概念上的分层都是相对的，存在一个逻辑边界，这里只是基于自己对各层级的理解，作为划分依据。*

#### 应用编程、协议栈与网卡驱动

首先请允许我简单粗暴地把所有基于协议栈应用接口开发的相关代码，都归属于应用编程下，在带操作系统的情况下，通常协议栈都会提供套接字（socket）接口，完成应用程序与 TCP/IP 协议的交互，因此这里所指的应用编程，是采用 Socket 相关操作（如 connect()、send()、recv() 等）完成的网络功能开发的过程，需要依赖协议栈提供的 Socket 接口。

再说到协议栈这一层，LwIP、uIP、这些优秀的软件属于常见的网络协议栈，但协议栈的内涵远不止于此，通常对「协议」往往偏从通信的角度去解释，这里不妨从底层和应用层的关联去考虑这件事：网络协议栈是否能看做硬件和软件之间的「兼容层」？数据从网卡到 Socket 接口中间需要经过多次处理、转换，不同的「协议」完成单独一类数据处理任务，多个协议堆叠形成协议栈（stack）。

对于 LwIP、uIP 等实现了 TCP/IP 模型的协议栈，硬件的网络模块即便只传输赤裸裸的以太网帧，协议栈也能给你一层层处理，最终拿出传输层的 TCP 数据，协议栈完成网络数据封包和解包。而对于单片机开发中常用 AT 指令 Wi-Fi 模块，如果想使用 Socket 接口编写网络程序，也需要协议栈完成字符串到 Socket 的转换。

不管协议栈、上次应用如何封装，数据都是要从经过网卡发送出去（或接收回来），不同网卡设备接入方式、硬件总线和软件接口也各不相同，有以太网和 Wi-Fi的，有串口也有 SPI 和 SDIO 接口的，带 TCP/IP 协议栈有不带的……唯一的共同点就是都会提供数据收发的驱动程序，你甚至用淘宝卖家贴出的示例程序就能轻松联网，然而要实现文章一开头提到的应用、协议栈、网卡三者独立，并不是那么容易。

因此，为了能在应用层程序调用统一的套接字接口开发，对接新的网卡或协议栈时，就要完成两大工作：1、网卡和协议栈的接入，2、协议栈和 Socket 接口的适配。

*本文内容侧重解释网络系统各层级之前的关联与配置，如何对接接口，对于各部分内部细节，如网络编程方法、LwIP协议栈结构、网卡工作方式等具体知识，这里不再多做赘述*

### RT-Thread 网络框架及组件

当单片机网络程序运行在一个物联网操作系统下，应用的开发和维护难度便大大降低，网络管理是任何一个现代 OS 都会提供的服务，在 RTT 中提供了标准的 BSD Socket 接口，同时也完成上面所说的两大工作。

为了解决协议栈与标准 Socket 的对接，RT-Thread 提供了一套 SAL（套接字抽象层）组件，为了解决网卡管理与协议栈的对接，RT-Thread 抽象出 netdev（网络接口设备） 组件用于网卡配置和控制，在前面三层架构的基础上形成了新的网络框架：

![图2 RTT 网络框架](https://i.loli.net/2019/11/05/oEacwLFTz6JA43v.png)

其中对网卡和协议栈设备进行了细分，依据网卡数据格式分为以太网设备和 AT 设备，对应的协议栈包括 LwIP TCP/IP、AT Socket 等，AT Socket 能够完成 AT 命令解析与管理，向上提供编程接口，这两类协议栈类型具体用 AF_INET 和 AF_AT 表示，对了，协议栈的类型在 RTT 文档中被称为**协议簇类型（family）**。

*注意：中文语义里协议簇和协议栈的含义常常被混淆，按照 RTT 官网文档的语境，这里指的协议簇（family）为同类协议栈（stack）的总称*

### SAL：统一协议栈 API 接口

协议栈本身作为独立使用的程序，自身提供了一系列的编程接口，在没有操作系统的情况下，应用层直接调用协议栈自带 Socket 接口开发，直观表现为，使用 LwIP 的 APP 代码中满是 `lwip_socket()`，使用 AT Socket 的 APP 代码中满是 `at_socket()`，这样一来当更换协议栈时应用代码也要随之更改，影响范围较大。

由于不同协议栈的编程接口在功能上差异不大，基本上都包含了 `xxx_socket()`、`xxx_connect()`、`xxx_send()`、`xxx_recv()` 等常规网络操作，在 RTT 的 SAL 套接字抽象层中把这些接口函数功能抽象，提供统一的 `sal_socket()`、`sal_connect()` 等接口，将不同的协议栈做了一层适配转换，应用层只需基于这些接口开发，SAL 再自动匹配调用对应协议栈接口。

在使用 SAL 之前要注册不同协议栈的接口函数，但大部分轮子已经造好，不用开发者过于操心，尤其像 LwIP 这样常用的协议栈早已做好了适配，在 RTT 4.0.x 版本源码 `components/net/sal_socket/impl/af_inet_lwip.c` 中可以阅读到相关代码。

简单来说就是通过 `struct sal_socket_ops` 结构填充协议栈函数，再利用 `struct sal_proto_family` 附上协议簇类型打包，最终存储到 `netdev->sal_user_data` 中。而对于 AT Socket，其对接代码在同级目录的 `af_inet_at.c` 文件下。

相关结构体类型定义：

```c
struct sal_proto_family
{
    int family;                                  /* primary protocol families type */
    int sec_family;                              /* secondary protocol families type */
    const struct sal_socket_ops *skt_ops;        /* socket opreations */
    const struct sal_netdb_ops *netdb_ops;       /* network database opreations */
};
```

```c
/* network interface socket opreations */
struct sal_socket_ops
{
    int (*socket)     (int domain, int type, int protocol);
    int (*closesocket)(int s);
    int (*bind)       (int s, const struct sockaddr *name, socklen_t namelen);
    int (*listen)     (int s, int backlog);
    int (*connect)    (int s, const struct sockaddr *name, socklen_t namelen);
    int (*accept)     (int s, struct sockaddr *addr, socklen_t *addrlen);
    int (*sendto)     (int s, const void *data, size_t size, int flags, const struct sockaddr *to, socklen_t tolen);
    int (*recvfrom)   (int s, void *mem, size_t len, int flags, struct sockaddr *from, socklen_t *fromlen);
    int (*getsockopt) (int s, int level, int optname, void *optval, socklen_t *optlen);
    int (*setsockopt) (int s, int level, int optname, const void *optval, socklen_t optlen);
    int (*shutdown)   (int s, int how);
    int (*getpeername)(int s, struct sockaddr *name, socklen_t *namelen);
    int (*getsockname)(int s, struct sockaddr *name, socklen_t *namelen);
    int (*ioctlsocket)(int s, long cmd, void *arg);
#ifdef SAL_USING_POSIX
    int (*poll)       (struct dfs_fd *file, struct rt_pollreq *req);
#endif
};
```

LwIP 下可以接多种以太网网卡，AT Socket 下可以接多种 AT 通讯模块，而这两大协议栈又都能注册到 SAL 中统一管理，如此一来，通过调用 SAL 的抽象接口，即可轻松使用各式网络硬件模块开发网络功能，不必再依赖单一具体协议栈 API，。

#### BSD Socket：条条大路通罗马

即使有了 `sal_socket()` 接口，通常也并不会直接调用，因为这还是在依赖具体的组件（只不过从依赖协议栈变成了依赖 SAL），更好的做法是用户程序调用**标准 BSD Socket API** `socket()` 开发：

```c
int socket (int family, int type, int protocol);
```

RT-Thread 的 DFS（虚拟文件系统）组件实现了标准 BSD Socket 接口，以统一管理文件 fd 和 socket，配置宏定义 `SAL_USING_POSIX`（4.0.x 版本之前为 `RT_USING_DFS_NET`）启用 Posix API 后，用户调用 `socket()` 函数，将会进到 `components/net/sal_socket/socket/net_sockets.c` 中的定义，完成 fd 的统一管理，通过 `sal_socket()` 根据不同协议族类型调用具体协议栈接口。

举个栗子，以 LwIP 为例，假设应用程序 WebClient 中调用了 `socket()` 连接网络服务器，协议栈的实际调用路径为：

```
webclient_connect() -> socket() -> sal_socket() -> inet_socket() -> lwip_socket()
```

*有意思的是，在 SAL 推出之前，其前身 DFS_NET 中的 `socket()` 实现仍然是直接调用 `lwip_socket()`，对 lwIP 有一定的依赖，存在局限性，而 SAL 解决了这个问题。*

启用 BSD Socket 的方法是在 env 中输入 menuconfig，找到 SAL 组件，再选择使用 BSD Socket 接口，各选项顺序：`RT-Thread Components  ---> Network  ---> Socket abstraction layer  ---> Enable BSD socket operated by file system API`

![图3 env 开启 BSD Socket](https://i.loli.net/2019/11/07/CuaG3zfb5Yqkmxh.png)

那如果不使用 SAL 的标准 BSD Socket 接口实现呢，应用层的 `socket()` 该怎么走？别担心，正所谓条条大路通罗马，RTT 中也是「个个 Socket 通网卡」（挺押韵啊）。

为了厘清各组件之间的关系，要分多钟情况讨论，还是以 LwIP 协议栈和 WebClient 应用为例，分析 `socket()` 的调用路径：

**第一条路：只开启协议栈（#define RT_USING_LWIP）**

只开 LwIP 的情况下，没办法，应用层的 `socket()` 只能直接对应到协议栈接口 `lwip_socket()`，实际调用路径为：

```
webclient_connect() -> lwip_socket()
```

此时源码中的 `socket()` 只不过是 `lwip_socket()` 的别名，利用条件编译，包含不同的头文件实现宏替换：

```c
// rt-thread/packages/webclient-v2.1.0/src/webclient.c

/* support both enable and disable "SAL_USING_POSIX" */
#if defined(RT_USING_SAL)
#include <netdb.h>
#include <sys/socket.h>
#else
#include <lwip/netdb.h>
#include <lwip/sockets.h>
#endif /* RT_USING_SAL */

```

未开启 SAL 情况下，应用代码将包含 `lwip/sockets.h` 头文件，这里面把 `lwip_socket()` 用 `socket()` 包装：

```c
// rt-thread/components/net/lwip-2.1.0/src/include/lwip/sockets.h

#if LWIP_COMPAT_SOCKETS
...
#define socket(a,b,c)         lwip_socket(a,b,c)
...
#endif /* LWIP_COMPAT_SOCKETS */
```

`LWIP_COMPAT_SOCKETS` 宏在 `lwipopts.h` 内由 `SAL_USING_POSIX` 控制使能，表示套接字函数名称依照 BSD Socket 风格进行封装。

```c
// rt-thread/components/net/lwip-2.1.0/src/lwipopts.h

/*
 * LWIP_COMPAT_SOCKETS==1: Enable BSD-style sockets functions names.
 * (only used if you use sockets.c)
 */
#ifdef SAL_USING_POSIX
#define LWIP_COMPAT_SOCKETS             0
#else
#ifndef LWIP_COMPAT_SOCKETS
#define LWIP_COMPAT_SOCKETS             1
#endif
#endif

```

**第二条路：开启协议栈和 SAL（#define RT_USING_LWIP & #define RT_USING_SAL）**

在上一种情况的基础上打开 SAL 组件，应用层将调用 `sal_socket()`，把 Socket 操作托管给 SAL 完成，从应用层到协议栈的调用路径如下： 

```
webclient_connect() -> sal_socket() -> inet_socket() -> lwip_socket()
```

重复上面的分析，`RT_USING_SAL` 被定义，那么 `webclient.c` 首先会包含 `sys\socket.h`，头文件中根据是否使用 Posix API，对标准 `socket()` 函数进行声明或用 `sal_socket()` 包装。

```c
// rt-thread\components\net\sal_socket\include\socket\sys_socket\sys\socket.h

#ifdef SAL_USING_POSIX
...
int socket(int domain, int type, int protocol);
...
#else
...
#define socket(domain, type, protocol)                     sal_socket(domain, type, protocol)
...
#endif /* SAL_USING_POSIX */

```

**第三条路：开启协议栈、SAL 和 POSIX API（#define RT_USING_LWIP & #define RT_USING_SAL & #define SAL_USING_POSIX）**

前两个的基础上再开启 DFS Socket 标准接口，即本节开篇所说的情况，调用路径和单独使用 SAL 大部分重叠：

```
webclient_connect() -> socket() -> sal_socket() -> inet_socket() -> lwip_socket()
```

此时不对 `socket()` 做任何宏替换包装，直接采用 DFS 中的实现，再通过 `sal_socket` 关联具体协议栈操作。

**补充（老版本无 SAL 情况）：开启协议栈和 BSD POSIX API（#define RT_USING_LWIP & #define RT_USING_DFS_NET）**

前面有提到，在还没有 SAL （是没有而不是没开）的老版本 RTT 中，DFS `socket()` 实现直接调用 `lwip_socket()`，这里也补充一下对应的调用路径：

```
webclient_connect() -> socket() -> lwip_socket()
```

具体代码位于旧版本源码 `rt-thread\components\dfs\filesystems\net\net_sockets.c` 中，鉴于篇幅就不再贴出，可以看到在 `socket()` 中是直接调用了 `lwip_socket()` 的。

也就是说，4.0.x 版本后的 RTT 网络组件下，DFS `socket()` 实现只能走 `sal_socket()` 函数，标准 BSD Socket API 成了 SAL 的一部分，宏开关的名称也由 `RT_USING_DFS_NET` 改为 `SAL_USING_POSIX`，跨版本移植组件时要注意一下。


> 参考资料
> 
> * [RT-Thread发布SAL套接字抽象层，带来全新物联网软件开发模式 - 物联网 - 电子发烧友网](http://www.elecfans.com/iot/639193.html)
> * [SAL套接字抽象层 - RT-Thread 文档中心](https://www.rt-thread.org/document/site/programming-manual/sal/sal/)
> * [netdev网卡 - RT-Thread 文档中心](https://www.rt-thread.org/document/site/programming-manual/netdev/netdev/)[netdev网卡 - RT-Thread 文档中心](https://www.rt-thread.org/document/site/programming-manual/netdev/netdev/)