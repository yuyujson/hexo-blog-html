---
title: 并发9:Condition源码解析
permalink: condition-yuan-ma-jie-xi
date: 2020-02-04 16:35:39
tags:
categories: concurrent
---

# 前言
> `Condition` 接口的实现类有两个, 本篇只介绍 `AbstractQueuedSynchronizer` 中的实现 `ConditionObject`

通过上一篇`Condition介绍及使用`我们看到在创建`Condition`对象时调用的是'newCondition()', 并且 `Condition`只能在`lock`锁内使用, 我们可以大胆猜想: 
1. `Condition` 和`AQS`相似, 内部都是一个链表结构
2. 在调用`#await`方法时会将当前线程从`AQS`列队中取出, 加入到`Condition`的列队中
3. 在调用`#signal`方法时, 会将线程从`Condition`中取出第一个, 加入到`AQS`列队中
4. 在调用`#signalAll`方法时, 会将线程从`Condition`中取出全部线程, 加入到`AQS`列队中
5. 在插入`AQS`列队前可能会依据公平锁or非公平锁来决定是否竞争锁后插入 `[Doug Lea 说我这条猜错了]`

<!--more-->

# 类结构
```
interface Condition {
    void await() throws InterruptedException;
    void signal();
    void signalAll();
}

interface Lock {
    Condition newCondition();
}

class ReentrantLock implements Lock{
    public Condition newCondition() {
        return sync.newCondition();
    }
    abstract static class Sync extends AbstractQueuedSynchronizer {
        final ConditionObject newCondition() {
            return new ConditionObject();
        }
    }

}

abstract class AbstractQueuedSynchronizer{
    public class ConditionObject implements Condition, java.io.Serializable {
        private transient Node firstWaiter;
        private transient Node lastWaiter;
        public ConditionObject() { }
    }
    static final class Node {
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
    }
}
```
从类结构上来看我们没有猜错, `ConditionObject` 内部构成的也是`Node`对象,证明猜想1 是对的!

> 有一个小细节, 这里的`Node`对象只加了`transient` 反序列化, 而没有加`volatile`. 这是因为`Condition`是在lock内部使用的, 也就是`#lock`和`#unLock`之间, 在这里运行的程序因为加了锁,所以都是单线程的

# 源码
## conditionObject.await()
> 将数据添加至`Condition`等待列队中,并释放当前线程,当线程被唤醒后从`※`位置开始抢占锁继续向下执行
```
class ConditionObject implements Condition{
    public final void await() throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        // 1. 加入 Condition 列队, 并返回当前线程的node对象
        Node node = addConditionWaiter();
        // 2. 释放锁并将当前节点从AQS列队中移出,并返回当前线程重入次数state
        int savedState = fullyRelease(node);
        int interruptMode = 0;
        // 3. 循环判断此node是否在AQS列队中, 如果不在则park
        while (!isOnSyncQueue(node)) {
            LockSupport.park(this);
            // 线程中断相关,略过
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
        }
        // ※ 到这个位置说明当前线程已经被unpark了
        // acquireQueued 自旋获取锁并返回线程是否中断(AQS有讲过)
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        // 移除被取消的节点
        if (node.nextWaiter != null) // clean up if cancelled
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            // 如果线程已经被中断，则根据之前获取的interruptMode的值来判断是继续中断还是抛出异常
            reportInterruptAfterWait(interruptMode);
    }
}
```
### addConditionWaiter()
> 加入 `Condition` 列队, 并返回当前线程的node对象, **注意: 在`Condition`不为空时添加的是`nextWaiter`字段, 而`AQS`使用的是`pre`和`next`**
```
class ConditionObject implements Condition{
    private Node addConditionWaiter() {
        // Condition 列队最后一个节点
        Node t = lastWaiter;
        // 如果Condition 列队最后一个被取消则遍历删除取消的节点
        if (t != null && t.waitStatus != Node.CONDITION) {
            unlinkCancelledWaiters();
            t = lastWaiter;
        }
        // 将当前线程构造出一个node , 等待标识为 -2 (等待)
        Node node = new Node(Thread.currentThread(), Node.CONDITION);
        // 列队为空则直接赋值
        if (t == null)
            firstWaiter = node;
        else
            t.nextWaiter = node;
        lastWaiter = node;
        // 返回当前线程node
        return node;
    }
}
```

### fullyRelease(node)
> 释放锁,并返回当前线程重入次数的status
```
class ConditionObject implements Condition{
    final int fullyRelease(Node node) {
        // 解锁是否成功的标识
        boolean failed = true;
        try {
            // 2.1 这里是AQS的重入次数的标识
            int savedState = getState();
            // 解锁,AQS方法, 前面讲过
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                // 如果解锁失败了, 直接修改状态为 1
                node.waitStatus = Node.CANCELLED;
        }
    }
}
```
#### getState()
> 这里是AQS的重入次数的标识
```
abstract class AbstractQueuedSynchronizer{
    protected final int getState() {
        return state;
    }
}
```

