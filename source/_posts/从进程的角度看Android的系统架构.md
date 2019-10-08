---
title: 从进程的角度看Android的系统架构
date: 2019-10-08 16:32:55
tags: 进程
categories: android
---

## 前言

* 上一篇文章[Android的系统架构概述](https://blog.csdn.net/Rain_9155/article/details/82889613)

上一篇文章从5个层次简述了Android的系统架构，那么这5个层次是怎么联系起来的呢？本文从进程的角度看Android的系统架构，简述一下Android系统启动的过程中，各大进程的启动顺序是如何的，本文并不会涉及到任何源码，只是为了让读者对Android的进程有个大概的了解。

<!--more-->

先看一张图：

{% asset_img android1.jpg android %}

从这张图中可以找到Android官方给出的5个层次（Application、Framework（java）、库和运行时（native）、HAL、Kernel）的影子，java层与native层之间通过**JNI调用**打通，native层与kernel层通过**Syscall调用**打通。这个图就是你启动Android手机时Android系统的启动过程，下面从下到上分别介绍：

## 1、系统启动（Loader）
长按电源键开机键：
### 1.1 Boot ROM
引导芯片代码固化在**ROM**中，当你长按电源键开机时，会引导引导芯片代码从预定义的代码处开始执行，然后加载引导程序Boot Loader到**RAM**中。
### 1.2 Boot Loader
Boot Loader是Android系统启动之前的引导程序，顾名思义，就是将系统拉起来并启动。
## 2、Linux内核启动（Kernel）
然后就到了Linux内核启动，内核启动时就会启动两个进程 --- swapper（pid=0），kthreadd（pid=2）：
### 2.1、swapper（pid=0）
swapper进程又称idle进程， 是系统初始化过程Kernel由无到有开创的第一个进程, 用于初始化进程管理、内存管理，加载Display,Camera Driver，Binder Driver等相关工作。
### 2.2、kthreadd（pid=2）
kthreadd是Linux系统的内核进程，是所有内核进程的鼻祖，会创建内核工作线程kworkder，软中断线程ksoftirqd，thermal等内核守护进程。
## 3、init进程启动（Native）
init进程是Linux系统的用户进程，它的pid=1，是所有用户进程的鼻祖，它是由许多源码文件组成的，它对应的源码目录在/system/core/init中。init进程有许多重要的职责：
* 孵化出许多用户空间的守护进程（ueventd、logd、healthd、installd、adbd、lmkd）
* 启动ServiceManager(binder服务管家)、bootanim(开机动画)等重要服务
* 孵化出Media Server进程，负责启动和管理整个C++ framework，包含AudioFlinger，Camera Service等服务。
* 孵化出Zygote进程
## 4、Zygote进程启动（Native -> java Framework）
Zygote进程是Android系统的第一个Java进程(即虚拟机进程)，Zygote是所有Java进程的父进程，它是由init进程通过解析init.rc文件后fork生成的，Zygote的启动脚本放在/system/core/rootdir目录中。Zygote进程的职责主要有：
* 创建JVM虚拟机，并为JVM注册JNI方法
* 响应AMS的请求去创建新的应用进程
* 孵化出SystemServer进程
## 5、SystemServer进程启动（java Framework）
SystemServer进程是Zygote孵化的第一个进程，负责创建系统服务如ActivityManagerService，WindowManagerService，PackageManagerService，InputManagerService等服务，和管理整个Java framework。SystemServer进程的职责主要有：
* 创建Binder线程池，这样就可以与其他进程进行跨进程通信
* 启动SystemServiceManger，它用来对系统服务进行创建、管理和启动
* 通过SystemServiceManger启动各种系统服务：引导服务（如AMS，PMS），核心服务，其他服务（如WMS，IMS）
## 6、Launcher进程启动（Application）
SystemServer进程启动的过程中会启动PMS和AMS，PMS会把系统中的应用程序安装完成，然后AMS会请求Zygote将Launcher启动起来，这就是用户看到的app桌面，然后Launcher会将已经安装了的应用的应用图标显示出来。Launcher进程是Zygote进程孵化出来的第一个App进程。
至此Android系统已经启动完毕，用户就可以点进桌面上的应用图标进入app，对于普通的app进程,跟SystemServer进程的启动过来有些类似，不同的是app进程是先发消息给SystemServer进程，由SystemServer向Zygote发出创建进程的请求，而SystemServer是由Zygote直接fork出来，前面已经说过Zygote是所有Java进程的父进程，SystemServer和所有的app进程都是由Zygote进程的子进程。

## 总结
Android系统底层基于Linux Kernel, 当Kernel启动过程会创建init进程, 该进程是所有用户空间的鼻祖,
init进程会启动ServiceManager(binder服务管家)、Zygote进程(Java进程的鼻祖)，Zygote进程会创建
system_server进程以及各种app进程，下图是这几个系统重量级进程之间的层级关系。

{% asset_img android2.pngandroid %}

从下而上，其中binder和socket都是Android中进程间的通信方式，而ServiceManager是binder服务的大管家，系统服务的binder实体都会注册到它身上。本文并没有深入的了解各个进程的启动，只是简单的让大家对Android系统主要的进程有个大概的了解，这样以后去研究相应的进程的源码时就会有个大概的方向。

参考资料：

[Android系统启动-综述](http://gityuan.com/2016/02/01/android-booting/)