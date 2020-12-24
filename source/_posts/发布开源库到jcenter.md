---
title: 发布开源库到jcenter
tags: jcenter
categories: 其他
date: 2019-06-05 18:41:30
---


## 前言

前几天写过一篇文章[快速发布开源库到jitpack](https://blog.csdn.net/Rain_9155/article/details/90516026)，在里面我控诉发布jcenter的发布过程繁琐，对新手不友好，直到这几天我遇到了一个[bintray-release](https://github.com/novoda/bintray-release)插件，发现它可以帮助你更简单的发布开源库到jcenter上，而且过程也很简单。

如果你还不懂jcenter是什么或者你不懂那些配置有什么作用，强烈建议你先看一下这个两篇文章[教你一步步发布一个开源库到 JCenter](https://juejin.im/post/5aef06e56fb9a07aae151406)、[手把手教你发布自己的开源库到jcenter](https://www.jianshu.com/p/cfc9669b732b)，这两篇文章也教了另外两种发布开源库到jcenter的方式。本文是第3种方式，使用bintray-release发布开源库到jcenter上。

## 步骤

如果你已经有账号和仓库了，就跳过1、2步。

### 1、注册bintray账号

打开[bintray](https://bintray.com/)网站(可能会有点慢，如果你没有翻墙)，点击下图的位置，如下：

{% asset_img step1.png step1 %}

千万不要点击绿色那个按钮，那个是给企业用的，我们是个人开发者，注册个人账号，然后出现以下画面：

{% asset_img step2.png step2 %}

可以关联github和google账号注册，但是如果你github和google关联的邮箱是中国的邮箱，会注册失败，所以这里是建议你先去注册一个Google邮箱，要有国外的邮箱才能注册成功。

这里假设你已经有Google邮箱了，然后你填好图中的信息，First Name填姓，Last Name填名，Username是填用户名，最好不要填中文，剩下的一看就直到填什么了，填好信息后点击**Create My Account**。这时它会发一封认证邮件给你的邮箱，因为邮箱是Google的，可能会有点慢，等你收到邮件点击确认后，它才会进入你的个人界面，否则就一直卡在那里，所以耐心等待邮件确认。

### 2、创建Maven库

收到邮件确认后，进入个人界面，如图：

{% asset_img step3.png step3 %}

点击**Add New Repository**，进入如下界面：

{% asset_img step4.png step4 %}

这里填你的maven仓库的信息，Name填maven仓库名，这个很重要，记住你填的仓库名，Type就选maven，Licences就默认，Description可选，填仓库的描述，填完后点击**Create**。

然后你就可以在个人界面看到你刚才创建的maven仓库，如下：

{% asset_img step5.png step5 %}

你以后发布的开源库都会放到这个仓库中。

### 3、配置根目录build.gradle

打开你的项目，在根目录的build.gradle种引入插件，这里以我的[Utils](https://github.com/rain9155/Utils)为例，如下：

{% asset_img step6.png step6 %}

```java
classpath 'com.novoda:bintray-release:0.9.1'
```

插件的最新版本可以去这里找[bintray-release](https://github.com/novoda/bintray-release/releases)。

### 4、配置库目录中的build.gradle

点开你的库目录，在build.gradle中添加如下代码：

{% asset_img step7.png step7 %}

```java
apply plugin: 'com.novoda.bintray-release'
publish {
    repoName = 'jianyu' //maven仓库名
    userOrg = "rain9155" //bintray.com注册的用户名
    groupId = "com.jianyu" //jcenter上的路径
    artifactId = 'utils' //项目名称
    publishVersion = "v1.0" //版本号
    desc = "a simple utils tool" //描述，不重要，要填的话不要填中文，不然会乱码
    website = "https://github.com/rain9155/Utils" ////网站，不重要，就填github上的地址就行了
}
```

我按上述填后，我到时库引用是这样：**implementation 'com.jianyu:utils:v1.0'**。

### 5、执行构建上传命令

打开**Terminal**面板，在命令行中输入：**gradlew clean build bintrayUpload -PbintrayUser=你的用户名 -PbintrayKey=你的Api密匙 -PdryRun=false**

用户名就是你上面的的用户名，我这里是rain9155，Api密匙需要到网站的个人简介中找，如下:

{% asset_img step8.png step8 %}

{% asset_img step9.png step9 %}

把命令填写完整后填点击回车，等待它上传，我大概等了10几分钟，出现以下代表上传成功：

{% asset_img step10.png step10 %}

否则出现**BUILD FAILED**，就是有错误，排错后再重新输入命令，重新上传，直到成功。一般的错误都是超时、maven库名字填错、无法找到该类。

### 6、Add to Jcenter

打开你的maven仓库，如下：

{% asset_img step11.png step11 %}

可以看到utils库上传成功，点进去，如下：

{% asset_img step12.png step12 %}

点击1处那里会有一个**Add to jcenter**按钮，因为我已经add过了，所以会消失，但是你们的会有，进入如下画面：

{% asset_img step13.png step13 %}

直接点击**Send**，等待几个小时后，jcenter的审核人员会给你发一封站内邮件，如下：

{% asset_img step14.png step14 %}

然后你就可以愉快的一句话引用你的库到项目中了，在上面2的红色圈那里已经圈出来了。

## 结语

有了插件的帮助，发布一个开源库还是挺简单的。大家尝试一下吧。

更多信息查看[Utils](https://github.com/rain9155/Utils)