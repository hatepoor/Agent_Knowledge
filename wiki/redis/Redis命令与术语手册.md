---
tags: [meta, redis, 术语, 命令, 索引, 零基础]
type: meta
created: 2026-09-29
updated: 2026-10-06
---

# Redis 命令与术语手册

> **这份文件存在的唯一目的：您永远不会因为"不认识某个词"而卡住。**
>
> 这里收录：Redis 命令、命令选项、redis-py 方法、专有术语、内部编码名、报错用词。
>
> 使用方式：按 `Ctrl+F` / `Cmd+F` 搜索那个词。**不需要理解，先查到再说。**
> 本页**只给签名与一句话**，详细机制与实操都在「出现在哪」指向的正文页。

**怎么读**

| 列 | 含义 |
|----|------|
| **一句话解释** | 这个词是干什么的 |
| **出现在哪** | 首次被讲解的页面 |

**标注**：⭐ = 入门阶段就会用到；`N.x+` = 需要 Redis N.x 及以上版本；📦 = redis-py 相关。

**查不到怎么办**：① 核对拼写（命令全大写、术语看英文原名）→ ② `redis-cli` 里 `HELP 命令` → ③ 官方命令文档 <https://redis.io/docs/latest/commands/>。还没有就是中段/进阶内容，见 [[redis/学习路线图]]。

---

## 一、服务与连接 ⭐

| 词 | 一句话解释 | 出现在哪 |
|----|-----------|----------|
| `redis-server` | 服务端本体程序，被连接的对象 | [[redis/01-入门阶段-基础教程/第00章-学前准备/0.2-安装与启动]] |
| `redis-cli` | 官方命令行客户端 | [[redis/01-入门阶段-基础教程/第00章-学前准备/0.3-redis-cli第一次连接]] |
| `PING` | 探活命令，回 `PONG` 即连通 | [[redis/01-入门阶段-基础教程/第00章-学前准备/0.2-安装与启动]] |
| `AUTH 密码` | 带密码服务的认证 | [[redis/01-入门阶段-基础教程/第00章-学前准备/0.4-看懂报错信息]] |
| `SELECT n` | 切换逻辑数据库（0~15） | [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.1-Key与Value的世界观]] |
| `DBSIZE` | 当前 db 的键数量，O(1) | 同上 |
| `INFO [段]` | 服务端状态摘要（`server`/`memory` 等） | [[redis/01-入门阶段-基础教程/第00章-学前准备/0.3-redis-cli第一次连接]] |
| `CLIENT LIST` | 列出当前连接的客户端 | 同上 |
| `CONFIG GET 参数` | 查配置项当前值 | 同上 |
| `FLUSHDB` / `FLUSHALL` | 清空当前 db / 全部 db（**危险**） | [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.1-Key与Value的世界观]] |
| `HELP 命令` / `HELP @组名` | redis-cli 内置帮助 | [[redis/01-入门阶段-基础教程/第00章-学前准备/0.3-redis-cli第一次连接]] |

## 二、通用 Key 命令 ⭐

| 词 | 一句话解释 | 出现在哪 |
|----|-----------|----------|
| `DEL key...` | 同步删除（大 key 会卡服务端） | [[redis/01-入门阶段-基础教程/第02章-过期机制与通用Key命令/2.2-通用Key命令与SCAN]] |
| `UNLINK key...` `4.0+` | 异步删除，删大 key 用它 | 同上 |
| `EXISTS key...` | 存在性；多 key 时返回存在的**个数** | 同上 |
| `TYPE key` | 查值的类型（排错第一反应） | [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.2-五种核心数据类型总览]] |
| `RENAME` / `RENAMENX` | 改名；前者**静默覆盖**目标，后者目标存在则放弃 | [[redis/01-入门阶段-基础教程/第02章-过期机制与通用Key命令/2.2-通用Key命令与SCAN]] |
| `RANDOMKEY` | 随机返回一个 key | 同上 |
| `COPY 源 目标` `6.2+` | 复制键 | 同上 |
| `OBJECT ENCODING key` | 查值的内部编码（观察用） | [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.3-String字符串]] |
| `KEYS 模式` | 全量匹配键，O(N) 阻塞，**生产禁用** | [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.1-Key与Value的世界观]] |
| `SCAN 游标 [MATCH m] [COUNT n] [TYPE t]` | 增量分批遍历，游标归 0 为结束 | [[redis/01-入门阶段-基础教程/第02章-过期机制与通用Key命令/2.2-通用Key命令与SCAN]] |
| `HSCAN` / `SSCAN` / `ZSCAN` | Hash/Set/ZSet 的分批遍历（同 SCAN 用法） | 同上 |

