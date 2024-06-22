---
title: 'TransmittableThreadLocal解决线程池变量传递以及原理解析'
date: 2021-05-07T00:00:00+08:00
lastmod: 2021-05-07T00:00:00+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["java","TransmittableThreadLocal"]
categories: ["技术"]
lightgallery: true
---

# TransmittableThreadLocal解决线程池变量传递以及原理解析

## 介绍

TransmittableThreadLocal是alibaba提供的一个工具包中的类，主要作用就是解决线程池场景下的变量传递问题。继承自InheritableThreadLocal，我们知道
InheritableThreadLocal解决了主线程与子线程之间的变量传递问题，但是在遇到线程池以及线程复用的情况下，就无能为力了，TransmittableThreadLocal通过对InheritableThreadLocal以及线程池的增强，解决了这个问题。

主要用途：
1. 就是应用中对于连接池中的全局链路追踪。
2. 解决例如Hystrix中出现的ThreadLocal无法传递的问题。

# 使用

## 依赖

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>transmittable-thread-local</artifactId>
    <version>2.11.2</version>
</dependency>
```

## 示例代码

```java
public class TestTTL {
    //定义一个线程池执行ttl，这里必须要用TTL线程池封装
    private static ExecutorService TTLExecutor = TtlExecutors.getTtlExecutorService(Executors.newFixedThreadPool(5));
    //定义另外一个线程池循环执行，模拟业务场景下多Http请求调用的情况
    private static ExecutorService loopExecutor = Executors.newFixedThreadPool(5);
    private static AtomicInteger i=new AtomicInteger(0);
    //TTL的ThreadLocal
    private static ThreadLocal tl = new TransmittableThreadLocal<>(); //这里采用TTL的实现
    public static void main(String[] args) {

        while (true) {
            /*
             这里就是循环执行10次，每次对数值加1并设置到threadlocal中，然后再使用TTL去执行来打印这个值。
             这里外部为什么使用线程池，是为了证明TTL确实可以达到我们想要的效果：即线程池中多任务带着
             父线程各自的ThreadLocal运行互不影响
            */
            loopExecutor.execute( () -> {
                if(i.get()<10){
                    tl.set(i.getAndAdd(1));
                    TTLExecutor.execute(() -> {
                        System.out.println(String.format("子线程名称-%s, 变量值=%s", Thread.currentThread().getName(), tl.get()));
                    });
                }
            });
        }
    }
}
```
执行结果：

![](https://images.intotw.cn/blog/2023/09/352e3f6f05bb8890a2f55a808f8fe789.jpeg)



# 原理

TransmittableThreadLocal执行的关键原理在于以下几个类做了几件事：

## TransmittableThreadLocal
TransmittableThreadLocal本身增加一个静态的holderMap，里面保存了所有使用过的TransmittableThreadLocal作为key的引用，这样在复制TransmittableThreadLocal的值到线程本身的ThreadLocal时，就可以通过该holder遍历到所有的TransmittableThreadLocal。

```java
/*
    可以看到，在TransmittableThreadLocal调用get和set方法时，都会将自己作为key放入holder，以便后续复制时遍历，
    这个holder其实就是一个储存全局TransmittableThreadLocal的集合，不过他不是通过手动add的，
    而是通过耦合到TransmittableThreadLocal的方法中自动的去增加。
*/
public final void set(T value) {
        super.set(value);
        if (null == value) {
            this.removeValue();
        } else {
            this.addValue();
        }
}
private void addValue() {
        if (!((WeakHashMap)holder.get()).containsKey(this)) {
            ((WeakHashMap)holder.get()).put(this, (Object)null);
        }

}
```


## Transmitter和Snapshot
这两个都是TransmittableThreadLocal的内部类，前者主要是一些工具方法，后者包含了2个map，ttl2Value和threadLocal2Value，分别存储ttl设置的ThreadLocal值和父线程中其他的ThreadLocal值。其中Snapshot是一个快照，用作备份还原现场使用。

```java
//Transmitter
/*
    可以看到这里就是通过holder来遍历所有TTLocal，以此来复制值到下面的Runable中。
*/
 private static WeakHashMap<TransmittableThreadLocal<Object>, Object> captureTtlValues() {
            WeakHashMap<TransmittableThreadLocal<Object>, Object> ttl2Value = new WeakHashMap();
            Iterator var1 = ((WeakHashMap)TransmittableThreadLocal.holder.get()).keySet().iterator();

            while(var1.hasNext()) {
                TransmittableThreadLocal<Object> threadLocal = (TransmittableThreadLocal)var1.next();
                ttl2Value.put(threadLocal, threadLocal.copyValue());
            }

            return ttl2Value;
}
//这个方法会复制一个快照返回，实际调用会将调用线程的2类ThreadLocal值复制为一个快照给Runable使用
@NonNull
public static Object capture() {
    return new TransmittableThreadLocal.Transmitter.Snapshot(captureTtlValues(), captureThreadLocalValues());
}

