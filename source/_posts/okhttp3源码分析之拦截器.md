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

> 本文源码基于okhttp3.14.x

okhttp项目地址：[okhttp](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2Fsquare%2Fokhttp)

<!--more-->

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

    //构造第一个Chain
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

getResponseWithInterceptorChain()干了三件事：1、添加拦截器到interceptors列表中；2、构造第一个Chain；3、调用Chain的proceed(Request)方法处理请求。下面分别介绍:

### 1、添加拦截器到interceptors列表中

除了添加我们自定义的拦截器外，还添加了默认的拦截器，如下：

* 1、RetryAndFollowUpInterceptor：负责失败重试和重定向。
* 2、BridgeInterceptor：负责把用户构造的Request转换为发送给服务器的Request和把服务器返回的Response转换为对用户友好的Response。
* 3、CacheInterceptor：负责读取缓存以及更新缓存。
* 4、ConnectInterceptor：负责与服务器建立连接并管理连接。
* 5、CallServerInterceptor：负责向服务器发送请求和从服务器读取响应。

这几个默认的拦截器是本文的重点，在后面会分别介绍。

### 2、构造第一个Chain

Chain是Interceptor的一个内部接口，它的实现类是RealInterceptorChain，我们要对它的传进来的前6个构造参数有个印象，如下：

```java
public final class RealInterceptorChain implements Interceptor.Chain {
    
    //...
    
    public RealInterceptorChain(List<Interceptor> interceptors, Transmitter transmitter, @Nullable Exchange exchange, int index, Request request, Call call, int connectTimeout, int readTimeout, int writeTimeout) {
        this.interceptors = interceptors;//interceptors列表
        this.transmitter = transmitter;//Transmitter对象，后面会介绍
        this.exchange = exchange;//Exchange对象，后面会介绍
        this.index = index;//interceptor索性，用于获取interceptors列表中的interceptor
        this.request = request;//请求request
        this.call = call;//Call对象
        //...
    }
    
    //...
}
```

在后面的拦截器中都可以通过Chain获取这些传进来的参数。我们知道，为了让每个拦截器都有机会处理请求，okhttp使用了责任链模式来把各个拦截器串联起来，拦截器就是责任链的节点，而Chain就是责任链中各个节点之间的连接点，负责把各个拦截器连接起来。那么是怎么连接的？看下面的Chain的proceed方法。

### 3、调用Chain的proceed(Request)方法处理请求

实际是RealInterceptorChain的proceed(Request)方法，如下：

```java
public final class RealInterceptorChain implements Interceptor.Chain {
    
    //...
    
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
}
```

proceed方法里面首先会再新建一个Chain并且**index + 1**作为构造参数传了进去，然后通过index从interceptors列表中获取了一个拦截器，接着就会调用拦截器的intercept方法，并把刚刚新建的Chain作为参数传给拦截器，我们再回顾一下上面所讲的拦截器intercept方法的模板，intercept方法处理完Request逻辑后，会再次调用传入的Chain的proceed(Request)方法，这样又会重复Chain的proceed方法中的逻辑，由于index已经加1了，所以这次Chain就会通过index获取下一个拦截器，并调用下一个拦截器的intercept(Chain)方法，然后如此循环重复下去，这样就把每个拦截器通过一个个Chain连接起来，形成一条链，把Request沿着链传递下去，直到请求被处理，然后返回Response，响应同样的沿着链传递上去，如下：

{% asset_img okhttp1.png okhttp %}

从上图可知，责任链首节点就是RetryAndFollowUpInterceptor，尾节点就是CallServerInterceptor，Request按照拦截器的顺序正向处理，Response则逆向处理，每个拦截器都有机会处理Request和Response，一个完美的责任链模式的实现。

知道了getResponseWithInterceptorChain()的整体流程后，下面分别介绍各个默认拦截器的功能。

## RetryAndFollowUpInterceptor

在自定义拦截器的时候就讲过，Interceptor的intercept(Chain)方法就是拦截器的拦截实现，RetryAndFollowUpInterceptor的intercept(Chain)方法如下：

```java
//RetryAndFollowUpInterceptor.java
@Override 
public Response intercept(Chain chain) throws IOException {

    //获取Request
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    //获取Transmitter
    Transmitter transmitter = realChain.transmitter();

    //重定向次数
    int followUpCount = 0;
    Response priorResponse = null;
    
    //一个死循环
    while (true) {
        
        //调用Transmitter的prepareToConnect方法，做好连接建立的准备
        transmitter.prepareToConnect(request);

        if (transmitter.isCanceled()) {
            throw new IOException("Canceled");
        }
        
        Response response;
        boolean success = false;
        try {
            //调用proceed方法，里面调用下一个拦截器BridgeInterceptor的intercept方法
            response = realChain.proceed(request, transmitter, null);
            success = true;
        }catch (RouteException e) {//出现RouteException异常
            //调用recover方法检测连接是否可以继续使用
            if (!recover(e.getLastConnectException(), transmitter, false, request)) {
                throw e.getFirstConnectException();
            }
            continue;
        } catch (IOException e) {//出现IOException异常，和服务端建立连接失败
            boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
            //调用recover方法检测连接是否可以继续使用
            if (!recover(e, transmitter, requestSendStarted, request)) throw e;
            continue;
        } finally {//出现其他未知异常
            if (!success) {
                //调用Transmitter的exchangeDoneDueToException()方法释放连接
                transmitter.exchangeDoneDueToException();
            }
        }
        
        //执行到这里，没有出现任何异常，连接成功, 响应返回

        //...

        //根据响应码来处理请求头
        Request followUp = followUpRequest(response, route);

        //followUp为空，不需要重定向，直接返回Response
        if (followUp == null) {
            //...
            return response;
        }

        //followUp不为空，需要重定向

        //...

        //MAX_FOLLOW_UPS值为20，重定向次数不能大于20次
        if (++followUpCount > MAX_FOLLOW_UPS) {
            throw new ProtocolException("Too many follow-up requests: " + followUpCount);
        }

        //以重定向后的Request再次重试
        request = followUp;
        priorResponse = response;
    }
}
```

RetryAndFollowUpInterceptor的intercept(Chain)方法中主要是失败重试和重定向的逻辑，该方法流程如下：

1、首先获取Transmitter类；

2、然后进入一个死循环，先调用Transmitter的prepareToConnect方法，准备建立连接；（连接真正的建立在ConnectInterceptor中）

3、接着调用Chain的proceed方法，继续执行下一个拦截器BridgeInterceptor的intercept方法：

