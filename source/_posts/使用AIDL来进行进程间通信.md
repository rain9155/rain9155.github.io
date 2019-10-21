---
title: 使用AIDL来进行进程间通信
tags:
  - IPC
  - AIDL
categories: android
date: 2019-10-21 21:04:01
---


## 前言

AIDL它是Android众多进程间通信方式中的一种，底层是Binder机制的实现，所以想要读懂AIDL自动生成的代码中各个类的作用，就必须对Binder有一定的了解，其实不止AIDL，Android进程间通信的大多数方式的底层都是Binder机制，本文主要介绍AIDL的使用，所以不会介绍Binder机制的原理，关于Binder机制的原理，我推荐下面的一篇文章，零基础也能看懂：

[写给 Android 应用工程师的 Binder 原理剖析](https://zhuanlan.zhihu.com/p/35519585)

看完上面的文章你也就能对Binder原理的实现有大概的了解，对于我们Android应用开发也就足够了，如果你还不满足，想从底层和源码了解Binder机制，你可以阅读下面的两个链接：

[Android Bander设计与实现](https://blog.csdn.net/universus/article/details/6211589#commentBox)

[Binder系列—开篇](http://gityuan.com/2015/10/31/binder-prepare/)

上面两个系列就是从设计和源码的角度去解读Binder，有点深入。好了，对Binder有一个大体上的认识后，接下来我们就要通过AIDL的使用来完成Android进程间通信的实践。

<!--more-->

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

## 进程间通信的准备 

### 1、 序列化

Android中要在进程间传输的数据都要是可序列化的，序列化就是把对象转换成二进制（字节流），从而可以存储到存储设备或者通过网络进行传输，有序列化，就有反序列，反序列化就是把二进制（字节流）转化为对象，其中8种基本数据类型默认可以在进程间传输，没有序列化的概念，序列化的作用对象是对象。在Android中，对象通过实现Serializable或Parcelable接口来实现序列化，下面简单介绍一下它们之间使用和区别：

#### 1.1、Serializable

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

#### 1.2、Parcelable

Parcelable是Android提供的一个接口，一个对象只要实现了这个接口，就可以实现序列化，如下：

```java
public class User implements Parcelable {

    private final String name;
    private final String password;

    public User(String name, String password) {
        this.name = name;
        this.password = password;
    }
    
    public String getName() {
        return name == null ? "" : name;
    }

    public String getPassword() {
        return password == null ? "" : password;
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

#### 1.3、Serializable和Parcelable的区别

Serializable和Parcelable都可以实现序列化，用于进程间的数据传递，它们之间有什么区别呢？在编码上，Serializable使用简单，而Parcelable稍显复杂；在效率上，Serializable的开销很大，因为它只是一个空接口，我们无需实现任何方法，Java便会对这个对象进行序列化操作，这是因为java会通过反射创建所需方法，导致序列化的过程较慢，而Parcelable的效率高，它的速度比Serializable快十倍，通过把一个完整的对象进行分解，而分解后的每一部分都是Intent所支持的数据类型，这样也就实现了传递对象的功能；在使用场景上，Parcelable主要用在内存序列化上，它不能使用在要将数据序列化在磁盘上的情况，因为在外界有变化的情况下，Parcelable不能很好的保证数据的持续性，所以如果需要序列化到磁盘或通过网络传输建议使用Serializable，尽管它的效率低点。

综上所述，在Android开发中，如果有序列化的要求，我们多点考虑使用Parcelable，毕竟它是Android自带的，虽然它使用稍显复杂，但是有插件的帮助，加上它的效率高，就略胜Serializable了。

### 2、开启多进程模式

既然是进程间通信，就一定会涉及到两个或两个进程以上，我们知道，在Android中，启动一个应用就代表启动了一个进程，但其实除了启动多个应用外，其实在一个应用中，也可以存在多个进程，只要你为四大组件在AndroidMenifest中指定“android:process”属性就行，比如我希望ClientActivity启动的时候，运行在独立的进程，我就会在AndroidMenifest中这样写，如下：

```xml
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />

        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>

<activity android:name=".ClientActivity"
          android:process="com.example.aidltest.client"/>
```

可以看到我只是简单的ClientActivity指定了“android:process”属性，这样ClientActivity在启动的时候就会运行在进程名为com.example.aidltest.client的独立进程上，而MainActivity启动时，就会运行在默认进程上，默认进程的进程名为当前包名即com.example.aidltest。

需要注意的是，此时这两个Activity虽然在同一个应用，但是却不在同一个进程，所以它们是不共享内存空间的，Android会为每一个进程分配一个独立的虚拟机，不同的虚拟机在内存分配上有不同的地址空间，所以MainActivity和ClientActivity的内存地址空间不一样，它们要访问对方的数据的时候，都要通过进程间通信的方式来进行。

接下来为了方便后面的讲解，我把ClientActivity指定为主活动，把MainActivity删除，再创建一个Server，并为它指定process属性，如下：

```xml
<activity android:name=".ClientActivity">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />

        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>

<service android:name=".RemoteService"
         android:enabled="true"
         android:exported="true"
         android:process="com.example.aidltest.remote"/>
```

ClientActivity启动时会运行在默认的进程上，RemoteService启动时会运行在名为com.example.aidltest.remote的进程上，它们在本文分别代表为本地客户端和远程服务端。

## 使用AIDL

前面花了一点篇幅来讲解序列化和其他进程间通信的方式，主要是让大家有个心理准备，下面进入正文：

### 1、AIDL是什么

它全称是Android Interface Definition Language，即Android接口定义语言，为了使其他的进程也可以访问本进程提供的服务，Android使用AIDL来公开服务的接口，它里面定义了本进程可以为其他进程提供什么服务，即定义了一些方法，其他进程就可以通过RPC（远程调用）来调用这些方法，从而获得服务，其中提供服务的进程称为服务端，获取服务的进程称为客户端。

### 2、AIDL接口的创建

AIDL接口用来暴露服务点提供给客户端的方法，新建一个AIDL接口文件，只需要在你的项目中 **点击包名 -> 右键 -> new -> AIDL -> Aidl.file**，然后输入AIDL接口名称，这里我输入了**IUserManager**，然后点击**Finish**，就会在你的main目录下创建了一个aidl文件夹，aidl文件夹里的包名和java文件夹里的包名相同，里面用来存放AIDL接口文件，如下：

{% asset_img aidl1.png aidl %}

在里面你会发现你刚刚创建的AIDL接口IUserManager，点进去，如下：

```java
// IUserManager.aidl
package com.example.aidltest;

// Declare any non-default types here with import statements

interface IUserManager {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);
}
```

里面声明了一个方法，写着AIDL支持的一些数据类型(int、long、boolean、float、double、String)，除了这些，AIDL还支持其他的基本数据类型、ArrayList(里面的每个元素都要被AIDL支持)、HashMap(里面的每个元素都要被AIDL支持)、实现了Parcelable接口的对象和AIDL接口本身，还有AIDL接口中只支持声明方法，不支持声明静态常量。

其中如果要在AIDL接口文件中使用AIDL对象，必须显式的 **import** 进来，即使它们在同一个包内，还有如果在AIDL接口文件用到了Parcelable对象，必须新建一个和它同名的AIDL文件，并在其中声明它为parcelable类型，接下来我要使用User这个Parcelable对象，所以我要在aidl文件夹下新建一个和他同名的AIDL文件，如下：

{% asset_img aidl2.png aidl %}

然后在User.aidl中添加如下内容：

```java
// User.aidl
package com.example.aidltest;

parcelable User;
```

在里面，我声明了User.java这个对象为parcelable类型，接下来把IUserManager中的basicTypes方法删除，添加一个根据用户姓名获得用户信息的方法，如下：

```java
// IUserManager.aidl
package com.example.aidltest;

import com.example.adiltest.User;

interface IUserManager {

   User getUser(String name);
}
```

在里面我显示的 import 了User这个AIDL文件，即使它们在同一个包内，并声明了一个getUser方法，这个方法将会在服务端实现，然后在客户端调用(RPC)。

### 3、根据AIDL接口生成 对应的Binder类

有了AIDL接口后我们需要根据AIDL接口生成客户端和服务端对应的Binder类，有两种方式生成，一种是通过SDK自动生成，另外一种是我们自己手动编码实现，其中能够进行手动编码实现的前提是基于对SDK自动生成的各种Binder类的充分理解，下面我们先来介绍SDK自动生成的Binder类。

#### 3.1、SDK自动生成

我们在AS导航栏 **Build -> ReBuild Project**，SDK就会替我们在 **app\build\generated\aidl_source_output_dir\debug\compileDebugAidl\out\包名** 下生成一个**IUserManager.java**，它就是根据IUserManager.aidl文件生成的，里面没有缩进，所以看起来不习惯，使用快捷键**ctrl+alt+L**，格式化一下代码，如下：

```java
//IUserManager.java

/**
 * 1、IUserManager接口，getUser方法定义在其中
 **/
public interface IUserManager extends android.os.IInterface {
   
    /**
     * 1、抽象类Stub，需要在远程服务端实现
     */
    public static abstract class Stub extends android.os.Binder implements com.example.aidltest.IUserManager {
        private static final java.lang.String DESCRIPTOR = "com.example.aidltest.IUserManager";

        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }
        
        public static com.example.aidltest.IUserManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.example.aidltest.IUserManager))) {
                return ((com.example.aidltest.IUserManager) iin);
            }
            return new com.example.aidltest.IUserManager.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            java.lang.String descriptor = DESCRIPTOR;
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(descriptor);
                    return true;
                }
                case TRANSACTION_getUser: {//case TRANSACTION_getUser分支
                    data.enforceInterface(descriptor);
                    java.lang.String _arg0;
                    _arg0 = data.readString();
                    //调用getUser方法的具体实现
                    com.example.aidltest.User _result = this.getUser(_arg0);
                    reply.writeNoException();
                    if ((_result != null)) {
                        reply.writeInt(1);
                        _result.writeToParcel(reply, android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
                    } else {
                        reply.writeInt(0);
                    }
                    return true;
                }
                default: {
                    return super.onTransact(code, data, reply, flags);
                }
            }
        }

        /**
         * 2、代理类Proxy，客户端使用
         */
        private static class Proxy implements com.example.aidltest.IUserManager {
            
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            @Override
            public com.example.aidltest.User getUser(java.lang.String name) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                com.example.aidltest.User _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeString(name);
                    //传进了TRANSACTION_getUser字段
                    mRemote.transact(Stub.TRANSACTION_getUser, _data, _reply, 0);
                    _reply.readException();
                    if ((0 != _reply.readInt())) {
                        _result = com.example.aidltest.User.CREATOR.createFromParcel(_reply);
                    } else {
                        _result = null;
                    }
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
        }

        static final int TRANSACTION_getUser = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
    }
    
 	public com.example.aidltest.User getUser(java.lang.String name) throws android.os.RemoteException;
}

