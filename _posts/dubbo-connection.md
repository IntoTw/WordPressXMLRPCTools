---
title: '分布式系统：dubbo的连接机制'
date:  2020-10-15T16:04:16+08:00
lastmod:  2020-10-15T16:04:16+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["dubbo","分布式"]
categories: ["技术"]
lightgallery: true
---



## 研究这个问题的起因
起因是一次面试，一次面试某电商网站，前面问到缓存，分布式，业务这些，还相谈甚欢。然后面试官突然甩出一句：“了解dubbo吗？dubbo是长连接还是短连接？”。当时我主要接触了解学习的还是spring cloud，dubbo作为知名的分布式rpc框架，只是有一定了解，并且连接这一块并没有很深入去了解，但是基于对分布式系统的了解，我不假思索的回答了：“长连接啊！”。其实分布式系统接触多了就知道了，分布式系统为了应对高并发，减少在高并发时的线程上下文切换损失以及重新建立连接的损失，往往都是采用长连接的。所以我当时我是这么想的：“dubbo作为处理小数据高并发分布式RPC框架，如果采用短连接，应该不可能达到那么高的吞吐吧。”。所以果断回答了长连接。可是没想到**面试官微微一笑，带着几分不屑**的说道：“短连接”。当时就给我整懵逼了，无法想象短连接如何处理高并发下重复建立连接以及线程上下文切换的问题。导致我回家的地铁上一直都处在怀疑人生的状态，回家后立马各种百度Google（甚至还怀疑查到的结果）。

## dubbo的连接机制
这里直接上结论了，**dubbo默认是使用单一长连接**，即消费者与每个服务提供者建立一个单一长连接，即如果有消费者soa-user1，soa-user2,提供者soa-account三台，则每台消费者user都会与3台account建立一个连接，结果是每台消费者user有3个长连接到分别到3台提供者，每台提供者account维持到soa-user1和soa-user2的2个长连接。
### 为什么这么做
dubbo这么设计的原因是，一般情况下因为消费者是在请求链路的上游，消费者承担的连接数以及并发量都是最高的，他需要承担更多其他的连接请求，而对提供者来说，承担的连接只来于消费者，所以每台提供者只需要承接消费者数量的连接就可以了，dubbo面向的就是消费者数量远大于服务提供者的情况。**所以说，现在很多项目使用的都是消费者和提供者不分的情况**，这种情况并没有很好的利用这个机制。

