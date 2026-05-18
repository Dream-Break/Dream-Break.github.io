---
title: 'CAS算法与Java原子类'
description: '深入理解CAS（Compare And Swap）算法原理及其在Java原子类中的应用'
pubDate: '2021-03-23'
tags: ['多线程']
category: 'Java'
---

## 什么是CAS？

CAS（Compare And Swap），即**比较并交换**，是一种乐观锁的实现方式。它包含三个操作数：

- **V（Value）**：要更新的变量值（内存值）
- **E（Expected）**：预期值（旧值）
- **N（New）**：新值

**CAS的核心逻辑**：当且仅当V的值等于E时，才会将V的值更新为N；如果V的值不等于E，说明已经有其他线程修改了V的值，当前线程不做任何操作。

```
if (V == E) {
    V = N;
    return true;
} else {
    return false;
}
```

## CAS的底层实现

在Java中，CAS操作是通过`sun.misc.Unsafe`类来实现的。Unsafe类提供了硬件级别的原子操作，它调用的是底层操作系统的**CPU指令**（如x86的`cmpxchg`指令），保证了操作的原子性。

```java
public final native boolean compareAndSwapInt(Object o, long offset, int expected, int x);
public final native boolean compareAndSwapLong(Object o, long offset, long expected, long x);
public final native boolean compareAndSwapObject(Object o, long offset, Object expected, Object x);
```

## Java原子类

JDK提供了`java.util.concurrent.atomic`包，包含一系列原子类，底层都是通过CAS实现的。

### AtomicInteger

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private volatile int value;

    // 原子性的获取并自增
    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }

    // 原子性的获取并自减
    public final int getAndDecrement() {
        return unsafe.getAndAddInt(this, valueOffset, -1);
    }

    // CAS操作
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
}
```

### Unsafe#getAndAddInt

```java
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        // 获取内存中的最新值
        var5 = this.getIntVolatile(var1, var2);
        // 自旋 + CAS，直到成功
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));
    return var5;
}
```

这就是典型的**自旋 + CAS**模式。

## CAS的三大问题

### 1. ABA问题

如果一个值原来是A，变成了B，又变回了A，那么使用CAS检查时会发现它的值没有变化，但实际上却变化过了。

**解决方案**：使用版本号机制。JDK提供了`AtomicStampedReference`类：

```java
AtomicStampedReference<Integer> ref = new AtomicStampedReference<>(1, 0);
// 每次修改版本号+1
ref.compareAndSet(1, 2, 0, 1);
```

### 2. 自旋开销大

如果CAS一直不成功，会进行自旋操作，给CPU带来非常大的开销。

### 3. 只能保证一个变量的原子操作

CAS只能保证对单个变量操作的原子性。如果需要对多个变量进行原子操作，可以：
- 使用锁
- 把多个变量封装成一个对象，使用`AtomicReference`

## 常用原子类一览

| 类型 | 原子类 |
|------|--------|
| 基本类型 | AtomicInteger, AtomicLong, AtomicBoolean |
| 数组类型 | AtomicIntegerArray, AtomicLongArray, AtomicReferenceArray |
| 引用类型 | AtomicReference, AtomicStampedReference, AtomicMarkableReference |
| 字段更新器 | AtomicIntegerFieldUpdater, AtomicLongFieldUpdater, AtomicReferenceFieldUpdater |

## 总结

- CAS是一种**无锁**的并发编程方式，属于乐观锁
- 底层依赖CPU的原子指令实现
- 通过**自旋 + CAS**保证操作的原子性
- 需要注意ABA问题、自旋开销等问题
- Java的原子类（atomic包）就是CAS的典型应用
