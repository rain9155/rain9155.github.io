---
date: 2019-07-22 19:57:11
title: View的工作原理
tags: view
categories: View机制
---

## 前言

在Android中View一直扮演着一个很重要的角色，它是我们开发中视觉的呈现，我平常也使用着Android提供的丰富且功能强大的控件，有时候遇到一个很炫酷的自定义View的开源库，我们也是拿来主义，时间长了你就会发现你只是一个只会使用控件和依赖被人开源库的程序员，这并不是一个开发者，所以我们并不能只满足于使用，我们要理解它背后的工作原理和流程，这样才能自己做出一个属于自己的控件，一直都说自定View是Android进阶中的一道门槛，当其实自定义View当你理解了它的原理后，你就会发现它也不过如此。本文将从源码的角度探讨View工作的三大流程，对View做进一步的认识。俗话说的好：源码才是最好的老师。

<!--more-->

	本文代码基于Android8.0，相关源码位置如下：
	frameworks/base/core/java/android/*.java(*代表View, ViewGroup, ViewRootImpl)
	frameworks/base/core/java/android/FrameLayout.java

## View何时开始绘制？- requestLayout()

提到View，就不得不讲起Window，在[Window,WindowManager和WindowManagerService之间的关系](https://rain9155.github.io/2019/03/22/Window,%20WindowManager和WindowManagerService之间的关系/)文章中讲过，Widnow是View得载体，在ViewRootImpl的setView方法中添加Winodw到WMS之前，会先调用requestLayout绘制整颗View Hierarchy的绘制，如下：

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
        if (!mTraversalScheduled) {//防止同一帧绘制多次
            mTraversalScheduled = true;
            //拦截同步Message，优先处理异步Message
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
             //1、Choreographer回调，里面执行最终会执行mTraversalRunnable中的绘制任务
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

doTraversal()里面会执行performTraversals方法，点开doTraversal方法看一下，如下：

```java
//ViewRootImpl.java
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        //移除拦截同步Message屏障
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        //1、今天的主角，performTraversals()方法
        performTraversals();
        //...
    }
}
```

在doTraversal() 方法里面我们终于看到我们熟悉的方法：**performTraversals()**。

## View树绘制的起点 - performTraversals()

performTraversals()它是整个View Hierarchy绘制的起点，它里面会执行View绘制的三大工作流程，我们先看一下精简版的performTraversals方法，如下：

```java
//ViewRootImpl.java
private void performTraversals() {
    //mView是在View与ViewRootImpl建立关联的时候被赋值的，即调用ViewRootImpl的setView方法时，它代表着View Hierarchy的根节点，即根视图
    final View host = mView;
    //...
    WindowManager.LayoutParams lp = mWindowAttributes;
    //desiredWindowWidth和desiredWindowHeight分别代表着屏幕的宽度和高度
    int desiredWindowWidth;
    int desiredWindowHeight;
    //...
    if (mLayoutRequested) {
        final Resources res = mView.getContext().getResources();
        //...
        //1、这里调用了measureHierarchy方法，里面会调用performMeasure方法，执行View Hierarchy的measure流程，见下方
        windowSizeMayChange |= measureHierarchy(host, lp, res,
     	                                         desiredWindowWidth, desiredWindowHeight);
        //...
    }
    //...
    if(didLayout){
          //2、这里调用了performLayout方法，执行View Hierarchy的layout流程
         performLayout(lp, mWidth, mHeight);
        //...
    }
    //...
    if (!cancelDraw && !newSurface) {
        //...
        //3、这里调用了performDraw方法，执行View Hierarchy的draw流程
   		performDraw();
    }
    //...
}

 private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,  final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
     int childWidthMeasureSpec;
     int childHeightMeasureSpec;
     //...
     //1.1、顶级View在调用performMeasure方法之前，会先调用getRootMeasureSpec方法来生成自身宽和高的MeasureSpec
     childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
     childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
     //1.2、这里调用performMeasure方法，执行View Hierarchy的measure流程
     performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
 }
```

performTraversals方法里面非常复杂，我们看的时候千万不要深究其中的细节，不然就走火入魔了，我们找出个整体框架就行，我们先看注释1、2、3，可以看到依此调用**measureHierarchy() -> performLayout() -> performDraw()，**而measureHierarchy()里面最终调用performMeasure()，所以performTraversals()可以看作依此调用了**performMeasure() -> performLayout() -> performDraw()，**分别对应顶级View的**measure、layout和draw流程，**顶级View可以理解为View Hierarchy的根节点，它一般是一个ViewGroup，就像Activity的DecorView一样。

> ps：
>
> 1、在performTraversals()方法中，performMeasure()可能会执行多次，而performLayout()和performDraw()最多执行一次。
>
> 2、本文讨论的顶级View你可以把它类比成Activity的DecorView，但是它其实就是View树的根结点，DecorView也是Activity中View树的根结点。

接下来我们就照着performTraversals() 中的整体框架来讲解View工作的三大流程。

## View的测量流程 - performMeasure()

### 1、MeasureSpec

讲解View的measure流程前，不得不先讲解一下MeasureSpec的含义，MeasureSpec是一个32位的int值，它是View的一个内部类，它的高2位代表着SpecMode，表示测量模式，它的低30位表示SpecSize，表示测量大小，系统通过位运算把SpecMode和SpecSize合二为一组成一个32位int值的MeasureSpec。

下面看一下MeasureSpec的里面组成，如下：

```java
//View.java
public static class MeasureSpec {
    //左移位数
    private static final int MODE_SHIFT = 30;
    //位掩码
    private static final int MODE_MASK  = 0x3 << MODE_SHIFT;
    //代表着三种SpecMode
    public static final int UNSPECIFIED = 0 << MODE_SHIFT;
    public static final int EXACTLY     = 1 << MODE_SHIFT;
    public static final int AT_MOST     = 2 << MODE_SHIFT;

    //makeMeasureSpec方法是把SpecMode和SpecSize通过位运算组成一个MeasureSpec并返回
    public static int makeMeasureSpec(int size,int mode) {
        if (sUseBrokenMakeMeasureSpec) {
            return size + mode;
        } else {
            return (size & ~MODE_MASK) | (mode & MODE_MASK);
        }
    }

    //getMode方法是从给定的MeasureSpec中取出SpecMode
    public static int getMode(int measureSpec) {
        return (measureSpec & MODE_MASK);
    }

    //getSize方法是从给定的MeasureSpec中取出SpecSize
    public static int getSize(int measureSpec) {
        return (measureSpec & ~MODE_MASK);
    }
}
```

可以看到MeasureSpec提供了三个工具方法分别用来组合MeasureSpec、从MeasureSpec中取出SpecMode、从MeasureSpec中取出SpecSize，其中SpecMode有三种取值，如下：

* UNSPECIFIED：它表示父容器对子View的绘制的大小没有任何限制，要多大给多大，这种情况一般适用于系统内部，表示一种测量状态。
* EXACTLY：它表示父容器已经测量出子View需要的精确大小SpecSize，这个时候View的最终大小就是SpecSize的值，它对应于LayoutParams中match_parcent和具体的数值这两种模式。
* AT_MOST：它表示父容器为子View的大小指定了一个最大值SpecSize，这个时候View的大小不能大于这个值，它对应于LayoutParams中的wrap_content这种模式。

#### 1.1 如何确定View的MeasureSpec？

除了顶级View，其他View的MeasureSpec都是由父容器的MeasureSpec和自身的LayoutParams共同决定的，LayoutParams就是你平时在编写View的xml属性时那些带有**layout_XX**前缀开头的布局属性，对于顶级View和在View树中子View的MeasureSpec的生成规则有点不一样，见下面分析：

##### 1.1.1、顶级View的MeasureSpec的创建 - getRootMeasureSpec()

由于顶级View是View树的根结点，所以它没有父容器，所以它的MeasureSpec是由屏幕窗口的尺寸和自身的LayoutParams来共同决定，上面注释1.1我们讲到顶级View在调用performMeasure方法之前，会先调用ViewRootImpl的getRootMeasureSpec方法来生成自身宽和高的MeasureSpec，我们来看一下getRootMeasureSpec方法，如下：

```java
//ViewRootImpl.java
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {
        case ViewGroup.LayoutParams.MATCH_PARENT://如果是MATCH_PARENT,那么就是EXACTLY
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT://如果是WRAP_CONTENT,就是AT_MOST
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default://如果是固定的值,也是EXACTLY
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
    }
    return measureSpec;
}
```

windowSize就是是传入的desiredWindowWidth或desiredWindowHeight，它表示屏幕的大小，rootDimension就是传入的屏幕窗口的LayoutParams的大小模式，对应我们平时写的layout_width或layout_height属性，该属性无非就三个值：match_parent、wrap_content和固定的数值，所以从getRootMeasureSpec方法可以看到，顶级View的MeasureSpec的创建规则如下：

{% asset_img view2.png view2 %}

其中rootSize表示顶级View大小。


#####  1.1.2、子View的MeasureSpec的创建 - getChildMeasureSpec()

在1中，顶级View的MeasureSpec已经创建好了，这时候就要根据这个MeasureSpec去生成子View的MeasureSpec，子View的MeasureSpec的创建是从ViewGroup的measureChildWithMargins方法开始，如下：

```java
//ViewGroup.java
rotected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed, int parentHeightMeasureSpec, int heightUsed) {
    //得到子View的margin
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
    //1、这里调用了getChildMeasureSpec方法，里面就是创建子View的MeasureSpec，这里创建子View宽的MeasureSpec
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec, mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin + widthUsed, lp.width);
    //同理，这里创建子View高的MeasureSpec
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,  mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin + heightUsed, lp.height);
    //如果子View是一个ViewGroup，递归measure下去
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}

```

上述方法会对子View进行measure，由注释1得知，在调用子View的measure方法前，会先调用getChildMeasureSpec方法获得子View的MeasureSpec，从getChildMeasureSpec方法的参数可以看出，子View的MeasureSpec的创建与父容器的MeasureSpec和子View本身的LayoutParams有关，此外还和View的margin及padding有关，下面我们来看ViewGroup的getChildMeasureSpec方法，如下：

```java
//ViewGroup.java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    //取出父容器的测量模式specMode
    int specMode = MeasureSpec.getMode(spec);
    //取出父容器的测量大小specSize
    int specSize = MeasureSpec.getSize(spec);
	// padding是指父容器中已占用的空间大小，因此子View最大可用大小size == 父容器剩余大小 == 父容器的尺寸减去padding 
    int size = Math.max(0, specSize - padding);
    int resultSize = 0;
    int resultMode = 0;
    switch (specMode) {
        case MeasureSpec.EXACTLY://如果父容器是EXACTLY
            if (childDimension >= 0) {//如果子View的LayoutParams是固定大小
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {//如果子View的LayoutParams是MATCH_PARENT	//子View的MeasureSpec为父容器剩余大小 + EXACTLY
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {//如果子View的LayoutParams是WRAP_CONTENT
                //子View的MeasureSpec为父容器剩余大小 + AT_MOST
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;
        case MeasureSpec.AT_MOST://如果父容器是AT_MOST
            if (childDimension >= 0) {
                //子View的MeasureSpec为子View大小 + EXACTLY
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                //子View的MeasureSpec为父容器剩余大小 + AT_MOST
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
               //子View的MeasureSpec为父容器剩余大小 + AT_MOST
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;
        case MeasureSpec.UNSPECIFIED://如果父容器是UNSPECIFIED，这个平时开发用不到
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
    }
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

可以看到getChildMeasureSpec方法里面的逻辑还是很清楚的，首先根据父容器的测量模式specMode分为三大类：**EXACTLY、AT_MOST和UNSPECIFIED，**每一类又和子View的LayoutParams的的三种大小模式：**固定大小、MATCH_PARENT和WRAP_CONTENT**组合，所以总共有3 X 3 = 9种组合，所以根据getChildMeasureSpec方法可以得出子View的MeasureSpec的创建规则如下：

{% asset_img view3.png view3 %}

其中childSize表示子View的大小，parentSize表示父容器剩余大小。

### 2、View和ViewGroup的measure流程

分析完View的MeasureSpec的创建后，我们继续回到View的measure流程，大家都知道ViewGroup是继承自View的，所以View的measure流程，分为两种情况，一种是View的measure流程，一种是ViewGroup的measure流程，但是不管是View的measure流程还是ViewGroup的measure流程都是从ViewRootImpl的performMeasure()开始，并且都会先调用View的measure方法，如下：

```java
//ViewRootImpl.java 
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    //...
    //1、调用了View的measure方法
    mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

我们继续看View的measure方法，如下：

```java
//View.java
  public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
      //...
      //1、调用了onMeasure方法
      onMeasure(widthMeasureSpec, heightMeasureSpec);
	 //...
  }
```

可以看到measure方法是一个final方法，说明这个方法不能够被子类重写，这个方法把measure的具体过程交给了onMeasure方法去实现，所以View和ViewGroup的measure流程的差异就从这个onMeasure方法开始，见下面分析。

#### 2.1、View的measure流程 

从上述知道View的measure起点在View的measure方法中，并且View的measure方法会调用View的onMeasure方法，**View::measure() -> View::onMeasure()**，所以我们直接看onMeasure方法在View中的实现，如下：

```java
//View.java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    //1、 如果View没有重写onMeasure方法，则会调用setMeasuredDimension方法设置宽高，在设置之前先调用getDefaultSize方法获取默认宽高
    setMeasuredDimension(
        			    getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                         getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec)
    );
}
```

View中的onMeasure方法的默认实现是先调用getDefaultSize方法获取默认宽高，然后再调用调用setMeasuredDimension方法设置View的宽高，当调用setMeasuredDimension方法设置View的宽高后，就可以通过getMeasureWidth()或getMeasureHeight()获得View测量的宽高，所以我们先看一下 getDefaultSize()方法是如何获取默认的宽高，该方法源码如下：

```java
//View.java
public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);
        switch (specMode) {
        //如果specMode是UNSPECIFIED，返回的大小就是传进来的size，而这个size就是通过getSuggestedMinimumWidth()或getSuggestedMinimumHeight()方法获得的
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        //如果specMode是AT_MOST或EXACTLY，返回的大小就是MeasureSpec中的specSize
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }

```

getDefaultSize方法的逻辑很简单，除了UNSPECIFIED这种模式，其他测量模式都返回MeasureSpec中的specSize，而这个specSize就等于父容器给View测量后的大小，所以我们可以得出一个结论：**直接继承View写自定义控件时需要重写onMeasure方法并设置wrap_content时自定义View自身的大小，这是因为如果自定义View在xml文件写了layout_XX = wrap_content这个属性，那么在创建它的MeasureSpec时，它的specMode就会等于AT_MOST，而从getDefaultSize方法看出，如果specMode是AT_MOST或EXACTLY，它们两个返回的值是一样的，都是MeasureSpec中的specSize，通过上面所讲的子View的MeasureSpec的创建规则可知specSize是等于parentSize即父容器剩余的大小，这样就会造成这个自定义View会填充满整个父容器，效果和match_parent一样，并不按你想象那样的大小**。所以以后在自定义View时，如果有wrap_content这个场景，就要重写onMeasure方法，可以参考下面的模板，如下：

```java
//View.java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {        			      	 
     int measureWidth = MeasureSpec.getSize(widthMeasureSpec); 
     int measureHeight = MeasureSpec.getSize(heightMeasureSpec);  
     int measureWidthMode = MeasureSpec.getMode(widthMeasureSpec);   
     int measureHeightMode = MeasureSpec.getMode(heightMeasureSpec);  
     int width, height;
     //经过计算，控件所占的宽和高分别对应width和height
     // …………   
     //我们只需要在View为wrap_content时设置我们经过计算得出的View的默认宽高width和height即可
     //其他模式如EXACTLY，就直接设置父容器给我们测量出来的宽高即可
 	 setMeasuredDimension(
         (measureWidthMode == MeasureSpec.AT_MOST) ? width : measureWidth , 
         (measureHeightMode == MeasureSpec.AT_MOST) ? height : measureHeight
     );
}

```

讲完了getDefaultSize()中AT_MOST和EXACTLY模式情况，接着讲UNSPECIFIED这种模式的情况，从getDefaultSize方法中可以看出如果specMode是UNSPECIFIED，返回的大小就是传进来的size，而这个size就是通过getSuggestedMinimumWidth()或getSuggestedMinimumHeight()方法获得的，所以我们以getSuggestedMinimumWidth方法为例子，看一些如果获取在UNSPECIFIED模式下的宽，getSuggestedMinimumHeight()方法同理，getSuggestedMinimumWidth方法源码如下：

```java
//View.java 
protected int getSuggestedMinimumWidth() {
    //根据View有无背景返回大小，getMinimumWidth()见下方
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}

//Drawable.java
public int getMinimumWidth() {
    //getIntrinsicWidth()返回Drawable的宽，默认返回-1
    final int intrinsicWidth = getIntrinsicWidth();
    return intrinsicWidth > 0 ? intrinsicWidth : 0;
}
```

mBackground就等于View的背景，即android:background属性，mMinWidth就等于你在View的xml布局中写了“android:minWidth”这个属性，mBackground.getMinimumWidth()就是获取View的背景的宽度，所以我们得出结论：**在UNSPECIFIED模式下，如果View没有设置背景，那么View的宽就等于android:minWidth，如果View设置了背景，那么View的宽就等于View的背景background的宽和android:minWidth的最大值，高度同理**。

> View的onMeasure方法执行完后，就可以通过getMeasureWidth()或getMeasureHeight()获得View测量的宽高，但是有可能会不准确，因为有时候系统会进行多次measure，才能确定最终测量宽高，所以最好是在onLayout方法中去获取View的宽高。

#### 2.2、ViewGroup的measure流程 (以FrameLayout为例) 

从上述知道ViewGroup的measure起点也在View的measure方法中，而View的measure方法会调用View的onMeasure方法，ViewGroup继承自View，但是它是一个抽象类并没有重写View的onMeasure方法，而是由ViewGroup的子类如LinearLayout、FrameLayout等重写onMeasure方法以实现不同的measure流程，这里以FrameLayout为例，**View::measure() -> FrameLayout::onMeasure() **，我们来看FrameLayout的onMeasure方法，如下：

```java
//FrameLayout.java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    //获取子View的个数
    int count = getChildCount();
    //...
    //遍历所有子View，测量每个子View的大小
    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (mMeasureAllChildren || child.getVisibility() != GONE) {//如果子View可见
            //1、调用ViewGroup的measureChildWithMargins方法，测量子View的大小
            measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            //子View测量完后，FrameLayout就可以通过View的getMeasuredWidth或getMeasuredHeight获得子View的宽高，从而得出自己的宽高
            //根据FrameLayout的叠加特性，它自身的测量宽高就是所有子View宽高中的最大值
            maxWidth = Math.max(maxWidth,
                                child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
            maxHeight = Math.max(maxHeight,
                                 child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
        }
    }
    //...
}
```

可以看到与View的onMeasure方法不同的是，FrameLayout的onMeasure方法是遍历它所有的子View，然后逐个测量子View的大小，这个测量子View是通过注释1的measureChildWithMargins方法来完成，这个方法已经在上面子View的MeasureSpec的创建中讲过一点，measureChildWithMargins方法是在FrameLayout的父类ViewGroup中，如下：

```java
protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed, int parentHeightMeasureSpec, int heightUsed) {
    //省略的这部分在上面已经讲过，主要是创建子View的MeasureSpec（childWidthMeasureSpec, childHeightMeasureSpec）
    //...
    //1、调用子View的measure方法，叫子View自己测量自己
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

measureChildWithMargins方法中首先会根据父容器传进来的parenXXMeasureSpec来创建子View的childXXMeasureSpec，然后调用子View的measure方法，把测量子View的任务又推给了子View，这个过程又回到了2.1所讲的View的measure流程，就不再赘述，所有子View测量完后，ViewGroup就可以得出自己的测量宽高。

### 3、小结

measure流程是三大流程中最复杂的一个，它的整体流程是：从ViewRootImp的performTraversals()方法进入performMeasure()方法，开始整颗View树的测量流程，在performMeasure方法里面会调用View的measure方法，然后measure方法会调用onMeasure方法，如果是View就直接开始测量，设置View的宽高，如果是ViewGroup，则在onMeasure方法中则会对所有的子View进行measure过程，如果子View是一个ViewGroup，那么继续向下传递，直到所有的View都已测量完成。如图：

{% asset_img view4.png view4 %}

measure过后就可以通过getMeasureWidth()或getMeasureHeight()获得View测量的宽高。

## View的布局流程 - performLayout()

前面讲解了View的measure过程，如果你理解了，那么View的布局过程也很容易理解的，和measure相似，View的布局过程是从ViewRootImpl的performLayout()开始的，如下：

```java
//ViewRootImpl.java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth, int desiredWindowHeight) {
    //...
    final View host = mView;
    //...
    //1、调用了顶级View的layout方法
     host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
    //...
}
```

在performLayout中主要调用了顶级View的layout方法，顶级View的实例有可能是View也有可能是ViewGroup，但是这个layout方法是在View中，它不像measure方法那样，它不是final修饰，所以它可以被重写，并且ViewGroup重写了layout方法，我们先看一下ViewGroup中的layout方法，如下：

```java
//ViewGroup.java
@Override
public final void layout(int l, int t, int r, int b) {
    if (...) {
        //...
         //1、ViewGroup中的重写的layout方法还是调用了父类即View的layout方法
        super.layout(l, t, r, b);
    } else {
       //...
    }
}
```

可以看到ViewGroup重写的layout方法只是做了一些判断，然后最终还是还是调用了父类即View的layout方法，所以我们直接看View的layout方法即可。

### 1、View和ViewGroup的layout流程

View的layout方法如下：

```java
//View.java
public void layout(int l, int t, int r, int b) {
    // 注意传进来的四个参数：
    // l 表示子View的左边缘相对于父容器的上边缘的距离
    // t 表示子View的上边缘相对于父容器的上边缘的距离
    // r 表示子View的右边缘相对于父容器的右边缘的距离
    // b 表示子View的下边缘相对于父容器的下边缘的距离
    //...
    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;
    //1、调用setFrame方法设定View的四个顶点的位置
    boolean changed = isLayoutModeOptical(mParent) ? setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        //2、调用onlayout方法
	    onLayout(changed, l, t, r, b);
        //...
    }
	//...
}
```

layout方法传进来的l、t、r、b分别代表着View的上下左右四个点的坐标，这个四个点的坐标是相对于它的父容器来说的，这个layout方法主要干了两件事：

* 1、注释1：调用View的setFrame方法设定View的四个顶点的位置，我们先看View的setFrame()方法，如下：

```java
//View.java
protected boolean setFrame(int left, int top, int right, int bottom) {
    boolean changed = false;
    if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
        changed = true;
        //...
        mLeft = left;
        mTop = top;
        mRight = right;
        mBottom = bottom;
        //...
    }
    return changed;
}
```

可以看到，setFrame方法主要把l、t、r、b分别赋值给mLeft、mTop、mBottom、mRight，即更新View的四个顶点的位置，这个四个顶点一旦确定，那么View在父容器中的位置也就确定了。

* 2、我们继续看注释2：调用了onLayout方法，这个方法在View中是一个空实现，如下：

```java
//View.java 
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
}
```

但是在ViewGroup中是一个抽象方法，如下：

```java
//ViewGroup.java
@Override
protected abstract void onLayout(boolean changed, int l, int t, int r, int b);
```

这是因为onLayout方法主要用途是给父容器确定子View的位置，所以如果本身就是一个View，就无需实现这个方法，但是如果是ViewGroup，它还要布局子View，所以是ViewGroup的子类就要强制实现这个方法，不同的ViewGroup具有不同的布局方式，所以不同的ViewGroup的onLayout方法的实现就不一样，我们还是以FrameLayout为例，看一下FrameLayout的onLayout方法的实现，如下：

```java
//FrameLayout.java
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        layoutChildren(left, top, right, bottom, false /* no force left gravity */);
}

