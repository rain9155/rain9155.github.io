---
title: Android消息机制(java层)
date: 2019-02-21 13:33:11
tags: 
- handler
- 源码
categories: 消息机制
---

## 前言
Android的消息机制用于**同进程的线程间通信**，它是由MessageQueue，Message，Looper，Handler共同组成，Android中大量的交互都是通过消息机制，比如四大组件启动过程与服务的交互、View的绘制、更新等都离不开消息机制，所以Android在某种意义上也可以说成是一个**以消息驱动的系统**，在Android中消息机制的运作分为java层和native层，它们之间的运作机制不一样，本文讲解的是java层的消息机制，如果你想了解native层的消息机制，可以阅读：

[Android消息机制(native层)](http://rain9155.coding.me/2019/02/21/Android消息机制native层/)

但其实java层的消息机制的核心功能都交给了native层的消息机制来完成，但作为应用开发，首先要掌握java层的消息机制。

> 本文源码基于Android8.0，源码相关位置:
> frameworks/base/core/java/android/os/*.java  (\*代表MessageQueue、Handler、Looper、Message)

## 消息机制概述
Android应用的每个事件都会转化为一个系统消息即Message，消息中包含了事件的相关信息和消息的处理人即Handler，消息要通过Handler发送，最终被投递到一个消息队列中即MessageQueue，它维护了一个待处理的消息列表，然后通过Looper开启了一个消息循环不断地从这个队列中取出消息，当从消息队列取出一个消息后，Looper根据消息的处理人（target）将此消息分发给相应的Handle处理，整个过程如下图所示。

{% asset_img handler1.png %}

它们的工作原理就像工厂的生产线，Looper是发动机，MessageQueue是传送带，Handler是工人，Message则是待处理的产品。

## java层消息机制架构图
{% asset_img handler2.png %}
* **Looper**  --- 是每个线程的MessageQueue管家，里面有一个MessageQueue消息队列，负责把消息从MessageQueue中取出并把消息传递到Handler中去，每个线程只有一个Looper；
* **MessageQueue**  ---  消息队列，有一组待处理的Message，主要用于存放所有通过Handler发送的消息，每个线程只有一个MessageQueue；
* **Message**  ---  是线程之间传递的消息，里面有一个用于处理消息的Handler；
* **Handler**  ---  主要用于发送和处理消息，里面有Looper和MessageQueue。

## 简单使用

在开发中，我们在子线程中执行完操作后通常需要更新UI，但我们都知道不能在子线程中更新UI，此时我们就要通过Handler将一个消息post到UI线程中，然后再在Handler中的handleMessage方法中进行处理，如果我们不传递UI线程所属的Looper去创建Handler，那么该Handler必须在主线程中创建，如下：

```java
//在主线程中创建Handler 
Handler mHandler = new Handler（）{
 @Override
 public void handleMessage(Message msg){
    //更新UI
 }
}

//在子线程中进行耗时操作
new Thread（）{
	public void run(){
		mHandler.sendEmptyMessage(0);
	}
}
```

以上就是我们平时使用Handler的常规用法了。

接下来我们以应用主线程(又叫做UI线程)的消息机制运作为例讲解Android消息机制的原理。

## 源码分析

### 1、 Looper的创建
我们知道Android应用程序的入口实际上是ActivityThread.main方法，而应用的消息循环也是在这个方法中创建，具体源码如下：
```java
//ActivityThread.java 
public static void main(String[] args) {
 	 //...
 	 //1、创建消息循环Looper
 	 Looper.prepareMainLooper();
 	 //...
 	 //2、执行消息循环
 	 Looper.loop();
 }
```
我们关注注释1，ActivityThread调用了Looper的prepareMainLooper方法来创建Looper实例，Looper的prepareMainLooper方法如下：

```java
//Looper.java 
public static void prepareMainLooper() {
    //1、创建不允许退出的Looper
    prepare(false);
    synchronized (Looper.class) {//保证只有一个线程执行
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already 			been 		prepared.");
        }
        //2、将刚刚创建的Looper实例赋值给sMainLooper
        //sMainLooper字段是Looper类中专门为UI线程保留的，只要某个线程调用了prepareMainLooper方法，把它创建的Looper实例赋值给sMainLooper，它就可以成为UI线程，prepareMainLooper方法只允许执行一次
        sMainLooper = myLooper();
    }
}

private static void prepare(boolean quitAllowed){//quitAllowed表示是否允许Looper运行时退出
	//Looper.prepare()只能执行一次
	if (sThreadLocal.get() != null) {
		throw new RuntimeException("Only one Looper may be created per thread"); 
	 }
	//把Looper保存到TLS中
	sThreadLocal.set(new Looper(quitAllowed));
}

public static @Nullable Looper myLooper() {
	//获取当前线程TLS区域的Looper
	return sThreadLocal.get();
}

private static Looper sMainLooper;  // guarded by Looper.class
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```

线程在调用prepareMainLooper方法后，就代表它成为了一个UI线程，可以看到prepareMainLooper方法中是调用Looper的**prepare方法**来创建Looper实例，当要获取创建的Looper实例时，是通过Looper的**myLooper**方法来获取的，在调用prepare方法时，会把创建的Looper实例set到**ThreadLocal**中，当获取时也会从**ThreadLocal**中通过get方法获取，那么ThreadLocal是什么？这里简单介绍一下ThreadLocal：

> **ThreadLocal**：线程本地存储区（Thread Local Storage，简称为TLS），每 个线程都有自己的私有的本地存储区域，不同线程之间彼此不能访问对方的TLS区域。
>
> **它的常用操作方法有**：
> ThreadLocal.set(T value)：将value存储到当前线程的TLS区域。
> ThreadLocal.get()：获取当前线程TLS区域的数据
>
> 关于ThreadLocal更多信息可以查看[ThreadLocal原理解析](https://rain9155.github.io/2019/02/21/ThreadLocal解析).

ThreadLocal的get()和set()方法操作的类型都是泛型，我们从ThreadLocal在Looper中的定义可以看出，sThreadLocal的get()和set()操作的类型都是Looper类型,  所以，由于ThreadLocal的作用，**每个线程只能保存一个Looper，不同线程的Looper是不相同的**，这样，通过调用**Looper.prepare(false)**方法，UI线程中就保存了它**对应的、唯一的**Looper实例，当在UI线程调用**Looper.myLooper**方法时它就会返回UI线程关联的Looper实例，当然，如果你在子线程中想要获得UI线程关联的Looper实例，就需要调用**getMainLooper**方法，该方法如下：

```java
public static Looper getMainLooper() {
    synchronized (Looper.class) {
        //返回UI线程的Looper实例
        return sMainLooper;
    }
}
```

以上就是UI线程的Looper的创建过程，执行ActivityThread.main后，应用程序就启动了，UI的消息循环也在Looper.loop方法中启动，此后Looper会一直从消息队列中取出消息，用户或系统通过Handler不断往消息队列中添加消息，这些消息不断的被取出，处理，回收，使得应用运转起来。

我们在平时开发时，一般是使用**不带参数的prepare()方法**来创建子线程对应的Looper实例，如下：

```java
//我们平时使用的prepare方法
public static void prepare() {
    //quitAllowed = true
    prepare(true);
}
```

和UI线程的区别是，UI线程的Looper是不允许推出的，而我们的Looper一般是允许退出的。

### 2、MessageQueue的创建

我们在上面知道，调用Looper的prepare方法就会创建Looper实例，同时会把Looper实例通过ThreadLocal保存到线程中，在创建Looper时还会同时在构造中创建**MessageQueue**实例，如下：

```java
//Looper.java
final MessageQueue mQueue;
//Looper唯一的构造函数
private Looper(boolean quitAllowed) {
    //可以看到MessageQueue是在Looper中创建的
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

Looper只有这一个构造函数，并且它是私有的，所以我们只能通过Looper的prepare方法创建Looper实例，由于ThreadLocal，每个线程最多只能对应一个Looper实例，而每个Looper内部只有一个MessageQueue实例，推出：**每个线程最多对应一个MessageQueue实例**。

我们看一下MessageQueue的构造，如下：

```java
//MessageQueue.java
private long mPtr; // used by native code
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    //通过native方法初始化消息队列，其中mPtr是供native代码使用
    mPtr = nativeInit();
}
```

MessageQueue的构造中会通过native方法在**native层**创建一个属于native层的消息队列NativeMessageQueue，然后把NativeMessageQueue的地址返回给**java层**保存在**mPtr**中，java层和native层之间的通信就通过这个**mPtr**指针。

> MessageQueue是消息机制的核心类，它是java层和native层的连接纽带，它里面有大量的native方法，Android有俩套消息机制（java层和native层，实现不一样），但本文只讲解java层的消息机制，不会涉及到native层.
>
> 关于native层的查看[Android消息机制（native层）](http://rain9155.coding.me/2019/02/21/Android消息机制native层/)

我们通过Looper的**myQueue**方法就能获取到它关联的MessageQueue实例，如下：

```java
//Looper.java
public static @NonNull MessageQueue myQueue() {
    //先通过TLS获取Looper实例，再从对应的Looper中获取MessageQueue实例
    return myLooper().mQueue;
}
```

### 3、消息循环的运行
在ActivityThread的mian方法在创建Looper后，通过**Looper.loop**方法就启动了消息循环，这个函数会不断的从MessageQueue中取出消息、处理消息，我们点进此方法看一下它的源码：
```java
//Looper.java
public static void loop() {   
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    //从Looper中取出MessageQueue
    final MessageQueue queue = me.mQueue;
    //...
    for (;;) {
        //1、从MessageQueue中取出消息，没有消息时会阻塞等待
        Message msg = queue.next(); 
        //next()返回了null，表示MessageQueue正在退出，即调用了Looper的quit或quitSafely方法
        if (msg == null) {
            return;
        }
        //...
        //2、分发消息
        msg.target.dispatchMessage(msg);
        //...
        //3、回收消息
        msg.recycleUnchecked();
    }
}
```
loop()中是一个死循环，我们关注注释1，loop()会调用MessageQueue的**next()**来获取最新的消息，当没有消息时，next()会一直阻塞在那里，这也导致loop()阻塞，唯一跳出循环的条件是next()返回null，这时代表Looper的quit()或quitSafely()被调用，从而调用MessageQueue的quit()来通知消息队列退出，如下：

```java
//Looper.java
public void quit() {
    mQueue.quit(false);
}

//quitSafely和quit方法的区别是，quitSafely方法会等MessageQueue中所有的消息处理完后才退出，而quit会直接退出
public void quitSafely() {
    mQueue.quit(true);
}
```

所以MessageQueue的**next()**是最关键的函数，我们来看看next函数的关键代码：

```java
//MessageQueue.java
Message next() {
    //mPtr是在构造中被赋值，是指向native层的MessageQueue
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }
    int pendingIdleHandlerCount = -1; 
    int nextPollTimeoutMillis = 0;
    //一个死循环，里面分为两部分，1、处理java层消息；2、如果没有消息处理，执行IdleHandler
    for(;;){
        //...
        
        //Part1：获取java层的消息处理
        
        //nativePollOnce方法用于处理native层消息，是一个阻塞操作
        //它在这两种情况下返回：1、等待nextPollTimeoutMillis时长后；2、MessageQueue被唤醒
        nativePollOnce(ptr, nextPollTimeoutMillis);
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            //mMessages是java层的消息队列，这里表示取出消息队列的第一个消息
            Message msg = mMessages;
            if (msg != null && msg.target == null) {//遇到同步屏障（target为null的消息）
                //在do-while中找到异步消息，优先处理异步消息
                //异步消息的isAsynchronous方法返回true
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {//有消息
                if (now < msg.when) { //消息还没到触发时间
                    //设置下一次轮询的超时时长（等待时长）
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {//now == msg.when, 消息到达触发时间
                    mBlocked = false;
                    //从mMessages的头部获取一条消息并返回 
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    //设置消息的使用状态，即flags |= FLAG_IN_US
                    msg.markInUse();
                    //返回这个消息
                    return msg;
                }
            } else {//msg == null，没有消息
                //设置nextPollTimeoutMillis为-1，准备进入阻塞，等待MessageQueue被唤醒
                nextPollTimeoutMillis = -1;
            }
            
            //调用了quit方法
            if (mQuitting) {
                dispose();
                return null;
            }
			
            //当处于以下2种情况时，就会执行到Part2：
            //1、mMessages == null，java层没有消息处理
            //2、now < msg.when，有消息处理，但是还没有到消息的执行时间
            //1、2两种情况都表明线程的消息队列处于空闲状态，处于空闲状态，就会执行IdleHandler
            
           //Part2：没有消息处理，执行IdleHandler
           //mIdleHandlers表示IdleHandler列表
           //pendingIdleHandlerCount表示需要执行的IdleHandler的数量
 
            if (pendingIdleHandlerCount < 0
                && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                //没有IdleHandler需要处理，可直接进入阻塞
                mBlocked = true;
                continue;
            }
		   //有IdleHandler需要处理
            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            //把mIdleHandlers列表转成mPendingIdleHandlers数组
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        //遍历mPendingIdleHandlers数组
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            //取出IdleHandler
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler
            boolean keep = false;
            try {
                //执行IdleHandler的queueIdle方法，通过返回值由自己决定是否保持存活状态
                keep = idler.queueIdle();
            }
            //...省略异常处理
            
            if (!keep) {
                // 不需要存活，移除
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }
        
        //重置pendingIdleHandlerCount和nextPollTimeoutMillis为0
        pendingIdleHandlerCount = 0;
        //nextPollTimeoutMillis为0，表示下一次循环会马上从Messages中取出一个Message判断是否到达执行时间
        //因为IdleHandler在处理事件的时间里，有可能有新的消息发送来过来，需要重新检查
        nextPollTimeoutMillis = 0;
    }
}

```
MessageQueue的next方法有点长，但是它里面的逻辑是很好理解的，它主要是通过一个死循环不断的返回Message给Looper处理，next方法可以分为两部分阅读：

**1、获取java层的消息处理**：

我们看Part1，先执行**nativePollOnce**方法，它是一个阻塞操作，其中nextPollTimeoutMillis代表下一次等待的超时时长，当nextPollTimeoutMillis = 0时或到达nextPollTimeoutMillis时，它会**立即返回**；当nextPollTimeoutMillis = -1时，表示MessageQueue中没有消息，会一直等待下去，直到Hanlder往消息队列投递消息，执行**nativeWake**方法后，MessageQueue被唤醒，nativePollOnce就会返回，但它此时并**不是立即**返回，它会先处理完native层的消息后，再返回，然后获取java层的消息处理；

接着next方法就会从mMessages链表的表头中获取一个消息，首先判断它是否是同步屏障，同步屏障就是**target为null**的Message，如果遇到同步屏障，MessageQueue就会优先获取异步消息处理，异步消息就是**优先级**比同步消息高的消息，我们平时发送的就是同步消息，通过Message的**setAsynchronous(true)**可以把同步消息变成异步消息，不管是同步还是异步，都是Message，获取到Message后；

接着判断Message是否到达它的执行时间(**if(now == msg.when)**)，如果到达了执行时间，next方法就会返回这条消息给Looper处理，并将其从单链表中删除；如果还没有到达执行时间，就设置nextPollTimeoutMillis为下一次等待超时时长，等待下次再次取出判断，可以发现虽然MessageQueue叫消息队列，但它却不是用队列实现的，而是用链表实现的。

> 通过MessageQueue的**postSyncBarrier**方法可以添加一个同步屏障，通过**removeSyncBarrier**方法可以移除相应的同步屏障，在Android，**Choreographer机制**中就使用到了异步消息，在View树绘制之前，会先往UI线程的MessageQueue添加一个同步屏障，拦截同步消息，然后发送一个异步消息，等待VSYN信号到来，触发View树绘制，这样就可以让绘制任务优先执行。

**2、没有消息处理，遍历IdleHandler列表，执行IdleHandler的queueIdle方法**：

IdleHandler是什么？IdleHandler是一个接口，它里面只有一个**queueIdle**方法，Idle是空闲的意思，在**MessageQueue空闲**的时候会执行IdleHandler的queueIdle方法，我们可以通过MessageQueue的**addIdleHandler**方法添加我们自定义的IdleHandler到mIdleHandlers列表中，如下：

```java
//MessageQueue.java
private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>(); 
public void addIdleHandler(@NonNull IdleHandler handler) {
    //...
    mIdleHandlers.add(handler);
}
```

MessageQueue空闲，就代表它处于以下两种情况：

(1) mMessages == null，消息队列中没有消息处理；

(2) now < msg.when，有消息处理，但是还没有到消息的执行时间.

以上两种情况之一都会触发next方法遍历IdleHandler列表，执行IdleHandler的queueIdle方法的操作。

> 在Android中，我们平时所说的线程空闲其实就是指线程的MessageQueue空闲，这时就可以执行我们添加的IdleHandler，例如在LeakCanary中，它通过添加IdleHandler，在UI线程空闲时执行内存泄漏的判断逻辑.

### 4、消息的发送

到这里UI线程已经启动了消息循环，那么消息从何而来？消息是由系统产生，然后通过Hanlder发送到MessageQueue中，Handler就是用来处理和发送消息的，应用程序的**Handler**在ActivityThread中被创建，如下：

```java
//ActivityThread.java 
final H mH = new H();

//H定义如下，继承Hanldler
//里面定义了大量的字段，跟Activity的启动，Application的绑定等有关
class H extends Handler {
    public static final int BIND_APPLICATION        = 110;
    public static final int EXIT_APPLICATION        = 111;
    public static final int CREATE_SERVICE          = 114;
    //....
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case BIND_APPLICATION:
                //...
            case EXIT_APPLICATION:
                //...
            case CREATE_SERVICE:
                //...
                //...
        }
        //...   
}
```
**mH**就是应用程序内部使用的Handler，我们外部是不可使用的，应用程序通过Handler往MessageQueue中投递消息，并通过Handler的handlerMessage方法处理消息，而我们在外部也可以使用自己创建的Handler往UI线程的MessageQueue投递消息。

既然Handle可以往MessageQueue中投递消息，这说明Handler**要和相应的MessageQueue关联**，我们看Handler的构造函数，Handler有两种构造函数：

一种是**指定Callback**的构造函数，Callback默认为null，如下：

```java
//Handler.java
public Handler() {
    this(null, false);
}

public Handler(Callback callback) {
    this(callback, false);
}

public Handler(boolean async) {
    this(null, async);
}

public Handler(Callback callback, boolean async) {
    //...
    //先通过TLS获取Looper实例，与Looper关联
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread " + Thread.currentThread()
            + " that has not called Looper.prepare()");
    }
    //通过Looper获取MessageQueue，与MessageQueue关联
    mQueue = mLooper.mQueue;
    //Callback的作用在消息的分发中会讲到
    mCallback = callback;
    mAsynchronous = async;
}

```

在构造中如果通过Looper.myLooper方法获取不到Looper，就会抛出“**Can't create handler inside threadxx that has not called Looper.prepare()**”异常，所以如果我们在子线程中使用Handler的默认构造，没有先调用Looper.prepare方法就创建Handler的话，就会抛出上述异常，但是在UI线程中就不会，因为应用程序启动时就已经调用了Looper的prepareMainLooper方法，在该方法里面已经调用了prepare(false)方法创建了UI线程的Looper实例，无需我们再次调用Looper.prepare方法。

另外一种是**指定Looper**的构造函数(如果不指定，Looper默认从当前线程的TLS区域获取)，如下：

```java
//Handler.java
public Handler(Looper looper) {
    this(looper, null, false);
}

public Handler(Looper looper, Callback callback) {
    this(looper, callback, false);
}

public Handler(Looper looper, Callback callback, boolean async) {
    //与Looper关联
    mLooper = looper;
    //通过Looper获取MessageQueue，与MessageQueue关联
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

不管哪一种构造函数，可以看到，在Handler的最终构造中都会和**对应线程**的Looper、MessageQueue关联，所以就算Hander在子线程创建，我们也可以通过: **Handler = new Handler(Looper.getMainLooper)；**把Handler关联上UI线程的Looper，并通过Looper关联上UI线程的MessageQueue，这样，就能把Handler运行在UI线程中。

**消息的发送**可以通过handler的**一系列post**方法和**一系列的send**方法，一系列post方法最终通过一系列send方法来实现，一系列send方法最终通过**enqueueMessage**方法来发送消息，如下：

{% asset_img handler3.png handler3 %}

可以发现Handler所有发送消息的方法，最终都是调用**Handler的enqueueMessag**方法，该方法源码如下：

```java
//Handler.java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```
在该方法中，首先把Message的target字段设置为当前发送消息的Handler, 然后设置Message是否是异步消息，最后把所有逻辑交给**MessageQueue的enqueueMessage**方法，该方法的相应源码如下：
```java
//MessageQueue.java
boolean enqueueMessage(Message msg, long when) {
    //...
    synchronized (this) {
        //正在退出时，回收msg，加入到消息池
        if (mQuitting) {
            //...
            msg.recycle();
            return false;
        }
        msg.markInUse();
        msg.when = when;
        boolean needWake;
        //取mMessages链表头部的消息
        Message p = mMessages;
        //满足以下2种情况之一就把msg插入到链表头部：
        //1、如果p为null，则代表mMessages没有消息
        //2、如果when == 0 或 when < p.when, 则代表msg的触发时间是链表中最早的
        if (p == null || when == 0 || when < p.when) {
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;//如果处于阻塞状态，需要唤醒
        } else {//3、如果p != null且msg并不是最早触发的，就在链表中找一个位置把msg插进去
            //如果处于阻塞状态，并且链表头部是一个同步屏障，并且插入消息是最早的异步消息，需要唤醒
            needWake = mBlocked && p.target == null && msg.isAsynchronous();            
            Message prev;
            //下面是一个链表的插入操作, 将消息按时间顺序插入到mMessages中
            for (;;) {
                prev = p;
                p = p.next；
                if (p == null || when < p.when) {
                    break;
                }  
                //如果在找到插入位置之前，发现了异步消息的存在，不需要唤醒
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p;
            prev.next = msg;
        }   
        
        //nativeWake方法会唤醒当前线程的MessageQueue
        if (needWake) {
            nativeWake(mPtr);
        }     
    }
    return true;
}
```
MessageQueue的enqueueMessage方法主要是一个链表的插入操作，返回true就代表插入成功，返回false就代表插入失败，它主要分为以下3种情况：

**1、插入链表头部**：

mMessages是按照Message触发时间的先后顺序排列的，**越早触发的排得越前**，头部的消息是将要最早触发的消息，当有消息需要加入mMessages时，如果mMessages为空或这个消息是最早触发的，就会直接插入链表头部；

**2、插入链表的中间位置**：

如果消息链表不为空并且插入的消息不是最早触发的，就会从链表头部开始遍历，直到找到消息应该插入的合适位置，以保证所有消息的时间顺序，这一个过程可以理解为**插入排序**；

**3、判断是否调用nativeWake方法**

最后，根据needWake是否为true，来决定是否调用**nativeWake**方法唤醒当前线程的MessageQueue，needWake默认为false，即不需要唤醒，needWake**为true**就代表此时处于以下2种情况：

（1）如果插入消息在链表头部并且mBlocked == true，表示此时**nativePollOnce**方法进入阻塞状态，等待被唤醒返回；

（2）如果插入消息在链表中间，消息链表的头部是一个同步屏障，同时插入的消息是链表中最早的异步消息，需要唤醒，即时处理异步消息。

### 5、消息的分发
在消息循环的运行中，如果loop方法中MessageQueue的**next方法返回了Message**，那么就会执行到这一句：**msg.target.dispatchMessage(msg)
**；Looper会把这条消息交给该Message的target（Handler对象）来处理,  实际上是转了一圈，Handler把消息发送给MessageQueue，Looper又把这个消息给Handler处理，下面来看消息分发逻辑，dispatchMessage()源码如下：

```java
//Handler.java
public void dispatchMessage(Message msg) {
    //1、检查msg的callback是否为空
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        //2、Handler的mCallback是否为空
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        //3、我们平常处理消息的方法，该方法默认为空，Handler子类通过覆写该方法来完成具体的逻辑。
        handleMessage(msg);
    }
}
```
里面代码量很少，分为3步：

1、检查msg.callback是否为空，不为空则执行" handleCallback(msg)", 源码如下:

```java
//Handler.java
private static void handleCallback(Message message) {
    message.callback.run();
}
```
msg.callback其实是一个Runnable对象，当我们通过Handler来post一个Runnable消息时，它就不为空，如下：
```java
//Handler.java
public final boolean post(Runnable r){
    return  sendMessageDelayed(getPostMessage(r), 0);
}
//把Runnable对象包装成Message对象
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```
可以看到，在post(Runnable r)中，会把Runnable包装成Message对象，并把Runnable设置给Message的callback字段，然后发送此消息。

2、如果msg.callback为空，检查mCallback是否为空，mCallback是一个Callback接口，定义如下：

```java
//Handler.java
public interface Callback {
    public boolean handleMessage(Message msg);
} 
```
当我们这样：**Handler handler = new Handler(callback)** 来创建Handler时, mCallback就不为空，它的意义是当我们不想派生Handler的子类重写handleMessage()来处理消息时，就可以通过Callback来实现。

3、如果mCallback为空，最后调用Handler的**handleMessage**方法来处理消息，这就是我们平时熟悉的处理消息的方法。

从1、2、3可以看出，在Handler中，处理消息的回调的优先级为：**Message的Callback > Handler的Callback > Handler的handleMessage方法**。

### 6、消息的回收复用

**1、消息的复用**

前面多次提到了Message，当我们通过Handler的obtainMessage()或Message的obtain()获取一个Message对象时，系统并不是每次都new一个出来，而是先从消息池中（sPool）尝试获取一个Message。Handler的obtainMessage()最终是调用了Message的**obtain()**，Message的obtain方法如下:
```java
//Message.java
public static Message obtain() {
        synchronized (sPoolSync) {
	    //从sPool头部取出一个Message对象返回，并把消息从链表断开（即把sPool指向下一个Message）
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0;//清除in-use flag
                sPoolSize--;//消息池的大小进行减1操作
                return m;
            }
        }
        //消息池中没有Message，直接new一个返回
        return new Message();
```
sPool的数据类型为Message，通过next成员变量，维护一个消息池，消息池的默认大小为50。定义如下:
```java
public final class Message implements Parcelable {
    public static final Object sPoolSync = new Object();//用于在获取Message对象时进行同步锁
    private static int sPoolSize = 0;//池的大小
    private static final int MAX_POOL_SIZE = 50;//池的可用大小
    private static Message sPool;
    Message next;
    //...
}
```
虽然叫消息池，其实是通过链表实现的，每个Message都有一个同类型的next字段，这个next就是指向下一个可用的Message，最后一个可用的Message的next为空，这样所有可用的Message对象就通过next串成一个Message池，sPool指向池中的第一个Message，复用消息其实是**从链表的头部获取一个Message返回**。

**2、消息的回收**

我们发现在obtain方法中新创建Message对象时，并不会直接把它放到池中再返回，那么Message对象是什么时候被放进消息池中的呢？是在**回收**Message时把它放入池中，Message中也有类似Bitmap那样的**recycler**函数，如下：

```java
//Message.java
public void recycle() {
	//判断消息是否正在使用
        if (isInUse()) {
            if (gCheckRecycle) {//Android 5.0以后的版本默认为true,之前的版本默认为false.
                throw new IllegalStateException("This message cannot be recycled because it "
                        + "is still in use.");
            }
            return;
        }
        recycleUnchecked();
    }
    
 //对于不再使用的消息，加入到消息池
 void recycleUnchecked() {
	 //将消息标示位置为IN_USE，并清空消息所有的参数
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        when = 0;
        target = null;
        callback = null;
        data = null;
        synchronized (sPoolSync) {
           //当消息池没有满时，将Message对象加入消息池（即把Message插入链表头部）
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;//消息池的可用大小进行加1操作
            }
        }
    }
```
recycler函数先判断该Message是否还在使用，如果还在使用，就会抛异常，否则就调用recyclerUnchecked函数根据MAX_POOL_SIZE判断是否把该消息回收，回收前还要先清空该消息的各个字段，回收消息就是**把自身插入到链表的表头**。

通过消息的复用回收，减少Message对象不断创建与销毁的过程，提升了效率。

### 7、小结

1、创建Looper时，只能通过Looper的**prepare**方法创建，在创建Looper时会在内部创建一个MessageQueue，并把Looper保存在线程的TLS区域中，**一个线程只能对应一个Looper，一个Looper只能对应一个MessageQueue**；

2、在创建MessageQueue时，MessageQueue与NativeMessageQueue建立连接，NativeMessageQueue存储地址存于MessageQueue的**mPtr**字段中，java层和native通过mPtr字段进行通信（native端通过Linux的**epoll机制**建立起消息机制）;

3、由于ThreadLocal的作用，Looper属于某个线程，而MessageQueue存储在Looper中，所以**MessageQueue则通过Looper与特定的线程关联上**，而Handler在构造中又与Looper和MessageQueue相关联，当我们通过Handler发送消息时，消息就会被**插入**到Handler关联的MessageQueue中，而Looper会不断的轮询消息，从MessageQueue中取出消息给相应的Handler处理，所以最终通过Handler发送的消息就会被执行到Looper所在线程上，这就是Handler**线程切换**的原理，**无论发送消息的Handler对象处于什么线程，最终处理消息的都是Looper所在线程**；

4、Looper从MessageQueue中取出消息后，会交给消息的target(Handler)处理，在Handler中，处理消息的回调的优先级为：**Message的Callback > Handler的Callback > Handler的handleMessage方法**；

5、因为应用程序启动时在ActivityThread.main方法中的Looper.prepareMainLooper()中已经调用了Looper.prepare(false),所以在主线程中创建Handler无需我们手动调用Looper.prepare()，而在子线程中，如果我们不传递UI线程所属的Looper去创建Handler，那么就需要调用Looper.prepare()后再创建Handle来传递消息，因为Handler要和某个线程中的MessageQueue和Looper关联，**只有调用Looper.prepare方法，Looper和MessageQueue才属于某个线程**；

6、消息池是一个单链表，**复用Message**时，从头出取出，如果取不到，则新建返回，**回收Message**时，也从头插入。

## 结语
本文从Android应用UI线程消息循环的创建，消息循环的启动，Handler与Looper、MessageQueue的关联，消息的发送与分发，还有消息的复用这几个角度来讲解了Message，Handler，MessageQueue，Looper之间是如何配合工作，在你了解java层的消息机制是如何运作后，希望大家去了解一下[native的消息机制](http://rain9155.coding.me/2019/02/21/Android消息机制native层/)，例如要想知道为什么loop方法是**死循环**但却不会消耗性能，这些只有native层的消息机制才能给你答案.

除此之外，我们还知道了MessageQueue的IdleHandler的作用，它会在线程空闲时工作，还有异步消息的处理，它的优先级高于同步消息，会被优先处理，还有它们的应用场景，一般我们是在子线程切换到UI线程时使用Handler机制，但其实我们也可以在子线程使用Handler机制，可以参考Android中的**HandlerThread**，它的底层就是Handler+Thread.

以上就是本文的全部内容，希望大家有所收获。

参考资料：

[Android消息机制1-Handler](http://gityuan.com/2015/12/26/handler-message-framework/)



