---
tags: [interview, langgraph, checkpointer, 持久化, 面试]
type: guide
created: 2026-10-18
updated: 2026-10-18
---

# 持久化与 Checkpointer

> **适用场景**：LangGraph 面试**第二高频**。答「就是存个对话历史」会直接暴露没上过生产。
> **主线**：Checkpointer 存的是**每一步的完整状态快照**，不是聊天记录。

---

## Q1. Checkpointer 是什么？它解决什么问题？

**答：**

**Checkpointer 是 LangGraph 的持久化机制：在执行的每个 superstep 结束时，把当前完整状态写成一个带 id 的快照（checkpoint），按 `thread_id` 串成一条时间线。**

它支撑四件事：

| 能力 | 说明 |
|------|------|
| **多轮对话** | 同一 `thread_id` 再次调用时，从最后一个 checkpoint 恢复状态 |
| **断点恢复** | 进程崩溃/重启后，从最近快照继续，不用重跑全流程 |
| **人机协同** | `interrupt()` 依赖 checkpoint 才能「暂停后退出进程、之后接着跑」 |
| **时间旅行** | 读取历史快照，回放或从某个历史点分叉 |

⚠️ **它不是「消息历史存储」**。消息只是 State 里的一个字段；Checkpointer 存的是**整个 State + 执行元数据**（哪些节点跑过、下一步是谁、待写入是什么）。

**追问：不挂 Checkpointer 会怎样？**
→ 图照样能跑，但每次 `invoke` 都是**全新执行**：没有多轮记忆、不能中断恢复、不能用 `thread_id` 隔离会话。适合无状态的一次性任务。

---

## Q2. 一个 checkpoint 里到底存了什么？保存粒度是节点级还是 superstep 级？

**答：**

**粒度是 superstep 级**，不是节点级。

一个 superstep 内如果有 3 个并行节点，它们执行完后**只产生一个 checkpoint**，包含这三个节点的合并结果。所以「10 个节点串行执行」大约产生 10 个 checkpoint；「1 个节点扇出 10 个并行节点」仍然只多 1 个。

一个 checkpoint 主要包含：

| 内容 | 作用 |
|------|------|
| `channel_values` | 各状态字段的当前值（真正的业务数据） |
| `channel_versions` | 每个 channel 的版本号，用于判断增量 |
| `versions_seen` | 各节点已见过的版本，决定「谁被激活」 |
| `pending_writes` | 本轮已产生但尚未提交的写入 |
| `checkpoint_id` | 单调递增的标识，时间旅行靠它定位 |
| `metadata` | `step`（第几轮）、`source`（input/loop/update）、父 checkpoint |

**追问：为什么要存 `versions_seen`？**
→ 它是**调度器的记忆**。每轮通过比较「channel 当前版本」和「节点已见版本」来判断该节点是否需要被激活。没有它，图无法知道哪些分支已经跑过、哪些还没跑。

---

## Q3. `thread_id` 的作用是什么？多租户怎么设计？

**答：**

`thread_id` 是**状态时间线的逻辑主键**：同一 `thread_id` 共享历史，不同 `thread_id` 完全隔离。

```python
config = {"configurable": {"thread_id": "user-42:session-7"}}
graph.invoke({"messages": [...]}, config)
```

多租户设计要点：

| 要点 | 做法 |
|------|------|
| **命名可解析** | `f"{tenant_id}:{user_id}:{session_id}"`，便于排查和按前缀清理 |
| **不要用自增/可猜测 id** | 防越权：拿到别人的 thread_id 就能读别人的状态 |
| **长度控制** | 部分部署环境/索引对键长度有限制，建议控制在 255 字符内，别把整段 JSON 塞进去 |
| **租户隔离** | 多租户可用独立 schema（`PostgresSaver(..., schema_name=...)`）或独立库 |
| **生命周期** | 会话结束后按 TTL 清理，避免 checkpoint 无限增长 |

⚠️ **安全提醒**：`thread_id` 是**授权边界**，不是普通参数。API 层必须校验「当前用户是否有权访问这个 thread」，不能直接透传前端传来的值。

**追问：子图的 checkpoint 怎么隔离？**
→ 用 `checkpoint_ns`（命名空间）。父图和子图在同一个 `thread_id` 下，用不同的 ns 区分层级。恢复时框架会按 ns 定位到具体子图。

---

## Q4. InMemorySaver、SqliteSaver、PostgresSaver 怎么选？

**答：**

