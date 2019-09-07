---
title: Builder模式
date: 2019-09-07 19:35:25
tags: 设计模式
categories: 设计模式
---

## 前言

Builder模式是一步步创建一个复杂对象的创建型模式，它允许用户在不知道内部构建细节的情况下，可以更加精准的控制对象的构造过程，为了在构建过程中，对外部隐藏实现细节，就可以使用Builder模式将部件和组装过程分离，使得构建过程和部件可以自由扩展，两者之间的耦合度也降到最低。

<!--more-->

## 定义

将一个复杂对象的构建和它的表示分离，使得同样的构建过程可以创建不同的表示。

## 使用场景

（1）相同的方法不同的执行顺序产生不同的事件结果时

（2）多个部件或零件，都可以装配到一个对象中，但是产生的运行结果又不同时

（3）产品类比较复杂或者产品类中的调用顺序不同产生不同的作用时

（4）当初始化一个对象非常复杂，如参数非常多，且很多参数都具有默认值时

## 类图

{% asset_img design1.png design %}

类图介绍：

- Produc - 产品的抽象类
- Builder - 抽象的Builder类，规范产品的组建，一般由子类实现具体的组建过程
- ConcreteBuilder - 具体的Builder类
- Director - 统一组装过程

## 简单实现

计算机的组装过程比较复杂且组装顺序说不固定的，下面把计算机的组装过程简化为构建主机，设置操作系统，设置显示器3部分，然后通过Director和具体Builder来构建计算机对象。

计算机抽象类，即Product

```java
public abstract class Computer {

    protected String mBroad;//主板
    protected String mDisplay;//显示器
    protected String mOS;//操作系统

    protected Computer(){}

    /**
     * 设置主板
     * @param broad
     */
    public void setmBroad(String broad){
        mBroad = broad;
    }

    /**
     * 设置显示器
     * @param display
     */
    public void setmDisplay(String display){
        mDisplay = display;
    }

    /**
     * 设置操作系统
     */
    public abstract void setmOS();

    @Override
    public String toString() {
        return "Computer {mBroad = " + mBroad + ", mDisplay = " + mDisplay + ", mOS = " + mOS + "}";
    }
}
```

苹果电脑，具体的Product

```java
public class Macbook extends Computer {

    protected Macbook() {}

    @Override
    public void setmOS() {
        mOS = "Mac OS X 10.10";
    }
}
```

抽象Builder类 

```java
public abstract class Builder {

    public abstract void buildBroad(String broad);//设置主机
    public abstract void buildDisplay(String display);//设置显示器
    public abstract void buildOS();//设置操作系统
    public abstract Computer create();//创建Computer

}
```

具体的Builder类，构造苹果电脑 

```java
public class MacBuilder extends Builder {

    private Computer mComputer = new Macbook();

    @Override
    public void buildBroad(String broad) {
        mComputer.setmBroad(broad);
    }

    @Override
    public void buildDisplay(String display) {
        mComputer.setmDisplay(display);
    }

    @Override
    public void buildOS() {
        mComputer.setmOS();
    }

    @Override
    public Computer create() {
        return mComputer;
    }
}
```

Director类，负责构造Computer 

```java
public class Director {
    
    private Builder mBuilder;

    public Director(Builder builder) {
        this.mBuilder = builder;
    }

    /**
     * 构建对象
     * @param broad
     * @param display
     */
    public void construct(String broad, String display){
        mBuilder.buildBroad(broad);
        mBuilder.buildDisplay(display);
        mBuilder.buildOS();
    }
}
```

测试代码 

```java
public class Test {

    public static void main(String[] args){
        Builder builder = new MacBuilder();
        Director director = new Director(builder);
        director.construct("英特尔主板", "Retina 显示器");
        System.out.println("Computer Info: " + builder.create().toString());
    }
}

输出结果 :
Computer Info: Computer {mBroad = 英特尔主板, mDisplay = Retina 显示器, mOS = Mac OS X 10.10}
```

