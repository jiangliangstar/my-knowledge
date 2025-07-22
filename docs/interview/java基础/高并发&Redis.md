## 秒发系统如何设计
### 痛点
1. 瞬时并发量大
	1. 大量用户同一时间访问
	2. 网站瞬时访问流量激增
2. 库存少
	1. 访问请求数量远远大于库存数量
	2. 只有少部分用户能够秒杀成功
### 架构设计
![[Pasted image 20250714160804.png]]
#### 访问层
动态页面，访问压力大
![[Excalidraw/Drawing 2025-06-26 14.40.31.excalidraw]]
将商品页静态化，用户访问速度 ⬆️，服务端压力 ⬇️。
![[Excalidraw/Drawing 2025-07-14 16.13.23.excalidraw]]
##### 秒杀按钮
1. 活动前禁用按钮
2. 点击后禁用按钮
3. 滑动验证码
4. 排队体验，提升用户体验
避免不必要的请求，防止用户重复提交，防止同一时间黄牛操控上百台手机薅羊毛，排队页面，防止用户不停刷新
#### 中间层
![[Pasted image 20250714163453.png]]
##### Nginx 负载均衡
Nginx 负载均衡，单台 Nginx 并发量 2~3 万左右请求，如果 Nginx 进行集群，就需要再上层就需要部署硬件级别的负载均衡器，F5/LVS。
Nginx 负载均衡到网关之后，在网关还有客户端负载均衡，比如 rabbion 来进行客户端的负载均衡分发，处理每秒 10 万以上的 QBS 的并发量。结合 Docker，K8S 进行云服务器的动态伸缩的部署。秒杀开始前自动扩容，结束后自动缩减服务器节点数量，有效的利用服务器资源，节省服务器成本。
##### 限流
还需要做好限流，请求中参杂着爬虫，羊毛党的一些无效请求，Nginx 配置限流，防止一些绕过前端的 dDos 攻击，网关层通过 Sentinel 对不同的服务节点，去设置限流以及熔断的机制。
##### 削峰填谷
通过MQ 做削峰填谷，通过mq，减少下流压力，防止请求搞垮数据库

#### 服务端
##### Redis
Redis 缓存，减轻数据库压力。
###### 预热 
通过定时器等，将秒杀用到的一些商品，库存等信息，预热到 Redis 中，防止 Resid 被击穿，压垮数据库
###### 库存
通过 lua 脚本，进行库存操作
###### 防重
set NX 命令，同一个用户的 token 或者 IP + 当前商品 URL ，保证同一个时间，同一个商品，同一个用户只能有一个有效。
###### 分布式锁

#### mq
用户秒杀成功，减完库存之后，需要进行下单操作，直接访问数据库的话，瞬时激增流量，数据库扛不住。
![[Excalidraw/Drawing 2025-07-14 17.07.01.excalidraw]]
异步下单，限流

#### 数据库
##### 读写分离
##### 分库分表

以上，从访问层，到数据库，还有一些其他的，比如安全问题，性能监控，预警等细节。


# 补充Redis 的 `SET NX` 命令

`SET NX` 是 Redis 中用于实现分布式锁和原子性设置键值的重要命令。

## 基本语法

```redis
SET key value NX
```

或者完整形式：
```redis
SET key value [NX] [XX] [EX seconds] [PX milliseconds] [KEEPTTL]
```

其中 `NX` 表示 "Only set the key if it does not already exist"（仅当键不存在时才设置）。

## 功能特性

1. **原子性操作**：设置键值和对存在性检查是原子操作
2. **分布式锁基础**：常用于实现简单的分布式锁
3. **返回值**：
   - 设置成功时返回 `"OK"`
   - 键已存在时返回 `nil`

## 典型应用场景

### 1. 实现分布式锁

```redis
SET lock:resource "unique_token" NX EX 30
```

- 如果返回 `OK` 表示获取锁成功
- 如果返回 `nil` 表示锁已被占用

### 2. 防止重复操作

```redis
SET order:123:processing "1" NX EX 60
```

防止同一订单被重复处理。

### 3. 限流控制

```redis
SET rate:limit:user1 "1" NX EX 10
```

实现简单的限流功能。

## 注意事项

