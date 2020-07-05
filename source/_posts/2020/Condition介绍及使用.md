---
title: 并发8:Condition介绍及使用
permalink: condition-jie-shao-use
date: 2020-02-03 10:43:56
tags:
categories: concurrent
---
# 前言
> 　Condition是用来替代传统的Object的wait()、notify()的，相比使用Object的wait()、notify()实现线程间协作更加安全和高效

# 介绍
1. 生成一个Condition的代码是lock.newCondition() 
2. 调用Condition的await()和signal()方法，都必须在lock之内，就是说必须在lock.lock()和lock.unlock之间才可以使用
3. Condition中的await()对应Object的wait()
4. Condition中的signal()对应Object的notify()
5. Condition中的signalAll()对应Object的notifyAll()


<!--more-->

# 简单的使用
## Object.wait()
```
public class ConditionDemo1 {

   Object lock = new Object();

    public static void main(String[] args) {
        ConditionDemo1 demo = new ConditionDemo1();
        new Thread(() -> demo.test1()).start();
        new Thread(() -> demo.test2()).start();

    }

    void test1() {
        synchronized (lock){
            System.out.println("1号  上锁");
            try {
                lock.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("1号  收到信号不等了");
        }
        System.out.println("1号  完全释放锁");
    }
    
    void test2() {
        synchronized (lock){
            System.out.println("2号  上锁");
            lock.notifyAll();
            System.out.println("2号  叫醒1号");
        }
        System.out.println("2号  完全释放锁");
    }

}
```
运行结果如下

- 1号  上锁
- 2号  上锁
- 2号  叫醒1号
- 2号  完全释放锁
- 1号  收到信号不等了
- 1号  完全释放锁

## lock.newCondition()
```
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class ConditionDemo2 {

    ReentrantLock lock;
    Condition condition;

    ConditionDemo2() {
        lock = new ReentrantLock();
        condition = lock.newCondition();
    }

    public static void main(String[] args) {
        ConditionDemo2 demo = new ConditionDemo2();
        new Thread(() -> demo.test1()).start();
        new Thread(() -> demo.test2()).start();
    }

    void test1() {
        try {
            lock.lock();
            System.out.println("1号  上锁");
            try {
                System.out.println("1号  等待");
                condition.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("1号  收到信号不等了");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
            System.out.println("1号  释放锁");
        }
    }


    void test2() {
        try {
            lock.lock();
            System.out.println("2号  上锁");
            condition.signalAll();
            System.out.println("2号  叫醒1号");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
            System.out.println("2号  释放锁");
        }
    }

}
```
运行结果如下

- 1号  上锁
- 1号  等待
- 2号  上锁
- 2号  叫醒1号
- 2号  完全释放锁
- 1号  收到信号不等了
- 1号  释放锁

# 生产消费模式

## Object.wait()
```
import java.util.LinkedList;

/**
 * 生产消费 列队最大10
 */
public class ConditionDemo3 {


    volatile LinkedList<Integer> list = new LinkedList();
    static final int MAX_SIZE = 10;

    Object lock = new Object();

    public static void main(String[] args) {
        ConditionDemo3 demo = new ConditionDemo3();
        new Thread(() -> demo.test1(), "c1").start();
        new Thread(() -> demo.test1(), "c2").start();
        new Thread(() -> demo.test2(), "p1").start();
        new Thread(() -> demo.test2(), "p2").start();
    }

    void test1() {
        String name = Thread.currentThread().getName() + ":";
        synchronized (lock) {
            System.out.println(name + "上锁");

            for (int i = 0; i < 100; i++) {
                while (list.size() == MAX_SIZE) {
                    try {
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(name + "等待");
                }
                list.add(i);
                lock.notifyAll();
            }
        }
        System.out.println(name + "释放锁");
    }

    void test2() {
        String name = Thread.currentThread().getName() + ":";
        synchronized (lock) {
            System.out.println(name + "上锁");
            while (true) {
                while (list.isEmpty()) {
                    try {
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(name + "list为空-等待");
                }
                System.out.println(name + "消费" + list.removeFirst() + "\t" + "list容量为:" + (list.size() + 1));
                lock.notifyAll();
            }
        }
    }

}
```
## lock.newCondition()
```
package com.luban.self;

import java.util.LinkedList;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 生产消费 列队最大10
 */
public class ConditionDemo4 {


    volatile LinkedList<Integer> list = new LinkedList();
    static final int MAX_SIZE = 10;

    ReentrantLock lock;
    Condition product;
    Condition customer;

    ConditionDemo4() {
        lock = new ReentrantLock();
        product = lock.newCondition();
        customer = lock.newCondition();
    }

    public static void main(String[] args) {
        ConditionDemo4 demo = new ConditionDemo4();
        new Thread(() -> demo.test1(), "c1").start();
        new Thread(() -> demo.test1(), "c2").start();
        new Thread(() -> demo.test2(), "p1").start();
        new Thread(() -> demo.test2(), "p2").start();
    }

    void test1() {
        String name = Thread.currentThread().getName() + ":";
        try {
            lock.lock();
            System.out.println(name + "上锁");

            for (int i = 0; i < 100; i++) {
                while (list.size() == MAX_SIZE) {
                    try {
                        product.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(name + "等待");
                }
                list.add(i);
                customer.signalAll();
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
            System.out.println(name + "释放锁");
        }
    }

    void test2() {
        String name = Thread.currentThread().getName() + ":";

        try {
            lock.lock();
            System.out.println(name + "上锁");
            while (true) {
                while (list.isEmpty()) {
                    try {
                        customer.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(name + "list为空-等待");
                }
                System.out.println(name + "消费" + list.removeFirst() + "\t" + "list容量为:" + (list.size() + 1));
                product.signalAll();
            }

        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

}
```
# 总结
可以看到使用`#newCondition`方法时可以明确的指定具体将那种类型的锁取消等待, 在线程多的情况下可以起到很大的优化