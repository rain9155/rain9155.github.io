---
title: Activity的启动流程（1）
date: 2019-05-19 16:11:17
tags: activity
categories: 四大组件
---

## 前言
Activity的启动流程有俩种过程，一种是根Activity的启动过程，即在Launch界面点击一个应用图标启动应用程序，根Activity指的是应用程序启动的第一个Activity；另一种是普通Activity的启动流程，即我们平时调用startActivity方法来启动一个Activity。本文讨论第二种，startActivity方法大家都知道是用来启动一个Activity的，那么大家有没有想过它在底层是怎么启动的呢？Activity的生命周期方法是如何被回调的？它启动过程中涉及到多少个进程？接下来我们通过撸一篇源码来了解Activity的大概启动流程，然后解答这几个问题。

> 本文源码基于Android8.0，本文涉及的源码文件位置如下：
> frameworks/base/core/java/android/app/Activity.java
> frameworks/base/services/core/java/com/android/server/am/*.java(*代表ActivityManagerService，ActivityStack，ActivityStarter，ActivityStackSupervisor，ActivityStack)

## Activity::startActivity()

startActivity有好几种重载方法，如下：

```java
@Override
    public void startActivity(Intent intent) {
        this.startActivity(intent, null);
    }

@Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            //我们一般没有传options参数给startActivity，所以options为空，就会走到这个分支
            //第二参数requestCode为-1，表示不需要知道Activity启动的结果
            startActivityForResult(intent, -1);
        }
    }

 //发现两个参数的startActivityForResult方法最终还是调用三个参数的startActivityForResult方法，options参数传入null
 public void startActivityForResult(@RequiresPermission Intent intent, int requestCode) {
        startActivityForResult(intent, requestCode, null);
  }

```

可以发现startActivity最终都会调用到startActivityForResult方法。

### 1、Activity::startActivityForResult()

这里我们来到了具有三个参数的startActivityForResult方法，如下：

```java
//Activity.java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode, @Nullable Bundle options) {
    	//mParent一直为空
        if (mParent == null) {
            //...
            //1、关注这里，调用Instrumentation的execStartActivity方法
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            //...
            //此时requestCode为-1
            if (requestCode >= 0) {
                mStartedActivity = true;
            }
            //...
        } else {
          //...
    }
```

在上面的的代码中，会进入mParent==null的这个分支里，mParent是Activity类型，它只会在LocalActivityManger构造中被赋值，在我们startActivity过程中一直为空（关于为甚么mParent一直为空，可以查看这篇文章[StartActivity路上的mParent](https://www.jianshu.com/p/3141d2c0194c)）。这里我们只关注注释1，调用Instrumentation的execStartActivity方法，Instrumentation是一个用来监控应用程序与系统交互的类，我们还要注意传入execStartActivity方法的两个参数：

* 1、mMainThread.getApplicationThread()：ApplicationThread类型，mMainThread是ActivityThread类型，它是应用程序的入口类，而mMainThread.getApplicationThread()就是获得一个**ApplicationThread**，它是ActivityThread的内部类，它实现了IApplicationThread.Stub，如下：

```java
//ActivityThread.java::ApplicationThread
private class ApplicationThread extends IApplicationThread.Stub {
    //...
}
```

IApplicationThread.java类是在编译时由IApplicationThread.aidl通过AIDL工具自动生成的，IApplicationThread的内部会自动生成一个 IApplicationThread.Stub类，它继承自Binder类，而Binder实现了IBinder接口，并且 IApplicationThread.Stub实现了IActivityManager接口。要想进行进程间通信，ApplicationThread只需要继承IApplicationThread.Stub类并实现相应的方法就可以，这样主线程ActivityThread就可以通过ApplicationThread就能对外提供远程服务。要记住这个**ApplicationThread**，他在Activity的启动过程中发挥重要作用。

* 2、mToken： 它的类型为IBinder，代表着当前Activity的token，它保存自己所处Activity的ActivityRecord信息

### 2、Instrumentation::execStartActivity()

我们继续看Instrumentation的execStartActivity方法，如下：

```java
//Instrumentation.java
public ActivityResult execStartActivity(Context who, IBinder contextThread, IBinder token, Activity target, Intent intent, int requestCode, Bundle options) {
    //还记得上面提到的ApplicationThread吗，这里把它转成了IApplicationThread，并在下面作为startActivity方法的参数
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    //...
    try {
        //...
        //1、关注这里，这里实际调用的是ActivityManagerService的startActivity方法
        int result = ActivityManager.getService()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                           intent.resolveTypeIfNeeded(who.getContentResolver()),
                           token, target != null ? target.mEmbeddedID : null,
                           requestCode, 0, null, options);
        //检查启动Activity的结果，无法正确启动一个Activiy时，这个方法抛出异常
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

我们看注释1，ActivityManager.getService()返回的是ActivityManagerService（下面简称AMS）在应用进程的本地代理，该方法在ActivityManager中，如下：

```java
//ActivityManager.java
public static IActivityManager getService() {
    	//IActivityManagerSingleton是Singleton类型，Singleton是一个单例的封装类
    	//第一次调用它的get方法时它会通过create方法来初始化AMS这个Binder对象，在后续调用中返回之前创建的对象
        return IActivityManagerSingleton.get();
}

 
  private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    //ServiceManager是服务大管家，这里通过getService获取到了IBinder类型的AMS引用
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    //这通过asInterface方法把IBinder类型的AMS引用转换成AMS在应用进程的本地代理
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
                }
  };
```

上面出现的IActivityManager.java类的意义类似于前面提到的ApplicationThread.java。要想进行进程间通信，AMS只需要继承IActivityManager.Stub类并实现相应的方法就可以，这样AMS就能对外提供远程服务，如下：

```java
//ActivityManagerService.java
public class ActivityManagerService extends IActivityManager.Stub{
    //...
}
```

所以继续回到Instrumentation的execStartActivity方法中，ActivityManager.getService()返回的是AMS的本地代理，注意AMS是在系统进程SystemServer中，所以注释1这里通过**Binder的IPC**，调用的其实是AMS的startActivity方法。

**在这里开始，Activity的启动过程从应用进程转移到AMS中去**。

## AMS::startActivity()

AMS的startActivity方法如下：

```java
   //AMS.java
    public final int startActivity(IApplicationThread caller, String callingPackage, Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }
```

简单的return了startActivityAsUser方法，该方法在最后多了个 UserHandle.getCallingUserId()参数，AMS根据这个确定调用者权限。我们再来看看其他参数：

* caller：IApplicationThread类型，还记得上面提到的ApplicationThread吗？到这里它已经被转成了ApplicationThread的本地代理，这个转换的过程发生在上面讲到的Binder的IPC中，就像上面提到的AMS本地代理转换一样。
* callingPackage：前面一直传过来的，代表调用者Activity所在的包名
* intent：前面startActivity时传递过来的intent
* resolvedType：从上面传过来，intent.resolveTypeIfNeeded()
* resultTo：IBinder类型，还记得上面提到的mToken吗？就是从上面一直传过来的，保存着的调用者Activity的ActivityRecord信息
* resultWho：String类型，调用者Activity的mEmbeddedID，前面一直传过来的
* requestCode：从上面一直传过来的，一直为-1
* startFlags：从上面传过来，为0
* profilerInfo：ProfilerInfo类型，从上面传过来，等于null
* bOptions：Bundle类型，从上面传过来，等于null

下面继续看AMS的startActivityAsUser方法。

### 1、AMS::startActivityAsUser()

startActivityAsUser方法如下：

```java
   //AMS.java
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage, Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
        //...
        return mActivityStarter.startActivityMayWait(caller, -1, callingPackage, intent,
                resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
                profilerInfo, null, null, bOptions, false, userId, null, null,
                "startActivityAsUser");
    }

```

省略了两个判断，1、判断调用者进程是否被隔离，2、判断调用者是否有权限，这些都不是重点。下面继续简单的return了mActivityStarter.startActivityMayWait方法，mActivityStarter是ActivityStarter类型，它是AMS中加载Activity的控制类，会收集所有的逻辑来决定如何将Intent和Flags转换为Activity，并将Activity和Task以及Stack相关联。传入startActivityMayWait方法的参数又多了几个，看一下几个：

* callingUid：第二个参数，等于-1
* inTask：倒数第二个参数，TaskRecord类型，代表要启动的Activity所在的栈，这里为null，表示还没创建
* reason：倒数第一个参数，值为"startActivityAsUser"，代表启动的理由
* 其他的参数有一些传入null，有一些是从上面传过来的

下面看ActivityStarter中的startActivityMayWait方法。

###  2、ActivityStarter::startActivityMayWait()

来看看这个方法的源码，如下：

```java
//ActivityStarter.java
final int startActivityMayWait(IApplicationThread caller, int callingUid, String callingPackage, Intent intent, String resolvedType, IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, IBinder resultTo, String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo, WaitResult outResult, Configuration globalConfig, Bundle bOptions, boolean ignoreTargetSecurity, int userId, IActivityContainer iContainer, TaskRecord inTask, String reason) {
    //...
    //把上面传进来的intent再构造一个新的Intent对象，这样即便intent被修改也不受影响
    intent = new Intent(intent);
    //...
    ResolveInfo rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId);
    if(rInfo == null){
        //...
    }
    //解析这个intent，收集intent指向的Activity信息
    ActivityInfo aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);
    //...
    final ActivityRecord[] outRecord = new ActivityRecord[1];
    //1、主要关注这里，调用了本身的startActivityLocked方法
    int res = startActivityLocked(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid, callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options, ignoreTargetSecurity, componentSpecified, outRecord, container, inTask, reason);
    //...
    return res;
}
```

ActivityInfo里面收集了要启动的Activity信息（关于ResolveInfo与ActivityInfo可以看这篇[如何获取Android应用与系统信息](https://blog.csdn.net/Rain_9155/article/details/89286415)），主要还是关注注释1，这里又调用了ActivityStarter中的startActivityLocked方法。传入startActivityLocked方法的参数又多了几个（callingPid等）。关于pid于与uid的介绍可以看这篇文章[Android手机中UID、PID作用及区别](https://blog.csdn.net/jiaoli_82/article/details/49802613)。

下面来看一下startActivityLocked方法。

#### 2.1、ActivityStarter::startActivityLocked()

该方法的相关源码如下：

```java
//ActivityStarter.java
int startActivityLocked(IApplicationThread caller, Intent intent, Intent ephemeralIntent, String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo, IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid, String callingPackage, int realCallingPid, int realCallingUid, int startFlags, ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
                          ActivityRecord[] outActivity, ActivityStackSupervisor.ActivityContainer container, TaskRecord inTask, String reason) {
         //这里对上面传进来值为"startActivityAsUser"理由参数判空
         if (TextUtils.isEmpty(reason)) {
            throw new IllegalArgumentException("Need to specify a reason.");
        }
       mLastStartReason = reason;
       mLastStartActivityTimeMs = System.currentTimeMillis();
       mLastStartActivityRecord[0] = null;
      //1、主要关注这里，调用了本身的startActivity方法
       mLastStartActivityResult = startActivity(caller, intent, ephemeralIntent, resolvedType,
                aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode,
                callingPid, callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                options, ignoreTargetSecurity, componentSpecified, mLastStartActivityRecord,
                container, inTask);
      if (outActivity != null) {
            outActivity[0] = mLastStartActivityRecord[0];
        }
      return mLastStartActivityResult;
  }
```

这里主要关注注释1，调用了ActivityStarter中的startActivity方法，该方法多了一个参数，最后一个mLastStartActivityRecord，mLastStartActivityRecord是一个ActivityRecord数组类型，ActivityRecord是用来保存一个Activity的所有信息的类。

下面来看ActivityStarter中的startActivity方法。

####  2.2、ActivityStarter::startActivity()

```java
 //ActivityStarter.java
private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent, String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo, IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid, String callingPackage, int realCallingPid, int realCallingUid, int startFlags, ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
ActivityRecord[] outActivity, ActivityStackSupervisor.ActivityContainer container, TaskRecord inTask) {
     int err = ActivityManager.START_SUCCESS;
     //...
     //获取调用者所在进程记录的对象，caller就是上面一直强调的代表调用者进程的ApplicationThread对象
     ProcessRecord callerApp = null;
     if (caller != null) {
         //这里调用AMS的getRecordForAppLocked方法获得代表调用者进程的callerApp
         callerApp = mService.getRecordForAppLocked(caller);
         if (callerApp != null) {
             //获取调用者进程的pid与uid并赋值
             callingPid = callerApp.pid;
             callingUid = callerApp.info.uid;
         } else {
             err = ActivityManager.START_PERMISSION_DENIED;
         }
     }
     //下面startActivity方法的参数之一，代表调用者Activity的信息
     ActivityRecord sourceRecord = null;
     if (resultTo != null) {
            sourceRecord = mSupervisor.isInAnyStackLocked(resultTo);
         	//...
     }
     //...
     //创建即将要启动的Activity的信息描述类ActivityRecord
      ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
                callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
                resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,
                mSupervisor, container, options, sourceRecord);
     //outActivity是ActivityRecord[]类型，从上面传进来，这里把ActivityRecord赋值给了它，下面会作为参数传进startActivity方法中
     if (outActivity != null) {
         outActivity[0] = r;
     }
     //...
     //1、关注这里，调用了本身的另一个startActivity方法
	return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags, true,
                options, inTask, outActivity);
 }
```

上面的startActivity代码非常长，省略了很多，上面讲的调用者进程，在这里等价于应用程序进程，ProcessRecord是用来描述一个应用进程的信息，ActivityRecord上面也讲过了，就是用来记录一个要启动的Activity的所有信息，在注释1处的调用了ActivityStarter的startActivity方法，这个方法参数少了很多，大多数有关要启动的Activity的信息都被封装进了ActivityRecord类中，作为参数r传了进去。

下面来看ActivityStarter的startActivity方法。

#### 2.3、ActivityStarter::startActivity()

该方法代码如下：

```java
 //ActivityStarter.java
private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
                           IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask, ActivityRecord[] outActivity) {
     int result = START_CANCELED;
     try {
            mService.mWindowManager.deferSurfaceLayout();
         	//1、主要关注这里，调用本身的startActivityUnchecked方法
            result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, outActivity);
          } finally {
           	//...
           mService.mWindowManager.continueSurfaceLayout();
     	}
     //...
	return result;
 }
```

这里主要调用了ActivityStarter的startActivityUnchecked方法。

#### 2.4、ActivityStarter::startActivityUnchecked()

该方法代码如下：

```java
//ActivityStarter.java
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask, ActivityRecord[] outActivity) {
    //把上面传进来的参数除了outActivity都传进去了，主要是把这些参数赋值给ActivityStarter的成员变量，如mDoResume = doResume, mStartActivity = r
    //mStartActivity就是即将要启动的Activity的信息
     setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
                voiceInteractor);
    //计算出启动Activity的模式，并赋值给mLaunchFlags
    computeLaunchingTaskFlags();
    //...
    //设置启动模式
     mIntent.setFlags(mLaunchFlags);
    //...
    boolean newTask = false;
    //1、下面会进行判断，到底需不需要创建一个新的Activity任务栈
    int result = START_SUCCESS;
    if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
        && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
        //1.1走这里就会在setTaskFromReuseOrCreateNewTask方法内部创建一个新的Activity任务栈
        newTask = true;
        result = setTaskFromReuseOrCreateNewTask(
            taskToAffiliate, preferredLaunchStackId, topStack);
    } else if (mSourceRecord != null) {
        //1.2走这里就会在setTaskFromSourceRecord方法内部获得调用者Activity的的任务栈赋值给mTargetStack
        result = setTaskFromSourceRecord();
    } else if (mInTask != null) {
        //1.3走这里就会在setTaskFromInTask方法内部直接把mInTask赋值给mTargetStack，前面已经说过mInTask等于null
        result = setTaskFromInTask();
    } else {
        //1.4、就是前面的条件都不满足了，但是这种情况很少发生
        setTaskToCurrentTopOrCreateNewTask();
    }
    if (result != START_SUCCESS) {
        return result;
    }
    //...
    //mDoResume等于上面传进来的doResume，为true
     if (mDoResume) {
            final ActivityRecord topTaskActivity =
                    mStartActivity.getTask().topRunningActivityLocked();
            if (!mTargetStack.isFocusable()
                    || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                    && mStartActivity != topTaskActivity)) {
              //走这里不会显示Activity，因为Activity还没有获取焦点或者Activity的栈溢出
              //...
            } else {
              //正常的话会走到这里
                if (mTargetStack.isFocusable() && !mSupervisor.isFocusedStack(mTargetStack)) {
                    mTargetStack.moveToFront("startActivityUnchecked");
                }
                //2、主要关注这里调用mSupervisor的resumeFocusedStackTopActivityLocked方法
                mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                        mOptions);
            }
        } else {
           //...
        }
     return START_SUCCESS;
}
```

上面的startActivityUnchecked方法也是很长，这个方法主要处理Activity栈管理相关的逻辑，如果对于这方面信息不熟悉的话可以查看这两篇文章[Android任务和返回栈完全解析](https://blog.csdn.net/guolin_blog/article/details/41087993)、[ActivityTask和Activity栈管理](http://liuwangshu.cn/framework/ams/2-activitytask.html)。一个或多个ActivityRecord会组成一个TaskRecord，TaskRecord用来记录Activity的栈，而ActivityStack包含了一个或多个TaskRecord。上面代码的mTargetStack就是ActivityStack类型，我们先来看注释1，注释1会根据mLaunchFlags等条件到底需不需要创建一个新的Activity任务栈，而本文所讨论的条件限定在从一个应用程序调用Activity的startActivity去启动另外一个Activity的情景，而且默认Activity的启动模式是standard，并不会创建一个新的任务栈，所以就会走到1.2的条件分支，然后我们再来看注释2，这里会调用mSupervisor.resumeFocusedStackTopActivityLocked方法，mSupervisor是ActivityStackSupervisor类型，ActivityStackSupervisor主要用来管理ActivityStack。启动Activity的过程从ActivityStack来到了ActivityStackSupervisor。

下面我们来看ActivityStackSupervisor的resumeFocusedStackTopActivityLocked方法。

### 3、ActivityStackSupervisor::resumeFocusedStackTopActivityLocked()

该方法源码如下：

```java
//ActivityStackSupervisor.java
boolean resumeFocusedStackTopActivityLocked(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
       	//...
    	//获取要启动的Activity所在栈的栈顶的ActivityRecord
        final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
    	//1、r是否null或是否为RESUMED状态
        if (r == null || r.state != RESUMED) {
            //2、关注这里，调用ActivityStack的resumeTopActivityUncheckedLocked方法
            mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
        } else if (r.state == RESUMED) {
            mFocusedStack.executeAppTransition(targetOptions);
        }
        return false;
    }
```

首先这里会获取要启动的Activity所在栈的栈顶的ActivityRecord赋值给r，因为要启动的Activity的还没有启动，所以此时栈顶就是调用者Activity，调用者Activity启动Activity，肯定会从RESUME状态转到其他状态如STPO，所以注释1满足r.state != RESUMED的条件，此时就是走带注释2，注释2调用了mFocusedStack的resumeTopActivityUncheckedLocked方法，mFocusedStack就是ActivityStack类型。启动Activity的过程从ActivityStackSupervisor又回到到了ActivityStack。

下面我们来看ActivityStack的resumeTopActivityUncheckedLocked方法。

####  3.1、ActivityStack:: resumeTopActivityUncheckedLocked()

该方法的源码如下：

```java
//ActivityStack.java
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
       //...
        boolean result = false;
        try {
            //1、关注这里，这里调用了本身的resumeTopActivityInnerLocked方法
            result = resumeTopActivityInnerLocked(prev, options);
        } finally {
           //...
        }
        //...
        return result;
    }
```

我们来看注释1，简单的调用了resumeTopActivityInnerLocked方法。

#####  3.1.1、 ActivityStack:: resumeTopActivityInnerLocked()

该方法源码如下：

```java
//ActivityStack.java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
        //...
    	//获得将要启动的Activity的信息
        final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */)
        //...
        if (next.app != null && next.app.thread != null) {
            //...
        }else{
            //...
            //1、关注这里，调用了ActivityStackSupervisor的startSpecificActivityLocked方法
            mStackSupervisor.startSpecificActivityLocked(next, true, true);
        }
    	if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
        return true;
}
```

topRunningActivityLocked方法获得将要启动的Activity的信息next，因为此时要启动的Activity还不属于任何进程，故它的ProcessRecord为空成立，就会走到else分支，所以注释1这里调用了ActivityStackSupervisor的startSpecificActivityLocked方法，又回到了ActivityStackSupervisor中。

下面来看ActivityStackSupervisor的startSpecificActivityLocked方法。

#### 3.2、ActivityStackSupervisor::startSpecificActivityLocked()

该方法源码如下：

```java
//ActivityStackSupervisor.java
void startSpecificActivityLocked(ActivityRecord r, boolean andResume, boolean checkConfig) {
        //获取要启动的Activity的所在应用程序进程
        ProcessRecord app = mService.getProcessRecordLocked(r.processName, 													r.info.applicationInfo.uid, true);
        r.getStack().setLaunchTime(r);
        //要启动的Activity的所在应用程序进程存在
        if (app != null && app.thread != null) {
            try {
                //...
                //1、关注这里，调用了本身的realStartActivityLocked方法
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                //...
            }
        }
    	//...
    }
