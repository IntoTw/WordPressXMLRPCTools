---
title: 'ostep学习笔记-9 Scheduling Proportional Share'
date: 2024-03-12T14:42:34+08:00
lastmod: 2024-03-12T14:42:34+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["操作系统"]
categories: ["ostep"]
lightgallery: true
math: true
---

除了第八章之外，第九章还介绍了其他一种种类的调度算法，包括Linux采用的算法CFS，介绍的过程也循序渐进

这类算法的特点是**按比例共享**，也就是Proportional Share的意思。这类算法主要考虑的是权重以及分配时间片的比例，最后我会稍微总结一下自己思考的和与第8章中算法的一些核心区别和为什么需要考虑这些。

## Lottery Scheduling（抽奖算法）

最基础的算法是Lottery Scheduling，抽奖算法。

抽奖算法基于ticket的概念，所有进程持有一定数量的ticket，调度程序通过随机数来随机出一个ticket数来决定哪个进程被调度，这比较简单浅显的实现了比例的概念，因为持有票数多的更容易被随机到。一个简单的例子就是A持有100票，B持有100票，调度程序产生一个【0-199】的随机数，当随机数=0-99时运行A，100-199时运行B。

抽奖算法还有几个其他的概念，这里简单的介绍一下。

一个是所谓的票据在用户维度的比例缩放，这里直接放下原文吧：

> For example, assume users A and B have each been given 100 tickets.User A is running two jobs, A1 and A2, and gives them each 500 tickets(out of 1000 total) in A’s currency. User B is running only 1 job and givesit 10 tickets (out of 10 total). The system converts A1’s and A2’s allocationfrom 500 each in A’s currency to 50 each in the global currency; similarly,B1’s 10 tickets is converted to 100 tickets. The lottery is then held over theglobal ticket currency (200 total) to determine which job runs.
>
> 例如，假设用户A和用户B分别获得了100张抽奖。用户A正在运行两个作业，A1和A2，并给予它们每个500张抽奖（总共1000张）。用户B只运行一个作业，并给予它10张抽奖（总共10张）。系统将A1和A2的分配从A的货币中的每个500张转换为全局货币中的每个50张；同样地，B1的10张票被转换为100张票。然后，使用全局抽奖货币（总共200张）进行抽奖来确定哪个作业运行。

另一个是票据的移交，在进程自身不需要执行时，可以主动将票据交给其他进程，其他进程忙碌完以后，再将票据归还

最后就是票据膨胀，简而言之就是进程自己多要一些票据提高自己的优先级

抽奖调度适合在进程互信的场景下，但是好像也并没有什么明显的优势。唯一的优点就是实现很简单：

+ 将所有进程使用列表排列
+ 根据总票数随机一个数值
+ 以此遍历每个进程，并再迭代过程中累加遍历过的进程持有的票数
+ 迭代时，如果当前迭代的节点持有票数+累加的票数<随机的票数，该节点获得执行权

## 步进调度

