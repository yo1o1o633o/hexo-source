---
title: 源码解析-HashMap
date: 2021-09-22 19:44:59
top: 3
categories:
    - JAVA
tags:
    - JAVA
    - 源码
---
<script type="text/javascript">
    // 禁止右键菜单
    // true是允许，false是禁止
    document.oncontextmenu = function(){ return false; };
    // 禁止文字选择
    document.onselectstart = function(){ return false; };
    // 禁止复制
    document.oncopy = function(){ return false; };
    // 禁止剪切
    document.oncut = function(){ return false; };
    // 禁止粘贴
    document.onpaste = function(){ return false; };
    // 禁止键盘事件
    // document.onkeydown = function(){ return false; };
</script>

{% note success %}
### 基本信息
{% endnote %}
- 使用数组+链表+红黑树实现, 当链表长度超过8且数组长度超过64时, 会将链表转成红黑树
- Key和Value可以为null, Key为null则永远保存在数组索引0的位置
- 扩容时链表采用高位参与运算, 且尾插形式迁移

{% note success %}
### 关键参数
{% endnote %}
threshold: 扩容阈值, 这个数值是根据初始容量和负载因子计算得到的, 即默认 16 * 0.75 = 12, 当数组元素超出这个阈值时就会进行扩容, 每次扩容2倍容量

loadFactor: 负载因子, 用来计算扩容阈值. 这个值不建议更改
- 数组的长度决定Hash的随机性, 数组容量越大则Hash的随机分布越均匀, Hash碰撞组装链表的几率越低
- 当内存紧张而效率要求不高时, 可以增加负载因子loadFactor的值
- 内存空间很多而对时间效率要求很高,可以降低负载因子loadFactor的值

size: 实际的键值对数量
```java
// 保存元素的数组, 每个元素对应一个链表
transient Node<K,V>[] table;
// 实际的键值对数量
transient int size;
// 扩容阈值, 数组最多容纳元素个数, 超出即触发扩容
int threshold;
// 负载因子, 默认0.75
final float loadFactor;
```

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
    this.loadFactor = DEFAULT_LOAD_FACTOR;
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
1. 获取Key的hashCode
2. 高位参与运算
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
3. 实际使用, 因为长度总是2的幂次方, 这个操作等同于hash % n, 即Hash值和长度取模, 而使用&操作比取模操作效率更高
```java
(n - 1) & hash
```

示例:
h = key.hashCode()
11111111 11111111 11110000 11101010

h >>> 16:            
00000000 00000000 11111111 11111111

h ^ h >>> 16:    
11111111 11111111 11110000 11101010  
00000000 00000000 11111111 11111111             ^    
\-------------------------------------------------------------------------------------
11111111 11111111 00001111 00010101 

(n - 1) & hash
11111111 11111111 00001111 00010101 
00000000 00000000 00000000 00001111             &
\-------------------------------------------------------------------------------------
00000000 00000000 00000000 00000101

结果为5

{% note success %}
### 添加Map
{% endnote %}
```java
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    // 准备添加的Map元素数量
    int s = m.size();
    if (s > 0) {
        // 数组没有初始化
        if (table == null) {
            float ft = ((float)s / loadFactor) + 1.0F;
            int t = ((ft < (float)MAXIMUM_CAPACITY) ? (int)ft : MAXIMUM_CAPACITY);
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        // 如果要添加的元素超过阈值, 则直接进行扩容
        else if (s > threshold)
            resize();
        // 循环添加元素
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
    // 数组为空, 则执行resize进行扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 根据Key计算数组索引, 当前索引元素为空, 则直接新建结点添加到该索引位置
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
    // 超过扩容阈值, 进行扩容
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
    // 原数组
    Node<K,V>[] oldTab = table;
    // 扩容前的数组长度
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    // 扩容前的扩容阈值
    int oldThr = threshold;
    // 定义新的数组容量, 新的扩容阈值
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 数组已经到达最大容量了, 不能扩容, 同时将threshold设置为最大值, 以后就不会扩容了
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 计算新容量, 新容量为原容量的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 扩容阈值也设为原来的2倍
            newThr = oldThr << 1; // double threshold
    }
    // 使用初始扩容阈值初始化容量
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    // 使用默认参数设置
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
                        // 计算高位是1还是0, 这里e.hash & oldCap 计算得到的是0或者原容量(16)
                        // 分别组装成2个链表
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
                    // 将两个链表分别保存到新数组中
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
示例:
如有以下两个Key的Hash值
11111111 11111111 00001111 00000101
11111111 11111111 00001111 00010101
通过上面的计算可知, 虽然这两个Key的Hash值不同, 但是最后4位二进制相同. 索引进行&运算后会放在同一个元素桶里, 以链表的形式存储
即:
11111111 11111111 00001111 00000101
00000000 00000000 00000000 00001111
\-------------------------------------------------------------------------------------
00000000 00000000 00000000 0000<font color=#FF0000>0101</font>  =  5

11111111 11111111 00001111 00010101
00000000 00000000 00000000 00001111
\-------------------------------------------------------------------------------------
00000000 00000000 00000000 0000<font color=#FF0000>0101</font>  =  5

那么此时需要扩容二倍, 索引的计算变成如下, 因为容量多了1位参与了运算
11111111 11111111 00001111 00000101
00000000 00000000 00000000 00011111
\-------------------------------------------------------------------------------------
00000000 00000000 00000000 000<font color=#FF0000>00101</font>  =  5

11111111 11111111 00001111 00010101
00000000 00000000 00000000 00011111
\-------------------------------------------------------------------------------------
00000000 00000000 00000000 000<font color=#FF0000>10101</font>  =  21

在原容量16的情况下, 最后4位参与位运算, 在扩容2倍后容量为32时, 最后5位参与运算.
那么可知, 当扩容前在同一个元素位置的链表中的元素, 在容量扩大2倍时, 要么在原来的索引位置即5, 要么就在原索引+原容量的索引位置上即5 + 16 = 21

只需要判断Key的Hash值的第5位是0还是1, 即可知道Key在扩容后的元素索引位置
代码中e.hash & oldCap 判断等于0还是旧容量, 这里是与oldCap = 16计算, 而在获取元素时是与(数组长度-1)计算
11111111 11111111 00001111 00000101
00000000 00000000 00000000 00010000
\-------------------------------------------------------------------------------------
结果为0


11111111 11111111 00001111 00010101
00000000 00000000 00000000 00010000
\-------------------------------------------------------------------------------------
结果为1
通过上述判断将一个链表根据高位的不同拆分成了两个链表

{% note success %}
### 链表转红黑树
{% endnote %}
如果数组长度小于64时, 会进行扩容而不是转换
```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    // 数组长度小于64, 才会进行扩容
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
        // 要移除的是头结点
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        // 在链表或者红黑树中找到要移除的结点
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
        // 移除结点操作
        if (node != null && (!matchValue || (v = node.value) == value || (value != null && value.equals(v)))) {
            // 红黑树结点的移除
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            // 链表头结点的移除
            else if (node == p)
                tab[index] = node.next;
            // 链表中的结点移除
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
