---
title: '工作记录：一次线上服务宕机问题排查'
date: 2020-10-15
lastmod: 2020-10-15
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["线上问题","工作记录"]
categories: ["技术"]
lightgallery: true
---
## 问题的发现
早上上班，运维告警，说账户模块的服务全部CPU以及内存告警，当时正在地铁早高峰，所以他们留下了一台在dump，其他机器立马重启，重启后恢复，上班后立马开始排查。
一开始dump文件没出来，后续运维告知dump也失败了，其实按照结果来看，这个问题要是有dump的话， 当时一眼就能看出来问题，可惜没dump走了不少弯路。

## 问题的分析
从日志看来，当时报错最早始于报了大量的dubbo远程调用超时，后续出现redis超时，mysql超时，所以一开始怀疑是网络问题，经过各种排查，只能查出来当时流量异常，服务器的流量全都打满了，但是网络问题比较玄学，无法下定论，一时半会儿陷入了困境。

后续考虑从其他方式分析，如不只看报错日志，看最近版本的提交记录和功能做代码层分析，最后定位到了问题：
1. 首先是druid，druid在执行耗时长时会打印出来当时正在执行的sql。发现有一条sql一直在执行，从系统异常开始到系统异常结束一直在执行。
2. 具体分析sql，发现是最近一次迭代的管理台功能中，查询用户卡号功能，当未输入任何参数时，因为mybatis的写法如下：

```xml
     select
    <include refid="Base_Column_List"/>
    from account_bank_info
    <where>
        <if test="accountName != null and  accountName !=''">
            and ACCOUNT_NAME LIKE concat('%', #{accountName,jdbcType=VARCHAR}, '%')
        </if>
        <if test="accountName != null and  accountName !=''">
            and ACCOUNT_NAME LIKE concat('%', #{accountName,jdbcType=VARCHAR}, '%')
        </if>
        <if test="accountName != null and  accountName !=''">
            and ACCOUNT_NAME LIKE concat('%', #{accountName,jdbcType=VARCHAR}, '%')
        </if>
        ……
    </where>
```


可以看到使用了大量where加if！=null的拼接，但是又未做默认limit，导致在未输出任何参数，即查询条件全部为null时，直接对几百万数据的绑卡表进行了拖库，上百万数据直接打满流量，导致dubbo和其他需要网络功能的组件全部网络超时。最后几百万的数据导致了垃圾回收完全回收不到任何数据，CPU以及内存打满，服务宕机。


## 总结
生产问题的排查，能直接从异常看出来的问题真的都不算什么问题，这种比较隐含的并且时候dump失败的，是真的很难排查。其实当时有dump的话，一看堆内对象，就能马上定位到这个sql了，没有的话，只能通过其他方式一一排查。事后叫运维优化了dump，除了检测oom之类的异常之外，在系统CPU长时间跑满的情况下也提前进行dump（CPU跑满基本就是垃圾回收触发的标志，长时间跑满一般就是垃圾回收失效，内存不够，疯狂的无限进行gc但没有释放任何空间），防止bug发酵导致无法dump的情况。