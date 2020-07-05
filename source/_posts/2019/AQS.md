---
title: 并发6:AQS
permalink: AQS
date: 2019-12-30 10:45:22
tags:
categories: concurrent
---
# 简介
> AQS 全拼为 `AbstractQueuedSynchronizer` , 即**队列同步器**. 它是构建锁或者其他同步组件的基础框架(如 ReentrantLock、ReentrantReadWriteLock、Semaphore 等). 它是 J.U.C 并发包中的核心基础组件。
> `JAVA API` 描述如下: 提供一个框架，用于实现依赖先进先出（FIFO）等待队列的阻塞锁和相关同步器（信号量，事件等）。 该类被设计为大多数类型的同步器的有用依据，这些同步器依赖于单个原子int值来表示状态。 子类必须定义改变此状态的受保护方法，以及根据该对象被获取或释放来定义该状态的含义。 给定这些，这个类中的其他方法执行所有排队和阻塞机制。 子类可以保持其他状态字段，但只以原子方式更新int使用方法操纵值getState() ， setState(int)和compareAndSetState(int, int)被跟踪相对于同步。

我们简单理解如下:


1. AQS 通过内置的 FIFO 同步队列来完成资源获取线程的排队工作：
    - 如果当前线程获取同步状态失败（锁）时，AQS 则会将当前线程以及等待状态等信息构造成一个节点（Node）并将其加入同步队列，同时会阻塞当前线程
    - 当同步状态释放时，则会把节点中的线程唤醒，使其再次尝试获取同步状态。
2. AQS 的主要使用方式是继承，子类通过继承AQS同步器，并实现它的抽象方法来管理同步状态。

3. AQS 使用一个 int 类型的成员变量 `state` 来表示同步状态：
    - 当 `state > 0` 时，表示已经获取了锁。
    - 当 `state = 0` 时，表示释放了锁。
4. 它提供了三个方法，来对同步状态 `state` 进行操作，并且 AQS 可以确保对 `state` 的操作是安全的：
    - getState()
    - setState(int newState)
    - compareAndSetState(int expect, int update)

<!--more-->

![](AQS/aqs流程图.png)

--- 
# CLH 同步列队

## AbstractQueuedSynchronizer
```
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    protected AbstractQueuedSynchronizer() { }
    // Node头节点
    private transient volatile Node head;

    // Node尾节点
    private transient volatile Node tail;

    // 同步状态 当 state > 0 时，表示已经获取了锁; 当 state = 0 时，表示释放了锁。
    private volatile int state;
}
```
注意此处`AbstractQueuedSynchronizer`抽象类的只有一个无参构造, 没有对任何常量进行赋值

## 内部静态类 Node
```
static final class Node {

    // 共享
    static final Node SHARED = new Node();
    // 独占
    static final Node EXCLUSIVE = null;
    /**
     * 因为超时或者中断，节点会被设置为取消状态，被取消的节点时不会参与到竞争中的，他会一直保持取消状态不会转变为其他状态
     */
    static final int CANCELLED =  1;
    /**
     * 后继节点的线程处于等待状态，而当前节点的线程如果释放了同步状态或者被取消，将会通知后继节点，使后继节点的线程得以运行
     */
    static final int SIGNAL    = -1;
    /**
     * 节点在等待队列中，节点线程等待在Condition上，当其他线程对Condition调用了signal()后，该节点将会从等待队列中转移到同步队列中，加入到同步状态的获取中
     */
    static final int CONDITION = -2;
    /**
     * 表示下一次共享式同步状态获取，将会无条件地传播下去
     */
    static final int PROPAGATE = -3;

    /** 等待状态 */
    volatile int waitStatus;

    /** 前驱节点，当节点添加到同步队列时被设置（尾部添加） */
    volatile Node prev;

    /** 后继节点 */
    volatile Node next;

    /** 等待队列中的后续节点。如果当前节点是共享的，那么字段将是一个 SHARED 常量，也就是说节点类型（独占和共享）和等待队列中的后续节点共用同一个字段 */
    Node nextWaiter;
    
    /** 获取同步状态的线程 */
    volatile Thread thread;

    final boolean isShared() {
        return nextWaiter == SHARED;
    }
    Node() {    // Used to establish initial head or SHARED marker
    }

    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```


## 整体模型
![](AQS/aqs内部结构.png)

