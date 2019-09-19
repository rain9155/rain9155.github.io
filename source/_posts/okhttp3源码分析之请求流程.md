---
title: okhttp3源码分析之请求流程
tags:
  - okhttp
  - 源码
categories: 优秀开源库分析
date: 2019-09-03 22:49:13
---


## 前言

在Android开发中，当下最火的网络请求框架莫过于okhttp和retrofit，它们都是square公司的产品，两个都是非常优秀开源库，值得我们去阅读它们的源码，学习它们的设计理念，但其实retrofit底层还是用okhttp来发起网络请求的，所以深入理解了okhttp也就深入理解了retrofit，它们的源码阅读顺序应该是先看okhttp，我在retrofit上发现它最近的一次提交才把okhttp版本更新到3.14，okhttp目前最新的版本是4.0.x，okhttp从4.0.x开始采用kotlin编写，在这之前还是用java，而我本次分析的okhttp源码版本是基本3.14.x，看哪个版本的不重要，重要的是阅读过后的收获，我打算分2篇文章去分析okhttp，分别是：

* 请求流程(同步、异步)
* 拦截器(Interceptor)

本文是第一篇 - okhttp的请求流程，okhttp项目地址：[okhttp](https://github.com/square/okhttp)

<!--more-->

## okhttp的简单使用

我们通过一个简单的GET请求来回忆一下okhttp的使用步骤，并以这个实例为例讲解okhttp的请求流程，如下：

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

当服务器处理完一个请求Request后，就会返回一个响应，在okhttp中用Response代表HTTP的响应，这就是一个典型的[HTTP](https://rain9155.github.io/2018/12/31/Http网络请求浅析/)请求/响应流程。下面简单介绍1~3步骤：

### 1、创建OkHttpClient

OkHttpClient是okhttp中的大管家，它将具体的工作分发到各个子系统中去完成，它使用[Builder模式](https://rain9155.github.io/2019/09/07/Builder%E6%A8%A1%E5%BC%8F/#more)配置网络请求的各种参数如超时、拦截器、分发器等，Builder中可配置的参数如下：

```java
//OkHttpClient.Builder
public static final class Builder {
    Dispatcher dispatcher;//分发器
    @Nullable Proxy proxy;//代理
    List<Protocol> protocols;//应用层协议
    List<ConnectionSpec> connectionSpecs;//传输层协议
    final List<Interceptor> interceptors = new ArrayList<>();//应用拦截器
    final List<Interceptor> networkInterceptors = new ArrayList<>();//网络拦截器
    EventListener.Factory eventListenerFactory;//http请求回调监听
    ProxySelector proxySelector;//代理选择
    CookieJar cookieJar;//cookie
    @Nullable Cache cache;//网络缓存
    @Nullable InternalCache internalCache;//内部缓存
    SocketFactory socketFactory;//socket 工厂
    @Nullable SSLSocketFactory sslSocketFactory;//安全套接层socket 工厂，用于HTTPS
    @Nullable CertificateChainCleaner certificateChainCleaner;//验证确认响应证书，适用 HTTPS 请求连接的主机名
    HostnameVerifier hostnameVerifier;//主机名字确认
    CertificatePinner certificatePinner;//证书链
    Authenticator proxyAuthenticator;//代理身份验证
    Authenticator authenticator;//本地身份验证
    ConnectionPool connectionPool;//连接池,复用连接
    Dns dns;//域名
    boolean followSslRedirects;//安全套接层重定向
    boolean followRedirects;//本地重定向
    boolean retryOnConnectionFailure;//错误重连
    int callTimeout;//请求超时，它包括dns解析、connect、read、write和服务器处理的时间
    int connectTimeout;//connect超时
    int readTimeout;//read超时
    int writeTimeout;//write超时
    int pingInterval;//ping超时

    //这里是配置默认的参数
    public Builder() {
      dispatcher = new Dispatcher();
      protocols = DEFAULT_PROTOCOLS;//Protocol.HTTP_2和Protocol.HTTP_1_1
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

    //这里通过另外一个OkHttpClient配置参数
    Builder(OkHttpClient okHttpClient) {
      this.dispatcher = okHttpClient.dispatcher;
      this.proxy = okHttpClient.proxy;
      this.protocols = okHttpClient.protocols;
      //...
    }
    
    //...
    
    //配置完参数后，通过Builder的参数创建一个OkHttpClient
    public OkHttpClient build() {
        return new OkHttpClient(this);
    }
}
```

### 2、创建请求Request

在okhttp中Request代表着一个HTTP请求，它封装了请求的具体消息，如url、header、body等，它和OkHttpClient一样都是使用Budiler模式来配置自己的参数，如下：

```java
//Request.Budiler
public static class Builder {
    HttpUrl url;
    String method;
    Headers.Builder headers;
    RequestBody body;

    //这里配置默认的参数
    public Builder() {
      this.method = "GET";//默认是GET请求
      this.headers = new Headers.Builder();
    }

    //这里通过另外一个Request配置参数
    Builder(Request request) {
      this.url = request.url;
      this.method = request.method;
      //...
    }
    
    //...
    
    //配置完参数后，通过Builder的参数创建一个Request
    public Request build() {
        if (url == null) throw new IllegalStateException("url == null");
        return new Request(this);
    }
}
```

### 3、创建用于发起网络请求的Call

Call是一个接口，它的具体实现类是RealCall，Call中定义了一些enqueue(Callback)`、`execute()等关键方法，如下：

```java
public interface Call extends Cloneable {
    //返回当前请求
    Request request();
    //同步请求方法，此方法会阻塞当前线程直到请求结果放回
    Response execute() throws IOException;
    //异步请求方法，此方法会将请求添加到队列中，然后等待请求返回
    void enqueue(Callback responseCallback);
    //取消请求
    void cancel();
	//判断请求是否在执行
    boolean isExecuted();
    //判断请求是否取消
    boolean isCanceled();
    //返回请求的超时时间
    Timeout timeout();
    //克隆一个新的请求
    Call clone();
    interface Factory {
        Call newCall(Request request);
    }
}

```

我们看到Call接口中有一个Factory接口，Factory中有一个newCall(Request)方法，这说明Call是通过[工厂模式](https://rain9155.github.io/2019/09/07/%E5%B7%A5%E5%8E%82%E6%A8%A1%E5%BC%8F/#more)创建，而OkHttpClient实现了Call.Factory接口，重写了newCall(Request)方法，返回了Call的具体实现类RealCall，如下：

```java
public class OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory {
    //...
    @Override 
    public Call newCall(Request request) {
        //调用了RealCall的newRealCall()
        return RealCall.newRealCall(this, request, false /* for web socket */);
    }
}

final class RealCall implements Call {
    //...
    static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) 	{
        //返回RealCall对象
        RealCall call = new RealCall(client, originalRequest, forWebSocket);
        call.transmitter = new Transmitter(client, call);
        return call;
    }
}

```

所以调用**client.newCall(request)**其实返回的是RealCall对象，而RealCall封装了请求的调用逻辑。

到这里也就走到了注释4，也就是第4步，okhttp通过Call的实现类RealCall的execute()或enqueue()方法发起同步或异步请求，也就是本文的重点，下面分别详细介绍:

## 同步请求 - RealCall :: execute() 

```java
//RealCall.java
@Override
public Response execute() throws IOException {
    //...
    try {
        //1、调用Dispatcher的executed(RealCall)方法
        client.dispatcher().executed(this);
        //2、调用getResponseWithInterceptorChain()方法
        return getResponseWithInterceptorChain();
    } finally {
        //3、同步请求任务执行完毕，调用Dispatcher的finished(RealCall)方法
        client.dispatcher().finished(this);
    }
}
```

client就是我们上面所讲的OkHttpClient的实例，它在创建RealCall时作为构造参数传了进去，而OkHttpClient的dispatcher()方法返回的是Dispatcher实例，它在OkHttpClient构造时被创建。

我们先讲一下Dispatcher，那Dispatcher是什么呢？Dispatcher是一个任务调度器，它负责进行请求任务的调度，它的内部维护着3个任务队列(readyAsyncCalls、runningAsyncCalls、runningSyncCalls)和1个[线程池](https://rain9155.github.io/2019/07/19/java线程池/)(executorService)，Dispatcher主要内容如下：

```java
public final class Dispatcher {
    private int maxRequests = 64;//最大请求数64个
    private int maxRequestsPerHost = 5;//每个主机最大请求数5个
    private @Nullable Runnable idleCallback;//idle任务回调，类似于Android的idlehandler, 可以在Dispatcher没有任务调度（空闲时）时执行idleCallback中的任务

    //线程池，执行runningAsyncCalls队列里面的请求任务
    private @Nullable ExecutorService executorService;

    //等待执行的异步请求任务队列
    private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

    //正在执行的异步请求任务队列
    private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

    //正在执行的同步请求任务队列
    private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

    synchronized void executed(RealCall call) {
        //...
    }

    void enqueue(AsyncCall call) {
        //...
    }  
    
    void finished(RealCall call) {
        //...
    }

    void finished(AsyncCall call) {
        //...
    }
    
    private boolean promoteAndExecute() {
        //...
    }

  //...  
}
```

Dispatcher提供了executed(RealCall)和enqueue(AsyncCall)方法来进行同步和异步请求任务的入队，还提供了finished(RealCall)和finished(AsyncCalll)方法来进行同步和异步请求任务的出队，可以看到okhttp把ReadCall当作同步请求任务的代表，把AsyncCall当作异步请求任务的代表，RealCall前面已经讲过了，而AsyncCal是RealCall的一个内部类，它本质上就是一个Runnable，Dispatcher的线程池执行任务主要执行的是runningAsyncCalls队列里面的异步请求任务，也就是AsyncCall异步任务，而Dispatcher的promoteAndExecute()方法就是用来进行异步任务的调度，它的逻辑主要是按顺序把readyAsyncCalls队列中准备执行的异步任务转移到runningAsyncCalls后，再由线程池执行，对于同步任务Dispatcher只是暂时保存在runningSyncCalls队列中，并不会由线程池执行。

我们继续回到RealCall的execute()方法，根据注释1、2、3分为3部分解释同步请求流程，如下：

### 1、Dispatcher :: executed(RealCall)

看RealCall的execute()方法的注释1，它首先调用了Dispatcher的executed(RealCall)方法，Dispatcher的executed(RealCall)方法实现如下：

```java
//Dispatcher.java
synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
}
```

可以看到没有做什么处理，只是简单的把同步请求任务放入runningSyncCalls队列。

### 2、RealCall  :: getResponseWithInterceptorChain()

看RealCall的execute()方法的注释2，调用getResponseWithInterceptorChain()方法，这里才是同步请求处理的地方，我们点进去，如下：

```java
//RealCall.java 
Response getResponseWithInterceptorChain() throws IOException {
    //拦截器的添加
    List<Interceptor> interceptors = new ArrayList<>();
    //添加用户自定义拦截器
    interceptors.addAll(client.interceptors());
    //添加默认拦截器
    interceptors.add(new RetryAndFollowUpInterceptor(client));
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    //添加的最后一个拦截器是CallServerInterceptor
    interceptors.add(new CallServerInterceptor(forWebSocket));

    //创建一个RealInterceptorChain，传入了interceptors和Request
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, transmitter, null, 0,
        originalRequest, this, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    try {
      //调用RealInterceptorChain的proceed(Request)方法处理请求
      Response response = chain.proceed(originalRequest);
      //...
      return response;
    } catch (IOException e) {
     //...
    } finally {
     //...
    }
  }
```

getResponseWithInterceptorChain()方法最终返回一个Response，也就是网络请求的响应，该方法中首先把用户自定义的拦截器和okhttp默认的拦截器封装到一个List中，然后创建RealInterceptorChain并执行proceed(Request)方法处理请求，RealInterceptorChain的proceed(Request)方法如下：

```java
//RealInterceptorChain.java
@Override 
public Response proceed(Request request) throws IOException {
    return proceed(request, transmitter, exchange);
 }

public Response proceed(Request request, Transmitter transmitter, @Nullable Exchange exchange)
    throws IOException {
    //...
    //再新建一个RealInterceptorChain，这里注意index加1，
    RealInterceptorChain next = new RealInterceptorChain(interceptors, transmitter, exchange,
                                                         index + 1, request, call, connectTimeout, readTimeout, writeTimeout);
    //获取interceptors列表中的下一个拦截器
    Interceptor interceptor = interceptors.get(index);
    //调用下一个拦截器的intercept(Chain)方法，传入刚才新建的RealInterceptorChain，返回Response
    Response response = interceptor.intercept(next);
    //...

    return response;
}
```

proceed()方法中再次新建了一个RealInterceptorChain，传入了index + 1，而获取拦截器时是通过index获取，这样每次都能获取到下一个拦截器，然后调用下一个拦截器的intercept(Chain)方法，intercept(Chain)方法中就是拦截器的主要功能实现，里面会继续调用传入的RealInterceptorChain的proceed()方法，这样又会重复上述逻辑，我们把拦截器看作一条链中的节点，这样每个拦截器就通过一个个RealInterceptorChain连接起来，形成一条链，这就是典型的[责任链模式](https://rain9155.github.io/2019/09/07/%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%BC%8F/#more)，从节点的首部开始把请求传递下去，每一个拦截器都有机会处理这个请求，这又像是一个递归的过程，直到最后一个拦截器器处理完请求后，才开始逐层返回Resquese，拦截器才是Okhttp核心功能所在，关于拦截器介绍下篇文章再讲，这里只需要知道每一个拦截器都代表了一个功能。

经过对拦截器的简单介绍后，我们知道最后一个添加的拦截器才是把请求发送出去并且返回响应的地方，我们看getResponseWithInterceptorChain()方法，最后一个拦截器的添加是CallServerInterceptor，所以我们直接看CallServerInterceptor的intercept(Chain)方法实现，如下：

```java
//CallServerInterceptor.java
@Override
public Response intercept(Chain chain) throws IOException {
    //强转成RealInterceptorChain
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    //获取Exchange
    Exchange exchange = realChain.exchange();
    //获取Request
    Request request = realChain.request();

	//1、通过Exchange的writeRequestHeaders(request)方法发送Request的header
    exchange.writeRequestHeaders(request);

    boolean responseHeadersStarted = false;
    Response.Builder responseBuilder = null;
    //因为前面已经讲了，默认是GET请求，而GET请求是没有body的，所以不会进入if分支
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      //省略的是发送Request的body过程
      //...
    } else {
      exchange.noRequestBody();
    }

    //GET请求body为空，进入这个分支，完成请求
    if (request.body() == null || !request.body().isDuplex()) {
      exchange.finishRequest();
    }

    //省略的是一些监听回调
    //...
    
    //下面开始获取网络请求返回的响应
    
    //2、通过Exchange的readResponseHeaders(boolean)方法获取响应的header
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

    //省略的是response对状态码code的处理
    //...
    
    //构造Response的body
    if (forWebSocket && code == 101) {
        //构造一个空的body的Response
        response = response.newBuilder()
            .body(Util.EMPTY_RESPONSE)
            .build();
    } else {
        //通过Exchange的openResponseBody(Response)方法获取响应的body，然后通过响应的body继续构造Response
        response = response.newBuilder()
            .body(exchange.openResponseBody(response))
            .build();
    }
    
    //...

    //返回响应Response
    return response;
  }
```

intercept(Chain）方法中主要做的就是**发送请求，获取响应**的事情，注释中已经写的很清楚了，发送请求要把header和body分开发送，而获取响应时也要分别获取header和body，而发送请求和获取响应两个过程都是通过一个Exchange对象进行的，Exchange是在构造RealInterceptorChain时就作为构造参数传进RealInterceptorChain中，一直都为null，直到在ConnectInterceptor的intercept()中才通过Transmitter的newExchange()被赋值，而ConnectInterceptor的下一个拦截器就是CallServerInterceptor，所以CallServerInterceptor可以通过Chain获取到Exchange实例，这里不用细究这个赋值过程，Exchange它主要是用来负责完成一次网络请求和响应的过程。

这里我以intercept(Chain）方法中注释1和注释2请求header的发送(wirte)和获取(read)为例了解Exchange的工作过程，首先看Exchange的writeRequestHeaders(Request)方法，如下：

```java
//Exchange.java
public void writeRequestHeaders(Request request) throws IOException {
    try {
        //主要是调用了codec的writeRequestHeaders(request)
        codec.writeRequestHeaders(request);
        //...
    } catch (IOException e) {
        //...
    }
}
```

我们再看Exchange的readResponseHeaders(boolean)方法，如下：

```java
//Exchange.java
public @Nullable Response.Builder readResponseHeaders(boolean expectContinue) throws IOException {
    try {
      //主要是调用了codec的readResponseHeaders(boolean)
      Response.Builder result = codec.readResponseHeaders(expectContinue);
      //...
      return result;
    } catch (IOException e) {
     //...
    }
  }
```

从Exchange的两个方法可以看出，它把 wirt和read header的任务都交给了codec，codec是什么呢？codec是ExchangeCodec类型，它是一个接口，它主要用来编码http请求并解码http返回结果，所以Exchange中真正干活的是ExchangeCodec，它的有两个实现类，分别是Http2ExchangeCodec和Http1ExchangeCodec，分别对应Http2.x和Http1.x，这里我们以Http1ExchangeCodec为例，查看它的writeRequestHeaders(request)和readResponseHeaders(boolean)方法，首先看Http1ExchangeCodec的writeRequestHeaders(request)方法，如下：

```java
//Http1ExchangeCodec.java
@Override 
public void writeRequestHeaders(Request request) throws IOException {
    String requestLine = RequestLine.get(
        request, realConnection.route().proxy().type());
    //调用了writeRequest()
    writeRequest(request.headers(), requestLine);
  }

 public void writeRequest(Headers headers, String requestLine) throws IOException {
    if (state != STATE_IDLE) throw new IllegalStateException("state: " + state);
    //可以看到通过sink把请求头写入IO流，发送到服务器，sink是BufferedSink类型
    sink.writeUtf8(requestLine).writeUtf8("\r\n");
    for (int i = 0, size = headers.size(); i < size; i++) {
      sink.writeUtf8(headers.name(i))
          .writeUtf8(": ")
          .writeUtf8(headers.value(i))
          .writeUtf8("\r\n");
    }
    sink.writeUtf8("\r\n");
    state = STATE_OPEN_REQUEST_BODY;
  }
```

我们再看Http1ExchangeCodec的readResponseHeaders(boolean)方法，如下：

```java
//Http1ExchangeCodec.java
@Override
public Response.Builder readResponseHeaders(boolean expectContinue) throws IOException {
   //...
    try {
        StatusLine statusLine = StatusLine.parse(readHeaderLine());
        Response.Builder responseBuilder = new Response.Builder()
            .protocol(statusLine.protocol)
            .code(statusLine.code)
            .message(statusLine.message)
            .headers(readHeaders());//调用了readHeaders()
        //...
        return responseBuilder;
    } catch (EOFException e) {
        //...
    }
}

 private Headers readHeaders() throws IOException {
    Headers.Builder headers = new Headers.Builder();
    //调用了readHeaderLine()，一行一行的读取header
    for (String line; (line = readHeaderLine()).length() != 0; ) {
      Internal.instance.addLenient(headers, line);
    }
    return headers.build();
  }

 private String readHeaderLine() throws IOException {
     //服务器响应返回，通过source从IO读取响应头，source是BufferedSource类型
    String line = source.readUtf8LineStrict(headerLimit);
    headerLimit -= line.length();
    return line;
  }

```

从Http1ExchangeCodec的两个方法可以看出，底层是通过BufferedSink把信息写入IO流，通过BufferedSource从IO流读取信息，BufferedSink和BufferedSource都是来自[okio](https://github.com/square/okio)这个开源库的，okhttp底层是通过okio来向网络中写入和读取IO的，想要了解更多可自行查看okio源码(okio也是square公司的产品)。

到此RealCall的 getResponseWithInterceptorChain()分析完，getResponseWithInterceptorChain()返回Response后，RealCall的execute() 方法就return了，我们就可以通过返回的Response获取我们想要的信息，但RealCall的execute() 方法就return后，还要继续执行finally 分支中的逻辑。

### 3、Dispatcher :: finished(RealCall)

我们继续看RealCall的execute()方法的注释3，调用Dispatcher的finished(AsyncCall)方法，如下：

```java
//Dispatcher.java
void finished(RealCall call) {
    //传进了runningSyncCalls队列
    finished(runningSyncCalls, call);
}

private <T> void finished(Deque<T> calls, T call) {
    Runnable idleCallback;
    synchronized (this) {
        //尝试移除队列中的同步请求任务
        if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
        idleCallback = this.idleCallback;
    }

    //紧接着调用promoteAndExecute()方法进行异步任务的调度，如果没有异步任务要进行，promoteAndExecute()返回false
    boolean isRunning = promoteAndExecute();

    //isRunning等于false且设置了idleCallback，会执行一遍idle任务
    if (!isRunning && idleCallback != null) {
        idleCallback.run();
    }
}
```

finished()方法中首先尝试从runningSyncCalls队列把刚才通过 executed()入队的同步任务RealCall移除，如果移除失败，就抛出异常，如果移除成功，就紧接着调用promoteAndExecute()方法进行异步任务的调度并尝试执行一遍idle任务，promoteAndExecute()方法在异步请求中再详细介绍。

### 小结

至此okhttp的同步请求过程分析完毕，这里总结一下：当我们调用call.execute()时，就会发起一个同步请求，而call的实现类是RealCall，所以实际执行的是realCall.execute()，realCall.execute()中执行Dispatcher的executed(RealCall)把这个同步请求任务保存进runningSyncCalls队列中，然后RealCall执行getResponseWithInterceptorChain()处理同步请求，请求经过层层拦截器后到达最后一个拦截器CallServerInterceptor，在这个拦截器中通过Exchange把请求发送到服务器，然后同样的通过Exchange获得服务器的响应，根据响应构造Response，然后返回，最后RealCall执行Dispatcher的finished(RealCall)把之前暂时保存的同步请求任务从runningSyncCalls队列中移除。

下面是同步请求过程的调用链：

{%  asset_img okhttp1.png okhttp %}

##  异步请求 - RealCall.enqueue(Callback)

```java
//RealCall.java
@Override
public void enqueue(Callback responseCallback) {
    //...
    //1、调用Dispatcher的enqueue(AsyncCall)方法
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```

异步请求执行的是RealCall的enqueue(Callback)方法，它比同步请求只是多了一个Callback，在Callback的 onResponse(Call, Response)回调中我们可以拿到网络响应返回的Response，RealCall的enqueue(Callback)方法中首先把Callback用AsyncCall包装起来，然后调用调用Dispatcher的enqueue(AsyncCall)方法。

### 1、Dispatcher :: enqueue(AsyncCall)

我们看Dispatcher的enqueue(AsyncCall)方法，如下：

```java
//Dispatcher.java
void enqueue(AsyncCall call) {
    synchronized (this) {
       readyAsyncCalls.add(call);
       //...
    }
    promoteAndExecute();
}

```

该方法首先把异步请求任务AsyncCall放入readyAsyncCalls队列，然后调用promoteAndExecute()进行异步任务的调度，我们看一下Dispatcher 是如何进行异步任务的调度的。

### 2、Dispatcher :: promoteAndExecute()

promoteAndExecute()方法如下：

```java
//Dispatcher.java
private boolean promoteAndExecute() {
    //准备一个正在执行任务列表executableCalls
    List<AsyncCall> executableCalls = new ArrayList<>();
    boolean isRunning;
    synchronized (this) {
        //1、这个for循环主要把readyAsyncCalls中等待执行的异步任务转移到runningAsyncCalls队列和executableCalls列表中去
        for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {

            //取出readyAsyncCalls中等待执行的异步任务
            AsyncCall asyncCall = i.next();

            //判断条件：1、正在运行的异步请求任务不能大于maxRequests；2、等待执行的异步任务的主机请求数不能大于maxRequestsPerHost
            if (runningAsyncCalls.size() >= maxRequests) break; 
            if (asyncCall.callsPerHost().get() >= maxRequestsPerHost) continue;
            //满足条件，进入下面逻辑

            //把这个等待执行的异步任务从readyAsyncCalls中移除
            i.remove();
            asyncCall.callsPerHost().incrementAndGet();
            
            //把这个等待执行的异步任务添加进executableCalls列表
            executableCalls.add(asyncCall);
            //把这个等待执行的异步任务添加进runningAsyncCalls队列
            runningAsyncCalls.add(asyncCall);
        }
        
        //runningCallsCount()里面的逻辑： return runningAsyncCalls.size() + runningSyncCalls.size();
        isRunning = runningCallsCount() > 0;
    }
    //2、这个for循环主要是执行executableCalls列表中的异步任务
    for (int i = 0, size = executableCalls.size(); i < size; i++) {
        AsyncCall asyncCall = executableCalls.get(i);
        //传进executorService，调用AsyncCall的executeOn()方法，由线程池执行这个异步任务
        asyncCall.executeOn(executorService());
    }

    return isRunning;
}
```

promoteAndExecute()方法中主要是2个for循环，注释1的第一个for循环是把符合条件的异步请求任务从readyAsyncCalls转移（提升）到runningAsyncCalls队列和添加到executableCalls列表中去，紧接着注释2的第二个for循环就是遍历executableCalls列表，从executableCalls列表中获取AsyncCall对象，并且调用它的executeOn()方法，executeOn()方法传进了一个Dispatcher的executorService，所以我们看AsyncCall的executeOn()方法，里面是真正执行异步请求任务的地方。

#### 2.1、AsyncCall :: executeOn(ExecutorService)

AsyncCall的executeOn()方法如下：

```java
//AsyncCall.java
void executeOn(ExecutorService executorService) {
    boolean success = false;
    try {
        //传进this，执行AsyncCall异步任务，AsyncCall本质是Runnable
        executorService.execute(this);
        success = true;
    } catch (RejectedExecutionException e) {
       //...
    } finally {
        if (!success) {
            //异步任务执行失败，调用Dispatcher的finished(AsyncCall)方法
            client.dispatcher().finished(this); 
        }
    }
    
```

可以看到，里面的主要逻辑就是调用 executorService.execute(this)执行当前的AsyncCall异步任务，前面已经说过AsyncCall实现了NamedRunnable，本质是Runnable，如下：

```java
final class AsyncCall extends NamedRunnable {
    //...
}

public abstract class NamedRunnable implements Runnable {
	//...	
  @Override
    public final void run() {
    String oldName = Thread.currentThread().getName();
    Thread.currentThread().setName(name);
    try {
      //run方法中执行execute()方法
      execute();
    } finally {
      Thread.currentThread().setName(oldName);
    }
  }

  protected abstract void execute();
}

```

线程池执行到此异步任务时，它的run方法就会被执行，而run方法主要调用execute()方法，而execute()方法是一个抽象方法，AsyncCall实现了NamedRunnable，所以AsyncCall重写了execute()实现了执行逻辑，所以我们直接看AsyncCal的execute()方法。

#### 2.2、AsyncCal :: execute()

AsyncCal的execute()方法如下：

```java
//AsyncCall.java
@Override 
protected void execute() {
    //...
    try {
        //调用RealCall的getResponseWithInterceptorChain()方法处理请求
        Response response = getResponseWithInterceptorChain();
        signalledCallback = true;
        //请求处理完毕，返回响应，回调Callback的onResponse()方法
        responseCallback.onResponse(RealCall.this, response);
    } catch (IOException e) {
        //...
    } finally {
        //异步请求任务执行完毕，调用Dispatcher的finished(AsyncCall)方法
        client.dispatcher().finished(this);
    }
}
```

AsyncCal的execute()方法的逻辑和前面介绍的同步请求过程殊途同归，首先调用RealCall的getResponseWithInterceptorChain()方法处理请求，请求处理完毕后，返回响应Response，这时回调我们调用Call.enqueue(Callback)时传进来的Callback的onResponse()方法，最后在finally语句中调用Dispatcher的finished(AsyncCall)方法来把异步请求任务从runningAsyncCalls队列中移除出去，这个移除逻辑和上面同步请求任务的移除逻辑一样，只是这次是从runningAsyncCalls移除而不是runningSyncCalls，如下：

```java
//Dispatcher.java
void finished(AsyncCal call) {
    //传进runningAsyncCalls，而不是runningSyncCalls
    finished(runningSyncCalls, call);
}
```

### 小结

至此okhttp的异步请求过程分析完毕，这里再次总结一下，当我们调用call.enqueue(Callback)时，就会发起一个异步请求，实际执行的是realCall.enqueue(Callback)，它比同步请求只是多了一个Callback参数，然后realCall.execute()中先把传进来的Callback包装成一个AsyncCall，然后执行Dispatcher的enqueue(AsyncCall)把这个异步请求任务保存进readyAsyncCalls队列中，保存后开始执行 promoteAndExecute()进行异步任务的调度，它会先把符合条件的异步请求任务从readyAsyncCalls转移到runningAsyncCalls队列和添加到executableCalls列表中去，然后遍历executableCalls列表，逐个执行AsyncCall 的executeOn(ExecutorService)，然后在这个方法中AsyncCall会把自己放进Dispatcher 的线程池，等待线程池的调度，当线程池执行到这个AsyncCall时，它的run方法就会被执行，从而执行重写的execute()方法，execute()方法中的流程和同步请求流程大致相同。

下面是异步请求过程的调用链：

{%  asset_img okhttp2.png okhttp %}

## 结语

okhttp通过Builder模式创建OkHttpClient、Request和Response，通过client.newCall(Resquest)创建一个Call，用于发起异步或同步请求，请求会经过Dispatcher、一系列拦截器，最后通过okio与服务器建立连接、发送数据并解析返回结果，这个过程如图：

{%  asset_img okhttp3.png okhttp %}

以上就是对okhttp的请求流程的分析，如有错误，欢迎指出。

参考文章：

[OkHttp 3.x 源码解析之Dispather分发器](https://tamicer.github.io/2017/10/31/okhttp3-2/)