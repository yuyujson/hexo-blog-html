---
title: jdk1.7-ConcurrentHashMap
permalink: jdk1-7-concurrent-hash-map
date: 2020-02-24 11:49:00
tags: jdk
categories: concurrent
---

# 前言

> `ConcurrentHashMap`是一种支持多线程的`hashmap`, 相对于`hashTable`它使用了分段锁的方式加锁, 大大提升了并发的性能

## 数据结构
![](jdk1-7-ConcurrentHashMap/1.7-currentHashMap.png)

`ConcurrentHashMap`与`hashmap`有所不同的是最初初始化的数组的每一个节点中存放的都是n个数组, 这些数组中存放的才是最终存放数据的链表.

<!--more-->

## UNSAFE

`UNSAFE`是Java中一个底层类，包含了很多基础的操作，比如数组操作、对象操作、内存操作、CAS操作、线程(park)操作、栅栏（Fence）操作，JUC包、一些三方框架都使用Unsafe类来保证并发安全。

[转载自:UNSAFE](https://www.cnblogs.com/throwable/p/9139947.html)

### arrayBaseOffset

```java
public native int arrayBaseOffset(Class arrayClass);
```

> 返回数组类型的第一个元素的偏移地址(基础偏移地址)。如果`arrayIndexScale`方法返回的比例因子不为0，你可以通过结合基础偏移地址和比例因子访问数组的所有元素。Unsafe中已经初始化了很多类似的常量如ARRAY_BOOLEAN_BASE_OFFSET等。

### arrayIndexScale

```java
public native int arrayIndexScale(Class arrayClass);
```

> 返回数组类型的比例因子(其实就是数据中元素偏移地址的增量，因为数组中的元素的地址是连续的)。此方法不适用于数组类型为"narrow"类型的数组，"narrow"类型的数组类型使用此方法会返回0(这里narrow应该是狭义的意思，但是具体指哪些类型暂时不明确，笔者查了很多资料也没找到结果)。Unsafe中已经初始化了很多类似的常量如ARRAY_BOOLEAN_INDEX_SCALE等。

### objectFieldOffset

```java
public native int objectFieldOffset(Class arrayClass);
```

> 返回给定的**非静态属性**在它的类的存储分配中的位置(偏移地址)。不要在这个偏移量上执行任何类型的算术运算，它只是一个被传递给不安全的堆内存访问器的cookie。注意：这个方法仅仅针对非静态属性，使用在静态属性上会抛异常。

### putObjectVolatile

```java
/**
* obj: 包含需要修改field的对象
* offset: obj中long型field的偏移量
* value: field将被设置的新值
*/
public native void putOrderedObject(Object obj, long offset, Object value);
```

> 此方法和`putObject`功能类似，不过附加了'volatile'加载语义，也就是设置值的时候强制(JMM会保证获得锁到释放锁之间所有对象的状态更新都会在锁被释放之后)更新到主存，从而保证这些变更对其他线程是可见的。类似的方法有putIntVolatile、putDoubleVolatile等等。这个方法要求被使用的属性被volatile修饰，否则功能和`putObject`方法相同。

### putOrderedObject

```java
/**
* obj: 包含需要修改field的对象
* offset: obj中long型field的偏移量
* value: field将被设置的新值
*/
public native void putOrderedObject(Object obj, long offset, Object value);
```

> 设置o对象中offset偏移地址offset对应的Object型field的值为指定值x。这是一个有序或者有延迟的putObjectVolatile方法，并且不保证值的改变被其他线程立即看到。只有在field被volatile修饰并且期望被修改的时候使用才会生效。类似的方法有`putOrderedInt`和`putOrderedLong`。

### putIntVolatile

```java
/**
* obj: 包含需要修改field的对象
* offset: obj中long型field的偏移量
* value: field将被设置的新值
*/
public native void putIntVolatile(Object obj, long offset, int value);
```

> 设置obj对象中offset偏移地址对应的整型field的值为指定值。支持volatile store语义

### getObject

```java
/**
* obj: 包含需要去读取的field的对象
* offset: obj中long型field的偏移量
*/
public native Object getObject(Object obj, long offset);
```

> 通过给定的Java变量获取引用值。这里实际上是获取一个Java对象o中，获取偏移地址为offset的属性的值，此方法可以突破修饰符的抑制，也就是无视private、protected和default修饰符。类似的方法有getInt、getDouble等等。

### getObjectVolatile

```java
/**
* obj: 包含需要去读取的field的对象
* offset: obj中long型field的偏移量
*/
public native Object getObjectVolatile(Object obj, long offset);
```

> 此方法和`getObject`功能类似，不过附加了`volatile`加载语义，也就是强制从主存中获取属性值。类似的方法有getIntVolatile、getDoubleVolatile等等。这个方法要求被使用的属性被volatile修饰，否则功能和`getObject`方法相同。

### compareAndSwapObject

```java
 /***
   * @param obj 包含要修改field的对象 
   * @param offset <code>obj</code>中object型field的偏移量
   * @param expect 希望field中存在的值
   * @param update 如果期望值expect与field的当前值相同，设置filed的值为这个新值
   * @return true 如果field的值被更改
   */
  public native boolean compareAndSwapObject(Object obj, long offset,
                                             Object expect, Object update);
```

> 针对Object对象进行CAS操作。即是对应Java变量引用o，原子性地更新o中偏移地址为offset的属性的值为x，当且仅的偏移地址为offset的属性的当前值为expected才会更新成功返回true，否则返回false。
>
> - o：目标Java变量引用。
> - offset：目标Java变量中的目标属性的偏移地址。
> - expected：目标Java变量中的目标属性的期望的当前值。
> - x：目标Java变量中的目标属性的目标更新值。
>
> 类似的方法有`compareAndSwapInt`和`compareAndSwapLong`，在Jdk8中基于CAS扩展出来的方法有`getAndAddInt`、`getAndAddLong`、`getAndSetInt`、`getAndSetLong`、`getAndSetObject`，它们的作用都是：通过CAS设置新的值，返回旧的值。

# static

```java
// Unsafe mechanics
private static final sun.misc.Unsafe UNSAFE;
private static final long SBASE;
private static final int SSHIFT;
private static final long TBASE;
private static final int TSHIFT;
private static final long HASHSEED_OFFSET;
private static final long SEGSHIFT_OFFSET;
private static final long SEGMASK_OFFSET;
private static final long SEGMENTS_OFFSET;

static {
  int ss, ts;
  try {
    UNSAFE = sun.misc.Unsafe.getUnsafe();
    Class tc = HashEntry[].class;
    Class sc = Segment[].class;
    TBASE = UNSAFE.arrayBaseOffset(tc);
    SBASE = UNSAFE.arrayBaseOffset(sc);
    ts = UNSAFE.arrayIndexScale(tc);
    ss = UNSAFE.arrayIndexScale(sc);
    HASHSEED_OFFSET = UNSAFE.objectFieldOffset(
      ConcurrentHashMap.class.getDeclaredField("hashSeed"));
    SEGSHIFT_OFFSET = UNSAFE.objectFieldOffset(
      ConcurrentHashMap.class.getDeclaredField("segmentShift"));
    SEGMASK_OFFSET = UNSAFE.objectFieldOffset(
      ConcurrentHashMap.class.getDeclaredField("segmentMask"));
    SEGMENTS_OFFSET = UNSAFE.objectFieldOffset(
      ConcurrentHashMap.class.getDeclaredField("segments"));
  } catch (Exception e) {
    throw new Error(e);
  }
  if ((ss & (ss-1)) != 0 || (ts & (ts-1)) != 0)
    throw new Error("data type scale not a power of two");
  SSHIFT = 31 - Integer.numberOfLeadingZeros(ss);
  TSHIFT = 31 - Integer.numberOfLeadingZeros(ts);
}
```

> 一堆做初始化的东西, 这些是在构造之前运行的, 没太搞懂, 先贴出来

# 内部类结构

```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {
  // 实际存放数据的链表数组
  transient volatile HashEntry<K,V>[] table;
  // 元素的个数
  transient int count;
  // segment元素修改次数
  transient int modCount;
  // 扩容指标
  transient int threshold;
  // 负载因子
  final float loadFactor;
  Segment(float lf, int threshold, HashEntry<K,V>[] tab) {
    this.loadFactor = lf;
    this.threshold = threshold;
    this.table = tab;
  }
}
static final class HashEntry<K,V> {
  final int hash;
  final K key;
  volatile V value;
  volatile HashEntry<K,V> next;

  HashEntry(int hash, K key, V value, HashEntry<K,V> next) {
    this.hash = hash;
    this.key = key;
    this.value = value;
    this.next = next;
  }
}
```

> 这里先简单说明基础的类结构,有个概念
>
> segment[] ->segment->HashEntry<K,V>[] table->HashEntry

# ConcurrentHashMap

```java
// 默认数组长度大小
static final int DEFAULT_INITIAL_CAPACITY = 16;
// 负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
// 默认线程并发度，默认最大允许16个线程并发访问，条件是这16个线程在分别在16个segments中
static final int DEFAULT_CONCURRENCY_LEVEL = 16;
// segment个数限制
static final int MAX_SEGMENTS = 1 << 16;
public ConcurrentHashMap() {
  this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR, DEFAULT_CONCURRENCY_LEVEL);
}

public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
  if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
    throw new IllegalArgumentException();
  // 最大并发数限制
  if (concurrencyLevel > MAX_SEGMENTS)
    concurrencyLevel = MAX_SEGMENTS;
  // Find power-of-two sizes best matching arguments
  // 找到一个大于等于传入的concurrencyLevel的最小2^n数
  int sshift = 0;
  int ssize = 1;
  while (ssize < concurrencyLevel) {
    ++sshift;
    ssize <<= 1;
  }
  this.segmentShift = 32 - sshift;
  this.segmentMask = ssize - 1;
  if (initialCapacity > MAXIMUM_CAPACITY)
    initialCapacity = MAXIMUM_CAPACITY;
  // 计算每个segment中table的容量, 除之后是省略小数点的, 所以再加一位就好了
  int c = initialCapacity / ssize;
  if (c * ssize < initialCapacity)
    ++c;
  int cap = MIN_SEGMENT_TABLE_CAPACITY;
  while (cap < c)
    cap <<= 1;
  // create segments and segments[0]
  // 创建segments中的第一个segment数组,其余的segment延迟初始化
  // 注意 segment中放的是HashEntry数组
  Segment<K,V> s0 =
    new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                     (HashEntry<K,V>[])new HashEntry[cap]);
  // 创建segments
  Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
  // ss的第一个segment赋值为s0
  UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
  // 赋值到对象中
  this.segments = ss;
}

```

假设传入的 `concurrencyLevel = 15`

1. 计算外层数组`segments`的长度:`ssize`
   1. `while (ssize < concurrencyLevel)` :计算每个segment中table的容量, 因为除之后是省略小数点的,所以再加1就好了.在进行循环后得到的结果是 `sshift = 4 ` ,`ssize=2^4`
2. 计算内层数组`table`的长度:`cap`
   1. `c = initialCapacity / ssize` 初始化数组总长度除以外侧数组长度=内侧数组的长度
   2. 但是我们需要保证内侧数组长度*外侧数组长度>=传入的数组总长度, 所以如果小了就自加一次
   3. `while (cap < c)` 保证内侧数组的长度为2^n, 并且最小为2
3. 为什么要初始化第一个数组`s0`?
   1. 创建内侧数组的初始化参数都是相同的, 初始化第一个后, 以后每次创建只需要直接从第一个数组取值就好了

# put

```java
public V put(K key, V value) {
  Segment<K,V> s;
  // key不可为null
  if (value == null)
    throw new NullPointerException();
  // 求hash
  int hash = hash(key);
  int j = (hash >>> segmentShift) & segmentMask;
  // UNSAFE.getObject(segments, (j << SSHIFT) + SBASE): 依据hash获取到对应分段
  if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
       (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
    // 如果分段为null则进行初始化
    s = ensureSegment(j);
  return s.put(key, hash, value, false);
}
```

## ensureSegment

```java
@SuppressWarnings("unchecked")
private Segment<K,V> ensureSegment(int k) {
  final Segment<K,V>[] ss = this.segments;
  // 获取偏移量
  long u = (k << SSHIFT) + SBASE; // raw offset
  Segment<K,V> seg;
  // 验证当前segment是否为null
  if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
    Segment<K,V> proto = ss[0]; // use segment 0 as prototype
    // 获取数组第一个位置的内部数组长度及扩容大小等
    int cap = proto.table.length;
    float lf = proto.loadFactor;
    int threshold = (int)(cap * lf);
    // 创建内部数组
    HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
    // 验证当前segment是否为null
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
        == null) { // recheck
      // 创建数组节点对象
      Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
      // 验证当前segment是否为null
      while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
             == null) {
       	// cas赋值
        if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
          break;
      }
    }
  }
  return seg;
}
```

## Segment.put

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
  // 尝试获取锁, 获取到 node=null, 没有获取到锁则调用scanAndLockForPut方法
  HashEntry<K,V> node = tryLock() ? null : scanAndLockForPut(key, hash, value);
  // 注意: 下方代码运行的前提是获取到当前segment的锁
  // 当前key对应的值
  V oldValue;
  try {
    // 内部数组
    HashEntry<K,V>[] tab = table;
    // 定位具体在哪个内部数组中
    int index = (tab.length - 1) & hash;
    HashEntry<K,V> first = entryAt(tab, index);
    // 遍历链表
    for (HashEntry<K,V> e = first;;) {
      // 如果不是链表最后一个的next就会走这个
      if (e != null) {
        K k;
        if ((k = e.key) == key ||
            (e.hash == hash && key.equals(k))) {
          // 如果相同则直接赋值
          oldValue = e.value;
          // 为putIfAbsent方法设置的
          if (!onlyIfAbsent) {
            e.value = value;
            ++modCount;
          }
          break;
        }
        e = e.next;
      }
      // 如果是链表最后一个的next会走这个
      else {
        // 只有走了 scanAndLockForPut 逻辑才不为null, 直接赋值
        if (node != null)
          // 头插法
          node.setNext(first);
        else
          // 新建HashEntry对象, 头插法
          node = new HashEntry<K,V>(hash, key, value, first);
        // 注意上方的插入操作并没有赋值到table[i]上
        // segment内n个链表的长度+1
        int c = count + 1;
        // 如果c达到扩容标准则进行扩容
        if (c > threshold && tab.length < MAXIMUM_CAPACITY)
          rehash(node);
        else
          // 赋值到table[i]
          setEntryAt(tab, index, node);
        ++modCount;
        count = c;
        oldValue = null;
        break;
      }
    }
  } finally {
    unlock();
  }
  return oldValue;
}
```

### scanAndLockForPut

> 如果没有获取到锁, 那么可以提前把hashenty准备好,尽管有可能不会用

```java
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
  // 依据hash获取需要插入的链表
  HashEntry<K,V> first = entryForHash(this, hash);
  HashEntry<K,V> e = first;
  HashEntry<K,V> node = null;
  // 用于控制流程的临时变量
  int retries = -1; // negative while locating node
  // 不断尝试获取锁
  while (!tryLock()) {
    // 如果没有获取到锁, 则走下面的流程
    HashEntry<K,V> f; // to recheck first below
    // 第一次进来会走这个,还有下方直接赋值后
    if (retries < 0) {
      if (e == null) {
        if (node == null) // speculatively create node
          // 如果都为空那个, 则创建HashEntry
          node = new HashEntry<K,V>(hash, key, value, null);
        retries = 0;
      }
      else if (key.equals(e.key))
        retries = 0;
      else
        e = e.next;
    }
    // 如果循环次数过多, 直接上锁, 直到获取到锁打断循环
    else if (++retries > MAX_SCAN_RETRIES) {
      lock();
      break;
    }
    // 如果创建node后, retries = 0, 这时发现当前链表已经被其他线程修改, 则回滚retries为出事状态, 回滚e
    else if ((retries & 1) == 0 &&
             (f = entryForHash(this, hash)) != first) {
      e = first = f; // re-traverse if entry changed
      retries = -1;
    }
  }
  return node;
}
```

### rehash

```java
private void rehash(HashEntry<K,V> node) {
  // 旧的链表数组
  HashEntry<K,V>[] oldTable = table;
  // 旧的链表数组长度
  int oldCapacity = oldTable.length;
  // 扩容为2倍
  int newCapacity = oldCapacity << 1;
  // 新的扩容阈值
  threshold = (int)(newCapacity * loadFactor);
  // 创建新的链表数组
  HashEntry<K,V>[] newTable =
    (HashEntry<K,V>[]) new HashEntry[newCapacity];
  int sizeMask = newCapacity - 1;
  // 循环遍历老的链表数组
  for (int i = 0; i < oldCapacity ; i++) {
    // 获取链表
    HashEntry<K,V> e = oldTable[i];
    if (e != null) {
      HashEntry<K,V> next = e.next;
      int idx = e.hash & sizeMask;
      if (next == null)   //  Single node on list
        // 如果链表中只有一个元素, 则直接赋值
        newTable[idx] = e;
      else { // Reuse consecutive sequence at same slot
        HashEntry<K,V> lastRun = e;
        int lastIdx = idx;
        // lastRun后面的所有node的 idx 与 lastRun的 idx 一致，这样我们就可以将lastRun及其后面的所有node看作一个整体
        for (HashEntry<K,V> last = next;
             last != null;
             last = last.next) {
          int k = last.hash & sizeMask;
          if (k != lastIdx) {
            lastIdx = k;
            lastRun = last;
          }
        }
        newTable[lastIdx] = lastRun;
        // 处理lastRun之前的数据
        for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
          V v = p.value;
          int h = p.hash;
          int k = h & sizeMask;
          HashEntry<K,V> n = newTable[k];
          newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
        }
      }
    }
  }
  int nodeIndex = node.hash & sizeMask; // add the new node
  node.setNext(newTable[nodeIndex]);
  newTable[nodeIndex] = node;
  table = newTable;
}
```

为什么链表中只有一个元素时可以直接赋值呢?为什么要引入一个lastRun机制呢?我们假设数组扩容前为最小值2

```java
int sizeMask = newCapacity - 1;
int idx = e.hash & sizeMask;
```

| newCapacity | sizeMask | sizeMask的二进制                        |
| ----------- | -------- | --------------------------------------- |
| 2           | 1        | 0000 0000 0000 0000 0000 0000 0000 0001 |
| 4           | 3        | 0000 0000 0000 0000 0000 0000 0000 0011 |
| 8           | 7        | 0000 0000 0000 0000 0000 0000 0000 0111 |
| 16          | 15       | 0000 0000 0000 0000 0000 0000 0000 1111 |
| 32          | 31       | 0000 0000 0000 0000 0000 0000 0001 1111 |

可以看到, 每次扩容后, hash进行运算时, 只向做移动了一位,由于`e.hash`不变,那么`e.hash & sizeMask`的结果只改变了最左侧的一位,所以之前节点的数据只会存在在两个位置中

# get

```java
public V get(Object key) {
  Segment<K,V> s; // manually integrate access methods to reduce overhead
  HashEntry<K,V>[] tab;
  // 获取hash值
  int h = hash(key);
  // 获取偏移量
  long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
  // 如果segments[i] != null && segment[i].table!= null
  if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
      (tab = s.table) != null) {
    // 依据hash从table中获取对应链表遍历
    for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
         (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
         e != null; e = e.next) {
      K k;
      // 如果key相同则返回
      if ((k = e.key) == key || (e.hash == h && key.equals(k)))
        return e.value;
    }
  }
  return null;
}
```

# 总结

1. `相对于`HashMap1.7`使用了分段锁的方法, 通过锁定`segment[i]`来提高效率
2. 在分段内增加了内部分段
3. 扩容不再扩容外部的`segments`,而是扩容内部的链表数组
4. 不支持`key = null`
5. 初始化时同时初始化了`segmens[0]`,为其他`segments[n]`的初始化提高了效率

