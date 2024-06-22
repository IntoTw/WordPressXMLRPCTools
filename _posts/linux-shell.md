---
title: 'Linux常用命令'
date: 2020-10-15
lastmod: 2020-10-15
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["linux"]
categories: ["技术"]
lightgallery: true
---

### 根据进程名杀死进程
ps -ef | grep process_name | grep -v grep | awk '{print $2}' | xargs kill -9

### docker拉取镜像
docker pull localhost:5000/v2/moyu-eureka-server:latest

### docker删除镜像
docker images |grep moyu|grep -v grep | awk '{print $3}' | xargs docker rmi

### docker关闭容器
docker ps |grep moyu|grep -v grep | awk '{print $1}' | xargs docker stop

### docker删除容器
docker ps -a|grep moyu|grep -v grep | awk '{print $1}' | xargs docker rm

### docker启动容器
docker run -v /etc/localtime:/etc/localtime --name moyu-eureka-server -itd -p 8761:8761 localhost:5000/v2/moyu-eureka-server  

### 删除指定目录
find 目录 -name "*.abc" | xargs rm

### 查看内存占用前五进程
ps auxw | head -1;ps auxw|sort -rn -k4|head -5 

### 查看CPU占用前三进程
ps auxw|head -1;ps auxw|sort -rn -k3|head -3
##查看TCP连接情况

###查看某端口的占用：
lsof -i :8080

###查看所有tcp连接：
lsof -i tcp

###统计mysql连接数：
lsof -i tcp|grep mysql|wc -lbr>

###查看文件被哪些进程使用
lsof -t $file_name

###查看进程使用了哪些文件
lsof -p $pid

###查看进程里有哪些线程
ps -mp pid -o THREAD,tid,time 输出pid16进制，对应jstack中的nid
printf "%x\n" tid
jstack pid |grep 上述16进制的nid -A 30
