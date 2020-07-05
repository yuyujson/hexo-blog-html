---
title: ConcurrentHashMap
permalink: concurrent-hash-map
date: 2018-10-27 12:01:08
tags: JDK
categories: concurrent
---
# ConcurrentHashMap
## 优势
**HashMap**

在并发编程过程中使用可能导致死循环，因为插入过程不是原子操作，每个HashEntry是一个链表节点，很可能在插入的过程中，已经设置了后节点，实际还未插入，最终反而插入在后节点之后，造成链中出现环，破坏了链表的性质，失去了尾节点，出现死循环。

<!--more-->

**HashTable**

内部是采用synchronized来保证线程安全的，但在线程竞争激烈的情况下HashTable的效率下降得很快因为synchronized关键字会造成代码块或方法成为为临界区(对同一个对象加互斥锁)，当一个线程访问临界区的代码时，其他线程也访问同一临界区时，会进入阻塞或轮询状态。究其原因，实际上是有获取锁意向的线程的数目增加，但是锁还是只有单个，导致大量的线程处于轮询或阻塞，导致同一时间段有效执行的线程的增量远不及线程总体增量。 
> 在查询时，尤其能够体现出ConcurrentHashMap在效率上的优势，HashTable使用Sychronized关键字，会导致同时只能有一个查询在执行，而Cocurrent则不采取加锁的方法，而是采用volatile关键字，虽然也会牺牲效率，但是由于Sychronized，于该文末尾继续讨论。

**ConcurrentHashMap**

利用锁分段技术增加了锁的数目，从而使争夺同一把锁的线程的数目得到控制。 
> 锁分段技术就是对数据集进行分段，每段竞争一把锁，不同数据段的数据不存在锁竞争，从而有效提高 高并发访问效率。

ConcurrentHashMap在get方法是无需加锁的，因为用到的共享变量都采用volatile关键字修饰，巴证共享变量在线程之间的可见性(每次读取都先同步缓存和内存，直接从内存中获取值，虽然不是原子操作，但根据JAVA内存模型的happen before原则，对volatile字段的写入操作先于读操作，能够保证不会脏读),volatile为了让变量提供线程之间的内存可见性，会禁止程序执行结果的重排序（导致缓存优化的效果降低）

## 结构
- ConcurrentHashMap是由Segment数组和HashEntry数组组成。
- Segment是重入锁(ReentrantLock),作为一个数据段竞争锁，每个HashEntry一个链表结构的元素，利用Hash算法得到索引确定归属的数据段，也就是对应到在修改时需要竞争获取的锁。