---
title: 'SRE-基于阿里云的告警体系建设'
date: 2023-11-06T14:53:50+08:00
lastmod: 2023-11-06T14:53:50+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["SRE","告警体系建设"]
categories: ["技术"]
lightgallery: true
---

基于数据源来做分类

## sls日志告警

### 配置以及查看方式

sls日志左侧点击铃铛进入告警中心配置

告警规则触发就是sls日志的查询语句，配置的规则时间内，查询语句查询的数量达到配置值，就会触发告警

![](https://images.intotw.cn/blog/2023/11/d59522ff5045e13102eb2b42dd1f1cd5.png)

### 现状

5XX告警

应用error日志告警

![](https://images.intotw.cn/blog/2023/11/a718e740ff0867c4477b408b279966df.png)

## 云产品监控告警

### 配置以及查看方式

阿里云直接搜索云监控

![](https://images.intotw.cn/blog/2023/11/b5a5a8c3101038b6499a18fdd7c6db99.png)

左边云产品监控，然后搜索要配置的云产品即可，比如redis，rds，kafka

![](https://images.intotw.cn/blog/2023/11/a610f5f8cfd4d1212637c4fec1fc2768.png)

进去搜索到对应的实例，点击报警规则进去配置

![](https://images.intotw.cn/blog/2023/11/e3d429935624acc9cff413d5c25c1f24.png)

### 现状

redis命中率，cpu等监控

mysql内存等监控

kafka堆积等监控

![](https://images.intotw.cn/blog/2023/11/720b35aa8eac34f397cefab5970b3aa6.png)

## arms监控告警

### 配置以及查看方式

arms-应用监控-应用监控告警规则

![](https://images.intotw.cn/blog/2023/11/c6293c50bba69e771a2c146a87575d78.png)

### 现状

pod的fullgc，内存，以及应用的接口环比，慢接口等指标

![](https://images.intotw.cn/blog/2023/11/023c82b503fb62e8acc6752bc091a6d0.png)

## xxl-job告警

### 配置以及查看方式

xxl-job管理台配置任务时选择告警组即可

![](https://images.intotw.cn/blog/2023/11/d52cc31b810dbe77f7c4fad1e479d485.png)

### 现状

![](https://images.intotw.cn/blog/2023/11/b30fd53727e33672042df96f954c4eb5.png)