上面代码中，通过具体的MacBuilder来构建Macbook对象，而Director封装构建复杂对象的过程，对外隐藏细节。Builder与Director一起将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的对象。

但在现实开发中，Director常常被忽略，直接使用一个Builder对象来进行链式调用构造，它的关键点是每个setter返回自身，如下，我们来组装一个华硕电脑：

华硕电脑，具体的产品类

```java
public class ASUSbook extends Computer {

    @Override
    public void setmOS() {
        mOS = "Windows 10 专业版";
    }
}
```

抽象Builder类 

```java
public abstract class Builder2 {

    public abstract Builder2 buildBroad(String broad);//设置主机
    public abstract Builder2 buildDisplay(String display);//设置显示器
    public abstract Builder2 buildOS();//设置操作系统
    public abstract Computer create();//创建Computer
    
}
```

具体的Builder类，构造ASUS电脑 

```java
public class ASUSBuilder extends Builder2 {

    private Computer mComputer = new ASUSbook();

    @Override
    public Builder2 buildBroad(String broad) {
        mComputer.setmBroad(broad);
        return this;
    }

    @Override
    public Builder2 buildDisplay(String display) {
        mComputer.setmDisplay(display);
        return this;
    }

    @Override
    public Builder2 buildOS() {
        mComputer.setmOS();
        return this;
    }

    @Override
    public Computer create() {
        return mComputer;
    }
}
```

测试代码 

```java
public class Test {

    public static void main(String[] args){
        Builder2 builder2 = new ASUSBuilder();
        ASUSbook asusbook = (ASUSbook) builder2
                .buildBroad("AMDB350socketAM4")
                .buildDisplay("AOC 显示器")
                .buildOS()
                .create();
        System.out.println("Computer Info: " + asusbook.toString());
//        Builder builder = new MacBuilder();
//        Director director = new Director(builder);
//        director.construct("英特尔主板", "Retina 显示器");
//        System.out.println("Computer Info: " + builder.create().toString());
    }
}

输出结果 :
Computer Info: Computer {mBroad = AMDB350socketAM4, mDisplay = AOC 显示器, mOS = Windows 10 专业版}
```

链式调用形式不仅去除Director角色，让整个结构简单，而且也能对product对象的组装过程有更加精准的控制。

然而上面的只是经典的实现方式，下面才是现在开发中最常用的，通过把Builder与产品类封装在一起，建立于上面的基础。

联想电脑，把Builder与产品类封装在一起

```java
public class LenovoBook extends Computer {

    private LenovoBook(Builder builder){
        setmOS();
        setmBroad(builder.broad);
        setmDisplay(builder.display);
    }

    @Override
    public void setmOS() {
        mOS = "Windows 10 家庭中文版";
    }

    public static class Builder extends Builder2{

        String broad;
        String display;

        @Override
        public Builder2 buildBroad(String broad) {
            this.broad = broad;
            return this;
        }

        @Override
        public Builder2 buildDisplay(String display) {
            this.display = display;
            return this;
        }

        @Override
        public Builder2 buildOS() {
            return this;
        }

        @Override
        public Computer create() {
            return new LenovoBook(this);
        }
    }
}
```

测试代码 

```java
public class Test {

    public static void main(String[] args){

       LenovoBook lenovoBook = (LenovoBook) new LenovoBook.Builder()
               .buildOS()
               .buildBroad("联想主板")
               .buildDisplay("联想显示器")
               .create();

       System.out.println("Computer Info: " + lenovoBook.toString());
    }
}

输出结果：
Computer Info: Computer {mBroad = 联想主板, mDisplay = 联想显示器, mOS = Windows 10 家庭中文版}
```

## 总结

Builder模式在开发中很常用，通过把产品类的构造器，字段私有化，只能通过Builder来设置属性，也通常作为配置类的构造器将配置的构建与表示分离开来，同时也是将配置从目标类中独立出来，避免过多的setter方法。Builder模式常用的实现形式是链式调用。

[本文源码相关位置](https://github.com/rain9155/DesignPatternDemo/tree/master/src/com/example/hy/designpatternDemo/builder)

