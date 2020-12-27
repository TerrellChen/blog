---
title: Linux下一次IO毛刺排查
date: 2020-05-31 17:04:00
tags:
---
<!-- toc -->

## 1 问题背景

排查RocketMQ的投递RT抖动问题，偶然发现生产环境的RocketMQ机器的写入耗时存在毛刺。由于暂时没有定位到投递RT抖动具体是磁盘IO导致还是网络IO导致，遂尝试先解决IO毛刺看问题能否恢复。下面分享下本次定位过程的思路和用到的工具。

先简单分享下结果，IO毛刺是系统writeback引起的。这个结果比较出我意料，因为在印象里，RocketMQ的所有文件，都有主动调用Flush，没道理轮到系统对脏页进行回收。

## 2 毛刺的发现-IO监控

首先想到iostat，这是监控磁盘运行负载状况的一大利器。关于iostat的使用网上一搜一大把，就不再赘述。当然更推荐直接使用man去理解iostat，很多时候监控指标的精确定义是理解指标的关键。

在机器上使用iostat结果如下：

```
#iostat -xmt 1
05/31/2020 02:22:54 AM
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.84    0.00    0.40    0.02    0.00   98.75

Device:         rrqm/s   wrqm/s     r/s     w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00  4255.00    0.00  591.00     0.00    32.26   111.80     0.03    0.05    0.00    0.05   0.05   2.70

05/31/2020 02:22:55 AM
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.65    0.00    0.38    0.04    0.00   98.94

Device:         rrqm/s   wrqm/s     r/s     w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00  4618.00    0.00 1105.00     0.00    32.16    59.61     6.78    4.70    0.00    4.70   0.03   39.30

05/31/2020 02:22:56 AM
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.58    0.00    0.36    0.04    0.00   99.02

Device:         rrqm/s   wrqm/s     r/s     w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00  4170.00    0.00  500.00     0.00    29.09   119.17     0.03    0.06    0.00    0.06   0.05   2.40

05/31/2020 02:22:57 AM
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.77    0.00    0.33    0.02    0.00   98.87

Device:         rrqm/s   wrqm/s     r/s     w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00  4223.00    0.00  515.00     0.00    30.33   120.62     0.02    0.04    0.00    0.04   0.04   2.30

05/31/2020 02:22:58 AM
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.71    0.00    0.38    0.04    0.00   98.87

Device:         rrqm/s   wrqm/s     r/s     w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00  4487.00    0.00  509.00     0.00    32.76   131.80     0.02    0.04    0.00    0.04   0.04   2.00

05/31/2020 02:22:59 AM
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           1.09    0.00    0.38    0.04    0.00   98.50

Device:         rrqm/s   wrqm/s     r/s     w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00  4400.00    0.00  521.00     0.00    30.50   119.91     0.04    0.07    0.00    0.07   0.06   3.30

05/31/2020 02:23:00 AM
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.63    0.00    0.38    0.06    0.00   98.94

Device:         rrqm/s   wrqm/s     r/s     w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00  4466.00    1.00  794.00     0.00    31.27    80.55     1.27    1.34    0.00    1.34   0.04   3.60

05/31/2020 02:23:01 AM
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.79    0.00    0.46    0.13    0.00   98.62

Device:         rrqm/s   wrqm/s     r/s     w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00  5116.00    0.00 2630.00     0.00    43.98    34.24     0.09    0.03    0.00    0.03   0.03   8.30
```



**w_await**

每5秒一次的w_await升高非常显眼，是首先需要关注的指标。关于w_await的定义如下：

```
The average time (in milliseconds) for write requests issued to the device to be served. This includes the time spent by the requests in queue and the time spent servicing them.
```

w_await辖内的时间既包括在请求队列中等待的时间，也包括硬件处理的时间。这个时间变长，意味着我们的IO耗时变长，也即是所谓的IO毛刺。

**%util**

util也是需要关注的指标，通常情况下，其定义如下：

```
Percentage  of  elapsed time during which I/O requests were issued to the device (bandwidth utilization for the device). Device saturation occurs when this value is close to 100%.
```

