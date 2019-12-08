---
title: 自定义View实践-使用贝塞尔曲线实现WaveLoadingView
tags: view
categories: 自定义View
---

## 前言

本文是自定义View实践第二篇，上一篇[仿微信滑动按钮](https://juejin.im/post/5d48e06d51882505723c9d30)实现了一个简单的滑动按钮，知道了一些自定义View的基本步骤，本文是使用贝塞尔曲线实现的一个加载中控件，所以阅读本文前你需要具备贝塞尔曲线的知识，懂得使用Android中相关的API。接下来进入正文讲解。

地址：[WaveLoadingView](https://github.com/rain9155/WaveLoadingView)

## 灵感来源

之前项目一直使用这个[WaveLoadingView](https://github.com/tangqi92/WaveLoadingView)来作为loading控件，我看它的波浪实现以为它的内部是使用贝塞尔曲线实现，直到某一天我看到它的源码时才发现不是，它的波浪实现也就是曲线实现其实是使用一个正弦函数**y = Asin(ωx + φ) + h**画出来的，当φ取两个不同的值，而A、ω、h保持相等时，就可以形成两条偏移量不同的正弦曲线，类似下面两条正弦曲线（一个φ等于0，一个φ等于2）：

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

怎么把控件弄成圆形、正方形、矩形这些形状，如果控件形状是正方形或矩形，还可以设置圆角，一个方法是通过BitmapShader实现，使用BitmapShader要经过3步：1、新建Bitmap；2、以1新建的Bitmap新建一个Canvas；3、在2新建的Canvas上画出波浪，然后新建一个BitmapShader与1的Bitmap关联，但我没有使用BitmapShader，因为波浪的移动需要开启一个无限循环动画，就会不断的调用onDraw()方法，而在onDraw()方法不断的新建对象是一个不推荐的做法，虽然Bitmap可以通过recycler()复用，但是还是避免不了每次都要新建Canvas对象。

所以为了减少对象分配，我使用了Canvas的clipPath()API来把画布裁剪成我想要的形状，然后把波浪画在裁剪后的画布上，这样也能实现与BitmapShader同样的效果，如下：

```kotlin
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
    //...            
}                                                      
```

在onDraw方法中使用canvas.clipPath()方法传入Path裁剪画布，这样以后作画的范围都被限定在这个画布形状之内。

### 3、画波浪

使用贝塞尔曲线画波浪，如下：

```kotlin
private fun preDrawWavePath() {
    wavePath.reset()
    //波长等于画布的宽度
    val waveLen = canvasWidth
    //波峰
    val waveHeight = (waveAmplitude * canvasHeight).toInt()
    //波浪的起始y坐标
    waveStartY = calculateWaveStartYbyProcess()
    //把path移到起始位置，这里使用了path.moveTo（）方法
    wavePath.moveTo(-canvasWidth * 2f, waveStartY)
    //下面就是画波浪的过程，都使用了path.rXX（）方法，表示把上一次结束点的坐标作为原点，从而简化计算量
    val rang = -canvasWidth * 2..canvasWidth
    for (i in rang step waveLen) {
        wavePath.rQuadTo(
            waveLen / 4f, waveHeight / 2f,
            waveLen / 2f, 0f
        )
        wavePath.rQuadTo(
            waveLen / 4f, -waveHeight / 2f,
            waveLen / 2f, 0f
        )
    }
    //波浪的深度就是画布的高度
    wavePath.rLineTo(0f, canvasHeight.toFloat())
    wavePath.rLineTo(-canvasWidth * 3f, 0f)
    //最后使用path.close()把波浪的路径关闭，使整个波浪围起来
    wavePath.close()
}
```

preDrawWavePath() 中把波浪路径的信息保存在path中，下面一张图很好的说明波浪的整个路径，如下：

{% asset_img waveloadingview7.png  waveloadingview7 %}

我把控件大小充满了父容器，所以控件的作画范围就是绿色框的大小，波浪的波长就是一个画布的宽度即绿色框的宽度，我把波浪的起始点移到屏幕范围外，从起始点开始，画了三个波长，把波浪画出屏幕的范围，从而方便的待会的波浪的上下移动，最后记得使用path.close()把波浪的路径关闭，使整个波浪围起来。

保存好波浪路径的信息的Path在onDraw方法中使用，如下：

```kotlin
override fun onDraw(canvas: Canvas?) {
    clipCanvasShape(canvas)
    drawWave(canvas)
    //...
}

private fun drawWave(canvas: Canvas?) {
    wavePaint.style = Paint.Style.FILL_AND_STROKE
    wavePaint.color = waveColor
    //...
    //使用canvas的drawPath()方法把波浪画在画布上
    canvas?.drawPath(wavePath, wavePaint)
}
```

使用canvas的drawPath()方法直接把波浪画在画布上，这时在屏幕上显示的效果如下：

{% asset_img waveloadingview8.png  waveloadingview8 %}

这样就画出了一条波浪了，第二条波浪呢？可以再用另外一个Path按照上述preDrawWavePath()方法的流程再画一条，只要波浪的起始点坐标不同就行，但我没有用这种办法，我是通过Canvas的translate()方法平移画布，利用两次平移的偏移量不一样，画出了第二条，如下：

```kotlin
private fun drawWave(canvas: Canvas?) {
    wavePaint.style = Paint.Style.FILL_AND_STROKE
    
    //首先保存两次画布状态，记为画布1、2
    canvas?.save()//画布1
    canvas?.save()//画布2
    
    //记当前画布为画布3
    //调用canvas的translate（）方法水平平移一下画布3
    canvas?.translate(canvasSlowOffsetX, 0)
    wavePaint.color = adjustAlpha(waveColor, 0.7f)
    //首先在画布3画出第一条波浪
    canvas?.drawPath(wavePath, wavePaint)
    
    //恢复保存的画布2状态
    canvas?.restore()
    
  	//下面是在画布2上作画
    //调用canvas的translate（）方法水平平移一下画布2
    canvas?.translate(canvasFastOffsetX, 0)
    wavePaint.color = waveColor
    //然后在画布2上画出第二条波浪
    canvas?.drawPath(wavePath, wavePaint)
    
    //恢复保存的画布1状态
    canvas?.restore()
    
    //后面都是在画布1上作画
}
```

熟悉Canvas的save()、restore()方法都知道，每调用一次save()，可以理解为画布的一次入栈（保存），每调用一次restore()，可以理解为画布的出栈（恢复），画布3是默认就有的，画布1、2是我保存生成的，所以上述画布1，2，3之间是独立的，互不影响的，而canvasSlowOffsetX和canvasFastOffsetX两个值是不一样的，这样就造成了画布2和3平移时偏移量不一样，所以**用同一个Path画在两个偏移量不一样的画布上就可以形成两条波浪**，效果图如下：

{% asset_img waveloadingview9.png  waveloadingview9 %}

#### 3.1、让波浪动起来

让波浪移动起来很简单，使用一个无限循环动画，在动画的进度回调中计算画布的偏移量，然后调用invalidate()就行，如下：

```kotlin
waveValueAnim.apply {
    duration = ANIM_TIME
    repeatCount = ValueAnimator.INFINITE//无限循环
    repeatMode = ValueAnimator.RESTART
    addUpdateListener{ animation ->
      	//...  
        canvasFastOffsetX = (canvasFastOffsetX + fastWaveOffsetX) % canvasWidth
        canvasSlowOffsetX = (canvasSlowOffsetX + slowWaveOffsetX) % canvasWidth
        invalidate()
     }
}
```

在适当的时机启动动画，如下：

```kotlin
override fun onDraw(canvas: Canvas?) {
    clipCanvasShape(canvas)
    drawWave(canvas)
    //...
    //启动动画
    startLoading()
}

fun startLoading(){
   if(!waveValueAnim.isStarted) waveValueAnim.start()
}
```

到这里整个控件就完成了。

### 4、优化

大家都知道手机的资源都是非常有限的，我在做自定义View时，特别是涉及到无限循环的动画时，要注意优化我们的代码，因为一般的屏幕刷新周期是16ms，这意味着在这16ms内你要把有关动画的所有计算和流程完成，不然就会造成掉帧，从而卡顿，在自定义View时我想到可以从下面几点做一些优化，提高效率：

#### 4.1、减少对象的内存分配，尽可能做到对象复用

每次系统GC的时候都会暂停系统ms级别的时间，而无限循环的动画的逻辑代码会在短时间内被循环往复的调用, 这样如果在逻辑代码中在堆上创建过多的临时变量，会导致内存的使用量在短时间内上升，从而频繁的引发系统的GC行为，这样无疑会拖累动画的效率，让动画变得卡顿。

在自定义View涉及到无限循环动画时，我们不能忽略对象的内存分配，不要经常在onDraw()方法中new对象：如果这些临时变量每次的使用都是固定，完全不需要每次循环执行的时候重复创建，我们可以考虑将它们从临时变量转为成员变量，在动画初始化或View初始化时将这些成员变量初始化好，需要的时候直接调用即可；对于不规则图形的绘制我们会需要到Path，并且对于越复杂的 Path，Canvas 在绘制的时候，也会更加的耗时，因此我们需要做的就是尽量优化 Path 的创建过程， 还有Path 类中本身提供reset()和rewind()方法用于复用Path对象， reset()方法是用于对象的复位，rewind()方法在对象的复位基础上还可以让Path对象不释放之前已经分配的内存就，重用之前分配的内存。

#### 4.2、抽取重复运算，尽可能减少浮点运算

在自定义View的时候不难免遇到大量的运算，特别在做无限循环动画时，其逻辑代码会在短时间内被循环往复的调用, 这样如果在逻辑代码中在做过多的重复运算无疑会降低动画的效率，特别是在做浮点运算时，CPU 在处理浮点运算时候、会变的特别的慢，要多个指令周期才能完成。

因此我们还应该努力减少浮点运算，在不考虑精度的情况下，可以将浮点运算转成整型来运算，同时我们还应该把重复的运算从逻辑代码中抽取出来，不用每次都运算，例如在WaveLoadingView中， 我创建Path的过程的计算大部分都是在onLayout()中成，把重复运算的结果提前用Path保存好，然后在onDraw()中使用，因为onDraw()在做动画时会被频繁的被调用。

#### 4.3、考虑使用SurfaceView 

传统的View的测量、布局、绘制都是在UI线程中完成的，而Android 的UI线程除了View的绘制之外，还需要进行额外的用户处理逻辑、轮询消息事件等，这样当View的绘制和动画比较复杂，计算量比较大的情况，就不再适合使用 View 这种方式来绘制了。这时候我们可以考虑使用SurfaceView ，SurfaceView 能够在非 UI 线程中进行图形绘制，释放了 UI 线程的压力。当然WaveLoadingView也可以使用SurfaceView 来实现。

## 结语

WaveLoadingView的实现就讲解完毕，更多实现查看文末地址，本次自定义View的过程都使用了kotlin进行编写，整体的代码量的确比java的减少了许多，但语言毕竟只是一个工具，我们主要是学习自定义View的实践过程，当你经常动手实践后，你会发现自定义View没有想象那么难，来来去去就那几个方法，大部分时间都是花在实现的细节和运算上，所以下一次就是自定义ViewGroup实践了。

地址：[WaveLoadingView](https://github.com/rain9155/WaveLoadingView)

