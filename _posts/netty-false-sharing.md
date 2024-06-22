---
title: 'Netty学习：伪共享'
date: 2020-10-14T00:00:00+08:00
lastmod: 2020-10-14T00:00:00+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["java","伪共享","netty"]
categories: ["技术"]
lightgallery: true
---

## Netty中的伪共享
先说为什么知道这个概念吧，期初看Netty源码的时候，看到了NioEventLoop的构建，其中有这么一句代码：
```java
private static Queue<Runnable> newTaskQueue0(int maxPendingTasks) {
        // This event loop never calls takeTask()
        return maxPendingTasks == Integer.MAX_VALUE ? PlatformDependent.<Runnable>newMpscQueue()
                : PlatformDependent.<Runnable>newMpscQueue(maxPendingTasks);
}
```
其实就是给这个EventLoop创建一个存放Event的队列嘛，看过前篇的都懂，但是问题来了，为什么是这个队列，于是点进去看，发现了更神奇的代码：
```java
public class MpscUnboundedArrayQueue<E> extends BaseMpscLinkedArrayQueue<E> {
    long p0;
    long p1;
    long p2;
    long p3;
    long p4;
    long p5;
    long p6;
    long p7;
    long p10;
    long p11;
    long p12;
    long p13;
    long p14;
    long p15;
    long p16;
    long p17;

    public MpscUnboundedArrayQueue(int chunkSize) {
        super(chunkSize);
    }

    protected long availableInQueue(long pIndex, long cIndex) {
        return 2147483647L;
    }
    ……
}
```
这就是NettyEventLoop使用的队列的原貌，是一个第三方包里的，百度了下，是专门用来提高吞吐的队列，适合Netty这种多生产者单消费者（就自己在循环消费，当然是单消费者）的情况。但是问题来了：**那么多long是做什么用的呢**？

## 伪共享的原理以及介绍
伪共享为什么会出现呢？我们都知道CPU访问内存，基本是通过寄存器->L1->L2->L3->主存这么个链路的，在多核CPU的高速缓存之间，因为考虑到对相同数据的使用，会有缓存一致性协议，即通过锁或者协议去同步缓存中数据的变更。举个例子，假如加载到A核中的L3缓存的某个数据进行了修改，如果此时B核中如果也保存了该数据，则需要A核与B核之间达到一个数据修改的同步。

如果A核以及B核都大量都对该数据进行修改呢？那么竞争就会十分激烈了，在这个时候，缓存一致性协议导致的CPU时间损耗比使用高速缓存的损耗还要多。所以干脆不使用缓存了。即使用固定的数据将对象填满，此时加载到缓存中的就是那些填充的不变的long数据了。
所以在自己编写多线程使用或者高吞吐的队列或者数据结构时，一定要考虑伪共享的问题，否则性能会非常低。
