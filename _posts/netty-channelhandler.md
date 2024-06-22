---
title: 'Netty学习：ChannelHandler执行顺序详解，附源码分析'
date: 2020-10-14
lastmod: 2020-10-14
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["java","netty"]
categories: ["技术"]
lightgallery: true
---

近日学习Netty，在看书和实践的时候对于书上只言片语的那些话不是十分懂，导致尝试写例子的时候遭遇各种不顺，比如decoder和encoder还有HttpObjectAggregator的添加顺序，研究了一番之后和大家分享一下自己的理解，希望后来人可以少走弯路。

## 模型浅析
简单描述下ChannelHandler的存储模型，ChannelHandler在ChannelPipeline中主要以AbstractChannelHandlerContext为基类存储，存储的数据结构为链表，传进去的ChannelHandler都会转化为DefaultChannelHandlerContext来存储在ChannelPipeline里，ChannelPipeline主要的实现为DefaultChannelPipeline。

DefaultChannelPipeline
DefaultChannelPipeline使用双向链表储存所有AbstractChannelHandlerContext，定义如下：
```java
public class DefaultChannelPipeline implements ChannelPipeline {
    static final InternalLogger logger = InternalLoggerFactory.getInstance(DefaultChannelPipeline.class);
    private static final String HEAD_NAME = generateName0(DefaultChannelPipeline.HeadContext.class);
    private static final String TAIL_NAME = generateName0(DefaultChannelPipeline.TailContext.class);
    private static final FastThreadLocal<Map<Class<?>, String>> nameCaches = new FastThreadLocal<Map<Class<?>, String>>() {
        protected Map<Class<?>, String> initialValue() throws Exception {
            return new WeakHashMap();
        }
    };
    final AbstractChannelHandlerContext head;//双向链表，头指针
    final AbstractChannelHandlerContext tail;//双向链表，尾指针
    private final Channel channel;
    private final ChannelFuture succeededFuture;
    private final VoidChannelPromise voidPromise;
    private final boolean touch = ResourceLeakDetector.isEnabled();
    private Map<EventExecutorGroup, EventExecutor> childExecutors;
    private Handle estimatorHandle;
    private boolean firstRegistration = true;
    private DefaultChannelPipeline.PendingHandlerCallback pendingHandlerCallbackHead;
    private boolean registered;
    ……
}
```
在调用addLast(First,Before）等方法添加ChannelHandler到ChannelPipeline时，实际上是new了一个DefaultChannelHandlerContext对象插入到链表中：
```java
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
        final AbstractChannelHandlerContext newCtx;
        synchronized(this) {
            checkMultiplicity(handler);
            newCtx = this.newContext(group, this.filterName(name, handler), handler);//第一个参数为eventgroup，第二个参数为通过方法获取的channelhandler名称，第三个为channelhandler
            this.addLast0(newCtx);//构造完成后，last使用前插法插入链表尾部，first使用后插法插入链表头部
            if(!this.registered) {
                newCtx.setAddPending();
                this.callHandlerCallbackLater(newCtx, true);
                return this;
            }
 
            EventExecutor executor = newCtx.executor();
            if(!executor.inEventLoop()) {
                newCtx.setAddPending();
                executor.execute(new Runnable() {
                    public void run() {
                        DefaultChannelPipeline.this.callHandlerAdded0(newCtx);
                    }
                });
                return this;
            }
        }
 
        this.callHandlerAdded0(newCtx);
        return this;
    }
```

### DefaultChannelHandlerContext
```java
final class DefaultChannelHandlerContext extends AbstractChannelHandlerContext {
    private final ChannelHandler handler;
 
    DefaultChannelHandlerContext(DefaultChannelPipeline pipeline, EventExecutor executor, String name, ChannelHandler handler) {
        super(pipeline, executor, name, isInbound(handler), isOutbound(handler));//调用父类构造方法创建context
        if(handler == null) {
            throw new NullPointerException("handler");
        } else {
            this.handler = handler;//保存handler引用
        }
    }
 
    public ChannelHandler handler() {
        return this.handler;
    }
 
    private static boolean isInbound(ChannelHandler handler) {
        return handler instanceof ChannelInboundHandler;
    }
 
    private static boolean isOutbound(ChannelHandler handler) {
        return handler instanceof ChannelOutboundHandler;
    }
}
```
## Handler传递顺序
现在我们知道Pipeline里实际是一个context的链表，现在我们来看看fireChannelRead和write的传递顺序

### fireChannelRead
调用fireChannelRead方法时，调用该方法的context会从自己开始在链表中根据自己的next指针来寻找下一个注册（invoke）的handler去处理事件，代码如下：
```java
public ChannelHandlerContext fireChannelRead(Object msg) {
        invokeChannelRead(this.findContextInbound(), msg);//将查找到的context传递进入执行channelRead方法，里面还有一些eventLoop的判断
        return this;
    }
private AbstractChannelHandlerContext findContextInbound() {
        AbstractChannelHandlerContext ctx = this;
 
        do {
            ctx = ctx.next;
        } while(!ctx.inbound);//从当前context开始，查找到下一个为inbound的handler，所以说outbound和inbound的插入顺序与执行顺序或执行成功与否没有任何关系，只与最后链表的结果有关，并且当handler过多时会影响遍历速度
 
        return ctx;
    }
```
### write
调用write方法时，调用该方法的context会从**自己开始**在链表中根据自己的pre指针来寻找上一个注册（invoke）的handler去处理事件，顺序与fireChannelRead相反,代码如下：
```java
private void write(Object msg, boolean flush, ChannelPromise promise) {
        AbstractChannelHandlerContext next = this.findContextOutbound();//查找到上一个outBoundContext
        Object m = this.pipeline.touch(msg, next);
        EventExecutor executor = next.executor();
        if(executor.inEventLoop()) {
            if(flush) {
                next.invokeWriteAndFlush(m, promise);
            } else {
                next.invokeWrite(m, promise);
            }
        } else {
            Object task;
            if(flush) {
                task = AbstractChannelHandlerContext.WriteAndFlushTask.newInstance(next, m, promise);
            } else {
                task = AbstractChannelHandlerContext.WriteTask.newInstance(next, m, promise);
            }
 
            safeExecute(executor, (Runnable)task, promise, m);
        }
 
    }
private AbstractChannelHandlerContext findContextOutbound() {
        AbstractChannelHandlerContext ctx = this;
 
        do {
            ctx = ctx.prev;
        } while(!ctx.outbound);
 
        return ctx;
    }
```
## 补充
如果write不调用context的write方法，而是调用context.channel().write()，则会直接调用使用pipeline的tail指针开始向前遍历outboundhandler执行，如果有特殊执行需求时可以考虑使用这种调用方法。相关源码就不贴了，有兴趣的小伙伴可以自己去看。
## 总结
到这里，Handler执行顺序已经介绍完毕了，总结为：

1. 对于channelInboundHandler,总是会从传递事件的开始，向链表末尾方向遍历执行可用的inboundHandler。

2. 对于channelOutboundHandler，总是会从write事件执行的开始，向链表头部方向遍历执行可用的outboundHandler。

举例说明如下代码：
```java
ch.pipeline().addLast(new OutboundHandler1());  
ch.pipeline().addLast(new OutboundHandler2());  
ch.pipeline().addLast(new InboundHandler1());  
ch.pipeline().addLast(new InboundHandler2());  
```
链表中的顺序为head->out1->out2->in1->in2->tail
那么Inbound的执行顺序为read->in1->in2
在Inbound执行write后，outbound执行顺序为out1<-out2<-write
1. 所以实际使用中，如果添加的顺序不好，很可能会意外跳过某些inbount或者outbound。建议实际使用上，先通过addFirst插入所有outBound再通过addLast插入所有inBound这样inBound与outBound的插入顺序与执行顺序完全一致，且不会出现跳过的情况。
**很多源码中的习惯都是只使用addLast或者addFirst插入，然后顺序在心中，具体方法见仁见智，保证顺序不错就行**
2. 所以一些统一编码解码的handler，例如ssl，httpcodec，最好是按照顺序放在链表头！这样才会保证进出都会执行到并且业务逻辑可以正常插入
