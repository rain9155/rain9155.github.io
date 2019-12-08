---
title: java学习总结之线程池
tags: 
- 线程池
categories: java
date: 2019-07-19 12:26:00
---


## 前言

* 上一篇文章：[java学习总结之线程](https://rain9155.github.io/2019/07/19/java%E7%BA%BF%E7%A8%8B/)

当我们需要频繁的创建多个线程时，每次都通过new一个Thread是一种不好的操作，创建一个线程是要消耗资源，频繁的创建会导致性能较差，而且我们还要管理多个线程的状态，管理不好还可能会出现死锁，浪费资源。这时就需要java提供的线程池，它能够有效的管理、调度线程，避免过多资源的消耗，通过线程池的统一调度、管理，使得多线程开发变得更简单。本文讲解一下有关线程池的知识点。

<!--more-->

## 一、Executor框架

线程池属于Executor框架的**一部分**，Executor框架包括**任务**，**任务的执行**、**任务执行的结果**三部分，其中线程池属于任务的执行那一部分，线程池的主要类和接口如下：

{% asset_img executor1.png executor %}

主要角色介绍：

* **Executor**：它是一个接口，里面只有一个方法execute(Runnable command)，用来提交任务到线程池执行；
* **ExecutorService**：它继承Executor，同样是一个接口，里面提供了更多的方法用于操作线程池，如Future<?> submit(Runnable task)可以提交有返回值的任务到线程池执行、shutdown和shutdownNow方法可以用来关闭线程池；
* **ThreadPoolExecutor**：它是真正的线程池的实现，它实现了上面接口的方法，还提供了一系列参数来配置线程池；
* **ScheduleThreadPoolExecutor**：它继承自ThreadPoolExecutor，实现了ScheduledExecutorService接口，也是线程池的实现，它在ThreadPoolExecutor的基础上提供了用于执行定时或延迟任务的方法，如scheduleAtFixedRate和scheduleWithFixedDelay方法，它可以用来取代java中的Timer；
* **Executors**：它是一个工厂类，通过它提供的工厂方法可以创建不同的线程池，即返回不同配置参数的ThreadPoolExecutor或ScheduleThreadPoolExecutor实例.

而线程池是用来执行**任务**的，在java中，任务被分为两种，一种是没有返回值的任务，它用**Runnable**接口表示，一种是有返回值的任务，它用**Callable**接口表示；而在线程池中，**任务的执行结果**用**Future**接口表示，只要实现了Furure接口的类都可以作为线程池的任务返回结果，在java中，Furure接口的一个主要实现类是**FutureTask**，任务和任务执行结果它们之间的关系如下：

{% asset_img executor2.png executor %}

主要角色介绍：

* **Runnable**：一个接口，代表没有返回值的任务，通过Executor的execute(Runnable)方法执行其中的run方法；
* **Callable**：一个接口，代表有返回值的任务，可以使用ExecutorService的submit(Callable)方法执行，还可以通过FutureTask**包装**后，使用ExecutorService的submit(Runnable)方法执行，任务完成后，返回V类型的结果，V是一个泛型；
* **Future**：一个接口，代表着异步任务的返回结果，只要实现了Furure接口的类都可以作为线程池的任务返回结果，通过get方法(阻塞)可以获取返回结果，通过cancel可以取消任务的执行；
* **FutureTask**：它实现了Runnable和Future接口，所以它里面会有run方法用来执行任务和get、cancel等方法用来操作任务，这说明FutureTask即可以被**当作任务**提交到线程池执行，又可以被当作线程池的**任务返回结果**，它的内部是通过AQS(AbstractQueuedSynchronizer)来实现同步管理，AQS是java5之后加入的一个同步框架.

> FutureTask它有两个构造器： FutureTask(Callable<V> )和FutureTask(Runnable, V) ，第一个构造器可以用来包装一个Callable对象；第二个构造器可以用来包装一个Runnable对象，里面会通过Executors的callable方法把Runnable对象适配成Callable对象，并以第二个参数的V类型作为返回值类型，如果没有返回值，传入null就可以.
>
> 所以FutureTask的run方法中最终执行的是**Callable的call方法**，返回V类型的结果，更多细节可以查看FutureTask内部实现。

**线程池使用的大概流程如下**：

首先程序创建实现了Runnable或者Callable接口的任务，然后通过Executors相应方法返回或自己配置一个**ThreadPoolExecutor**，然后把任务通过ThreadPoolExecutor的相应方法提交，如果提交的是Callable任务，ThreadPoolExecutor还会把它包装成FutureTask任务，由于FutureTask也实现了Runnable接口，所以不管提交的是Runnable还是Callable任务，ThreadPoolExecutor最终执行的还是Runnable类型的任务；

如果你使用的是ThreadPoolExecutor的submit(XX)来提交任务，它会返回一个实现了Future接口的对象，在java中，默认返回的是FutureTask，然后程序就可以通过FutureTask.get()来等待任务执行完成，也可以通过FutureTask.cancel(boolean)来取消任务的执行；

如果你使用的是ThreadPoolExecutor的execute(XX)来提交任务，任务就会等待ThreadPoolExecutor调度执行直到完成或抛出异常，你无法操作它的执行过程。

## 二、线程池的配置参数

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

**含义**：线程池中的核心线程数。

线程池启动后默认是空的，只有任务到来时才会创建线程以处理请求，如果调用了ThreadPoolExecutor的**prestartAllCoreThreads方法**，可以在线程池启动后立即创建所有的核心线程以等待任务。

还有在默认情况下，核心线程一旦创建后就会在线程池中一直存活，即使它们处于空闲状态，如果设置ThreadPoolExecutor的**allowCoreThreadTimeOut(boolean value)方法为true**，那么空闲的核心线程在等待新任务到来时就会有超时策略，这个超时时间由keepAliveTime指定，当等待时间超过keepAliveTime后，核心线程就会被终止。

### 2、int  maximumPoolSize

**含义**：线程池所能创建的最大线程数，它与**corePoolSize**、**workQueue**共同调整线程池中实际运行的线程数量。

 当线程池中的工作线程数小于corePoolSize时，每次来任务的时候都会创建一个新的工作线程。不管工作线程集合中有没有线程是处于空闲状态；当池中工作线程数大于等于 corePoolSize 的时候，每次任务来的时候都会首先尝试将线程放入队列，而不是直接去创建线程。

如果放入队列失败，说明队列满了，且当线程中线程数小于 maximumPoolSize 的时候，则会创建一个工作线程（非核心线程）来执行这个任务，如果线程池中的线程数大于maximumPoolSize，调用给定的拒绝策略；如果任务成功放入队列，就等待线程取出执行。

如图，线程池的工作流程如下:

{% asset_img executor3.png executor3%}

> 工作线程：执行任务的线程;
> 空闲线程：已经执行完任务，并且还在存活着的线程.

### 3、 long  keepAliveTime

**含义**：非核心线程空闲时的超时时长，超过这个时长，非核心线程就会被回收。

默认情况下只对非核心线程有作用，我们可以通过调用**allowCoreThreadTimeout(true)**来将这种策略应用给核心线程，这样核心线程也会有超时机制。

### 4、TimeUnit  unit

**含义**：指定keepAliveTime的单位，可选值有毫秒、秒、分等。

### 5、 BlockingQueue   workQueue

**含义**：线程池中的任务队列，用来保存等待执行任务的阻塞队列。

首先 BlockingQueue 是一个接口，这是一个很特殊的队列，如果 BlockQueue 是空的，从 BlockingQueue 取东西的操作将会被阻断进入等待状态，直到 BlockingQueue 进了东西才会被唤醒。同样，如果 BlockingQueue 是满的，任何试图往里存东西的操作也会被阻断进入等待状态，直到 BlockingQueue 里有空间才会被唤醒继续操作。

BlockingQueue 大致有四个实现类，如下：

* **ArrayBlockingQueue**：规定大小的基于数据结构的 BlockingQueue，即有界队列，其构造函数必须带一个 int 参数来指明其大小，其所含的对象是以 FIFO(先入先出)顺序排序的，如果队列满了调用给定的拒绝策略；
* **LinkedBlockingQueue**： 大小不定的基于链表结构的 BlockingQueue，既可以有界也可以无界，若其构造函数带一个规定大小的参数，生成的 BlockingQueue 有大小限制，若不带大小参数，所生成的 BlockingQueue 的大小由 Integer.MAX_VALUE 来决定，其所含的对象是以 FIFO(先入先出)顺序排序的；所以如果该队列是无界的，则可以忽略给定的拒绝策略，因为它永远都不会满，同时还可以忽略maximumPoolSize 参数，因为起当核心线程都在忙的时候，新的任务被放在队列上，永远不会有大于 corePoolSize 的线程被创建；
* **PriorityBlockingQueue**：优先级队列，类似于 LinkedBlockQueue，可以有界也可以无界，但其所含对象的排序不是 FIFO，而是依据对象的自然排序顺序或者是构造函数的 Comparator 决定的顺序；
* **SynchronousQueue**：特殊的 BlockingQueue，对其的操作必须是放和取交替完成的，因为其特殊的操作，所以如果有一个任务要插入队列，那么它必须要等到另一个移除任务的操作，所以使用该队列会直接把任务提交给线程池，而不会将任务加入队列，如果线程池没有任何可用的线程处理，就调用给定的拒绝策略。

BlockingQueue 的常用方法：

- add(object)：把 object 加到 BlockingQueue 里，即如果 BlockingQueue 可以容纳，则返回 true，否则报异常；
- offer(object)：表示如果可能的话，将 object 加到 BlockingQueue 里，即如果 BlockingQueue 可以容纳，则返回 true，否则返回 false；
- put(object)：把 object 加到 BlockingQueue 里，如果 BlockQueue 没有空间，则调用此方法的线程被阻断直到 BlockingQueue 里面有空间再继续；
- take()：取走 BlockingQueue 里排在首位的对象，若 BlockingQueue 为空，阻断进入等待状态直到 Blocking 有新的对象被加入为止；
- poll(time)：取走 BlockingQueue 里排在首位的对象，若不能立即取出，则可以等 time 参数规定的时间，取不到时返回 null。

### 6、ThreadFactory  threadFactory

**含义**：线程工厂，让用户可以定制创建线程的过程。

ThreadFactory  是一个接口，它只有一个**Thread newThread(Runnable)方法**，如果没有指定threadFactory，默认调用Executors的**defaultThreadFactory方法**返回一个DefaultThreadFactory，DefaultThreadFactory创建的线程都属于同一个线程组和拥有同样的优先级。

除了默认的ThreadFactory，我们可以实现ThreadFactory 接口自定义自己的ThreadFactory，这样就可以自定义线程的名字、线程组合等状态，如果newThread方法返回null，线程池将不会执行任何任务。

### 7、RejectedExecutionHandler handler

**含义**：当新任务到来时，线程池被**关闭**或线程数maximumPoolSize和任务队列大小已经**达到上限**的时候，对新任务采取的拒绝策略。

RejectedExecutionHandler 同样是一个接口，里面只有一个**rejectedExecution(Runnable, ThreadPoolExecutor)方法**，下面介绍一下几个默认的实现，都定义在ThreadPoolExecutor中，都实现了RejectedExecutionHandler 接口：

* **AbortPolicy**：直接抛出 RejectedExecutionException 异常，线程池的默认实现；
* **CallerRunsPolicy**：这个策略将会使用 Caller 线程来执行这个新任务，可以降低任务提交的速度；
* **DiscardPolicy**：这个策略将会直接丢弃新任务；
* **DiscardOldestPolicy**：这个策略将会把任务队列头部的任务丢弃，然后重新尝试执行新任务，如果还是失败则继续实施该策略（这样的结果是最后加入的任务反而更有可能先被执行）.

和ThreadFactory  一样，我们也可以实现RejectedExecutionHandler 接口自定义自己的拒绝策略。

## 三、线程池的生命周期

线程池的生命周期包含3种状态，如下：

### 1、运行

线程池创建后就进入运行状态，这个时候可以向线程池提交任务，可以通过ThreadPoolExecutor的execute方法或submit方法，只有处于运行的状态才能提交任务。

**execute方法**用于提交不需要返回值的任务，所以无法判断任务是否被线程池执行成功；而**submit方法**用于提交需要返回值的任务，这时线程池会返回一个Future类型的对象，通过这个Future对象可以判断任务是否执行成功，并且可以通过Future的get方法来获取返回值，get方法会阻塞当前线程直到任务完成，而使用get（long timeout，TimeUnit unit）方法则会阻塞当前线程timeout时间后立即返回，这时候有可能任务没有执行完，当线程池的任务还没有执行完时，会报超时异常。

### 2、关闭

当调用ThreadPoolExecutor的shutdown或shutdownNow方法后，便会进入关闭状态，这时意味线程池不再接受新的任务，这时isShutdown方法返回true。

调用**shutdown方法**会中断没有正在执行任务的线程，而正在执行任务的线程，会等待线程执行完毕后再关闭线程池；但是如果调用的是**shutdownNow方法**，它不管你是空闲线程还是正在执行任务的线程，它都会立即中断。

shutdown方法和shutdownNow方法方法中断线程的原理都是通过调用线程的**interrupt方法**，所以如果你的没有正确处理中断事件，你的线程还是不会马上停止，而是等到线程执行完毕或抛出异常后才停止，如何正确中断一个线程? 可以查看我的上一篇文章[java学习总结之线程](https://rain9155.github.io/2019/07/19/java%E7%BA%BF%E7%A8%8B/);

> 可以看到，线程池关闭的时候会把所有线程中断，并让线程池处于关闭状态，这时你的线程池就无法再次提交任何任务了，所以如果你只是想中断线程池中的一个或几个任务，可以通过使用 submit方法来提交任务，它会返回一个 Future<?> 对象，通过调用该对象的 cancel(true) 方法就可以中断这个任务。

### 3、终止

在关闭状态的线程池执行完所有已经提交的任务后，就变为终止状态，这时调用isTerminated方法会返回true。

## 四、线程池的分类

通过配置ThreadPoolExecutor的构造函数的参数就可以实现不同类型的线程池，但是配置一个ThreadPoolExecutor会很繁琐，需要了解那么多参数，所以我们可以使用工厂类来Executors创建线程池，Executors已经为我们配置好了四种类型的线程池，它们分别是：FixedThreadPool、CachedThreadPool、ScheduleThreadPool和SingleThreadExecutor，我们通过调用Executors的newXX方法就可以得到这些线程池的实例，下面分别介绍：

### 1、FixedThreadPool

ThreadPoolExecutor类型，通过传入的参数大小，创建一种固定线程数量的线程池，如下：

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
}
```

可以看到核心线程数和最大线程数相同，没有超时机制，队列为无界队列，每当新任务到来时，如果线程池中的线程数还没有到达核心线程数，就会立即创建一个工作线程来处理这个任务，如果线程数达到核心线程数，那么新任务就会被放入任务队列等待，并且这个队列能够容纳无限个任务，这样做的后果是导致**最大线程数和超时机制无效**，只要线程池没有被关闭，那么对于新任务的到来，只有两种处理：被核心线程执行或放入任务队列，永远不会创建一个非核心线程。

FixedThreadPool适用于资源有限，需要限制当前线程数量的场景。

### 2、CachedThreadPool

 ThreadPoolExecutor类型，与FixedThreadPool相反，它是一种线程数量不固定的线程池，如下：

```java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
}
```

它的核心线程数为0，最大线程数为Integer.MAX_VALUE，相当于无限大，这说明线程池中的线程足够多，每个线程的超时时间为60秒，超过60秒空闲的线程就会被回收，它的任务队列是SynchronousQueue，它是一种特殊的队列，它不会保存任务，每当有新任务插入队列，它都会把新任务**立即**提交给线程池处理，如果线程池没有空闲线程，它会**立即**创建一个线程处理，如果有空闲线程就交给空闲线程处理，所以只要线程池没有被关闭，对于每个新任务，它都来者不拒。

CachedThreadPool适用于任务执行时间短、并发量比较大的场景。

> 在极端情况下，任务数量非常多，任务执行时间非常长，CachedThreadPool会因为创建过多线程而导致耗尽CPU资源和内存资源。


### 3、SingleThreadExecutor

ThreadPoolExecutor，它是只有一个核心线程的线程池。如下：

```java
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

