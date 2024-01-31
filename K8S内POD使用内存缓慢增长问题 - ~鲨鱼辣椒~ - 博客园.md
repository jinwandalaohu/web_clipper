# K8S内POD使用内存缓慢增长问题 - ~鲨鱼辣椒~ - 博客园
背景
--

生产环境服务容器化后，部分服务频繁触发内存使用超80%告警，POD内存限制内存以及JVM内存设置如下

```null
resources:
    requests:
    	cpu: 1000m
    	memory: 2200Mi
    limits:
    	cpu: 3000m
    	memory: 3000Mi

```

```null
JAVA_OPTS='-Xmx2000m'

```

问题排查步骤
------

通过查看K8S pod的监控，发现POD使用内存缓慢上升，并且超过限制的80%，那么首先想到的是JVM存在内存泄漏，那么首先排查JVM的内存使用

### JVM

通过命令登录到容器内，通过命令行查看JVM内存的使用情况

```null
root@m2-queryserver-5d6d84d7f6-7mkfm:/# ps -ef | grep java
root           1       0  0 Nov21 ?        00:00:00 /bin/sh -c java $JAVA_OPTS -jar /opt/app.jar --server.port=$PORT --spring.profiles.active=$PROFILE
root           7       1  1 Nov21 ?        03:50:49 java -Xmx2000m -jar /opt/app.jar --server.port=8215 --spring.profiles.active=prod
root       64352   64329  0 10:43 pts/1    00:00:00 grep java


root@m2-queryserver-5d6d84d7f6-7mkfm:/# jhsdb jmap --heap --pid 7
Attaching to process ID 7, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 11.0.16+8

using thread-local object allocation.
Garbage-First (G1) GC with 2 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 40
   MaxHeapFreeRatio         = 70
   MaxHeapSize              = 2097152000 (2000.0MB)
   NewSize                  = 1363144 (1.2999954223632812MB)
   MaxNewSize               = 1258291200 (1200.0MB)
   OldSize                  = 5452592 (5.1999969482421875MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 1048576 (1.0MB)

Heap Usage:
G1 Heap:
   regions  = 2000
   capacity = 2097152000 (2000.0MB)
   used     = 168122616 (160.33422088623047MB)
   free     = 1929029384 (1839.6657791137695MB)
   8.016711044311524% used
G1 Young Generation:
Eden Space:
   regions  = 31
   capacity = 817889280 (780.0MB)
   used     = 32505856 (31.0MB)
   free     = 785383424 (749.0MB)
   3.9743589743589745% used
Survivor Space:
   regions  = 8
   capacity = 8388608 (8.0MB)
   used     = 8388608 (8.0MB)
   free     = 0 (0.0MB)
   100.0% used
G1 Old Generation:
   regions  = 124
   capacity = 484442112 (462.0MB)
   used     = 127228152 (121.33422088623047MB)
   free     = 357213960 (340.66577911376953MB)
   26.262818373642958% used

```

通过分析堆内存的使用情况可以看出，堆内存的使用率只有8%，应该不是堆内存中存在泄露的问题，那么是否是堆外内存泄露呢？

```null
root@m2-queryserver-5d6d84d7f6-7mkfm:/# jstat -gc 7
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT    CGC    CGCT     GCT   
 0.0   8192.0  0.0   8192.0 798720.0 226304.0  473088.0   124246.2  115184.0 108560.3 14832.0 12093.4  18981  223.826   0      0.000 14212  189.636  413.461

```

*   `S0C`, `S1C`: Survivor 0 和 Survivor 1 区域的当前容量（单位：KB）。
*   `S0U`, `S1U`: Survivor 0 和 Survivor 1 区域已使用的空间（单位：KB）。
*   `EC`: Eden 区域的当前容量（单位：KB）。
*   `EU`: Eden 区域已使用的空间（单位：KB）。
*   `OC`: 老年代的当前容量（单位：KB）。
*   `OU`: 老年代已使用的空间（单位：KB）。
*   `MC`: Metaspace 的当前容量（单位：KB）。
*   `MU`: Metaspace 已使用的空间（单位：KB）。
*   `CCSC`: Compressed Class Space 的当前容量（单位：KB）。
*   `CCSU`: Compressed Class Space 已使用的空间（单位：KB）。
*   `YGC`: 新生代垃圾收集次数。
*   `YGCT`: 新生代垃圾收集所花费的总时间（单位：秒）。
*   `FGC`: Full GC (整堆垃圾回收) 次数。
*   `FGCT`: Full GC 所花费的总时间（单位：秒）。
*   `CGC`: 并发标记扫描垃圾回收次数，对应 G1 垃圾收集器，对付其他GC策略此列无数据。
*   `CGCT`: 并发标记扫描垃圾回收所花费的总时间（单位：秒）。
*   `GCT`: GC 总时间，包括新生代GC、Full GC以及并发标记扫描GC的时间（单位：秒）。

