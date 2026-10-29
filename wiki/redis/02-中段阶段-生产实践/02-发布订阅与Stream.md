---
tags: [redis, 中段, stream, pubsub, 消息队列, 生产]
type: guide
created: 2026-10-01
updated: 2026-10-01
---

# 发布订阅与 Stream

## 学习目标

读完本篇，你应当能够：

1. 说清 Pub/Sub 的"fire-and-forget"本质与它不适合做任务队列的原因
2. 熟练使用 Stream 的消费者组三件套：`XADD` / `XREADGROUP` / `XACK`
3. 为"Agent 间任务分发"设计一个消息不丢的 Stream 方案（含 pending 处理）
4. 在 List 队列 / Pub/Sub / Stream 三者间做正确选型

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.4-List列表]] | List 队列模式与"弹出即丢"的局限 |
| [[redis/01-入门阶段-基础教程/第03章-Python客户端redis-py/3.1-连接与基础操作]] | 本篇 redis-py 示例的连接基础 |

**门槛声明**：不要求读过其他中段专题。

---

## 一、三种"消息"机制，先分清定位

| 机制 | 一句话 | 消息会丢吗 | 多消费者 |
|------|--------|-----------|----------|
| **Pub/Sub** | 无记忆的广播：发布时不落盘，在线的订阅者才收到 | **会**（没人听就没了，断线重连也不补） | 是（各自独立收到） |
| **List 队列** | 有记忆的排队：弹出即从队列消失 | 消费者**弹出后崩溃会丢**（弹出 ≠ 处理完） | 多消费者靠抢，**没有确认机制** |
| **Stream** | 可回放的消息日志：消息按 ID 持久化，消费者组有**确认与重投** | 设计上不丢（配合持久化与 pending 处理） | 是（组内竞争 + 各组独立） |

一句话选型：**实时通知（丢了无所谓）用 Pub/Sub；一次性简单任务用 List；不许丢的任务/事件流用 Stream。**

---

## 二、Pub/Sub：五分钟讲完

```
SUBSCRIBE channel        # 订阅（进入订阅态后只能执行订阅类命令）
PUBLISH channel message  # 发布，返回收到的订阅者数量
PSUBSCRIBE news.*        # 按模式订阅
UNSUBSCRIBE channel      # 退订
```

机制要点：消息**不持久化、不排队**——`PUBLISH` 时把消息推给当前所有在线订阅者，完事即焚。订阅者掉线期间的发布，重连后**拿不到**。

适用：配置变更通知、缓存失效广播、实时仪表盘推送。**绝不**用于"必须送达"的任务。

---

## 三、Stream 机制层：日志 + 消费者组

Stream 的心智模型 = **一条只追加的日志（每个消息有唯一 ID）+ 若干独立的消费者组（每组记一个"读到哪"的游标）**。

```
stream: jobs
   ID                消息（field-value 对）
   1759300000000-0 →  {task: summarize, doc: a.pdf}
   1759300001000-0 →  {task: embed, doc: b.txt}
        ▲                          ▲
   消费者组 A 的游标           消费者组 B 的游标
   （A、B 各自独立推进，互不影响；消息本身保留）
```

- **ID 形如 `毫秒时间戳-序号`**，时间序递增；不指定时由服务端生成（`*`）。
- **消费者组（consumer group）**：组内多个消费者**竞争**消费（每条消息只给组内一个消费者）；不同组**各自**消费全量消息（广播语义）。
- **pending 列表（PEL）**：`XREADGROUP` 取走消息后进入"已投递未确认"状态；处理成功后 `XACK` 确认移出。**没 ACK 的消息不会丢，组内可重新投递**——这是 Stream 与 List 队列的本质区别。

---

## 四、核心命令实操

```
# 生产者
XADD jobs * task summarize doc a.pdf          # 返回消息 ID
XADD jobs MAXLEN ~ 10000 * task embed doc b.txt   # 限长（~ 为近似修剪，省性能）

# 建消费者组（0 表示从头部开始；$ 表示只消费新消息）
XGROUP CREATE jobs workers 0

# 消费者 1 取消息（> 表示取从未投递过的新消息）
XREADGROUP GROUP workers consumer-1 COUNT 10 BLOCK 5000 STREAMS jobs >

# 处理成功后确认
XACK jobs workers 1759300000000-0

# 查看组状态
XINFO GROUPS jobs
XINFO CONSUMERS jobs
XPENDING jobs workers            # pending 列表：谁领了什么、多久没 ACK

# 处理失败的消息：转移给其他消费者重投（6.2+，按空闲时间批量转移）
XAUTOCLAIM jobs workers consumer-2 60000 0 COUNT 10

# 回放/范围读（不依赖组）
XRANGE jobs - + COUNT 10
XREVRANGE jobs + - COUNT 1
XLEN jobs
```

**故障恢复的标准动作**（生产必配）：消费者崩溃后，它领走的消息躺在 pending 里。另一个（或同一个）消费者用 `XAUTOCLAIM` 按"空闲超过 X 毫秒"接管重投；重投 N 次仍失败的消息进死信处理（另存一个 Stream / 记日志报警）。

`Redis 8.10` 的增强：`XREADGROUP` 支持 `MAXCOUNT`（组内总条数上限）与 `MAXSIZE`（总字节上限），一次读取的返回体可控——大消息场景防"一次读爆网络"。

