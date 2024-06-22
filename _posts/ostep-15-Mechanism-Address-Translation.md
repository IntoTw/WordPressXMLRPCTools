---
title: 'ostep-15 Mechanism Address Translation'
date: 2024-03-18T16:25:22+08:00
lastmod: 2024-03-18T16:25:22+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["操作系统"]
categories: ["ostep"]
lightgallery: true
math: true
---

这章顾名思义，主要介绍了OS中地址转换的细节，之前提到过，用户进程看到的一定是虚拟地址空间，其中虚拟地址空间的转换就由地址转换来做。

这章的后半部分结合地址翻译器，总结了一下目前OS所有需要通过硬件支持做到的事情，包括中断、限制直接执行、地址翻译等等。

其实这章真正描述的地址转换相关内容很少，大概有以下要点：

+ 地址转换主要通过硬件实现，在每个CPU上，会有2个寄存器，专门保存一个base和一个bound，base保存地址的开始，bound保存这块内存的size
+ 某个进程获取内存时，由OS通过内存的free list（暂时理解为一个记录空闲内存的列表），在**物理地址**中选取一块区域分配给进程，然后因为当前在内核模式，可以直接去修改寄存器中的base为内存**物理地址**的开始地址，bound为分配给进程的这块内存的大小
+ 在进程执行代码或者尝试操作内存时，实际操作的**物理地址**=虚拟地址+base（偏移量），这样用户进程就可以认为自己的内存实际从0开始拥有整块内存
+ 同时在执行内存相关操作时，CPU会通过实际访问的内存空间，结合base+bound圈定实际访问的内存范围，如果进程访问到了超过base+bound之外的内存，那么显然这个是越权违法操作，CPU就会触发OS的error handler让OS去决策如何操作（一般是终止进程）
+ 当然，OS在进行进程切换上下文操作的时候，需要保存和恢复base和bound寄存器中的数据，这样才能保证进程切换正确

## 截止到目前为止的硬件对OS的核心支持能力

这里书中描述的很详细了，直接贴图吧

![](https://images.intotw.cn/blog/2024/03/1f07ca388605c1ab0041046e0e16d2a7.png)

操作系统针对内存虚拟化需要具备的能力

![image-20240618155430143](https://images.intotw.cn/blog/2024/06/5fc3e8e8555ef2ca48f4992f4c0989dc.png)

操作系统启动时需要初始化的一系列东西，我们可以看到有很多handler处理中断，操作系统需要初始化trap table，中断表来响应这些中断，还有启动定时中断器来定时获取整个计算机的控制权

![image-20240618155500249](https://images.intotw.cn/blog/2024/06/7e3362418d250e4d0378df2329a91e58.png)

然后是一个时序图，完整的描述了目前位置我们的知识中，操作系统对进程的创建、执行、调度，以及操作系统如何和硬件一起帮助用户进程执行

![](https://images.intotw.cn/blog/2024/03/d37e3df2c7b078fdb0708fa2916b6488.png)
