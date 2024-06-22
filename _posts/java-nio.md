---
title: 'Java中的NIO'
date: 2020-10-15T16:04:16+08:00
lastmod: 2020-10-15T16:04:16+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["java","nio"]
categories: ["技术"]
lightgallery: true
---

近日学习Netty，在看书和实践的时候对于书上只言片语的那些话不是十分懂，导致尝试写例子的时候遭遇各种不顺，比如decoder和encoder还有HttpObjectAggregator的添加顺序，研究了一番之后和大家分享一下自己的理解，希望后来人可以少走弯路。

## IO与NIO的区别
IO是Input与Output的缩写，主要意思就是输入输出，主要以及经常使用到的包括网络，文件中的IO。
传统IO完整的名称是同步阻塞IO，特点是IO过程中线程阻塞，等待IO返回，Java中普通的Sokcet通讯，都是同步阻塞IO。
NIO完整的名称是同步非阻塞IO，注意这里的区别，并不是异步。NIO需要操作系统底层支持，在Linux中以前有select和poll的机制，现在被更优秀的epoll代替，但这也产生了JavaNIO里selector关于epoll的一个Bug。

两者主要的区别和优缺点列举一下：

### IO
传统IO即SIO等待IO时阻塞线程，所以面对高并发比较无力，连接数越多，需要的线程数多，基本1请求对应1线程，但延迟较低，线程等待数据返回后只需要再等到CPU时间片就可以马上处理在Java中API简单易懂，方便使用，线程模型简单，可以轻松完成1请求1返回的对应
### NIO
通过epoll等机制，可以做到少数线程只轮询事件，轮询到事件后再交由其他线程处理，线程利用率高,面对高并发表现优秀，更少的线程数处理更多的连接，线程无需阻塞等待IO只需处理已经触发的事件,延迟稍高，虽然epoll在事件调度上比select和poll优秀了很多，但是应用程序还是需要遍历事件依次处理,在Java中API看似简单，实际坑多，需要好的线程模型支持，请求与返回之间较难1对1对应


## Java中的NIO
首先大概解释下Java的NIO中比较重要的几个概念

SocketChannel：包括ServerSocketChannel这类，可以简单的认为是代表了连接，事实上使用过程中它一般表现的也像个连接。

Selector：可以认为是一种工具，执行selector可以从连接中获取各个连接就绪好的事件，如读（Read）写（Write）就绪，开启关闭，从ServerSocketChannel中可以接收到新进连接（Accept）事件。

ByteBuffer：封装了一些字节操作方法，可以认为是一个byte数组，类似于IO中读时常定义的byte[] buffer这种，区别是方法更多，可以操作堆内内存，也可以使用堆外内存。

下面上一个简单例子，说明这3个东西具体怎么使用
```java
public static void main(String[] args) throws IOException {
        int channelId=0;
        //开启ServerSocketChannel和Selector
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        Selector selector = Selector.open();
        InetSocketAddress inetSocketAddress = new InetSocketAddress(
                InetAddress.getLocalHost(), 4700);
        //使用ServerSocketChannel绑定端口
        serverSocketChannel.socket().bind(inetSocketAddress);
        //设置非阻塞模式
        serverSocketChannel.configureBlocking(false);
        //设置非阻塞模式将selector注册到该Channel上，并设置该selector在该channel上监听Accept事件，该事件标致一个连接建立
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT).attach(
                channelId++);
        //循环遍历事件
        for (;;)  {
            //执行事件查询，去各个注册到该selector的channel上检查是否有就绪事件，该方法会阻塞，可以设置等待最长时间
            selector.select();
            //获取selector检查到的就绪事件
            Set<SelectionKey> readySelectionKey = selector
                    .selectedKeys();
            Iterator<SelectionKey> it = readySelectionKey.iterator();
            while (it.hasNext()) {
                SelectionKey selectionKey = it.next();
                //事件判断
                if (selectionKey.isAcceptable()) {// 客户请求连接
                    //从key中获取channel
                    ServerSocketChannel channel = (ServerSocketChannel) selectionKey
                            .channel();
                    //因为是Accept事件，调用方法获取具体的连接Channel，SocketChannel
                    SocketChannel socketChannel=channel.accept();
                    //同样，给具体的连接注册读写事件，直接使用同一个selector去注册
                    socketChannel.configureBlocking(false)
                            .register(
                                    selector,
                                    SelectionKey.OP_READ
                                            | SelectionKey.OP_WRITE).attach(channelId++);
                }
                if (selectionKey.isReadable()) {
                    //获取附加对象
                    int nowChannelId= (int) selectionKey.attachment();
                    //可读事件，通过key的方法获取到可读的channel
                    SocketChannel clientChannel=(SocketChannel)selectionKey.channel();
                    //创建一个大小为64字节
                    ByteBuffer receiveBuf = ByteBuffer.allocate(64);
                    //从channel中读取数据
                    clientChannel.read(receiveBuf);
                }
                if (selectionKey.isWritable()) {// 写数据
                    //同读取
                    SocketChannel clientChannel = (SocketChannel) selectionKey.channel();
                    ByteBuffer sendBuf = ByteBuffer.allocate(64);
                    String sendText = "hello,world";
                    sendBuf.put(sendText.getBytes());
                    sendBuf.flip();
                    clientChannel.write(sendBuf);
                }
                if (selectionKey.isConnectable()) {
 
                }
                //处理完后remove该事件
                it.remove();
            }
        }
    }
```

解释几个关键点：
1. 常用的有2种channel，一个ServerSocketChannel，用来绑定网络端口，获得具体链接，从他的Accept事件中获取具体的连接，另一种则可以认为是具体的某个连接，可以在其之上处理读写事件
2. selector的注册，参数为（channel，监听的事件，还有附加对象），这里附加对象可以在注册时创建一个对象附加，可以通过key去获取，一般用来标识该channel的种类或者用来持有该channel事件的具体处理对象。
3. 流程简而言之就是不断的select获取连接，再针对每个连接注册读写事件，然后如此循环处理
