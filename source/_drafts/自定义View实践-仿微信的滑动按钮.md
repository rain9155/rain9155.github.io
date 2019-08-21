---
title: 自定义View实践-仿微信的滑动按钮
tags: view
categories: 自定义View
---

## 前言

前几天写过一篇文章[View的工作原理](https://juejin.im/post/5d35a5fc518825019b0a3678)，有原理不行，还要有实践，刚好把以前项目写过的仿微信滑动按钮控件封装一下，所以本文记录一下我实现这个控件的细节。

## 效果图

控件使用效果如下：

{% asset_img sb1.gif sb1 %}

除了颜色，看起来和微信的还是挺像的。

## 准备

### 1、选择自定义View的方式

自定义View有3种途径实现：1、组合控件；2、继承现有控件(如Button)；3、继承View。下面分别介绍一下：

- 1、组合控件：我们并不需要自己去绘制视图上显示的内容，而是将几个系统原生的控件组合到一起，这样创建出的控件就被称为组合控件，比如标题栏就是个很常见的组合控件。
- 2、继承现有控件：我们并不需要自己重新去实现一个控件，只需要去继承一个现有的控件，然后在这个控件上增加一些新的功能。它的优点就是不仅能够按照我们的需求加入相应的功能，并且还可以继承现有控件已经封装好的属性，同时不用自己定义测量流程。
- 3、继承View：我们继承View，重写相应的方法，重新去实现一个控件。它的优点就是灵活性高，它给你一张白纸，你用画笔尽情发挥。

现实情况使用什么方式根据实际情况考虑，我这个控件的选择是方式3: **继承View**，重写onMeasure方法定义它的测量流程，重写onDraw()方法定义它的绘制流程。

### 2、选择让控件内容滑动的方式

既然是滑动按钮，肯定有滑动，当我点击按钮时，如果是打开，按钮的小圆会滑向右边，如果是关闭，按钮的小圆会滑向左边。让控件的内容滑动起来我想到的有3种方式：

- 1、通过Scroller：调用Scroller的startScroll()方法，传入起始点坐标和终点坐标，然后重写View的computeScroll()方法，在这个方法里面调用Scroller的computeScrollOffset()方法开始滑动计算，然后调用View的scrollTo()或scrollBy()方法完成View的滑动距离的更新，然后调用View的invalidate()或postInvalidate()方法重绘View。
- 2、通过Handler不断的发送延时消息：通过Handler的 sendMessageDelayed(Message msg, long delayMillis)方法不断的发送延时消息，在Handler的handlerMessage()中收到消息后，完成滑动距离的更新，然后调用View的invalidate()或postInvalidate()方法重绘View。
- 3、通过动画：利用补间动画或属性动画的平移动画可以让View动起来，或者通过ValueAnimator，设定一个初始值和结点值，当调用ValueAnimator的start()方法后，就可以在回调中获取动画的进度，然后根据动画的进度更新滑动距离，然后调用View的invalidate()或postInvalidate()方法重绘View。

对于方法1，它更适用于自定义ViewGroup的情景，如果自定义ViewGroup中有许多子View需要滑动起来，就可以考虑使用Scroller，例如Android的ViewPager内部就是使用了Scroller；而对于自定义View，可能方法2和3更适用，我这个控件的选择是方式3: **通过ValueAnimator动画**，在构造ValueAnimator时传入起点和终点，然后开启动画，根据动画进度计算滑动距离，让按钮的小圆滑动起来。

### 3、要不要考虑padding属性

如果你在自定义控件中没有考虑padding属性，那么用户定义控件的padding值就会失效，我的选择是**不考虑用户的padding值**，因为滑动按钮中的内容只有一个小圆，且只在一边，padding的意义不大，考虑padding会让很多地方的坐标计算复杂，我还不如让用户直接控制小圆的半径，这样也类似于padding的效果，也简化了计算。

所以现实情况要不要考虑padding属性需要根据实际情况考虑。而margin值是由父ViewGroup决定，不是由View控制的，我们不用考虑margin值。

## 实现

### 1、定义控件属性

在自定义滑动按钮之前，我们先思考可以让用户自定义这个控件的什么属性，如按钮颜色，打开状态和关闭状态的颜色等，在 res -> values 中，右键新建一个名为attrs的xml文件，在这个文件中定义控件属性，如下：

```xml
<resources>

    <declare-styleable name="SwitchButton" >
        <attr name="sb_openBackground" format="color"/>
        <attr name="sb_closeBackground" format="color"/>
        <attr name="sb_circleColor" format="color"/>
        <attr name="sb_circleRadius" format="dimension"/>
        <attr name="sb_status">
            <enum name="close" value="0"/>
            <enum name="open" value="1"/>
        </attr>
        <attr name="sb_interpolator">
            <enum name="Linear" value="0"/>
            <enum name="Overshoot" value="1"/>
            <enum name="Accelerate" value="2"/>
            <enum name="Decelerate" value="3"/>
            <enum name="AccelerateDecelerate" value="4"/>
            <enum name="LinearOutSlowIn" value="5"/>
        </attr>
    </declare-styleable>

</resources>
```

这样用户在引用这个控件时就能使用这些属性，如下：

```xml
    <com.example.library.SwitchButton
            android:id="@+id/sb_button2"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            app:sb_interpolator="Accelerate"
            app:sb_status="open"
            app:sb_circleRadius="10dp"
            app:sb_closeBackground="@android:color/black"
            app:sb_openBackground="@android:color/holo_blue_bright"
            app:sb_circleColor="@android:color/white" />
```

属性的名称要做到见名知意，app只是一个命名空间，取什么名字都可以，不要和系统android相同就行。关于这些属性什么意思可以看[SwitchButton](https://github.com/rain9155/SwitchButton)。

### 2、初始化控件属性

重写View的3个构造方法，分别在3个构造函数中调用init()方法获取控件属性并初始化控件，如下：

```java
public class SwitchButton extends View {
    public SwitchButton(Context context) {
        super(context);
        init(context, null);
    }

    public SwitchButton(Context context, AttributeSet attrs) {
        super(context, attrs);
        init(context, attrs);
    }

    public SwitchButton(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init(context, attrs);
    }

    private void init(Context context, @Nullable AttributeSet attrs) {
        TypedArray typedValue = context.obtainStyledAttributes(attrs, R.styleable.SwitchButton);
        mOpenBackground = typedValue.getColor(R.styleable.SwitchButton_sb_openBackground, DEFAULT_OPEN_BACKGROUND);
        mCloseBackground = typedValue.getColor(R.styleable.SwitchButton_sb_closeBackground, DEFAULT_CLOSE_BACKGROUND);
        //...
        typedValue.recycle();
         //...
        //初始画笔，动画等
    }
}
```

我们在attrs中定义的控件属性都在AttributeSet这个集合中，然后通过TypedArray这个类帮助我们把值获取出来，最后一定要记得调用  typedValue.recycle() 方法回收资源。

为什么要重写3个构造函数呢？因为你的控件有可能在代码中引用或者在xml布局中引用，如果你的控件在xml布局中被引用，那么系统就会调用含有两个参数的构造函数来初始化控件；如果你直接在代码中 new 一个控件然后 add 到容器中，那么大多数情况你会使用含有一个参数的构造函数来初始化控件，如：SwitchButton button = new SwitchButton(this)，而不管一个参数的还是两个参数的系统最终都会调用含有三个参数的构造函数，以防万一，3个构造函数都要重写。

### 3、重写onMeasure方法，设定按钮的测量宽高

重写onMeasure方法在这个方法设定滑动控件的测量宽高，如下：

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int measuredWidthMode = MeasureSpec.getMode(widthMeasureSpec);
    int measuredHeightMode = MeasureSpec.getMode(heightMeasureSpec);
    //取出系统测量宽高
    int measuredWidth = MeasureSpec.getSize(widthMeasureSpec);
    int measureHeight = MeasureSpec.getSize(heightMeasureSpec);
    
    int defaultWidth = (int) TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, 60, getResources().getDisplayMetrics());//控件的默认宽
    int defaultHeight = (int) (defaultWidth *  0.5f);//控件的默认高是默认宽的一半
    
    //OFFSET == 6
    int offset = (int) TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, OFFSET * 2 * 1.0f, getResources().getDisplayMetrics());//控件宽和高的差距不能小于12dp, 否则按钮就不好看了
    
    //考虑wrap_content情况
    if(measuredWidthMode == MeasureSpec.AT_MOST && measuredHeightMode == MeasureSpec.AT_MOST){
        measuredWidth = defaultWidth;
        measureHeight = defaultHeight;
    }else if(measuredHeightMode == MeasureSpec.AT_MOST){
        measureHeight = defaultHeight;
        if(measuredWidth - measureHeight < offset)
            measuredWidth = defaultWidth;
    }else if(measuredWidthMode == MeasureSpec.AT_MOST){
        measuredWidth = defaultWidth;
        if(measuredWidth - measureHeight < offset)
            measureHeight = defaultHeight;
    }else {
        //处理输入非法的宽高情况，即高度大于宽度，把它们交换就行
        if(measuredWidth < measureHeight){
            int temp = measuredWidth;
            measuredWidth = measureHeight;
            measureHeight = temp;
        }
    }
    
    if(Math.abs(measureHeight - measuredWidth) < offset) throw new IllegalArgumentException("layout_width cannot close to layout_height nearly, the diff must less than 12dp!");
    
    setMeasuredDimension(measuredWidth, measureHeight);
    
}
```

如果知道View的工作原理，那么理解上面的代码就很简单，主要是考虑wrap_content情况，我们要给滑动按钮设置一个默认的宽或高，默认的宽是60dp，默认高是30dp即宽的一半，如果不是wrap_content情况就让View直接使用系统测量的宽或高，最后一定要记得调用setMeasuredDimension()设定View的测量宽高。

同时我们还要考虑理输入非法的宽高情况，一定要保证宽 > 高，如果用户输入的宽高是 宽 < 高，这样会导致按钮竖起来，这种情况，我直接让高度与宽度交换；如果用户输入的宽高是 宽 > 高，但是如果高很接近宽甚至相等，那么导致滑动控件就是一个圆形，按钮就不好看了，所以我们还要控制宽高不能相差得太近，为了美观，我设定阈值是12dp，如果宽高相差小于12dp，我就抛个异常提示用户。

### 4、在onLayout()方法中根据View的宽高计算坐标

滑动控件被分为4个部分：**左圆、矩形、右圆、小圆**，如下：

{% asset_img sb2.png sb2 %}

在onDraw()方法中也会按顺序绘制滑动按钮的4个部分，在View的工作原理中讲到，onMeasure()有可能会被系统调用多次，所以最好在onLayout()方法中通过getHeight()和getWidth()方法获得View的真实宽高，所以在onLayout()方法中首先根据View的宽高计算出左圆的半径，小圆的半径，矩形左边界的x坐标，矩形右边界的x坐标，还有小圆圆心的x坐标，如下：

```java
@Override
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    super.onLayout(changed, left, top, right, bottom);
    //得出左圆的半径
    mLeftSemiCircleRadius = getHeight() / 2;
    //小圆的半径 = 大圆半径减OFFER，OFFER = 6
    if(!checkCircleRaduis(mCircleRadius)) mCircleRadius = mLeftSemiCircleRadius - OFFSET;
    //矩形左边的x坐标
    mLeftRectangleBolder = mLeftSemiCircleRadius;
    //矩形右边的x坐标
    mRightRectangleBolder = getWidth() - mLeftSemiCircleRadius;
    //小圆的圆心x坐标一直在变化
    mCircleCenter = isOpen ? mRightRectangleBolder : mLeftRectangleBolder;
}
```

可以看到左圆的半径等于View高的一半，然后基于左圆的半径得出其他坐标，小圆与左圆之间会有一些空隙，所以左圆半径减去offset值得出小圆半径，矩形左边的x坐标直接等于左圆的半径，矩形右边的x坐标View的宽度减左圆的半径，小圆圆心的x坐标根据初始状态是开启还是关闭，决定它的圆心的初始坐标是在矩形的右边界还是左边界。

在接下来只要你不断的改变小圆圆心的x坐标并重绘View，就可以让滑动按钮滑动起来。

### 5、重写onDraw()方法，绘制按钮内容

View的工作原理中我们知道，View会在onDraw()方法中绘制自己，所以我们重写onDraw()方法，绘制滑动按钮的四个部分，如下：

```java
@Override
protected void onDraw(Canvas canvas) {
    //左圆
    canvas.drawCircle(mLeftRectangleBolder, mLeftSemiCircleRadius, mLeftSemiCircleRadius, mPathWayPaint);
    //矩形
    canvas.drawRect(mLeftRectangleBolder, 0, mRightRectangleBolder, getMeasuredHeight(), mPathWayPaint);
    //右圆
    canvas.drawCircle(mRightRectangleBolder, mLeftSemiCircleRadius, mLeftSemiCircleRadius, mPathWayPaint);
    //小圆
    canvas.drawCircle(mCircleCenter, mLeftSemiCircleRadius, mCircleRadius, mCirclePaint);
}

