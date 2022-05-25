---
title: 揭秘kotlin协程的实现原理
date: 2022-05-26 02:56:26
tags: 协程
categories: kotlin
---

## 前言

- 上一篇文章：[揭秘kotlin协程中的CoroutineContext](https://juejin.cn/post/6926695962354122765#heading-0)

上一篇文章中介绍了kotlin协程的CoroutineContext的主要组成以及它的结构，kotlin协程的CoroutineContext它是一个K-V数据结构，保存了跟协程相关联的运行上下文例如协程的线程调度策略、异常处理逻辑、日志记录、运行标识、名字等，本篇文章是作为上一篇文章的补充，在使用kotlin协程一年多之后，对kotlin协程的实现有了新的认识，本文会深入介绍kotlin协程的实现原理，例如Continuation和CPS，协程是如何被创建、启动、调度，suspend方法的含义以及背后的原理，同时使用kotlin-stdlib提供的intrinsics原语实现一个简化版的没有生命周期管理的协程，从而帮助我们更好地理解kotlin协程的整个设计思想，kotlin协程的源码被放在了两个库中，一部分是在kotlin标准库[kotlin-stdlib](https://github.com/JetBrains/kotlin/tree/1.4.0/libraries/stdlib/src/kotlin/coroutines)中，一部分是在kotlin协程官方实现库[kotlinx-coroutines](https://github.com/Kotlin/kotlinx.coroutines/tree/native-mt-1.4.20/kotlinx-coroutines-core)中，其中kotlinx-coroutines是基于kotlin-stdlib的，kotlin-stdlib库提供了实现协程所需的基本原语。

> 本文涉及到的源码都是基于kotlin1.4版本