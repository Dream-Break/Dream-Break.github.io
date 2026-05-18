---
title: 'ArrayList解析'
description: '深入分析ArrayList的底层实现原理，包括扩容机制、常用方法源码解析'
pubDate: '2021-03-22'
tags: ['集合']
category: 'Java'
---

## 概述

ArrayList是Java中最常用的集合类之一，底层基于**数组**实现，支持动态扩容。

### 核心特点

- 底层使用`Object[]`数组存储数据
- 支持随机访问，通过索引获取元素时间复杂度为O(1)
- 插入和删除操作可能需要移动元素，时间复杂度为O(n)
- 非线程安全
- 允许存储`null`值

## 类定义

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable {

    // 默认初始容量
    private static final int DEFAULT_CAPACITY = 10;

    // 空数组实例
    private static final Object[] EMPTY_ELEMENTDATA = {};

    // 默认大小的空数组实例
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    // 存储ArrayList元素的数组缓冲区
    transient Object[] elementData;

    // ArrayList的实际大小（包含的元素数量）
    private int size;
}
```

## 构造方法

```java
// 指定初始容量
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: " + initialCapacity);
    }
}

// 无参构造，默认空数组，首次add时扩容为10
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

## 扩容机制

ArrayList的核心机制之一就是**自动扩容**。当数组容量不够时，会创建一个更大的数组并将原数据复制过去。

```java
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    // 新容量 = 旧容量 + 旧容量的一半（即1.5倍）
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // 数组复制
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

**扩容规则**：每次扩容为原来的**1.5倍**（`oldCapacity + oldCapacity >> 1`）。

## add方法

```java
public boolean add(E e) {
    // 确保容量足够
    ensureCapacityInternal(size + 1);
    // 在数组末尾添加元素
    elementData[size++] = e;
    return true;
}

public void add(int index, E element) {
    rangeCheckForAdd(index);
    ensureCapacityInternal(size + 1);
    // 将index及其后面的元素往后移动一位
    System.arraycopy(elementData, index, elementData, index + 1, size - index);
    elementData[index] = element;
    size++;
}
```

## remove方法

```java
public E remove(int index) {
    rangeCheck(index);
    modCount++;
    E oldValue = elementData(index);
    int numMoved = size - index - 1;
    if (numMoved > 0)
        // 将index后面的元素往前移动一位
        System.arraycopy(elementData, index + 1, elementData, index, numMoved);
    // 将最后一位置空，帮助GC
    elementData[--size] = null;
    return oldValue;
}
```

## get方法

```java
public E get(int index) {
    rangeCheck(index);
    return elementData(index);
}
```

由于底层是数组，所以通过索引获取元素的时间复杂度为O(1)，这也是ArrayList实现了`RandomAccess`接口的原因。

## 与LinkedList的对比

| 特性 | ArrayList | LinkedList |
|------|-----------|------------|
| 底层结构 | 数组 | 双向链表 |
| 随机访问 | O(1) | O(n) |
| 头部插入 | O(n) | O(1) |
| 尾部插入 | O(1)均摊 | O(1) |
| 内存占用 | 连续内存 | 每个节点额外指针开销 |

## 总结

- ArrayList底层是动态数组，默认初始容量为10
- 扩容为原来的1.5倍
- 适合随机访问场景，不适合频繁插入删除
- 非线程安全，多线程环境下可以使用`CopyOnWriteArrayList`或`Collections.synchronizedList()`
