---
title: 基于P2P的局域网即时通信应用
tags: 
- android
- p2p
categories: 开源项目
date: 2019-07-12 19:25:30
---


## 前言

这是一个使用java语言开发的基于P2P的局域网即时通信Android应用，界面是高仿微信的聊天界面，在里面你将会学到java多线程并发编程、Socket编程、UDP广播、TCP连接、还有图片加载相关知识等。

项目地址：[P2P](https://github.com/rain9155/P2P)

## 设计思路

P2P不同于C/S方式，它没有集中式的服务器，在P2P中，程序既是服务器又是客户端，在同一个局域网内，每个用户发送的消息不会经过路由器转发到其他局域网，那么如何保证大家都在同一个局域网内呢？答案是只要大家都连上同一个WIFI就行，这样就保证大家在同一个局域网内，这时你手机或电脑就会被路由器分配一个ip地址。

如下图：

{% asset_img core1.png core1 %}

下面是设计思路：

### 1、用户登陆阶段

（1）用户A打开P2P程序的，选择一个名字和头像后，点击登陆，就开始登陆上局域网，用户A登陆时程序同时会启动两个线程，一个线程里面启动UDP服务端(端口号9156)，用来等待其他用户的登陆，另一个线程里面启动TCP服务端(端口号9155)，用来等待其他用户的Socket连接，在登陆同时用户A还会使用UDP广播一个UDP包出去。这个UDP包包含了用户姓名和ip地址等信息，UDP包会发送给同一局域网内所有具有相同端口的UDP服务端程序，包括本程序。

（2）这时如果有其他在线的用户，那么每个在线的用户程序中的UDP服务端就会收下这个UDP包，然后把用户信息取出来并把用户A加入在线列表，因为UDP包中包含用户A ip地址，所以每个在线的用户使用用户A的ip地址向用户A发一个回复。

（3）用户A等待一段时间后，就会收到所有在线用户的回复，然后把所有的在线用户加入自己的在线列表。

登陆阶段如图：

{% asset_img user1.png user1 %}

### 2、 用户聊天阶段
（1）用户A选择自己的在线列表中的用户B聊天，这时用户A就会向用户B发起TCP连接，与此同时用户B的TCP服务端中就会收到一个Socket请求，用户B把这个Socket请求缓存起来，同理用户A发起连接时也会产生一个Socket，用户A也把这个Socket保存缓存起来，这样下一次就不用重复建立连接。

（2）这样双方都拥有一个Socket，双方基于Socket与Socket之间建立的连接上聊天（传输文件，文字等）。

聊天阶段如图：

{% asset_img user2.png user2 %}

### 3、用户退出阶段

（1）当用户A离开程序，退出局域网时，用户A就像登陆一样使用广播地址广播一个UDP包出去，UDP中包含了要退出登陆的信息，那么在局域网内的在线用户收到这个UDP后，就把用户A移除出在线用户列表，如果有用户A的Socket连接，就把Socket连接关闭掉。

（2）用户A发出退出广播后，也把自己缓存的所有Socket连接关闭掉。 

{% asset_img user3.png user3 %}

## 程序运行截图

首先用户A和用户B登陆，分别选择一个头像和姓名，如下

{% asset_img p2p1.png p2p1 %}

登陆后，双方正常来讲，是只有一个在线用户的，但是考虑到平时测试两台手机不方便，就没有把自己过滤掉，所以现在双方都有两个在线用户，用户A的在线用户是用户B和自己，用户B的在线用户是用户A和自己，如下：

{% asset_img p2p2.png p2p2 %}

下面是双方聊天的过程，现在可以发文字、图片、语音、文件(支持发送大文件，发送大文件有进度显示)。

{% asset_img p2p3.png p2p3 %}

## 关键性的代码

下面红色方框内的是关键性类，如下：

{% asset_img core2.png core2 %}

下面讲解一些关键性代码：

### 1、初始化TCP服务端

初始化TCP服务端，在**ConnectManager**类中，如下：

```java
 /**
   * 初始化ServerSocket监听，绑定端口号, 等待客户端连接
   */
public void initListener(){
    mExecutor.execute(() -> {
        try {
            //创建ServerSocket监听，并绑定端口号
            mServerSocket = new ServerSocket(PORT);
            LogUtil.d(TAG, "开启服务端监听，端口号 = " + PORT);
        } catch (IOException e) {
            e.printStackTrace();
            LogUtil.e(TAG, "绑定端口号失败，e = " + e.getMessage());
        }
        while (true){
            try {
                //调用accept()开始监听，等待客户端的连接
                Socket socket = mServerSocket.accept();
                String ipAddress = socket.getInetAddress().getHostAddress();
                if(isClose(ipAddress)){
                    LogUtil.d(TAG, "一个用户加入聊天，socket = " + socket);
                    //每个客户端连接用一个线程不断的读
                    ReceiveThread receiveThread = new ReceiveThread(socket);
                    //缓存客户端的连接
                    mClients.put(ipAddress, socket);
                    //放到线程池中执行
                    mExecutor.execute(receiveThread);
                    LogUtil.d(TAG, "已连接的客户端数量：" + mClients.size());
                    //简单的心跳机制
                    heartBeat(ipAddress);
                }
            } catch (IOException e) {
                e.printStackTrace();
                LogUtil.e(TAG, "调用accept()监听失败， e = " + e.getMessage());
                break;
            }
        }
        try {
            //释放掉ServerSocket占用的端口号
            mServerSocket.close();
        } catch (IOException e) {
            e.printStackTrace();
            LogUtil.e(TAG, "关闭端口号失败， e = " + e.getMessage());
        }
    });
}
```

ConnectManager是用来管理每个用户的连接，ConnectManager的initListener()方法里面会绑定一个端口号，然后调用accept()方法等待其他客户端的连接，如果有客户端的连接请求，就会为每一个客户端的连接创建一个Thread，这个Thread会不停等待接收客户端的消息。如下：

```java
public class ReceiveThread implements Runnable{
    //...
    @Override
    public void run() {
        while (true){
            Mes mes;
            try{
                InputStream in = mSocket.getInputStream();
                mes = receiveMessageByType(in);
               //...
            } catch (Exception e) {
                e.printStackTrace();
                LogUtil.e(TAG, "获取客户端消息失败，e = " + e.getMessage());
                //两端的Socker连接都要关闭
                ConnectManager.getInstance().removeConnect(mClientIp);
                ConnectManager.getInstance().removeReceiveCallback(mClientIp);
                ConnectManager.getInstance().cancelScheduledTask(mClientIp);
                break;
            }
        }
    }
//...
}
```

### 2、初始化UDP服务端

初始化UDP服务端，在**OnlineUserManager**类中，如下：

```java
 /**
   * 初始化监听，绑定指定端口, 等待接受广播
   */
public void initListener(){
    mExecutor.execute(() -> {
        try {
            mDatagramSocket = new DatagramSocket(PORT);
            LogUtil.d(TAG, "开启广播监听，端口号 = " + PORT);
        } catch (SocketException e) {
            e.printStackTrace();
            LogUtil.e(TAG, "创建DatagramSocket监听失败， e = " + e.getMessage());
        }
        while (true){
            try {
                byte[] buffer = new byte[MAX_RECEIVE_DATA];
                DatagramPacket datagramPacket = new DatagramPacket(buffer, buffer.length);
                mDatagramSocket.receive(datagramPacket);
                byte[] data = datagramPacket.getData();
                //获得发送方的ip地址
                String receiveIp = datagramPacket.getAddress().getHostAddress();
                //解析数据
                Data datas = resolveData(data);
                if(datas != null){
                    //用户数据
                    int code = datas.getCode();
                    User user = datas.getUser();
                    user.setIp(receiveIp);
                    if(code == 0){
                        //把它加入在线用户列表
                        if(!mOnlineUsers.containsKey(receiveIp)){
                            mOnlineUsers.put(receiveIp, user);
                            //通知主活动用用户加入
                            if(mUserCallback != null){
                                mHandler.obtainMessage(TYPE_JOIN_USER, mOnlineUsers.get(receiveIp)).sendToTarget();
                            }
                            LogUtil.d(TAG, "一个用户加入，地址 = " + receiveIp);
                        }
                        //回复它
                        reply(receiveIp);
                    }else if(code == 1){
                        //用户退出在线用户列表
                        if(mOnlineUsers.containsKey(receiveIp)){
                            User exitUser = mOnlineUsers.remove(receiveIp);
                            if(mUserCallback != null){
                                mHandler.obtainMessage(TYPE_EXIT_USER, exitUser).sendToTarget();
                            }
                            LogUtil.d(TAG, "一个用户退出，地址 = " + receiveIp);
                        }

                    }else {
                        //得到所有在线用户列表
                        if(!mOnlineUsers.containsKey(receiveIp)) {
                            mOnlineUsers.put(receiveIp, user);
                            //通知主活动用用户加入
                            if(mUserCallback != null){
                                mHandler.obtainMessage(TYPE_JOIN_USER, mOnlineUsers.get(receiveIp)).sendToTarget();
                            }
                            LogUtil.d(TAG, "获得一个用户信息，地址 = " + receiveIp);
                        }
                    }
                }
                LogUtil.d(TAG, "当前在线用户，count = " + mOnlineUsers.size());
            } catch (IOException e) {
                e.printStackTrace();
                LogUtil.e(TAG, "接受广播失败， e = " + e.getMessage());
                break;
            }
        }
        if(mDatagramSocket != null){
            mDatagramSocket.close();
        }
    });
}
```

OnlineUserManager是用来管理在线用户的，OnlineUserManager的initListener()方法里面也是会绑定一个端口号，然后调用receive()方法等待用户的广播信息，如果有用户的广播信息，就根据用户的广播信息类型做出不同的动作，如把用户加入在线用户列表。

### 3、Mes类的设计

Mes类是用户之间建立连接后传输消息的实体类，如下：

```java
public class Mes<T>{

    public ItemType itemType;//Mes的Item类型
    public MesType mesType;//Mes的类型
    public String userIp;//发送Mes的用户的ip
    public T data;//具体消息
    //...
}
```

其中**T**是一个泛型，它可以代表着文本、音频、文件、图片的类型，所以在构造一个Mes时，就要确定它是属于什么类型，然后文本、音频、文件、图片分别在对应一个实体类。

### 4、User类的设计

User就代表着用户，如下：

```java
public class User implements Serializable {
    private String mName;//名字
    private String mIp;//ip
    private String mImagePath;//头像路径
    private int mImageLen;//头像长度
    //...
}
```

它在传输前中会转成一个Json字符串，收到后再把它转成User类，这样就很容易的把它里面的数据解析出来也方便了传输。

### 5、关于心跳机制的实现

心跳机制是什么？它就每隔一段事件发一个探测，探测在线的用户是否存活。有些在线用户由于手机关机，不正常退出应用等会导致它无法正常退出登陆，这时就需要每隔一段时间探测它是否存活。

P2P中实现了一个简单的心跳机制，其实它就是一个定时任务，线程池中可以提交周期执行的任务，如下：

```java
/**
  * 简单心跳机制
  */
private void heartBeat(String ipAddress) {
    if(!mScheduledTasks.containsKey(ipAddress)){
        ScheduledFuture task = mScheduledExecutor.scheduleAtFixedRate(() -> {
            int result = PingManager.getInstance().ping(ipAddress);
            Log.d(TAG, "探测对方是否在线, result = " + result + ", ipAddress = " + ipAddress);
            if(result != 0){
                removeConnect(ipAddress);
                cancelScheduledTask(ipAddress);
            }
        }, 10, 10, TimeUnit.SECONDS);
        mScheduledTasks.put(ipAddress, task);
    }
}
```

它每隔10秒就会执行一次，然后会ping一下用户的ip地址，如果它不连通了，就要把它从在线用户中移除。

## 开发过程中遇到的问题及解决办法

- **问题1：获取获得在线用户列表和如果告诉别人我上线了？**

因为第一次开发P2P应用，所以不知道用户体系建立的逻辑，尝试的第一种方法是：使用Ping命令把ip地址的最后三位用循环从0~255不断的ping，如果ping通，就说明这个ip地址的用户连接着局域网的同一个WIFI，把它记录下来，但是这有一个缺点，能ping通的ip地址只是说明这个用户连着WIFI，并没有说明这个用户打开了P2P应用，也并不代表这个用户上线了，所以这种方法不行；后来想到了一种改进办法：就是把ping通的ip地址列表逐个建立Socket连接，如果能够连接上，就说明这个用户打开了P2P应用并且上线了，这个办法可以，但是逐个建立连接有很麻烦，耗时。

解决办法：就是使用UDP的广播，UDP广播能够告诉同一局域网内的所有打开了同一端口的在线用户我上线了，并且收到他们的回复。

- **问题2：用户头像的发送？**

因为使用UDP广播，但是UDP广播每次最大只能发送64Kb数据，一个头像就算压缩了，也有几百Kb，所以如何把头像发送出去是一个问题，尝试的第一种方法是：把头像转化成字节数组和用户信息一起转化为json数据，json数据再转化为字节数组，然后把json数据的字节数组分段发送出去，但是这有一个缺点，就是会额外增大发送时UDP的字节数组的长度导致发送额外多的字节，耗时，这种方法不行；尝试的第二种方法是：把头像和用户信息分开发送，先发送用户信息，然后再把头像转化为字节数组分段发送，但是有一个无法解决的问题，就是UDP是不可靠，很难保证分段后的字节再重新组合成一个完整的头像字节数组，会有顺序问题，所以这种办法不行。

解决办法：用户信息用UDP广播发送，因为用户信息短，不用分段，然后等获取到在线用户列表后再逐一建立TCP连接把用户头像发送给在线用户列表，TCP可靠。 

 