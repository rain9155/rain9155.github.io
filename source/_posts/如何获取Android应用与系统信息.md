---
title: '如何获取Android应用和系统信息'
date: 2019-07-12 22:36:32
tags: 
- android
- ActivityManager
- PackageManager
categories: android
---

## 前言

本主要了解一下Android系统信息的获取，apk应用信息的获取。

<!--more-->

	本文相关源码在文末给出

## Android系统信息的获取

有时我们想要获取手机系统的配置信息，通常可以从以下俩方面获取：

* android.os.Build

* SystemProperty


###  1、android.os.Build
android.os.Build包含了系统编译时的大量设备，配置信息，它里面的字段含义如下：

* Build.BOARD; //主板
* Build.BRAND; //Android系统指定商
* supported_abis = null;//CPU指令集
* Build.DEVICE;//设备参数
* Build.DISPLAY;//显示屏参数
* Build.FINGERPRINT;//唯一编号
* Build.SERIAL;//硬件序列号
* Build.ID;//修订版本列表
* Build.MANUFACTURER;//硬件制造商
* Build.MODEL;//版本
* Build.HARDWARE;//硬件名
* Build.PRODUCT;//手机产品名
* Build.TAGS;//描述Build的标签
* Build.TYPE;//Builder类型
* Build.VERSION.CODENAME;//当前开发号
* Build.VERSION.INCREMENTAL;//源码版本控制号
* Build.VERSION.RELEASE;//版本字符串
* Build.VERSION.SDK_INT;//版本号
* Build.HOST;//Host值
* Build.USER;//User名
* Build.TIME;//编译时间
### 2、SystemProperty
SystemProperty包含许多系统，配置属性值和参数，很多与上面通过android.os.Build获取的相同,下面给出常用的信息：
* System.getProperty("os.version");//os 版本
* System.getProperty("os.name");//os 名称
* System.getProperty("os.arch");//os 架构
* System.getProperty("user.home");//Home 属性
* System.getProperty("user.name");//Name 属性
* System.getProperty("user.dir");//Dir 属性
* System.getProperty("user.timezone");//时区
* System.getProperty("path.separator");//路径分隔符
* System.getProperty("line.separator");//行分隔符
* System.getProperty("file.separator");//文件分隔符
* System.getProperty("java.vendor.url");//java vender URL 属性
* System.getProperty("java.class.path");//java Class 路径
* System.getProperty("java.class.version");//java Class 版本
* System.getProperty("java.vendor");//java Vender 属性
* System.getProperty("java.version");//java 版本
* System.getProperty("java_home");//java HOME属性
### 3、实例

通过一个简单的示例查看如何使用（更多细节查看文末源码），如下：

```java
        /* 通过android.os.Build，可以直接获得一些Build提供的系统信息 */
        String board = Build.BOARD;
        String brand = Build.BRAND;
        Log.d(TAG, "android.os.Build，board：" + board);
        Log.d(TAG, "android.os.Build，brand： " + brand);
        /* 通过SystemProperty，要使用System.getProperty("XXX") */
        String os_version = System.getProperty("os.version");
        String os_name = System.getProperty("os.name");
        Log.d(TAG, "SystemProperty，os_version: " + os_version);
        Log.d(TAG, "SystemProperty, os_name: " + os_name);
```

可以看到，获取系统信息还是很简单的。

## Apk应用信息的获取

Apk应用信息的获取无非分为，apk包信息的获取与应用进程信息的获取。

### 1、PackageManager

通过PackageManager可以获得应用的apk包信息，先看下面一张图：

{% asset_img system1.png system1 %} 

最里面的框代表了整个Activity的信息，系统提供了ActivityInfo类来进行封装，以此类推。

#### 1.1、下面列举一些常用的系统封装信息

下面这些封装信息与PackageManager在同一个包内。

* **ActivityInfo** 
  ActivityInfo 封装了在Manifest文件中\<activity>\</activity>和\<receiver>\</receiver>之间的所有信息，包括name，icon，label，launchmod等 。
* **ServiceInfo**  
  和ActivityInfo类似，它封装了\<service>\</service>之间的所有信息 。
* **ApplicationInfo**  
  它封装了\<application>\</application>之间的信息，特别的是，Application包含很多Flag，FLAG_SYSTEM表示为系统应用，FLAG_EXTERNAL_STORAGE表示为安装在sd卡上的应用等，通过这些FLAG可以很方便的判断应用类型 。