1. **锁释放**：获取锁后必须确保最终释放锁
2. **锁续期**：长时间操作可能需要锁续期机制
3. **竞争条件**：简单实现可能存在竞争条件，生产环境建议使用 Redlock 等更完善的算法
4. **值唯一性**：锁的值应使用唯一标识，防止误删其他客户端的锁

## 相关命令

- `SETNX`：旧版命令，功能等同于 `SET NX`（但不支持设置过期时间）
- `GETSET`：可用于实现锁的获取和释放
- `DEL`：用于释放锁

`SET NX` 是 Redis 分布式系统编程的基础构建块之一，正确使用可以解决很多并发控制问题。


## 商品类型菜单树优化  
1. 直接从数据库中读取，随着数据量增加，数据库瓶颈展示出来，越查越慢  
2. 使用redis缓存，先从redis中读取，没有再从数据库中读取，减少数据库的查询次数，提高查询效率，缓存10分钟，但是每隔10分钟，访问首页就会特别慢  
3. 加入定时任务，每隔5分钟，从数据库中读取数据，存入redis中 4. 压测之后，发现 QBS 只有 100 多，redis 遇到瓶颈，需要部署集群，但是资金有限，无法部署集群，只能采用其他方式解决，使用二级缓存，使用 JVM 缓存，首先从 JVM 缓存中读取，没有再从 redis 中读取，没有再从数据库中读取，同时更新 JVM 缓存和 redis 缓存  
4. 商品分类达到上万之后，数据传输到前端之后，比较消耗网络性能，再 nginx 开启 gzip 压缩，减少网络传输的数据量，提高网络传输效率，原来 1M 的数据，压缩后只有 100K，减少了 90% 的网络传输数据量  
5. redis 中存在大 key，之前分类树采用 key：value 进行存储，首先把商品分类中多余的字段去掉，单独封装一个新的简洁版商品分类，把每个字段的名字进行简写，再通过 gzip 压缩，再获取的时候，将 gzip 压缩的数据进行解压，再反序列化成json数据，再转换成分类树

引入redis，定时任务，二级缓存，Nginx 数据压缩，优化 redis 存储数据结构

## 如何定位线上 OOM
### 造成 OOM 的原因
1. 应用一次性申请对象太多，数据量大。分页等方式
2. 内存资源耗尽，未释放，线程，数据库查询，高并发的场景，会内存溢出。使用池化技术。
3. 本身分配资源不足， jmap -heap 查看堆信息。
### 如何快速定位 OOM
1. 系统挂掉了：找到 dump 文件，`-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=`
2. 系统运行中，还未 OOM，导出 dump 文件 `jmap -dump:format=b,file=xxx.hporf 端口`
3. 结合工具进行调试

## Redis实现上亿用户实时积分排行榜
1. 数据量大
2. 实时
redis 中存储 ZSet


# Redis 中的 ZSet（有序集合）

ZSet（Sorted Set）是 Redis 中一种特殊的数据结构，它结合了 Set 和 Hash 的特点，每个元素都会关联一个分数（score），Redis 通过这个分数来为集合中的元素进行排序。

## 基本特性

1. **唯一性**：与 Set 类似，ZSet 中的元素是唯一的，不能重复
2. **有序性**：元素按照分数（score）从小到大排序
3. **高性能**：即使集合中有大量元素，也能高效地进行操作
![[Pasted image 20250714175333.png]]

分桶， 
![[Pasted image 20250714175427.png]]
## 常用命令

### 添加/更新元素
```bash
ZADD key [NX|XX] [CH] [INCR] score member [score member ...]
```
- `NX`：只添加新元素，不更新已存在元素
- `XX`：只更新已存在元素，不添加新元素
- `CH`：返回被修改的元素总数（包括新增和更新的）
- `INCR`：将分数增加指定值（类似 ZINCRBY）

### 查询操作
```bash
# 获取元素分数
ZSCORE key member

# 获取元素排名（从0开始）
ZRANK key member     # 升序排名
ZREVRANK key member  # 降序排名

# 按分数范围获取元素
ZRANGE key start stop [WITHSCORES]      # 升序
ZREVRANGE key start stop [WITHSCORES]   # 降序

# 按分数范围获取元素（指定分数区间）
ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]
ZREVRANGEBYSCORE key max min [WITHSCORES] [LIMIT offset count]

# 获取集合中元素数量
ZCARD key

# 获取指定分数范围内的元素数量
ZCOUNT key min max
```

