---
title: 使用AIDL来进行进程间通信
tags: 
- IPC
- AIDL
categories: android
---

## 前言

AIDL它是Android众多进程间通信方式中的一种，底层是Binder机制的实现，所以想要读懂AIDL自动生成的代码中各个类的作用，就必须对Binder有一定的了解，其实不止AIDL，Android进程间通信的大多数方式的底层都是Binder机制，本文主要介绍AIDL的使用，所以不会介绍Binder机制的原理，关于Binder机制的原理，我推荐下面的一篇文章，零基础也能看懂：

[写给 Android 应用工程师的 Binder 原理剖析](https://zhuanlan.zhihu.com/p/35519585)

看完上面的文章你也就能对Binder原理的实现有大概的了解，对于我们Android应用开发也就足够了，如果你还不满足，想从底层和源码了解Binder机制，你可以阅读下面的两个链接：

[Android Bander设计与实现](https://blog.csdn.net/universus/article/details/6211589#commentBox)

[Binder系列—开篇](http://gityuan.com/2015/10/31/binder-prepare/)

上面两个系列就是从设计和源码的角度去解读Binder，有点深入。好了，对Binder有一个大体上的认识后，接下来我们就要通过AIDL的使用来完成Android进程间通信的实践。

## Android进程间通信的方式

在介绍AIDL之前，我们先来看一下Android中除了AIDL还有哪些进程间通信的方式：

**1、Bundle**

Bundle实现了Parcelable，所以在Android中我们可以通过Intent在不同进程间传递Bundle数据。

但是在Intent 传输数据的过程中，用到了 Binder，Intent中的数据，即Bundle数据，会作为 Parcel 存储在Binder 的事务缓冲区(Binder transaction buffer)中的对象进行传输，而这个 Binder 事务缓冲区具有一个有限的固定大小，约为1MB，而且这个1Mb并不是当前进程所独享，而是所有进程共享的，所以由于1Mb的限制，Bundle不能存放大量的数据，不然会报TransactionTooLargeException，并且Bundle中存放的数据也要求能够被序列化，所以Bundle只适用于数据量小和简单的进程间通信。

**2、文件共享**

两个进程间通过读写同一个文件来交换数据。

但是由于Android允许并发的对同一个文件读写，所以如果两个进程同时的对这个文件写，就会出现问题，所以这适用于对数据同步要求不高的进程间通信。

**3、ContentProvider**

ContentProvider是Android中专门用于不同应用之间，即不同进程之间进行数据共享的方式，它的底层实现是Binder。

ContentProvider是四大组件之一，它的使用比较简单，我们只需要继承自ContenProvider，并重写它的六个方法：onCreate、query、updata、insert、delete和getType，其中onCreate用于初始化，getType用于返回一个Uri请求代表的MIME类型，剩下的都是CRUD操作，除了onCreate方法运行在主线程，其他方法都运行在Binder线程池，然后在另外一个进程中注册这个ContentProvider，在本进程的Activity中通过getContentResolver()获得ContentResolver后，再通过ContentResolver来完成对这个ContentProvider的CRUD操作。

**4、套接字(Socket)**

一种传统的Linux IPC方式，除了用于不同进程之间还可以用于不同机器之间（通过网络传输）的通信，但是它的传输效率也是非常的低。

**5、Messenger**

通过Messenger可以在不同进程之间传递Message对象，Message中可以放入我们需要传递的数据，它的底层实现是AIDL。

但是Messenger在服务端的Handler是以串行的方式处理来自客户端的Message，所以如果有大量的并发请求，Messenger就效率低下，所以Messenger适用于数据并发量低的进程间通信。

**6、AIDL**

也就是本文的主角，它的底层实现是Binder。

可以看到Android中进程间通信的方式中，除了文件共享和Socket，底层都是需要通过Binder来传输数据，可见Binder对Android的进程间通信来说是多么的重要。

## 进程间通信的准备 - 序列化

Android中要在进程间传输的数据都要是可序列化的，序列化就是把对象转换成二进制（字节流），从而可以存储到存储设备或者通过网络进行传输，有序列化，就有反序列，反序列化就是把二进制（字节流）转化为对象，其中8种基本数据类型默认可以在进程间传输，没有序列化的概念，序列化的作用对象是对象。在Android中，对象通过实现Serializable或Parcelable接口来实现序列化，下面简单介绍一下它们之间使用和区别：

### 1、Serializable

Serializable是java中提供的一个序列化接口，它是一个空接口，对象实现这个Serializable接口，就标记这个对象是可序列化的，然后我们通过ObjectOutputStream的writeObject方法就能完成对象的序列化，如下：

```java
User user = new User();
ObjectOutputStream os = new  ObjectOutputStream(new FileOutputStream("D://test.txt"));
os.writeObject(user);
os.close();
```

同理，通过ObjectOutputStream的readObject方法就能完成对象的反序列化，如下：

```java
ObjectOutputStream in = new  ObjectInputStream(new FileInputStream("D://test.txt"));
User user = in.readObject();
in.close();
```

如果你在序列化这个对象之后有可能会改变这个对象的类结构，例如为类添加新的字段，这时在序列化这个对象之前你就要为这个对象的类加上一个serialVersionUID参数，如下：

```java
public class User{
    //serialVersionUID取什么值没关系，只要保持不变就行
    private static final long serialVersionUID = 1L;
}
```

如果你确保这个对象的类结构在序列化之后是一直不变的，那么这个serialVersionUID可以忽略，为什么会这样呢？这是因为在序列化的时候，系统会把当前类的serialVersionUID写入到序列化的文件中，当反序列化时系统就会去检测文件中的serialVersionUID是否和当前类的serialVersionUID相同，如果相同就说明序列化的类和当前类的版本是相同的，这时候系统可以正确的反序列化，如果不一致，就说明序列化的类和当前类相比发生了某些变化，例如当前类添加了新的字段，这时就会反序列化失败，抛出异常。所以如果你手动指定了serialVersionUID的值，并一直保持不变，就算你改变了类的结构（不要改变类名和变量类型），序列化和反序列化时两者的serialVersionUID都是相同的，可以正常的进行反序列化，相反，如果你没有指定serialVersionUID的值，系统会根据类的结构自动生成它的hash值作为serialVersionUID的值，这时你改变了类的结构，在进行序列化和反序列化时两者生成的serialVersionUID不相同，这时反序列化就会失败，所以当你没有指定serialVersionUID的值，你要确保这个对象的类结构在序列化之后是一直不变。当然大多数情况下我们还是会指定serialVersionUID的值，以防我们不小心修改类的结构导致无法恢复对象。

这里只是列举了用Serializable实现对象序列化最简单的使用，其实我们可以通过transient关键字控制哪些字段不被序列化，还有静态成员不属于对象，不会参与序列化过程，还可以手动控制Serializable对象的序列化和反序列化过程，还关于Serializable更多使用方式可以看[Java对象表示方式1：序列化、反序列化和transient关键字的作用](https://www.cnblogs.com/szlbm/p/5504166.html)。

### 2、Parcelable

Parcelable是Android提供的一个接口，一个对象只要实现了这个接口，就可以实现序列化，如下：

```java
public class User implements Parcelable {

    private static final long serialVersionUID = 1L;

    private final String name;
    private final String password;

    public User(String name, String password) {
        this.name = name;
        this.password = password;
    }

    //下面就是实现Parcelable接口需要实现的方法
    
    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(this.name);
        dest.writeString(this.password);
    }

    public static final Parcelable.Creator<User> CREATOR = new Parcelable.Creator<User>() {
        @Override
        public User createFromParcel(Parcel source) {
            return new User(source);
        }

        @Override
        public User[] newArray(int size) {
            return new User[size];
        }
    };
    
    protected User(Parcel in) {
        this.name = in.readString();
        this.password = in.readString();
    }
}
```

User中需要序列化的字段都会包装进Parcel中，当反序列化时，从Parcel中读取数据，Parcel内部包装了可序列化的数据，可以在进程间传输，其中describeContents方法用来完成内容描述，一般返回0，writeToParcel方法用来实现序列化，CREATOR对象的内部方法用来实现反序列化，其中createFromParcel方法用来从序列化后的对象中创建原始对象，newArray方法用来创建指定长度的原始对象数组。

Parcelable的用法相比Serializable复杂了一点，如果每次需要序列化一个对象，都要写这么多重复样本代码，会有点累，这里我推荐一个插件[android-parcelable-intellij-plugin](https://github.com/mcharmas/android-parcelable-intellij-plugin)，专门用于自动生成这些重复样本代码。

### 3、Serializable和Parcelable的区别

Serializable和Parcelable都可以实现序列化，用于进程间的数据传递，它们之间有什么区别呢？在编码上，Serializable使用简单，而Parcelable稍显复杂；在效率上，Serializable的开销很大，因为它只是一个空接口，我们无需实现任何方法，Java便会对这个对象进行序列化操作，这是因为java会通过反射创建所需方法，导致序列化的过程较慢，而Parcelable的效率高，它的速度比Serializable快十倍，通过把一个完整的对象进行分解，而分解后的每一部分都是Intent所支持的数据类型，这样也就实现了传递对象的功能；在使用场景上，Parcelable主要用在内存序列化上，它不能使用在要将数据序列化在磁盘上的情况，因为在外界有变化的情况下，Parcelable不能很好的保证数据的持续性，所以如果需要序列化到磁盘或通过网络传输建议使用Serializable，尽管它的效率低点。

综上所述，在Android开发中，如果有序列化的要求，我们多点考虑使用Parcelable，毕竟它是Android推自带的，虽然它使用稍显复杂，但是有插件的帮助，加上它的效率高，就略胜Serializable了。

## 使用AIDL

前面花了一点篇幅来讲解序列化和其他进程间通信的方式，主要是让大家有个心理准备，下面进入正文：

### 1、AIDL是什么？

它全称是Android Interface Definition Language，即Android接口定义语言，为了使其他的进程也可以访问本进程提供的服务，Android使用AIDL来公开服务的接口，它里面定义了本进程可以为其他进程提供什么服务，即定义了一些方法，其他进程就可以通过RPC（远程调用）来调用这些方法，从而获得服务。





## 结语

参考资料：

[Android 接口定义语言 (AIDL)](https://developer.android.google.cn/guide/components/aidl#Defining)

[Android系统中Parcelable和Serializable的区别](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0204/2410.html)



