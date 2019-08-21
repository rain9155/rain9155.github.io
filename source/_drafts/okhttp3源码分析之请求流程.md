---
title: okhttp源码分析之请求流程
tags: 
- okhttp
- 源码
categories: 优秀开源库分析
---

## 前言

在Android开发中，当下最火的网络请求框架莫过于okhttp和retrofit，它们都是square公司的产品，两个都是非常优秀开源库，值得我们去阅读它们的源码，学习它们的设计理念，但其实retrofit底层还是用okhttp来发起网络请求的，所以深入理解了okhttp也就深入理解了retrofit，它们的源码阅读顺序应该是先看okhttp，我在retrofit上发现它最近的一次提交才把okhttp版本更新到3.14，okhttp目前最新的版本是4.0.x，okhttp从4.0.x开始采用kotlin编写，在这之前还是用java，而我本次分析的okhttp源码版本是基本3.14.x，看哪个版本的不重要，重要的是阅读过后的收获，我打算分3篇文章去分析okhttp，分别是：

* 请求流程(同步、异步)
* 拦截器(Interceptor)
* 分发器(Dispatcher)

本文是第一篇 - okhttp的请求流程，okhttp项目地址：[okhttp](https://github.com/square/okhttp)

```
本文源码基于okhttp_3.14.x分支
```

## okhttp的简单使用

我们通过一个简单的GET请求来回忆一下okhttp的使用步骤，如下：

```java
//1、创建OkHttpClient
OkHttpClient client = new OkHttpClient.Builder()
    .readTimeout(5, TimeUnit.SECONDS)
    .build();

//2、创建请求Request
Request request = new Request.Builder()
    .url("http://www.baidu.com")
    .build();
//3、创建一个Call，用于发起网络请求
Call call = client.newCall(request);

//4、发起GET请求

//4.1、同步请求，调用Call的execute()方法
try {
    //接收到回复Response
    Response response = call.execute();
    Log.d(TAG, response.body().string());
} catch (IOException e) {
    e.printStackTrace();
}

//4.2、异步请求, 调用Call的enqueue()方法
call.enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {
    }

    @Override
    public void onResponse(Call call, Response response) throws IOException {
        //接收到回复Response
        Log.d(TAG, response.body().string());
    }
});
```

可以看到，使用okhttp发起网络请求要经过4步：

* 1、创建OkHttpClient
* 2、创建请求Request
* 3、通过OkHttpClient和Request创建一个Call，用于发起网络请求
* 4、调用Call的execute()或enqueue()方法发起同步或异步请求

当网络请求的响应返回时使用Response来接收，这就是一个典型的HTTP的请求/响应流程(不熟悉HTTP的，可以看[Http网络请求浅析](https://blog.csdn.net/Rain_9155/article/details/83066847))，对应okhttp的Request和Response。

### 1.1、创建OkHttpClient

OkHttpClient是okhttp中的大管家，它将具体的工作分发到各个子系统中去完成，它使用[Builder模式](https://blog.csdn.net/Rain_9155/article/details/82936136)配置网络请求的各种参数如超时、拦截器、分发器等，Builder中可配置的参数如下：

```java
//OkHttpClient.Builder
public static final class Builder {
    Dispatcher dispatcher;//分发器
    @Nullable Proxy proxy;
    List<Protocol> protocols;
    List<ConnectionSpec> connectionSpecs;
    final List<Interceptor> interceptors = new ArrayList<>();//应用拦截器
    final List<Interceptor> networkInterceptors = new ArrayList<>();//网络拦截器
    EventListener.Factory eventListenerFactory;
    ProxySelector proxySelector;
    CookieJar cookieJar;
    @Nullable Cache cache;
    @Nullable InternalCache internalCache;
    SocketFactory socketFactory;
    @Nullable SSLSocketFactory sslSocketFactory;
    @Nullable CertificateChainCleaner certificateChainCleaner;
    HostnameVerifier hostnameVerifier;
    CertificatePinner certificatePinner;
    Authenticator proxyAuthenticator;
    Authenticator authenticator;
    ConnectionPool connectionPool;
    Dns dns;
    boolean followSslRedirects;
    boolean followRedirects;
    boolean retryOnConnectionFailure;
    int callTimeout;
    int connectTimeout;
    int readTimeout;
    int writeTimeout;
    int pingInterval;

    //这里是配置默认的参数
    public Builder() {
      dispatcher = new Dispatcher();
      protocols = DEFAULT_PROTOCOLS;
      connectionSpecs = DEFAULT_CONNECTION_SPECS;
      eventListenerFactory = EventListener.factory(EventListener.NONE);
      proxySelector = ProxySelector.getDefault();
      if (proxySelector == null) {
        proxySelector = new NullProxySelector();
      }
      cookieJar = CookieJar.NO_COOKIES;
      socketFactory = SocketFactory.getDefault();
      hostnameVerifier = OkHostnameVerifier.INSTANCE;
      certificatePinner = CertificatePinner.DEFAULT;
      proxyAuthenticator = Authenticator.NONE;
      authenticator = Authenticator.NONE;
      connectionPool = new ConnectionPool();
      dns = Dns.SYSTEM;
      followSslRedirects = true;
      followRedirects = true;
      retryOnConnectionFailure = true;
      callTimeout = 0;
      connectTimeout = 10_000;
      readTimeout = 10_000;
      writeTimeout = 10_000;
      pingInterval = 0;
    }

    //这里传入自己配置的构建参数
    Builder(OkHttpClient okHttpClient) {
      this.dispatcher = okHttpClient.dispatcher;
      this.proxy = okHttpClient.proxy;
      this.protocols = okHttpClient.protocols;
      //...
    }
    
    //...
    
    //通过Builder的参数创建一个OkHttpClient
    public OkHttpClient build() {
        return new OkHttpClient(this);
    }
}
```

### 1.2、创建请求Request