```

这个文件有3个主要的类：

**1、IUserManager接口**

它声明了IUserManager.aidl定义的getUser方法，继承自IInterface。

**2、IUserManager.Stub抽象类**

IUserManager的静态内部类，它继承自Binder，**说明它是一个 Binder 本地对象**（Binder下面介绍），它虽然实现了IUserManager接口，但是它继续声明为一个抽象类，并没有实现IUserManager接口中的getUser方法，表明子类需要实现getUser方法，返回具体的User信息，而服务端将会实现这个Stub抽象类。

**3、IUserManager.Stub.Proxy代理类 **

IUserManager.stub的静态内部类，它实现了IUserManager接口，并且实现了getUser方法，但是里面只是把数据装进data这个Parcel对象，通过mRemote的transact方法发送给服务端，接着用reply这个Parcel对象等待服务端数据的返回，这一切都是通过mRemote这个IBinder对象进行，**mRemote代表着Binder对象的本地代理**（IBinder下面介绍），mRemote会通过Binder驱动来完成与远程服务端的Stub的通信。

可以看到Stub类和Stub.Proxy类都实现了IUserManager接口，这就是一个典型的[代理模式](https://rain9155.github.io/2019/10/15/%E4%BB%A3%E7%90%86%E6%A8%A1%E5%BC%8F/)，它们的getUser方法有着不同的实现，Stub类它将会在远程的服务端完成getUser方法的具体实现，而Stub.Proxy类是本地客户端的一个代理类，它已经替我们默认的实现了getUser方法，该方法里面通过**mRemote**这个Binder引用的**transact方法**把请求通过驱动发送给服务端，我们注意到mRemote发送请求时还传进了**TRANSACTION_getUser**这个代表着getUser方法的标识名，这表示客户端告诉服务端我要调用getUser这个方法，当驱动把请求转发给服务端后，服务端的Stub类的**onTransact方法**就会回调，它里面有一个switch语句，根据code来调用不同的方法，这时它就会走到case TRANSACTION_getUser这个分支，然后调用getUser方法的在服务端的具体实现，如果有返回值的话，还会通过reply返回给客户端，这样就通过Binder驱动完成了一次远程方法调用(RPC)。

这里要注意的是客户端通过mRemote的transact方法把请求发送给客户端之后，这时会阻塞UI线程等待服务端的返回，而服务端的onTransact方法回调时，服务端的getUser方法会被回调，这时服务端的getUser方法是运行在服务端Binder线程池中，所以如果此时有UI操作需要回到UI线程再进行UI操作。

我们还注意到IUserManager接口继承了IInterface，IUserManager.Stub继承自Binder，它们是干什么的？我们来认识一下：

**IInterface**

这是一个接口，用来表示服务端提供了哪些服务，如果服务端需要暴露调用服务的方法给客户端使用，就一定要继承这个接口，它里面有个asBinder方法，用于返回当前的Binder对象。

**IBinder**

这时一个跨进程通信的Base接口，它声明了跨进程通信需要实现的一系列抽象方法，实现了这个接口就说明可以进行跨进程通信，Binder和BinderProxy都继承了这个接口。

**Binder**

代表的是 Binder 本地对象（Binder实体），它继承自IBinder，所有本地对象都要继承Binder，Binder中有一个内部类BinderProxy，它也继承自IBinder，它代表着Binder对象的本地代理(Binder引用)，Binder实体只存在于服务端，而Binder引用则遍布于各个客户端。

接下来我们动手实践一下，首先在服务端RemoteService中，我们要这样做：

```java
public class RemoteService extends Service {

