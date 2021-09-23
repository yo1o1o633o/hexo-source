---
title: HashMap源码解析
date: 2021-09-22 19:44:59
categories:
    - JAVA
tags:
    - JAVA
    - 源码
---

{% note success %}
### 基本信息
{% endnote %}
1. 非并发安全的Map容器
2. 可预估存储数据量提前指定初始容量避免扩容的性能损耗
3. 容量总是2的幂次方


{% note success %}
### 构造方法
{% endnote %}
自定义容量和负载因子, 容量不能超过MAXIMUM_CAPACITY, 容量会使用tableSizeFor方法计算出最近的2的幂次方数
```java
public HashMap(int initialCapacity, float loadFactor) {
    // 指定的初始容量非负
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
    // 如果指定的初始容量大于最大容量, 则为最大容量
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    // 负载因子为正
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}
```
自定义容量, 使用默认的负载因子
```java
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
```
所有参数使用默认值
```java
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```
用一个Map初始化此HashMap
```java
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

{% note success %}
### Hash计算
{% endnote %}
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

{% note success %}
### 添加Map
{% endnote %}
```java
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    // 准备添加的Map元素数量
    int s = m.size();
    if (s > 0) {
        // 数组没有初始化
        if (table == null) { // pre-size
            float ft = ((float)s / loadFactor) + 1.0F;
            int t = ((ft < (float)MAXIMUM_CAPACITY) ? (int)ft : MAXIMUM_CAPACITY);
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        // 如果要添加的元素超过阈值, 则直接进行扩容
        else if (s > threshold)
            resize();
        // 循环添加元素, 如果此时超过阈值, 那么在put方法里进行扩容
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```

{% note success %}
### 获取元素
{% endnote %}
对key进行hash运算, 调用获取结点方法(getNode)获取
```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

{% note success %}
### 获取元素结点
{% endnote %}
Key的Hash值和Key值都相等, 则表示找到元素, 可以返回
1. 判断Table数组是否非空
2. 通过Key的Hash计算数组下标, 根据判断数组下标是否有值
3. 判断数组下标对应的元素是否满足, 满足则返回结果
4. 如果是树结点, 则使用树结点方法getTreeNode尝试获取Node
5. 如果是链表, 则遍历链表找到对应的Node
6. 未找到则返回null
```java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // 数组非空同时通过hash获取到了元素
    if ((tab = table) != null && (n = tab.length) > 0 && (first = tab[(n - 1) & hash]) != null) {
        // 检查第一个Node是否是要获取的Node, 如果是则返回
        if (first.hash == hash && ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 第一个Node不是要获取的, 则判断链表或者红黑树是否有下一个指针, 有就继续处理, 没有则表示不存在直接跳出
        if ((e = first.next) != null) {
            // 第一个Node是红黑树Node, 调用getTreeNode获取元素
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 第一个Node是链表Node, 遍历链表找到元素
            do {
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

{% note success %}
### 添加元素
{% endnote %}
对Key进行Hash运算, 调用添加结点方法(putVal)
```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

{% note success %}
### 添加元素结点
{% endnote %}
```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 数组为空, 则初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 当前计算的数组下标为空, 则直接添加结点到该下标位置
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    // 索引下标有值, 出现了Hash冲突的Key
    else {
        Node<K,V> e; K k;
        // 判断下标位置的元素, 即第一个Node是否和要添加的Key相等
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 当前数组下标的元素是红黑树
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 当前数组下标的元素是链表
        else {
            // 遍历链表
            for (int binCount = 0; ; ++binCount) {
                // 遍历到了链表尾
                if ((e = p.next) == null) {
                    // 证明当前Key不存在, 创建结点放在链表尾部
                    p.next = newNode(hash, key, value, null);
                    // 转换红黑树阈值检验, 如果超出则转成红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // Key已经存在, 跳出循环
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // e不为null, e为已经存在的结点
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // 入参判断, 是否覆盖原值
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            // 返回旧值
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

{% note success %}
### 初始化和扩容
{% endnote %}
```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 数组已经到达最大容量了, 不能扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 计算新容量, 新容量为原容量的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ? (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    // 初始化一个新表, 用来迁移数据
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 遍历旧表
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            // 数组下标有值, 进行处理
            if ((e = oldTab[j]) != null) {
                // 把旧表的对应下标置空
                oldTab[j] = null;
                // 当前元素结点没有后续结点, 则直接放到新表中
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 当前是红黑树结点
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                // 当前是链表结点
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

{% note success %}
### 链表转红黑树
{% endnote %}
如果数组长度小于64时, 会进行扩容而不是转换
```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    // 数组长度小于64, 进行扩容
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```

{% note success %}
### 移除元素
{% endnote %}
```java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ? null : e.value;
}
```

{% note success %}
### 移除元素结点
{% endnote %}
```java
final Node<K,V> removeNode(int hash, Object key, Object value, boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 && (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        if (node != null && (!matchValue || (v = node.value) == value || (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

{% note success %}
### 清空Map
{% endnote %}
遍历数组进行置空
```java
public void clear() {
    Node<K,V>[] tab;
    modCount++;
    if ((tab = table) != null && size > 0) {
        size = 0;
        for (int i = 0; i < tab.length; ++i)
            tab[i] = null;
    }
}
```