**注意:** 
> Node 是链表结构, 此处的多个 Node 是存放在 `AbstractQueuedSynchronizer` 类的 `private transient volatile Node head` 中
> 1. `Node链表 = head.next`
> 2. **`head` 节点存放的是当前获取锁的线程, 对应的thread为空, 可以理解为第一个人正在办理,从第二个人开始才算排队**
> 3. `tail = Node链表最后一个 = head最后一个元素`

## 源码
简单想一下, 大致流程应该分为三步
1. tail 指向新节点
2. 新节点的 pre 指向老节点
3. 老节点的 next 指向新节点 

再考虑到并发情况, 使用类似 synchronized 之类的给这段代码加锁
### 入列
```
private Node addWaiter(Node mode) {
    // 新建节点 this.nextWaiter = mode; this.thread = thread;
    Node node = new Node(Thread.currentThread(), mode);
    // 记录原尾节点
    Node pred = tail;
    // 快速尝试，添加新节点为尾节点
    if (pred != null) {
        // 设置新 Node 节点的尾节点为原尾节点
        node.prev = pred;
        // CAS 设置新的尾节点
        if (compareAndSetTail(pred, node)) {
            // 成功，原尾节点的下一个节点为新节点
            pred.next = node;
            return node;
        }
    }
    // 失败，多次尝试，直到成功
    enq(node);
    return node;
}

private Node enq(final Node node) {
    // 多次尝试，直到成功为止
    for (;;) {
        // 记录原尾节点
        Node t = tail;
        // 原尾节点不存在，创建首尾节点都为 new Node()
        if (t == null) {
            if (compareAndSetHead(new Node()))
                tail = head;
        // 原尾节点存在，添加新节点为尾节点
        } else {
            //设置为尾节点
            node.prev = t;
            // CAS 设置新的尾节点
            if (compareAndSetTail(t, node)) {
                // 成功，原尾节点的下一个节点为新节点
                t.next = node;
                return t;
            }
        }
    }
}
```
注意 :
> 1. 在调用 `#enq(final Node node)` 方法时, 如果tail为null, 创建的是一个新的Node节点, 并且这个Node节点里面的字段值都没有初始化
> 2. 然后进行第二次循环, 这时 tail 已经不为Null, 再进行赋值

不太明白为什么要有快速尝试那一步,  `#enq(final Node node)` 方法的代码有着同样的功能, 可能是为了代码的可读性吧.

### 出列
```
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}
```
这个方法比较简单, 此处为单线程操作,不需加锁

---

# 同步状态的获取与释放
AQS 的设计模式采用的模板方法模式，子类通过继承的方式，实现它的抽象方法来管理同步状态。对于子类而言，它并没有太多的活要做，AQS 已经提供了大量的模板方法来实现同步，主要是分为三类：
- 独占式获取和释放同步状态
- 共享式获取和释放同步状态
- 查询同步队列中的等待线程情况。

## 类内部方法
AQS 主要提供了如下方法：

- `#getState()`：返回同步状态的当前值。
- `#setState(int newState)`：设置当前同步状态。
- `#compareAndSetState(int expect, int update)`：使用 CAS 设置当前状态，该方法能够保证状态设置的原子性。
- 【可重写】`#tryAcquire(int arg)`：独占式获取同步状态，获取同步状态成功后，其他线程需要等待该线程释放同步状态才能获取同步状态。
- 【可重写】`#tryRelease(int arg)`：独占式释放同步状态。
- 【可重写】`#tryAcquireShared(int arg)`：共享式获取同步状态，返回值大于等于 0 ，则表示获取成功；否则，获取失败。
- 【可重写】`#tryReleaseShared(int arg)`：共享式释放同步状态。
- 【可重写】`#isHeldExclusively()`：当前同步器是否在独占式模式下被线程占用，一般该方法表示是否被当前线程所独占。

