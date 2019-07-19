---
title: java线程池
tags: 
- 线程池
- java
categories: java
date: 2019-07-19 12:26:00
---


## 前言

* 上一篇文章：[java线程](https://rain9155.github.io/2019/07/19/java%E7%BA%BF%E7%A8%8B/)

当我们需要频繁的创建多个线程时，每次都通过new一个Thread是一种不好的操作，创建一个线程是要消耗资源，频繁的创建会导致性能较差，而且我们还要管理多个线程的状态，管理不好还可能会出现死锁，浪费资源。这时就需要java提供的线程池，它能够有效的管理、调度线程，避免过多资源的消耗，通过线程池的统一调度、管理，使得多线程开发变得更简单。本文讲解一下有关线程池的知识点。

<!--more-->

## Executor、ExecutorService、Executors、ThreadPoolExecutor之间的关系

Executor是一个接口，里面只有一个方法execute(Runnable command)，用来提交任务到线程池执行。ExecutorService继承Executor，同样是一个接口，里面提供了更多的方法用于操作线程池，如Future<?> submit(Runnable task)可以提交有返回值的任务到线程池执行，shutdown()用来关闭线程池。ThreadPoolExecutor是真正的线程池的实现，它实现了上面接口的方法，还提供了一系列参数来配置线程池。Executors是一个工厂类，通过它提供的工厂方法可以创建不同的线程池。

下面一张图说明的它们之间的关系。

{% asset_img thread1.jpg thread1 %}

## ThreadPoolExecutor

ThreadPoolExecutor是线程池的真正实现，它的构造方法提供了一系列的参数来配置线程池，如下：

```java
 public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler)
```

下面对这几个参数进行说明：

### 1、int  corePoolSize

线程池中的核心线程数。

线程池启动后默认是空的，只有任务到来时才会创建线程以处理请求，ThreadPoolExecutor的prestartAllCoreThreads()方法可以在线程池启动后立即创建所有的核心线程以等待任务。

还有在默认情况下，核心线程一旦创建后就会在线程池中一直存活，即使它们处于空闲状态，如果设置ThreadPoolExecutor的allowCoreThreadTimeOut(boolean value)方法为true，那么空闲的核心线程在等待新任务到来时就会有超时策略，这个超时时间由keepAliveTime指定，当等待时间超过keepAliveTime后，核心线程就会被终止。

### 2、int  maximumPoolSize

线程池所能创建的最大线程数，它与**corePoolSize**、**workQueue**共同调整线程池中实际运行的线程数量。

当线程池中的工作线程数小于corePoolSize时，每次来任务的时候都会创建一个新的工作线程。不管工作线程集合中有没有线程是处于空闲状态。

当池中工作线程数大于等于 corePoolSize 的时候，每次任务来的时候都会首先尝试将线程放入队列，而不是直接去创建线程。

如果放入队列失败，说明队列满了，且当线程中线程数小于 maximumPoolSize 的时候，则会创建一个工作线程（非核心线程）来执行这个任务，如果线程池中的线程数大于maximumPoolSize，调用给定的拒绝策略。

如果任务成功放入队列，则看看是否需要开启新的线程来执行任务，只有当当前工作线程数为0的时候才会创建新的线程，因为之前的线程有可能因为都处于空闲状态或因为工作结束等待超时而被移除，否则就从队列中一个个取出任务给空闲的线程执行。

如图，线程池的工作流程如下:

{% asset_img thread3.png thread %}

### 3、 long  keepAliveTime

非核心线程空闲时的超时时长，超过这个时长，非核心线程就会被回收。

这是一种减少不必要资源消耗的策略，这个参数可以在运行时被改变，我们同样可以将这种策略应用给核心线程，我们可以通过调用 allowCoreThreadTimeout 来实现。

### 4、TimeUnit  unit

指定keepAliveTime的单位，可选值有毫秒、秒、分等。

### 5、 BlockingQueue   workQueue

线程池中的任务队列，用来保存等待执行任务的阻塞队列。

首先 BlockingQueue 是一个接口，这是一个很特殊的队列，如果 BlockQueue 是空的，从 BlockingQueue 取东西的操作将会被阻断进入等待状态，直到 BlockingQueue 进了东西才会被唤醒。同样，如果 BlockingQueue 是满的，任何试图往里存东西的操作也会被阻断进入等待状态，直到 BlockingQueue 里有空间才会被唤醒继续操作。

BlockingQueue 大致有四个实现类，如下：

* ArrayBlockingQueue：规定大小的基于数据结构的 BlockingQueue，即有界队列，其构造函数必须带一个 int 参数来指明其大小。其所含的对象是以 FIFO(先入先出)顺序排序的。如果队列满了调用给定的拒绝策略。
* LinkedBlockingQueue： 大小不定的基于链表结构的 BlockingQueue，既可以有界也可以无界，若其构造函数带一个规定大小的参数，生成的 BlockingQueue 有大小限制，若不带大小参数，所生成的 BlockingQueue 的大小由 Integer.MAX_VALUE 来决定。其所含的对象是以 FIFO(先入先出)顺序排序的。所以如果该队列是无界的，则可以忽略给定的拒绝策略，因为它永远都不会满，同时还可以忽略maximumPoolSize 参数，因为起当核心线程都在忙的时候，新的任务被放在队列上，永远不会有大于 corePoolSize 的线程被创建。
* PriorityBlockingQueue：类似于 LinkedBlockQueue，但其所含对象的排序不是 FIFO，而是依据对象的自然排序顺序或者是构造函数的 Comparator 决定的顺序。
* SynchronousQueue：特殊的 BlockingQueue，对其的操作必须是放和取交替完成的。因为其特殊的操作，所以如果有一个任务要插入队列，那么它必须要等到另一个移除任务的操作。所以使用该队列会直接把任务提交给线程池，而不会将任务加入队列，如果线程池没有任何可用的线程处理，就调用给定的拒绝策略。

BlockingQueue 的常用方法：

- add(anObject)：把 anObject 加到 BlockingQueue 里，即如果 BlockingQueue 可以容纳，则返回 true，否则报异常。
- offer(anObject)：表示如果可能的话，将 anObject 加到 BlockingQueue 里，即如果 BlockingQueue 可以容纳，则返回 true，否则返回 false。
- put(anObject)：把 anObject 加到 BlockingQueue 里，如果 BlockQueue 没有空间，则调用此方法的线程被阻断直到 BlockingQueue 里面有空间再继续。
- take()：取走 BlockingQueue 里排在首位的对象，若 BlockingQueue 为空，阻断进入等待状态直到 Blocking 有新的对象被加入为止。
- poll(time)：取走 BlockingQueue 里排在首位的对象，若不能立即取出，则可以等 time 参数规定的时间，取不到时返回 null。

### 6、ThreadFactory  threadFactory

线程工厂，让用户可以定制创建线程的过程。

ThreadFactory  是一个接口，它只有一个Thread newThread(Runnable r)方法，如果没有指定threadFactory，默认的 Executors的defaultThreadFactory 将被使用，这个时候创建的线程将都属于同一个线程组，拥有同样的优先级和 daemon 状态。

我们可以扩展配置 ThreadFactory，我们可以配置线程的名字、线程组合 、daemon 状态。如果调用 ThreadFactory的newThread失败，将返回 null，executor 将不会执行任何任务。

### 7、RejectedExecutionHandler handler

当新任务到来时，线程池被关闭或线程数和队列已经达到上限的时候，对新任务采取的处理策略。

RejectedExecutionHandler 同样是一个接口，里面只有一个rejectedExecution(Runnable r, ThreadPoolExecutor executor)方法，下面介绍一下几个默认的实现，都定义在ThreadPoolExecutor中：

* AbortPolicy：直接抛出 RejectedExecutionException 异常。线程池的默认实现。
* CallerRunsPolicy：这个策略将会使用 Caller 线程来执行这个任务，这是一种 feedback 策略，可以降低任务提交的速度。
* DiscardPolicy：这个策略将会直接丢弃任务。
* DiscardOldestPolicy：这个策略将会把任务队列头部的任务丢弃，然后重新尝试执行，如果还是失败则继续实施策略。这样的结果是最后加入的任务反而更有可能被执行。

## 线程池的生命周期

线程池的生命周期包含3种状态，如下：

### 1、运行

线程池创建后就进入运行状态，这个时候可以向线程池提交任务，可以通过ThreadPoolExecutor的execute()和submit()方法。

execute方法用于提交不需要返回值的任务，所以无法判断任务是否被线程池执行成功。

而submit方法用于提交需要返回值的任务。线程池会返回一个future类型的对象，通过这个future对象可以判断任务是否执行成功，并且可以通过future的get方法来获取返回值，get方法会阻塞当前线程直到任务完成，而使用get（long timeout，TimeUnit unit）方法则会阻塞当前线程timeout时间后立即返回，这时候有可能任务没有执行完，当线程池的任务还没有执行完时，会报超时异常。

### 2、关闭

当调用ThreadPoolExecutor的shutdown或shutdownNow方法后，便会进入关闭状态，这时意味线程池不再接受新的任务。这时isShutdown方法返回true。

调用shutdown方法会等待线程执行完毕后再关闭线程池，但是如果调用的是shutdownNow方法，则相当于调用每个线程的interrupt方法。

如果只想中断 Executor 中的一个线程，可以通过使用 submit() 方法来提交一个线程，它会返回一个 Future<?> 对象，通过调用该对象的 cancel(true) 方法就可以中断线程。

### 3、终止

在关闭状态的线程池执行完所有已经提交的任务后，就变为终止状态，这时调用isTerminated方法会返回true。

## 线程池的分类

通过配置ThreadPoolExecutor的构造函数的参数就可以实现不同类形的线程池，它们分别是：FixedThreadPool、CachedThreadPool、ScheduleThreadPool和SingleThreadExecutor。

### 1、FixedThreadPool

顾名思义，就是一种固定线程数量的线程池。前面讲过，创建线程池是通过工厂类来Executors创建的，代码如下：

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
}
```

可以看到核心线程数和最大线程数相同，并且没有超时机制，而且任务队列为无界队列。

这说明当线程处于空闲状态时，它们并不会被回收，除非线程池关闭，当有新任务到来时，它能快速的处理这个任务，如果所有的线程都处于工作状态，那么新任务就会被放入等待队列，并且任务队列能容纳无限个任务。

### 2、CachedThreadPool

   与第一种相反，它是一种线程数量不定的线程池。代码如下：

```java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
}
```

它的核心线程数为0，最大线程数为Integer.MAX_VALUE，相当于无限大，这说明线程池中的线程足够多，每个线程的超时时间为60秒，超过60秒空闲的线程就会被回收，它的任务队列是SynchronousQueue，它是一种特殊的队列，每当有任务插入队列，它都会把它直接提交给线程池处理。

所以这个线程池适用于任务并发量比较大的场景，每当有新任务到来，如果没有空闲线程，它都会创建一个线程处理，如果有空闲线程就交给空闲线程处理。

### 3、ScheduleThreadPool

它的核心线程数是固定，但是非核心线程数是不定的线程池。代码如下：

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
}

 public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE,
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
              new DelayedWorkQueue());
    }

 private static final long DEFAULT_KEEPALIVE_MILLIS = 10L;
 MILLISECONDS(TimeUnit.MILLI_SCALE),

```