    //1、实现Stub类中的getUser方法
    private IUserManager.Stub mBinder = new IUserManager.Stub() {
        @Override
        public User getUser(String name) throws RemoteException {
            //这里只是简单的返回了一个用户名为name，密码为123456的用户实例
            return new User(name, "123456");
        }
    };

    public RemoteService() {
    }

    @Override
    public IBinder onBind(Intent intent) {
        //2、在onBinder方法中返回Stub类的实现，Stub类继承自Binder，Binder实现了IBinder，这样返回是没问题的
        return mBinder;
    }
}
```

在服务端我们需要实现Stub类中的getUser方法，然后在onBinder方法中返回Stub类的实现，这样客户端绑定服务时就会收到这个Stub类的Binder引用。

然后在客户端ClientActivity中，我们要这样做：

```java
public class ClientActivity extends AppCompatActivity {

    private static final String TAG = ClientActivity.class.getSimpleName();

    //1、创建ServiceConnection
    private ServiceConnection mServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.d(TAG, "onServiceConnected, 与服务端连接成功！");
            
            //把服务端返回的Binder引用，通过Stub.asInterface方法包装成本地代理类IUserManager.Stub.Proxy，Proxy类实现了IUserManager，所以这样写是没问题的
            IUserManager userManager = IUserManager.Stub.asInterface(service);
            
