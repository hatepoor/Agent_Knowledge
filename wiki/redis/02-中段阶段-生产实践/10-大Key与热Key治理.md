---
tags: [redis, 中段, 大key, 热key, 性能, 生产]
type: guide
created: 2026-10-06
updated: 2026-10-06
---

# 大 Key 与热 Key 治理

## 学习目标

读完本篇，你应当能够：

1. 给出大 key / 热 key 的判定标准，并量化它们各自的危害（网络、阻塞、倾斜）
2. 用在线与离线两套工具体系定位大 key 和热 key，说清每个工具的原理与局限
3. 按标准 SOP 完成大 key 治理（拆分 / 压缩 / 外置 / 安全删除）
4. 为热 key 设计本地缓存或打散方案，并写出应急预案

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[redis/02-中段阶段-生产实践/05-管道与性能]] | RTT 数学、`--bigkeys`/`SLOWLOG` 工具初见 |
| [[redis/02-中段阶段-生产实践/07-内存淘汰与maxmemory]] | lazyfree 异步释放、淘汰与内存水位 |
| [[redis/01-入门阶段-基础教程/第02章-过期机制与通用Key命令/2.2-通用Key命令与SCAN]] | SCAN 语义与 `UNLINK` 异步删除 |
| [[redis/01-入门阶段-基础教程/第00章-学前准备/0.1-Redis是什么]] | 单线程模型——所有"阻塞"分析的根 |

**门槛声明**：本篇是 05（性能排查）与 07（内存淘汰）中"大 key"内容的**专篇深化**——工具用法在 05 已初见，本篇补齐判定标准、离线工具、治理 SOP 与热 key 完整方案。先读 05、07 再读本篇，收益最大。

---

## 一、机制层：什么是大 key / 热 key，危害有多大

两个词都没有官方红线，工程界通用判定标准（**经验值，供容量规划参考**）：

| 类别 | 判定（满足其一即关注） | 危险线 |
|------|----------------------|--------|
| 大 key（按大小） | String value > 10KB | > 1MB 必须治理 |
| 大 key（按规模） | 集合类型元素数 > 5000 | > 5 万必须治理 |
| 热 key | 单 key QPS 达到实例总 QPS 的 **5%+**，或绝对值数千 | 单 key 数万 QPS |

### 大 key 的危害，逐条量化

1. **网络放大**：100KB 的 value × 1000 QPS = **100MB/s** 出口流量——千兆网卡直接打满，且挤占其他 key 的带宽。
2. **删除阻塞**：`DEL` 是同步释放内存。百万元素的 List，`DEL` 主线程阻塞量级可达**秒级**——期间所有命令排队，等于整个实例停摆。这是大 key 最致命的危害（解法 `UNLINK`，见第三节）。
3. **读写阻塞**：单线程模型下，服务端处理一个 1MB value 的序列化/拷贝期间，其他命令全部等待。`HGETALL` 一个 10 万元素的 Hash 同理。
4. **主从同步放大**：全量同步时快照传输、增量同步时命令传播，大 key 都在放大流量；从库重放 `DEL`/`UNLINK` 同样可能阻塞。
5. **过期雪崩**：大 key 集中在同一时刻过期，定期删除的清理耗时集中爆发（`LATENCY` 的 `expire-cycle` 事件，复习 05）。

### 热 key 的危害

热 key 是**单点流量倾斜**问题：单 key QPS 过高 → 单线程服务端 CPU 全在伺候它 → 其他所有命令延迟上升；集群模式下该 key 所在分片 CPU 打满而其他分片空闲（**数据倾斜**），扩容也无法缓解——因为 hash 不变，它永远落在同一分片。

**与击穿的区别**（复习 03）：击穿是"热点 key **过期瞬间**"的瞬时回源问题，解法是互斥重建；本篇热 key 是"持续高频访问"的**稳态**问题，解法是分流。两者常相伴：先有热 key，key 一过期就击穿。