- `acquire(int arg)`：独占式获取同步状态。如果当前线程获取同步状态成功，则由该方法返回；否则，将会进入同步队列等待。该方法将会调用可重写的 `#tryAcquire(int arg)` 方法；
- `#acquireInterruptibly(int arg)`：与 `#acquire(int arg)` 相同，但是该方法响应中断。当前线程为获取到同步状态而进入到同步队列中，如果当前线程被中断，则该方法会抛出InterruptedException 异常并返回。
- `#tryAcquireNanos(int arg, long nanos)`：超时获取同步状态。如果当前线程在 nanos 时间内没有获取到同步状态，那么将会返回 false ，已经获取则返回 true 。
- `#acquireShared(int arg)`：共享式获取同步状态，如果当前线程未获取到同步状态，将会进入同步队列等待，与独占式的主要区别是在同一时刻可以有多个线程获取到同步状态；
- `#acquireSharedInterruptibly(int arg)`：共享式获取同步状态，响应中断。
- `#tryAcquireSharedNanos(int arg, long nanosTimeout)`：共享式获取同步状态，增加超时限制。
- `#release(int arg)`：独占式释放同步状态，该方法会在释放同步状态之后，将同步队列中第一个节点包含的线程唤醒。
- `#releaseShared(int arg)`：共享式释放同步状态。

从上面的方法看下来，基本上可以分成 3 类：
- 独占式获取与释放同步状态
- 共享式获取与释放同步状态
- 查询同步队列中的等待线程情况



## 独占式
同一时刻，仅有一个线程持有同步状态。
### 独占式同步状态获取
> `#acquire(int arg)` 方法，为 AQS 提供的模板方法。该方法为独占式获取同步状态，但是该方法对中断不敏感。也就是说，由于线程获取同步状态失败而加入到 CLH 同步队列中，后续对该线程进行中断操作时，线程不会从 CLH 同步队列中移除。代码如下：
```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
可以拆分成下面这样看
```
public final void acquire(int arg) {
    // 首先尝试获取独占锁 (该方法需要自己实现), 尝试失败后执行下方代码
    if(!tryAcquire(arg)){
        // static final Node EXCLUSIVE = null;
        // 上方讲到的入列的动作
        Node node = addWaiter(Node.EXCLUSIVE);
        // 自旋直到获得同步状态成功 
        // 当返回 true 时，表示在这个过程中，发生过线程中断
        boolean flag = acquireQueued(node, arg);
        if(flag){
            // 恢复线程中断的标识
            selfInterrupt();
        }
    }
}

static void selfInterrupt() {
    Thread.currentThread().interrupt();
}
```

#### acquireQueued
该方法为一个自旋的过程，也就是说，当前线程（Node）进入同步队列后，就会进入一个自旋的过程，每个节点都会自省地观察，当条件满足，获取到同步状态后，就可以从这个自旋过程中退出，否则会一直执行下去。
![流程图](acquireQueued.png)
```
final boolean acquireQueued(final Node node, int arg) {
    // 记录是否获取同步状态成功
    boolean failed = true;
    try {
        // 记录过程中，是否发生线程中断
        boolean interrupted = false;
        /*
         * 自旋过程，其实就是一个死循环而已
         */
        for (;;) {
            // 当前线程的前驱节点
            final Node p = node.predecessor();
            // 前驱节点是头结点
            if (p == head) {
                // 尝试获取独占锁
                if(tryAcquire(arg)){
                    // 设置当前节点( 线程 )为新的 head
                    setHead(node);
                    // 设置老的头节点 p 不再指向下一个节点，让它自身更快的被 GC 。
                    p.next = null;
                    // 标记 failed = false ，表示获取同步状态成功
                    failed = false;
                    // 返回记录获取过程中，是否发生线程中断。
                    return interrupted;
                }
            }
            // 判断获取失败后，是否当前线程需要阻塞等待。
            if (shouldParkAfterFailedAcquire(p, node)){
               // 线程等待, 调用的是 LockSupport.park(this); 这个方法会让出CPU
                if(parkAndCheckInterrupt()){
                    interrupted = true;
                }
            }
        }
    } finally {
        // 获取同步状态发生异常，取消获取。（一般用不到， 可忽略）
        if (failed)
            cancelAcquire(node);
    }
}

private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
class Thread{
    public static boolean interrupted() {
        return currentThread().isInterrupted(true);
    }
}