            try {
                //通过本地代理对象远程调用服务端的方法
                User user = userManager.getUser("rain");
                
                Log.d(TAG, "onServiceConnected，向服务端获取用户信息成功，User = ["
                        + "name = " + user.getName()
                        + "password = " + user.getPassword());
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            Log.d(TAG, "onServiceDisconnected, 与服务端断开连接");

        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        //2、启动并通过ServiceConnection绑定远程服务
        bindService(
                new Intent(this, RemoteService.class),
                mServiceConnection,
                Context.BIND_AUTO_CREATE
        );
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        //3、解绑服务
        unbindService(mServiceConnection);
    }
}
```

在服务端我们首先要创建ServiceConnection，然后通过ServiceConnection来绑定RemoteService，在成功绑定后，ServiceConnection的onServiceConnected方法就会回调，它的第二个输入参数就是RemoteService在onBind方法返回的Stub类的Binder引用，我们拿到这个引用后，就可以通过通过Stub.asInterface方法转换为本地代理类Stub.Proxy，然后调用它的getUser方法，Proxy的getUser方法会远程调用RemoteService的getUser方法，方法返回后，在log中打印出User的信息，最后，活动结束，我们记得解绑服务，这个过程和上面介绍的一次RPC过程是一样的。

我们发现完成一次进程间的通信是非常的简单，这就好像只是简单的调用一个对象的方法，但其实这都得益于Binder在底层为我们做了更多工作，我们上层使用得有多简单，它得底层实现就有多复杂，上面的一次进程间通信，可以简单的用下图表示：

{% asset_img aidl3.png aidl %}

其中ServiceManager就是根据Binder的名字查找Binder引用并返回，如果你对Binder的通信架构有一定的了解，理解这个就不难，对象转换就是完成Binder实体 -> Binder引用，Binder引用 -> Binder实体的转换，在java层继承自Binder的都代表Binder本地对象，即Binder实体，而Binder类的内部类BinderProxy就代表着Binder对象的本地代理，即Binder引用，这两个类都继承自IBinder, 因而都具有跨进程传输的能力，在跨越进程的时候，Binder 驱动会自动完成这两个对象的转换。

我们重点讲一下图中的第3步：**将Binder引用赋值给Proxy的mRemote字段**，Proxy就是前面介绍的Stub.Proxy，所以接着我们看看**IUserManager.Stub.asInterface(IBinder)**方法是如何把服务端返回的Binder引用赋值给本地的代理类Proxy的mRemote字段，asInterface方法如下：

```java
//IUserManager.Stub.java
public static com.example.aidltest.IUserManager asInterface(android.os.IBinder obj) {
    if ((obj == null)) {
        return null;
    }
    //根据DESCRIPTOR调用IBinder的queryLocalInterface方法
    android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
    if (((iin != null) && (iin instanceof com.example.aidltest.IUserManager))) {
        return ((com.example.aidltest.IUserManager) iin);
    }
    return new com.example.aidltest.IUserManager.Stub.Proxy(obj);
}

