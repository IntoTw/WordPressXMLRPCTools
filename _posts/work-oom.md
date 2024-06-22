---
title: '工作记录：记一次线上内存泄露问题的排查'
date: 2020-10-14T00:00:00+08:00
lastmod: 2020-10-14T00:00:00+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["内存","线上问题","工作记录"]
categories: ["技术"]
lightgallery: true
---

## 问题的发现
发现当然还是运维大哥因为发现告警，包括自己邮箱也一堆告警，然后运维大哥做了dump以及jstack后立马重启，重启后暂时解决。
## 问题的排查
有dump和jstack记录，当然是好分析的，先分析这两个，原因就比较明显了：
1. dump记录拉到本地用java自带的工具查看，发现大量netty的MpscArrayQueue对象没有释放，占用了大量内存，罪魁祸首就是这个，再看上级对象，是redisson使用到了netty，所以初步确定是redisson的问题。
2. 查看jstack，发现大量RUNNABLE线程，定位线程，发现是该定时任务模块自定义的线程池执行自定义的线程，那么为什么会把线程跑满呢？后面看了一下代码:
```java
  <bean id="threadPoolExecutor" class="com.bwton.core.threadpool.SimpleThreadPoolExecutorFactoryBean">
        <property name="corePoolSize" value="100"/>
        <property name="maximumPoolSize" value="200"/>
        <property name="keepAliveTime" value="3000"/>
        <property name="workQueueCapacity" value="10000"/>
        <property name="rejectedExecutionHandler" ref="rejectedExecutionHandler"/>
        <property name="threadPackageScan" value="com.bwton.v2.thread"/>
    </bean>
```
```java
public class SimpleRejectedExecutionHandler implements RejectedExecutionHandler {

    private static Logger logger = LoggerFactory.getLogger(SimpleRejectedExecutionHandler.class);

    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        // 记录异常、报警处理等
        logger.warn("【simple线程池】等待队列已满，正尝试以阻塞方式提交线程到等待队列.....");
        // 由blockingqueue的offer改成put阻塞方法
        try {
            executor.getQueue().put(r);
        } catch (InterruptedException e) {
            logger.warn("【simple线程池】等待队列已满，以阻塞方式提交线程到等待队列时被打断，fail!");
            e.printStackTrace();
        }
        logger.warn("【simple线程池】线程已加入等待队列，success!");
    }
}
```

这里封装了一个线程池，主要就是原生线程池加一个任务执行队列，出问题的主要就是这个线程池的拒绝策略，如果下游出现大量超时，新任务执行时，线程池会触发拒绝策略，使用put尝试继续放到该线程池的执行队列中，将调用的线程阻塞。**又因为这个模块是定时任务**，每分钟会触发，查出200条待补偿的数据扔到线程池中补偿，所以是会不断产生任务的，最后导致大量线程在跑满，另外大量线程在阻塞等待放进队列，**实际的执行逻辑是有很多Redisson操作的**，最后导致Redisson资源释放不掉，导致GC无法回收任何内存，最后OOM，这里据查Redisson官方Issue，之所以Redisson没有释放掉资源，也是和Redisson的一个bug有关，redisson在3.12.4的版本之前，在IdleConnectionTimeout该参数过小且负载过高时，redisson的线程池会无法得到有效的回收，从而导致oom，后续版本中也升级了Redisson的版本，不过触发原因是因为这个线程池的错误逻辑。

后续修改了这个线程池的逻辑，并且增大了队列的大小，把补偿的业务逻辑也做成了异步的，就再也没有出现过这个问题了。


## 总结
最好还是不要自己造轮子写线程池这些，可以写作练手或者学习，但是不要上生产，实在不得已，也要经过充分的测试。
dump和jstack堆栈信息，是排查java问题的第一帮手。
