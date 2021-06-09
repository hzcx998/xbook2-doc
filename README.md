# xbook2 简介

xbook2操作系统内核是一个基于intel x86平台的32位处理器的系统内核，可运行在qemu，bochs，virtual box，vmware等虚拟机中。也可以在物理机上面运行（需要有系统支持的驱动才行）

## xbook2 概述

xbook2，由x、book和2组成，x代表着增强，book代表着bookos，2代表着内核的第二代。它是一个分时多进程/多线程，支持MMU的现代操作系统，它和Linux内核，WindowsNT内核是同一个数量级的。最开始是因为对操作系统感兴趣，学习，研究，才开发了它，但现在，有了更多的战略打算，希望有朝一日能够安装到个人电脑上面，为用户提供一个迷你版本的系统内核。

## 许可协议

xbook2采用MIT开源协议，可以自由的复制和修改代码，可以在此基础上DIY一个属于你自己的操作系统内核。

## xbook2 的架构

xbook2的内核是典型的宏内核，宏内核的优点是性能高效，简单，直接。由于Linux内核比较庞大，学习成本比较高，而xbook2内核比较精简，学习成本比较低，更适合初学者入门学习。

![xbook2 内核框架图](figures/framework_diagram.png)

它具体包括一下内容：

* x86处理器支持
* elf可执行文件格式
* 内核多线程/多进程/用户态多线程
* memcache内存管理
* FSAL文件系统
* pipe,fifo,sema,msgqueue,shm等进程间通信
* net网络（基于lwip协议栈）
* view图形系统
* ...

