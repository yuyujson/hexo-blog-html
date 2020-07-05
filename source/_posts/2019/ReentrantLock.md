---
title: 并发7:ReentrantLock
permalink: ReentrantLock
date: 2020-02-02 22:00:00
tags:
categories: concurrent
---

> `ReentrantLock` 依赖 AQS 实现，请先理解AQS

# 介绍
## 小 DEMO

```
ReentrantLock lock = new ReentrantLock();

public void demo() {
    lock.lock();
    try {
        doSomething();
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        lock.unlock();
    }
}
```
如上为`ReentrantLock`的简单使用, 线程进来时进行加锁, 运行方法, 解锁, 十分简洁

<!--more-->

## 猜想
### 初步方案
```
volatile int status=0;//标识---是否有线程在同步块-----是否有线程上锁成功
void lock(){
	while(!compareAndSet(0,1)){
	    sleep(10);
	}
	//lock
}
void unlock(){
	status=0;
}
boolean compareAndSet(int except,int newValue){
	//cas操作,修改status成功则返回true
}
```
缺点: 
1. 睡眠多久合适?
2. 如果并发高, 每一个都睡眠后做比对,很占用 CPU
3. 不能实现公平锁

### 完美方案
```
volatile int status=0;//标识---是否有线程在同步块-----是否有线程上锁成功
void lock(){
	while(!compareAndSet(0,1)){
	    // 将该线程加入等待列队
	    addAQS(Thread.currentThread());
	    // 将该线程睡眠让出CPU
	    park();
	}
	//lock
}
void unlock(){
	status=0;
	// 解锁时从列队中取出一个, 然后让其运行
	Thread t = takeAQS();
	unPark(t);
}
boolean compareAndSet(int except,int newValue){
	//cas操作,修改status成功则返回true
}
```
那么问题来了, 如果将指定线程阻塞与运行呢? `LockSupport`类解决了这个问题

### LockSupport
```
public class LockSupport {
    private LockSupport() {}
    
    public static void park() {
        UNSAFE.park(false, 0L);
    }
    public static void unpark(Thread thread) {
        if (thread != null)
            UNSAFE.unpark(thread);
    } 
}
```
这个类中的方法是直接调用 `UNSAFE` 类的方法, 都是 `native` 方法; 调用的是c或c++函数. 我们理解如何使用就好
```
// 当前线程让出CPU
LockSupport.park();
// 指定线程t运行
LockSupport.unPark(t);
```

# 底层实现
## 类结构
```
class ReentrantLock{

    // 内部抽象类 继承 AQS
    abstract static class Sync extends AbstractQueuedSynchronizer {
    }
    
    // 内部类-公平锁
    static final class FairSync extends Sync {
    }
    
    // 内部类-非公平锁
    static final class NonfairSync extends Sync {
    }
}
```
## 举例
我们依据排队买票举例: 现有abcd四位客人来买票如下

`窗口 : a < b < c < d`

此时a正在办理那么排队的人中有bcd, 可以理解为
```
办理中: a
排队: null < b < c < d
```

### 创建对象
`ReentrantLock lock = new ReentrantLock();`

```
public ReentrantLock() {
    // 默认创建非公平锁
    sync = new NonfairSync();
}

// 传入boolean值来创建 
// true: 公平锁 false: 非公平锁
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

### 加锁
`lock.lock();`

```
public void lock() {
    sync.lock();
}
```
此处调用的是抽象类的lock方法, 让我们来看下它们有什么区别

```
// 公平锁
FairSync extends Sync {
    final void lock() {
        // 排队
        acquire(1);
    }
}

// 非公平锁
NonfairSync extends Sync {
    final void lock() {
        // 尝试获取锁
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            // 获取不到锁就排队
            acquire(1);
    }
}
```
**可以看到, 公平锁和非公平锁的第一个区别是在非公平锁调用时先去尝试获取下锁**

举例: 

`窗口 : a < b < c < d`

```
办理中: a
排队: null < b < c < d
```

此时来个新乘客f
```
公平锁: 
    办理中: a
    排队: null < b < c < d < e
