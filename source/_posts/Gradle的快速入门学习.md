---
title: Gradle的快速入门学习
tags: gradle
Categories: grade
date: 2020-06-26 22:36:22
categories:
---


## 前言

Gradle是一个灵活和高效自动化构建工具，它的构建脚本采用Groovy或kotlin语言编写，Groovy或Kotlin都是基于JVM的语言，它们的语法和java的语法有很多的类似并且兼容java的语法，所以对于java开发者，只需很少的学习成本就能快速上手Gradle开发，同时Gradle也是Android官方的构建工具，学习Gradle，能够帮助我们更好的了解Android项目的构建过程，当项目构建出现问题时，我们也能更好的排查问题，所以Gradle的学习能帮助我们更好的管理Android项目，Gradle的官方地址如下：

[Gradle官网](https://docs.gradle.org/current/userguide/userguide.html)

[Github地址](https://github.com/gradle/gradle)

## Gradle的特点

1、Gradle构建脚本采用Groovy或Kotlin语言编写，如果采用Groovy编写，构建脚本后缀为.gradle，在里面可以使用Groovy语法，如果采用Kotlin编写，构建脚本后缀为.gradle.kts，在里面可以使用Kotlin语法；

2、因为Groovy或Kotlin都是面向对象语言，所以在Gradle中处处皆对象，Gradle的.gradle或.gradle.kts脚本本质上是一个Project对象，在脚本中一些带名字的配置项如buildscript、allprojects等本质上就是对象中的方法，而配置项后面的闭包{}就是参数，所以我们在使用这个配置项时本质上是在调用对象中的一个方法；

3、在Groovy或Kotlin中，函数和类一样都是一等公民，它们都提供了很好的闭包{}支持，所以它们很容易的编写出具有[DSL](https://zh.wikipedia.org/wiki/%E9%A2%86%E5%9F%9F%E7%89%B9%E5%AE%9A%E8%AF%AD%E8%A8%80)风格的代码，用DSL编写构建脚本的Gradle比其他采用xml编写构建脚本的构建工具如maven、Ant等的可读性更强，动态性更好，整体更简洁；

4、Gradle中主要有Project和Task对象，Project是Gradle中构建脚本的表示，一个构建脚本对应一个Project对象，Task是Gradle中最小的执行单元，它表示一个独立的任务，Project为Task提供了执行的上下文。

## Groovy基础入门

本文的所有示例都是采用Groovy语言编写，在阅读本文前先简单的入门Groovy：

[Groovy 使用完全解析](https://blog.csdn.net/zhaoyanjun6/article/details/70313790)

下面主要讲Groovy与java的主要区别：

1、Groovy语句后面的分号可以忽略

```groovy
int num1 = 1
int num2 = 2
int result = add(num1, num2)

int add(int a, int b){
    return a + b
}
```

2、Groovy支持动态类型推导，使用def来定义变量和方法时可以不指定变量的类型和方法的返回值类型，同时定义方法时参数可以不指定类型

```groovy
def num1 = 1
def num2 = 2
def result = add(num1, num2)

def add(a, b){
    return a + b
}
```

3、Groovy的方法调用传参时可以不添加括号，方法不指定return语句时，最后一行默认为返回值

```groovy
def result = add 1, 2

def add(a, b){
  a + b
}
```

4、Groovy可以用单、双、三引号来表示字符串，其中单引号表示普通字符串，双引号表示的字符串可以使用取值运算符**${}**，而**$**在单引号只只是表示一个普通的字符，三引号表示的字符串又称为模版字符串，它可以保留文本的换行和缩进格式，三引号同样不支持**$**

```groovy
def world = 'world'

def str1 = 'hello ${world}'
def str2 = "hello ${world}"
def str3 = '''hello
&{world}'''

//打印输出：
//hello ${world}
//hello world
//hello
//&{world}
```

5、Groovy会为类中每个没有可见性修饰符的字段生成get/set方法，我们访问这个字段其实是调用它的get/set方法，同时如果类中的方法以get/set开头，我们也可以像普通字段一样访问这个方法

```groovy
class Person{
    def name
    private def country
      
    def setNation(nation){
        this.country = nation
    }
    def getNation(){
        return country
    }
}

def person = new Person()
//访问字段
person.name = 'rain9155'
println person.name
//像字段一样访问这个以get/set开头的方法
person.nation = "china"
println person.nation
```

6、Groovy中的闭包是用**{参数列表 -> 代码体}**表示的代码块，当参数 <= 1个时，**->**箭头可以省略，同时当只有一个参数时，如果不指定参数名默认以**it**为参数名，在Groovy中闭包对应的类型是[Closure](https://groovy-lang.org/closures.html)，所以闭包可以作为参数传递，同时闭包有一个delegate字段,  通过delegate可以把闭包中的执行代码委托给任意对象来执行

```groovy
class Person{
    def name
    def age
}

def person = new Person()
//定义一个闭包
def closure = {
    name = 'rain9155'
    age = 21
}
//把闭包委托给person对象执行
closure.delegate = person
//执行闭包，或者调用closure.call()也可以执行闭包
closure()

//以上代码把闭包委托给person对象执行，所以闭包就执行时就处于person对象上下文
//所以闭包就可以访问到person对象中的name和age字段，完成赋值
println person.name//输出：rain9155
println person.age//输出：21
```

在Gradle中很多地方都使用了闭包的委托机制，通过闭包完成一些特定对象的配置，在Gradle中，如果你没有指定闭包的delegate，delegate默认为当前项目的Project对象。

以上Groovy知识，认为对于有java基础的人来说，用于学习Gradle足够了，当然对于一些集合操作、文件操作等，等以后使用到时可以到[Groovy官网](http://www.groovy-lang.org/single-page-documentation.html)来查漏补缺。

## Gradle的安装配置

官方教程：[Installing Gradle](https://gradle.org/install/)

安装Gradle前需要确保你的电脑已经配置好JDK，JDK的版本要求是8或更高，可以通过包管理器自动安装Gradle或手动下载Gradle安装两种方式，在Window平台下我推荐使用手动安装，在Mac平台下我推荐使用Homebrew包管理器自动安装：

### 1、Window平台

* 1、在Gradle的[安装页面](https://gradle.org/releases/)选择一个Gradle版本，下载它的binary-only或complete版本，binary-only版本表示下载的Gradle压缩包只包含Gradle的源码，complete版本表示下载的Gradle压缩包包含Gradle的源码和源码文档说明，这里我下载了gradle-6.5-bin版本；
* 2、下载好Gradle后，把它解压到特定目录，如我这里解压到：D:/gradle，然后像配置java环境一样把D:/gradle/gradle-6.5/bin路径添加到系统的PATH变量下；
* 3、打开cmd，输入`gradle -v`校验是否配置成功，输出以下类似信息则配置成功.

```bash
C:\Users\HY>gradle -v

------------------------------------------------------------
Gradle 6.5
------------------------------------------------------------

Build time:   2020-06-02 20:46:21 UTC
Revision:     a27f41e4ae5e8a41ab9b19f8dd6d86d7b384dad4

Kotlin:       1.3.72
Groovy:       2.5.11
Ant:          Apache Ant(TM) version 1.10.7 compiled on September 1 2019
JVM:          10.0.2 ("Oracle Corporation" 10.0.2+13)
OS:           Windows 10 10.0 amd64
```

### 2、Mac平台

* 1、安装[Homebrew](https://brew.sh/index_zh-cn)；
* 2、打开终端，输入`brew install gradle`，它默认会下载安装binary-only版本;
* 3、当Homebrew安装Gradle完成后，在终端输入`gradle -v`校验是否安装成功.

## Gradle的项目结构

Gradle项目可以使用Android Studio、IntelliJ IDEA等IDE工具或文本编辑器来编写，这里我以Mac平台为例采用文本编辑器配合命令行来编写，Window平台类似。

新建一个目录，如我这里为：～/GradleDemo，打开命令行，输入`cd ~/GradleDemo`切换到这个目录，然后输入`gradle init`，接着gradle会执行`init`这个task任务，它会让你选择生成的项目模版、编写脚本的语言、项目名称等，我选择了basic模版(即原始的Gradle项目)、Groovy语言、项目名称为GradleDemo，如下：

{% asset_img gradle1.png gradle1 %}

这样会`init`任务就会自动替你生成相应的项目模版，如下：

{% asset_img gradle2.png gradle1 %}

忽略掉以**.**开头的隐藏文件或目录，gradle init为我们自动生成了以下文件或目录：

### 1、build.gradle

它表示Gradle的项目构建脚本，在里面我们可以通过Groovy来编写脚本，在Gradle中，一个build.gradle就对应一个项目，build.gradle放在Gradle项目的根目录下，表示它对应的是根项目，build.gradle放在Gradle项目的其他子目录下，表示它对应的是子项目，Gradle构建时会把build.gradle解析成[Project](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html#org.gradle.api.Project)对象，你在里面编写的DSL，其实就是Project接口中的方法。

### 2、settings.gradle

它表示Gradle的多项目配置脚本，存放在Gradle项目的根目录下，在里面可以通过include来决定哪些子项目会参与构建，Gradle构建时会把settings.gradle解析成[Settings](https://docs.gradle.org/current/dsl/org.gradle.api.initialization.Settings.html#org.gradle.api.initialization.Settings)对象，include也只是Settings接口中的一个方法。

### 3、Gradle Wrapper

`gradle init`执行时会同时执行`wrapper`任务，`wrapper`任务会创建gradle/wrapper目录，并创建gradle/wrapper目录下的gradle-wrapper.jar、gradle-wrapper.properties这两个文件，还同时创建gradlew、gradlew.bat这两个脚本，它们统称为Gradle Wrapper，是对Gradle的一层包装。

Gradle Wrapper的作用就是可以让你的电脑在**不安装配置Gradle环境**的前提下运行Gradle项目，例如当你的Gradle项目被用户A clone下来时，而用户A的电脑上没有安装配置Gradle环境，用户A通过Gradle构建项目时，Gradle Wrapper就会从指定下载位置下载Gradle，并解压到电脑的指定位置，然后用户A就可以在不配置Gradle系统变量的前提下在Gradle项目的命令行中运行gradlew或gradlew.bat脚本来使用gradle命令，假设用户A要运行`gradle -v`命令，在linux平台下只需要运行`./gradle -v`，在window平台下只需要运行`gradlew -v`，只是把`gradle`替换成`gradlew`。

Gradle Wrapper的每个文件含义如下：

**1、gradlew**：用于在linux平台下执行gradle命令的脚本；

**2、gradlew.bat**：用于在window平台下执行gradle命令的脚本；

**3、gradle-wrapper.jar**：包含Gradle Wrapper运行时的逻辑代码；

**4、gradle-wrapper.properties**：用于指定Gradle的下载位置和解压位置；

gradle-wrapper.properties中各个字段解释如下：

| 字段名           | 解释                                                         |
| ---------------- | ------------------------------------------------------------ |
| distributionBase | 下载的Gradle的压缩包解压后的主目录，为GRADLE_USER_HOME，在window中它表示**C:/用户/你电脑登录的用户名/.gradle/**，在mac中它表示**～/.gradle/** |
| distributionPath | 相对于distributionBase的解压后的Gradle的路径，为wrapper/dists |
| distributionUrl  | Grade压缩包的下载地址，在这里可以修改下载的Gradle的版本和版本类型(binary或complete)，例如gradle-6.5-all.zip表示Gradle 6.5的complete版本，gradle-6.5-bin.zip表示Gradle 6.5的binary版本 |
| zipStoreBase     | 同distributionBase，不过是表示存放下载的Gradle的压缩包的主目录 |
| zipStorePath     | 同distributionPath，不过是表示存放下载的Gradle的压缩包的路径 |

使用Gradle Wrapper后，就可以统一项目在不同用户电脑上的Gradle版本，同时不必让运行这个Gradle项目的人安装配置Gradle环境，提高了开发效率。

## Gradle的多项目配置

现在我们创建的Gradle项目默认已经有一个根项目了，它的build.gradle文件就处于Gradle项目的根目录下，如果我们想要添加多个子项目，这时就需要通过settings.gradle进行配置。

首先我们在GradleDemo中创建多个文件夹，这里我创建了4个文件夹，分别为：subproject_1、subproject_2，subproject_3，subproject_4，然后在每个新建的文件夹下创建build.gradle文件，如下：

{% asset_img gradle3.png gradle1 %}

接着打开settings.gradle，添加如下：

```groovy
include ':subproject_1', ':subproject_2', ':subproject_3', ':subproject_4'
```

这样就完成了子项目的添加，打开命令行，切换到GradleDemo目录处，输入`gradle projects`，执行projects任务展示所有项目之间的依赖关系，如下：

```bash
# in ~/GradleDemo 
$ gradle projects     

> Task :projects
Root project 'GradleDemo'
+--- Project ':subproject_1'
+--- Project ':subproject_2'
+--- Project ':subproject_3'
\--- Project ':subproject_4'

BUILD SUCCESSFUL in 540ms
1 actionable task: 1 executed
```

可以看到，4个子项目依赖于根项目，接下来我们来配置项目，配置项目一般在当前项目的build.gradle中进行，可以通过buildscript方法、repositories方法、dependencies方法等Project接口提供的方法进行配置，但是如果有多个项目时，而每个项目的某些配置又一样，那么在每个项目的build.gradle进行相同的配置是很浪费时间，而Gradle的Project接口为我们提供了allprojects和subprojects方法，在根项目的build.gradle中使用这两个方法可以全局的为所有子项目进行配置，allprojects和subprojects的区别是：**allprojects的配置包括根项目而subprojects的配置不包括根项目**，例如:

```groovy
//根项目的build.gradle

//为所有项目添加maven repo地址
allprojects {
    repositories {
        mavenCentral()
    }
}
//为所有子项目添加groovy插件
subprojects {
    apply plugin: 'groovy'
}
```

## Gradle构建的生命周期

当在命令行输入`gradle build`构建整个项目或`gradle task名称`执行某个任务时就会进行Gradle的构建，它的构建过程分为3个阶段：

**init(初始化阶段) -> configure(配置阶段) -> execute(执行阶段)**

* **init**：初始化阶段主要是解析settings.gradle，生成Settings对象，确定哪些项目需要参与构建，为需要构建的项目创建Project对象；
* **configure**：配置阶段主要是解析build.gradle，配置init阶段生成的Project对象，构建根项目和所有子项目，同时生成和配置在build.gradle中定义的Task对象，构造Task的关系依赖图，关系依赖图是一个有向无环图；
* **execute**：根据configure阶段的关系依赖图执行Task.

Gradle在上面3个阶段中每一个阶段的开始和结束都会hook一些监听，暴露给开发者使用，方便开发者在Gradle的不同生命周期阶段做一些事情。

settings.gradle和build.gradle分别代表Settings对象和Project对象，它们都有一个[Gradle](https://docs.gradle.org/current/dsl/org.gradle.api.invocation.Gradle.html#org.gradle.api.invocation.Gradle)对象，我们可以在Gradle项目根目录的settings.gradle或build.gradle中获取到Gradle对象，然后进行生命周期监听，如下：

```groovy
//build.gradle或settings.gradle

this.gradle.buildStarted {
    println "Gradle构建开始"
}

//---------------------init开始--------------------------------
this.gradle.settingsEvaluated {
    println "settings.gradle解析完成"
}
this.gradle.projectsLoaded {
    println "所有项目从settings加载完成"
}
//---------------------init结束--------------------------------
  
//-------------------configure开始-----------------------------
this.gradle.beforeProject {project ->
    //每一个项目构建之前被调用
    println "${project.name}项目开始构建"
}
this.gradle.afterProject {project ->
    //每一个项目构建完成被调用
    println "${project.name}项目构建完成"
}
this.gradle.projectsEvaluated {
    println "所有项目构建完成"
}
this.gradle.taskGraph.whenReady {
    println("task图构建完成")
}
//-------------------configure结束-----------------------------

//-------------------execute开始-----------------------------
this.gradle.taskGraph.beforeTask {task ->
    //每个task开始执行时会调用这个方法
    println("${task.name}task开始执行")
}
this.gradle.taskGraph.afterTask {task ->
    //每个task执行结束时会调用这个方法
    println("${task.name}task执行完成")
}
//-------------------execute结束-----------------------------

this.gradle.buildFinished {
    println "Gradle构建结束"
}
```

上面监听方法的放置顺序就是整个Gradle构建的顺序，但是要注意的是Gradle的buildStarted方法永远不会被回调，因为我们注册监听的时机太晚了，当解析settings.gradle或build.gradle时，Gradle就已经构建开始了，所以这个方法也被Gradle标记为废弃的了，因为我们没有机会监听到Gradle构建开始，同时如果你是在build.gradle中添加上面的所有监听，那么Gradle的settingsEvaluated和projectsLoaded方法也不会被回调，因为settings.gradle的解析是在build.gradle之前，在build.gradle中监听这两个方法的时机也太晚了。

> 也可以通过Gradle对象的addBuildListener方法添加[BuildListener](https://docs.gradle.org/current/javadoc/org/gradle/BuildListener.html)来监听Gradle的整个生命周期回调

上面是监听整个Gradle构建的生命周期回调监听，那么我怎么监听我的当前单个项目的构建开始和结束呢？只需要在你当前项目的build.gradle中添加：

```groovy
//build.gradle

this.beforeEvaluate {
    println '项目开始构建'
}
this.afterEvaluate {
    println '项目构建结束'
}
```

但是要注意的是在根项目的build.gradle添加上述方法，其beforeEvaluate方法是无法被回调的，因为注册时机太晚，解析根项目的的build.gradle时根项目已经开始构建了，但是子项目的build.gradle添加上述方法是可以监听到项目构建的开始和结束，因为根项目构建完成后才会轮到子项目的构建。

## Task

### 1、Task的创建

[Task](https://docs.gradle.org/current/dsl/org.gradle.api.Task.html#org.gradle.api.Task)是Gradle中最小执行单元，它是一个接口，默认实现类为[DefaultTask](https://docs.gradle.org/current/dsl/org.gradle.api.DefaultTask.html#org.gradle.api.DefaultTask)，在Project中提供了task方法来创建Task，所以Task的创建必须要处于Project上下文中，这里我在subproject_1/build.gradle中创建Task，如下：

```groovy
//subproject_1/build.gradle

//通过Project的task方法创建一个Task
task task1{
	doFirst{
		println 'one'
	}
	doLast{
 		println 'two'
 	}
}
```

上述代码通过task方法创建了一个名为task1的Task，在创建Task的同时可以通过闭包配置它的doFirst和doLast动作，doFirst和doLast都是Task中的方法，其中doFirst方法会在Task的action执行前执行，doLast方法会在Task的action执行后执行，而action就是Task的执行单元，在后面自定义Task会介绍到，除此之外还可以在创建Task之后再指定它的doFirst和doLast动作，如下：

```groovy
//subproject_1/build.gradle

//通过Project的task方法创建一个Task
def t = task task2
t.doFirst {
	println 'one'
}
t.doLast{
	println 'two'
}
```

上面通过Project的task方法创建的Task默认被放在Project的[TaskContainer](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.TaskContainer.html#org.gradle.api.tasks.TaskContainer)类型的容器中，我们可以通过Project的getTasks方法获取到这个容器，而TaskContainer提供了create方法来创建Task，如下：

```groovy
//subproject_1/build.gradle

//通过TaskContainer的create方法创建一个Task
tasks.create(name: 'task3'){
	doFirst{
		println 'one'
	}
	doLast{
 		println 'two'
 	}
}
```

以上就是创建Task最常用的几种方式，创建Task之后，就可以执行它，执行一个Task只需要把task名称接在`gradle`命令后，如下我在命令行输入`gradle task1`执行了task1:

```bash
# in ~/GradleDemo 
$ gradle task1   

> Task :subproject_1:task1
one
two

BUILD SUCCESSFUL in 463ms
1 actionable task: 1 executed
```

如果你想精简输出的信息，只需要添加`-q`参数，如：`gradle -q task1`，这样输出就只包含task1的输出：

```bash
# in ~/GradleDemo 
$ gradle -q task1
one
two
```

如果要执行多个Task，多个task名称接在`gradle`命令后用空格隔开就行，如：`gradle task1 task2 task3`。

### 2、Task的属性配置

Gradle为每个Task定义了默认的属性(Property)， 比如description、group、dependsOn、inputs、outputs等, 我们可以配置这些Property，如下配置Task的描述和分组：

```groovy
//subproject_2/build.gradle

//我们可以在定义Task时对这些Property进行赋值
task task1{
 	group = 'MyGroup'
 	description = 'Hello World'

 	doLast{
 		println "task分组：${group}"
 		println "task描述：${description}"
 	}
}
```

Gradle在执行一个Task之前，会先配置这个Task的Property，然后再执行这个Task的执行代码块，所以配置Task的代码块放在哪里都无所谓，如下：

```groovy
//subproject_2/build.gradle

//在定义Task之后才对Task进行配置
task task2{
	doLast{
		println "task分组：${group}"
 		println "task描述：${description}" 
	}
}
task2{
	group = 'MyGroup'
 	description = 'Hello World'
}

//等效于task2
task task3{
	doLast{
		println "task分组：${group}"
 		println "task描述：${description}" 
	}
}
task3.description = 'Hello World!'
task3.group = "MyGroup"

//等效于task3
task task4{
	doLast{
		println "task分组：${group}"
 		println "task描述：${description}" 
	}
}
task4.configure{
	group = 'MyGroup'
 	description = 'Hello World'
}
```

我们可以通过Task的**dependsOn**属性指定Task之间的依赖关系，如下：

```groovy
//subproject_2/build.gradle

//创建Task时通过dependsOn声明Task之间的依赖关系
task task5(dependsOn: task4){
	doLast{
		println 'Hello World'
	}
}

//或者在创建Task之后再声明task之间的依赖关系
task4.dependsOn task3
```

上述的依赖关系是task3 -> task4 -> task5，依赖的Task先执行，所以当我在命令行输入`gradle task5`执行task5时，输出：

```bash
# in ~/GradleDemo 
$ gradle task5  

> Task :subproject_2:task3
task分组：MyGroup
task描述：Hello World!

> Task :subproject_2:task4
task分组：MyGroup
task描述：Hello World

> Task :subproject_2:task5
Hello World
task5task执行完成

BUILD SUCCESSFUL in 2s
3 actionable tasks: 3 executed
```

依次执行task3、 task4 、task5。

### 3、自定义Task

前面创建的Task默认都是[DefaultTask](https://docs.gradle.org/current/dsl/org.gradle.api.DefaultTask.html#org.gradle.api.DefaultTask)类型，我们可以通过继承DefaultTask来自定义Task类型，Gradle中也内置了很多具有特定功能的Task，它们都间接继承自DefaultTask，如Copy(复制文件)、Delete(文件清理)等，我们可以直接在build.gradle中自定义Task，如下：

```groovy
//subproject_3/build.gradle

class MyTask extends DefaultTask{

  	def message = 'hello world from myCustomTask'

    @TaskAction
    def println1(){
      println "println1: $message"
    }

    @TaskAction
    def println2(){
      println "println2: $message"
    }
}
```

在MyTask中，通过**@TaskAction**注解的方法就是该Task的action，action是Task最主要的组成，它表示Task的一个执行动作，当Task中有多个action时，多个action的执行顺序按照**@TaskAction**注解的方法放置的逆顺序，所以执行一个Task的过程就是：doFirst方法 -> action方法 -> doLast方法，在MyTask中定义了两个action，接下来我们使用这个Task，如下：

```groovy
//subproject_3/build.gradle

//在定义Task时通过type指定Task的类型
task myTask(type: MyTask){
      message = 'custom message'
}
```

在定义Task时通过**type**指定Task的类型，同时还可以通过闭包配置MyTask中的message参数，在命令行输入`gradle myTask`执行这个Task，如下：

```bash
# in ~/GradleDemo 
$ gradle myTask        

> Task :subproject_3:myTask
println2: custom message
println1: custom message

BUILD SUCCESSFUL in 611ms
1 actionable task: 1 executed
```

我们自定义的Task本质上就是一个类，除了直接在build.gradle文件中编写自定义Task，还可以在Gradle项目的根目录下新建一个buildSrc目录，在buildSrc/src/main/[java/kotlin/groovy]目录中定义编写自定义Task，可以采用java、kotlin、groovy三种语句之一，或者在一个独立的项目中编写自定义Task然后发布到远程仓库托管平台，在后面自定义Plugin时会讲到这几种方式。

### 4、让Task支持增量式构建

上述我们自定义的Task每次执行时，它的action都会被执行，进行全量构建，其实Gradle支持增量式构建的Task，增量式构建就是**当Task的输入和输出没有变化时，跳过action的执行，当Task输入或输出发生变化时，在action中只对发生变化的输入或输出进行处理**，这样就可以避免一个没有变化的Task被反复构建，还有当Task发生变化时只处理变化部分，这样就会提高整个Gradle的构建效率，大大缩短整个Gradle的构建时间，所以当我们编写复杂的Task时，让Task支持增量式构建是很有必要的，让Task支持增量式构建只需要做到两步：

1、让Task的inputs和outputs参与Gradle的[Up-to-date检查](https://docs.gradle.org/current/userguide/more_about_tasks.html#sec:up_to_date_checks)；

2、让Task的action支持[增量式构建](https://docs.gradle.org/current/userguide/custom_tasks.html#incremental_tasks);

下面我们通过这两步自定义一个简单的、支持增量式构建的Copy任务，这个Copy任务的作用是把输入的文件复制到输出的位置中：

首先我们要让Copy任务的inputs和outputs参与Gradle的Up-to-date检查，每一个Task都有inputs和outputs属性，它们的类型分别为[TaskInputs](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/TaskInputs.html)和[TaskOutputs](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/TaskOutputs.html)，Task的inputs和outputs主要有以下三种类型：

- 可序列化类型：可序列化类型是指实现了Serializable的类或者一些基本类型如int、string等；
- 文件类型：文件类型是指标准的[java.io.File](https://docs.oracle.com/javase/8/docs/api/java/io/File.html)或者Gradle衍生的文件类型如[FileCollection](https://docs.gradle.org/current/javadoc/org/gradle/api/file/FileCollection.html)、[FileSystemLocation](https://docs.gradle.org/current/javadoc/org/gradle/api/file/FileSystemLocation.html)等；
- 自定义类型：自定义类型是指自己定义的类，这个类含有Task的部分输入和输出属性，或者说任务的部分输入和输出属性嵌套在这个类中.

我们可以在自定义Task时通过**注解**指定Task的inputs和outputs，通过注解指定的inputs和outputs会参与Gradle的Up-to-date检查，它是编写增量式Task的前提，Up-to-date检查是指Gradle每次执行Task前都会检查Task的输入和输出，如果一个Task的输入和输出自上一次构建以来没有发生变化，Gradle就判定这个Task是可以跳过执行的，这时你就会看到Task构建旁边会有一个**UP-TO-DATE**文本，Gradle提供了很多注解让我们指定Task的inputs和outputs，**常用**的如下：

| 注解                                                         | 对应的类型                                         | 含义                                                         |
| ------------------------------------------------------------ | -------------------------------------------------- | ------------------------------------------------------------ |
| @[Input](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/Input.html) | 可序列化类型                                       | 指单个输入可序列化的值，如基本类型int、string或者实现了Serializable的类 |
| @[InputFile](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/InputFile.html) | 文件类型                                           | 指单个输入文件，不表示文件夹，如File、RegularFile等          |
| @[InputDirectory](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/InputDirectory.html) | 文件类型                                           | 指单个输入文件夹，不表示文件，如File、Directory等            |
| @[InputFiles](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/InputFiles.html) | 文件类型                                           | 指多个输入的文件或文件夹，如FileCollection、FileTree等       |
| @[OutputFile](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/OutputFile.html) | 文件类型                                           | 指单个输出文件，不表示文件夹，如File、RegularFile等          |
| @[OutputDirectory](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/OutputDirectory.html) | 文件类型                                           | 指单个输出文件夹，不表示文件，如File、Directory等            |
| @[OutputFiles](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/OutputFiles.html) | 文件类型                                           | 指多个输出的文件，如FileCollection、Map<String, File>等      |
| @[OutputDirectories](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/OutputDirectories.html) | 文件类型                                           | 指多个输出的文件夹，如FileCollection、Map<String, File>等    |
| @[Nested](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/Nested.html) | 自定义类型                                         | 指一种自定义的类，这个类它可能没有实现Serializable，但这个类里面至少有一个属性使用本表中的一个注解标记，即这个类会含有Task的输入或输出 |
| @[Internal](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/Internal.html) | 任何类型                                           | 它可以用在可序列化类型、文件类型、还有自定义类型上，它指该属性只在Task的内部使用，即不是Task的输入也不是Task的输出，通过@Internal注解的属性不参与Up-to-date检查 |
| @[Optional](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/Optional.html) | 任何类型                                           | 它可以用在可序列化类型、文件类型、还有自定义类型上，它指该属性是可选的，通过@Optional注解的属性可以不为它赋值，关闭校验 |
| @[Incremental](https://docs.gradle.org/current/javadoc/org/gradle/work/Incremental.html) | Provider\<FileSystemLocation\> 或者 FileCollection | 它和@InputFiles或@InputDirectory一起使用，它用来指示Gradle跟踪文件属性的更改，通过@Incremental注解的文件属性可以通过InputChanges的getFileChanges方法查询文件的更改，帮助实现增量构建Task |
| @[SkipWhenEmpty](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/SkipWhenEmpty.html) | 文件类型                                           | 它和@InputFiles或@InputDirectory一起使用，它用来告诉Gradle如果相应的文件或文件夹为空就跳过该Task，通过@SkipWhenEmpty注解的所有输入属性如果都为空，就会导致Gradle跳过这个Task，从而在控制台产出一个**NO-SOURCE**输出结果 |

我们自定义Task时可以使用表中的注解来指定输入和输出，其中@InputXX是用来指定输入属性，@OuputXX是用来指定输出属性，@Nested是用来指定自定义类，这个类里面至少含有一个使用@InputXX或@OuputXX指定的属性，而@Internal和@Optional是可以用来指定输入或输出的，最后的@Incremental和@SkipWhenEmpty是用来与@InputFiles或@InputDirectory一起使用的，用于支持增量式构建任务，后面会讲，还有一点要注意的是这些注解只有声明在属性的get方法中才有效果，前面讲过groovy的字段默认都生成了get/set方法，而如果你是用java自定义Task的，要记得声明在属性的get方法中，我们来看Copy任务的实现，如下：

```groovy
//subproject_3/build.gradle

class CopyTask extends DefaultTask{

  //使用@InputFiles注解指定输入
  @InputFiles
  FileCollection from

  //使用@OutputDirectory注解指定输出
  @OutputDirectory
  Directory to

  //复制过程：把from的文件复制到to文件夹
  @TaskAction
  void execute(){
    File file = from.getSingleFile()
    if(file.isDirectory()){
      from.getAsFileTree().each {
        copyFileToDir(it, to)
      }
    }else{
      copyFileToDir(from, to)
    }
  }

  private static void copyFileToDir(File src, Directory dir){
    File dest = new File("${dir.getAsFile().path}/${src.name}")
    if(!dest.exists()){
      dest.createNewFile()
    }
    dest.withOutputStream {
      it.write(new FileInputStream(src).getBytes())
    }
  }
}
```

这里Copy任务只使用了@InputFiles和@OutputDirectory，通过@InputFiles指定Copy任务复制的来源文件，通过@OutputDirectory指定Copy任务复制的目标文件夹，然后在action方法中执行复制步骤，然后我们来使用这个Copy任务，如下：

```groovy
//subproject_3/build.gradle

task copyTask(type: CopyTask){
    from = files('from')
    to = layout.projectDirectory.dir('to')
}
```

为了使用这个Copy任务，我在subproject_3目录下创建了一个from文件夹，里面只有一个名为text1.txt的文本文件，然后把from文件夹指定为Copy任务的输入，to文件夹指定为Copy任务的输出，在命令行输入`gradle copyTask`执行这个Task，如下：

```bash
# in ~/GradleDemo 
$ gradle copyTask

> Task :subproject_3:copyTask

BUILD SUCCESSFUL in 2s
1 actionable task: 1 executed
```

任务执行成功后就可以把from文件夹中的文件复制到to文件夹，此时文件结构如下：

```
subproject_3
|_ build.gradle
|_ from
|  |_ text1.txt 
|_ to
   |_ text1.txt
```

接着我们再次在命令行输入`gradle copyTask`执行这个Task，如下：

```bash
# in ~/GradleDemo 
$ gradle copyTask

> Task :subproject_3:copyTask UP-TO-DATE

BUILD SUCCESSFUL in 634ms
1 actionable task: 1 up-to-date
```

再次执行时由于Copy任务的输入和输出都没有变化，所以Gradle判定为UP-TO-DATE，跳过执行。

目前Copy任务已经支持Up-to-date检查，但还不支持增量构建，即如果此时我们往from文件夹新增一个text2.txt文件，由于Copy任务的输入发生变化，这时重新执行Copy任务时就会重新执行action方法，进行全量构建，把text1.txt、text2.txt文件复制到to文件夹中，你会发现text1.txt被重复复制了，我们希望的是每次from中新增或修改文件时，只对新增或修改的文件进行复制，而之前没有变化的文件不进行复制，所以要做到这一步，还要让Task的action方法支持增量构建，要让Task的action方法支持增量式构建，只需要让action方法带一个[InputChanges](https://docs.gradle.org/current/dsl/org.gradle.work.InputChanges.html#org.gradle.work.InputChanges)类型的参数就可以，带**InputChanges**类型参数的action方法表示这是一个增量任务操作方法，该参数告诉Gradle，该action方法仅需要处理更改的输入，此外，Task还需要通过使用**@Incremental或@SkipWhenEmpty**来指定至少一个增量文件输入属性，我们继续看Copy任务的实现，如下：

```groovy
//subproject_3/build.gradle

class CopyTask extends DefaultTask{

    //新增@Incremental注解
    @Incremental
    @InputFiles
    FileCollection from

    @OutputDirectory
    Directory to

    // @TaskAction
    // void execute(){
    //    File file = from.getSingleFile()
    //    if(file.isDirectory()){
    //        from.getAsFileTree().each {
    //            copyFileToDir(it, to)
    //        }
    //    }else{
    //        copyFileToDir(from, to)
    //    }
    // }

    //带有InputChanges类型参数的action方法
    @TaskAction
    void executeIncremental(InputChanges inputChanges) {
        println "execute: isIncremental = ${inputChanges.isIncremental()}"
        inputChanges.getFileChanges(from).each {change ->
            if(change.fileType != FileType.DIRECTORY){
                println "changeType = ${change.changeType}, changeFile = ${change.file.name}"
                if(change.changeType != ChangeType.REMOVED){
                    copyFileToDir(change.file, to)
                }
            }
        }
    }

    private static void copyFileToDir(File src, Directory dir){
        File dest = new File("${dir.getAsFile().path}/${src.name}")
        if(!dest.exists()){
            dest.createNewFile()
        }
        dest.withOutputStream {
            it.write(new FileInputStream(src).getBytes())
        }
    }
}
```

Copy任务中通过@Incremental指定了需要增量处理的输入，然后在action方法中通过InputChanges进行增量复制文件，我们可以通过InputChanges的**getFileChanges**方法获取变化的文件，该方法接收一个FileCollection类型的参数，传入的参数必须要通过@Incremental或@SkipWhenEmpty注解，getFileChanges方法返回的是一个[FileChange](https://docs.gradle.org/current/javadoc/org/gradle/work/FileChange.html)列表，FileChange持有变化的文件File、文件类型[FileType](https://docs.gradle.org/current/javadoc/org/gradle/api/file/FileType.html)和文件的变化类型[ChangeType](https://docs.gradle.org/current/javadoc/org/gradle/work/ChangeType.html)，这样我们就可以根据**变化的文件、ChangeType、FileType**进行增量输出，ChangeType有三种取值：

- ADDED：表示这个文件是新增的；
- MODIFIED：表示这个文件被修改了；
- REMOVED：表示这个文件被删除了.

同时并不是每次执行都是增量构建，我们可以通过InputChanges的**isIncremental**方法判断本次构建是否是增量构建，当处于以下情况时，Task会以非增量形式即全量执行：

- 该Task是第一次执行；
- 该Task只有输入没有输出；
- 该Task的upToDateWhen条件返回了false；
- 自上次构建以来，该Task的某个输出文件已更改。
- 自上次构建以来，该Task的某个属性输入发生了变化，例如一些基本类型的属性；
- 自上次构建以来，该Task的某个非增量文件输入发生了变化，非增量文件输入是指没有使用@Incremental或@SkipWhenEmpty注解的文件输入；

当Task处于非增量构建时，即InputChanges的isIncremental方法返回false时，通过InputChanges的getFileChanges方法能获取到所有的输入文件，并且每个文件的ChangeType都为ADDED，当Task处于增量构建时，即InputChanges的isIncremental方法返回true时，通过InputChanges的getFileChanges方法能获取到只发生变化的输入文件。

接下来让我们在上面的基础下执行Copy任务，在命令行输入`gradle copyTask`执行这个Task，如下：

```bash
# in ~/GradleDemo 
$ gradle copyTask

> Task :subproject_3:copyTask
execute: isIncremental = false
changeType = ADDED, changeFile = text1.txt

BUILD SUCCESSFUL in 22s
1 actionable task: 1 executed
```

可以看到第一次执行Task，以非增量方式执行，把from/text1.txt文件复制到了to文件夹，接下来，我们在from文件夹下新增text2.txt、text3.text，然后再次在命令行输入`gradle copyTask`执行Copy任务，如下：

```bash
# in ~/GradleDemo 
$ gradle copyTask

> Task :subproject_3:copyTask
execute: isIncremental = true
changeType = ADDED, changeFile = text2.txt
changeType = ADDED, changeFile = text3.txt

BUILD SUCCESSFUL in 2s
1 actionable task: 1 executed
```

可以看到当我们新增文件后第二次执行Task，以增量方式执行，只把新增的text2.txt、 text3.txt复制到to文件夹，而text1.txt没有被重复复制，此时文件结构如下：

```
subproject_3
|_ build.gradle
|_ from
|  |_ text1.txt 
|  |_ text2.txt
|  |_ text3.txt
|_ to
   |_ text1.txt
   |_ text2.txt
   |_ text3.txt
```

接下来我们把text1.txt修改在里面添加一行hello world，然后再次在命令行输入`gradle copyTask`执行Copy任务，如下：

```bash
# in ~/GradleDemo 
$ gradle copyTask

> Task :subproject_3:copyTask
execute: isIncremental = true
changeType = MODIFIED, changeFile = text1.txt

BUILD SUCCESSFUL in 2s
1 actionable task: 1 executed
```

可以看到当我们修改文件后第三次执行Task，以增量方式执行，只把修改的text1.txt复制到to文件夹，而其他文件没有被重复复制，接下来我们把to文件夹里的text1.txt删除，然后再次在命令行输入`gradle copyTask`执行Copy任务，如下：

```bash
# in ~/GradleDemo 
$ gradle copyTask

> Task :subproject_3:copyTask
execute: isIncremental = false
changeType = ADDED, changeFile = text1.txt
changeType = ADDED, changeFile = text2.txt
changeType = ADDED, changeFile = text3.txt

BUILD SUCCESSFUL in 2s
1 actionable task: 1 executed
```

可以看到当我们删除某个输出文件后第四次执行Task，以非增量方式执行，把from文件夹下的文件重新复制到to文件夹，并且所有的文件状态都是ADDED，此时如果我们不做任何操作，再次在命令行输入`gradle copyTask`执行Copy任务会输出UP-TO-DATE。

到现在我们就实现了一个支持增量式构建的Copy任务，通过Gradle提供的注解和InputChanges，我们很容易就自定义一个支持增量构建的Task，当你编写的Task支持增量构建后，你还可以考虑更进一步，让你的Task的支持延迟配置 - [Lazy Configuration](https://docs.gradle.org/current/userguide/lazy_configuration.html)，这是Gradle提供的对属性的一种状态管理，就不在本文展开了。

## 自定义Plugin

Plugin可以理解为一系列Task的集合，通过实现**Plugin<T>**接口的**apply**方法就可以自定义Plugin，自定义的Plugin本质上也是一个类，所以和Task类似，在Gradle中也提供了3种方式来编写自定义Plugin：

- **1、在build.gradle中直接编写**：可以在任何一个build.gradle文件中编写自定义Plugin，此方式自定义的Plugin只对该build.gradle对应的项目可见；
- **2、在buildSrc目录下编写**：可以在Gradle项目根目录的buildSrc/src/main/[java/kotlin/groovy]目录中编写自定义Plugin，可以采用java、kotlin、groovy三种语句之一，Gradle在构建时会自动的编译buildSrc/src/main/[java/kotlin/groovy]目录下的所有类文件为class文件，供本项目所有的build.gradle引用，所以此方式自定义的Plugin只对本Gradle项目可见；
- **3、在独立项目中编写**：可以新建一个Gradle项目，在该Gradle项目中编写自定义Plugin，然后把Plugin源码打包成jar，发布到maven、lvy等托管平台上，这样其他项目就可以引用该插件，所以此方式自定义的Plugin对所有Gradle项目可见.

由于在上面自定义Task的介绍中已经讲过了如何在build.gradle中直接编写，自定义Plugin也类似，所以下面就主要介绍**2、3**两种方式，而且这两种方式也是平时开发中自定义Plugin最常用的方式。

### 1、在buildSrc目录下编写

在GradleDemo中新建一个buildSrc目录，然后在buildSrc目录新建src/main/groovy目录，如果你要使用java或kotlin，则新建src/main/java或src/main/kotlin，src/main/groovy目录下你还可以继续创建package，这里我的package为com.example.plugin，然后在该package下新建一个类MyPlugin.groovy，该类继承自Plugin接口，如下：

{% asset_img gradle4.png gradle1 %}

```groovy
class MyPlugin implements Plugin<Project>{
	@Override
  void apply(Project project){}
}
```

现在MyPlugin中没有任何逻辑，我们平时是在build.gradle中通过**apply plugin: 'Plugin名'**来引用一个Plugin，而apply plugin中的apply就是指apply方法中的逻辑，而apply方法的参数project指的就是引用该Plugin的build.gradle对应的Project对象，接下来我们让我们在apply方法中编写逻辑，如下：

```groovy
package com.example.plugin

import org.gradle.api.*

class MyPlugin implements Plugin<Project>{
  
	@Override
  void apply(Project project){
      //通过project的ExtensionContainer的create方法创建一个名为outerExt的扩展，扩展对应的类为OuterExt
      OuterExt outerExt = project.extensions.create('outerExt', OuterExt.class)
      
      //通过project的task方法创建一个名为showExt的Task
      project.task('showExt'){
          doLast{
              //使用OuterExt实例
              println "outerExt = ${outerExt}"
          }
      }
  }
  
  /**
   * 自定义插件的扩展对应的类
   */
  static class OuterExt{
      
      String message
      
      @Override
      String toString(){
          return "[message = ${message}]"
      }
  }
}
```

上述我在apply方法中创建了一个扩展和一个Task，其中Task好理解，那么扩展是什么？我们平时引用android插件时，一定见过这样类似于android这样的命名空间，如下：

```groovy
apply plugin: 'com.android.application'

android {
    compileSdkVersion 29
    buildToolsVersion "29.0.3"
  	//...
}
```

它并不是一个名为android的方法，它而是android插件中名为android的扩展，该扩展对应一个bean类，该bean类中有compileSdkVersion、buildToolsVersion等方法，所以配置android就是在配置andorid对应的bean类，现在回到我们的MyPlugin中，MyPlugin也定义了一个bean类：OuterExt，该bean类有messag字段，Groovy会自动为我们生成messag的get/set方法，而apply方法中通过project实例的ExtensionContainer的create方法创建一个名为outerExt的扩展，扩展对应的bean类为OuterExt，扩展的名字可以随便起，其中[ExtensionContainer](https://docs.gradle.org/current/javadoc/org/gradle/api/plugins/ExtensionContainer.html)类似于TaskContainer，它也是Project中的一个容器，这个容器存放Project中所有的扩展，通过ExtensionContainer的**create**方法可以创建一个扩展，create方法返回的是扩展对应的类的实例，这样我们使用MyPlugin就可以这样使用，如下：

```groovy
//subproject_4/build.gradle

apply plugin: com.example.plugin.MyPlugin

outerExt {
    message 'hello'
}

//执行gradle showExt, 输出:
//outerExt = [message = hello]
```

扩展的特点就是可以通过闭包来配置扩展对应的类，这样就可以通过扩展outerExt来配置我们的Plugin，很多自定义Plugin都是都通过添加扩展这种方式来配置自定义的Plugin，很多人就问了，那么类似于android的嵌套DSL如何实现，如下：

```groovy
apply plugin: 'com.android.application'

android {
    compileSdkVersion 29
    buildToolsVersion "29.0.3"

    defaultConfig {
        applicationId "com.example.myapplication"
        minSdkVersion 16
        targetSdkVersion 29
        //...
    }
  //...
}
```

android{}中有嵌套了一个defaultConfig{}，但是defaultConfig并不是一个扩展，而是一个名为defaultConfig的方法，参数为[Action](https://docs.gradle.org/current/javadoc/org/gradle/api/Action.html)类型，它是一个接口，里面只有一个execute方法，这里就我参考android 插件的内部实现实现了嵌套DSL，嵌套DSL可以简单的理解为**扩展对应的类中再定义一个类**，MyPlugin的实现如下：

```groovy
package com.example.plugin

import org.gradle.api.*
import org.gradle.api.model.* //新引入
import javax.inject.Inject //新引入

class MyPlugin implements Plugin<Project>{

	@Override
	void apply(Project project){
        
		OuterExt outerExt = project.extensions.create('outerExt', OuterExt.class)

		project.task('showExt'){
			doLast{
                //使用OuterExt实例和InnerExt实例
                println "outerExt = ${outerExt}, innerExt = ${outerExt.innerExt}"
			}
		}
	}

    static abstract class OuterExt{

        String message

        //嵌套类
        InnerExt innerExt

        //定义一个使用@Inject注解的、抽象的获取ObjectFactory实例的get方法
        @Inject
        abstract ObjectFactory getObjectFactory()

        OuterExt(){
            //通过ObjectFactory的newInstance方法创建嵌套类innerExt实例
            this.innerExt = getObjectFactory().newInstance(InnerExt.class)
        }

        //定义一个方法，方法名为可以随意起，方法的参数类型为Action，泛型类型为嵌套类InnerExt
        void inner(Action<InnerExt> action){
            //调用Action的execute方法，传入InnerExt实例
            action.execute(innerExt)
        }

        @Override
        String toString(){
            return "[message = ${message}]"
        }

        static class InnerExt{

            String message

            @Override
            String toString(){
                return "[message = $message]"
            }
        }
    }
}
```

使用MyPlugin就可以这样使用，如下：

```groovy
//subproject_4/build.gradle

apply plugin: com.example.plugin.MyPlugin

outerExt {
  message 'hello'

  //使用inner方法
  inner{
    message 'word'
  }
}

//执行gradle showExt, 输出:
//outerExt = [message = hello], innerExt = [message = word]
```

outerExt {}中嵌套了inner{}，其中inner是一个方法，参数类型为Action，Gradle内部会把inner方法后面的闭包配置给InnerExt类，这是Gradle中的一种转换机制，总的来说，定义嵌套DSL的大概步骤如下：

- 1、定义嵌套的DSL对应的bean类，如这里为InnerExt；
- 2、定义一个使用@Inject注解的、抽象的获取ObjectFactory实例的get方法，或者定义一个使用@Inject注解的带ObjectFactory类型参数的构造，@Inject是javax包下的，[ObjectFactory](https://docs.gradle.org/current/javadoc/org/gradle/api/model/ObjectFactory.html)是属于Gradle的model包下的类，当Gradle实例化OuterExt时，它会自动注入通过@Inject注解的实例，例如这里就自动注入了ObjectFactory实例，需要注意的是通过@Inject注解的方法或构造必须是public的；
- 3、在构造中通过ObjectFactory对象的newInstance方法来创建bean类实例，通过ObjectFactory实例化的对象可以被闭包配置；
- 4、定义一个方法，该方法的参数类型为Action，泛型类型为嵌套的DSL对应的bean类，方法名随便起，如这里为inner，然后在方法中调用Action的**execute**方法，传入bean类实例.

上面4步就是定义嵌套DSL时需要在自定义Plugin中做的事，还有一点要注意的是，对于扩展对应的bean类，如果你把它定义在自定义的Plugin的类文件中，一定要用**static**修饰，如这里的OuterExt类、InnerExt类使用了static修饰，或者把它们定义在单独的类文件中。

除了以上这种为单个对象配置的方式，Gradle还为我们提供了更为灵活地对多个相同类型对象进行配置的方式，又名命名对象容器，它类似于android中buildType{}, 如下：

```groovy
android {
  //...
  buildTypes {
    release {
      //..
    }
    debug {
      //...
    }
  }
}
```

上面buildTypes中定义了2个命名空间，分别为：release、debug，每个命名空间都会生成一个BuildType配置，在不同的场景下使用，并且我还可以根据使用场景定义更多的命名空间如：test、testDebug等，buildTypes{}中的命名空间的**数量不定**、**名字不定**，这是因为buildTypes是通过[NamedDomainObjectContainer](https://docs.gradle.org/current/dsl/org.gradle.api.NamedDomainObjectContainer.html#org.gradle.api.NamedDomainObjectContainer)容器实现的，下面在前面的基础上实现OuterExt对应的命名对象容器：

```groovy
package com.example.plugin

import org.gradle.api.*
import org.gradle.api.model.*
import javax.inject.Inject

class MyPlugin implements Plugin<Project>{

	@Override
	void apply(Project project){
        
        //通过project的ObjectFactory的domainObjectContainer方法创建OuterExt的Container实例
        NamedDomainObjectContainer<OuterExt> outerExtContainer = project.objects.domainObjectContainer(OuterExt.class)
        
        //然后再通过project的ExtensionContainer的add方法添加名称和OuterExt的Container实例的映射
		project.extensions.add('outerExts', outerExtContainer)
        
        //通过project的task方法创建一个名为showExts的Task
        project.task('showExts'){
            doLast{
                //遍历OuterExt的Container实例，逐个输出配置的值
                outerExtContainer.each{ext ->
                    println "${ext.name}: outerExt = ${ext}, innerExt = ${ext.innerExt}"
                }
            }
        }
	}

    static abstract class OuterExt{

        String message

        InnerExt innerExt

        @Inject
        abstract ObjectFactory getObjectFactory()

        //NamedDomainObjectContainer要求它的元素必须要有一个只可读的、名为name的常量字符串
        private final String name

        //只可读的name表示name要私有的，并且提供一个get方法，name的值在构造函数中注入
        String getName(){
            return this.name
        }

        //通过@Inject注解带有String类型参数的构造
        @Inject
        OuterExt(String name){
            //在构造中为name赋值
            this.name = name
            this.innerExt = getObjectFactory().newInstance(InnerExt.class)
        }

        void inner(Action<InnerExt> action){
            action.execute(innerExt)
        }

        @Override
        String toString(){
            return "[message = ${message}]"
        }

        static class InnerExt{

            String message

            @Override
            String toString(){
                return "[message = ${message}]"
            }
        }
    }
}
```

使用MyPlugin就可以这样使用，如下：

```groovy
//subproject_4/build.gradle

apply plugin: com.example.plugin.MyPlugin

outerExts{
    
    //定义名为ext1的命名空间
	ext1{
		message 'hello'

		inner{
			message 'word'
		}
	}

    //定义名为ext2的命名空间
	ext2{
		message 'hello'

		inner{
			message 'word'
		}
	}
}

//执行gradle showExts, 输出:
//ext1: outerExt = [message = hello], innerExt = [message = word]
//ext2: outerExt = [message = hello], innerExt = [message = word]
```

outerExts可以想象为一个容器，然后容器的元素就是OuterExt，放进容器中的元素都要有一个名字，这个名字是任意的，上面我们通过DSL在outerExts扩展中配置了两个OuterExt，一个名为ext1，一个名为ext2，然后我们在Plugin中就可以通过遍历outerExts获取每个元素对应的配置，所以总的来说，定义命名对象容器作为扩展的大概步骤如下：

- 1、首先要确定NamedDomainObjectContainer容器的元素类型，如我这里为OuterExt类，容器中的每个元素必须要有一个只可读的、名为name的常量字符串，然后提供一个使用@Inject注解的带有String类型参数的构造，在构造中为name赋值，这个name就对应你在DSL中填写的命名，如我这里为ext1、ext2，当我们在DSL中添加命名空间时，其实就是在往容器中添加元素，这时Gradle会自动调用元素带有String类型参数的构造来实例化它，并在构造中传入命名字符串；

- 2、通过Project的getObjects方法获取ObjectFactory实例，调用ObjectFactory的**domainObjectContainer**方法创建容器实例，前面讲过通过ObjectFactory实例化的对象可以被闭包配置，如我这里创建的容器为NamedDomainObjectContainer\<OuterExt\>；

- 3、然后再通过Project的getExtensions方法获取ExtensionContainer实例，调用ExtensionContainer的**add**方法添加扩展名和容器实例的映射，如我这里添加了名为outerExts的扩展，扩展对应的类为NamedDomainObjectContainer\<OuterExt\>.

上面3步就是定义命名对象容器作为扩展时需要在自定义Plugin中做的事，1、2步骤是如何定义一个命名对象容器，2步骤是把命名对象容器添加为一个扩展，但是往往命名对象容器只是作为某个扩展中的一个嵌套DSL，而不是直接作为扩展，这时我们只需要结合前面讲过的定义扩展步骤、定义嵌套DSL步骤和这里的1、2步骤就行，这时我们就可以实现下面的DSL写法：

```groovy
//扩展
outerExt{
    message 'hello'

    //通过命名对象容器配置
    exts{
        ext1{
            message 'word'
        }

        ext2{
            message 'word'
        }
    }
}
```

这时我们已经很接近android gradle plugin提供的android{}扩展的DSL写法了，在自定义Plugin中如何实现就留给大家自己实现了，只要结合一下前面所讲过的方法就行，上面所介绍的扩展、嵌套DSL、命名对象容器已经能满足自定义Plugin开发时获取配置的大部分场景了。

### 2、在独立项目中编写

在独立项目中编写和在buildSrc目录下编写是一样的，只是多了一个**发布**过程，这里我为了方便就不新建一个独立项目了，而是在GradleDemo中新增一个名为gradle_plugin的子项目，然后在gradle_plugin下新建一个src/main/groovy和src/main/resources目录，接着把刚刚在buildSrc编写的com.example.plugin.MyPlugin复制到src/main/groovy下，最后在GradleDemo新建一个repo目录，当作待会发布插件时的仓库，此时GradleDemo结构如下：

{% asset_img gradle5.png gradle1 %}

在Gradle中有两种方式在脚本中引入一个Plugin，一种是在buildScript中引入类路径，然后通过apply plugin引入插件，如下

```groovy
//build.gradle
buildscript {
	repositories {
		//定义插件所属仓库
	}

	dependencies {
		classpath '插件类路径'
	}
}

apply plugin: '插件id'
```

一种是直接通过plugins DSL引入，如下：

```groovy
//setting.gradle
pluginManagement{
    repositories{
        //定义插件所属仓库
    }
}

//build.gradle
plugins{
    id '插件id' version '插件版本'
}
```

两种方式分别对应着两种类型的发布方式，下面分别介绍：

#### 2.1、自定义META-INF发布

因为gradle_plugin项目中的MyPlugin.groovy使用了Gradle的相关api，如Project等，所以你要在gradle_plugin/build.gradle中引进Gradle api，打开gradle_plugin/build.gradle，添加如下：

```groovy
//gradle_plugin/build.gradle

dependencies {
    //这样gradle_plugin/src/main/groovy/中就可以使用Gradle和Groovy语法
    implementation gradleApi()
}
```

现在MyPlugin已经有了，我们需要给插件起一个名字，在gradle_plugin/src/main/resources目录下新建**META-INF/gradle-plugins**目录，然后在该目录下新建一个**XX.properties**文件，XX就是你想要给插件起的名字，就是apply plugin后填写的插件名字，也称为插件id，例如andorid gradle plugin的插件id叫com.android.application，所以它的properties文件为com.android.application.properties，这里我给我的MyPlugin插件起名为**myplugin**，所以我新建myplugin.properties，如下：

{% asset_img gradle6.png gradle1 %}

打开myplugin.properties文件，添加如下：

```groovy
# 在这里指明自定义插件的实现类
implementation-class=com.example.plugin.MyPlugin
```

通过implementation-class指明你要发布的插件的实现类，如这里为com.example.plugin.MyPlugin，接下来我们就来发布插件。

发布插件你可以选择你要发布到的仓库，如maven、lvy，我们最常用的就是maven了，所以这里我选择maven，Gradle提供了[maven-publish插件](https://docs.gradle.org/current/userguide/publishing_maven.html#publishing_maven)来帮助我们发布到maven仓库，在gradle_plugin/build.gradle添加如下：

```groovy
//gradle_plugin/build.gradle

plugins{
    //引入maven-publish, maven-publish属于Gradle核心插件，核心插件可以省略version
    id 'maven-publish'
}

//publishing是maven-publish提供的扩展，通过repositories定义发布的maven仓库位置，可以指定本地目录地址或远端repo地址
publishing.repositories {
    //这里我指定了项目根目录下的repo目录
    maven {
        url '../repo'
    }
    
    //可以指定远端repo地址
    //maven{
    //    url 'https://xxx'
    //    credentials {
    //        username 'xxx'
    //        password xxx
    //   }
    //}
}

//通过publications定义发布的组件
publishing.publications  {
    //类似于命令容器对象，添加名为myplugin的的发布
    myplugin(MavenPublication){
        //配置自定义插件的groupId、artifactId和version
        groupId = 'com.example.customplugin'
        artifactId = 'myplugin'
        version = '1.0'
        //通过from引入打包jar的components
        from components.java
    }
}
```

我们首先通过repositories{}定义了发布仓库，这里为本地目录，然后在publications{}中定义了名为myplugin的[发布](https://docs.gradle.org/current/dsl/org.gradle.api.publish.maven.MavenPublication.html)，它会被maven-publish插件用来生成发布任务，在其中通过groupId、artifactId和version配置了插件的**GAV**坐标，这样发布后我们就可以通过GAV坐标引用插件，即在dependencies{}中使用**groupId:artifactId:version**的形式来引用我们发布的插件，同时还引入了components.java，它是java插件为我们提供的把代码打包成jar的能力，当你做完这一切后，同步Gradle项目，maven-publish插件会为我们生成两种类型的发布任务：**publishXXPublicationToMavenRepository**和**publishXXPublicationToMavenLocal**，publishXXPublicationToMavenRepository任务表示把插件发布到maven远程仓库中，publishXXPublicationToMavenLocal任务表示把插件发布到maven本地仓库，其中XX就是在publications{}定义的发布名字，如我这里为myplugin。

接着在Gradle项目所处目录的命令行输入`gradle publishMypluginPublicationToMavenLocal`把myplugin插件发布到maven本地仓库，任务执行成功后，在maven本地仓库(mac下为～/.m/repository/，在window下为：C:/用户/用户名/.m/repository/)中会看到发布的插件jar和pom文件，如下：

{% asset_img gradle7.png gradle1 %}

接着在Gradle项目所处目录的命令行输入`gradle publishMypluginPublicationToMavenRepository`把myplugin插件发布到maven远程仓库，任务执行成功后，在GradleDemo/repo/中会看到发布的插件jar和pom文件，如下：

{% asset_img gradle8.png gradle1 %}

现在MyPlugin 1.0已经发布成功了，接下来让我们使用这个插件，在subproject_4/build.gradle中添加如下：

```groovy
//subproject_4/build.gradle

buildscript {
    repositories { 
        //添加maven本地仓库
        mavenLocal()
        
        //添加发布时指定的maven远程仓库
        maven {
            url uri('../repo')
        }
        
        //可以添加maven中央仓库
        //mavenCentral()
    }

    dependencies {
        //classpath填写插件的GAV坐标，gradle编译时会扫描该classpath下的所有jar文件并引入
        classpath 'com.example.customplugin:myplugin:1.0'
    }
}
```

这里使用了Project的**buildscript**方法，通过该方法可以为当前项目的build.gradle脚本导入外部依赖，在该方法中可以使用repositories方法和dependencies方法指定外部依赖的所在仓库和类路径，当前项目在构建时会去repositories定义的仓库地址寻找classpath指定路径下的所有jar文件并引入，如我这里repositories方法中指定了mavenLocal()和maven {url uri('../repo')}，其中mavenLocal()对应maven的本地仓库即/.m/repository，maven {url uri('../repo')}就是刚刚发布时指定的maven远程仓库，如果你通过发布任务发布插件到了maven中央仓库，则可以新增mavenCentral()，它代表maven中央仓库的远端地址，而dependencies方法中则通过classpath指定了myplugin插件在仓库下的类路径即GAV坐标: com.example.customplugin:myplugin:1.0，现在我们可以通过apply plugin使用myplugin了，在subproject_4/build.gradle中添加如下：

```groovy
//subproject_4/build.gradle

//通过插件id引用插件
apply plugin: 'myplugin'

//使用DSL配置插件的属性
outerExt{
    message 'hello'

    inner{
        message 'word'
    }
}

//执行gradle showExt，输出：
//outerExt = [message = hello], innerExt = [message = word]
```

其中apply plugin后面的插件id就是我们在gradle_plugin/src/main/resources/META-INF/gradle-plugins目录下编写的properties文件的名称。

现在已经成功发布了一个插件并引入使用，但是上面的发布过程和引入插件过程不免有点繁琐，Gradle为我们提供了[java-gradle-plugin](https://docs.gradle.org/current/userguide/java_gradle_plugin.html#java_gradle_plugin)用于简化我们的发布过程，同时根据java-gradle-plugin发布的插件使用时只需要直接引入插件id就行，而无需在buildscript中指定classpath。

#### 2.2、使用java-gradle-plugin生成META-INF发布

还是上面MyPlugin的例子，我们在gradle_plugin/build.gradle中引入java-gradle-plugin插件，通过java-gradle-plugin提供的**gradlePlugin{}**扩展配置插件的META-INF信息，同时通过Project的**setGroup**方法和**setVersion**方法定义插件发布的groupId和version，我们还要引入maven-publish插件，因为要依赖它提供的发布能力，如下:

```groovy
//gradle_plugin/build.gradle

plugins{
    //用来生成插件元数据(META-INF)和插件标记工件(Plugin Marker Artifact)
    id 'java-gradle-plugin'
    //用来生成发布插件任务
    id 'maven-publish'
}

//通过maven-publish提供的publishing.repositories{}定义发布的maven仓库位置
publishing.repositories{
    maven {
        url '../repo'
    }
}

//插件发布的groupId
group = 'com.example.customplugin'

//这里没有定义artifactId，maven-publish会自动取当前项目的名字作为artifactId，这里为gradle_plugin

//插件发布的version
version = '2.0'

//通过java-gradle-plugin提供的gradlePlugin{}配置插件
gradlePlugin {
    plugins {
        //配置myplugin插件
        myplugin{
            //插件id
            id = 'com.example.customplugin.myplugin'
            //插件实现类
            implementationClass = 'com.example.plugin.MyPlugin'
        }

        //myplugin2{
        //    id = 'xxx'
        //    implementationClass = 'xxx'
        //}
        //以此类推可以配置多个插件
    }
}
```

通过在gradlePlugin{}中的配置，执行发布任务时java-gradle-plugin会自动为我们生成插件的META-INF信息并打包进插件的jar中，同时java-gradle-plugin会自动为我们在dependencies引入gradleApi()，除此之外，执行发布任务时java-gradle-plugin还为插件生成了一个**插件标记工件 **- [Plugin Marker Artifact](https://docs.gradle.org/current/userguide/plugins.html#sec:plugin_markers)，它的作用就是用来定位插件的位置，我们在**plugins DSL**中通过插件id来引用插件时并不需要定义插件的classpath即GAV坐标就可以直接使用插件，那么Gradle是如何根据插件id定位到插件的呢？答案就是插件标记工件，它跟插件发布时一起发布，它里面只有一个pom文件，pom文件里面依赖了插件真正的GAV坐标，这样通过插件id来引用插件时就会先下载这个pom文件，从而解析到插件的GAV坐标，再根据插件的GAV坐标把插件的jar文件引入，那么Gradle又是如何定位到插件标记工件的呢？答案就是使用插件id按一定的规则生成插件标记工件的GAV坐标，插件标记工件的GAV生成规则为**pluginId:pluginId.gradle.plugin:pluginVersion**，如这里myplugin的插件标记工件生成的GAV为com.example.customplugin.myplugin:com.example.customplugin.myplugin.gradle.plugin:2.0，发布插件时会同时把插件标记工件发布到生成的GAV处，然后通过插件id引用时又根据插件id拼接出相同的GAV从而定位到插件标记工件，这么说有点绕，让我们执行发布任务看看生成的产物就懂了。

我们在Gradle项目所处目录的命令行输入`gradle publishPluginMavenPublicationToMavenRepository publishMypluginPluginMarkerMavenPublicationToMavenRepository`来执行插件发布任务和插件标记工件发布任务，这两个任务都是maven-publish根据java-gradle-plugin定义的发布过程为我们生成好的，除此之外还生成了发布到maven本地仓库的任务，这里只以发布到maven远程仓库任务做示例，两个任务执行成功后，在GradleDemo/repo/中会看到发布的插件和插件标识工件，如下：

{% asset_img gradle9.png gradle1 %}

可以看到在repo下发布了两个组件，一个为插件，artifactId为gradle_plugin，一个为插件标识工件，artifactId为com.example.customplugin.myplugin.gradle.plugin，它们都处于com.example.customplugin空间下，版本都为2.0，我们看一下插件标识工件里面的内容，如下：

{% asset_img gradle10.png gradle1 %}

可以看到插件标识工件中并没有jar文件只有pom文件，我们打开它的pom文件，如下：

{% asset_img gradle11.png gradle1 %}

可以看到依赖了真正的插件的GAV坐标，为com.example.customplugin:gradle_plugin:2.0，这样通过插件id引用时就可以根据插件标识工件定位到插件，通过java-gradle-plugin这种方式发布插件时还有一点要注意的是，插件的id最好以groupId为前缀，这样才能保证插件标识工件和插件发布时都处于同一个仓库路径。

现在MyPlugin 2.0已经发布成功了，接下来让我们使用这个插件，需要通过plugins DSL引用的插件不需要定义classpath，但还是要定义仓库位置，我们在settings.gradle中添加如下：

```groovy
//一定要放在settings.gradle中的第一行
pluginManagement{
    repositories{
        //添加发布时指定的maven远程仓库
        maven {
            url uri('repo')
        }
    }
}
```

plugins DSL通过**pluginManagement{}**管理插件仓库还有插件，而且pluginManagement{}必须要放在settings.gradle脚本的头部，然后我们可以通过plugins DSL使用myplugin了，在subproject_4/build.gradle中添加如下：

```groovy
//通过插件id引用插件
 plugins{
 	id 'com.example.customplugin.myplugin' version '2.0'
 }

//使用DSL配置插件的属性
outerExt{
	message 'hello'

	inner{
		message 'word'
	}
}

//执行gradle showExt，输出：
//outerExt = [message = hello], innerExt = [message = word]
```

plugins DSL根据插件id寻找插件时默认会先从Gradle中央仓库 - [Gradle Plugin Portal](https://plugins.gradle.org/)找，找不到再从pluginManagement中指定的仓库找。

> 上面发布时为了讲解原理把发布任务逐个执行，其实maven-publish还为我们生成了更为方便的publishAllPublicationsToMavenRepository 、publishToMavenLocal、publish发布任务，只需要执行一个任务就可以把多个发布任务同时执行：
>
> 执行publishAllPublicationsToMavenRepository任务表示执行所有发布到maven远程仓库的任务；
>
> 执行publishToMavenLocal任务表示执行所有发布到maven本地仓库的任务；
>
> 执行publish任务表示执行publishAllPublicationsToMavenRepository和publishToMavenLocal任务

## 结语

本文通过Gradle的特点、项目结构、生命周期、Task、自定义Plugin等来全面的学习Gradle，掌握上面的知识已经足够让你入门Gradle，但是如果你想要更进一步的学习Gradle，掌握本文的知识点是完全不足的，你可能还需要熟悉掌握Project中各个api的使用、能独立的自定义一个有完整功能的Plugin、能熟练地编写Gradle脚本来管理项目等，下面有一份Gradle DSL Reference，你可以在需要来查找Gradle相关配置项的含义和用法：

[Gradle DSL Reference](https://docs.gradle.org/current/dsl/index.html)

对于android开发者，我们引入的android插件中也有很多配置项，下面的Android  Plugin DSL Reference，你可以在需要来查找android 插件相关配置项的含义和用法：

[Android Plugin DSL Reference](https://developer.android.com/reference/tools/gradle-api)

本文的源码位置：

[GradleDemo](https://github.com/rain9155/GradleDemo)

以上就是本文的全部内容，希望大家有所收获！

参考内容：

[Gradle官网](https://docs.gradle.org/current/userguide/userguide.html)

[Gradle学习系列](https://www.cnblogs.com/davenkin/p/gradle-learning-1.html)

[Gradle插件从入门到进阶](https://juejin.im/post/5ccf02e36fb9a0322e73a3db#heading-52)

[Gradle 源码分析](https://juejin.cn/post/6844903858641043463#heading-30)



















