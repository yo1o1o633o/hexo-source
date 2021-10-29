---
title: 源码解析-CopyOnWriteArrayList
date: 2021-09-25 22:46:41
top: 2
categories:
    - JAVA
tags:
    - JAVA
    - 源码
---

{% note success %}
### 基本信息
{% endnote %}
- 线程安全, 采用写时复制策略(读写分离), 在对数组进行写操作时为数组创建新的副本, 消耗性能, 读取元素不加锁, 所以此容器适用于读多写少情况
- 并发写操作通过使用ReentrantLock锁来保证线程安全, 避免多线程操作拷贝出多份新数组
- 资源占用高, 写操作需要进行数组拷贝, 有额外的内存资源占用
- 无法保证数据实时一致性, 只能保证数据最终一致性
- 没有初始化容量, 每新增一个元素就申请原数组+1长度的新数组进行拷贝

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
1. 加ReentrantLock锁, 保证只有一个线程操作该数组
2. 拷贝一个新数组
3. 在新数组上对指定索引进行赋值
4. 将新数组的引入赋值给原数组
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
在数组最后追加一个元素
1. 加ReentrantLock锁, 保证只有一个线程操作该数组
2. 拷贝一个新数组, 容量为原数组长度+1
3. 将新元素放到新数组的最后一位
4. 将新数组的引入赋值给原数组
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
1. 加ReentrantLock锁, 保证只有一个线程操作该数组
2. 计算要移动的元素数量
3. 拷贝指定索引前元素
4. 拷贝指定索引后元素
5. 将指定元素放到新数组空出的位置上
4. 将新数组的引入赋值给原数组
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
        // 要移动的元素数量
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

{% note success %}
### 移除指定索引元素
{% endnote %}
并发的移除操作, 移除指定索引下标的元素, 将后续元素向左移动, 采用ReentrantLock锁来保证线程安全
```java
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    // 加锁, 保证多个线程并发移除元素安全
    lock.lock();
    try {
        // 获取原数组
        Object[] elements = getArray();
        int len = elements.length;
        // 获取要移除的元素值
        E oldValue = get(elements, index);
        int numMoved = len - index - 1;
        // 要移除的是最后一个元素
        if (numMoved == 0)
            setArray(Arrays.copyOf(elements, len - 1));
        // 要移除的元素在数组中间
        else {
            // 定义一个原数组-1长度的数组
            Object[] newElements = new Object[len - 1];
            // 拷贝指定索引前元素
            System.arraycopy(elements, 0, newElements, 0, index);
            // 拷贝指定索引后元素
            System.arraycopy(elements, index + 1, newElements, index, numMoved);
            // 将新数组赋值到原数组
            setArray(newElements);
        }
        // 返回移除的元素
        return oldValue;
    } finally {
        // 释放锁
        lock.unlock();
    }
}
```
