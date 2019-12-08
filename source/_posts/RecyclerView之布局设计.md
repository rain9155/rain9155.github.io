---
title: RecyclerView之布局设计
date: 2019-03-01 15:10:00
tags: 
- recyclerView
- 源码
categories: recyclerView
---

## 前言
RecyclerView功能强大，自推出以来受到了无数人的喜爱，它可以通过一个LayoutManager将一个RecyclerView显示为不同的样式，例如ListView、GridView样式、瀑布流样式，所以加深对于RecyclerView的学习对于开发有很重要的意义。关于RecyclerView如何使用网上有很多文章，本篇文章从源码讲解RecyclerView如何通过layoutManager来进行布局。
<!--more-->

    本文相关源码基于Android8.0，相关源码位置如下:
    frameworks/support/v7/recyclerview/src/android/support/v7/widget/RecyclerView.java
    frameworks/support/v7/recyclerview/src/android/support/v7/widget/LinearLayoutManager.java

## RecyclerView.onLayout()
Android中每一个控件从它被定义到xml布局文件到呈现在屏幕上都要经过onMeasure -> onLayout -> onDraw 三个阶段，RecyclerView同样不例外，它的布局在OnLayout函数中进行，该方法相关源码如下:
```java
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    dispatchLayout();
    mFirstLayoutComplete = true;
}
```
可以看到该方法只是简单的调用了dispatchLayout方法,并记录了是第一次布局，dispatchLayout()相关源码如下：
```java
 void dispatchLayout() {
        //1、检查是否设置了Adapter和LayoutManager
        if (mAdapter == null) {
            return;
        }
        if (mLayout == null) {
            return;
        }
        //...
        //2、RecyclerView的布局分3步，即dispatchLayoutStep1()，dispatchLayoutStep2()，dispatchLayoutStep3()，下面分情况进行dispatchLayoutStep1()，dispatchLayoutStep2()
        //2.1、没有执行过布局流程，执行 dispatchLayoutStep1()， dispatchLayoutStep2()
        if (mState.mLayoutStep == State.STEP_START) {
            dispatchLayoutStep1();
            mLayout.setExactMeasureSpecsFrom(this);
            dispatchLayoutStep2();
        //2.2、已经执行过布局流程，但是因为数据变化或布局大小发生改变，重新执行 dispatchLayoutStep2()
        }else if (mAdapterHelper.hasUpdates() || mLayout.getWidth() != getWidth()
                || mLayout.getHeight() != getHeight()) {
            mLayout.setExactMeasureSpecsFrom(this);
            dispatchLayoutStep2();
        //2.3、已经执行过布局流程并且数据和布局大小也确定了
        } else {
            //设置RecyclerView的宽高为精确模式（即MeasureSpecMode == MeasureSpec.EXACTLY）
            mLayout.setExactMeasureSpecsFrom(this);
        }
        //3、RecyclerView的布局第3步 dispatchLayoutStep3()
        dispatchLayoutStep3();
}
```
上面源码我分3部分解释，首先注释1，没有设置RecyclerView的Adapter和LayoutManager直接return，这也解释了为什么我们平时忘记设置它们时RecyclerView会显示不出数据。
然后注释2、3，这两部分一起讲，因为RecyclerView的布局过程分为3步：dispatchLayoutStep1，dispatchLayoutStep2和dispatchLayoutStep3。在讲解之前先讲解mState.mLayoutStep，mState是State类型用于保存RecyclerView的状态，mLayouStep定义在State中，有三种取值分别代表了布局过程的3个步骤，如下：
```java
//RecyclerView.State
public static class State {
        static final int STEP_START = 1;           //还未执行dispatchLayoutStep1()，初始步骤
        static final int STEP_LAYOUT = 1 << 1;     //已经执行了dispatchLayoutStep1()或dispatchLayoutStep2()，布局步骤
        static final int STEP_ANIMATIONS = 1 << 2; //已经执行dispatchLayoutStep2()，动画步骤
        //...
        int mLayoutStep = STEP_START;
```
可以看到mLayoutStep默认是STEP_START取值，下面我们简单分析RecyclerView的布局过程3步分别做了什么，首先dispatchLayoutStep1()的相关源码如下：
```java
private void dispatchLayoutStep1() {
    //确保dispatchLayoutStep1()还未被执行过
    mState.assertLayoutStep(State.STEP_START);
    //...
    //1、处理Adapter数据更新的问题，计算需要运行的动画类型
    processAdapterUpdatesAndSetAnimationFlags();
    //2、存储关于View的一些状态和信息
    //...
    //3、 如果有必要，会进行预言性的布局，并且保存相关信息。
    if (mState.mRunSimpleAnimations) {
          //...
    }
    if(mState.mRunPredictiveAnimations){
         //...
    }
    //更新mLayoutStep的值，进入布局步骤
     mState.mLayoutStep = State.STEP_LAYOUT;
}
```
省略了很多东西，dispatchLayoutStep1()主要是来存储当前子View的状态并确定是否要执行动画、如果过有必要，会进行预言性的布局，并且保存相关信息，本文重点不在此，然后来看看dispatchLayoutStep2()，相关源码如下：
```java
private void dispatchLayoutStep2() {
        //方法执行期间不能要求RequestLayout()
        startInterceptRequestLayout();
        
        //确保已经执行了dispatchLayoutStep1()或dispatchLayoutStep2(), 从这里可以看出dispatchLayoutStep2()可能会被多次执行
        mState.assertLayoutStep(State.STEP_LAYOUT | State.STEP_ANIMATIONS);
        //1、设置好初始状态
        mAdapterHelper.consumeUpdatesInOnePass();
        mState.mItemCount = mAdapter.getItemCount();
        mState.mDeletedInvisibleItemCountSincePreviousLayout = 0;
        mState.mInPreLayout = false;
        //2、调用布局管理器去布局（布局核心方法）
        mLayout.onLayoutChildren(mRecycler, mState)；
        // 动画相关状态
        mState.mRunSimpleAnimations = mState.mRunSimpleAnimations && mItemAnimator != null;
        //更新mLayoutStep值，进入动画步骤
        mState.mLayoutStep = State.STEP_ANIMATIONS;
        
        stopInterceptRequestLayout(false);
}
```
dispatchLayoutStep2()大部分源码都在此，它才是本文的重点，它在里面调用 mLayout.onLayoutChildren(）将布局的具体策略交给了LayoutManager，下面我们会重点分析这个函数，最后我们再来看看dispatchLayoutStep3()，相关源码如下：
```java
private void dispatchLayoutStep3() {
    //确保已经执行dispatchLayoutStep2()
    mState.assertLayoutStep(State.STEP_ANIMATIONS);
    //...
    //重置mLayoutStep的值
    mState.mLayoutStep = State.STEP_START;
    //1、触发动画
    if(mState.mRunSimpleAnimations){
        //...
    }
    //2、保存View的一些信息
    //...
    //3、清除状态和清除无用的信息
    mViewInfoStore.clear()
    //...
}
```
省略了大量代码，dispatchLayoutStep3()同样跟动画相关，它主要保存关于Views的所有信息、触发动画、做必要的清理操作，它也不是本文的重点。
可以看到mLayoutStep与dispatchLayoutStep()对应关系如下：
```
STEP_START -->  dispatchLayoutStep1()
STEP_LAYOUT --> dispatchLayoutStep2()
STEP_ANIMATIONS --> dispatchLayoutStep2(), dispatchLayoutStep3()
```

讲完3个步骤我们在回到RecyclerView.dispatchLayout()，RecyclerView的布局入口OnLayout()会执行dispatchLayout()，dispatchLayout（）会根据RecyclerView的布局步骤执行dispatchLayoutStep1、2、3。那么为什么dispatchLayout（）中会分2.1, 2.2, 2.3条件执行dispatchLayoutStep1、2，而不直接按顺序dispatchLayoutStep1、2、3执行布局流程？这是因为在RecyclerView的onMeasure中，dispatchLayoutStep1、2就已经有可能因为RecyclerView自动测量模式中由于测量出来的宽高不精确而被调用，相应代码如下：
```java
 protected void onMeasure(int widthSpec, int heightSpec) {
     //...
     //设置了layoutaManager后，layoutaManager默认开启自动测量模式
     if (mLayout.isAutoMeasureEnabled()) {
        final int widthMode = MeasureSpec.getMode(widthSpec);
        final int heightMode = MeasureSpec.getMode(heightSpec);
        //首先执行LayoutManager的onMeasure方法,里面会调用RecyclerView的onMeasure方法测量自身width和height
        mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
        //Measure过后检查RecyclerView的width和height是否是精确值
        final boolean measureSpecModeIsExactly = widthMode == MeasureSpec.EXACTLY && heightMode == MeasureSpec.EXACTLY;
        //如果RecyclerView的width和height是精确值，就跳过下面步骤
        if (measureSpecModeIsExactly || mAdapter == null) {
                return;
        }
        //如果RecyclerView的width和height不是精确值，则会进行下面步骤
        //1、dispatchLayoutStep1()还未被执行过，执行 dispatchLayoutStep1()
        if (mState.mLayoutStep == State.STEP_START) {
            dispatchLayoutStep1();
        }
        mLayout.setMeasureSpecs(widthSpec, heightSpec);
        mState.mIsMeasuring = true;
        //2、执行dispatchLayoutStep2()进行布局
        dispatchLayoutStep2();
        //3、布局过程结束，该方法里面会根据childView中的边界信息计算并设置RecyclerView长宽的测量值
        mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
        //下面省略一些代码，下面还会再次检查，如果RecyclerView的宽高还不是精确值或至少有一个childView的宽高还不是精确值，还会再次执行执行dispatchLayoutStep2()进行布局
        //...
     }
     //...
 }
```
RecyclerView是一个ViewGroup，如果自身的宽高设置了warp_content必须先调用dispatchLayoutStep2()布局childView后才能测量出准确宽高。所以我们再看回dispatchLayout()中的3个判断:

*  dispatchLayout()中2.1条件：如果mLayoutStep == State.STEP_START，证明OnMeasure中还没有进行过布局，如果mLayoutStep ！= State.STEP_START，证明OnMeasure中进行过布局了，直接跳到2.3条件，不用重复布局，直接使用直接使用之前数据设置RecyclerView的宽高为精确模式。
* dispatchLayout()中2.2条件：2.1条件不成立时为什么直接跳到2.3条件不到2.2条件，因为上述条件基于RecyclerView正常的测量布局绘制到呈现在屏幕的过程，如果在这之后你对RecyclerView调用了notifXX函数，就会造成数据变化从而要求重新布局（requestLayout()函数调用），此时2.2条件就会成立，RecyclerView会调用dispatchLayoutStep2()重新布局。

* dispatchLayout()中2.3条件：2.1条件中分析过了。

3个判断后，最终一定会调用dispatchLayoutStep3()。至此分析完RecyclerView的onLayout()。

## RecyclerView.dispatchLayoutStep2() -> LayoutManager.onLayoutChildren（）

RecyclerView真正布局的进行就是在LayoutManager.onLayoutChildren（）中进行，LayoutManager的onLayoutChildren()的实现在LayoutManager的三个子类中：LinearLayoutManager、GridLayoutManager、StaggeredGridLayoutMnager，分别对应3种不同的布局样式。这里以LinearLayoutManager中的实现为例，下面是该函数在LinearLayoutManager实现中的相关源码:
```java
//源码很长，这里先抛出它的主要步骤：
//1、通过检查childView和其他变量，找出锚点的坐标（coordinate）和位置（position），并把锚点信息设置到AnchorInfo
//2、根据锚点向俩边填充

//这里还讲一下下面出现End和Start的方法或字段的意思：
//如果LinearLayoutManager的Orientation是VERTICAL方向，End指屏幕的最下面（即Bottom），Start指屏幕的最上面(即Top)
//如果LinearLayoutManager的Orientation是HORIZONTAL方向，End指屏幕的最左边（即Left），Start指屏幕的最右边(即Right)
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
    //...
    //解析布局方向，设置mShouldReverseLayout的值，它是一个Boolean类型，false表示LinearLayoutManager的Orientation是VERTICAL方向或者LinearLayoutManager的Orientation是HORIZONTAL方向并且你在manifest中没有设置RTL布局，true表示LinearLayoutManager的Orientation是HORIZONTAL方向并且你在manifest中设置了RTL布局
    resolveShouldLayoutReverse();
    //...
    //根据mStackFromEnd（表示从End开始填充itemView，默认是false）和mShouldReverseLayout决定mLayoutFromEnd的值，mLayoutFromEnd表示itemView从End开始布局还是从Start开始布局，从Start开始布局为false 从End开始布局是为true，这里一般都为false，即从Start到End开始布局
    mAnchorInfo.mLayoutFromEnd = mShouldReverseLayout ^ mStackFromEnd;
    
    //1、计算AnchorInfo的信息，即找出锚点的position和coordinate
    updateAnchorInfoForLayout(recycler, state, mAnchorInfo);
    //...
    //从End到Start开始布局，这里省略不讲，原理和从Start到End开始布局一样
    if (mAnchorInfo.mLayoutFromEnd) {
        //...
    //这里我们只讨论从Start到End开始布局
    }else{
        //更新LayoutState，确定从锚点到RecyclerView底部有多少可用空间
        updateLayoutStateToFillEnd(mAnchorInfo);
        mLayoutState.mExtra = extraForEnd;
        //2.1、第一次填充itemView，从锚点向底部填充
        fill(recycler, mLayoutState, state, false);
        endOffset = mLayoutState.mOffset;
        final int lastElement = mLayoutState.mCurrentPosition;
        if (mLayoutState.mAvailable > 0) {
            extraForStart += mLayoutState.mAvailable;
        }
        //更新LayoutState，确定从锚点到RecyclerView顶部有多少可用空间
        updateLayoutStateToFillStart(mAnchorInfo);
        mLayoutState.mExtra = extraForStart;
        mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
        //2.2、第二次填充itemView，从锚点向顶部填充
        fill(recycler, mLayoutState, state, false);
        startOffset = mLayoutState.mOffset;
        //如果屏幕上还有剩余的空间
        if (mLayoutState.mAvailable > 0) {
            extraForEnd = mLayoutState.mAvailable;
            updateLayoutStateToFillEnd(lastElement, endOffset);
            mLayoutState.mExtra = extraForEnd;
            fill(recycler, mLayoutState, state, false);
            endOffset = mLayoutState.mOffset;
        }
    }
    //...
}
```
onLayoutChildren方法有接近200行代码，但怎么也逃不出注释的2步，首先确定锚点（大部分情况下锚点就是RecyclerView上的itemView），并设置锚点的信息AnchorInfo。它定义在LinearLayoutManager中，有几个关键的属性：
```java
 static class AnchorInfo {
        OrientationHelper mOrientationHelper;//根据LinearLayoutManager的布局方向来测量itemView位置信息的帮助类，当你调用LinearLayoutManager的setOrientation(int orientation)方法时，LinearLayoutManager会根据不同orientation创建不同的OrientationHelper实现并设置给mOrientation属性
        int mPosition;//锚点在Adapter中的索引位置
        int mCoordinate;//锚点相对于LinearLayoutManager的布局方向在屏幕上的坐标，如果是VERTICAL方向，代表y轴偏移量，如果是HORIZONTAL方向，代表x轴偏移量
        boolean mLayoutFromEnd;//上面解释过了
        //...
}
```
那么它是怎么确定锚点信息的？我们来看注释1 updateAnchorInfoForLayout方法的源码：
```java
private void updateAnchorInfoForLayout(RecyclerView.Recycler recycler, RecyclerView.State state,
            AnchorInfo anchorInfo) {
    //1、如果屏幕上有itemView并且RecyclerView要滚动到某个itemView，则以这个itemView为锚点
    if (updateAnchorFromPendingData(state, anchorInfo)) {
        return;
    }
    //2、如果屏幕上有itemView,则根据anchorInfo.mLayoutFromEnd找出最接近End或Start位置的itemView为锚点
    if (updateAnchorFromChildren(recycler, state, anchorInfo)) {
        return;
    }
    //3、如果屏幕上没有itemView，则根据anchorInfo.mLayoutFromEnd和RecyclerView的padding来决定锚点coordinate和mStackFormEnd决定锚点的position
    anchorInfo.assignCoordinateFromPadding();
    anchorInfo.mPosition = mStackFromEnd ? state.getItemCount() - 1 : 0;
}
```
对于里面的俩个updataAnchorFormXX函数就不展开了，对于情况1一般是我们滚动了RecyclerView的itemView或调用了RecyclerView的scrolltoXX函数，对于情况2一般是我们itemView已经加载到屏幕上了并且此时我们调用notifiXX函数来刷新或增删itemView，而情况3就是我们现在讨论的情况，RecyclerView加载到屏幕上，此时还没有布局itemView。我们点进AnchorInfo的assignCoordinateFromPadding()看看干了什么，相关源码如下：
```java
 void assignCoordinateFromPadding() {
        mCoordinate = mLayoutFromEnd ? mOrientationHelper.getEndAfterPadding() :     mOrientationHelper.getStartAfterPadding();
}
     
//下面只给出LinearLayoutManager的Orientation为VERTICAL方向的实现
public int getEndAfterPadding() {
    return mLayoutManager.getHeight() - mLayoutManager.getPaddingBottom();
}
 public int getStartAfterPadding() {
    return mLayoutManager.getPaddingLeft();
}
```
可以看到如果此时RecyclerView中没有itemView并且LinearLayoutManager的布局方向为VERTICAL和mLayoutFromEnd值为false：anchorInfo的mCoordinate就是RecyclerView的paddingLeft，anchorInfo的position就是0（锚点为RecyclerView左上角的位置）。
{% asset_img rv1.png 图一 %}

我们回到onlayoutChildern方法，确定了锚点后，然后就要根据AnchorInfo开始填充itemView，在开始填充之前，LinearLayoutManager会用LayoutState暂时保存一些布局信息，它定义在LinearLayoutManager中，有几个关键属性：
```java
static class LayoutState {
    int mAvailable;//表示当前的布局方向中，RecyclerView中要用于填充itemView的可用空间大小
    int mOffset;//表示当前的布局方向中，在RecyclerView中距离锚点的位置偏移量
    int mExtra = 0;//表示自己设置的额外布局的范围，一般不会设置
    int mLayoutDirection;//表示布局往哪个方向填充，俩个取值：LAYOUT_START为向RecyclerView顶部，LAYOUT_END为向底部
    int mCurrentPosition;//表示当前锚点在Adapter中的索引，可用它获得下一个itemView的索引
    int mItemDirection;//决定由mCurrentPosition获得下一个itemView的索引时是+1还是-1，俩个取值：ITEM_DIRECTION_HEAD表示索引-1，ITEM_DIRECTION_TAIL表示索引+1
    //...
}
```
updateLayoutStateToFillEnd函数会在向下填充前更新layoutState的值，相关源码如下：
```java
private void updateLayoutStateToFillEnd(AnchorInfo anchorInfo) {
        updateLayoutStateToFillEnd(anchorInfo.mPosition, anchorInfo.mCoordinate);
}

private void updateLayoutStateToFillEnd(int itemPosition, int offset) {
    //下面基于在当前讨论的情景中：
    //这里可用布局空间mLayoutState.mAvailable就是RecyclerView的高度
    mLayoutState.mAvailable = mOrientationHelper.getEndAfterPadding() - offset;
    //这里为LayoutState.ITEM_DIRECTION_TAIL，索引+1
    mLayoutState.mItemDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_HEAD :
            LayoutState.ITEM_DIRECTION_TAIL;
    //当前锚点索引
    mLayoutState.mCurrentPosition = itemPosition;
    //这里为向RecyclerView底部填充
    mLayoutState.mLayoutDirection = LayoutState.LAYOUT_END;
    //这里offet为0
    mLayoutState.mOffset = offset;
}
```
准备好layoutState后，就调用fill方法进行填充itemView，核心源码如下:
```java
int fill(RecyclerView.Recycler recycler, LayoutState layoutState, RecyclerView.State state, boolean stopOnFocusable) {
    //前面讲过，表示可用布局空间大小
    final int start = layoutState.mAvailable;
    //...
    // 1、计算剩余可用的填充空间，可用布局空间加上额外布局空间
    int remainingSpace = layoutState.mAvailable + layoutState.mExtra;
    //用于记录每一次while循环的填充一个itemView后的结果
    LayoutChunkResult layoutChunkResult = mLayoutChunkResult;
    //2、while判断条件，屏幕还有剩余可用空间并且还有数据就继续执行
    while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
        //重置LayoutChunkResult
        layoutChunkResult.resetInternal();
        //3、循环调用layoutChunk方法一个一个的填充itemView，里面会根据LinearLayoutmanager的orientation方向布局itemView（布局子View的核心方法）
        layoutChunk(recycler, state, layoutState, layoutChunkResult);
        if (layoutChunkResult.mFinished) {
            break;
        }
        //计算填充一次itemView消耗了多少空间，或者说计算距离锚点的偏移量
        layoutState.mOffset += layoutChunkResult.mConsumed * layoutState.mLayoutDirection;
        //4、如果layoutChunkResult没有要求忽略这次消耗或这次布局的不是ScrapView或我们不是在做预布局，就更新可填充空间的大小
        if (!layoutChunkResult.mIgnoreConsumed || mLayoutState.mScrapList != null
                    || !state.isPreLayout()) {
                layoutState.mAvailable -= layoutChunkResult.mConsumed;
                remainingSpace -= layoutChunkResult.mConsumed;
            }
        //下面省略的是，如果是因滚动引起的布局，会通过判断滑动后view是否滑出边界决定是否回收View
        //...
    }
    //填充完成，修改起始位置，即填充到哪个位置
    return start - layoutState.mAvailable;
}
```
上面的注释很详细，大概流程就是在while循环中根据剩余可用空间不断的调用layoutChunk（）函数进行布局itemView，layoutChunk方法会在里面根据RecyclerView的缓存机制获取一个View从而把它填充到RecyclerView中去，下面继续来看layoutChunk方法相关源码：
```java
void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,  LayoutState layoutState, LayoutChunkResult result) {
    //1、获取一个View
    View view = layoutState.next(recycler);
    if (view == null) {
        result.mFinished = true;
        return;
    }
    RecyclerView.LayoutParams params = (RecyclerView.LayoutParams) view.getLayoutParams();
   //...
   //2、测量itemView
   measureChildWithMargins(view, 0, 0);
   //3、计算该itemView消耗的高度或宽度
   result.mConsumed = mOrientationHelper.getDecoratedMeasurement(view);
   int left, top, right, bottom;
   //4、按竖直方向布局，计算itemView的上下左右布局
   if (mOrientation == VERTICAL) {
        if (isLayoutRTL()) {
            right = getWidth() - getPaddingRight();
            left = right - mOrientationHelper.getDecoratedMeasurementInOther(view);
        } else {
            left = getPaddingLeft();
            right = left + mOrientationHelper.getDecoratedMeasurementInOther(view);
        }
        if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
            bottom = layoutState.mOffset;
            top = layoutState.mOffset - result.mConsumed;
        } else {
            top = layoutState.mOffset;
            bottom = layoutState.mOffset + result.mConsumed;
        }
    //水平布局的计算方式
    } else {
        //...
    }
    //5、布局itemView
    layoutDecoratedWithMargins(view, left, top, right, bottom);
    //消耗可用布局空间如果itemView没有被移除或没有改变
    if (params.isItemRemoved() || params.isItemChanged()) {
        result.mIgnoreConsumed = true;
    }
}
```
在layoutChunk方法中首先从layoutState中根据mCurrentPosition获取itemView，然后获取itemView的布局参数，并且根据布局方式(横向或纵向)计算出itemView的上下左右布局，最后调用layoutDecoratedWithMargins方法实现布局itemView，layoutDecoratedWithMargins方法定义在LayoutManger中，具体代码如下：
```java
 public void layoutDecoratedWithMargins(@NonNull View child, int left, int top, int right,  int bottom) {
    final LayoutParams lp = (LayoutParams) child.getLayoutParams();
    final Rect insets = lp.mDecorInsets;
    child.layout(left + insets.left + lp.leftMargin, top + insets.top + lp.topMargin,
right - insets.right - lp.rightMargin,
            bottom - insets.bottom - lp.bottomMargin);
 }
```
可以看到，只是调用了itemView的layout函数将itemView布局到具体的位置。

我们再回到onlayoutChildern方法，按照上面图一，我们已经填充了下面，但是上面是不用填充的，因为没有可用空间，所以注释2.2基本下是不会走的了。而fill towaards Start步骤和fill towards End差不多。那么为什么RecyclerView进行两次填充呢？因为RecyclerView理想的锚点如下图：
{% asset_img rv2.png 图二 %}

上面是RecyclerView的方向为VERTICAL的情况，当为HORIZONTAL方向的时候填充算法是不变的。但我们一般是图一的情况，从上往下填充。
## 总结
一图胜千言，下图是LayoutManager循环布局所有的itemView。

{% asset_img rv3.png 图三 %}

可以看到RecyclerView将布局的职责分离到LayoutManager中，使得RecyclerView更加灵活，我们也可以自定义自己的LayoutManger，实现自己想要的布局。可以看到RecyclerView具有很强大的扩展性，所以深入学习这个控件是很有必要的。能看到这里的都是有毅力的人，本文只是RecyclerView学习的第一篇，以后会继续分析RecyclerView的缓存设计。

参考资料：

《Android源码设计与分析》

[RecyclerView和ListView原理](https://blog.csdn.net/feather_wch/article/details/81613313#观察者模式)

[RecyclerView源码分析(三)--布局流程](https://www.jianshu.com/p/898479f103b6)





