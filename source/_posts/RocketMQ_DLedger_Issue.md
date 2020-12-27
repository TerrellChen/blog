---
title: RocketMQ DLedger毛刺排查
date: 2020-06-17 13:55:00
tags:
---
<!-- toc -->

## 1 问题

RocketMQ上线DLedger后频繁报出异常，以TIMEOUT_CLEAN_QUEUE为主

```
CODE: 2 DESC: [TIMEOUT_CLEAN_QUEUE]broker busy, start flow control for a while, period in queue: [waitTimeMillsInSendQueue]ms, size of queue: [current_queue_size]
```

根据日志内容可以确认超时逻辑是在MQ服务端，针对投递消息的请求队列中的请求，如果超过一定时间(waitTimeMillsInSendQueue)没有处理完成，则会直接返回异常。

集群机器均为物理机、SSD，问题是请求是如何在服务端超时的？

## 2 排查

### 2.1 投递请求处理逻辑

投递请求是线程池在处理，其中加锁的逻辑仅有「将msg序列化，并写入本机buffer；产生一个Future对象用于等待获取法定多数机器ACK」

加锁逻辑外，需要等待Future对象有返回或3s超时后，才继续执行。这里有必要对加锁逻辑，及等待Future对象返回，两处逻辑的执行时间进行分析。

### 2.2 增加耗时统计的日志分析

日志及分析过程就不放这里了，简单说下日志分析结论，毛刺与上述锁内逻辑强相关：

- 锁内的操作的耗时出现毛刺时，处理send请求整体处理耗时一定出现对应或更高级别的毛刺
- 大于200ms的严重毛刺均为锁内的操作出现毛刺导致
- 小于200ms的轻度毛刺看起来与等待其他节点ACK的操作的耗时出现毛刺也有相关。数据太多，暂时不好分析。先解决大的毛刺。

### 2.3 锁内操作分析

在DLedgerCommitLog的逻辑中，锁内逻辑主要包括

- 对message进行序列化
- 调用DLedger中的方法进行append

序列化很明显是cpu操作，实际观察下来在发生毛刺时cpu没有波动，那么只能是DLedger中进行append的地方耗时有问题。

在DLedger的append逻辑中，操作主要包括：

- 一些内存对象的创建
- DataFile及IndexFile的append

这里的append都是对MappedByteBuffer的操作，将数据进行put。在此增加了Debug日志后发现，确为对buffer的put操作耗时较高

```
2020-06-11 16:07:21 INFO DLedgerStoreStatsService - [AppendEntryInLocal] TotalPut 11129, DistributeTime [<=0ms]:11036 [0~10ms]:90 [10~50ms]:2 [50~100ms]:0 [100~200ms]:0 [200~500ms]:1 [500ms~1s]:0 [1~2s]:0 [2~3s]:0 [3~4s]:0 [4~5s]:0 [5~10s]:0 [10s~]:0
```

### 2.4 MappedByteBuffer的操作

```
ByteBuffer byteBuffer = this.mappedByteBuffer.slice();
byteBuffer.position(currentPos);
byteBuffer.put(data, offset, length);
```

代码中对MappedByteBuffer的写入使用如上方法。ByteBuffer的类型为DirectByteBuffer，而put方法最终调用链路如下

```
java.nio.Bits#copyFromArray
->
sun.misc.Unsafe#copyMemory(java.lang.Object, long, java.lang.Object, long, long)
->
String.h#memmove(c库函数）
```

在memmove中，仅是对两个内存地址进行操作。当然这里是虚拟地址，映射到物理地址还需要linux内核进行操作，但是无法继续跟踪并分析。

这里尝试换一种思路继续跟进。

### 2.5 写一个Demo复现

