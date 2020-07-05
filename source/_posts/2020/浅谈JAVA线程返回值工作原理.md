---
title: 线程池导读2:浅谈JAVA线程返回值工作原理
permalink: qian-tan-java-thread-pool-return
date: 2020-02-27 11:00:31
tags:
categories: thread-pool
---

# 前言

> 请先看[浅谈JAVA线程池工作原理](https://www.chenguanting.top/2020/浅谈JAVA线程池工作原理)后再看本篇内容

# 小Demo

```java
ExecutorService executorService = Executors.newCachedThreadPool();
Future<Demo1> future = executorService.submit(new Callable<Demo1>() {
  @Override
  public Demo1 call() throws Exception {
    return new Demo1();
  }
});
// 一直阻塞到子线程结束拿到返回值
Demo1 demo = future.get();
```

相信大家都看过上面这种类型的方法, 实现了将线程池中的线程阻塞, 并且拿到了返回值. 

<!--more-->

**以下代码均为伪代码,目的是提供思路**

# 分析

1. 线程调用的方法需要同时支持两种类型`Callable`和`Runnable`
   1. 不能影响原有类型, 所以我们需要搞个包装类
2. 经上篇分析传入的参数是保存在`Runnable`中的,  那么我们只需要在这个`Runnable`外层搞个包装类,增加一个成员变量`status`,就可以判断线程是否执行完成,并且可以完成如果没有完成就等待的问题
3. 那么我们的返回值呢?没错, 返回值也放在包装类中, 等线程执行完返回就好!

# 实现

## 分析1

>  这个版本我们只实现这个线程调用的方法需要同时支持两种类型`Callable`和`Runnable`

### 自定义接口

```java
public interface MyCallable {

    void call();
}
```



### 包装类

```java
public class MyFutureTask implements Runnable{

    // 对象都包在这里面
    private MyCallable callable;

    public MyFutureTask(MyCallable callable) {
        this.callable = callable;
    }

    public MyFutureTask(Runnable runnable) {
        this.callable = new InerCallable(runnable);
    }

		// 一个包装类
    class InerCallable implements MyCallable{
        private Runnable runnable;

        public InerCallable(Runnable runnable) {
            this.runnable = runnable;
        }
        @Override
        public void call() {
            runnable.run();
        }
    }

    @Override
    public void run() {
        callable.call();
    }
}
```

### 线程池

> 在上一篇的自定义线程池基础上增加了`submit`方法

```java
public class MyThreadPool {

    // 一个阻塞列队
    ArrayBlockingQueue<Runnable> que ;
    // 创建出来的线程,为了简化我这里只初始化进来一个线程
    Thread thread ;
    // 构造方法
    public MyThreadPool() {
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
  
    void submit(Runnable runnable) {
        MyFutureTask myFutureTask = new MyFutureTask(runnable);
        execute(myFutureTask);
    }
    void submit(MyCallable callable) {
        MyFutureTask myFutureTask = new MyFutureTask(callable);
        execute(myFutureTask);
    }
}

```

## 分析2

>  这个版本我们加入判断线程是否执行完的方法

### 包装类

> 此处增加了线程的执行状态, 以及如果线程没执行完就死循环的方法

```java
public class MyFutureTask implements Runnable{

    // 对象都包在这里面
    private MyCallable callable;
    
    // 0:未执行/正在执行 1:执行完毕
    private volatile int status;

    public MyFutureTask(MyCallable callable) {
        this.callable = callable;
        status = 0;
    }

    public MyFutureTask(Runnable runnable) {
        this.callable = new InerCallable(runnable);
        status = 0;
    }


    class InerCallable implements MyCallable{
        private Runnable runnable;

        public InerCallable(Runnable runnable) {
            this.runnable = runnable;
        }
        @Override
        public void call() {
            runnable.run();
        }
    }

    @Override
    public void run() {
        callable.call();
        status = 1;
    }
    
    public void get(){
        if (status == 0){
            // 如果状态为未执行就开始死循环, 直到状态被修改为执行完毕
            while (true){
                if (status==1){
                    break;
                }
                TimeUnit.SECONDS.sleep(1);
            }
        }
    }
}

```

### 线程池

> 之前的方法省略, 此处仅仅添加了调用的方法

```java
public class MyThreadPool {
// 之前的方法省略, 此处仅仅添加了调用的方法
    void invokeAll(List<MyCallable> callableList) {
        List<MyFutureTask> taskList = new ArrayList<>();
        for (MyCallable callable : callableList) {
            MyFutureTask myFutureTask = new MyFutureTask(callable);
            taskList.add(myFutureTask);
            execute(myFutureTask);
        }
        for (MyFutureTask myFutureTask : taskList) {
            myFutureTask.get();
        }
    }
}
```

## 分析3

> 返回值也放在包装类中,此处需要用到泛型

### 自定义接口

> 此处修改了返回值类型为泛型

```java
public interface MyCallable<T> {

    T call();
}
```

### 包装类

> 1. 增加了泛型的返回值
> 2. `InerCallable`调用位置也进行了修改, 直接返回`null`
> 3. `call`方法返回后值赋值给包装类中的成员变量`result`
> 4. 调用`get`方法执行完后会返回`result`

```java
public class MyFutureTask<T> implements Runnable {

    // 对象都包在这里面
    private MyCallable<T> callable;

    // 0:未执行/正在执行 1:执行完毕
    private volatile int status;

    private T result;

    public MyFutureTask(MyCallable<T> callable) {
        this.callable = callable;
        status = 0;
    }

    public MyFutureTask(Runnable runnable) {
        this.callable = new InerCallable(runnable);
        status = 0;
    }


    class InerCallable implements MyCallable<T> {
        private Runnable runnable;

        public InerCallable(Runnable runnable) {
            this.runnable = runnable;
        }

        @Override
        public T call() {
            runnable.run();
            return null;
        }
    }

    @Override
    public void run() {
        result = callable.call();
        status = 1;
    }

    public T get() {
        if (status == 0) {
            // 如果状态为未执行就开始死循环, 直到状态被修改为执行完毕
            while (true) {
                if (status == 1) {
                    return result;
                }
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
        return null;
    }
}
```

