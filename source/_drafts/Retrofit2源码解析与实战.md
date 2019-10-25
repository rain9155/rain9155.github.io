---
title: Retrofit2源码解析与实战
date: 2019-10-23 22:00:22
tags: 
- retrofit
- 源码
categories: 优秀开源库分析
---

## 前言

* 上两篇文章：

[okhttp3源码分析之请求流程](https://rain9155.github.io/2019/09/03/okhttp3源码分析之请求流程/)
[okhttp3源码分析之拦截器](https://rain9155.github.io/2019/09/07/okhttp3源码分析之拦截器/)

Retrofit与okhttp是Android开发中最最热门的网络请求库，它们都来自square公司，okhttp在前面的两篇文章中已经通过源码从请求流程和拦截器两个角度分析过，本文的主角是Retrofit，经过这几天的研究，我发现Retrofit只是一个对okhttp网络请求框架的巧妙包装，它可以通过注解去定义一个HTTP请求，然后在底层通过okhttp发起网络请求，就是这样的一个简单的过程，其间运用了大量的设计模式：外观模式、动态代理模式、策略模式、适配器模式、装饰者模式等，其最核心的是动态代理模式，所以在此之前大家对动态代理要有一个了解：

[静态和动态代理模式](https://rain9155.github.io/2019/10/15/代理模式/)

其他的设计模式我会在讲解的过程中简单介绍，除了使用了大量的设计模式，Retrofit还应用了面向接口编程的思想，使得整个系统解耦彻底，本文会通过一个简单的Retrofit使用，然后引出Retrofit的请求流程、核心类、动态代理、面向接口思想，通过这几部分来解剖Retrofit。

Retrofit的项目地址：[retrofit](https://github.com/square/retrofit)

> 本文源码基于retrofit2.4

## Retrofit的简单使用

