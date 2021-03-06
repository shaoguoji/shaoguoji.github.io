---
layout:     post
title:      Linux 软链接与硬链接
subtitle:	文件访问，能软能硬？
date:       2018-01-28
author:     Shao Guoji
header-img: img/post-bg-hard-soft-link.jpg
catalog:    true
tag:
    - Linux
    - 操作系统
---

### 说在前面

最近从常用命令开始学 Linux，看到 ln 命令时突然蹦出硬链接和软连接这两个概念，在阅读了一些的书籍和资料后，决定写篇文章整理记录一下思路。hard link 和 soft link 作为两种不同的文件访问方式，提供了灵活的文件共享特性。在讨论两者的区别之前，了解一点关于文件的背景知识很有必要。

---

### 数据访问 —— 从文件名到硬盘

我们总是对硬盘如何正确地储取文件感到好奇， 它看起来似乎非常复杂抽象。当你随手按下 Ctrl + S 保存文件时，编辑器通过调用操作系统提供的接口对硬盘进行写操作，将珍贵的数据记录起来。想象一下如果没有操作系统和复杂的磁盘驱动程序，用户得拿着磁铁高速摩擦盘片进行数据读写，这般骚操作对非单身人士来讲一定是个噩梦。

#### 文件名与文件数据

操作系统让用户使用文件名（包括路径）就能访问数据，**文件名**和**文件数据**共同组成文件，文件名方便用户对文件的使用、为用户服务，文件数据则与操作系统相关。文件名和文件数据在 Linux 中是分开存储与管理的，并通过特殊的方式建立联系。为了更好理解**文件数据**，必须继续深入探讨。

#### 文件数据的组成

文件系统实现对数据的组织和存储，使得对其访问和查找变得容易。因此除了用户数据（文件内容本身的数据）外，操作系统还会记录文件的额外信息，如文件大小、所占磁盘块、文件类型等，这些数据统称为**元数据（Metadata）**。**文件数据实际上是由用户数据和元数据两部分组成。且用户数据不能直接访问，必须通过元数据访问。**

再把思路捋一捋：文件由文件名和文件数据构成，文件数据又可分为元数据和用户数据。在访问文件时分三步走：用户给出文件名，操作系统先用文件名找到元数据，再通过元数据顺藤摸瓜找到文件真实数据，如下图所示：

![图1 访问文件三部曲](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/77900037.jpg)

#### 元数据

**元数据的存储方式与具体文件系统相关**，在目前主流 Linux 发行版采用的 ext2 系列（ext2、ext3、ext4）文件系统中，元数据通过索引节点 inode 进行保存。Linux 同时在虚拟文件系统（VFS）层也实现了抽象的 inode 结构，以适配不同的具体文件系统，下文的 inode 都是指后者。

关于 inode， 有几个需要注意的特性（**划重点！！！**）：

1. inode 随着文件的存在而存在，创建文件时系统会创建相应的 inode 
2. inode 都有一个编号，操作系统内部使用 inode 号来识别不同的文件。
3. inode 中保存文件数据块（data block）的指针，且数据块与 inode 一一对应，因此 inode 号才是文件的唯一标识而非文件名

在命令行 `ls` 查看文件时可使用 `-i` 参数显示文件 inode 号，不同文件有不同的 inode 号：

![图2 显示文件 inode 号](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/48280187.jpg)

#### 文件名与 inode

通过 inode 访问文件是操作系统内部的方式，让用户记住一串毫无意义的 inode 号来使用文件显然是不合理的，使用文件名可以很好解决这个问题。文件名本质上是 inode 号的别名，OS 最终还得通过 inode 定位文件数据块，那如何把文件名与 inode 对应起来呢？

前面提到过，在 Linux 中文件名和文件数据（inode 节点）是分开存储与管理的，并通过特殊的方式建立联系，这种特殊的方式叫做目录项（directory entry 或 dentry）。目录文件（Linux 一切皆文件，目录也不例外）通过若干目录项记录着该目录下所有文件的信息。**每个目录项由两部分组成：所包含文件的文件名，以及该文件名对应的 inode 号。**目录项建立文件名与 inode 对应关系，从而实现「文件名 --> inode --> 数据块」的访问流程：

