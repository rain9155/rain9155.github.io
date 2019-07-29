---
title: View的事件分发机制
tags: view
categories: View机制
date: 2019-07-29 23:14:42
---


## 前言

前几天写过一篇文章[View的工作原理](https://rain9155.github.io/2019/07/22/View的工作原理/)，讲述的View工作的三大流程，其实与View的工作流程同样重要还有View的事件分发机制，平时我们经常通过setOnClickListener()方法来设置一个View的点击监听，那你有没有想过这个点击事件底层是怎么样传递到这个View的呢？当你自定义控件时，如果要处理滑动事件，那么到底返回true还是false？还有当你遇到了滑动嵌套的情景，你要怎么解决滑动嵌套引起的冲突？所以，本文通过 源码 + 流程图 来深入了解一个事件分发机制，当你掌握了它之后，当你遇到与滑动相关的问题时就更加的游刃有余。

<!--more-->

> 本文源码基于Android8.0

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

从手指按下屏幕到抬起，在这个过程中所产生的一系列事件，就是一个事件序列，这个事件序列以down事件开始，中间含有数量不定的move事件，最终以up事件结束。所以可能会有下面两种事件序列：

* ACTION_DOWN -> ACTION_UP：手指按下屏幕后又抬起
* ACTION_DOWN -> ACTION_MOVE -> ... -> ACTION_MOVE -> ACTION_UP：手指按下屏幕，滑动一会，然后抬起

在分析事件分发的过程时，会有事件序列这个概念。

### 4、事件分发的起点，事件从何而来

我想大家都知道View的事件分发机制的起点是View的dispatchTouchEvent()方法，但是如果从View的dispatchTouchEvent()继续追溯上去，事件是从哪里来的呢？

Android的输入设备有很多种，如屏幕、键盘、鼠标、轨迹球等，而屏幕是我们接触最多的设备，当用户手指触摸屏幕时就会产生触摸事件，这时Android的输入系统就会为这个触摸事件在/dev/input/路径下写入以event[NUMBER]为名的输入设备节点，这时输入系统中的EventHub就会监听到这个输入事件，然后InputReader就会把这个原始输入事件读取并经过加工后交给输入系统中的InputReader，InputDispatcher会在Window列表中会找到合适的Window，然后把这个输入事件分发给合适的Window，然后Window就会把这个事件分发给顶级View，**然后顶级View就把这个输入事件在View树中层层分发下去，直到找到合适的View来处理这个事件，这来到了我们熟悉的View的事件分发机制**。

上面的一些名词如EventHub、InputReader、InputReader都是属于Android的输入系统，这部分是一个很复杂的知识，我只是概括了一下。所以我们只要知道，**输入系统监听到输入事件后，就会先交给Window，然后Window再交给顶级View，然后顶级View在把它分发下去**。(关于Window和View的关系可以看这篇文章[Window, WindowManager和WindowManagerService之间的关系](https://juejin.im/post/5d32acdbf265da1bc4148e86))

这个顶级View可能是View，也有可能是ViewGroup，具体情况看你添加Window到WMS时你的**addView(View view, ViewGroup.LayoutParams params)**方法中的View是View实例还是ViewGroup实例，所以本文接下来就分别分析View的事件分发和ViewGroup的事件分发。

## View的事件分发

### 1、View::dispatchTouchEvent()

View的事件分发比ViewGroup的简单，因为它只是一个单独的元素，所以它只需要处理自己的事件，View的事件分发从View的dispatchTouchEvent()方法开始，所以我们看它的dispatchTouchEvent方法，如下：

```java
//View.java
public boolean dispatchTouchEvent(MotionEvent event) {
    //...
    //result默认为false
    boolean result = false;
    //...
     ListenerInfo li = mListenerInfo;
    if (
        li != null//如果ListenerInfo不为空
        && li.mOnTouchListener != null//如果触摸事件的监听不为空
        && (mViewFlags & ENABLED_MASK) == ENABLED//如果该控件是ENABLED状态
        && li.mOnTouchListener.onTouch(this, event)//如果onTouch方法返回了true
    ){
        result = true;
    }
    if (
        !result//如果上面四个条件都不满足，result默认为false
        && onTouchEvent(event)//如果onTouchEvent()方法返回了true
    ) {
        result = true;
    }
    //...
    return result;
}

//View.java
static class ListenerInfo {
     public OnClickListener mOnClickListener;//点击事件的监听
     protected OnLongClickListener mOnLongClickListener;//长按事件的监听
     private OnTouchListener mOnTouchListener;//触摸事件的监听
    //...
}
```

从View的dispatchTouchEvent()方法的伪代码可以看出，dispatchTouchEvent()方法首先会根据4个条件来决定是否调用View的onTouchEvent方法，如下：

* 1、如果ListenerInfo不为空：ListenerInfo里面有View的各种监听，那么mListenerInfo是什么时候被赋值的呢？答案是给View设置监听的时候，在我们给View设置任何监听的时候，如果这个mListenerInfo还没初始化就会先初始化，比如设置触摸事件的监听，我们看setOnTouchListener()方法，如下：

  ```java
  //View.java
  public void setOnTouchListener(OnTouchListener l) {
      //先调用getListenerInfo方法初始化mListenerInfo，然后把触摸事件的监听赋值给mOnTouchListener
      getListenerInfo().mOnTouchListener = l;
  }
  
  //View.java
  ListenerInfo getListenerInfo() {
        if (mListenerInfo != null) {
            return mListenerInfo;
        }
        mListenerInfo = new ListenerInfo();
        return mListenerInfo;
    }
  ```

* 2、如果触摸事件的监听不为空：即ListenerInfo的mOnTouchListener不为空，从1可以看出，当你给View设置OnTouchListener时，就已经满足了1、2条件了。
* 3、如果该控件是ENABLED状态：即该Vew处于enable状态，如果你没有手动调用过View的setEnable(false)设置控件为不可用的话，这个条件就为true，控件默认为enable状态。
* 4、如果onTouch方法返回了true：当你给View设置OnTouchListener，并且在onTouch方法中返回了true，表示消费了这次事件，那么这个条件就为true。所以到这里，如果4个条件都满足的话，result就会等于true，就会导致下面无法调用View的onTouchEvent()方法。

但是如果你**没有给你给View设置OnTouchListener或者你给View设置了OnTouchListener，但是onTouch方法返回了false**，只要满足这两个条件之一，就会让result保持默认值false，从而满足下面的条件调用View的onTouchEvent()方法。这里得出一个结论：**OnTouchListener的onTouch方法的优先级高于onTouchEvent()方法**。

假设现在不满足上面4个条件，从而调用View的onTouchEvent()方法，我们来看View的onTouchEvent()方法。

### 2、View::onTouchEvent()

onTouchEvent()方法里面会处理View点击事件、长按事件，即回调你设置的OnClickListener的onClick()方法和OnLongClickListener的OnLongClick()方法，在你设置OnClickListener或OnLongClickListener回调时会同时把你的View设置为可点击状态即clickable状态，有些控件默认可点击如Button，而有些控件需要设置点击回调或setClickable(true)才可以点击如TextView。

接下来我们看View的onTouchEvent()方法的主要源码，如下：

```java
//View.java
public boolean onTouchEvent(MotionEvent event) {
    final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;
    final int action = event.getAction();
	//该View是否可点击，可以看到这里点击包含3种点击：CLICKABLE、LONG_CLICKABLE和CONTEXT_CLICKABLE(回调OnContextClickListener)
    //这里我们关注CLICKABLE和LONG_CLICKABLE就行，即点击和长按
    final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
        || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
        || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;
    //1、如果View处于disabled状态，即不可用状态
    if ((viewFlags & ENABLED_MASK) == DISABLED) {
        if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
            setPressed(false);
        }
        mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
        //这里说明了即使View处于不可用状态，但是如果它可以点击，它还是会消费点击事件
        return clickable;
    }
    //2、如果View设置有代理机制，那么就会执行TouchDelegate的onTouchEvent()方法
    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }
    //3、下面是onTouchEvent()对点击事件和长按事件的处理
    //如果控件可以点击
    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
        switch (action) {
            case MotionEvent.ACTION_UP:
                //...
                break;
            case MotionEvent.ACTION_DOWN:
                //...
                break;

            case MotionEvent.ACTION_CANCEL:
               //...
                break;
            case MotionEvent.ACTION_MOVE:
               //...
                break;
        }
        //从这里看出，如果我们的View是可以点击的，最终一定返回true，表示消费了此事件
        return true;
    }
    //4、最终虽然控件可用，但是不可点击，返回false，不消费此事件
    return false;
}
```

这个onTouchEvent()方法有点长，这里我截取了整体框架，这里我们先明确一点的是onTochEvent()中返回true就表示这个事件由这个View消费，返回false就表示这个View不消费这个事件然后它的父容器会继续找合适的View消费。首先我们看注释1，它说明了即使View处于不可用状态，但是如果它可以点击即clickable = true，它会返回true，表明不可用状态下的View它还是会消费事件，即使这个View会没有响应，反之返回false；接着注释2，如果设置了mTouchDelegate，则会将事件交给代理者处理，直接return true，如果大家希望自己的View增加它的touch范围，可以尝试使用TouchDelegate；接着注释3，如果控件可以点击，就判断事件类型：ACTION_UP、ACTION_DOWN、ACTION_CANCEL、ACTION_MOVE，然后根据不同的事件类型做出不同的行为，然后都返回了true，表示消费了此事件；最后注释4如果控件不可点击，就返回false，不消费此事件。

接下来我们重点看注释3，看onTouchEvent()是如何在ACTION_UP、ACTION_DOWN、ACTION_CANCEL、ACTION_MOVE中触发onClick()和onLingClick()回调的。

#### 2.1、case ACTION_DOWN:

```java
  switch (action) {
      case MotionEvent.ACTION_DOWN:
          //...
          //1、设置mHasPerformedLongPress为false
          mHasPerformedLongPress = false;
          //...
          //2、给mPrivateFlags设置一个PREPRESSED标识
           mPrivateFlags |= PFLAG_PREPRESSED;
          //3、通过postDelayed发送一个延时100毫秒后执行的任务mPendingCheckForTap
          postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
          //...
          break;
  }
return true;
```

我们想象一下，我们的手指按下这个View，这时进入ACTION_DOWN分支，在这个分支里，在注释1中它首先设置mHasPerformedLongPress为false，表示长按事件还没有触发，然后在注释2给mPrivateFlags设置一个PREPRESSED的标识，表示开始检查长按事件，然后在注释3通过postDelayed发送了一个延时消息，ViewConfiguration.getTapTimeout()返回100毫秒，即100毫秒后会执行任务mPendingCheckForTap，它一个CheckForTap类型任务，它是用来**检测长按事件**的。我们看这个任务是什么，如下：

```java
//View.java 
//用来检测长按事件
private final class CheckForTap implements Runnable {
        public float x;
        public float y;

        @Override
        public void run() {
            //1、mPrivateFlags清除PFLAG_PREPRESSED标识
            mPrivateFlags &= ~PFLAG_PREPRESSED;
            //2、见下面调用链，这里传入true，即给mPrivateFlags设置一个PFLAG_PRESSED标识
            setPressed(true, x, y);
            //3、调用checkForLongClick()方法，传入100毫秒
            checkForLongClick(ViewConfiguration.getTapTimeout(), x, y);
        }
    }

private void setPressed(boolean pressed, float x, float y) {
        //...
        //主要调用了带一个参数的setPressed(pressed)方法
        setPressed(pressed);
    }

//设置控件是否处于按下状态
public void setPressed(boolean pressed) {
    //...
    if (pressed) {//如果pressed为true
        //给mPrivateFlags设置一个PFLAG_PRESSED标识
        mPrivateFlags |= PFLAG_PRESSED;
    } else {//如果pressed为false
        //清除mPrivateFlags之前设置的PFLAG_PRESSED标识
        mPrivateFlags &= ~PFLAG_PRESSED;
    }
   //...
}

```

mPendingCheckForTap的run方法里面在注释1首先会先清除mPrivateFlags中PFLAG_PREPRESSED标识，然后在注释2设置PFLAG_PRESSED标识，表示准备执行长按事件，最主要的是注释3，我们看checkForLongClick方法里面干了什么，如下：

```java
//View.java
private void checkForLongClick(int delayOffset, float x, float y) {
    //1、检查mViewFlags，如果可以进行长按事件LONG_CLICKABLE
    if ((mViewFlags & LONG_CLICKABLE) == LONG_CLICKABLE || (mViewFlags & TOOLTIP) == TOOLTIP) {
        //此时mHasPerformedLongPress标志位还是false
        mHasPerformedLongPress = false;
        if (mPendingCheckForLongPress == null) {
            //2、创建了一个CheckForLongPress类型的任务
            mPendingCheckForLongPress = new CheckForLongPress();
        }
        //...
        //3、ViewConfiguration.getLongPressTimeout()返回500毫秒，再减100毫秒等于400毫秒
        //通过postDelayed()发送延时400毫秒后执行的任务mPendingCheckForLongPress
        postDelayed(mPendingCheckForLongPress,
                    ViewConfiguration.getLongPressTimeout() - delayOffset);
    }
}
```

这个方法在注释1首先检测该View是否可以进行长按事件，View的LONG_CLICKABLE属性默认为false，但是在setOnLongClickListener（）时就会把它设置为true，然后在注释2创建了一个CheckForLongPress类型的任务，然后在注释3通过postDelayed()发送了一个延时消息，即400毫秒后执行mPendingCheckForLongPress任务，它是用来**执行长按事件**的，我们看这个任务的具体实现，如下：

```java
//View.java
//用来执行长按事件
private final class CheckForLongPress implements Runnable {
        private float mX;
        private float mY;

        @Override
        public void run() {
            if ((mOriginalPressedState == isPressed())//1、首先检查mPrivateFlags中是否清除了PFLAG_PRESSED标识，如果清除了表示长按事件取消
                && (mParent != null)
                && mOriginalWindowAttachCount == mWindowAttachCount) {
                //2、调用performLongClick()方法
                if (performLongClick(mX, mY)) {
                    //3、设置mHasPerformedLongPress为true
                    mHasPerformedLongPress = true;
                }
            }
           //...
            if (performLongClick(mX, mY)) {
                mHasPerformedLongPress = true;
            }
        }
	//...
}

public boolean isPressed() {
    //返回true表示检测mPrivateFlags中清除了PFLAG_PRESSED标识，false反之
    return (mPrivateFlags & PFLAG_PRESSED) == PFLAG_PRESSED;
}
```

CheckForLongPress就是用于执行长按事件的，它的run方法里面会先检查mPrivateFlags中是否清除了PFLAG_PRESSED标识，如果清除了就表示长按事件取消，否则就调用performLongClick()方法，里面会最终回调onLongClick()方法回调，如果performLongClick()返回true，就会设置mHasPerformedLongPress为true，否则mHasPerformedLongPress还是为false，即**mHasPerformedLongPress是否为true取决performLongClick(float x, float y)是否返回true**，接下来我们看performLongClick(float x, float y)方法，如下：

```java
//View.java
public boolean performLongClick(float x, float y) {
       //...
        final boolean handled = performLongClick();
        //...
        return handled;
    }

public boolean performLongClick() {
    return performLongClickInternal(mLongClickX, mLongClickY);
}

private boolean performLongClickInternal(float x, float y) {
    boolean handled = false;
    final ListenerInfo li = mListenerInfo;
    //如果设置了OnLongClickListener回调
    if (li != null && li.mOnLongClickListener != null) {       
        //回调OnLongClickListener的onLongClick方法
        handled = li.mOnLongClickListener.onLongClick(View.this);
    }
    //...
    return handled;
}

```

在这个方法中一路跟进，最终来到了performLongClickInternal(float x, float y)方法中，在performLongClickInternal()方法中，如果我们通过setOnLongClickListener()设置了OnLongClickListener回调，这里就会回调我们熟悉的onLongClick()方法，而performLongClickInternal()是否返回true取决于我们在onLongClick()方法中是否返回true，performLongClick()是否返回true取决于performLongClickInternal()是否返回true，然后这里结合上面的黑体字得出一个结论：**如果你设置了onLongClickListener，mHasPerformedLongPress是否为true取决我们在onLongClick()方法中是否返回true，如果没有设置，mHasPerformedLongPress就一直为false**，这个mHasPerformedLongPress是否为true会影响我们在ACTION_UP是否能够回调onClick()方法的关键。

现在我们通过case ACTION_DOWN知道：**如果我们按下手指在500毫秒内没有抬起，就会触发长按事件**。下面分析ACTION_UP。

#### 2.2、case ACTION_UP:

```java
switch (action) {
    case MotionEvent.ACTION_UP:
        //...
        boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
        //1、如果mPrivateFlags中包含PFLAG_PRESSED或PFLAG_PREPRESSED标识，都会进入if分支
        if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
           //...
            if (prepressed) {//如果mPrivateFlags中只包含PFLAG_PREPRESSED标识，表示用户在100毫秒内抬起了手指，还没执行CheckForTap任务
                //2、这里传入为true，即给mPrivateFlags设置一个PFLAG_PRESSED标识
                //这里主要让用户看到控件还是按下状态
                setPressed(true, x, y);
            }
		   //3、这个mHasPerformedLongPress为false就进入if分支，mIgnoreNextUpEvent默认为false
            if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                //4、移除长按事件CheckForLongPress任务消息，即取消长按事件
                removeLongPressCallback();
                //...
                if (mPerformClick == null) {
                    //5、如果mPerformClick为null，初始化一个实例
                    mPerformClick = new PerformClick();
                }
                //6、通过Handler把mPerformClick添加到消息队列，但其实PerformClick中的run方法还是执行performClick()方法，所以我们只要看performClick()方法就行
                if (!post(mPerformClick)) {
                    //如果post一个PerformClick失败就执行performClick()方法
                    performClick();
                }
            }
            //...
            //7、移除检测长按事件CheckForTap任务消息，即取消检测长按事件
            removeTapCallback();
        }
        break;
}
return true;

//View.java
//PerformClick中的run方法还是执行performClick()方法
 private final class PerformClick implements Runnable {
     @Override
     public void run() {
         performClick();
     }
 }
```

我们想象一下，现在我们抬起了手指，分三个时间段抬起：

* 1、如果你在**100毫秒内**抬起手指，那么mPrivateFlags肯定只有PFLAG_PREPRESSED标识，且mHasPerformedLongPress为false，根据注释1和3，这样就会执行PerformClick()方法，在执行PerformClick()方法前，在注释4调用removeLongPressCallback()移除长按事件CheckForLongPress任务，即不会触发onLongClick()回调。

* 2、如果你在**100毫秒后到500毫秒**才抬起，那么mPrivateFlags肯定只有PFLAG_PRESSED标识，且mHasPerformedLongPress为false，接下来的逻辑和1一样。

* 3、如果你在**500毫秒后**才抬起，那么mPrivateFlags肯定只有PFLAG_PRESSED标识，而mHasPerformedLongPress是否为true取决我们是否设置onLongClickListener并在onLongClick()方法中是否返回true。如果你设置了onLongClickListener回调并在onLongClick()方法中返回了false或者你没有设置onLongClickListener回调，那么你还是可以走到注释6执行performClick()方法；但是如果你设置了onLongClickListener回调并在onLongClick()方法中返回了true，那么你就不能执行performClick()方法了。

对照ACTION_DOWN的流程和ACTION_UP的流程就能更好的理解上面3个时间段，所以从这里我们知道：**如果你在500毫秒内抬起手指，那么你就只能执行点击事件，不能执行长按事件；如果你在500毫秒后抬起，并且你设置了onLongClickListener并在onLongClick()方法中返回了false 或者 你没有设置onLongClickListener回调，那么你执行完长按事件后还可以执行点击事件，但是如果你设置了onLongClickListener回调并在onLongClick()方法中返回了true，那么你就不能执行点击事件**。performClick()和 performLongClick()方法类似，它里面最终回调onClick()方法，如下：

```java
//View.java
public boolean performClick() {
     final boolean result;
     final ListenerInfo li = mListenerInfo;
     if (li != null && li.mOnClickListener != null) {
         //1、执行了OnClickListener的onClick()方法
         li.mOnClickListener.onClick(this);
         result = true;
     } else {
         result = false;
     }
     //...
     return result;
 }
```

performClick()方法中的逻辑是，如果你设置了OnClickListener回调，那么就会执行onClick(）方法，大家也注意到performClick()会返回一个true或者false，但是这个返回值对于onTouchEvent()方法没有任何意义，因为上面提到switch语句块的后面一定返回true。这里我们再得出一个结论：**OnLongClickListener的onLongClick()方法的优先级高于onClickListener的onClick()方法**。

好了现在我们的手指从按下到抬起，就已经分析完onTouchEvent()中的ACTION_DOWN和ACTION_UP分支，如果你的手指在抬起前，不小心移动了一下，就会触发ACTION_CANCEL或ACTION_MOVE，这个时候它就会根据条件(手指是否移出View的范围)通过调用 removeLongPressCallback()或 removeTapCallback()方法移除CheckForLongPress或CheckForTap任务，即取消长按或点击，这里限于篇幅就不再展开分析，大家可自行分析。

### 3、小结

1、View没有子View，所以它的的分发比较简单，从View的dispatchTouchEvent()方法开始进入View的事件分发流程，该方法只负责事件的分发，没有进行实际事件的处理，进行实际事件的处理有两处地方：1、通过外部设置的onTouchListener的onTouch()方法，2、View的onTouchEvent()方法。

2、当一个View要处理点击事件时，如果它设置了onTouchListener，那么onTouch方法就会回调，这时事件如何处理还要看onTouch()方法的返回值，如果返回true，那么onTouchEvent()方法将不会被调用，dispatchTouchEvent()方法直接返回true；如果返回false，onTouchEvent()方法会被调用，这时事件如何处理就要看onTouchEvent()的返回值，在onTouchEvent()中，不管控件可用还是不可用，返回值取决于控件是否可点击，如果控件可点击(clickabale或longClickabale，只要有一个为true)，onTouchEvent()返回true，如果控件不可点击(clickabale和longClickabale都为false)，onTouchEvent()返回false。

3、如果我们同时设置了OnTouchListener、OnLongClickListener和OnClickListener回调，根据优先级，事件的传递顺序是：**onTouch() -> onLongClick() -> onClick()**，其中除了onClick()都有boolean返回值，返回值能决定下一个方法是否被调用，onClick()优先级最低，连返回值都没有。

4、 对于ViewGroup（也就是当前 View 的父容器）而言，它只认识子 View的dispatchTouchEvent()方法，不认识另外两个处理事件的方法。子View的 onTouch()  和 onTouchEvent() 都是在自己的 dispatchTouchEvent() 里面调用的，他们两个会影响 dispatchTouchEvent() 的返回值，但是对于上级 ViewGroup 而言，它只认识 dispatchTouchEvent() 的返回值。

流程图：

{% asset_img view2.png view2 %}

## ViewGroup的事件分发

### 1、ViewGroup::dispatchTouchEvent()

ViewGroup是View的子类，它是一组View的集合，它包含很多子View和子ViewGroup，所以ViewGroup的事件分发比View的复杂，但是ViewGroup的事件分发才是整个事件分发机制的精髓，和View一样ViewGroup的事件分发的起点也是dispatchTouchEvent()，虽然这个方法在View中，但是ViewGroup重写了它，因为它们的分发逻辑不一样。所以我们看ViewGroup的dispatchTouchEvent()方法，如下：

```java
//ViewGroup.java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    //...
    //本次事件处理结果
    boolean handled = false;
    if (onFilterTouchEventForSecurity(ev)) {
         final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;
        //1、如果本次事件是ACTION_DOWN
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            //置空mFirstTouchTarget
            cancelAndClearTouchTargets(ev);
            //清除mGroupFlags中的FLAG_DISALLOW_INTERCEPT标志位，这个标志等同于下面的disallowIntercept
            resetTouchState();
        }
        //ViewGroup是否拦截本次事件标志
        final boolean intercepted;
        //2、如果本次事件是ACTION_DOWN 或者 mFirstTouchTarget为空
        if (actionMasked == MotionEvent.ACTION_DOWN
            || mFirstTouchTarget != null) {
            //子View是否禁止ViewGroup拦截事件标志
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {//如果子View允许ViewGroup拦截事件
                //调用onInterceptTouchEvent()方法询问ViewGroup是否拦截事件，intercepted的值由onInterceptTouchEvent(ev)决定
                intercepted = onInterceptTouchEvent(ev);
                //...
            } else {//如果子View禁止ViewGroup拦截事件
                intercepted = false;//intercepted值为false
            }
        } else {//如果本次事件不是ACTION_DOWN又没有target
            //intercepted值为true，在此之后，当前事件序列中的所有事件序列都由ViewGroup处理，不会再传递给子View
            intercepted = true;
        }
        //...
        //检查本次事件是否是ACTION_CANCEL
        final boolean canceled = resetCancelNextUpFlag(this) || actionMasked == MotionEvent.ACTION_CANCEL;
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;
        //3、如果本次事件不取消并且不拦截，就寻找合适的子View处理
        if (!canceled && !intercepted) {
            //...
            //3.1、如果本次事件是ACTION_DOWN
            if (actionMasked == MotionEvent.ACTION_DOWN
                || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
               
			  final int childrenCount = mChildrenCount;
                //如果target是null并且ViewGroup有子View，就寻找某个子View当mFirstTouchTarget
                if (newTouchTarget == null && childrenCount != 0) {
                    final float x = ev.getX(actionIndex);
                    final float y = ev.getY(actionIndex);
                    final View[] children = mChildren;
                    //从后往前逐个取出子View
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
                        final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);
                        //判断子View能否接受点击事件：子View可见或在播放动画，并且触摸点在子View范围内
                        if (!canViewReceivePointerEvents(child)
                            || !isTransformedTouchPointInView(x, y, child, null)) {
                            ev.setTargetAccessibilityFocus(false);
                            continue;
                        }
					  //走到这里表示子View满足处理事件的条件
                        //...
                        //dispatchTransformedTouchEvent()里面会调用子View的dispatchTouchEvent()方法，在这个方法里把事件分发给子View
                         if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                              //...
                               //如果dispatchTransformedTouchEvent()返回true，表示找到子View消费本次事件了，就会走到这里, 所以这个子View就被当作mFirstTouchTarget，这里会调用addTouchTarget()方法为mFirstTouchTarget赋值
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }
						//...
                    }//end...for（）
                    
                    //...
                    
                }//end...if(newTouchTarget == null && childrenCount != 0)
                
            }//end...if(actionMasked == MotionEvent.ACTION_DOWN...)
            
        }///end...if(!canceled && !intercepted)
        
        //4、根据mFirstTouchTarget是否为null做出不同行为
        if (mFirstTouchTarget == null) {//这一般有三种情况导致mFirstTouchTarget为空：
            //1、ViewGroup没有子View；
            //2、子View处理了ACTION_DOWN事件，但是在dispatchTouchEvent()返回了false；
            //3、ViewGroup在DOWN事件中的onInterceptTouchEvent(ev)返回了true
       		//在这三种情况下ViewGroup就会自己处理事件
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                                                    TouchTarget.ALL_POINTER_IDS);
        } else { //有两种情况mFirstTouchTarget不为空，表示找到合适的子View为target：
            //1、本次事件是ACTION_DOWN，遍历完ViewGroup所有的子View后找到了合适的子View为target；
            //2、本次事件是除了ACTION_DOWN以外的其他事件，但是在ACTION_DOWN时已经找到了合适的子View为target
           //所以接下来就直接把事件分发给mFirstTouchTarget的child处理处理就行
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            //mFirstTouchTarget是一个单链表结构
            while (target != null) {
                final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {//情况1的处理
                    //因为在找到target时已经调用过dispatchTransformedTouchEvent()了，表示该target的View已经消费了该事件，handle直接等于true
                    handled = true;
                } else {//情况2的处理
                     final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;//注意这个intercepted，如果为true，cancelChild为true，会导致子View收到一个ACTION_CANCEL, 表示子View的本次事件取消
                    //调用dispatchTransformedTouchEvent()方法把事件分发给target
                    if (dispatchTransformedTouchEvent(ev, cancelChild, target.child, target.pointerIdBits)) {
                        //handle的是否为true取决于子View的dispatchTouchEvent()返回值
                        handled = true;
                    }
                    //清空这个子View对应的target，导致该事件序列的后序事件该子View都无法再收到
                     if (cancelChild) {
                         //...
                         target.recycle();
                         target = next;
                         continue;
                     }
                }
                predecessor = target;
                target = next;
            }//end...while (target != null) 
            
         }//end...if (mFirstTouchTarget == null)
        
        //...
        
    }//end...if (onFilterTouchEventForSecurity(ev))
    
}
```

这个方法特别长，里面就是整个ViewGroup的事件分发逻辑，我知道大家也没有想看的欲望了，这个方法对应的流程图如下：

{% asset_img view3.png view3   %}

可以看到，**在同一个事件序列内（从down开始，到up结束）**，ViewGroup的dispatchTouchEvent()方法可以分为两大过程：**1、ACTION_DOWN事件的处理流程；2、除了ACTION_DOWN以外的事件处理流程**。下面跟着这两个流程分别走一遍。

### 2、ViewGroup处理ACTION_DOWN事件的流程

{% asset_img view4.png view4  %}

ACTION_DOWN事件的处理流程又可以分为两个流程即：**ViewGroup拦截事件(intercepted = true)与不拦截事件（intercepted = false）**。

看流程图，在dispatchTouchEvent()方法注释2中的if语句会决定 intercepted 的值，如下：

```java
//ViewGroup是否拦截本次事件标志
final boolean intercepted;
//2、如果本次事件是ACTION_DOWN 或者 mFirstTouchTarget为空
if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
    //2.1、子View是否禁止ViewGroup拦截事件标志
    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
    if (!disallowIntercept) {//如果子View允许ViewGroup拦截事件
        //调用onInterceptTouchEvent()方法询问ViewGroup是否拦截事件，intercepted的值由onInterceptTouchEvent(ev)决定
        intercepted = onInterceptTouchEvent(ev);
        //...
    } else {//如果子View禁止ViewGroup拦截事件
        intercepted = false;//intercepted值为false
    }
} else {
    //...
}
```

如果本次事件是ACTION_DOWN也会进入这个if分支，看注释2.1检查 mGroupFlags 中是否包含FLAG_DISALLOW_INTERCEPT标识，默认没有，即默认disallowIntercept为false，所以就会调用onInterceptTouchEvent()方法询问ViewGroup是否拦截事件，intercepted的值由onInterceptTouchEvent()决定，onInterceptTouchEvent()默认返回false，所以**intercepted = false**。

#### 2.1、intercepted = false

当DOWN事件没有被ViewGroup拦截，**intercepted = false**，它就会进入dispatchTouchEvent()方法注释3的if语句，如下：

```java
 //检查本次事件是否是ACTION_CANCEL
final boolean canceled = resetCancelNextUpFlag(this) || actionMasked == MotionEvent.ACTION_CANCEL;
//...
//3、如果本次事件不取消并且不拦截，就寻找合适的子View处理
if (!canceled && !intercepted) {
    //...
    //如果本次事件是ACTION_DOWN
    if (actionMasked == MotionEvent.ACTION_DOWN
        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {

        final int childrenCount = mChildrenCount;
        //如果target是null并且ViewGroup有子View，就寻找某个子View当mFirstTouchTarget
        if (newTouchTarget == null && childrenCount != 0) {
            final float x = ev.getX(actionIndex);
            final float y = ev.getY(actionIndex);
            final View[] children = mChildren;
            //从后往前逐个取出子View
            for (int i = childrenCount - 1; i >= 0; i--) {
                final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
                final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);
                //3.1、判断子View能否接受点击事件：子View可见或在播放动画，并且触摸点在子View范围内
                if (!canViewReceivePointerEvents(child)
                    || !isTransformedTouchPointInView(x, y, child, null)) {
                    ev.setTargetAccessibilityFocus(false);
                    continue;
                }
                //走到这里表示子View满足处理事件的条件
                //...
                //3.2、dispatchTransformedTouchEvent()里面会调用子View的dispatchTouchEvent()方法，在这个方法里把事件分发给子View
                if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                    //...
                    //3.3、如果dispatchTransformedTouchEvent()返回true，表示找到子View消费本次事件了，就会走到这里, 所以这个子View就被当作mFirstTouchTarget，这里会调用addTouchTarget()方法为mFirstTouchTarget赋值
                    newTouchTarget = addTouchTarget(child, idBitsToAssign);
                    alreadyDispatchedToNewTouchTarget = true;
                    break;
                }
                //...
            }//end...for（）

            //...

        }//end...if(newTouchTarget == null && childrenCount != 0)

    }//end...if(actionMasked == MotionEvent.ACTION_DOWN...)

}///end...if(!canceled && !intercepted)
```

如果是DOWN事件，假设ViewGroup有子View，就会进入for循环，ViewGroup就会遍历所有子View，先在注释3.1中判断这个子View是否满足接收事件的条件，如果不满足，就再找下一个子View，如果满足，就来到了注释3.2，然后调用dispatchTransformedTouchEvent()方法看这个子View是否消费DOWN事件。

dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)方法如下：

```java
//ViewGroup.java
//dispatchTransformedTouchEvent（）只需要关注两个参数：
//@params cancel 是否取消本次事件
//@params child 准备接收分发事件的子View
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel, View child, int desiredPointerIdBits) {
     final boolean handled;
    final int oldAction = event.getAction();
    //1、如果cancel为true，进入这个if分支
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        //设置ACTION_CANCEL事件
        event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
            handled = super.dispatchTouchEvent(event);
        } else {
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }
    //...
    //2、如果cancel为false，进入这个if分支
    if (child == null) {//如果child为空
        //调用 super.dispatchTouchEvent(event)，表示ViewGroup自己决定是否处理本次事件
        handled = super.dispatchTouchEvent(event);
    } else {//如果child不为空
        //...
        //调用child.dispatchTouchEvent(event)，表示让子View决定是否处理本次事件
        handled = child.dispatchTouchEvent(event);
    }
    return handled;
}
```

因为传入cancel为false，所以来带注释2的if分支，因为传入的child不为空，所以调用child.dispatchTouchEvent(event)，表示让子View决定是否处理本次事件，**到这里DOWN事件就传递给子View，如果子View是一个View，那么它的处理流程就像前面介绍的View的事件分发一样，如果子View是一个ViewGroup，那么它的处理流程就又是ViewGroup的事件分发**。

好了，假设子View消费这个事件，返回true，则dispatchTransformedTouchEvent()返回true，ViewGrou找到了要消费这个DOWN事件的子View，这时进入注释3.3，调用addTouchTarget(child, idBitsToAssign)方法，如下：

```java
//ViewGroup.java
private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {
    final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    //给mFirstTouchTarget赋值
    mFirstTouchTarget = target;
    return target;
}

//ViewGroup::TouchTarget
public static TouchTarget obtain(@NonNull View child, int pointerIdBits) {
    //...
    target.child = child;
    return target;
}
```

如果找到了要消费这个DOWN事件的子View，那么这个子View就会被赋值给mFirstTouchTarget的child字段，这个就相当于做了一个记录，当下一个事件到来时，如果发现mFirstTouchTarget不为空，我就可以直接把事件分发给mFirstTouchTarget中的View，就不用再去遍历子View了。那么mFirstTouchTarget是什么？它是一个TouchTarget类型，如下：

```java
///ViewGroup::TouchTarget
private static final class TouchTarget {
    //当前消费事件的View
    public View child;
    //它的下一个结点
    public TouchTarget next;
    //...
}
```

它是一个链表结构，为什么mFirstTouchTarget是一个链表？我的猜测是由于多点触控的存在，例如我5个手指可以同时触摸到列表的5个子View，如果5个子View都是要消费这个DOWN事件的话，那么就要用链表把它们记录起来，当下一个事件到来时，5个子View都能分发到事件。

好了，现在找到可以消费事件的子View了，并且mFirstTouchTarget也被赋值了，就一个break跳出for循环，直接来到dispatchTouchEvent()方法的注释4，如下：

```java
   //4、根据mFirstTouchTarget是否为null做出不同行为
if (mFirstTouchTarget == null) {
    //...
} else {//有两种情况mFirstTouchTarget不为空，表示找到合适的子View为target：
    //1、本次事件是ACTION_DOWN，遍历完ViewGroup所有的子View后找到了合适的子View为target；
    //2、本次事件是除了ACTION_DOWN以外的其他事件，但是在ACTION_DOWN时已经找到了合适的子View为target
    //所以接下来就直接把事件分发给mFirstTouchTarget的child处理就行
    TouchTarget predecessor = null;
    TouchTarget target = mFirstTouchTarget;
    //mFirstTouchTarget是一个单链表结构
    while (target != null) {
        final TouchTarget next = target.next;
        if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {//情况1的处理
            //因为在找到target时已经调用过dispatchTransformedTouchEvent()了，表示该target的View已经消费了该事件，handle直接等于true
            handled = true;
        } else {//情况2的处理
            //...
        }
        predecessor = target;
        target = next;
    }//end...while (target != null) 

}//end...if (mFirstTouchTarget == null)
```

mFirstTouchTarget不为空，就来到else分支，然后因为是DOWN事件，在上面的for循环中找到子View消费事件后alreadyDispatchedToNewTouchTarget赋值为true并且mFirstTouchTarget等于newTouchTarget实例，就来到情况1的处理的if分支，这里直接返回了true，因为上面在for循环中target的View已经消费了该事件，handle直接等于true。

到这里在DOWN事件下ViewGroup不拦截的情况下分析完毕。上面是假设找到了子View并且子View消费了事件，这样当下一次事件到来时mFirstTouchTarget不为空，就直接把这个事件给子View；但是如果上面是找到子View而这个子View不消费这个DOWN事件，即子View的dispatchTouchEvent()方法返回false，那么dispatchTransformedTouchEvent()返回false，就导致无法为mFirstTouchTarget赋值，mFirstTouchTarget为空，当下一次事件序列到来时，ViewGroup会直接处理，而不再转发给子View。这里得出一个结论：**子View如果不消费ACTION_DOWN事件，那么同一事件序列的其他事件都不会再交给它来处理，而是交给它的父ViewGroup处理；子View一旦消费ACTION_DOWN事件，那么同一事件序列的其他事件都会交给它处理**。

所以如果此时子View没有消费ACTION_DOWN事件，并且我重写了ViewGroup的onInterceptTouchEvent()并返回了true，那么ViewGroup就会开始拦截事件，接下来看在DOWN事件下ViewGroup拦截的情况，即**intercepted = true**。

#### 2.2、intercepted = true

{% asset_img view6.png view6   %}

如果ViewGroup拦截DOWN事件，那么**intercepted = true**，就不会进入dispatchTouchEvent()方法的注释3的if语句，这样在DOWN事件下ViewGroup就不会遍历它的子View，也就无法调用dispatchTransformedTouchEvent()找到要消费事件的子View，同理无法调用addTouchTarget()方法为mFirstTouchTarget赋值，就会导致在DOWN事件下mFirstTouchTarget为空，这样就直接来到了dispatchTouchEvent()方法的注释4的if语句，如下：

```java
 //检查本次事件是否是ACTION_CANCEL
 final boolean canceled = resetCancelNextUpFlag(this) || actionMasked == MotionEvent.ACTION_CANCEL;
//...   
//4、根据mFirstTouchTarget是否为null做出不同行为
if (mFirstTouchTarget == null) {//这一般有三种情况导致mFirstTouchTarget为空：
    //1、ViewGroup没有子View；
    //2、子View处理了ACTION_DOWN事件，但是在dispatchTouchEvent()返回了false；
    //3、ViewGroup在DOWN事件中的onInterceptTouchEvent(ev)返回了true
    //在这三种情况下ViewGroup就会自己处理事件
    //注意第三个参数传入null，表示ViewGroup自己处理事件
    handled = dispatchTransformedTouchEvent(ev, canceled, null,
                                            TouchTarget.ALL_POINTER_IDS);
} else {         
    //...           
}//end...if (mFirstTouchTarget == null)
```

很明显这里是情况3，所以没有找到子View，dispatchTransformedTouchEvent()方法的第三个参数为空，而第二个参数为false，因为不是ACTION_CANCEL事件，我们参考上面的dispatchTransformedTouchEvent()方法分析，如下：

```java
//ViewGroup.java
//dispatchTransformedTouchEvent（）只需要关注两个参数：
//@params cancel 是否取消本次事件
//@params child 准备接收分发事件的子View
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel, View child, int desiredPointerIdBits) {
     final boolean handled;
    //...
    //2、如果cancel为false，进入这个if分支
    if (child == null) {//如果child为空
        //调用 super.dispatchTouchEvent(event)，表示ViewGroup自己决定是否处理本次事件
        handled = super.dispatchTouchEvent(event);
    } else {//如果child不为空
        //...
        //调用child.dispatchTouchEvent(event)，表示让子View决定是否处理本次事件
        handled = child.dispatchTouchEvent(event);
    }
    return handled;
}
```

里面就会调用 super.dispatchTouchEvent(event)，表示ViewGroup自己决定是否处理本次事件，ViewGroup的父类是View，所以super.dispatchTouchEvent(event)里面的处理逻辑就是View的事件分发的处理逻辑，见前面分析的View的事件分发。

到这里在DOWN事件下ViewGroup拦截的情况分析完毕。这里得出一个结论：**ViewGroup如果在onInterceptTouchEvent()方法的ACTION_DOWN事件中返回true，那么整个事件序列都会交给ViewGroup处理，不再交给子View**。

我们回到dispatchTouchEvent()方法，还有一点要注意的是在ACTION_DOWN下不管拦截还是不拦截都会进入dispatchTouchEvent()方法中注释1的if语句，如下：

```java
 //1、如果本次事件是ACTION_DOWN
if (actionMasked == MotionEvent.ACTION_DOWN) {
    //置空mFirstTouchTarget
    cancelAndClearTouchTargets(ev);
    //清除mGroupFlags中的FLAG_DISALLOW_INTERCEPT标志位，这个标志等同于下面的disallowIntercept
    resetTouchState();
}
```

这个if语句的作用就是防止前一次事件序列对本次事件序列造成影响，所以它会向先调用 cancelAndClearTouchTargets(ev)清空mFirstTouchTarget，然后调用resetTouchState()清除FLAG_DISALLOW_INTERCEPT标志位，因为ACTION_DOWN事件是一个新的事件序列的开始，所以dispatchTouchEvent()方法首先要做的就是判断是不是迎来了一个新的事件序列，所以要判断该事件是否是ACTION_DOWN 事件，如果是 ACTION_DOWN 事件，作为一个事件序列的开头，应当要消除前面的事件序列可能留下的影响。关于FLAG_DISALLOW_INTERCEPT标志位后面会讲。

到这里ViewGroup处理ACTION_DOWN事件的流程分析完毕，下面我们来看除了ACTION_DOWN以外的事件的处理流程。

### 3、ViewGroup处理除了ACTION_DOWN以外的事件的流程

{% asset_img view5.png view5 %}

ACTION_DOWN事件的处理流程又可以分为两个流程即：**mFirstTouchTarget != null与mFirstTouchTarget == null**。你会发现intercepted这个标记位似乎已经没有多大作用， 它如果是true，它根本不会进入dispatchTouchEvent()方法的注释3，就算是false进入了dispatchTouchEvent()方法的注释3，它也不会满足注释3.1的条件。所以我们就直接来到注释4。

#### 3.1、mFirstTouchTarget == null

{% asset_img view7.png view7  %}

```java
 //检查本次事件是否是ACTION_CANCEL
 final boolean canceled = resetCancelNextUpFlag(this) || actionMasked == MotionEvent.ACTION_CANCEL;
//...   
//4、根据mFirstTouchTarget是否为null做出不同行为
if (mFirstTouchTarget == null) {//这一般有三种情况导致mFirstTouchTarget为空：
    //1、ViewGroup没有子View；
    //2、子View处理了ACTION_DOWN事件，但是在dispatchTouchEvent()返回了false；
    //3、ViewGroup在DOWN事件中的onInterceptTouchEvent(ev)返回了true
    //在这三种情况下ViewGroup就会自己处理事件
    //注意第三个参数传入null，表示ViewGroup自己处理事件
    handled = dispatchTransformedTouchEvent(ev, canceled, null,
                                            TouchTarget.ALL_POINTER_IDS);
} else {         
    //...           
}//end...if (mFirstTouchTarget == null)
```

这里1、2、3情况都有可能发生，**从ACTION_DOWN的处理流程我们知道为mFirstTouchTarget赋值的过程只会在处理ACTION_DOWN事件的时候出现**，所以如果在处理ACTION_DOWN事件的时候ViewGroup没有子View，不会进入for循环，导致mFirstTouchTarget为空；如果ViewGroup有子View，进入了for循环，但是View不消费DOWN事件，即在dispatchTouchEvent()返回了false，导致无法调用addTouchTarget()方法为mFirstTouchTarget赋值，导致mFirstTouchTarget为空；ViewGroup在DOWN事件中的onInterceptTouchEvent(ev)返回了true，不会进入注释3的if语句，导致mFirstTouchTarget为空；所以在处理ACTION_DOWN事件的时候没有找到mFirstTouchTarget，就会导致在除了ACTION_DOWN其他事件到来时mFirstTouchTarget == null，这里就直接让ViewGroup自己处理事件了。

#### 3.2、mFirstTouchTarget != null

{% asset_img view8.png view8 %}

```java
   //4、根据mFirstTouchTarget是否为null做出不同行为
if (mFirstTouchTarget == null) {
    //...
} else {//有两种情况mFirstTouchTarget不为空，表示找到合适的子View为target：
    //1、本次事件是ACTION_DOWN，遍历完ViewGroup所有的子View后找到了合适的子View为target；
    //2、本次事件是除了ACTION_DOWN以外的其他事件，但是在ACTION_DOWN时已经找到了合适的子View为target
    //所以接下来就直接把事件分发给mFirstTouchTarget的child处理就行
    TouchTarget predecessor = null;
    TouchTarget target = mFirstTouchTarget;
    //mFirstTouchTarget是一个单链表结构
    while (target != null) {
        final TouchTarget next = target.next;
        if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {//情况1的处理
           //...
        } else {//情况2的处理
             //4.1
              final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;//注意这个intercepted，如果为true，cancelChild为true，会导致子View收到一个ACTION_CANCEL, 表示子View的本次事件取消
                    //4.2、调用dispatchTransformedTouchEvent()方法把事件分发给target
                    if (dispatchTransformedTouchEvent(ev, cancelChild, target.child, target.pointerIdBits)) {
                        //handle的是否为true取决于子View的dispatchTouchEvent()返回值
                        handled = true;
                    } 
                    //4.3、清空这个子View对应的target，导致该事件序列的后序事件该子View都无法再收到
                     if (cancelChild) {
                         //...
                         target.recycle();
                         target = next;
                         continue;
                     }
                }
                predecessor = target;
                target = next;
        }
        predecessor = target;
        target = next;
    }//end...while (target != null) 

}//end...if (mFirstTouchTarget == null)
```

mFirstTouchTarget != null，表示在处理ACTION_DOWN事件的时候已经找到mFirstTouchTarget，就会进入注释4的else分支，这里是情况2，就会进入情况2的处理的else分支，注释4.1的cancelChild这个值会决定子View是收到ACTION_CANCEL事件还是其他事件，而cancelChild的值取决于intercepted的值，所以如果ViewGroup在除了ACTION_DOWN以外的其他事件中的onInterceptTouchEvent(ev)方法返回了true，导致intercepted = true，从而cancelChild = true，而如果ViewGroup一直保持默认状态，intercepted = false，从而cancelChild = false，紧接着在注释4.2把cancelChild和target.child传进了dispatchTransformedTouchEvent()方法中。

我再贴一下dispatchTransformedTouchEvent()方法的代码，如下：

```java
//ViewGroup.java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel, View child, int desiredPointerIdBits) {
     final boolean handled;
    final int oldAction = event.getAction();
    //1、如果cancel为true，进入这个if分支
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        //设置ACTION_CANCEL事件
        event.setAction(MotionEvent.ACTION_CANCEL);
        //分发ACTION_CANCEL事件
        if (child == null) {
            handled = super.dispatchTouchEvent(event);
        } else {
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }
    //...
    //2、如果cancel为false，进入这个if分支
    if (child == null) {
        //调用 super.dispatchTouchEvent(event)，表示ViewGroup自己决定是否处理本次事件
        handled = super.dispatchTouchEvent(event);
    } else {
        //...
        //调用child.dispatchTouchEvent(event)，表示让子View决定是否处理本次事件
        handled = child.dispatchTouchEvent(event);
    }
    return handled;
}
```

可以看到如果cancel为true，进入注释1这个if分支，里面会set一个ACTION_CANCEL事件，然后传递给target记录的子View，如果cancel为true，进入注释2这个else分支，调用child.dispatchTouchEvent(event)，表示让target记录的子View决定是否处理本次事件，前面已经讲过了。

好，现在我们走出dispatchTransformedTouchEvent()方法，来到注释4，如果cancelChild为true，就会调用TouchTarget的recycler()方法回收这个target，这样做的后果是什么呢？这样相当于清空了mFirstTouchTarget，当下一次事件到来时mFirstTouchTarget == null，ViewGroup直接处理事件，不会再分发给子View。

到这里ViewGroup处理除了ACTION_DOWN以外事件的流程分析完毕。

### 4、子View如何禁止ViewGroup拦截事件

前面的分析都是默认子View不禁止ViewGroup拦截事件，所以ViewGroup可以通过onInterceptTouchEvent()返回true从而拦截下子View的事件，但此时子View希望依然能够响应这些事件该怎么办呢？Android给我们提供了一个方法：requestDisallowInterceptTouchEvent(boolean) 用于设置是否允许拦截，如下：

```java
//ViewGroup.java
@Override
public void requestDisallowInterceptTouchEvent(boolean disallowIntercept) {
    //...
    if (disallowIntercept) {
        mGroupFlags |= FLAG_DISALLOW_INTERCEPT;
    } else {
        mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
    }
    // Pass it up to our parent
    if (mParent != null) {
        mParent.requestDisallowInterceptTouchEvent(disallowIntercept);
    }
}
```

当子View调用getParent.requestDisallowInterceptTouchEvent(true)，mGroupFlags就会有FLAG_DISALLOW_INTERCEPT标识，当子View调用getParent.requestDisallowInterceptTouchEvent(false)，mGroupFlags就会清除FLAG_DISALLOW_INTERCEPT标识，那么FLAG_DISALLOW_INTERCEPT标识又是怎么控制ViewGroup的拦截的呢？如下：

```java
final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
if (!disallowIntercept) {//如果子View允许ViewGroup拦截事件
    //调用onInterceptTouchEvent()方法询问ViewGroup是否拦截事件，intercepted的值由onInterceptTouchEvent(ev)决定
    intercepted = onInterceptTouchEvent(ev);
    //...
} else {//如果子View禁止ViewGroup拦截事件
    intercepted = false;//intercepted值为false
}
```

子View通过调用getParent.requestDisallowInterceptTouchEvent(true)，来禁止ViewGroup拦截除了ACTION_DOWN以外的其他事件，这样当下一个事件到来时就会交给这个子View，

为什么是除了ACTION_DOWN以外的其他事件？因为ACTION_DOWN事件是事件序列的开始，ACTION_DOWN事件会先经过ViewGroup的onInterceptTouchEvent()方法，从**ACTION_DOWN事件的处理流程 - intercepted = true**我们知道，如果ViewGroup一开始在onInterceptTouchEvent()的ACTION_DOWN返回true，它就不会进入dispatchTouchEvent()方法的注释3的if语句，这样在DOWN事件下就无法找到mFirstTouchTarget，这样当同一个事件序列的其他事件到来时，mFirstTouchTarget == null，这样ViewGroup只能把事件交给自己处理，无法传递给子View，也就无法调用子View的dispatchTouchEvent()方法，这样子View在dispatchTouchEvent()方法中调用getParent.requestDisallowInterceptTouchEvent(true)就没有意义了。

### 5、小结

从ViewGroup的事件分发中得出几个结论：

1、ViewGroup如果在onInterceptTouchEvent()方法的ACTION_DOWN事件中返回true，那么整个事件序列都会交给ViewGroup处理，不再交给子View，从而导致无法调用子View的dispatchTouchEvent()方法，导致子View调用getParent.requestDisallowInterceptTouchEvent(true)失效。

2、ViewGroup如果在onInterceptTouchEvent()方法中一旦拦截除了ACTION_DOWN的事件，那么子View将会收到一个ACTION_CANCEL事件，并且接下来的事件都是交给ViewGroup处理。

3、1、2点的含有都是ViewGroup决定拦截事件，那么一旦ViewGroup决定拦截事件，那么接下来的事件都是交给ViewGroup处理，并且ViewGroup的onInterceptTouchEvent()方法在这个事件序列内不会再调用，这说明ViewGroup的onInterceptTouchEvent()方法不是每次都调用。

3、在ViewGroup中ACTION_DOWN 事件负责寻找 target，即寻找能够消费ACTION_DOWN事件的子View，如果找到，那么接下来同一事件序列内的所有事件都会交给这个子View处理，不再交给ViewGroup；如果没有找到，有两种情况：1、ViewGroup没有子View，2、子View处理了ACTION_DOWN事件，但是在dispatchTouchEvent()返回了false，那么接下来同一事件序列下的所有事件都是ViewGroup自己处理。

4、子View如果不消费ACTION_DOWN事件，那么同一事件序列的其他事件都不会再交给它来处理，而是交给它的父ViewGroup处理；子View一旦消费ACTION_DOWN事件，如果ViewGroup不拦截，那么同一事件序列的其他事件都会交给它处理，

5、当调用super.dispatchTouchEvent(event)就代表ViewGroup开始自己处理事件，里面的逻辑和View的事件分发一样。

## 结语

当点击事件到达ViewGroup时，它的dispatchTouchEvent()方法就会被调用，如果这个ViewGroup的onInterceptTouchEvent()方法返回true，就表示它要拦截当前事件，接下来这个事件序列内的事件都会交给它处理，即super.dispatchTouchEvent()方法得到调用；如果这个ViewGroup的onInterceptTouchEvent()方法返回false，就表示它不拦截当前事件，这时当前事件就会传递给它的子View，接着子View的dispatchTouchEvent()方法就会被调用，如果子View是一个View，那么它的处理流程就像前面介绍的View的事件分发一样，如果子View是一个ViewGroup，那么它的处理流程就又是ViewGroup的事件分发，如此递归，**从上到下**，直到整颗View树都收到事件，接下来递归返回，**从下到上**，每一层的返回值都决定是否消费本次事件，如果消费，返回true，它的上一层就无法处理这个事件，如果不消费，返回false，它的上一层又继续传给上一层，直到根视图。

View的事件分发小结和ViewGroup的事件分发小结都可以在源码中找到证明，可以自行验证一下，本文通过源码 + 流程图 说明了整个View的事件分发体制， 

参考资料：

[Android事件分发完全解析之事件从何而来](https://blog.csdn.net/qq_43660664/article/details/84026785)

[通过流程图来分析Android事件分发](https://blog.csdn.net/u010707039/article/details/85211658#commentBox)