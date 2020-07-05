---
title: Redis实用API
permalink: redis-shi-yong-api
date: 2020-02-16 22:32:47
tags:
categories: redis
---

# 前言

> 学习过程中遇到的感觉比较有意思的`API`做一个整理

# setNx

> 如果指定的Key不存在，则设定该Key持有指定字符串Value，此时其效果等价于SET命令。相反，如果该Key已经存在，该命令将不做任何操作并返回。

<!--more-->

这个命令常用语分布式锁, 用法如下

```java
void test(){
  Boolean flag = false;
  try {
    flag = redis.setnx("key", 1);
    if (flag){
      redis.expire(lockKey, 10);
      fun();
    }
  }finally {
    if (flag){
      redis.del("key");
    }
  }
}
```

1. 设置锁的位置和需要执行的方法要在`try`内
2. 在`finally`内部判断加锁成功后删除锁
3. 为了防止一个方法超时,长时间占用锁, 我们一般会加一个过期时间
4. 过期时间添加需谨慎, 假设方法执行了12s会出现如下情况
   1. 0s:第一个线程进来后加锁
   2. 10s:第一个线程的锁过期
   3. 10s:第二个线程获取到锁,并执行方法
   4. 12s:第一个线程执行完,进入`finally`删除第二个线程的锁
   5. 12s:第三个线程进入并获取锁

解决4的情况除了正确预估时间外, 还可以使用将`value`的值设置为每一请求的独立编号,如商品id等, 在删除时先比较`value`是否相等再删除. 这样可以在少量锁超时时不会误删除其他线程的锁

# incr

> 将指定Key的Value原子性的递增1。如果该Key不存在，其初始值为0，在incr之后其值为1。如果Value的值不能转换为整型值，如Hello，该操作将执行失败
>
> 并返回相应的错误信息。注意：该操作的取值范围是64位有符号整型。

这个一般用于`id`递增的情况, 但是redis毕竟是一个缓存性质的数据库, 建议代码逻辑自增

# blpop/brpop

> Blpop: 移出并获取列表的第一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。
>
> Brpop: 移出并获取列表的最后一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。

一般当做一个发布订阅的消息列队使用

1. 数据量小的情况下:`ArrayBlockingQueue`不香吗?
2. 数据量大的情况下: `mq`不香吗?
3. 数据量不尴不尬, 应用范围不算太大,又没钱: 真香!

# HyperLogLog

> Redis HyperLogLog 是用来做基数统计的算法，HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定 的、并且是很小的。
>
> 在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基 数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。
>
> 但是，因为 HyperLogLog 只会根据输入元素来计算基数，而不会储存输入元素本身，所以 HyperLogLog 不能像集合那样，返回输入的各个元素。

一般用于计算页面访问量, 同一个页面无论访问多少次都计算一次

具体没有太搞懂, 如果需要实现上述功能也可以使用下面这个

# bitmaps

> Redis提供的Bitmaps这个“数据结构”可以实现对位的操作。Bitmaps本身不是一种数据结构，实际上就是字符串，但是它可以对字符串的位进行操作
>
> 可以把Bitmaps想象成一个以位为单位数组，数组中的每个单元只能存0或者1，数组的下标在bitmaps中叫做偏移量。单个bitmaps的最大长度是512MB，即2^32个比特位

位图, 一般用于统计每天的用户活跃数/签到/点赞等

在将id等设置进位图时可以考虑先进行减最小id,尽可能的把数组的长度缩小, 否则初始化时间会比较长,同时造成资源浪费

# Pipeline

> 大批量数据写入`redis`,同时,都不需要返回值, 可以使用`Pipeline`加速

用于热点数据的初始化

1. [分布式缓存Redis之Pipeline（管道）](https://blog.csdn.net/johnf_nash/article/details/87894802)
2. [Redis系列十：Pipeline详解](https://blog.csdn.net/w1lgy/article/details/84455579)

# GEO

> **GEO**功能在Redis3.2版本提供，支持存储地理位置信息用来实现诸如附近位置、摇一摇这类依赖于地理位置信息的功能.geo的数据类型为zset.

可以实现获取附近的人功能

# RedisBloom

> 这是一个`redis4.0`布隆过滤器的插件

用途及使用请看这篇[布隆过滤器](https://www.chenguanting.top/2020/布隆过滤器)

扩展[Redis 高级主题之布隆过滤器(BloomFilter)](https://blog.csdn.net/weixin_34410662/article/details/91434798)

