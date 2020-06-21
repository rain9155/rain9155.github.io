---
title: Gradle的快速入门学习
tags: cradle
Categories: grade
---

## 前言

Gradle是一个灵活和高效自动化构建工具，它的构建脚本采用Groovy或kotlin语言编写，Groovy或Kotlin都是基于JVM的语言，它们的语法和java的语法有很多的类似并且兼容java的语法，所以对于Android开发者，只需很少的学习成本就能快速上手Gradle开发，同时Gradle也是Android官方的构建工具，学习Gradle，能够帮助我们更好的了解Android项目的构建过程，当项目构建出现问题时，我们也能更好的排查问题，所以Gradle的学习能帮助我们更好的管理Android项目，本文的所有示例都是采用Groovy语法编写，在阅读本文前可以先简单的入门Groovy：

[Groovy 使用完全解析](https://blog.csdn.net/zhaoyanjun6/article/details/70313790)

[Groovy官网](http://www.groovy-lang.org/single-page-documentation.html)

Gradle的官方地址如下：

[Gradle官网](https://docs.gradle.org/current/userguide/userguide.html)

[Github地址](https://github.com/gradle/gradle)

## Gradle的特点

1、Gradle构建脚本采用Groovy或Kotlin语言编写，如果采用Groovy编写，构建脚本后缀为.gradle，在里面可以使用Groovy语法，如果采用Kotlin编写，构建脚本后缀为.gradle.kts，在里面可以使用Kotlin语法；

2、因为Groovy或Kotlin都是面向对象语言，所以在Gradle中处处皆对象，Gradle的.gradle或.gradle.kts脚本本质上是一个Project对象，在脚本中一些带名字的配置项如buildscript、allprojects等本质上就是对象中的方法，而配置项后面的闭包{}就是参数，所以我们在使用这个配置项时本质上是在调用对象中的一个方法；

3、在Groovy或Kotlin中，函数和类一样都是一等公民，它们都提供了很好的闭包{}支持，所以它们很容易的编写出具有[DSL](https://zh.wikipedia.org/wiki/%E9%A2%86%E5%9F%9F%E7%89%B9%E5%AE%9A%E8%AF%AD%E8%A8%80)风格的代码，用DSL编写构建脚本的Gradle比其他采用xml编写构建脚本的构建工具如maven、Ant等的可读性更强，动态性更好，整体更简洁；

4、Gradle中主要有Project和Task对象，Project是Gradle中构建脚本的表示，一个构建脚本对应一个Project对象，Task是Gradle中最小的执行单元，它表示一个独立的任务，Project为Task提供了执行的上下文。

## Gradle的安装配置

官方教程：[Installing Gradle](https://gradle.org/install/)

安装Gradle前需要确保你的电脑已经配置好JDK，JDK的版本要求是8或更高，可以通过包管理器自动安装Gradle或手动下载Gradle安装两种方式，在Window平台下我推荐使用手动安装，在Mac平台下我推荐使用Homebrew包管理器自动安装：

### 1、Window平台

* 1、在Gradle的[安装页面](https://gradle.org/releases/)选择一个Gradle版本，下载它的binary-only或complete版本，binary-only版本表示下载的Gradle压缩包只包含Gradle的源码，complete版本表示下载的Gradle压缩包包含Gradle的源码和源码文档说明；
* 2、
* 3、打开cmd，输入`gradle -v`校验是否配置成功.

### 2、Mac平台

* 1、安装[Homebrew](https://brew.sh/index_zh-cn)；
* 2、打开终端，输入`brew install gradle`，它默认会下载安装binary-only版本;
* 3、当Homebrew安装Gradle完成后，在终端输入`gradle -v`校验是否安装成功.

## Gradle的项目结构

Gradle项目可以使用Android Studio、IntelliJ IDEA等IDE工具或文本编辑器来编写，这里我以Mac平台为例采用文本编辑器配合命令行来编写。

新建一个目录，如我这里为：～/GradleDemo，打开命令行，输入`cd ~/GradleDemo`切换到这个目录，然后输入`gradle init`，接着gradle会执行`init`这个task任务，它会让你选择生成的项目模版、编写脚本的语言、项目名称等，我选择了basic模版(即原始的Gradle项目)、Groovy语言、项目名称为GradleDemo，如下：

{% asset_img gradle1.png gradle1 %}

这样会`init`任务就会自动替你生成相应的项目模版，如下：

{% asset_img gradle2.png gradle1 %}

忽略掉隐藏的文件或目录，gradle init为我们自动生成了以下文件或目录：

### 1、build.gradle

它表示Gradle的项目构建脚本，在里面我们可以通过Groovy来编写脚本，在Gradle中，一个build.gradle就对应一个项目，build.gradle放在Gradle项目的根目录下，表示它对应的是根项目，build.gradle放在Gradle项目的其他子目录下，表示它对应的是子项目，Gradle构建时会把build.gradle解析成[Project](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html#org.gradle.api.Project)对象，你在里面编写的DSL，其实就是Project接口中的方法。

### 2、settings.gradle

它表示Gradle的多项目配置脚本，存放在Gradle项目的根目录下，在里面可以通过include来决定哪些子项目会参与构建，Gradle构建时会把settings.gradle解析成[Settings](https://docs.gradle.org/current/dsl/org.gradle.api.initialization.Settings.html#org.gradle.api.initialization.Settings)对象，include也只是Settings接口中的一个方法。

### 3、gradle/wrapper/、gradlew、gradlew.bat

`gradle init`执行时会同时执行`wrapper`任务，`wrapper`任务会创建gradle/wrapper目录，并创建gradle/wrapper目录下的gradle-wrapper.jar、gradle-wrapper.properties这两个文件，同时创建gradlew、gradlew.bat这两个脚本，它们统称为Gradle Wrapper，是对Gradle的一层包装。

Gradle Wrapper的作用就是可以让你的电脑在**不安装配置Gradle环境**的前提下运行Gradle项目，例如当你的Gradle项目被用户A clone下来时，而用户A的电脑上没有安装配置Gradle环境，用户A通过Gradle构建项目时，Gradle Wrapper就会从指定下载位置下载Gradle，并解压到电脑的指定位置，然后用户A就可以在不配置Gradle系统变量的前提下在Gradle项目的命令行中运行gradlew或gradlew.bat脚本来使用gradle命令，假设用户A要运行`gradle projects`命令来查看Gradle项目下的所有项目，在linux平台下只需要运行`./gradle projects`，在window平台下只需要运行`gradlew projects`，只是把`gradle`替换成`gradlew`。

Gradle Wrapper的每个文件含义如下：

**1、gradlew**：用于在linux平台下执行gradle命令的脚本；

**2、gradlew.bat**：用于在window平台下执行gradle命令的脚本；

**3、gradle-wrapper.jar**：包含Gradle Wrapper运行时的逻辑代码；

**4、gradle-wrapper.properties**：用于指定Gradle的下载位置和解压位置；

gradle-wrapper.properties中各个字段解释如下：

| 字段名           | 解释                                                         |
| ---------------- | ------------------------------------------------------------ |
| distributionBase | 下载的Gradle的压缩包**解压后**的主目录，为GRADLE_USER_HOME，在window中它表示**C:/用户/你电脑登录的用户名/.gradle/路径**，在mac中它表示**～/.gradle/路径** |
| distributionPath | 相对于distributionBase的解压后的Gradle的路径，为wrapper/dists |
| distributionUrl  | Grade压缩包的下载地址，在这里可以修改下载的Gradle的版本和版本类型(binary或complete)，例如gradle-6.5-all.zip表示Gradle 6.5的complete版本，gradle-6.5-bin.zip表示Gradle 6.5的binary版本 |
| zipStoreBase     | 同distributionBase，不过是表示存放下载的Gradle的压缩包的主目录 |
| zipStorePath     | 同distributionPath，不过是表示存放下载的Gradle的压缩包的路径 |

使用Gradle Wrapper后，就可以统一项目在不同用户电脑上的Gradle版本，同时不必让运行这个Gradle项目的人安装配置Gradle环境，提高了开发效率。









































