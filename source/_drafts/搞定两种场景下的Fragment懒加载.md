---
title: 搞定两种场景下的Fragment懒加载
tags: fragment
categories: fragment
---

## 前言

我对懒加载的定义是：数据的加载要等到页面对用户可见时才加载，否则的话会浪费用户流量。网上实现懒加载的方案非常多，但大多数都是解决了我下面说到的场景一的懒加载，本文还解决场景二的懒加载方式。

如果不想看下面的分析，直接这个类导入你的项目中，需要懒加载的Fragment继承这个类，并重写相应的方法就行：[传送门](https://github.com/rain9155/lazyLoagingFragment/tree/master/app/src/main/java/com/example/lazyloading/fragment)。

## 场景一： Viewpager + Tablayout + Fragment

什么？不会用Viewpager，可以看一下这个入门系列：[ViewPager 详解（一）---基本入门](https://blog.csdn.net/harvic880925/article/details/38453725)。

场景一应该是很多人都遇到过的情况，界面整体使用Viewpager + Tablayout + Fragment组合，左右滑动界面以展示数据给用户，当你滑动到下一页的时候，Fragment已经有数据了，但是这个时候我希望开始加载数据而不是已经有了数据，特别是Viewpager 的适配器使用**FragmentPagerAdapter**的时候，因为这个适配器它会预加载好相邻的Fragment页面，这个预加载数量可以通过如下设置：

```java
viewPager.setOffscreenPageLimit(0);
```

那么上面这句代码不是把预加载数量设置为0了吗？这样Fragment就不会预先加载了，这样想你就太天真，通过看setOffscreenPageLimit的源码得知，如果你传入的数值小于1，那么ViewPager就会把预加载数量设置成默认值，而默认值就是1，所以说就算你传入了0，ViewPager还是会预先加载好当前页面的左右两个Fragment页面。

### 懒加载原理

那么怎么解决呢？这时要认识Fragment中的一个函数：**setUserVisibleHint(boolean isVisibleToUser)**：

setUserVisibleHint方法是Fragment中的一个回调函数。当前Fragment可见对用户可见时，setUserVisibleHint()回调，其中参数isVisibleToUser=true，当前Fragment由可见到不可见或实例化时，setUserVisibleHint()回调，其中参数isVisibleToUser=false。

下面看一下这个方法在Fragment生命周期中的调用时机：

* 1、当Fragment被实例化时，即Fragment被装载进ViewPager适配器中，并：setUserVisibleHint() ->onAttach()  -> onCreate() -> onCreateView() -> onViewCreated() -> onActivityCreate()  ->  onStart()   -> onResume()。此时setUserVisibleHint() 中的参数为false。
* 2、在Fragmente可见时，即ViewPager滑动到当前页面时：setUserVisibleHint()。只会调用setUserVisibleHint方法，因为已经预加载过了，Fragment在之前生命周期已经走到onResume() 了。此时setUserVisibleHint() 中的参数为true。
* 3、在Fragment由可见变为不可见，即ViewPager由当前页面滑动到另一个页面：setUserVisibleHint()。只会调用setUserVisibleHint方法，因为还要保持当前页面的预加载过程，此时setUserVisibleHint() 中的参数为false。
* 4、点击由TabLayout直接跳转到一个未预加载的页面，此时生命周期的回调过程：setUserVisibleHint() -> setUserVisibleHint() -> onAttach()  -> onCreate() -> onCreateView() -> onViewCreated() -> onActivityCreate()  -> onStart() -> onResume()。回调了两次setUserVisibleHint() ，一次代表初始化时，传入参数是false，一次代表可见时，传入参数是true。

可以看到此时setUserVisibleHint的调用时机总是在**初始化时调用，可见时调用，由可见转换成不可见时调用。**

### 实现思路

下面讲讲场景一的懒加载实现思路：我们一般在Fragment的onActivityCreated中加载数据，这个时候我们可以判断此时的Fragment是否对用户可见，调用fragment.getUserVisibleHint()可以获得isVisibleToUser的值，如果为true，表示可见，就加载数据，如果不可见，就不加载数据了，代码如下：

```java
  @Override
    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        if(isFragmentVisible(this) && this.isAdded()){
            if (this.getParentFragment() == null || isFragmentVisible(this.getParentFragment())) {
                onLazyLoadData();
                isLoadData = true;
                //...
            }
        }
    }
```

判读Fragment是否对用户可见封装在isFragmentVisible方法中， onLazyLoadData()是子类需要重写的方法，用来加载数据，加载完数据后把isLoadData设置为true，表示已经加载过数据。

上面就控制了当Fragment不可见时就不加载数据，而且此时Fragment的生命周期也走到onResume了，那么当我滑到这个Fragment时，只会调用它的setUserVisibleHint方法，那么就要在setUserVisibleHint方法中加载数据，代码如下：

```java
   @Override
    public void setUserVisibleHint(boolean isVisibleToUser) {
        super.setUserVisibleHint(isVisibleToUser);
        if(isFragmentVisible(this) && !isLoadData && isViewCreated && this.isAdded()){
            onLazyLoadData();
            isLoadData = true;
        }
    }
```

isViewCreated字段表示布局是否被初始化，它在onViewCreated方法中被赋值为true，如下：

```java
 @Override
    public void onViewCreated(@NonNull View view, @Nullable Bundle savedInstanceState) {
        isViewCreated = true;
    }

```

onViewCreated方法的回调在onCreateView方法后，当调用onViewCreated方法时，Fragment的View布局一定创建好了。

我们再回到setUserVisibleHint方法中，在if中它会依此判断当前Fragment可见、还没有加载数据、布局已经创建好等这些条件满足后才加载数据，并把isLoadData赋值为true。

### 应用示例

下面是我在项目中使用的情况：

{% asset_img fragment1.gif fragment1 %}

可以看到，当我滑倒这个Fragment时才加载数据。

## 场景二：FragmentManager + FragmentTransaction+ Fragment

这个场景就是你把几个Fragment通过FragmentTransaction的add方法add到FragmentManager 中，切换Fragment的时候通过FragmentTransaction的hide和show方法配合使用，类似于微信的主界面，底部有一个tab，然后点击tab，切换页面。

当Fragment被add进manager中时，Fragment生命周期已经执行到onResume了，所以在后续的hide和show方法切换Fragment时，Fragment已经有数据了，在我的项目中，我想要的效果是，当我点到这个tab时，该tab对于的Fragment才加载数据，所以我对这种情况实现了懒加载。

### 懒加载原理

那么要怎么实现呢？照搬场景2的实现方式？可惜了，不行，因为这种情况下setUserVisibleHint方法不会被调用。这个时候我们又重新认识一个方法**onHiddenChanged（boolean hidden）**：

onHiddenChanged方法是当Fragment的隐藏状态变化示被调用，当Fragment没有被隐藏时即调用show方法，当前onHiddenChanged回调，其中参数hidde=false，当Fragment被隐藏时即调用hide了方法，onHiddenChanged()回调，其中参数hidde=true。还有一点注意的是使用hide和show时，fragment的所有生命周期方法都不会调用，除了onHiddenChanged（）。

下面看一下这个方法在Fragment生命周期中的调用时机：

* 1、当Fragment被add进manager时：onAttach()  -> onCreate() -> onCreateView() -> onViewCreated() -> onActivityCreate()  ->  onHiddenChanged() -> onStart()   -> onResume()。此时onHiddenChanged() 中的参数为false。
* 2、当用hide方法隐藏Fragment时：onHiddenChanged()，只会调用onHiddenChanged方法，此时setUserVisibleHint() 中的参数为true。
* 3、当用show方法显示Fragment时：onHiddenChanged()，只会调用onHiddenChanged方法，此时setUserVisibleHint() 中的参数为false。

可以看到此时onHiddenChanged的调用时机总是在**初始化时调用，hide时调用，show时调用。**

### 实现思路

场景二是在setUserVisibleHint方法中做文章，而这次是在onHiddenChanged方法中做文章，如下：

```java
  @Override
    public void onHiddenChanged(boolean hidden) {
        super.onHiddenChanged(hidden);
        //1、onHiddenChanged调用在Resumed之前，所以此时可能fragment被add, 但还没resumed
        if(!hidden && !this.isResumed())
            return;
        //2、使用hide和show时，fragment的所有生命周期方法都不会调用，除了onHiddenChanged（）
        if(!hidden && isFirstVisible && this.isAdded()){
            onLazyLoadData();
            isFirstVisible = false;
        }
    }
```

首先看注释1，因为当add的时候，onHiddenChanged调用在onResumed之前，此时还没有执行onResume方法，用户还看不见这个Fragment，如果此时加载数据就没有什么用，等于用户看到这个Fragmen时它就已经执行完数据了，如果这里要加一个判断，如果Fragment还没有Resume，就直接return，不做操作。

接下来看注释2，执行到注释2表示此时Fragment已经可见了，就可以通过hidden字段控制懒加载，hidden为false表示调用了show方法，通过isFirstVisible控制只加载一次，为什么要用isFirstVisible呢，因为在onActivityCreate方法中就有可能已经加载过数据，如果加载过就不用再加载了，在onActivityCreate中会把这个字段赋值为true，如下：

```java
  @Override
    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        if(isFragmentVisible(this) && this.isAdded()){
            if (this.getParentFragment() == null || isFragmentVisible(this.getParentFragment())) {
                onLazyLoadData();
                isLoadData = true;
                if(isFirstVisible)
                    isFirstVisible = false;
            }
        }
    }
```

### 应用示例

下面是我在项目中使用的情况：

{% asset_img fragment2.gif fragment2 %}

可以看到，当我点击到这个tab时，对应的Fragment才加载数据。

## 结语

以上就是我的懒加载历程，虽然现在也有一些Fragment库可以实现这个效果，但是它的原理也是这个，我们要知其所以然，还有该懒加载类已经用到了我的[小项目](https://github.com/rain9155/WanAndroid)中，该[懒加载类](https://github.com/rain9155/lazyLoagingFragment/tree/master/app/src/main/java/com/example/lazyloading/fragment)整合场景一和场景二，只有简单的几句代码，只要继承就能在两种场景下使用。

参考文章：

[Fragment 知识梳理(3)](https://www.jianshu.com/p/354fbb20ffe3)

[FragmentPagerAdapter与FragmentStatePagerAdapter区别](https://blog.csdn.net/u013588712/article/details/52145217)