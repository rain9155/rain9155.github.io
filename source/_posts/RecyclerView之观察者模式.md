---
title: RecyclerView之观察者模式
date: 2019-03-09 17:39:00
tags:  
- recyclerView
- 设计模式
categories: recyclerView
---

## 前言
RecyclerView是Android开发中的一个重要的模式，通常我们往RecyclerView添加数据时，都会调用Adapter的notifiyXX函数，这是为什么呢，今天我们就从源码来探究一下，对观察者模式不熟悉的读者，可以看一下这一篇博客[观察者模式](https://blog.csdn.net/Rain_9155/article/details/83004247), RecyclerView在更新数据时也算是对观察者模式的一种应用。
<!--more-->

    本文源码基于Android8.0, 相关源码位置如下
    frameworks/support/v7/recyclerview/src/android/support/v7/widget/RecyclerView.java
    frameworks/base/core/java/android/database/Observable.java

## Adapter.notifyDataSetChange()
我们来看一下我们平常可能使用到的notifyXX方法：
{% asset_img rv5.png rv5 %}
可以看到，RecyclerView可ListView相比多了很多notifyItemXX方法，说明RecyclerView支持定向刷新，如果只有部分itemView数据发生变化，在使用ListView时我们没得选择只能使用notifyDataSetChange()方法来对整体itemView更新数据，但是在RecyclerView中，我们可以只对发生数据变化的itemView更新，当样也可以整体更新，而且相信大家现在在使用RecyclerView更新itemView时使用最多的方法还是Adapter.notifyDataSetChange()吧。那我们就以这个方法为例，该方法的源码如下:
```java
 public final void notifyDataSetChanged() {
            mObservable.notifyChanged();
 }
```
这个mObservable是声明在Adapter中的AdapterDataObservable对象，如下:
```java
public abstract static class Adapter<VH extends ViewHolder> {
    private final AdapterDataObservable mObservable = new AdapterDataObservable();
    //...
}
```
而AdapterDataObservable定义在RecyclerView中，Adapter.notifyDataSetChange()调用了AdapterDataObservable.notifyChanged()方法，该方法源码如下:
```java
static class AdapterDataObservable extends Observable<AdapterDataObserver> {
     public void notifyChanged() {
        for (int i = mObservers.size() - 1; i >= 0; i--) {
            mObservers.get(i).onChanged();
        }
    }
//...
}
```
可以看到该方法做的事情是遍历mObservers集合，然后逐个调用onChanged()方法，那么mObservers是什么东西？mObservers其实就是一个观察者列表，而mObservable就是一个被观察者，每个Adapter中只有一个被观察者，被观察者中有一个观察者列表，当有数据更新时，被观察者就会调用遍历调用注册到观察者列表中观察者的onChanged方法，来通知观察者更新数据。
mObservers其实是一个ArrayList，它定义在AdapterDataObservable的父类Observable中，Observable的定义如下:
```java
public abstract class Observable<T> {
    protected final ArrayList<T> mObservers = new ArrayList<T>();//观察者集合列表
    
    //注册一个观察者
    public void registerObserver(T observer) {
        //...
        mObservers.add(observer);
    }
    
    //取消该观察者的注册
    public void unregisterObserver(T observer) {
        //...
        mObservers.remove(index);
    }

    //取消所有观察者的注册
    public void unregisterAll() {
        synchronized(mObservers) {
            mObservers.clear();
        }
    }
}
```
Observable中只有几个简单的方法，所以我们要向观察者列表中注册一个观察者，才能接受到更新通知，那么RecyclerView是怎么注册一个观察者的吗？其实是通过RecyclerView.setAdapter()方法实现的。

## RecyclerView.setAdapter()
我们每次使用RecyclerView都要调用setAdapter()设置一个Adapter，不然数据就无法展示，该方法的源码如下:
```java
public void setAdapter(@Nullable Adapter adapter) {
    //...
    setAdapterInternal(adapter, false, true);
}
```
在setAdapter又调用了setAdapterInternal(), 该方法相关源码如下:
```java
private void setAdapterInternal(@Nullable Adapter adapter, boolean compatibleWithPrevious, boolean removeAndRecycleViews) {
    //1、移除旧的Adapter，并注销观察者
    if (mAdapter != null) {
        mAdapter.unregisterAdapterDataObserver(mObserver);
    }
    
    //2、compatibleWithPrevious为false，表示不使用旧的Adapter中ViewHolder，所以调用removeAndRecycleViews方法把ViewHolder旧的Adapter中的ViewHolder回收复用
    if (!compatibleWithPrevious || removeAndRecycleViews) {
        removeAndRecycleViews();
    }
    
    //3、更新Adapter并注册一个观察者
    final Adapter oldAdapter = mAdapter;
    mAdapter = adapter;
    if (adapter != null) {
        //注册一个观察者
        adapter.registerAdapterDataObserver(mObserver);
    }
    //...
}
```
setAdapterInternal()方法中就主要做了上面3件事，而且上面Adapter中调用的registerXX或unregisterXX最终调用mObservable的方法, 如下:
```java
public abstract static class Adapter<VH extends ViewHolder> {

     public void registerAdapterDataObserver(@NonNull AdapterDataObserver observer) {
        mObservable.registerObserver(observer);
     }
     
    public void unregisterAdapterDataObserver(@NonNull AdapterDataObserver observer) {
        mObservable.unregisterObserver(observer);
    }
    
    //...
}
```
这些方法的含义已经在上面讲解Observable时解释过了。那么在setAdapterInternal()中注册的观察者mObserver是什么呢？它其实就是一个RecyclerViewDataObserver类型，定义在RecyclerView中，如下:
```java
public class RecyclerView extends ViewGroup implements ScrollingView, NestedScrollingChild2 {
    private final RecyclerViewDataObserver mObserver = new RecyclerViewDataObserver();
    //...
}
```
而RecyclerViewDataObserver是AdapterDataObserver的子类，它定义在RecyclerView中，如下：
```java
private class RecyclerViewDataObserver extends AdapterDataObserver {
      
        @Override
        public void onChanged() {
            //...
            if (!mAdapterHelper.hasPendingUpdates()) {
                //请求重新布局
                requestLayout();
            }
        }
        
    //...
}
```
可以看到我们熟悉的onChange方法，RecyclerViewDataObserver重写了它，当满足一定条件时就会重新布局从而从可以从Adapter中获取更新数据并绑定数据到itemView,达到更新itemView的目的。
所以RecyclerView在设置Adapter是时，会注册一个观察者mObserver到Adapter的被观察者mObservable中。
## 总结
最后我们来整理一下这个过程，RecyclerView中有一个观察者mObserver，是RecyclerViewDataObserver类型，在RecyclerView设置Adapter时会把它注册到Adapter中，而Adapter中包含一个被观察者mObservable，是AdapterDataObservable类型，注册到Adapter中的观察最终会注册到mObservable的mObservers列表中，当我们手动调用Adapter的notifyXX函数时，notifyXX函数实际上会调用AdapterDataObservable的notifyXX函数，该函数会遍历所有观察者的onChange函数，在RecyclerViewDataObserver的onChange函数中会要求RecyclerView调用requestLayout()重新布局,更新用户界面。如图：
{% asset_img rv6.png rv6 %}