它相当于大小为一的FixedThreadPool，因为只有一个线程用来执行任务，所以使得这些任务之间不需要处理线程同步的问题，任务都按顺序的排队执行。

SingleThreadExecutor适用于需要按顺序执行任务的场景。

### 4、ScheduleThreadPool

它和前面3个线程池不一样，它是ScheduledThreadPoolExecutor类型的，ScheduledThreadPoolExecutor继承自ThreadPoolExecutor，实现了ScheduledExecutorService接口，ScheduledExecutorService接口提供了一些用于用于执行定时任务和周期任务的方法，如scheduleAtFixedRate和scheduleWithFixedDelay方法。

我们看一下Executors的newScheduledThreadPool方法，如下：

```java
 private static final long DEFAULT_KEEPALIVE_MILLIS = 10L;
 MILLISECONDS(TimeUnit.MILLI_SCALE),

public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
}

 public ScheduledThreadPoolExecutor(int corePoolSize) {
       //super就是ThreadPoolExecutor的构造函数
        super(corePoolSize, Integer.MAX_VALUE,
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,//DEFAULT_KEEPALIVE_MILLIS为10L，单位为毫秒
              new DelayedWorkQueue());
    }

```

可以看到它是一个核心线程数大小为corePoolSize，最大线程数大小为 Integer.MAX_VALUE的线程池，它的空闲线程的超时时间为10毫秒，并且使用**DelayedWorkQueue**作为任务队列，DelayedWorkQueue是一个延时队列，它也是实现了BlockingQueue接口的队列，并且是一个无界队列，所以**最大线程数和超时机制无效**，当一个新任务到来时，它会把新任务放入任务队列中，因为任务队列是一个DelayedQueue，所以任务会按照它的执行时间排序，越先执行的排在越前面，队列中的任务等时间到了，会被线程池中的线程取出执行，任务执行后，修改时间为下次执行时间，再放入队列，等待下次再次执行。

