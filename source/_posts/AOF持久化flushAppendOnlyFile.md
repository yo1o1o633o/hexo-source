---
title: AOF持久化-flushAppendOnlyFile()
date: 2022-01-11 23:36:24
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

该方法在AOF持久化时执行, 将数据从aof_buf缓冲区中写入到AOF文件中, 同时根据不同的同步刷盘策略进行同步到磁盘文件中

同步策略：
#define AOF_FSYNC_NO 0          // 由操作系统决定
#define AOF_FSYNC_ALWAYS 1      // 每次事件循环后刷盘
#define AOF_FSYNC_EVERYSEC 2    // 间隔1秒以上执行刷盘

当fsync策略为AOF_FSYNC_EVERYSEC时. 如果后台线程中有fsync在进行, 那么要延迟刷新
因为在Linux系统上write操作会被fsync阻塞.所以这里即使这里不延迟处理也会被系统的fsync阻塞住

{% note success %}
### AOF文件写入
{% endnote %}
检查是否需要在AOF缓冲区为空的情况下进行刷盘
```C
// 当AOF缓冲区没有数据时
if (sdslen(server.aof_buf) == 0) {
    // 在5.0以上版本新增的处理, 检查是否需要在AOF缓冲区为空的情况下进行刷盘
    // 当刷盘策略为AOF_FSYNC_EVERYSEC模式, 缓冲区有数据未刷盘, 当前时间已经超过1秒, 同时当前没有fsync在执行
    // 在AOF_FSYNC_EVERYSEC模式下, 只有在aof缓冲区不为空时才会调用fsync, 所以如果用户在一秒钟内调用fsync之前停止写命令, 页面缓存中的数据将无法及时刷盘
    if (server.aof_fsync == AOF_FSYNC_EVERYSEC && server.aof_fsync_offset != server.aof_current_size && server.unixtime > server.aof_last_fsync && !(sync_in_progress = aofFsyncInProgress())) {
        goto try_fsync;
    } else {
        return;
    }
}
```
AOF_FSYNC_EVERYSEC模式下, 需要检测是否有AOF同步事件在执行fsync操作, 如果有则要延迟处理此次数据写入, 除非延迟时间超过了2秒
```C
// 当刷盘策略为AOF_FSYNC_EVERYSEC模式时, 调用函数判断BIO_AOF_FSYNC类型事件的待处理的数量是否等于0. 如果大于0则表示有AOF同步事件在执行
if (server.aof_fsync == AOF_FSYNC_EVERYSEC)
    sync_in_progress = aofFsyncInProgress();
// 当刷盘策略为AOF_FSYNC_EVERYSEC模式, 且指定入参为0
if (server.aof_fsync == AOF_FSYNC_EVERYSEC && !force) {
    // 表示有AOF同步事件在执行fsync操作, 那么这里要判断是否进行延迟处理
    if (sync_in_progress) {
        // 之前没有延迟处理, 此次是第一次碰到要延迟处理. 那么记录我们要延迟操作, 记录下当前时间, 此次操作直接返回
        if (server.aof_flush_postponed_start == 0) {
            server.aof_flush_postponed_start = server.unixtime;
            return;
        // 此前已经触发了延迟处理, 距离上次延迟处理的时间间隔还没超过2秒, 此次不处理, 直接返回继续等待
        } else if (server.unixtime - server.aof_flush_postponed_start < 2) {
            return;
        }
        // 此前已经触发了延迟处理, 且距离上次延迟处理的时间间隔已经超过2秒, 不能忍受, 直接进行写入操作
        server.aof_delayed_fsync++;
        serverLog(LL_NOTICE,"Asynchronous AOF fsync is taking too long (disk is busy?). Writing the AOF buffer without waiting for fsync to complete, this may slow down Redis.");
    }
}
```
真正执行写入文件操作, 将aof_buf中的数据写入到AOF文件中, 同时返回写入成功的字节数. -1表示写入错误
同时记录一些日志信息, 用于对Redis的监控使用
```C
if (server.aof_flush_sleep && sdslen(server.aof_buf)) {
    usleep(server.aof_flush_sleep);
}
// 开始监控一个事件, 设置当前时间, 和latencyEndMonitor方法组成一个监控, 监控AOF的写入操作时间
latencyStartMonitor(latency);
// 真正执行将AOF缓冲区数据写入AOF文件(只是应用层面的写入, 系统级磁盘同步在后面)
nwritten = aofWrite(server.aof_fd,server.aof_buf,sdslen(server.aof_buf));
latencyEndMonitor(latency);
// 记录一些时间, 用于监控
if (sync_in_progress) {
    latencyAddSampleIfNeeded("aof-write-pending-fsync",latency);
} else if (hasActiveChildProcess()) {
    latencyAddSampleIfNeeded("aof-write-active-child",latency);
} else {
    latencyAddSampleIfNeeded("aof-write-alone",latency);
}
latencyAddSampleIfNeeded("aof-write",latency);
// 我们执行了写入, 因此将推迟的刷新标记重置为零
server.aof_flush_postponed_start = 0;
```
针对部分失败进行补救处理, 全部成功的状态变更
1. 当部分写入失败时, 需要尝试将写入的部分数据进行移除. 如果移除失败那么裁剪aof_buf中的数据. 要确保写入到AOF中的文件是正确的
2. AOF_FSYNC_ALWAYS模式下如果出现部分失败, 则无法进行错误处理. 直接退出程序

