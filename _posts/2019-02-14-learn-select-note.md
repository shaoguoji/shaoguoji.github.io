---
layout:          post
title:           select() 学习笔记
subtitle:        初步认知、例程及原理简析
date:            2019-02-14 15:39:39 +0800
author:          Shao Guoji
header-img:      
catalog:         true
tag:
    - 学习笔记
    - C语言
    - 操作系统
---

## 初步认知

* `select()` 能实现非阻塞数据读写，避免阻塞线程以及维护多线程带来的额外消耗。
* `select()` 是一个系统调用（system call），由具体操作系统实现，不难理解，因为其行为涉及到 IO、文件系统等操作，而这些服务都是操作系统提供的。
* `select()` 函数监视多个 fd（file descrīptor 文件描述符）的状态变化，这些 fd 可分为三大类：readset（读 fd 集合）、writeset（写 fd 集合）和 exceptset（错误异常集合）。
* `select()` 常用于 socket 网络编程， 但理论上适用于能用 fd 描述的一切 IO，如 stdio、pipes，甚至是 uart。

## 函数原型

```c
int select(int maxfdp1, fd_set *readset, fd_set *writeset, fd_set *exceptset,
           const struct timeval *timeout);

/* Returns: positive count of ready descriptors, 0 on timeout, –1 on error */
```

### 参数说明

|   参数  |               说明                      |
|---------|----------------------------------------|
| maxfdp1 | 最大 fd 数值 +1，大于此数值的 fd 会被忽略 |
| readset | 读变化监视 fd 集合，可为 NULL            |
| writeset| 写变化监视 fd 集合，可为 NULL            |
| exceptset| 错误异常变化监视 fd 集合，可为 NULL      |
| timeout | 超时时间，NULL：阻塞，0：非阻塞，>0：等待  |

## fd 相关操作

伴随 select() 高频出现的几个 fd 操作：`FD_ZERO`、`FD_SET`、`FD_ISSET`、`FD_CLR`。

|   操作  |              用法        |          说明              |
|---------|-------------------------|----------------------------|
| FD_ZERO | FD_ZERO(fd_set*);       | 清空 fd_set 集合            |
| FD_SET  | FD_SET(int ,fd_set *);  | 将指定 fd 加入集合           |
| FD_ISSET| FD_ISSET(int ,fd_set*); | 检查 fd 状态，返回非零表示 ok |
| FD_CLR  | FD_CLR(int ,fd_set*);   | 从集合移除指定 fd            |

## socket 例程（client）

```c
#include <stdio.h>
#include <sys/time.h>
#include <sys/select.h>
#include <sys/socket.h>

#define BUFSZ   1024

static char url[] = "192.168.0.148";
static int port = 8080;

int main(void)
{
    int ret;
    char recv_data[BUFSZ];
    int bytes_received;
    int sock = -1;
    struct sockaddr_in server_addr;

    struct timeval timeout;
    fd_set readset;

    /* 创建一个socket，类型是SOCKET_STREAM，TCP类型 */
    if ((sock = socket(AF_INET, SOCK_STREAM, 0)) == -1)
    {
        /* 创建socket失败 */
        printf("Create socket error");
        return;
    }

    /* 初始化预连接的服务端地址 */
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(port);
    server_addr.sin_addr.s_addr = inet_addr(url);

    /* 连接到服务端 */
    if (connect(sock, (struct sockaddr *)&server_addr, sizeof(struct sockaddr)) == -1)
    {
        /* 连接失败 */
        printf("Connect fail!");
        goto __exit;
    }

    /* 设置超时时间 */
    timeout.tv_sec = 3;
    timeout.tv_usec = 0;

    while (1)
    {
        FD_ZERO(&readset);
        FD_SET(sock, &readset);

        /* Wait for read */
        ret = select(sock + 1, &readset, RT_NULL, RT_NULL, &timeout);

        if (ret < 0)
        {
            printf("select error...\n");
            break;
        }
        else if (ret == 0)
        {
            printf("select timeout...\n");
            continue;
        }
        else if (ret > 0 && FD_ISSET(sock, &readset))
        {
            /* 从sock连接中接收最大BUFSZ - 1字节数据 */
            bytes_received = recv(sock, recv_data, BUFSZ - 1, 0);
            recv_data[bytes_received] = '\0';
            printf("Received data = %s\n", recv_data);
        }
    }

__exit:
    return 0;
}
```

