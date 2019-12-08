---
title: Android消息机制(java层)
date: 2019-02-21 13:33:11
tags: 
- handler
- 源码
categories: 消息机制
---

## 前言
Android的消息机制用于同进程的线程间通信，它是由MessageQueue，Message，Looper，Handler共同组成。Android中有大量的交互都是通过消息机制，比如Android的四大组件的启动过程的交互就离不开消息机制，所以Android在某种意义上也可以说成是一个以消息驱动的系统。

<!--more-->


	本文源码基于Android8.0，源码相关位置:
	frameworks/base/core/java/android/os/MessageQueue.java
	frameworks/base/core/java/android/os/Handler.java
	frameworks/base/core/java/android/os/Looper.java
	frameworks/base/core/java/android/os/Message.java

## 消息机制概述
Android应用的每个事件都会转化为一个系统消息即Message，消息中包含了事件的相关信息和消息的处理人即Handler，消息要通过Handler发送，最终被投递到一个消息队列中即MessageQueue，它维护了一个待处理的消息列表，然后通过Looper开启了一个消息循环不断地从这个队列中取出消息，当从消息队列取出一个消息后，Looper根据消息的处理人（target）将此消息分发给相应的Handle处理。它们的工作原理就像工厂的生产线，Looper是发动机，MessageQueue是传送带，Handler是工人，Message则是待处理的产品。整个过程如下图所示。

{% asset_img handler1.png %}

## 消息机制架构图
{% asset_img handler2.png %}
* Looper  --- 是每个线程的MessageQueue管家，里面有一个MessageQueue消息队列，负责把消息从MessageQueue中取出并把消息传递到Handler中去，每个线程只有一个Looper。
*  MessageQueue  ---  消息队列，有一组待处理的Message，主要用于存放所有通过Handler发送的消息，每个线程只有一个MessageQueue。
*  Message  ---  是线程之间传递的消息，里面有一个用于处理消息的Handler。
*  Handler  ---  主要用于发送和处理消息，里面有Looper和MessageQueue。

