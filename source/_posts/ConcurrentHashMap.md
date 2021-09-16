---
title: ConcurrentHashMap源码解析
categories:
    - JAVA
tags:
    - JAVA
    - 源码
---

{% note success %}
### 特点
{% endnote %}
- 利用CAS自旋锁+Synchronized保证并发安全的Map
- 使用数组+链表+红黑树实现
- key和value不能为null

{% note success %}
### 注意
{% endnote %}
1. 初始化容量计算
    初始容量采用公式 n + n / 2 + 1, 例如创建时指定32, 则根据公式  32 + 32 / 2 + 1 = 49.  然后会取49之后的2的幂数是64
2. 多线程协助扩容
    在扩容过程中会根据CPU按段对数组进行迁移处理, 对处理过的元素进行标记, 当有另一个线程添加元素发现是标记元素时, 则协助扩容, 也领取一段元素进行扩容处理
    处理完成后再次循环领取任务处理, 处理是从数组末尾向前处理的, 最小的段大小是16个元素
3. 元素数量计数
    元素计数采用CAS赋值和赋值失败对数组元素赋值的方式进行, CAS对baseCount累加失败, 则循环对数组中的元素进行累加操作, 数组累加操作成功一次就跳出, 失败则重新计算索引再次尝试累加
    总元素个数是baseCount+数组元素累加的值
4. 大量的使用CAS自旋操作, 以避免使用锁

{% note success %}
### 关键参数sizeCtl
{% endnote %}

1. sizeCtl为0, 表示数组未初始化, 且初始容量为16
2. sizeCtl为正数, 如果数组未初始化，则记录的是数组初始容量，如果已经初始化，则记录数组扩容阈值（初始容量*0.75）
3. sizeCtl为-1， 表示数组正在初始化
4. sizeCtl小于0且不是-1，表示数组正在扩容,-(1-n)表示n个线程正在对数组扩容

