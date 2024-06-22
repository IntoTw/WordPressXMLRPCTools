---
title: 'Java基础：ThreadLocal及其原理'
date: 2021-03-02
lastmod: 2021-03-02
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["java","threadlocal"]
categories: ["技术"]
lightgallery: true
---

## ThreadLocal的用处

ThreadLocal是一个多线程的辅助工具类，目的是方便开发者维护多线程中的共享变量。我们知道如果我们想要在一个线程中一直访问一个变量或者在线程上下文中保存一个变量，我们要么将该变量声明为static静态，要么就在每一步函数调用中均传入该变量。这两种方式，static方式不能解决每个线程同时分别持有的问题。每一步函数传入对代码的侵入性过高。

所以ThreadLocal实际的使用效果就是以线程维度，让每个线程拥有了一个自己的上下文变量池，防止大量的静态声明。这也是为什么它并不在juc并发包里的原因。

## ThreadLocal的实现

通过上述TheadLocal的用处和它解决的问题，我们可以马上想到一个很简单的实现来补充static实现的缺点：我们可以通过定义一个static的Map，其中key为线程或线程id，value为需要保存的变量。
TheadLocal的实际实现方法也类似，不过在其中进行了一定改进：
1. TheadLocal实现体系中，由Thead本身持有该Map，其中Key为TheadLocal，value为值。
2. TheadLocal本身仅通过同包可以访问的特性，提供了一系列访问Thead中的Map的Api。


## ThreadLocal防止内存泄露

我们知道，TheadLocal这种实现体系下，Thread会持有一个Map，并且该Map中会大量使用TheadLocal作为key，Map引用关系默认是强引用。在线程池或连接池这种会对线程进行复用的场景下，为了防止TheadLocal被强引用导致一直不能被回收，所以Thead中对TheadLocal的维护Map使用的是一个简单实现的WeakReferenceMap。该Map有以下特点：
1. 该Map的哈希冲突使用跳跃式处理，而不是HashMap中的桶。原因应该是把该空间当做栈来思考的话，使用跳跃式可以减少更多的空间分配，并且更好的利用cpu缓存。
2. 该Map的Entry，由继承了WeakReference的Entry实现，该Entry本身为TheadLocal的一个弱引用，当TheadLocal的强引用失去，被gc回收后。该Map在其各方法中都增加了清理Map节点的步骤，根据该弱引用对应的弱引用回收队列中的节点，清理Map中对应的键值对。

## ThreadLocal为什么会内存泄露

这点其实应该放在上面那点之前说，但是这个确实也困扰了比较久，钻了牛角尖，一直没有明白。
我们日常中一般使用TheadLocal的方式都如下，比如使用一个Util类来维护请求的TraceId，进行链路追踪的效果：
```java
public class TraceIDUtils {
	private static final ThreadLocal<String> SEQUENCE_ID = new ThreadLocal<String>();
}
```
那么我们会发现，该TheadLocal对象，本身就是一个static声明，永远存在来源于该类的强引用，那么该TheadLocal本身就不会被回收，Thead中的Map是否使用弱引用好像毫无意义。

当时也是思考了比较久，后面才发现钻了牛角尖，当时大牛们设计出来，应该是考虑类似情况：
```java
public class Test {
    ThreadLocal<String> threadLocal;
    public void run(){
        this.threadLocal = new ThreadLocal<>();
        this.threadLocal.set("abc");
        //do something
        //……
        //end
    }
    public static void main(String[] args) throws Exception {
        new Test().run();
    }
}
```
我们现在当然知道，不使用static定义，肯定是不合适的，但是Java的大牛们作为Api的设计者，肯定是要考虑到其他人在任何情况下使用该工具都应是安全的。而在这种使用方式下，在线程被销毁前，显然如果线程中的Map对该TheadLocal为强引用，那么在该方法执行完毕后该方法内的强引用消失后，依旧在Map上会直接造成内存泄露。

这也能使人明白为什么Java不选择在TheadLocal中维护一个以Thead为Key的Map来实现。