它主要用于执行定时任务和具有固定周期的重复任务，下面演示一下如何使用：

```java
ublic class ScheduledThreadPoolDemo {

    public void doWork(){
        //创建定时执行的线程池
        ScheduledExecutorService executor = Executors.newScheduledThreadPool(3);
        //参数1是执行的任务，参数2是第一次运行任务延迟的时间，参数3是定视任务的周期，参数4是单位
        executor.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                System.out.println("工作线程： " + Thread.currentThread().getName() + ", 结果：" + fibc(10));
            }
        }, 1, 2, TimeUnit.SECONDS);
    }

    private int fibc(int n){
        if(n == 0) return 0;
        if(n == 1) return 1;
        return fibc(n - 1) + fibc(n - 2);
    }

}
```

上面设计了一个定时任务，计算10的斐波那契数，它会延时1秒后开始执行，然后每隔2秒重复执行一次。

使用：

```java
 public static void main(String[] args) throws InterruptedException {
        new ScheduledThreadPoolDemo().doWork();
    }
```

输出结果：

```java
工作线程： pool-1-thread-1, 结果：55
工作线程： pool-1-thread-1, 结果：55
工作线程： pool-1-thread-2, 结果：55
工作线程： pool-1-thread-2, 结果：55
...
```

### 4、SingleThreadExecutor

它是只有一个核心线程的线程池。如下：

```java
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

它相当于大小为一的FixedThreadPool。

因为只有一个线程用来执行任务，所以使得这些任务之间不需要处理线程同步的问题，任务都按顺序的排队执行。

##                                                                                                                                      结语

有关线程池的知识先介绍到这里了，了解线程池后，才能更好得去运用它。

参考资料：

[Java并发之线程池](https://www.jianshu.com/p/80797a141e66)

