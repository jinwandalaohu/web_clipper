# Redis巡检及优化建议 - 机猿巧合 - 博客园
redis巡检

定期主动执行数据库巡检可以及时有效的发现数据库潜在问题，降低数据库运行风险。本巡检报告是针对Redis数据库主机的系统资源、数据库运行状态指标采集分析，得出调整建议供用户决策评估

2.1 系统部分[#](#3075329500)
------------------------

| **巡检项** | **描述** |
| --- | --- |
| 磁盘空间可用率 | 评估系统磁盘空间是否充足。 |
| 内存使用率 | 评估系统内存是否充足。 |
| swap分区配置 | 评估系统swap分区的使用权重 |
| numa架构 | 评估是否开启numa导致性能问题 |
| 操作系统日志 | 评估系统是否存在运行异常 |
| 系统资源ulimit | 评估数据库用户是否收到系统ulimit资源限制 |
| 操作系统版本/内核版本 | 评估是否会触发操作系统bug等问题 |
| 透明大页 | 评估系统是否开启透明大页（THP）特性 |
| 内存分配策略 | 评估系统是否可以充分分配物理内存 |
| TCP连接 | 评估系统高并发下是否容易发生丢包问题 |

2.2 数据库部分[#](#1656811875)
-------------------------

| **巡检项** | **描述** |
| --- | --- |
| 连续运行时长 | 可反映数据库连续的可用性 |
| **数据分布** | **反映不同****schema****的数据量** |
| **性能参数配置** | **反映数据库是否高效运行** |
| 慢日志 | 反应数据库是否高效运行 |
| 安全性参数配置 | 反映数据库是否存在安全风险 |
| 高可用架构 | 反映数据库架构及服务可用性 |
| 数据库复制状态 | 反映数据库复制同步状态及延迟情况 |
| 错误日志 | 反映数据库运行过程中是否存在异常 |

3.1版本信息[#](#2931508318)
-----------------------

### 3.1.1 系统版本[#](#1418815716)

| **实例**/ip | **检查结果**(操作系统版本) | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  |  | Normal |  |
|  |  | Normal |  |
|  |  | Normal |  |

### 3.1.2 Redis版本[#](#3428695324)

| **实例** | **检查结果**(redis版本) | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  |  | Normal |  |
|  |  | Normal |  |
|  |  | Normal |  |

3.2 系统参数[#](#3324280201)
------------------------

### 3.2.1 内存透明大页（THP）[#](#1131650121)

巡检方式

```


Copy

Copy

`# cat /sys/kernel/mm/transparent_hugepage/enabled
# cat /sys/kernel/mm/transparent_hugepage/defrag` 
```

巡检结果

| **实例**(ip:port) | **检查结果** | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  |  | Warning | 建议关闭，开启后fork子进程时会导致增加内存消耗 |
|  |  | Warning |  |
|  |  | Warning |  |

> 标准大页管理是预分配方式，而透明大页管理则是动态分配方式
> 
> 为什么Transparent HugePages(透明大页)对系统的性能会产生影响.
> 
> 　　在khugepaged进行扫描进程占用内存，并将4 kPage交换为Huge Pages的这个过程中，对于操作的内存的各种分配活动都需要各种内存锁，直接影响程序的内存访问性能。并且，这个过程对于应用是透明的，
> 
> 在应用层面不可控制，对于专门为4 k page优化的程序来说，可能会造成随机的性能下降现象。

### 3.2.2 内存策略[#](#3754205166)

```


Copy

Copy

`# cat /proc/sys/vm/overcommit_memory` 
```

巡检结果

| **实例**(ip:port) | **检查结果** | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  | 0 | Warning | 强烈建议设置为1，避免系统限制分配物理内存 |
|  | 0 | Warning |  |
|  | 0 | Warning |  |

> overcommit_memory取值又三种分别为0， 1， 2  
> overcommit_memory=0， 表示内核将检查是否有足够的可用内存供应用进程使用；如果有足够 的可用内存，内存申请允许；否则，内存申请失败，并把错误返回给应用进程。  
> overcommit_memory=1， 表示内核允许分配所有的物理内存，而不管当前的内存状态如何。  
> overcommit_memory=2， 表示内核允许分配超过所有物理内存和交换空间总和的内存  
> overcommit_memory参数就是控制分配内存是否可以超过CommitLimit，默认是0,即启发式的overcommitting handle,会尽量减少swap的使用,root可以分配比一般用户略多的内存。1表示允许超过CommitLimit,2表示不允许超过CommitLimit

### 3.2.3 SWAP[#](#3414699031)

巡检方式

```


Copy

Copy

`# cat /proc/sys/vm/swappiness` 
```

巡检结果

| **实例** | **检查结果** | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  | 30 | Warning | 强烈建议设置为1，避免使用 swap |

|  | 30 | Warning |  |

> 异步任务中 将部分内存交互到磁盘，后续又需要使用，需要交换出来  
> 导致任务执行变长，比如 垃圾回收时，需要遍历进程中维护的全局对象，但是若  
> 交换出去了，还需要在交换进来

### 3.2.4 TCP SYN[#](#4007304781)

巡检方式

```


Copy

Copy

`# cat /proc/sys/net/ipv4/tcp_max_syn_backlog` 
```

巡检结果

| **实例** | **检查结果** | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  | 2621440 | Normal | 建议设置为65535 |
|  | 2621440 | Normal |  |
|  | 2621440 | Normal |  |

> 服务因为队列满产生丢包，其中一个做法就是加大半/全连接队列的长度。 半连接队列长度Linux内核中，主要受tcp\_max\_syn_backlog影响 加大它到一个合适的值就可以

### 3.2.5 CP Accept[#](#3970354294)

巡检方式

```


Copy

Copy

`# cat /proc/sys/net/core/somaxconn` 
```

巡检结果

| **实例** | **检查结果** | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  | 4096 | Notice | 建议设置为65535 |
|  | 4096 | Notice |  |
|  | 4096 | Notice |  |

> 定义了系统中每一个端口最大的监听队列的长度
> 
> 该内核参数默认值一般是128，对于负载很大的服务程序来说大大的不够。一般会将它修改为2048或者更大。

### 3.2.6 TCP Timewait[#](#2014519888)

巡检方式

```


Copy

Copy

`# cat /proc/sys/net/ipv4/tcp_max_tw_buckets
# cat /proc/sys/net/ipv4/tcp_tw_reuse
# cat /proc/sys/net/ipv4/tcp_tw_recycle` 
```

巡检结果

| **实例** | **检查结果** | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  | 131072/1/1 | notice | 建议tcp\_tw\_reuse和/tcp\_tw\_recycle参数不同时开启 |
|  | 131072/1/1 | notice |  |
|  | 131072/1/1 | notice |  |

> tcp\_max\_tw\_buckets 该参数设置系统的TIME\_WAIT的数量，如果超过默认值则会被立即清除。
> 
> tcp\_tw\_reuse 表示是否允许将处于TIME-WAIT状态的socket(TIME-WAIT的端口)用于新的TCP
> 
> tcp\_tw\_recycle 能够更快地回收TIME-WAIT套接字

### 3.2.7 **TCP Keepalive**[#](#3637046754)

巡检方式

```


Copy

Copy

`# cat /proc/sys/net/ipv4/tcp_keepalive_time
# cat /proc/sys/net/ipv4/tcp_keepalive_intvl
# cat /proc/sys/net/ipv4/tcp_keepalive_probes` 
```

巡检结果

| **实例** | **检查结果** | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  | 6/1/3 | Notice | 建议设置为120/15/3，加速回收无效连接 |
|  | 6/1/3 | Notice |  |
|  | 6/1/3 | Notice |  |

> tcp\_keepalive\_time 控制 TCP/IP 尝试验证空闲连接是否完好的频率
> 
> 需要更快地发现丢失了接收方，考虑减小这个值。 如果长期不活动的空闲连接出现次数较多，而丢失接收方的情况出现较少，需要提高该值以减少开销。
> 
> tcp\_keepalive\_time，在TCP保活打开的情况下，最后一次数据交换到TCP发送第一个保活探测包的间隔，即允许的持续空闲时长，或者说每次正常发送心跳的周期，默认值为7200s（2h）。
> 
> tcp\_keepalive\_intvl，在tcp\_keepalive\_time之后，没有接收到对方确认，继续发送保活探测包的发送频率，默认值为75s。
> 
> tcp\_keepalive\_probes 在tcp\_keepalive\_time之后，没有接收到对方确认，继续发送保活探测包次数，默认值为9（次）

### 3.2.8 TCP Syncookies[#](#2017962997)

巡检方式

```


Copy

Copy

`# cat /proc/sys/net/ipv4/tcp_syncookies` 
```

巡检结果

| **实例** | **检查结果** | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  | 1 | Notice | 建议设置为0，关闭cookies特性 |
|  | 1 | Notice |  |
|  | 1 | Notice |  |

```


Copy

Copy

`服务端半连接池满了以后是否开启syncookie机制 如果 SYN 半连接队列已满，默认会丢弃连接
开启 syncookies 功能就可以在不使用 SYN 队列的情况下成功建立连接。
 0 表示关闭该功能；
2 表示无条件开启功能；
1 则表示仅当 SYN 半连接队列放不下时，再启用它。
应当把 tcp_syncookies 设置为 1，仅在队列满时再启用。` 
```

3.3 安全[#](#1235746905)
----------------------

### 3.3.1 认证[#](#2995007127)

巡检方式

```


Copy

Copy

`redis> CONFIG GET requirepass` 
```

巡检结果

| **实例** | **检查结果** | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  | 已开启 | Normal | 建议提高密码强度 |
|  | 已开启 | Normal |  |
|  | 已开启 | Normal |  |

### 3.3.2 复制认证[#](#2970516889)

巡检方式

```


Copy

Copy

`redis> CONFIG GET "masterauth"` 
```

巡检结果

| **实例** | **检查结果** | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  | 已开启 | Normal | 建议提高密码强度 |
|  | 已开启 | Normal |  |
|  | 已开启 | Normal |  |

### 3.3.3 命令重命名[#](#1994128716)

巡检方式

```


Copy

Copy

`# cat redis.conf |grep -i rename-command` 
```

巡检结果

| **实例** | **检查结果** | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  | 未开启 | Notice | 建议将风险较高的命令重命名避免误操作如 FLUSHDB FLUSHALL |
|  | 未开启 | Notice |  |
|  | 未开启 | Notice |  |

3.4 持久化[#](#2838509518)
-----------------------

### 3.4.1 RDB持久化[#](#1857012204)

巡检方式

```


Copy

Copy

`redis> CONFIG GET save` 
```

巡检结果

| **实例** | **检查结果** | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  | 已开启（900 1 300 10 60 10000） | Notice | 当前使用默认持久化设置建议可根据业务需求做出调整 |
|  | 已开启（900 1 300 10 60 10000） | Notice |  |
|  | 已开启（900 1 300 10 60 10000） | Notice |  |

### 3.4.2 AOF 持久化[#](#471613620)

巡检方式

```


Copy

Copy

`redis> CONFIG GET appendonly` 
```

巡检结果

| **实例** | **检查结果** | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  | No | Notice | 可根据持久化需求选择性开启 |
|  | No | Notice |  |
|  | No | Notice |  |

3.5 内存[#](#2950988732)
----------------------

### 3.5.1 当前内存使用情况[#](#1976642133)

巡检方式

```


Copy

Copy

`redis > info Memory` 
```

巡检结果

| **实例** | **检查结果** | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  | 24.12M | Normal | 当前内存使用率接近机器内存的一半，需要注意，若服务器上还有其他占用内存的进程，redis在有fork等操作时可能出现问题 |
|  | 14.45M | Normal |  |
|  | 14.44M | Normal |  |

### 3.5.2 当前内存使用上限[#](#2246517979)

巡检方式

```


Copy

Copy

`redis > CONFIG GET maxmemory` 
```

巡检结果

| **实例** | **检查结果** | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  | 未设置 | Notice | 视业务需求调整，建议设置为3-4G |
|  | 未设置 | Notice |  |
|  | 未设置 | Notice |  |

### 3.5.3 内存淘汰策略[#](#1975090812)

巡检方式

```


Copy

Copy

`redis > CONFIG GET "maxmemory-policy"` 
```

> volatile-lfu 是从所有配置了过期时间的键中驱逐使用频率最少的键
> 
> volatile-lru 是加入键的时候如果过限，首先从设置了过期时间的键集合中驱逐最久没有使用的键

巡检结果

| **实例** | **检查结果** | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  | Noeviction | Notice | 默认无删除策略，写满后禁止新的写入。建议根据键的使用调整为 其他删除策略。 |
|  | Noeviction | Notice |  |
|  | Noeviction | Notice |  |

### 3.5.4 内存碎片率[#](#2842389686)

巡检方式

```


Copy

Copy

`redis > INFO Memory` 
```

巡检结果

| **实例** | **检查结果** | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  | 1.16 | Normal | 正常 |
|  | 1.42 | Normal |  |
|  | 1.21 | Normal |  |

3.6 SLOWLOG[#](#2540700805)
---------------------------

巡检方式

```


Copy

Copy

`redis > CONFIG GET "slowlog-max-len"
redis > CONFIG GET slowlog-log-slower-than` 
```

巡检结果

| **实例** | **检查结果** | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  | 128/10000 | Notice | 建议增大慢日志条数，降低阈值 |
|  | 128/10000 | Notice |  |
|  | 128/10000 | Notice |  |

3.7 服务日志[#](#64887237)
----------------------

巡检方式

```


Copy

Copy

`redis > CONFIG GET "loglevel"` 
```

巡检结果

| **实例** | **检查结果** | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  | Notice | Normal |  |
|  | Notice | Normal |  |
|  | Notice | Normal |  |

3.8 超时时间[#](#3146886299)
------------------------

巡检方式

```


Copy

Copy

`redis > CONFIG GET "timeout"` 
```

巡检结果

| **实例** | **检查结果** | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  | 0 | Normal | 0 为无限制，建议设置为 1800 或 3600 |
|  | 0 | Normal |  |
|  | 0 | Normal |  |

3.9 复制缓冲[#](#1896763471)
------------------------

巡检方式

```


Copy

Copy

`redis > CONFIG GET "repl-backlog-size"` 
```

巡检结果

| **实例** | **检查结果** | **Normal/Notice/Warning** | **备注** |
| --- | --- | --- | --- |
|  | 1048576（1M） | Notice | 复制及压缓冲区建议设置为1G，避免网络抖动后进行全量同步 |
|  | 1048576（1M） | Notice |  |
|  | 1048576（1M） | Notice |  |

| **IP地址** | **系统参数** | **安全** | **持久化** | **内存使用** | **SLOWLOG** | **日志** | **Timeout** | **复制** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
|  | 可优化调整 | 正常 | 可优化调整 | 正常 | 可优化调整 | 正常 | 可优化调整 | 可优化调整 |
|  | 可优化调整 | 正常 | 可优化调整 | 正常 | 可优化调整 | 正常 | 可优化调整 | 可优化调整 |
|  | 可优化调整 | 正常 | 可优化调整 | 正常 | 可优化调整 | 正常 | 可优化调整 | 可优化调整 |

4.1 风险评估[#](#302221295)
-----------------------

### 4.1.1 操作系统[#](#3837232093)

### 4.1.2 redis服务[#](#2546674707)

4.2 优化建议[#](#2950788204)
------------------------

### 4.2.1 操作系统[#](#2344592205)

> 1.  建议关闭透明大页，提高Redis持久化效率，降低fork期间的内存消耗
>     
> 2.  建议设置为1，将SWAP使用倾向降到最低，注意不要设置为0
>     
> 3.  vm.overcommit_memory =0 在内存不足是无法分配物理内存，建议设置为 1
>     
> 4.  建议关闭syncookie，调高tcp\_max\_syn_backlog、somaxconn值，避免高并发情况下增大连接建立的开销、出现网络丢包
>     
> 5.  tcp\_max\_tw\_buckets、tcp\_tw\_reuse、tcp\_tw_recycle等参数当前为默认值，无需调整，若调整不当会导致TCP连接丢包
>     

4.2.2 Redis服务

**注：以下优化建议可能在当前Reids版本无法全部适用**

> 1.  建议使用Redis 5.0版本
>     
> 2.  降低RDB持久化触发的频率，或者在业务允许的情况下关闭RDB持久化
>     
> 3.  建议设置最大内存上限及内存淘汰策略，避免Redis当存储使用
>     
> 4.  建议增大Redis复制缓冲，保证网络抖动尽可能的不出发全量同步
>     
> 5.  建议将风险度较高的命令重命名，避免误操作影响Redis服务运行
>     
> 6.  建议增大慢日志记录条数
>     
> 7.  建议配置redis日志，logfile
>     

sentinel建议：

> 1.  建议配置日志文件，有助于问题的排查:logfile
>     
> 2.  建议根据业务情况调整down-after-milliseconds,现在是10ms，建议调整为30ms
>