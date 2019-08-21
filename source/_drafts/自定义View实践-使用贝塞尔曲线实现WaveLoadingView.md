---
title: 自定义View实践-使用贝塞尔曲线实现WaveLoadingView
tags: view
categories: 自定义View
---

## 前言

本文是自定义View实践第二篇，上一篇[仿微信滑动按钮](https://juejin.im/post/5d48e06d51882505723c9d30)实现了一个简单的滑动按钮，知道了一些自定义View的基本步骤，本文是使用贝塞尔曲线实现的一个加载中控件，所以阅读本文前你需要具备贝塞尔曲线的知识，懂得使用Android中相关的API。接下来进入正文讲解。

## 灵感来源

之前项目一直使用这个[WaveLoadingView](https://github.com/tangqi92/WaveLoadingView)来作为loading控件，我看它的波浪实现以为它的内部是使用贝塞尔曲线实现，直到某一天我看到它的源码时才发现不是，它的波浪实现也就是曲线实现其实是使用一个正弦函数**y = Asin(ωx + φ) + h**画出来的，当φ取两个不同的值，而A、ω、h保持相等时，就可以形成两条偏移量不同的正弦曲线，类似下面两条正弦曲线（一个φ等于0，一个φ等于1）：

{% asset_img waveloadingview1.png waveloadingview1 %}

当形成两条曲线后，画在新建的Bitmap画布上，然后使用设置了**BitmapShader**的Paint画在最终的Canvas上就形成了有固定形状的两条波浪，当你不断的改变φ值时就可以不断的移动波浪 。关于BitmapShader的原理自行查找文章，它简单的来说就是使用Bitmap来填充Paint，这样使用Paint的时候就可以指定Bitmap画在Canvas上的形状。

然而正弦函数的知识我早已经还给了老师，而且使用正弦函数画曲线的计算有点复杂，所以为了简化计算量，我就使用**贝塞尔曲线代替正弦函数**重新实现了一遍这个WaveLoadingView，名字还是叫WaveLoadingView，见下面两个WaveLoadingView的对比图：

{% asset_img waveloadingview2.gif  waveloadingview2 %}

左边是我实现的WaveLoadingView，右边是原来的WaveLoadingView，除了默认的颜色不一样，它们的效果基本都一样，对于WaveLoadingView不同形状的实现，我没有使用BitmapShader，而是使用了Canvas的相关画布裁剪API，至于为什么不用BitmapShader，见下面实现分析。

## 实现分析

交代了一下WaveLoadingView的背景，下面开始讲主要的实现步骤：

### 1、测量控件大小

我使用一个Shape枚举表示控件的4种形状，如下：

```kotlin
enum class Shape{
    CIRCLE,//圆形，默认形状 
    SQUARE, //正方形
    RECT, //矩形
    NONE//没有形状约束
}
```

对于圆形和正方形，控件的测量宽和高应该保持一样的，而对于矩形和NONE，控件的测量宽和高可以不一样，如下：

```kotlin
 override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
     val measureWidth = MeasureSpec.getSize(widthMeasureSpec)
     val measureHeight = MeasureSpec.getSize(heightMeasureSpec)
     when(shape){
         Shape.CIRCLE, Shape.SQUARE -> {//圆形或正方形
             val measureSpec = if(measureHeight < measureWidth) heightMeasureSpec else widthMeasureSpec
             //传入的measureSpec一样
             super.onMeasure(measureSpec, measureSpec)
         }else -> {//矩形或NONE
             //传入的measureSpec不一样
             super.onMeasure(widthMeasureSpec, heightMeasureSpec)
         }
     }
 }

```

所以如果用户使用圆形或正方形，但是输入的宽高不一样，我就取宽和高的最小值的测量模式去测量控件，这样就保证了控件的测量宽高一样；而用户如果使用矩形或NONE，就保持原来的测量就行了。

一个控件有可能经过多次measure后才确定测量宽高，在多次onMeasure()方法调用后，接下来会调用onSizeChanged()方法，且只会调用一次，这个方法调用后接下来就会调用onLayout()方法确定控件的最终宽高，我在onSizeChanged()里面获取测量宽高确定了控件作画的范围大小和暂时的控件大小，如下：

```kotlin
override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
    //控件作画的范围大小
    canvasWidth = measuredWidth
    canvasHeight = measuredHeight
    //控件大小，暂时等于canvas大小，后面在onlayout()中会改变
    viewWidth = canvasWidth
    viewHeight = canvasHeight
    //...
}

```

控件作画的范围大小和控件大小关系如下：

{% asset_img waveloadingview3.png  waveloadingview3 %}

绿色框就是控件作画的范围大小，红色框就是控件大小。

很多人会有疑问？你上面的赋值情况控件大小不就是等于控件作画的范围大小吗？的确是这样，**正常情况下画布的大小就是控件的大小**，当是也有特殊情况，在特殊情况下，我并不需要在整个控件上作画，我只需要取控件的居中的一部分作画就行，**特殊情况就是当父布局是ConstraintLayout，控件宽或高取match_parent时**，如下：

控件大小：layout_width = "match_parent" ; layout_height = "200dp"

{% asset_img waveloadingview4.png  waveloadingview4 %}

**图1**

控件大小：layout_width = "200dp" ; layout_height = "match_parent"

{% asset_img waveloadingview5.png  waveloadingview5 %}

**图2**

蓝色框就是手机屏幕，黑色背景就是控件大小，你还记得我上面在onMeasure()方法讲过，如果控件的形状是圆形，那么控件的测量宽高应该相等的，并取最小值为基准，所以如果控件大小输入是layout_width = "match_parent" ; layout_height = "200dp" 或 layout_width = "200dp" ; layout_height = "match_parent"，经过测量后控件大小应该是宽 = 高 = 200dp，效果应该都是如下图：

{% asset_img waveloadingview6.png  waveloadingview6 %}

**图3**

可实际情况却不是图3，而是图1或图2，这是因为ConstraintLayout布局会让子控件的setMeasuredDimension()失效，所以导致 measuredHeight 和 height 不一样，宽同理。所以在遇到父布局是ConstraintLayout时，并且控件的宽或高设置了“match_parent”，并且你自定义了测量过程，就会导致自定义View过程中测量出来大小不等于View最终大小，即getMeasureHeigth()或getMeasureWidth() != getWidth()或getHeigth()。

为什么ConstraintLayout就会有这种情况而其他如Linearlayout就没有？我也不知道，可能需要大家通过源码了解了，而我是遇到了这种情况，就通过自己的办法解决，解决办法就是让每次作画的范围在控件的中心，就像图1和图2一样，这样就不会那么难看。

### 2、裁剪画布形状

怎么把控件弄成圆形、正方形、矩形这些形状，如果控件形状是正方形或矩形，还可以设置圆角，想要画出不同的形状可以通过BitmapShader实现，使用BitmapShader要经过3步：1、新建Bitmap；2、以1新建的Bitmap新建一个Canvas；3、在2新建的Canvas上画出波浪，然后新建一个BitmapShader与1的Bitmap关联，但我没有使用BitmapShader，因为波浪的移动需要开启一个无限循环动画，就会不断的调用onDraw()方法，而在onDraw()方法不断的新建对象是一个不推荐的做法，虽然Bitmap可以通过recycler()复用，但是还是避免不了每次都要新建Canvas。

所以为了减少对象分配，我使用了Canvas的clipPath()API来把画布裁剪成我想要的形状，这样也能实现与BitmapShader同样的效果，如下：

```kotlin
 override fun onLayout(changed: Boolean, left: Int, top: Int, right: Int, bottom: Int) {     
     super.onLayout(changed, left, top, right, bottom)                                       
     preDrawShapePath(width, height)                                                         
     //...                                                                      
 }                                                                                           

private fun preDrawShapePath(w: Int, h: Int) {                                                     
    clipPath.reset()                                                                               
    when (shape) {                                                                                    
        Shape.CIRCLE -> {                                                                                       //...
            //path路径为圆形
            clipPath.addCircle(                                                                     
                shapeCircle.centerX, shapeCircle.centerY,                                           
                shapeCircle.circleRadius,                                                           
                Path.Direction.CCW                                                                 
            )                                                                                         
        }                                                                                           
        Shape.SQUARE -> {                                                                           
            //...
            //path路径为正方形或圆角正方形
            if (shapeCorner == 0f)                                                                 
            clipPath?.addRect(shapeRect, Path.Direction.CCW)                                   
            else                                                                                   
            clipPath.addRoundRect(                                                             
                shapeRect,                                                                     
                shapeCorner, shapeCorner,                                                       
                Path.Direction.CCW     
            )                                                                                   
        }                                                                                           
        Shape.RECT -> {                                                                             
            //...
        }                                                                                           
    }                                                                                               
}                                                                                                     
```

preDrawShapePath()中根据Shape来add不同的形状给Path来把这些路径信息预先保存下来，//...省略的都是居中计算，前面已经讲过每次作画的范围都在控件的中心，保存好形状的Path在onDraw方法中使用，如下：

```kotlin
override fun onDraw(canvas: Canvas?) {
    clipCanvasShape(canvas)           
    //...                 
}    

private fun clipCanvasShape(canvas: Canvas?) {    
    //调用canvas的clipPath方法裁剪画布
    if (shape != Shape.NONE) canvas?.clipPath(clipPath)
    canvas?.drawColor(waveBackgroundColor)             
}                                                      
```

在onDraw方法中使用canvas.clipPath()方法传入Path裁剪画布，这样以后作画的范围都被限定在这个画布形状之内。

### 3、画波浪



### 4、优化

不知道大家有没有发现，在整个过程中，我在onDraw()中做了两个优化：

* 1、减少对象的分配
* 2、抽取重复计算，减少浮点运算

## 结语

地址：

