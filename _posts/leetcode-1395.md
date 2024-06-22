---
title: 'leetcode刷题录-1395'
date: 2020-10-15
lastmod: 2020-10-15
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["leetcode"]
categories: ["技术"]
lightgallery: true
---

## 题目
题目地址：https://leetcode-cn.com/problems/count-number-of-teams/

> n 名士兵站成一排。每个士兵都有一个 独一无二 的评分 rating 。
> 每 3 个士兵可以组成一个作战单位，分组规则如下：
> 从队伍中选出下标分别为 i、j、k 的 3 名士兵，他们的评分分别为 rating[i]、rating[j]、rating[k]
> 作战单位需满足： rating[i] < rating[j] < rating[k] 或者 rating[i] > rating[j] > rating[k] ，其中  0 <= i < j < k < n
> 请你返回按上述条件可以组建的作战单位数量。每个士兵都可以是多个作战单位的一部分。

## 思考过程
这个题目乍一看，就觉得像是贪心或者是DP，最近做了几道DP题，马上就先找有没有最优子结构来导出结果。

但是仔细想了想，找不到什么最优子结构，浪费了不少时间，但是发现了一个规律，即对于任意i来说，以i为根节点构建一颗树，以i开头的任意子节点均满足递增或递减，就可以快速计算出总数目。如题例 [2,5,3,4,1]，可以构建2->3->4,5->3->1,5->4->1，这2棵树。其中保存的就是结果，对于深度为3的来说，满足的结果数就是1，对于深度为4来说，满足的结果数就是4，对于深度为5来说的树，满足的结果数就是10，规律也比较简单，深度每加1，增加的结果数就是排列组合求的Cn-1 2，比如深度3为结果1，深度4=1+C3 2=4，深度5=4+C4 2=10。根据这种方法，对数组进行遍历并且构建一颗树，然后进行BFS查看深度进行计算就可以了。并且这种方法是会保留结果的方法。

## 查看别人分享的思路
虽然有了解法，但是毕竟一开始初衷还是贪心或者DP，还是不甘心，于是看了一下别人分享的题解，确实有比较取巧的解法，是我一开始的思路错了，这种思路简而言之就是对于任意一个i，把他当做中间数，求得左侧小于i的数的个数left_small，左侧大于i的数的个数left_big，右侧小于i的数的个数right_small，右侧大于i的数的个数right_big，此时就有以i为中间数的结果=left_small*right_big+left_big*right_small，这种思路实现简单，缺点是没有保存中间结果，最后上下代码
```golang
func numTeams(rating []int) int {
	ans:=0
	for i:=0;i<len(rating);i++{
		left_big:=0
		right_big:=0
		left_small:=0
		right_small:=0
		for j:=i-1;j>=0;j--{
			if rating[j]>rating[i] {
				left_big++
			}
			if rating[j]<rating[i]{
				left_small++
			}
		}
		for k:=i+1;k<len(rating);k++{
			if rating[k]>rating[i] {
				right_big++
			}
			if rating[k]<rating[i]{
				right_small++
			}
		}
		ans+=left_small*right_big+left_big*right_small
	}
	return ans
}
```
## 总结
最近刷LeetCode，水题太水，有手就行，还是要刷点稍微需要一些思考的题目，才能提升自己，以后尽量坚持3简单+1中等的进度。
