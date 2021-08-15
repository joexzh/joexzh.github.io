---
layout:     post
title:      "几个例子关于 DDD 中 Entity 和 Value Object 的区别"
subtitle:   "about Entity and Value Object"
date:       2017-04-16 02:00:00
author:     "Joe"
tags:
    - DDD
categories:
    - [coding]
---

老外对 DDD 的释义, 我是比较赞成的.

> There are four concepts of importance here:  
> * `Entity`: a unique object within the domain that has significance (e.g. Customer) and can change over time.  
> * `Value Object`: an immutable object within the domain that has no significance outside of its properties (e.g. date, address).  
> * `Aggregate`: a collection of entities or value objects that are related to each other through a root object.  
> * `Aggregate Root`: An Entity that “owns” an Aggregate and serves as a gateway for all modifications within the Aggregate

*例子*

* 剪刀
    * 在家急需剪纸, 翻箱倒柜找到一把, 不管是'*张小泉*'还是'*王麻子*', 能用就行. *--剪刀是 `Value Object`*
    * 凶杀现场, 找到一把疑似凶器--带有血迹和指纹的剪刀. 作为证据, 不能用任何一把剪刀代替. *-- 剪刀是 `Entity`*
* 地址
    * 为了知道c的地址, 并且想降低不靠谱的概率, 你准备问两个人. a说c住在皇后大道100号, b说c住在皇后大道100号. 这时候你知道了'*c地址*', 而不是'*a说的c地址*'或'*b说的c地址*'. *-- 地址是 `Value Object`*