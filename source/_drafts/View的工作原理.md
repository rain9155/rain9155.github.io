---
title: View的工作原理
tags: view
categories: View机制
---

## 前言

在Android中View一直扮演着一个很重要的角色，它是我们开发中视觉的呈现，我平常也使用着Android提供的丰富且功能强大的控件，有时候遇到一个很炫酷的自定义View的开源库，我们也是拿来主义，时间长了你就会发现你只是一个只会使用控件和依赖被人开源库的程序员，这并不是一个开发者，所以我们并不能只满足于使用，我们要理解它背后的工作原理和流程，这样才能自己做出一个属于自己的控件，一直都说自定View是Android进阶中的一道门槛，当其实自定义View当你理解了它的原理后，你就会发现它也不过如此。本文将从源码的角度探讨View工作的三大流程，对View做进一步的认识。俗话说的好：源码才是最好的老师。

	本文代码基于Android8.0

## View何时开始绘制？

提到View，就不得不讲起Window，在[Window,WindowManager和WindowManagerService之间的关系](https://rain9155.github.io/2019/03/22/Window,%20WindowManager和WindowManagerService之间的关系/)文章中讲过，Widnow是View得载体，在ViewRootImpl的setView方法中添加Winodw到WMS之前，会先调用requestLayout绘制整颗View Hierarchy的绘制，如下：

{% asset_img view1.png view1 %}

在requestLayout中最终会调用到performTraversals方法，它是整个View Hierarchy绘制的起点，在它里面会依此调用performMeasure() -> performLayout() -> performDraw()，分别对应顶级View的measure、layout和draw流程，顶级View可以理解为View Hierarchy的根节点，它一般是一个ViewGroup，就像Activity的DecorView一样，然后顶级View的measure、layout和draw流程又会分别调用onMeasure()、onLayout()和onDraw()方法，在onMeasure()方法中会对所有child进行measure过程，同理onLayout()方法中会对所有child进行layout过程，onDraw()方法中会对所有child进行draw过程，如此递归直到完成整颗View Hierarchy的遍历，如图:

{% asset_img view1.png view1 %}

所以我们先从requestLayout()中看起，该方法如下：

```java
//ViewRootImpl.java
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        //检查是否在主线程，在子线程绘制UI会抛出异常，见下方
        checkThread();
        //是否measure和layout布局的开关
        mLayoutRequested = true;
        //1、准备开始遍历View Hierarchy绘制
        scheduleTraversals();
    }
}

void checkThread() {
    if (mThread != Thread.currentThread()) {
        throw new CalledFromWrongThreadException(
            "Only the original thread that created a view hierarchy can touch its views.");
    }
}
```

requestLayout()中首先会检查线程的合法性，Android规定必须在主线程中操作UI，那么为什么不能在子线程中访问UI呢？这是因为Android的UI控件都不是线程安全的，如果在多线程环境下并发访问控件会导致控件处于不可预测状态。接着我们来看注释1，调用了ViewRootImpl的scheduleTraversals方法，如下：

```java
//ViewRootImpl.java
void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            //拦截同步Message，优先处理异步Message
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
             //1、Choreographer回调，里面执行最终执行绘制操作
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            //...
        }
    }

```

在Android4.1之前Android的UI流畅性很差，所以在Android4.1之后引入了Choreographer机制和Vsync机制用来解决这个问题，Choreographer管理者动画、输入和绘制的时机，Vsync叫Vertical Synchronization（垂直同步）信号，每隔 16ms Choreographer就会收到来自native层的Vsync信号，这时Choreographer就会根据事件类型进行相应的回调操作，Choreographer支持4种事件类型回调：输入(CALLBACK_INPUT)、绘制(CALLBACK_TRAVERSAL)、动画(CALLBACK_ANIMATION)、提交(CALLBACK_COMMIT)，并通过postCallback方法在对应需要同步Vsync刷新处进行注册，等待回调，关于这个细节和原理可以看[Android图形系统-Choreographer](https://www.jianshu.com/p/bab0b454e39e)和[Android垂直同步和三重缓存](http://www.apkbus.com/blog-705730-61226.html)，这里我们并不深究Choreographer机制和Vsync机制，我们看到注释1中的Choreographer的postCallback方法提交了CALLBACK_TRAVERSAL类型的回调，它对应着mTraversalRunnable绘制操作，而mTraversalRunnable是一个TraversalRunnable类型的绘制任务，最终回调会执行这个任务，mTraversalRunnable的run方法源码如下：

```java
//ViewRootImpl.java
final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            //1、里面会执行performTraversals()
            doTraversal();
        }
}
```

doTraversal()里面会执行