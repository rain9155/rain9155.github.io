---
title: 揭秘kotlin协程中的suspend方法
date: 2022-05-06 18:38:05
tags: 协程
categories: kotlin
---

## 前言

- 上一篇文章：[揭秘kotlin协程中的CoroutineContext](https://juejin.cn/post/6926695962354122765#heading-0)

上一篇文章中介绍了kotlin协程的**CoroutineContext**的主要组成以及它的结构，kotlin协程的CoroutineContext它是一个K-V数据结构，保存了跟协程相关联的运行上下文例如协程的线程调度策略、异常处理逻辑、日志记录、运行标识、名字等，我们还可以自定义Key和对应的Element把它们放进协程的CoroutineContext中，然后在适当的时候从CoroutineContext中根据Key取出我们自定义的Element并执行相应的逻辑，所以你可以把协程的CoroutineContext简单地类比为线程的[ThreadLocal](https://blog.csdn.net/Rain_9155/article/details/103447399)，与线程的ThreadLocal不同的是协程的CoroutineContext的是**不可变的**而线程的ThreadLocal是**可变的**，所以我们每次对CoroutineContext的修改返回的都是一个新的CoroutineContext，kotlin协程另外一个很重要的概念是**suspend**关键字，通过调用suspend方法我们可以很轻松地完成协程的**挂起和恢复**操作从而把异步回调(callback hell)代码转变成同步调用代码，让我们的代码逻辑更加地优雅，本文的重点就是揭秘suspend关键字背后的原理，但在此之前我会讲解一些前置知识例如Continuation和CPS，kotlin协程是如何被创建、启动、调度，从而更好的理解suspend方法。

> 本文涉及到的源码都是基于kotlin1.4版本，[kotlin coroutines源码地址](https://github.com/Kotlin/kotlinx.coroutines)

## Continuation和CPS



















## Coroutine的创建、启动与调度



## Coroutine的挂起与恢复





## 结语