通常w_await变高，伴随着%util达到100%。这是一个很简单的关联逻辑，当磁盘负载满时，IO请求队列中的请求无法得到及时响应，w_await自然也就升高。这里的%util一次达到了39%，一次仅有3.6%，说明此时磁盘负载并没有完全爆满。

但是这里有一个需要注意点是iostat的统计时间。如果iostat每1秒输出一次，统计1秒内的数据，那么这里的%util意味着1秒内，块设备的负载情况并没有爆满，同时也保留了块设备在这1秒内的某一小段时间，如100ms的范围内，负载完全爆满的可能性。

**avgqu-sz**

直接上定义：

```
The average queue length of the requests that were issued to the device.
```

设备请求队列平均长度，通常这个值超过经验值1，即说明磁盘无法及时处理完IO请求队列。这里一次达到6.14，一次是1.58，结合%util，基本可以确定，在统计的这1秒内的某一小段时间，设备负载是满的。

**w/s 与 wrqm/s**

```
wrqm/s
The number of write requests merged per second that were queued to the device.
w/s
The number (after merges) of write requests completed per second for the device.
```

图中可以看到，写请求的数量有一个倍增，然而写合并的数量并没有变化。这里可以得到两个信息：写请求是周期性的，并且非常分散。

**此部分的总结**

到这里，我基本可以确定这里的IO写毛刺，是周期性（5s一次）的大量写请求导致的。根据我对RocketMQ代码的了解，其中并没有什么刷盘逻辑是5秒一次的，遂直接往系统怀疑。

系统周期性的IO，很自然的可以想到脏页的写回（Dirty Page Writeback）。下面将验证这一点。

## 3 脏页与写回(Dirty Page & Writeback)

相信大家对Linux的内核子系统都有个大概的了解。在这里我简单总结下脏页与写回的概念：

```
	为了在CPU性能与块设备性能之间做平衡，引入了多级缓存的概念。将块设备中的文件，映射到内存中的某一段区域，对文件的修改即转化为对内存的修改，速度得以指数级提升。当然，这就涉及到何时将修改的内存写入到块设备中去的问题。
	显然，将一个文件映射的全部内存完整的覆盖到块设备中效率非常低下，那么对这部分内存空间进行逻辑上的切分，并且只将修改的部分写入块设备，显而易见是一个更好的方案。
	这里的缓存空间即被称作Page Cache，切分后的变成一页页Page，而其中被修改过的Page即是脏页。脏页需要被操作系统定期的更新至块设备中以防内存断电后数据丢失，更新的操作即是写回。
```

关于脏页与写回，内核中有许多参数提供给用户配置，是用户具有调整其中部分逻辑的能力，完整的相关参数介绍可以见：https://www.kernel.org/doc/Documentation/sysctl/vm.txt。配置文件位于/proc/sys/vm/。

说回上面5秒一次周期性IO毛刺，很显然首先需要关注的点是系统写回的频率，这个参数是dirty_writeback_centisecs（单位厘秒，这个参数具体是影响脏页周期性检查的间隔，详见上文链接）。查看该参数：

```
cat /proc/sys/vm/dirty_writeback_centisecs
500
```

与5秒一次的周期完全吻合。为了验证是否是该参数导致的问题，我在这里试着将该参数调整为10秒，之后iostat中可见，毛刺的周期变为了10s一次：

```
sudo echo 1000 > /proc/sys/vm/dirty_writeback_centisecs
```

到这里，可以看出IO毛刺与Writeback是存在关联的。但是，奇怪的点也在这里，RocketMQ的主要文件包括CommitLog、IndexFile、ConsumeQueue，这所有的文件，印象中在代码里均有主动Flush逻辑，且间隔不超过1秒（后面的事实证明我在之前没有带着问题看代码的时候错过了一些细节^_^）。

为了找到（实锤）Writeback具体操作的哪些文件，我尝试对Writeback这一行为进行Debug。感谢内核在4.x之后存在（提供）了相应的工具，得以完成这一工作。