| 后端 | 存储 | 适用 | 关键限制 |
|------|------|------|---------|
| `InMemorySaver` | 进程内存 | 单测、Notebook、演示 | 进程退出即丢，不能多进程共享 |
| `SqliteSaver` | 本地文件 | 单机桌面应用、开发环境 | 文件锁，多进程/多实例并发写会冲突 |
| `PostgresSaver` | PostgreSQL | **生产默认** | 需要建表（`setup()`），运维成本更高 |
| `AsyncPostgresSaver` | PostgreSQL | 异步服务（FastAPI） | 同上，配连接池使用 |

选型判断：

- **要不要跨进程/跨实例共享？** 要 → PG；
- **会不会并发写同一 thread？** 会 → PG（SQLite 的文件锁扛不住）；
- **只是跑个 Demo？** → InMemory 足够，别提前上 PG。

**追问：为什么生产不用 SQLite？**
→ 三个原因：① **写并发**——SQLite 是库级/文件级锁，多 worker 同时写会 `database is locked`；② **多实例**——容器化部署后每个副本一个本地文件，状态不共享；③ **运维**——备份、监控、扩容都没有成熟方案。

**追问：Redis 能做 Checkpointer 吗？**
→ 社区有实现，但**不建议做唯一存储**：Redis 是内存型，持久化是异步的，崩溃可能丢最近写入；且历史快照会迅速吃满内存。常见做法是 **Redis 做热会话缓存、PG 做权威存储**，但两套存储的一致性要自己保证，复杂度不低。

---

## Q5. `PostgresSaver` 怎么初始化？`setup()` 为什么不能省？

**答：**

```python
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://user:pass@localhost:5432/agent?sslmode=disable"

with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpointer.setup()                     # 建表/建索引，幂等
    graph = builder.compile(checkpointer=checkpointer)
    graph.invoke(inputs, {"configurable": {"thread_id": "t1"}})
```

`setup()` 做的是 **DDL**：创建 `checkpoints`、`checkpoint_writes`、`checkpoint_blobs` 等表及索引。

为什么不能省：

1. **它不会自动执行**。不调用就等着运行时报「relation does not exist」。
2. **它是幂等的**，可以在应用启动时无脑调用一次。
3. **升级版本后可能需要重跑**，因为新版本可能加列/加表。

**生产建议**：

- 用**连接池**（`AsyncPostgresSaver` + `psycopg_pool`），不要每次请求新建连接；
- `setup()` 放在**启动钩子**里执行一次，不要放在请求路径上；
- 数据库账号要能建表，或者先用迁移脚本建好再收回 DDL 权限。

**追问：多副本同时启动，`setup()` 会冲突吗？**
→ 它是幂等的（`CREATE TABLE IF NOT EXISTS` 之类），一般不会。但多副本同时执行 DDL 仍可能撞锁，稳妥做法是**由迁移任务单独执行**，应用只做健康检查。

---

## Q6. 「时间旅行」怎么实现？`get_state_history` 和 `update_state` 分别有什么用？

**答：**

**读历史：**

```python
for snapshot in graph.get_state_history(config):
    print(snapshot.values, snapshot.next, snapshot.config["configurable"]["checkpoint_id"])
```

返回按时间倒序的 `StateSnapshot`，每个带自己的 `checkpoint_id` 和 `next`（下一步待执行的节点）。

**回到某个历史点：**

```python
old_config = {"configurable": {"thread_id": "t1", "checkpoint_id": "..."}}
graph.invoke(None, old_config)     # 从该快照继续，产生新的分支
```

**手动改状态：**

```python
graph.update_state(config, {"status": "approved"}, as_node="review")
```

`update_state` 的用途：

| 用途 | 说明 |
|------|------|
| 人工修正 | 审批时改字段，再继续跑 |
| 注入数据 | 外部系统回填结果 |
| 模拟分叉 | 从某个点改一个值，看后续走向 |
| 修复脏状态 | 生产事故后手工纠正 |

⚠️ `as_node` 很重要：它决定「这次更新算哪个节点写的」，进而影响下一轮哪些节点被激活。写错会导致该跑的不跑。

**追问：时间旅行会覆盖原来的历史吗？**
→ 不会。它是**从旧 checkpoint 分叉出新分支**，原分支保留。这也是排查线上问题时最有效的手段：拿到出问题的 thread，回到出错前一步，改一个变量重放。

---

## Q7. Checkpointer 和 Store 有什么区别？各自存什么？

**答：**

| 维度 | Checkpointer | Store |
|------|-------------|-------|
| **作用域** | 单个 `thread_id` 内 | **跨 thread** 全局 |
| **存什么** | 图执行的完整状态快照 | 键值对（长期记忆、用户画像、事实） |
| **生命周期** | 会话级，可清理 | 长期 |
| **访问方式** | 框架自动读写 | 节点里显式 `store.get/put/search` |
| **典型用途** | 多轮对话、恢复、HITL | 跨会话记忆、共享知识、用户偏好 |

