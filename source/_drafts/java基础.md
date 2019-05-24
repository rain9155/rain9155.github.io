---
title: java基础
tags: java
categories: java
---

## 前言
万事开头难，准备从零把java相关知识点捡起来，把自己所学的Java知识点归纳，下面是关于java的一些基本知识点。

## java代码的运行过程
![](https://img-blog.csdnimg.cn/20190330152933530.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JhaW5fOTE1NQ==,size_16,color_FFFFFF,t_70)
* 创建java源程序，扩展名为.java
* 使用javac命令编译源程序为字节码文件，扩展名为.class
* 使用java命令运行字节码文件，在不同平台执行
## 数据类型
下面用一张表概括：
|   数据类型   |    类型说明符    | 位数 | 字节 |
| :----------: | :--------------: | :--: | :--: |
|     整形     |       int        |  32  |  4   |
|    短整型    |      short       |  16  |  2   |
|    长整形    |       long       |  64  |  8   |
|    字节型    |       byte       |  8   |  1   |
| 单精度浮点型 |      float       |  32  |  4   |
| 双精度浮点型 |      double      |  64  |  8   |
|   布尔类型   |     boolean      |  8   |  1   |
|   字符类型   |       char       |  16  |  2   |
|  字符串类型  |      String      |  -   |  -   |
|  自定义类型  | public class ... |  -   |  -   |
其中java的数据类型又分为：
* 基本类型/原始类型（primitive type）：用来保存简单的单个数据，如：int、short、long、byte、float、double、boolean、char共8种。
* 类类型/引用类型（class type or reference types）：用来保存复杂的组合数据，如String和自定义类型。
> ps: 在java中，char类型实际是一个16位的无符号整数（<=65535）,可以保存中文和转义字符(`\b`,`\t`,`\n`等)
## 成员变量和局部变量
```java
public class 类名{

    数据类型 变量1；//成员变量
    
    返回值类型 方法名(){
    
        数据类型 变量2；//局部变量
        
    }
    
}
```
1、成员变量的作用域在整个类都是可见的。
2、局部变量的作用域仅在定义它们的方法中可见。
3、成员变量有默认的初始值（数字为0，对象为null）。
4、局部变量没有默认的初始值，需要赋初值再使用。
5、成员变量和局部变量重名时，局部变量的优先级更高。

## 字符串
### 1、String类型
在java中String是引用类型，它的构造器如下：
![](https://img-blog.csdnimg.cn/20190330153012958.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JhaW5fOTE1NQ==,size_16,color_FFFFFF,t_70)了用以上new的方法创建一个字符串，还可以用以下方式：
```java
String name1 = "rain";
String name2 = "rain";
```
那么用`new`和用`=`有什么不同的呢? new出来字符串是一个String对象，它被放在堆中，地址不一样。用=赋值的字符串是从字符串池（String Pool，保存着所有字符串字面量，这些字面量在编译时期就确定）中拿的，如果这个字符串在池中没有，就会先放进池，所以上面两个name1和name2是同一个字符串。
**String中常用的方法：**
![](https://img-blog.csdnimg.cn/20190330153103238.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JhaW5fOTE1NQ==,size_16,color_FFFFFF,t_70)
**String的比较：**
![](https://img-blog.csdnimg.cn/20190330153115503.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JhaW5fOTE1NQ==,size_16,color_FFFFFF,t_70)
**获取String的字串的方法：**
![](https://img-blog.csdnimg.cn/20190330153140466.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JhaW5fOTE1NQ==,size_16,color_FFFFFF,t_70)

> ps: 
> 1、在字符串的比较中`==`是用来比较地址的，String的equals方法才是用来比较两个字符串是否相等 
> 2、字符串的拼接可以使用`+`,这个运算符的底层其实是调用了String的concat方法，返回一个新的字符串
>
> 3、在 Java 7 之前，String Pool 被放在运行时常量池中，它属于永久代。而在 Java 7，String Pool 被移到堆中。这是因为永久代的空间有限，在大量使用字符串的场景下会导致 OutOfMemoryError 错误
>
> 4、在 Java 8 中，String 内部使用 char 数组存储数据，在 Java 9 之后，String 类的实现改用 byte 数组存储字符串，这些数组在都用final修饰，所以才保证String是不可变的

### 2、StringBuffer和StringBuilder
与String类不同的是，StringBuffer和StringBuilder的对象可以被多次修改，并且不产生新的未使用对象，而String是不可变的。StringBuilder是在JDK5中被提出来的，它和StringBuffer之间最大的不同是StringBuilder的方法都不是线程安全的，而StringBuffer的方法都是线程安全的。
**StringBuffer的构造方法（StringBuilder类似）：**
![](https://img-blog.csdnimg.cn/20190330153158130.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JhaW5fOTE1NQ==,size_16,color_FFFFFF,t_70)
**StringBuffer的常用方法：**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190330153404711.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JhaW5fOTE1NQ==,size_16,color_FFFFFF,t_70)
### 3、 思考：
分别使用+、StringBuffer和StringBuilder来拼接字符串，各自的效率怎么样？
## 包装类
包装类就是将java的基本数据类型打包成对象处理，包装类都在java.lang包中，下面用一个表显示：
| 基本数据类型 |  包装类   |
| :----------: | :-------: |
|     int      | Interger  |
|    short     |   Short   |
|     long     |   Long    |
|     char     | Character |
|     byte     |   Byte    |
|    float     |   Float   |
|    double    |  Double   |
|   boolean    |  Boolean  |
它涉及到以下两种操作：
### 1、装箱(boxing)
以double装箱为例：
```java
double num = 3.14；
Double dnum1 = new Double(num);//1
Double dnum2 = Double.valueOf(num);//2
Double dnum3 = num;//3
```
注释1、2都是手动装箱，注释3是自动装箱。
### 2、拆箱(unboxing)
同样上述double拆箱为例：
```java
num = dnum1;//1
num = dnum2.doubleValue;//2
```
注释1是自动拆箱，注释2是手动拆箱。
> ps:
> 1、包装类没有无参构造，所有包装类的实例都是不可变的，一旦创建对象后，它们的内部值就不能再改变。
> 2、每个基本类型包装类都有常量MAX_VALUE和MIN_VALUE。
>
> 3、每个包装类都会有一个默认大小的缓存池，例如Integer，缓存池默认大小是-128-127
>
> 3、编译器会在自动装箱过程中会调用 valueOf() 方法，因此多个值相同，且值在缓存池范围内的包装类实例使用自动装箱来创建，那么就会引用相同的对象。

## 关键字

### 1、final

防止扩展和重写。

* 修饰成员变量：常量（可以是编译时常量，也可以是在运行时被初始化后不能被改变的常量），不可更改（对于基本数据类型，final使数值不能改变，对于引用类型，final使引用不能改变，即不能引用其他对象，但引用本身可以更改）
* 修饰方法：不可被重写
* 修饰类：不可被继承

### 2、static

可以通过类名直接访问它修饰的属性，静态属性和方法都是优先于类的实例存在。

* 修饰变量：称为静态变量（区别于实例变量）、类变量，类的所有实例都共享静态变量，静态变量在内存中只存在一份
* 修饰方法：称为静态方法，静态方法必须有实现，它不依赖于任何实例，静态方法中只能调用类的静态属性和静态方法，方法中不能有 this 和 super 关键字
* 修饰语句块：称为静态语句块，在类初始化时运行一次
* 修饰内部类：称为静态内部类，非静态内部类依赖于外部类的实例，而静态内部类不需要

存在继承的情况下，初始化顺序为：

父类(静态变量、静态语句块) -> 子类（静态变量、静态语句块） -> 父类（实例变量、普通语句块）-> 父类（构造函数）-> 子类（实例变量、普通语句块） -> 子类（构造函数）

## 关于hashCode（）和equal（）方法

hashCode()用来返回hash值，而equal()是用来判断两个对象是否等价，所以：

* 要比较两个对象是否相等，必须使用equal方法，如果相等，那么调用两个对象的 hashCode 方法必须返回相同的结果

* 如果两个对象根据 equals方法比较是不相等的，则 hashCode 方法不一定得返回不同的整数

* 对同一个对象调用多次hashcode方法必须返回相同的hash值

* 两个不同对象的hashcode可能相等

* 两个不同hashcode的对象一定不相等

> ps:  在使用hashXX集合添加对象时，集合先调用该对象的equal方法查看当前集合是否有与之相等的对象，如果有，就再调用该对象的hashCode方法返回hash值，看是否与相等的对象的hashCode方法返回的hash值相等，如果相等，就不添加这个对象，否则就添加，因此在覆盖 equals() 方法时应当总是覆盖 hashCode() 方法，保证等价的两个对象hash值也相等，不然会导致集合中出现重复的元素。

## 异常

异常就是一种对象(Exception), 表示阻止程序正常执行的错误。异常类的层次结构如下：

{% asset_img error1.png error1 %}

* 1、RuntimeException和Error以及它们的子类都称为免检异常
* 2、除了免检异常，其他异常都称为必检异常

由于免检异常可能在程序的任何一个地方出现，为了避免过多的使用try-catch块，java语言不强制要求编写代码捕获免检异常，也不要求在方法头显示声明免检异常。

### 1、常见的异常类型

{% asset_img error2.png error2 %}

### 2、java中的异常处理机制

异常处理机制就是可以使程序处理非预期的情景，并继续正常执行，异常处理机制的主要组成如下：

* try：监控有可能产生异常的语句块
* catch：以合理的方式捕获异常
* finally：不管有没有异常，一定会执行的语句块（一般用来释放资源），除了遇到System.exit(0)语句
* throw：手动引发异常
* throws: 指定由方法引发的异常

所以一个异常捕获处理语句可以如下形式：

```java
try{
    //监控可能产生异常的语句块
}catch(Exception1 e){
    //捕获异常，处理异常，如打印异常信息，日志纪录
}catch(Exception2 e){
    //JDK7后简化写法catch(Exception1|Exception2|Exception3|... e)
}finally{
    //不管有无异常，一定会执行的语句，用来释放资源
}
```

try块中的代码可能会引发多种类型的异常，当引发异常时，会按照catch的顺序进行匹配异常类型，并执行第一个匹配的catch语句

## 结语

本文简单介绍了java语言的基本知识点，希望大家有所收获。