---

## 二、发现工具全景

### 在线工具（连着实例跑）

```bash
# 1. 各类型的"最大 key"（SCAN 采样，对服务影响低）
redis-cli --bigkeys
#    原理：SCAN 采样遍历，对每种类型记录元素数最大的 key
#    局限：① 只报每类型的"第一名"，报不出"前 100 个大户"
#          ② 采样（COUNT 默认 100），极小概率漏 key
#          ③ string 报字节长度，集合报元素数——不等于内存占用

# 2. 按内存排序找大户
redis-cli --memkeys        # 元素数 ≠ 内存占用：1 个字段 10MB 的 Hash 会被 --bigkeys 低估

# 3. 热 key（要求 maxmemory-policy 为 LFU 系）
redis-cli --hotkeys
#    原理：采样后按 OBJECT FREQ 排序
#    ★ LRU 策略下不可用（LRU 不记录访问频率）——想用先切 allkeys-lfu

# 4. 精查单个 key 的内存
MEMORY USAGE 某key                  # 字节；加 SAMPLES 0 精确统计集合
OBJECT ENCODING 某key               # 内部编码（判断是否走了紧凑编码）
OBJECT FREQ 某key                   # LFU 策略下的访问频率计数
```

SCAN + `MEMORY USAGE` 组合找"前 N 大户"（`--bigkeys` 的局限补偿）：

```python
import redis

r = redis.Redis(decode_responses=True)
top = []
for key in r.scan_iter(count=1000):
    mem = r.memory_usage(key) or 0
    top.append((mem, key))

top.sort(reverse=True)
for mem, key in top[:20]:
    print(f"{mem/1024:8.1f} KB  {key}")
# 注意：每个 key 一次 MEMORY USAGE 往返，大实例用 pipeline 批量（复习 05），
# 且在低峰期跑——全量遍历本身有成本
```

### 离线工具（分析 RDB 文件，零线上影响）

```bash
# 对 RDB 快照离线分析（rdb-tools / redis-rdb-cli 等社区工具）
rdb --command memory dump.rdb --bytes 10240 -f memory.csv
# 输出每个 key 的内存明细 → 用 Excel/pandas 聚合出 TOP 大户与类型分布
```

离线分析的独特价值：**全量、精确、不碰线上**。生产 SOP 通常是"低峰 BGSAVE 拿快照 → 离线出 TOP 榜 → 在线精查确认"。

### 客户端与代理统计

业务代码里对 `GET`/`SET` 埋点计数（key 粒度 QPS），或在代理层（如 Codis/Envoy 类）看访问分布。适合"已经怀疑某类 key 热，要确认"的场景，不适合全量普查。

---

## 三、大 key 治理 SOP

```
1. 发现     在线普查（低峰）+ RDB 离线分析 → TOP 榜
2. 评估     MEMORY USAGE 精查；这个 key 读写 QPS 多少？还能长多大？
            value 为什么这么大？（设计问题还是数据堆积？）
3. 治理     按下表选模式：拆分 / 裁剪 / 压缩 / 外置
4. 删除     UNLINK 或游标分批删；开 lazyfree 系配置（复习 07）
5. 防复发   代码评审卡"无界集合"；容量告警（07 水位）
```

### 治理模式选型

| 模式 | 做法 | 适用 |
|------|------|------|
| **Hash 分桶拆分** | `user:{id}:friends`（10 万元素）→ 桶 `user:{id}:friends:{id%64}`，每桶 ~1500 | 集合类型元素多；读时按桶定位 |
| **时间分片** | 消息历史 `chat:{id}:2026-10`（按月一个 key），旧数据归档 | 随时间无限增长的数据（会话、日志） |
| **LTRIM 裁剪** | List 只保留最近 N 条：每次 `LPUSH` 后 `LTRIM 0 999` | 只关心"最近"的列表（动态流） |
| **value 压缩** | 序列化后 zstd/msgpack 压缩再存 | String 大对象（JSON 配置、模型输出） |
| **外置引用** | value 放对象存储/S3，Redis 只存 URL + 元数据 | 超大且低频改的数据（文件、报告） |

