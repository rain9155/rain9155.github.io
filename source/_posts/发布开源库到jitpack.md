---
title: 发布开源库到jitpack
tags: jitpack
categories: 其他
date: 2019-06-10 18:47:41
---

## 前言

最近几天准备发布一个开源库，方便自己使用，一开始了解到的是发布到jcenter仓库中，它是目前世界上最大的java和Android开源软件构件仓库，而且 JCenter 是 Android Studio 默认使用的服务器仓库，只需要一句话就可以搞定整个包的导入过程，但是它的发布过程繁琐，而且对于新手来说特别不友好，就算你跟着网上教程来发布，运气好的话你就会发布成功，如果运气不好，你就会遇到很多构建失败、上传失败、翻墙等问题，而且在网上还找不到答案。我就属于运气不好的一类，我发布的时候几乎是使用了所有方法，但还是失败，就在我心灰意冷的时刻，我发现了**jitpack**，几步操作就把开源库发布成功，真的是简单了许多，有种相见恨晚的感觉，下面就把我发布的过程分享给大家。

## JitPack是什么？

在讲解发布流程之前先简单介绍一下jitpack是什么，[JitPack](https://jitpack.io/)是一个网站，它允许你把git托管的java或android项目（貌似目前仅支持github和码云），轻松发布到jitpack的maven仓库上，它所有内容都通过内容分发网络（CDN）使用加密https连接获取。

## 发布步骤

下面开始讲解发布步骤。以我的开源库[Loading](https://github.com/rain9155/Loading)为例。

### 1、准备好你要发布的library

library不同于app工程，library是没有applicationId的，还有build.gradle中apply的的插件也不一样，在AS中按如下操作新建一个library：File -> new -> New Moudle -> Android Library -> next -> 填写好信息后 -> finish。

### 2、给你要发布libaray添加配置

2.1、首先在项目的根目录的build.gradle下添加maven插件，如下：

{% asset_img step1.png step1 %}

可以去这里查看插件的最新版本[android-maven-gradle-plugin](https://github.com/dcendents/android-maven-gradle-plugin)。

2.2、然后再library目录的build.gradle下apply插件和添加group，如下：

{% asset_img step2.png step2 %}

group填**com.github.你的github账号名**，这里我的是rain9155。

### 3、执行gradlew命令，排错

再命令行下输入**gradlew install**命令，这个命令会构建你的library到你的本地 maven 仓库($HOME/.m2/repository)中，如下：

{% asset_img step3.png step3 %}

如果出现BUILD SUCCESS，说明构建成功，如果出现BUILD FAIL，说明构建失败，这时候你就要按照失败提示去排错，排错完后在执行一遍gradlew install命令，直到出现BUILD SUCCESS。

### 4、本地打tag，上传到github中

4.1、在打tag前，你要先执行**git add .**和**git commit -m "XX"**命令，把代码提交到本地git仓库

4.2、然后开始在本地git仓库打tag，如下：

{% asset_img step4.png step4 %}

打tag本质就是提交一个commit，-a后面写版本号，一般是v1.0或v1.0.0，-m后面写描述信息，这里写了第一版，然后把tag push到github上面。

### 5、github上面发布release

打开你的libary的github界面，点击**release**，如下：

{% asset_img step5.png step5 %}

然后点击**Draft a new release**，新建一个release，如下：

{% asset_img step6.png step6 %}

然后填信息，如下：

{% asset_img step7.png step7 %}

填好信息后，点击**publich release**，如下：

{% asset_img step8.png step8 %}

其实relese就是一个加了描述信息的tag。

### 6、用GitHub账号登陆、注册[jitpack](<https://jitpack.io/)，Look Up -> Get it 

登陆jitpack后，在jitpack的地址栏中输入你的library的的github项目地址，然后点击**Look Up**，如下：

{% asset_img step9.png step9 %}

点击Look Up后，下面会出现项目在github上发布的release版本，你有多少个release，下面就会显示多少个，然后点击**Get it**，如下：

{% asset_img step10.png step10 %}

点击Get it后，它会滚到下面去，你要滚回上面去，先等一会，等jitpack那里构建完，会出现一个绿色的log，如下：

{% asset_img step11.png step11 %}

如果出现红色的log，说明构建失败，你可以点击进去看一下失败原因，出现绿色的代表成功，然后再点Get it，它会滚到下面去，如下：

{% asset_img step12.png step12 %}

可以看到，它提示了你如何用gradle方式引用你的开源库，`maven { url "https://jitpack.io" }`就是指定私有Maven库为[JitPack](https://link.jianshu.com/?t=https://jitpack.io/)，`implementation com.github.rain9155:Loading:Tag`则是指定具体的包。然后你就可以愉快的在项目中按照它的提示引用你的开源库。

更多请查看示例[**Loading**](<https://github.com/rain9155/Loading>)。

## 其他

jitpack也是有勋章的，如下：

{% asset_img step13.png step13 %}

点击那个jitpack，把它的链接复制到你的REMAED中去，如下：

{% asset_img step14.png step14 %} 

## 结语

[JitPack](https://link.jianshu.com?t=https://jitpack.io/)是基于GitHub Releases的发布。当你打完tag，生成一个Release时，源文件会自动打包成zip。在[JitPack](https://link.jianshu.com?t=https://jitpack.io/)上点击【Get it】，就可以编译这个tag的源文件，把版本发布到这个私有Maven库中，并且可以提供给其他人使用。比起Bintray的JCenter，或者Maven Central这个官方中央仓库来说，[JitPack](https://link.jianshu.com?t=https://jitpack.io/)背靠GitHub，少了一大堆流程。