# Lab1 Booting a PC

## 概述

本次实验的主要内容是探究计算机的启动过程。计算机的启动过程是指从CPU加电到操作系统被加载到内存中计算机所进行的一系列操作。通俗地讲，就是从我们按下开机键到计算机被完全启动的这一过程。

## The PC's Physical Address Space

<br/>

>注：这里我略过了*Getting Started with x86 assembly*和*Simulating the x86*这两部分内容。第一部分是学习x86汇编语法，需要读者自己学习。第二部分是使用qemu虚拟机模拟x86环境，这一部分我们已经在[环境准备](https://github.com/uncle-lv/MIT-6.828/blob/main/0.environ-preparation.md#%E9%AA%8C%E8%AF%81%E7%8E%AF%E5%A2%83)这一章节中学习过了，不再赘述。

<br/>

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

基于Intel 8088 16位处理器的早期PC只能操作1M的物理内存。被标记为*Low Memory*的640K空间是早期PC唯一可以使用RAM（random-access memory）。


