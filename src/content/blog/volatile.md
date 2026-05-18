---
title: 'volatile关键字'
description: '深入理解volatile关键字的作用、底层原理以及使用场景'
pubDate: '2021-03-23'
tags: ['多线程']
category: 'Java'
---

## volatile是什么？

`volatile`是Java中的一个关键字，用于修饰变量。它主要有两个作用：

1. **保证可见性**：一个线程修改了volatile变量的值，其他线程能够立即看到修改后的值
2. **禁止指令重排序**：保证代码的执行顺序与程序中的顺序一致

> 注意：volatile**不保证原子性**！

## Java内存模型（JMM）

要理解volatile，首先需要了解Java内存模型。

JMM规定：
- 所有变量都存储在**主内存**中
- 每个线程都有自己的**工作内存**（本地内存）
- 线程对变量的操作必须在工作内存中进行，不能直接操作主内存
- 线程间的通信必须通过主内存来完成

```
线程A 工作内存  ←→  主内存  ←→  线程B 工作内存
```

### 可见性问题

正因为每个线程有自己的工作内存，当线程A修改了共享变量的值，如果没有及时刷新到主内存，或者线程B没有从主内存中重新读取，就会导致线程B看到的值是旧的。

## volatile如何保证可见性？

当一个变量被`volatile`修饰后：

1. **写操作**：线程修改了volatile变量后，会立即将值刷新到主内存
2. **读操作**：线程读取volatile变量时，会从主内存中重新读取最新值

底层实现是通过**内存屏障（Memory Barrier）** 来实现的：

- 在volatile写操作后插入`StoreStore`和`StoreLoad`屏障
- 在volatile读操作前插入`LoadLoad`和`LoadStore`屏障

## volatile如何禁止重排序？

编译器和处理器为了优化性能，可能会对指令进行重排序。但有些场景下重排序会导致程序语义改变。

volatile通过内存屏障禁止了特定类型的重排序：

| | 普通读/写 | volatile读 | volatile写 |
|---|---|---|---|
| 普通读/写 | - | - | 不允许重排 |
| volatile读 | 不允许重排 | 不允许重排 | 不允许重排 |
| volatile写 | - | 不允许重排 | 不允许重排 |

## 经典应用：双重检查锁定（DCL）

```java
public class Singleton {
    // 必须使用volatile！
    private volatile static Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {                 // 第一次检查
            synchronized (Singleton.class) {
                if (instance == null) {         // 第二次检查
                    instance = new Singleton(); // 非原子操作！
                }
            }
        }
        return instance;
    }
}
```

**为什么需要volatile？** 因为`instance = new Singleton()`实际上分为三步：

1. 分配内存空间
2. 初始化对象
3. 将instance指向分配的内存地址

如果没有volatile，步骤2和3可能被重排序。如果线程A执行了1和3（还没执行2），线程B进入第一次检查时会发现instance不为null，直接返回了一个**未初始化完成的对象**！

## volatile不保证原子性

```java
public class VolatileTest {
    volatile int count = 0;

    public void increment() {
        count++; // 这不是原子操作！
    }
}
```

`count++`实际上包含三步操作：
1. 读取count的值
2. 将值+1
3. 写回新值

多线程环境下，这三步之间可能被其他线程打断。

**解决方案**：
- 使用`synchronized`
- 使用`AtomicInteger`等原子类

## volatile的使用场景

### 1. 状态标志

```java
volatile boolean running = true;

public void shutdown() {
    running = false;
}

public void run() {
    while (running) {
        // do something
    }
}
```

### 2. 一次性安全发布

```java
volatile Singleton instance;
```

### 3. 独立观察（多个线程观察同一个值的变化）

```java
volatile int temperature;

// 传感器线程
public void update(int temp) {
    temperature = temp;
}

// 显示线程
public void display() {
    System.out.println("当前温度: " + temperature);
}
```

## volatile vs synchronized

| 对比项 | volatile | synchronized |
|--------|----------|-------------|
| 可见性 | 保证 | 保证 |
| 原子性 | 不保证 | 保证 |
| 有序性 | 保证 | 保证 |
| 阻塞 | 不会 | 会 |
| 使用范围 | 变量 | 变量、方法、代码块 |
| 性能 | 较好 | 相对较差 |

## 总结

- volatile保证**可见性**和**有序性**，但不保证**原子性**
- 底层通过**内存屏障**实现
- 适用于"一写多读"的场景
- 对于复合操作（如i++），需要配合其他同步手段
- 比synchronized轻量，但使用场景有限
