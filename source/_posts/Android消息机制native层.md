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

MessageQueue中有多个native方法，MessaeQueue是Android消息机制的Java层和native层的连接纽带，Android的java层和native层通过**JNI调用**打通，java层和native各有一套消息机制，实现不一样，本文讲解native层的Android消息机制，了解了native层的消息机制，你就能明白**为什么java层的loop方法是死循环但却不会消耗性能**这个问题。

## native层消息机制架构图
{% asset_img Android消息机制(native层).jpg %}
* **MessageQueue**  --- 里面有一个Looper，和java层的MessageQueue同名;
* **NativeMessageQueue**  ---  MessageQueue的继承类，native层的消息队列，**只是一个代理类**，其大部分方法操作都转交给Looper的方法;
* **Looper**  ---  native层的Looper，**其功能相当于java层的Handler**，它可以取出消息，发送消息，处理消息；
* **MessageHandler**  ---  native层的消息处理类，Looper把处理消息逻辑转交给此类;
* **WeakMessageHanlder**  ---  MessageHandler的继承类，也是消息处理类，但最终还是会把消息处理逻辑转交给MessageHandler。

## java层的MessageQueue
要讲解native层的消息机制，我们需要从java层消息机制调用到的MessageQueue的**native方法**讲起，MessageQueue中所有的native方法如下:
```java  
//MessageQueue.java
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
我们主要讲解三个：nativeInit()、 nativePollOnce(long ptr, int timeoutMillis)和 nativeWake(long ptr)，**nativeInit**方法在java层的MessageQueue**构造**的时候调用到，**nativePollOnce**方法在java层的MessageQueue的**next**方法调用到，**nativeWake**方法在java层的MessageQueue的**enqueueuMessage**方法调用到。

### 1、nativeInit()

java层中，在ActivityThread的main方法创建UI线程的消息循环，**Looper.prepareMainLooper -> Looper.prepare -> new Looper -> new MessageQueue**，MessageQueue是在Looper的构造中创建的，在MessageQueue的构造中:

```java
//MessageQueue.java
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    mPtr = nativeInit();
}
```
在java层中，mPtr保存了nativeInit()返回的值，nativeInit方法的实现在android_os_MessageQueue.cpp文件中的**android_os_MessageQueue_nativeInit**方法中，该方法源码如下:
```c++
//frameworks/base/core/jni/android_os_MessageQueue.cpp
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
可以看到在android_os_MessageQueue_nativeInit方法中会创建一个NativeMessageQueue对象，并增加其引用计数，并将NativeMessageQueue指针mPtr保存在Java层的MessageQueue中，现在我们来看**NativeMessageQueue的构造函数**, 如下:
```c++
//frameworks/base/core/jni/android_os_MessageQueue.cpp
NativeMessageQueue::NativeMessageQueue() 
    : mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    //获取TLS中的Looper(Looper::getForThread相当于java层的Looper.mLooper中的ThreadLocal.get方法) 
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
        //创建native层的Looper
        mLooper = new Looper(false);
        //保存Looper到TLS中(Looper::setForThread相当于java层的ThreadLocal.set方法)
        Looper::setForThread(mLooper);
    }
}
```
在NativeMessageQueue的构造中会先调用Looper的getForThread方法从当前线程获取Looper对象，如果为空，就会创建一个Looper并调用Looper的setForThread方法设置给当前线程。

