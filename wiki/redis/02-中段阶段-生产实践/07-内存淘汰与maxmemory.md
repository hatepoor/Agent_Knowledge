---
tags: [redis, 中段, 内存淘汰, maxmemory, 大key, 生产]
type: guide
created: 2026-10-01
updated: 2026-10-01
---

# 内存淘汰与 maxmemory

## 学习目标

读完本篇，你应当能够：

1. 配置 `maxmemory` 与 8 种淘汰策略，并按"缓存 vs 存储"选对策略
2. 解读 `INFO memory` 的关键字段，建立内存水位的监控与报警
3. 执行一次大 key 治理（发现 → 评估 → 拆分/删除）
4. 解释"写满后为什么报 OOM / 为什么数据悄悄消失"

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[redis/01-入门阶段-基础教程/第02章-过期机制与通用Key命令/2.1-Key的过期机制]] | TTL 与过期删除（淘汰策略的前半场） |
| [[redis/02-中段阶段-生产实践/01-持久化-RDB与AOF]] | fork 与 COW 对内存余量的要求 |
| [[redis/02-中段阶段-生产实践/05-管道与性能]] | 大 key 三宗罪、`--bigkeys` 工具 |

**门槛声明**：不依赖事务/Lua、Stream 专题；强依赖持久化篇的 COW 概念。

---

## 一、机制层：过期删除 vs 内存淘汰（先分清两个概念）

| | 过期删除（复习入门 2.1） | 内存淘汰（本篇） |
|---|---|---|
| 触发 | 键**到了 TTL** | 内存**达到 maxmemory 上限**且还要写 |
| 对象 | 只针对**设了 TTL 的键** | 按策略选择牺牲者（可能包括无 TTL 的键） |
| 目的 | 键的生命周期管理 | **保命**：内存有限，必须有舍有得 |

一句话：TTL 管"该死的什么时候死"，淘汰策略管"没到 TTL 的谁先死"。

## 二、8 种淘汰策略

```
CONFIG SET maxmemory 4gb
CONFIG SET maxmemory-policy allkeys-lru
```

| 策略 | 淘汰范围 | 规则 | 适用 |
|------|----------|------|------|
| `noeviction`（默认） | 不淘汰 | 内存满后**拒绝写**（读/删可以），返回 OOM 错误 | 把 Redis 当存储、宁可拒绝写 |
| `allkeys-lru` | 所有键 | 淘汰**最久未访问** | ✅ **纯缓存首选** |
| `allkeys-lfu`（`4.0+`） | 所有键 | 淘汰**访问频率最低** | 缓存、且访问分布有明显冷热 |
| `volatile-lru` | 仅带 TTL 的键 | 最久未访问 | 混合场景：部分键不许丢 |
| `volatile-lfu`（`4.0+`） | 仅带 TTL 的键 | 频率最低 | 同上 |
| `allkeys-random` / `volatile-random` | 全部 / 带 TTL | 随机 | 几乎不用（键访问概率差不多时才等价 LRU） |
| `volatile-ttl` | 仅带 TTL 的键 | 剩余寿命短的先死 | 想尊重业务设定的优先级 |

三个必须理解的细节：

1. **volatile 系列的隐患**：如果**一个带 TTL 的键都没有**，`volatile-*` 策略下没有可淘汰对象 → 行为退化为拒绝写。混布存储（无 TTL）+ 缓存（有 TTL）时才会用 volatile 系列。
2. **LRU 是近似的**：Redis 不维护全局链表（太贵），而是**随机采样 N 个键**（`maxmemory-samples 5`）淘汰其中最久未用的。代价是"可能没淘汰掉真正最冷的键"，收益是 O(1) 开销。
3. **LFU 的计数会衰减**：`lfu-log-factor` 控制计数增长的对数斜率，`lfu-decay-time`（分钟）让久未访问的键计数自动衰减——防止历史热点永久占据内存。

### 淘汰与"数据消失"的语义差别

- 被淘汰的键**不写进 AOF、不传播 DEL 到从库**（它本来就该消失）。
- 但如果你把**无 TTL 的存储数据**放在 `allkeys-*` 策略下 → 数据会被悄悄淘汰且无法恢复。**这是把缓存策略用错在存储上，事故级错误。**

---

## 三、实操：容量规划与水位监控

### maxmemory 设多少

```
maxmemory = 物理内存 × 50% ~ 60%（单机部署时）
```

余量给谁：① 复制缓冲区与客户端缓冲区（不算在 used 里但占真内存）② fork/COW 峰值（[[redis/02-中段阶段-生产实践/01-持久化-RDB与AOF]]）③ 片化（`mem_fragmentation_ratio`）。

### INFO memory 关键字段解读

```bash
redis-cli INFO memory
```

