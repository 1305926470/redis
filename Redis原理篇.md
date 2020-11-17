---
title:  Redis原理篇
weight: 4
highlightjslanguages : [ "go"]
---
[TOC]
##Redis原理篇



###redis线程IO模型
redis是单线程的，单线程异步非阻塞实现高性能是很常见的，比如nodejs，nginx等。

通常可以用封装了io多路复用函数，如select，poll，evport，epoll(linux)和kqueue(freebsd & macosx)，比如libevent，memcache就是基于libevent来实现的。

```bash
追踪redis系统调用
[root@xxxxx ~]# strace -cp 31746
strace: Process 31746 attached
^Cstrace: Process 31746 detached
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 36.58    0.001813          16       114           epoll_wait
 29.62    0.001468          13       113           open
 27.02    0.001339           6       229           read
  6.50    0.000322           3       113           close
  0.28    0.000014          14         1           write
```

### Redis通信协议RESP
RESP(Redis Serialization Protocol) redis序列化协议，纯文本协议，没有其他而二进制协议高效，但是影响不大，数据库系统的瓶颈一般不在于网络流量，而是数据库自身内部逻辑处理上。文本协议简单，且便于阅读，结合业务场景，是不错选择。

常见的编程语言的redis客户端都实现了RESP协议。

**RESP协议内容** 

1. 单行字符串 以+符号开头；
2. 多行字符串 以$符号开头，后跟字符串长度；
3. 整数值 以:符号开头，后跟整数的字符串形式；
4. 错误消息 以-符号开头；
5. 数组 以*号开头，后跟数组的长度；

```bash
$11\r\nhello world\r\n
*3\r\n:1\r\n:2\r\n:3\r\n
*3\r\n$3\r\nset\r\n$6\r\nauthor\r\n$8\r\ncodehole\r\n
```

### redis过期策略怎么实现的？
redis 会将每个设置了过期时间的 key 放入到一个独立的字典中，以后会定时遍历这个字典来删除到期的 key。
除了定时遍历之外，还有惰性策略删除就是在客户端访问这个 key 的时候，对 key 的过期时间进行检查，如果过期了就立即删除。定时删除是集中处理，惰性删除是零散处理。

**定时扫描策略**：不是遍历所有的，而是从过期字典中随机 20 个 key，删除其中过期的key，如果过期的 key 比率超过 1/4，那就重复执行；为了不影响性能每次不会超过25ms。
但是这个25ms会让客户端请求等待（那为何过期策略不单独一个线程呢？）

**Redis 是单线程的，如果同一时间太多的 key甚至是全部key都过期会怎么样？**

Redis 会持续扫描过期字典 (循环多次)，直到过期字典中过期的 key 变得稀疏，此外内存还要频繁回收内存页，也会增加CPU开销，这样会导致读写请求出现明显的卡顿现象。所以redis的key过期时间应该具有一定的随机性。

**redis从库过期策略**从库时被动跟随主库的，不会有过期检测，但有del操作。如果同一时间过期太多，不及时同步的话，可能造成主从不一致。

### 内存回收机制
操作系统回收内存是以页为单位，如果这个页上只要有一个 key 还在使用，那么它就不能被回收。所以删除了key内存不会立刻就能回收。mysql删除数据也不是立刻被回收的。只有该页面所有的key都没了再回被回收。


### 哪些操作可能带来redis的卡顿呢？
对于单线程应用，时间复杂度为 O(n) 级别的指令，需要慎用，可能会造成卡顿。比如keys指令，比如同一时间有大量的key过期。
keys扫描大量的key；同一时间有太多的key过期；刷新增也同步磁盘；


### Redis Cluster

`Redis Cluster 是去中心化的`，将所有数据划分为 16384 的 slots，每个节点负责其中一部分槽位（是不是有点像一致性hash环的虚拟节点？）。

Redis Cluster槽位的信息每个redis节点都有存储，当 Redis Cluster 的客户端来连接集群时，它也会得到一份集群的槽位配置信息（这个信息长这么样子）。这样当客户端要查找某个 key 时，可以直接定位到目标节点（寻找到目标节点的算法又是什么？）。

客户端在拿到服务端操作信息后，需要缓存下来，用于定位key所在的节点。（如果服务端节点有变更这么通知客户端？）

#### 槽位定位算法
槽位定位算法，即根据key定位所在的redis节点。Cluster 默认会对 key 值使用 crc16 算法进行 hash 得到一个整数值，然后用这个整数值对 16384 进行取模来得到具体槽位。还有槽位和对应redis节点的映射表。

Cluster 还允许用户强制某个 key 挂在特定槽位上，即通过在 key 字符串里面嵌入 tag 标记，强制 key 所挂在的槽位等于 tag 所在的槽位。

**跳转**：当客户端向一个错误的节点发出了指令，该节点会发现指令的 key 所在的槽位并不归自己管理，这时它会向客户端发送一个特殊的跳转指令携带目标操作的节点地址，告诉客户端去连这个节点去获取数据。如，`MOVED 3999 127.0.0.1:6381`,MOVED 指令的第一个参数 3999 是 key 对应的槽位编号，后面是目标节点地址。客户端收到 MOVED 指令后，要立即纠正本地的槽位映射表。后续所有 key 将使用新的槽位映射表。

**槽点迁移**：在不同的redis节点间迁移槽点，均衡集群负载。可人工干预可自动。迁移过程...

**高可用**：每个节点可以设置从节点，单主节点故障时，集群会自动将其中某个从节点提升为主节点。

**网络抖动**：`cluster-node-timeout`设置节点失联超时时间，当某个节点持续 timeout 的时间失联时，才可以认定该节点出现故障，需要进行主从切换。如果没有这个选项，网络抖动会导致主从频繁切换 。

