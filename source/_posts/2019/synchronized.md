---
title: 并发2:Synchronized
permalink: Synchronized
date: 2019-12-17 11:45:44
tags: 
categories: concurrent
---

## 锁机制有如下两种特性：
1. 互斥性：即在同一时间只允许一个线程持有某个对象锁，通过这种特性来实现多线程中的协调机制，这样在同一时间只有一个线程对需同步的代码块(复合操作)进行访问。互斥性我们也往往称为操作的原子性。
2. 可见性：必须确保在锁被释放之前，对共享变量所做的修改，对于随后获得该锁的另一个线程是可见的（即在获得锁时应获得最新共享变量的值），否则另一个线程可能是在本地缓存的某个副本上继续操作从而引起不一致。

## Synchronized 的用法
<table><tr><td >分类</td><td >具体分类</td><td >被锁对象</td><td >举例</td></tr><tr><td rowspan="2" >方法</td><td >方法</td><td >当前类的实例对象</td><td > public Synchronized void test() { }</td></tr><tr><td >静态方法</td><td >类对象</td><td > public static Synchronized void test() { }</td></tr><tr><td rowspan="3" colspan="2" >代码块</td><td >当前类的实例对象</td><td > public void test() { Synchronized (this){ } }</td></tr><tr><td >class对象</td><td > public void test() { Synchronized (Test.getClass()){ } }</td></tr><tr><td >任意实例对象</td><td > private Object lock = new Object(); public void test() { Synchronized (lock){ } }</td></tr></table>


<!--more-->

## 锁对象的分类
1. 对象锁
> 如果对class文件进行反编译可以看到,对象在执行时首先要先执行monitorenter指令，退出的时候monitorexit指令。

2. 类锁
> 在 Java 中，针对每个类也有一个锁，可以称为“类锁”，类锁实际上是通过对象锁实现的，即类的 Class 对象锁。每个类只有一个 Class 对象，所以每个类只有一个类锁。

---

## 同步代码块
### 对象锁执行流程
在对带有 ```Synchronized``` 锁的类的class文件进行反编译后可以得到带有下方关键字的流程
```
...
monitorenter  //进入同步方法
...
monitorexit   //退出同步方法
...
monitorexit // 退出同步方法(相当于finally)
...
```
其中 ```monitorenter``` 指令指向同步代码块的开始位置，```monitorexit``` 指令则指明同步代码块的结束位置, 当执行 ```monitorenter``` 指令时，当前线程将试图获取 **对象锁** 所对应的 monitor 的持有权，当 **对象锁** 的 monitor 的进入计数器为 0，那线程可以成功取得 monitor，并将计数器值设置为 1，取锁成功。如果当前线程已经拥有 **对象锁** 的 monitor 的持有权，那它可以重入这个 monitor，重入时计数器的值也会加 1。倘若其他线程已经拥有 **对象锁** 的 monitor 的所有权，那当前线程将被阻塞，直到正在执行线程执行完毕，即 ```monitorexit```指令被执行，执行线程将释放 monitor(锁)并设置计数器值为0 ，其他线程将有机会持有 monitor 。

值得注意的是编译器将会确保无论方法通过何种方式完成，方法中调用过的每条 ```monitorenter``` 指令都有执行其对应 ```monitorexit``` 指令，而无论这个方法是正常结束还是异常结束。为了保证在方法异常完成时 ```monitorenter``` 和 ```monitorexit``` 指令依然可以正确配对执行，编译器会自动产生一个异常处理器，这个异常处理器声明可处理所有的异常，它的目的就是用来执行 ```monitorexit``` 指令。从字节码中也可以看出多了一个```monitorexit```指令，它就是异常结束时被执行的释放monitor 的指令。

锁的重入性: 即在同一锁程中，线程不需要再次获取同一把锁。```Synchronized```先天具有重入性。每个对象拥有一个计数器，当线程获取该对象锁后，计数器就会加一，释放锁后就会将计数器减一。


### 对象在堆内存的结构
在JVM中，对象在内存中的布局分为三块区域：对象头、实例数据和填充数据
1. 实例变量：存放类的属性数据信息，包括父类的属性信息，如果是数组的实例部分还包括数组的长度，这部分内存按4字节对齐。
2. 填充数据：由于虚拟机要求对象起始地址必须是8字节的整数倍。填充数据不是必须存在的，仅仅是为了字节对齐，这点了解即可。
3. 对象头,采用2个字来存储对象头(如果对象是数组则会分配3个字，多出来的1个字记录的是数组长度)，其主要结构是由Mark Word 和 Class Metadata Address 组成
   
### 对象头
它实现```Synchronized```的锁对象的基础，这点我们重点分析它，一般而言，```Synchronized```使用的锁对象是存储在Java对象头里的，jvm中采用2个字来存储对象头(如果对象是数组则会分配3个字，多出来的1个字记录的是数组长度)，其主要结构是由Mark Word 和 Class Metadata Address 组成，其结构说明如下表：

虚拟机位数 | 头对象结构 | 说明
:-: | :-: | :-:
32/64bit | Mark Word | 存储对象的hashCode、锁信息或分代年龄或GC标志等信息
32/64bit | Class Metadata Address | 类型指针指向对象的类元数据，JVM通过这个指针确定该对象是哪个类的实例。