## 三、过期相关 ⭐

| 词 | 一句话解释 | 出现在哪 |
|----|-----------|----------|
| `EXPIRE key 秒` / `PEXPIRE 毫秒` | 设置存活时间（**重置**，不叠加） | [[redis/01-入门阶段-基础教程/第02章-过期机制与通用Key命令/2.1-Key的过期机制]] |
| `EXPIREAT key 时间戳` | 定点到某时刻过期 | 同上 |
| `TTL` / `PTTL` | 剩余时间；正数 / `-1` 永久 / `-2` 不存在 | 同上 |
| `PERSIST key` | 移除过期时间（变永久） | 同上 |
| `EXPIRE ... NX/XX/GT/LT` `7.0+` | 条件设置：无/有/更长/更短时才设 | 同上 |
| `GETEX` `6.2+` | 取值兼设过期的组合命令 | 同上 |
| 惰性删除 | 访问键时才检查并删除过期键 | 同上 |
| 定期删除 | 后台周期抽样清理过期键（`hz` 配置） | 同上 |

## 四、String ⭐

| 词 | 一句话解释 | 出现在哪 |
|----|-----------|----------|
| `SET key v [EX s|PX ms] [NX|XX] [KEEPTTL] [GET]` | 写入；`NX` 不存在才写，`KEEPTTL` `6.0+` 保留寿命，`GET` `6.2+` 返回旧值 | [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.3-String字符串]] |
| `GET key` | 读取；不存在返回 nil | 同上 |
| `SETNX` / `SETEX` / `PSETEX` | 旧式分体命令，**已废弃**，改用 SET 选项 | 同上 |
| `INCR` / `DECR` / `INCRBY` / `DECRBY` | 原子 ±1 / ±n（不存在从 0 起） | 同上 |
| `INCRBYFLOAT` | 原子加小数 | 同上 |
| `MSET` / `MGET` | 批量写 / 读（结果与请求顺序对齐） | 同上 |
| `APPEND` / `STRLEN` | 追加 / 字节长度 | 同上 |
| `GETRANGE` / `SETRANGE` | 取 / 覆写字节子串 | 同上 |
| `GETDEL` `6.2+` | 取值并删除 | 同上 |
| `int` / `embstr` / `raw` | String 的三种内部编码 | 同上 |

## 五、List ⭐

| 词 | 一句话解释 | 出现在哪 |
|----|-----------|----------|
| `LPUSH` / `RPUSH key v...` | 左端 / 右端压入 | [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.4-List列表]] |
| `LPOP` / `RPOP key [n]` | 弹出左 / 右端（`n` 为 `6.2+`） | 同上 |
| `LRANGE key 起 止` | 取闭区间片段（`0 -1` 全部） | 同上 |
| `LLEN` | 长度 | 同上 |
| `LINDEX` / `LSET` | 按下标读 / 改（O(N)） | 同上 |
| `LREM key 计数 v` | 按值删：正数从头、负数从尾、0 全删 | 同上 |
| `LTRIM key 起 止` | 只保留区间其余删除 | 同上 |
| `LMOVE 源 目标 L/R L/R` `6.2+` | 原子跨列表搬移 | 同上 |
| `BLPOP` / `BRPOP key 超时` | 阻塞版弹出（0 = 无限等） | 同上 |
| listpack / quicklist | List 的内部编码（小→紧凑块；大→链串） | 同上 |

## 六、Hash ⭐

