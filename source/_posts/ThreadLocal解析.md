---
title: ThreadLocal原理解析
date: 2019-02-21 13:55:50
tags: 
- ThreadLocal
- 源码
categories: java
---

## 概述
ThreadLocal，线程本地存储区（Thread Local Storage，简称为TLS），通过它可以在指定的线程中存储数据，数据存储之后，只能在指定的线程中可以获取到存储的数据，对于其他线程来说则无法获取到数据。

## 使用

ThreadLocal 提供了 get()，set(T value)，remove() 3个对外方法，来看一个简单的例子：

```java
public class Main {

    //定义两个ThreadLocal
    private static ThreadLocal<Integer> mThreadLocal = new ThreadLocal<>();
    private static ThreadLocal<Integer> mThreadLocal2 = new ThreadLocal<>();

    public static void main(String[] args) {

        //主线程
        mThreadLocal.set(0);//mThreadLocal存值
        System.out.println(Thread.currentThread().getName() + " - " + mThreadLocal.get());////mThreadLocal取值

        mThreadLocal2.set(1);//mThreadLocal2存值
        System.out.println(Thread.currentThread().getName() + " - " + mThreadLocal2.get());////mThreadLoca2取值

        //子线程1
        new Thread(() -> {
            mThreadLocal.set(2);//mThreadLocal存值
            System.out.println(Thread.currentThread().getName() + " - " + mThreadLocal.get());//mThreadLocal取值
        }).start();

        //子线程2
        new Thread(() -> System.out.println(Thread.currentThread().getName() + " - " + mThreadLocal.get())).start();//mThreadLocal取值
    }
}

/* 输出结果：*/
main - 0
main - 1
Thread-0 - 2
Thread-1 - null
```
上面代码中，有三个线程，分别为主线程(main)、子线程1(Thread-0)、子线程2(Thread-1)，有两个不同的ThreadLocal实例，分别为mThreadLocal、mThreadLocal2，根据输出结果，得出以下结论：

* **1、在同一线程中，通过不同的ThreadLocal存值，则通过相应的ThreadLocal取出的值也不一样**，例如这里在主线程通过分别设置mThreadLocal的值为0，mThreadLocal2的值为1，从输出结果可以看出mThreadLocal取出的值还是0，mThreadLocal2取出的值还是1；
* **2、在不同线程中，访问的是同一个ThreadLocal对象，但通过同一个ThreadLocal获取的值却不一样**，例如这里在子线程1设置mThreadLocal的值2，在子线程2没有设置mThreadLocal的值，从输出结果可以看出通过同一个ThreadLocal获取的值不一样，一个为2，一个为null；

这里给出先给出解释：

在java中，线程的表示是用java.lang.Thread类来表示，在Thread类中定义了一个ThreadLocalMap字段，如下：

```java
//Thread.java
ThreadLocal.ThreadLocalMap threadLocals = null;
```

ThreadLocalMap是ThreadLocal中的一个**静态内部类**，它是一个简化版的HashMap**容器**，也就是说这个容器是以**key-value**来存值的，每个线程都管理着自己的容器，我们可以从外部拿到这个threadLocals，然后往这个容器中存值、取值等，而ThreadLocal就是这样做的，当我们通过ThreadLocal的**set方法**存值时，它会以**当前ThreadLocal实例为key，要存的值为value**，把这个映射保存进相应的线程的容器中，当我们通过ThreadLocal的**get方法**取值时，它会以当前ThreadLocal实例为Key，然后去相应线程的容器中查找这个键为Key的value值返回。

所以：

* 1、**在上面的主线程中**，mThreadLocal和mThreadLocal2这两个ThreadLocal实例都往主线程的容器中存值，但由于mThreadLocal和mThreadLocal2的两个实例不一样，导致key不一样，所以在容器中存放value的位置也不一样，这样就可以根据相应的key获取出相应的value；
* 2、**在上面的子线程1和子线程2中**，都是通过mThreadLoca这个ThreadLocal实例来存取值，但是由于线程实例不一样，导致获取的容器也不一样，所以根据同一个key从不同的容器中获取的value也就不一样。

如果不是很理解，就来看一下下面关于ThreadLocal的源码分析：

