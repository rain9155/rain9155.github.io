---
title: RecyclerView之缓存设计
date: 2019-03-08 14:56:50
tags: 
- recyclerView
- 源码
categories: recyclerView
---

## 前言

* 上一篇文章[RecyclerView之布局设计](https://rain9155.github.io/2019/03/01/RecyclerView之布局设计)

RecyclerView，见名之义，这个View代表了可循环使用的视图集合控件，封装了View的缓存逻辑判断，RecyclerView的基本单元是ViewHolder，里面有一个itemView代表了视图上的子View，所以RecyclerView的缓存基本单元也是ViewHolder。本文将从源码的角度来讲解RecyclerView的缓存设计。
<!--more-->

    本文相关源码基于Android8.0，相关源码位置如下:
    frameworks/support/v7/recyclerview/src/android/support/v7/widget/RecyclerView.java
    frameworks/support/v7/recyclerview/src/android/support/v7/widget/LinearLayoutManager.java

## Recycler
这里首先介绍一下Recycler，它定义在RecyclerView中，如下：
```java
public final class Recycler {
    final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();//缓存着在屏幕中显示的ViewHolder
    final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();//缓存着已经滚动出屏幕的ViewHolder,即屏幕外的ViewHolder
    RecycledViewPool mRecyclerPool;//ViewHolder的缓存池，屏幕外缓存的mCachedViews已满时，会将ViewHolder缓存到RecycledViewPool中。
    private ViewCacheExtension mViewCacheExtension;//自定义缓存，自己实现ViewCacheExtension类来实现缓存。
    ArrayList<ViewHolder> mChangedScrap = null;//屏幕内缓存，缓存着数据已经改变的ViewHolder
    int mViewCacheMax = DEFAULT_CACHE_SIZE;//mCachedViews默认缓存数量
    static final int DEFAULT_CACHE_SIZE = 2;//默认缓存数量为2
    private int mRequestedCacheMax = DEFAULT_CACHE_SIZE; //可以设置mCachedViews的最大缓存数量，默认为2。
}
```
Recycler是RecyclerView的核心类，是RecyclerView的缓存实现类，它有着四级缓存：

* 1、mAttachedScrap
当我们调用notifiXX函数重新布局时，在布局之前，LayoutManager会调用detachAndScrapAttachedViews(recycler)把在RecyclerView中显示的ViewHolder一个个的剥离下来,然后缓存在mAttachedScrap中，等布局时会先从mAttachedScrap查找，再把ViewHolder一个个的放回RecyclerView中去，而mAttachedScrap中的ViewHolder并不会参与回收复用，只是单纯的为了从RecyclerView中剥离下来，再重新放回RecyclerView，如果还有剩余的ViewHolder没有参加新布局，会从mAttachedScrap移到mCachedViews中。
* 2、mCachedViews
在RecyclerView滚动时，对于那些不在RecyclerView中显示的ViewHolder，LayoutManager会调用removeAndRecycleAllViews(recycler)把这些已经移除的ViewHolder缓存在mCacheViews中，它的默认大小是2，当它满了的时候，就会利用先进先出原则，把老的ViewHolder移到mRecyclerPool中。mCachedViews也不参与回收复用，它只是缓存最新被移除的ViewHolder。
* 3、mViewCacheExtension
自定义缓存实现，一般而言，我们不会自定义缓存实现，使用Recycler提供的3级缓存足够。
* 4、mRecyclerPool
其实真正参与回收复用的是mRecyclerPool，通过前面1、2可以知道，真正废弃的ViewHolder最终移到mRecyclerPool，当我们向RecyclerView申请一个HolderView来使用的时，如果在mAttachedScrap、mCachedViews匹配不到，即使他们中有ViewHolder也不会返回给我们使用，而是会到mRecyclerPool中去拿一个废弃的ViewHolder返回。mRecyclerPool内部维护了一个SparseArray，在mRecyclerPool中会根据每个ViewType把ViewHolder分别存储在不同的列表中，每个ViewType默认缓存5个ViewHolder。RecyclerViewPool大概结构如下：
```java
 public static class RecycledViewPool {
        private static final int DEFAULT_MAX_SCRAP = 5;
        static class ScrapData {
            final ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
            int mMaxScrap = DEFAULT_MAX_SCRAP;
            //...
        }
        SparseArray<ScrapData> mScrap = new SparseArray<>();
        //...
}
```
而且RecyclerViewPool也可以是多个RecyclerView之间的ViewHolder的缓存池，只要通过RecyclerView.setRecycledViewPool(RecycledViewPool)设置同一个RecycledViewPool，设置时，不需要自己去new 一个 RecyclerViewPool，每个RecyclerView默认都有一个RecyclerViewPool，只需要通过mRecyclerView.getRecycledViewPool()获取。

所以我们从Recycler中获取一个ViewHolder时，是这样的顺序：mAttachedScrap -> mCachedViews -> mViewCacheExtension -> mRecyclerPool,当上述步骤都找不到了，就会调用Adapter的creat函数创建一个ViewHolder。那这里为什么省略mChangedScrap不讲呢？因为mChangedScrap是跟RecyclerView的预布局有关，缓存着RecyclerView中数据改变过的ViewHolder，而预布局默认为false，一般是RecyclerView执行动画时才会为true，我们上一篇文章也没有讨论执行动画的时候的布局过程，所以这里就不分析mChangedScrap。

## Recycler.getViewForPosition()
在上篇文章中，提到在layoutChunk函数中，首先会调用LayoutState对象的next函数获取到一个itemView，然后布局这个itemView，我们来看LayoutState的next函数相关实现:
```java
View next(RecyclerView.Recycler recycler) {
    //省略了一个mScrapList，属于LayoutManager，跟执行动画时的缓存有关，这里不分析
    //...
    //这里才是核心，调用Recycler中的getViewForPosition获取itemView
    final View view = recycler.getViewForPosition(mCurrentPosition);
    //把itemView索引移到下一个位置
    mCurrentPosition += mItemDirection;
    return view;
}
```
上述代码实际是调用RecyclerView.Recycler对象的getViewForPosition方法获取itemView，而该函数最终会获取一个ViewHolder，从而返回ViewHolder中的itemView，我们来看该函数相关调用和实现：
```java
public View getViewForPosition(int position) {
    return getViewForPosition(position, false);
}

View getViewForPosition(int position, boolean dryRun) {
    //可以看到最终返回的是ViewHolder中的itemView
     return tryGetViewHolderForPositionByDeadline(position, dryRun, FOREVER_NS).itemView;
}

//获取一个ViewHolder
ViewHolder tryGetViewHolderForPositionByDeadline(int position, boolean dryRun, long deadlineNs) {
    //...
}
```
Recycler的getViewForPosition方法最终会调用到tryGetViewHolderForPositionByDeadline方法，tryGetViewHolderForPositionByDeadline方法的意图是通过给定的position从Recycler的scrap, cache，RecycledViewPool获取一个ViewHolder或者通过Adapter直接创建一个ViewHolder。我们来看tryGetViewHolderForPositionByDeadline方法相关源码：
```java
//参数解释：
//position：要获得哪个位置的ViewHolder
//dryRun: 代表position的ViewHolder是否已经从scrap或cache列表中移除，这里为false，表示没有，因为布局函数layoutChildren中一定会调用detachAndScrapAttachedViews(recycler)函数，表示把ViewHolder放入scrap列表中
ViewHolder tryGetViewHolderForPositionByDeadline(int position, boolean dryRun, long deadlineNs) {
    //省略了跟预布局有关的mChangedScrap获取ViewHolder，mChangedScrap不属于常规缓存
    //...
    
    ViewHolder holder = null;
    
    if (holder == null) {
    
        //1、第一次查找，通过position从scrap或hidden或cache中找ViewHolder
         holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
            //如果找到ViewHolder，检查ViewHolder的合法性
            if (holder != null) {
                //检查ViewHolder的是否被移除，position是否越界等，如果检查通过返回true，失败返回false
                if (!validateViewHolderForOffsetPosition(holder)) {
                    //检查不通过
                    //上述讲过dryRun为false
                    if (!dryRun) {
                        //设置这个ViewHolder为无效标志
                        holder.addFlags(ViewHolder.FLAG_INVALID);
                        //把这个ViewHolder从scrap列表中移除
                        if (holder.isScrap()) {
                            removeDetachedView(holder.itemView, false);
                            holder.unScrap();
                        } 
                        //...
                        //把这个ViewHolder放入cache列表中或mRecyclerPool中
                        recycleViewHolderInternal(holder);
                    }
                    //置空不匹配的ViewHolder，进入下一步查找
                    holder = null;
                } else {
                    //检查通过了
                    fromScrapOrHiddenOrCache = true;
                }
            }
            
    }
    
     if (holder == null) {
     
        //...
        final int offsetPosition = mAdapterHelper.findPositionOffset(position);
        //这里可以看到我们熟悉的Adapter中的getItemViewType方法，重写此方法可以让RecyclerView显示多种type的itemView
        final int type = mAdapter.getItemViewType(offsetPosition);
        
        //如果mAdapter.hasStableIds()为true，就进入第2次查找，默认返回false
         if (mAdapter.hasStableIds()) {
            //2、第2次查找，根据ViewHolder的type和id从scrap或cached列表查找
            holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
                    type, dryRun);
            if (holder != null) { //找到了
                //更新ViewHolder的位置
                holder.mPosition = offsetPosition;
                fromScrapOrHiddenOrCache = true;
            }
        }
        
        if (holder == null && mViewCacheExtension != null) {
            //3、第3次查找，从自定义缓存中查找，一般我们不会重写ViewCacheExtension
            final View view = mViewCacheExtension.getViewForPositionAndType(this, position, type);
            //...
        }
        
         if (holder == null) {
            //4、第4次查找，从RecycledViewPool中查找，可以看到这里会根据type返回一个使用过的ViewHolder给你
            holder = getRecycledViewPool().getRecycledView(type);
            if (holder != null) {//找到了
                //重置ViewHolder中的信息
                holder.resetInternal();
                //...
            }
        }
        
        //前面的4次还找不到合适的ViewHolder，就重新创建一个
        if (holder == null) {
             //...
             //5、这里会调用Adapter中的OnCreateViewHolder方法
             holder = mAdapter.createViewHolder(RecyclerView.this, type);
        }
    
    }
    
    //...
    boolean bound = false;
    //这里会根据情况调用Adapter中的OnBindViewHolder方法
    if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
        final int offsetPosition = mAdapterHelper.findPositionOffset(position);
        //这里最终调用Adapter中的OnBindViewHolder方法
        bound = tryBindViewHolderByDeadline(holder, offsetPosition, position, deadlineNs);
    }
    
}
```
看起来函数很长但是步骤还是很清晰的，我们一步步来看：

### 注释1：getScrapOrHiddenOrCachedHolderForPosition()
注释1中通过position从scrap或hidden或cache中找ViewHolder，我们来看getScrapOrHiddenOrCachedHolderForPosition方法的关键源码：
```java
 ViewHolder getScrapOrHiddenOrCachedHolderForPosition(int position, boolean dryRun) {
 
    //1.1、第一次尝试，从mAttachedScrap找到一个精确，没有失效的ViewHolder并返回
    final int scrapCount = mAttachedScrap.size();
    for (int i = 0; i < scrapCount; i++) {
        final ViewHolder holder = mAttachedScrap.get(i);
        if (!holder.wasReturnedFromScrap() && holder.getLayoutPosition() == position
                && !holder.isInvalid() && (mState.mInPreLayout || !holder.isRemoved())){
            //标志这个ViewHolder是从mAttachedScrap取出并返回的
            holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
            return holder;
        }
    }
    
    //1.2、第二次尝试，dryRun为false，从RecyclerView中隐藏的itemView中找，如果找到合适的View，就让它显示并把它从RecyclerView中剥离，然后根据这个View的LayoutParam获取ViewHolder，最后把这个ViewHolder放入mAttachedScrap并返回
    if (!dryRun) {
        View view = mChildHelper.findHiddenNonRemovedView(position);
        if (view != null) {
            //获取ViewHolder
            final ViewHolder vh = getChildViewHolderInt(view);
            //显示这个View
            mChildHelper.unhide(view);
            //从RecyclerView剥离这个View
            mChildHelper.detachViewFromParent(layoutIndex);
            //把这个ViewHolder放入mAttachedScrap
            scrapView(view);
            vh.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP
                    | ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST);
            //返回
            return vh;
        }
    }
    
    //1.3、第三次尝试，从mCachedViews找到没有失效的ViewHolder并返回
    final int cacheSize = mCachedViews.size();
    for (int i = 0; i < cacheSize; i++) {
        final ViewHolder holder = mCachedViews.get(i);
        if (!holder.isInvalid() && holder.getLayoutPosition() == position) {
            if (!dryRun) {
                mCachedViews.remove(i);
            }
            return holder;
        }
    }
    return null;
}
```
可以看到注释1的第一次查找，里面分为3步：

* 1.1、从mAttachedScrap找。
* 1.2、如果上一步没有得到合适的缓存，从HiddenViews找。
* 1.3、如果上一步没有得到合适的缓存，从mCachedViews找。

从上面3个步骤之一找到，就返回ViewHolder，然后检查ViewHolder的有效性，如果无效，则从mAttachedScrap中移除，并加入到mCacheViews或者mRecyclerPool中，并且将ViewHolder置为null，走到下一步。

### 注释2：getScrapOrCachedViewForId()
下一步就是注释2，如果我们通过Adapter.setHasStableIds(boolean)设置为true，就会进入,里面根据ViewHolder的type和id从scrap或cached列表查找ViewHolder，我们来看一下相关源码该方法的相关源码：
```java
ViewHolder getScrapOrCachedViewForId(long id, int type, boolean dryRun) {

//2.1、第一次尝试，从mAttachedScrap找到一个id相同并且没有从mAttachedScrap取出并返回过的ViewHolder，还要type相同的ViewHolder返回
    final int count = mAttachedScrap.size();
    for (int i = count - 1; i >= 0; i--) {
        final ViewHolder holder = mAttachedScrap.get(i);
        if (holder.getItemId() == id && !holder.wasReturnedFromScrap()) {
            //id相同type相同
            if (type == holder.getItemViewType()) {
                holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
                //...
                }
                return holder;
            } else if (!dryRun) {
                //id相同但type不同
                //从mAttachedScrap移除这个ViewHolder
                mAttachedScrap.remove(i);
                removeDetachedView(holder.itemView, false);
                //把这个ViewHolder放入caches或RecyclerViewPool
                quickRecycleScrapView(holder.itemView);
            }
        }
    }
    
    //2.2、第2次尝试，从mCachedViews中找到一个id相同并且type相同的ViewHolder返回
    final int cacheSize = mCachedViews.size();
    for (int i = cacheSize - 1; i >= 0; i--) {
        final ViewHolder holder = mCachedViews.get(i);
        if (holder.getItemId() == id) {
            //id相同并且type相同
            if (type == holder.getItemViewType()) {
                if (!dryRun) {
                    //从cache中移除
                    mCachedViews.remove(i);
                }
                return holder;
            } else if (!dryRun) {
                //id相同type不相同
                //把这个ViewHolder从cache中移除并放入RecyclerViewPool中
                recycleCachedViewAt(i);
                return null;
            }
        }
    }
    return null;
}
```
可以看到注释2的第二次查找，里面分为2步：

* 2.1、从mAttachedScrap找。
* 2.2、如果上一步没有得到合适的缓存，从mCachedViews找。

第二次查找跟第一次不同的是，它是通过Adapter.getItemId(position)获得该位置ViewHolder的id，来查找ViewHolder，我们可以重写Adapter.getItemId(position)返回每个position的ViewHolder的id，默认返回RecyclerView.NO_ID。从上面2个步骤之一找到，就返回ViewHolder，如果找不到就进入下一步。

### 注释3：mViewCacheExtension
注释3的第三次查找是从自定义缓存中查找，这个没什么好说，可以直接到下一步。

#### 注释4：
下一步就是第4次查找，从RecyclerdViewPool中查找，可以看到这里先使用getRecyclerViewPool获得Recycler中的RecyclerViewPool，然后调用RecyclerViewPool的getRecycledView(type)根据type获取一个ViewHolder，我们来看该方法的源码：
```java
 public ViewHolder getRecycledView(int viewType) {
    //根据type取出ScrapData
    final ScrapData scrapData = mScrap.get(viewType);
    if (scrapData != null && !scrapData.mScrapHeap.isEmpty()) {
        //取出ScrapData中的ViewHolder列表
        final ArrayList<ViewHolder> scrapHeap = scrapData.mScrapHeap;
        //返回一个ViewHolder并从pool中删除
        return scrapHeap.remove(scrapHeap.size() - 1);
    }
    return null;
}
```
mScrap是SparseArray类型，它会根据type把ViewHolder存放在不同ScrapData中，ScrapData中有一个mScrapHeap，是ArrayList类型，它会存放RecyclerViewPool中放进来的ViewHolder。所以上面这个方法首先会根据type取出ScrapData，然后取出mScrapHeap，如果mScrapHeap有元素，就返回并删除，然后重置这个ViewHolder让它复用，如果没有就进入下一步。

### 注释5：Adapter.createViewHolder()
既然缓存中没有就创建一个，该方法的相关源码如下：
```java
  public final VH createViewHolder(@NonNull ViewGroup parent, int viewType) {
        //...
        final VH holder = onCreateViewHolder(parent, viewType);
}
```
可以看到，调用了我们熟悉的onCreateViewHolder方法，该方法就是用来创建ViewHolder。

    ps：从上面知道当缓存中不能提供ViewHolder就会调用adapter的onCreateViewHolder创建一个，那么我们同样熟悉的OnBindViewHolder方法是什么时候执行的呢？bind方法是用来绑定数据，对于从mAttachedScrap和mCachedViews中拿出来的ViewHolder是不用重新bind的，而对于从mRecyclerPool拿出和通过Create方法创建的ViewHolder是需要重新bind的，所以前面说RecyclerViewPool才是真正用来回收复用ViewHolder的。

## 总结
本文中源码角度简单的分析RecyclerView布局一个itemView时是怎样通过Recycler来获取一个ViewHolder，从而获取itemView，如图：
{% asset_img rv4.png rv %}
准确的来说，Recycler是RecyclerView的itemView的提供者和管理者，它在内部封装了RecyclerView的缓存设计实现，在RecyclerView中有着四级缓存：AttachedScrap,mCacheViews,ViewCacheExtension,RecycledViewPool，正因为这样RecyclerView在使用的时候效率更好。

参考文章:

[RecyclerView和ListView原理](https://blog.csdn.net/feather_wch/article/details/81613313#观察者模式)










