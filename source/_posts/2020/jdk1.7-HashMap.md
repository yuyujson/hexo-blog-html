---
title: jdk1.7-HashMap
permalink: jdk1-7-hash-map
date: 2020-02-19 21:12:19
tags: jdk
categories: jdk
---

# 前言

> [jdk1.7下载链接](https://www.oracle.com/java/technologies/javase/javase7-archive-downloads.html#jdk-7u80-oth-JPR)
>
> [jdk1.8下载链接](https://www.oracle.com/java/technologies/javase/javase-jdk8-downloads.html)
>
> 以上为官网下载链接, 没有账号的同学建议搜索oracle账号共享,使用共享账号下载.
>
> 以mac为例,两个版本下载完成后续添加至idea
>
> 添加:  file->project structure->platorm setting->sdks->点击加号
>
> 选择项目使用的jdk: file->project structure->project setting->project->project sdk->选择需要的版本

<!--more-->

# Entry

```java
static class Entry<K,V> implements Map.Entry<K,V> {
  final K key;
  V value;
  Entry<K,V> next;
  int hash;

  // 创建一个新的节点, 将传入的接点挂在到新的节点下
  Entry(int h, K k, V v, Entry<K,V> n) {
    value = v;
    next = n;
    key = k;
    hash = h;
  } 
}
```

# newHashMap()

```java
// 默认的初始容量-必须是2的幂。
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

// 默认负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 一般为 capacity*loadFactory,如果数组容量大于threshold就会扩容
int threshold;

public HashMap() {
	this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
}
public HashMap(int initialCapacity, float loadFactor) {
  if (initialCapacity < 0)
    throw new IllegalArgumentException("Illegal initial capacity: " +
                                       initialCapacity);
  // 初始容量大于最大值,则设置为最大值
  if (initialCapacity > MAXIMUM_CAPACITY)
    initialCapacity = MAXIMUM_CAPACITY;
  if (loadFactor <= 0 || Float.isNaN(loadFactor))
    throw new IllegalArgumentException("Illegal load factor: " +
                                       loadFactor);

  this.loadFactor = loadFactor;
  threshold = initialCapacity;
  // 空的, linkedHashMap才有
  init();
}
```

# put(K key, V value)

```java
// 放置元素的数组, 大小为2的n次幂
transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;

public V put(K key, V value) {
  // 如果数组为空, 那么进行膨胀
  if (table == EMPTY_TABLE) {
    // 此时 threshold = initialCapacity = 1 << 4 = 16
    inflateTable(threshold);
  }
  // 如果key为null, 走一段特殊逻辑, 将它挂到数组的第一个下标位置
  if (key == null)
    return putForNullKey(value);
  // 获取hash值, 这个hash值和hash因子有关
  int hash = hash(key);
  // 获取数组下标位置
  int i = indexFor(hash, table.length);
  // 遍历数组查看里面有没有相同的key, 如果有则覆盖,并返回老的value
  for (Entry<K,V> e = table[i]; e != null; e = e.next) {
    Object k;
    if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
      V oldValue = e.value;
      e.value = value;
      e.recordAccess(this);
      return oldValue;
    }
  }
	// 快速失败机制
  modCount++;
  // 添加入链表
  addEntry(hash, key, value, i);
  return null;
}
```

## inflateTable(int toSize)

```java
// 扩容数组, 仅仅是扩容数组,并没有转移元素
private void inflateTable(int toSize) {
  // 获取大于等于 toSize 的最小2的n次幂
  int capacity = roundUpToPowerOf2(toSize);
	// 确定下次扩容时数组的大小
  threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
  // 初始化数组长度
  table = new Entry[capacity];
  // 初始化哈希掩码值
  initHashSeedAsNeeded(capacity);
}
```

### roundUpToPowerOf

```java
// 获取大于等于 toSize 的最小2的n次幂
private static int roundUpToPowerOf2(int number) {
  // assert number >= 0 : "number must be non-negative";
  // Integer.highestOneBit(i) 获取最大的小于等于i的 2的n次幂
  return number >= MAXIMUM_CAPACITY
    ? MAXIMUM_CAPACITY
    : (number > 1) ? Integer.highestOneBit((number - 1) << 1) : 1;
}
```

1. 如果`number >= 数组最大容量`: 返回`数组最大容量`
2. 如果`number <= 1`:返回`1`
3. `num - 1 后左移1位` 以这个数为参数取小于等于它的 2^n

这里的逻辑有点神奇, 把`number`简单的运算后,调用了一个返回小于等于n的2^n的方法就达到了目的, 我们来一点一点分析:

假设: n=16 (2^4)

```java
0000 1000//16
0000 0111//16-1
0000 1110//左移一位
小于等于这个数的最大2^n为
0000 1000
```

假设:n=15 (2^4 - 1)

```java
0000 0111//15
0000 0110//15-1
0000 1100//左移一位
小于等于这个数的最大2^n为
0000 1000
```

假设:n=9 (2^3 + 1)

```java
0000 0101//9
0000 0100//9-1
0000 1000//左移一位
小于等于这个数的最大2^n为
0000 1000
```

使用*来代替

```
0000 01**//n
0000 01**//n-1
0000 1**0//左移一位
```

上面的规律如果感觉比较空洞,可以尝试以数学的方式来证明:

```java
设最初传入参数为:m, 最终获取的值为:2^n
  
由上述条件可知: 
  number范围为:
    2^(n-1) < m <= 2^n
  我们传入的参数范围为:
    2^n <= f(m) < 2^(n+1)
      
解:
因为
  2^(n-1) < m <= 2^n
  且m和n都是大于1的正整数
所以 
	2^(n-1) <= m-1 < 2^n
左右公式同时乘2得
	2^n <= (m-1)*2 < 2^(n+1)
所以
  f(m) = 2(m-1)

```

### highestOneBit

```java
// 返回小于等于i的 2的n次幂
public static int highestOneBit(int i) {
  // HD, Figure 3-1
  i |= (i >>  1);
  i |= (i >>  2);
  i |= (i >>  4);
  i |= (i >>  8);
  i |= (i >> 16);
  return i - (i >>> 1);
}
```

分析: 

> int类型有4个字节, 每个字节8位, 共计32位**以下分析仅仅代表i为正数的情况下**

```java
假设i=1070,
0000 0000 0000 0000 0000 0100 0010 1110// i
0000 0000 0000 0000 0000 0010 0001 0111// i >> 1
0000 0000 0000 0000 0000 0110 0011 1111// i |= (i >>  1)
  
0000 0000 0000 0000 0000 0001 1000 1111// i >> 2
0000 0000 0000 0000 0000 0111 1011 1111// i |= (i >>  2)
  
0000 0000 0000 0000 0000 0000 0111 1011// i >>  4
0000 0000 0000 0000 0000 0111 1111 1111// i |= (i >>  4)
  
0000 0000 0000 0000 0000 0000 0000 0111// i >>  8
0000 0000 0000 0000 0000 0111 1111 1111// i |= (i >>  8)
  
0000 0000 0000 0000 0000 0000 0000 0000// i >> 16
0000 0000 0000 0000 0000 0111 1111 1111// i |= (i >> 16)

0000 0000 0000 0000 0000 0011 1111 1111// i >>> 1
0000 0000 0000 0000 0000 0100 0000 0000// i - (i >>> 1)
  
```

将上述代码分析后我们可以看到, 在i为正数的情况下, 右移次数从2^0到2^4,共计右移了32次, 每次都使用了或运算符, 最后得到的i为`大于i的2^(n+1) - 1`

```java
将i的值再次右移后得到
2^n-1
两者相减
  (2^(n+1) - 1) - (2^n-1)
= 2^(n+1) - 2^n
= 2^n +2^n -2^n
= 2^n
```

最终求出的值正是小于等于i的`2^n`

## putForNullKey

```java
// key为null的特殊逻辑
private V putForNullKey(V value) {
  // 遍历数组第一个下标里的链表, 如果包含key = null,那么替换value,并返回旧的value
  for (Entry<K,V> e = table[0]; e != null; e = e.next) {
    if (e.key == null) {
      V oldValue = e.value;
      e.value = value;
      e.recordAccess(this);
      return oldValue;
    }
  }
  // 一个快速事变参数
  modCount++;
  // 将null值添加到链表中, 该方法稍后说
  addEntry(0, null, value, 0);
  return null;
}
```

## indexFor

```java
static int indexFor(int h, int length) {
  // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
  return h & (length-1);
}
```

`&`操作必须两个数组的二进制对应位置同时为1才为1,所以返回值必定小于等于length-1

```java
0000 0000 0000 0000 1111 0100 0000 0100// a
0000 0000 0000 0000 0000 0000 0001 0101// b
0000 0000 0000 0000 0000 0000 0000 0100// a&b

```

# addEntry

```java
// 添加节点, 该节点位置可能为空
void addEntry(int hash, K key, V value, int bucketIndex) {
  if ((size >= threshold) && (null != table[bucketIndex])) {
    // 数组扩容并转移元素
    resize(2 * table.length);
    // 重新获取hash
    hash = (null != key) ? hash(key) : 0;
    // 重新获取下标
    bucketIndex = indexFor(hash, table.length);
  }
	// 将数据插入节点
  createEntry(hash, key, value, bucketIndex);
}
```

1. size >= threshold : 这个是扩容条件, 每次添加一个新的值的时候, size都会进行自加操作, 
2. null != table[bucketIndex]: 当前节点为空的情况下直接添加, 反正没挂东西,效率很高的

## resize

```java
void resize(int newCapacity) {
  Entry[] oldTable = table;
  int oldCapacity = oldTable.length;
  // 如果数组长度达到最大值, 则不扩容
  if (oldCapacity == MAXIMUM_CAPACITY) {
    threshold = Integer.MAX_VALUE;
    return;
  }
	// 创建一个新的数组
  Entry[] newTable = new Entry[newCapacity];
  // 将之前的数据转移到新数组
  transfer(newTable, initHashSeedAsNeeded(newCapacity));
  // 将新数组赋值给成员变量, 完成替换
  table = newTable;
  // 刷新下次扩容的大小
  threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}
```

## createEntry

```java
void createEntry(int hash, K key, V value, int bucketIndex) {
  Entry<K,V> e = table[bucketIndex];
  // 创建一个新的节点, 将原有节点挂载在当前节点下, 新的节点替换原有的节点
  table[bucketIndex] = new Entry<>(hash, key, value, e);
  size++;
}
```

> 这位置表达起来比较麻烦, 理解就好, 很简单

### transfer

```java
// 将之前的数据转移到新数组
void transfer(Entry[] newTable, boolean rehash) {
  int newCapacity = newTable.length;
  // table: 旧的数组
  for (Entry<K,V> e : table) {
    while(null != e) {
      Entry<K,V> next = e.next;
      // rehash; 是否需要重新计算hash
      if (rehash) {
        e.hash = null == e.key ? 0 : hash(e.key);
      }
      // 计算当前值在新数组中的位置
      int i = indexFor(e.hash, newCapacity);
      // 将数组节点数据挂到当前节点
      e.next = newTable[i];
      // 进行替换
      newTable[i] = e;
      // 遍历下一个节点
      e = next;
    }
  }
}
```

---

# get

```java
public V get(Object key) {
  if (key == null)
    // 如果为null, 走一段特殊逻辑,直接从数组第一位取值
    return getForNullKey();
  // 实际获取方法
  Entry<K,V> entry = getEntry(key);
  return null == entry ? null : entry.getValue();
}
```

## getForNullKey

```java
private V getForNullKey() {
  // 如果map为空, 则直接返回null
  if (size == 0) {
    return null;
  }
  // 从数组第一个元素中循环查找null值
  for (Entry<K,V> e = table[0]; e != null; e = e.next) {
    if (e.key == null)
      return e.value;
  }
  return null;
}
```

## getEntry

```java
final Entry<K,V> getEntry(Object key) {
  // 如果map为空, 则直接返回null
  if (size == 0) {
    return null;
  }
	// 获取hash
  int hash = (key == null) ? 0 : hash(key);
  // 依据hash找到对应数组节点, 遍历
  for (Entry<K,V> e = table[indexFor(hash, table.length)];
       e != null;
       e = e.next) {
    Object k;
    // 比较
    if (e.hash == hash &&
        ((k = e.key) == key || (key != null && key.equals(k))))
      return e;
  }
  return null;
}
```

