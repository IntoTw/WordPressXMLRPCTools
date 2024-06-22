---
title: ' 分布式系统：负载均衡算法'
date: 2020-10-14T00:00:00+08:00
lastmod: 2020-10-14T00:00:00+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["负载均衡","分布式"]
categories: ["技术"]
lightgallery: true
---



## 负载均衡算法
负载均衡算法，一般在分布式场景的中大量使用，负载均衡一般分为调用方负载均衡，和服务方负载均衡，spring cloud中的ribbon就是使用的调用方负载均衡，而通过nginx的配置来进行负载均衡，明显更像是服务端的负载均衡。但是原理是一致的，算法的目的就是在一个服务器集合中，选择其中一个合适的服务器，进行请求的处理。

常见的负载均衡算法包括：随机，轮询，最小压力，哈希。主要为这几大类，其中各自有如带权重的实现，一致性哈希等更好的算法。
这篇文章起源于接手了公司的另一个组的一个cloud项目，是一个架构师搭的，所以仔细看了一遍，看到了基于ribbon的带权重轮询算法实现，一开始看了好久不知道在干什么，补充了相关知识后豁然开朗，于是对这2种算法做一个总结和记录，本文主要描述带权重的轮询算法以及平滑的带权重轮询算法的原理以及实现。

## 带权重的轮询负载均衡算法
轮询负载均衡即在所有服务之间，依次选择每个服务，若服务器有A、B、C、D、E，则调度顺序必定为A、B、C、D、E、A、B、C、D……的循环。**带权重的轮询**即为在轮询的基础上，考虑每个服务的权重，如服务器A、B、C、D、E对应权重{5,1,1,1,1}，一般来说在某总权重(5+1+1+1+1)=9次调度内调度顺序就为AAAAABCDE。

该算法可以很简单的实现，只需要每次选取后将权重-1，直到总权重为0，重置权重数组，即可完成处理。代码如下

```go
func doSimpleWeightBound(servers []Server,initWeight []int){
	res:=list.New()
	weight:=make([]int,len(initWeight))
	count:=make([]int, len(weight))
	totalWeight:=0
	for i := 0; i< len(initWeight);i++  {
		totalWeight+=initWeight[i]
	}
	copy(weight,initWeight)
    nowWeight:=totalWeight
    //一共做totalWeight*100次，查看结果。
	for i := 0; i < totalWeight*100; i++ {
		//1. 选出当前权重最高的机器
		position:=selectHighWeight(weight)
		count[position]++
		res.PushBack(servers[position].name)
		//2. 选出的机器权重数减1
		weight[position]--
		//3. 总权重减1
		nowWeight--
		if(nowWeight<=0){
			copy(weight,initWeight)
			nowWeight=totalWeight
		}
	}
	for i :=0; i < len(weight); i ++ {
		fmt.Printf("%d ",weight[i])
	}
	fmt.Println()
	for i := res.Front(); i != nil; i = i.Next() {
		fmt.Printf("%s ",i.Value)
	}
	fmt.Println()
	for i :=0; i < len(count); i ++ {
		fmt.Printf("%s: %d\n",string('A'+i),count[i])
	}
}
```
代码执行结果如下：
![](https://images.intotw.cn/blog/2023/09/1568b93852ff6094689091da31c0054c.jpg)
可以看到，执行结果在带权综合次数内，分布为AAAAABCDE。

## 平滑带权重的轮询负载均衡算法
带权重的轮询负载均衡算法存在一个问题，即在权重相差很大时，连续的调用在A机器上过于频繁，即使带权重的目的本身就是让权重大的机器处理更多请求，但显然让连续的调用更加平均分散在各台机器上更好。
该算法的实现参照nginx开发者提供的思路，进行如下步骤：
1. 建立两个数组，initWeight记录初始各机器的权重，weight记录过程中的权重。
2. 求得初始总权重totalWeight。
3. 从各机器中选出当前权重weight最高的一台机器S1。
4. S1对应的当前权重减去总权重totalWeight。
5. 每台机器的当前权重weight[i]依次增加各机器初始的权重initWeight[i]
6. 重复3-5步骤。
```go
func doSmoothlyWeightBound(servers []Server,initWeight []int){
	//一共做30次，分别看选出的是哪台机器。
	res:=list.New()
	weight:=make([]int,len(initWeight))
	count:=make([]int, len(weight))
	totalWeight:=0
	for i := 0; i< len(initWeight);i++  {
		totalWeight+=initWeight[i]
	}
	copy(weight,initWeight)
	for i := 0; i < totalWeight*8; i++ {
		//1. 选出当前权重最高的机器
		position:=selectHighWeight(weight)
		count[position]++
		res.PushBack(servers[position].name)
		//2. 选出的机器减去总权重
		weight[position]-=totalWeight
		//3. 对每台机器，增加初始权重
		for j := 0; j < len(weight); j++ {
			weight[j]+=initWeight[j]
		}
	}
	for i :=0; i < len(weight); i ++ {
		fmt.Printf("%d ",weight[i])
	}
	fmt.Println()
	for i := res.Front(); i != nil; i = i.Next() {
		fmt.Printf("%s ",i.Value)
	}
	fmt.Println()
	for i :=0; i < len(count); i ++ {
		fmt.Printf("%s: %d\n",string('A'+i),count[i])
	}
}
```
代码执行结果如下：
![](https://images.intotw.cn/blog/2023/09/66c791a7707ae8c046abfbe9eea0df2a.jpeg)
可以看到，执行结果在带权综合次数内，总次数满足权重比例，且顺序不再是纯粹的连续顺序。

### 缺点
该算法的缺点也很明显，每次总权重周期内的顺序必定是相同的，实际上在服务与权重不变的情况下，只要生成一次顺序，以后就按照顺序去轮询即可，不需要维护计算。
## 思考
### 如何在服务列表变化的情况下，执行算法？
以上算法的实现非常明显的只考虑到了在服务列表以及权重不变的情况下，进行多次选择。但是如果感知到服务列表或者权重比例变化的情况下，该如何处理呢？
通过过程可以发现，算法主要依赖的就是初始权重以及服务列表，在感知到服务列表变化时，应通过一些同步机制，及时的更新正在进行选择时的服务列表以及权重比例，及时重新计算相关数组以及总权重数。具体可容忍什么程度的一致性，就用什么同步方式去更新。

## 总结
通过看高手的代码以及实现，深入了解了一部分知识点。
