---
title: 源码解析-SkipList跳表
date: 2022-01-02 00:57:01
categories:
- Redis
tags:
- Redis
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

文章是基于Redis6.0版本的


{% note success %}
### 数据结构
{% endnote %}

一个跳表包含一个头节点, 一个尾节点, 集合元素个数和当前表的最大层数
```C
typedef struct zskiplist {
    // 头节点指针和尾节点指针
    struct zskiplistNode *header, *tail;
    // 当前集合的元素个数
    unsigned long length;
    // 当前跳跃表的最大层数
    int level;
} zskiplist;
```

跳表单个元素节点
```C
typedef struct zskiplistNode {
    // 保存的value值
    sds ele;
    // 用于排序的得分
    double score;
    // 后退指针, 指向前一个节点
    struct zskiplistNode *backward;
    // 节点数组, 即跳表的层
    struct zskiplistLevel {
        // 前进指针, 指向下一个节点
        struct zskiplistNode *forward;
        // 跨度, 当前节点到下一个节点的跨度
        unsigned long span;
    } level[];
} zskiplistNode;
```

{% note success %}
### 跳表结构图
{% endnote %}
下面是一个跳表的结构图, header头节点包含32个层, 每个层forward都指向下一个节点, 每个层的span保存了到达下一个节点的跨度, length值为7表示跳表中有7个元素, level值为3表示当前跳表最大的层是3
{% asset_img SkipList2.jpg %}

{% note success %}
### 创建跳表
{% endnote %}
```C
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;
	// 分配内存
    zsl = zmalloc(sizeof(*zsl));
    // 初始最大层数为1层
    zsl->level = 1;
    // 初始元素个数为0个
    zsl->length = 0;
    // 创建一个拥有32层的头节点, 节点都是空的
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;
    zsl->tail = NULL;
    return zsl;
}
```

{% note success %}
### 生成随机层
{% endnote %}
初始层数为1, 采用随机方式增加层数, 每次循环有1/4几率增加一层. 最大不能超过32层
ZSKIPLIST_P 的值为 0.25
1. 初始level = 1
2. 生成随机数, 随机数 & 65535 < 65535 * 0.25 时 level+1
   随机数 & 65535 会得到一个 0 ~ 65535 之间的一个整数, 然后和 65535 * 0.25比较
```C
int zslRandomLevel(void) {
    // 初始为1层
    int level = 1;
    // 1/4几率增加层数
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    // 最多32层
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```

{% note success %}
### 插入元素
{% endnote %}

```C
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    // 声明一个容量为最大层数的节点数组, 用来保存每一层要更新的Node节点
    // 声明一个节点, 用来临时保存节点对象用
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    // 声明一个容量为最大层的整数数组, 用来保存每一层要更新的span值
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;
    
    // 校验一下要插入的分数是否合法
    serverAssert(!isnan(score));
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        // 为了到达插入的位置需要跳过的跨度
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        // 遍历当前层的链表, 找到新节点要插入的位置, 先对比分数, 分数相同时按字典序比较成员
        while (x->level[i].forward && (x->level[i].forward->score < score || (x->level[i].forward->score == score && sdscmp(x->level[i].forward->ele,ele) < 0))) {
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        // 将每一层找到的要插入的位置保存到数组中
        update[i] = x;
    }

    // 生成一个随机层数
    level = zslRandomLevel();
    // 如果生成的层数比当前跳表最大层数大, 则需要对超出的层数赋值默认值
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }
    // 根据入参生成新的节点
    x = zslCreateNode(level,score,ele);
    for (i = 0; i < level; i++) {
        // 根据每层的插入位置, 将新节点插入到链表当中
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        // 更新span的值, 即每个节点到达下一个节点的间隔
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    // 未发生更新的层, 增加跨度
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    // 如果要插入的是第一层的头节点后边, 则表示当前新节点是链表的第一个元素, 后退指针就是NULL
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    // 如果插入的节点有后续节点
    if (x->level[0].forward)
        // 则对后续节点的后退指针赋值为新创建的节点
        x->level[0].forward->backward = x;
    else
        // 新节点没有后续节点, 表示新节点就是跳表的最后一个节点, 那么把他设置为有序集合的尾节点指针
        zsl->tail = x;
    // 更新元素个数
    zsl->length++;
    return x;
}
```

{% note success %}
### 插入示例
{% endnote %}
基于上面的跳表结构图, 例如要插入一个元素score得分为14, ele值为RabbitMQ会执行一下流程
遍历跳表, 寻找所有需要处理的节点并保存到update数组中
```C
x = zsl->header;
// 此时i取值为2, 1, 0. 即表示要处理跳表的2层1层和0层
for (i = zsl->level-1; i >= 0; i--) {
    // 为了到达插入的位置需要跳过的跨度
    rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
    // 遍历当前层的链表, 找到新节点要插入的位置, 先对比分数, 分数相同时按字典序比较成员
    while (x->level[i].forward && (x->level[i].forward->score < score || (x->level[i].forward->score == score && sdscmp(x->level[i].forward->ele,ele) < 0))) {
        rank[i] += x->level[i].span;
        x = x->level[i].forward;
    }
    // 将每一层找到的要插入的位置保存到数组中
    update[i] = x;
}
```
当前跳表最大层数为3时, 此部分代码执行流程如下如所示：
{% asset_img SkipList-insert.jpg %}

最终将需要更新的节点放入到update数组中, 将要更新的跨度值span放入到rank数组中, 为后续真正插入操作时使用