## 深入了解Android的消息机制
### 1、 Looper的创建，Handler与Looper的关联
我们知道Android应用程序的入口实际上是ActivityThread.main方法，在该方法中首先会创建Application和默认启动的Activity，并将它们关联在一起，而该应用的UI线程的消息循环也是在这个方法中创建，具体源码如下：
```java
 public static void main(String[] args) {
 	 //...
 	 //1、创建UI线程的消息循环Looper
 	 Looper.prepareMainLooper();
 	 //...
 	 //2、执行消息循环
 	 Looper.loop();
 }
```
执行ActivityThread.main后，应用程序就启动了，UI的消息循环也在Looper.loop（）中启动，此后Looper会一直从消息队列中取出消息，用户或系统通过Handler不断往消息队列中添加消息，这些消息不断的被取出，处理，回收，使得应用运转起来。Android应用程序的Handler在ActivityThread中被创建，如下：
```java
 final H mH = new H();
 //H定义如下，里面定义了大量的字段，跟Activity的启动，Application的绑定等有关
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
```
在开发中，我们在子线程中执行完操作后通常需要更新UI，但我们都知道不能在子线程中更新UI，此时我们就要通过Handler将一个消息post到UI线程中，然后再在Handler中的handleMessage（）中进行处理，在这里要注意的是如果我们不传递UI线程所属的Looper去创建Handler，那么该Handler必须在主线程中创建，如下：
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
如果不这样做将会抛出一个异常"Can't create handler inside threadxx that has not called Looper.prepare()", 为什么会这样呢？看异常描述是我们没有调用Looper.prepare()，那么问题又来了，**为什么在主线程中创建Handler时我们没有手动调用Looper.prepare()不会抛异常，而在子线程创建Handler时没有调用Looper.prepare()就会抛异常？**
这里我们先从Handler的默认构造函数看起，源码如下：
```java
 public Handler() {
        this(null, false);
}
public Handler(Callback callback, boolean async) {
	//...
	//与Looper关联
	 mLooper = Looper.myLooper();
      if (mLooper == null) {
          throw new RuntimeException(
              "Can't create handler inside thread " + Thread.currentThread()
                      + " that has not called Looper.prepare()");
       }
      //通过Looper获取MessageQueue
      mQueue = mLooper.mQueue;
      mCallback = callback;
      mAsynchronous = async;
}
```
可以看到，Handler构造中会和Looper关联，如果通过Looper.myLooper()获取不到Looper，就会抛出上述所讲的异常，然后再通过Looper获取它持有的MessageQueue。我们继续点进myLooper()中看它如何工作，如下：
```java
public static @Nullable Looper myLooper() {
	//获取当前线程TLS区域的Looper
	return sThreadLocal.get();
}
```
可以看到myLooper方法是通过ThreadLocal获取的，这里简单介绍一下ThreadLocal（关于ThreadLocal更多信息可以查看[ThreadLocal原理解析](https://rain9155.github.io/2019/02/21/ThreadLocal解析)）:

	ThreadLocal： 线程本地存储区（Thread Local Storage，简称为TLS），每 个线程都有自己的私有的本地存储区域，不同线程之间彼此不能访问对方的TLS区域。
	它的常用操作方法有：
	ThreadLocal.set(T value)：将value存储到当前线程的TLS区域。
	ThreadLocal.get()：获取当前线程TLS区域的数据

ThreadLocal的get()和set()方法操作的类型都是泛型，接着回到前面提到的sThreadLocal变量，其在Looper中定义如下：
```java
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```
可见sThreadLocal的get()和set()操作的类型都是Looper类型,也就是说每个线程只能有一个Looper，不同线程的Looper是不相同的。那么Looper是什么时候被set到ThreadLocal中的？其实答案就在上面写到的ActivityThread.main方法中的Looper.perpareMainLooper中，相关源码如下：
```java
 public static void prepareMainLooper() {
    //设置不允许退出的Looper
    prepare(false);
    synchronized (Looper.class) {
    	    //将当前的Looper保存为主Looper，每个线程只允许执行一次。
	    if (sMainLooper != null) {
	    		throw new IllegalStateException("The main Looper has already 			been 		prepared.");
	    }
	sMainLooper = myLooper();
    }
 }
 
//quitAllowed表示是否允许Looper运行时退出
private static void prepare(boolean quitAllowed){
	//Looper.prepare()只能执行一次
	if (sThreadLocal.get() != null) {
		throw new RuntimeException("Only one Looper may be created per thread"); 
	 }
	//把Looper保存到TLS中
	sThreadLocal.set(new Looper(quitAllowed));
}

//Looper的构造函数
 private Looper(boolean quitAllowed) {
 	//可以看到MessageQueue是在Looper中创建的
     mQueue = new MessageQueue(quitAllowed);
     //...
 }
```
我们再回到Handler中来，Looper属于某个线程，MessageQueue存储在Looper中，MessageQueue则通过Looper与特定的线程关联上，而Handler在构造中又与Looper和MessageQueue关联，所以最终通过Handler发送的消息就会被执行到这个线程上。同时因为应用程序启动时在ActivityThread.main方法中的Looper.prepareMainLooper()中已经调用了Looper.prepare(),所以在主线程中创建Handler无需我们手动调用Looper.prepare()，而在子线程中，如果我们不传递UI线程所属的Looper去创建Handler，那么就需要调用Looper.prepare()后再创建Handle来传递消息（**总的来说是因为Handler要和某个线程中的MessageQueue和Looper关联，只有调用Looper.prepare()，Looper和MessageQueue才属于某个线程**）。
### 2、消息循环的运作
在创建Looper后，通过Looper.loop()就启动了消息循环，这个函数会不断的从消息队列中取出消息、处理消息。我们点进此方法看一下它的源码：
```java
public static void loop() {   
     final Looper me = myLooper();
     if (me == null) {
	throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
     }
     final MessageQueue queue = me.mQueue;
     //...
     for (;;) {
     	//从MessageQueue中取出消息，没有消息时会阻塞等待
		 Message msg = queue.next(); 
		 //next()返回了null，表示MessageQueue正在退出，即调用了Looper的quit或quitSafely方法
		 if (msg == null) {
                	return;
          	 }
          	 //...
          	 //分发消息
		 msg.target.dispatchMessage(msg);
		 //...
}
```
loop()中是一个死循环，loop()会调用MessageQueue的next()来获取最新的消息，当没有消息时，next()会一直阻塞在那里，这也导致loop()阻塞，唯一跳出循环的条件是next()返回null，这时代表Looper的quit()或quitSafely()被调用，从而调用MessageQueue的quit()来通知消息队列退出。MessageQueue的next()是最关键的函数，我们来看看next函数的关键代码：
```java
 Message next() {
 	//mPtr是在构造中被赋值，是指向native层的MessageQueue
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }
        int nextPollTimeoutMillis = 0;
        for(;;){
            //1、处理native层事件，是一个阻塞操作，当等待nextPollTimeoutMillis时长，或者消息队列被唤醒，都会返回
            nativePollOnce(ptr, nextPollTimeoutMillis);
            synchronized (this) {
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                //java层的消息队列
                Message msg = mMessages;
                //...
                if (msg != null) {//有消息
	            //2、消息还没到触发时间
                    if (now < msg.when) {
                      	//设置下一次轮询的超时时长（等待时长）
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {   
                        mBlocked = false;
                        //3、获取一条消息并返回 
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        //设置消息的使用状态，即flags |= FLAG_IN_US
                        msg.markInUse();
                        return msg;
                    }
                } else {//没有消息
                    nextPollTimeoutMillis = -1;
                }
            }
            //...
 }
 
```
next函数看起来有点多代码，但这里只分析核心部分，其实MessageQueue是消息机制的Java层和C++层的连接纽带，大部分核心方法都交给native层来处理，这里的mPtr是指向native层的NativeMessageQueue对象，在MessageQueue构造中被赋值, 如下：
```java
 MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    //通过native方法初始化消息队列，其中mPtr是供native代码使用
    mPtr = nativeInit();
 }
```
next方法也是一个死循环，最主要的方法是nativePollOnce(),它是一个阻塞操作，其中nextPollTimeoutMillis代表下一个消息到来前，还需要等待的时长，当nextPollTimeoutMillis = -1时，表示消息队列中无消息，会一直等待下去，则next方法将会一直阻塞在这里，当有新消息到来时，next方法会返回这条消息并将其从单链表中删除。可以发现虽然MessageQueue叫消息队列，但它却不是用队列实现的，而是用链表实现的。

	MessageQueue是消息机制的核心类，它里面有大量的native方法，Android有俩套消息机制（java层和native层，实现不一样），但本文只讲解java层的消息机制，不会涉及到native层。

关于native层的查看[Android消息机制（native层）](https://rain9155.github.io/2019/02/21/Android%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6native%E5%B1%82/)。

### 3、消息的分发
如果loop方法中next()返回了null，那么就会执行到这一句" msg.target.dispatchMessage(msg)
",Looper会把这条消息交给Message的target（Handler对象）来处理, 实际上是转了一圈，Handler把消息发送给消息队列，Looper又把这个消息给Handler处理。**注意：在本文的情景下，loop方法这个时候是执行在主线程的，因为Looper是在主线程中创建的，所以到了这里，消息的处理就切换到主线程了，这就是Handler线程切换的原理，Handler发送的消息的线程不处理消息，只有在Looper.loop()中将消息取出来后再进行处理，所以在Handler机制中，无论发送消息的Handler对象处于什么线程，最终处理都是运行在Looper.loop()所在线程。**
下面来看消息分发逻辑，dispatchMessage()源码如下：

```java
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
里面代码量很少，首先第一步，检查msg.callback是否为空，不为空则执行" handleCallback(msg)", 源码如下:
```java
 private static void handleCallback(Message message) {
        message.callback.run();
    }
```
msg.callback其实是一个Runnable对象，当我们通过Handler来post一个Runnable消息时，它就不为空，如下：
```java
public final boolean post(Runnable r)
    {
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

如果msg.callback为空，就到第二步，检查mCallback是否为空，mCallback是一个Callback接口，定义如下：
```java
 public interface Callback {
        public boolean handleMessage(Message msg);
    } 
```
当我们这样来创建Handler: Handler handler = new Handler(callback)时, mCallback就不为空，它的意义是当我们不想派生Handler的子类重写handleMessage()来处理消息时，就可以通过Callback来实现。

如果mCallback为空，就到第三步，调用Handler的handleMessage方法来处理消息。
### 4、消息的发送
前面讲到消息的接收处理最终是在Handler中进行，而消息的发送也是通过Handler进行，消息的发送可以通过handler的一系列post方法和一系列的send方法，一系列post方法最终通过一系列send方法来实现，如图：
{% asset_img handler3.png %}
从上图，可以发现所有的发消息方式，最终都是调用MessageQueue.enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis)，该方法源码如下：
```java
 private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
在该方法中，首先把Message的target字段设置为当前发送消息的Handler,然后设置Message是否是异步消息，最后把所有逻辑交给MessageQueue的enqueueMessage(Message msg, long when)方法，该方法的相应源码如下：
```java
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
            //1、取队头消息
            Message p = mMessages;
            //2、如果p为null，则代表MessageQueue没有消息
            //如果when == 0 或 when < p.when, 则代表msg的触发时间是队列中最早的
            //满足上述条件就把msg插入到队列头部
            if (p == null || when == 0 || when < p.when) {
                msg.next = p;
                mMessages = msg;
                //...
            } else {
                //3、如果p != null且msg并不是最早触发的
                //...              
                Message prev;
                //下面是一个链表的插入操作,将消息按时间顺序插入到MessageQueue 
                for (;;) {
                    prev = p;
                    p = p.next；
                    if (p == null || when < p.when) {
                        break;
                    }  
                    //...
                }
                msg.next = p;
                prev.next = msg;
            }   
            //消息没有退出，此时mPtr != 0，native层处理逻辑
            if (needWake) {
                nativeWake(mPtr);
            }     
        }
       return true;
 }