在另一条思路的分析中：[Linux下一次IO毛刺排查](https://www.yuque.com/chentairan-klqff/wnx49g/fth1gr) 基本可以确定毛刺与Dirty writeback有关系。这里写了一个Demo部署到4.9.4内核的物理机上尝试复现这个问题。

然后问题真的复现了！Demo写了几类场景，在单线程write CommitLog、单线程flush文件的条件下，问题最容易复现，比较关键的日志如下

```
2020-06-11 15:55:19 [AppendTotal] TotalPut 15130, DistributeTime [<=0ms]:15119 [0~10ms]:11 [10~50ms]:0 [50~100ms]:0 [100~200ms]:0 [200~500ms]:0 [500ms~1s]:0 [1~2s]:0 [2~3s]:0 [3~4s]:0 [4~5s]:0 [5~10s]:0 [10s~]:0
2020-06-11 15:55:19 [FlushTotal] TotalPut 99, DistributeTime [<=0ms]:99 [0~10ms]:0 [10~50ms]:0 [50~100ms]:0 [100~200ms]:0 [200~500ms]:0 [500ms~1s]:0 [1~2s]:0 [2~3s]:0 [3~4s]:0 [4~5s]:0 [5~10s]:0 [10s~]:0
0 0 347
[CUTTING]
Append WARN
2020-06-11 15:55:20 [AppendTotal] TotalPut 9706, DistributeTime [<=0ms]:9678 [0~10ms]:26 [10~50ms]:0 [50~100ms]:0 [100~200ms]:0 [200~500ms]:1 [500ms~1s]:0 [1~2s]:0 [2~3s]:0 [3~4s]:0 [4~5s]:0 [5~10s]:0 [10s~]:0
2020-06-11 15:55:20 [FlushTotal] TotalPut 100, DistributeTime [<=0ms]:100 [0~10ms]:0 [10~50ms]:0 [50~100ms]:0 [100~200ms]:0 [200~500ms]:0 [500ms~1s]:0 [1~2s]:0 [2~3s]:0 [3~4s]:0 [4~5s]:0 [5~10s]:0 [10s~]:0
[CUTTING]
```

可以看到，15:55:20的时候，存在1次append请求处理时间在200-500ms之间。同时对这次耗时进行了拆分后的输出“0 0 347”，其中347是指的，对byteBuffer的put操作耗时347ms

同时，Demo单线程的场景非常利于对线程进行观测，这里采用bcc的offcputime工具对write线程的内核耗时进行分析，下面是这次毛刺前后5秒的一个统计结果

```
#/usr/share/bcc/tools/offcputime -K -m 1000 -t 10578
Tracing off-CPU time (us) of TID 10578 by kernel stack... Hit Ctrl-C to end.
^C
    finish_task_switch
    schedule
    futex_wait_queue_me
    futex_wait
    do_futex
    sys_futex
    do_syscall_64
    return_from_SYSCALL_64
    -                java (10578)
        51355
 
 
    finish_task_switch
    schedule
    wait_transaction_locked
    add_transaction_credits
    start_this_handle
    jbd2__journal_start
    __ext4_journal_start_sb
    ext4_dirty_inode
    __mark_inode_dirty
    generic_update_time
    file_update_time
    ext4_page_mkwrite
    do_page_mkwrite
    do_wp_page
    handle_mm_fault
    __do_page_fault
    do_page_fault
    page_fault
    -                java (10578)
        346004
  
PS1:这是内核调用栈
PS2:这里统计的累计时间超过1ms的
PS3：51355及346004是时间，单位微秒
```

futex是线程调度相关的，可以忽略，而另一个从page_fault开始的堆栈，耗时346ms，与这次毛刺的时间基本吻合。

### 2.6 在MQ机器上的验证

发现了可疑的堆栈，那么在线上mq机器尝试查找类似的线索，下面是对某一次毛刺的信息抓取

```
writeback情况
16:07:10  8:0      57580    periodic         0.000
16:07:15  8:0      56521    periodic         1.799
16:07:15  8:0      56521    periodic         0.000
16:07:20  8:0      52197    periodic         256.480
16:07:20  8:0      51945    periodic         17.172
16:07:20  8:0      51932    periodic         0.140
16:07:20  8:0      51932    periodic         0.000
 
 
mq刷盘情况
2020-06-11 16:07:21 INFO DLedgerStoreStatsService - [AppendEntryInLocal] TotalPut 11129, DistributeTime [<=0ms]:11036 [0~10ms]:90 [10~50ms]:2 [50~100ms]:0 [100~200ms]:0 [200~500ms]:1 [500ms~1s]:0 [1~2s]:0 [2~3s]:0 [3~4s]:0 [4~5s]:0 [5~10s]:0 [10s~]:0
 
 
mq线程较大耗时调用栈的抓取情况
    finish_task_switch
    schedule
    wait_transaction_locked
    add_transaction_credits
    start_this_handle
    jbd2__journal_start
    __ext4_journal_start_sb
    ext4_dirty_inode
    __mark_inode_dirty
    generic_update_time
    file_update_time
    ext4_page_mkwrite
    do_page_mkwrite
    do_fault
    handle_mm_fault
    __do_page_fault
    do_page_fault
    page_fault
    -                java (19108)
        250897
  
PS：
250897 = 250ms
19108 = 4aa4
 
对应线程
"SendMessageThread_102" #594 prio=5 os_prio=0 tid=0x00007db28404e800 nid=0x4aa4 waiting on condition [0x00007dadd8ccb000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x0000000540023508> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
        at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1067)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1127)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)
 
 
   Locked ownable synchronizers:
        - None
```

这里是mq机器上的一次200-500ms耗时范围的毛刺，具体时间在mq应用内没有打出，但一直怀疑这个时间与dirty writeback相关，查看writeback这里的耗时，姑且认为mq的毛刺也是256ms左右。

再看mq进程的内核调用堆栈（过滤了其他无关的）也存在这个page_falut开始，至wait_transaction_locked结束的堆栈，耗时250ms，与mq毛刺基本吻合。

到这里基本可以确定，200-500ms这种大毛刺，基本是由于这个调用栈导致的。

## 3 page_fault ~ wait_transaction_locked的分析

个人理解有两个相对较易的切入点：

- 为什么会有这个堆栈：page_fault怎么发生的
- 这个堆栈为什么慢：wait_transaction_locked为什么慢

下面分别进行分析

### 3.1 page_fault怎么发生的

page_fault出现的原因通常有两种

- 页表中找不到对应虚拟地址的PTE(第一种比较为人熟知，即内存中找不到对应的页)
- 对应虚拟地址的PTE拒绝访问

#### 3.1.1 缺页

内存中没有为CommitLog分配好足够的页，查看DLedger的源码，仅创建了mmap，而没有预热（提前申请好空间），那么频繁缺页基本是无可避免的。

这里可以通过预热来尝试优化。

#### 3.1.2 PTE拒绝访问

无论是demo还是mq机器上的该堆栈，都没有出现过handle_pte_fault，理论上这个堆栈不是因为PTE拒绝访问而进入的。但从我下载的源码看，经过handle_pte_fault应该是这个链路的必经之路（不排除学艺不精，以及源码对不上机器版本，源码版本4.9.4，机器版本4.9.4-1）。这里姑且还是分析一下pte_fault的场景作为参考。

由于毛刺均与dirty writeback关联，那么dirty writeback就是一个很好的排查方向

```
粗略调用栈
wb_do_writeback (fs/fs-writeback.c)
wb_check_old_data_flush （fs/fs-writeback.c)
for (;;) {
    (__writeback_inodes_wb) (fs/fs-writeback.c) (可能的分支路径，也到下面这里）
    writeback_sb_inodes (fs/fs-writeback.c)
    __writeback_single_inode    (fs/fs-writeback.c)
    do_writepages   (mm/page-writeback.c)
    generic_writepages  (mm/page-writeback.c)
    write_cache_pages   (mm/page-writeback.c)
    ## lock_page
    clear_page_dirty_for_io (mm/page-writeback.c)
    page_mkclean_one
    ## pte 写保护
    ## unlock page
}
```

在writeback流程中，确实存在pte写保护的情况，在有writeback的情况下，page_fault无可避免。但这一点暂时没想到优化方案。

### 3.2 wait_transaction_locked

如果要定位这里为啥慢，超出能力范围太多，可能投入产出不成正比

但是！注意这里的堆栈

```
finish_task_switch
schedule
wait_transaction_locked
add_transaction_credits
start_this_handle
jbd2__journal_start
__ext4_journal_start_sb
ext4_dirty_inode
__mark_inode_dirty
generic_update_time
file_update_time
ext4_page_mkwrite
do_page_mkwrite
do_fault
handle_mm_fault
__do_page_fault
do_page_fault
page_fault
-                java (19108)
    250897
```

发现wait_transaction_locked来自jdb2__journal_start这里，那么怀疑是否是系统开启了journal才会走到这里？关闭journal是否能避免这个方法执行，也就避免毛刺的发生？

检查系统配置发现，果然是开启了journal

```
#dumpe2fs /dev/sda5|grep has_journal
dumpe2fs 1.42.9 (28-Dec-2013)
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype needs_recovery extent 64bit flex_bg sparse_super large_file huge_file uninit_bg dir_nlink extra_isize

```

这里可以通过关闭journal来尝试优化。

## 4 下一步的方向

经过进一步的分析得出上面的这些内容，目前可以整理出两个可行的尝试方向：

* 为DLedger的数据文件增加预热，减少缺页中断发生（原来的RMQ是有这个功能的，重点怀疑）
* 关闭文件系统的journal或切换到别的文件系统

会先利用Demo应用进行测试，再确认下一步线上如何调整。 

## 5 后续
关闭journal后效果极佳，毛刺消失

## 6 关于数据安全的思考
关闭journal会损失linux文件系统的健壮性，这样是否会影响操作系统宕机时RocketMQ数据安全？评估的结果是不会：RocketMQ并不依赖单机的数据安全，由于raft协议多数ack的机制，在其他副本上是存在完全的数据的
