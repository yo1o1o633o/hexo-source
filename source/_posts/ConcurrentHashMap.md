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
### 初始化Table
initTable()
{% endnote %}

在添加元素时， 如果数组Table为空则进行初始化

- 初始化过程中， 会判断sizeCtl, 如果小于0则表示已经有线程在初始化, 当前线程让出CPU
- sizeCtl在初始化前保存的时候初始化容量, 初始化完成后保存扩容阈值
- sc = n - (n >>> 2);计算扩容阈值, n>>>2就是n除以4, n - n / 4就是n乘以4分之一n. 即sc = 0.75*n. 用右移计算避免除法的性能损耗

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
### 添加元素
putVal(K key, V value, boolean onlyIfAbsent)
{% endnote %}

- 添加元素方法内部实现了一个死循环, 一个自旋操作, 循环内部会有3种逻辑, 添加元素成功后会退出循环
- 首先判断Table是否为空, 如果为空则执行初始化方法initTable
    ```java
    if (tab == null || (n = tab.length) == 0)
        tab = initTable();
    ```
- 如果已经初始化, 则添加操作会有3种情况发生
    1. 要添加的key对应的Table数组索引位置为空, 即元素不存在
        以CAS形式添加元素, 这里可能会出现多线程并发添加的情况, 如果CAS赋值失败, 则表示其他线程先一步添加元素了, 则当前线程进入下一次循环, 再次判断元素是否存在, 如果存在则会进入到元素已存在的处理逻辑内, 即第3种情况
        ```java
        // 计算要添加的key对于数组的索引下标
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
           // 以原子性操作对这个索引赋值, 将要添加的结点放到索引位置上
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                // CAS形式添加到数组成功, 直接返回
                break;
        }
        ```
    2. Table数组正在扩容, 则触发协助扩容逻辑, 扩容的标志是元素结点的hash值为-1
        ```java
        tab = helpTransfer(tab, f);
        ```
    3. 根据hash值获取到的元素不为空, 则证明出现hash冲突, 进行链表或者红黑树处理, 分为以下3步
        - 对当前数组元素进行加锁synchronized, 确保只有一个线程能操作这个元素对应的桶（链表或红黑树）
        - 通过hash值来判断这个元素结点是链表或者红黑树, 如果hash值为正数, 则当前元素是链表头结点, 否则是红黑树的父结点
        - 开始循环遍历, 如果发现key值相同, 则表示要添加的元素key已经存在, 直接覆盖value, 否则将元素结点添加到链表尾部或者红黑树里

        ```java
        // 加锁, f是数组Table的元素, 对其加锁保证多个线程只能一个线程操作当前元素对应的桶
        synchronized (f) {
            if (tabAt(tab, i) == f) {
                // 链表的hash值是正数或0
                if (fh >= 0) {
                    // 循环遍历自增数值
                    binCount = 1;
                    // 循环遍历链表各个结点
                    for (Node<K,V> e = f;; ++binCount) {
                        K ek;
                        // 当key相同时, 表示key已经存在了, 进行值覆盖操作
                        if (e.hash == hash && ((ek = e.key) == key || (ek != null && key.equals(ek)))) {
                            oldVal = e.val;
                            // 判断下入参是否元素不存在时才放入, 即是否覆盖原key的值
                            if (!onlyIfAbsent)
                                e.val = value;
                            break;
                        }
                        Node<K,V> pred = e;
                        // 把新的元素结点添加到链表尾部
                        if ((e = e.next) == null) {
                            pred.next = new Node<K,V>(hash, key, value, null);
                            break;
                        }
                    }
                }
                // 元素结点是红黑树
                else if (f instanceof TreeBin) {
                    Node<K,V> p;
                    binCount = 2;
                    if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
                        oldVal = p.val;
                        if (!onlyIfAbsent)
                            p.val = value;
                    }
                }
            }
        }
        ```
    4. 当添加元素执行完成后, 判断当前链表长度是否大于8, 如果是8则调用转换红黑树方法
        ```java
        if (binCount != 0) {
            // 链表长度大于等于8则进入转换方法
            if (binCount >= TREEIFY_THRESHOLD)
                treeifyBin(tab, i);
            if (oldVal != null)
                return oldVal;
            break;
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
### 添加计数和检查扩容
addCount(long x, int check)
{% endnote %}

- 添加元素完成后, 添加元素数量计数操作

    ```java
    addCount(1L, binCount);
    ```

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
