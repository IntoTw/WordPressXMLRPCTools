---
title: 'KeepAlive安装以及简单配置'
date: 2020-10-14T00:00:00+08:00
lastmod: 2020-10-14T00:00:00+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["keepalive","分布式"]
categories: ["技术"]
lightgallery: true
---

操作系统：Centos7.3

## 一.依赖安装
首先安装相关依赖：
```shell
yum install -y gcc openssl-devel popt-devel
yum -y install libnl libnl-devel
yum install -y libnfnetlink-devel
```
基本依赖就安装完毕了，如果还缺少什么依赖在下一步编译的时候会有warning，百度去解决就好了

## 二.编译安装源码
首先下载源码到指定目录: 
```shell
cd /usr/local/src
wget http://www.keepalived.org/software/keepalived-1.3.4.tar.gz
```
然后解压，配置
```shell
tar zxvf keepalived-1.3.4.tar.gz 
cd keepalived-1.3.4
./configure --prefix=/usr/local/keepalived
```
之后编译
```shell
make  
make install
```
注意这一步make之后可能会有warning，一般都是缺少依赖造成的，把warning关键字百度一下去yum安装对应依赖就可以了  

## 三.修改配置文件地址
安装完成后，keepalived的默认配置文件地址和我们安装的地址不一样，所以cp过去就可以了
```shell
cp ../keepalived-1.3.4/keepalived/etc/init.d/keepalived /etc/init.d/
mkdir /etc/keepalived
cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
cp keepalived-1.3.4/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
cp /usr/local/keepalived/sbin/keepalived /usr/sbin/
```


之后只要修改/etc/keepalived/ 目录下的keepalived.conf配置文件就可以了

使用service start keepalived启动服务

### 四.附keepalived简单配置文件
```json
! Configuration File for keepalived

global_defs {
   router_id lb01 #设置本机路由id，做区分的
}

vrrp_instance VI_1 {
    state MASTER #主从标记，仅做标识
    interface eth0 #虚拟路由的网卡名
    virtual_router_id 51 #虚拟路由路由id，想要配置在同一个虚拟ip必须要有相同id
    priority 150 #优先级，优先级最高的自动为主机，主机宕机后按照优先级选择热备从机
    advert_int 1 #主备通讯时间间隔
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
         172.17.0.199 #配置到哪个虚拟ip，这里我是在docker中，所以是这个docker的默认网段的一个ip，主备机这个地方ip要相同
    }
}
```