## dubbo同步转异步
dubbo的底层是使用netty，netty之前介绍过是非阻塞的，但是dubbo调用我们大多数时候都是使用的同步调用，那么这里是怎么异步转同步的呢？这里其实延伸下，不只是dubbo，大多数在web场景下，还是同步请求为主，那么netty中要如何将异步转同步？我这边描述一下关键步骤。
### dubbo的实现
```java
 //DubboInvoker
protected Result doInvoke(final Invocation invocation) throws Throwable {
      
          //...
            if (isOneway) {//2.异步没返回值
                boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
                currentClient.send(inv, isSent);
                RpcContext.getContext().setFuture(null);
                return new RpcResult();
            } else if (isAsync) {
                //1.异步有返回值，异步的直接返回带future的result就完事了
                ResponseFuture future = currentClient.request(inv, timeout);
                FutureAdapter<Object> futureAdapter = new FutureAdapter<>(future);
                RpcContext.getContext().setFuture(futureAdapter);
                Result result;
                if (isAsyncFuture) {
                    result = new AsyncRpcResult(futureAdapter, futureAdapter.getResultFuture(), false);
                } else {
                    result = new SimpleAsyncRpcResult(futureAdapter, futureAdapter.getResultFuture(), false);
                }
                return result;
            } else {//3.异步变同步，这里是同步的返回，主要阻塞的原因在于.get(),实际上就是HeaderExchangeChannel里返回的DefaultFuture的.get()方法
                RpcContext.getContext().setFuture(null);
                return (Result) currentClient.request(inv, timeout)//返回下面的future
                                            .get();//进入get()方法，是当前线程阻塞。那么当有结果返回时，唤醒这个线程
            }
        }
```
```java
//--HeaderExchangeChannel
 public ResponseFuture request(Object request, int timeout) throws RemotingException {
        Request req = new Request();
        req.setVersion(Version.getProtocolVersion());
        req.setTwoWay(true);
        req.setData(request);
        //这里在发送前，构建自定义的future，用来让调用线程等待，注意这里的future和netty的channelFuture不同。
        DefaultFuture future = DefaultFuture.newFuture(channel, req, timeout);
        try {
            channel.send(req);//使用实际的channel，里面封装了netty的channel，最后是调用到了nettyChannel的send的。
        } catch (RemotingException e) {
            future.cancel();
            throw e;
        }
        return future;
    }
```
```java
//--DefaultFuture--实现阻塞调用线程的逻辑，接收到结果
//阻塞的逻辑
 public Object get(int timeout) throws RemotingException {
        if (timeout <= 0) {
            timeout = 1000;
        }
        //done是自身对象的一个可重入锁成员变量的一个condition，这里的逻辑就是：
        //如果获取到了锁，并且条件不满足，则await线程等到下面receive方法唤醒。
        //其实我想吐槽下，这个condition命名为done，又有一个方法叫isDone，但是isDone又是判断response！=null的和done没有任何关系，这个命名不是很科学。
        if (!this.isDone()) {
            long start = System.currentTimeMillis();
            this.lock.lock();

            try {
                while(!this.isDone()) {
                    this.done.await((long)timeout, TimeUnit.MILLISECONDS);
                    if (this.isDone() || System.currentTimeMillis() - start > (long)timeout) {
                        break;
                    }
                }
            } catch (InterruptedException var8) {
                throw new RuntimeException(var8);
            } finally {
                this.lock.unlock();
            }

            if (!this.isDone()) {
                throw new TimeoutException(this.sent > 0L, this.channel, this.getTimeoutMessage(false));
            }
        }

        return this.returnFromResponse();
    }
//收到结果时唤醒的逻辑
  public static void received(Channel channel, Response response) {
        try {
            DefaultFuture future = FUTURES.remove(response.getId());
            if (future != null) {
                future.doReceived(response);
            } else {
            }
        } finally {
            CHANNELS.remove(response.getId());
        }
    }
  private void doReceived(Response res) {
        lock.lock();
        try {
            response = res;//拿到了响应
            if (done != null) {
                done.signal();//唤醒线程
            }
        } finally {
            lock.unlock();
        }
        if (callback != null) {
            invokeCallback(callback);
        }
    }
```
```java
//那么我们知道DefaultFuture被调用received方法时会被唤醒，那么是什么时候被调用的呢？
//--HeaderExchangeHandler-- netty中处理的流就是handler流，之前有篇文章讲到过，这里也是在handler中给处理的，其实上面还有ExchangeHandlerDispatcher这类dispatcher预处理，将返回
//分给具体的channelHandler处理，但是结果到了这里
public class HeaderExchangeHandler implements ChannelHandlerDelegate {
        public void received(Channel channel, Object message) throws RemotingException {
                channel.setAttribute(KEY_READ_TIMESTAMP, System.currentTimeMillis());
                HeaderExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);

                try {
                if (message instanceof Request) {
                        Request request = (Request)message;
                        if (request.isEvent()) {
                        this.handlerEvent(channel, request);
                        } else if (request.isTwoWay()) {
                        Response response = this.handleRequest(exchangeChannel, request);
                        channel.send(response);
                        } else {
                        this.handler.received(exchangeChannel, request.getData());
                        }
                } else if (message instanceof Response) {
                        //主要是这里
                        handleResponse(channel, (Response)message);
                } else if (message instanceof String) {
                        if (isClientSide(channel)) {
                        Exception e = new Exception("Dubbo client can not supported string message: " + message + " in channel: " + channel + ", url: " + channel.getUrl());
                        logger.error(e.getMessage(), e);
                        } else {
                        String echo = this.handler.telnet(channel, (String)message);
                        if (echo != null && echo.length() > 0) {
                                channel.send(echo);
                        }
                        }
                } else {
                        this.handler.received(exchangeChannel, message);
                }
                } finally {
                HeaderExchangeChannel.removeChannelIfDisconnected(channel);
                }

    }
    //handleResponse，到这里直接调用静态方法，回到了上面接受结果那步。
    static void handleResponse(Channel channel, Response response) throws RemotingException {
        if (response != null && !response.isHeartbeat()) {
            DefaultFuture.received(channel, response);
        }

    }
}
```
### 纯netty的简单实现
纯netty的简单实现，其实也很简单，在创建handler时，构造时将外部的FutureTask对象构造到hanlder中，外面使用FutureTask对象get方法阻塞，handler中在最后有结果时，将FutureTask的结果set一下，外部就取消了阻塞。
```java
    public SettableTask<String> sendAndSync(FullHttpRequest httpContent){
        //创建一个futureStask
        final SettableTask<String> responseFuture = new SettableTask<>();
        ChannelFutureListener connectionListener = future -> {
            if (future.isSuccess()) {
                Channel channel = future.channel();
                //创建一个listener，在连接后将新的futureTask构造到一个handler中
                channel.pipeline().addLast(new SpidersResultHandler(responseFuture));
            } else {
                responseFuture.setExceptionResult(future.cause());
            }
        };
        try {
            Channel channel = channelPool.acquire().syncUninterruptibly().getNow();
            log.info("channel status:{}",channel.isActive());
            channel.writeAndFlush(httpContent).addListener(connectionListener);
        } catch (Exception e) {
            log.error("netty写入异常!", e);
        }
        return responseFuture;
    }
    //重写一个可以手动set的futureTask
    public class SettableTask<T> extends FutureTask<T> {
        public SettableTask() {
            super(() -> {
                throw new IllegalStateException("Should never be called");
            });
        }

        public void setResultValue(T value) {
            this.set(value);

        }

        public void setExceptionResult(Throwable exception) {
            this.setException(exception);
        }

        @Override
        protected void done() {
            super.done();
        }
    }
    //resultHandler
    public class SpidersResultHandler extends SimpleChannelInboundHandler<String> {
        private SettableTask<String> future;
        public SpidersResultHandler(SettableTask<String> future){
            this.future=future;
        }
        @Override
        protected void channelRead0(ChannelHandlerContext channelHandlerContext, String httpContent) throws Exception {
            log.info("result={}",httpContent);
            future.setResultValue(httpContent);
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext channelHandlerContext, Throwable throwable) throws Exception {
            log.error("{}异常", this.getClass().getName(), throwable);

        }
}
```
## 总结
dubbo的高性能，也源于他对每个点不断的优化，最早的时候我记得看到一篇文章写到：dubbo的异步转同步机制，是使用的CountDownLatch实现的。现在想来，可能是在乱说或者是已经过时的信息。一些框架的原理，还是要自己多思考多翻看，才能掌握。