> 关于TLS更多信息可以查看[ThreadLocal原理解析](https://rain9155.github.io/2019/02/21/ThreadLocal解析)

也就是说Looper和MessageQueue在java层和native层都有，但它们的功能并不是一一对应，此处native层的Looper与Java层的Looper没有任何的关系，只是在native层重实现了一套类似功能的逻辑，我们来看看native层在创建Looper时做了什么，**Looper的构造函数**如下:

```c++
//system/core/libutils/Looper.cpp
Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mPolling(false), mEpollFd(-1), mEpollRebuildRequired(false),
        mNextRequestSeq(0), mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
    //1、构造唤醒事件的fd（文件描述符）
    mWakeEventFd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
    //...
    //2、重建epoll事件
    rebuildEpollLocked();
}
```
这里我们忽略一大堆字段赋值，只关注字段**mWakeEventFd**和函数:**rebuildEpollLocked()**，mWakeEventFd就是用于**唤醒线程的文件描述符**，而rebuildEpollLocked方法就是用来重建epoll事件，建立起epoll机制，通过epoll机制监听各种文件描述符.

> 文件描述符是什么？它就是一个int值，又叫做句柄，在Linux中，打开或新建一个文件，它会返回一个文件描述符，读写文件需要使用文件描述符来指定待读写的文件，所以文件描述符就是指代被打开的文件，所有对这个文件的IO操作都要通过文件描述符
>
> 但其实文件描述符也不仅仅是指代文件，它还有更多的含义，可以看后文的epoll机制解释。

**rebuildEpollLocked**方法的核心源码如下:

```c
//system/core/libutils/Looper.cpp
void Looper::rebuildEpollLocked() {
    //1、关闭旧的管道
    if (mEpollFd >= 0) {
        close(mEpollFd);
    }

    //2、创建一个新的epoll文件描述符，并注册wake管道
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);//EPOLL_SIZE_HINT为8

    struct epoll_event eventItem;
    memset(& eventItem, 0, sizeof(epoll_event)); //置空eventItem
    //3、设置监听事件类型和需要监听的文件描述符
    eventItem.events = EPOLLIN;//监听可读事件（EPOLLIN）
    eventItem.data.fd = mWakeEventFd;//设置唤醒事件的fd（mWakeEventFd）

    //4、将唤醒事件fd(mWakeEventFd)添加到epoll文件描述符(mEpollFd)，并监听唤醒事件fd(mWakeEventFd)
    int result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeEventFd, & eventItem);   
    
    //5、将各种事件，如键盘、鼠标等事件的fd添加到epoll文件描述符(mEpollFd)，进行监听
    for (size_t i = 0; i < mRequests.size(); i++) {
        const Request& request = mRequests.valueAt(i);
        struct epoll_event eventItem;
        request.initEventItem(&eventItem);

        int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, request.fd, & eventItem);
        if (epollResult < 0) {
            ALOGE("Error adding epoll events for fd %d while rebuilding epoll set: %s",
                  request.fd, strerror(errno));
        }
    }
}
}
```
Looper的构造函数中涉及到Linux的epoll机制，epoll机制是Linux**最高效的I/O复用机制, 使用一个文件描述符管理多个描述符**，这里简单介绍一下它的使用方法:


> epoll操作过程有3个方法，分别是:
>
> **1、int epoll_create(int size)**：用于创建一个epoll的文件描述符，创建的文件描述符可监听size个文件描述符;
> **参数介绍**：
> **size**：size是指监听的描述符个数
>
> **2、int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)**： 用于对需要监听的文件描述符fd执行op操作，比如将fd添加到epoll文件描述符epfd;
> **参数介绍**:
> **epfd**：是epoll_create()的返回值
> **op**：表示op操作，用三个宏来表示，分别为EPOLL_CTL_ADD(添加)、EPOLL_CTL_DEL(删除)和EPOLL_CTL_MOD(修改)
> **fd**：需要监听的文件描述符
> **epoll_event**：需要监听的事件，有4种类型的事件，分别为EPOLLIN(文件描述符可读)、EPOLLOUT(文件描述符可写), EPOLLERR(文件描述符错误)和EPOLLHUP(文件描述符断)
>
> **3、int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout)**： 等待事件的上报, 该函数返回需要处理的事件数目，如返回0表示已超时;
> **参数介绍**：
> **epfd**：等待epfd上的io事件，最多返回maxevents个事件
> **events**：用来从内核得到事件的集合
> **maxevents**：events数量，该maxevents值不能大于创建epoll_create()时的size
> **timeout**：超时时间（毫秒，0会立即返回）
>
> 关于更多资料可以自行查找资料，有Linux基础的可以阅读[源码解读epoll内核机制](http://gityuan.com/2019/01/06/linux-epoll/) 

要了解epoll机制，首先要知道，在Linux中，**文件、socket、管道(pipe)**等可以进行IO操作的对象都可以称之为流，既然是IO流，那肯定会有两端：read端和write端，我们可以创建两个文件描述符wiretFd和readFd，对应read端和write端，当流中没有数据时，读线程就会阻塞(休眠)等待，当写线程通过wiretFd往流的wiret端写入数据后，readFd对应的read端就会感应到，唤醒读线程读取数据，大概就是这样的一个读写过程，读线程进入阻塞后，并不会消耗CPU时间，这是epoll机制高效的原因之一。

说了一大堆，我们再回到rebuildEpollLocked方法，rebuildEpollLocked方法中使用了epoll机制，在Linux中，线程之间的通信一般是通过**管道(pipe)**，在rebuildEpollLocked方法中，首先通过**epoll_create**方法创建一个epoll专用文件描述符(mEpollFd)，同时**创建了一个管道**，然后设置监听可读事件类型（EPOLLIN），最后通过**epoll_ctl**方法把Looper对象中的唤醒事件的文件描述符（mWakeEventFd）添加到epoll文件描述符的监控范围内，当mWakeEventFd那一端发生了写入，这时mWakeEventFd可读，就会被epoll监听到（**epoll_wait**方法返回），我们发现epoll文件描述符不仅监听了mWakeEventFd，它还监听了其他的如键盘、鼠标等事件的文件描述符，所以**一个epoll文件描述符可以监听多个文件描述符**。

至此，native层的MessageQueue和Looper就构建完毕，底层通过**管道与epoll机制**也建立了一套消息机制。

我们跟着MessageQueue#nativeInit()一路走下来，这里小结一下：

* 1、首先java层的Looper对象会在构造函数中创建java层的MessageQueue对象;
* 2、 java层的MessageQueue对象又会调用nativeInit函数初始化native层的NativeMessageQueue，NativeMessageQueue的构造函数又会创建native层的Looper，并且在Looper中通过管道与epoll机制建立一套消息机制;
* 3、native层构建完毕，将NativeMessageQueue对象转换为一个long类型存储到java层的MessageQueue的mPtr中。

### 2、nativePollOnce()

在native层通过epoll机制也建立了一套消息机制后，java层的消息循环也就创建好，在此之后就会在java层中启动消息循环，**Looper.loop -> MessageQueue.next**，在java层中每次循环去读消息时，都会调用MessageQueue的next函数，如下:

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
next方法返回一个Message，没有消息时，会调用nativePollOnce方法进入阻塞，nativePollOnce方法的实现在android_os_MessageQueue.cpp文件中的**android_os_MessageQueue_nativePollOnce**方法中，该方法的源码如下:
```c++
//frameworks/base/core/jni/android_os_MessageQueue.cpp
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jlong ptr, jint timeoutMillis) {
    //把ptr强转为NativeMessageQueue
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
}
```
ptr是从java层传过来的mPtr的值，mPtr在初始化时保存了NativeMessageQueue的指针，此时首先把传递进来的ptr转换为NativeMessageQueue，然后调用**NativeMessageQueue的pollOnce**函数，该函数核心源码如下:
```c++
//frameworks/base/core/jni/android_os_MessageQueue.cpp
void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    //...
    //核心是调用了native层的Looper的pollOnce方法
    mLooper->pollOnce(timeoutMillis);  
    //...
}
```
NativeMessageQueue是一个代理类，所以它把逻辑转交给Looper，这段代码主要就是调用了native层的Looper的**pollOnce(timeoutMillis)**方法，该方法定义在Looper.h文件中，如下：

```C++
//system/core/libutils/Looper.h
inline int pollOnce(int timeoutMillis) {
    //调用了带4个参数的pollOnce方法
    return pollOnce(timeoutMillis, NULL, NULL, NULL);
}
```

 pollOnce(timeoutMillis)方法会调用Looper的**polOnce(timeoutMillis, NULL, NULL, NULL)**，该方法的实现在Looper.cpp文件中，如下:

```c++
//system/core/libutils/Looper.cpp
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    //一个死循环
    for (;;) {
        //...
        //当result不等于0时，就会跳出循环，返回到java层
        if (result != 0) {
            //...
            return result;
        }
        //处理内部轮询
        result = pollInner(timeoutMillis);
    }
}
```
该方法内部是一个死循环，核心在于调用了**pollInner**方法，pollInner方法返回一个int值result，代表着本次轮询是否成功处理了消息，当result不等于0时，就会跳出循环，**返回到java层继续处理java层消息**，result有以下4种取值：

```java
enum {
    //表示Looper的wake方法被调用，即管道的写端的write事件触发
    POLL_WAKE = -1,

    //表示某个被监听fd被触发。
    POLL_CALLBACK = -2,

    //表示等待超时
    POLL_TIMEOUT = -3,
    
    //表示等待期间发生错误
    POLL_ERROR = -4,
};
```

我们接着来看**pollInner**方法，如下:

```c++
//system/core/libutils/Looper.cpp
int Looper::pollInner(int timeoutMillis) { 
    //timeoutMillis等于-1，并且mNextMessageUptime不等于LLONG_MAX
    //这说明java层没有消息但是native层有消息处理，这时在epoll_wait中，线程不能因为timeoutMillis等于-1而进入休眠，它还需要处理native层消息
    //所以这里会根据mNextMessageUptime把timeoutMillis更新为大于0的值
    if (timeoutMillis != 0 && mNextMessageUptime != LLONG_MAX) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        int messageTimeoutMillis = toMillisecondTimeoutDelay(now, mNextMessageUptime);
        if (messageTimeoutMillis >= 0
                && (timeoutMillis < 0 || messageTimeoutMillis < timeoutMillis)) {
            //更新timeoutMillis为大于0的值，这个大于0的值就是需要等待多久后，才会到达native层消息的执行时间，等待timeoutMillis后，epoll_wait就会返回处理native层消息
            timeoutMillis = messageTimeoutMillis;
        }
        //...
    }
    int result = POLL_WAKE;
    //...
    //事件集合(eventItems)，EPOLL_MAX_EVENTS为最大事件数量，它的值为16
    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    
    //1、等待事件发生或者超时(timeoutMillis)，如果有事件发生，就从管道中读取事件放入事件集合(eventItems)返回，如果没有事件发生，进入休眠等待，如果timeoutMillis时间后还没有被唤醒，就会返回
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);  
    
    //获取锁
    mLock.lock();
    
    //...省略的逻辑是：如果eventCount <= 0 都会直接跳转到Done:;标记的代码段

    //2、遍历事件集合（eventItems），检测哪一个文件描述符发生了IO事件
    for (int i = 0; i < eventCount; i++) {
        //取出文件描述符
        int fd = eventItems[i].data.fd;
        //取出事件类型
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeEventFd) {//如果文件描述符为mWakeEventFd
            if (epollEvents & EPOLLIN) {//并且事件类型为EPOLLIN（可读事件）
                //这说明当前线程关联的管道的另外一端写入了新数据
                //调用awoken方法不断的读取管道数据，直到清空管道
                awoken();
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake event fd.", epollEvents);
            }
        } else {//如果是其他文件描述符，就进行它们自己的处理逻辑
           //...
        }
    }
    
    //2、下面是处理Native的Message
    Done:;
    //mNextMessageUptime如果没有值，会被赋值成LLONG_MAX，但是如果mNextMessageUptime已经有值，它还是保持原来的值
    mNextMessageUptime = LLONG_MAX;
    //mMessageEnvelopes是一个Vector集合，它代表着native中的消息队列
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        //取出MessageEnvelope，MessageEnvelop有收件人Hanlder和消息内容Message
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        //判断消息的执行时间
        if (messageEnvelope.uptime <= now) {//消息到达执行时间
            {
        	    //获取native层的Handler
                sp<MessageHandler> handler = messageEnvelope.handler;
                //获取native层的消息
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                //释放锁
                mLock.unlock();
                //通过MessageHandler的handleMessage方法处理native层的消息
                handler->handleMessage(message);
            }
            mLock.lock();
            mSendingMessage = false;
            //result等于POLL_CALLBACK，表示某个监听事件被触发
            result = POLL_CALLBACK;
        } else {//消息还没到执行时间
            //把消息的执行时间赋值给mNextMessageUptime
            mNextMessageUptime = messageEnvelope.uptime;
            //跳出循环，进入下一次轮询
            break;
        }
    }
    //释放锁
    mLock.unlock();
    //...
    return result;
}
```
pollInner方法很长，省略了一大堆代码，这里讲解一些核心的点，pollInner实际上就是从管道中读取事件，并且处理这些事件，pollInner方法可分为3部分：

**1、执行epoll_wait方法，等待事件发生或者超时**

这里再次贴出epoll_wait方法的作用：

> **int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout)**： 等待事件的上报, 该函数返回需要处理的事件数目，如返回0表示已超时;
> **参数介绍**：
> **epfd**：等待epfd上的io事件，最多返回maxevents个事件；
> **events**：用来从内核得到事件的集合；
> **maxevents**：events数量，该maxevents值不能大于创建epoll_create()时的size；
> **timeout**：超时时间（毫秒，0会立即返回）.

