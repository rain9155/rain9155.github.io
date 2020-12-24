---
title: android6.0动态权限申请的正确流程
tags: android
categories: 开源项目
date: 2019-06-06 18:13:59
---


## 前言

从 Android 6.0（API 级别 23）开始，用户开始在应用运行时向其授予权限，而不是在应用安装时授予。所以如果你的应用使用到了一些危险权限，就必须在AndroidManifest.xml 中静态地声明需要用到的权限，并在使用到该功能时要动态的申请，否则在调用到相应权限功能时候，会抛出 SecurityException异常。所以本文探讨一下动态权限的申请的正确流程，并把它封装成一个库，简化了申请过程。

* [PermissionHelper](https://github.com/rain9155/PermissionHelper)

## 权限的分类

在讲解之前，先看一下android权限的分类，android权限分为四类，如下：

### 1、普通权限

普通权限也叫正常权限，它不需要动态申请，你只需要在用到它的时候在AndroidManifest.xml 中静态地声明，然后系统在app运行时就会自动的授予该app相应的权限。这类权限主要在你的app想要接触app沙盒外的数据或资源的时用到，它不会涉及到系统的操作，也不会泄漏或篡改用户的隐私数据。如下：

```java
ACCESS_LOCATION_EXTRA_COMMANDS 
ACCESS_NETWORK_STATE 
ACCESS_NOTIFICATION_POLICY 
ACCESS_WIFI_STATE 
BLUETOOTH 
BLUETOOTH_ADMIN 
BROADCAST_STICKY 
CHANGE_NETWORK_STATE 
CHANGE_WIFI_MULTICAST_STATE 
CHANGE_WIFI_STATE 
DISABLE_KEYGUARD 
EXPAND_STATUS_BAR 
FOREGROUND_SERVICE 
GET_PACKAGE_SIZE 
INSTALL_SHORTCUT 
INTERNET 
KILL_BACKGROUND_PROCESSES 
MANAGE_OWN_CALLS 
MODIFY_AUDIO_SETTINGS 
NFC 
READ_SYNC_SETTINGS 
READ_SYNC_STATS 
RECEIVE_BOOT_COMPLETED 
REORDER_TASKS 
REQUEST_COMPANION_RUN_IN_BACKGROUND 
REQUEST_COMPANION_USE_DATA_IN_BACKGROUND 
REQUEST_DELETE_PACKAGES 
REQUEST_IGNORE_BATTERY_OPTIMIZATIONS 
SET_ALARM 
SET_WALLPAPER 
SET_WALLPAPER_HINTS 
TRANSMIT_IR 
USE_FINGERPRINT 
VIBRATE 
WAKE_LOCK 
WRITE_SYNC_SETTINGS 
```

### 2、签名权限

该类权限只对拥有相同签名的应用开放。例如某个应用自定义了一个permission 且在权限标签中加入 android:protectionLevel=”signature”，其他应用想要访问该应用中的某些数据时，必须要在AndroidManifest.xml中声明该权限，而且还要与该应用具有相同的签名，系统会在app运行时自动授予该权限。这类我们用的比较少。

### 3、危险权限

也叫敏感权限，运行时权限，跟普通权限相反，一旦某个应该获取了该类权限，用户的隐私数据就面临被泄露篡改的风险。所以你想使用该权限就必须在AndroidManifest.xml 中静态地声明需要用到的权限，并在使用到该功能时要动态的申请，除非用户同意该权限，否则你不能使用该权限对应的功能。如下：

{% asset_img p1.png p1 %}

可以看到android把危险权限分为10组，所以申请危险权限的时候都是按组申请，我们只要申请组内的任意一个危险权限就行，当用户一旦同意授权该危险权限，那么该权限所对应的权限组中的所有其他权限也会同时被授权。

### 4、特殊权限

特殊权限我了解的有三个，如下：

- SYSTEM_ALERT_WINDOW：设置悬浮窗
- WRITE_SETTINGS：修改系统设置
- REQUEST_INSTALL_PACKAGES： 允许应用安装未知来源应用

它也是要要申请的，但是它不同于危险权限的申请，危险权限的申请会弹出一个对话框询问你是否同意，而特殊权限的申请需要跳转到指定的界面，让你手动确认同意。

## 动态权限申请流程

所以动态权限的申请就是申请**危险权限或特殊权限**，权限的申请在不同的Android版本有不同的行为，如下：

* 如果设备运行的是 Android 5.1 或更低版本，或者应用的 targetSdkVersion 为 22 或更低：如果您在 Manifest 中列出了危险权限，则用户必须在安装应用时系统会要求用户授予此权限，如果他们不授予此权限，系统根本不会安装应用，用户一旦全部同意授予，他们撤销权限的唯一方式是卸载应用。
* 如果设备运行的是 Android 6.0 或更高版本，并且应用的 targetSdkVersion为23 或更高：应用必须在 Manifest 中列出权限，并且它必须在运行时请求其需要的每项危险权限。用户可以授予或拒绝每项权限，且即使用户拒绝权限请求，应用仍可以继续运行有限的功能。用户可以随时进入应用的“Settings”中调整应用的动态权限授权。所以你每次使用到该权限的功能时，都要动态申请，因为用户有可能在“Settings”界面中把它再次关闭掉。

我这里讨论的是6.0后的动态申请，所以从 Android 6.0开始，无论您的应用面向哪个 API 级别，您都应对应用进行测试，以验证它在缺少需要的权限时行为是否正常。

如果还不了解动态权限申请的详细步骤，可以看一下这篇文章：[Android 6.0运行权限解析（高级篇](https://www.jianshu.com/p/6a4dff744031))。

这里我假设大家已经知道那些方法了，我把权限申请的流程分为单个和多个权限申请，分别画了个图。

### 1、单个权限申请流程

{% asset_img p2.png p2 %}

### 2、多个权限申请流程

{% asset_img p3.png p3 %}

### 3、自定义提示权限组的提示框

上面两个图有有提到自定义提示权限组，那么它主要包含以下内容：

* 1、包含需要授权的权限列表或单个权限提示
* 2、包含跳转到应用设置授权界面中的跳转按钮
* 3、包含放弃授权的取消按钮，即取消这个提示框

> 注意：如果用户不授权，则不能使用该功能或应用无法运行，可以考虑取消第3步的取消按钮，即无法取消这个提示框，一定要用户去“Settings”授权。

## 其他注意点

除了特殊权限外，还有一个location权限也比较特殊，需要通过 **LocationManager的isProviderEnabled(LocationManager.GPS_PROVIDER)**判断是否打开定位开关后再进行权限申请，如下：

```java
	lm = (LocationManager) this.getSystemService(this.LOCATION_SERVICE);
    if (lm.isProviderEnabled(LocationManager.GPS_PROVIDER)) {//开了定位服务
        //请求定位功能
       PermissionHelper.getInstance().with(this).requestPermission(
                Manifest.permission.ACCESS_FINE_LOCATION,
                new IPermissionCallback() {
                    @Override
                    public void onAccepted(Permission permission) {
                        //...
                    }

                    @Override
                    public void onDenied(Permission permission) {
						//...
                    }
                }
        );
    } else {
        //跳转到开启定位的地方
        Toast.makeText(this, "系统检测到未开启GPS定位服务,请开启", Toast.LENGTH_SHORT).show();
        Intent intent = new Intent();
        intent.setAction(Settings.ACTION_LOCATION_SOURCE_SETTINGS);
        startActivityForResult(intent, PRIVATE_CODE);
    }
}
```

## 结语

本文主要让让大家对权限的申请流程有进一步的认识，然后可以通过对动态权限的封装，将检测动态权限，请求动态权限，权限设置跳转，监听权限设置结果等处理和业务功能隔离开来，业务以后可以非常快速的接入动态权限支持，提高开发效率，更多细节查看[PermissionHelper](https://github.com/rain9155/PermissionHelper)。

参考资料：

[Permissions](https://developer.android.google.cn/guide/topics/permissions/overview)

[安卓系统权限，你真的了解吗？](https://mp.weixin.qq.com/s/w82temt7NjQb2eATONuEsA)

