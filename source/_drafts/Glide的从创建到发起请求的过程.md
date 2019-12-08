---
layout: drafts
title: Glide的从创建到发起请求的过程
date: 2019-12-05 17:48:10
tags: 
- glide
- 源码
categories: 优秀开源库分析
---

## 前言

源码基于Glide4.8。

## 构造方法

在构造Glide的时候，Glide会初始化3个重要的组件：

- 1、**RequestManagerRetriever**：用于管理和创建RequestManager
- 2、**Engine**：用于启动加载图片的EngineJop(异步线程)，管理图片资源的缓存
- 3、**Registry**：注册表，根据图片类型XX.class，为每一个图片类型注册一个图片解码器ResourceDecoder和编码器ResourceEncoder，和根据图片model，即load传入的XX，为每一个图片model注册一个ModelLoader，通过ModelLoader的buildLoadData方法可以为每一个图片加载创建一个LoadData，LoadData中有一个DataFetcher，是真正加载图片的地方（从网络、从磁盘等）

## with()

with(XX)方法里面会调用到RequestManagerRetriever的get(XX)方法，get(XX)方法里面根据传入的参数把Context分为两种类型：1、Application的Context；2、非Application的Context(如Activity、Fragment、View)，分别处理:

- 1、**Application的Context情况**：如果是Application的Context，get(XX)就返回一个和Application关联的RequestManager，并创建一个ApplicationLifecycle传进RequestManager中，但ApplicationLifecycle里面是个空实现，没有add和remove LifecycleListener的功能，这样Glide就和Application的生命周期绑定在一起了，如果应用程序关闭，Glide的加载自然会终止；
- 2、**非Application的Context情况**：如果是非Application的Context，这样Context就属于Activity或Fragment或View，无论是哪一种情况，都可以间接的转换成属于Actiivty的Context，这样RequestManagerRetriever就会把Context强转成对应的Activity，然后向对应的Activity添加一个无界面的Fragment，所以最终会返回一个和Fragment关联的RequestManager，并把Fragment中创建的ActivityFragmentLifecycle传进RequestManager，ActivityFragmentLifecycle里面实现了Lifecycle，可以add和remove  LifecycleListener，这样Glide就和Fragment的生命周期绑定在一起了，如果Activity被销毁了，Fragment就可以自动监听到，onDestory回调，就可以通过LifecycleListener回调出去，这样Glide的加载也会同时终止.

**小结：**

Glide的with(XX)方法最终返回一个和Application关联或和Fragment关联的RequestManager，这个RequestManager在整个应用中只会创建一次并实现了LifecycleListener，并持有一个来自Application或Fragment的Lifecycle。

RequestManager会把自己添加到Lifecycle中，由于Lifecycle，RequestManager就拥有了感知生命周期的能力，而RequestManager的作用就是用来创建和管理用于加载图片的Request，所以RequestManager就有能力在Actiivty或Application销毁时接收到回调，然后通过RequestTracker终止Request(图片的加载)。

## load()

根据with(XX)得到的RequestManager，执行RequestManager的load(XX)方法，load(XX)方法主要根据传入的图片路径或图片通过asXX()方法得出要加载的图片类型，有4种图片类型：

- 1、**string、url、byte[]、id、drawable、bitmap、object、File类型或调用了asDrawable()方法**：得到Drawable.class；
- 2、**调用了asFile()方法**：得到File.class；
- 3、**调用了asBitmap()方法**：得到Bitmap.class；
- 4、**调用了asGif()方法**：得到GifDrawable.class.

asXX(XX)方法最终调用到as(class)方法，as(class)方法中会创建RequestBuilder并把图片类型.class传进RequestBuilder中同时赋值给Class<TranscodeType> 类型的transcodeClass字段，创建RequestBulder后，接着在RequestManager的load(XX)中继续执行RequestBuilder的load(XX)，所有的load(XX)都会调用到loadGeneric(object)，XX会赋值给Object类型的model字段。

**小结：**

RequestManager的load(XX)，最终的返回一个RequestBuilder，并把图片类型.class赋值给Class<TranscodeType> 类型的transcodeClass字段和把XX赋值给Object类型的model字段。

一个RequestBuilder对应一个Request，RequestBuilder的build方法将会在into中执行，而transcodeClass代表不同图片类型的转码类型（用来编码、解码图片），model代表不同图片类型的加载（用来加载图片）。

## into()

根据load(XX)得到的RequestBuilder，执行Builder的into(imageView)方法，RequestBuilder可以apply一个RequestOptios来配置Glide的图片加载过程，在into方法首先会调用GlideContext的buildImageViewTarget方法创建一个Target，然后把Target、RequestOptios作为参数调用带3个参数的into方法：