| 字段 | 含义 | 看什么 |
|------|------|--------|
| `used_memory_human` | Redis 数据占用 | 与 maxmemory 比的水位（**>70% 报警**） |
| `used_memory_rss_human` | 操作系统视角的实际占用 | |
| `mem_fragmentation_ratio` | rss / used | **>1.5 片化严重**；<1 说明部分被换出到 swap（更糟） |
| `maxmemory_human` | 上限 | 确认配置生效 |
| `maxmemory_policy` | 当前策略 | 变更留痕 |

```bash
redis-cli INFO stats | grep -E "evicted_keys|expired_keys"
```

`evicted_keys` 持续增长 = 内存压力大、在持续淘汰——**缓存场景要评估命中率损失，存储场景 = 事故**。

### 大 key 治理流程

```
1. 发现     redis-cli --bigkeys（采样） / MEMORY USAGE 精查
2. 评估     这个 key 的读写频率、能否拆分（Hash 分桶、List 裁剪）、能否外置
3. 处理     拆分（user:1000 → user:1000:0..N 分片）| 裁剪（LTRIM 保头）| 外置（值放对象存储，Redis 存引用）
4. 删除     大 key 用 UNLINK（异步）；4.0+ 可开 lazyfree-lazy-* 让过期/淘汰/重命名也异步释放
```

相关配置：

```conf
lazyfree-lazy-eviction yes       # 淘汰时异步释放
lazyfree-lazy-expire yes         # 过期时异步释放
lazyfree-lazy-server-del yes     # 隐式删除（如 RENAME 覆盖）异步释放
```

---

## 四、生产实践：按角色配置

| 角色 | maxmemory-policy | 理由 |
|------|------------------|------|
| 纯缓存实例 | `allkeys-lru`（或访问倾斜明显用 `allkeys-lfu`） | 任何键都可牺牲，命中优先 |
| 缓存 + 少量持久状态（如会话） | `volatile-lru` + **状态键必须带 TTL** | 让缓存先死，状态留到最后 |
| 队列/存储实例（Stream、锁外的正式数据） | `noeviction` + 严格容量规划 | 数据不能悄悄消失；满了宁可拒绝写并报警 |
| Agent 的 LLM/embedding 缓存 | `allkeys-lfu` | prompt/embedding 访问冷热分明，LFU 保住高频 prompt |

**混布的纪律**：一个实例里同时放"可淘汰的缓存"与"不可淘汰的数据"是**反模式**——淘汰策略无法区分价值。标准做法：**缓存一个实例，存储一个实例**，各自配策略。做不到分实例时，存储数据全部带 TTL（长 TTL），全部走 volatile 策略。

---

## 五、陷阱

1. **不设 maxmemory** → Redis 吃到物理内存极限，被 OOM killer 杀掉（比策略拒绝写更突然、更难查）。
2. **`noeviction` + 缓存场景** → 内存满后**写全失败**，应用大面积报错。缓存实例别用默认值。
3. **`allkeys-*` + 存储数据混布** → 正式数据被悄悄淘汰，AOF 里也没有——**无声明丢失**。按第四节分实例。
4. **volatile 策略下没有任何带 TTL 的键** → 退化为拒绝写，且很难第一时间想到原因。
5. **maxmemory 顶到物理内存 90%+** → COW 期间 RSS 冲顶，触发 OOM killer 或系统级卡顿。50~60% 是纪律。
6. **只看 used_memory 不看 fragmentation** → 片化 1.8 时实际占用远超数据量；片化来源多为大量增删后的内存空洞，缓解靠重启（维护窗口）或主动整理数据结构。
7. **`maxmemory-samples` 调太大追求精确 LRU** → 淘汰判断变慢；5~10 足够。

---

## 小结

- 过期删除管 TTL，内存淘汰管 maxmemory——两个机制别混淆
- 缓存选 `allkeys-lru/lfu`；存储选 `noeviction` + 容量规划；混布用 volatile 系列 + 存储键必带 TTL
- maxmemory = 物理内存 50~60%，给 COW 与缓冲区留命
- 监控三水位：used_memory 70%、evicted_keys 增长、fragmentation 1.5
- 大 key 治理：发现 → 评估 → 拆分/外置 → UNLINK

## 练习

1. 用小 maxmemory（`CONFIG SET maxmemory 10mb`）+ 脚本灌数据，分别体验 `allkeys-lru`（旧键消失）与 `noeviction`（写入报 OOM）。
2. `INFO memory` 找出你实例的 fragmentation ratio；`CONFIG SET maxmemory-policy allkeys-lfu` 后用 `OBJECT FREQ 某key`（`4.0+`）观察访问频率计数。
3. 造一个 10 万元素的 List，分别用 `DEL` 与 `UNLINK` 删除，对比 `LATENCY LATEST` 的事件记录。

## 相关专题

- COW 与内存余量的来源：[[redis/02-中段阶段-生产实践/01-持久化-RDB与AOF]]
- 淘汰之后的多活方案：[[redis/02-中段阶段-生产实践/08-高可用-主从哨兵集群]]
