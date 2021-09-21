---
title: ConcurrentHashMap源码解析
categories:
    - JAVA
tags:
    - JAVA
    - 源码
---

{% note success %}
### 基本信息
{% endnote %}
- 利用CAS自旋锁+Synchronized保证并发安全的Map
- 使用数组+链表+红黑树实现
- key和value不能为null

1. 构造函数进行初始化数组容量, 会根据传入容量进行计算. 实际容量高于传入的容量
2. 支持多线程同时扩容, 其他线程发现当前数组在扩容时会进行协助扩容
3. 元素计数采用baseCount和数组形式以支持多线程并发更新计数
4. 获取元素时不需要加锁

{% note success %}
### 常量和变量
{% endnote %}

```java
// 表的最大可能容量
private static final int MAXIMUM_CAPACITY = 1 << 30;
// 表的默认初始容量
private static final int DEFAULT_CAPACITY = 16;
// 链表转成红黑树的阈值, 将元素添加到至少具有这么多节点的链表时, 链表会转换为树. 该值必须大于2, 并且应至少为8
static final int TREEIFY_THRESHOLD = 8;
// 红黑树收缩为链表的阈值
static final int UNTREEIFY_THRESHOLD = 6;
// 将链表转成红黑树的数组容量阈值, 数组容量超过这个阈值会转红黑树, 否则会对数组扩容
static final int MIN_TREEIFY_CAPACITY = 64;
// 扩容迁移数组过程中, 每个线程处理的最小数据元素数, 即每个线程领取的任务数量
private static final int MIN_TRANSFER_STRIDE = 16;
// sizeCtl 中用于生成标记的位数. 对于32位数组, 必须至少为6
private static int RESIZE_STAMP_BITS = 16;
// 协助扩容的最大线程数
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
// sizeCtl中记录大小标记的位移位
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

// 保存桶的数组, 第一次插入数据延迟初始化, 始终是2的幂次方
transient volatile Node<K,V>[] table;
// 用于扩容时迁移使用, 只会在扩容时非空
private transient volatile Node<K,V>[] nextTable;
// 基本计数保存遍历, 在没有线程争用时使用, 也会在表初始化竞争期间的后备使用, 采用CAS更新
private transient volatile long baseCount;
// 关键状态参数
private transient volatile int sizeCtl;
// 扩容时下一个要处理的索引
private transient volatile int transferIndex;
// 调整大小和/或创建 CounterCell 时使用的自旋锁(通过 CAS 锁定)
private transient volatile int cellsBusy;
// 计数器数据表, 当baseCount线程竞争时使用, 当非空时, 大小是 2 的幂。
private transient volatile CounterCell[] counterCells;
```

{% note success %}
### 关键参数sizeCtl
{% endnote %}

1. sizeCtl为0, 表示数组未初始化, 且初始容量为16
2. sizeCtl为正数, 如果数组未初始化，则记录的是数组初始容量，如果已经初始化，则记录数组扩容阈值（初始容量*0.75）
3. sizeCtl为-1， 表示数组正在初始化
4. sizeCtl小于0且不是-1，表示当前正在扩容的线程数. -(1 + 扩容线程数)

{% note success %}
### 获取Key的Hash值
{% endnote %}
```java
static final int HASH_BITS = 0x7fffffff;
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```

