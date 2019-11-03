---
title: Android消息机制(native层)
date: 2019-02-21 13:51:51
tags: 
- handler
- 源码
- native
categories: 消息机制
---

## 前言

* 上一篇文章：[Android消息机制java层](https://rain9155.github.io/2019/02/21/Android%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6java%E5%B1%82/)

在上一篇文章中讲到MessaeQueue是Android消息机制的Java层和native层的连接纽带，Android的java层和native层通过JNI调用打通，MessageQueue中有多个native方法，java层和native各有一套消息机制，实现不一样，本文讲解native层的Android消息机制。
<!--more-->

	本文基于Android8.0，相关源码文件如下:
	frameworks/base/core/jni/android_os_MessageQueue.cpp
	frameworks/base/core/jni/android_os_MessageQueue.h
	system/core/libutils/Looper.cpp 
	system/core/libutils/Looper.h

## native层消息机制架构图
{% asset_img Android消息机制(native层).jpg %}
* MessageQueue  --- 里面有一个Looper。
* NativeMessageQueue  ---  MessageQueue的继承类，见名知意，native层的消息队列，只是一个代理类，其大部分方法操作都转交给Looper的方法。
* Looper  ---  native层的Looper，其功能相当于java层的Handler，它可以取出消息，发送消息，处理消息。
* MessageHandler  ---  消息处理类，Looper把处理消息逻辑转交给此类。
* WeakMessageHanlder  ---  MessageHandler的继承类，也是处理消息类，但最终还会把消息处理逻辑转交给MessageHandler。

## java层的MessageQueue
要讲解native层的消息机制，我们可以从java层消息机制调用到的MessageQueue的native方法讲起，MessageQueue中所有的native方法如下:
```java  
public final class MessageQueue {
    private native static long nativeInit();
    private native static void nativeDestroy(long ptr);
    private native void nativePollOnce(long ptr, int timeoutMillis); 
    private native static void nativeWake(long ptr);
    private native static boolean nativeIsPolling(long ptr);
    private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);
    //...
}
```
我们主要讲解三个：nativeInit()， nativePollOnce(long ptr, int timeoutMillis)， nativeWake(long ptr)。
### 1、MessageQueue#nativeInit()
在java层中，MessageQueue是在Looper中创建的，在MessageQueue的构造中:
```java
//MessageQueue.java
 MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
 }
```
在java层中，mPtr保存了nativeInit()返回的值，nativeInit方法的实现在android_os_MessageQueue.cpp文件中的android_os_MessageQueue_nativeInit方法中，该方法源码如下:
```c
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {   
    //创建native消息队列NativeMessageQueue
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    //...
    //增加引用计数
    nativeMessageQueue->incStrong(env);
    //使用C++强制类型转换符reinterpret_cast把NativeMessageQueue指针强转成long类型并返回到java层
    return reinterpret_cast<jlong>(nativeMessageQueue);
}
```
可以看到在android_os_MessageQueue_nativeInit方法中会创建一个NativeMessageQueue对象，并增加其引用计数，并将NativeMessageQueue指针mPtr保存在Java层的MessageQueue中。现在我们来看NativeMessageQueue的构造函数, 如下:
```c
NativeMessageQueue::NativeMessageQueue() :
   	 mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    //获取TLS中的Looper(Looper::getForThread相当于java层的Looper.mLooper中的ThreadLocal.get) 
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
 	//创建native层的Looper
        mLooper = new Looper(false);
        //保存Looper到TLS中(Looper::setForThread相当于java层的ThreadLocal.set)
        Looper::setForThread(mLooper);
    }
}
```
（关于TLS更多信息可以查看[ThreadLocal原理解析](https://rain9155.github.io/2019/02/21/ThreadLocal解析)），在NativeMessageQueue的构造中会先调用Looper的getForThread方法从当前线程获取Looper对象，如果为空，就会创建一个Looper并调用Looper的setForThread方法设置给当前线程。也就是说Looper和MessageQueue在java层和native层都有，但它们的功能并不是一一对应，此处native层的Looper与Java层的Looper没有任何的关系，只是在native层重实现了一套类似功能的逻辑。我们来看看native层在创建Looper时做了什么，Looper的构造函数如下:
```c
Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mPolling(false), mEpollFd(-1), mEpollRebuildRequired(false),
        mNextRequestSeq(0), mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
    //构造唤醒事件的fd（文件描述符）
    mWakeEventFd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
    //...
    //重建epoll事件
    rebuildEpollLocked();
}
```
这里我们忽略一大堆字段赋值，只关注一个函数:  rebuildEpollLocked(), 该函数核心源码如下:
```c
void Looper::rebuildEpollLocked() {
    //1、关闭旧的管道
    if (mEpollFd >= 0) {
       close(mEpollFd);
    }
    
    //2、创建新的epoll实例（文件描述符），并注册wake管道
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);
    
    struct epoll_event eventItem;
    memset(& eventItem, 0, sizeof(epoll_event)); //置空eventItem
    //3、设置事件类型和文件描述符
    eventItem.events = EPOLLIN;//可读事件
    eventItem.data.fd = mWakeEventFd;//唤醒事件的fd（文件描述符）
    
    //4、将唤醒事件(mWakeEventFd)添加到epoll实例(mEpollFd)，并监听事件
    int result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeEventFd, & eventItem);   
    //...
    }
}
```
Looper的构造函数中涉及到Linux的epoll机制，关于更多资料可以自行查找资料(有Linux基础的可以阅读[源码解读epoll内核机制](http://gityuan.com/2019/01/06/linux-epoll/) )，这里简单介绍一下:

	epoll机制是Linux最高效的I/O复用机制, 使用一个文件描述符管理多个描述符。
	epoll操作过程有3个方法，分别是:
	1、int epoll_create(int size)；   用于创建一个epoll的文件描述符，size是指监听的描述符个数。
	2、int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)； 用于对需要监听的文件描述符(fd)执行op操作，比如将fd加入到epoll文件描述符, 参数:
	epfd：是epoll_create()的返回值
	op：表示op操作，用三个宏来表示，分别代表添加(EPOLL_CTL_ADD)、删除(EPOLL_CTL_DEL)和修改( EPOLL_CTL_MOD)对fd的监听事件
	fd：需要监听的文件描述符
	epoll_event：需要监听的事件
	3、int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout)； 等待事件的上报, 该函数返回需要处理的事件数目，如返回0表示已超时,参数：
	epfd：等待epfd上的io事件，最多返回maxevents个事件；
	events：用来从内核得到事件的集合；
	maxevents：events数量，该maxevents值不能大于创建epoll_create()时的size；
	timeout：超时时间（毫秒，0会立即返回）。

要使用epoll机制，首先通过 epoll_create创建一个epoll专用文件描述符，并创建了一个管道，最后通过epoll_ctl函数来设置监听的事件类型为EPOLLIN（可读事件），在这里Looper对象中的mWakeEventFd(唤醒事件的文件描述符)添加到epoll监控范围内。至此，native层的MessageQueue和Looper就构建完毕，底层通过管道与epoll机制也建立了一套消息机制。

我们跟着MessageQueue#nativeInit()一路走下来，这里小结一下：
* 1、首先java层的Looper对象会在构造函数中创建java层的MessageQueue对象。
* 2、 java层的MessageQueue对象又会调用nativeInit函数初始化native层的NativeMessageQueue，NativeMessageQueue的构造函数又会创建native层的Looper，并且在Looper中通过管道与epoll机制建立一套消息机制。
* 3、native层构建完毕，将NativeMessageQueue对象转换为一个long类型存储到java层的MessageQueue的mPtr中。

在此之后就会在java层中启动消息循环，Looper.loop -> Looper.next -> MessageQueue.next ->
MessageQueue.nativePollOnce，下面我们来看MessageQueue#nativePollOnce()。
### 2、MessageQueue#nativePollOnce()
在java层中每次循环去读消息时，都会调用这个函数，如下:
```java
//MessageQueue.java
 Message next() {
 	//...
 	for(;;){
 		 nativePollOnce(ptr, nextPollTimeoutMillis);
 		 //...
 	}
 	//...
 }
```
nativePollOnce函数的实现在android_os_MessageQueue.cpp文件中的android_os_MessageQueue_nativePollOnce方法中，该方法的源码如下:
```c
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jlong ptr, jint timeoutMillis) {
    //把ptr强转为NativeMessageQueue
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
}
```
ptr是从java层传过来的mPtr的值，mPtr在初始化时保存了NativeMessageQueue的指针，此时首先把传递进来的ptr转换为NativeMessageQueue，然后调用NativeMessageQueue的pollOnce函数，该函数核心源码如下:
```c
void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    //...
    //核心是调用了native层的Looper的pollOnce方法
    mLooper->pollOnce(timeoutMillis);  
    //...
}
```
这段代码主要就是调用了native层的Looper的pollOnce(timeoutMillis)方法，该方法会调用Looper的 pollOnce(timeoutMillis, NULL, NULL, NULL)，相关源码如下:
```c
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
     //...
     //处理内部轮询
     result = pollInner(timeoutMillis);
    }
}
```
该方法核心在于调用了pollInner函数，该函数相关源码如下:
```c
int Looper::pollInner(int timeoutMillis) {    
    //...
    //事件集合，EPOLL_MAX_EVENTS为最大事件数量
    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    //1、等待事件发生或者超时，从管道中读取事件
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);  
    //获取锁
    mLock.lock();
    //...
    Done:;
    //处理Native的Message，调用相应回调方法
    mNextMessageUptime = LLONG_MAX;
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        //2、判断消息的执行时间
        if (messageEnvelope.uptime <= now) {
            {
        	//3、获取native层的Handler
                sp<MessageHandler> handler = messageEnvelope.handler;
                //4、获取native层的消息
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                //释放锁
                mLock.unlock();
                //5、处理消息事件
                handler->handleMessage(message);
            }
            //请求锁
            mLock.lock();
            mSendingMessage = false;
            // 发生回调
            result = POLL_CALLBACK;
        } else {
           //消息还没到执行时间
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }
    //释放锁
    mLock.unlock();
    //...
}
```
pollInner函数很长，省略了一大堆代码，这里讲解一些核心的点，在此之前先讲解一下MessageEnvelope，正如其名字，信封，其结构体定义在Looper类中，如下:
```c
class Looper : public RefBase { 
    struct MessageEnvelope {
        MessageEnvelope() : uptime(0) { }

        MessageEnvelope(nsecs_t u, const sp<MessageHandler> h,
                const Message& m) : uptime(u), handler(h), message(m) {
        }

        nsecs_t uptime;
        sp<MessageHandler> handler;
        Message message;
    };   
    //...
}
```
MessageEnvelope里面记录着收信人（handler，MessageHandler类型，是一个消息处理类），发信时间(uptime)，信件内容(message，Message类型)。Message结构体，消息处理类，Looper类都定义在Looper.h/ Looper.cpp文件中。pollInner函数的流程如下：
* 1、先调用epoll_wait()，这是阻塞方法，从管道取到事件或等待超时都会返回。
* 2、进入Done标记位的代码段, 处理Native的Message，调用Native 的Handler来处理该Message。

pollInner实际上就是从管道中读取事件，并且处理这些事件。在native中事件存储在管道中，而在java层中事件存储在消息链表中，但这俩个层次的事件都通过java层的Looper消息循环进行不断的获取，处理等操作。

我们更着MessageQueue#nativePollOnce()一路走下来，小结一下：
* 1、当在java层通过Looper启动消息循环后，就会走到MessageQueue的nativePollOnce方法，在该方法native实现中，会把保存在java层的mPtr再转换为NativeMessageQueue。
* 2、然后调用NativeMessageQueue的pollOnce方法，该方法中最终会调用native层的Looper的pollInner方法，Looper的pollInner方法是阻塞方法，等从管道取到事件或超时就会返回，并通过native层的Handler处理native层的Message消息。
* 3、处理完native层消息后，又会返回到java层处理java层的消息。

可以看到，native层的NativeMessageQueue实际上并没有做什么实际工作，只是把操作转发给native层的Looper，而native层的Looper则扮演了java层的Handle角色，它可以取出，发送，处理消息。
### 3、MessageQueue#nativeWake()
nativeWake()用于唤醒功能，我们在Java层通过Hanlder发送消息时，实际是把消息添加到消息队列，会调用到MessageQueue的enqueueMessage方法, 该方法中会调用到nativeWake方法，如下：
```java
//MessageQueue.java
  boolean enqueueMessage(Message msg, long when) {
  	//...
  	synchronized (this) {
  		//...
  		 if (needWake) {
                nativeWake(mPtr);
            }
  	}
  }
```

 或者把消息从消息队列中全部移除，调用MessageQueue的quit方法，在有需要时都会调用nativeWake方法。MessageQueue的nativeWake方法的实现在android_os_MessageQueue.cpp文件中的android_os_MessageQueue_nativeWake方法中，该方法的源码如下:
 ```c
 static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->wake();
}
 ```
 可以看到，步骤和上面讲的差不多，首先把传递进来的ptr转换为NativeMessageQueue，然后调用NativeMessageQueue的wake函数，该函数源码如下:
 ```c
 void NativeMessageQueue::wake() {
    mLooper->wake();
}
 ```
前面说过在native层中NativeMessageQueue只是一个代理Looper的角色，该方法把操作转发给native层的Looper，Looper的wake方法核心源码如下:
```c
void Looper::wake() {
    uint64_t inc = 1;
    //通过write函数向管道mWakeEventFd写入字符inc
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
   //...
```
其中TEMP_FAILURE_RETRY 是一个宏定义， 当执行write失败后，会不断重复执行，直到执行成功为止。Looper的wake方法就是向管道mWakeEventfd写入字符。

我们跟着MessageQueue#nativeWake一路走下来，小结一下：
* 1、在java层插入消息到消息队列后，就会根据需要判断是否要调用nativeWake方法，如果调用，就转到2。
* 2、在nativeWake方法native实现中，会把保存在java层的mPtr再转换为NativeMessageQueue，然后调用NativeMessageQueue的wake方法，最终调用Looper的wake方法。
* 3、前面讲到Looper::pollInner方法是一个阻塞操作，当管道中没有事件时当前线程就会进入等待，当管道有事件就会立即返回，从管道中读取事件并处理。而Looper::wake方法就是一个唤醒操作，它就是通过前面创建的唤醒事件文件描述符mWakeEventFd来往管道中写入内容，这时另外等待管道事件的线程就会被唤醒。

## 总结
Java层和Native层的MessageQueue通过JNI建立关联，从而使得MessageQueue成为Java层和Native层的枢纽，既能处理上层消息，也能处理native层消息。Handler/Looper/Message这三大类Java层与Native层并没有任何的真正关联，只是分别在Java层和Native层的handler消息模型中具有相似的功能。都是彼此独立的，各自实现相应的逻辑。另外，消息处理流程是先处理Native Message，最后处理Java Message。

参考资料：

《Android源码设计与分析》

[Android消息机制2-Handler(Native层)](http://gityuan.com/2015/12/27/handler-message-native/)

