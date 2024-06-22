---
title: 'ostep学习笔记-14 Interlude: Memory API'
date: 2024-03-18T14:38:39+08:00
lastmod: 2024-03-18T14:38:39+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["操作系统"]
categories: ["ostep"]
lightgallery: true
math: true
---

这章主要介绍了操作系统给进程提供用的内存的API，主要是C语言的视角。

## 内存类型

### 栈

栈内存由进程在编译时编译器就可以决定，栈内存主要是函数中的这类内存：

```c
void func() {int x; // declares an integer on the stack...}
```

简而言之就是函数内的局部变量，这些变量在编译时就可以确定大小以及数量，在函数执行完后销毁。

> java与其何其相似，jvm也是将局部变量分配在函数的栈帧上，随着函数的调用分配以及销毁的。只能说内存管理大道至简，大同小异

### 堆

栈内存只在函数上，并且由编译器管理，如果需要跨越函数的上下文域或者在进程整个周期中传递使用，那么就需要在堆中申请内存。

``` c
void func() {int*x = (int*) malloc(sizeof(int));}
```

使用malloc申请堆内存，入参是内存的大小字节，返回的是内存起始地址的地址指针。这个例子就申请了一个int的位置（4字节），并且返回了一个指向地址初始地址的指针x。注意一个比较特殊的点，这里其实既有一个栈的内存又有一个堆的内存，(int *x)是在栈上的，他所指向的内存是在堆上的。

> eh，和java何其相似，Object a=new Object()。a在栈上，后面的对象在堆上。

## 内存API

### malloc()

上面有例子，也比较简单，入参就是要申请的大小（字节），返回为void，需要用户强转成自己需要的类型

### free()

释放malloc的内存

```c
int*x = malloc(10*sizeof(int));...free(x);
```

注意这里只用送起始的指针，因为实际的大小由内存分配库自己去追踪

## 容易出现的问题

堆内存管理因为由用户自己管理，很容易出现问题

+ 忘记申请内存

+ 申请的内存过小

+ 忘记初始化内存

+ 忘记释放内存

+ 在处理完成前提前释放了内存

+ 重复的释放了内存

  >
  >
  >看看java解决了什么，java的堆和gc解决了123456，真是太苦啦，可惜代价就是性能

  
