---
title: '分布式系统：分布式任务调度xxl-job较深入使用'
date: 2020-10-14T00:00:00+08:00
lastmod: 2020-10-14T00:00:00+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["xxl-job","分布式"]
categories: ["技术"]
lightgallery: true
---

&#160; &#160; &#160; &#160;xxl-job是一个分布式定时任务调度框架，功能强大，底层使用自己实现的rpc框架进行注册和管理，数据库使用mysql，调度触发使用数据库锁来作为调度锁。


&#160; &#160; &#160; &#160;xxl-job主要分为调度中心admin以及任务，任务引入依赖jar包并配置启动类为spring所管理的bean后，将自动通过spring-bean提供的initMethod进行启动线程选择一个端口进行注册以及监听任务调度。


&#160; &#160; &#160; &#160;公司目前引入xxl-job框架代替quartz框架作为分布式任务调度组件，并在其之上进行一定开发以及优化，所以这篇文章主要分享一些深入使用，主要是概念的详细介绍。

## 系统关键概念介绍


### 执行器
&#160; &#160; &#160; &#160;配置中心配置的执行器，概念上对应执行定时任务的服务，支持分布式调度以及调度的各种路由规则配置。注册方式支持自动注册和手动配置机器地址两种方式，心跳时间间隔默认为30s，失效时间90s。

&#160; &#160; &#160; &#160;执行器自动注册后，调度中心页面依旧有最长30秒的延迟显示，原因是数据库中注册表更新后，展示执行器的表是由另一个守护线程去更新的，更新频率为默认心跳时间30s，所以管理台展示会有延迟，但不影响任务调度。


### 任务

&#160; &#160; &#160; &#160;任务以执行器为维度配置，每个任务必须属于一个执行器，当任务触发时会根据该任务所属的执行器去寻找执行器的地址列表，然后通过配置的路由规则以及阻塞规则去去执行。

&#160; &#160; &#160; &#160;任务支持本地任务以及远程任务，本地任务即按照执行方写好的业务逻辑执行。远程任务通过GLUE，在调度中心管理台写好代码，分发到执行方去执行。建议无特殊需求的话，统一使用本地任务。

#### 任务配置项描述


