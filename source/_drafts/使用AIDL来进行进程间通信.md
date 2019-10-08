---
title: 使用AIDL来进行进程间通信
tags: 
- IPC
- AIDL
categories: android
---

## 前言

AIDL它是Android众多进程间通信方式中的一种，底层是Binder机制的实现，所以想要读懂AIDL自动生成的代码中各个类的作用，就必须对Binder有一定的了解，本文主要介绍AIDL的使用，所以不会介绍Binder机制的原理，关于Binder机制的原理，我推荐下面的一篇文章，零基础也能看懂：

[写给 Android 应用工程师的 Binder 原理剖析](https://zhuanlan.zhihu.com/p/35519585)

看完上面的文章你也就能对Binder原理的实现有大概的了解，对于我们Android应用开发也就足够了，如果你想从底层和源码了解Binder机制，你可以阅读下面的两个链接：

[Android Bander设计与实现](https://blog.csdn.net/universus/article/details/6211589#commentBox)

[Binder系列—开篇](http://gityuan.com/2015/10/31/binder-prepare/)

上面两篇就是从设计和源码的角度去解读Binder，有点深入。好了，对Binder有一个大体上的认识后，我们就要通过AIDL的使用完成Android进程间通信的实践。