```
这个方法主要操作就是一个链表的插入操作，MessageQueue是按照Message触发时间的先后顺序排列的，队头的消息是将要最早触发的消息。当有消息需要加入消息队列时，如果消息队列为空或这个消息是最早触发的，就会直接插入队头，否则会从队列头开始遍历，直到找到消息应该插入的合适位置，以保证所有消息的时间顺序。
### 5、消息的复用
前面多次提到了Message，当我们通过Handler的obtainMessage()或Message的obtain()获取一个Message对象时，系统并不是每次都new一个出来，而是先从消息池中（sPool）尝试获取一个Message。Handler的obtainMessage()最终是调用了Message的obtain()。Message#obtain()的源码如下:
```java
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
虽然叫消息池，其实是通过链表实现的，每个Message都有一个同类型的next字段，这个next就是指向下一个可用的Message，最后一个可用的Message的next为空，这样所有可用的Message对象就通过next串成一个Message池，sPool指向池中的第一个Message。
那么Message对象是什么时候被放进消息池中的呢？其实在obtain方法中创建Message对象时，并不会直接把它放到池中，而是在回收Message时把它放入池中，Message中也有类似Bitmap那样的recycler函数，如下：
```java
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
recycler函数先判断该Message是否还在使用，如果还在使用，就会抛异常，否则就调用recyclerUnchecked函数根据MAX_POOL_SIZE判断是否把该消息回收，回收前还要先清空该消息的各个字段，回收消息就是把自身插入到链表表头。

通过消息的复用，减少Message对象不断创建与销毁的过程，提升了效率。
## 结语
能看到这里的，证明你已经了解了java层的消息机制是如何运作的了，本文从Android应用UI线程消息循环的创建出发，通过讲解Looper与Handler的关联，如何启动消息循环，消息的发送与分发，还有消息的复用来讲解了Message，Handler，MessageQueue，Looper之间是如何配合工作。掌握了这些，在以后开发中又能更加随心所欲了。

参考资料：

《Android开发艺术探索》

《Android源码设计与分析》

[Android消息机制1-Handler(java层)](http://gityuan.com/2015/12/26/handler-message-framework/)



