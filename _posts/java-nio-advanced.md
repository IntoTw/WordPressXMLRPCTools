---
title: 'Java中的NIO进阶'
date: 2020-10-15
lastmod: 2020-10-15
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["java","nio"]
categories: ["技术"]
lightgallery: true
---

## 前言
之前一篇文章简单介绍了NIO，并附了一个简单的例子，但是自己试一下就会知道，简单的使用NIO是无法满足开发需要的，因为NIO处理的思路和日常servlet加spring中习惯的一连接一线程有很大不同。

## NIO与多线程
上篇那个例子实现了一个简单的NIO，但是实际使用中我们不可能仅仅在单线程下使用，肯定会使用多线程提高处理的效率，但是这样就会有几个难点。

## Readable和Writeable的空触发
之前的文章的代码有写到通过key去判断readable和writeable这两个事件，从而去读写，但是会出现以下情况

实际上这2个事件并不像看上去那样，在writeable之中负责写内容，以最常用的http的请求-返回场景来看，通常上返回的处理是在read处理完后写内容到channel里去，writeable里只做一个flush操作，并不是readable只负责读，writeable只负责写。

在多线程下，这两个事件的处理尤为糟糕，并不能简单的去实现Runable再套用函数进去执行，因为这两个事件本身在key被remove前，会一直触发，如果第一个执行的线程执行完之前，这个key因为没有remove被再次获取到去执行，那么可能就会产生很多问题，比如读取不到数据（已经被第一个线程读取了），channel已经关闭（第一个线程或其他线程写完后，本机或者远端关闭了连接）。这样都会影响多线程实际执行。

Readable和Writeable事件产生一般对应着待读数据的到来和待写数据的就绪，就绪后产生key提醒应用去处理。但是有2种情况下会产生实际没什么事可做的空触发。一种是有线程在处理key，但是没有处理完，key未被取消，这时每次select依旧会出现这个key，但实际此key正在处理中。另外一种就是只要连接还在，NIO会频繁触发Writeable事件，这就是为什么1中写道通常writeable只做flush，同样，Readable只要有数据待读就一直会触发，这会使做消息聚合、报文拼接的时候很难处理。

## 请求与返回的处理
NIO因为是非阻塞的原因，请求与返回并不是一一对应的，仅做服务端还好，以接收请求为主，在Readable事件处理完后写返回内容就可以了。但是如果既做请求端又做服务端的情况下，请求Write后，处理Read的早已不知道是哪个时间点的哪个进程了，此时如果请求和返回有关联，则较难一一对应上，这一点连著名的Netty框架都没有提供一些简便方式去完成，不过这也体现了他的纯粹，在dubbo中是通过CountDownLatch实现的，一般还有使用futureTask去实现异步转同步的操作。

## 事件的处理机制
通常情况下，接收到请求肯定要根据请求的内容进行不同的处理，简单来说起码要根据交易码不同分类不同的交易处理。这样的话面对复杂协议（如http），从报文处理到请求分类都有一个很复杂的逻辑和过程，如果没有合适的事件处理机制模型，会导致处理起来有太多的判断代码，难以维护。

## NIO多线程使用的一个例子
这里参照了Java大神Doug Lea的思路，提供一个多线程的例子，主要思路包括

1. 通过注册时的attach，将该channel的处理类直接关联起来，后面取出key时直接调用处理类去处理key
2. 处理类在处理时添加一个volatile成员做状态标记，该channel正在处理事件时，该标记为处理中，此时该channel的其他key处理到时都将判断该标记，如果该channel在处理中，则直接退出等待下次select出来继续判断。 
3. 使用了线程池去处理
```java
public static void main(String[] args) throws IOException {
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        Selector selector = Selector.open();
        serverSocketChannel.bind(new InetSocketAddress(8080));
        serverSocketChannel.configureBlocking(false);
        //主ServerSocket绑定Accept事件，处理类为Accept
        SelectionKey key=serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT, new Accept(serverSocketChannel, selector));
        for(;;) {
            selector.select();
            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = selectionKeys.iterator();
 
            while (iterator.hasNext()) {
                SelectionKey selectionKey = iterator.next();
                //分发处理key
                dispatchTask(selectionKey);
            }
            selectionKeys.clear();
        }
    }
 
    private static void dispatchTask(SelectionKey selectionKey) {
        //这里的attachment，针对ServerSocketChannel，是Accept，针对进来的连接SocketChannel，是TaskHandler
        Runnable runnable = (Runnable)selectionKey.attachment();
        if (runnable != null) {
            runnable.run();
        }
    }
```

```java
public class Accept implements Runnable{
    ServerSocketChannel serverSocketChannel;
    Selector selector;
    public Accept(ServerSocketChannel serverSocketChannel, Selector selector) {
        this.serverSocketChannel=serverSocketChannel;
        this.selector=selector;
    }
 
    @Override
    public void run() {
        try {
            SocketChannel socketChannel=serverSocketChannel.accept();
            //给进来的连接注册selector，这里直接通过构造函数来了
            new TaskHandler(socketChannel,selector);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
```java
public class TaskHandler implements Runnable{
    private SocketChannel socketChannel;
    private SelectionKey selectionKey;
    private volatile int state = PROCESSED;
    private static final ExecutorService pool = Executors.newFixedThreadPool(4);
 
    static final int PROCESSING=1;
    static final int PROCESSED=2;
 
    public TaskHandler(SocketChannel socketChannel, Selector selector) throws IOException {
        this.socketChannel=socketChannel;
        socketChannel.configureBlocking(false);
        //把自己attach上去
        this.selectionKey=socketChannel.register(selector,SelectionKey.OP_READ,this);
        selector.wakeup();
    }
 
    @Override
    public void run() {
        //判断处理状态
        if(state==PROCESSED)
        {
            pool.execute(new Process(selectionKey));
        }
    }
    class Process implements Runnable{
        private SelectionKey selectionKey;
 
        public Process(SelectionKey selectionKey) {
            this.selectionKey = selectionKey;
            state=PROCESSING;
        }
 
        @Override
        public void run() {
            try {
                if(socketChannel.socket().isClosed())
                    return ;
                if(selectionKey.isReadable())
                {
                    read();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
 
        private void read() throws IOException {
            ByteBuffer byteBuffer=ByteBuffer.allocate(64);
            SocketChannel socketChannel= (SocketChannel) selectionKey.channel();
            int read;
            System.out.println("readding");
            if ((read = socketChannel.read(byteBuffer)) < 0)  {
                state = PROCESSED;
                socketChannel.close();
                return;
            }
            System.out.println(new String(byteBuffer.array()));
            state=PROCESSED;
        }
    }
}
```