//Binder
public @Nullable IInterface queryLocalInterface(@NonNull String descriptor) {
    if (mDescriptor != null && mDescriptor.equals(descriptor)) {
        //mOwner等于Stub类实例
        return mOwner;
    }
    return null;
}

//Binder#BinderProxy
public IInterface queryLocalInterface(String descriptor) {
    return null;
}
```

可以发现它里面根据DESCRIPTOR标识调用IBinder的queryLocalInterface方法在查找一些什么，DESCRIPTOR是什么？DESCRIPTOR是Binder实体的唯一标识，一般用当前的Binder的类名表示，它定义在Stub类中，如本文的Stub的 DESCRIPTOR = "com.example.aidltest.IUserManager"，前面已经讲过Binder和BinderProxy都继承自IBinder，所以它们的queryLocalInterface有不同的实现，我们看到BinderProxy的直接返回null；而Binder的需要和自己的DESCRIPTOR比较，如果相同就返回mOwner，否则返回null，其中mOwner就等于Stub类实例，在Stub类构造的时候赋值，如下：

```java
public static abstract class Stub extends android.os.Binder implements com.example.aidltest.IUserManager {
    
    private static final java.lang.String DESCRIPTOR = "com.example.aidltest.IUserManager";

    public Stub() {
        this.attachInterface(this, DESCRIPTOR);
    }

    public void attachInterface(@Nullable IInterface owner, @Nullable String descriptor) {
        mOwner = owner;
        mDescriptor = descriptor;
    }
    
    //...
}