---

## 五、redis-py 消费者骨架

```python
import redis, json, time

r = redis.Redis(decode_responses=True)
STREAM, GROUP, CONSUMER = "jobs", "workers", "c1"

try:
    r.xgroup_create(STREAM, GROUP, id="0")        # 首次启动建组
except redis.ResponseError as e:
    if "BUSYGROUP" not in str(e):                 # 组已存在是正常情况
        raise

while True:
    # 取新消息
    resp = r.xreadgroup(GROUP, CONSUMER, {STREAM: ">"}, count=10, block=5000)
    for stream, messages in resp or []:
        for msg_id, fields in messages:
            try:
                handle(fields)                    # 你的业务处理
                r.xack(STREAM, GROUP, msg_id)
            except Exception:
                log.error("处理失败: %s %s", msg_id, fields)   # 留在 pending，等 XAUTOCLAIM 接管
    # 周期性接管同伴的僵尸消息（生产可独立成定时任务）
    claimed, *_ = r.xautoclaim(STREAM, GROUP, CONSUMER, min_idle_time=60000, count=10)
    for msg_id, fields, _ in claimed or []:
        handle_with_retry_limit(msg_id, fields)
```

要点逐条：`xreadgroup` 的 `block=5000` 对应 `BLOCK 5000`（不空转）；`BUSYGROUP` 是重复建组的正常报错；处理异常时**不 ACK**，消息留在 pending 由 `xautoclaim` 接管——这就是"不丢"的实现路径。

---

## 六、生产实践：Agent 任务分发的完整设计

场景：多个 Agent worker 消费"文档处理任务"（你的 docling → chunking → embedding 流水线）。

| 设计点 | 决策 | 理由 |
|--------|------|------|
| 队列选型 | Stream + 消费者组 | worker 崩溃不能丢任务；需要多 worker 水平扩容 |
| 消息体 | 只放**引用**（`doc_id`、`uri`），不放文档内容 | 防大消息；内容放对象存储/库 |
| 限长 | `MAXLEN ~ 100000` | 防 Stream 无限膨胀吃内存（Stream 不设 TTL 不会自动清理） |
| 消费者命名 | 实例 ID 派生（如 `c-{pod_id}`） | pending 归属清晰，`XINFO CONSUMERS` 好排查 |
| 僵尸接管 | `XAUTOCLAIM` min_idle=60s，重投 3 次进死信 Stream | 标准 at-least-once 补偿 |
| 语义 | **at-least-once**：可能重复投递 → `handle` 必须**幂等** | 用 `doc_id + 任务哈希` 做幂等键（`SET done:{id} 1 NX EX 86400`） |
| 持久化 | 见 [[redis/02-中段阶段-生产实践/01-持久化-RDB与AOF]]：AOF everysec | 队列消息不许丢，纯 RDB 会丢分钟级任务 |

---

## 七、陷阱

1. **Pub/Sub 拿来做任务队列** → 消费者重启期间的发布全部蒸发，且无任何报错。用 Stream。
2. **Stream 不设 MAXLEN / 不清理** → 日志只增不减，内存被慢性吃光。每个 Stream 都要有长度或 TTL 策略（`XTRIM` / `XADD MAXLEN ~`）。
3. **`XGROUP CREATE ... $` 的时间陷阱**：`$` 只消费**建组之后**的新消息，建组前积压的消息全部跳过。想处理历史用 `0`。
4. **pending 只增不减**：只消费不 ACK、不 `XAUTOCLAIM`，PEL 无限堆积且消息"卡死"。监控 `XPENDING` 的总数。
5. **重复投递不是 bug**：接管重投必然导致 at-least-once。业务不幂等时，重复处理比丢失更危险——幂等设计是使用 Stream 的前置条件。
6. **大消息**：单条塞几 MB 会拖慢所有消费者。Stream 存引用，数据外置。
7. **`XREADGROUP` 不带 BLOCK 的轮询** → 空转打满 CPU 与网络。永远 `BLOCK` + count。

---

## 小结

- Pub/Sub = 无记忆广播（丢消息）；List = 弹出即丢；Stream = 日志 + 消费者组（确认、重投、可回放）
- 三件套：`XADD` 生产、`XREADGROUP` 消费、`XACK` 确认；`XAUTOCLAIM` 接管僵尸消息
- at-least-once 语义 → 消费端必须幂等
- Stream 必须限长（MAXLEN ~），监控 XPENDING

## 练习

1. 两个终端建同一个组的不同消费者，交替 `XREADGROUP`，验证"每条消息只给一个消费者"；再建第二个组，验证广播语义。
2. 消费者取消息后**不 ACK** 并退出；重新启动后用 `XAUTOCLAIM` 接管，观察 `XPENDING` 变化。
3. 给你的 Agent 项目写一个最小任务分发骨架（第五节代码 + 幂等键），单机两个进程互相消费。

## 相关专题

- 队列消息的持久化保障：[[redis/02-中段阶段-生产实践/01-持久化-RDB与AOF]]
- 多消费者竞争的正确性工具（事务/脚本）：[[redis/02-中段阶段-生产实践/04-事务与Lua脚本]]