```

FrameLayout的onLayout方法只调用了layoutChildren方法，该方法如下：

```java
void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
        final int count = getChildCount();
		//获取padding值
        final int parentLeft = getPaddingLeftWithForeground();
        final int parentRight = right - left - getPaddingRightWithForeground();
        final int parentTop = getPaddingTopWithForeground();
        final int parentBottom = bottom - top - getPaddingBottomWithForeground();
		//遍历所有子View，布局每个子View
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            //如果子View可见
            if (child.getVisibility() != GONE) {
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                //获得measue流程测量出来的子View的宽高
                final int width = child.getMeasuredWidth();
                final int height = child.getMeasuredHeight();
                //子View的左边缘位置
                int childLeft;
                //子View的上边缘位置
                int childTop;
                //下面是获取布局方向
                //...
                //下面根据布局方向计算出childLeft和childTop
                //...
			   //1、根据上面的计算，就算出了一个子View的左边缘位置childLeft和上边缘位置childTop
                //从而根据childLeft和childTop得出子View的右边缘位置childRight = childLeft + width，下边缘位置childButtom = childTop + height
                //然后调用子View的layout方法
                child.layout(childLeft, childTop, childLeft + width, childTop + height);
            }
        }
    }
```

可以发现layoutChildren里面过程和onMeasure里面的过程很像，只是注释1中调用的是子View的layout方法而不是measure方法，如果这个子View是一个View，那么layout方法里面就可以通过setFrame方法直接确定自身的位置，如果这个子View是一个ViewGroup，除了调用setFrame方法确定自身的位置外，还要重复onLayout方法中确定子View位置的过程，最后一层一层的往下，直到全部都子View的layout完成。

### 2、小结

我们再来看一下layout的整体流程：从ViewRootImp的performTraversals()方法进入performLayout()方法，开始整颗View树的布局流程，在performLayout方法里面会调用layout方法，我们发现，View的布局过程其实也可想测量过程那样分为View的layout流程和ViewGroup的layout流程，对于View来说，执行layout方法时只需要直接确定自身四个顶点的位置即可，而onLayout方法是一个空实现；对于ViewGroup来说，执行layout方法时除了要确定自身的四个顶点的位置外，那么它在onLayout方法中还要对自己所有的子View进行layout，最后一层一层的往下，直到全部都layout完成。如下：

{% asset_img view5.png view5 %}

layout过后就可以通过View的getWidth()和getHeight()来获取最终的宽高的，这个两个方法的实现如下：

```java
//View.java
public final int getWidth() {
    return mRight - mLeft;
}