![图3 文件名与 inode](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/2778079.jpg)

---

### 硬链接

所谓硬链接，就是上图第一个箭头，即**文件名到 inode 的链接（目录项）**。创建文件时。系统将会分配数据块、创建 inode、添加 dentry 条目到所属目录文件，「文件三件套」一样不少。显然，**Linux 下所有文件默认都会有一个硬链接，用来生成文件名（文件至少要有一个硬链接才能访问其内容）：**

![图4 所有文件默认1个硬链接](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/17949369.jpg)

#### 硬链接的创建

使用 `ln` 或 `ln -P` 命令创建一个硬链接，格式 `ln [-P] 源文件 目标文件`。下面为 `hello.c` 文件创建一个硬链接 `hello-hd.c`：

![图5 ln 命令创建硬链接](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/51267528.jpg)

可以看到两个文件的文件的类型、大小一致，重点在于 inode 号是相同的 —— 说明 **`hello.c` 和 `hello-hd.c` 其实是同一个文件**，这两个文件名共用一块磁盘空间，系统无法区分两者的差别。列表第三列是文件链接数（引用数），创建链接后两个文件的链接数变为 2，表示可以通过 2 个名字都访问到同一个 inode 下相同的数据块。**当我们手动创建硬链接时，相当于额外添加了一个目录项指向已存在的 inode（下图黄箭头），但文件内容只有一份**：

![图6 添加额外目录项](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/85018393.jpg)

创建硬链接等价于给文件取别名，例子中给 `hello.c` 起了一个别名叫 `hello-hd.c`，这意味着可以用不同的文件名访问同样的内容，就像同一个人有不同的绰号，虽然我被人喊过「邵国际」、「国际」、「邵哥」、「国哥」、「国际哥」，但是我只有一副躯体。

#### 硬链接的特点

由于**硬链接是同一文件的不同文件名**，因此文件有相同的 inode 及 data block，对文件内容进行修改会影响到所有文件名，但是删除一个文件名（只要不是最后一个）不影响其他文件名对数据的访问。

硬链接的核心思想在于**通过文件名直接确定唯一的内容数据**，要实现这一点必须依赖具体文件系统，于是就不难理解硬链接的使用限制：

1. 硬链接不可跨不同文件系统使用
2. 硬链接只允许链接到**已存在**的文件，而不能是目录
3. 文件至少要有一个硬链接（*同一内容的所有硬链接被删除时，文件无法再被使用，系统回收相应 inode 与数据块，这时文件真正被删除*）

---

### 软连接

细心的读者会发现，刚刚我们谈论硬链接时始终都在围绕着同一个文件（由始至终只出现了一份 inode 和数据块）。下面要讨论的软链接将涉及两个不同的文件。

#### Linux 中的「快捷方式」

软链接（soft link）又称符号链接（symbolic link 或 symlink），*软链接与硬链接不同，若文件用户的数据块中存放的内容是另一文件的路径名的文本指向，则该文件就是软连接*。**软链接就是一个普通文件，只是数据块内容有点特殊。**事实上我更喜欢「符号链接」这个名字，因为它暗示了软链接作为一个文件，其内容只是一些符号而已。这些神秘的符号，就是目标文件的文件名（包括路径）。

如果说硬链接是「文件名到文件数据」，那么软连接就是「文件数据到文件名」，具体一点，是「一个文件数据到另一个文件名」，即存在两个不同的文件参与其中。**软链接有着自己的 inode 和数据块（见下图），文件的内容保存了目标文件的路径文件名，原理同 Windows 下的快捷方式**：

![图7 软链接示意图](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/30198626.jpg)

#### 软链接的创建

使用 `ln -s` 命令创建一个软链接，格式 `ln -s 源文件 目标文件`。下面为 `hello.c` 文件创建一个软链接 `hello-sym.c`：

