---
title: '大循环与小循环嵌套的性能比较（分支预测）'
date: 2021-03-02
lastmod: 2021-03-02
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["计算机技术","cs"]
categories: ["技术"]
lightgallery: true
---

面试被问到一个很有意思的问题：大循环和小循环，哪个在外哪个在里有区别吗？为什么？哪种更快？

当时确实没有答上来也没想到，明明之前看CSAPP了解过CPU的指令分支预测的，但是实在没有想到这里去。

先上个图：
![](https://images.intotw.cn/blog/2023/09/c23ac577ea92a466b0420511e70b7c2f.png)

再来个解释的比较清楚的博客：

[https://segmentfault.com/a/1190000006889989](https://segmentfault.com/a/1190000006889989)

简而言之，就是当进行循环时，因为判断循环条件也是属于分支预测，所以大循环在内时，分支预测连续成功的次数会更高，会进行更少的指令回退，CPU执行的会更快。


后面实际测试了一下，下述代码：
```java
 long k=0;
        final long currentTimeMillis = System.currentTimeMillis();
        for (int j = 0; j < 1000000000; j++) {
            for (int i = 0; i < 100; i++) {
                k++;
            }
        }
        System.out.println(System.currentTimeMillis()-currentTimeMillis);
```
当大循环在内时，平均执行时间：
```java
avg(2255 2291  2395 2237 2318)=2299.2
```
当大循环在外时，平均执行时间：
```java
avg(4683 4651 4708 4810 4856)=4741.6
```
确实慢了1倍多。