![xxl-job的图片](https://images.intotw.cn/blog/2023/09/ebb4325ec7a8e0a9bd3fc3ed885e8424.jpg "test")

**1. 执行器**：选择该任务由哪个执行器去执行

**2. 任务描述**：简单描述该任务的功能以及作用，如：订单定时跑批

**3. 路由策略**：设置任务执行时，如何去选择执行器，高频任务建议使用一致性哈希或者第一台执行

**4. Cron**：Cron表达式，描述任务运行的时间

**5. 运行模式**：BEAN即为接入服务配置在本地对应的handler运行，其他方式均为管理台设置代码交由接入服务远程执行

**6. JobHandler**：运行模式为BEAN时必填，值应当为接入服务本地执行任务的handler

**7. 阻塞策略**：当同一任务多次调度到同一台执行器时，执行器应当使用的策略

**8. 子任务ID**：如配置，则该任务完成后自动触发一次子任务的执行

**9. 任务超时时间**：配置后当任务超时时将自动终止任务执行。

**10. 失败重试次数**：任务失败后重试的次数。

**11. 负责人**：一般为该任务接入方的负责人

**12. 报警邮件**：任务报警后发送的邮件地址

**13. 任务参数**：若配置了任务参数，任务调度时将发送任务参数至执行方handler。




### 阻塞策略

阻塞策略即同一个任务在执行器的阻塞执行策略。由执行器端控制。典型场景为：任务A分发到执行器A执行，此时任务A再次触发并分发到执行器A，此时根据阻塞策略选择的不同将会有以下三种执行策略：

**1. 单机串行** 该策略下，**同一执行器**收到**同一任务**的调度触发时，若已有任务正在执行，会将后续的任务放入执行线程的队列中，等待线程轮询继续执行，可能会导致线程队列阻塞过多任务导致内存过高，**高频且耗时较长任务慎用**。

**2. 丢弃后续调度** 该策略下，**同一执行器**收到**同一任务**的调度触发时，若已有任务正在执行，会直接丢弃后续同一任务的调度，**推荐使用**。

**3. 覆盖之前调度** 该策略下，**同一执行器**收到**同一任务**的调度触发时，若已有任务正在执行，将会直接停止正在执行的任务（通过线程InterruptedException异常以及volatile变量判断），并将新任务放入队列。**一般情况下不建议使用**。




### 路由策略

路由策略即任务在配置中心进行调度分发时，选择执行器的策略。由配置中心端控制。典型场景为：任务A触发执行，任务A对应的执行器有执行器A，B，C，D，此时根据路由策略的选择将会有以下几种分发情况

**1. 第一个**：始终选择第一台执行器作为任务执行器，不论该任务执行器是否正常。

**2. 最后一个**：始终选择最后一台作为任务执行器

**3. 轮询**：每个执行器轮流执行

**4. 随机**：随机选择一个执行器执行

**5. 一致性HASH**：根据任务ID做一致性哈希选择执行器，**同一个任务必定只分发到同一个执行器。高频或耗时较长任务推荐使用**

**6. 最不经常使用**：选择平均使用频率最低的执行器。

**7. 最近最久未使用**：选择最近的最久未使用的执行器。

**8. 故障转移**：分别进行心跳检测，选择第一台心跳检测正常的机器执行。

**9. 忙碌转移**：分别进行忙碌检测，选择第一台空闲的机器执行。

**10. 分片广播**：广播到所有执行器执行，并提供分片参数，分片参数获取方式如下，应用在被触发时动态获取自己是第几个分片，共有几个分片。




## 日志问题

xxl-job相关日志使用默认使用slf4j作为日志框架，使用专门的API写入日志时，会输出2种日志，客户端日志与服务端日志

### 客户端日志
客户端日志根据配置文件中配置的logpath指定，根据源码分析，客户端日志将通过FileOutputStream写到对应文件，且无法通过配置修改，所以只好修改了源码中的逻辑，改为该值为空未配置时，直接通过slf4j写入。

### 服务端日志
使用xxljob的日志api输出日志时，日志也会在调度管理台看到，能看到的原理是xxl-job管理台会通过rpc调用执行器的接口，执行器收到请求后从指定的日志文件中读取执行的日志并返回，这里存在一个比较麻烦的问题，就是xxl-job这种日志的逻辑，无法很好的兼容到项目统一的日志模块里，十分不便。

所以实际使用过程中，我们在xxl-job管理台查询日志时，对其进行了改造，修改为不从rpc查询，而是走我们日志管理的搜索引擎根据执行的jobid查询相关日志，结合客户端日志输出的改造，从而统一xxl-job和我们系统间的日志管理。


## 框架目前发现的缺点以及存在的问题

1. 目前定时任务的调度串行是依赖db锁的，某个子微服务或者子系统内部使用还好，但是不适合整个公司级别共用，这点和数据库的耦合比较高，并没有本地缓存之类的，对数据库的HA依赖非常高。
2. 管理模块以及权限模块，组内和小公司用用可以，不适合作为多个系统多个部门之间共用的中间件。
3. 管理台存在一些简单的安全bug，sql注入和js脚本注入非常简单。（公司的安全测试测出来的）
4. 相关协议支持较差，使用自己实现的RPC协议,如果需要dubbo或者spring cloud，需要自己拓展。并且虽然底层是netty，但是本身对netty异常的封装并不是很好，导致一些奇怪的网络问题或者其他的协议，会报莫名其妙的错，没有一定netty理解的人是看不懂的，这点我提了Issue，不过出于这套框架解耦与独立的设计，估计是不会支持dubbo和spring cloud的。
5. 上面提到的，日志模块更建议自己整体修改下写入逻辑，管理台看不到无所谓，（毕竟定时任务也没谁会去管理台看日志，并且这部分日志写多了对性能和网络开销有影响的），写到本地使用elk之类的再查就可以了。
6. 定时任务触发的流水号或者跟踪id，需要改动原框架，否则也会影响后续日志追踪的问题。

总体而言这是一个很不错的框架，关于定时任务的执行器和调度器关系也很优雅,值得拓展或者进行一定定制开发。目前使用来，稳定性也没有问题。
