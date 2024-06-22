---
title: '工作记录：新生代老年代比例错误问题'
date: 2023-09-19T16:25:10+08:00
lastmod: 2023-09-19T16:25:10+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/09/e738025845be87723dbf01a3442ea7b7.jpeg"
featuredImagePreview: "https://images.intotw.cn/blog/2023/09/e738025845be87723dbf01a3442ea7b7.jpeg"
tags: ["java","工作记录","线上问题"]
categories: ["技术"]
lightgallery: true
---

# 线上排查：新生代老年代比例错误问题

## 起因

线上一个应用频繁full gc，排查发现单pod总内存3g的情况下新生代只有200mb，很奇怪，于是到容器里查看jvm参数。

jamp -heap 1，打印

```java
Heap Configuration:
   MinHeapFreeRatio         = 40
   MaxHeapFreeRatio         = 70
   MaxHeapSize              = 3221225472 (3072.0MB)
   NewSize                  = 261685248 (249.5625MB)
   MaxNewSize               = 261685248 (249.5625MB)
   OldSize                  = 2959540224 (2822.4375MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 268435456 (256.0MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 268435456 (256.0MB)
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
New Generation (Eden + 1 Survivor Space):
   capacity = 235536384 (224.625MB)
   used     = 45215760 (43.12110900878906MB)
   free     = 190320624 (181.50389099121094MB)
   19.19693222428005% used
Eden Space:
   capacity = 209387520 (199.6875MB)
   used     = 33745216 (32.18194580078125MB)
   free     = 175642304 (167.50555419921875MB)
   16.116154391627543% used
From Space:
   capacity = 26148864 (24.9375MB)
   used     = 11470544 (10.939163208007812MB)
   free     = 14678320 (13.998336791992188MB)
   43.866318628602755% used
To Space:
   capacity = 26148864 (24.9375MB)
   used     = 0 (0.0MB)
   free     = 26148864 (24.9375MB)
   0.0% used
concurrent mark-sweep generation:
   capacity = 2959540224 (2822.4375MB)
   used     = 279529808 (266.5803985595703MB)
   free     = 2680010416 (2555.8571014404297MB)
   9.445041690367646% used
```

NewRatio确实是2，但是新生代是224mb，老年代是2.8g，比例确实和监控一样，不符合默认的1:2配比

## 排查

一番排查，google“NewRatio not work”后，发现一个JDK的bug：
https://bugs.openjdk.org/browse/JDK-8153578

检查下JVM参数，确实使用了UseConcMarkSweepGC

## 解决


在启动参数中指定新生代GC算法-XX:+UseParNewGC后，重新发布解决
解决方式还可以在启动参数里指定-XXNewRatio=2解决，不过我们是用的指定新生代算法