* **PackageInfo**  
  它用于封装Manifest文件相关节点的信息，包含了所有Activity、Service等信息 。
* **ResolveInfo** 
  这个比较特殊，它封装的是包含\<intent>信息的上一级信息，所以它可以返回ActivityInfo，ServiceInfo等包含<intent>的信息，它经常用来帮助我们找到那些包含特定intent条件的信息，如带分享，播放功能的应用。

    通过上面的对象，PackageManager 就可以通过调用各种方法来返回不同类型的Bean

#### 1.2、PackageManager常用方法
* **getPackagerManager**: 通过调用这个方法返回一个PackageManager对象
* **getApplicationInfo**: 以ApplicationInfo形式返回指定包名的Application
* **getApplicationIcon**：返回指定包名的icon
* **getInstalledApplications**： 以ApplicationInfo形式返回安装的应用
* **getInstalledPackages**：以PackageInfo的形式返回安装的应用
* **queryIntentActivities**: 返回指定intent的ResolveInfo对象、Activity集合
* **queryIntentServices**：返回指定intent的ResolveInfo对象、service集合
* **resolveActivity**：返回指定intent的Activity
* **resolveService**：返回指定的intentService
#### 1.3、实例

下面通过一个例子来了解如何通过PackageManager来选出不同类型的app，判断app类型的依据，就是利用Applicationinfo中的FLAG_SYSTEM来进行判断，如下：

```java
app.flags & Applicationinfo.FLAG_SYSTEM
```

* **如果flags & Applicationinfo.FLAG_SYSTEM ！= 0，则为系统应用**
* **如果flags & Applicationinfo.FLAG_SYSTEM <= 0, 则为第三方应用** 
* **如果flags & Applicationinfo.FLAG_EXTERNAL_STORAGE != 0, 则为SD卡上的应用** 
* **特殊的，当系统应用经过升级后，也将成为第三方应用：flags & Applicationinfo.FLAG_UPDATED_SYSTEM_APP != 0**

首先封装一个Bean保存我所需要的app信息：

```java
public class PMAppInfo {

    private String appName;//app名称
    private Drawable appIcon;//图标
    private String pkgName;//所在包名

    public PMAppInfo(String appLabel, Drawable appIcon, String pkgName) {
        this.appName = appLabel;
        this.appIcon = appIcon;
        this.pkgName = pkgName;
    }

    public String getAppLabel() {
        return appName;
    }

    public void setAppLabel(String appName) {
        this.appName = appName;
    }

    public Drawable getAppIcon() {
        return appIcon;
    }

    public void setAppIcon(Drawable appIcon) {
        this.appIcon = appIcon;
    }

    public String getPkgName() {
        return pkgName;
    }

    public void setPkgName(String pkgName) {
        this.pkgName = pkgName;
    }

}
```

接下来，通过上面所说的方法判断各种类型的app：

```java
private List<PMAppInfo> getAppInfoList(int flag){
        pm = this.getPackageManager();
        //获取应用信息
        List<ApplicationInfo> applicationInfoList = pm.getInstalledApplications(PackageManager.GET_UNINSTALLED_PACKAGES);
        List<PMAppInfo> appInfoList = new ArrayList<>();
        //判断应用类型
        switch (flag){
            case ALL_APP:
                appInfoList.clear();
                for (ApplicationInfo app : applicationInfoList) {
                    appInfoList.add(new PMAppInfo(
                            ((String) app.loadLabel(pm)),
                            app.loadIcon(pm),
                            app.packageName));
                }
                break;
            case SYSTEM_APP:
                appInfoList.clear();
                for (ApplicationInfo app : applicationInfoList) {
                    if ((app.flags & ApplicationInfo.FLAG_SYSTEM) != 0) {
                        appInfoList.add(new PMAppInfo(
                                ((String) app.loadLabel(pm)),
                                app.loadIcon(pm),
                                app.packageName));
                    }
                }
                break;
            case THIRD_APP:
                appInfoList.clear();
                for (ApplicationInfo app : applicationInfoList) {
                    if ((app.flags & ApplicationInfo.FLAG_SYSTEM) <= 0) {
                        appInfoList.add(new PMAppInfo(
                                ((String) app.loadLabel(pm)),
                                app.loadIcon(pm),
                                app.packageName));
                    } else if ((app.flags & ApplicationInfo.FLAG_UPDATED_SYSTEM_APP) != 0) {
                        appInfoList.add(new PMAppInfo(
                                ((String) app.loadLabel(pm)),
                                app.loadIcon(pm),
                                app.packageName));
                    }
                }
                break;
            case SDCARD_APP:
                appInfoList.clear();
                for (ApplicationInfo app : applicationInfoList) {
                    if ((app.flags & ApplicationInfo.FLAG_EXTERNAL_STORAGE) != 0) {
                        appInfoList.add(new PMAppInfo(
                                ((String) app.loadLabel(pm)),
                                app.loadIcon(pm),
                                app.packageName));
                    }
                }
                break;
            default:
                return null;
            }
        return appInfoList;
    }
```
运行效果图：