这个算法包括后面的CFS，都有和之前总结的负载均衡算法有异曲同工之妙，甚至是相同的：[分布式系统：负载均衡算法](https://intotw.cn/posts/load-balance/)

简而言之就是初始给每个进程一个初始步长，然后每个调度周期取出步长最小的进程去执行，然后每个进程根据权重增加一定数量的步长

这么看来，这种负载均衡算法实在是既易于实现又经典，牛啊

## CFS（Completely Fair Scheduler )

CFS是Linux使用的调度策略，特点也和名字一样，公平（尽可能）的调度是它的目标

CFS的执行步骤其实很简单，重复2步

+ 给每个进程一个vruntime，虚拟运行时间
+ 每次窗口选择一个vruntime最短的进程去执行，执行后累加vruntime，累加值一般为实际执行的物理时间乘以一定比例

比较复杂的是其中的一些情况带来的几个参数：

+ sched_latency：该参数为了解决一个问题，“一次进程执行该设计成多久？”，如果每次执行太短，那么进程上下文切换就占用了太多时间。如果每次执行太久，那么在短期内的公平性就会降低的非常离谱。所以这个参数设定了每次执行的窗口，也就是过多久操作系统会通过定时中断来执行上述的调度逻辑选取其他进程执行，在有多个cpu的情况这个值会除以CPU数，一般在linux上这个值是48ms
+ min_granularity：这个参数描述了最小执行时间，因为只依赖上述参数，在过多时，例如cpu>48（极端情况），会将调度延迟改为1ms及以下，这个参数在单cpu上就失去了意义，所以也为调度延迟设置了下限，即实际一般min_granularity<调度延迟<sched_latency，这个值在linux一般是6ms

另外，CFS使用时钟中断来获取或者说夺回cpu控制权，也就是说可以认为上面那些参数组合计算出的time_slice就是CFS时钟中断的间隔

### 权重

CFS除了相对公平的默认实现意外，还支持权重的配置，默认进程的权重为0，权重参与计算的值weight由一个数组维护

```c
static const int prio_to_weight[40] = {
/*-20*/ 88761, 71755, 56483, 46273, 36291,
/*-15*/ 29154, 23254, 18705, 14949, 11916,
/*-10*/  9548,  7620,  6100,  4904,  3906,
/*-5*/  3121,  2501,  1991,  1586,  1277,
/*0*/  1024,   820,   655,   526,   423,
/*5*/   335,   272,   215,   172,   137,
/*10*/   110,    87,    70,    56,    45,
/*15*/    36,    29,    23,    18,    15,};
```



权重影响实际时间片的公式如下：

![](https://images.intotw.cn/blog/2024/03/ff29315aa2ee0200b447a26278a84c37.png)

看原文看了半天一直没有理解分式下面的weight i累加到底是什么意思，查了下其他人写的关于CFS的文章，下面实际是系统内所有进程权重的累加值，比如原文中说的A的权重是-5，对应的就是3121，B的权重是0，对应的1024，这个比例对A来说就是: $\cfrac  {3121}{3121+1024}≈\cfrac  {3}{4}$，B就是$\cfrac  {1}{4}$。

除了该进程根据权重时的时间片需要更换计算方式以外，虚拟执行时间的累加也要根据权重计算，才能保证整体平衡。虚拟执行时间的累加公式如下：

![](https://images.intotw.cn/blog/2024/03/a6d32f16cda26ffc03024f69c39e69a5.png)

简而言之，就是根据原有应该的执行时间累加值*$\cfrac  {1024}{该进程的权重weight}$，比如对A来说，这个累加值就是大约$\cfrac  {1024}{3121}≈\cfrac  {1}{3}$

### 红黑树

这章还简单介绍了下红黑树，CFS因为每次调度周期要挑选出vruntime最小的进行执行，所以对有序集合的性能要求较高，红黑树作为平衡二叉树在查询和插入的综合性能上可以认为是最适合这个场景的数据结构了，不过需要注意的是，CFS的红黑树中只包含了那些running状态的进程，也就是那些正在运行的，因为中断或者sleep进入ready或者waiting

### I/O以及sleep的处理

CFS将

对于那些因为io或者中断进入sleep的进程，操作系统在他们被唤醒时会采取当前队列中最低的vruntime作为他们的新vruntime再把他们加回那个维护running状态的红黑树，来保证尽可能补偿回他们因为sleep而失去的cpu时间。这样做有好有坏，虽然这样避免了那些经常sleep的进程使得其他进程饿死（因为如果某进程sleep10s，那么当该进程被唤醒后，很可能他将独占10s的cpu），但是一定程度破坏了公平性

## Others

还有一些中间看时遇到的问题，其中一个对于公式的不理解上面已经写过了，还有一些理解自己查阅资料以及思考后，觉得比较有价值的问题和结论

> Q：书中最后提到了，CFS被Linux和大部分datacenter的操作系统采用作为基础的调度算法，而上一张介绍的MLFQ被BSD Unix，Windows（查一下的话发现还有Mac）作为基本的调度算法，为什么呢？
>
> A：
>
> ​	对比下文中不算强调但是有这方面描述的两个算法的特点，以及两个算法的特性来看，其实这个问题不难得到答案。MLFQ可以通过对进入I/O等响应用户交互的进程通过提高优先队列级别等方式来单独保证用户交互的response_time（比如提到过的一个对于那些经常主动放弃CPU的进程，不降低他们的优先队列级别的策略），所以MLFQ大多在具备图形交互等操作系统中作为基本的逻辑，这也是为什么Windows和Mac都是基于这个调度。
>
> ​	而CFS则集中在CPU使用的公平性，在Linux这种主要以终端操作为主（用户界面交互较弱），或者datacenter这种以计算为主的背景下，更在意的就是计算任务的公平性和CPU时间分配了

> 作为操作系统调度任务的核心算法，MLFQ和CFS都兼具简单和强大两个优点。
>
> Simplicity is prerequisite for reliability 
>
> ​															-**Edsger Dijkstra** 
>
> 