## 源码分析

### 1、ThreadLocal

#### 1.1、get方法

该方法用于获取当前线程TLS区域的数据，该方法的源码如下：
```java
//ThreadLocal.java
public T get() {
    //1、获取当前线程
    Thread t = Thread.currentThread();
    //2、 以当前线程为参数，获取一个 ThreadLocalMap 对象
    ThreadLocalMap map = getMap(t)；
        if (map != null) {
            //2.1、map不为空，则以当前 ThreadLocal 对象实例作为key值，去map中取值，有找到直接返回
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                T result = (T)e.value;
                return result;
            }
        }
    //2.2、 map 为空或者在map中取不到值，那么走这里，返回默认初始值
    return setInitialValue();
}

ThreadLocalMap getMap(Thread t) {
    //getMap方法返回传进来的线程中的threadlocals字段，threadlocals是一个ThreadLocalMap对象，它就是我们上面提到的每个线程中保存数据的Map容器
    return t.threadLocals;
}
```
ThreadLocal的get方法，首先获取当前线程，接着以当前线程为参数调用getMap方法，getMap方法返回当前线程中保存数据的**Map容器**，调用getMap之后就得到了当前线程的数据存储容器即map，然后判断它是否为null：

* **注释2.1**：当map不为空时，就以当前ThreadLocal实例为参数调用map.getEntry方法，该方法返回一个ThreadLocalMap.Entry对象，Entry就是线程容器中表示**key-value**映射的类，它里面有一个key和一个value值，而value值就是我们需要的数据；
* **注释2.2**： 当map为空时或者在map中找不到数据即map.getEntry返回了null，就调用setInitialValue方法返回默认初始值，该方法源码如下：
```java
//ThreadLocal.java
private T setInitialValue() {
    //1. 获取初始值，默认返回Null，允许重写
    T value = initialValue();
    //2、获取当前线程并以当前线程为参数，获取一个ThreadLocalMap 对象
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    //3、当map不为空，设置初始值给map
    if (map != null)
        map.set(this, value);
    else//当map为空， 创建当前线程的数据存储容器map
        createMap(t, value);
    //返回初始值
    return value;
}

protected T initialValue() {
     return null;
}
```
该方法分为以下3步：

1、 调用initialValue()，可以看到它默认返回null，但是我们可以**重写该方法并返回你想要的初始值**；

2、获取当前线程，再次以当前线程为参数，获取一个ThreadLocalMap 对象；

3、再次判断map是否为空，如下：

* 当map不为空时，以当前ThreadLocal实例为key，initialvalue方法获取到的初始值为value，将（key - value）值保存到map中；
* 当map为空时，就调用createMap方法， ThreadLocal 中的 createMap() 方法就是对当前Thread 中的 threadLocals成员变量赋值，该方法源码如下:
```java
void createMap(Thread t, T firstValue) {
    //以当前ThreadLocal实例对象为key，传进来的value为值，创建一个ThreadLocalMap实例
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```
Thread 中的 threadLocal 成员变量初始值为 null，并且在 Thread 类中没有任何赋值的地方，只有在 ThreadLocal 中的 createMap方法中对其赋值，而调用createMap方法的地方就两个：**set 和 setInitialValue方法**，而调用 setInitialValue() 方法的地方只有 get方法。

到此，get方法就已经讲完了。

#### 1.2、set方法

set方法是将value存储到当前线程的TLS区域，在上面的get方法中，ThreadLocal会根据线程取出线程的容器，然后再根据key（ThreadLocal实例）去容器中取值，如果取不到值，就会返回初始值，初始值默认是null，那是因为ThreadLocal要调用set方法后，容器中才有我们想要的值，set方法的源码如下:
```java
//ThreadLocal.java
public void set(T value) {	
    //1. 取当前线程对象
    Thread t = Thread.currentThread();
    //2. 取当前线程的数据存储容器
    ThreadLocalMap map = getMap(t);
    if (map != null)
        //2.1. 如果map不为空，以当前ThreadLocal实例对象为key，存值
        map.set(this, value);
    else
        //2.2. 如果map为空，新建一个当前线程的数据存储容器
        createMap(t, value);
}
```
set方法的步骤是不是感觉似曾相识，没错，它和get方法中所讲的**setInitialValue方法几乎一模一样**，只是没有调用initialValue方法返回初始值，因为set方法的参数value就是我们想要保存的值，而不用调用initialValue方法设置默认初始值。