## 4 更底层的监控

万幸在做Java开发之前干过几天系统运维的工作，使我得以想到tracepoint这个linux 内核的基础设施。再次基础上经过一番资料查阅后，找到**bpftrace**及**bcc**这两个工具。

```
bpftrace: https://github.com/iovisor/bpftrace
bcc: https://github.com/iovisor/bcc
```

对于这两个工具的具体细节暂时还不够水平也没有精力进行分析，在大多数情况下，根据其文档会用应该能满足非系统应用开发的需要了。关于安装和使用在主页中有对应文档，不再赘述。值得一提的是，这两个工具对Linux内核版本貌似要求4.x以上。

幸运的是，bpftrace在自带的工具中存在对writeback进程监控的脚本，在bcc的脚本中，我又找到了记录系统所有IO请求的脚本，下面来看两个工具的使用。

**bpftrace/writeback.bt-监控writeback的发生及耗时**

工具安装完成后使用非常简单：

```
#/usr/share/bpftrace/tools/writeback.bt
Attaching 4 probes...
Tracing writeback... Hit Ctrl-C to end.
TIME      DEVICE   PAGES    REASON           ms
01:59:47  8:0      46702    periodic         0.590
01:59:47  8:0      46702    periodic         0.000
01:59:52  8:0      46965    periodic         100.305
01:59:52  8:0      46965    periodic         0.002
01:59:52  8:0      46965    periodic         0.000
01:59:57  8:0      46917    periodic         0.729
01:59:57  8:0      46917    periodic         0.000
02:00:02  8:0      48674    periodic         82.553
02:00:02  8:0      48674    periodic         0.000
02:00:07  8:0      48224    periodic         0.791
02:00:07  8:0      48224    periodic         0.000
02:00:12  8:0      43899    periodic         77.174
02:00:12  8:0      43899    periodic         0.000
```

非常友好的输出，可以看出，定时任务(periodic)每5秒一次，会有一个巨大的延时，也是对应之前提到的IO毛刺。这里的REASON项，除了periodic之外，还可能有background等触发脏页写回的其他原因导致的启动。

**bcc/biosnoop.py-监控所有IO请求**

使用同样比较简单：

```
#/usr/share/bcc/tools/biosnoop
TIME(s)     COMM           PID    DISK    T SECTOR     BYTES  LAT(ms)
0.000000    java           10958  sda     W 997907720  135168    0.07
0.000212    jbd2/sda5-8    1573   sda     W 1933891352 204800    0.07
.......................
1.906624    kworker/u96:0  2406   sda     W 254486424  4096      0.22
1.906639    kworker/u96:0  2406   sda     W 284985880  4096      0.23
.......................
7.074782    kworker/u96:0  2406   sda     W 759013280  4096      0.91
7.074786    kworker/u96:0  2406   sda     W 759013744  4096      0.91
7.074789    kworker/u96:0  2406   sda     W 759013800  4096      0.91
.......................
```

以执行命令开始的时间为基准进行累加，工具统计了接下来发生的所有IO请求的信息，包括顺序、发起的进程、IO类型、扇区、数据大小、耗时。

COMM一列下，kworker即是系统进程，这里的写入均可以认为是Writeback发起的。而根据IO请求的扇区我们即可拿到Writeback到底写了哪些文件。

**fdisk/debugfs-查询扇区对应文件**

这里通过四步步拿到扇区对应的文件：

- 查询扇区所在设备
- 计算扇区Block
- 根据Block查询Inode
- 根据Inode查询文件

```
#fdisk -l
.......

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1            2048        4095        1024   83  Linux
/dev/sda2   *        4096   104861695    52428800   83  Linux
/dev/sda3       104861696   125833215    10485760   82  Linux swap / Solaris
/dev/sda4       125833216  3748659199  1811412992    5  Extended
/dev/sda5       125835264  3748659199  1811411968   83  Linux
```

第一步使用fdisk命令可以拿到各设备的扇区范围，以SECTOR 254486424为例，它在/dev/sda5上。

```
#expr \( 254486424 - 125835264 \) / 8
16081395
```