非公平锁:
    a正好办理完了, f会和b争夺锁, 如果f抢到了, 那么f直接执行
        办理中: f
        排队: null < b < c < d < e
    a没办理完
        办理中: a
        排队: null < b < c < d < e
```

### tryAcquire(arg)

```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
关于 `#acquire` 方法在 AQS中已经讲过了, 该方法中调用了 `抽象方法#tryAcquire` , 而 `ReentrantLock` 对该方法编写了两套不同的实现

#### 公平锁

```
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    // 获取当前加锁状态
    int c = getState();
    // 未加锁
    if (c == 0) {
        // hasQueuedPredecessors() 判断自己是否需要排队 [后面单独说]
        // compareAndSetState() 如果不需要排队那么尝试竞争锁
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            // setExclusiveOwnerThread() 设置当前线程为拥有锁的线程，方面后面判断是不是重入
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 加锁的话需要判断重入, 如果该线程等于正在执行线程, 则state + 1
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
#### 非公平锁
```
protected final boolean tryAcquire(int acquires) {
    // 调用父类 sync 的方法
    return nonfairTryAcquire(acquires);
}


final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

### 公平锁与非公平锁的区别
比较后发现, 公平锁比非公平锁在未加锁状态下多了判断是否需要排队, 其他代码一模一样

至此`FairSync` 和 `NonfairSync` 进行实现的方法已经比较完毕, 我们可以得出结论:
1. 非公平锁在新的线程进来时直接去抢占锁, 抢占不到再排队
2. 公平锁进来后直接进行排队获取


### hasQueuedPredecessors()
```
public final boolean hasQueuedPredecessors() {
    // 尾结点
    Node t = tail;
    // 头结点
    Node h = head;
    Node s;
    // h.next 为队列首个对象, 如果这个
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

1. **h != t**
    - 列队为空的情况下h=null t=null
    - 列队不为null并且没有排队的情况下, 头结点和尾结点指向同一个对象, 对象中的thread为空
2. **((s = h.next) == null || s.thread != Thread.currentThread())**
    - 如果头结点!=尾结点, 那么判断列队中第一个线程是否是当前线程, 如果是当前线程, 则认为没有排队

### 解锁
```
class ReentrantLock {
    private final Sync sync;
    public void unlock() {
        // 1. 调用 sync 父类 AQS 的方法
        sync.release(1);
    }
}
```

```
abstract static class Sync extends AbstractQueuedSynchronizer {
    // 4. 该方法如果解锁成功会返回true, 否则返回false
    protected final boolean tryRelease(int releases) {
        // 对状态值进行 -1 操作
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        // c == 0 解锁成功
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        // 重入锁修改 state
        setState(c);
        return free;
    }
}
```

```
abstract class AbstractQueuedSynchronizer {
    // 2.
    public final boolean release(int arg) {
        // 3. tryRelease(arg) 调用 Sync 的方法 返回true表示解锁成功
        if (tryRelease(arg)) {
            Node h = head;
            // h.waitStatus != 0 代表后面有人排队
            if (h != null && h.waitStatus != 0)
                // 唤醒下一个线程
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
    
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            // 头结点 waitStatus 改为0
            compareAndSetWaitStatus(node, ws, 0);
    
        // 获取下一个线程
        Node s = node.next;
        // 如果下一个线程为空或者标识为取消
        if (s == null || s.waitStatus > 0) {
            // 置空
            s = null;
            // 从尾结点向前进行循环, 如果循环到的值为null 或者 循环到 node 的位置 则停止
            for (Node t = tail; t != null && t != node; t = t.prev)
                // 此操作是为了从后向前找出第一个可用的结点
                if (t.waitStatus <= 0)
                    s = t;
        }
        // 如果存在下一个线程则进行unpark操作
        if (s != null)
            LockSupport.unpark(s.thread);
    }
}
```