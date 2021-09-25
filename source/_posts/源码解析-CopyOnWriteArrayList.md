---
title: 源码解析-CopyOnWriteArrayList
date: 2021-09-25 22:46:41
categories:
    - JAVA
tags:
    - JAVA
    - 源码
---

{% note success %}
### 基本信息
{% endnote %}
- 线程安全, 通过在底层为数组创建新的副本实现
- 并发的添加, 设置操作通过使用ReentrantLock锁来保证线程安全
- 没有预扩容操作, 每次添加都进行容量调整

{% note success %}
### 构造方法
{% endnote %}
创建一个空列表
```java
public CopyOnWriteArrayList() {
    setArray(new Object[0]);
}
```
创建一个包含指定集合的元素的列表,按照集合的迭代器返回的顺序
```java
public CopyOnWriteArrayList(Collection<? extends E> c) {
    Object[] elements;
    // 检查要添加的集合类型
    if (c.getClass() == CopyOnWriteArrayList.class)
        elements = ((CopyOnWriteArrayList<?>)c).getArray();
    else {
        elements = c.toArray();
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elements.getClass() != Object[].class)
            elements = Arrays.copyOf(elements, elements.length, Object[].class);
    }
    setArray(elements);
}
```
创建一个包含给定数组副本的列表
```java
public CopyOnWriteArrayList(E[] toCopyIn) {
    setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
}
```

{% note success %}
### 设置指定索引元素
{% endnote %}
并发的设置操作, 对指定索引的位置赋值, 采用ReentrantLock锁来保证线程安全
```java
public E set(int index, E element) {
    final ReentrantLock lock = this.lock;
    // 加锁, 保证多个线程的设置操作安全
    lock.lock();
    try {
        // 获取原数组
        Object[] elements = getArray();
        // 获取原数组指定索引位置元素
        E oldValue = get(elements, index);
        // 新元素值和当前元素值不同, 进行设置
        if (oldValue != element) {
            int len = elements.length;
            // 复制一个原数组的副本, 容量为原数组长度
            Object[] newElements = Arrays.copyOf(elements, len);
            // 对指定索引位置赋值
            newElements[index] = element;
            // 将新数组赋值给原数组
            setArray(newElements);
        } else {
            // Not quite a no-op; ensures volatile write semantics
            // 新元素值和当前元素值相同, 不是完全没有操作, 将原数组赋值给原数组
            setArray(elements);
        }
        // 返回旧值
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```

{% note success %}
### 添加元素
{% endnote %}
添加一个元素到数组最后
并发的添加操作, 采用ReentrantLock锁来保证线程安全
```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    // 加锁, 保证多个线程并发添加元素安全
    lock.lock();
    try {
        // 获取原数组
        Object[] elements = getArray();
        int len = elements.length;
        // 复制一个原数组的副本, 容量为原数组+1
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        // 添加元素到最后一个索引位置
        newElements[len] = e;
        // 将新数组赋值到原数组
        setArray(newElements);
        return true;
    } finally {
        // 释放锁
        lock.unlock();
    }
}
```

{% note success %}
### 添加指定索引元素
{% endnote %}
并发的添加操作, 在指定索引位置添加元素, 将当前在该位置的元素(如果有)和任何后续元素向右移动, 采用ReentrantLock锁来保证线程安全
```java
public void add(int index, E element) {
    final ReentrantLock lock = this.lock;
    // 加锁, 保证多个线程并发添加元素安全
    lock.lock();
    try {
        // 获取原数组
        Object[] elements = getArray();
        int len = elements.length;
        // 下标越界检查
        if (index > len || index < 0)
            throw new IndexOutOfBoundsException("Index: "+index+", Size: "+len);
        // 定义一个新的空数组
        Object[] newElements;
        // 要移动的索引
        int numMoved = len - index;
        // 插入的索引为数组末尾, 不需要移动, 直接拷贝数组
        if (numMoved == 0)
            newElements = Arrays.copyOf(elements, len + 1);
        // 插入的索引在数组中间, 需要移动数组元素
        else {
            // 初始化新数组长度
            newElements = new Object[len + 1];
            // 拷贝指定索引前元素
            System.arraycopy(elements, 0, newElements, 0, index);
            // 拷贝指定索引后元素
            System.arraycopy(elements, index, newElements, index + 1, numMoved);
        }
        // 上面移动操作完成后, 要插入的索引已经空出, 直接赋值
        newElements[index] = element;
        // 将新数组赋值到原数组
        setArray(newElements);
    } finally {
        // 释放锁
        lock.unlock();
    }
}
```