**epoll_wait**方法就是用来等待事件发生返回或者超时返回，它是一个阻塞方法， 如果**epoll_create**方法创建的epoll文件描述符（mEpollFd）所监听的任何事件发生，**epoll_wait**方法就会监听到，并把发生的事件从管道读取放入事件集合(eventItems)中，返回发生的事件数目eventCount，如果没有事件，epoll_wait方法就会让当前线程进入**休眠**，如果休眠timeout后还没有其他线程写入事件**唤醒**，就会返回，而此时返回的eventCount == 0，表示已经超时，timeout就是从java层一直传过来的**nextPollTimeoutMillis**，它的含义和nextPollTimeoutMillis一样，当timeout == -1时，表示native层的消息队列中没有消息，会一直等待下去，直到被唤醒，当timeout = 0时或到达timeout 时，它会立即返回。

我们发现epoll机制只会把**发生了的事件**放入事件集合中，这样线程对事件集合的每一个事件的相应IO操作都有意义，这也是epoll机制高效的原因之一。

**2、遍历事件集合（eventItems），检测哪一个文件描述符发生了IO事件**

遍历事件集合中，如果是**mWakeEventFd**，就调用awoken方法不断的读取管道数据，直到清空管道，如果是其他的文件描述符发生了IO事件，让它们自己处理相应逻辑。

