---
title: 线程池导读1:浅谈JAVA线程池工作原理
permalink: qian-tan-java-thread-pool
date: 2020-02-26 23:06:24
tags:
categories: thread-pool
---

# 前言

> 可以先看看这篇博客: [JAVA中的线程和操作系统中的线程的关系](https://blog.csdn.net/lihao19920629/article/details/94999059),从这里面我们可以直到以下几点
>
> 1. Thread类调用start方法会通过jvm调用系统底层函数创建线程。
> 2. 创建完一个新的线程后, 新的线程会回调我们的run方法
> 3. 创建线程很麻烦,开销很大

<!--more-->

**以下代码均为伪代码,目的是提供思路**

# 分析

```java
final List list = new ArrayList();
new Thread(new Runnable() {
  @Override
  public void run() {
    fun(list);
  }
}).start();
```

 我们的run方法是写在一个实现`Runable`接口的对象中的, 因为该对象是一个匿名内部类, 所以传入的参数`list`的引用地址会被赋值到匿名内部类的一个虚拟成员变量中, 我们可以理解如下:

```java
class A implements Runnable{
  List list;
  public void A(List list){
    this.list = list;
  }
  void run(){
    fun(list);
  }
}
```

新的线程会调用我们实例化出来的类`A`的`run()`;执行完`run()`线程关闭;

可以看出来, 废了那么大的劲, 我们的目的就是为了调用`run()`,而该方法执行时的参数都在我们的匿名内部类`A`中, 那我们完全可以将`A`存到一个列队里,我们新建出来线程池后不销毁, 直接从列队中获取到`A`然后调用`A`的`run()`方法就好了

# 实现一个简单的线程池

```java
class ThreadPool{
  // 一个阻塞列队
  ArrayBlockingQueue<Runnable> que ;
  // 创建出来的线程,为了简化我这里只初始化进来一个线程
  Thread thread ;
	// 构造方法
  public ThreadPool() {
    // 初始化列队
    que = new ArrayBlockingQueue<Runnable>(10);
    // 初始化线程
    this.thread = new Thread(new Runnable() {
      @Override
      public void run() {
        // 创建出来的线程不销毁,一致占用着遍历列队, 列队中有值后就获取Runnable,并调用run()
        while (true){
          try {
            Runnable runnable = que.take();
            runnable.run();
          } catch (InterruptedException e) {
            e.printStackTrace();
          }
        }
      }
    }) ;
  }
	
  // 向线程池中加入任务的时候,直接扔到列队里, 让初始化的那个线程去扫描
  void execute(Runnable command){
    try {
      que.put(command);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
  }
}
```

# 调用

```java
public static void main(String[] args) {
  ThreadPool pool = new ThreadPool();

  List<String> list = new ArrayList();
  for (String s : list) {
    pool.execute(new Runnable() {
      @Override
      public void run() {
        System.out.println(s);
      }
    });
  }
}
```

# 警惕

参数传入匿名内部类后,加入修改了包装对象中的值, 而**匿名内部类中保存的是地址值**, 所以获取到的也是新的值,而我们无法保证线程池中线程的执行时间, 所以该包装对象千万不要修改!

```java
public static void a(){
  final List<Integer> list = new ArrayList();
  list.add(1);
  new Thread(new Runnable() {
    @Override
    public void run() {
      TimeUnit.SECONDS.sleep(2);
      list.add(3);
      System.out.println("run" + list);
    }
  }).start();
  TimeUnit.SECONDS.sleep(1);
  list.add(2);
  System.out.println("main"+list);
}
```

结果如下

```java
main[1, 2]
run[1, 2, 3]
```

