---
title: Android8.0的Activity启动流程(2)
date: 2019-05-19 16:12:34
tags: 
- activity
- 源码
categories: 四大组件
---

## 前言
* 上一篇文章[Android8.0的Activity启动流程(1)](https://rain9155.github.io/2019/05/19/Android8.0%E7%9A%84Activity%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B1/)

上一篇文章讲了应用进程请求AMS启动Activity过程和Activity在AMS中的启动过程，然后Activity启动的代码逻辑就从AMS所在进程，又重新回到了应用进程所在的ApplicationThread中。我们还留下了一个问题，**Activity的生命周期方法是如何被回调的？**，下面我们就带着这个疑问，去走一遍源码，看一下在应用进程中ApplicationThread启动Activity的过程。

<!--more-->

	本文基于android8.0，本文相关源码文件位置如下：
	frameworks/base/core/java/android/app/Activity.java
	frameworks/base/core/java/android/app/ActivityThread.java
	frameworks/base/core/java/android/app/Instrumentation.java

## ApplicationThread::scheduleLaunchActivity()
上文结尾讲到在ActivityStackSupervisor的realStartActivityLocked()中调用了ApplicationThread中的scheduleLaunchActivity方法，这里是Activity启动的开始。ApplicationThread是ActivityThread的内部类，实现了IApplicationThread.stub接口。ActivityThread代表应用程序进程的主线程，它管理着当前应用程序进程的线程。

我们来看一下scheduleLaunchActivity的相关源码：

```java
//ActivityThread::ApplicationThread
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                int procState, Bundle state, PersistableBundle persistentState,
                List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                                          boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {
     		
    	   ActivityClientRecord r = new ActivityClientRecord();

            r.token = token;
            r.ident = ident;
            r.intent = intent;
            r.referrer = referrer;
            r.voiceInteractor = voiceInteractor;
            r.activityInfo = info;
            r.compatInfo = compatInfo;
            r.state = state;
            r.persistentState = persistentState;

            r.pendingResults = pendingResults;
            r.pendingIntents = pendingNewIntents;

            r.startsNotResumed = notResumed;
            r.isForward = isForward;

            r.profilerInfo = profilerInfo;

            r.overrideConfig = overrideConfig;
            updatePendingConfiguration(curConfig);

            sendMessage(H.LAUNCH_ACTIVITY, r);
 }
```

上述方法中只是简单的把从AMS传过来的有关启动Activity的参数，封装成ActivityClientRecord，然后调用sendMessage向H发送LAUNCH_ACTIVITY的消息，并且将ActivityClientRecord作为参数传了过去，H是ActivityThread中的内部类，是Handler类型，有关Activity的启动消息都交给这个Handler处理，为什么这里要进行切换到主线程处理消息呢？因为此时这里还运行在Binder的线程池中，不能进行Activity的启动，所以要切换到主线程中才能进行Activity的生命周期的方法回调。

下面我们来看看sendMessage方法。

### 1、ApplicationThread::sendMessage()

该方法如下：

```java
//ActivityThread::ApplicationThread
private void sendMessage(int what, Object obj) {
        sendMessage(what, obj, 0, 0, false);
}

 private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
        //...
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        if (async) {
            msg.setAsynchronous(true);
        }
        mH.sendMessage(msg);
 }
```

可以看到，sendMessage方法中将H.LAUNCH_ACTIVITY与ActivityClientRecord封装成一个Message，然后调用mH的sendMessage方法，mH就是H的实例，如下：

```java
final H mH = new H();
```

熟悉android消息机制的都知道(不了解的，可以看这一篇文章[Android消息机制java层](https://rain9155.github.io/2019/02/21/Android%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6java%E5%B1%82/))，Handler发送消息后，都会统一到handlerMessage方法中处理。

我们来看一下Handler H对消息的处理。

### 2、H::handleMessage（）

如下：

```java
//ActivityThread.java
private class H extends Handler {
     public static final int LAUNCH_ACTIVITY         = 100;
     public static final int PAUSE_ACTIVITY          = 101;
     public static final int RESUME_ACTIVITY         = 107;
     public static final int DESTROY_ACTIVITY        = 109;
     public static final int BIND_APPLICATION        = 110;
     //...
    public void handleMessage(Message msg) {
        switch (msg.what) {
                case LAUNCH_ACTIVITY: {
                    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                }
                break;
                //...
        }
    }
}
```

H中有很多关于四大组件的消息处理的字段，如Activity的启动，这里我们只关心前面发送过来的LAUNCH_ACTIVITY字段的消息处理，可以看到这里首先把msg中的obj字段转换为ActivityClientRecord，然后为ActivityClientRecord的packageInfo赋值，packageInfo是LoadedApk类型，它表示已加载的APK文件，接下来调用了外部类ActivityThread的handleLaunchActivity方法。

接下来我们来看一下ActivityThread的handleLaunchActivity方法。

## ActivityThread::handleLaunchActivity（）

该方法源码如下：

```java
//ActivityThread.java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
    //...
    //最终回调Activity的onConfigurationChanged方法
    handleConfigurationChanged(null, null);
    //这里面获取WindowManager系统服务的本地代理
    WindowManagerGlobal.initialize();
    //1、关注这里，启动Activity，调用了ActivityThread的performLaunchActivity方法，会最终回调Activity的onCreate，onStart方法
    Activity a = performLaunchActivity(r, customIntent);
    if (a != null) {
         //...
        //2、关注这里，调用了ActivityThread的handleResumeActivity方法，会最终回调Activity的onResume方法
         handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
		//...
    }else {
        //如果出错了，这里会告诉AMS停止启动Activity  
        try {
            ActivityManager.getService()
                .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                                Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
}
```

本文的重点是Activity的生命周期如何被回调，所以上面我们只需要关注注释1、2。注释1中调用了ActivityThread的performLaunchActivity方法，该方法最终完成了Activity对象的创建和启动过程，如果启动出错就会通知AMS停止启动Activity，并且在注释2中ActivityThread通过handleResumeActivity将被启动的Activity置为Resume状态。

我们首先看注释1的performLaunchActivity方法。

### 1、ActivityThread::performLaunchActivity()

该方法相关源码如下：

```java
//ActivityThread.java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    //从ActivityClientRecord中获取ActivityInfo。
    ActivityInfo aInfo = r.activityInfo;
    //获取packageInfo，packageInfo是前面讲到的LoadedApk类型
    if (r.packageInfo == null) {
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                                       Context.CONTEXT_INCLUDE_CODE);
    }
    //获取ComponentName
     ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }
     if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }
    //创建要启动Activity的上下文环境
    ContextImpl appContext = createBaseContextForActivity(r);
    //构造Activity对象，并设置参数
    Activity activity = null;
     try {
         	//获取类加载器
            java.lang.ClassLoader cl = appContext.getClassLoader();
         	//通过Instrumentation，用类加载创建该Activity实例
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            //设置相关参数准备初始化Activity
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
          //初始化Activity失败
          //....
      }
     try {
         //创建Application
         Application app = r.packageInfo.makeApplication(false, mInstrumentation);
         //...
         if (activity != null) {
              //构造Configuration对象
              Configuration config = new Configuration(mCompatConfiguration);
             //...
             //把该Activity和ContextImpl关联
             appContext.setOuterContext(activity);
             //通过attach方法将上述创建的信息保持到Activity内部，用来完成对Activity的初始化，如ContextImpl，Application，Configuration
             activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);
             //...
             //获取Activity的主题并设置
             int theme = r.activityInfo.getThemeResource();
             if (theme != 0) {
                    activity.setTheme(theme);
             }
              activity.mCalled = false;
             //1、根据是否需要持久化，调用Instrumentation的callActivityOnCreate方法通知Activity已经被创建，里面最终会调用Activity的onCreate方法
              if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                  //关注这里，走这个分支
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
              if (!activity.mCalled) {
                   //...
                  //无法调用Activity的onCreate方法，抛出异常
                }
              r.activity = activity;
              r.stopped = true;
              if (!r.activity.mFinished) {
                  //2、里面最终会调用Activity的onStart方法
                  activity.performStart();
                  r.stopped = false;
              }
              if (!r.activity.mFinished) {
                    //根据是否需要持久化，调用Instrumentation的callActivityOnRestoreInstanceState方法通知Activity已经被创建，里面最终会调用Activity的onRestoreInstanceState方法
                    if (r.isPersistable()) {
                        if (r.state != null || r.persistentState != null) {
                            mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                    r.persistentState);
                        }
                    } else if (r.state != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                    }
                }
             //...
         }
          r.paused = true;
         //把ActivityClientRecord缓存起来，以便在以后使用。mActivities的定义：ArrayMap<IBinder, ActivityClientRecord> mActivities = new ArrayMap<>();
		 mActivities.put(r.token, r);
     } catch (SuperNotCalledException e) {
          //...
     } catch (Exception e) {
          //...
          //抛异常，无法启动Activity
      }
    return activity;
}
```

该方法的主要代码都贴出来了，并做了注释，它主要做了以下事情：

* 1、从ActivityClientRecord中获取待启动的Activity的组件信息，如ActivityInfo，ComponentName。

ActivityInfo类用于存储代码中AndroidManifes设置的activity节点信息，ComponentName类中保存了该Activity的包名和类名

* 2、通过createBaseContextForActivity方法创建要启动Activity的上下文环境ContextImp，并在下面作为参数传进attach方法中。

ContextImp是Context的具体实现，Context中的大部分逻辑都是交给ContextImpl来完成，Context中定义了许多与四大组件启动、系统级服务获取、类加载、资源获取等有密切关系的方法，而Activity继承自ContextThemeWrapper，ContextThemeWrapper继承自ContextWrapper，ContextWrapper继承自Context，ContextWrapper内部有一个Context类型的mBase引用，而在Activity的attach方法中会调用attachBaseContext方法把该ContextImp赋值给mBase，所以Activity是ContextImpl的包装类，Activity扩展了Context中的方法。（这里就是一个[装饰者模式](https://blog.csdn.net/Rain_9155/article/details/89250729)）

* 3、通过LoadedApk的makeApplication方法创建Application。

makeApplication方法里面最终是通过Instrumentation的newApplication方法用类加载器创建Application，如果Application已经被创建过了，那么就不会重复创建，如果创建成功，会紧接着通过Instrumentation的callApplicationOnCreate来调用Application的onCreate方法。

* 4、通过Instrumentation的newActivity方法用类加载器创建Activity对象。

Instrumentation是一个用来监控应用程序与系统交互的类，通过它可以创建Activity、Applicationd实例，还与Activity生命周期的回调有关，所以在下文看到mInstrumentation.callActivityOnXX, 一般都是要回调某个Activity的生命周期方法。

* 5、通过Activity的attach方法来完成一些重要数据的初始化，如ContextImpl，Application，Configuration等。

在attach方法中会创建Window对象（PhoneWindow）并与Activity自身进行关联，这样当Window接收到外部的输入事件后就可以将事件传递给Activity。第2点讲过，还会把ContextImpl与Activity关联。

* 6、调用Instrumentation的callActivityOnCreate方法，里面最终会调用Activity的onCreate方法。

这里就是重点关注的注释1，注释还写到这里会根据是否需要持久化来调用不同参数的mInstrumentation的callActivityOnCreate方法，这个持久化是什么？其实这是在API 21后，Activity新增的一个”**persistableMode**“属性，在AndroidManifest.xml的activity节点将他它设为**android:persistableMode=”persistAcrossReboots**，Activity就有了持久化的能力，这时候我们可以数据保存在**outPersistentState**（Bundle类型），那么即使是关机，仍然可以恢复这些数据。关于PersistableMode更多信息可以看这篇文章[PersistableMode使Activity数据持久化保存](https://www.rainng.com/android-persistablemode/)，这不是本文的重点。

所以一般情况下我们没有使用这个属性，就会走到else分支，调用 mInstrumentation.callActivityOnCreate(activity, r.state)方法。

#### 1.1、Instrumentation::callActivityOnCreate()

该方法源码如下：

```java
 //Instrumentation.java
public void callActivityOnCreate(Activity activity, Bundle icicle) {
        prePerformCreate(activity);
    	//1、关注这里，调用了Activity的performCreate方法
        activity.performCreate(icicle);
        postPerformCreate(activity);
    }
```

我们看注释1，调用了activity的performCreate方法，见名知意。

##### 1.1.1、Activity::performCreate()

该方法源码如下：

```java
//Activity.java 
final void performCreate(Bundle icicle, PersistableBundle persistentState) {
        restoreHasCurrentPermissionRequest(icicle);
    	//1、看到我们的主角吧！onCreate方法
        onCreate(icicle, persistentState);
        mActivityTransitionState.readState(icicle);
        performCreateCommon();
    }
```

看注释1，调用了我们熟悉的onCreate方法，我们平时一般在这里面进行Activity的控件、资源等初始化操作。

下面继续回到ActivityThread的performLaunchActivity方法，接着上面的第6点。

* 7、调用Activity的performStart方法，里面最终会调用Activity的onStart方法。

这里也就是重点关注的注释2，下面看一下performStart方法。

#### 1.2、Activity::performStart()

我们继续点进去看一下：

```java
//Activity.java
final void performStart() {
    //...
    mCalled = false;
    //1、关注这里，调用了Instrumentation的callActivityOnStart方法
    mInstrumentation.callActivityOnStart(this);
    if (!mCalled) {
             //...
             //无法调用Activity的onStart方法，抛出异常
        }
    //...
}
```

看注释1，该方法还是一样的套路，调用 mInstrumentation.callActivityOnStart方法，我们看一下 Instrumentation的callActivityOnStart方法：

##### 1.2.1、Instrumentation::callActivityOnStart()

该方法如下：

```java
//Instrumentation.java
public void callActivityOnStart(Activity activity) {
    	//看到我们的主角吧！onStart方法
        activity.onStart();
    }
```

很简单的一句代码，调用了我们熟悉的onStart方法。

继续回到我们的ActivityThread的performLaunchActivity方法，还有一点没分析完，接下来到了根据需要调用Instrumentation的callActivityOnRestoreInstanceState方法，里面最终会调用Activity的onRestoreInstanceState方法，关于这个方法的作用已经不是本文的重点，但我们可以得出一个结论，onRestoreInstanceState方法的调用时机是在onStart方法之后。最后ActivityThread把ActivityClientRecord缓存起来。

分析完这个长长的方法，其实跟本文有关也就第6、7点。我们跳出ActivityThread::performLaunchActivity方法，回到ActivityThread的handleLaunchActivity方法。现在我们的Activity已经回调了onCreate和onStart方法，接下来应该是onResume方法。

下面我们我们接着来看handleLaunchActivity方法中注释2的handleResumeActivity方法。

### 2、ActivityThread::handleResumeActivity()

该方法源码如下：

```java
//ActivityThread.java
final void handleResumeActivity(IBinder token,
                                boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
    //从缓存中拿出ActivityClientRecord
    ActivityClientRecord r = mActivities.get(token);
    //...
    //1、主要关注这里，调用了performResumeActivity方法，这里最终会调用Activity的onResume方法
    r = performResumeActivity(token, clearHide, reason);
    
    //2、下面if（r ！= null）{}分支里面的逻辑都是把Activity显示出来
	if (r != null) {
        //拿到Activity
        final Activity a = r.activity;
        //...
        boolean willBeVisible = !a.mStartedActivity;
        if (!willBeVisible) {
                try {
                    willBeVisible = ActivityManager.getService().willActivityBeVisible(
                            a.getActivityToken());
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
         }
        if (r.window == null && !a.mFinished && willBeVisible) {
            //得到Activity关联的Window
             r.window = r.activity.getWindow();
            //得到Activity的DecorView，即Activity的顶级View
            View decor = r.window.getDecorView();
            //先把DecorView设为不可见
            decor.setVisibility(View.INVISIBLE);
            //得到ViewManager，用于添加DecorView
            ViewManager wm = a.getWindowManager();
            //得到布局参加
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            //下面设置布局参数
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            //...
             if (a.mVisibleFromClient) {
                 if (!a.mWindowAdded) {
                     a.mWindowAdded = true;
                     //用ViewManager添加DecorView
                     wm.addView(decor, l);
                 }
                 //...
             }
        }
        //...
        //此时用于承载DecorView的Window已经被WM添加了，但是还处于INVISIBLE状态,所以下面就把它值为VISIBLE
        if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
            //...
             if (r.activity.mVisibleFromClient) {
                 //把Activity显示出来
                 r.activity.makeVisible();
             }
        }
        
	}else {
        //...
        //Activity不能够Resume，通知AMS结束掉该Activity
    }
}
```

handleResumeActivity方法里面的代码也有点长，这个方法主要是把Activity置为Resume状态，并把Activity显示出来，所以我们只关注注释1，注释2是把Activity值为VISIBLE状态，大家要明白的是Activity其实也可以说是一个View，它的顶级View叫做DecorView，但系统回调完Activity的onResume函数时，只是说明Activity1已经完成所有的资源准备工作，Activity已经做好显示给用户的准备，所以还要通过类似于setVisible的方式把它显示出来，这个过程涉及到WindowManage的相关知识，为什么要这样做？大家可以看这篇文章[Window,WindowManager和WindowManagerService之间的关系](https://rain9155.github.io/2019/03/22/Window,%20WindowManager%E5%92%8CWindowManagerService%E4%B9%8B%E9%97%B4%E7%9A%84%E5%85%B3%E7%B3%BB/)了解一下，所以注释2不是本文重点就不讲了。

下面我们来看注释1的performResumeActivity方法。

#### 2.1 ActivityThread::performResumeActivity()

该方法主要源码如下：

```java
//ActivityThread.java
public final ActivityClientRecord performResumeActivity(IBinder token,
                                                       boolean clearHide, String reason) {
    //从缓存中拿到ActivityClientRecord
     ActivityClientRecord r = mActivities.get(token);
    if (r != null && !r.activity.mFinished) {
         if (clearHide) {
                r.hideForNow = false;
                r.activity.mStartedActivity = false;
          }
        //...
        try {
            //...
            //1、主要关注这里，调用了Activity的performResume方法
            r.activity.performResume();
            //...
             r.paused = false;
             r.stopped = false;
             r.state = null;
             r.persistentState = null;
        }catch(Exception e) {
             //...
            //抛异常，无法resume Activity
        }
    }
    return r;
}
```

主要关注注释1，调用了Activity的performResume，和上面的performCreate，performStart很相似。

下面我们来看Activity的performResume方法。

##### 2.1.1 Activity::performResume()

该方法主要源码如下：

```java
//Activity.java
final void performResume() {
    //这里处理Activity生命周期中的Restart流程
    performRestart();
    //...
	mCalled = false;
    //1、关注这里，调用Instrumentation的callActivityOnResume方法
    mInstrumentation.callActivityOnResume(this);
    if (!mCalled) {
       //...
        //抛异常，无法调用Activity的onResume方法
    }
}
```

在这个方法中我们看到了 performRestart()方法，这个是根据情况处理Restart流程，里面会执行onReStart() -> onStart() ，到这里就执行onResume()， 所以我们看到注释1会Instrumentation的callActivityOnResume方法，这个和上面的callActivityOnCreate()、callActivityOnStart（）类似。

本着执着的态度，我们还是看一下Instrumentation的callActivityOnResume方法。

###### 2.1.2、Instrumentation::callActivityOnResume()

```java
 //Instrumentation.java
public void callActivityOnResume(Activity activity) {
    activity.mResumed = true;
    //又看到了我们的主角之一，onResume方法
    activity.onResume();
    //...
}
```

该方法先把Activity的mResumed 赋值为true，然后回调了我们熟悉的onResume方法。

我们跳出ActivityThread的handleResumeActivity方法，回到handleLaunchActivity方法，至此handleLaunchActivity方法分析完，Activity已经显示到用户面前。

## 总结

到目前为止我们已经回调了Activity的三个生命周期方法：onCreate -> onStart -> onResume，onRestart也介绍了一下，可以说开头那个问题已经解解决了一半，我先来看一下本文的时序图：

{% asset_img activity1.jpg activity1 %}

所以现在我们知道了在应用进程中ApplicationThread启动Activity的过程。

那么还有三个方法：onPause -> onStop -> onDestory 什么时候被回调呢？大家都知道Activity有7个生命周期方法，除去onRestart，其他3个都是一 一对应的，结合前面那篇文章[Android8.0的Activity启动流程(1)](https://rain9155.github.io/2019/05/19/Android8.0%E7%9A%84Activity%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B1/)我们知道：

* 1、在AMS中含有ApplicatiThread的本地代理，所以AMS所在进程可以通过这个代理与ActivityThread的主线程通信，也就能调用ApplicatiThread的一些方法。

* 2、在应用进程中也含有系统服务AMS的本地代理对象，所以应用进程可以通过这个代理与AMS通信，可以请求AMS启动一个Activity。

* 3、双方都含有双方的代理，通过Binder，也就建立起双方的通信通道。

每个应用都有自己专属Activity任务栈，Activity任务栈的管理是在AMS那边，在本文的情况下，一个Activity已经被启动了，该Activity被加入到栈顶中去，如果此时我按back键返回上一个Activity，那么该Activity就会调用相应的回调onPause -> onStop -> onDestory方法，这个过程在AMS那边对应一个出栈动作，此时AMS也就像启动Activity调用scheduleLaunchActivity方法那样调用ApplicationThread中schedulePauseActivity、scheduleStopActivity、scheduleDestroyActivity方法来结束掉这个Activity，这个调用过程是IPC，所以大家通过本文举一反三也就明白了Activity的其他生命周期是如何被回调的，这个过程离不开与AMS的交互。

至此我们已经走完startActivity后发生的流程。在这整个过程中也发现了自己平常很多遗落的知识点，让我更进一步的认识了Activity。希望大家也有所收获。

参考资料：

[Activity生命周期回调是如何被回调的](https://www.jianshu.com/p/91984327690e)

[Android8.0 根Activity启动过程（后篇）](http://liuwangshu.cn/framework/component/7-activity-start-2.html)

《Android源码分析与实战》