**3、处理native层的Message**

只要epoll_wait方法返回后，都会进入**Done标记位**的代码段,  就开始处理处理native层的Message,  在此之前先讲解一下**MessageEnvelope**，正如其名字，信封，其结构体定义在Looper.h中，如下:

```c++
//system/core/libutils/Looper.h
class Looper : public RefBase { 
    struct MessageEnvelope {
        MessageEnvelope() : uptime(0) { }

        MessageEnvelope(nsecs_t u, const sp<MessageHandler> h, const Message& m) : uptime(u), handler(h), message(m) {}

        nsecs_t uptime;
        //收信人handler
        sp<MessageHandler> handler;
        //信息内容message
        Message message;
    };   
    //...
}
```
MessageEnvelope里面记录着收信人（handler，**MessageHandler**类型，是一个消息处理类），发信时间(uptime)，信件内容(message，**Message**类型)，Message结构体，消息处理类MessageHandler都定义在Looper.h文件中,  在java层中，消息队列是一个**链表**，在native层中，消息队列是一个C++的**Vector向量**，Vector存放的是MessageEnvelope元素，接下来就进入一个while循环，里面会判断消息是否达到执行时间，如果到达执行时间，就会取出信封中的**MessageHandler**和**Message**，把Message交给MessageHandler的**handlerMessage**方法处理；如果没有到达执行时间，就会更新**mNextMessageUptime**为消息的执行时间，这样在下一次**轮询**时，如果由于java层没有消息导致timeoutMillis等于-1，就会根据mNextMessageUptime更新timeoutMillis为需要等待执行的时间，超时后返回继续处理native层消息队列的头部信息。