| 词 | 一句话解释 | 出现在哪 |
|----|-----------|----------|
| `HSET key f v [f v...]` | 写一个/多个字段（覆盖旧值） | [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.5-Hash哈希]] |
| `HGET` / `HMGET` | 读单 / 批量字段 | 同上 |
| `HGETALL` | 全量读（大 key 慎用） | 同上 |
| `HDEL` / `HEXISTS` / `HLEN` | 删字段 / 字段存在？ / 字段数 | 同上 |
| `HKEYS` / `HVALS` | 所有字段名 / 所有值 | 同上 |
| `HSETNX` | 字段不存在才写 | 同上 |
| `HINCRBY` / `HINCRBYFLOAT` | 字段原子计数 | 同上 |
| `HRANDFIELD` `6.2+` | 随机取字段名 | 同上 |

## 七、Set ⭐

| 词 | 一句话解释 | 出现在哪 |
|----|-----------|----------|
| `SADD` / `SREM` | 加 / 删成员（重复自动忽略） | [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.6-Set集合]] |
| `SISMEMBER` / `SMISMEMBER` `6.2+` | 在不在（O(1)）/ 批量判断 | 同上 |
| `SCARD` | 成员数（cardinality 基数） | 同上 |
| `SMEMBERS` | 全量成员（大集合慎用） | 同上 |
| `SRANDMEMBER key [n]` | 随机取，**不删除** | 同上 |
| `SPOP key [n]` | 随机取并**删除** | 同上 |
| `SINTER` / `SUNION` / `SDIFF` | 交 / 并 / 差（`...STORE` 变体写结果） | 同上 |
| `SINTERCARD` `7.0+` | 只求交集数量 | 同上 |
| intset / hashtable | Set 的内部编码 | 同上 |

## 八、Sorted Set (ZSet) ⭐

| 词 | 一句话解释 | 出现在哪 |
|----|-----------|----------|
| `ZADD key 分数 成员...` | 加成员 / 改分数 | [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.7-SortedSet有序集合]] |
| `ZSCORE` / `ZCARD` | 查分数 / 成员数 | 同上 |
| `ZRANK` / `ZREVRANK` | 名次（升 / 降序，**0 起**） | 同上 |
| `ZRANGE key 起 止 [WITHSCORES]` | 按**名次**区间取 | 同上 |
| `ZRANGE key min max BYSCORE [REV]` `6.2+` | 按**分数**区间取；`(n` 开区间；`+inf`/`-inf` 无穷 | 同上 |
| `ZRANGEBYSCORE` | 上述的旧写法，已弃用 | 同上 |
| `ZREVRANGE` | 名次倒序取 | 同上 |
| `ZCOUNT key min max` | 分数区间内成员数 | 同上 |
| `ZINCRBY key 增量 成员` | 分数原子增减（返回 float） | 同上 |
| `ZREM` / `ZPOPMIN` / `ZPOPMAX` | 删成员 / 弹最低 / 弹最高 | 同上 |
| 跳表（skiplist） | ZSet 的有序结构（+哈希表查分数） | 同上 |

## 九、redis-py 📦

| 词 | 一句话解释 | 出现在哪 |
|----|-----------|----------|
| `redis.Redis(...)` | 客户端实例；方法名 = 命令小写 | [[redis/01-入门阶段-基础教程/第03章-Python客户端redis-py/3.1-连接与基础操作]] |
| `decode_responses=True` | 返回 str 而非 bytes（文本场景必设） | 同上 |
| `redis.from_url("redis://host:port/db")` | URL 形式建连接 | 同上 |
| `ConnectionPool` / `max_connections` | 连接池；`redis-py 8.0+` 默认上限 100 | 同上 |
| `socket_timeout` | 单命令超时；`redis-py 8.0+` 默认 5s | 同上 |
| `redis.asyncio` | asyncio 客户端（FastAPI 用）；关闭用 `aclose()` | 同上 |
| `r.delete(key)` | `DEL` 的方法名（Python 关键字例外） | 同上 |
| `r.scan_iter(match, count)` | SCAN 的生成器封装（自动接力游标） | 同上 |
| `r.pipeline()` / `.execute()` | 管道批量；默认**非事务** | [[redis/01-入门阶段-基础教程/第03章-Python客户端redis-py/3.2-操作五种数据类型]] |
| `r.hset(key, mapping={...})` | Hash 批量写的参数形态 | 同上 |
| hiredis | 可选的 C 解析器（`pip install redis[hiredis]`） | [[redis/01-入门阶段-基础教程/第03章-Python客户端redis-py/3.1-连接与基础操作]] |