{% note success %}
### 获取元素
tabAt(Node<K,V>[] tab, int i)
{% endnote %}
传入数组Table和数组下标, 获取下标对应的元素
table是用volatile修饰的, 但是无法保证线程每次都能拿到table最新元素
使用Unsafe.getObjectVolatie()获取索引元素, 这个方法可以直接获取内存的数据, 保证每次拿到数据都是最新的
```java
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
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
### 添加计数
fullAddCount(long x, boolean wasUncontended)
{% endnote %}

添加计数死循环自旋处理, 根据不同情况进入3中条件中
1. CounterCell[]不为空, 数组长度大于0时, 对数组内元素操作
2. 如果没有其他线程在初始化数组, 则进行初始化操作, 数组大小为2, 执行完成后标记初始化完成
3. 尝试对baseCount进行CAS累加操作, 操作成功就跳出循环, 否则进行下次循环进入对数组操作逻辑

```java
private final void fullAddCount(long x, boolean wasUncontended) {
    int h;
    // 线程的探针哈希值未初始化, 则进行初始化操作, 然后获取线程的探针哈希值->h
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();      // force initialization
        h = ThreadLocalRandom.getProbe();
        wasUncontended = true;
    }
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        CounterCell[] as; CounterCell a; int n; long v;
        // 数组不为空, 即已经执行完初始化了
        if ((as = counterCells) != null && (n = as.length) > 0) {
            // hash对应数组元素为空, 进行创建元素
            if ((a = as[(n - 1) & h]) == null) {
                if (cellsBusy == 0) {            // Try to attach new Cell
                    // 调用有参构造器进行创建对象
                    CounterCell r = new CounterCell(x); // Optimistic create
                    if (cellsBusy == 0 && U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                        boolean created = false;
                        try {               // Recheck under lock
                            CounterCell[] rs; int m, j;
                            if ((rs = counterCells) != null && (m = rs.length) > 0 && rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        // 经过一系列判断, 创建成功跳出循环, 否则下次循环
                        if (created)
                            break;
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
            else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                break;
            // counterCells数组大小超出CPU核数, 线程的并发数不会超出CPU核数
            else if (counterCells != as || n >= NCPU)
                collide = false;            // At max size or stale
            else if (!collide)
                collide = true;
            else if (cellsBusy == 0 && U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                try {
                    if (counterCells == as) {// Expand table unless stale
                        CounterCell[] rs = new CounterCell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        counterCells = rs;
                    }
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            h = ThreadLocalRandom.advanceProbe(h);
        }
        // 数组为空, 则进行创建数组操作
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
### 扩容方法
transfer(Node<K,V>[] tab, Node<K,V>[] nextTab)
{% endnote %}

1. 计算stride, 一个线程处理hash表多少个数组元素, 每个元素对应一个桶, 就是表示一个线程处理多少个桶数据，根据当前CPU核数计算要处理的段大小, 最小16个元素
2. nextTable数组是扩容操作要迁移到的目标数组, 如果nextTable数组为空， 则进行初始化操作, 数组大小为当前数组的2倍
- 扩容有两种情况, 当前线程扩容和协助扩容

扩容的原理, 多线程可能同时进行扩容, 那么将整个数组根据CPU核数计算出平均的段, 每个线程处理一个段, 当处理完一个元素结点后, 在这个位置放置一个ForwardingNode元素结点占位表示正在处理, 即MOVED

新建一个2倍原数组大小的nextTable数组,用来保存扩容后的新数组

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // 如果CPU核数大于1, 则当前 stride = 数组大小/8/CPU核数, 如果是单核则为数组大小
    // 计算结果如果小于16, 则赋值16, 即最小每个线程处理16个元素
    // 此计算保证每个线程处理的段大小相同, 避免迁移任务出现不均匀
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    // 要迁移的数组为null, 则创建2倍大的新数组
    if (nextTab == null) {            // initiating
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
    // 创建ForwardingNode结点, 用来标记正在迁移的元素, hash值为-1
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        // 为当前线程分配要处理的段位移值
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            else if (U.compareAndSwapInt (this, TRANSFERINDEX, nextIndex, nextBound = (nextIndex > stride ? nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            // 扩容完成, 将新的table赋值给原table
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            // 判断是否扩容完成
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
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
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
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
初始容量一定是2的幂次, 根据构造函数可知调用此方法传入的一定不是个2的幂次, 因为有+1操作
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
空参的构造方法, 什么都不做, 会在对Map操作时使用默认容量(16)初始化
```java
public ConcurrentHashMap() {
}
```
传入初始容量构造方法, 根据传入的初始容量计算实际的初始容量, 并向sizeCtl参数赋值, 因为数组还未初始化, sizeCtl表示初始容量
initialCapacity + (initialCapacity >>> 1) + 1 = 16 + 16 / 2 + 1 = 25
调用tableSizeFor方法获取最近的2的幂次
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
```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    // 计算Key的Hash
    int h = spread(key.hashCode());
    // 数组不为null
    // 数组长度不为0
    // 根据Hash值计算的索引位置元素不为null
    if ((tab = table) != null && (n = tab.length) > 0 && (e = tabAt(tab, (n - 1) & h)) != null) {
        // 如果获取的元素Hash和Key的Hash相同
        if ((eh = e.hash) == h) {
            // 如果Key相等, 则直接返回Value
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
调用获取元素方法进行判断
```java
public boolean containsKey(Object key) {
    return get(key) != null;
}
```

{% note success %}
### VALUE是否存在
{% endnote %}
找到一个VALUE, 最坏可能遍历整个Map, 性能很慢
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
3. 如果当前元素hash为-1, 则表示当前元素正在进行扩容移动, 调用协助扩容方法
4. 以上都不成立, 表明当前数组元素有值, 对这个元素加synchronized后进行遍历处理
    - 链表: 如果key在链表中存在, 则进行覆盖, 否则添加到链表尾部
    - 红黑树: 使用红黑树对象方法进行处理

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
### 清空MAP
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
竞争成功的线程将sizeCtl设置成-1, 表明数组正在进行初始化
初始化完成后将sizeCtl设置为扩容阈值
扩容阈值计算sc = n - (n >>> 2)
例如: n = 16, 则 16 - 16 / 2 / 2 = 12, 即16 * 0.75 = 12, 扩容阈值为12, 当元素数量达到12个时会触发扩容操作. 右移操作替代除法, 提高运算效率

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
### 添加计数和检查扩容
addCount(long x, int check)
{% endnote %}

- 添加计数
    为了在并发情况下, 多个线程同时进行添加元素操作, 对总元素个数不能使用普通累加操作, 则采用数值+数组的形式进行处理
    baseCount：用来进行累加操作
    CounterCell[]：数组用来当baseCount累加操作失败时保存数据
    流程：首先CAS对baseCount进行累加操作, 如果操作成功则继续向下, 否则使用CounterCell数组中的一个元素来保存累加数值, 当需要获取总数时, 使用BaseCount和数组的每个元素的总计
    计数操作完成后判断是否需要扩容, 并进行扩容处理

```java
// 添加计数
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
        // 统计数组元素个数
        s = sumCount();
    }
    // 当发生添加操作时进入此逻辑判断是否要对数组扩容
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        // 此时sizeCtl保存的是扩容阈值, s为当前数组元素个数, 此处判断是否需要扩容
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null && (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            // 小于0则表示当前有线程在扩容, 进行协助扩容操作
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 || sc == rs + MAX_RESIZERS || (nt = nextTable) == null || transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            // 大于等于0表示开始扩容, 同时将sizeCtl的值设置成负数
            else if (U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2))
                // 扩容方法
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```