````
#### shouldParkAfterFailedAcquire （是否需要park）
```
/**
 * 因为超时或者中断，节点会被设置为取消状态，被取消的节点时不会参与到竞争中的，他会一直保持取消状态不会转变为其他状态 (一般用不到，忽略)
 */
static final int CANCELLED =  1;
/**
 * 后继节点的线程处于等待状态，而当前节点的线程如果释放了同步状态或者被取消，将会通知后继节点，使后继节点的线程得以运行
 * 默认值为 0，第二次循环会变为-1，如果为-1 ，当前线程park。 可以使用status ！= 0 来判断是否是尾节点，（如果是尾结点，status = 0）
 */
static final int SIGNAL    = -1;
/**
 * 节点在等待队列中，节点线程等待在Condition上，当其他线程对Condition调用了signal()后，该节点将会从等待队列中转移到同步队列中，加入到同步状态的获取中
 */
static final int CONDITION = -2;
/**
 * 表示下一次共享式同步状态获取，将会无条件地传播下去
 */
static final int PROPAGATE = -3;
```
```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    // 获得前一个节点的等待状态
    int ws = pred.waitStatus;
    // 等待状态为 Node.SIGNAL 时，表示 pred 的下一个节点 node 的线程需要阻塞等待。在 pred 的线程释放同步状态时，会对 node 的线程进行唤醒通知。
    // 返回 true ，表明当前线程可以被 park，安全的阻塞等待。
    if (ws == Node.SIGNAL) 
        return true;
    if (ws > 0) {
        // 等待状态为 NODE.CANCELLED 时，则表明该线程的前一个节点已经等待超时或者被中断了，则需要从 CLH 队列中将该前一个节点删除掉，循环回溯，直到前一个节点状态 <= 0 
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        // 等待状态为 0 或者 Node.PROPAGATE 时，通过 CAS 设置，将状态修改为 Node.SIGNAL
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```
#### cancelAcquire（一般用不到， 可忽略）
```
private void cancelAcquire(Node node) {
    // 将节点的线程置空
    node.thread = null;

    // 前置节点
    Node pred = node.prev;
    // 前置节点超时或等待, 删除前置节点
    while (pred.waitStatus > 0){
        node.prev = pred = pred.prev;
    }
    // 由于存在并发情况, 我们需要将 pred.next 暂存起来
    Node predNext = pred.next;
    
    // 设置 node 节点的为取消的等待状态 Node.CANCELLED
    // 这里可以使用直接写，而不是 CAS
    // 在这个操作之后，其它 Node 节点可以忽略 node
    node.waitStatus = Node.CANCELLED;

    // 如果node为尾结点 并且--cas设置this的尾结点为缓存的前置节点, 期望值为node-- 成功
    if (node == tail && compareAndSetTail(node, pred)) {
        // cas设置缓存的前置节点的后置节点为 null, 期望值为predNext
        compareAndSetNext(pred, predNext, null);
    } else {
        // If successor needs signal, try to set pred's next-link
        // so it will get one. Otherwise wake it up to propagate.
        int ws;
        // 前置节点不为头结点
        // 并且 (前置节点的等待状态为 Node.SIGNAL 或者 (前置节点的等待状态 <0 并且 尝试将前置节点的等待状态修改为 Node.SIGNAL 成功 并且 前置节点的线程不为null)
        if (pred != head && 
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                // CAS 设置 pred 的下一个节点为 next
                compareAndSetNext(pred, predNext, next);
        } else {
            // 唤醒 node 的下一个节点的线程等待
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
private final boolean compareAndSetTail(Node expect, Node update) {
    // 变更 this的尾结点 为 update ,期望变更为 expect
    return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
}
private static final boolean compareAndSetNext(Node node, Node expect, Node update) {
    // 变更 node 的后置节点 为 update ,期望变更为 expect
    return unsafe.compareAndSwapObject(node, nextOffset, expect, update);
}
```
### 独占式获取响应中断
AQS 提供了`acquire(int arg)` 方法，以供独占式获取同步状态，但是该方法对中断不响应，对线程进行中断操作后，该线程会依然位于CLH同步队列中，等待着获取同步状态。为了响应中断，AQS 提供了` #acquireInterruptibly(int arg)` 方法。该方法在等待获取同步状态时，如果当前线程被中断了，会立刻响应中断，并抛出 `InterruptedException` 异常。



**大部分摘抄自**: 
1. [芋道源码](http://www.iocoder.cn/categories/JUC/?vip)
2. [小明哥](http://cmsblogs.com/?p=2197)
3. [JUC AQS ReentrantLock源码分析](https://blog.csdn.net/java_lyvee/article/details/98966684)