```

这里首先会获取要启动的Activity所在的应用进程app，当app进程已经运行时，就会调用注释1处的realStartActivityLocked方法，注意这里多了一个参数，把代表应用进程的app传了进去。

下面来看ActivityStackSupervisor的realStartActivityLocked方法。

####  3.3、ActivityStackSupervisor::realStartActivityLocked()

```java
 final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app, boolean andResume, boolean checkConfig) throws RemoteException {
     //...
     //1、把应用所在进程信息赋值给要启动的Activity的ActivityRecord
     r.app = app;
     //...
     try{
         //...
         //2、关注这里，app是ProcessRecord类型，app.thread是IApplicationThread类型
         //app.thread是应用进程的ApplicationThread在AMS的本地代理，前面已经讲过
         //所以这里实际调用的是ApplicationThread的scheduleLaunchActivity方法
         app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info,
                    mergedConfiguration.getGlobalConfiguration(),
                    mergedConfiguration.getOverrideConfiguration(), r.compat,
                    r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                    r.persistentState, results, newIntents, !andResume,
                    mService.isNextTransitionForward(), profilerInfo);
         //...
     }catch (RemoteException e) {
         //...
     }
     //...
     return true;
 }
```

正如这方法名所示，realStartActivity，兜兜转转，这里就是真正启动Activity的地方，注释1处app指的是传入的要启动的Activity的所在的应用程序进程，是从前面传过来的，这里把它赋值给了要启动的Activity的ActivityRecord的app字段去，这样就可以说要启动的Activity属于应用进程，我们再来看注释2这里，app.thread就是我们上面一直强调的ApplicationThread，所以这里通过**Binder的IPC**其实调用的是ApplicationThread中的scheduleLaunchActivity方法。

**当前的代码逻辑执行在AMS所在进程，从这里开始Activity的启动流程最终又回到了应用进程所在的ApplicationThread中。**

## 总结

本来一篇文章写完Activity的启动，写到这里才发现，篇幅太长，所以**Activity在应用进程中的启动过程**就放到下一篇文章。本文简单的介绍了**应用进程请求AMS启动Activity过程和Activity在AMS中的启动过程**，现在让我们来回答一下开头给出的几个问题：

* Activity的启动流程是怎样的？

从应用调用一个startActivity方法开始，应用进程开始请求AMS启动Activity，然后在AMS中Activity完成它的一系列准备，最后再回到应用进程中开始回调Activity的生命周期，本文回答了一半这个问题，即本文讲解了应用进程开始请求AMS启动Activity，然后在AMS中完成它的一系列准备的过程，这个过程用时序图表示如下：

![](https://img-blog.csdnimg.cn/20190420181934413.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JhaW5fOTE1NQ==,size_16,color_FFFFFF,t_70)

* Activity的生命周期方法是如何被回调的？

本文并没有解答这个问题，这个问题要到下一篇文章才能有答案。

* 它启动过程中涉及到多少个进程？

答案是2个，前言已经讲过本文讨论的是普通Activity的启动流程，即我们平时调用startActivity方法来启动一个Activity，所以本文这个过程涉及的进程可以可以用下面这个图表示：

![](https://img-blog.csdnimg.cn/20190420181958732.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JhaW5fOTE1NQ==,size_16,color_FFFFFF,t_70)

图中AppProcess代表应用所在进程，systemServer代表AMS所在进程，两个进程之间通过Binder进行通信，实现了XX.Stub的类就可以进行Binder通信，如本文的ApplicationThread和AMS都实现了各自的Stub类，所以应用进程startActivity时请求AMS启动Activity，AMS准备好后，再发送scheduleLaunchActivity请求告诉应用可以开始启动Activity了。

那么如果是前言所讲的第一种启动Activity的过程，即即在Launch界面点击一个应用图标启动应用程序，那么会涉及多少个进程？答案是4个，如图：

![](https://img-blog.csdnimg.cn/20190420182008912.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JhaW5fOTE1NQ==,size_16,color_FFFFFF,t_70)

可以看到会涉及Launcher进程、SystemServer进程、App进程、Zygote进程。关于这些进程的简单信息可以看这篇[从进程的角度看Android的系统架构](https://blog.csdn.net/Rain_9155/article/details/88831678)

阅读源码真的是一个漫长的过程，又时候看别人写的那么简单，但是当自己去写，才发现要考虑的东西很多，所以这是一个日积月累的过程，所以阅读源码的时候，最后跟着前人的文章阅读，这样理解的更快。

参考文章：

《Android开发艺术探索》

[startActivity启动过程分析](http://gityuan.com/2016/03/12/start-activity/)

[Android8.0 根Activity启动过程](http://liuwangshu.cn/framework/component/6-activity-start-1.html)