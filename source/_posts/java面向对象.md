---
title: java面向对象
tags: java
categories: java
date: 2019-07-19 12:29:50
---

## 前言
* 上一篇文章[java基础](https://rain9155.github.io/2019/07/19/java%E5%9F%BA%E7%A1%80/#more)
  本篇文章继续Java知识点的归纳，梳理一下关于面向对象的知识点，涉及到封装、继承、多态，还有接口，类之间的关系。

<!--more-->

## 接口和抽象类

### 1、抽象类

抽象类和抽象方法都用abstract关键字进行声明，抽象类不能被实例化，不能直接创建，抽象方法必须放在抽象类中。

```java
public abstract class Hero{
    public abstract void fight();
}

//子类实现抽象方法，完成实际操作
public class Warrior extends Hero{
    @Override
    public void fight(){}
}

//子类继续声明为抽象类
public abstract class LongRange extends Hero{}
```

### 2、接口

接口被认为是一种特殊的抽象类，同样不能使用new实例化，包含常量和待实现的方法，java8以后接口中可以有方法的实现，如下：

```java
interface Eat{
    //...
    default public void eating(){
        System.out.println("eating");
    }
}
```

### 3、接口和抽象类的比较

|        |               变量                |           成员方法            | 构造方法 |               使用场合               |
| :----: | :-------------------------------: | :---------------------------: | :------: | :----------------------------------: |
| 抽象类 |              无限制               |            无限制             |  可以有  |       强的“is a”关系（是一种）       |
|  接口  | 所有变量必须是public static final | 所有方法必须是public abstract |    无    | 弱的“is a”关系（is kind of，是一类） |

在很多情况下，接口优先于抽象类。因为接口没有抽象类严格的类层次结构要求，可以灵活地为一个类添加行为。并且从 Java 8 开始，接口也可以有默认的方法实现，使得修改接口的成本也变的很低，而且接口可以实现多继承。

## 面向对象三大特性

### 1、封装

尽可能地隐藏对象内部的实现细节，只保留一些对外接口使之与外部发生联系。用户无需知道对象内部的细节，但可以通过对象对外提供的接口来访问该对象。

### 2、继承

继承是一种“is a”关系，父类和子类之间必须存在“is a”关系，父类的私有属性在子类中不能直接访问，例如Cat 和 Animal 就是一种 “is a” 关系，因此 Cat 可以继承自 Animal，从而获得 Animal 非 private 的属性和方法。

#### 2.1、父类构造和子类构造

* 构造方法不可继承，使用super关键字调用父类构造
* 默认会先调用父类构造，再调用子类构造

#### 2.2、子类调用父类信息

* 使用super关键字
* 可以调用父类的公有属性和方法
* 可以调用父类的protected属性和方法

下面一张表给出java中访问权限修饰符的访问范围：

|  修饰符   | 在同一类中可访问 | 在同一包内可访问 | 在子类内可访问 | 在不同包可访问 |
| :-------: | :--------------: | :--------------: | :------------: | :------------: |
|  public   |       可以       |       可以       |      可以      |      可以      |
| protected |       可以       |       可以       |      可以      |       --       |
|  default  |       可以       |       可以       |       --       |       --       |
|  private  |       可以       |        --        |       --       |       --       |

#### 2.3、方法重写

在子类中提供一个对方法的新的实现。

* 方法重写发生在通过继承而相关的不同类中
* 方法重写具有相同的方法签名和返回值
* 子类重写方法时子类方法访问权限大于父类的
* 子类重写方法时子类抛出的异常类型是父类抛出异常的子类。
* @Overiide称为重写标注，用来保证重写的方法和原方法的签名和返回值一致

ps:方法重载：方法重载是指在于同一个类中，一个方法与已经存在的方法名称上相同，但是参数类型、个数、顺序至少有一个不同。应该注意的是，返回值不同，其它都相同不算是重载。

### 3、多态

同一个实体，具有多种形式，多态分为以下两种：

* 编译时多态：主要指方法的重载
* 运行时多态：指程序中定义的对象引用所指向的具体类型在运行期间才确定（运行时多态有三个条件：继承，方法重写，向上转型）

例如下面的代码中，子类Warrior**继承**父类Hero，它**重写**了父类的fight方法，并且在main函数中父类Hero引用子类Warrior（父类引用指向子类对象称为**向上转型**），在Hero引用调用fight方法时，会执行实际对象所在类的fight方法，即Warrior类的fight方法，而不是Hero的fight方法。

```java
public class Hero{
    public void fight(){
        System.out.println("hero");
    }
}

public class Warrior extends Hero{
    @Override
    public void fight(){
        System.out.println("Warrior");
    }
}

public static void main(String[] args) {
    Hero warrior = new Warrior();
    warrior.fight();
}

输出：Warrior
```

## 类图

了解下面6种关系有助于看懂UML图。

### 1、泛化关系（Generalization）

泛化关系用一条带空心箭头的直线表示，在类图中表示为类的继承关系（“is a”关系），在java中用extends关键字表示，最终代码中，泛化关系表现为继承非抽象类。例如下面ASUS继承自Laptop，ASUS是一台笔记本，ASUS与Laptop之间是泛化关系。

{% asset_img generalization.png generalization %}

### 2、实现关系（Realization）

实现关系用一条带空心箭头的虚线表示，在类图中表示实现了一个接口（在java中用implements 关键字表示），或继承抽象类，实现了抽象类中的方法。例如下面Laptop实现了IO接口，同时它继承Computer这个抽象类，Laptop是它们的具体实现。

{% asset_img realization.png realization %}

### 3、聚合关系（Aggregation）

聚合关系用一条带空心菱形箭头的直线表示，表示整体是由部分组成的，但是整体和部分之间并不是强依赖的，整体不存在了，部分还是会存在。例如下面表示Staff聚合到Department，或者说部门是由员工组成的，部门不存在了，员工还是会存在的。

{% asset_img aggregation.png aggregation %}

### 4、组合关系  ( Composition )

组合关系是用一条带实心菱形箭头的直线表示，和聚合关系不同，组合关系中整体和部分是强依赖的，即整体不存在了部分也不存在，组合关系是一种强依赖的特殊聚合关系。例如下面表示Department组合到Company中，或者说Company是由Department组成的，但是公司不存在了，部门也将不存在。

{% asset_img composition.png composition %}

### 5、关联关系（Association）

关联关系用一条直线表示，表示不同类对象之间有关联，这是一种静态关系，与运行过程的状态无关，在最开始就可以确定。因此也可以用 1 对 1、多对 1、多对多这种关联关系来表示.。例如下面学生和学校就是一种关联关系，一个学校可以有很多学生，但是一个学生只属于一个学校，因此这是一种多对一的关系，在运行开始之前就可以确定。

{% asset_img association.png association %}

关联关系默认不强调方向，表示对象之间相互知道，如果要特别强调方向，如下图，表示A知道B，但是B不知道A，这又叫DirectedAssociation。

{% asset_img directedassociation.png directedassociation %}

```
ps: 在最终代码中，关联对象通常以成员变量的形式存在。
```

### 6、依赖关系（Dependency）

依赖关系用一条带箭头的虚线表示，与关联关系不同的是，他描述一个对象在运行期间会用到另一个对象的关系，是一种动态关系，并且随着运行时的变化， 依赖关系也可能发生变化。依赖也有方向，但是我们总是应该保持单向依赖，避免双向依赖的产生。例如下面表示A依赖于B，A的一个方法中使用到了B作为参数。

{% asset_img dependency.png dependency%}

```
ps: 在最终代码中，依赖关系主要表现为：
1、A 类是 B 类方法的局部变量；
2、A 类是 B 类方法或构造的传入参数；
3、A 类向 B 类发送消息，从而影响 B 类发生变化;
```

箭头的指向为调用关系。

## 结语

本文都是关于面向对象的一些知识，虽然简单，但是也挺繁琐的，积少成多，希望大家阅读过后有所收获。

参考资料：

[看懂UML类图和时序图](https://www.jianshu.com/p/00cb3c4e6336)

