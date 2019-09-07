---
title: okhttp3源码分析之拦截器
date: 2019-09-07 15:01:40
tags: 
- okhttp
- 源码
categories: 优秀开源库分析
---

## 前言

* 上一篇文章：[okhttp3源码分析之请求流程](https://rain9155.github.io/2019/09/03/okhttp3源码分析之请求流程/)

本篇文章继续通过源码来探讨okhttp的另外一个重要知识点：拦截器，在上一篇文章我们知道，在请求发送到服务器之前有一系列的拦截器对请求做了处理后才发送出去，在服务器返回响应之后，同样的有一系列拦截器对响应做了处理后才返回给发起请求的调用者，可见，拦截器是okhttp的一个重要的核心功能，在分析各个拦截器功能的同时又会牵扯出okhttp的缓存机制、连接机制。

okhttp项目地址：[okhttp](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2Fsquare%2Fokhttp)

## 拦截器的简单使用

自定义一个拦截器需要实现Interceptor接口，接口定义如下：

```java
public interface Interceptor {
   
  //我们需要实现这个intercept(chain)方法，在里面定义我们的拦截逻辑
  Response intercept(Chain chain) throws IOException;

  interface Chain {
      
     //返回Request对象
    Request request();

    //调用Chain的proceed(Request)方法处理请求，最终返回Response
    Response proceed(Request request) throws IOException;


    //如果当前是网络拦截器，该方法返回Request执行后建立的连接
    //如果当前是应用拦截器，该方法返回null
    @Nullable Connection connection();

    //返回对应的Call对象
    Call call();

     //下面的方法见名知意，返回或写入超时
    int connectTimeoutMillis();
    Chain withConnectTimeout(int timeout, TimeUnit unit);
    int readTimeoutMillis();
    Chain withReadTimeout(int timeout, TimeUnit unit);
    int writeTimeoutMillis();
    Chain withWriteTimeout(int timeout, TimeUnit unit)
  }
}

```

可以看到Interceptor由两部分组成：intercept(Chain)方法和内部接口Chain，下面是自定义一个拦截器的通用逻辑，如下：

```java
public class MyInterceptor implements Interceptor {   
    @Override
    public Response intercept(Chain chain) throws IOException {
        
        //1、通过传进来的Chain获取Request
        Request request = chain.request();
        
      	//2、 处理Request，逻辑自己写
        //...
        
        //3、调用Chain的proceed(Request)方法处理请求，得到Response
        Response response = chain.proceed(request);
     
        //4、 处理Response，逻辑自己写
        //...
        
        //5、返回Response
        return response;
    }
}
```

上述就是一个拦截器的通用逻辑，首先我们继承Interceptor实现intercept(Chain)方法，完成我们自己的拦截逻辑，即根据需要进行1、2、3、4、5步，不管是自定义拦截器还是后面介绍的okhttp默认的拦截器大概都是这个模板实现，定义完拦截器后，我们在构造OkhttpChient时就可以通过addInterceptor(Interceptor)或addNetworkInterceptor(Interceptor)添加自定义拦截器，如下：

```java
OkHttpClient client = new OkHttpClient.Builder()
     .addInterceptor(new MyInterceptor())
     .build();

或

OkHttpClient client = new OkHttpClient.Builder()
     .addNetworkInterceptor(new MyInterceptor())
     .build();
```

这样okhttp在链式调用拦截器处理请求时就会调用到我们自定义的拦截器，那么addInterceptor(Interceptor)和addNetworkInterceptor(Interceptor)有什么不一样呢？它们一个是添加应用拦截器，一个是添加网络拦截器，主要是调用的时机不一样，更多区别可以参考官方WIKI文档[Okhttp-wiki 之 Interceptors 拦截器](https://www.jianshu.com/p/2710ed1e6b48)，当我们平时做应用开发使用addInterceptor(Interceptor)就行了。

上述是我们自定义的拦截器，下面我们来看看okhttp默认的拦截器都干了什么。

## RealCall  :: getResponseWithInterceptorChain()

在上一篇文章知道RealCall的getResponseWithInterceptorChain()是处理、发送请求并且返回响应的地方，我们再看一遍getResponseWithInterceptorChain()方法的源码，如下：

```java
//RealCall.java
Response getResponseWithInterceptorChain() throws IOException {
    //新建一个List用来保存拦截器
    List<Interceptor> interceptors = new ArrayList<>();
    //添加我们自定义的应用拦截器
    interceptors.addAll(client.interceptors());
    //添加负责重试重定向的拦截器
    interceptors.add(new RetryAndFollowUpInterceptor(client));
    //添加负责转换请求响应的拦截器
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    //添加负责缓存的拦截器
    interceptors.add(new CacheInterceptor(client.internalCache()));
    //添加负责管理连接的拦截器
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {//没有特殊要求，不使用WebSocket协议，WebSocket是什么？自行百度
      //添加我们自定义的网络拦截器
      interceptors.addAll(client.networkInterceptors());
    }
    //添加负责发起请求获取响应的拦截器
    interceptors.add(new CallServerInterceptor(forWebSocket));

    //构造首节点Chain
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, transmitter, null, 0,
        originalRequest, this, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    boolean calledNoMoreExchanges = false;
    try {
       //调用Chain的proceed(Request)方法处理请求
      Response response = chain.proceed(originalRequest);
      //...
      //返回响应
      return response;
    }
    //...省略异常处理
  }
```

getResponseWithInterceptorChain()干了三件事：1、添加拦截器到interceptors列表中；2、构造首节点Chain；3、调用Chain的proceed(Request)方法处理请求。下面分别介绍:

### 1、添加拦截器到interceptors列表中

除了添加我们自定义的拦截器外，还添加了默认的拦截器，如下：

* 1、RetryAndFollowUpInterceptor：负责失败重试和重定向。
* 2、BridgeInterceptor：负责把用户构造的Request转换为发送给服务器的Request和把服务器返回的Response转换为对用户友好的Response。
* 3、CacheInterceptor：负责读取缓存以及更新缓存。
* 4、ConnectInterceptor：负责与服务器建立连接并管理连接。
* 5、CallServerInterceptor：负责向服务器发送请求和从服务器读取响应。

这几个默认的拦截器是本文的重点，在后面会分别介绍。

### 2、构造首节点Chain

Chain的实现类是RealInterceptorChain，我们要对它的传进来的前6个构造参数有个印象，如下：

```java
public final class RealInterceptorChain implements Interceptor.Chain {
    
    //...
    
    public RealInterceptorChain(List<Interceptor> interceptors, Transmitter transmitter, @Nullable Exchange exchange, int index, Request request, Call call, int connectTimeout, int readTimeout, int writeTimeout) {
        this.interceptors = interceptors;//interceptors列表
        this.transmitter = transmitter;//Transmitter对象
        this.exchange = exchange;//Exchange对象
        this.index = index;//interceptor索性
        this.request = request;//请求request
        this.call = call;//Call对象
        //...
    }
    
    //...
}
```

我们知道，为了让每个拦截器都有机会处理请求，okhttp使用了责任链模式来把各个拦截器串联起来，所以Chain你可以把它想象成一条链中的一个节点，而这里的这个Chain就是请求传递的起点，也就是责任链的首节点。

### 3、调用Chain的proceed(Request)方法处理请求

如下：

```java
//RealInterceptorChain.java
@Override 
public Response proceed(Request request) throws IOException {
    return proceed(request, transmitter, exchange);
 }

public Response proceed(Request request, Transmitter transmitter, @Nullable Exchange exchange)
    throws IOException {
    
    //index不能越界
     if (index >= interceptors.size()) throw new AssertionError();
    
    //...
    
    //再新建一个Chain，这里注意index加1，
    RealInterceptorChain next = new RealInterceptorChain(interceptors, transmitter, exchange,
                                                         index + 1, request, call, connectTimeout, readTimeout, writeTimeout);
    
    //获取interceptors列表中的下一个拦截器
    Interceptor interceptor = interceptors.get(index);
    
    //调用下一个拦截器的intercept(Chain)方法，传入刚才新建的RealInterceptorChain，返回Response
    Response response = interceptor.intercept(next);
    
    //...
    
    //返回响应
    return response;
}
```

proceed(Request)方法中会调用下一个拦截器的intercept(Chain)方法，从interceptors列表获取拦截器时是通过index获取的，而构造Chain时传入了index + 1，并在调用下一个拦截器的intercept(Chain)方法时传了进去，我们再回顾一下上面所讲的拦截器intercept(Chain)方法的模板，里面会再次调用传入的Chain的proceed(Request)方法，这样又会重复上述逻辑，这样就把每个拦截器通过一个个Chain连接起来，形成一条链，把Request沿着链传递下去，直到请求被处理，然后返回Response，响应同样的沿着链传递上去，如下：



从上图可知，责任链的尾节点就是CallServerInterceptor对应的Chain，CallServerInterceptor处于最底层，Request是按照interpretor的顺序正向处理，而Response是逆向处理的，每个interpretor都有机会处理Request和Response，一个完美的责任链模式的实现。

知道了getResponseWithInterceptorChain()的整体流程后，下面分别介绍各个默认拦截器的功能。

## RetryAndFollowUpInterceptor

## BridgeInterceptor

## CacheInterceptor

## ConnectInterceptor

## CallServerInterceptor

## 结语