public final int getHeight() {
    return mBottom - mTop;
}
```

可以发现就是通过View的四个顶点的差值来得到View的准确宽高。

## View的绘制流程 - performDraw()

和上面两步相似，View的绘制从ViewRootImpl的performDraw()开始的，如下：

```java
//ViewRootImpl.java
private void performDraw() {
    //...
    final boolean fullRedrawNeeded = mFullRedrawNeeded;
    //...
    //1、调用ViewRootImpl的draw方法
    draw(fullRedrawNeeded);
    //...
}
```

performDraw()方法中并不是先调用View的draw方法，而是先调用ViewRootImpl的draw方法，如下：

```java
//ViewRootImpl.java
private void draw(boolean fullRedrawNeeded) {
    //获取surface绘制表面
    Surface surface = mSurface;
    //...
    //如果surface表面需要更新
    if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
        //判断是否启用硬件加速，即是否使用GPU绘制
        if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
            //...
            //使用GPU绘制
            mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
        }else {
            //...
            //1、调用drawSoftware方法，使用CPU绘制
            if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
                return;
            }
        }
    }
    //...
}
```

在ViewRootImpl的draw方法中首先获取需要绘制的区域，然后判断是否使用GPU进行绘制，使用硬件加速是为提高了Android系统显示和刷新的速度，是在在API 11之后引入GPU加速的支持，关于这部分知识可自行查阅资料，不是本文重点，这里我们只关心注释1，通常情况下我们使用的是CPU绘制，也就是调用ViewRootImpl的drawSoftware方法来绘制，ViewRootImpl的drawSoftware()方法如下：

```java
//ViewRootImpl.java
private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff, boolean scalingRequired, Rect dirty) {
    final Canvas canvas;
    //...
    try {
        final int left = dirty.left;
        final int top = dirty.top;
        final int right = dirty.right;
        final int bottom = dirty.bottom;
        //1、获取指定区域的Canvas对象，即画布，用于绘制
        canvas = mSurface.lockCanvas(dirty);
        //...
    }//省略catch
    try {
        //...
        try {
            //...
            //2、从View树的根节点开始绘制，触发整颗View树的绘制
            mView.draw(canvas);
        } finally {
            //...
        }
    } finally {
        try {
            //3、释放Canvas锁，然后通知SurfaceFlinger更新这块区域
            surface.unlockCanvasAndPost(canvas);
        } catch (IllegalArgumentException e) {
           //...
        }
    }
    return true;
}
```

drawSoftware方法中主要做了3件事：

* 1、获取Surface对象并锁住Canvas绘图对象
* 2、从View树的根视图开始绘制整颗视图树
* 3、释放Surface对象并解锁Canvas，通知SurfaceFlinger更新视图

### 1、View和ViewGroup的draw流程

第1和第3点都是操作Surface的基本流程，我们主要看第二点即注释2，调用了View的draw方法，它就是一个模板方法，定义了几个固定的绘制步骤，如下：

```java
//View.java
public void draw(Canvas canvas) {
     final int privateFlags = mPrivateFlags;
     final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
                (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */
    //1、绘制背景
    if (!dirtyOpaque) {
        drawBackground(canvas);
     }
    //...
    //2、保存Canvas图层，为fadin做准备
    saveCount = canvas.getSaveCount();
    //...
    //3、 绘制自身内容，setWillNotDraw()可以控制dirtyOpaque这个标志位
    if(!dirtyOpaque) onDraw(canvas);
    //4、如果是ViewGroup，绘制子View
    dispatchDraw(canvas);
    //...
    //5、如果需要的话，绘制View的fading边缘并恢复图层
    canvas.drawRect(left, top, right, top + length, p);
	//...
    //6、绘制装饰，如滚动条
    onDrawForeground(canvas);
}
```

你看那英文注释，它已经替我们把draw方法中的6大步骤写出来了，其中最重要的就是注释3和4，我们分别来介绍一下：

* **onDraw(canvas)**：onDraw方法是用来绘制自身内容，如果你的自定义View或ViewGroup需要绘制内容，就要重写这个方法在Canvas上绘制自身内容。
* **dispatchDraw(canvas)**：如果是ViewGroup，除了绘制自身内容外，还需要绘制子View的内容，所以dispatchDraw就是把View的绘制一层一层的传递下去，直到整颗View树绘制完毕，ViewGroup重写了该方法，我们看一下它的主要源码如下：

```java
//ViewGroup.java
@Override
protected void dispatchDraw(Canvas canvas) {
    final int childrenCount = mChildrenCount;
    final View[] children = mChildren;
    //...
    for (int i = 0; i < childrenCount; i++) {
        //...
        //如果子View可见
        if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
            //调用drawChild方法，见下面
            more |= drawChild(canvas, child, drawingTime);
        }
    }
}