​	3.1、如果在请求的过程中抛出RouteException异常或IOException异常，就会调用recover方法检测连接是否可以继续使用，如果不可以继续使用就抛出异常，整个过程结束，否则就再次重试，这就是失败重试；

​	3.2、如果在请求的过程中抛出除了3.1之外的异常，就会调用Transmitter的exchangeDoneDueToException()方法释放连接，整个过程结束。

4、没有任何异常抛出，当响应Response返回后，就会调用followUpRequest方法，里面根据返回的Response的响应码来决定是否需要重定向（构造followUp请求），如果不需要重定向，就直接返回Response，如果需要重定向，那么以重定向后的Request再次重试，重定向次数不能大于20次。

### 1、Transmitter 

在整个方法的流程中出现了一个Transmitter，这里介绍一下，它是okhttp中应用层和网络层的桥梁，管理同一个Cal的所有连接、请求、响应和IO流之间的关系，它在RealCall创建后就被创建了，如下：

```java
//RealCall.java
static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    //创建RealCall
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    //创建Transmitter，赋值给call的transmitter字段
    call.transmitter = new Transmitter(client, call);
    return call;
}
```

创建后，在构造节点Chain时作为参数传了进去，在getResponseWithInterceptorChain方法中有讲到，所以在intercept方法中它可以通过chain.transmitter()获得，它的整个生命周期贯穿了所有拦截器，在接下来的ConnectInterceptor和CallServerInterceptor中你都可以见到它的身影，我们看一下它的主要成员，如下：

```java
public final class Transmitter {
    
    private final OkHttpClient client;//OkHttpClient大管家
    private final RealConnectionPool connectionPool;//连接池，管理着连接
    public RealConnection connection;//本次连接对象
    private ExchangeFinder exchangeFinder;//负责连接的创建
    private @Nullable Exchange exchange;//负责连接IO流读写
    private final Call call;//Call对象
    
    //...
    
    public Transmitter(OkHttpClient client, Call call) {
        this.client = client;
        this.connectionPool = Internal.instance.realConnectionPool(client.connectionPool());
        this.call = call;
        this.eventListener = client.eventListenerFactory().create(call);
        this.timeout.timeout(client.callTimeoutMillis(), MILLISECONDS);
    }

    
    public void prepareToConnect(Request request) {
        if (this.request != null) {
            if (sameConnection(this.request.url(), request.url()) && exchangeFinder.hasRouteToTry()) {
                return; // Already ready.
            }
           //...
        }
        this.request = request;
        //创建ExchangeFinder
        this.exchangeFinder = new ExchangeFinder(this, connectionPool, createAddress(request.url()), call, eventListener);
    }
    
}
```

在Transmitter中client和call我们都认识，剩下的RealConnectionPool、RealConnection、ExchangeFinder、Exchange都和okhttp的连接机制有关，都会在ConnectInterceptor中介绍，Transmitter就是负责管理它们之间的关系。这里我们只要记住，Transmitter的prepareToConnect方法中主要是创建了一个ExchangeFinder，为在ConnectInterceptor中连接的建立做了一个准备。

## BridgeInterceptor

BridgeInterceptor的intercept(Chain)方法如下：

```java
//BridgeInterceptor.java
@Override 
public Response intercept(Chain chain) throws IOException {
    
    //获取Request
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();
  
    //下面都是根据需要为Request的header添加或移除一些信息
    
    RequestBody body = userRequest.body();
    if (body != null) {
      MediaType contentType = body.contentType();
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString());
      }

      long contentLength = body.contentLength();
      if (contentLength != -1) {
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }
    }

    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }

    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive");
    }

    boolean transparentGzip = false;
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true;
      requestBuilder.header("Accept-Encoding", "gzip");
    }

    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }

    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }

    //调用proceed方法，里面调用下一个拦截器CacheInterceptor的intercept方法
    Response networkResponse = chain.proceed(requestBuilder.build());

    //返回Response后
    //下面都是根据需要为Response的header添加或移除一些信息
    
    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);

    if (transparentGzip && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding")) && HttpHeaders.hasBody(networkResponse)) {
      GzipSource responseBody = new GzipSource(networkResponse.body().source());
      Headers strippedHeaders = networkResponse.headers().newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build();
      responseBuilder.headers(strippedHeaders);
      String contentType = networkResponse.header("Content-Type");
      responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));
    }

    return responseBuilder.build();
  }
```

BridgeInterceptor中的逻辑是在所有默认拦截器中是最简单，它主要就是对Request或Response的header做了一些处理，把用户构造的Request转换为发送给服务器的Request，还有把服务器返回的Response转换为对用户友好的Response。例如，对于Request，当开发者没有添加Accept-Encoding时，它会自动添加Accept-Encoding : gzip，表示客户端支持使用gzip；对于Response，当Content-Encoding是gzip方式并且客户端是自动添加gzip支持时，它会移除Content-Encoding、Content-Length，然后重新解压缩响应的内容。

## CacheInterceptor

CacheInterceptor的intercept(Chain)方法如下：