其中Mark Word在默认情况下存储着对象的HashCode、分代年龄、锁标记位等以下是32位JVM的Mark Word默认存储结构

锁状态 | 25bit | 4bit | 1bit是否是偏向锁 | 2bit 锁标志位
:-: | :-: | :-: | :-: | :-:
无锁状态 | 对象HashCode | 对象分代年龄 | 0 | 01

由于对象头的信息是与对象自身定义的数据没有关系的额外存储成本，因此考虑到JVM的空间效率，Mark Word 被设计成为一个非固定的数据结构，以便存储更多有效的数据，它会根据对象本身的状态复用自己的存储空间，如32位JVM下，除了上述列出的Mark Word默认存储结构外，还有如下可能变化的结构：


<table><tr><td rowspan="2">锁状态</td><td colspan="2" >25bit</td><td rowspan="2" >4bit</td><td >1bit</td><td >2bit</td></tr><tr><td >23bit</td><td >2bit</td><td >是否偏向锁</td><td >锁标志位</td></tr><tr><td >无锁状态</td><td colspan="2" >对象的hashCode</td><td >对象分代年龄</td><td >0</td><td >01</td></tr><tr><td >轻量级锁</td><td colspan="4" >指向栈中锁记录的指针</td><td >00</td></tr><tr><td >重量级锁</td><td colspan="4" >指向互斥量(重量级锁)的指针</td><td >10</td></tr><tr><td >GC标记</td><td colspan="4" >空</td><td >11</td></tr><tr></tr></table>

---

## 同步方法
### 同步方法执行流程
在对带有 ```Synchronized``` 锁方法的class文件进行反编译后可以得到带有下方关键字
```
  //==================syncTask方法======================
  public Synchronized void syncTask();
    descriptor: ()V
    //方法标识ACC_PUBLIC代表public修饰，ACC_SYNCHRONIZED`指明该方法为同步方法
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: dup
         2: getfield      #2                  // Field i:I
         5: iconst_1
         6: iadd
         7: putfield      #2                  // Field i:I
        10: return
      LineNumberTable:
        line 12: 0
        line 13: 10
}
```
方法级的同步是隐式，即无需通过字节码指令来控制的，它实现在方法调用和返回操作之中。JVM可以从方法常量池中的方法表结构(method_info Structure) 中的 ACC_SYNCHRONIZED 访问标志区分一个方法是否同步方法。当方法调用时，调用指令将会 检查方法的 ACC_```Synchronized``` 访问标志是否被设置，如果设置了，执行线程将先持有monitor（虚拟机规范中用的是管程一词）， 然后再执行方法，最后再方法完成(无论是正常完成还是非正常完成)时释放monitor。在方法执行期间，执行线程持有了monitor，其他任何线程都无法再获得同一个monitor。如果一个同步方法执行期间抛 出了异常，并且在方法内部无法处理此异常，那这个同步方法所持有的monitor将在异常抛到同步方法之外时自动释放

### 同步方法原理

从字节码中可以看出，```Synchronized```修饰的方法并没有monitorenter指令和monitorexit指令，取得代之的确实是ACC_```Synchronized```标识，该标识指明了该方法是一个同步方法，JVM通过该ACC_```Synchronized```访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。这便是```Synchronized```锁在同步代码块和同步方法上实现的基本原理。同时我们还必须注意到的是在Java早期版本中，```Synchronized```属于重量级锁，效率低下，因为监视器锁（monitor）是依赖于底层的操作系统的Mutex Lock来实现的，而操作系统实现线程之间的切换时需要从用户态转换到核心态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高，这也是为什么早期的```Synchronized```效率低的原因。不过Java 6之后Java官方对从JVM层面对```Synchronized```较大优化.

---

## 锁的升级
1. 无锁状态:没有加锁
2. 偏向锁：在对象第一次被某一线程占有的时候，是否偏向锁置1，锁表01，写入线程号，当其他的线程访问的时候->竞争->失败->轻量级锁
    1. 第一次占有它的线程获取几率会比较大  
    2. CAS算法 campany and set（CAS）
    3. 无锁状态时间非常接近
    4. 竞争不激烈的时候适用
3. 轻量级锁：线程有交替适用，互斥性不是很强, CAS失败, 00
4. 自旋锁：竞争失败的时候，不是马上转化级别，而是执行几次空循环5 10 
5. 重量级锁：强互斥, 等待时间长, 10
    1. 重量级锁也就是通常说```Synchronized```的对象锁，锁标识位为10，其中指针指向的是monitor对象（也称为管程或监视器锁）的起始地址。每个对象都存在着一个 monitor 与之关联，对象与其 monitor 之间的关系有存在多种实现方式，如monitor可以与对象一起创建销毁或当线程试图获取对象锁时自动生成，但当一个 monitor 被某个线程持有后，它便处于锁定状态。在Java虚拟机(HotSpot)中，monitor是由ObjectMonitor实现的，(位于HotSpot虚拟机源码ObjectMonitor.hpp文件，C++实现的）
6. 锁消除：JIT在编译的时候通过对运行上下文的扫描，去除不可能存在共享资源竞争的锁

---

## 使用注意事项
1. 与moniter关联的对象不能为空
2. 多个锁的交叉导致死锁

> 参考: [深入理解Java并发之 Synchronized 实现原理](https://blog.csdn.net/javazejian/article/details/72828483)