```

这说明了如果queryLocalInterface的返回不为空，iin != null，表示obj是Binder实体（Stub的子类），客户端和服务端在同一个进程，asInterface方法返回的就是Binder实体；如果queryLocalInterface的返回为空，iin == nul，表示obj实际上是Binder引用，客户端和服务端在不同的进程，asInterface构造一个Proxy对象返回，并把Binder引用通过构造传了进去，我们看Proxy的构造函数，如下：

```java
private static class Proxy implements com.example.aidltest.IUserManager {
    private android.os.IBinder mRemote;

    Proxy(android.os.IBinder remote) {
        mRemote = remote;
    }
    
    //...
}
```

可以看到，**传进去的Binder引用赋值给了mRemote字段**，所以Proxy中的mRemote就是Binder引用，客户端就是通过这个mRemote来和服务端通信。

也就是说，如果在同一个进程，asInterface返回的是Stub的实现类，因为不存在跨进程调用，直接调用该方法就行，如果在不同进程，asInterface返回的是Proxy对象，客户端调用Proxy中的同名方法，通过mRemote的transact方法挂起当前线程等待服务端返回，服务端收到请求后响应返回数据。

到这里，相信大家在SDK把帮助下已经会使用AIDL来完成简单的进程间通信，接下来通过手动编码实现。

#### 3.2、手动编码实现

我们发现使用AIDL系统会自动的帮我们生成上述代码，是为了方便我们的开发，系统根据AIDL文件生成的java文件格式是固定，我们完全可以抛开AIDL直接手写对应的Binder类，下面我们本着单一原则把原本IUserManager.java的里面的Stub类和Stub类中的Proxy类独立出来，所以我们总共要写3个类，分别是：IUserManager、Stub、Proxy。

**1、声明一个继承自IInterface的接口**

声明一个继承自IInterface的接口，在里面定义我们想要让客户端调用的方法，如下：

```java
public interface IUserManager extends IInterface {
    User getUser(String name);
}
```

**2、声明服务端的Stub类**

Stub类需要继承Binder，表明它是一个Binder本地对象，它还需要实现IUserManager接口，但是继续声明为抽象类，不需要实现IUserManager的getUser方法，接着我们做以下几步：

1、在里面定义一个字符串DESCRIPTOR，表示Binder实体的唯一标识，用当前的Stub类的类名表示，并把它的可见修饰符改为public，待会在Proxy需要用到.

2、在构造函数中把this 和 DESCRIPTOR字符串 attach 给父类Binder中的mOwner和mDescriptor字段.

3、定义一个TRANSACTION_getUser整型数值代表着getUser方法的标识名，赋值格式照抄自动生成的Stub类的TRANSACTION_getUser.

4、定义一个asInterface静态方法，里面的内容实现照抄自动生成的Stub类的asInterface方法，需要注意里面的IUserManager接口需要换成我们刚刚定义的IUserManager接口.

5、最后重写IInterface的asBinder方法和Binder的onTransact方法，里面的内容实现照抄自动生成的Stub类的asBinder和onTransact方法.

最终这个Stub类如下：

```java
public abstract class Stub extends Binder implements IUserManager {

    public static final String DESCRIPTOR = "com.example.aidltest.Stub";//这里改成com.example.aidltest.Stub

    static final int TRANSACTION_getUser = (IBinder.FIRST_CALL_TRANSACTION + 0);

    public Stub() {
        this.attachInterface(this, DESCRIPTOR);
    }

    public static IUserManager asInterface(android.os.IBinder obj) {//导入我们自定义的IUserManager
        if ((obj == null)) {
            return null;
        }
        android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
        if (((iin != null) && (iin instanceof IUserManager))) {
            return (IUserManager) iin;
        }
        return new Proxy(obj);
    }

    @Override
    public android.os.IBinder asBinder() {
        return this;
    }