### 删除操作
```bash
# 删除指定元素
ZREM key member [member ...]

# 按排名范围删除
ZREMRANGEBYRANK key start stop

# 按分数范围删除
ZREMRANGEBYSCORE key min max
```

### 集合运算
```bash
# 并集
ZUNIONSTORE destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM|MIN|MAX]

# 交集
ZINTERSTORE destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM|MIN|MAX]
```

## 使用场景

1. **排行榜系统**：用分数表示用户得分，自动排序
2. **带权重的队列**：分数可以表示优先级
3. **时间线系统**：用时间戳作为分数
4. **范围查询**：如查找价格在某个区间的商品
5. **延迟队列**：用时间戳作为分数，定时获取到期任务

## 示例

```bash
# 添加元素
ZADD leaderboard 1000 "user1" 2000 "user2" 3000 "user3"

# 获取排名
ZRANK leaderboard "user2"  # 返回1（升序排名）

# 获取前两名
ZRANGE leaderboard 0 1 WITHSCORES
# 返回：
# 1) "user1"
# 2) "1000"
# 3) "user2"
# 4) "2000"

# 按分数范围查询
ZRANGEBYSCORE leaderboard 1500 2500
# 返回：
# 1) "user2"

# 增加用户分数
ZINCRBY leaderboard 500 "user1"  # user1的分数变为1500
```

ZSet 是 Redis 中非常强大的数据结构，特别适合需要排序和范围查询的场景。

## 订单超时自动取消怎么实现
使用队列
1. jdk 自带队列
2. MQ 延时队列
3. Redis 过期监听
![[Pasted image 20250714181218.png]]
![[Pasted image 20250714180123.png]]
![[Pasted image 20250714181303.png]]
![[Pasted image 20250714181320.png]]

![[Pasted image 20250714181533.png]]![[Pasted image 20250714181847.png]]

# 分布式锁
Nginx 分布式集群部署后，同步锁是 JVM 级别，只能锁住单个进程，主流的方案有 Redis 和 Zookeeper，
Redis 的 setNX 实现分布式锁，需要过期时间，防止服务器挂掉导致死锁。

问题：当锁过期时间到了，线程1 业务代码没有处理完，锁会释放，线程1，处理完，释放的是 线程2 的锁，其他的线程就会开始处理业务。
1：过期时间  2：释放了其他线程的锁
1：加长过期时间，添加子线程，确认主线程是否在线，在线的话，将过期时间重设，锁续命
2：给锁加唯一 id（UUID），保证不会释放其他线程的锁。
开源组件， redisson
### redisson原理
![[Pasted image 20250715164616.png]]
当 Redis 主节点关掉了，怎么办。使用 redlock

# 共同好友
zset 存储，用户的 ID 作为 key，好友作为 value 存储，`sinterstore` 取交集
内存当磁盘用，有的浪费，1亿好用不建议放到 Redis 中。
MySQL
Neo4j 
Hbase + Hadoop

# 内存受限的情况读取大文件，统计重复内容

使用缓冲区，分块读取 + Map
极端场景的话 Map 会很大

内存存不下，就放到磁盘中，`buffer[i] % 1024`，得到 1024 个分片文件，然后逐个统计。

# 如何定位&避免死锁
死锁：并发下，线程因为互相等待对方资源，导致“永久”阻塞的现象。
互斥：共享资源只能被一个线程占有 -- 互斥锁
占有且等待：

# Redis 
## 击穿，雪崩，穿透
击穿：一个key 非常热点，在不停的扛着高并发，当 key 失效瞬间，持续的大并发就击穿缓存，直达数据库，导致性能问题。
	永不过期
	互斥锁，分布式锁控制只有一个请求去查询数据库
	提前续期
雪崩：缓存集中过期，或者缓存服务器宕机，导致大量请求访问数据库，造成数据库压力大，宕机。
	差异化过期时间
	多级缓存
	熔断降级
	