我们跟着MessageQueue#nativePollOnce()一路走下来，小结一下：
* 1、当在java层通过Looper启动消息循环后，就会走到MessageQueue的nativePollOnce方法，在该方法native实现中，会把保存在java层的mPtr再转换为NativeMessageQueue；
* 2、然后调用NativeMessageQueue的pollOnce方法，该方法中最终会调用native层的Looper的pollInner方法，Looper的pollInner方法是阻塞方法，等从管道取到事件或超时就会返回，并通过native层的Handler处理native层的Message消息；
* 3、处理完native层消息后，又会返回到java层处理java层的消息，这俩个层次的消息都通过java层的Looper消息循环进行不断的获取，处理等操作.

可以看到，native层的NativeMessageQueue实际上并没有做什么实际工作，只是把操作转发给native层的Looper，而native层的Looper则扮演了java层的Handle角色，它可以取出，发送，处理消息，

### 3、nativeWake()
我们在Java层通过Hanlder发送消息时，实际是把消息添加到消息队列，**Handler.sendXX -> Handler.enqueueMessage -> MessageQueuue.enqueueMessage**，最终会调用到MessageQueue的enqueueMessage方法,，如下：

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

该方法中如果需要进行唤醒MessageQueuue的话，都会调用到nativeWake方法,，MessageQueue的nativeWake方法的实现在android_os_MessageQueue.cpp文件中的**android_os_MessageQueue_nativeWake**方法中，该方法的源码如下:
 ```c++
//frameworks/base/core/jni/android_os_MessageQueue.cpp
static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->wake();
}
 ```