## 十、报错用词 ⭐

看到 `(error)` 开头的输出，先来这里找词。

| 报错词 | 一句话含义 | 出现在哪 |
|--------|-----------|----------|
| `WRONGTYPE` | 命令与值的类型不匹配（**入门第一大报错**，先 `TYPE` 查类型） | [[redis/01-入门阶段-基础教程/第00章-学前准备/0.4-看懂报错信息]] |
| `ERR unknown command` | 命令拼错，或当前版本没有此命令 | 同上 |
| `ERR wrong number of arguments` | 参数个数不对（常为值含空格没加引号） | 同上 |
| `ERR value is not an integer or out of range` | 对非整数值做算术命令 | 同上 |
| `NOAUTH` | 服务端有密码但当前连接未认证 | 同上 |
| `OOM command not allowed` | 内存超 `maxmemory` 且策略拒绝写入（中段） | 同上 |
| `MISCONF` | RDB 快照写盘失败导致拒绝写入（中段） | 同上 |
| `Connection refused` | 连不上服务（服务没起 / 端口没映射 / 连错地址） | 同上 |
| `QUEUED` | 事务模式下命令成功入队的响应（不是错误） | [[redis/02-中段阶段-生产实践/04-事务与Lua脚本]] |
| `BUSYGROUP` | 消费者组已存在的报错（重复建组，可安全忽略） | [[redis/02-中段阶段-生产实践/02-发布订阅与Stream]] |

## 十一、术语与名词 ⭐

| 词 | 一句话解释 | 出现在哪 |
|----|-----------|----------|
| 键空间（keyspace） | 一个 db 里全部 key 的集合 | [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.1-Key与Value的世界观]] |
| 二进制安全（binary-safe） | "当字节序列处理，不假设编码" | 同上 |
| 原子性（atomic） | 命令执行中不被其他命令插队（单线程保证） | [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.3-String字符串]] |
| I/O 多路复用 | 单线程同时监听大量连接（进阶） | [[redis/01-入门阶段-基础教程/第00章-学前准备/0.1-Redis是什么]] |
| O(1) / O(N) | 操作耗时与数据量**无关** / **成正比** | [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.4-List列表]] |
| 大 key | 值/成员数巨大的键，读写删都危险 | [[redis/01-入门阶段-基础教程/第02章-过期机制与通用Key命令/2.2-通用Key命令与SCAN]] |
| 游标（cursor） | SCAN 每批返回的"接力棒"，归 0 即结束 | 同上 |
| TTL | Time To Live，key 的剩余寿命 | [[redis/01-入门阶段-基础教程/第02章-过期机制与通用Key命令/2.1-Key的过期机制]] |
| 网络往返（RTT） | 一条命令 = 客户端→服务端→回来；批量的动机 | [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.3-String字符串]] |
| 管道（pipeline） | N 条命令一次发出（网络优化，非事务） | [[redis/01-入门阶段-基础教程/第03章-Python客户端redis-py/3.2-操作五种数据类型]] |
| 热 key | 单 key QPS 占比过高的键，引发单分片倾斜 | [[redis/02-中段阶段-生产实践/10-大Key与热Key治理]] |

## 十二、中段扩展命令（生产实践专题用）

### 持久化