分桶代码骨架（拆分 + 读取）：

```python
BUCKETS = 64

def friend_bucket(uid: int, fid: int) -> str:
    return f"user:{uid}:friends:{fid % BUCKETS}"

def add_friend(uid: int, fid: int):
    r.sadd(friend_bucket(uid, fid), fid)          # 写散到 64 个小 Set

def is_friend(uid: int, fid: int) -> bool:
    return r.sismember(friend_bucket(uid, fid), fid)   # 读只碰一个桶，O(1)

def all_friends(uid: int):                        # 全量读才碰全部桶
    keys = [friend_bucket(uid, i) for i in range(BUCKETS)]
    return set().union(*(r.smembers(k) for k in keys))   # pipeline 批量更优（复习 05）
```

### 游标分批删除（不能 UNLINK 的场合 / 旧版本）

```python
def delete_big_hash(key: str, batch=500):
    cursor = 0
    while True:
        cursor, fields = r.hscan(key, cursor, count=batch)
        if fields:
            r.hdel(key, *fields.keys())
        if cursor == 0:
            break
    r.delete(key)          # 此时已空，DEL 秒回
```

---

## 四、热 key 应对

### 方案 1：本地二级缓存（首选，简单有效）

进程内缓存挡掉绝大部分读：

```python
from cachetools import TTLCache

hot_cache = TTLCache(maxsize=1000, ttl=5)   # 进程内，5 秒过期

def get_hot(key: str):
    if key in hot_cache:
        return hot_cache[key]
    val = r.get(key)                        # 未命中才到 Redis
    if val is not None:
        hot_cache[key] = val                # 5 秒内该 key 的流量全在进程内
    return val
```

- 效果：单实例内热 key 的 Redis QPS 从 N 降到 N/(5s 内请求数)，**N 个应用实例合计削减 95%+ 很常见**。
- 代价：**最多 5 秒的旧数据**（TTL 就是新鲜度上限）——放热 key 缓存前先回答"业务接受多久旧值"。
- 多级缓存整体结构（本地 → Redis → 库）复习 03 的雪崩分层。

### 方案 2：key 打散（写扩散 / 读随机）

本地缓存装不下（value 大）或多实例本地命中率低时，把一个 key 变 N 个：

```python
N = 16
def write_hot(key: str, val: str, ttl=60):
    pipe = r.pipeline(transaction=False)
    for i in range(N):                       # 写：N 个副本
        pipe.set(f"{key}:copy{i}", val, ex=ttl)
    pipe.execute()

def read_hot(key: str):
    i = random.randrange(N)                  # 读：随机打到一个副本
    return r.get(f"{key}:copy{i}")
```

读流量被均匀分到 16 个 key（集群下可跨分片）；写成本 ×16（val 大时慎用，适合中小 value 的超高读频场景）。

### 方案 3：读写分离

热 key 是**读**型时，读流量引到从库（复习 08 的 `READONLY` 与 replica 架构）。局限：集群下单分片读写都重，扩从库只分摊读；写型热 key（如计数器）无解，只能拆（计数器场景用 `INCRBY` 分片聚合，或直接上本地聚合定期回写）。

### 热 key 应急预案（生产时刻的动作清单）

```
发现 CPU 单分片打满 / 某分片 ops 异常：
1. redis-cli --hotkeys / 客户端埋点 → 锁定 key
2. 判断读写型：
   读型 → 立即上本地缓存（部署快）；或读从库分流
   写型 → 评估能否降级（采样统计、异步聚合）
3. 兜底：限流该业务入口（应用层），保住 Redis 不被打死
4. 事后：把该 key 纳入本地缓存常驻 + 打散方案，写进复盘
```

---

## 五、陷阱

