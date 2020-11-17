---
title: Redis基础使用篇二
weight: 2
---
[TOC]
##  Redis基础使用篇二

### 自带压测工具redis-benchmark
```bash
# 测试set命令的qps
[vagrant@xxxx bin]$ ./redis-benchmark -t set -q
SET: 81699.35 requests per second

# -P参数，它表示单个管道内并行的请求数量
[vagrant@xxxx bin]$ ./redis-benchmark -t set -P 2 -q
SET: 159489.64 requests per second
```

### Redis事务
MULTI 、 EXEC 、 DISCARD 和 WATCH 是 Redis 事务相关的命令。
`multi 事务的开始，exec 事务的提交，discard 事务的丢弃回滚`

 Redis 的事务根本不能算「原子性」，而仅仅是满足了事务的「隔离性」，隔离性中的串行化——当前执行的事务有着不被其它事务打断的权利。

通常 Redis 的客户端在执行事务时都会结合 pipeline 一起使用，这样可以将多次 IO 操作压缩为单次 IO 操作。
```bash
pipe = redis.pipeline(transaction=true)
pipe.multi()
pipe.incr("books")
pipe.incr("books")
values = pipe.execute()
```

**watch 机制**
一种乐观锁，watch 会在事务开始之前即multi之前盯监视1 个或多个关键变量，当事务执行时，即服务器收到了 exec 指令时，Redis 会检查关键变量自 watch 之后，是否被修改了 (包括当前事务所在的客户端)。如果关键变量被人动过了，exec 指令就会返回 null 回复告知客户端事务执行失败，这个时候客户端一般会选择重试。

关于，事务还需要在深入研究下，对比mysql的事务。redis为何不支持事务回滚？

### 弃用PubSub
**放弃PubSub**
一个消费者突然挂掉了，生产者会继续发送消息，Redis 停机重启，PubSub 的消息是不会持久化，消息丢失。PubSub 作为消息队列缺点众多，不适合生产环境使用。Redis5.0 新增了 Stream 数据结构，可以持久化消息队列，PubSub 可以放弃了。做为消息队列还有很多更专业的消息中间件。



### 两种持久化机制对比
`RDB快照 VS AOF日志`

#### RDB快照
快照是redis在持久化时，会fork出子进程，对内存数据拍快照，把快照内容同步到磁盘中。

快照的实现，通过`写时复制`(Copy On Write)实现。fork的子进程其实就是父进程的拷贝（系统的fork进程也是写时复制），子进程和进程拥有相同的内存数据，打开的文件描述，堆栈信息，代码等。父子进程内容一样，不是真的复制一份，而只是复制了引用，当对应的内容变化时才会产生新的副本。

子进程做数据持久化，它只是对内存数据结构进行遍历读取，然后序列化写到磁盘中。而父进程在后续的请求中可能会修改内存中的数据，此时利用写时复制，父进程会拷贝数据对应的页，在新的页上进行数据修改。

**快照触发规则**：n秒内如果超过m个key被修改就自动做快照（可配置），如：save 300 10 #300秒内容如超过10个key被修改，则发起快照保存。


#### AOF日志
AOF 日志存储，就是把对内容修改的指令的文本，追加保存到日志文件中。

但是AOF 日志在长期的运行过程中会变很大，数据库重启时需要加载 AOF 日志进行指令重放，就会无比漫长。所以需要定期进行 AOF 重写，给 AOF 日志进行瘦身，即AOF日志指令合并。

AOF 重写，即对AOF日志指令合并。Redis 提供了 bgrewriteaof 指令用于对 AOF 日志进行瘦身。开辟一个子进程对内存进行遍历转换成一系列 Redis 的操作指令，序列化到一个新的 AOF 日志文件中。序列化完毕后再将操作期间发生的增量 AOF 日志追加到这个新的 AOF 日志文件中。如incr num命令100次，原日志文件中保存100次这个命令，合并后只需用set num 100 即可代替原来的100个指令。

**AOF日志写磁盘机制fsync**

1. `appendfsync always `,收到写命令就立即强制写入磁盘，最慢的，但是保证完全的持久化，不推荐使用.
2. `appendfsync everysec `,每秒钟强制写入磁盘一次，在性能和持久化方面做了很好的折中，推荐
3. `appendfsync no`,完全依赖操作系统会将写操作先缓存，一定条件下写入磁盘，性能最好,持久化没保证

日志同步到磁盘的机制，每种开源软件都有类似的选择，如nginx日志，mysql的binlog，redolog等，我们需要在可靠性和性能之间做出取舍。充分利用写buffer性能高，但可靠性差点，强制同步到磁盘可靠，但是太慢。

`同步AOF日志到磁盘，是由异步线程完成的。`

#### RDB快照 VS AOF日志
快照需要fork子进程，开销大，还要遍历内存，做全量备份而不是增量备份，大块写磁盘会加重系统负载。
AOF 的 fsync 是一个耗时的 IO 操作，它会降低 Redis 性能，同时也会增加系统 IO 负担。

1. **可靠性**：AOF优于RDB
2. **日志文件**：AOF大于RDB
3. **性能**：RDB优于AOF
4. **灾备恢复**：很少使用 rdb 来恢复内存状态，因为会丢失大量数据。但是重放 AOF 日志性能相对 rdb 来说要慢很多。