至此，set方法讲解完毕。

get和set两个方法内部都会自动根据当前线程选择相对应的容器存取，所以其实**ThreadLocal的核心还是ThreadLocalMap对象**，get方法会调用ThreadLocalMap的**getEntry(ThreadLocal<?>)**根据ThreadLocal实例获取一个Entry对象，该Entry对象保存了key-value映射，set方法会调用ThreadLocalMap的**set(ThreadLocal<?>, Object)**保存key-value映射到ThreadLocalMap中，下面我们就简单的讲解一下ThreadLocalMap的组成。

### 2、ThreadLocal::ThreadLocalMap

```java
//ThreadLocal::ThreadLocalMap
static class ThreadLocalMap {
  
    private static final int INITIAL_CAPACITY = 16;//初始容量为16，必须为2^n
    private int size = 0;//table中Entry的数量
    private int threshold;//扩容阈值, 当size >= threshold时就会触发扩容逻辑
    private Entry[] table;//table数组
    
    private void setThreshold(int len) {
         //ThreadLocalMap的threshold为table数组长度的2/3
         threshold = len * 2 / 3;
    }
    
    //以下为ThreadLocal调用ThreadLocalMap的主要方法
    //分别对应ThreadLocal的get、set、remove方法
    
    private Entry getEntry(ThreadLocal<?> key) {
        //...
    }
    
    private void set(ThreadLocal<?> key, Object value) {
        //...
    }

    private void remove(ThreadLocal<?> key) {
        //...
    }
    
    //...
}
```
ThreadLocalMap 就是一个用于存储数据的容器类, 它是ThreadLocal中的静态内部类， 它的底层实现类似于hashMap的实现，也是基于哈希算法，里面table数组就是真正的存储每个线程的数据，数组的每个元素类型就是一个具有（key-value）键值对的Entry，key对应ThreadLocal实例，value对应要存储的数据，Entry在数组中的index值是根据key的**threadLocalHashCode**用**hash算法**算出来的，threadLocalHashCode是ThreadLocal中的一个字段，如下：