| 词 | 一句话解释 | 出现在哪 |
|----|-----------|----------|
| `BGSAVE` / `SAVE` | 手动触发快照（后台 / 同步阻塞，后者生产禁用） | [[redis/02-中段阶段-生产实践/01-持久化-RDB与AOF]] |
| `BGREWRITEAOF` | 手动触发 AOF 重写 | 同上 |
| `save 秒 次数`（conf） | RDB 触发规则 | 同上 |
| `appendonly` / `appendfsync`（conf） | AOF 开关 / 落盘策略（always/everysec/no） | 同上 |
| `aof-use-rdb-preamble`（conf） | 混合持久化（`4.0+`，默认开） | 同上 |
| `redis-check-rdb` / `redis-check-aof --fix` | 持久化文件校验修复工具 | 同上 |
| fork / COW（写时复制） | 快照与重写的底层机制：子进程共享内存页 | 同上 |

### Stream 与 Pub/Sub

| 词 | 一句话解释 | 出现在哪 |
|----|-----------|----------|
| `XADD k [MAXLEN ~ n] * f v...` | 追加消息，自动生成时间序 ID | [[redis/02-中段阶段-生产实践/02-发布订阅与Stream]] |
| `XGROUP CREATE k 组 ID` | 建消费者组（0 全量 / $ 只新消息） | 同上 |
| `XREADGROUP GROUP 组 消费者 COUNT n BLOCK ms STREAMS k >` | 组内取消息（> 取未投递的） | 同上 |
| `XACK` | 确认消息（移出 pending） | 同上 |
| `XPENDING` / `XAUTOCLAIM`（`6.2+`） | 查 pending / 按空闲时间接管重投 | 同上 |
| `XRANGE` / `XREVRANGE` / `XLEN` / `XTRIM` | 回放 / 长度 / 修剪 | 同上 |
| `XINFO GROUPS` / `XINFO CONSUMERS` | 组与消费者的状态 | 同上 |
| `SUBSCRIBE` / `PUBLISH` / `PSUBSCRIBE` / `UNSUBSCRIBE` | Pub/Sub 四件套（无持久化广播） | 同上 |
| pending 列表（PEL） | 已投递未确认的消息集合，不丢消息的关键 | 同上 |
| at-least-once / 幂等 | 可能重复投递 → 消费端去重设计 | 同上 |

### 事务与脚本

| 词 | 一句话解释 | 出现在哪 |
|----|-----------|----------|
| `MULTI` / `EXEC` / `DISCARD` | 事务：入队 / 执行 / 放弃 | [[redis/02-中段阶段-生产实践/04-事务与Lua脚本]] |
| `WATCH k` | 乐观锁：被改过则 EXEC 返回 nil | 同上 |
| `EXECABORT` | 入队阶段有错，整个事务放弃 | 同上 |
| `EVAL 脚本 numkeys k... a...` | 执行 Lua 脚本（服务端原子） | 同上 |
| `EVALSHA` / `SCRIPT LOAD` / `NOSCRIPT` | 按摘要执行 / 预载 / 缓存未命中 | 同上 |
| `FUNCTION LOAD` / `FCALL`（`7.0+`） | 持久化函数库 | 同上 |
| `SCRIPT KILL` | 终止只读长脚本（写过就不能杀） | 同上 |
| `redis.WatchError` / `register_script` 📦 | redis-py 的乐观锁异常 / 脚本封装 | 同上 |

### 性能

| 词 | 一句话解释 | 出现在哪 |
|----|-----------|----------|
| `SLOWLOG GET/RESET`（conf: slowlog-log-slower-than） | 慢命令日志（服务端执行耗时） | [[redis/02-中段阶段-生产实践/05-管道与性能]] |
| `LATENCY HISTORY/LATEST`（conf: latency-monitor-threshold） | 服务端停摆事件（fork 等） | 同上 |
| `MEMORY USAGE k` | 某键精确内存占用 | [[redis/02-中段阶段-生产实践/05-管道与性能]] / [[redis/02-中段阶段-生产实践/10-大Key与热Key治理]] |
| `redis-cli --bigkeys` / `--memkeys` / `--hotkeys` | 采样扫描大 key / 内存大户 / 热 key（hotkeys 需 LFU 策略） | [[redis/02-中段阶段-生产实践/05-管道与性能]] / [[redis/02-中段阶段-生产实践/10-大Key与热Key治理]] |
| `redis-benchmark` | 官方压测工具 | 同上 |
| `OBJECT FREQ`（`4.0+`，LFU 策略下） | 查键的访问频率计数 | [[redis/02-中段阶段-生产实践/07-内存淘汰与maxmemory]] |

