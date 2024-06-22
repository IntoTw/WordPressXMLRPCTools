---
title: 'ostep学习笔记-11-12 Summary Cpi Virtualization'
date: 2024-03-18T13:50:28+08:00
lastmod: 2024-03-18T13:50:28+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["操作系统"]
categories: ["ostep"]
lightgallery: true
math: true
---

第11以及第12章主要是通过对话总结并且引申出了内存的虚拟化，并且都是对话内容，所以一起写一篇

## CPU的虚拟化

cpu的虚拟化大概有以下要点要理解

### 进程切换

进程的切换主要由操作系统完成，通过对进程的定义（进程对象中使用一个结构体来保存寄存器的状态），在切换进程时由操作系统将当前CPU的寄存器状态从内存中还原回CPU或者从CPU保存到寄存器中

### Limited direct execution内核模式

操作系统通过Cpu提供的limited direct execution（限制直接执行），使得OS运行在高优先级的内核模式（privilege mode or *kernal* mode），而用户代码则运行在低优先级的user mode。对于设备以及内存和CPU的访问（比较具体的在第15章），只有OS在内核模式才可以执行。

### 调度

用户进程通过OS提供的api进入中断来使用一些高优先级的功能，例如IO。在进程生命周期模型上

+ 中断后的进程会进入Blocking状态，阻塞等到OS完成操作的执行
+ OS执行完进程的要求后，会将进程修改为Waiting状态，等待调度程序执行
+ 调度程序通过调度策略（MLFQ或者CFS），确定该进程该得到执行后，将进程执行，此时为Running状态
+ Running状态的进程，当时间片执行完（其实是由调度策略的定时中断或者操作系统主动的定时中断触发的）或者再次发起IO等中断时，再次回到Waiting或者Blocking状态

操作系统为了防止一直由用户进程持有CPU执行，会在操作系统启动时注册定时中断，以此来定时从用户进程中中断来获取cpu控制权来执行那些操作系统该操作的事

主要实现调度的策略有MLFQ和CFS。

其中MLFQ使用多级反馈队列，通过进程执行时的表现来决定进程执行的优先级以此把他们在不同优先级的队列中来回移动。MLFQ适用于带交互界面的操作系统，如Mac和Windows等，更重视的是用户交互进程的response time响应，保障用户使用时不会有过于明显的卡顿

CFS是Linux或datacenter系统使用的，主要考虑的是进程之间对于CPU资源的公平性以及权重。主要使用vruntime这个一定比例由实际使用cpu的realtime乘以权重等数值计算而来的*虚拟执行时间*来保证CPU资源的公平分配，所以适用于那些交互界面简单（命令行），以及主要执行计算任务的OS。

## 内存的虚拟化

这里就比较简单了，主要是2个概念：

+ 操作系统通过内存API来为用户进程提供内存的使用
+ 所有用户侧看到的内存都是虚拟化的
+ 由操作系统提供进程间内存的隔离性以及保护