```java
//ThreadLocal.java
private final int threadLocalHashCode = nextHashCode();//threadLocalHashCode的值等于nextHashCode方法的返回值
private static AtomicInteger nextHashCode = new AtomicInteger();
private static final int HASH_INCREMENT = 0x61c88647;
private static int nextHashCode() {
    //每次调用nextHashCode方法都会在原本的int值加上0x61c88647后再返回
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

而ThreadLocalMap 的hash算法，即计算index值的算法如下：

```java
//hash算法：threadLocalHashCode与（table数组长度-1）相与
//这样会使得i均匀的分布在数组的长度之内
int i = key.threadLocalHashCode & (table.length - 1);
```

当出现冲突时，ThreadLocalMap是使用**线性探测法**来解决冲突的，即如果i位置已经有了key-value映射，就会在i + 1位置找，直到找到一个合适的位置。

我们看一下Entry的实现，如下：

```java
static class Entry extends WeakReference<ThreadLocal> {
    Object value;
    Entry(ThreadLocal<?> k, Object v) {
        //key是弱引用
        super(k);
        value = v;
    }
}
```

Entry 继承至 **WeakReference**，并且它的**key是弱引用**，但是**value是强引用**，所以如果**key关联的ThreadLocal实例**没有强引用，只有弱引用时，在gc发生时，ThreadLocal实例就会被gc回收，当ThreadLocal实例被gc回收后，由于value是强引用，导致table数组中存在着**null - value**这样的映射，称之为**脏槽**，这种脏槽会浪费table数组的空间，所以需要及时清除，所以ThreadLocalMap 中提供了**expungeStaleEntry**方法和**expungeStaleEntries**方法去清理这些脏槽，每次ThreadLocalMap 运行getEntry、set、remove等方法时，都会主动的间接使用这些方法去清理脏槽，从而释放更多的空间，避免无谓的扩容操作。

> java中有4种引用，分别为：强引用、软引用、弱引用和虚引用.


### 3、小结

根据使用实例和源码分析，我们得出以下两个结论：

1、**使用同一个ThreadLocal对象，可以维护着不同线程的数据副本：**

这是因为，这些数据本来就是存储在各自线程中了，ThreadLocal 的 get() 方法内部其实会先去获取当前的线程对象，然后直接将线程存储数据的容器(ThreadLocalMap)取出来，如果为空就会先创建并将初始值和当前 ThreadLocal 对象绑定存储进去，这样不同线程即使调用了同一 ThreadLocal 对象的get方法，取的数据也是各自线程的数据副本，这样自然就可以达到维护不同线程各自相互独立的数据副本，且以线程为作用域的效果了。

2、**在同一线程中不同ThreadLocal对象虽然共用同一个线程中的容器，但却可以相互独立运作：**

这是因为，ThreadLocal 的 get() 方法内部根据线程取出map后，当map不为空时，会根据ThreadLocal实例去map中查找value，换句话说，在将数据存储到线程的容器map中是以当前 ThreadLocal 对象实例为 key 存储，这样，即使在同一线程中调用了不同的 ThreadLocal 对象的 get() 方法，所获取到的数据也是不同的，达到同一线程中不同 ThreadLocal 虽然共用一个容器，但却可以相互独立运作的效果。

## 使用场景
如果你是单线程环境，那么不用考虑使用ThreadLocal了，ThreadLocal是用来在多线程环境下的。

在多线程环境下，如果某个变量只在特定的某个线程中使用，即我们对这个变量的操作**只限定在同一个线程内**，那么就不需要使用同步来保证这个变量的正确性，因为没有存在竞争，这时我们可以把这个变量直接存储在线程内部中，要使用这个变量时直接从线程内部拿出来后再操作，这就**避免了使用同步带来的性能消耗**，典型的例子有Android中的Looper，通过Looper.myLooper方法就可以返回当前线程关联的Looper。

总的来说，当某些数据是以线程为作用域并且不同线程具有不同的数据副本的时候，就可以考虑采用 ThreadLocal。

## 正确使用

上面介绍ThreadLocalMap时提到，如果ThreadLocalMap中的key关联的ThreadLoca实例被回收了，就会导致ThreadLocalMap还残留着这个key对应的value实例，出现了脏槽，而脏槽是通过ThreadLocalMap**主动的调用expungeStaleEntry或expungeStaleEntries方法清理**，而这两个方法只会在主动调用ThreadLocalMap的set(ThreadLocal , Object)、getEntry(ThreadLocal)和remove(ThreadLocal)等方法时才会被调用，而ThreadLocalMap的set、getEntry、remove方法只有在调用ThreadLocal的set、get、remove方法时才会被调用。

所以想象这样的一种情况：我们使用ThreadLocal的set方法往线程的ThreadLocalMap中保存了一个**非常大**的数据，从这之后，我没有再调用过ThreadLocal的set、get、remove等方法，当这个数据对应的**ThreadLocal实例被gc回收**后，ThreadLocalMap中还残留这这个null-value映射，并且这个**线程的生命周期是和程序同步**的，直到程序结束它才会结束，这样就导致了内存泄漏的发生，产生内存浪费。

所以我们平常使用完ThreadLocal后，应该手动的调用**remove方法**把映射删除，如下：

```java
private static ThreadLocal<Integer> mThreadLocal = new ThreadLocal<>();
try {
    //调用mThreadLocal的set方法
} finally {
    threadLocal对象.remove();
}
```

为什么ThreadLocal要定义为静态变量？，可以参考：

{% asset_img threadlocal1.png thread %}

## 总结

本文从使用到源码简单的分析了一下ThreadLocal，介绍了ThreadLocal的使用场景和正确使用方法，从ThreadLocal的get和set方法中都可以看出，它们所操作的对象都是当前线程中的容器ThreadLocalMap，所以在不同线程中访问同一个ThreadLocal的get和set方法，它们对ThreadLocal所做的读写操作仅限与线程内部。

参考资料：

[ThreadLocal 相关的各种面试问法了解一下？](https://mp.weixin.qq.com/s/vZXg87bKA5UrNKcrFNuQvQ)