## 关于 FD_ISSET

`select()` 函数只是说明了集合中存在 fd 状态变化，至于具体是哪个 fd 改变，从返回值是看不出来的，因此需要用 `FD_ISSET` 操作进一步确认，找到具体改变的 fd。然而上述例程的 `FD_ISSET` 可以去掉，因为集合里就一个 fd。

如果是多个 fd 的情况，需要先使用数组存储 fd，并在调用 `select()` 后遍历判断，因此你可能会看到像这样的代码：

```c
int fd_array[BACKLOG];
int fd_count;

select(...);

// check every fd in the set
for (i = 0; i < fd_count; i++) 
{
    if (FD_ISSET(fd_array[i], &fdsr)) 
    {
        // data processing...
    }
}
```

另外，对于 `FD_ISSET` 作用的解释，网上资料有 「检查 fd 是否准备好」、「检测 fd 状态是否变化」、「确认数据可读写」等几种说法，多少有点让人捉摸不透。令人惊喜的是，在 [Oracle Solaris 的一篇文档](https://docs.oracle.com/cd/E86824_01/html/E54766/fd-isset-3c.html)中，`FD_ISSET` 部分的相关说明为：

> FD_ISSET(fd, &fdset)
Returns a non-zero value if the bit for the file descriptor fd is set in the file descriptor set pointed to by fdset, and 0 otherwise.

由此不妨大胆猜测，有没可能 fdset 本质上就是一个 bitmap（状态位标志数组），其相关的操作只是来来去去的位运算？有意思。

## 如何发生？

闲逛了一下 Stack Overflow 后，自己对 select 以及 `FD_ISSET` 的运行机制有了初步的想法：个人猜测，select 的实现中维护了一个 bitmap，通过 FD_SET 关联 fd 与 fdset，建立 bitmap 到具体 fd 的映射关系，调用 `select()` 时，操作系统检查并返回 I/O 状态，同时反映到 bitmap 中，最后应用程序通过 FD_ISSET 从 bitmap 中获取结果。

### 想法验证

至于到底是不是这样，还是怎样，我决定斗胆阅读 os 内核源码，一探究竟。鉴于 Linux 过于庞杂（其实就是菜），从「小而美的物联网操作系统」RT-Thread 入手是个不错的选择，正好，发现在 [rt-thread/components/dfs/src/select.c](https://github.com/RT-Thread/rt-thread/blob/master/components/dfs/src/select.c) 下有 `select()` 函数的实现。

可以看到，几个 fd 操作其实是 set bit 和 clear bit 的宏定义，fdset 充当了前面所说的 bitmap：

```c
#define FD_SET(n, p)  ((p)->fd_bits[(n)/8] |=  (1 << ((n) & 7)))
#define FD_CLR(n, p)  ((p)->fd_bits[(n)/8] &= ~(1 << ((n) & 7)))
#define FD_ISSET(n,p) ((p)->fd_bits[(n)/8] &   (1 << ((n) & 7)))
#define FD_ZERO(p)    memset((void*)(p),0,sizeof(*(p)))
```

rtt 的 `select()` 是通过 `poll()` 实现的 —— 先从 0 到 fd + 1 遍历 fdset，计算总的 fd 数量，再为每个 fd 分配 poll 资源、注册到 poll 机制，接着清空 fdset、调用 `poll()`，保存结果回 fdset 中，最后返回变化的 fd 计数。

```c
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout)
{
    int fd;
    int npfds;
    int msec;
    int ndx;
    int ret;
    struct pollfd *pollset = RT_NULL;

    /* How many pollfd structures do we need to allocate? */
    for (fd = 0, npfds = 0; fd < nfds; fd++)
    {
        /* Check if any monitor operation is requested on this fd */
        if ((readfds   && FD_ISSET(fd, readfds))  ||
            (writefds  && FD_ISSET(fd, writefds)) ||
            (exceptfds && FD_ISSET(fd, exceptfds)))
        {
            npfds++;
        }
    }

    /* Allocate the descriptor list for poll() */
    if (npfds > 0)
    {
        pollset = (struct pollfd *)rt_calloc(npfds, sizeof(struct pollfd));
        if (!pollset)
        {
            return -1;
        }
    }

    /* Initialize the descriptor list for poll() */
    for (fd = 0, ndx = 0; fd < nfds; fd++)
    {
        int incr = 0;

        /* The readfs set holds the set of FDs that the caller can be assured
         * of reading from without blocking.  Note that POLLHUP is included as
         * a read-able condition.  POLLHUP will be reported at the end-of-file
         * or when a connection is lost.  In either case, the read() can then
         * be performed without blocking.
         */

        if (readfds && FD_ISSET(fd, readfds))
        {
            pollset[ndx].fd         = fd;
            pollset[ndx].events |= POLLIN;
            incr = 1;
        }

        if (writefds && FD_ISSET(fd, writefds))
        {
            pollset[ndx].fd      = fd;
            pollset[ndx].events |= POLLOUT;
            incr = 1;
        }

        if (exceptfds && FD_ISSET(fd, exceptfds))
        {
            pollset[ndx].fd = fd;
            incr = 1;
        }

        ndx += incr;
    }

    RT_ASSERT(ndx == npfds);

    /* Convert the timeout to milliseconds */
    if (timeout)
    {
        msec = timeout->tv_sec * 1000 + timeout->tv_usec / 1000;
    }
    else
    {
        msec = -1;
    }

    /* Then let poll do all of the real work. */

    ret = poll(pollset, npfds, msec);

    /* Now set up the return values */
    if (readfds)
    {
        fdszero(readfds, nfds);
    }

    if (writefds)
    {
        fdszero(writefds, nfds);
    }

    if (exceptfds)
    {
        fdszero(exceptfds, nfds);
    }

    /* Convert the poll descriptor list back into selects 3 bitsets */

    if (ret > 0)
    {
        ret = 0;
        for (ndx = 0; ndx < npfds; ndx++)
        {
            /* Check for read conditions.  Note that POLLHUP is included as a
             * read condition.  POLLHUP will be reported when no more data will
             * be available (such as when a connection is lost).  In either
             * case, the read() can then be performed without blocking.
             */

            if (readfds)
            {
                if (pollset[ndx].revents & (POLLIN | POLLHUP))
                {
                    FD_SET(pollset[ndx].fd, readfds);
                    ret++;
                }
            }

            /* Check for write conditions */
            if (writefds)
            {
                if (pollset[ndx].revents & POLLOUT)
                {
                    FD_SET(pollset[ndx].fd, writefds);
                    ret++;
                }
            }

            /* Check for exceptions */
            if (exceptfds)
            {
                if (pollset[ndx].revents & POLLERR)
                {
                    FD_SET(pollset[ndx].fd, exceptfds);
                    ret++;
                }
            }
        }
    }

    if (pollset) rt_free(pollset);

    return ret;
}
```

## 参考资料

* [select(2): synchronous I/O multiplexing - Linux man page](https://linux.die.net/man/2/select)
* [c - Select function in socket programming - Stack Overflow](https://stackoverflow.com/questions/4171270/select-function-in-socket-programming)
* [Chapter 6. I/O Multiplexing: The select and poll Functions - Shichao's Notes](https://notes.shichao.io/unp/ch6/)
* [FD_ISSET - man pages section 3: Basic Library Functions](https://docs.oracle.com/cd/E86824_01/html/E54766/fd-isset-3c.html)
* [细谈select函数（C语言） - piaojun_pj的专栏 - CSDN博客](https://blog.csdn.net/piaojun_pj/article/details/5991968)
* [Linux中对文件描述符的操作(FD_ZERO、FD_SET、FD_CLR、FD_ISSET - ustbgaofan的个人空间 - 开源中国](https://my.oschina.net/floristgao/blog/311612)
* [深入select多路复用内核源码加驱动实现 - 黑客画家的个人空间 - 开源中国](https://my.oschina.net/fileoptions/blog/911091)
* [linux socket的select函数例子 - faraway - 博客园](https://www.cnblogs.com/faraway/archive/2009/03/06/1404449.html)
* [多线程非阻塞网络编程 - RT-Thread 文档中心](https://www.rt-thread.org/document/site/application-note/components/network/an0019-tcpclient-socket/)
* [[PDF]Select：非阻塞Socket 编程 - RT-Thread](https://www.rt-thread.org/document/site/tutorial/qemu-network/tcpclient_select/tcpclient_select.pdf)