```

canvas是系统提供给我们的画布，在canvas绘制的东西就是View显示的内容，根据在onLayout中的计算，我们用画笔Paint在canvas中绘制出滑动按钮的4个部分，绘制后显示如下：

{% asset_img sb3.png sb3 %}

接下来就是让它滑动起来，这样就能达到效果图的效果。

### 6、重写onTouchEvent()方法，让按钮滑动起来

在[View的事件分发机制](https://juejin.im/post/5d3f0fc3f265da03e921a397)讲到，触摸事件如果不被拦截，最终会分发到View的onTouchEvent()方法中，在这个方法中我们可以根据事件的类型做出滑动按钮的不同行为，我们知道当手指按下按钮然后抬起，滑动按钮的小圆就会滑动到另一边；当手指按下按钮然后移动，滑动按钮的小圆也会跟随手指移动，知道了这两个行为后，我们看onTouchEvent()方法如下：

```java
@Override
public boolean onTouchEvent(MotionEvent event) {
    //不在动画的时候可以点击
    if(isAnim) return false;
    switch(event.getAction()){
        case MotionEvent.ACTION_DOWN:
            //开始的x坐标
            startX = event.getX();
            break;
        case MotionEvent.ACTION_MOVE:
            float distance = event.getX() - startX;
            //更新小圆圆心坐标
            mCircleCenter += distance / 10;
            //控制范围
            if (mCircleCenter > mRightRectangleBolder) {//最右
                mCircleCenter = mRightRectangleBolder;
            } else if (mCircleCenter < mLeftRectangleBolder) {//最左
                mCircleCenter = mLeftRectangleBolder;
            }
            invalidate();
            break;
        case MotionEvent.ACTION_UP:
            float offset = Math.abs(event.getX() - Math.abs(startX));
            float diff;
            //分2种情况
            if (offset < mMinDistance) { //1.点击, 按下和抬起的距离小于mMinDistance确定是点击了
                if(isOpen){
                    diff = mLeftRectangleBolder - mCircleCenter;
                }else{
                    diff = mRightRectangleBolder - mCircleCenter;
                }
            } else {//2.滑动
                if (mCircleCenter > getWidth() / 2) {//滑过中点，滑到最右
                    this.isOpen = false;
                    diff = mRightRectangleBolder - mCircleCenter;
                } else{//没滑过中点,回归原点
                    this.isOpen = true;
                    diff = mLeftRectangleBolder - mCircleCenter;
                }
            }
            mValueAnimator.setFloatValues(0, diff);
            mValueAnimator.start();
            startX = 0;
            break;
        default:
            break;
    }
    return true;
}

