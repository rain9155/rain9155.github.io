---
title: java学习总结之面向对象
tags: 面向对象
categories: java
date: 2019-07-19 12:29:50
---

## 前言
* 上一篇文章[java基础](https://rain9155.github.io/2019/07/19/java%E5%9F%BA%E7%A1%80/#more)
  本篇文章继续Java知识点的归纳，梳理一下关于面向对象的知识点，涉及到封装、继承、多态，还有接口，类之间的关系。

<!--more-->

## 接口和抽象类

### 1、抽象类

抽象类和抽象方法都用abstract关键字进行声明，抽象类不能被实例化，不能直接创建，抽象方法必须放在抽象类中，类中如果有抽象方法，这个类继续声明为抽象类，如下：

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

接口被认为是一种特殊的抽象类，同样不能使用new实例化，接口的字段和方法都默认为public，且不能声明为private或protected，接口只能包含常量(static final)和待实现的方法，java8以后接口中可以有方法的实现，如下：

```java
interface Eat{
    //...
    default public void eating(){
        System.out.println("eating");
    }
}
```

> 在接口中，也可以定义内部类，但是只能是public static修饰的内部类，所以接口中只能定义静态内部类，但是在接口中定义内部类的用处不大，很少使用。

### 3、接口和抽象类的比较

|        |               变量                |           成员方法            | 构造方法 |               使用场合               |
| :----: | :-------------------------------: | :---------------------------: | :------: | :----------------------------------: |
| 抽象类 |              无限制               |            无限制             |  可以有  |       强的“is a”关系（是一种）       |
|  接口  | 所有变量必须是public static final | 所有方法必须是public abstract |    无    | 弱的“is a”关系（is kind of，是一类） |

在很多情况下，接口优先于抽象类。因为接口没有抽象类严格的类层次结构要求，可以灵活地为一个类添加行为。并且从 Java 8 开始，接口也可以有默认的方法实现，使得修改接口的成本也变的很低，而且接口可以实现多继承。

## 外部类和内部类

把一个类放在另外一个类的内部，处于类内部的类就叫做内部类，包含内部类的类就叫做外部类，一个外部类不一定有内部类，但是一个内部类就一定有一个”依附“的外部类，内部类比外部类多使用3个修饰符：private、protected、static，外部类不能使用这3个修饰符，虽然外部类和内部类整体是一个类文件，但是编译之后，它会分别生成外部类的class文件和内部类的class文件，这说明在运行时外部类和内部类其实是两个独立的类，内部类可以分为：**非静态内部类**、**静态内部类**、**局部内部类**、**匿名内部类**。

### 1、非静态内部类

```java
public class OuterClass {
    
    private String outer = "外部类私有变量";
    private void outerMethod(){
        System.out.println("外部类私有方法" );
    }

    public class InnerClass{
        
        public void innerMethod(){
            //非静态内部类调用外部类的私有方法
            outerMethod();
            //非静态内部类访问外部类的私有变量
            System.out.println(outer);
        }
        
    }
}
```

没有使用static修饰的就是非静态内部类，它依赖于**外部类实例**，要创建非静态内部类实例，必须先创建外部类实例，如下：

```java
public static void main(String[] args) {
    OuterClass outerClass = new OuterClass();
    //通过 外部类实例 调用非静态内部类的构造器
    OuterClass.InnerClass innerClass = outerClass.new InnerClass();
    innerClass.innerMethod();
}
```

总的来说，非静态内部类有以下几个特点：

- 1、非静态内部类属于外部类实例，必须依赖外部类实例才能够创建；
- 2、非静态内部类编译之后，会保留一个外部类对象的引用(在构造方法中传入)；
- 3、非静态内部类不能含有静态成员、静态方法和static语句块；
- 4、非静态内部类中可以直接访问外部类的所有成员和方法(包括private的)，但是在外部类中不能直接访问内部类的任何成员和方法，必须先创建实例后才能访问(包括private的)。

> 如果外部类和内部类的成员变量重名，可以通过**this**和**外部类.this**区分。

### 2、静态内部类

```java
public class OuterClass {

    private static String outer = "外部类私有变量";
    private static void outerMethod(){
        System.out.println("外部类私有方法" );

    }

    public static class InnerClass{

        public void innerMethod(){
            //静态内部类调用外部类的私有静态方法
            outerMethod();
            //静态内部类访问外部类的私有静态变量
            System.out.println(outer);
        }
        
    }
}
```

使用static修饰的就是静态内部类，它依赖于**外部类**，创建静态内部类实例时，不需要先创建外部类实例，因为静态内部类依赖的是类而不是类实例，如下：

```java
public static void main(String[] args) {
    //通过 外部类 调用静态内部类的构造器
    OuterClass.InnerClass innerClass = new OuterClass.InnerClass();
    innerClass.innerMethod();
}
```

总的来说，静态内部类有以下几个特点：

- 1、静态内部类属于类，而不属于类实例；
- 2、静态内部类编译之后，不会含有外部类对象的引用；
- 3、静态内部类可以包含静态成员、静态方法和static语句块，也能包含非静态的；
- 4、静态内部类不能直接访问外部类的实例成员或方法，只能访问静态成员或方法(包括private的)，外部类可以直接通过类名访问静态内部类的静态成员或方法(包括private的)，如果要访问实例成员或方法，必须先创建实例后才能访问(包括private的)。

> 外部类和内部类之间要访问对方的private/protected成员时，编译器会自动生成合适的“access method”静态方法来提供合适的可访问性，这样就绕开了原本的成员的可访问性不足的问题。

### 3、局部内部类

```java
public class OuterClass {

    public static void main(String[] args) {
        //局部内部类
       class InnerClass{
            private String inner = "内部类私有变量";
            private void innerMethod(){
                System.out.println("内部类私有方法");
            }
       }

       InnerClass innerClass = new InnerClass();
       //访问局部内部类的私有变量
       System.out.println(innerClass.inner);
       //调用局部内部类的私有方法
        innerClass.innerMethod();
    }
}
```

把一个类在方法里定义，这个类就是局部内部类，局部内部类只能在方法里面使用，它的作用域在方法之内，由于局部内部类不能在方法之外使用，所以它不能使用static和任何访问修饰符修饰，局部内部类在开发中很少用到，因为它的作用域太小了，不能被其他方法复用。

### 4、匿名内部类

```java
public class OuterClass {

    private void outerMethod(){
        //创建匿名内部类
        InnerClass innerClass = new InnerClass(){
            @Override
            public void innerMethod() {
                System.out.println("内部类私有方法");
            }
        };
        //调用匿名内部类的方法
        innerClass.innerMethod();
    }

    public abstract class InnerClass{
        int num = 1;
        abstract void innerMethod();
    }
}
```

匿名内部类就是没有名字的内部类，它没有使用class关键字来定义类，而是在使用时直接创建接口或抽象父类的实现类来使用，匿名内部类一般在只使用一次的场景下使用，总的来说，匿名内部类有以下几个特点：

- 1、匿名内部类不能继续声明为抽象类，它必须实现接口或抽象父类的所有抽象方法；
- 2、匿名内部类不能定义构造器，因为它没有名字，但是它可以使用实例语句块来完成初始化；
- 3、匿名内部类必须实现一个接口或抽象父类，但最多只能实现一个接口或抽象父类；
- 4、匿名内部类编译之后，会保留一个外部类对象的引用。

> 在java8之前，局部内部类或匿名内部类访问方法的局部变量时，这个局部变量必须使用final修饰，在java8之后，被局部内部类或匿名内部类访问的局部变量编译器会自动加上final，无需显式声明，但是如果局部变量在内部类中被修改，那么它还是要显式声明final，总之，在局部内部类或匿名内部类中使用局部变量时，必须按照有final修饰的方式来用(一次赋值，以后不能重复赋值)。

## 面向对象三大特性

面向对象是一种程序设计方法，它的基本思想是封装、继承、多态，根据现实世界中各种事物的本质特点，把它们抽象成类，作为系统的基本构成单元，然后这些类可以生成系统中的多个对象，而这些对象则是映射成客观世界的各种事物的实例。

### 1、封装

封装就是尽可能地隐藏对象内部的实现细节，只保留一些对外接口使之与外部发生联系。用户无需知道对象内部的细节，但可以通过对象对外提供的接口来访问该对象。

### 2、继承

继承是面向对象实现复用的手段，继承是一种“is a”关系，父类和子类之间必须存在“is a”关系，通过继承，子类可以获得父类的所有非 private 属性和方法，父类的私有属性在子类中不能直接访问，例如Cat 和 Animal 就是一种 “is a” 关系，因此 Cat 可以继承自 Animal，从而获得 Animal 非 private 的属性和方法。

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

* 方法重写发生在通过继承而相关的不同类中；
* 方法重写具有相同的方法签名和返回值；
* 子类重写方法时子类方法访问权限大于父类的；
* 子类重写方法时子类抛出的异常类型是父类抛出异常的子类；
* @Overiide称为重写标注，用来保证重写的方法和原方法的签名和返回值一致。

> 方法重载：方法重载是指在于同一个类中，一个方法与已经存在的方法名称上相同，但是参数类型、个数、顺序至少有一个不同。应该注意的是，返回值不同，其它都相同不算是重载。

### 3、多态

多态是指子类对象可以直接赋值给父类变量，但是运行时依然表现出子类的行为特征，这表示同一个类型的对象在运行时可以具有多种形态，多态分为以下两种：

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

[为什么内部类的private变量可被外部类直接访问？](https://www.zhihu.com/question/54730071)

