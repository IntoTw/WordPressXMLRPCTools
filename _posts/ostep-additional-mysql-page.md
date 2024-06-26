---
title: 'ostep-扩展思考-mysql的B+树与内存页'
date: 2024-05-124T14:42:34+08:00
lastmod: 2024-05-24T14:42:34+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["操作系统"]
categories: ["ostep"]
lightgallery: true
math: true

---

​	了解完操作系统内存中的页以后，忽然想到了一个知识点，就是mysql的b+树设计，所以简单做下扩展知识。记录一下思考，基本也能覆盖b+树的面试题

## b+树页大小

​	页大小是16kb，因为操作系统默认的内存页大小是4kb或者8kb，16kb选择作为整数倍，整存整取提高性能

## b+树层数与存储的数据

​	b+树一般为3-4层，因为我们参考操作系统多级页表设计可以得知，在操作系统中，多级页表的每级每个节点都是设计成页大小。所以从性能考虑mysql也把叶子节点也设计成页的大小，即整棵树的所有节点的大小都是页大小。

​	那么我们可以简单的估计下，在一个16kb的节点上，如果储存一个索引信息需要8b的话，可以存储约2048个索引信息。3层的b+树，在叶子结点存储数据的情况下，可以表示4194304个索引信息。也就是最后一层共有420w页，最后存储的行数=4200000\*(16\*1024/每行的大小）。

​	在叶子节点，对于一页16kb一般来说可以保存10行左右，所以是约4000w行。但是除了索引信息，每个索引还要保存下一层的指针，所以总共会比4000w少一些，大约是2000-4000w。

















