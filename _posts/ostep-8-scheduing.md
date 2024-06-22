---
title: 'ostep学习笔记-8 Scheduing'
date: 2024-03-11T15:24:01+08:00
lastmod: 2024-03-11T15:24:01+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["操作系统"]
categories: ["ostep"]
lightgallery: true
---

从第八章开始，写一下学习笔记，因为从这张开始深入到一些算法以及原理了，最好还是写笔记记录一下

这张主要通过一系列问题的讨论，介绍并优化了MLFQ，Multi-Level Feedback Queue（多级反馈队列）这一操作系统调度任务的算法。

这个算法主要逻辑（讨论后最终给出）如下：

+ Rule 1:If Priority(A)>Priority(B), A runs (B doesn’t).如果任务A的优先级大于B，将先运行A。
+ Rule 2:If Priority(A)=Priority(B), A & B run in round-robin fash-ion using the time slice (quantum length) of the given queue.如果A的优先级大于B，将在AB之间使用轮询算法轮流执行。
+ Rule 3:When a job enters the system, it is placed at the highestpriority (the topmost queue).当任务进入系统时，将被放在优先级最高级的队列中
+ Rule 4:Once a job uses up its time allotment at a given level (re-gardless of how many times it has given up the CPU), its priority isreduced (i.e., it moves down one queue).当一个任务使用完在当前队列所分配给他的时间片之后，降低他的优先级，也就是移动到下一优先级队列中去
+ Rule 5:After some time period S, move all the jobs in the system to the topmost queue.在给定的S时间间隔以后，把所有任务移动到最高级队列中去

其中123显而易见可以作为基本的规则，4是为了解决在高优先级使用一些把戏去抢占cpu（比如在最初的设计中，采用的是如果任务发生了I/O或者放弃CPU的操作，就不会对优先级进行降级，而是保存在原队列中，以至于低优先级任务永远得不到执行的问题。5是为了在不断有任务到达，或者某个CPU高占用的任务过一段时间后转换成了I/O多的任务时，给低优先级任务一个被重新规划队列的机会。

针对这个队列，不同操作系统细节的实现以及不同的调参影响了实际的性能，一般来说，以下几个参数比较常用

+ 优先队列的数量，这个指的是优先队列一共有多少级，比如说一般是60级优先队列
+ 优先队列的时间片分配，即在该级别队列中，单个任务分配到多少时间片执行，一般来说优先级越高的队列越短，优先级越低的队列越长。比如在最高级队列中可能是10ms一个时间片去执行任务，在最低级队列中可能是100ms一个时间片去执行
+ 优先队列的升级周期s，就是过多久，所有任务统一上升到最高级队列，也就是5中的概念，一般是1秒

这里补充下这个模型的理解，队列中的任务其实是进程，假设ABC在最顶级队列p0，那么操作系统就会按照ABC轮询的顺序，执行A10ms后中断切换上下文去执行B，执行B10ms后继续C。所以我们可以看到如果规则4不这么设计，那么ABC可以无限执行直到ABC都执行完。所以再加上规则4后，ABC执行一次后就移动到了下一等级的队列p1，下一等级的队列如果时间窗口是20ms，那么就会在执行完下一队列p1内原有的任务后再执行ABC，然后没执行完的话大家继续降级。等待s秒一次的重启。