首先把传递进来的ptr转换为NativeMessageQueue，然后调用**NativeMessageQueue的wake**函数，该函数源码如下:
 ```c++
//frameworks/base/core/jni/android_os_MessageQueue.cpp
void NativeMessageQueue::wake() {
    mLooper->wake();
}
 ```
前面说过在native层中NativeMessageQueue只是一个代理Looper的角色，该方法把操作转发给native层的Looper，**Looper的wake**方法核心源码如下:
```c++
//system/core/libutils/Looper.cpp
void Looper::wake() {
    uint64_t inc = 1;
    //使用write函数通过mWakeEventFd往管道写入字符inc
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
    //...
}
```
Looper的wake方法其实是**使用write函数通过mWakeEventFd往管道写入字符inc**，其中TEMP_FAILURE_RETRY 是一个宏定义， 当执行write方法失败后，会不断重复执行，直到执行成功为止，在**nativeinit**中，我们已经通过**epoll_create**方法监听了mWakeEventFd的可读事件，当mWakeEventFd可读时，epoll文件描述符就会监听到，这时**epoll_wait**方法就会从管道中读取事件返回，返回后就执行消息处理逻辑，所以这里的往管道写入字符inc，其实起到一个**通知的作用**，告诉监听的线程有消息插入了消息队列了，快点**醒过来**(因为进入了休眠状态)处理一下。