```C
// 当写入AOF文件的长度和AOF缓冲区长度不一致, 表示部分写入失败或者全部失败
if (nwritten != (ssize_t)sdslen(server.aof_buf)) {
    static time_t last_write_error_log = 0;
    int can_log = 0;

    // 限制日志速率, 限制为每AOF_WRITE_LOG_ERROR_RATE秒写一次刷新错误日志
    if ((server.unixtime - last_write_error_log) > AOF_WRITE_LOG_ERROR_RATE) {
        can_log = 1;
        last_write_error_log = server.unixtime;
    }

    // 此次写入完全失败, 一点也没写进去
    if (nwritten == -1) {
        // 判断可以写日志
        if (can_log) {
            serverLog(LL_WARNING,"Error writing to the AOF file: %s", strerror(errno));
            server.aof_last_write_errno = errno;
        }
    } else {
        if (can_log) {
            serverLog(LL_WARNING,"Short write while writing to ""the AOF file: (nwritten=%lld, ""expected=%lld)", (long long)nwritten, (long long)sdslen(server.aof_buf));
        }
                
        // 尝试回退刚刚写入的不完整数据, 如果操作失败则记录日志
        if (ftruncate(server.aof_fd, server.aof_current_size) == -1) {
            if (can_log) {
                serverLog(LL_WARNING, "Could not remove short write ""from the append-only file.  Redis may refuse ""to load the AOF the next time it starts.  ""ftruncate: %s", strerror(errno));
            }
        } else {
            // 尝试从AOF文件中移除部分成功的数据, 成功后将写入状态置为-1
            nwritten = -1;
        }
        server.aof_last_write_errno = ENOSPC;
    }

    // 如果是AOF_FSYNC_ALWAYS模式
    if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
        // 当fsync策略为ALWAYS时, 因为给客户端的恢复已经在输出缓冲区中, 同时和请求方约定, 确认写入的数据已经同步到磁盘上, 无法进行恢复, 退出
        serverLog(LL_WARNING,"Can't recover from AOF write error when the AOF fsync policy is 'always'. Exiting...");
        exit(1);
    } else {
        // 从失败的写入中恢复，将数据留在缓冲区中。 但是，只要错误条件未清除，就设置错误以停止接受写入。
        server.aof_last_write_status = C_ERR;

        // 如果是部分写入成功, 同时ftruncate函数也没有成功将这部分数据从AOF文件中清除. 那么裁剪AOF缓冲区
        if (nwritten > 0) {
            server.aof_current_size += nwritten;
            sdsrange(server.aof_buf,nwritten,-1);
        }
        // 此次AOF写入失败了, 直接返回, 然后在下次调用此方法写入AOF数据时再次进行尝试
        return;
    }
} else {
    // 此前AOF处于错误状态, 但是此次写入成功了. 则恢复为OK状态.表示AOF写入错误已经恢复了,可以继续写入数据
    if (server.aof_last_write_status == C_ERR) {
        serverLog(LL_WARNING, "AOF write error looks solved, Redis can write again.");
        server.aof_last_write_status = C_OK;
    }
}
server.aof_current_size += nwritten;
```
写入成功要清除aof_buf数据. 根据数据的多少决定是清空原空间还是重新申请空间
```C
// 判断下当前的AOF缓冲区大小, 如果小于4000, 则清除缓冲区复用. 
// 如果大于4000, 那么释放缓冲区重新申请新的缓冲区
if ((sdslen(server.aof_buf)+sdsavail(server.aof_buf)) < 4000) {
    sdsclear(server.aof_buf);
} else {
    sdsfree(server.aof_buf);
    server.aof_buf = sdsempty();
}
```