下面演示一下如何使用：

```java
public class ScheduledThreadPoolDemo {

    public void doWork(){
        
        //创建定时执行的线程池
        ScheduledExecutorService executor = Executors.newScheduledThreadPool(3);
        
        //参数1是执行的任务
        //参数2是第一次运行任务延迟的时间
        //参数3是定时任务的周期
        //参数4是单位
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

ScheduleThreadPool适用于资源有限，需要有限个线程执行周期任务的场景.

> Executors中还有一个newSingleThreadScheduledExecutor方法，用于创建大小为1的ScheduleThreadPool，适用于需要单个线程执行周期任务，并且任务需要排队处理的场景。

##                                                                                                                                      结语

本文简单的介绍了Executor框架和介绍了线程池ThreadPoolExecutor的配置参数，还介绍了Executors工厂类提供的四种类型线程池，分别是：FixedThreadPool、CachedThreadPool、ScheduleThreadPool和SingleThreadExecutor，它们都有着各自的应用场景，在开发中，如果Executors中提供的线程池无法满足我们，就需要我们自己手动去配置，所以一定要**熟悉**线程池的各种配置参数，不然会导致你配置出一个错误的线程池，最简单的办法就是参考Executors中的配置，我们只需要合理的修改一下参数的大小和队列类型就能为我们所用。

有关线程池的基础知识先介绍到这里了，希望大家有所收获！

