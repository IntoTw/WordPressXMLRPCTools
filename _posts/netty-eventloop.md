---
title: 'Netty学习：EventLoop事件机制'
date: 2020-10-15
lastmod: 2020-10-15
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["java","netty"]
categories: ["技术"]
lightgallery: true
---

## EventLoop是什么
如果你去百度EventLoop，肯定会百度到很多关于JavaScript，NodeJS的文章，是的，这两种语言的事件机制就依赖于EventLoop，但是EventLoop到底是什么，可以先思考2个问题：
1. 一般情况下，当我们要实现令一个线程不断处理任务，都是选择使用while(true){……}这样的结构，但是往往为了防止无限循环空跑占用CPU时间，会在死循环中使用sleep()来空出时间。有更好的方法吗？
2. Redis的性能毋庸置疑，但是Redis是单线程的，单线程的Redis是如何达到不逊色于多线程的性能的呢？(当然也是EventLoop)。

EventLoop事实上是一种线程编程模型，简而言之，将需要执行的任务封装成原子化的Event，都交到一个线程去执行，而这个线程唯一的任务就是不断的从自己的Event事件池中取出来Event去执行。那么为什么叫Loop呢？因为这种模型适用于非阻塞的操作，在非阻塞Event发生后，将该Event以及回调函数封装成一个CheckEvent，继续扔到Event池中，在不断执行Event的过程中，当执行到CheckEvent时，会去检查非阻塞操作是否成功，成功则执行回调，未成功则继续扔进去等待下一次，所以这种模型叫做EventLoop=事件轮询。

## EventLoop适用的场景
或许你会说，EventLoop性能这么高，这么帅，那大家都用不就好了？事实上，EventLoop也有他局限的场景，如上所说，这种编程场景多用于该系统大多数事件可以封装成异步事件时，EventLoop会有更好的吞吐和执行效率，比如JavaScript要在js执行完后交由浏览器进行绘制这种图形客户端场景，或者redis这种使用密集IO型，或者Netty这种天生NIO框架。简而言之：异步非阻塞操作多，回调机制多。当你需要Check的事件不多，都是实际需要执行的任务时，EventLoop比起线程池，优势就微乎其微了，并且EventLoop的实现还更为复杂。所以不是什么场景都适合使用的。

## Netty中的EventLoop
Netty中使用的当然也是EventLoop，这是它的一个优点或者说提升它性能的原因。
```java
EventLoopGroup group = new NioEventLoopGroup();        
Bootstrap b = new Bootstrap();
b.group(group)
```
Netty代码肯定少不了这几句，其中NioEventLoopGroup就是Netty的EventLoop实现，一个Group中包含多个EventLoop，类似线程池和线程的关系。支持构造函数传进去里面的线程数，如果不传的话默认是CPU数*2。

### Netty中的大量inEventLoop判断
看过Netty源码的很多人肯定会看到Netty的源码中有大量的如下判断：
```java
if (!inEventLoop && !executor.inEventLoop(currentThread)) {
    executor.execute(new Runnable() {
        
    });
}
```
这个判断是为什么呢？我一开始也很疑惑，后面随着对Netty的理解，才知道，对Netty来说，Netty许多函数的调用方，它并不知道调用方是谁，是在自己的内部EventLoop内，还是在用户线程里，所以它在一些用户很可能在用户线程里调用的方法，增加了这类判断，将这些方法的执行转移到EventLoop中。

### Netty是如何建立连接并监听端口的-NIOSocketChannel
初看Netty源码，Netty是怎么建立端口监听以及连接的，都找了我许久。先上一段标准代码，这是netty源码例子中的代码。
```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();
final EchoServerHandler serverHandler = new EchoServerHandler();
try {
    ServerBootstrap b = new ServerBootstrap();
    b.group(bossGroup, workerGroup)
        .channel(NioServerSocketChannel.class)
        .option(ChannelOption.SO_BACKLOG, 100)
        .handler(new LoggingHandler(LogLevel.INFO))
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            public void initChannel(SocketChannel ch) throws Exception {
                ChannelPipeline p = ch.pipeline();
                if (sslCtx != null) {
                    p.addLast(sslCtx.newHandler(ch.alloc()));
                }
                //p.addLast(new LoggingHandler(LogLevel.INFO));
                p.addLast(serverHandler);
            }
        });

    // Start the server.
    ChannelFuture f = b.bind(PORT).sync();

    // Wait until the server socket is closed.
    f.channel().closeFuture().sync();
} finally {
    // Shut down all event loops to terminate all threads.
    bossGroup.shutdownGracefully();
    workerGroup.shutdownGracefully();
}
```
可以看到其中主要就是配置了EventLoopGroup,channel，handler，这三个东西，这三个东西就是netty三大件，总领了Netty。建立连接的就是其中的NioServerSocketChannel，这个NioSocketChannel这种是Netty包装了Java原生的SocketChannel，在启动时（server）或者进行请求时（client）通过反射进行创建并运行。

### Netty,bossGroup与workerGroup
Netty默认构造函数，支持设置2个EventLoopGroup，其中分为Boss和Worker，其中Boss只负责处理连接请求的建立，以及key的select，类似主线程，Worker负责具体的读写，handler处理等其他任务，官方建议这两个的线程数比是1:5，实际使用中可以看情况设置。