```

我们先看ACTION_DOWN，当手指按下，我们记录手指按下的x坐标。

接着看ACTION_MOVE，如果按下后移动，我们就让小圆跟随手指移动即可，所以ACTION_MOVE中先计算出手指移动的距离distance，往右移distance是正数，往左移distance是负数，然后加到小圆的圆心坐标，还要控制小圆的圆心坐标的范围，不要超出矩形左右边界，最后调用 invalidate()重绘View，这样onDraw()方法就会重新执行，更新小圆的位置，就会让小圆慢慢滑动起来。

最后看ACTION_UP，**mMinDistance = new ViewConfiguration().getScaledTouchSlop()**，它是系统定义的临界值，当抬起手指时，如果移动的距离offset大于mMinDistance ，就认为抬起手指前，手指在移动，否则就认为在点击。如果手指在移动后抬起，这时就判断小圆圆心是否滑过中点算出滑动距离，如果滑过中点(getWidth() / 2)，就让小圆滑到最右，如果没有滑过中点，就让小圆滑到最左；如果手指只是在点击控件，这时就根据控件目前处于开启还是关闭状态算出滑动距离，如果目前处于开启状态，就让小圆滑到最左，如果目前处于关闭状态就让小圆滑到最右；而这个滑动距离diff就是小圆圆心到矩形边界的距离，至于是距离左边界还是右边界，就看上述情况了，计算出滑动距离后设置给ValueAnimator，最后开启动画，在ValueAnimator的updateListener中接收动画进度，如下：

```java
mValueAnimator.addUpdateListener(animation -> {
    float value = (float)animation.getAnimatedValue();
    mCircleCenter -= mPreAnimatedValue;
    //更新小圆圆心坐标
    mCircleCenter += value;
    mPreAnimatedValue = value;
    invalidate();
});
```

在里面根据动画进度更新小圆圆心坐标，然后调用 invalidate()重绘View，这样onDraw()方法就会重新执行，更新小圆的位置，这样重复执行直到动画结束，就会让小圆慢慢滑动起来。

## 结语

到最后就已经实现了效果图的效果，整个过程的原理还是挺简单，使用到了动画还有自定义View的基础知识，赶快动手实践一下。

地址：[SwitchButton](https://github.com/rain9155/SwitchButton)