我们跟着MessageQueue#nativeWake一路走下来，小结一下：
* 1、在java层插入消息到消息队列后，就会根据需要判断是否要调用nativeWake方法，如果调用，就转到2。
* 2、在nativeWake方法native实现中，会把保存在java层的mPtr再转换为NativeMessageQueue，然后调用NativeMessageQueue的wake方法，最终调用Looper的wake方法。
* 3、前面讲到Looper::pollInner方法是一个阻塞操作，当管道中没有事件时当前线程就会进入休眠等待，当管道有事件就会立即返回，从管道中读取事件并处理，而Looper::wake方法就是一个唤醒操作，它就是通过前面创建的唤醒事件文件描述符mWakeEventFd来往管道中写入内容，这时另外等待管道事件的线程就会被唤醒处理事件。

### 4、小结

1、在创建java层的MessageQueue对象同时会在构造中调用**nativeInit**方法创建native层的NativeMessageQueue，在创建NativeMessageQueue同时会在构造中创建native层的Looper对象，并把它保存到TLS区域中，然后返回NativeMessageQueue的指针给java层的**mPtr**保存；

2、在创建Looper时会在构造中通过管道与epoll机制建立一套native层的消息机制，它首先创建一个唤醒文件描述符**mWakeEventFd**，然后使用epoll_create方法创建一个epoll文件描述符**mEpollFd和管道**，然后使用epoll_ctl把mWakeEventFd添加到mEpollFd的监控范围内；

3、当java层使用Handler发送消息时，会把消息插入到消息队列中，然后根据情况调用**nativeWake**方法唤醒阻塞线程，nativeWake方法会调用到native层的Looper的wake方法，里面会通过mWakeEventFd往管道中写入一个字符，唤醒阻塞线程处理消息；

4、当java层使用Looper的loop方法取消息时，如果没有消息，调用**nativePollOnce方法**进入阻塞状态，这时nativePollOnce方法会调用到native层的Looper的pollInner方法，里面会使用epoll_wait等待事件发生或超时，当mEpollFd监听的**任何文件描述符（包括mWakeEventFd）**的相应IO事件发生时，epoll_wait方法就会返回，返回就会通过native层的MessageHandler处理native层的Message，处理完native层消息后，再返回处理java层的消息。

## 总结

Java层和Native层的MessageQueue通过JNI建立关联，从而使得MessageQueue成为Java层和Native层的枢纽，既能处理上层消息，也能处理native层消息，而Handler/Looper/Message这三大类在Java层与Native层之间没有任何的关联，只是分别在Java层和Native层的消息模型中具有相似的功能，都是彼此独立的，各自实现相应的逻辑。

这里我们可以回答为什么java层的loop方法是死循环但却不会消耗性能这个问题：

因为java层的消息机制是依赖native层的消息机制来实现的，而native层的消息机制是通过Linux的**管道和epoll机制**实现的，epoll机制是一种高效的IO多路复用机制， 它使用一个文件描述符管理多个描述符，**java层通过mPtr指针也就共享了native层的epoll机制的高效性**，当loop方法中取不到消息时，便阻塞在MessageQueue的next方法，而next方法阻塞在nativePollOnce方法，nativePollOnce方法通过JNI调用进入到native层中去，最终nativePollOnce方法阻塞在**epoll_wait**方法中，epoll_wait方法会让当前线程释放CPU资源进入**休眠状态**，等到下一个消息到达(mWakeEventFd会往管道写入字符)或监听的其他事件发生时就会唤醒线程，然后处理消息，所以**就算loop方法是死循环，当线程空闲时，它会进入休眠状态，不会消耗大量的CPU资源**。

以上就是本文的所有内容，希望大家有所收获。

参考资料：

[epoll、looper.loop主线程阻塞](https://www.cnblogs.com/muouren/p/11706457.html)

[Android消息机制2-Handler](http://gityuan.com/2015/12/27/handler-message-native/)