//ViewGroup.java
 protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
     //还是调用了View的draw方法
     return child.draw(canvas, this, drawingTime);
 }
```

可以看到，dispatchDraw方法把绘制子View的任务通过drawChild方法分发给它的子View，如果是一个ViewGroup，又会重复dispatchDraw()过程。

### 2、onDraw()绘制开关 - setWillNotDraw()

```java
//View.java
public void setWillNotDraw(boolean willNotDraw) {
    setFlags(willNotDraw ? WILL_NOT_DRAW : 0, DRAW_MASK);
}
```

但是如果你不需要绘制任何内容，你可以通过View的setWillNotDraw(true)方法关闭绘制，在默认情况下，View没有启用这个优化标志位，但是ViewGroup会启用，所以**当你的自定义ViewGroup需要通过onDraw来绘制内容时，需要显式的打开这个开关setWillNotDraw(false)，当你的自定义View不需要onDraw来绘制内容时，需要显式的关闭这个开关setWillNotDraw(true)**。

### 3、小结

到这里，我们走完了View的绘制过程，我们再来看一下draw的整体流程：从ViewRootImp的performTraversals()方法进入performDraw()方法，开始整颗View树的绘制流程，在performDraw()方法中经过层层调用：**ViewRootImpl :: draw() -> ViewRootImpl :: drawSoftware() -> View :: draw()**，来到View的draw()方法，它里面定义了View绘制的6大步骤，其中对于View来说，直接调用onDraw()方法绘制自身，对于ViewGroup来说，还要通过dispatchDraw()把绘制子View的流程分发下去，一层层传递，直到所有View都绘制完毕。如图：

{% asset_img view6.png view6 %}

## 总结

我们一直讲View的工作原理，但有没有发现ViewRootImpl也出现的很频繁，它虽然不是一个View，但它是连接View和Window之间的纽带，View三大工作流程的起点就是ViewRootImpl的performTraversals()方法，performTraversals()中依此调用了**performMeasure() -> performLayout() -> performDraw()**，分别对应顶级View的**measure、layout和draw流程**，然后顶级View的**measure流程**和**layout流程**又会分别调用我们熟悉的**onMeasure()、onLayout()方法** ，**draw流程**有点特别，它是通过**dispatchDraw()方法**来进行draw流程的传递, 而onDraw()方法只是单纯的绘制自身内容，在onMeasure()方法中会对所有child进行measure过程，同理onLayout()方法中会对所有child进行layout过程，dispatchDraw()方法中会对所有child进行draw过程，如此递归直到完成整颗View Hierarchy的遍历。

该过程如图:

{% asset_img view7.png view7 %}

有的人说分析源码的阅读，只是追踪方法的调用链，这种过程毫无意义，但是我想说的是，要想更加深入的了解Android的机制，只有源码才能给你答案，在这个阅读过程要加入自己的思考，把它的知识点用自己的语言整理，然后有所收获，我觉得这就是它的意义所在。

参考资料：

《Android开发艺术探索》



