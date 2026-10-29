---
tags: [redis, 中段, 持久化, RDB, AOF, 生产]
type: guide
created: 2026-10-01
updated: 2026-10-01
---

# 持久化：RDB 与 AOF

## 学习目标

读完本篇，你应当能够：

1. 讲清 RDB（快照）与 AOF（追加日志）两种机制的工作原理与取舍
2. 为一个真实服务选择持久化方案并写出对应配置（含混合持久化）
3. 完成"备份 → 模拟损坏 → 恢复"的完整演练
4. 解释"为什么重启后丢了 X 分钟数据"并给出补救配置

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[redis/01-入门阶段-基础教程/第02章-过期机制与通用Key命令/2.1-Key的过期机制]] | 过期键对持久化文件的影响 |
| [[redis/01-入门阶段-基础教程/第00章-学前准备/0.2-安装与启动]] | 知道数据卷 `/data` 与默认快照行为 |

**门槛声明**：本篇是中段起点，不要求读过其他中段专题；但假设你已学完入门阶段。

---

## 一、为什么需要持久化（生产视角）

入门阶段反复强调：Redis 数据在内存里。**进程一重启，内存清零。** 持久化回答的是一个问题：

> 断电 / 崩溃 / 重启之后，数据恢复到"多久之前"？

这个"丢失窗口"是持久化方案的核心指标。三种选择对应三种丢失窗口：

| 方案 | 丢失窗口 | 代价 |
|------|----------|------|
| RDB（定时快照） | 两次快照之间（默认配置可能 **数分钟**） | 恢复快、文件紧凑、对主线程影响小 |
| AOF everysec | **≤ 1 秒** | 文件大、恢复慢一点、持续磁盘写 |
| AOF always | **≈ 0**（每条命令落盘） | 吞吐显著下降，一般不用 |

---

## 二、机制层：两种机制怎么工作

### RDB：fork 出子进程拍快照

```
触发 save 规则 → 主进程 fork() 出子进程 → 子进程把全量数据写入临时 RDB 文件 → 原子替换旧文件
```

三个关键点：

1. **fork + 写时复制（COW，Copy-On-Write）**：fork 瞬间父子进程共享同一份物理内存；此后主进程的写操作触发内核为**被改动的内存页**复制副本。所以快照是"fork 那一瞬间的完整视图"，且主线程几乎不参与写盘。
2. **save 规则**是触发条件（满足任意一条即触发）：`save 3600 1` = 1 小时内至少 1 次修改。
3. `BGSAVE` 是手动触发；`SAVE`（同步，阻塞主线程）生产禁用。

**RDB 的代价点**：fork 本身要复制页表，**实例内存越大 fork 越慢**（10GB 级实例 fork 可达数十至数百毫秒——期间主线程停摆）；COW 期间写入越频繁，内存膨胀越多（极端情况接近翻倍）。这是"大实例 + 高写入"场景的经典故障来源。

### AOF：把每条写命令追加进日志

```
命令执行成功 → 追加进 AOF 缓冲区 → 按 appendfsync 策略写盘
```

| appendfsync | 行为 | 丢失窗口 |
|-------------|------|----------|
| `always` | 每条命令都 fsync | ≈0，吞吐最低 |
| `everysec`（默认/推荐） | 后台线程每秒 fsync 一次 | ≤1s |
| `no` | 交给操作系统（约 30s） | 不可控 |

三个配套机制：

1. **AOF 重写（rewrite）**：日志会无限膨胀，所以 Redis 会按文件增长比例（`auto-aof-rewrite-percentage 100` = 比上次翻倍）或大小（`auto-aof-rewrite-min-size 64mb`）触发重写——fork 子进程按**当前内存数据的最小等价命令集**生成新文件（与 RDB 一样走 fork + COW，同样的 fork 卡顿风险）。
2. **混合持久化（`aof-use-rdb-preamble yes`，`Redis 4.0+`，默认开启）**：重写时前半段存 RDB 格式全量、后半段追加 AOF 增量。恢复快 + 丢失窗口小，**8.x 的默认推荐形态**。
3. **损坏处理**：崩溃可能让 AOF 末尾残缺。`aof-load-truncated yes`（默认）会截掉残尾继续恢复；彻底损坏用 `redis-check-aof --fix` 修复。RDB 对应工具是 `redis-check-rdb`。

### 过期键在持久化文件里的处理（衔接入门）

- RDB：快照时**已过期的键不会写入**文件。
- AOF：过期删除发生时追加一条显式 `DEL`（惰性/定期删除都会记录）。
- 主从复制时从库不自主删除过期键，等主库的 DEL 同步——所以从库可能读到刚过期但还没被删的键（读时校验 TTL，`Redis 7.4+` 改进了行为）。

---