{% note success %}
### 删除元素
{% endnote %}
删除跳表元素, 也和插入操作一样, 将要更新的节点保存到一个update数组中, 不过要注意的是跳表中会有分数相同的节点, 所以要同时判断分数和元素对象同时正确的才是要删除的节点
```C
/**
* 删除节点
* zskiplist *zsl        跳表对象
* double score          要删除的分数
* sds ele               要删除的元素对象
* zskiplistNode **node  引用节点
* 只有分数和元素对象都符合的节点会被删除
* node引用节点如果传入NULL, 则会释放删除节点的资源, 否则会将删除的节点引用赋值给这个node节点, 并供调用方使用
*/
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    x = zsl->header;
    // 和插入逻辑一样, 从当前最大层开始遍历跳表, 找到所有要更新的节点并保存到update数组中
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward && (x->level[i].forward->score < score || (x->level[i].forward->score == score && sdscmp(x->level[i].forward->ele,ele) < 0))) {
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    // 当前x是跳表的第0层的要删除节点
    x = x->level[0].forward;
    // 跳表中会有多个分数相同的节点, 此处要找到分数和元素对象都正确的节点
    if (x && score == x->score && sdscmp(x->ele,ele) == 0) {
        // 删除跳表节点
        zslDeleteNode(zsl, x, update);
        // 只有当传入的node为NULL时, 才会释放这个节点的资源. 否则只会取消链表中的链接, 同时将该节点赋值给node, 在调用侧可以引用使用
        if (!node)
            zslFreeNode(x);
        else
            *node = x;
        return 1;
    }
    // 未发现分数和元素对象都正确的节点
    return 0;
}

/**
* 删除节点, 此方法为多个内部方法调用
* zskiplist *zsl            跳表对象
* zskiplistNode *x          要删除的节点
* zskiplistNode **update    保存了每层要删除节点的前一个节点
* 链表的删除原理1指向2,2指向3(1->2->3). 直接改为1指向3(1->3). 就是删除了2这个节点
* 跳表同理, 逐层的改变节点间的指向, 让每一层都跳过要删除的节点, 就是删除了这个节点. 在删除的同时更新跨度值  
*/
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    int i;
    for (i = 0; i < zsl->level; i++) {
        // 碰到了要删除的节点
        if (update[i]->level[i].forward == x) {
            // 更改跨度, 将删除节点的跨度加到前一个节点的跨度值上
            update[i]->level[i].span += x->level[i].span - 1;
            // 更改指针, 将指向要删除的节点指针改为指向删除节点的下一个节点, 和链表移除一个节点原理一样
            update[i]->level[i].forward = x->level[i].forward;
        } else {
            // 当前层没有要删除的节点, 因为跳表移除了一个节点, 则这些层的跨度要减一
            update[i]->level[i].span -= 1;
        }
    }
    // x为要删除的节点
    // 如果x有下一个节点的指针, 则x的下一个节点的后退指针要更新为x的前一个节点
    // 如果没有, 则表示x是跳表最后一个节点, 则要更新有序集合的尾节点
    if (x->level[0].forward) {
        x->level[0].forward->backward = x->backward;
    } else {
        zsl->tail = x->backward;
    }
    // 检测, 当有序集合的层数超过1层时, 如果有哪个层已经空了, 则要减少最大层数
    while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
        zsl->level--;
    // 更新元素个数
    zsl->length--;
}
```

{% note success %}
### 更新得分
{% endnote %}
```C
/**
* 根据分数找到节点并更新为新分数
* zskiplist *zsl    跳表对象
* double curscore   被更新的分数
* sds ele           被更新的元素对象
* double newscor    新分数
*/
zskiplistNode *zslUpdateScore(zskiplist *zsl, double curscore, sds ele, double newscore) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    x = zsl->header;
    // 和插入逻辑一样, 从当前最大层开始遍历跳表, 找到所有要更新的节点并保存到update数组中
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward && (x->level[i].forward->score < curscore || (x->level[i].forward->score == curscore && sdscmp(x->level[i].forward->ele,ele) < 0))) {
            x = x->level[i].forward;
        }
        update[i] = x;
    }

    // x 就是要更新的节点
    x = x->level[0].forward;
    // 校验分数和元素对象都是要更新的节点
    serverAssert(x && curscore == x->score && sdscmp(x->ele,ele) == 0);

    // 后退指针为NULL, 或者后一个节点分数比新分数小
    // 前进指针为NULL, 或者前一个节点分数比新分数大
    // 同时满足上述条件, 表示新分数还是在跳表的原位置, 只需要更新分数即可, 不用调整跳表的结构
    if ((x->backward == NULL || x->backward->score < newscore) && (x->level[0].forward == NULL || x->level[0].forward->score > newscore)) {
        x->score = newscore;
        return x;
    }

    // 无法在原来的节点上直接更新, 那么要删除原来的节点, 再以新分数插入新节点
    zslDeleteNode(zsl, x, update);
    zskiplistNode *newnode = zslInsert(zsl,newscore,x->ele);
    // 因为插入时创建了一个新节点, 所以要把原来的节点释放掉
    x->ele = NULL;
    zslFreeNode(x);
    return newnode;
}
```

{% note success %}
### 释放资源
{% endnote %}
```C
/*
* 释放跳表节点资源
*/
void zslFreeNode(zskiplistNode *node) {
    sdsfree(node->ele);
    zfree(node);
}

/*
* 释放整个跳表资源
*/
void zslFree(zskiplist *zsl) {
    // 保留第一层的第一个节点
    zskiplistNode *node = zsl->header->level[0].forward, *next;
    // 释放头节点
    zfree(zsl->header);
    // 从第一个元素节点向后遍历, 逐个节点释放资源
    while(node) {
        next = node->level[0].forward;
        zslFreeNode(node);
        node = next;
    }
    // 释放整个跳表资源
    zfree(zsl);
}
```