1. **`--hotkeys` 在 LRU 策略下静默不可用**：它依赖 LFU 频率计数，`allkeys-lru` 下跑不出结果。用 LFU 策略的实例才有热 key 探测能力（策略选型复习 07）。
2. **本地缓存没有失效联动**：Redis 侧删了缓存（一致性方案复习 09），进程内 TTLCache 还留着旧值最多一个 TTL 周期——热 key 本地缓存必须配短 TTL（秒级），并把"新鲜度上限"跟业务确认过。
3. **分桶后 pipeline 跨 slot**：集群模式下 `{id}` 不在 hash tag 里的话，64 个桶散在多分片，pipeline 被客户端自动拆分、收益打折（复习 05 陷阱 3）。分桶 key 设计时就该把 `hash tag` 定好。
4. **`SCAN` + `MEMORY USAGE` 在业务高峰跑全量**：全量遍历 + 每 key 一次往返，本身就是一次"自造慢查询"。低峰期、pipeline 批量化、加进度日志。
5. **治理只删不防**：删掉一个 100 万元素的 List，三个月后又长出一个——根因是代码里"无界集合"（消息历史、任务列表只进不出）。治理的最后一环是**代码评审规则**：所有集合 key 必须有上界策略（`LTRIM`/`XTRIM`/分片/归档）。
6. **把大 key 迁去另一个实例就当治好了**：网络放大、主从同步放大是**跨实例**的，迁移只是挪了位置。该拆还是拆。
7. **`UNLINK` 万能论**：异步释放靠后台线程排队，瞬时大量 `UNLINK` 仍会耗尽释放队列（`lazyfree` 相关监控 `lazyfree_pending_objects` 增长即信号）。批量删除场景用游标分批删更稳。

---

## 小结

- 大 key 看大小与元素数（10KB/5000 起关注），热 key 看占比（5%+）；大 key 伤网络与阻塞，热 key 伤倾斜
- 发现三路：在线（`--bigkeys`/`--memkeys`/`--hotkeys`，各有局限）、离线（RDB 分析，零影响）、客户端/代理埋点
- 大 key 治理五步：发现 → 评估 → 拆分/裁剪/压缩/外置 → `UNLINK`/分批删 → 防复发（无界集合评审）
- 热 key 三招：本地二级缓存（首选，秒级 TTL）、key 打散（读随机写广播）、读写分离；写型热 key 靠拆
- 防 > 治：所有集合 key 设计时就要回答"它最大会长到多大"

## 练习

1. 造一个大 key（`RPUSH` 50 万元素），用 `--bigkeys` 找到它；分别用 `DEL` 与 `UNLINK` 删除，从 `LATENCY LATEST` 对比阻塞事件（07 练习 3 的深化：这次看清楚差多少毫秒）。
2. 给你的 `maxmemory-policy` 临时切 `allkeys-lfu`，写几个 key 不同频率访问，用 `--hotkeys` 与 `OBJECT FREQ` 验证频率计数；确认后切回。
3. 实现 `friend_bucket` 分桶方案，压测单 key 版与分桶版的 `SADD` + `SISMEMBER` 混合负载，观察单分片实例的 ops 与延迟差异。
4. 审查你现有项目：找出所有"只进不出"的集合 key，为每个补上界策略（LTRIM / 按时间分片 / 归档）。

## 相关专题

- 工具初见与 RTT 数学：[[redis/02-中段阶段-生产实践/05-管道与性能]]
- lazyfree 与内存水位：[[redis/02-中段阶段-生产实践/07-内存淘汰与maxmemory]]
- 热 key 过期瞬间的击穿防护：[[redis/02-中段阶段-生产实践/03-缓存三大问题与缓存模式]]
- 集群分片与 hash tag：[[redis/02-中段阶段-生产实践/08-高可用-主从哨兵集群]]
- 无界集合的典型场景治理：[[redis/02-中段阶段-生产实践/12-Agent场景实战-缓存会话与任务分发]]
