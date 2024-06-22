---
title: 'Ostep 18 Paging Introduction'
date: 2024-03-26T16:46:04+08:00
lastmod: 2024-03-26T16:46:04+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["操作系统"]
categories: ["ostep"]
lightgallery: true
math: true
---

这章引出了现在操作系统内存管理中最重要，也是最复杂的概念：页。就本书而言，都使用了长达3章的大量篇幅来介绍

基于段管理带来的复杂性以及带来的碎片，还有另外一种方式去管理内存，也就是把内存划分为固定大小fixed-sized的页

文中通过不少深入浅出的概念循序渐进的介绍，在笔记里我们就只总结关键的概念和内容了

## 页的概念

将内存分为固定大小的块，每块称之为一页。划分以后，管理的维度也就会变成页，即回收、分配等操作，都以页为单位进行。所以页的划分可以在OS一开始boot的时候就进行，直接确定页的大小以及数量。比如32K的内存，每页1K的话，内存理所当然就有32页了。

有两类页，一些是虚拟的页，一些是真实物理内存的页，其中虚拟的页属于每个进程自己的地址空间，真实物理内存的页由操作系统管理，其中当然也需要地址转换。文中没有这类图，自己简单画了一个，左边就是各进程自己的虚拟页，右边是物理内存里实际的页。注意这里虚拟页不存储数据，只是一个地址空间，后面会继续讲

![](https://images.intotw.cn/blog/2024/03/a6993176ebc7600477f511a38dc53bc3.png)

## 页地址的转换

从上面的图可以看出来，虚拟页需要对应到实际的物理内存的页，才可以拿到数据，我们看下结构和转换逻辑

![](https://images.intotw.cn/blog/2024/03/c7ac0acf85194d9ca08c0a295102ec8f.png)

从图中我们可以看到，虚拟页的寻址，将地址划分为VPN（虚拟页码）以及offset（偏移），物理页也类似。其中如图，假设我们虚拟地址的前两位认为存储的是VPN，后面其余位认为是存储的offset的话，那么一个虚拟地址访问对应实际物理地址访问的实际过程可以概括如下

1. 通过掩码位运算得到VPN以及offset
2. VPN通过**地址转换程序**，得到物理页码PFN
3. 物理页码PFN*物理页大小+offset，就等于实际访问的物理地址

## 页表

因为从虚拟页到物理页的转换，页表page table就必须存在，用于维护虚拟页和物理页实际的映射关系。

页表自身也需要内存，甚至会占用大量内存。因为每个进程都需要拥有独立的页表（回忆一下之前的前提：每个进程都认为自己拥有整块内存，所以每个进程都拥有一个独立的虚拟地址空间来映射到物理地址，因此他们都需要一个独立的页表），所以我们假设我们使用32位长度的虚拟地址空间，其中每页的大小我们假如认为是4kb，那么因为页大小是4kb，所以后12位（$2^{12}=4096$）用于描述页内的offset。而前面的20位就拿来描述页码了。也就是说，页码的范围是0-$2^{20}=1048576$。

如果页表每个元素需要4byte来表示，那么一张页表我们就需要4mb来描述了。要知道页表是每个进程存在一个的，如果现在有100个进程在运行，那么我们光储存页表都需要400mb的空间。这也是这种朴素的线性页表设计所存在的问题。占用了过量的空间。

## 页表的元素中大概有那些信息（举例）

大概如图，梳理下书中的这些字段

![](https://images.intotw.cn/blog/2024/03/4272a46ec5f4cecdaf6f418360541ff0.png)

为什么没有VFN呢，这里主要是认为VFN直接就是Index，所以里面只有对应的PFN。里面的字段如下：

+ valid bit，这个元素是否有用，比如说进程刚启动，大部分页都还没有被使用，或者这个页已经不再被用了，会被修改为invalid
+ protection bits，标记可读可写可执行，对应的是不同进程对同一个物理内存页的权限
+ present bit，标记映射到的物理页是否在内存中，因为我们后面会知道，内存中的页也会经常被交换到硬盘中
+ reference bit，引用标记，一般用于记录页面的使用次数等信息，服务于页面汰换算法

## 页的代价

使用页也带来了一些代价，上面说过的内存400MB占用例子是其一。

还有一个影响就是其中转换逻辑不简单，参考这段最简单的示例代码，哪怕只是简单的处理一下虚拟页和物理页的页码转换，都需要做2次掩码和一次计算

```c
// Extract the VPN from the virtual address
	VPN = (VirtualAddress & VPN_MASK) >> SHIFT
// Form the address of the page-table entry (PTE)
	PTEAddr = PTBR + (VPN*sizeof(PTE))
// Fetch the PTE8PTE = AccessMemory(PTEAddr)
// Check if process can access the page
if (PTE.Valid == False)
	RaiseException(SEGMENTATION_FAULT)
else if (CanAccess(PTE.ProtectBits) == False)
	RaiseException(PROTECTION_FAULT)
else
	// Access is OK: form physical address and fetch it
	offset   = VirtualAddress & OFFSET_MASK
	PhysAddr = (PTE.PFN << PFN_SHIFT) | offset19Register = AccessMemory(PhysAddr)
```

上面逻辑看起来简单，毕竟简单的*和&还有位移操作对CPU来说只是洒洒水，但是我们考虑到实际情况：

```c
int array[1000];...for (i = 0; i < 1000; i++)array[i] = 0;
```

一个简单的循环，我们除了在寄存器上进行单纯的加之外，仅仅获取到实际的物理页转换所要做的额外操作，就已经远大于寄存器加法了~

所以这章的Page基本概念也就介绍到此了，上面的问题也引出了后面2章的内容，19章我们尝试去解决这个转换速率的开销。20章我们去解决内存占用的开销。