### 淘汰与高可用

| 词 | 一句话解释 | 出现在哪 |
|----|-----------|----------|
| `maxmemory`（conf） | 内存上限；触发淘汰的开关 | [[redis/02-中段阶段-生产实践/07-内存淘汰与maxmemory]] |
| `maxmemory-policy`（conf） | 8 种淘汰策略（noeviction 默认） | 同上 |
| `maxmemory-samples` / `lfu-decay-time`（conf） | 近似 LRU 采样数 / LFU 衰减 | 同上 |
| `evicted_keys`（INFO stats） | 已被淘汰的键数——存储场景 >0 即事故 | 同上 |
| `lazyfree-lazy-*`（conf） | 过期/淘汰/隐式删除的异步释放 | 同上 |
| `REPLICAOF 主 端口`（旧名 `SLAVEOF`） | 使本库成为从库 | [[redis/02-中段阶段-生产实践/08-高可用-主从哨兵集群]] |
| `READONLY` | 从库默认只读的报错词 | 同上 |
| `WAIT n 毫秒` | 等待 n 个从库确认（缓解异步复制旧读） | 同上 |
| `repl-backlog-size` / `min-replicas-to-write`（conf） | 复制积压缓冲 / 脑裂防护 | 同上 |
| `SENTINEL monitor ... quorum` / `redis-sentinel` | 哨兵配置与进程 | 同上 |
| SDOWN / ODOWN | 主观 / 客观下线（quorum 判定） | 同上 |
| `master_for` 📦 | redis-py 经哨兵发现主库的客户端 | 同上 |
| `CLUSTER` / `MOVED` / `CROSSSLOT` / hash tag `{...}` | 集群路由 / 重定向 / 跨 slot 报错 / 同 slot 手段 | 同上 |
| 脑裂（split-brain） | 网络分区期间旧主还在收写的丢失窗口 | 同上 |
| 跳变窗口 / 看门狗（watchdog） | 锁提前失效问题 / 自动续期机制 | [[redis/02-中段阶段-生产实践/06-分布式锁]] |
| Redlock / fencing token | 多数派锁方案及其争议 / 单调编号防旧写 | 同上 |
| 布隆过滤器（Bloom Filter） | 概率判存结构：说不存在必真不存在 | [[redis/02-中段阶段-生产实践/03-缓存三大问题与缓存模式]] |
| Cache-Aside / 逻辑过期 / 延迟双删 | 旁路缓存模式 / 不阻塞的热点重建 / 清在途回填 | 同上 |

### 一致性与缓存同步

| 词 | 一句话解释 | 出现在哪 |
|----|-----------|----------|
| 先库后删 | 写序标准答案：更新数据库后删缓存 | [[redis/02-中段阶段-生产实践/09-缓存一致性-与数据库同步]] |
| 在途回填 | 读请求把旧值写回缓存的时序竞争 | 同上 |
| binlog / WAL 订阅 | 订阅数据库变更日志统一删缓存（Canal / Debezium） | 同上 |
| 读己之写 | 用户写后立刻读的部分强制走主库 | 同上 |

### 安全与监控