![图8 ln 命令创建软链接](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/22490650.jpg)

`hello.c` 和 `hello-sym.c` 两个文件的类型、大小一目了然，inode 号不同说明这是两个不同的文件，系统能区分出两者的差异。其中 `hello-sym.c` 显示的文件类型为 `l` 前缀，是 Linux 中的链接文件，行末处显示软链接实际指向 `hello.c`。或许你还有一个疑惑：链接文件的大小为什么是 7 呢？哈哈前面走神了吧，因为文件名 `hello.c` 作为文件内容一共 7 个字符吖！

![图9 创建软链接示意图](https://raw.githubusercontent.com/shaoguoji/blogpic/master/post-img/85394048.jpg)

创建软链接并不会增加文件的链接数（上图命令行中所有文件链接数均为 1），软链接并不会直接访问 inode，仅仅指向目标数据的文件名。至于从目标文件名到目标数据的工作，还是硬链接那一套。

#### 软链接特点

由于只是简单保存目标文件的文件名，并不涉及具体的 inode 和数据块，这让软链接弥补了硬链接的不足：

1. 软连接可以跨不同文件系统使用
2. 允许链接到不存在的目标
3. 链接对象可为目录（虽然上面一直用文件举例）
4. 即使删除所有链接文件，对源文件也无影响。但如果删除被指向的原文件，则相关软连接被称为死链接（即 dangling link，若重新创建被指向路径文件，死链接可恢复为正常的软链接）。

---

### 软链接和硬链接的对比

作为文件的两种链接方式，无论是通过软链接还是硬链接都可以修改源文件数据，只不过硬链接和已有文件数据绑死，而软链接与具体文件解耦，是上层更灵活的实现，所以实际情况使用软链接更多。硬链接本身就是文件的一部分，当文件只剩一个 hard link 时，unlink 和 remove 就是一个意思了，而对软链接怎么删也不会影响源文件。

---

### 链接文件典型应用 —— 软件更新

链接文件使得更新软件变得简单。假设我们有一个 2.6 版本的游戏，文件名为 `贪玩蓝月-v2.6`，然后创建一个指向 `贪玩蓝月-v2.6` 的链接文件 `一刀满级`。这样当我们每次打开 `一刀满级` 文件就能运行 `贪玩蓝月-v2.6` 开始愉快的玩耍。如果使用软链接我们还可以查看到游戏的当前实际版本。

当游戏要升级为 2.7 版本的 `贪玩蓝月-v2.7` 时，升级程序只需把 `贪玩蓝月-v2.7` 下载到系统中，删除旧的链接文件，并创建新的 `一刀满级` 文件链接到 `贪玩蓝月-v2.7` 即可。即使游戏在运行中也毫无影响，下次打开便已经是「船新版本」。此时系统同时保留两个版本的游戏，如果发现新版本有 bug，用同样的方式可以很方便切换回 2.6 旧版本继续玩耍 :)

嗯，所以，要不然，先来把游戏再说？

> 参考资料
> 
> * [The Linux Command Line](http://www.linuxzasve.com/preuzimanje/TLCL-09.12.pdf)
> * [理解 Linux 的硬链接与软链接](https://www.ibm.com/developerworks/cn/linux/l-cn-hardandsymb-links/index.html)
> * [理解 inode - 阮一峰的网络日志](http://www.ruanyifeng.com/blog/2011/12/inode.html)
> * [linux - Where does metadata go when you save a file? - Super User](https://superuser.com/questions/1160927/where-does-metadata-go-when-you-save-a-file/1161036)
> * [Linux文件管理 - Vamei - 博客园](http://www.cnblogs.com/vamei/archive/2012/09/09/2676792.html)
> * [Linux File System 文件系统 - Freewheel的个人空间](https://my.oschina.net/Bruce370/blog/886657)
> * [unix - What is the difference between a symbolic link and a hard link? - Stack Overflow](https://stackoverflow.com/questions/185899/what-is-the-difference-between-a-symbolic-link-and-a-hard-link)