```python
def remember(state, config, store):
    store.put(("users",), "u-42", {"name": "张三"})     # 长期
    return {}
```

**一句话**：**Checkpointer 管「这次对话走到哪了」，Store 管「这个用户是谁」。**

**追问：为什么不把长期记忆也塞进 State？**
→ 会跟着每个 checkpoint 重复存储，越滚越大；而且它天然是跨会话的，放在 thread 级快照里既浪费又难查。**记忆的粒度应该和它的生命周期匹配。**

---

## Q8. 生产跑久了 checkpoint 表暴涨，怎么治理？

**答：**

先定位再治理：

```sql
-- 按 thread 看 checkpoint 数量
SELECT thread_id, count(*) AS n
FROM checkpoints
GROUP BY thread_id
ORDER BY n DESC
LIMIT 20;
```

常见原因：

| 原因 | 表现 | 治理 |
|------|------|------|
| 长循环 + 消息只增不减 | 单 thread 几千个 checkpoint | 循环内裁剪/摘要消息 |
| State 里放大对象 | 单行几十 MB | 大对象外置，State 只存 id |
| 从不清理 | 表无限增长 | 按时间/会话状态定期归档删除 |
| 高频轮询式调用 | 每轮都产生快照 | 合并调用，减少无意义 superstep |

治理手段：

1. **从源头减量**：State 瘦身 + 消息裁剪，这是收益最大的；
2. **定期归档**：把已结束会话的 checkpoint 迁到冷表/对象存储；
3. **按租户分 schema**：便于按租户整体清理；
4. **关注新版本的增量存储能力**（如 DeltaChannel 一类只存增量的机制），但要先确认稳定性和兼容性，别在生产盲上 beta。

**追问：能直接 `DELETE` 旧 checkpoint 吗？**
→ 可以，但要按 `thread_id` 整条时间线删，**不要只删中间某条**——会破坏父子链，导致恢复时找不到父快照。删除前先确认没有待恢复的会话。

---

## Q9. 上线后改了 State 的字段，老的 checkpoint 还能恢复吗？

**答：**

**不一定，这是生产升级最容易踩的坑。**风险点：

| 变更 | 风险 |
|------|------|
| **新增字段** | 老快照里没有该字段，节点读到 `KeyError` |
| **删除字段** | 老快照里的多余字段一般被忽略，但依赖它的节点会挂 |
| **改类型** | 反序列化或使用时类型不匹配 |
| **改 reducer** | 历史值的合并语义变了，重放结果不一致 |

工程做法：

1. **新增字段必须给默认值**，节点用 `state.get("x", default)` 读，别直接下标；
2. **先兼容再清理**：先加新字段、双写一段时间，确认没有老会话后再删旧字段；
3. **版本化 State**：在 State 里放 `schema_version`，入口节点做迁移/兼容分支；
4. **升级前先回放**：拿几条真实 thread 在预发环境跑 `get_state_history` + 恢复，验证兼容性；
5. **必要时切新 thread**：大改版直接开新会话，老会话只读不写。

**追问：为什么不能靠「反序列化时自动补默认值」？**
→ 因为快照是**按字段存的**，框架不知道你新加的字段应该是什么。默认值属于业务语义，只能由你在代码里给。

---

## 本篇自测

- [ ] Checkpointer 支撑的四件事，以及它不是「消息存储」
- [ ] checkpoint 的粒度是 superstep 还是节点？并行节点算几个？
- [ ] checkpoint 里至少说出四个组成部分
- [ ] `thread_id` 为什么是授权边界？多租户怎么命名和隔离？
- [ ] 三种后端的选择依据，SQLite 为什么不能上生产
- [ ] `setup()` 做了什么，为什么必须显式调用
- [ ] 时间旅行的两个 API，`as_node` 的作用
- [ ] Checkpointer 与 Store 的分工
- [ ] checkpoint 膨胀的三个原因和治理顺序
- [ ] State 加字段后，老会话怎么保证能恢复

---

## 导航

[[langgraph/interview/README|LangGraph 面试题库首页]] · 上一篇：[[langgraph/interview/03-控制流与动态编排]] · 下一篇：[[langgraph/interview/05-人机协同与中断恢复]] · 详解：[[langgraph/modules/langgraph_checkpoint/概述]] · [[langgraph/modules/langgraph_checkpoint/thread_id与会话管理]]