| 词 | 一句话解释 | 出现在哪 |
|----|-----------|----------|
| `bind 地址`（conf） | 服务监听的网卡地址（永不配公网） | [[redis/02-中段阶段-生产实践/11-安全加固与生产监控]] |
| `protected-mode`（conf） | 无密码且未显式 bind 时拒绝远程（默认开，非认证） | 同上 |
| `ACL SETUSER 用户 on >密码 ~模式 +@类 -@类` | 创建具名用户：启用/密码/key 模式/命令类别（`6.0+`） | 同上 |
| `ACL LIST/GETUSER/CAT/WHOAMI/SAVE/LOG` | 查用户 / 查命令类别 / 当前身份 / 持久化 / 违规审计 | 同上 |
| `aclfile`（conf） | ACL 用户规则的独立文件 | 同上 |
| `default` 用户 | 内置账号，加固时必须设密码或禁用 | 同上 |
| `masteruser` / `masterauth`（conf） | 从库连主库的认证 | 同上 |
| `rename-command`（conf） | 命令改名治理（遗留方式，推荐 ACL 替代） | 同上 |
| `tls-port` / `tls-cert-file`（conf） | TLS 加密端口与证书（`6.0+`） | 同上 |
| `keyspace_hits` / `keyspace_misses`（INFO stats） | 命中/未命中计数 → 命中率 = hits/(hits+misses) | 同上 |
| `rejected_connections`（INFO stats） | 超 `maxclients` 被拒的连接数，>0 即告警 | 同上 |
| `latest_fork_us`（INFO stats） | 最近一次 fork 耗时（微秒） | 同上 |
| `master_link_status`（INFO replication） | 从库连主状态，down 即告警 | 同上 |
| `redis_exporter` + Prometheus + Grafana | 常见监控采集展示栈（指标同源 INFO） | 同上 |
| `--user / --pass`（redis-cli） | ACL 具名用户的登录参数 | 同上 |

### Agent 实战与限流

| 词 | 一句话解释 | 出现在哪 |
|----|-----------|----------|
| 死信队列（DLQ） | 重试超限任务进入的告警流 | [[redis/02-中段阶段-生产实践/12-Agent场景实战-缓存会话与任务分发]] |
| `XREADGROUP ... CLAIM min-idle`（`8.4+`） | 取新消息兼接管空闲 pending 的一步式命令 | 同上 |
| 固定窗口限流 | `INCR`+`EXPIRE` 用 Lua 绑原子的计数限流 | 同上 |
| 令牌桶（token bucket） | 按速率补充令牌的平滑限流算法 | 同上 |
| `RedisSaver` / `langgraph-checkpoint-redis` 📦 | LangGraph 的 Redis 检查点器 | 同上 |
| `CacheBackedEmbeddings` 📦 | LangChain 的 embedding 缓存封装（Redis 作后备） | 同上 |

## 十三、中段/进阶术语（只报名字，防恐慌用）

| 词 | 一句话解释 | 详见 |
|----|-----------|------|
| RDB / AOF | 两种持久化（快照 / 追加日志）机制 | [[redis/学习路线图]] 中段专题 1 |
| RESP / RESP3 | Redis 的客户端-服务端通信协议 | 进阶专题 2 |
| maxmemory / noeviction | 内存上限 / 默认的"满了拒绝写"策略 | 中段专题 7 |
| 哨兵（Sentinel） | 自动故障转移的看护进程 | 中段专题 8 |
| 集群（Cluster） | 分片扩展方案（只有 db0） | 中段专题 8 |
| 主从复制（replication） | 数据复制到从库 | 中段专题 8 |
| 分布式锁 | 多实例间用 `SET NX` 互斥 | 中段专题 6 |
| 缓存穿透 / 击穿 / 雪崩 | 缓存三大经典故障模式 | 中段专题 5 |
| `MULTI` / `EXEC` | 事务命令对 | 中段专题 3 |
| Pub/Sub | 发布订阅消息模式 | 中段专题 2 |
| Stream | 带消费者组的消息队列类型 | 中段专题 2 |
| HyperLogLog / Bitmap / GEO | 省内存计数 / 位图 / 地理类型 | 中段 |
| JSON / Search / TimeSeries | 8.0+ 开源版内置的扩展类型（旧称 redis-stack） | [[redis/README]] 版本基线 |
| Valkey | 2024 年从 Redis 7.2（BSD 末版）分叉出的社区项目 | [[redis/README]] 版本基线 |
| AGPLv3 / RSALv2 / SSPLv1 | Redis 8.0 起的三选一许可证 | [[redis/README]] 版本基线 |
