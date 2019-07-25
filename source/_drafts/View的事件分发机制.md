---
title: View的事件分发机制
tags: view
categories: View机制
---

## 前言

前几天写过一篇文章[View的工作原理](https://rain9155.github.io/2019/07/22/View的工作原理/)，讲述的View工作的三大流程，其实与View的工作流程同样重要还有View的事件分发机制，平时我们经常通过setOnClickListener()方法来设置一个View的点击监听，那你有没有想过这个点击事件底层是怎么样传递到这个View的呢？当你自定义控件时，如果要处理滑动事件，那么到底返回true还是false？还有当你遇到了滑动嵌套的情景，你要怎么解决滑动嵌套引起的冲突？所以，本文通过 源码 + 流程图 来深入了解一个事件分发机制，当你掌握了它之后，当你遇到与滑动相关的问题时就更加的游刃有余。

<!--more-->

## 准备知识

### 1、什么是触摸事件

触摸事件事件就是当你的手触摸到手机屏幕时所产生的最小单元事件，所谓最小单元，就是不可再拆分的，它一般有4种类型：**按下（down)、移动（move）、抬起（up）、取消(cancel)**。然后由若干个不可再拆分的最小单元事件就组成了**点击事件、长按事件、滑动事件**等。

### 2、什么是MotionEvent

MotionEvent就是Android对上面触摸事件相关信息的封装，View的事件分发中的**事件**就是这个MotionEvent，当这个MotionEvent产生后，那么系统就会将这个MotionEvent传递给View的层级，MotionEvent在View的层级传递的过程就是事件分发。MotionEvent封装了事件类型和坐标两类信息。

事件类型可以通过 **motionEvent.getAction()** 方法获得，它返回一个常量，对应着一个事件类型，事件类型主要有以下4种：

```java
//MotionEvent.java
public final class MotionEvent extends InputEvent implements Parcelable {
    //按下（down)
    public static final int ACTION_DOWN             = 0;
	//抬起（up）
    public static final int ACTION_UP               = 1;
    //移动（move）
    public static final int ACTION_MOVE             = 2;
    //取消(cancel)
    public static final int ACTION_CANCEL           = 3;
    //还有很多就不一 一列举
    //...
}
```

坐标信息也是通过MotionEvent获取，**motionEvent.getRawX()、motionEvent.getRawY()** 可以获得以屏幕作为参考系的坐标值，**motionEvent.getX()、motionEvent.getY()** 可以获得以被触摸的 View 作为参考系的坐标值。参考下面的视图坐标：

{% asset_img view1.png view1 %}

蓝色点想象成手指触摸屏幕的位置。

### 3、一个事件序列

从手指按下屏幕到抬起，就是一个事件序列，这中间还可能由于手抖产生移动，所以可能会有下面两种事件序列：

* ACTION_DOWN -> ACTION_UP：手指按下屏幕后又抬起
* ACTION_DOWN -> ACTION_MOVE -> ... -> ACTION_MOVE -> ACTION_UP：手指按下屏幕，滑动一会，然后抬起

在分析事件分发的过程时，会有事件序列这个概念。

