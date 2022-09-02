# Lab1 Booting a PC

## 概述

本次实验的主要内容是探究计算机的启动过程。计算机的启动过程是指从CPU加电到操作系统被加载到内存中计算机所进行的一系列操作。通俗地讲，就是从我们按下开机键到计算机被完全启动的这个过程。

## Part 1: PC Bootstrap

<br>

### The PC's Physical Address Space

<br>

> 注：这里我略过了*Getting Started with x86 assembly*和*Simulating the x86*这两部分内容。第一部分是学习x86汇编语法，需要读者自己学习。第二部分是使用qemu虚拟机模拟x86环境，这一部分我们已经在[环境准备](https://github.com/uncle-lv/MIT-6.828/blob/main/0.environ-preparation.md#%E9%AA%8C%E8%AF%81%E7%8E%AF%E5%A2%83)这一章节中学习过了，不再赘述。

<br>

通常来说，PC的物理地址空间分布如下图所示：
```
+------------------+  <- 0xFFFFFFFF (4GB)
|      32-bit      |
|  memory mapped   |
|     devices      |
|                  |
/\/\/\/\/\/\/\/\/\/\

/\/\/\/\/\/\/\/\/\/\
|                  |
|      Unused      |
|                  |
+------------------+  <- depends on amount of RAM
|                  |
|                  |
| Extended Memory  |
|                  |
|                  |
+------------------+  <- 0x00100000 (1MB)
|     BIOS ROM     |
+------------------+  <- 0x000F0000 (960KB)
|  16-bit devices, |
|  expansion ROMs  |
+------------------+  <- 0x000C0000 (768KB)
|   VGA Display    |
+------------------+  <- 0x000A0000 (640KB)
|                  |
|    Low Memory    |
|                  |
+------------------+  <- 0x00000000
```

基于Intel 8088 16位处理器的早期PC只能操作1M的物理内存。被标记为*Low Memory*的640K空间是早期PC唯一可以使用的RAM（random-access memory）。

从0x000A0000到0x000FFFFF的384KB区域是预留给硬件设备的，用做视频显示缓冲区或者保存固件等特殊用途。预留区域中最重要的部分是BIOS（Basic Input/Output System），它占据从0x000F0000到0x000FFFFF的64KB空间。在早期PC中，BIOS被保存在真正的ROM（read-only memory）中，但是如今的PC将BIOS存储在可更新的闪存中。BIOS负责基本系统的初始化，进行显卡激活和存储器安装数量检查等工作。初始化完成后，BIOS将从适当的位置加载操作系统，比如软盘、硬盘、CD-ROM、甚至是互联网，并将控制权移交给操作系统。

> “真正的ROM”听起来多少有点奇怪。从ROM的名称，我们就可以知道它应该是只读的。但是对硬件稍有了解的同学都知道如今的ROM是可以多次写入的。这实际上是一个历史遗留问题，早期的ROM确实是一次性写入、不可更改的。
> 接触过嵌入式的同学应该都听说过“烧录”一词。我在学习嵌入式时就曾有过这样一个疑惑：单片机的程序明明可以多次写入，为什么这个过程要叫“烧录”呢？老师说，以前的单片机确实只能写入一次。但随着技术的发展，后来可以多次写入了，但“烧录”一词却传承了下来。
> “ROM”一词也属于这种情况。

尽管后来的Intel处理器突破了1MB的限制，达到了16MB、甚至4GB的物理地址空间。但为了向后兼容，仍然保留了最初的1MB物理地址空间。因此，现在的PC在0x000A0000到0x00100000的物理地址空间上有一个分割*low memory*（或者*conventional memory*）区域和*extended memory*区域的“洞”。除此之外，部分位于32位PC物理地址空间顶部的区域，现在通常预留给BIOS，提供给32位的PCI设备使用。

当前的x86处理器能支持超过4GB的物理RAM，所以RAM地址已经远远超出了0xFFFFFFFF。在这种情况下，BIOS必须在RAM的32位可寻址区域顶部规划出第二个洞，为一些32位设备映射留出空间。因为设计的局限，JOS只能使用256MB的物理地址空间，所以我们假设PC只有32位的物理地址空间。但处理复杂的物理地址空间与其他硬件设备之间的组织关系已经成为了操作系统开发实践中最大的挑战之一。

### The ROM BIOS

<br>

在本部分实验中，你将使用qemu的debug工具追踪一个Intel 32位电脑是如何启动的。

打开两个终端窗口，使用`cd`命令进入你的实验目录。在其中一个窗口中，输入`make qemu-gdb`（或者`make qemu-nox-gdb`）。qemu将被启动，但会在处理器执行第一条指令之前被挂起，并等待来自GDB的连接。在另一个窗口中，运行`make gdb`，你可以看到如下输出：
```
Copyright (C) 2021 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
+ target remote localhost:26000
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
The target architecture is set to "i8086".
[f000:fff0]    0xffff0: ljmp   $0x3630,$0xf000e05b
0x0000fff0 in ?? ()
+ symbol-file obj/kern/kernel
warning: A handler for the OS ABI "GNU/Linux" is not built into this configuration
of GDB.  Attempting to continue with the default i8086 settings.

(gdb) 
```

`[f000:fff0]    0xffff0: ljmp   $0x3630,$0xf000e05b`是GDB反汇编出的第一条执行指令。由该输出，我们可以得出以下几条结论：
- IBM PC从物理地址0x000ffff0处开始执行，这是BIOS 64KB预留区域的顶部。
- PC开始执行时，CS（段寄存器）和IP（指令指针寄存器）的地址分别为0xf000、0xfff0。
- 第一条指令是`jmp`指令，它将转跳至分段地址`CS = 0xf000, IP = 0xe05b`处。

qemu之所以这样启动，是因为Intel 8088处理器就是如此设计的，IBM将其用在了他们的早期PC上。PC的BIOS是被“硬连线（hard-wired）”到物理地址的0x000f0000-0x000fffff范围内的，这样的设计确保了BIOS总是能够在上电或重启后，第一时间控制设备。这是至关重要的，因为在上电时，RAM中没有任何其他程序可以交予处理器执行。qemu自带BIOS，它位于处理器的模拟物理地址空间中。当处理器复位时，（模拟）处理器会进入实模式，将CS和IP分别设置为0xf000和0xfff0，并从该分段地址（CS:IP）开始执行程序。那么分段地址`0xf000:fff0`是如何转换为一个物理地址的呢？

> “硬连线（hard-wired）”一词，我没有找到合适的翻译，姑且直译了。
> “hard-wired”一般用于形容计算机中不可更改的程序。

要回答这个问题，我们需要了解一点实模式寻址相关的知识。在实模式下，地址按照这个公式进行转换：*物理地址 = 16 * 段基址地址 + 段内偏移（physical address = 16 * segment + offset）*。所以，当PC把CS和IP分别设置为0xf000和0xfff0时，所指代的物理地址是：
```
16 * 0xf000 + 0xfff0   # in hex multiplication by 16 is
= 0xf0000 + 0xfff0     # easy--just append a 0.
= 0xffff0 
```
0xffff0是BIOS的最后16个字节（0x100000）。因此，BIOS所做的第一件事便是转跳回较为靠前的位置。这仅仅16个字节究竟能完成多少工作呢？

> **Exercise 2**.使用GDB的`si`（步进指令）命令追踪更多的ROM BIOS指令，并尝试推测指令的作用。
> 你可以查阅[Phil Storrs I/O Ports Description](http://web.archive.org/web/20040404164813/members.iweb.net.au/~pstorr/pcbook/book2/book2.htm)和[6.828 reference materials page](https://pdos.csail.mit.edu/6.828/2018/reference.html)。
> 你不必弄清楚所有指令的具体含义，只需要先对BIOS的工作有一个大概的了解即可。

当BIOS启动时，它会设置一个中断描述符表，并初始化多个设备，比如VGA显示器。这时，你会在qemu窗口看到一条“Starting SeaBIOS”信息。

在完成PCI总线和所有重要设备的初始化后，BIOS会搜寻引导设备，比如软盘、硬盘、CD-ROM。当BIOS找到启动盘后，它会从中读取boot loader，最后将控制权移交给boot loader。

## Part 2: The Boot Loader

PC的软盘和硬盘被划分为一个个大小为512字节、被称为扇区的区域（*sector*）。扇区是磁盘最小的传输单元：每次读写操作都必须包含一个或多个扇区，并与扇区边界对齐。如果磁盘是启动盘，则磁盘的第一个扇区被称为引导扇区，boot loader的代码就保存在这里。当BIOS找到启动盘时，它会把启动盘的512字节引导扇区加载到内存中物理地址从0x7c00到0x7dff的区域。然后，使用`jmp`指令设置CS:IP为`0000:7c00`，并将控制权转交给boot loader。和BIOS的加载地址一样，这些地址是相当随意的，但对于PC来说，它们是固定且统一的。

> 最后这句话英文原句如此。我翻译水平有限，读起来似乎有些歧义，故此说明一下。
> 前半句说这些地址*相当随意（fairly arbitrary）*是指这些地址本身是没有特殊含义的。但出于历史原因及兼容性考虑，这些地址已经成为了固定格式，所以后半句又说*它们是固定且统一的（fixed and standardized）*。
> 这就好比，文字本身是无异议的符号，但是人类在交流的过程中，逐渐赋予了其特殊且固定的含义。

从CD-ROM中加载操作系统的机制在PC的发展进程中出现得比较晚。因此，PC架构师趁机稍微重新设计了一下启动过程。所以，现代BIOS从CD-ROM中引导操作系统的方式有一点复杂（功能也更加强大了）。CD-ROM使用2048个字节的扇区，而不是512个字节。因此，在移交控制权之前，BIOS能够从磁盘中加载更大的启动镜像（不止一个扇区）。你可以查看["El Torito" Bootable CD-ROM Format Specification](https://pdos.csail.mit.edu/6.828/2018/readings/boot-cdrom.pdf)中了解更多的信息。

但在6.828中，我们将采用传统的硬盘启动机制，这意味着我们的boot loader大小必须限制在512个字节内。boot loader由一个汇编文件`boot/boot.S`和一个C文件`boot/main.c`组成。请仔细阅读这些源代码，并确保你理解了他们的工作流程。boot loader必须完成两个主要任务：
- 首先，boot loader将处理器的工作模式由实模式切换到32位的保护模式。因为只有在保护模式下，软件才能访问处理器物理地址空间中1MB以上的内存。在[PC Assembly Language](https://pdos.csail.mit.edu/6.828/2018/readings/pcasm-book.pdf)的章节1.2.7和1.2.8中，简要地介绍了保护模式。如果你想了解更多与保护模式相关的知识，可以查阅英特尔架构手册。目前，你只需要了解在保护模式下，分段地址转换为物理地址的方式与实模式是不同的，并且转换后的地址是32位的，而非16位。
- 其次，boot loader通过x86的特殊IO指令使用IDE硬盘设备寄存器，直接从硬盘中读取内核。如果你想深入学习这些特殊IO指令，可以查看[the 6.828 reference page](https://pdos.csail.mit.edu/6.828/2018/reference.html)的“IDE hard drive controller”章节。在本课程中，你不需要过多地学习如何为特定设备编程：实际上，编写设备驱动是操作系统开发中非常重要的一个环节。但如果从概念化和整体架构的角度看，它是最无聊的部分之一。

在你理解了boot loader的源码之后，请阅读`obj/boot/boot.asm`文件。该文件是GNUmakefile在编译boot loader后创建的boot loader反汇编文件。通过这个反汇编文件，我们可以轻松、准确地查看boot loader代码在物理内存中的位置。在使用GDB单步调试boot loader时，也可以方便地追踪执行过程。`obj/kern/kernel.asm`是JOS内核的反汇编，对于我们的debug同样重要。

在GDB中，你可以使用`b`命令来设置地址断点。比如，`b *0x7c00`表示在地址0x7C00处设置一个断点。在断点处，你可以使用`c`和`si`命令继续向下执行：`c`会执行到qemu的下一个断点（或者直到你在GDB中按下`Ctrl-C`），`si N`会一次性执行N条指令。

你可以使用`x/i`命令来查看内存中的指令（除了下一条将要执行的指令，GDB会自动打印它）。这个命令还有一个`x/Ni ADDR`语法，其中，N表示要连续反汇编的指令数，ADDR表示开始反汇编的地址。

> **Exercise3**.阅读[lab tools guide](https://pdos.csail.mit.edu/6.828/2018/labguide.html)，尤其是与GDB命令有关的内容，即使你已经非常熟悉GDB。其中一些（你不知道的）高级的GDB命令对你的操作系统实验非常有帮助。
> 
> 在地址0x700c处设置一个断点，这里是加载引导扇区的地方。继续执行，直至到达该断点。追踪`boot/boot.S`中的代码，通过源码和反汇编文件`obj/boot/boot.asm`，持续追踪你所处的位置。同时，使用GDB的`x/i`命令，反汇编boot loader中指令序列，并比较原本的boot loader源码与`obj/boot/boot.asm`、GDB中的反汇编代码的区别。
> 
> 追踪`boot/main.c`中的`bootmain()`函数，并进入`readsect()`函数。弄清楚`readsect()`中每条语句对应的汇编指令。继续追踪`readsect()`函数的剩余部分，直至返回`bootmain()`函数中。然后，找到从磁盘读取内核剩余扇区的for循环的开始与结束处。找出循环结束后，将会执行的代码，并设置断点。然后，继续执行至该断点处。最后，单步调试boot loader余下的部分。

回答以下问题：
- 处理器从哪里开始执行32位代码？处理器是如何从16位模式切换到32位模式的？
- boot loader执行的最后一条指令是什么？内核加载后的第一条指令是什么？
- 内核的第一条指令在哪里？
- boot loader是怎么知道从磁盘中加载整个内核需要读取多少个扇区的？它是如何获知该信息的？