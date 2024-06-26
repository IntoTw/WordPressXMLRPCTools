---
title: '经验总结：内存泄露的原因以及分析'
date: 2022-03-17T00:00:00+08:00
lastmod: 2022-03-17T00:00:00+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["java","经验总结","oom"]
categories: ["技术"]
lightgallery: true
---



内存泄露是Javaer听到最多的关于内存的事了，这篇文章就来谈谈这件事。

## 内存泄露与资源泄露

什么是泄露？泄露在计算机语境下，通常指的是某个资源无法被访问，也无法被释放。

内存泄露一般发生在某个对象的引用丢失，无法再访问到该引用，但是该引用却依旧引用着某个对象，导致这个对象无法回收，最终导致内存溢出OOM。

资源泄露一般发生在连接池，IO流等场景，如从连接池中每次都新建连接但不关闭，每次都打开新的IO流但不关闭，等等情况。

## 内存泄露发生的情况

内存泄露多发生于static的集合中，比如当你定义了一个static HashMap，此时将某个key-value放入其中后，方法段结束。

这时，除非调用map的clear方法，否则显然该value将无限持有对象的引用，无法释放。

```java
public static Map<String,Object> objectMap=new HashMap<>();
public static void main(String[] args) {
    Integer a=new Integer(1);
    objectMap.put("testKey",a);
}
```

这种写法看似可笑，却很难避免，尤其在大量框架代码中，反而更容易发生，因为大部分框架代码对业务代码的增强，都是通过AOP方式来做的，此时对业务代码来说，这类隐式的static Map难以防范。

## 内存使用过高，一定是内存泄露吗？

内存使用过高，并不一定是内存泄露导致的结果，具体要看内存堆的分析。

一般内存泄露最直观的体现就是：
1. 内存使用高
2. GC回收不了内存，即GC前后堆大小几乎无变化
3. JVM疯狂GC,CPU打满
4. Java进程触发Linux操作系统的OOM-killer，Java进程被杀死
5. 或者CPU被GC任务打满，服务器实际宕机。

但是这不一定是泄露导致的，也有可能是内存的错误使用导致的，不过大同小异，主要还是需要排查异常内存的使用。

Ps：之所以作者这么说，是因为作者曾经在线上遇到了架构组修改日志框架，错误的将日志内容作为了key存入了map，本应的key-value应该为traceId-日志内容，结果架构组却将key-value搞反了，导致大量的巨大key打满了内存，堆dump文件里全是几十k几十k的字符串。

## 如何避免内存泄露

根据上面说的内存泄露多数发生的情况，避免内存泄露的策略也就十分简单了。

1. 尽量使用局部变量
2. 减少使用static集合
3. 如果必要的使用static集合，尽量使用弱引用等低级引用。比如参照ThreadLocal中的设计：[TheadLocal原理](https://intotw.cn/posts/java-threadlocal/)

## 内存泄露问题如何排查

内存泄露或内存持续使用较高时，通常通过堆的情况来排查。

首先可以通过jmap -histo:live pid|less 命令，查看堆内对象使用情况。此时如果内存泄露，一般都是会某个基本类型对象过多，然后可以与正常的服务作对比，看哪个对象的数量异常的多，此时如果可以判断出来，也没必要dump了。

如果通过jmap无法断定，则可以使用jmap -dump:live,format=b,file=<filename> 命令，生成dump文件。

将dump文件通过java原生的软件或者eclipse的mat工具，就可以看到哪些对象占用过多，此时你应该关注的是非基本类型对象的其他对象，因为一般来说都是基本类型的数量和大小最多。

一般来说，你会看到以下现象：
1. 某个map的Node十分多，有几十万个。
2. 某个框架的某个对象十分多。
3. char数据，也就是C[]，占用十分多，因为有很多大字符串。
