---
title: 'Git'
date: 2020-10-14T00:00:00+08:00
lastmod: 2020-10-14T00:00:00+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/09/ef5b4f8fe5b5a52acd020e25418132ce.png"
featuredImagePreview: "https://images.intotw.cn/blog/2023/09/ef5b4f8fe5b5a52acd020e25418132ce.png"
tags: ["xxl-job","分布式"]
categories: ["技术"]
lightgallery: true
---

之前公司要做技术分享，因为Git虽然看似简单，但是实际上是使用较多而且较为重要的一个工具，所以做了一下大概的总结。

因为新来的同事问了一个问题，紧急版本要拉hotfix分支修改，但是hotfix分支如何优雅的合并到各个分支上去呢？尤其是hotfix修改的位置比较敏感的情况下。

所以顺带也研究了一下这个问题

下面根据Git最关键的几个概念，穿插了Git几个命令实际做了什么

## 版本号

Git本身有版本号的概念，版本号基于每次**commit**，查看最近的**commit**可以使用**git log**查看，查看所有历史版本可以使用**git reflog**查看所有本地commit的版本号，使用这两个命令可以查看到commit过的记录和commit时填写的信息，但是命令行内查看好像暂时不支持中文的commit信息显示。

## 仓库

Git有2个仓库概念，一个是远端仓库，一个是本地仓库。在本地和远端之间同步的时候一定要谨慎

## 本地仓库

实际上，在不使用**fetch，pull，push，merge**这几个命令的情况下，Git是仅使用本地仓库就可以完成版本管理的，主要的命令（或者说是操作）涉及到**add，commit**。

**add操作**用于将目录下的文件添加到Git的工作空间里去，同目录下未使用add命令添加过的文件，是不参与到git版本管理中去的，也就是说是独立于Git的。

**commit操作**用于提交修改，仅限于本地，用于提交每次修改，commit会产生一个commit id，这个id标识这次commit，在本地的话，实际上版本管理就是不断基于commit做的版本管理，假设commit过3次内容为A,B,C。C为最后commit的版本，此时想回退，只需要git reset commitid --hard 命令，就能把本地仓库回退到B或A的那次commit。

**revert**用于在某次commit之后，假设commit id为A，此时做了一系列修改，但未commit，此时revert会将所有修改取消，回归到commit id为A时的状态。

## 远程仓库

实际上，远程仓库仅仅可以看做是一个代码的备份，是一个公共的备份，也就是说，在每次**push**后，**你的本地仓库和远程仓库所有文件应该是一模一样的**，通过push的话，会把本地的所有commit同步到远程仓库上去，也就是说，**远程仓库的commit过程应该是每个人在本地commit的总和，但可能顺序不同。**

对本地仓库来说，pull就等于同步远端仓库到本地仓库加merge，push就等于把本地仓库同步到远端仓库加merge。

Merge的本质，是将两个分支同步，实际上都是在本地merge。说一下这个过程，假设现在远端仓库版本为A，此时小g和小w同时从A上pull到了本地仓库将他们自己的本地仓库同步到了远端的版本A，此时他们的本地仓库我们假设为A1，A2，他们基于A1,A2做了自己的修改，并且同时修改了test.txt，然后都想提交到远端，第一个提交的人肯定可以提交成功，但是第二个提交的人必定会出现push失败，需要merge该文件，此时他在idea中merge了该文件确认自己改的才是对的，然后再push，此时就成功了。
    

解释一下，此时的merge实际过程是他处理了冲突之后，又进行了一次针对merge的commit，此次commit专门用来处理冲突并提交，具体体现在idea里你会发现，有的时候会让你选accept theirs,accept yours，你选了accept yours或者全部按照你本地来作为最终版本的文件，不需要commit，如果你选的是accept theirs或者merge的时候选择了远端的修改，那么你本地需要全部commit一次再提交，也就是说实际处理的merge是在本地确认了merge结果，这次commit和push实际idea是做了特殊处理的，带着merge的头，所以可以不会再次触发冲突，idea直接通过强制-force push到远端。



## Bug修改，Hotfix

之前考虑到生产环境修改，紧急bug修复，会基于生产环境分支创建一个hotfix分支，做修改完成后由prod分支合并hotfix分支，然后删除hotfix分支，但是这带来了一个问题，该hotfix分支难以合并到例如dev，test等分支，因为：

1.配置文件问题，从hotfix分支merge到dev分支，会导致dev等分支配置文件被覆盖，十分麻烦

2.版本管理问题，如果dev此时分支远远超前prod分支，那么此时将不可能合并hotfix的修改到dev上

可以通过git的stash功能解决这个问题。

stash和他的操作unstash的本质，是创建一个修改副本。他的应用场景和操作模式在于，有的时候本地做了一系列修改，但是忽然又要全部回到修改前的时候进行一下打包，此时要么从新从远端拉一个分支到本地，要么就直接copy一份项目到其他目录，然后这个目录下的revert掉，改完覆盖回来。

stash可以轻松解决，stash可以对当前所有的修改（具体提现到所有蓝色的文件）做一个备份，创建一个stash后，项目会自动回到当前commit id的状态（相当于revert），相当于对所有修改做了一个另外的保存，然后可以通过unstash可以把这些修改还原回来。

这样的话通过这个功能可以很轻松的解决这个问题，本地check出来hotfix分支之后，修改完毕不要先commit，先进行stash，然后unstash回来，再进行commit，push。完成后回到dev分支进行unstash同样进行一下更改，就把更改应用在dev分支上了。