### isOnSyncQueue(node)
> 判断该node是否在AQS列队中
```
abstract class AbstractQueuedSynchronizer{
    final boolean isOnSyncQueue(Node node) {
        // 1. node.waitStatus == Node.CONDITION
        // 2. node.prev == null
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        // 3. node.next != null
        if (node.next != null) // If has successor, it must be on queue
            return true;
        
         /*
         * node.prev can be non-null, but not yet on queue because
         * the CAS to place it on queue can fail. So we have to
         * traverse from tail to make sure it actually made it.  It
         * will always be near the tail in calls to this method, and
         * unless the CAS failed (which is unlikely), it will be
         * there, so we hardly ever traverse much.
         */
         // 遍历 AQS 列队, 确定是否在列队中 
        return findNodeFromTail(node);
    }
}
```
1. 节点等待状态为等待,那它一定在`Condition`列队中
2. 使用到`pre`字段的只有`AQS`,`AQS`除了代表当前获取锁线程的头结点,其他节点的`pre`字段都是有值的, 而`AQS`的首个并不属于排队的节点; 所以它在`Condition`列队中
3. 使用到`next`字段的只有`AQS`, `next != null`说明在`AQS`列队中

#### findNodeFromTail(Node node)
```
/**
 * Returns true if node is on sync queue by searching backwards from tail.
 * 如果节点在同步队列上，则通过从尾部向后搜索返回true
 * Called only when needed by isOnSyncQueue.
 * 仅供isOnSyncQueue方法调用
 * @return true if present
 */
private boolean findNodeFromTail(Node node) {
    Node t = tail;
    for (;;) {
        if (t == node)
            return true;
        if (t == null)
            return false;
        t = t.prev;
    }
}
```
在将`node`追加到`AQS列队尾端时, 是先`node.prev = tail`,然后进行cas操作将`node`节点追加入列队的.在运行此处代码时可能在cas自旋中. 所以需要从尾部进行遍历

> 源码注释上写道: node.prev是非空的，但还没放在队列上，因为将它放在队列上的cas可能会失败。所以我们必须从尾部开始遍历以确保它确实成功了。在对这个方法的调用中，它总是在尾部附近，除非CAS失败(这是不太可能的)，否则它将在那里，因此我们几乎不会遍历太多。(软件翻译的,理解就好)

## conditionObject.signal()

```
public final void signal() {
    // 判断当前线程是否获取锁,没有获取就抛出异常(方法很简单,不贴了)
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    // Condition 中首个等待的节点
    Node first = firstWaiter;
    if (first != null)
        // 实际唤醒的方法
        doSignal(first);
}
```

### doSignal(first)
```
private void doSignal(Node first) {
    do {
        // 先把firstWaiter的给改为下一个等待的,如果下一个为null,那么把lastWaiter也改为null
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        // 1. 将需要进行操作的node单独拎出来
        first.nextWaiter = null;
        // 循环遍历整个Condition列队
        // 2. transferForSignal(first) 转移节点到AQS列队,成功true,失败false
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
            
}
```

1. 注意上方已经将 firstWaiter 的值给修改了, 所以first和firstWaiter指向的位置是不同的, 修改first的时候firstWaiter不会改变
2. `#transferForSignal`
    - 如果成功了, while的条件为`false && ? = false` ;打断循环
    - 如果失败了, while的条件为`true&& ?`
        - `Condition` 还有等待数据,`true&& true`; 执行下一个等待数据
        - `Condition` 没有等待数据,`true&& false`; 打断循环

#### transferForSignal(first)
```
abstract class AbstractQueuedSynchronizer{
    final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
         // 1. cas将等待状态改为0, 如果修改失败说明这个节点被取消了,返回false
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;
    
        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
         // 插入AQS列队,并返回当前节点在AQS中的前置节点
        Node p = enq(node);
        int ws = p.waitStatus;
        // ws > 0 前置节点被取消
        // 2. compareAndSetWaitStatus(p, ws, Node.SIGNAL) cas前置节点从当前状态改为等待
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
}
```
1. `Condition` 中的waitstatus有两种状态, 等待or取消
    - 等待:修改等待状态为0后继续执行
    - 取消:`doSignal(first)`方法在调用前已经把当前节点删除了,返回false后while循环会继续执行下去,这时取消的节点已经不在`Condition`列队中了
2. 如果这时前置节点取消了, 就不会执行后面的cas操作


## conditionObject.signalAll()
```
public final void signalAll() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        // 这里和signal()不同, 其他一致
        doSignalAll(first);
}
```
### doSignalAll(Node first)
```
private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);
        first = next;
    } while (first != null);
}
```
直接循环, `Condition`中数据全部转移到`AQS`列队中

# 总结
1. `AQS`的列队是双向链表,并且第一个节点的`thread`为`null`为代表获取锁的当前线程;而`Condition`的列队是单向的一个链表,
2. `Condition`接口的`#await`,`#signal`,`#signalAll`三个方法必须在`lock`内执行
3. `Condition`中的列队被唤醒后是有序的加入到`AQS`列队中的
4. 一个`lock`可以维护多个`Condition`列队, 我们可以让指定列队停止或运行