```java
//CacheInterceptor.java
@Override
public Response intercept(Chain chain) throws IOException {
    
    //根据Request得到Cache中缓存的Response，Cache是什么，后面介绍
    Response cacheCandidate = cache != null ? cache.get(chain.request()) : null;

    long now = System.currentTimeMillis();

    //创建缓存策略：网络、缓存、或两者都使用，CacheStrategy是什么，后面介绍
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    //得到networkRequest
    Request networkRequest = strategy.networkRequest;
    //得到cacheResponse，cacheResponse等于上面的cacheCandidate
    Response cacheResponse = strategy.cacheResponse;

    //...

    //这个Response缓存无效，close掉它
    if (cacheCandidate != null && cacheResponse == null) {
        closeQuietly(cacheCandidate.body()); 
    }

    //1、networkRequest为null且cacheResponse为null：表示强制使用缓存，但是没有缓存，所以构造状态码为504，body为空的Response
    if (networkRequest == null && cacheResponse == null) {
        return new Response.Builder()
            .request(chain.request())
            .protocol(Protocol.HTTP_1_1)
            .code(504)//状态码504
            .message("Unsatisfiable Request (only-if-cached)")
            .body(Util.EMPTY_RESPONSE)
            .sentRequestAtMillis(-1L)
            .receivedResponseAtMillis(System.currentTimeMillis())
            .build();
    }

   //2、networkRequest为null但cacheResponse不为null：表示强制使用缓存，并且有缓存，所以直接返回缓存的Response
    if (networkRequest == null) {
        return cacheResponse.newBuilder()
            .cacheResponse(stripBody(cacheResponse))
            .build();
    }

    Response networkResponse = null;
    try {
        //networkRequest不为null，所以可以发起网络请求，调用chain.proceed(Request)，里面调用下一个拦截器BridgeInterceptor的intercept方法，会返回网络请求得到的networkResponse
        networkResponse = chain.proceed(networkRequest);
       
    } finally {
        //发起网络请求出现IO异常或其他异常的处理
        //...
    }

    //3、networkRequest不为null且cacheResponse不为null：因为cacheResponse不为null，所以根据网络请求得到的networkResponse和缓存的cacheResponse做比较，来决定是否更新cacheResponse
    if (cacheResponse != null) {
        if (networkResponse.code() == HTTP_NOT_MODIFIED) {//HTTP_NOT_MODIFIED等于304，304表示服务器缓存有更新，所以客户端要更新cacheResponse
            //下面根据networkResponse更新(重新构造)cacheResponse
            Response response = cacheResponse.newBuilder()
                .headers(combine(cacheResponse.headers(), networkResponse.headers()))
                .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
                .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
                .cacheResponse(stripBody(cacheResponse))
                .networkResponse(stripBody(networkResponse))
                .build();
            networkResponse.body().close();
            //更新cacheResponse在本地缓存
            cache.trackConditionalCacheHit();
            cache.update(cacheResponse, response);
            return response;
        } else {//不需要更新cacheResponse，close掉它
            closeQuietly(cacheResponse.body());
        }
    }

    //4、networkRequest不为null但cacheResponse为null：cacheResponse为null，没有缓存使用，所以从networkResponse读取网络响应，构造Response准备返回
    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    //把Response缓存到Cache中
    if (cache != null) {
        if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
            //cacheResponse为null，所以这里是第一次缓存，把Response缓存到Cache中
            CacheRequest cacheRequest = cache.put(response);
            return cacheWritingResponse(cacheRequest, response);
        }

        if (HttpMethod.invalidatesCache(networkRequest.method())) {
            try {
                cache.remove(networkRequest);
            } catch (IOException ignored) {
                // The cache cannot be written.
            }
        }
    }

    //返回Response
    return response;
}
```

CacheInterceptor的intercept(Chain)里面定义了okhttp的缓存机制，我们先来了解两个类：Cache和CacheStrategy，这样才能看懂intercept(Chain)里面的逻辑。

### 1、Cache - 缓存实现

Cache是okhttp中缓存的实现，内部使用了DiskLruCache，如下：

```java
public final class Cache implements Closeable, Flushable {
    //...
    //内部都是通过DiskLruCache实现
    final DiskLruCache cache;
    
    //有一个InternalCache实现，都调用了Cache中的方法
    final InternalCache internalCache = new InternalCache() {

        @Override public @Nullable Response get(Request request) throws IOException {
            return Cache.this.get(request);
        }

        @Override public @Nullable CacheRequest put(Response response) throws IOException {
            return Cache.this.put(response);
        }

        @Override public void remove(Request request) throws IOException {
            Cache.this.remove(request);
        }

        @Override public void update(Response cached, Response network) {
            Cache.this.update(cached, network);
        }

        @Override public void trackConditionalCacheHit() {
            Cache.this.trackConditionalCacheHit();
        }

        @Override public void trackResponse(CacheStrategy cacheStrategy) {
            Cache.this.trackResponse(cacheStrategy);
        }
    }
    
    //可以通过下面两个构造函数构造一个Cache
    
    public Cache(File directory, long maxSize) {
        this(directory, maxSize, FileSystem.SYSTEM);
    }

    Cache(File directory, long maxSize, FileSystem fileSystem) {
        this.cache = DiskLruCache.create(fileSystem, directory, VERSION, ENTRY_COUNT, maxSize);
    }

    //下面是主要方法
    
    @Nullable Response get(Request request) {
      	//...
    }

    @Nullable CacheRequest put(Response response) {
       //...
    }

    void remove(Request request) throws IOException {
        //...
    }

    void update(Response cached, Response network) {
      //...
    }
    
    synchronized void trackConditionalCacheHit() {
       //...
    }

    synchronized void trackResponse(CacheStrategy cacheStrategy) {
       //...
    }

    @Override public void flush() throws IOException {
       //...
    }

    @Override public void close() throws IOException {
        //...
    }
}
```

