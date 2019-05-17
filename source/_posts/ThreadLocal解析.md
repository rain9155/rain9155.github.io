---
title: ThreadLocal解析
date: 2019-02-21 13:55:50
tags: ThreadLocal
categories: android
---

## 前言
最近在研究[Android的消息机制](https://rain9155.github.io/2019/02/21/Android消息机制java层)时遇到了一个疑问:**在不同线程中调用 Looper.myLooper() 为什么可以返回各自线程的 Looper 对象呢？明明我们没有传入任何线程信息，内部是如何找到当前线程对应的 Looper 对象呢？**，查看源码得知是ThreadLocal的作用，通过它可以在线程的内部存储数据，本着好奇的心态，就通过源码深入的了解一下ThreadLocal的工作原理。
<!--more-->

	本文的源码是基于Android8.0

## ThreadLocal概述
ThreadLocal，线程本地存储区（Thread Local Storage，简称为TLS），通过它可以在指定的线程中存储数据，数据存储之后，只能在指定的线程中可以获取到存储的数据，对于其他线程来说则无法获取到数据。来看一个简单的例子：
```java
public class Main {

    private static ThreadLocal<Integer> mThreadLocal = new ThreadLocal<>();
    private static ThreadLocal<Integer> mThreadLocal2 = new ThreadLocal<>();

    public static void main(String[] args) {

        //主线程
        mThreadLocal.set(0);
        System.out.println(Thread.currentThread().getName() + " - " + mThreadLocal.get());

        mThreadLocal2.set(1);
        System.out.println(Thread.currentThread().getName() + " - " + mThreadLocal2.get());

        //子线程1
        new Thread(() -> {
            mThreadLocal.set(2);
            System.out.println(Thread.currentThread().getName() + " - " + mThreadLocal.get());
        }).start();

        //子线程2
        new Thread(() -> System.out.println(Thread.currentThread().getName() + " - " + mThreadLocal.get())).start();
    }
}

/* 输出结果：*/
main - 0
main - 1
Thread-0 - 2
Thread-1 - null
```
上面代码中，有三个线程。**对于同一个mThreadLoca，不同线程**，在主线程设置mThreadLocal的值为0，在子线程1设置mThreadLocal的值2，在子线程2没有设置mThreadLocal的值，从输出结果可以看出虽然在不同线程访问的是同一个ThreadLocal对象，但通过ThreadLocal获取的值却不一样。**对于不同的mThreadLocal和mThreadLocal2，同一线程**，在主线程分别设置mThreadLocal的值为0，mThreadLocal2的值为1，从输出结果可以看出在同一线程中，通过不同的ThreadLocal存值，则通过相应的ThreadLocal取出的值也不一样。

这里可以提出关于ThreadLocal的俩个问题：
1、ThreadLocal 是如何做到同一个对象，却维护着不同线程的数据副本呢？
2、ThreadLocal是如何做到同一线程中不同 ThreadLocal 虽然共用同一个线程中的容器，但却可以相互独立运作？
## 源码分析
掌握了ThreadLocal的get和set方法的原理，也就掌握了ThreadLocal的工作原理。
### 1、ThreadLocal#get()
获取当前线程TLS区域的数据，该方法的源码如下：
```java
  public T get() {
  	//1、获取当前线程
        Thread t = Thread.currentThread();
        //2、 以当前线程为参数，获取一个 ThreadLocalMap 对象
        ThreadLocalMap map = getMap(t)；
        if (map != null) {
            //3、map不为空，则以当前 ThreadLocal 对象实例作为key值，去map中取值，有找到直接返回
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        //4. map 为空或者在map中取不到值，那么走这里，返回默认初始值
        return setInitialValue();
    }
```
首先获取当前线程，接着以当前线程为参数调用getMap函数，该方法源码如下:
```java
 ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```
getMap方法返回传进来的线程中的threadlocals字段，threadlocals是一个ThreadLocalMap对象，它就是我们上面提到的每个线程中保存数据的容器，定义如下:
```java
//ThreadLocal$ThreadLocalMap
static class ThreadLocalMap {
    static class Entry extends WeakReference<ThreadLocal> {
        Object value;
    }
    private Entry[] table;

    private void set(ThreadLocal key, Object value) {
        ...
    }

    private Entry getEntry(ThreadLocal key) {
        ...   
    }
}
```
ThreadLocalMap 就是一个用于存储数据的容器类, 里面table数组就是真正的存储每个线程的数据，数组的每个元素类型就是一个具有（key-value）键值对的Entry，key对应ThreadLocal实例，value对应要存储的数据，数组的index值是以ThreadLocal实例为key根据hash算法算出来的，里面的set和getEntry方法就是存取table数组的具体算法实现，这里我们不深究，有兴趣可以自行查找资料。
我们继续ThreadLocal#get方法, 调用getMap之后就的到了当前线程的数据存储容器即map：
* 当map不为空时，就以当前ThreadLocal实例为参数调用map.getEntry方法，该方法返回一个ThreadLocalMap.Entry对象, ThreadLocalMap在前面已经介绍过了，它里面有一个元素类型为Entry的table数组，getEntry方法就是以ThreadLocal实例作为key值，然后用key值转成table数组中的index值，返回table中的index位置的元素，而Entry又是一个具有（key-value）键值对的Entry，key对应ThreadLocal实例，value对应要存储的数据，所以e.value就返回了结果即要获取的数据。
* 当map为空时或者在map中找不到数据即map.getEntry返回了null，就调用setInitialValue方法返回默认初始值。该方法源码如下：
```java
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
        //4、返回初始值
        return value;
    }
```
setInitialValue方法返回初始值，第一步调用initialValue()，方法源码如下:
```java
 protected T initialValue() {
        return null;
    }
```
可以看到默认返回null，但是该方法可以继承自ThreadLocal并重写它返回你想要的初始值。
第二步获取当前线程并以当前线程为参数，获取一个ThreadLocalMap 对象，与上面分析get方法时的1、2步骤一样，就不再复述。
第三步同样判断map是否为空:
* 当map不为空时，以当前ThreadLocal实例为key，initialvalue方法获取到的初始值为value，将（key - value）值保存到map中。
* 当map为空时，就调用createMap方法， ThreadLocal 中的 createMap() 方法就是对当前Thread 中的 threadLocals成员变量赋值，该方法源码如下:
```java
//以当前ThreadLocal实例对象为key，存值
void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```
Thread 中的 threadLocal 成员变量初始值为 null，并且在 Thread 类中没有任何赋值的地方，只有在 ThreadLocal 中的 createMap() 方法中对其赋值，而调用 createMap() 的地方就两个：set() 和 setInitialValue()，而调用 setInitialValue() 方法的地方只有 get()。可以看到这里的赋值就是new一个ThreadLocalMap对象,以当前ThreadLocal实例对象为key，存值。

到此，get方法就已经讲完了，我们现在可以回答上面俩个问题：
* 问题1: ThreadLocal

是如何做到同一个对象，却维护着不同线程的数据副本呢？
**原来，这些数据本来就是存储在各自线程中了，ThreadLocal 的 get() 方法内部其实会先去获取当前的线程对象，然后直接将线程存储数据的容器(ThreadLocalMap)取出来，如果为空就会先创建并将初始值和当前 ThreadLocal 对象绑定存储进去，这样不同线程即使调用了同一 ThreadLocal 对象的get方法，取的数据也是各自线程的数据副本，这样自然就可以达到维护不同线程各自相互独立的数据副本，且以线程为作用域的效果了。**
* 问题2:hThreadLocal是如何做到同一线程中不同 ThreadLocal 虽然共用同一个线程中的容器，但却可以相互独立运作？

**原来，ThreadLocal 的 get() 方法内部根据线程取出map后，当map不为空时，会根据ThreadLocal实例去map中查找value，换句话说，在将数据存储到线程的容器map中是以当前 ThreadLocal 对象实例为 key 存储，这样，即使在同一线程中调用了不同的 ThreadLocal 对象的 get() 方法，所获取到的数据也是不同的，达到同一线程中不同 ThreadLocal 虽然共用一个容器，但却可以相互独立运作的效果。**
### 2、ThreadLocal#set(T value)
将value存储到当前线程的TLS区域。在上面的get方法中，ThreadLocal会根据线程取出线程的容器，然后再根据key（ThreadLocal实例）去容器中取值，如果取不到值，就会返回初始值，初始值默认是null，那是因为ThreadLocal要调用set方法后，容器中才有我们想要的值，ThreadLocal#set方法的源码如下:
```java
 public void set(T value) {	
	//1. 取当前线程对象
        Thread t = Thread.currentThread();
        //2. 取当前线程的数据存储容器
        ThreadLocalMap map = getMap(t);
        if (map != null)
        //3. 如果map不为空，以当前ThreadLocal实例对象为key，存值
            map.set(this, value);
        else
        //4. 如果map为空，新建一个当前线程的数据存储容器
            createMap(t, value);
    }
```
set方法中的步骤在get方法中已经分析过了，所以读者在看到set方法时是不是感觉似曾相识，get和set两个方法内部会自动根据当前线程选择相对应的容器存取。
## ThreadLocal应用场景
一般来说，当某些数据是以线程为作用域并且不同线程具有不同的数据副本的时候，就可以考虑采用 ThreadLocal。
## 总结
从get和set方法中都可以看出，它们所操作的对象都是当前线程中的容器ThreadLocalMap，所以在不同线程中访问同一个ThreadLocal的get和set方法，它们对ThreadLocal所做的读写操作仅限与线程内部。一句话总结ThreadLocal原理:**不同线程访问同一个ThreadLocal的get方法，ThreadLocal内部会从各自的线程取出一个容器ThreadLocalMap，然后再从容器中根据当前ThreadLocal的实例去查找对应的value值，这就是为什么通过ThreadLocal可以在不同的线程中维护一套数据的副本并且彼此互不干扰，在同一线程中不同 ThreadLocal 虽然共用同一个线程中的容器，但却可以相互独立运作。**

参考资料:

《Android开发艺术探索》