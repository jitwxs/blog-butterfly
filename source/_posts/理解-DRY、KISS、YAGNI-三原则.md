---
title: 理解 DRY、KISS、YAGNI 三原则
typora-root-url: ..
categories: 综合
tags: 设计原则
abbrlink: 234befaa
date: 2019-09-20 00:15:06
copyright_author: valarchie
references:
  - name: DRY、KISS、YAGNI三原则的理解
    url: https://segmentfault.com/a/1190000020208797
    rel: nofollow noopener noreferrer
    target: _blank
  - name: 3 Key Software Principles You Must Understand
    url: https://code.tutsplus.com/tutorials/3-key-software-principles-you-must-understand--net-25161
    rel: nofollow noopener noreferrer
    target: _blank
---

在软件的设计当中前人已经总结了许多的设计原则和设计模式。例如 `SOLID`，`GRASP` 设计原则，这些原则都是基于面向对象设计总结而来的。而 `GOF23` 是基于许多常见的场景总结出了一套设计模式，在我们遇到类似的场景，都可以套用设计模式。

而今天所讲到的软件三原则是适用于在软件设计的各个层面的。它不仅适用于面向对象的设计，也适用于面向过程的程序设计；不仅适用于类的设计，也适用于模块、子系统的设计。就连在项目架构运维部署中也适用于这一套简单的法则。

![](/images/posts/20190921001429772.png)

## DRY - Don't Repeat Yourself

> A basic strategy for reducing complexity to managable units is to divide a system into pieces.

第一条准则是**不要总是用相同代码解决相同问题**。尽量在项目中减少重复的代码行，重复的方法，重复的模块。

其实许多设计原则和模式最本质的思想都是在消除重复。我们经常提起的重用性和可维护性其实是基于减少重复这一简单的思想。为什么现在微服务盛行呢？正是因为将系统中的服务进行抽取的话，便减少了重复。重复冗余在维护代码的时候将是非常困难的。

DRY 意味着系统内的每一个部件都应该是唯一的并且是不模糊的。我们可以通过应用单一职责接口隔离等原则尽量拆分系统，模块，类，方法·。使其每一个部件都是职责明确的并且可重用的。

## KISS - Keep It Simple & Stupid

> The simplest explanation tends to be the right one.

第二条准则是**让代码简单直接**。从小到几行代码的写法大到整个系统的架构我们都应该保持简单易懂。高手高就高在可以将复杂的东西“简单”的实现出来。

刚入行的时候，我总喜欢用三目运算符将复杂的逻辑用一句冗长的代码行写出来，亦或者是使用 Stream 流写出非常复杂的表达式，到后面才发现这是非常愚蠢的。当需求变更或他人再阅读的时候的时候，看着代码非常费劲难以下手。所以我们应该致力于代码的可理解性。降低复杂度也意味着维护变得简单。

Martin Flower在《重构》中有一句经典的话："任何一个傻瓜都能写出计算机可以理解的程序，只有写出人类容易理解的程序才是优秀的程序员。其实不光是程序，这个原则也可以延伸到产品的设计，业务的设计，项目结构的设计上。

## YAGNI - You Ain’t Gonna Need It

> Coding is about building things.

第三条准则是**适可而止**。只有当你需要的时候才去添加额外的功能，不要过度设计。

我们经常会在开发当中尽可能的迎合未来可能的需求。而为了迎合某些产生概率极低的需求而设计的成本是非常高的，这种过度设计的收益非常低。可能你深思熟虑的设计花了不少时间成本，却在未来的两三年内这个设计却完全没有派上用场。一些设计是否必要，更多的应该基于当前的情况。而不是为了应对未来的各种变化，画蛇添足的设计。

如果淘宝一开始就往日均交易上亿的情况进行设计的话，那么可能就不会有今天的淘宝了。因为创业公司的时间是非常宝贵的，比其他公司早一步退出新的服务就能抢占先机。并不是说淘宝不需要考虑以后交易量暴增的情况，而是不应该以当前日均交易才几万的情况下去设计编码日均交易上亿的项目。过度设计往往会延缓开发迭代的速度。
