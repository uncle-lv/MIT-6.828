# Lab1 Booting a PC

## 概述

本次实验的主要内容是探究计算机的启动过程。计算机的启动过程是指从CPU加电到操作系统被加载到内存中计算机所进行的一系列操作。通俗地讲，就是从我们按下开机键到计算机被完全启动的这个过程。

## Part 1: PC Bootstrap

<br>

### The PC's Physical Address Space

<br>

>注：这里我略过了*Getting Started with x86 assembly*和*Simulating the x86*这两部分内容。第一部分是学习x86汇编语法，需要读者自己学习。第二部分是使用qemu虚拟机模拟x86环境，这一部分我们已经在[环境准备](https://github.com/uncle-lv/MIT-6.828/blob/main/0.environ-preparation.md#%E9%AA%8C%E8%AF%81%E7%8E%AF%E5%A2%83)这一章节中学习过了，不再赘述。

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

要回答这个问题，我们需要了解一点实模式寻址相关的知识。在实模式下，地址按照这个公式进行转换：*物理地址 = 16 * 段基址地址 + 段内偏移（address = 16 * segment + offset）*。所以，当PC把CS和IP分别设置为0xf000和0xfff0时，所指代的物理地址是：
```
16 * 0xf000 + 0xfff0   # in hex multiplication by 16 is
= 0xf0000 + 0xfff0     # easy--just append a 0.
= 0xffff0 
```
0xffff0是BIOS的最后16个字节（0x100000）。因此，BIOS所做的第一件事便是转跳回较为靠前的位置。这仅仅16个字节究竟能完成多少工作呢？

> **Exercise 2.**使用GDB的`si`（步进指令）命令追踪更多的ROM BIOS指令，并尝试推测指令的作用。
> 你可以查阅[Phil Storrs I/O Ports Description](http://web.archive.org/web/20040404164813/members.iweb.net.au/~pstorr/pcbook/book2/book2.htm)和[6.828 reference materials page](https://pdos.csail.mit.edu/6.828/2018/reference.html)。
> 你不必弄清楚所有指令的具体含义，只需要先对BIOS的工作有一个大概的了解即可。

当BIOS启动时，它会设置一个中断描述符表，并初始化多个设备，比如VGA显示器。这时，你会在qemu窗口看到一条“Starting SeaBIOS”信息。

在完成PCI总线和所有重要设备的初始化后，BIOS会搜寻引导设备，比如软盘、硬盘、CD-ROM。当BIOS找到启动盘后，它会从中读取boot loader，最后将控制权移交给boot loader。