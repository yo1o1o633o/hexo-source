---
title: 源码解析-ArrayList
date: 2021-09-25 18:25:17
top: 1
categories:
    - JAVA
tags:
    - JAVA
    - 源码
---

{% note success %}
### 基本信息
{% endnote %}
- 内部采用动态数组形式存储(添加, 删除等操作都是依赖System.arraycopy进行)
- 每次扩容为原来容量的0.5倍, 当数组填满不足以保存添加的元素时开始触发扩容
- 如果要大量添加元素, 可提前调用ensureCapacity方法进行扩容, 直接扩容到指定容量, 不用多次进行0.5倍扩容操作

{% note success %}
### 构造方法
{% endnote %}
构造一个具有指定容量的列表, 容量不能为负
```java
public ArrayList(int initialCapacity) {
    // 指定容量不能为负, 为负时抛出异常.
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: " + initialCapacity);
    }
}
```
构造一个初始容量为10的空列表
```java
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```
按照集合的迭代器返回的顺序构造一个包含指定集合元素的列表
```java
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // 初始化为空数组
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

{% note success %}
### 裁剪列表
{% endnote %}
将列表收缩为实际元素数量的大小.节省内存空间
```java
public void trimToSize() {
    modCount++;
    if (size < elementData.length) {
        elementData = (size == 0) ? EMPTY_ELEMENTDATA: Arrays.copyOf(elementData, size);
    }
}
```

{% note success %}
### 预扩容
{% endnote %}
当已知要向列表添加大量元素时, 可提前调用此方法一次性扩容到指定容量. 避免在添加的过程中频繁扩容
```java
public void ensureCapacity(int minCapacity) {
    int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
        // any size if not default element table
        ? 0
        // larger than default for default empty table. It's already
        // supposed to be at default size.
        : DEFAULT_CAPACITY;

    if (minCapacity > minExpand) {
        // 扩容以至少满足可以容纳指定的元素
        ensureExplicitCapacity(minCapacity);
    }
}
```

{% note success %}
### 扩容列表
{% endnote %}
增加容量, 以确保可以至少容纳指定的元素数量
```java
private void grow(int minCapacity) {
    // overflow-conscious code
    // 当前数组容量
    int oldCapacity = elementData.length;
    // 增加0.5倍容量
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    // 增加0.5倍不足以容纳此次扩容的元素, 则使用指定的容量
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    // 新长度超出数组最大长度限制, 可能发生OOM
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```


{% note success %}
### 快速移除元素
{% endnote %}
采用数组拷贝的形式快速移除元素, 可以跳过边界检查
```java
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index, numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```