{% note success %}
### 计算初始容量
{% endnote %}
返回一个大于等于传入数值的2的幂次方数
```java
private static final int tableSizeFor(int c) {
    int n = c - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

{% note success %}
### 构造方法
{% endnote %}
空参的构造方法, 会在对Map操作时使用默认容量(16)进行初始化
```java
public ConcurrentHashMap() {
}
```
传入初始容量构造方法, 根据传入的初始容量计算实际的初始容量, 并向sizeCtl参数赋值, 因为数组还未初始化, sizeCtl表示初始容量, tableSizeFor方法会返回一个2的幂次方数
示例: 
initialCapacity + (initialCapacity >>> 1) + 1 = 16 + 16 / 2 + 1 = 25
tableSizeFor(25) = 32
```java
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}
```
传入一个Map的构造方法, 初始化容量为默认容量, 底层调用的是putAll方法
```java
public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this.sizeCtl = DEFAULT_CAPACITY;
    putAll(m);
}
```
使用传入的初始容量, 扩容阈值, 调用下边的构造方法
```java
public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, 1);
}
```
使用传入的初始容量, 扩容阈值计算初始化容量
```java
public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
    // 入参判断
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    // 估计的并发更新线程数, 根据这个值调整初始容量大小
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    // 如: initialCapacity = 16, loadFactor = 0.75
    // (long)(1.0 + (long)initialCapacity / loadFactor) = 1.0 + 16 / 0.75 = 22.33
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    // tableSizeFor((int)size) = tableSizeFor(22.33) = 32
    int cap = (size >= (long)MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : tableSizeFor((int)size);
    // 向sizeCtl参数赋值, 因为数组还未初始化, sizeCtl表示初始容量
    this.sizeCtl = cap;
}
```

{% note success %}
### 获取Map元素数量
{% endnote %}
通过sumCount()方法计算Map元素总数, 总数是可能出现负数或者大于Integer.MAX_VALUE的情况, 如果发送就返回0或者Integer.MAX_VALUE
```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 : (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE : (int)n);
}
```

{% note success %}
### 判断Map是否为空
{% endnote %}
通过sumCount()方法计算Map元素总数来判断
```java
public boolean isEmpty() {
    return sumCount() <= 0L; // ignore transient negative values
}
```

{% note success %}
### 获取元素
{% endnote %}
1. 计算Key的Hash值
2. 通过Hash值在数组中获取元素
3. 如果存在则比较Key值是否相等, 相等则表示找到元素,直接返回Value
4. 如果hash为负表示正在扩容或者时红黑树结点, 调用结点find方法查找并返回
5. 如果是链表, 则遍历链表找到对于的key值并返回Value
6. 获取元素不需要加锁, tabAt采用Volatile方式读取内存的数据, 保证线程可见性, 如果是扩容则会使用find方法找到nextTable中的元素
```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    // 计算Key的Hash
    int h = spread(key.hashCode());
    // 数组不为null
    // 数组长度不为0
    // 根据Hash值计算的索引位置元素不为null
    if ((tab = table) != null && (n = tab.length) > 0 && (e = tabAt(tab, (n - 1) & h)) != null) {
        // 当前数组元素就是要获取的key, 直接返回
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 如果Hash为负数, 则数组可能在扩容或者元素是一个红黑树结点
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        // 上述都不满足, 则当前元素结点是链表, 在链表中遍历查找对应的Key
        while ((e = e.next) != null) {
            if (e.hash == h && ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

{% note success %}
### KEY是否存在
{% endnote %}
调用get方法进行判断
```java
public boolean containsKey(Object key) {
    return get(key) != null;
}
```

{% note success %}
### VALUE是否存在
{% endnote %}
找到一个Value, 最坏可能遍历整个Map, 性能很慢
通过Traverser对象构建的迭代器进行查找
```java
public boolean containsValue(Object value) {
    // value不能为null
    if (value == null)
        throw new NullPointerException();
    Node<K,V>[] t;
    if ((t = table) != null) {
        Traverser<K,V> it = new Traverser<K,V>(t, t.length, 0, t.length);
        for (Node<K,V> p; (p = it.advance()) != null; ) {
            V v;
            if ((v = p.val) == value || (v != null && value.equals(v)))
                return true;
        }
    }
    return false;
}
```

{% note success %}
### 添加元素put()
{% endnote %}
内部调用putVal方法, key和value不能为null, 第三个参数表示发现key已存在是否覆盖, 传入false表示覆盖原value
```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
```

{% note success %}
### 添加元素putVal()
{% endnote %}
添加元素操作会有多线程并发处理情况, 采用CAS赋值和synchronized加锁的形式保证线程安全
1. 如果table为null, 则表示还未初始化, 调用初始化方法
2. 如果table已经初始化, 同时hash对应的数组元素为null, 则直接进行CAS赋值操作, 成功则跳出循环, 失败则重新进入循环
3. 如果当前元素hash为-1, 则表示当前元素正在进行扩容移动, 进行协助扩容
4. 以上都不成立, 表明当前数组元素有值, 对这个元素加synchronized后进行遍历处理
    - 链表: 如果key在链表中存在, 则进行覆盖, 否则添加到链表尾部
    - 红黑树: 使用红黑树对象方法进行处理
5. 检查是否符合链表转成红黑树条件, 符合进行转换
6. 元素计数操作同时进行扩容检查
```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // 校验下key和value不能为null
    if (key == null || value == null) throw new NullPointerException();
    // 获取key的Hash值
    int hash = spread(key.hashCode());
    // 初始化操作数
    int binCount = 0;
    // 自旋操作, 如果不满足break或者return逻辑, 则会自旋
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 判断当前数组未初始化, 进行初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // 判断根据Hash值获取到的元素为null, 则以CAS形式将Key保存在这个索引位置上, CAS如果赋值失败了则表明有其他线程赋值成功了, 则进行下一次循环, 下一次循环时此if条件不成立进入其他逻辑处理
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                // CAS形式添加到数组成功, 直接返回
                break;                  
        }
        // 当前Hash值为MOVED, 表示数组正在进行扩容, 进行协助扩容操作, 操作完成后再进行下一次循环
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        // 当前元素索引位置有值, 则进行链表或者红黑树的处理
        else {
            V oldVal = null;
            // 对当前数组元素加锁
            synchronized (f) {
                // 加锁后再次判断获取到元素是否发生了变化, 因为在上面逻辑获取到元素和加锁之间有可能出现其他线程对这个元素进行了修改
                if (tabAt(tab, i) == f) {
                    // Hash大于等于0, 表示当前元素是一个链表结点
                    if (fh >= 0) {
                        binCount = 1;
                        // 遍历链表, 同时记录操作数
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 遍历中判断key是否相等, 相等则表示Key已存在
                            if (e.hash == hash && ((ek = e.key) == key || (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                // 根据入参onlyIfAbsent, 是否对已存在的key覆盖value
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            // 表示链表中不存在该key, 则将新的key添加到链表尾部
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value, null);
                                break;
                            }
                        }
                    }
                    // 元素结点是红黑树结点
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        // 则使用putTreeVal方法将key添加到树种
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            // 当使用链表保存数据时, 每次遍历都会自增binCount, 即记录链表中的元素个数, 如果有值同时这个值大于等于转换红黑树的条件时, 调用treeifyBin进行转换
            if (binCount != 0) {
                // binCount >= 8
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 添加元素操作完成后, 进行元素个数记录操作
    addCount(1L, binCount);
    return null;
}
```

{% note success %}
### 链表转红黑树
treeifyBin(Node<K,V>[] tab, int index)
{% endnote %}

进入转换方法, 判断Table数组元素数量如果小于64则只进行扩容, 大于等于64个则转成红黑树
```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        // 先判断数组Table的元素个数是否小于64个, 小于的话调用尝试扩容方法
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1);
        // 大于等于64个元素进行转成红黑树操作
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val, null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```

{% note success %}
### 批量添加元素
{% endnote %}
传入一个Map, 将Map中元素全部添加到当前Map中, 先根据要添加Map的元素个数进行尝试扩容, 然后调用putVal()方法循环添加元素
```java
public void putAll(Map<? extends K, ? extends V> m) {
    // 尝试调整Map容量, 以容纳要添加的Map
    tryPresize(m.size());
    // 循环添加元素
    for (Map.Entry<? extends K, ? extends V> e : m.entrySet())
        putVal(e.getKey(), e.getValue(), false);
}
```

{% note success %}
### 移除元素
{% endnote %}
内部调用替换结点方法, value替换成null
```java
public V remove(Object key) {
    return replaceNode(key, null, null);
}
```

{% note success %}
### 替换元素结点
{% endnote %}
和putVal()基本一致, 只是没有了初始化table操作, 同时需要根据入参来选择替换还是移除元素操作, 处理完成后重新统计元素数量
```java
final V replaceNode(Object key, V value, Object cv) {
    int hash = spread(key.hashCode());
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 如果数组未初始化, 或者数组没有元素, 或者未找到key对应的元素则不处理, 直接跳出循环
        if (tab == null || (n = tab.length) == 0 || (f = tabAt(tab, i = (n - 1) & hash)) == null)
            break;
        // 如果当前数组元素正在扩容迁移, 则协助扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            boolean validated = false;
            // 对元素加锁, 保证只有一个线程操作
            synchronized (f) {
                // 再次判断当前元素是否发生变化, 因为在上面逻辑获取到元素和加锁之间有可能出现其他线程对这个元素进行了修改
                if (tabAt(tab, i) == f) {
                    // 当前元素是链表结点
                    if (fh >= 0) {
                        validated = true;
                        for (Node<K,V> e = f, pred = null;;) {
                            K ek;
                            if (e.hash == hash && ((ek = e.key) == key || (ek != null && key.equals(ek)))) {
                                V ev = e.val;
                                if (cv == null || cv == ev || (ev != null && cv.equals(ev))) {
                                    oldVal = ev;
                                    if (value != null)
                                        e.val = value;
                                    else if (pred != null)
                                        pred.next = e.next;
                                    else
                                        setTabAt(tab, i, e.next);
                                }
                                break;
                            }
                            pred = e;
                            if ((e = e.next) == null)
                                break;
                        }
                    }
                    // 元素结点是红黑树结点
                    else if (f instanceof TreeBin) {
                        validated = true;
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> r, p;
                        if ((r = t.root) != null && (p = r.findTreeNode(hash, key, null)) != null) {
                            V pv = p.val;
                            if (cv == null || cv == pv || (pv != null && cv.equals(pv))) {
                                oldVal = pv;
                                if (value != null)
                                    p.val = value;
                                else if (t.removeTreeNode(p))
                                    setTabAt(tab, i, untreeify(t.first));
                            }
                        }
                    }
                }
            }
            if (validated) {
                if (oldVal != null) {
                    if (value == null)
                        addCount(-1L, -1);
                    return oldVal;
                }
                break;
            }
        }
    }
    return null;
}
```

{% note success %}
### 清空Map
{% endnote %}
遍历数组, 对每个元素进行清空操作, 如果元素为null则跳过, 如果当前元素正在进行扩容, 则协助扩容后重置数组遍历索引, 重新遍历数组进行清空操作
```java
public void clear() {
    // 记录操作数
    long delta = 0L; // negative number of deletions
    int i = 0;
    Node<K,V>[] tab = table;
    while (tab != null && i < tab.length) {
        int fh;
        Node<K,V> f = tabAt(tab, i);
        // 元素为null直接跳过
        if (f == null)
            ++i;
        // 当前元素正在扩容迁移, 进行协助扩容操作, 操作完成后重置索引为0, 重新遍历数组进行处理
        else if ((fh = f.hash) == MOVED) {
            tab = helpTransfer(tab, f);
            i = 0; // restart
        }
        // 元素不为null, 则处理链表或者红黑树
        else {
            synchronized (f) {
                // 再次判断当前元素是否发生变化, 因为在上面逻辑获取到元素和加锁之间有可能出现其他线程对这个元素进行了修改
                if (tabAt(tab, i) == f) {
                    // 如果是链表则取链表头结点, 如果是红黑树则取first结点
                    Node<K,V> p = (fh >= 0 ? f : (f instanceof TreeBin) ? ((TreeBin<K,V>)f).first : null);
                    while (p != null) {
                        --delta;
                        p = p.next;
                    }
                    setTabAt(tab, i++, null);
                }
            }
        }
    }
    // 根据操作数进行元素数量处理
    if (delta != 0L)
        addCount(delta, -1);
}
```

{% note success %}
### 初始化Table
initTable()
{% endnote %}
数组未初始化, 则进行初始化操作, 初始化依赖sizeCtl参数
sizeCtl如果小于0则表示已经有线程在初始化, 当前线程让出CPU, 否则采用CAS方式争夺初始化操作权, 多线程并发情况, 同时进行CAS竞争, 竞争成功的进行初始化操作, 竞争失败的让除CPU时间片等待初始化完成
竞争成功的线程将sizeCtl设置成-1, 表明数组正在进行初始化, 初始化完成后将sizeCtl设置为扩容阈值
计算扩容阈值使用右移操作替代除法, 提高运算效率: sc = n - (n >>> 2), 
示例: 
n = 16
sc = n - (n >>> 2) = 16 - 16 / 2 / 2 = 12, 即16 * 0.75 = 12
扩容阈值为12, 当元素数量达到12个时会触发扩容操作

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        // 小于0表示其他线程正在初始化Table, 当前线程调用yield方法让出CPU时间片
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        // 以原子性操作对sizeCtl赋值成-1, 表示正在扩容
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    // 初始化容量
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    // 创建数组
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // 计算扩容阈值
                    sc = n - (n >>> 2);
                }
            } finally {
                // 初始化完成, sizeCtl保存扩容阈值
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

{% note success %}
### 添加计数和扩容检查
addCount(long x, int check)
{% endnote %}
此方法执行两个操作
- 处理元素数量计数, 增加或者减少
- 判断是否需要扩容, 并进行扩容

```java
// x表示数量变化数, check大于0表示需要进行扩容检查
private final void addCount(long x, int check) {
    // CounterCell数组就是保存元素个数的数组
    CounterCell[] as; long b, s;
    // 如果CounterCell数组不为空, 或者CAS对BaseCount设置值失败, 第一次调用时数组是空的, 主要判断第二个条件baseCount是否设置成功
    if ((as = counterCells) != null || !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        // ThreadLocalRandom.getProbe() 得到线程的探针哈希值, 如果发生hash冲突则会生成一个新的不同hash, 和Map的hash不同的是, 此处会随机生成hash
        // 数组为空, 或者数组没有元素, 或者当前线程的探针哈希值对应元素不存在
        if (as == null || (m = as.length - 1) < 0 || (a = as[ThreadLocalRandom.getProbe() & m]) == null || !(uncontended = U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            // 关键方法, 进行元素个数添加操作
            fullAddCount(x, uncontended);
            return;
        }
        // 如果是不需要检查扩容的调用, 则此处可以直接返回了
        if (check <= 1)
            return;
        // 统计数组元素个数, 为后面的扩容检查做准备
        s = sumCount();
    }
    // 判断是否要对数组容量进行调整
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        // 此时sizeCtl保存的是扩容阈值, s为当前数组元素个数, 此处判断是否需要扩容
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null && (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            // 小于0则表示当前有线程在扩容, 进行协助扩容操作
            if (sc < 0) {
                // sc >>> RESIZE_STAMP_SHIFT) != rs 扩容已经结束
                // sc == rs + 1                     扩容已经结束
                // sc == rs + MAX_RESIZERS          协助扩容线程已经到最大值
                // (nt = nextTable) == null         扩容已经结束
                // transferIndex <= 0               扩容任务已经领取完毕, 没有需要处理的元素了
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 || sc == rs + MAX_RESIZERS || (nt = nextTable) == null || transferIndex <= 0)
                    // 跳出循环
                    break;
                // 尝试进行协助扩容, 如果CAS竞争到权限, 则执行transfer扩容方法
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            // 如果当前没有线程在扩容, 则sc为整数, 当前线程尝试进行扩容
            else if (U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2))
                // 扩容方法
                transfer(tab, null);
            // 重新计数, 判断是否需要下一次循环
            s = sumCount();
        }
    }
}
```



{% note success %}
### 添加计数
fullAddCount(long x, boolean wasUncontended)
{% endnote %}
自旋处理
1. CounterCell[]数组为初始化, 则执行初始化操作
2. 如果当前线程并发竞争激烈, 则尝试对baseCount进行CAS累加操作
3. CounterCell[]数组已经初始化
    - 如果当前索引有值, 则进行CAS累加操作
    - 如果当前索引为空, 则进行CAS赋值操作
    - 当上述CAS操作都失败时, 会判断当前数组尝试是否小于CPU核数, 如果小于则进行扩容 

```java
private final void fullAddCount(long x, boolean wasUncontended) {
    int h;
    // 线程的探针哈希值未初始化, 则进行初始化操作, 然后获取线程的探针哈希值->h, 是一个随机数
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();      // force initialization
        h = ThreadLocalRandom.getProbe();
        wasUncontended = true;
    }
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        CounterCell[] as; CounterCell a; int n; long v;
        // 数组已经初始化
        if ((as = counterCells) != null && (n = as.length) > 0) {
            // 根据当前线程probe获取CounterCell[]数组下标元素
            if ((a = as[(n - 1) & h]) == null) {
                if (cellsBusy == 0) {            // Try to attach new Cell
                    // 构造CounterCell对象, 保存传入的数量
                    CounterCell r = new CounterCell(x); // Optimistic create
                    // CAS操作, 竞争对数组的操作, 防止其他线程并发操作
                    if (cellsBusy == 0 && U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                        boolean created = false;
                        try {               // Recheck under lock
                            CounterCell[] rs; int m, j;
                            // 再次判断当前索引位置没有元素, 将CounterCell对象放到对应的索引位置上
                            if ((rs = counterCells) != null && (m = rs.length) > 0 && rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        // 创建成功,跳出循环
                        if (created)
                            break;
                        // 说明当前数组索引不为null, 则进行下一次循环
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
            // 当前索引下标元素不为null, 使用CAS对其进行累加操作
            else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                break;
            // 如果有其他线程建立了新的counterCells数组或者counterCells数组元素大于CPU核数, 线程的并发数不会超出CPU核数
            // 即n >= NCPU是不会进行扩容
            else if (counterCells != as || n >= NCPU)
                // 标记当前线程的循环失败
                collide = false;            // At max size or stale
            // 恢复collide状态, 标记下次循环进行扩容
            else if (!collide)
                collide = true;
            // 表明当前数组容量不足, 线程竞争激烈, 标记并进行扩容
            else if (cellsBusy == 0 && U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                try {
                    if (counterCells == as) {// Expand table unless stale
                        // 扩容2倍容量
                        CounterCell[] rs = new CounterCell[n << 1];
                        // 迁移到扩容后的数组
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        counterCells = rs;
                    }
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                // 继续下一次循环
                continue;                   // Retry with expanded table
            }
            // 重新计算线程的prode随机值, 即下一次计算的元素下标会变化
            h = ThreadLocalRandom.advanceProbe(h);
        }
        // CAS形式初始化数组, 初始化完成后将传入的数量设置进去, 成功后退出循环
        else if (cellsBusy == 0 && counterCells == as && U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
            boolean init = false;
            try {
                // 初始化CounterCell数组, 数组大小2
                if (counterCells == as) {
                    CounterCell[] rs = new CounterCell[2];
                    rs[h & 1] = new CounterCell(x);
                    counterCells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            if (init)
                break;
            }
        // 尝试对baseCount进行累加操作, 操作成功就跳出循环, 否则进行下次循环进入对数组操作逻辑
        else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
            break;                          // Fall back on using base
    }
}
```

{% note success %}
### 协助扩容
{% endnote %}
```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab && (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 || sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

{% note success %}
### 尝试扩容用以保存给定数量的元素
{% endnote %}
size不需要完全准确
```java
private final void tryPresize(int size) {
    // 根据给定size计算需要的数组容量, 如果size超出最大容量的一半, 则取最大容量值, 否则扩容至0.75倍后的2的幂次
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY : tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;
        // 数组未初始化, 进行初始化然后进行下一次循环
        if (tab == null || (n = tab.length) == 0) {
            // 取sizeCtl和刚刚计算出的容量中较大的值作为扩容容量
            n = (sc > c) ? sc : c;
            // 尝试将sizeCtl设置成-1, 即竞争初始化数组权限
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    // 再次判断数组table未发生变化
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        // 初始化数组
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        // 计算出新的扩容阈值
                        sc = n - (n >>> 2);
                    }
                } finally {
                    // 赋值新的扩容阈值
                    sizeCtl = sc;
                }
            }
        }
        // 已经不能再调整容量了
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        // 已经初始化了数组
        else if (tab == table) {
            // 返回用于调整大小为n的标记位
            int rs = resizeStamp(n);
            if (sc < 0) {
                Node<K,V>[] nt;
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 || sc == rs + MAX_RESIZERS || (nt = nextTable) == null || transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}
```

{% note success %}
### 扩容操作
{% endnote %}
1. 根据当前服务器CPU计算每个线程每次处理的元素个数(数组上的一段元素)
2. 从数组末尾, 以CAS形式尝试竞争一段元素的处理权, 此操作是从数组末尾向前取值. 如果竞争失败了就尝试取前一段元素
3. 竞争到处理权限后, 开始进行迁移操作, 根据元素结点的类型是链表还是红黑树进行分别处理
4. 迁移完成后使用ForwardingNode结点进行占位, 表示当前元素结点已经处理完成

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // 如果CPU核数大于1, 则当前 stride = 数组大小/8/CPU核数, 如果是单核服务器则stride为数组大小
    // 计算结果如果小于16, 则赋值16, 即最小每个线程处理16个元素
    // 此计算保证每个线程处理的段大小相同, 避免迁移任务出现不均匀
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE;
    // 要迁移的数组为null, 则创建2倍大的新数组
    if (nextTab == null) {
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    int nextn = nextTab.length;
    // 创建占位结点, 该结点指向新数组Table
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    // 标识位, 标记是否需要向前推进, 当前线程如果没有竞争到处理这一段数据, 那么就要向前推进尝试获取前一段
    boolean advance = true;
    // 标记扩容操作是否完成
    boolean finishing = false; // to ensure sweep before committing nextTab
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        // 需要向前推进, 循环尝试竞争要处理的段
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            // 已经到数组起始位了, 表示都已经被处理了
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            // 使用CAS形式开始竞争当前线程要处理的段, 每段的大小是stride个, 在数组上从后向前推进, 如果没抢到就向前推进继续抢
            else if (U.compareAndSwapInt(this, TRANSFERINDEX, nextIndex, nextBound = (nextIndex > stride ? nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            // 扩容完成, 将新数组赋值给原数组, 同时更新新的扩容阈值
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            // 使用CAS对sizeCtl的低16位进行-1操作, 表示当前线程处理完成
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                // 第一个线程开始扩容时会设置sizeCtl为resizeStamp(n) << RESIZE_STAMP_SHIFT
                // 每一个线程加入进来sizeCtl + 1
                // 每一个线程完成后退出sizeCtl - 1
                // 当最后一个线程退出后一定会有(sc - 2) == resizeStamp(n) << RESIZE_STAMP_SHIFT)
                // 如果这个条件不满足, 说明扩容已经结束了
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                // 标记为扩容完成, 再次检查整个数组, 没必要
                i = n; // recheck before commit
            }
        }
        // 当前数组元素为null, 直接以CAS形式放置占位结点表示已处理
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // 当前结点正在进行移动, 表示有其他线程处理了这个元素, 那么重新循环并继续向前推进获取前一段进行处理
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        // 迁移当前元素
        else {
            // 对当前元素加锁, 保证一个线程操作
            synchronized (f) {
                // 二次校验, 防止加锁前当前元素发生变化
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    // 当前元素是链表
                    if (fh >= 0) {
                        // 计算头结点是高位0结点还是高位1结点
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        // 如果头结点是高位0结点, 那么这里找到链表尾部连续的高位1结点, 反之亦然, 赋值给lastRun, runBit
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        // 链表尾部连续结点是高位0结点, 那么lastRun赋值给ln, ln高位0链表
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        // 链表尾部连续结点是高位1结点, 那么lastRun赋值给hn, hn高位1链表
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        // 采用头插形式将链表中的元素根据高位0或者高位1保存到ln和hn链表中, 即根据高位是0或者1拆分成了两个链表
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        // 将高位0的链表插入到新数组中, 索引位置和原数组相同
                        setTabAt(nextTab, i, ln);
                        // 将高位1的链表插入到新数组中, 索引位置是原数组索引位置间隔数组长度的索引位
                        setTabAt(nextTab, i + n, hn);
                        // 给当前元素设置为占位结点, 表示处理完成
                        setTabAt(tab, i, fwd);
                        // 标记向前推进
                        advance = true;
                    }
                    // 当前元素是红黑树
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>(h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) : (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) : (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```
示例:
```java
int runBit = fh & n;
```
使用高位与操作
11111111 11111111 11111111 0<font color=#FF0000>0</font>011010
00000000 00000000 00000000 0<font color=#FF0000>1</font>000000
结果为0

11111111 11111111 11111111 0<font color=#FF0000>1</font>011010
00000000 00000000 00000000 0<font color=#FF0000>1</font>000000
结果为1

链表结点迁移
1. 计算头结点是高位0结点还是高位1结点
2. 找出与头结点高位不相同的连续链表尾部结点
3. 遍历链表, 根据元素结点是高位0还是高位1采用头插法分别保存在两个链表中ln,hn
4. 将两个链表分别保存到新数组的索引位置, 高位0保存在和原数组相同的索引位置, 高位1保存到原数组索引+原数组长度的索引位置

红黑树结点迁移
1. 计算每个红黑树结点, 判断是高位0还是高位1
2. 根据高位0和高位1拆分成两个链表, 新链表采用尾插法
3. 将两个新链表保存到新数组中, 插入位置的选择同上
4. 如果拆分后的链表长度超过8个元素, 则转成红黑树


{% note success %}
### Unsafe类操作
{% endnote %}
数组table虽然加了volatile属性, 但是此时volatile语义只对数组的引入生效, 而不是数组元素, 此时如果有其他线程对元素进行读写操作, 不一定可以获得最新值
```java
// 采用Unsafe基于反射以volatile读的形式读取元素结点, 保证了其他线程修改当前线程的元素可见性
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}

// 采用Unsafe直接对内存进行比较并赋值
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i, Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}

static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
    U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
}
```