查看Metaspace所使用的内存很小，系统中也不存在直接操作内存的功能，那么堆外内存可以暂时排除嫌疑

### 内存占用排查

JVM所使用的内存应该没有问题，那么POD中缓慢上升的内存被哪里占用了呢？只能从POD本身去排查

容器中使用`free -h`命令查看POD中内存的使用

```null
root@m2-queryserver-5d6d84d7f6-7mkfm:/# free -h
               total        used        free      shared  buff/cache   available
Mem:            14Gi       5.8Gi       4.3Gi        13Mi       4.8Gi       8.8Gi
Swap:             0B          0B          0B

```

可以看到POD中执行`free`命令显示是宿主机的总体内存状态，并不是容器本身的内存限制或者使用情况。这是因为大多数容器技术（包括Docker和Kubernetes）使用的是Linux的cgroups（控制组）功能来限制资源，而在容器内部运行的进程实际上是直接运行在宿主机的内核上的。

那么如果想要查看容器的内存限制或者用量可以查看`/sys/fs/cgroup/memory`目录下的相关文件

*   `/sys/fs/cgroup/memory/memory.limit_in_bytes` 文件会显示容器的内存限制。
*   `/sys/fs/cgroup/memory/memory.usage_in_bytes` 文件会显示容器当前的内存用量。
*   `/sys/fs/cgroup/memory/memory.stat` 文件会显示容器当前的内存明细

查看`memory.stat`可以看到POD的内存使用明细

```null
root@m2-queryserver-5d6d84d7f6-7mkfm:/# cat /sys/fs/cgroup/memory/memory.stat
cache 741126144     # 缓存内存的大小，它是为文件系统页缓存分配的内存。
rss 2392379392      # RSS是物理内存中未被交换出去的部分，包含了所有非可换出的、已被应用程序占用的内存。
rss_huge 2006974464 # 使用大页（通常是2MB大小）的内存量。
shmem 0             # 共享内存的大小，包括tmpfs等。
mapped_file 135168  # 已映射到文件的内存的大小。
dirty 0             # 等待写回到磁盘的脏页数量。
writeback 135168    # 正在被写回到磁盘的脏页数量。
swap 0              # 交换空间的使用量。
pgpgin 2149653      # 从磁盘读入（或从交换区调入）的总页数。
pgpgout 1932268     # 写回到磁盘（或写入交换区）的总页数。
pgfault 1177737     # 表示缺页异常的次数。
pgmajfault 0        # 需要从磁盘读取数据的主要缺页异常的次数。
inactive_anon 2392178688   # 表示在匿名内存中不活跃的页数。
active_anon 0			   # 表示在匿名内存中活跃的页数。
inactive_file 616534016    # 表示文件缓存中不活跃的页数。
active_file 124129280      # 表示文件缓存中活跃的页数。
unevictable 0              # 无法被驱逐的内存页数。
hierarchical_memory_limit 3145728000  # 针对该cgroup及其子cgroup的内存限制。
hierarchical_memsw_limit 3145728000   # 针对该cgroup及其子cgroup的内存加交换空间的限制。
# 前缀为 total_ 的字段表示所有子cgroup加上当前cgroup的总值。
total_cache 741126144     
total_rss 2392379392
total_rss_huge 2006974464
total_shmem 0
total_mapped_file 135168
total_dirty 0
total_writeback 135168
total_swap 0
total_pgpgin 2149653
total_pgpgout 1932268
total_pgfault 1177737
total_pgmajfault 0
total_inactive_anon 2392178688
total_active_anon 0
total_inactive_file 616534016
total_active_file 124129280
total_unevictable 0

```

文件中输出的内容较多，我们只需要关注前几项，根据显示的数据

*   `cache`: 741126144字节，约706.79MB，可能表示该容器用于缓存文件的内存。
*   `rss`: 2392379392字节，约2281.55MB，表示该容器实际使用的物理内存（不包括缓存和缓冲区）。
*   `rss_huge`: 1937768448字节，约1914MB，表示分配给该容器的大页内存。

POD中的cache为什么占用了如此多的内存呢？