第二步，计算器计算对应的Block，公式为（目标扇区-设备起始扇区）/ 8。

```
#debugfs /dev/sda5
debugfs 1.42.9 (28-Dec-2013)
debugfs:  icheck 16081395
Block	Inode number
16081395	33292549
debugfs:  ncheck 33292549
Inode	Pathname
33292549	/............./consumequeue/............
```

第三步第四步，使用debugfs，可以依次查询到block所属的inode，及inode对应的文件。至此，定位Writeback涉及到哪些文件的工作即完成了大半，剩下的即是查询所有出现过的请求中的扇区对应的文件了。

## 5 震惊，RocketMQ竟然是这样刷盘的

这一小节与RocketMQ相关，与Linux无关，不感兴趣RocketMQ的可以直接跳过。

经过一段时间艰苦的统计，Writeback涉及的文件如下：

- 50%的IO次数（20%左右的数据量） ConsumeQueue
- 10%的IO次数（60%左右的数据量）IndexFile
- 剩余：各种日志文件

出现了这么多的ConsumeQueue及IndexFile，与我的印象出现了非常大的偏差。仔细阅读RocketMQ相关代码发现，所谓的刷盘逻辑竟然还有内置的机关！

```
ConsumeQueue刷盘
	1秒执行一次，遍历所有ConsumeQueue，但只在ConsumeQueue新增数据量超过一定大小（指定page数）才执行Flush
	60秒（默认，可配置）的内置计时器，会触发所有ConsumeQueue一定Flush一次
IndexFile刷盘
	只对关闭的上一个IndexFile主动执行Flush
```

果然不带着问题的阅读一定会错过很多细节，简单来说可以认为IndexFile完全没刷盘，依赖5秒一次的系统Writeback进行Flush，而ConsumeQueue随缘刷盘，流量较低的ConsumeQueue基本上会被系统Writeback调用刷盘。

## 6 为什么会有毛刺

再回到之前拿到的所有IO请求的记录上，可以每次Writeback的系统IO请求量大而连续。

基本可以断定IO毛刺出现的原因是短时间（1秒内的某个ms级别的时间短）内磁盘负载因为请求队列数量过高而满载，同时也阻塞了应用进程的IO请求。

IO请求队列内的请求处理顺序由调度算法决定，各种算法的详细介绍可以自行搜索，这里仅简单介绍以满足阅读需要

```
noop
	大致FIFO，做了一些优化可能导致饿死的情况
cfq
	每个线程/进程一个队列，尽量确保公平
deadline
	读写队列分开，尽量避免饿死
```

当前生效的IO调度算法在/sys/block/${device}/queue/scheduler这里可以查看或设置，这台机器目前使用的deadline算法

```
#cat /sys/block/sda/queue/scheduler
noop [deadline] cfq
```

deadline算法的核心参数包括write_expire(单位ms)，会尽量避免写操作的等待执行时间超过该参数

```
#cat /sys/block/sda/queue/iosched/write_expire
5000
```

这里系统的毛刺峰值离5秒还有些距离，也意味着处理写请求基本按照FIFO原则，即Writeback的同一时间（极短时间内）产生的大量请求会阻塞其他进程的IO请求，导致毛刺。同时大量的IO请求本身也会导致较高的写延迟。

## 7 不均匀的Writeback-迷之dirty_expire_centisecs

在之前，毛刺的产生已经定位，这里是另一个问题。在尝试拉长**iostat**与**biosnoop**监控结果的时间线进行分析后，又发现了一个问题：

- 5秒一次的毛刺之外，存在着30秒一次的大毛刺
- 在30秒的大毛刺之际，Writeback产生的IO请求的数量是平时5秒毛刺时的10倍左右

基本可以认定，这个较大的毛刺是因为更多的Writeback产生的IO请求而导致，那么为什么会出现这样不均匀的情况？

了解过Writeback机制的你一定知道**dirty_expire_centisecs**这个参数。对Dirty Page进行Writeback有两类条件，一类占用内存到一定比例，一类即是Page被标记为Dirty超过一定时间，这个一定时间即是**dirty_expire_centisecs**(单位ms)参数。

