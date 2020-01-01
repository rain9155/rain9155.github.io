---
title: 自定义View实践-使用贝塞尔曲线实现一个loading控件
tags: view
categories: 自定义View
---

## 前言

上一篇文章：[仿微信滑动按钮](https://juejin.im/post/5d48e06d51882505723c9d30)

本文是自定义View实践第二篇，上一篇实现了一个简单的滑动按钮，知道了一些自定义View的基本步骤，本文是使用贝塞尔曲线实现的一个加载中控件，接下来进入正文讲解。

地址：[WaveLoadingView](https://github.com/rain9155/WaveLoadingView)

## 效果图

{% asset_img waveloadingview1.gif waveloadingview %}

可以看到，WaveLoadingView除了用于loading外，还可以用于显示进度的场景。

## 实现方式

在效果图中，波浪是曲线的形式的，所以我们需要想办法把曲线画出来，在数学领域中，用于实现曲线的函数有很多种，但在开发中，比较常用的就是正弦曲线和贝塞尔曲线了，下面简单介绍一下:

### 1、正弦曲线

正弦曲线是我们非常熟悉的曲线，它的函数如下：

**y = Asin(ωx + φ) + h**

**A**表示振幅，用于表示曲线的波峰和波谷的距离;

**ω**表示角速度，用于控制正弦曲线的周期;

**φ**表示初相，用于控制正弦曲线的左右移动;

**h**表示编剧，用于控制曲线的上下移动.

当A、ω、h取一定的值，φ取不同的值时，就可以让曲线在水平方向移动起来，如下：

{% asset_img waveloadingview2.gif waveloadingview %}

上面是A = 2，ω = 0.8， h = 0， φ不断变化的正弦曲线。

### 2、贝塞尔曲线

贝塞尔曲线有一阶、二阶、... 、n阶，一阶的贝塞尔曲线是一条直线，从第2阶开始才是曲线，n阶的贝塞尔曲线可以由(n - 1)阶贝塞尔曲线推导出来，关于贝塞尔曲线的推导可以阅读[**深入理解贝塞尔曲线**](https://juejin.im/post/5b854e1451882542fe28a53d#heading-1)。

这里我使用二阶贝塞尔曲线，它的函数如下：

**f(t) = (1- t)^2 * P0 + 2t(1- t)P1 + t^2 * P2   (0<= t <= 1)**

**P0、P1、P2**都是已知的点，称为控制点,   **t**是一个变量，范围为0到1，函数值随着t的变化而变化.

下面我们取P0 = **(x0,  y0)** = **(-20,  0)**，P1 = **(x1,  y1)** = **(-10,  20)**，P2 = **(x2,  y2)** = **(0,  0)**，然后把这3个点的值代入二阶贝塞尔曲线函数，形成的曲线如下：

{% asset_img waveloadingview3.png waveloadingview %}

**图一**

这样就画出了一条曲线（那两条直线是用于辅助的），接下来我们继续取P3 = **(x3, y3)** = **(10,  -20)**，P4 = **(x4,  y4)** = **(20,  0)**，然后把P2、P3、P4**再次代入**二阶贝塞尔曲线函数，形成的曲线如下：

{% asset_img waveloadingview4.png waveloadingview %}

**图二**

这样就有点接近正弦曲线了，只要我们**不断的取控制点，不断的代入二阶贝塞尔曲线函数，就可以形成一条周期的曲线**，到这里我们也发现了二阶贝塞尔曲线函数不是一个周期函数，所以它不像正向曲线那样连绵不绝，一个二阶贝塞尔曲线函数一次只能通过3个控制点画出一条曲线。

### 3、如何选择？

我们也发现了贝塞尔曲线相对正弦曲线的实现有点复杂，但是，在Android中，贝塞尔曲线已经有了**封装好的api**供我们使用，使用起来非常简单，不需要我们去用代码实现那个函数，相反正弦曲线就需要我们从零做起，要用代码去实现正弦函数，还要进行大量计算、范围检查等，所以从使用的复杂来看，**选用贝塞尔曲线**的工作量更小一点。

在Android中，贝塞尔曲线是通过**Path**来实现的，在Path中，与二阶贝塞尔曲线有关的函数是：

```kotlin
path.quadTo(x1, y1, x2, y2)//绝对坐标
path.rQuadTo(x1, y1, x2, y2)//相对坐标
```

再贴一次图一的贝塞尔曲线：

{% asset_img waveloadingview3.png waveloadingview %}

**图一**

假设坐标系参考图一的xy轴，即x轴向右，y轴向上，原点是(0, 0)， 通过以下代码就可以画出上图的曲线，如下：

```kotlin
var path = Path()
path.moveTo(-20f, 0f)//(x0,  y0) = (-20,  0)
path.quadTo(
    -10f, 20f, //(x1,  y1) = (-10,  20)
    0f, 0f     //(x2,  y2) = (0,  0)
)

//上面是绝对坐标，下面代码使用相对坐标的方式画出

var path = Path()
path.moveTo(-20f, 0f)//(x0,  y0) = (-20,  0)
path.rQuadTo(
    10f, 20f,   //(x1,  y1)相对(x0,  y0)为(10, 20)
    20f, 0f     //(x2,  y2)相对(x0, y0)为(20, 0)
)
```

如果想要画出图二的贝塞尔曲线，只需要在前面曲线的基础上再加一句**quadTo或rQuadTo**，如下：

```kotlin
var path = Path()
path.moveTo(-20f, 0f)//(x0,  y0) = (-20,  0)
path.quadTo(
    -10f, 20f, //(x1,  y1) = (-10,  20)
    0f, 0f     //(x2,  y2) = (0,  0)
)
path.quadTo(
    10f, -20f,  //(x3,  y3) = (10,  -20)
    20f, 0f    //(x4,  y4) = (20,  0)
)


//上面是绝对坐标，下面代码使用相对坐标的方式画出

var path = Path()
path.moveTo(-20f, 0f)//(x0,  y0) = (-20,  0)
path.rQuadTo(
    10f, 20f,   //(x1,  y1)相对(x0,  y0)为(10, 20)
    20f, 0f     //(x2,  y2)相对(x0, y0)为(20, 0)
)
path.rQuadTo(
    10f, -20f,  //(x3,  y3)相对(x2,  y2)为(10, -20)
    20f, 0f     //(x4,  y4)相对(x2, y2)为(20, 0)
)
```

**绝对坐标**的每个点都是以**坐标系的原点**为参考；而**相对坐标**是以**moveTo方法那个点**为原点作为参考，如果只调用了一次moveTo方法，而调用了多次rQuadTo方法，那么从第二次rQuadTo方法开始，它参考**上一次rQuadTo方法的最后一个坐标值**，例如上面相对坐标计算中，第二次rQuadTo方法的(x3,  y3)，(x4,  y4)是参考(x2,  y2)计算出来的，而不是参考(x0,  y0)。

> 上面是为了讲解方便把坐标系说成x轴向右，y轴向上，但是在android中，坐标系是**x轴向右，y轴向下，原点是View的左上角**，这一点要注意。

## 实现步骤

下面开始讲主要的实现步骤：

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

{% asset_img waveloadingview5.png  waveloadingview3 %}

绿色框就是控件作画的范围大小，红色框就是控件大小，也就是说每次控件大小确定之后，我只取中间的部分绘制，很多人会有疑问？为什么只取中间的部分绘制，而不在整个控件范围绘制？这是因为**当控件的父布局是ConstraintLayout，控件宽或高取match_parent时**，会出现以下情况：

{% asset_img waveloadingview6.png  waveloadingview4 %}

**图1** ：控件大小：layout_width = "match_parent" ， layout_height = "200dp"

{% asset_img waveloadingview7.png  waveloadingview5 %}

**图2**：控件大小：layout_width = "200dp" ， layout_height = "match_parent"

蓝色框就是手机屏幕，黑色背景就是控件大小，你还记得我上面在onMeasure()方法讲过，如果控件的形状是圆形，那么控件的测量宽高应该相等的，并取最小值为基准，所以如果控件大小输入是layout_width = "match_parent" ，layout_height = "200dp" 或 layout_width = "200dp" ，layout_height = "match_parent"，经过测量后控件大小应该是**宽 = 高 = 200dp**，效果应该都是如下图：

{% asset_img waveloadingview8.png  waveloadingview6 %}

**图3**

可实际情况却不是图3，而是图1或图2，这是因为**ConstraintLayout布局会让子控件的setMeasuredDimension()失效**，所以导致 measuredHeight 和 height 不一样，宽同理，所以在遇到父布局是ConstraintLayout时，并且控件的宽或高设置了“match_parent”，并且你自定义了测量过程，就会导致自定义View过程中测量出来大小不等于View最终大小，即**getMeasureHeigth()或getMeasureWidth() != getWidth()或getHeigth()**，为什么ConstraintLayout就会有这种情况而其他如Linearlayout就没有？我也不知道，可能需要大家通过源码了解了，而我的解决办法就是让每次作画的范围在控件的中心，就像图1和图2一样，这样就不会那么难看。

### 2、裁剪画布形状

怎么把控件弄成圆形、正方形、矩形这些形状，如果控件形状是正方形或矩形，还可以设置圆角，一个方法是通过BitmapShader实现，使用BitmapShader要经过3步：

1、新建Bitmap；

2、以1新建的Bitmap创建一个Canvas，在Canvas上画出波浪；

3、最后新建一个BitmapShader与1的Bitmap关联，然后设置给画笔，用画笔在onDraw方法传进来的Canvas上画一个形状出来，然后这个形状就会含有波浪.

但我没有使用BitmapShader，因为波浪的移动需要开启一个无限循环动画，就会不断的调用onDraw()方法，而在onDraw()方法不断的新建对象是一个不推荐的做法，虽然Bitmap可以通过recycler()复用，但是还是避免不了每次都要新建Canvas对象, 所以为了减少对象分配，我使用了Canvas的**clipPath**API来把画布裁剪成我想要的形状，然后把波浪画在裁剪后的画布上，这样也能实现与BitmapShader同样的效果，如下：

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

preDrawShapePath()中根据Shape来add不同的形状给Path来把这些路径信息预先保存下来，前面已经讲过每次作画的范围都在控件的中心，//...省略的都是居中计算，保存好形状的Path将在onDraw方法中使用，如下：

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

{% asset_img waveloadingview9.png  waveloadingview7 %}

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

{% asset_img waveloadingview10.png  waveloadingview8 %}

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

{% asset_img waveloadingview11.png  waveloadingview9 %}

### 4、让波浪动起来

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

### 5、优化

大家都知道手机的资源都是非常有限的，我在做自定义View时，特别是涉及到无限循环的动画时，要注意优化我们的代码，因为一般的屏幕刷新周期是16ms，这意味着在这16ms内你要把有关动画的所有计算和流程完成，不然就会造成掉帧，从而卡顿，在自定义View时我想到可以从下面几点做一些优化，提高效率：

#### 5.1、减少对象的内存分配，尽可能做到对象复用

每次系统GC的时候都会暂停系统ms级别的时间，而无限循环的动画的逻辑代码会在短时间内被循环往复的调用, 这样如果在逻辑代码中在堆上创建过多的临时变量，会导致内存的使用量在短时间内上升，从而频繁的引发系统的GC行为，这样无疑会拖累动画的效率，让动画变得卡顿。

在自定义View涉及到无限循环动画时，我们不能忽略对象的内存分配，不要经常在onDraw()方法中new对象：如果这些临时变量每次的使用都是固定，完全不需要每次循环执行的时候重复创建，我们可以考虑将它们从临时变量转为成员变量，在动画初始化或View初始化时将这些成员变量初始化好，需要的时候直接调用即可；对于不规则图形的绘制我们会需要到Path，并且对于越复杂的 Path，Canvas 在绘制的时候，也会更加的耗时，因此我们需要做的就是尽量优化 Path 的创建过程， 还有Path 类中本身提供reset()和rewind()方法用于复用Path对象， reset()方法是用于对象的复位，rewind()方法在对象的复位基础上还可以让Path对象不释放之前已经分配的内存就，重用之前分配的内存。

#### 5.2、抽取重复运算，尽可能减少浮点运算

在自定义View的时候不难免遇到大量的运算，特别在做无限循环动画时，其逻辑代码会在短时间内被循环往复的调用, 这样如果在逻辑代码中在做过多的重复运算无疑会降低动画的效率，特别是在做浮点运算时，CPU 在处理浮点运算时候、会变的特别的慢，要多个指令周期才能完成。

因此我们还应该努力减少浮点运算，在不考虑精度的情况下，可以将浮点运算转成整型来运算，同时我们还应该把重复的运算从逻辑代码中抽取出来，不用每次都运算，例如在WaveLoadingView中， 我创建Path的过程的计算大部分都是在onLayout()中成，把重复运算的结果提前用Path保存好，然后在onDraw()中使用，因为onDraw()在做动画时会被频繁的被调用。

#### 5.3、考虑使用SurfaceView 

传统的View的测量、布局、绘制都是在UI线程中完成的，而Android 的UI线程除了View的绘制之外，还需要进行额外的用户处理逻辑、轮询消息事件等，这样当View的绘制和动画比较复杂，计算量比较大的情况，就不再适合使用 View 这种方式来绘制了。这时候我们可以考虑使用SurfaceView ，SurfaceView 能够在非 UI 线程中进行图形绘制，释放了 UI 线程的压力。当然WaveLoadingView也可以使用SurfaceView 来实现。

## 结语

WaveLoadingView的实现就讲解完毕，本次自定义View的过程都使用了kotlin进行编写，整体的代码量的确比java的减少了许多，但语言毕竟只是一个工具，我们主要是学习自定义View的实践过程，当你经常动手实践后，你会发现自定义View没有想象那么难，来来去去就那几个方法，大部分时间都是花在实现的细节和运算上，更多实现请查看文末地址。

地址：[WaveLoadingView](https://github.com/rain9155/WaveLoadingView)