## 三、实操：配置一个生产形态的持久化

推荐配置（写在 `redis.conf`，或 `CONFIG SET` 在线改）：

```conf
# ---- RDB：作为兜底快照 ----
save 3600 1          # 1 小时内 1 次修改
save 300 100         # 或 5 分钟内 100 次
save 60 10000        # 或 1 分钟内 1 万次
dbfilename dump.rdb

# ---- AOF：主恢复来源 ----
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes          # 混合持久化（4.0+ 默认）
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes

# ---- 通用 ----
dir /data              # 所有持久化文件的目录（Docker 里就是挂卷的 /data）
```

验证与演练（生产上线前必做一次）：

```bash
# 1. 写入测试数据
redis-cli SET persist:test hello

# 2. 触发一次完整保存
redis-cli BGSAVE
redis-cli BGREWRITEAOF      # 手动触发 AOF 重写

# 3. 记录文件
ls -lh /data/               # dump.rdb + appendonly.aof（8.x AOF 是目录 appendonlydir/）

# 4. 杀掉 Redis（模拟崩溃）
docker stop my-redis && docker start my-redis

# 5. 验证恢复
redis-cli GET persist:test   # "hello"
```

备份策略（最小可用版）：

1. **RDB 文件异地备份**：定时把 `/data` 下的 RDB/AOF 同步到对象存储 / 另一台机器（`redis-cli BGSAVE` 后 cp，或直接 cp AOF）。
2. **不要只依赖本机卷**：容器/宿主机磁盘损坏时挂卷救不了你。
3. **恢复演练**：定期在另一台机器用备份文件起一个空实例验证能恢复——没演练过的备份等于没有备份。

---

## 四、方案选型决策表

| 你的场景 | 推荐方案 | 理由 |
|----------|----------|------|
| **纯缓存**（丢了从库里/源头重建） | `save ""`（关掉）或保留默认 RDB | 持久化是纯开销；重启后预热重建 |
| **重要状态**（会话、计数、队列） | AOF everysec + 混合持久化 | 丢 1 秒可接受，恢复快 |
| **把 Redis 当主存储** | AOF everysec + 每日 RDB 异地备份 + 主从（见 [[redis/02-中段阶段-生产实践/08-高可用-主从哨兵集群]]） | 仍建议最终以 PostgreSQL 等为准，Redis 不做唯一事实来源 |

---

## 五、陷阱

1. **fork 卡顿**：大内存（>10GB）+ 高写入实例的 BGSAVE/BGREWRITEAOF 期间出现毫秒到秒级延迟尖刺。缓解：拆小实例、错峰重写、用 `INFO stats` 的 `latest_fork_usec` 观测。
2. **COW 内存膨胀**：快照/重写期间写压力大 → 内存可能逼近两倍。`maxmemory` 要给 COW 留余量（设到物理内存的 50~60% 以下）。
3. **"everysec 丢 1 秒"是上限不是承诺**：磁盘 IO 卡住时可能丢更多（AOF 写盘阻塞会拖慢所有命令——`INFO persistence` 看 `aof_delayed_fsync` 是否增长）。
4. **`CONFIG SET appendonly yes` 在线开启**会触发一次全量重写，大实例上等于手动 BGREWRITEAOF——业务低峰操作。
5. **挂了卷但不等于有备份**：卷解决的是"容器重建数据还在"，不解决"磁盘坏了数据没了"。
6. **8.x 的 AOF 目录结构**：`appendonlydir/` 下分 `appendonly.aof.*`（incr/base/manifest 多文件），与 6.x 的单文件布局不同——老恢复脚本要适配。

---

## 小结

- 丢失窗口三角：RDB（分钟级）/ AOF everysec（秒级）/ always（≈0 但慢）
- 机制都是 fork + COW：快照走子进程，代价是 fork 卡顿 + COW 内存
- 生产推荐：AOF everysec + 混合持久化 + RDB 兜底 + 异地备份 + 恢复演练
- 纯缓存可以关持久化——先想清楚"数据丢了从哪重建"

## 练习

1. 本地起一个带卷的 Redis，按第三节配置 AOF；写数据 → `docker stop/start` → 验证恢复；对比 `appendfsync` 改为 `no` 时的丢失差异。
2. `INFO persistence` 逐字段解读：`rdb_last_save_time`、`rdb_changes_since_last_save`、`aof_delayed_fsync`。
3. （思考）你的 Agent 会话存储该用哪种持久化？丢失 1 秒会话数据可接受吗？

## 相关专题

- 数据多副本，不只靠磁盘：[[redis/02-中段阶段-生产实践/08-高可用-主从哨兵集群]]
- 内存上限与逐出：[[redis/02-中段阶段-生产实践/07-内存淘汰与maxmemory]]