众所周知的是操作系统的内存会有一部分被buffer、cache所占用，Linux会将这部分的内存算到已使用中，那么对于容器而言，容器引发的cache会算到容器占用的内存上。

这里简单介绍一下cache和buffer的区别：

*   Buffers 是内核缓冲区用到的内存，对应的是 /proc/meminfo 中的 Buffers 值。
*   Cache 是内核页缓存和 Slab 用到的内存，对应的是 /proc/meminfo 中的 Cached 与 SReclaimable 之和。

这些数值都来自 /proc/meminfo，关于 Buffers、Cached 和 SReclaimable 的含义如下所示：

*   Buffers 是对原始磁盘块的临时存储，也就是用来缓存磁盘的数据，通常不会特别大(20MB左右)。这样，内核就可以把分散的写集中起来,统一优化磁盘的写入，比如可以把多次小的写合并成单次大的写等等。
*   Cached是从磁盘读取文件的页缓存，也就是用来缓存从文件读取的数据。这样，下次访问这些文件数据时，就可以直接从内存中快速获取，而不需要再次访问缓慢的磁盘。
*   SReclaimable 是Slab的一部分。Slab 包括两部分，其中的可回收部分，用SReclaimable记录;而不可回收部分，用SUnreclaim记录。

**简单的说 buff 是对磁盘的缓存，而cache是对文件的缓存。** 

回到我们最初的问题，既然容器中的cache占用较高，那么会不会是日志文件频繁的读写造成的呢？

现在POD中服务产生的日志会先写到指定路径的文件中，然后由K8S的SLS插件将日志收集到SLS系统中。

查看历史的日志可以发现每天的日志量大概在2-3G左右，那么很大的可能是由于对于日志的频繁读写导致容器使用的cache较高，那么我们将cache清掉观察看内存是否会下降

proc/sys是一个虚拟文件系统，可以通过对它的读写操作做为与kernel实体间进行通信的一种手段。我们可以通过修改/proc中的文件，来对当前kernel的行为做出调整。通过调整/proc/sys/vm/drop_caches来释放内存。其默认数值为0。

当其值为 1时，表示仅清除页面缓存（PageCache）：

```null
sync; echo 1 > /proc/sys/vm/drop_caches

```

当其值未2 时，表示清除目录项和inode：

```null
sync; echo 2 > /proc/sys/vm/drop_caches 

```

当其值未3 时，表示清空所有缓存（pagecache、dentries 和 inodes）

```null
sync; echo 2 > /proc/sys/vm/drop_caches 

```

那么我们在容器中执行清除缓存的命令：

```null
root@m2-queryserver-5d6d84d7f6-7mkfm:/data/logs/history# sync; echo 3 > /proc/sys/vm/drop_caches
bash: /proc/sys/vm/drop_caches: Read-only file system

```

执行后发现是一个只读文件，不允许我们进行更改。

我们的POD是运行在宿主机上的，对于系统内核以及相应的变更不允许在POD中进行更改，那么我们需要在宿主机上执行缓存清除的命令。

```null
[root@iZuf6ippaf67o0c5ukfyq0Z ~]# sync;echo 3 > /proc/sys/vm/drop_caches;

```

宿主机上清除缓存后，进入POD中查看内存使用情况：

```null
root@m2-queryserver-5d6d84d7f6-7mkfm:~# cat /sys/fs/cgroup/memory/memory.stat
cache 1622016
rss 2392567808
rss_huge 2006974464
......

```

观察cache的变化，可以发现从700Mi变为1M左右，查看K8S监控发现POD所使用的内存明显下降

解决方案
----

由于系统产生日志较多导致操作系统在读写日志时占用了部分内存，而POD中对于缓存的计算并不是通过我们部署POD设置的Limit进行计算，而是根据宿主机的内存量进行cache大小的计算，宿主机可用内存较大，有可能会导致cache较大，从而导致POD使用内存缓慢增长，甚至超过Limt限制引发OOM问题。

那么如何解决这个问题呢，有几个解决方案

1.  宿主机上定时清除缓存。对宿主机上所有POD都会有影响，会降低机器性能，需要尽量在空闲时间执行。
2.  减少日志的输出量。治标不治本，并不能完全解决问题。
3.  日志不再写文件落盘，直接将日志发送到日志系统（SLS）。不再读写文件自然不会存在cache缓慢增加，但是需要更改代码。
4.  增加POD的Limit大小，给cache的增长留下足够的空间。这种方式对于内存的有些浪费，而且具体预留多少内存不太好设定。