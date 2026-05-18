---
title: 'synchronized关键字'
description: '全面解析synchronized关键字的使用方式、底层原理和锁升级过程'
pubDate: '2021-03-23'
tags: ['多线程']
category: 'Java'
---

## synchronized简介

`synchronized`是Java中最基本的同步机制，用于保证在同一时刻只有一个线程可以执行某段代码，从而保证线程安全。

## 三种使用方式

### 1. 修饰实例方法

锁对象是当前实例对象`this`：

```java
public synchronized void method() {
    // 临界区代码
}
```

### 2. 修饰静态方法

锁对象是当前类的`Class`对象：

```java
public static synchronized void method() {
    // 临界区代码
}
```

### 3. 修饰代码块

锁对象是括号中指定的对象：

```java
public void method() {
    synchronized (this) {
        // 临界区代码
    }
}

// 也可以指定其他对象作为锁
private final Object lock = new Object();
public void method() {
    synchronized (lock) {
        // 临界区代码
    }
}
```

## 底层原理

### Monitor（监视器锁）

`synchronized`的底层是通过**Monitor**（监视器锁/管程）来实现的。每个Java对象都关联一个Monitor。

- **同步代码块**：编译后会在同步块前后生成`monitorenter`和`monitorexit`指令
- **同步方法**：方法的`flags`中会标记`ACC_SYNCHRONIZED`

### monitorenter/monitorexit

```
monitorenter:
  线程尝试获取monitor的所有权
  如果monitor的进入计数为0，线程进入monitor，计数设为1
  如果线程已经拥有monitor，重新进入，计数+1（可重入）
  如果其他线程拥有monitor，当前线程阻塞等待

monitorexit:
  执行该指令的线程必须是monitor的所有者
  计数-1，如果计数为0，线程释放monitor
```

## 对象头与锁

Java对象在内存中的布局包含**对象头**，对象头中的**Mark Word**存储了锁相关的信息。

| 锁状态 | Mark Word内容 | 标志位 |
|--------|--------------|--------|
| 无锁 | hashCode、GC分代年龄 | 01 |
| 偏向锁 | 线程ID、Epoch、GC分代年龄 | 01 |
| 轻量级锁 | 指向栈中锁记录的指针 | 00 |
| 重量级锁 | 指向Monitor的指针 | 10 |
| GC标记 | 空 | 11 |

## 锁升级过程

JDK 1.6之后，`synchronized`引入了**锁升级**机制来优化性能：

### 无锁 → 偏向锁

当一个线程第一次访问同步块时，会在对象头中记录线程ID，标记为偏向锁。下次同一线程再访问时，只需检查线程ID是否匹配即可，无需CAS操作。

**适用场景**：只有一个线程访问同步块。

### 偏向锁 → 轻量级锁

当第二个线程尝试获取锁时，偏向锁会撤销，升级为轻量级锁。

轻量级锁通过**CAS + 自旋**的方式获取锁，避免了线程阻塞。

**适用场景**：多个线程交替执行同步块（竞争不激烈）。

### 轻量级锁 → 重量级锁

当自旋一定次数后仍然无法获取锁，或者多个线程同时竞争锁，轻量级锁会膨胀为重量级锁。

重量级锁会导致线程阻塞，需要操作系统介入进行线程切换。

**适用场景**：多个线程激烈竞争锁。

## synchronized的特性

| 特性 | 说明 |
|------|------|
| 原子性 | 同步块中的操作要么全部执行，要么全部不执行 |
| 可见性 | 线程解锁前，必须把修改后的值刷回主内存 |
| 有序性 | 同步块内部可以重排序，但不影响同步块整体的执行顺序 |
| 可重入性 | 同一线程可以多次获取同一把锁 |

## synchronized vs Lock

| 对比项 | synchronized | Lock |
|--------|-------------|------|
| 实现方式 | JVM层面（关键字） | API层面（接口） |
| 释放锁 | 自动释放 | 需要手动unlock() |
| 可中断 | 不可中断 | 可中断（lockInterruptibly） |
| 公平性 | 非公平锁 | 可选公平/非公平 |
| 条件绑定 | 一个条件 | 可绑定多个Condition |
| 锁状态 | 无法判断 | 可判断（tryLock） |

## 总结

- `synchronized`是Java内置的同步机制，使用简单
- 底层通过Monitor实现，依赖对象头中的Mark Word
- JDK 1.6之后引入锁升级优化：偏向锁 → 轻量级锁 → 重量级锁
- 具有原子性、可见性、有序性和可重入性
- 在低竞争场景下性能已经很好，不一定需要使用Lock