```
#cat /proc/sys/vm/dirty_expire_centisecs
3000
```

查看该参数为3000，这意味着一个Page将在被标记Dirty的30s之后，被下一个周期的Writeback线程Flush到块设备。通常看来，如果系统流量平均，那么每时每刻，达到30s阈值的Dirty Page数量应该同样多，为什么会产生一个30秒周期的大毛刺，难道说系统流量（RocketMQ）每30秒有一波大流量（然而并没有）？

不禁怀疑自己掌握的**dirty_expire_centisecs**的概念是否正确，但查阅内核文档后与自己的印象完全吻合：

```
# https://www.kernel.org/doc/Documentation/sysctl/vm.txt

dirty_expire_centisecs

This tunable is used to define when dirty data is old enough to be eligible
for writeout by the kernel flusher threads.  It is expressed in 100'ths
of a second.  Data which has been dirty in-memory for longer than this
interval will be written out next time a flusher thread wakes up.
```

再尝试搜索了一下**dirty_expire_centisecs**的实现，有收获：

```
# https://stackoverflow.com/questions/18353467/implementation-of-dirty-expire-centisecs

I asked this question on the linux-kernel mailing list and got an answer from Jan Kara. The timestamp that expiration is based on is the modtime of the inode of the file. Thus, multiple pages dirtied in the same file will all be written when the expiration time occurs because they're all associated with the same inode.

http://lkml.indiana.edu/hypermail/linux/kernel/1309.1/01585.html
```

在下面附上的邮件链接中也写到：

```
Well, let me explain the mechanism in more detail: When the first page is
dirtied in an inode, the current time is recorded in the inode. When this
time gets older than dirty_expire_centisecs, all dirty pages in the inode
are written. So with this mechanism in mind the behavior you describe looks
expected to me.
```

这是说，当一个inode下首次有dirty page时，时间即开始记录在inode上，达到超时时间后，inode所有的dirtypage（即是没到这个dirty page的超时时间）都会被一同writeback。

具体到RocketMQ的业务场景来说，由于密集的请求，在MQ启动之后，每一个检查周期所有的的ConsumeQueue文件都可能被修改过。那么很大概率，大部分ConsumeQueue在MQ启动的首个检查周期即被计时，直到**dirty_expire_centisecs**时间达到后，又一同被writeback，往复循环。所以会存在一个30s的大毛刺。

## 8 优化方案

目前想到三个思路：

- 引入更频繁的Writeback避免5秒一次的毛刺
- 减少Writeback避免短时间大量IO请求的拥堵
- 尝试别的IO请求调度算法，使业务线程的IO请求不被阻塞出耗时毛刺

**引入更频繁的Writeback避免5秒一次的毛刺**

这个操作的理论基础是将5秒一次的IO分散到更细粒度。但比较可惜在这里并不适合RocketMQ的业务场景，因为RocketMQ的流量过于巨大，每1秒产生的脏页所涉及到的文件，与每5秒统计一次并没有差别。

实测结果：将**dirty_writeback_centisecs**调整为1s后，每一秒都是毛刺。

**减少Writeback避免短时间大量IO请求的拥堵**

RocketMQ自身的所有数据文件基本都有定时flush的机制，可以不依赖系统Writeback。那么可以直接把周期性的Writeback的间隔拉长，或直接关闭周期性的Writeback。

实测结果：

- 间隔拉长：将dirty_expire_centisecs参数调整为180秒后，5秒一次的毛刺完全消失，但180秒的毛刺仍然存在。

- 关闭周期性的writeback：理论上只影响日志的flush，不会引入断电后数据丢失的问题。但目前只有线上环境，还不敢测试。

**尝试别的IO请求调度算法**

对其他的IO请求算法还需要更细致的调研才有进行测试的必要和把握，粗略看来cfq这种进程间相对更公平的算法，也许能够避免在Writeback出现大量IO时，应用进程仍能够合理分配到足够的份额。
