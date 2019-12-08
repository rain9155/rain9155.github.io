---
title: 通过源码全面解剖Retrofit2的整体流程
tags:
  - retrofit
  - 源码
categories: 优秀开源库分析
date: 2019-10-23 22:00:22
---


## 前言

上两篇文章：

* [okhttp3源码分析之请求流程](https://rain9155.github.io/2019/09/03/okhttp3源码分析之请求流程/)
* [okhttp3源码分析之拦截器](https://rain9155.github.io/2019/09/07/okhttp3源码分析之拦截器/)

Retrofit与Okhttp是Android开发中最最热门的网络请求库，它们都来自square公司，Okhttp在前面的两篇文章中已经通过源码从请求流程和拦截器两个角度分析过，本文的主角是Retrofit，经过这几天的研究，我发现Retrofit只是一个对Okhttp网络请求框架的巧妙包装，它通过注解去定义一个HTTP请求，然后在底层通过Okhttp发起网络请求，就是这样的一个简单的过程，其间运用了很多的设计模式：外观模式、动态代理模式、适配器模式、装饰者模式等，其最核心的是动态代理模式，所以在此之前大家对动态代理要有一个了解：

[静态和动态代理模式](https://rain9155.github.io/2019/10/15/代理模式/)

其他的设计模式我会在讲解的过程中简单介绍，除了使用了大量的设计模式，Retrofit还应用了面向接口编程的思想，使得整个系统解耦彻底，本文会通过一个简单的Retrofit使用示例，然后引出Retrofit的核心类，面向接口思想、构建过程、动态代理和网络请求过程，通过这几部分来解剖Retrofit。

Retrofit的项目地址：[Retrofit](https://github.com/square/Retrofit)

> 本文源码基于Retrofit2.4

<!--more-->

## 一、Retrofit的简单使用

首先来回忆一下Retrofit的使用，我这里使用的是[Github](https://developer.github.com/)平台的开放api，这个api根据用户名获取一个用户信息，首先在你的AndroidManifest.xml中声明网络权限，然后：

1、创建一个Api接口

```java
public interface GithubService {

    @GET("users/{user}")
    Call<ResponseBody> getUserInfo(@Path("user") String user);

}
```

我用Retrofit的注解声明了一个GET请求的方法，Call是Retrofit中的Call，而不是Okhttp中的Call，而ResponseBody是Okhttp的ResponseBody。

2、创建Retrofit实例

```java
Retrofit Retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build();
```

使用Builder模式构建Retrofit实例，传入baseUrl，我们平常开发一般会添加Rxjava2CallAdapterFactory和GsonConverterFactory，但这里我没有使用addCallAdapterFactory(Factory)来添加CallAdapterFactory，也没有使用addConverterFactory(Factory)来添加ConverterFactory，都使用默认的CallAdapter和Converter，默认的CallAdapter返回的就是Retrofit中的Call类型，默认的Converter会把网络请求返回数据转化为Okhttp中的ResponseBody，这也就是我上面定义接口时，Cal<T>的T是ResponseBody的原因。

3、创建Api接口实例

```java
GithubService service = Retrofit.create(GithubService.class);
```

通过Retrofit实例的create方法创建Api接口实例GithubService。

4、调用Api接口的方法，生成Call

```java
 Call<ResponseBody> call = service.getUserInfo("rain9155");
```

调用Api接口的方法，会返回一个Call实例。

5、通过Call发起同步或异步请求，然后获取返回结果Response

```java
//同步请求
Response<ResponseBody> response = call.execute();
或
//异步请求 
call.enqueue(new Callback<ResponseBody>() {
            @Override
            public void onResponse(Call<ResponseBody> call, Response<ResponseBody> response){
            	//通过Response获取网络请求返回结果
                ResponseBody body = response.body();
                try {
                    Log.d(TAG, "请求结果：" +  body.string());
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }

            @Override
            public void onFailure(Call<ResponseBody> call, Throwable t) {

            }
        });
```

通过Call的execute或enqueue方法发起网络请求，当网络请求结果返回时，就可以通过Response的body方法获取，这里因为使用默认的Converter，所以获取到的body的是Okhttp的ResponseBody，log的输出结果是**JSON数据**。

这就是Retrofit发起网络请求的五步曲，如果除去第一步，和Okhttp的使用还是非常的相似的，因为Retrofit和Okhttp的类名有很多的重复，下面如果涉及Okhttp的相关类，我会特别说明，否则默认都是属于Retrofit的。

##  二、Retrofit的相关类介绍

先简单的介绍一下重要的类，会对后面的阅读有帮助，当我们把Retrofir的源码clone下来，发现里面有4个主要的模块，分别是：

* retrofit-adapter 
* retrofit-converters
* retrofit-mock
* retrofit

其中retrofit-mock是测试时用的，不关我们的事，和我们开发相关的是retrofit-adapter、retrofit-converters和retrofit模块，当我们没有使用外置的CallAdapter和Converters时，我们只需要依赖retrofit模块，retrofit模块中有3个非常重要的接口，分别是：

* **Call**: 网络请求执行器，用于执行同步或异步的网络请求，内部最终通过Okhttp的Call发起网络请求
* **CallAdapter**：网络请求适配器，它用于把默认的网络请求执行器的调用形式，适配成在不同平台下的网络请求执行器的调用形式，例如Retrofit默认通过Call，内部使用ExecutorCallbackCall通过handler来执行网络请求后的线程切换，通过添加RxjavaCallAdapter后，RxjavaCallAdapter把默认的网络请求执行器适配成Observerable或Flowable，这样我可以使用Rxjava的链式调用方式来执行网络请求后的线程切换。
* **Converter**：数据转化器，两个方向的转化，把Api接口方法的参数注解的值转化为网络请求执行器需要的数据类型，和把网络返回的数据转化为我们需要的数据类型。

还有一个Callback接口，用于回调网络请求成功或失败，很简单，就不介绍了，其中CallAdapter和Converter内部都有一个Factory类，它都是通过[工厂模式](https://blog.csdn.net/Rain_9155/article/details/82942275)创建，工厂模式就是**将复杂对象的实例化任务交给一个类去实现，使得使用者不用知道具体参数就可以实例化出所需要的对象**，在Retrofit中，想要获得CallAdapter或Converter的实例都需要通过Factory来获取。下面分别简单的介绍一下Call、CallAdapter和Converter接口的作用和在Retrofit下的默认实现。

### 1、Call

网络请求执行器，用于执行同步或异步的网络请求，Call接口如下：

```java
public interface Call<T> extends Cloneable {
  
  //发起同步请求，返回Response
  Response<T> execute() throws IOException;

  //发起异步请求，使用callback把Response回调出去
  void enqueue(Callback<T> callback);

 //当执行了execute或enqueue后，该方法返回true
  boolean isExecuted();
    
  //取消这次网络请求
  void cancel();

  //当执行了cancel()后，这个方法返回true
  boolean isCanceled();

  //clone一个Call
  Call<T> clone();

  //返回HTTP网络请求，这个Request是来自okhttp的
  Request request();
}

```

Retrofit的Call和Okhttp的Call接口定义的方法差不多，只是多了一个clone方法，在Retrofit中，Call的默认实现类是OkHttpCall，如下：

```java
final class OkHttpCall<T> implements Call<T> {
    private final ServiceMethod<T, ?> serviceMethod;//这个ServiceMethod很重要，它封装了Api接口方法中的注解和参数信息，一个ServiceMethod对应一个Method（在创建Api接口实例会讲到）

    private final @Nullable Object[] args;//代表着Api接口方法中的参数

    private @Nullable okhttp3.Call rawCall;//这是一个来自Okhttp的Call

    private @Nullable Throwable creationFailure;
    private boolean executed;
    private volatile boolean canceled;

    Override 
    public Response<T> execute() throws IOException {
        okhttp3.Call call;
        //...
        call = rawCall;
        return parseResponse(call.execute());
    }
    
    @Override 
    public void enqueue(final Callback<T> callback) {
        okhttp3.Call call;
        call = rawCall;
		//...
        call.enqueue(new okhttp3.Callback() {
            @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
                //...
                callback.onResponse(OkHttpCall.this, response);
            }

            @Override public void onFailure(okhttp3.Call call, IOException e) {
                callFailure(e);
            }
        });
    }
    
    //...
}
```

一个OkHttpCall实例就代表着一次网络请求，OkHttpCall里面大部分方法的逻辑都是转发给Okhttp的Call方法。

> 在本文，不管来自Okhttp的Call，还是来自Retrofit的Call，都可以理解为网络请求执行器。

### 2、CallAdapter

网络请求适配器，用于把默认的网络请求执行器的调用形式，适配成在不同平台下的网络请求执行器的调用形式，CallAdapter接口如下：

```java
public interface CallAdapter<R, T> {
  
  //返回响应body转换为Java对象时使用的类型
  //例如Call<ResponseBody>, Type就是ResponseBody，Type是来自java.lang.reflect包的
  Type responseType();

  //把Call<R>适配成 T 类型，就是将Retrofit的Call适配成另外一个T类型的'Call'
  T adapt(Call<R> call);

  //用于创建CallAdapter实例，通过get方法可以返回一个CallAdapter实例或null
  abstract class Factory {
    
    //根据Api接口定义的方法的返回值和注解信息，创建一个CallAdapter实例返回，如果这个Factory不能处理这个方法的返回值和注解信息，返回null，
    //注意这里的returnType != 上面的responseType，例如Call<ResponseBody>，returnType的Type就是Call
    public abstract @Nullable CallAdapter<?, ?> get(Type returnType, Annotation[] annotations,
        Retrofit Retrofit);

    //返回ParameterizedType的上限，即泛型类型的上限
    //例如Call<? extends ResponseBody>, Type就是ResponseBody
    protected static Type getParameterUpperBound(int index, ParameterizedType type) {
      return Utils.getParameterUpperBound(index, type);
    }

   //返回Type的原始类型
   //例如Type为Call<ResponseBody>，返回Call.class
    protected static Class<?> getRawType(Type type) {
      return Utils.getRawType(type);
    }
  }
}
```

CallAdapter接口很简单，只有两个方法responseType和adapt方法，和一个Factory类，其中Factory类的get方法可以获取一个CallAdapter示例，CallAdapter的responseType方法就是获得Call<T>中的T类型，这个Call<T>就是我们在定义Api接口方法时方法的返回参数，CallAdapter的**adapt方法**用于把传入的Call<R>适配成另外一个我们所期待的'Call'，这里使用到了[适配器模式](https://blog.csdn.net/Rain_9155/article/details/87903640)，适配器模式就是**在两个因接口不兼容的类之间加一个适配器，将一个类的接口变成客户端所期待的另一种接口，从而使得它们工作在一起**，至于怎么适配就需要看适配器中得adapt方法的实现，接下来我们看adapt方法在Retrofit中的默认实现。（在Android Platform下的默认实现，Platform的概念在构建过程中会讲到）

在Retrofit中，CallAdapter的默认实现是一个匿名类，可以通过CallAdapter的Factory获得，CallAdapter的Factory的默认实现是**ExecutorCallAdapterFactory**，如下：

```java
final class ExecutorCallAdapterFactory extends CallAdapter.Factory {
    
  //线程切换执行器（在Retrofit的创建过程中会讲）
  final Executor callbackExecutor;

  ExecutorCallAdapterFactory(Executor callbackExecutor) {
    this.callbackExecutor = callbackExecutor;
  }

  @Override
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit Retrofit) {
      if (getRawType(returnType) != Call.class) {
          return null;
      }
      //可以通过returnType获得responseType
      //因为returnType = Call<T>, 有了Call<T>, 当然可以获得Call中的T，而T就是responseType
      final Type responseType = Utils.getCallResponseType(returnType);
      //返回一个CallAdapter匿名类
      return new CallAdapter<Object, Call<?>>() {

          @Override 
          public Type responseType() {
              //返回responseType
              return responseType;
          }

          @Override 
          public Call<Object> adapt(Call<Object> call) {
              //默认CallAdapter的adapt方法返回
              return new ExecutorCallbackCall<>(callbackExecutor, call);
          }
      };
  }
  
 //网络请求执行器（在创建Api接口实例中会讲）
 static final class ExecutorCallbackCall<T> implements Call<T> {
     //...
  }  
    
}
```

在ExecutorCallAdapterFactory的get方法中，new了一个CallAdapter返回，在CallAdapter的adapt方法实现中，new了一个ExecutorCallbackCall返回，并把入参call和callbackExecutor传进了ExecutorCallbackCall的构造中，ExecutorCallbackCall就是一个实现了Call接口的类，还是一个Call，它就是Retrofit的默认网络请求执行器，可以看到Retrofit的默认的网络请求执行器适配，即adapt方法的默认实现就是**用ExecutorCallbackCall包装传进来的Call，并返回ExecutorCallbackCall，这个传进来的Call就是Call的默认实现OkHttpCall**，待会在Retrofit的构建过程中还会讲到。

既然CallAdapter能把默认的网络请求执行器的调用形式，适配成在不同平台下的网络请求执行器的调用形式，那么它支持哪些平台呢？这个在retrofit-adapter 模块中可以找到答案，如下：

{% asset_img retrofit1.png retrofit%}

Retrofit还支持guava、java8、rxjava、scala这四个平台，它们里面都各自实现了retrofit模块暴露出去的CallAdapter接口和CallAdapter接口中的Factory接口，在CallAdapter的adapt方法中提供各自平台的适配，我们可以通过addCallAdapterFactory(Factory)来添加不同平台的CallAdapter工厂。

### 3、Converter

数据转化器，把我们在Api接口定义的方法注解和参数转化为网络请求执行器需要的请求类型，和把网络返回的数据转化为我们需要的数据类型，Converter接口如下：

```java
public interface Converter<F, T> {
   
  //把F 转化为 T，用于在网络请求中实现对象的转化
  T convert(F value) throws IOException;

  //通过Factory的responseBodyConverter或requestBodyConverter方法获得一个Converter实例或null
  abstract class Factory {
    
    //返回一个处理网络请求响应（Response）的body的Converter实例，如果不能处理这些类型(type)和注解，返回null
    //这个Converter会把 ResponseBody 转化成 ?，这个ResponseBody是来自okhttp的
    //例如使用GsonConverter，？代表某个java对象类型
    public @Nullable Converter<ResponseBody, ?> responseBodyConverter(Type type,
        Annotation[] annotations, Retrofit Retrofit) {
      return null;
    }

    //返回一个处理网络请求（Request）的body的Converter实例，如果不能处理这些类型(type)和注解，返回null
    //这个Converter会把 ？转化成 RequestBody，这个RequestBody是来自okhttp的
    //这个Converter主要处理@Body、 @Part、@PartMap类型的注解
    public @Nullable Converter<?, RequestBody> requestBodyConverter(Type type,
        Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit Retrofit) {
      return null;
    }

    //返回一个处理网络请求（Request）的body的Converter实例
    //这个Converter会把 ？转化成 String
    //这个Converter主要处理@Field、@FieldMap、@Header、HeaderMap @HeaderMap、@Path、@Query、@QueryMap类型的注解
    public @Nullable Converter<?, String> stringConverter(Type type, Annotation[] annotations,
        Retrofit Retrofit) {
      return null;
    }

    //下面两个方法和上面CallAdapter的Factory中同名方法的意思一样
  
    protected static Type getParameterUpperBound(int index, ParameterizedType type) {
      return Utils.getParameterUpperBound(index, type);
    }

    protected static Class<?> getRawType(Type type) {
      return Utils.getRawType(type);
    }
  }
}

```

Converter接口也很简单，只有一个convert方法，和一个Factory类，因为Converter要提供两个方向的转化，所以Factory类就提供了两个方法用于获取不同方向的转化，其中responseBodyConverter方法就是获得一个把网络返回的数据转化为我们需要的数据类型的Converter实例，而requestBodyConverter方法就是获得一个把我们在Api接口定义的方法注解和参数转化为网络请求的Converter实例，那么要怎么转化呢？就要看Converter的convert方法的实现，convert方法把F类型 转化为 T类型，接下来我们看convert方法在Retrofit中的默认实现。

Converter在Retrofit的默认实现有五个，都是Converter的Factory的内部类，可通过Converter的Factory获得，Converter的Factory的默认实现是**BuiltInConverters**，如下：

```java
final class BuiltInConverters extends Converter.Factory {
    
  @Override
  public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
      Retrofit Retrofit) {
    if (type == ResponseBody.class) {//支持ResponseBody类型的转化
      
      //如果是二进制流形式，就返回StreamingResponseBodyConverter实例
      //如果是字符流形式，就返回BufferingResponseBodyConverter实例
      return Utils.isAnnotationPresent(annotations, Streaming.class)
          ? StreamingResponseBodyConverter.INSTANCE
          : BufferingResponseBodyConverter.INSTANCE;
    }
    if (type == Void.class) {//支持Void类型的转化
        
      //返回VoidResponseBodyConverter实例
      return VoidResponseBodyConverter.INSTANCE;
    }
      
    //除了以上两种类型，其他类型都不支持，返回null
    return null;
  }

  @Override
  public Converter<?, RequestBody> requestBodyConverter(Type type,
      Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit Retrofit) {
    //支持向RequestBody类型转化
    if (RequestBody.class.isAssignableFrom(Utils.getRawType(type))) {
        
      //返回RequestBodyConverter实例
      return RequestBodyConverter.INSTANCE;
    }
    
    //除了RequestBody类型，不支持向其他类型转化，返回null
    return null;
  }

   static final class VoidResponseBodyConverter implements Converter<ResponseBody, Void> {
    static final VoidResponseBodyConverter INSTANCE = new VoidResponseBodyConverter();

    @Override public Void convert(ResponseBody value) {
      value.close();
      return null;
    }
  }

  static final class RequestBodyConverter implements Converter<RequestBody, RequestBody> {
    static final RequestBodyConverter INSTANCE = new RequestBodyConverter();

    @Override public RequestBody convert(RequestBody value) {
      return value;
    }
  }

  static final class StreamingResponseBodyConverter
      implements Converter<ResponseBody, ResponseBody> {
    static final StreamingResponseBodyConverter INSTANCE = new StreamingResponseBodyConverter();

    @Override public ResponseBody convert(ResponseBody value) {
      return value;
    }
  }

  static final class BufferingResponseBodyConverter
      implements Converter<ResponseBody, ResponseBody> {
    static final BufferingResponseBodyConverter INSTANCE = new BufferingResponseBodyConverter();

    @Override public ResponseBody convert(ResponseBody value) throws IOException {
      try {
        // Buffer the entire body to avoid future I/O.
        return Utils.buffer(value);
      } finally {
        value.close();
      }
    }
  }

  static final class ToStringConverter implements Converter<Object, String> {
    static final ToStringConverter INSTANCE = new ToStringConverter();

    @Override public String convert(Object value) {
      return value.toString();
    }
  }
}

```

BuiltInConverters实现了Factory中的responseBodyConverter和requestBodyConverter方法，内部含有五个Converter默认实现类，在Factory的responseBodyConverter和requestBodyConverter方法中分别返回这几Converter实例，其中只有ToStringConverter没有使用到，我们还发现了这五个Converter的convert方法的实现除了BufferingResponseBodyConverter，大部分都是，入参是是什么，返回就是什么，所以Retrofit中Converter默认实现的convert方法大部分**都没有对数据进行转化，返回原始数据，这些原始数据是String或来自Okhttp的ResponseBody、RequestBody**，例如ResponseBodyConverter的convert方法就是返回Okhttp的ResponseBody，RequestBodyConverter的convert方法就是返回Okhttp的RequestBody。

Converter除了默认的返回原始数据，它还支持哪些数据转化呢？这个在retrofit-converters模块中可以找到答案，如下：

{% asset_img retrofit2.png Retrofit %}

可以看到Retrofit还支持json、xml、protobuf等多种数据类型的转化，这些子模块都各自实现了retrofit模块暴露出来的Converter接口和Converter接口中的Factory接口，在Converter的adapt方法中实现不同数据类型的转化逻辑，我们可以通过addConverterFactory(Factory)来支持不同数据类型转化的Converter工厂。

### 4、面向接口设计

在retrofit模块中提供了**Call、Callback、CallAdapter、Converter接口**供外部模块使用，CallAdapter和Converter接口中还有相应的Factory接口，各模块之前通过接口依赖主模块Retrofit，将网络请求、网络请求适配、请求处理与返回解析完全解耦，当需要修改Retrofit中的默认实现时，只需要add一个外部模块提供的工厂，具体创建什么样的实例由工厂方法来负责，这样就能以最小的代价（不需要改动代码）换成其他模块上的实现，Retrofit本身并不参与这个过程，它只是负责提供一些主要的参数供它们进行决策，以及进行参数的处理，模块之间依赖于接口，而不依赖于具体的实现，这是一种很好的编程思路，面向接口编程。

> 面向接口编程：也被熟知为基于接口的设计，是一种基于组件级别的，面向对象语言的模块化编程设计实现，面向接口编程是面向对象编程的一种模块化实现形式，理论上说具有对象概念的程序设计都可以称之为面向对象编程，而面向接口编程则是从组件的级别来设计代码，将抽象与实现分离。

好了，在简单的了解了一下待会和构建过程涉及到的相关类，接下来分析Retrofit的构建过程。

## 三、Retrofit的创建过程

```java
Retrofit Retrofit = new Retrofit.Builder()//1、构造Builder
    .baseUrl("https://api.github.com/")//2、配置Builder
    .build();//3、创建Retrofit实例
```

Retrofit是使用[Builder模式](https://blog.csdn.net/Rain_9155/article/details/82936136)构建出一个Retrofit实例的，Builder模式的好处**就是将一个复杂对象的构建和它的表示分离，在用户在不知道内部构建细节的情况下，可以更加精准的控制对象的构造过程**，所以我们直接看Retrofit的内部类Builder就行，因为在Builder中配置的字段最终都在build时赋值给Retrofit中相应的字段，Builder只是暂时保存这些配置字段而已，下面我们分注释的3步去看Retrofit的Builder.

### 1、构造Builder

Builder类如下：

```java
//Retrofit.java
public static final class Builder {

     private final Platform platform;//Retrofit运行的平台
     //...

     //根据Platform构造
     Builder(Platform platform) {
         this.platform = platform;
     }

     public Builder() {
         //调用Platform的get方法获取一个Platform
         this(Platform.get());
     }

     //根据另外一个Retrofit构造
     Builder(Retrofit Retrofit) {
         platform = Platform.get();
         callFactory = Retrofit.callFactory;
         baseUrl = Retrofit.baseUrl;
         //...
     }

   //...
  }
```

new Retrofit.Builder()，我们使用了无参的构造函数来创建Builder，Builder的无参构造函数首先通过**Platform.get()**获取了一个Platform实例赋值给了platform字段，前面提到过Retrofit是有Platform的概念，Retrofit支持java和Android平台，我们来看一下Platform这个类，如下：

```java
class Platform {   
    //单例
    private static final Platform PLATFORM = findPlatform();

    //get方法
    static Platform get() {
        //返回Platform实例
        return PLATFORM;
    }

    private static Platform findPlatform() {
        
        try {
            //android.os.Build这个类是Android独有的
            //这里要求JVM查找并加载Build.class对象
            Class.forName("android.os.Build");
            if (Build.VERSION.SDK_INT != 0) {
                //如果是Android平台，创建一个Android实例返回，Android继承自Platform
                return new Android();
            }
        } catch (ClassNotFoundException ignored) {
            //找不到android.os.Build，会抛出异常，捕捉，继续执行
        }
        
        try {
            //java.util.Optional这个类是java8之后才有的，是java独有的
            Class.forName("java.util.Optional");
            //创建一个Java8实例返回，Java8继承自Platform
            return new Java8();
        } catch (ClassNotFoundException ignored) {
            //找不到java.util.Optional，会抛出异常，捕捉，继续执行
        }
        
        //返回默认Platform实例
        return new Platform();
    }

    //返回默认的线程切换执行器
    Executor defaultCallbackExecutor() {
        return null;
    }
    
    //返回默认的网络请求适配器工厂
    CallAdapter.Factory defaultCallAdapterFactory(@Nullable Executor callbackExecutor) {
        if (callbackExecutor != null) {
            return new ExecutorCallAdapterFactory(callbackExecutor);
        }
        return DefaultCallAdapterFactory.INSTANCE;
    }

    boolean isDefaultMethod(Method method) {
        return false;
    }

    @Nullable Object invokeDefaultMethod(Method method, Class<?> declaringClass, Object object,
                                         @Nullable Object... args) throws Throwable {
        throw new UnsupportedOperationException();
    }

    @IgnoreJRERequirement // Only classloaded and used on Java 8.
    static class Java8 extends Platform {
        //...
    }

    static class Android extends Platform {
        //...
    }
}

```

首先Platform的get方法通过[单例模式](https://rain9155.github.io/2019/11/16/%E5%8D%95%E4%BE%8B%E6%A8%A1%E5%BC%8F/)返回了一个Platform实例，单例模式就是**保证单例对象的类在同一进程中，只有一个实例存在**，Platform实例通过findPlatform方法创建，可以看到findPlatform方法里面区分了Android、java和其他平台返回了不同的Platform实现，由于我这里是Android平台，只关注Android平台的实现，Android类如下：

```java
//Platform.java
static class Android extends Platform {

    @Override public Executor defaultCallbackExecutor() {
        return new MainThreadExecutor();
    }

    @Override CallAdapter.Factory defaultCallAdapterFactory(@Nullable Executor callbackExecutor) {
        if (callbackExecutor == null) throw new AssertionError();
        return new ExecutorCallAdapterFactory(callbackExecutor);
    }

    //线程切换执行器
    static class MainThreadExecutor implements Executor {
        //构造Handler时通过Looper.getMainLooper()来构造
        //所以Handler的消息都会执行在主线程
        private final Handler handler = new Handler(Looper.getMainLooper());

        @Override public void execute(Runnable r) {
            handler.post(r);
        }
    }
}
```

Android继承自Platform，重写了它其中的两个方法：defaultCallbackExecutor方法和defaultCallAdapterFactory方法，defaultCallbackExecutor方法返回了一个MainThreadExecutor，它是一个[Executor](https://rain9155.github.io/2019/07/19/java%E7%BA%BF%E7%A8%8B%E6%B1%A0/)，它的execute方法的实现就是简单通过[Handler](https://blog.csdn.net/Rain_9155/article/details/86684083)把任务Runnable切换回主线程执行，就是说，线程池会把每一个线程提交的任务都切回主线程执行，我们再来看defaultCallAdapterFactory方法，这个方法返回了我们上面介绍过的ExecutorCallAdapterFactory，并把callbackExecutor传了进去，其实这个**callbackExecutor就是MainThreadExecutor**，待会在第3步构建Retrofit实例时就会讲到。

**这里我们来小结一下：**

new Retrofit.Builder()里面会通过Platform的get方法来获取一个Platform实例，在Android中，它会返回Android平台实例，这样就指定了Retrofit的运行平台是Android，然后就可以通过Android平台实例的**defaultCallbackExecutor方法**返回一个线程切换执行器MainThreadExecutor，它用于把任务切换回主线程执行，和通过**defaultCallAdapterFactory方法**返回一个网络请求适配器工厂ExecutorCallAdapterFactory，工厂中持有一个线程切换执行器实例，通过工厂就可以获取网络请求适配器实例。

有了Builder实例后，下面开始配置Builder.

### 2、配置Builder

```java
//Retrofit.java
public static final class Builder {
    
    private final Platform platform;//Retrofit运行的平台
    private @Nullable okhttp3.Call.Factory callFactory;//网络请求执行器工厂，用创建网络请求执行器实例，来自okhttp, 不是Retrofit中的那个Call
    private HttpUrl baseUrl;//网络请求的Url地址
    private final List<Converter.Factory> converterFactories = new ArrayList<>();//数据转化器工厂列表
    private final List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>();//网络请求适配器工厂列表
    private @Nullable Executor callbackExecutor;//线程切换执行器
    private boolean validateEagerly;//标志位，是否提前对serviceMethod进行缓存(在创建Api接口实例会讲到)
    
    //...
    
     //我们传入的String类型的Url，最终还是会解析成HttpUrl类型，它是Retrofit中url的代表
    public Builder baseUrl(String baseUrl) {
        checkNotNull(baseUrl, "baseUrl == null");
        HttpUrl httpUrl = HttpUrl.parse(baseUrl);
        if (httpUrl == null) {
            throw new IllegalArgumentException("Illegal URL: " + baseUrl);
        }
        return baseUrl(httpUrl);
    }

   
    public Builder baseUrl(HttpUrl baseUrl) {
        checkNotNull(baseUrl, "baseUrl == null");
        List<String> pathSegments = baseUrl.pathSegments();
        if (!"".equals(pathSegments.get(pathSegments.size() - 1))) {
            //baseUrl最后一定要接一个 /
            throw new IllegalArgumentException("baseUrl must end in /: " + baseUrl);
        }
        this.baseUrl = baseUrl;
        return this;
    }

    //把ConverterFactory添加到数据转化器工厂列表中
    public Builder addConverterFactory(Converter.Factory factory) {
        converterFactories.add(checkNotNull(factory, "factory == null"));
        return this;
    }

    //把CallAdapterFactory添加到网络请求适配器工厂列表中
    public Builder addCallAdapterFactory(CallAdapter.Factory factory) {
        callAdapterFactories.add(checkNotNull(factory, "factory == null"));
        return this;
    }
    
   //获取网络请求适配器工厂列表
    public List<CallAdapter.Factory> callAdapterFactories() {
      return this.callAdapterFactories;
    }

   //获取数据转化器工厂列表
    public List<Converter.Factory> converterFactories() {
      return this.converterFactories;
    }

    //修改validateEagerly标志位
    public Builder validateEagerly(boolean validateEagerly) {
        this.validateEagerly = validateEagerly;
        return this;
    }

    //...
}

```

Builder中所有的字段都贴出来了，方法只贴了几个常用的出来，我们调用Builder的相应方法，就是在为Builder的相应字段赋值，很简单，就不介绍了。

其中要注意的是：当我们添加ConverterFactory或CallAdapterFactory时，它们都是添加到各自的列表中，这说明在Retrofit中Converter和CallAdapter是可以存在多个的，为什么呢？这是因为Retrofit允许我们为Api接口里面定义的每一个方法都定义对应的Converter和CallAdapter，每当我们调用到Api接口的某个方法时，Retrofit都会遍历网络请求适配器工厂列表callAdapterFactories，把方法的返回值returnType和注解信息annotations传进每个Factory的get方法中，看某个Factory是否愿意处理这个方法，为这个方法创建对应CallAdapter实例；同理，当Retrofit解析某个Api接口方法的网络请求数据时，它同样会遍历数据转化器工厂列表converterFactories，把方法的相关信息传给Factory的responseBodyConverter或requestBodyConverter方法，看某个Factory是否愿意处理这个方法，为这个方法创建对应ResponseBodyBodyConverter或RequestBodyConverter实例，这两个过程在待会的源码分析都会体现到。由于我们平常开发都只添加了一个CallAdapter和一个Converter，所以Retrofit对Api接口定义的每一个方法的adapt和convert都是相同的处理。

配置好Builder后，接下来调用build方法创建Retrofit实例.

### 3、创建Retrofit实例

````java
//Retrofit.java
public static final class Builder {

    //...

    public Retrofit build() {
        if (baseUrl == null) {
            throw new IllegalStateException("Base URL required.");
        }

        //配置网络请求执行器工厂callFactory
        okhttp3.Call.Factory callFactory = this.callFactory;
        //如果没有指定，使用默认的
        if (callFactory == null) {
            //默认指定为OkHttpClient， OkHttpClient实现了Call.Factory接口，OkHttpClient和Call都是来自okhttp的
            callFactory = new OkHttpClient();
        }

        //配置线程切换执行器callbackExecutor
        Executor callbackExecutor = this.callbackExecutor;
        //如果没有指定，使用默认的
        if (callbackExecutor == null) {
            //默认指定为运行平台默认的线程切换执行器
            //在Android中，defaultCallbackExecutor方法的返回值就是MainThreadExecutor实例
            callbackExecutor = platform.defaultCallbackExecutor();
        }

        //配置网络请求适配器工厂列表
        //添加用户指定的网络请求适配器工厂列表
        List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
        //添加运行平台默认的网络请适配器工厂到列表中
        //在Android中，defaultCallAdapterFactory方法返回值就是ExecutorCallAdapterFactory实例，并把MainThreadExecutor实例传进方法，所以ExecutorCallAdapterFactory持有MainThreadExecutor实例（回去看构造Builder阶段）
        callAdapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

        //配置数据转化器工厂列表
        List<Converter.Factory> converterFactories =
            new ArrayList<>(1 + this.converterFactories.size());
	   //添加Retrofit默认的数据转化器工厂BuiltInConverters
        converterFactories.add(new BuiltInConverters());
        //添加用户指定的数据转化器工厂列表
        converterFactories.addAll(this.converterFactories);

        //创建一个Retrofit实例返回，并把上面的配置好的字段传进构造
        //这些字段和Retrofit中相应的字段同名,unmodifiableList就是创建一个不可修改的列表
        return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories),
                            unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
    }
}
````

build方法中把没有在配置Builder阶段中赋值的字段指定了默认值，然后把网络请求Url地址baseUrl、网络请求执行器工厂callFactory、线程切换执行器callbackExecutor、网络请求适配器工厂列表callAdapterFactories、数据转化器工厂列表converterFactories、标志位validateEagerly这六件套传进了Retrofit的构造，创建一个Retrofit实例返回。

其中要注意的是，按照添加顺序，工厂列表的优先级为：用户指定的网络请适配器工厂列表 > 运行平台默认的网络请求适配器工厂；Retrofit默认的数据转化器工厂 > 用户指定的数据转化器工厂列表。在遍历这些列表时是从前往后遍历的，越靠前的越先被访问。

### 4、小结

经过这3步后，指定了Retrofit的运行平台，配置好了Retrofit的网络请求Url地址、网络请求执行器工厂、线程切换执行器、网络请求适配器工厂列表、数据转化器工厂列表等字段，并创建了一个Retrofit实例.

有了Retrofit实例后，就可以通过Retrofit的create方法创建一个Api接口实例，这也是整个Retrofit最核心的地方.

## 四、创建Api接口实例

```java
GithubService service = Retrofit.create(GithubService.class);
```

我先讲一下Retrofit的create方法的作用，Retrofit的create方法里面干了两件事，首先它会根据validateEagerly标志位是否为true，而从决定是否把Api接口里定义的所有方法的注解和参数信息提前封装到ServiceMethod中然后缓存起来，接着就通过Proxy的newProxyInstance方法为GithubService接口创建一个代理对象返回，这里就使用到了[外观模式](https://www.jianshu.com/p/1b027d9fc005)，外观模式就是**定义一个统一接口，外部（用户）通过该统一的接口对子系统里的其他接口进行访问**，这里的这个统一的接口就是**Retrofit.create方法**，我们只需要调用create方法，Retrofit内部就替我们把这两件事给做了，不用我们访问Retrofit的内部子系统，降低了我们使用的复杂度。

 Retrofit的create方法如下：

```java
//Retrofit.java
//泛型T就是GithubService接口
public <T> T create(final Class<T> service) {
    //传进来的必须是一个接口，并且这个接口没有继承其他接口，否则抛异常
    Utils.validateServiceInterface(service);
    
    //1、判断validateEagerly标志位是否为true
    if (validateEagerly) {
      //如果为true，把Api接口里定义的所有方法的注解和参数信息封装到ServiceMethod中，提前缓存起来
      eagerlyValidateMethods(service);
    }
    
    //2、通过Proxy的newProxyInstance方法为GithubService接口创建一个代理对象
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
         		//...
          }
        });
  }

//Retrofit.java
private void eagerlyValidateMethods(Class<?> service) {
    //首先获取运行平台，这里为Android
    Platform platform = Platform.get();
    //getDeclaredMethods方法会以 Method[] 形式返回接口定义的所有方法的Method对象，所以这里就是在遍历接口的所有方法
    for (Method method : service.getDeclaredMethods()) {
      //在Android平台，isDefaultMethod方法返回false
      if (!platform.isDefaultMethod(method)) {
        //为每一个方法都调用一次loadServiceMethod方法
        //loadServiceMethod方法中就会把方法的注解和参数信息封装到ServiceMethod中，然后缓存起来
        loadServiceMethod(method);
      }
    }
  }
```

我们先看create方法的注释1，validateEagerly这个标志位决定是否对serviceMethod进行提前缓存，默认为false，当为true时，create方法中就会调用eagerlyValidateMethods方法，里面会遍历Api接口的所有方法，为每一个方法都调用一次loadServiceMethod方法（invoke方法中会讲到这个方法），loadServiceMethod方法中就会把方法的注解和参数信息封装到ServiceMethod中，以方法method为键，ServiceMethod为值，放入一个名叫**serviceMethodCache**的Map中缓存起来，这个Map定义在Retrofit中，如下：

```java
public final class Retrofit {
  private final Map<Method, ServiceMethod<?, ?>> serviceMethodCache = new ConcurrentHashMap<>();
  //...
}
```

可以看到这个Map是一个线程安全的HashMap，它的key = Method，value = ServiceMethod。所以可以根据需要设置validateEagerly标志位开启ServiceMethod的预加载，这样等到动态执行invoke方法时可以直接从缓存中获取ServiceMethod实例。

我们重点看create方法的注释2，通过Proxy的newProxyInstance方法为GithubService接口创建一个代理对象，这里就使用了[动态代理模式](https://juejin.im/post/5db2fbd0518825645a5ba18b)，Proxy的newProxyInstance方法会在**代码运行时**，根据第一个参数的ClassLoader，生成一个代理Class对象，该代理Class对象实现了传入的第二个参数对应的Interface列表，在获取到代理Class对象后，根据第三个参数InvocationHandler引用通过反射创建一个代理对象实例，所以**newProxyInstance最终的结果是生成一个代理对象实例，该代理对象会实现给定的接口列表，同时内部持有一个InvocationHandler引用，我们调用代理对象的方法时，这个方法处理逻辑就会委托给InvocationHandler实例的invoke方法执行**。

### 小结

当我们调用Retrofit的create方法后，它返回了一个代理对象实例，这个代理对象实现了create方法传进去的接口，接下来当我们调用Api接口的方法时，就是在调用代理对象的同名方法，这个方法处理逻辑就会委托给InvocationHandler实例的invoke方法执行，使用了动态代理之后的好处就是我们可以把发起网络请求的参数获取都集中到invoke方法中处理，而不需要为每一个接口定义一个实现类，这样降低了实现的难度。

接下来我们开始调用Api接口的方法，它会返回一个网络请求执行器。

## 五、调用Api接口实例的方法

```java
 Call<ResponseBody> call = service.getUserInfo("rain9155");
```

根据动态代理的知识，我们知道，当我们调用Api接口实例的方法，就是在调用代理对象的方法，这个方法处理逻辑就会委托给InvocationHandler实例的invoke方法执行，也就是说，当我们在外部调用Api接口的相应方法时，这些方法都会转到invoke方法执行，例如我调用service.getUserInfo("rain9155")时，这个方法会转到invoke方法去执行。

所以我们直接看InvocationHandler的invoke方法逻辑：

```java
//Retrofit.java
public <T> T create(final Class<T> service) {
     //...
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
            
          private final Platform platform = Platform.get();
            
          /**
           * 当我调用service.getUserInfo("rain9155")时，这个方法会转到invoke方法去执行。
           * @param proxy 代理对象实例
           * @param method 被调用的方法，如这里的getUserInfo方法
           * @param args 被调用的方法的参数，如这里的"rain9155"
           */
          @Override 
          public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
              
            //如果这个方法是Object类的，调用Object类的方法
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
             
            //在Android平台，isDefaultMethod方法返回false，不会进入这个分支
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
              
            //走到这里表示，调用的不是Object类的方法，而是GithubService接口的相应方法
              
            //1、调用loadServiceMethod方法，为这个方法获取一个ServiceMethod
            ServiceMethod<Object, Object> serviceMethod = (ServiceMethod<Object, Object>) loadServiceMethod(method);
            
            //2、把该方法的ServiceMethod和args传进okHttpCall，创建一个okHttpCall实例
            //args就是方法的参数
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            
            //3、调用adapt方法，传入okHttpCall，把okHttpCall适配成另一个Call
            //里面其实调用的是CallAdapter的adapt方法
            return serviceMethod.adapt(okHttpCall);
          }
        });
  }
```

invoke方法中分为注释1、2、3，下面分别讲解：

### 注释1、loadServiceMethod方法

```java
//1、调用loadServiceMethod方法，为这个方法获取一个ServiceMethod
ServiceMethod<Object, Object> serviceMethod = (ServiceMethod<Object, Object>) loadServiceMethod(method);
```

loadServiceMethod方法如下：

```java
//Retrofit.java 
ServiceMethod<?, ?> loadServiceMethod(Method method) {
    //首先从serviceMethodCache根据方法method获取ServiceMethod
    ServiceMethod<?, ?> result = serviceMethodCache.get(method);
     
    //如果获取得到就直接返回
    if (result != null) return result;

     //如果获取不到就为这个method创建一个ServiceMethod
    //上锁，所以创建的ServiceMethod是一个单例
    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
         //1、为method创建一个ServiceMethod
        result = new ServiceMethod.Builder<>(this, method).build();
         //然后以方法method为键，ServiceMethod为值，放入serviceMethodCache缓存起来
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```

可以看到这里首先从serviceMethodCache根据方法method获取ServiceMethod，如果获取不到再为方法创建一个ServiceMethod，然后以方法method为键，ServiceMethod为值，放入serviceMethodCache缓存起来，并返回，我们重点来看注释1，看这个**ServiceMethod是怎样根据方法的Method对象创建出来的**。

#### 1、ServiceMethod的创建过程

```java
result = new ServiceMethod.Builder<>(this, method)//1
      .build();//2
```

ServiceMethod同样是通过Builder模式创建，所以我们同样分注释的2步去看ServiceMethod的创建.

##### 1.1、构造Builder

所以我们先看ServiceMethod的Builder的构造，如下：

```java
//ServiceMethod.java
static final class Builder<T, R> {
    final Retrofit retrofit;
    final Method method;
    final Annotation[] methodAnnotations;
    final Annotation[][] parameterAnnotationsArray;
    final Type[] parameterTypes;
	//...

    Builder(Retrofit retrofit, Method method) {
      this.retrofit = retrofit;
      this.method = method;
      //获取方法里的所有注解，如@GET、@POST，@PATCH等
      this.methodAnnotations = method.getAnnotations();
      //获取方法里的所有参数类型，如int、String等
      this.parameterTypes = method.getGenericParameterTypes();
      //获取方法里所有参数的注解，如@Query，@Body、@PartMap等
      this.parameterAnnotationsArray = method.getParameterAnnotations();
    }
```

构造中依此获取方法的注解、参数类型、参数的注解并依此赋值给methodAnnotations、parameterAnnotationsArray、parameterTypes，注意parameterAnnotationsArray是一个二维数组，因为一个参数可以被多个注解修饰，一个注解可以修饰多个参数。

##### 1.2、创建ServiceMethod实例

我们接着看Builder的build方法，如下：

```java
//ServiceMethod.java
static final class Builder<T, R> {
    final Retrofit retrofit;
    final Method method;
    final Annotation[] methodAnnotations;
    final Annotation[][] parameterAnnotationsArray;
    final Type[] parameterTypes;

    Type responseType;
    ParameterHandler<?>[] parameterHandlers;
    Converter<ResponseBody, T> responseConverter;
    CallAdapter<T, R> callAdapter;
    //...

    public ServiceMethod build() {
        
        //1、调用createCallAdapter方法创建网络请求适配器CallAdapter
        callAdapter = createCallAdapter();

        //通过CallAdapter的responseType方法获取返回数据的响应body的类型
        //如Call<ResponseBody>，获取到ResponseBody类型
        responseType = callAdapter.responseType();
        //....省略异常处理

        //2、调用createResponseConverter方法创建网络返回数据的数据转化器Converter
        responseConverter = createResponseConverter();

        //遍历该方法的所有注解
        for (Annotation annotation : methodAnnotations) {
            //3、调用parseMethodAnnotation方法解析注解，如@GET、@POST，@PATCH等
            parseMethodAnnotation(annotation);
        }
        //...省略异常处理

        int parameterCount = parameterAnnotationsArray.length;
        parameterHandlers = new ParameterHandler<?>[parameterCount];
        //为方法中的每个参数创建一个ParameterHandler<?>对象并解析每个参数使用的注解
        for (int p = 0; p < parameterCount; p++) {
            
            //获取参数类型，如int、String等
            Type parameterType = parameterTypes[p];
             //...省略异常处理

            //获取修饰参数的注解，如@Query，@Body、@PartMap等
            Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
            //...省略异常处理

            //4、调用parseParameter方法解析每个参数使用的注解，并返回一个parameterHandlers
            parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
        }
        //...省略异常处理

  		//创建ServiceMethod实例，并把Builder实例传进去
        //ServiceMethod构造里面会把Builder配置好的字段赋值给ServiceMethod中相应的字段
        return new ServiceMethod<>(this);
    }

```

build方法里面分4步，我们先看注释1、2，注释1调用createCallAdapter方法创建网络请求适配器CallAdapter，并赋值给callAdapter字段，createCallAdapter方法如下：

```java
//ServiceMethod::Builder.java
private CallAdapter<T, R> createCallAdapter() {
    //获取方法的返回值类型，如Call<ResponseBody>
    Type returnType = method.getGenericReturnType();
    //...省略异常处理
    
    //获取方法的所有注解，如@GET、@POST，@PATCH等
    Annotation[] annotations = method.getAnnotations();
    try {
        //调用Retrofit的callAdapter方法，返回一个CallAdapter实例
        return (CallAdapter<T, R>) retrofit.callAdapter(returnType, annotations);
    }
    //...省略异常处理
}

//Retrofit.java
 public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
  }

public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
    //...省略异常处理

    int start = callAdapterFactories.indexOf(skipPast) + 1;
    //遍历网络请求适配器工厂callAdapterFactories
    for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
      //把该方法的返回值类型returnType和注解信息annotations传给CallAdapterFactory的get方法，通过get方法获取一个CallAdapter实例或null
      //在Retrofit相关类介绍讲过，Android平台的CallAdapterFactor的默认实现是ExecutorCallAdapterFactory，ExecutorCallAdapterFactory的get方法返回一个CallAdapter匿名实现类或null
      CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
```

createCallAdapter方法中，获取到方法的返回值类型（如Call<ResponseBody>类型）和注解信息后，就调用Retrofit的callAdapter方法，callAdapter方法就调用nextCallAdapter方法，在nextCallAdapter方法中，会遍历Retrofit的网络请求适配器工厂callAdapterFactories，把方法的返回值returnType和注解信息annotations传进每个Factory的get方法中，看某个Factory是否愿意处理这个方法，如果愿意，就为这个方法创建对应CallAdapter实例，在Android平台中，CallAdapterFactor的默认实现是ExecutorCallAdapterFactory，ExecutorCallAdapterFactory的get方法返回一个CallAdapter匿名实现类。**（查看前面讲过的Retrofit相关类介绍，里面有对ExecutorCallAdapterFactory的详细介绍）**

我们再看build方法里的注释2，注释2调用createResponseConverter方法创建网络返回数据的数据转化器Converter，赋值给responseConverter字段，createResponseConverter方法如下：

```java
//ServiceMethod::Builder.java
private Converter<ResponseBody, T> createResponseConverter() {
    //获取方法的所有注解，如@GET、@POST，@PATCH等
    Annotation[] annotations = method.getAnnotations();
    try {
        //调用Retrofit的responseBodyConverter方法，返回一个Converter实例
        return retrofit.responseBodyConverter(responseType, annotations);
    } 
    //...省略异常处理
}

//Retrofit.java
public <T> Converter<ResponseBody, T> responseBodyConverter(Type type, Annotation[] annotations) {
    return nextResponseBodyConverter(null, type, annotations);
}

public <T> Converter<ResponseBody, T> nextResponseBodyConverter(
    @Nullable Converter.Factory skipPast, Type type, Annotation[] annotations) {
    //...省略异常处理

    int start = converterFactories.indexOf(skipPast) + 1;
    //遍历数据转化器工厂converterFactories
    for (int i = start, count = converterFactories.size(); i < count; i++) {
        //把该方法的返回值的泛型类型和注解信息annotations传给ConverterFactory的responseBodyConverter方法，通过responseBodyConverter方法获取一个处理网络返回数据的数据转化器Converter实例或null
        //在Retrofit相关类介绍讲过，Retrofit的ConverterFactory的默认实现是BuiltInConverters，BuiltInConverters的responseBodyConverter方法返回一个处理网络返回数据的数据转化器Converter实例或null
        Converter<ResponseBody, ?> converter =
            converterFactories.get(i).responseBodyConverter(type, annotations, this);
        if (converter != null) {
            //noinspection unchecked
            return (Converter<ResponseBody, T>) converter;
        }
    }
    //...省略异常处理
}

```

createResponseConverter方法中的流程和createCallAdapter方法的流程差不多，最终是遍历Retrofit的数据转化器工厂converterFactories，把该方法的返回值的泛型类型（如Call<ResponseBody>，泛型类型就是ResponseBody类型）和注解信息annotations传给每个Factory的responseBodyConverter方法中，看某个Factory是否愿意处理这个方法，如果愿意，就为这个方法创建对应的网络返回数据的数据转化器Converter实例，在Android平台中ConverterFactory的默认实现是BuiltInConverters**（查看前面讲过的Retrofit相关类介绍，里面有对BuiltInConverters的详细介绍）**。有ResponseConverter就有相应的RequestConverter，RequestConverter用来处理网络请求数据的数据转化，RequestConverter并不在ServiceMethod的build方法中创建，而是在parseParameter方法中创建。

至于build方法的注释3、4的parseMethodAnnotation方法和parseParameter方法，限于篇幅，就留给大家自己探索了，我讲一下里面的大体逻辑：

* **parseMethodAnnotation方法**：该方法里面首先判断是哪种HTTP请求的注解，然后为不同的HTTP请求注解调用parseHttpMethodAndPath方法，该方法里面会获取注解中省略域名的Url和把Url里面需要替换地方用正则找出来放到一个Set中，如：@GET("users/{user}")，省略域名的Url = users/{user}，需要替换的地方是{user}，把user找出来，最终parseMethodAnnotation方法获取到的信息是： **HTTP请求方式、省略域名的Url和Url中需要替换值的地方**。
* **parseParameter方法**：这个方法是解析方法参数的注解，如@Quary、@Path等，里面会判断是哪种类型的注解，根据不同类型的注解，**取出注解中的值，创建一个RequestConverter实例，最后把注解的值和RequestConverter实例传进ParameterHandler构造，为这个注解创建一个ParameterHandler实例返回**，不同的注解有不同的ParameterHandler类型，如@Quary就有ParameterHandler.Quary，@Path就有ParameterHandler.Path。

都是通过注解解析出注解参数，然后一并封装到ServiceMethod中去，build方法执行完毕，就创建了一个ServiceMethod实例返回。

**我们来小结一下ServiceMethod的创建过程：**

经过这2步后，为这个方法method创建了一个对应的ServiceMethod实例，这个ServiceMethod封装了网络请求所需要的所有参数和持有CallAdapter实例、处理网络返回数据转化的Converter实例，同时这个ServiceMethod是一个单例，也就是说**Api接口里的每一个方法都分别对应着一个ServiceMethod实例，这个ServiceMethod实例持有着网络请求所需要的所有参数**。

我们继续回到loadServiceMethod方法中。

#### 2、小结

调用loadServiceMethod方法后，创建了一个ServiceMethod实例并缓存到一个Map中，第二次使用时直接从缓存获取ServiceMethod实例，不必重复创建，提高效率。

我们回到invoke方法，有了ServiceMethod实例后，就可以创建一个okHttpCall实例.

### 注释2、创建一个okHttpCall实例

```java
//2、把该方法的ServiceMethod和args(args就是方法的参数)传进okHttpCall，创建一个okHttpCall实例
OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
```

OkHttpCall的构造如下：

```java
final class OkHttpCall<T> implements Call<T> {
    private final ServiceMethod<T, ?> serviceMethod;//封装了Api接口方法中的注解和参数信息，一个ServiceMethod对应一个Method
    private final @Nullable Object[] args;//代表着Api接口方法中的参数
    private @Nullable okhttp3.Call rawCall;//这是一个来自Okhttp的Call
	//...

  OkHttpCall(ServiceMethod<T, ?> serviceMethod, @Nullable Object[] args) {
    this.serviceMethod = serviceMethod;
    this.args = args;
  }
    
 //...
```

可以看到Okhttp把刚刚得到的ServiceMethod实例和接口方法的参数保存起来，OkHttpCall在前面的Retrofit的相关类介绍已经简单介绍过了，它就是实现了Call的一个类，它里面的方法大部分逻辑都转发给Okhttp的Call 。

得到okHttpCall实例后，通过adapt方法把它适配成另外一个Call.

### 注释3、把OkHttpCall适配成另一个Call

```java
 //3、调用adapt方法，传入okHttpCall，把okHttpCall适配成另一个Call
return serviceMethod.adapt(okHttpCall);
```

ServiceMethod的adapt方法如下：

```java
//ServiceMethod.java
T adapt(Call<R> call) {
    return callAdapter.adapt(call);
}
```

adapt方法传进来的call是OkhttpCall实例，callAdapter就是刚刚用createCallAdapter方法创建的CallAdapter实例，在Android平台中，这个CallAdapter实例的adapt方法的默认实现就是用ExecutorCallbackCall包装传进来的OkHttpCall，并返回ExecutorCallbackCall实例，如下：

```java
final class ExecutorCallAdapterFactory extends CallAdapter.Factory {

    //线程切换执行器, 在创建Retrofit实例时传进ExecutorCallAdapterFactory中
    //在Android平台，就是MainThreadExecutor实例
    final Executor callbackExecutor;

    ExecutorCallAdapterFactory(Executor callbackExecutor) {
        this.callbackExecutor = callbackExecutor;
    }

    //get方法
    @Override
    public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit Retrofit) {
        //...
        final Type responseType = Utils.getCallResponseType(returnType);
        //返回一个CallAdapter匿名类
        return new CallAdapter<Object, Call<?>>() {

            @Override 
            public Type responseType() {
                return responseType;
            }

             //adapt方法
            @Override 
            public Call<Object> adapt(Call<Object> call) {
                //创建了一个ExecutorCallbackCall，并把线程切换执行器实例和OkhttpCall实例传进构造
                return new ExecutorCallbackCall<>(callbackExecutor, call);
            }
        };
    }

    //网络请求执行器
    static final class ExecutorCallbackCall<T> implements Call<T> {
		//...
    }
}
```

所以callAdapter.adapt(call)就是**把OkhttpCall 适配成 ExecutorCallbackCall**，我们来看一下ExecutorCallbackCall，它是ExecutorCallAdapterFactory的内部类，如下：

```java
//ExecutorCallAdapterFactory.java
//网络请求执行器
static final class ExecutorCallbackCall<T> implements Call<T> {
    
    final Executor callbackExecutor;//线程切换执行器实例, 在get方法通过构造传进
    final Call<T> delegate;//OkhttpCall实例,  在get方法通过构造传进

    ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
        this.callbackExecutor = callbackExecutor;
        this.delegate = delegate;
    }

    @Override 
    public void enqueue(final Callback<T> callback) {
        checkNotNull(callback, "callback == null");

        //ExecutorCallbackCall的enqueue方法委托给OkhttpCall执行
        delegate.enqueue(new Callback<T>() {
            @Override 
            public void onResponse(Call<T> call, final Response<T> response) {
                //当网络数据返回时，通过线程切换执行器callbackExecutor，切换到主线程执行callback回调
                callbackExecutor.execute(new Runnable() {
                    @Override public void run() {
                        if (delegate.isCanceled()) {
                            callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
                        } else {
                            callback.onResponse(ExecutorCallbackCall.this, response);
                        }
                    }
                });
            }

            @Override 
            public void onFailure(Call<T> call, final Throwable t) {
                //当网络数据返回时，通过线程切换执行器callbackExecutor，切换到主线程执行callback回调
                callbackExecutor.execute(new Runnable() {
                    @Override public void run() {
                        callback.onFailure(ExecutorCallbackCall.this, t);
                    }
                });
                
            }
        });
    }

    @Override 
    public Response<T> execute() throws IOException {
        //ExecutorCallbackCall的execute方法委托给OkhttpCall执行
        return delegate.execute();
    }

    //...

}
```

可以看到，ExecutorCallbackCall和OkhttpCall一样都实现了Call接口中的方法，并且ExecutorCallbackCall中持有OkhttpCall的实例，它的enqueue方法和execute方法都委托给OkhttpCall的enqueue方法和execute方法执行，当网络数据返回时，通过线程切换执行器callbackExecutor，切换到主线程执行callback回调，这里采用了[装饰者模式](https://juejin.im/post/5db2fbd0518825645a5ba18b)，采用装饰者模式的好处就是**不用通过继承就可以扩展一个对象的功能，动态的给一个对象添加一些额外的操作**，这里额外的操作就是通过线程切换执行器切换回主线程后再执行callback回调，因为OkHttpCall的enqueue方法是进行网络的异步请求，当异步请求结果返回时，回调是在子线程中，需要通过线程切换执行器转换到主线程中再进行callback回调。

最终，ServiceMethod的adapt方法把**OkhttpCall 适配成不同平台的网络请求执行器，并返回**，在Android平台中，OkhttpCall 适配成了 ExecutorCallbackCall实例。

### 4、小结

当我们调用Api接口实例中的方法时，这些方法的处理逻辑都会转发给invoke方法，在invoke方法中，会为每一个方法创建一个ServiceMethod，这个ServiceMethod封装了网络请求所需要的所有参数和持有CallAdapter实例、处理网络返回数据转化的Converter实例，接着就会把这个ServiceMethod实例和接口方法的参数传进OkhttpCall的构造中，创建一个OkhttpCall实例，接着把这个OkhttpCall实例传给ServiceMethod的adapt方法，ServiceMethod的adapt方法就会调用CallAdapter实例的adapt方法，把OkhttpCall适配成不同平台的网络请求执行器，所以如果你添加了其他平台的CallAdapterFactory，你就可以得到不同平台下的网络请求执行器，在Android平台中，CallAdapter实例的adapt方法会把OkhttpCall适配成ExecutorCallbackCall，这个ExecutorCallbackCall持有OkhttpCall实例和线程切换器MainThreadExecutor实例，当我们发起网络请求时，ExecutorCallbackCall就会委托OkhttpCall发起网络请求，当网络请求数据返回时，ExecutorCallbackCall就会通过MainThreadExecutor把线程切换主线程执行回调。

得到网络请求执行器之后，就可以发起，网络请求了.

## 六、发起网络请求（以异步为例）

不同平台下的网络请求执行器不同，在Android平台，调用Api接口实例的方法后返回的是ExecutorCallbackCall，有了ExecutorCallbackCall后，我们就可以发起同步或异步请求，同步和异步请求的流程类型，唯一不同的是异步请求需要把结果通过Callback回调给上层，而同步请求则是直接return结果给上层。

下面以异步请求为例：

```java
//用户发起异步请求 
call.enqueue(new Callback<ResponseBody>() {
            @Override
            public void onResponse(Call<ResponseBody> call, Response<ResponseBody> response){
            	//通过Response获取网络请求返回结果
                ResponseBody body = response.body();
                try {
                    Log.d(TAG, "请求结果：" +  body.string());
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }

            @Override
            public void onFailure(Call<ResponseBody> call, Throwable t) {

            }
        });
```

调用的是ExecutorCallbackCall的enqueue方法，如下：

```java
//ExecutorCallAdapterFactory::ExecutorCallbackCall.java
public void enqueue(final Callback<T> callback) {
    checkNotNull(callback, "callback == null");

    //ExecutorCallbackCall的enqueue方法委托给OkhttpCall执行
    delegate.enqueue(new Callback<T>() {
        @Override 
        public void onResponse(Call<T> call, final Response<T> response) {
            //当网络数据返回时，通过线程切换执行器callbackExecutor，切换到主线程执行callback回调
            callbackExecutor.execute(new Runnable() {
                @Override public void run() {
                    if (delegate.isCanceled()) {
                        callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
                    } else {
                        callback.onResponse(ExecutorCallbackCall.this, response);
                    }
                }
            });
        }

        @Override 
        public void onFailure(Call<T> call, final Throwable t) {
            //当网络数据返回时，通过线程切换执行器callbackExecutor，切换到主线程执行callback回调
            callbackExecutor.execute(new Runnable() {
                @Override public void run() {
                    callback.onFailure(ExecutorCallbackCall.this, t);
                }
            });

        }
    });
```

调用的是OkhttpCall的enqueue方法，如下：

```java
//OkhttpCall.java
private @Nullable okhttp3.Call rawCall;//这是一个来自Okhttp的Call

public void enqueue(final Callback<T> callback) {
    checkNotNull(callback, "callback == null");

    //来自Okhttp的Call
    okhttp3.Call call;
    Throwable failure;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      //1、获取Okhttp的Call实例
      call = rawCall;
      failure = creationFailure;
      if (call == null && failure == null) {
        try {
          //第一次发起网络请求，创建Okhttp的Call
          call = rawCall = createRawCall();
        } catch (Throwable t) {
          throwIfFatal(t);
          failure = creationFailure = t;
        }
      }
    }

    if (failure != null) {
       //通过callback把错误回调出去
      callback.onFailure(this, failure);
      return;
    }

    if (canceled) {
      call.cancel();
    }

    //2、调用Okhttp的Call的enqueue方法发起异步请求
    //传入的是Okhttp的Callback
    call.enqueue(new okhttp3.Callback() {
        
       //Okhttp的结果回调
      @Override 
       public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
        Response<T> response;
        try {
          //3、调用parseResponse方法把Okhttp的Response解析成Retrofit的Response
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          callFailure(e);
          return;
        }

        try {
            //通过callback把结果回调出去
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

     //Okhttp的错误回调
      @Override
      public void onFailure(okhttp3.Call call, IOException e) {
        callFailure(e);
      }

      private void callFailure(Throwable e) {
        try {
           //通过callback把错误回调出去
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }
    });
  }
```

OkhttpCall的enqueue方法大体分为3步：

1、获取Okhttp的Call实例，赋值给临时变量call：

​	1.1、如果是第一次发起网络请求，就调用createRawCall方法就创建来自Okhttp的Call，赋值给临时变量call和成员变量rawCall;

​	1.2、如果不是第一次发起网络请求，就把上次的rawCall实例赋值给临时变量call.

2、调用Okhttp的Call的enqueue方法发起异步请求（更多细节请查看上一篇文章[okhttp3源码分析之请求流程](https://rain9155.github.io/2019/09/03/okhttp3源码分析之请求流程/)）；

3、当网络请求结果正确返回时，用parseResponse方法把Okhttp的Response解析成Retrofit的Response.

所以我们来看一下两个关键的办法，createRawCall方法和parseResponse方法：

### 1、创建Okhttp的Call 

createRawCall方法如下：

```java
//OkhttpCall.java
private okhttp3.Call createRawCall() throws IOException {
    //调用serviceMethod的toCall方法，并把args即接口方法参数传了进去
    //args在创建OkhttpCall实例时通过构造传进来的
    okhttp3.Call call = serviceMethod.toCall(args);
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }
```

createRawCall方法调用ServiceMethod的toCall方法，如下：

```java
//ServiceMethod.java
okhttp3.Call toCall(@Nullable Object... args) throws IOException {
    
    //1、创建一个用于构造Request的Builder，并把ServiceMethod中封装的参数传进去
    RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl, headers,
                                                       contentType, hasBody, isFormEncoded, isMultipart);
    
    ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;

    int argumentCount = args != null ? args.length : 0;
      //...省略异常处理

    //2、遍历每个方法参数的ParameterHandler
    for (int p = 0; p < argumentCount; p++) {
        //调用ParameterHandler的apply方法，把ParameterHandler中的参数apply到requestBuilder中
        handlers[p].apply(requestBuilder, args[p]);
    }

    //3、requestBuilder.build方法创建一个Request实例传进newCall方法，这个Request是来自Okhttp的
    //然后调用callFactory的newCall方法创建一个Call
    //这个callFactory就是OkHttpClient时，在创建Retorfit时配置
    //所以这里返回的是来自Okhttp的Call
    return callFactory.newCall(requestBuilder.build());
}
```

首先创建一个用于构造Request的requestBuilder，并把ServiceMethod中封装的参数传进去，然后遍历每个方法参数的ParameterHandler，通过apply方法取出ParameterHandler中的参数放入requestBuilder中，ParameterHandler前面讲过，它里面保存了每个方法参数的注解的值和处理这些值的Converter实例，构造Request的参数都有了，接着就通过requestBuilder.build方法创建一个Okhttp的Request实例并传进newCall方法，最后通过OkHttpClient的newCall方法创建一个Okhttp的Call实例返回。

最终createRawCall方法返回一个来自Okhttp的Call实例。

### 2、解析返回结果

parseResponse方法如下：

```java
//ServiceMethod.java
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    //取出Okhttp的Response的body
    ResponseBody rawBody = rawResponse.body();

    //...省略的是一些状态码处理

    //用ExceptionCatchingRequestBody包装一下rawBody
    ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
    try {
        //调用ServiceMethod的toResponse方法把原始的body转化成我们需要的数据类型
        T body = serviceMethod.toResponse(catchingBody);
        //
        return Response.success(body, rawResponse);
    } 
    //...省略异常处理
}
```

首先把Okhttp的Response的body取处理，然后用ExceptionCatchingRequestBody包装一下，这个ExceptionCatchingRequestBody是继承自Okhttp的ResponseBody，接着把这个ResponseBody传进ServiceMethod的toResponse方法，里面会使用ServiceMethod保存的处理网络返回数据的Converter实例来把这个ResponseBody转化成我们需要的数据类型，ServiceMethod的toResponse方法如下：

```java
//ServiceMethod.java
R toResponse(ResponseBody body) throws IOException {
    return responseConverter.convert(body);
  }
```

这个responseConverter就是在创建ServiceMethod时用createResponseConverter方法创建的Converter实例，把body转化成R类型，不同的数据转化有不同的实现，在Retrofit的默认实现中，它就是直接返回ResponseBody。

我们再回到parseResponse方法，调用完ServiceMethod的toResponse方法，得到了转化后的body，最后调用了Response的success方法，把Okhttp的Response和转化后的body传了进去，最终返回一个Retrofit的Response。

### 3、小结

以异步为例，Retrofit通过ExecutorCallbackCall的enqueue方法发起网络请求，最终会通过OkhttpCall的enqueue方法来发起网络请求，OkhttpCall的enqueue方法中，首先会调用创建一个来自Okhttp的Call实例，然后通过这个Okhttp的Call实例的enqueue方法来发起异步请求，当网络结果Okhttp的Response返回时，调用parseResponse方法解析Response，parseResponse方法里面还会调用ServiceMethod的toResponse方法通过Converter实例的convert方法把ResponseBody转化成我们想要的数据，不同的数据转化有不同的实现，在Retrofit的默认实现中，它就是直接返回Okhttp的ResponseBody，最后把这个转化后的body和原始的Okhttp的Response一并封装成Retrofit的Response返回，最后把parseResponse方法返回的Response通过callback回调出去，这时ExecutorCallbackCall收到回调，通过线程切换执行器callbackExecutor，切换到主线程执行callback回调，一次异步请求就完成了，同步请求也是大同小异，只是少了个回调，就留给大家自己分析了。

## 总结

能够阅读到这里，说明你对Retrofit的理解又更上一层楼了，其实从整体去看Retrofit，它的流程并不复杂，它的使用也非常的简单，这得益于它优秀的架构，和运用得当的设计模式，其中最核心的当属动态代理模式，通过一个代理类InvocationHandler代理N多个接口，它把每一个方法的处理逻辑都集中到了invoke方法中，这样就能在同一处地方处理所有方法的注解解析，还有它那面向接口的设计，使得各个子模块之间降低耦合，让我们以最小的代价替换成我们需要的实现，Retrofit的整体流程图如下：

{% asset_img retrofit3.png Retrofit %}

一句话概括Retrofit：它是一个通过动态代理把Api接口方法的注解解析成网络请求所需参数，最后通过Okhttp执行网络请求的封装库。

以上就是本文的所有内容，如有错误，欢迎指出。

参考资料：

[Retrofit解析之面向接口编程](https://www.jianshu.com/p/3855eee03793)





​	

















