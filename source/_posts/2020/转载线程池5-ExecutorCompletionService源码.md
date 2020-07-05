---
title: 线程池5:ExecutorCompletionService源码
permalink: executor-completion-service-yuanma
date: 2020-03-01 22:32:08
tags:
categories: thread-pool
---

转载自[《线程池系列五》-ExecutorCompletionService原理分析](https://www.jianshu.com/p/9b85c47a6ff4)

> ExecutorCompletionService可以很好的配合线程池使用，它的内部封装了线程池（线程池需要在构造对象时传入），将提交的任务代理给线程池执行（但任务已经不再是FutureTask类型，而是FutureTask的子类QueueingFuture，QueueingFuture重写了done()方法，该方法在FutureTask类中是空实现），因为提交的任务被转换为QueueingFuture对象，该对象在任务处理完成之后，会主动将该任务放到ExecutorCompletionService维护的阻塞队列中，因此执行完成的任务都会被放到阻塞队列中，使用结果时，只需调用take()或者poll()方法获取即可。 

# 使用示例 

使用ExecutorCompletionService需要一下几步： 

  1. 创建线程池对象
  2. 创建ExecutorCompletionService对象，将线程池对象作为参数。
  3. 向ExecutorCompletionService对象提交任务，任务最终还是会代理给线程池对象执行。
  4. 执行take()方法获取执行完成的任务，通过get()方法获取任务执行结果。

<!--more-->

示例代码如下： 
```java
package com.feng.validation;

import java.util.concurrent.*;

/**
 * Created by xinfeng.xu on 2017/10/16.
 */
public class ExecutorCompletionServiceDemo {

    private int threadCount = Runtime.getRuntime().availableProcessors();
    private ThreadPoolExecutor executor = new ThreadPoolExecutor(threadCount, threadCount,
            0, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>(10),
            new ThreadPoolExecutor.DiscardPolicy());
    private ExecutorCompletionService<String> service = new ExecutorCompletionService<String>(executor);

    static class Task implements Callable<String>{

        @Override
        public String call() throws Exception {
            return "OK";
        }
    }

    public void executeTasks() throws InterruptedException, ExecutionException {

        //提交任务
        for(int i=0; i<100; i++){
            service.submit(new Task());
        }
        //获取结果
        for(int i=0; i<100; i++) {
            //获取完成的任务，如果没有完成的任务，将会阻塞
            Future task = service.take();
            //获取任务结果
            System.out.println(task.get());
        }
    }
    public static void main(String[] args) {

        try {
            ExecutorCompletionServiceDemo demo = new ExecutorCompletionServiceDemo();
            demo.executeTasks();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```
    

# CompletionService接口 

ExecutorCompletionService实现了CompletionService接口，该接口定义提交任务，获取执行完成任务的方法 

  * 提交任务操作  
submit() 可以提交callable对象和runnable对象
  * 获取执行完成任务操作  
take( ) 阻塞式获取完成的任务  
poll( ) 非阻塞式获取完成任务  
poll(timeout) 有限阻塞获取完成任务

# QueueingFuture类 

ExecutorCompletionService的内部类，该类实现了FutureTask类，重写了done()方法（该方法在FutureTask中是空实现），该方法什么时候被调用的，可以回忆一下FutureTask的finishCompletion()，该方法在任务执行完成时调用，唤醒等待线程之后调用done()方法，FutureTask的finishCompletion()方法源码如下： 
```java
private void finishCompletion() {
    // assert state > COMPLETING;
    for (WaitNode q; (q = waiters) != null;) {
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            for (;;) {
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    LockSupport.unpark(t);
                }
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }

    done();

    callable = null;        // to reduce footprint
}
```

QueueingFuture 源码如下： 
```java
private class QueueingFuture extends FutureTask<Void> {
    QueueingFuture(RunnableFuture<V> task) {
        super(task, null);
        this.task = task;
    }
    protected void done() { completionQueue.add(task); }
    private final Future<V> task;
}
```

从源码中可以看出，任务执行完成后，就会将任务添加到completionQueue的阻塞队列中。 

# 成员变量 

  * Executor executor  
线程池对象，执行任务由该对象执行
  * AbstractExecutorService aes  
该对象的主要作用是将runnable或者callable对象转换为FutureRunnable对象。由于不知道Executor具体是哪一种实现，因此如果是AbstractExecutorService的子类，那么就将executor强制转化为AbstractExecutorService类型，只是表明能够直接调用newTaskFor()方法而已。如果不是AbstractExecutorService类型，那么就直接装换为FutureTask类型。  
**这里为什么不全都转化为FutrueTask类型的， 因为实现AbstractExecutorService的子类不一定都是使用的FutureTask类，可能是自己定义的类，要保证一致。**
  * BlockingQueue<Future<V>> completionQueue  
存放执行完成的任务，done()方法中添加

# 构造方法 

为三个成员变量赋值，其中executor参数是需要传入的。源码如下： 
```java
public ExecutorCompletionService(Executor executor) {
    if (executor == null)
        throw new NullPointerException();
    this.executor = executor;
    this.aes = (executor instanceof AbstractExecutorService) ?
            (AbstractExecutorService) executor : null;
    this.completionQueue = new LinkedBlockingQueue<Future<V>>();
}
```

除此之外，还提供了一个可传入阻塞队列的构造方法，源码不再贴出。 

# 转换任务类型方法newTaskFor() 

该方法类型为protected方法，外部方法不能调用。  
如果接收的executor是AbstractExecutorService的子类，那么直接调用它的newTaskFor()方法；如果不是，构造为FutureTask类，源码如下： 
```java
private RunnableFuture<V> newTaskFor(Callable<V> task) {

    if (aes == null)
        return new FutureTask<V>(task);
    else
        return aes.newTaskFor(task);
}
```

# submit()方法 

先将提交的任务callable对象或者runnable对象，转换为RunnableFuture对象，然后将RunnableFuture对象转换为QueueingFuture 对象，并将该对象作为execute()方法提交。  
**这里需要主要两点：**

  1. 必须提交QueueingFuture对象，因为只有该对象才会在执行完成的任务放到阻塞队列中
  2. 必须由execute()方法提交任务，不能使用submit()方法，因为如果使用submit()提交任务，任务的会被封装成FutureTask对象，就不在是QueueingFuture类型的任务了。  
源码如下：只展示一个submit()的源码，其他重载方法类似：
```java
public Future<V> submit(Callable<V> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<V> f = newTaskFor(task);
    executor.execute(new QueueingFuture(f));
    return f;
}
```

# 获取执行完成任务方法 

任务执行完成之后，就会将任务放到阻塞队列中，获取执行完成的任务就转换为去阻塞队列中获取元素的操作。  
其中take() poll() poll(timeout)都是直接调用阻塞队列的方法。源码如下： 
```java
public Future<V> take() throws InterruptedException {
    return completionQueue.take();
}

public Future<V> poll() {
    return completionQueue.poll();
}

public Future<V> poll(long timeout, TimeUnit unit)
        throws InterruptedException {
    return completionQueue.poll(timeout, unit);
}
```


