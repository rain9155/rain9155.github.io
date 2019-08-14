---
title: okhttp3源码分析之请求流程
tags: 
- okhttp
- 源码
categories: 优秀开源库分析
---

## 前言

在Android开发中，当下最火的网络请求框架莫过于okhttp和retrofit，它们都是square公司的产品，两个都是非常优秀开源库，值得我们去阅读它们的源码，学习它们的设计理念，但其实retrofit底层还是用okhttp来发起网络请求的，所以深入理解了okhttp也就深入理解了retrofit，它们的源码阅读顺序应该是先看okhttp，我在retrofit上发现它最近的一次提交才把okhttp版本更新到3.14，okhttp目前最新的版本是4.0.x，okhttp从4.0.x开始采用kotlin编写，在这之前还是用java，而我本次分析的okhttp源码版本是基本3.14.x，看哪个版本的不重要，重要的是阅读过后的收获，我打算分3篇文章去分析okhttp，分别是：请求流程(同步异步)、拦截器(Interceptor)、分发器(Dispather)。本文是第一篇 - okhttp的请求流程。

okhttp项目地址：[okhttp](https://github.com/square/okhttp)

```
本文源码基于okhttp_3.14.x分支
```