Cache中有一个内部实现类InternalCache，见名知意，它是okhttp内部使用的，它实现了InternalCache接口，接口中的方法都和Cache中的方法同名，而且这个实现类的所有方法都是调用了Cache中相应的方法，也就是说InternalCache的方法实现和Cache相应的方法一样，但Cache和InternalCache不一样的是，Cache比InternalCache多了一些方法供外部调用如flush()、 close()等，提供了更多对缓存的控制，而InternalCache中的方法都只是缓存的基本操作，如get、put、remove、update等方法，这些方法的逻辑都是基于Cache中的DiskLruCache实现，详情可以看[DiskLruCache](https://blog.csdn.net/guolin_blog/article/details/28863651)的原理实现。

要知道，okhttp默认是不使用缓存，也就是Cache为null，如果要使用缓存，我们需要自行配置，通过下面方法使用okhttp的缓存机制：

```java
//缓存的路径
File cacheDir = new File(Constant.PATH_NET_CACHE);
//这里通过带有两个参数的构造函数构造一个Cache
Cache cache = new Cache(cacheDir, 1024 * 1024 * 10);//缓存的最大尺寸10M

//然后设置给OkHttpClient
OkHttpClient client = new OkHttpClient.Builder()
    .cache(cache)
    .build();

```

通过上面全局设置后，Cache和InternalCache都不会为null，因为在创建Cache时InternalCache也一起创建了，okhttp的缓存机制就会生效。

我们先回到CacheInterceptor的intercept方法，它首先一开始就要判断cache是否等于null，那么CacheInterceptor的cache在哪里来的呢？是在构造函数中，如下：

```java
public final class CacheInterceptor implements Interceptor {
    final @Nullable InternalCache cache;

    public CacheInterceptor(@Nullable InternalCache cache) {
        this.cache = cache;
    }
    //...
}
```

可用看到它是InternalCache实例，在 getResponseWithInterceptorChain()中添加拦截器时就通过client为这个InternalCache赋值了，如下：

```java
//RealCall.java
Response getResponseWithInterceptorChain() throws IOException {
    //...
    //添加负责缓存的拦截器
    interceptors.add(new CacheInterceptor(client.internalCache()));
    //...
}
```

注意到new CacheInterceptor(client.internalCache())，所以我们看client的internalCache方法，如下：

```java
//OkHttpClient.java
@Nullable InternalCache internalCache() {
    return cache != null ? cache.internalCache : internalCache;
  }
```

cache就是上面全局设置的cache实例，所以不为null，返回cache中的internalCache实例，这样CacheInterceptor中就持有internalCache实例。

### 2、CacheStrategy - 缓存策略

CacheStrategy是okhttp缓存策略的实现，okhttp缓存策略遵循了HTTP缓存策略，因此了解okhttp缓存策略前需要有HTTP缓存相关基础：[HTTP 协议缓存机制详解](https://my.oschina.net/leejun2005/blog/369148)，了解了HTTP缓存策略后，我们再来看CacheStrategy，如下：

```java
public final class CacheStrategy {

    //CacheStrategy两个主要的成员变量：networkRequest、cacheResponse
    public final @Nullable Request networkRequest;
    public final @Nullable Response cacheResponse;

    CacheStrategy(Request networkRequest, Response cacheResponse) {
        this.networkRequest = networkRequest;
        this.cacheResponse = cacheResponse;
    }
    
    //...
    
    //通过工厂模式创建CacheStrategy
    public static class Factory {
        final long nowMillis;
        final Request request;
        final Response cacheResponse;

        public Factory(long nowMillis, Request request, Response cacheResponse) {
            this.nowMillis = nowMillis;
            this.request = request;
            this.cacheResponse = cacheResponse;
            //...
        }

        public CacheStrategy get() {
            CacheStrategy candidate = getCandidate();
            //...
            return candidate;
        }
        
         //...
    }
}
```

CacheStrategy是通过[工厂模式](https://rain9155.github.io/2019/09/07/工厂模式/#more)创建的，它有两个主要的成员变量：networkRequest、cacheResponse，CacheInterceptor的intercept方法通过CacheStrategy的networkRequest和cacheResponse的组合来判断执行什么策略，networkRequest是否为空决定是否请求网络，cacheResponse是否为空决定是否使用缓存，networkRequest和cacheResponse的4种组合和对应的缓存策略如下：

* 1、networkRequest为null且cacheResponse为null：没有缓存使用，又不进行网络请求，构造状态码为504的Response。
* 2、networkRequest为null但cacheResponse不为null：有缓存使用，且缓存在有效期内，所以直接返回缓存的Response。
* 3、networkRequest不为null且cacheResponse不为null：有缓存使用，但缓存在客户端的判断中表示过期了，所以请求服务器进行决策，来决定是否使用缓存的Response。
* 4、networkRequest不为null但cacheResponse为null：没有缓存使用，所以直接使用服务器返回的Response

networkRequest和cacheResponse在创建CacheStrategy时通过构造参数赋值，那么CacheStrategy在那里被创建呢？当调用CacheStrategy.Factory(long, Request, Response).get()时就会返回一个CacheStrategy实例，所以CacheStrategy在Factory的get方法中被创建，我们来看Factory的get方法，如下：

```java
//CacheStrategy.Factory
public CacheStrategy get() {
    CacheStrategy candidate = getCandidate();
    //...
    return candidate;
}
```

可以看到CacheStrategy通过Factory的getCandidate方法创建，getCandidate方法如下：

```java
//CacheStrategy.Factory
private CacheStrategy getCandidate() {
    //1、没有Response缓存，直接进行网络请求
    if (cacheResponse == null) {
        return new CacheStrategy(request, null);
    }

    //2、如果TLS握手信息丢失，直接进行网络请求
    if (request.isHttps() && cacheResponse.handshake() == null) {
        return new CacheStrategy(request, null);
    }

    //3、根据Response状态码，Expired和Cache-Control的no-Store进行判断Response缓存是否可用
    if (!isCacheable(cacheResponse, request)) {
        //Response缓存不可用，直接进行网络请求
        return new CacheStrategy(request, null);
    }

    
    //获得Request的缓存控制字段CacheControl
    CacheControl requestCaching = request.cacheControl();
    //4、根据Request中的Cache-Control的noCache和header是否设置If-Modified-Since或If-None-Match进行判断是否可以使用Response缓存
    if (requestCaching.noCache() || hasConditions(request)) {
        //不可以使用Response缓存，直接进行网络请求
        return new CacheStrategy(request, null);
    }
    
 	//走到这里表示Response缓存可用

    //获得Response的缓存控制字段CacheControl
    CacheControl responseCaching = cacheResponse.cacheControl();

    //获得该Response已经缓存的时长
    long ageMillis = cacheResponseAge();
    //获得该Response可以缓存的时长
    long freshMillis = computeFreshnessLifetime();

    if (requestCaching.maxAgeSeconds() != -1) 
        //一般取max-age
        freshMillis = Math.min(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds()));
    }

    long minFreshMillis = 0;
    if (requestCaching.minFreshSeconds() != -1) {
        //一般取0
        minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds());
    }

    long maxStaleMillis = 0;
    if (!responseCaching.mustRevalidate() && requestCaching.maxStaleSeconds() != -1) {
        //取max-stale，
        maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds());
    }

	//5、判断缓存是否过期，决定是否使用Response缓存：Response已经缓存的时长 < max-stale + max-age
    if (!responseCaching.noCache() && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
        Response.Builder builder = cacheResponse.newBuilder();
        if (ageMillis + minFreshMillis >= freshMillis) {
            builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"");
        }
        long oneDayMillis = 24 * 60 * 60 * 1000L;
        if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
            builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"");
        }
        //5.1、缓存没有过期，直接使用该Response缓存
        return new CacheStrategy(null, builder.build());
    }

   //5.2、缓存过期了，判断是否设置了Etag或Last-Modified等标记
    String conditionName;
    String conditionValue;
    if (etag != null) {
        conditionName = "If-None-Match";
        conditionValue = etag;
    } else if (lastModified != null) {
        conditionName = "If-Modified-Since";
        conditionValue = lastModifiedString;
    } else if (servedDate != null) {
        conditionName = "If-Modified-Since";
        conditionValue = servedDateString;
    } else {
        //缓存没有设置Etag或Last-Modified等标记，所以直接进行网络请求
        return new CacheStrategy(request, null);
    }

	//缓存设置了Etag或Last-Modified等标记，所以添加If-None-Match或If-Modified-Since请求头，构造请求，交给服务器判断缓存是否可用
    Headers.Builder conditionalRequestHeaders = request.headers().newBuilder();
    Internal.instance.addLenient(conditionalRequestHeaders, conditionName, conditionValue);

    Request conditionalRequest = request.newBuilder()
        .headers(conditionalRequestHeaders.build())
        .build();
    //networkRequest和cacheResponse都不为null
    return new CacheStrategy(conditionalRequest, cacheResponse);
}
```

 getCandidate()方法中根据[**HTTP的缓存策略**](https://my.oschina.net/leejun2005/blog/369148)决定networkRequest和cacheResponse的组合，从getCandidate()方法中我们可以看到HTTP的缓存策略分为两种：

* 1、强制缓存：**客户端参与决策决定是否继续使用缓存**，客户端第一次请求数据时，服务端返回了缓存的过期时间：Expires或Cache-Control，当客户端再次请求时，就判断缓存的过期时间，没有过期就可以继续使用缓存，否则就不使用，重新请求服务端。
* 2、对比缓存：**服务端参与决策决定是否继续使用缓存**，客户端第一次请求数据时，服务端会将缓存标识：Last-Modified/If-Modified-Since、Etag/If-None-Match和数据一起返回给客户端 ，当客户端再次请求时，客户端将缓存标识发送给服务端，服务端根据缓存标识进行判断，如果缓存还没有更新，可以使用，则返回304，表示客户端可以继续使用缓存，否则客户端不能继续使用缓存，只能使用服务器返回的新的响应。

而且强制缓存优先于对比缓存，我们再贴出来自[HTTP 协议缓存机制详解](https://my.oschina.net/leejun2005/blog/369148)的一张图，它很好的解释了getCandidate()方法中1~5步骤流程，如下：

{% asset_img okhttp2.png okhttp %}

### 3、缓存机制

我们再回到CacheInterceptor的intercept方法，它的1~4步骤就是CacheStrategy的networkRequest和cacheResponse的4种组合情况，都有详细的注释，每一种组合对应一种缓存策略，而缓存策略又是基于getCandidate()方法中写死的HTTP缓存策略，再结合okhttp本地缓存的实现Cache，我们得出结论：**okhttp的缓存机制 = Cache缓存实现 + 基于HTTP的缓存策略**，整个流程图如下：

{% asset_img okhttp3.png okhttp %}

了解了okhttp的缓存机制后，我们接着下一个拦截器ConnectInterceptor。

## ConnectInterceptor

ConnectInterceptor的intercept(Chain)方法如下：

```java
//ConnectInterceptor.java
@Override
public Response intercept(Chain chain) throws IOException {
   
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    //获取Transmitter
    Transmitter transmitter = realChain.transmitter();
	
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    //1、新建一个Exchange
    Exchange exchange = transmitter.newExchange(chain, doExtensiveHealthChecks);

    //调用proceed方法，里面调用下一个拦截器CallServerInterceptor的intercept方法
    //这里调用的proceed方法是带有三个参数的，它传进了Request、Transmitter和刚刚新建的Exchange
    return realChain.proceed(request, transmitter, exchange);
}
```

ConnectInterceptor的intercept(Chain)方法很简洁，里面定义了okhttp的连接机制，它首先获取Transmitter，然后通过Transmitter的newExchange方法创建一个Exchange，把它传到下一个拦截器CallServerInterceptor，Exchange是什么？Exchange负责从创建的连接的IO流中写入请求和读取响应，完成一次请求/响应的过程，在CallServerInterceptor中你会看到它真正的作用，这里先忽略。所以注释1的newExchange方法是连接机制的主要逻辑实现，我们继续看Transmitter的newExchange方法，如下：

```java
//Transmitter.java
Exchange newExchange(Interceptor.Chain chain, boolean doExtensiveHealthChecks) {

    //...省略异常处理

    //1、通过ExchangeFinder的find方法找到一个ExchangeCodec
    ExchangeCodec codec = exchangeFinder.find(client, chain, doExtensiveHealthChecks);

    //创建Exchange，并把ExchangeCodec实例codec传进去，所以Exchange内部持有ExchangeCodec实例
    Exchange result = new Exchange(this, call, eventListener, exchangeFinder, codec);

    //...
    
    return result;
}
```

重点是注释1，ExchangeFinder对象早在RetryAndFollowUpInterceptor中通过Transmitter的prepareToConnect方法创建，它的find方法是连接真正创建的地方，ExchangeFinder是什么？ExchangeFinder就是负责连接的创建，把创建好的连接放入连接池，如果连接池中已经有该连接，就直接取出复用，所以ExchangeFinder管理着两个重要的角色：RealConnection、RealConnectionPool，下面讲解一下RealConnectionPool和RealConnection，有助于连接机制的理解。

### 1、RealConnection - 连接实现

连接的真正实现，实现了Connection接口，内部利用Socket建立连接，如下：

```java
public interface Connection {
    //返回这个连接使用的Route
    Route route();

    //返回这个连接使用的Socket
    Socket socket();

    //如果是HTTPS，返回TLS握手信息用于建立连接，否则返回null
    @Nullable Handshake handshake();

    //返回应用层使用的协议，Protocol是一个枚举，如HTTP1.1、HTTP2
    Protocol protocol();
}

public final class RealConnection extends Http2Connection.Listener implements Connection {

    public final RealConnectionPool connectionPool;
    //路由
    private final Route route;
    //内部使用这个rawSocket在TCP层建立连接
    private Socket rawSocket;
    //如果没有使用HTTPS，那么socket == rawSocket，否则这个socket == SSLSocket
    private Socket socket;
    //TLS握手
    private Handshake handshake;
    //应用层协议
    private Protocol protocol;
    //HTTP2连接
    private Http2Connection http2Connection;
    //okio库的BufferedSource和BufferedSink，相当于javaIO的输入输出流
    private BufferedSource source;
    private BufferedSink sink;


    public RealConnection(RealConnectionPool connectionPool, Route route) {
        this.connectionPool = connectionPool;
        this.route = route;
    }


    public void connect(int connectTimeout, int readTimeout, int writeTimeout, int pingIntervalMillis, boolean connectionRetryEnabled, Call call, EventListener eventListener) {
        //...
    }

    //...
}
```

RealConnection中有一个connect方法，外部可以调用该方法建立连接，connect方法如下：

```java
//RealConnection.java
public void connect(int connectTimeout, int readTimeout, int writeTimeout, int pingIntervalMillis, boolean connectionRetryEnabled, Call call, EventListener eventListener) {
    if (protocol != null) throw new IllegalStateException("already connected");

    RouteException routeException = null;
    List<ConnectionSpec> connectionSpecs = route.address().connectionSpecs();
    ConnectionSpecSelector connectionSpecSelector = new ConnectionSpecSelector(connectionSpecs);

    //路由选择
    if (route.address().sslSocketFactory() == null) {
      if (!connectionSpecs.contains(ConnectionSpec.CLEARTEXT)) {
        throw new RouteException(new UnknownServiceException(
            "CLEARTEXT communication not enabled for client"));
      }
      String host = route.address().url().host();
      if (!Platform.get().isCleartextTrafficPermitted(host)) {
        throw new RouteException(new UnknownServiceException(
            "CLEARTEXT communication to " + host + " not permitted by network security policy"));
      }
    } else {
      if (route.address().protocols().contains(Protocol.H2_PRIOR_KNOWLEDGE)) {
        throw new RouteException(new UnknownServiceException(
            "H2_PRIOR_KNOWLEDGE cannot be used with HTTPS"));
      }
    }

    //开始连接
    while (true) {
      try {
        if (route.requiresTunnel()) {//如果是通道模式，则建立通道连接
          connectTunnel(connectTimeout, readTimeout, writeTimeout, call, eventListener);
          if (rawSocket == null) {
            // We were unable to connect the tunnel but properly closed down our resources.
            break;
          }
        } else {//1、否则进行Socket连接，大部分是这种情况
          connectSocket(connectTimeout, readTimeout, call, eventListener);
        }
        //建立HTTPS连接
        establishProtocol(connectionSpecSelector, pingIntervalMillis, call, eventListener);
        break;
      }
      //...省略异常处理

    if (http2Connection != null) {
      synchronized (connectionPool) {
        allocationLimit = http2Connection.maxConcurrentStreams();
      }
    }
  }
```

我们关注注释1，一般会调用connectSocket方法建立Socket连接，connectSocket方法如下：

```java
//RealConnection.java
private void connectSocket(int connectTimeout, int readTimeout, Call call,
                           EventListener eventListener) throws IOException {
    Proxy proxy = route.proxy();
    Address address = route.address();

    //根据代理类型的不同创建Socket
    rawSocket = proxy.type() == Proxy.Type.DIRECT || proxy.type() == Proxy.Type.HTTP
        ? address.socketFactory().createSocket()
        : new Socket(proxy);

    eventListener.connectStart(call, route.socketAddress(), proxy);
    rawSocket.setSoTimeout(readTimeout);
    try {
        //1、建立Socket连接
        Platform.get().connectSocket(rawSocket, route.socketAddress(), connectTimeout);
    }
    //...省略异常处理

    try {
        //获得Socket的输入输出流
        source = Okio.buffer(Okio.source(rawSocket));
        sink = Okio.buffer(Okio.sink(rawSocket));
    } 
     //...省略异常处理
}
```

我们关注注释1，Platform是okhttp中根据不同Android版本平台的差异实现的一个兼容类，这里就不细究，Platform的connectSocket方法最终会调用rawSocket的connect()方法建立其Socket连接，建立Socket连接后，就可以通过Socket连接获得输入输出流source和sink，okhttp就可以从source读取或往sink写入数据，source和sink是BufferedSource和BufferedSink类型，它们是来自于[okio库](https://github.com/square/okio)，它是一个封装了java.io和java.nio的库，okhttp底层依赖这个库读写数据，Okio好在哪里？详情可以看这篇文章[Okio好在哪](https://www.jianshu.com/p/2fff6fe403dd)。

### 2、RealConnectionPool -  连接池

连接池，用来管理连接对象RealConnection，如下：

```java
public final class RealConnectionPool {

    //线程池
    private static final Executor executor = new ThreadPoolExecutor(
        0 /* corePoolSize */,
        Integer.MAX_VALUE /* maximumPoolSize */, 
        60L /* keepAliveTime */, 
        TimeUnit.SECONDS,
        new SynchronousQueue<>(), 
        Util.threadFactory("OkHttp ConnectionPool", true));
 
    boolean cleanupRunning;
    //清理连接任务，在executor中执行
    private final Runnable cleanupRunnable = () -> {
        while (true) {
            //调用cleanup方法执行清理逻辑
            long waitNanos = cleanup(System.nanoTime());
            if (waitNanos == -1) return;
            if (waitNanos > 0) {
                long waitMillis = waitNanos / 1000000L;
                waitNanos -= (waitMillis * 1000000L);
                synchronized (RealConnectionPool.this) {
                    try {
                        //调用wait方法进入等待
                        RealConnectionPool.this.wait(waitMillis, (int) waitNanos);
                    } catch (InterruptedException ignored) {
                    }
                }
            }
        }
    };

    //双端队列，保存连接
    private final Deque<RealConnection> connections = new ArrayDeque<>();

    void put(RealConnection connection) {
        if (!cleanupRunning) {
            cleanupRunning = true;
            //使用线程池执行清理任务
            executor.execute(cleanupRunnable);
        }
        //将新建连接插入队列
        connections.add(connection);
    }

    long cleanup(long now) {
        //...
    }

    //...
}
```

RealConnectionPool 在内部维护了一个线程池，用来执行清理连接任务cleanupRunnable，还维护了一个双端队列connections，用来缓存已经创建的连接。要知道创建一次连接要经历TCP握手，如果是HTTPS还要经历TLS握手，握手的过程都是耗时的，所以为了提高效率，就需要connections来对连接进行缓存，从而可以复用；还有如果连接使用完毕，长时间不释放，也会造成资源的浪费，所以就需要cleanupRunnable定时清理无用的连接，okhttp支持5个并发连接，默认每个连接keepAlive为5分钟，keepAlive就是连接空闲后，保持存活的时间。

当我们第一次调用RealConnectionPool 的put方法缓存新建连接时，如果cleanupRunnable还没执行，它首先会使用线程池执行cleanupRunnable，然后把新建连接放入双端队列，cleanupRunnable中会调用cleanup方法进行连接的清理，该方法返回现在到下次清理的时间间隔，然后调用wiat方法进入等待状态，等时间到了后，再次调用cleanup方法进行清理，就这样往复循环。我们来看一下cleanup方法的清理逻辑：

```java
//RealConnectionPool.java
long cleanup(long now) {
    
    int inUseConnectionCount = 0;//正在使用连接数
    int idleConnectionCount = 0;//空闲连接数
    RealConnection longestIdleConnection = null;
    long longestIdleDurationNs = Long.MIN_VALUE;

    synchronized (this) {
        //遍历所有连接，记录空闲连接和正在使用连接各自的数量
        for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
            RealConnection connection = i.next();

            //如果该连接还在使用，pruneAndGetAllocationCount种通过引用计数的方式判断一个连接是否空闲
            if (pruneAndGetAllocationCount(connection, now) > 0) {
                //使用连接数加1
                inUseConnectionCount++;
                continue;
            }
            
            //该连接没有在使用

            //空闲连接数加1
            idleConnectionCount++;

            //记录keepalive时间最长的那个空闲连接
            long idleDurationNs = now - connection.idleAtNanos;
            if (idleDurationNs > longestIdleDurationNs) {
                longestIdleDurationNs = idleDurationNs;
                //这个连接很可能被移除，因为空闲时间太长
                longestIdleConnection = connection;
            }
        }
        
        //跳出循环后

        //默认keepalive时间keepAliveDurationNs最长为5分钟，空闲连接数idleConnectionCount最大为5个
        if (longestIdleDurationNs >= this.keepAliveDurationNs || idleConnectionCount > this.maxIdleConnections) {//如果longestIdleConnection的keepalive时间大于5分钟 或 空闲连接数超过5个
            //把longestIdleConnection连接从队列清理掉
            connections.remove(longestIdleConnection);
        } else if (idleConnectionCount > 0) {//如果空闲连接数小于5个 并且 longestIdleConnection连接还没到期清理
            //返回该连接的到期时间，下次再清理
            return keepAliveDurationNs - longestIdleDurationNs;
        } else if (inUseConnectionCount > 0) {//如果没有空闲连接 且 所有连接都还在使用
            //返回keepAliveDurationNs，5分钟后再清理
            return keepAliveDurationNs;
        } else {
            // 没有任何连接，把cleanupRunning复位
            cleanupRunning = false;
            return -1;
        }
    }

    //把longestIdleConnection连接从队列清理掉后，关闭该连接的socket，返回0，立即再次进行清理
    closeQuietly(longestIdleConnection.socket());

    return 0;
}
```

从cleanup方法得知，okhttp清理连接的逻辑如下：

1、首先遍历所有连接，记录空闲连接数idleConnectionCount和正在使用连接数inUseConnectionCount，在记录空闲连接数时，还要找出空闲时间最长的空闲连接longestIdleConnection，这个连接是很有可能被清理的；

2、遍历完后，根据最大空闲时长和最大空闲连接数来决定是否清理longestIdleConnection，

​	2.1、如果longestIdleConnection的空闲时间大于最大空闲时长 或 空闲连接数大于最大空闲连接数，那么该连接就会被从队列中移除，然后关闭该连接的socket，返回0，立即再次进行清理；

​	2.2、如果空闲连接数小于5个 并且 longestIdleConnection的空闲时间小于最大空闲时长即还没到期清理，那么返回该连接的到期时间，下次再清理；

​	2.3、如果没有空闲连接 且 所有连接都还在使用，那么返回默认的keepAlive时间，5分钟后再清理；

​	2.4、没有任何连接，idleConnectionCount和inUseConnectionCount都为0，把cleanupRunning复位，等待下一次put连接时，再次使用线程池执行cleanupRunnable。

了解了RealConnectionPool和RealConnection后，我们再回到ExchangeFinder的find方法，这里是连接创建的地方。

### 3、连接创建（连接机制）

ExchangeFinder的fing方法如下：

```java
//ExchangeFinder.java
public ExchangeCodec find( OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
   //...
    try {
        
      //调用findHealthyConnection方法，返回RealConnection
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,  writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);
        
      return resultConnection.newCodec(client, chain);  
    }
    //...省略异常处理
  }

 
  private RealConnection findHealthyConnection(int connectTimeout, int readTimeout, int writeTimeout, int pingIntervalMillis, boolean connectionRetryEnabled, boolean doExtensiveHealthChecks) throws IOException {
     //一个死循环
    while (true) {
        
       //调用findConnection方法，返回RealConnection
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          pingIntervalMillis, connectionRetryEnabled);
        
	  //...

        //判断连接是否可用
        if (!candidate.isHealthy(doExtensiveHealthChecks)) {
            candidate.noNewExchanges();
            continue;
        }

      return candidate;
    }
  
```

ExchangeFinder的find方法会调用findHealthyConnection方法，里面会不断调用findConnection方法，直到找到一个可用的连接返回。ExchangeFinder的findConnection方法如下：

```java
//ExchangeFinder.java
private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout, int pingIntervalMillis, boolean connectionRetryEnabled) throws IOException {
    boolean foundPooledConnection = false;
    RealConnection result = null;//返回结果，可用的连接
    Route selectedRoute = null;
    RealConnection releasedConnection;
    Socket toClose;
    synchronized (connectionPool) {
       if (transmitter.isCanceled()) throw new IOException("Canceled");
      hasStreamFailure = false; .

	 //1、尝试使用已经创建过的连接，已经创建过的连接可能已经被限制创建新的流
      releasedConnection = transmitter.connection;
      //1.1、如果已经创建过的连接已经被限制创建新的流，就释放该连接（releaseConnectionNoEvents中会把该连接置空），并返回该连接的Socket以关闭
      toClose = transmitter.connection != null && transmitter.connection.noNewExchanges
          ? transmitter.releaseConnectionNoEvents()
          : null;

        //1.2、已经创建过的连接还能使用，就直接使用它当作结果、
        if (transmitter.connection != null) {
            result = transmitter.connection;
            releasedConnection = null;
        }

        //2、已经创建过的连接不能使用
        if (result == null) {
            //2.1、尝试从连接池中找可用的连接，如果找到，这个连接会赋值先保存在Transmitter中
            if (connectionPool.transmitterAcquirePooledConnection(address, transmitter, null, false)) {
                //2.2、从连接池中找到可用的连接
                foundPooledConnection = true;
                result = transmitter.connection;
            } else if (nextRouteToTry != null) {
                selectedRoute = nextRouteToTry;
                nextRouteToTry = null;
            } else if (retryCurrentRoute()) {
                selectedRoute = transmitter.connection.route();
            }
        }
    }
	closeQuietly(toClose);
    
	//...
    
    if (result != null) {
        //3、如果在上面已经找到了可用连接，直接返回结果
        return result;
    }
    
    //走到这里没有找到可用连接

    //看看是否需要路由选择，多IP操作
    boolean newRouteSelection = false;
    if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {
        newRouteSelection = true;
        routeSelection = routeSelector.next();
    }
    List<Route> routes = null;
    synchronized (connectionPool) {
        if (transmitter.isCanceled()) throw new IOException("Canceled");

        //如果有下一个路由
        if (newRouteSelection) {
            routes = routeSelection.getAll();
            //4、这里第二次尝试从连接池中找可用连接
            if (connectionPool.transmitterAcquirePooledConnection(address, transmitter, routes, false)) {
                //4.1、从连接池中找到可用的连接
                foundPooledConnection = true;
                result = transmitter.connection;
            }
        }

        //在连接池中没有找到可用连接
        if (!foundPooledConnection) {
            if (selectedRoute == null) {
                selectedRoute = routeSelection.next();
            }

           //5、所以这里新创建一个连接，后面会进行Socket连接
            result = new RealConnection(connectionPool, selectedRoute);
            connectingConnection = result;
        }
    }

    // 4.2、如果在连接池中找到可用的连接，直接返回该连接
    if (foundPooledConnection) {
        eventListener.connectionAcquired(call, result);
        return result;
    }

    //5.1、调用RealConnection的connect方法进行Socket连接，这个在RealConnection中讲过
    result.connect(connectTimeout, readTimeout, writeTimeout, pingIntervalMillis, connectionRetryEnabled, call, eventListener);
    
    connectionPool.routeDatabase.connected(result.route());

    Socket socket = null;
    synchronized (connectionPool) {
        connectingConnection = null;
        //如果我们刚刚创建了同一地址的多路复用连接，释放这个连接并获取那个连接
        if (connectionPool.transmitterAcquirePooledConnection(address, transmitter, routes, true)) {
            result.noNewExchanges = true;
            socket = result.socket();
            result = transmitter.connection;
        } else {
            //5.2、把刚刚新建的连接放入连接池
            connectionPool.put(result);
            //5.3、把刚刚新建的连接保存到Transmitter的connection字段
            transmitter.acquireConnectionNoEvents(result);
        }
    }
    
    closeQuietly(socket);
    eventListener.connectionAcquired(call, result);
    
    //5.4、返回结果
    return result;
}
```

这个findConnection方法就是整个ConnectInterceptor的核心，我们忽略掉多IP操作和多路复用(HTTP2)，假设现在我们是第一次请求，连接池和Transmitter中没有该连接，所以跳过1、2、3，直接来到5，创建一个新的连接，然后把它放入连接池和Transmitter中；接着我们用同一个Call进行了第二次请求，这时连接池和Transmitter中有该连接，所以就会走1、2、3，如果Transmitter中的连接还可用就返回，否则从连接池获取一个可用连接返回，所以整个连接机制的大概过程如下：

{% asset_img okhttp4.png okhttp %}

Transmitter中的连接和连接池中的连接有什么区别？我们知道每创建一个Call，就会创建一个对应的Transmitter，一个Call可以发起多次请求（同步、异步），不同的Call有不同的Transmitter，连接池是在创建OkhttpClient时创建的，所以连接池是所有Call共享的，即连接池中的连接所有Call都可以复用，而Transmitter中的那个连接只是对应它相应的Call，只能被本次Call的所有请求复用。

了解了okhttp的连接机制后，我们接着下一个拦截器CallServerInterceptor。

## CallServerInterceptor

CallServerInterceptor的intercept(Chain)方法如下:

```java
//CallServerInterceptor.java
@Override
public Response intercept(Chain chain) throws IOException {
    
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    //获取Exchange
    Exchange exchange = realChain.exchange();
    //获取Request
    Request request = realChain.request();

	//通过Exchange的writeRequestHeaders(request)方法写入请求的header
    exchange.writeRequestHeaders(request);

    boolean responseHeadersStarted = false;
    Response.Builder responseBuilder = null;
    
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
        //...
        if (responseBuilder == null) {
            //通过okio写入请求的body
            if (request.body().isDuplex()) {
                exchange.flushRequest();
                BufferedSink bufferedRequestBody = Okio.buffer(
                    exchange.createRequestBody(request, true));
                request.body().writeTo(bufferedRequestBody);
            } else {
                BufferedSink bufferedRequestBody = Okio.buffer(
                    exchange.createRequestBody(request, false));
                request.body().writeTo(bufferedRequestBody);
                bufferedRequestBody.close();
            }
        } else {
           //...
        }
    } else {
      exchange.noRequestBody();
    }

    //...
    
    //下面开始获取网络请求返回的响应
    
    //通过Exchange的readResponseHeaders(boolean)方法读取响应的header
    if (responseBuilder == null) {
        responseBuilder = exchange.readResponseHeaders(false);
    }
    
    //获取响应后，通过Builder模式构造Response
    Response response = responseBuilder
        .request(request)
        .handshake(exchange.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    //...
    
    //构造Response的body
    if (forWebSocket && code == 101) {
        //构造一个空的body的Response
        response = response.newBuilder()
            .body(Util.EMPTY_RESPONSE)
            .build();
    } else {
        //通过Exchange的openResponseBody(Response)方法读取响应的body，然后通过响应的body继续构造Response
        response = response.newBuilder()
            .body(exchange.openResponseBody(response))
            .build();
    }
    
    //...

    //返回响应Response
    return response;
  }
```

在ConnectInterceptor中我们已经建立了连接，连接到了服务器，获取了输入输出流，所以CallServerInterceptor的intercept(Chain)方法逻辑就是把请求发送到服务器，然后获取服务器的响应，如下：

1、发送请求：

​	1.1、通过Exchange的writeRequestHeaders(request)方法写入请求的header；

​	1.2、如果请求的body不为空，通过okio写入请求的body。

2、获取响应：

​	2.1、通过Exchange的readResponseHeaders(boolean)方法读取响应的header；

​	2.2、通过Exchange的openResponseBody(Response)方法读取响应的body。

这个发送获取的过程通过Exchange进行，前面已经讲过它在ConnectInterceptor中创建，在process方法中传进来，所以这里可以通过Chain获取Exchange，Exchange它是负责从IO流中写入请求和读取响应，完成一次请求/响应的过程，它内部的读写都是通过一个ExchangeCodec类型的codec来进行，而ExchangeCodec内部又是通过Okio的BufferedSource和BufferedSink进行IO读写，这个过程在上一篇文章已经分析过了，这里不在累述。

## 结语

结合上一篇文章，我们对okhttp已经有了一个深入的了解，首先，我们会在请求的时候初始化一个Call的实例，然后执行它的execute()方法或enqueue()方法，内部最后都会执行到getResponseWithInterceptorChain()方法，这个方法里面通过拦截器组成的责任链，依次经过用户自定义普通拦截器、重试拦截器、桥接拦截器、缓存拦截器、连接拦截器和用户自定义网络拦截器和访问服务器拦截器等拦截处理过程，来获取到一个响应并交给用户。okhttp的请求流程、缓存机制和连接机制是当中的重点，在阅读源码的过程中也学习到很多东西，下一次就来分析它的搭档Retrofit。