//方法名就是重放，就是将快照的值，copy到执行线程中。
@NonNull
public static Object replay(@NonNull Object captured) {
    TransmittableThreadLocal.Transmitter.Snapshot capturedSnapshot = (TransmittableThreadLocal.Transmitter.Snapshot)captured;
    return new TransmittableThreadLocal.Transmitter.Snapshot(replayTtlValues(capturedSnapshot.ttl2Value), replayThreadLocalValues(capturedSnapshot.threadLocal2Value));
}
//还原现场，在执行实际runnable之后，将执行前的备份，copy回执行线程
public static void restore(@NonNull Object backup) {
        TransmittableThreadLocal.Transmitter.Snapshot backupSnapshot = (TransmittableThreadLocal.Transmitter.Snapshot)backup;
        restoreTtlValues(backupSnapshot.ttl2Value);
        restoreThreadLocalValues(backupSnapshot.threadLocal2Value);
    }
```

```java
//一个快照类，保存ttl和原生threadlocal的值
private static class Snapshot {
        final WeakHashMap<TransmittableThreadLocal<Object>, Object> ttl2Value;
        final WeakHashMap<ThreadLocal<Object>, Object> threadLocal2Value;

        private Snapshot(WeakHashMap<TransmittableThreadLocal<Object>, Object> ttl2Value, WeakHashMap<ThreadLocal<Object>, Object> threadLocal2Value) {
            this.ttl2Value = ttl2Value;
            this.threadLocal2Value = threadLocal2Value;
        }
}
```

## TtlRunnable

这个类就是前面使用TTL封装线程池的意义，TTL封装线程池，重写了其中的sumbit和execute等方法，使得提交到线程池中的实际Runable是封装过后的TtlRunnable，并且该类完成了复制ThreadLocal值和还原现场等操作。

```java
// ExecutorServiceTtlWrapper，封装线程池
/*
    这里面的TtlCallable.get就会return一个new的TtlRunnable
*/
@NonNull
    public <T> Future<T> submit(@NonNull Callable<T> task) {
        return this.executorService.submit(TtlCallable.get(task));
    }

    @NonNull
    public <T> Future<T> submit(@NonNull Runnable task, T result) {
        return this.executorService.submit(TtlRunnable.get(task), result);
    }

    @NonNull
    public Future<?> submit(@NonNull Runnable task) {
        return this.executorService.submit(TtlRunnable.get(task));
    }
```


```java
// TtlRunnable，封装线程池
public final class TtlRunnable implements Runnable, TtlEnhanced, TtlAttachments {
    /* 
        这个变量是最核心的变量，在初始化时，会调用Transmitter，
        将父线程或者说调用线程的TTLThreadLocal和原生ThreadLocal中的值copy出一个快照，
        即上面的SnapShot，并且在这里持有那个SnapShot的引用，
        注意，这里的构造函数初始化，是在调用线程里执行的，所以拿到的就是调用线程的ThreadLocal值。
    */
    private final AtomicReference<Object> capturedRef = new AtomicReference(Transmitter.capture());
    //其他代码省略

    //实际执行函数
    public void run() {
        //获取上面的那个快照值，此时已经在线程池中执行了
        Object captured = this.capturedRef.get();
        if (captured != null && (!this.releaseTtlValueReferenceAfterRun || this.capturedRef.compareAndSet(captured, (Object)null))) {
            //调用工具类，此时我们有了快照，并且已经在当前线程池中执行了，所以把快照的值全部赋值到当前线程池中的ThreadLocal中去，下一步被包装的Runable执行run时，就是无感的了。
            Object backup = Transmitter.replay(captured);

            try {
                this.runnable.run();
            } finally {
                //执行完成后，我们需要将该线程的ThreadLocal还原回之前的快照，因为在线程池中，线程可能复用，为了防止在runnable执行中对该线程的ThreadLocal产生了污染，然后该线程被复用去执行其他Runable时该值已被修改，不再是调用线程的值了，所以需要还原现场。
                Transmitter.restore(backup);
            }

        } else {
            throw new IllegalStateException("TTL value reference is released after run!");
        }
    }
}
```

用两张图可以更加详细的说明执行的过程。我们给前面的两个线程池中的线程分别命个名，然后debug代码。

我们debug到TtlRunnable初始化，可以看到此时初始化的线程是loopExecutor，也就是调用线程，理所当然此时可以制作一个调用线程的ThreadLocal快照。

![](https://images.intotw.cn/blog/2023/09/5df52758ba1ef09de391a67900963353.jpeg)



然后我们debug到runnable执行时走到的replay方法，此时执行的线程就是TTL线程，也就是线程池中的线程了，此时我们当然也可以将之前保存的快照，来赋值到线程池中该线程了，此后的还原也是同理。

![](https://images.intotw.cn/blog/2023/09/d99ac0dc3175f3e317dfbaf3fce31c2b.jpeg)



# 总结

该工具类解决了特定场景下的需求，实现方式核心就是封装+快照。非常值得学习。不过在实际中也不建议在ThreadLocal值过多或者较大时频繁使用，因为会产生过多的SnapShot临时对象增加gc负担，并且每次线程执行都会带来更多的copy和还原负担。
