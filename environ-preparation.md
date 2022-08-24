# 环境准备

## 概述

你需要在**Linux**环境下进行6.828实验。我所使用的Linux发行版是WSL2下的Debian 11，其具体信息如下：
```
Linux b6412ca92ec6 5.10.16.3-microsoft-standard-WSL2 #1 SMP Fri Apr 2 22:23:49 UTC 2021 x86_64 GNU/Linux
```

如果你所使用的宿主系统也是Windows，我建议使用WSL2搭建的你Linux环境。

> 注：如果你所使用的Linux是Debian的衍生版本（如Ubuntu、Deepin），那么以下命令应该是能够通用的。

## 安装工具

使用如下命令安装实验所需的软件工具：

```bash
sudo apt install -y git build-essential gdb gcc-multilib qemu-system-i386
```

**git**是一个开源的分布式版本控制工具。MIT使用git对6.828的实验代码进行管理。

**build-essential**包含了编译C/C++所需要的软件，包括gcc、g++、make等。

**gdb**是Linux下常用的程序调试工具。

**gcc-multilib**是一个交叉编译工具。如今大部分平台都是64位的，但是我们需要编译能在32位环境下运行的程序。

**qemu-system-i386**是一款高性能虚拟机，我们用它来模拟一个32位的计算机平台。

## 验证环境

如果你已经完成了上述软件的安装，现在就可以按照如下步骤验证实验环境是否可以正常运行了。

1.首先，按照lab1的步骤，下载代码：
```
mkdir ~/6.828
cd ~/6.828
git clone https://pdos.csail.mit.edu/6.828/2018/jos.git lab
cd lab
```

2.在lab目录下，使用 `make` 命令生成系统镜像。

如果看到以下输出，说明成功生成镜像：
```
+ as kern/entry.S
+ cc kern/entrypgdir.c
+ cc kern/init.c
+ cc kern/console.c
+ cc kern/monitor.c
+ cc kern/printf.c
+ cc kern/kdebug.c
+ cc lib/printfmt.c
+ cc lib/readline.c
+ cc lib/string.c
+ ld obj/kern/kernel
ld: warning: section `.bss' type changed to PROGBITS
+ as boot/boot.S
+ cc -Os boot/main.c
+ ld boot/boot
boot block is 396 bytes (max 510)
+ mk obj/kern/kernel.img
```

3.使用 `make qemu` 命令启动虚拟机，看到如下界面说明虚拟机已成功启动：
```
Booting from Hard Disk..6828 decimal is XXX octal!
entering test_backtrace 5
entering test_backtrace 4
entering test_backtrace 3
entering test_backtrace 2
entering test_backtrace 1
entering test_backtrace 0
leaving test_backtrace 0
leaving test_backtrace 1
leaving test_backtrace 2
leaving test_backtrace 3
leaving test_backtrace 4
leaving test_backtrace 5
Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
K> 
```

使用 `ctrl + a  x` 即可退出虚拟机。

<font color="red">注意</font>：如果没有桌面环境，使用 `make qemu` 会出现如下错误：
```
qemu-system-i386 -drive file=obj/kern/kernel.img,index=0,media=disk,format=raw -serial mon:stdio -gdb tcp::25000 -D qemu.log 
Unable to init server: Could not connect: Connection refused
gtk initialization failed
make: *** [GNUmakefile:156: qemu] Error 
```
将命令更改为 `make qemu-nox` 即可。


## 补充

我准备了一个已搭建好实验环境的docker镜像，你也可以直接拉取这个[镜像](https://hub.docker.com/repository/docker/unclelv/6.828)来作为你的实验环境。