### 1、buildImageViewTarget()

GlideContext的buildImageViewTarget(imageView, transcodeClass)方法，根据transcodeClass的类型，分三种情况返回Target：

- 1、**Bitmap.class=transcodeClass**：返回BitmapImageViewTarget，并把imageView传进Target;
- 2、**Drawable.class=transcodeClass**：返回DrawableImageViewTarget，并把imageView传进Target;
- 3、**不是1和2两种情况**：报错。

### 2、带3个参数的into方法

into(target, null, options)方法中首先通过buildRequest方法构建一个Request，并把Request设置给Target，然后通过RequestManager的track方法执行Request：

#### 2.1、buildRequest方法

RequestBulder的buildRequest(target, targetListener, options)方法：

- 构造ErrorRequest；
- 如果ErrorRequest为null，构造ThumbnailRequest；
- 如果ThumbnailRequest为null，构造SingleRequest。

#### 2.2、RequestManager的track方法

RequestManager的track(target, request)方法，这个方法首先调用TargetTracker的track方法把Target放入TargetTracker中的一个set集合，方便控制所有Target，然后再通过RequestTracker的runRequest方法来执行Request：

- 1、TargetTracker的track(target)方法：TargetTracker实现了LifrcycleListener；

- 2、RequestTracker的runRequest(request)方法：里面首先把Request放入RequestTracker的一个set集合中，然后判断Glide是否时paused状态，如果不是pause状态就调用Request的begin方法启动图片加载，否则的话就先将Request添加到待执行队列pending里面，等暂停状态解除了之后再执行，**转到2.3**。

#### 2.3、Request的begin方法(SingleRequest)

Request的begin方法(SingleRequest):

- 如果model等于null，model就是load(XX)传进来的XX，调用onLoadFailed方法，通知Target显示ErrorDrawable;
- 如果model不等于null，但是这个Request在RUNNING状态，报错;
- Request不在RUNNING状态，但是这个Request在COMPLETE状态，调用onResourceReady方法，通知Target显示照片;
- 这个Request不在COMPLETE状态，计算ImageView的大小，调用onSizeReady方法，进入RUNNING状态，通知Target显示PlaceholderDrawable，然后调用Engin的load方法准备进行图片加载并传入Request的回调，**转到2.4**。

**图片显示回调：**SingleRequest的onResourceReady方法：

- 里面进行一些错误判读后，从Resource<Drawable>取出Drawable，如果Drawable.class不等于原本的transcodeclass，就通知Target显示错误视图；
- 否则，调用onResourceReady方法，进入COMPLETE状态，然后通过Target回调，把Drawable回调给Target显示；
- 接着Target的onResourceReady方法回调，在里面Target调用setDrawable把图片显示出来。

#### 2.4、Engine的load方法

Engine的load方法：

- load方法中首先判断弱引用缓存中有没有图片资源EngineResource，如果有，直接通过回调把图片资源回调给SingleRequest;
- 如果弱引用缓存中没有图片资源，就判断LruCache中有没有图片资源，如果有，直接通过回调把结果回调给SingleRequest;
- 如果没有内存缓存，就创建一个EngineJob，用于异步加载图片，和创建一个DecodeJob，用于图片加载完成后的解码，然后通过EngineJob的start方法开启异步加载，**转到2.5**

#### 2.5、EngineJob的start方法

EngineJob的start方法:

里面会通过Glide获取一个Executor，然后执行DecodeJob任务，而DecodeJob实现了Runnable接口，注意EngineJob没有实现Runnable接口，它只实现了DecodeJob的相关回调接口，所以DecodeJob即用来加载图片也用来解码图片，然后就会异步执行DecodeJob的run方法，**转到2.6**。

**图片加载完成并解码后的回调：**EngineJob的onResourceReady方法:

- 里面首先通过Handler发送了一个MSG_COMPLETE消息，接着就会在Handler的handleMessage方法中的MSG_COMPLETE分支调用EngineJob的handleResultOnMainThread()方法
  - **EngineJob的handleResultOnMainThread()方法:**
    - 把图片用EngineResource包装，然后调用EngineResource的release方法把图片引用加一，然后把EngineResource和key回调给Engine的onEngineJobComplete方法;
    - 然后把EngineResource<Drawable>和DataSource回调给每一个Request的onResourceReady方法.

#### 2.6、DecodeJob的run方法

DecodeJob的run方法：

