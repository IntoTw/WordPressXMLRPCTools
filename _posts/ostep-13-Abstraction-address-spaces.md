---
title: 'ostep学习笔记-13 Abstraction Address Spaces'
date: 2024-03-18T14:25:50+08:00
lastmod: 2024-03-18T14:25:50+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["操作系统"]
categories: ["ostep"]
lightgallery: true
math: true
---

这章主要描述了内存中对进程来说地址空间是什么东西

## 地址空间

在进程来看，地址空间分为3部分，代码（code），堆（heap），栈（stack）

在进程自己的地址空间中视图如下

![](https://images.intotw.cn/blog/2024/03/ce4aff19c5178ae920488e3d77230f8c.png)

其中随着内存的申请和栈的扩大，堆会在内存中向下扩展，而栈是向上，也就是会逐渐填满free那块。

还有段代码比较经典，打印进程中main函数的地址（代码），堆的地址（使用malloc），还有栈的地址（函数内的变量），这里比较神奇的是c竟然可以使用main来指代main函数，这看起来像是个函数式的东西，神奇

```c
#include <stdio.h>
#include <stdlib.h>
int main(int argc, char*argv[]) {
    printf("location of code : %p\n", main);
    printf("location of heap : %p\n", malloc(100e6));
    int x = 3;
    printf("location of stack: %p\n", &x);
    return x;
}
```

然后就是总结下实现虚拟内存的3个目标：

+ 透明性，对用户进程来说内存应该是透明的，用户进程不需要知道底层的实现，它可以认为它拥有整个内存
+ 性能，这点不比多说
+ 保护性，我们需要保护不同进程间的内存安全，甚至保护OS自己的内存安全，防止用户进程修改OS