{% note success %}
### write文件写入
{% endnote %}
```C
/**
* 调用系统write函数写入数据
* write有写入限制, 不能一次将所有数据写入, 需要程序控制
* 正常情况下传入的len和这个函数的返回值应该一致, 不一致则可能发生写入失败仅有部分写入成功
* int fd             AOF文件描述符
* const char *buf    AOF缓冲区, 一个字符串
* size_t len         AOF缓冲区长度, 字符串长度
* 返回成功写入的长度
*/
ssize_t aofWrite(int fd, const char *buf, size_t len) {
    ssize_t nwritten = 0, totwritten = 0;

    while(len) {
        nwritten = write(fd, buf, len);
        // 返回小于0, 表示写入失败了
        if (nwritten < 0) {
            // 因为系统中断导致写入失败, 则重试
            if (errno == EINTR) 
                continue;
            // 说明I/O失败了返回已写入的字节数
            return totwritten ? totwritten : -1;
        }
        // 重新计算长度, write有写入限制不能一次全部写入, 需要控制偏移量, 进行循环写入
        len -= nwritten;
        buf += nwritten;
        totwritten += nwritten;
    }
    return totwritten;
}
```

{% note success %}
### fsync同步磁盘
{% endnote %}
1. 如果是AOF_FSYNC_ALWAYS, 则直接执行fsync进行磁盘同步
2. 如果是AOF_FSYNC_EVERYSEC, 则判断时间是否已经过了1秒. 然后放入队列中交由子线程消费队列执行刷盘任务

```C
try_fsync:
    // 如果正在进行AOF重写操作, 并且有子进程在后台执行 I/O, 则不要fsync
    if (server.aof_no_fsync_on_rewrite && hasActiveChildProcess())
        return;

    // 如果刷盘策略为AOF_FSYNC_ALWAYS, 则进行fsync
    if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
        latencyStartMonitor(latency);
        // redis_fsync被定义为Linux的fdatasync函数,避免刷新元数据.调用系统函数fsync进行刷盘
        redis_fsync(server.aof_fd);
        latencyEndMonitor(latency);
        latencyAddSampleIfNeeded("aof-fsync-always",latency);
        server.aof_fsync_offset = server.aof_current_size;
        // 更新最后刷盘时间
        server.aof_last_fsync = server.unixtime;
        // 如果刷盘策略为AOF_FSYNC_EVERYSEC, 则判断时间是否满足, 满足则进行fsync
    } else if ((server.aof_fsync == AOF_FSYNC_EVERYSEC && server.unixtime > server.aof_last_fsync)) {
        if (!sync_in_progress) {
            // redis提供了一个链表形式的队列, 主线程向队列中添加任务, 子线程阻塞等待读取任务, 并执行刷盘任务
            aof_background_fsync(server.aof_fd);
            server.aof_fsync_offset = server.aof_current_size;
        }
        server.aof_last_fsync = server.unixtime;
    }
}
```