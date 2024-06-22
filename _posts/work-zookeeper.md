---
title: '工作记录：记一次线上ZK掉线问题排查'
date: 2020-10-14T00:00:00+08:00
lastmod: 2020-10-14T00:00:00+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["zookeeper","线上问题","工作记录"]
categories: ["技术"]
lightgallery: true
---



## 问题的发现
最早问题的发现在于用户提的，用户提出他支付时支付失败，过了一会儿再试就好了，于是翻日志，查询到当时duboo调用出现了下类错误：
```java
[TraceID:20200527145701489] DEBUG c.y.c.s.w.s.m.m.a.HandlerMethodAspect - Throw: {}
com.alibaba.dubbo.rpc.RpcException: Forbid consumer 172.17.40.16 access service com.bwton.rpc.account.IAccountBindCardRpc from registry 10.10.2.201:2181 use dubbo version 2.8.4, Please check registry access list (whitelist/blacklist).
        at com.alibaba.dubbo.registry.integration.RegistryDirectory.doList(RegistryDirectory.java:579)
        at com.alibaba.dubbo.rpc.cluster.directory.AbstractDirectory.list(AbstractDirectory.java:73)
        at com.alibaba.dubbo.rpc.cluster.support.AbstractClusterInvoker.list(AbstractClusterInvoker.java:260)
        at com.alibaba.dubbo.rpc.cluster.support.AbstractClusterInvoker.invoke(AbstractClusterInvoker.java:219)
        at com.alibaba.dubbo.rpc.cluster.support.wrapper.MockClusterInvoker.invoke(MockClusterInvoker.java:72)
        at com.alibaba.dubbo.rpc.proxy.InvokerInvocationHandler.invoke(InvokerInvocationHandler.java:52)
        at com.alibaba.dubbo.common.bytecode.proxy2.findAccountBindCardList(proxy2.java)
        at com.bwton.controller.account.bindcard.BindCardController.findAccountsByUser_aroundBody6(BindCardController.java:212)
        at com.bwton.controller.account.bindcard.BindCardController$AjcClosure7.run(BindCardController.java:1)
        at org.aspectj.runtime.reflect.JoinPointImpl.proceed(JoinPointImpl.java:149)
        at com.qbao.cat.plugin.DefaultPluginTemplate.proxyCollector(DefaultPluginTemplate.java:97)
        at com.qbao.cat.plugin.DefaultPluginTemplate.doAround(DefaultPluginTemplate.java:77)
        at com.qbao.cat.plugin.spring.SpringControllerPluginTemplate.doAround(SpringControllerPluginTemplate.java:27)
        at com.bwton.controller.account.bindcard.BindCardController.findAccountsByUser(BindCardController.java:201)
        at com.bwton.controller.account.bindcard.BindCardController$$FastClassBySpringCGLIB$$9f6f4f14.invoke(<generated>)
        at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:204)
        at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.invokeJoinpoint(CglibAopProxy.java:721)
        at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:157)
        at org.springframework.aop.aspectj.MethodInvocationProceedingJoinPoint.proceed(MethodInvocationProceedingJoinPoint.java:85)
        at com.yanyan.core.spring.web.servlet.mvc.method.annotation.HandlerMethodAspect.aroundInvoke(HandlerMethodAspect.java:98)
```

这明显不科学啊？就算dubbo和zk的连接短时间内出现了问题，但是dubbo本地是有服务端列表缓存的，为什么会直接报这个没有提供者的错误呢？后续在elk上通过日志对问题进行定位，这报错在那段时间内，还真是多，而且各个soa上都出现了，怎么回事呢？通过zabbix监控看，这些服务器的cpu和内存占用并没有异常，流量也没有打满，为什么会大批量出现掉线？dubbo侧看不出来什么问题，于是决定去zk上看看。

## zk的情况以及分析
登上几台zk看日志，发现每隔半小时到一小时，在某X5时间，都会出现大量断开连接的日志。
```java
2020-05-27T00:00:00+08:00 14:53:20,610 [myid:1] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxnFactory@215] - Accepted socket connection from /10.10.1.11:35730
2020-05-27T00:00:00+08:00 14:53:20,610 [myid:1] - WARN  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxn@376] - Unable to read additional data from client sessionid 0x0, likely client has closed socket
2020-05-27T00:00:00+08:00 14:53:20,610 [myid:1] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxn@1040] - Closed socket connection for client /10.10.1.11:35730 (no session established for client)
2020-05-27T00:00:00+08:00 14:53:20,917 [myid:1] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxnFactory@215] - Accepted socket connection from /10.10.7.51:60135
2020-05-27T00:00:00+08:00 14:53:20,921 [myid:1] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:ZooKeeperServer@938] - Client attempting to establish new session at /10.10.7.51:60135
2020-05-27T00:00:00+08:00 14:53:20,977 [myid:1] - INFO  [CommitProcessor:1:ZooKeeperServer@683] - Established session 0x102d62c8fbab875 with negotiated timeout 10000 for client /10.10.7.51:60135
```
可以看到，zk的日志大量刷屏，并且都是认为是远端没有了心跳，或者心跳超时，所以主动断开了连接，这也就解释了为什么明明dubbo在消费者端是有服务列表缓存的，但是还是会报上面的异常，原来是zk认为这些提供者下线了，所以通过通知，主动把整体服务的列表都刷新了，并不是提供者和zk之间单方面的问题，那么zk为什么会出现这样的问题呢？说实话，查了非常久，也是查了好几天，最后定位在：问题暴露在每五分钟的单位时间，即最短5分钟间隔，最长60分钟间隔，都会出现大量的批量下线后再批量上线的问题，虽然过程转瞬即逝，但是还是有部分业务会被影响到。于是查看有什么东西是5分钟的，查阅资料，zk和dubbo并没有什么缺省的5分钟或者更小整数倍数是5分钟的配置，我们也没有做类似配置，最后锁定到了一个嫌疑者：
```shell
*/5 * * * * /usr/sbin/ntpdate X.X.X.X  && hwclock -w >/dev/null 2>&1
```
熟悉linux的同学都知道，这个是一个定时任务，用来同步服务器时钟的，每五分钟触发一次，后续经过测试发现，zk第三台机器与时钟同步机器的网络连接状况很差，所以第三台机器的时间在时钟同步后会出现波动，导致心跳机制出现问题，认为大量dubbo节点失效，所以主动踢除了这些被认为下线的节点，所以出现了如上bug。后续找网络工程师排查了服务器的网络问题后，该问题成功解决。

## 总结
在排查时，是真的有些乍一看让人摸不着头脑的问题，只能通过排除法以及相关信息去做推测并验证，并且客观反映了，AP较之CP，确实有其优越性，因为CP本身虽然可以保证一致性，但是谁有能保证CP自身的正确，如果CP因为某些原因自身出现了问题，那么风险也是不可承受的。这也是为什么eureka采用AP而不是CP的原因。