- 1、run方法里面会调用DecodeJob的runWrapped方法，在runWrapped方法中，进入INITIALIZE分支，getNextStage（根据硬盘缓存策略获取一个State） -> getNextGenerator（根据State生成一个Generator）->
  - Stage.RESOURCE_CACHE，表示从硬盘缓存读取，这个缓存是图片解码后的缓存；
  - Stage.DATA_CACHE，表示从硬盘缓存读取，这个缓存是图片解码前的缓存；
  - Stage.SOURCE，表示从源（网络或本地）获取.

- 2、然后执行runGenerators方法，在这个方法中首先会执行Generator的startNext方法，如果不满足条件，返回false，然后又执行getNextStage -> getNextGenerator，如此递归，最终会获取一个SourceGenerator，这个SourceGenerator中持有DecodeHelper，和DecodeJob的回调，然后调用DecodeJob的reschedule重新执行run方法，并把会进入SWITCH_TO_SOURCE_SERVICE分支，在这里又执行runGenerators方法，而这次Generator是SourceGenerator，所以接下来就是执行SourceGenerator的startNext方法, **转到2.7**。

**图片流回调：**DecodeJop的onDataFetcherReady方法，里面有两个选择

- 1、在里面首先判断是否是子线程，如果不在子线程，再reschedule一次，在run方法中就会进入DECODE_DATA分支，然后在子线程调用decodeFromRetrievedData方法解码图片；
- 2、如果是在子线程，直接调用，decodeFromRetrievedData方法解码图片；
- 所以最终还是调用DecodeJob的decodeFromRetrievedData方法解码图片，里面会调用decodeFromFetcher方法
- **Decodejob的decodeFromFetcher方法：**
  - 里面调用DecodeHelper的getLoadPath方法，传入图片流.class，getLoadPath方法里面首先会通过Glide获取注册表，然后通过图片流.class和在load阶段得到的transcodeClass来一个LoadPath，这个LoadPath里面会持有一个DecodePath列表，每一个DecodePath都持有图片流类型，和一个图片解码器ResourceDecoder列表，和一个图片转码器ResourceTranscoder；
  - 得到LoadPath后，通过LoadPaht的load方法来解码图片，load方法会从DecodePath列表找到一个可以解码这张图片的DecodePath；
  - 得到DecodePath后，调用DecodePath的decode方法来解码图片，decode方法会图片解码器ResourceDecoder列表中找到一个可以解码这张图片的解码器；
  - 得到解码器ResourceDecoder后，以BitmapDrawableDecoder为例，然后调用ResourceDecoder的decode方法，将图片进行解码后，得到一个Resource<BitmapDrawable>；
  - 得到一个Resource<BitmapDrawable>后，将Resource<BitmapDrawable>通过ResourceTranscode转成Glide的Target支持展示的图片类型，因为默认的Target只支持展示Bitmap和Drawable，假设经过transcoder转化后Resource<BitmapDrawable>变成了Resource<Drawable>，然后返回，一路返回到decodeFromRetrievedData方法，然后调用notifyEncodeAndRelease方法
    - **notifyEncodeAndRelease方法：**
      - 里面会调用notifyComplete方法，然后notifyComplete方法中会调用Request的回调，把根据图片流解码得到的Resource<Drawable>和表示图片来源的DataSource(网络、本地等)回调给EngineJob的onResourceReady方法.

#### 2.7、SourceGenerator的startNext方法

SourceGenerator的startNext方法：

- 里面首先把上次网络请求成功的结果dataToCache缓存到硬盘中去，然后直接从硬盘中读取返回；
- 然后通过DecodeHelper的getLoadData方法获取一个LoadDate，getLoadData方法里面首先通过Glide获取注册表Registry，然后通过Registry获取ModelLoader列表，然后从modelLoader列表中找出一个合适的modelLoader，然后调用modelLoaders的buildLoadData方法传入model创建一个LoadData返回，这个LoadData会持有一个相应model的DataFetcher如HttpUrlFetcher；
- 有了LoadData后，就通过LoadData的DataFetcher的loadData方法，传入SourceGenerator回调，开始加载图片，这里假设从网络加载图片，所以DataFetcher就是HttpUrlFetcher，**转到2.8**.

**图片流回调：**SourceGenerator的onDataReady方法，里面两个选择：

- 1、如果硬盘的缓存策略可以缓存远程返回的图片流（未解码前的图片），把流赋值给dataTocache，等待下一次reschedule，把dataToCache缓存到硬盘，然后直接从硬盘中读取
- 2、通过DecodeJob回调的onDataFetcherReady方法把图片流回调出去

#### 2.8、HttpUrlFetcher的loadData方法

- 里面最终通过HttpURLConnection通过Uri从网络获取图片的InputStream，获取到InputStream后，就通过SourceGenerator回调的onDataReady方法把流回调出去，**结束**。



