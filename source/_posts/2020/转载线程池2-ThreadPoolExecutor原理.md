---
title: 线程池2:ThreadPoolExecutor原理
permalink: thread-pool-executor-yuan-li
date: 2020-02-27 13:49:54
tags:
categories: thread-pool
---
转载自[《线程池系列二》-ThreadPoolExecutor-线程池原理解析](https://www.jianshu.com/p/4002212ee4f5)
> 相信大家都使用过线程池，也了解使用线程池的好处。我们使用线程池最多的还是使用Executors工具类创建FixedThreadPool、SingleThreadPool以及CachedThreadPool三种线程池，如果我们不了解其工作原理，将会碰到很多意想不到的问题，例如内存被撑爆，cpu被打满，线程池无故中断，关闭线程池应该使用shutdown()还是shutdownNow()等等一系列的问题，这篇文章将讲解为什么要是用线程池，为什么会出现上述的问题，线程池的工作原理是什么，应该选择何种线程池，如何定义线程池线程的个数等等。本文采用JDK8源码进行讲解，主要讲解原理为主，不过多的涉及到源码。  


# 线程是什么 

不讲书本知识，只是抛出一个问题：new Thread(); new Runnable(); new Callable(); （尽管不能new 接口，这里只是说明意思，不要较真），请问这代表线程吗？ 

一定要注意，这都是类，java中的类，不是线程，千万不要看到thread， runnable， callable就认为是线程，它们和Object， List，Map一样是java语言的类而已。 或者可以说他们是任务，线程执行的任务，因为他们内部都有线程要执行的方法run()或者call()，那什么才是线程呢？ 

new Thread().start(); 调用了start()方法才会在操作系统层面启动一个线程，除此之外都是承载了线程执行方法的类而已。这一点大家一定要分清楚。 

<!--more-->

# 多线程 

多线程一定比单线程快吗？这个答案是否定的，在单核处理器环境下，多个线程执行任务势必会引起线程上下文切换，上下文切换会对当前线程的执行环境进行保存，并还原将要执行线程的执行环境，存在开销。多线程与单线程相比，多出了上下文切换的时间，因此在单核处理器环境下，多线程并不会提高性能。 

现今，处理基本上都是多核多处理器，因此合理使用多线程编程将取很大的性能提升。但是当线程数过多，引起过多的上下文切换，当上下文切换的开销大于多线程带来的收益的话，性能将会下降。滥用多线程将会是一场灾难。 

# 线程池的引入 

首先从线程类Thread讲起，Thread类具有两个功能： 

  1. 维护线程 线程的创建、休眠、中断、暂停、销毁
  2. 执行任务 Thread类及其子类run()， Runnable对象的run(), Callable对象的call() （更准确的说应该是FutureTask，因为Callable对象并不能传入Thread类）

Thread类将线程和任务耦合在一起，一般的使用方式为：有多少个任务就需要多少个线程去执行，并发的任务数太多，就会引起大量的上下文切换，以及线程的创建与销毁（线程的创建和销毁都设计到内核态和用户态的转换，开销也不容小觑）。 

为了能对线程进行统一的管理和复用，引入了线程池。线程池对线程进行统一的管理，并可以弹性的扩展，将执行任务和线程完全分离，任务存放到阻塞队列中，线程不断的去阻塞队列中取任务执行。从而达到线程复用的目的（说白了，线程在死循环中去阻塞队列获取数据，如果获取不到就阻塞，如果获取到就执行，其run（）方法一直执行），这样线程与任务个数比为m:n 其中m<<<n 

因此，编写多线程程序时，我们最好使用线程池。 

# 线程池的参数 

  * corePoolSize

核心线程数，当提交任务时如果线程数小于corePoolSize，则直接创建线程执行该任务，否则，将任务添加到阻塞队列 

  * maximumPoolSize

最大线程数，当提交任务时，任务需添加到阻塞队列且阻塞队列满时，如果线程数小于maximumPoolSize，则创建线程执行该任务，否则执行拒绝策略 

注：如果阻塞队列采用的是无界队列的话，该参数无意义，因为阻塞队列无界就永远不会满 

  * keepAliveTime

线程空闲时间，空闲时间超过该时间则销毁线程，只对大于corePoolSize~maximumPoolSize的线程有效，即至少保留corePoolSize个线程，即便空闲时间大于keepAliveTime也不销毁。（核心线程也是可以销毁的，需要设置核心线程过期） 

注：如果阻塞队列为无界，则maximumPoolSize无意义，那么keepAliveTime也就无意义 

  * unit

keepAliveTime的时间单位 

  * workQueue

阻塞队列，分为有界队列和无界队列，一般使用LinkedBlockingQueue、SynchronousQueue，用于存放任务，阻塞队列的泛型必须是Runnable 

  * threadFactory

线程工厂，负责创建线程，指定线程名，线程组，线程优先级，是否为守护线程等信息 

  * handler

拒绝策略，当阻塞队列放不下，且线程数达到最大值maximumPoolSize时，再提交任务，改任务会被拒绝。目前，JDK提供了四种拒绝策略 

  1. CallerRunsPolicy 调用线程执行策略，当前执行的线程执行该任务，可以保证任务不丢失，减缓任务添加的速度
  2. AbortPolicy 直接抛出异常，会导致线程池抛异常，线程池不可用，默认拒绝策略
  3. DiscardPolicy 直接丢弃该任务
  4. DiscardOldestPolicy 丢弃最老的任务，重试添加该任务

注：如果阻塞队列为无界，则拒绝策略无效，因为不会存在任务放不下的情况，也可以自定义自己的拒绝策略。该参数一定要重视 

# 线程池的构造函数 

构造线程池无非就是为上节中介绍的几个参数赋值，源码如下 
```java
public ThreadPoolExecutor(int corePoolSize,
              int maximumPoolSize,
              long keepAliveTime,
              TimeUnit unit,
              BlockingQueue<Runnable> workQueue,
              ThreadFactory threadFactory,
              RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

其他的构造函数，都是间接调用该构造函数 

# 线程池的工作原理 

  1. 提交任务，如何当前线程数<corPoolSize，**不管是否有空闲线程都会创建新的线程执行**
  2. 如何当前线程数>=corPoolSize，将任务提交给阻塞队列
  3. 如果阻塞队列不满，添加到阻塞队列，否则执行4
  4. 如果当前线程数<maxPoolSize,且不存在空闲线程则创建一个线程执行该任务，否则执行5
  5. 执行拒绝策略

**第一步需要注意的是在提交任务时，excutor会不会判断有无空闲线程，答案是不会，因为如果每次提交任务都需要判断有无空闲线程，将会造成很大的开销，excutor的做法是，启动的每一个worker在空闲时都会去阻塞队里阻塞的获取任务，如果没有任务则worker会阻塞，因为worker到底空不空闲worker自己是最清楚的。**

线程执行完一个任务之后，会从阻塞队列中获取任务，如果没有任务可以获取，则阻塞等待，如果有任务则直接获取执行。与此同时，线程池中有专门的线程坚持线程的空闲时间（等待任务的时间），如果超过指定时间且线程数>corePoolSize，就销毁线程。 

## Executors提供的三种线程池 

  * FixedThreadPool

固定大小的线程池，其源代码如下： 
```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
      0L, TimeUnit.MILLISECONDS,
      new LinkedBlockingQueue<Runnable>());
}
```

通过源码可以看出，线程池的corePoolSize和maximumPoolSize都为指定大小，阻塞队列使用无界阻塞队列（看到无界阻塞队列，就应该想到maximumPoolSize、keepAliveTime、handler都无效），因此，该方法中有用的参数只有corePoolSize和workQueue是有意义的。 

存在的问题：当任务执行的较慢，且任务提交的速度过快时，会有大量的任务存放到阻塞队列中，阻塞队列会越来越大，内存会被撑爆，使用该线程池时，一定要考虑清楚。 

除了该方法外，Executors还提供了重载方法，可以指定ThreadFactory，但是却没有提供修改阻塞队列的重载方法 

使用场景： 负载较重的服务器 

  * SingleThreadPool

当个线程的线程池，与FixedThreadPool相比就是将线程数指定为1，同样该线程池存在FixedThreadPool存在的问题，其源码如下： 
```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                0L, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<Runnable>()));
}
```

与FixedThreadPool类型，Executors也提供了指定ThreadFactory的重载方法 

使用场景： 单线程执行环境，保证顺序执行各个任务的场景 

  * CachedThreadPool

使用SynchronousQueue阻塞队列，该队列不保存元素，有任务提交到阻塞队列时，任务必须立即被处理。源码如下： 
```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                  60L, TimeUnit.SECONDS,
                  new SynchronousQueue<Runnable>());
}
```

从源码中可以看出，maximumPoolSize的值为Integer.MAX_VALUE，意味着只要有任务到达，且线程池内没有空闲线程，就给任务开辟一个线程去执行。线程空闲60s就销毁 

存在问题：如果任务执行时间长，提交速度快，那么会产生大量的线程，引起上下文切换，应用可能会出现假死或者崩溃的情况。 

同样，这种类型的线程池，也提供了一个指定ThreadFactory的重载方法  
使用场景：适用于大量短期异步任务，或者负载较轻的服务器 

由此可见：Executors提供的三种线程池都各自有优缺点，如果使用线程池，建议不要使用这三种线程池，而是直接通过线程池的构造方法指定自己的corePoolSize，maximumPoolSize，keepAliveTime，阻塞队列workQueue，ThreadFactory，拒绝策略，自己指定的优点就是可以根据自己的场景灵活的对各个参数进行配置。 

# 线程池提交任务 

  * submit()

提交有返回值的任务，返回值为Future类型（真正的类型是RunnableFuture，而实现RunnableFuture接口的在JDK实现中对外可以使用的就只有FutureTask类 

有关FutureTask的相关知识可以参考我的另外一篇文章： [FutureTask原理讲解与源码剖析][1]） 

   [1]: https://www.jianshu.com/p/36f8f3b8ee55

  * execute()

提交没有返回值的任务 

## 线程池关闭 

  * shutdown()

将线程池的状态修改为shutdown，禁止向线程池中提交任务，并执行完已经提交的任务 

  * shutdonwNow()

将线程池的状态修改为stop， 立即终止线程池中的线程， 不处理阻塞队列中的任务，返回没有执行任务的列表 

可以通过isTerminated()方法判断线程池是否完全关闭  
也可以通过awaitTermination(long timeout, TimeUnit unit)最长等待一段时间后退出，但并不能保证关闭 

# 如何分配线程池的大小 

一般来讲没有上下文切换的多线程程序是最好的，因此，如果有n个核，那么启动n个线程就可以。但是线程并不是一直处于运行状态（可能在等待IO放弃了cpu资源），这样cpu资源就会浪费，因此我们一般针对不同的任务设定不同的线程数。 

首先我们应该获取服务器的线程数，可以通过如下代码获取： 
```
Runtime.getRuntime().availableProcessors();
```
    

注意，如果使用docker容器，使用该参数获取的是实机的核数，并不是分配给docker容器的核数，如果碰到需要修改， 具体情况具体分析。 

  1. 针对IO密集型任务：一般分配2*p个线程（p代表服务器cpu总核数）
  2. 针对cpu密集型任务： 一般分配 p+1个线程

# 线程池的监控 

线程池提供了很多参数，来记录线程池中各个状态，了解即可： 

  1. taskCount 线程池执行任务总数
  2. completedTaskCount 已执行完成任务数量
  3. largestPoolSize 创建过最大的线程数
  4. getPoolSize() 当前线程数量
  5. getActiveCount() 活动线程数

除此之外，还可以继承线程池类定义自己的线程池实现， 可以重写 beforeExecute(), afterExecute(), terminated()方法设置监控 