**可能下线与确定下线 **：一个节点认为某个节点失联了并不代表所有的节点都认为它失联了。所以集群还得经过一次协商的过程，只有当大多数节点都认定了某个节点失联了，集群才认为该节点需要进行主从切换来容错。

槽位迁移感知，集群变更感知....


### redis 真的只有一个线程吗？
redis是单线程异步模型，可以实现高性能，同时，单线程避免了并发编程的同步竞争问题，上下文切换开销也少了。但是还是有其他的线程和进程的。

快照持久化时，会fork子进程，同步数据到磁盘。还有几个异步线程专门用来处理一些耗时的操作，如在 4.0 版本引入了 unlink 指令，如果删除的key很大，它能对删除操作进行懒处理，丢给后台线程来异步回收内存即`懒惰删除(lazy free)`。同步AOF日志到磁盘，也是由异步线程完成的。


### 缓存淘汰算法 LFU vs LRU
当redis使用的物理内存超出限制（可配置），会执行缓存淘汰。

 LRU：最近时间最少使用；LFU （Least Frequently Used），最不经常使用，表示按最近的访问频率进行淘汰；如果一个 key 不是经常被访问，只是刚刚偶然被访问一下，也可能被LRU 算法当成热点数据。LFU 是需要追踪最近一段时间的访问频率，如果某个 key 只是偶然被访问一次是不足以变得很热的。

#### Redis LRU 实现
```c
struct redisServer {
       pid_t pid; 
       char *configfile; 
       //全局时钟
       unsigned lruclock:LRU_BITS; 
       ...
};
```
Redis内维护了一个24位时钟，默认是 Unix 时间戳对 2^24 取模的结果，大约 97 天清零一次。

当新增key对象的时候会把系统的时钟赋值到这个内部对象时钟redisObject .lru。每当key被访问时，redisObject .lru被重新赋值redis时钟，否则redisObject .lru不变。而redis时钟是在不断计时的，所以可以通过`redisServer.lruclock`与`redisObject .lru`相减计算出key的空闲时间。

如先后通过debug object bigstr查看
```bash
127.0.0.1:6379[2]> debug object bigstr
Value at:0x7f2441a57300 refcount:1 encoding:raw serializedlength:51 lru:14567278 lru_seconds_idle:7535

127.0.0.1:6379[2]> get bigstr
"lllllllllasdfsdfdskkkkkkkkkkkkkkkkkkkkkkkkkkkkasdfasdfskkkkkkkkkkkkkkkkkkkkkkkkadfasdfasdfsafkkkkkkkkkkkasdfasdfasfkkkkkkkk"

127.0.0.1:6379[2]> debug object bigstr
Value at:0x7f2441a57300 refcount:1 encoding:raw serializedlength:51 lru:14574823 lru_seconds_idle:3
```

常规LRU ，如果元素被访问了就被移到队列前面（可以链表实现），如果元素慢了，会淘汰掉队列尾部的元素。
redis的LRU不是通过队列来维护的。那么问题来了，怎么淘汰key？

从所有的key中随机选择N个（N可以配置）或从所有的设置了过期时间的key中选出N个键，然后再从这N个键中选出最久没有使用的一个key进行淘汰。（用的是堆排序吗？也许该翻一下redis源码了，呵呵）

#### LFU 实现
在 LFU 模式下，lru 字段 24 个 bit 用来存储两个值，高16位用来记录访问时间（单位为分钟），低8位用来记录访问频率，简称counter。
```
   16 bits             8 bits
  +------------------+---------------+
  + Last access time | logc counter  |
  +------------------+---------------+
```
低8位，最大才255，怎么存下较大的访问频率呢？
这里的频次不是简单的线性计数器，而是基于对数计数器来实现的。对应的概率分布计算公式为：
```
// LFU_INIT_VAL为5,默认概率因子server.lfu_log_factor=10
1/((counter-LFU_INIT_VAL)*server.lfu_log_factor+1)
```
随着key的访问量增长，counter最终都会收敛为255。且这个counter值还会随时间衰减。logc counter衰减将现有的 logc counter值减去对象的空闲时间 (分钟数) 除以一个衰减系数(lfu_decay_time)。logc counter值小则在内存不足时就更容易被回收。


### 惰性删除
Redis使用异步线程对已删除的节点进行内存回收。使用异步线程可以解放主线程，不必阻塞请求操作。但是多线程就涉及并发编程同步问题。

主线程需要将删除任务传递给异步线程，通过一个普通的双向链表来传递的。这里就会涉及线程同步竞争问题。（解决同步竞争问题要么悲观锁，要么乐观锁，CAS）redis使用了加锁保护来实现线程安全操作。当主线程将删除任务追加到队列之前它需要加锁，追加完毕后，再释放锁。既然涉及到加锁，释放锁，性能开销是不小的。


### 定时任务serverCron

Redis 的很多定时任务都是在serverCron里面完成的，比如大型 hash 表的渐进式迁移、过期 key 的主动淘汰、触发 bgsave、bgaofrewrite 等等。serverCron函数，每100ms执行一次，负责管理redis服务器资源。主要任务：

- 更新服务器缓存时间。我们平常获取时间戳会涉及系统调用，开销较大，redis会缓存时间戳，避免频繁的系统调用。
- 更新LRU时钟。
- 更新服务器每秒执行的次数等状态信息。即info命令的内容。
- 处理SIGTREAM信号。
- 管理客户端资源，管理数据库资源，
- 执行bgaofrewrite 
- 检查持久化操作运行状态等

