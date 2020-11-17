---
title: Redis基础使用篇一
weight: 1
---
[TOC]
##  Redis基础使用篇一


### Redis  5 种基础数据结构
string (字符串)、list (列表)、set (集合)、hash (哈希) 和 zset (有序集合)。
应用自己脑补...

### redis的原子操作
incr decr

### 分布式锁
多个进程同时去设置key，存在说明已经加锁，加锁失败。如果设置key成功，则说明获得了锁。对于value也要具有随机性，最好是不会重复的，比如时间戳+客户端实例编号。解锁时还要凭此值解锁。对于随机数官方推荐从 /dev/urandom 中取 20 个 byte作为随机数。

```bash
# 申请分布式锁， 值要足够随机，解锁时还要用
SET resource_name random_value NX PX 10000
```

`SET NX` 命令只会在 key 不存在的时候给 key 赋值。
`PX` 表示保存这个 key 10000ms。即`锁过期时间`

### 延时队列
Redis 的消息队列不是专业的消息队列，没有非常多的高级特性，没有 ack 保证，如果对可靠性有着极致的追求，就不适合使用。使用rpush/lpush操作入队列，使用lpop 和 rpop来出队列。

#### blpop/brpop阻塞读
阻塞读在队列没有数据的时候，会立即进入休眠状态，一旦数据到来，则立刻醒过来。消息的延迟几乎为零。从而避免while(1) 中的sleep延时问题了。

#### Redis 的消息队没有可靠性保证。
pop出消息后，list 中就没这个消息了，如果处理消息的程序拿到消息还未处理就挂掉了，那消息就丢失了，所以是不可靠队列。

### 位图bitmap
位图不是特殊的数据结构，它的内容其实就是普通的字符串，也就是 byte 数组。使用普通的 get/set 直接获取和设置整个位图的内容，也可以使用位图操作 getbit/setbit 等将 byte 数组看成「位数组」来处理。
`零存零取，整存零取`

### 布隆过滤器 (Bloom Filter) 
原理，见算法数据结构。 在位图基础上结合hash映射的改进，用于判断是否出现过，但是非常节省空间。`存在的可能存在（小概率误判），不存在的一定不存在（准确）`

Redis 4.0 之后，布隆过滤器作为一个插件加载到 Redis Server 中。布隆过滤器有二个基本指令，`bf.add` 添加元素，`bf.exists` 查询元素是否存在
```bash
# Bloom Filter
127.0.0.1:6379> bf.add ucenter user1
(integer) 1
127.0.0.1:6379> bf.add ucenter user2
(integer) 1
127.0.0.1:6379> bf.exists ucenter user1
(integer) 1
127.0.0.1:6379> bf.exists ucenter user3
(integer) 0
```

### HyperLogLog
HyperLogLog 提供不精确的去重计数方案，虽然不精确但是也不是非常不精确，标准误差是 0.81%。

HyperLogLog 提供了两个指令。pfadd 和 pfcount，增加计数和获取计数。
```bash
127.0.0.1:6379> pfadd ucenter user1
(integer) 1
127.0.0.1:6379> ucenter user2
(integer) 1
127.0.0.1:6379> pfadd ucenter user3 user4 user5 user6
(integer) 1
127.0.0.1:6379> pfcount ucenter 
(integer) 6
```
HyperLogLog算法原理。感兴趣可以研究下。

**限流和频率限制**
利用一个时间滑动窗口，（tcp的读写缓冲区也是个滑动窗口，也起到控制速度的作用）。结合zset的score。

###  地理位置 GEO 模块GeoHash
使用 Redis 来实现附近的餐馆，附近的人功能。

Redis的GEO 模块基于地理位置距离排序算法 GeoHash 有序集合zset实现。

Geohash其实就是将整个地图或者某个区域进行一次划分，对每个区域进行编号。对更小的区域再次进行划分和编号。重复。

有点像身份证前六位编号。第一、二位表示居民所在省。第三、四位表示居民所在市。第五、六位表示居民所在县。比如321323为江苏宿迁泗阳。

这样`前缀匹配`越多的地点就越接近，越精确。GeoHash就是一种将经纬度转换成字符串的方法，字符串前缀匹配越多的距离越近。

GeoHash 算法会继续对位置整数做一次 base32 编码 (0-9,a-z 去掉 a,i,l,o 四个字母) 变成一个字符串。在 Redis 里面，经纬度使用 52 位的整数进行编码，放进了 zset 里面，score 是 GeoHash 的 52 位整数值即地理位置。通过score即可得到附近的地点了。
```bash
# Redis 提供的 Geo 指令
geoadd 添加元素（地点和坐标）
geodist 计算两个元素之间的距离
geopos 获取集合中任意元素的经纬度坐标
geohash 获取元素的经纬度编码字符串
georadiusbymember 按半径查询指定元素附近的其它元素
```

### 暴力扫描指令keys
暴力匹配，支持模式匹配，时间复杂度O(n)，更可怕的是没有limit和offset之类的东东，如果key非常多，会找出所有匹配的，会造成卡顿，由于redis单线程，会影响其他的请求。线上慎用这个指令。
```bash
127.0.0.1:6379[2]> keys *
 1) "enter"
 2) "glass"
 3) "deny"
 4) "abc"
 5) "dddd"
 6) "dess"
 7) "about"
 8) "desc"
 9) "diss"
10) "footer"
11) "dkkk"
12) "know"

# 找出a开头的所有key
127.0.0.1:6379[2]> keys a*
1) "abc"
2) "about"
```


### 改进版扫描scan

scan算是keys的改进版，通过游标分步进行，不阻塞线程；提供 count参数（用于限制扫描行数，不是返回行数）
`scan cursor [MATCH pattern] [COUNT count]`
```bash
127.0.0.1:6379[2]> keys d*
1) "deny"
2) "dddd"
3) "dess"
4) "desc"
5) "diss"
6) "dkkk"

127.0.0.1:6379[2]> scan 0 match d* count 5
1) "10"
2) 1) "deny"
   2) "dkkk"

127.0.0.1:6379[2]> scan 0 match d* count 15
1) "0"
2) 1) "deny"
   2) "dkkk"
   3) "dess"
   4) "desc"
   5) "diss"
   6) "dddd"

```