#### Redis 4.0 混合持久化
为了解决rdb不可靠，aof太慢的问题，4.0将两者结合，AOF 日志不再是全量的日志，而是自持久化开始到持久化结束的这段时间发生的增量 AOF 日志，体积大大缩小。 Redis 重启的时候，可以先加载 rdb 的内容，然后再重放增量 AOF 日志就可以完全替代之前的 AOF 全量文件重放，重启效率因此大幅得到提升。

### 主从同步
主从同步， 从从同步

**增量同步**

同步的是指令流，Redis 的复制内存 buffer 是一个定长的环形数组，如果数组内容满了，就会从头开始覆盖前面的内容。如果因为网络状况不好，主节点中那些没有同步的指令在 buffer 中有可能已经被后续的指令覆盖掉了。

**快照同步**

非常耗费资源的操作，将快照文件的内容全部传送到从节点。从节点将快照文件接受完毕后，执行一次全量加载，加载之前先要将当前内存的数据清空。加载完毕后通知主节点继续进行增量同步。

在整个快照同步进行的过程中，主节点的复制 buffer 还在不停的往前移动，如果快照同步的时间过长或者复制 buffer 太小，都会导致同步期间的增量指令在复制 buffer 中被覆盖，这样就会导致快照同步完成后无法进行增量复制，然后会再次发起快照同步，如此极有可能会陷入快照同步的死循环。

所以要配置一个合适的复制环形 buffer 大小参数，避免快照复制的死循环。

**消息丢失**

Redis 主从采用异步复制，意味着当主节点挂掉时，从节点可能没有收到全部的同步消息，这部分未同步的消息就丢失了。wait 指令可以让异步复制变身同步复制。


### Sentinel哨兵
高可用，责持续监控主从节点的健康，当主节点挂掉时，自动选择一个最优的从节点切换为主节点。客户端来连接集群时，会首先连接 sentinel，通过 sentinel 来查询主节点的地址，然后再去连接主节点进行数据交互。当主节点发生故障时，客户端会重新向 sentinel 要地址，sentinel 会将最新的主节点地址告诉客户端。

Sentinel 会持续监控已经挂掉了主节点，待它恢复后，重新加入集群。

### info查看redis运行情况
主要有这些内容`Server,Clients,Memory,Persistence,Stats,Replication,CPU,Cluster,Keyspace`

```bash
# 获取所有信息
> info
# 获取内存相关信息
> info memory
# 获取复制相关信息
> info replication
```
Redis 每秒执行多少次指令？

```bash
> info stats
instantaneous_ops_per_sec:0 # 每秒执行多少次指令
total_net_input_bytes:9009657
total_net_output_bytes:7849006
...
```

Redis 连接了多少客户端？
```bash
> info clients
# Clients
connected_clients:1	# 客户端连接数
client_longest_output_list:0
client_biggest_input_buf:0
```

Redis 内存占用多大 ?
```bash
> info memory
used_memory_human:827.46K # 内存分配器 (jemalloc) 从操作系统分配的内存总量
used_memory_rss_human:3.61M  # 操作系统看到的内存占用 ,top 命令看到的内存
used_memory_peak_human:829.41K  # Redis 内存消耗的峰值
used_memory_lua_human:37.00K # lua 脚本引擎占用的内存大小
```

复制积压缓冲区多大？
```bash
> redis-cli info replication |grep backlog
repl_backlog_active:0
repl_backlog_size:1048576  # 积压缓冲区大小
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
```
复制积压缓冲区大小非常重要，它严重影响到主从复制的效率。当从库因为网络原因临时断开了主库的复制，然后网络恢复了，又重新连上的时候，这段断开的时间内发生在 master 上的修改操作指令都会放在积压缓冲区中,积压缓冲区是环形的，如果过多，后来的指令会覆盖掉前面的内容。

### redis与lua

用户可以向服务器发送 lua 脚本来执行自定义动作，获取脚本的响应数据。Redis 服务器会`单线程原子性执行 lua 脚本`，保证 lua 脚本在处理的过程中不会被任意其它请求打断。

通过lua可以将`del_if_equals`即匹配key和删除key合并为一个原子操作。
```lua
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

通过`EVAL 指令`执行lua脚本，可以将 lua 脚本压缩成一行以单引号围起来是为了方便命令行执行。参数放后面。
```bash
127.0.0.1:6379> set name james
OK
# EVAL 指令的第一个参数1是脚本内容字符串
127.0.0.1:6379> eval 'if redis.call("get",KEYS[1]) == ARGV[1] then return redis.call("del",KEYS[1]) else return 0 end' 1 name james
(integer) 1
```

**SCRIPT LOAD 和 EVALSHA 指令** 
SCRIPT LOAD 指令用于将客户端提供的 lua 脚本传递到服务器而不执行，但是会得到脚本的唯一 ID，用来标识服务器缓存的这段 lua 脚本。后面客户端就可以通过 EVALSHA 指令反复执行这个脚本了。

Redis 会保护主线程不会因为脚本的错误而导致服务器崩溃，近似于在脚本的外围有一个很大的 try catch 语句包裹。在 lua 脚本执行的过程中遇到了错误，同 redis 的事务一样，那些通过 redis.call 函数已经执行过的指令对服务器状态产生影响是无法撤销的。