    @Override
    public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
        java.lang.String descriptor = DESCRIPTOR;
        switch (code) {
            case INTERFACE_TRANSACTION: {
                reply.writeString(descriptor);
                return true;
            }
            case TRANSACTION_getUser: {
                data.enforceInterface(descriptor);
                java.lang.String _arg0;
                _arg0 = data.readString();
                com.example.aidltest.User _result = this.getUser(_arg0);
                reply.writeNoException();
                if ((_result != null)) {
                    reply.writeInt(1);
                    _result.writeToParcel(reply, android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
                } else {
                    reply.writeInt(0);
                }
                return true;
            }
            default: {
                return super.onTransact(code, data, reply, flags);
            }
        }
    }
}
```

**3、声明客户端的Proxy类**

Proxy类只需要实现IUserManager接口，并且实现IUserManager中的getUser方法，接着我们做以下几步：

1、定义一个IBinder类型的mRemote字段，并在构造函数中赋值.

2、实现IUserManager中的getUser方法和IInterface的asBinder方法，里面的内容实现照抄自动生成的Stub.Proxy的getUser方法和asBinder方法，需要注意getUser中的一些字段需要导入我们刚刚在Stub类中定义的字段.

最终这个Proxy类如下：

```java
public class Proxy implements IUserManager {

    private IBinder mRemote;

    Proxy(android.os.IBinder remote) {
        mRemote = remote;
    }

    @Override
    public android.os.IBinder asBinder() {
        return mRemote;
    }

    @Override
    public com.example.aidltest.User getUser(java.lang.String name) throws android.os.RemoteException {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        com.example.aidltest.User _result;
        try {
            _data.writeInterfaceToken(Stub.DESCRIPTOR);//这里引用我们自己写的Stub的DESCRIPTOR
            _data.writeString(name);
            mRemote.transact(Stub.TRANSACTION_getUser, _data, _reply, 0);//这里引用我们自己写的Stub的TRANSACTION_getUser
            _reply.readException();
            if ((0 != _reply.readInt())) {
                _result = com.example.aidltest.User.CREATOR.createFromParcel(_reply);
            } else {
                _result = null;
            }
        } finally {
            _reply.recycle();
            _data.recycle();
        }
        return _result;
    }

}
```

最终整个工程目录如下：

{% asset_img aidl4.png aidl %}

**4、使用**

使用就很简单了，只需要把上面自动生成的IUserManager、IUserManager.Stub、IUserManager.Stub.Proxy替换成我们自己写的IUserManager、Stub、Proxy就行，就不再贴代码了，输出如下：

{% asset_img aidl5.png aidl %}

## 学完AIDL能干什么

平常在Android中应用到进程间通信的场景非常的少，但这并不是说AIDL没有用，一个最直观的应用就是在阅读Android系统源码的时候，例如Activity的启动流程：

[Activity的启动流程（1）](https://rain9155.github.io/2019/05/19/Activity的启动流程%EF%BC%881%EF%BC%89/)

[Activity的启动流程（2）](https://rain9155.github.io/2019/05/19/Activity的启动流程%EF%BC%882%EF%BC%89/)

在android8.0之后Activity的有关进程间的通信都是通过AIDL来实现，Android根据IActivityManager.aidl文件来生成进程间通信所需的Binder类，如ApplicationThread在AMS的本地代理，AMS在ActivityThread的本地代理，当我们通过startActiivty发起Activity的启动请求时，ActivityThread就会通过AMS本地代理调用AMS的相应方法，当Actiivty在AMS准备好后，AMS就会通过ActivityThread本地代理回调应用进程的Activity的生命周期方法，这里就不在多述了，这个过程如下：

{% asset_img aidl6.png aidl %}

主要是通过这个例子说明，学习完AIDL后，能够帮助我们更好的理解系统源码有关跨进程的一些术语，类等，通过AIDL也能更好的加深我们对Android进程间通信的原理的理解，也掌握了一种进程间通信的方式。

## 结语

本文简单的介绍了一下Android几种进程间通信的方式，然后通过SDK自动生成AIDL代码来理解了一下生成的代码中各个类的作用和关系，还根据自动生成AIDL代码来手动实现了一遍简单的跨进程通信，加深理解，掌握了一些基础AIDL知识，可能会有些不全面，但是足够基本使用，想要了解更全面的AIDL知识，最好的途径还是参阅官方文档：[Android 接口定义语言 (AIDL)](https://developer.android.google.cn/guide/components/aidl#Defining)

[本文源码地址](https://github.com/rain9155/AIDLTest)

参考资料：

[Android系统中Parcelable和Serializable的区别](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0204/2410.html)

[关于Binder，作为应用开发者你需要知道的全部](https://www.jianshu.com/p/062a6e4f5cbe)



