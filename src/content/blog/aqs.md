---
title: 'AQS详解'
description: '深入分析AbstractQueuedSynchronizer的实现原理，包括独占锁和共享锁的加锁解锁流程'
pubDate: '2021-03-21'
tags: ['多线程']
category: 'Java'
---

## 序言

AQS是AbstractQueuedSynchronizer类的简称，虽然我们不会直接使用这个类，但是这个类是Java很多并发工具的实现基础。

CountDownLatch，Semaphore，ReentrantLock等常见的工具类都是由AQS来实现的。

## AQS基本架构

AQS类的定义如下：

```java
/**
 * AbstractQueuedSynchronizer是一个抽象类
 * 实现了Serializable接口
 * @since 1.5
 * @author Doug Lea
 */
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {

    /**
     * state变量表示锁的状态
     * 0 表示未锁定
     * 大于0表示已锁定
     * 这个值可以用来实现锁的【可重入性】
     * 同时这个变量还是用volatile关键字修饰的，保证可见性
     */
    private volatile int state;

    /** 等待队列的头节点 */
    private transient volatile Node head;

    /** 等待队列的尾结点 */
    private transient volatile Node tail;
}
```

AQS本质是一个队列，其内部维护着FIFO的双向链表（CLH锁定队列）。

### Node内部类

队列中的每一个元素都是一个Node：

```java
static final class Node {
    // 共享模式标记
    static final Node SHARED = new Node();
    // 独占模式标记
    static final Node EXCLUSIVE = null;

    // waitStatus可选值
    static final int CANCELLED =  1;  // 节点被取消
    static final int SIGNAL    = -1;  // 后继节点处于等待状态
    static final int CONDITION = -2;  // 节点处于condition队列中
    static final int PROPAGATE = -3;  // 共享模式下无条件传播

    volatile int waitStatus;   // 节点的等待状态
    volatile Node prev;        // 前驱节点
    volatile Node next;        // 后继节点
    volatile Thread thread;    // 获取同步状态的线程
    Node nextWaiter;           // condition队列等待节点
}
```

## 核心方法 - 独占式

AQS实现了两套加锁解锁的方式：独占式和共享式。先看独占式的实现，从`ReentrantLock#lock()`开始。

### ReentrantLock#lock

```java
public void lock() {
    sync.lock();
}
```

公平锁和非公平锁的区别：

```java
// FairSync 公平锁
final void lock() {
    acquire(1);
}

// NonfairSync 非公平锁
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```

非公平锁多了一个步骤：通过CAS尝试更改state状态，这就是**公平锁和非公平锁的本质区别**。

### acquire

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

### tryAcquire（公平锁实现）

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        // 可重入性的实现
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

### addWaiter

把当前线程封装成Node节点，加入队列尾部：

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```

`enq`方法通过**自旋 + CAS**的方式保证线程安全和插入成功。

### acquireQueued

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

### 独占锁解锁 - ReentrantLock#unlock

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

`tryRelease`方法计算剩余的重入次数，当state为0时表示完全释放了锁。`unparkSuccessor`方法用于唤醒后继节点。

---

## 核心方法 - 共享式

以`Semaphore`为例，它允许多个线程同时执行，是共享锁的实现。

### Semaphore#acquire

```java
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

### tryAcquireShared（公平模式）

```java
protected int tryAcquireShared(int acquires) {
    for (;;) {
        if (hasQueuedPredecessors())
            return -1;
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

依然是**自旋 + CAS**的方式保证线程安全。

### 共享锁解锁

`tryReleaseShared`通过CAS增加state值（归还许可），`doReleaseShared`唤醒后继节点。

---

## 总结

AQS是并发编程中比较难的一个类，但是理解AQS非常重要，因为它是JDK中锁和其他同步工具实现的基础。

核心要点：
- AQS维护一个FIFO的CLH队列
- 通过state变量表示锁状态，实现可重入性
- 采用**自旋 + CAS**保证线程安全
- 使用**模板方法设计模式**，由子类实现具体的加锁解锁逻辑
- 支持独占式和共享式两种模式