{% asset_img system2.png system2 %}  

如上图所示，通过点击不同的按钮，在下方显示出不同类型的apk信息。

### 2、ActivityManager

前面使用了PackageManager获得了应用包的信息，PackageMessager重点在于获得应用的包信息，而ActivityManager重点在于获得在运行时的应用程序的进程信息。

#### 2.1、同PackageManager一样， ActivityManager也封装了很多Bean对象

下面的封装类都在ActivityManager的类中。

* **ActivityManager.MemoryInfo** 
  它封装了内存信息，MemoryInfo中有几个重要的字段：availMem - 系统可用内存，totalMen - 总内存，threshold - 低内存的阈值，即区分是否是低内存，lowMemory - 是否处于低内存。

* **Debug.MemoryInfo** 
  android中还有一个MemoryInfo，它来自Debug.MemoryInfo,前面的MemoryInfo通常用于获取全局的内存使用信息，而它用于统计进程下的内存信息。

* **RunningAppProcessInfo**  
  顾名思义，就是运行间进程信息，储存的字段自然进程相关的信息，processName - 进程名， pid - 进程pid, uid - 进程uid，pkgList - 该进程下所有的包 。 

* **RunningServiceInfo**
  RunningServiceInfo与RunningAppProcessInfo类似，用于封装运行时的服务信息，它包含了进程信息的同时还包含了其他的信息，activeSince - 第一次被激活的时间，方式，foreground - 服务是否在后台运行。

#### 2.2、实例

下面同样通过一个例子来使用ActivityManager，先封装一个Bean来保存一个我们需要的字段：

```java
public class AMProcessInfo {

    private String uid;
    private String pid;
    private String memorySize;
    private String processName;

    public AMProcessInfo() {
    }

    public String getPid() {
        return pid;
    }

    public void setPid(String pid) {
        this.pid = pid;
    }

    public String getUid() {
        return uid;
    }

    public void setUid(String uid) {
        this.uid = uid;
    }

    public String getMemorySize() {
        return memorySize;
    }

    public void setMemorySize(String memorySize) {
        this.memorySize = memorySize;
    }

    public String getProcessName() {
        return processName;
    }

    public void setProcessName(String processName) {
        this.processName = processName;
    }
}
```
通过调用getRunningAppProcesses（）返回当前运行时的进程信息集合：
```java
  private List<AMProcessInfo> getRunningProcessesInfo() {
        amProcessInfoList = new ArrayList<>();
        //获取正在运行的进程集合
        List<ActivityManager.RunningAppProcessInfo> appProcessList = am.getRunningAppProcesses();
        for (int i = 0; i < appProcessList.size(); i++) {
            ActivityManager.RunningAppProcessInfo info = appProcessList.get(i);
            int pid = info.pid;
            int uid = info.uid;
            String processName = info.processName;
            //获取该进程下的内存
            int[] memoryPid = new int[]{pid};
            Debug.MemoryInfo[] memoryInfo = am.getProcessMemoryInfo(memoryPid);
            int memorySize = memoryInfo[0].getTotalPss();
            AMProcessInfo processInfo = new AMProcessInfo();
            //
            processInfo.setPid("" + pid);
            processInfo.setUid("" + uid);
            processInfo.setMemorySize("" + memorySize);
            processInfo.setProcessName(processName);
            amProcessInfoList.add(processInfo);
        }
        return amProcessInfoList;
    }
```
运行效果图： 

{% asset_img system3.png system3 %} 

上图运行给出了当前运行的一个进程的pid，uid，占用内存，进程名的信息。

## 结语

PackageManager是用来获取apk包信息的，ActivityManager是用来获取运行时进程信息的，如果想要获取手机系统信息，可以通过SystemProperty和android.os.Build，本文同样是一篇学习记录，希望大家读完后和我一样有所收获。

[本文相关源码位置](https://github.com/rain9155/AndroidMessageObtainTest)

参考资料：

《android群英传》
