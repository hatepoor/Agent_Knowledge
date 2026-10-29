---
tags: [redis, 中段, agent, langgraph, llm缓存, 会话, 任务分发, 限流, 生产]
type: guide
created: 2026-10-06
updated: 2026-10-06
---

# Agent 场景实战：缓存、会话与任务分发

## 学习目标

读完本篇，你应当能够：

1. 为一个多智能体系统画出"Redis 四角色"架构图，并按"缓存 / 状态"分实例规划
2. 实现 LLM 响应缓存与 embedding 缓存的生产版（含流式、非确定性输出的边界处理）
3. 用 Redis 存 Agent 会话（含 LangGraph Checkpointer 的 Redis 方案），配好滑动续期
4. 用 Stream 消费者组实现 Agent 间任务分发（幂等、僵尸接管、死信），并实现限流

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[redis/02-中段阶段-生产实践/03-缓存三大问题与缓存模式]] | Cache-Aside、互斥重建、Agent 缓存 key 初版设计 |
| [[redis/02-中段阶段-生产实践/02-发布订阅与Stream]] | 消费者组三件套（XREADGROUP/XACK/XAUTOCLAIM） |
| [[redis/02-中段阶段-生产实践/06-分布式锁]] | 防止同一任务被重复执行 |
| [[redis/02-中段阶段-生产实践/07-内存淘汰与maxmemory]] | 分实例纪律与淘汰策略选型 |
| [[redis/02-中段阶段-生产实践/05-管道与性能]] | pipeline 批量与连接池参数 |

**门槛声明**：本篇是中段的**整合实战篇**——不引入新的核心机制，把 8 个专题的知识组合成 Agent 系统的完整方案。所有专题至少过一遍再读本篇；只关心某个场景可按节跳读。

---

## 一、总览：Redis 在 Agent 系统里的四个角色

```
                     ┌──────────────────────────────┐
   Agent 进程 x N     │        Redis 实例群           │       外部依赖
 ┌───────────────┐   │ ┌──────────────────────────┐ │   ┌─────────────┐
 │ 调用 LLM 前    │──▶│ │ ① 缓存实例（allkeys-lfu） │ │──▶│ LLM API     │
 │ 查响应缓存     │   │ │   llm:* / emb:* / rag:*  │ │   │（贵、慢）    │
 ├───────────────┤   │ └──────────────────────────┘ │   └─────────────┘
 │ 读写会话/检查点│──▶│ ┌──────────────────────────┐ │   ┌─────────────┐
 │               │   │ │ ② 状态实例（noeviction+AOF）│ │──▶│ PostgreSQL  │
 ├───────────────┤   │ │   session:* / checkpoint:*│ │   │（正式存储）   │
 │ 投递/消费任务  │──▶│ └──────────────────────────┘ │   └─────────────┘
 │               │   │ ┌──────────────────────────┐ │
 ├───────────────┤   │ │ ③ 队列实例（noeviction）    │ │
 │ 全局限流/互斥  │──▶│ │   task:* / dlq:*          │ │
 └───────────────┘   │ └──────────────────────────┘ │
                     │ ④ 限流/锁可并入 ②（小系统并一实例）│
                     └──────────────────────────────┘
```

**分实例纪律落地**（复习 07"混布反模式"）：可淘汰的缓存（①）与不可丢的状态/队列（②③）**分开部署**——淘汰策略、持久化配置、故障预案都不同。小系统可以 ②③④ 并一个实例（都配 `noeviction` + AOF），① 独立。

| 实例 | 淘汰策略 | 持久化 | 丢了会怎样 |
|------|----------|--------|-----------|
| ① 缓存 | `allkeys-lfu` | 可关（TTL 兜底，丢了回填） | 多花一次 LLM/embedding 调用费 |
| ② 状态（会话/检查点） | `noeviction` | AOF everysec + 混合 | **用户会话丢失**，事故 |
| ③ 队列（任务/死信） | `noeviction` | AOF everysec | 任务丢失，事故 |

---

## 二、LLM 响应缓存（生产版）

03 第四节给了 key 设计初版，生产版补四个边界。

### key 设计的四条生产规则

```python
import hashlib
import json

def llm_cache_key(model: str, messages: list, **params) -> str:
    # 规则 1：影响输出的**全部**参数都进 key（model/温度/工具定义/system 版本）
    # 规则 2：参数做规范化序列化（排序 + 紧凑分隔），否则 "a","b" 与 "b","a" 算两个
    payload = json.dumps({"m": model, "msgs": messages, **params},
                         sort_keys=True, ensure_ascii=False)
    return f"llm:v1:{hashlib.sha256(payload.encode()).hexdigest()}"
    # 规则 3：key 带版本前缀 v1 —— prompt 模板/系统指令升级时把 v1 改 v2，
    #         旧缓存整体自然过期，不用清洗
    # 规则 4：temperature > 0 的调用**不缓存**——每次输出都不同，缓存是错的
```

### 流式响应的边界

流式输出（逐 token 推给前端）**不能直接缓存中间流**。生产做法：流结束后把**完整结果**写入缓存；下次命中则把完整结果**模拟成流**（分块吐出）——用户无感，成本归零：

```python
def chat_stream(messages, **params):
    if params.get("temperature", 0) == 0:
        key = llm_cache_key("deepseek-chat", messages, **params)
        cached = cache_r.get(key)
        if cached is not None:
            cache_r.incr("llm:stats:hit")
            for chunk in json.loads(cached):      # 缓存命中 → 模拟流
                yield chunk
            return
    chunks = []
    for chunk in call_llm_stream(messages, **params):   # 真实流式调用
        chunks.append(chunk)
        yield chunk
    if params.get("temperature", 0) == 0:
        cache_r.set(key, json.dumps(chunks), ex=86400 + random.randint(0, 3600))
        cache_r.incr("llm:stats:miss")
```

### 命中率就是钱：统计与收益估算

```python
hit = int(cache_r.get("llm:stats:hit") or 0)
miss = int(cache_r.get("llm:stats:miss") or 0)
print(f"命中率 {hit / max(hit + miss, 1):.1%}")
```

省钱账：确定性调用（temperature=0 的分类/抽取/总结类任务）命中率通常 30%+，**每 1% 命中率 = 1% 的该类 token 成本**。热门 prompt 的击穿防护用互斥重建（复习 03 第二节）。

---

## 三、Embedding 缓存

embedding 计算贵且**确定性**（同文本同向量），是收益最高、最安全的缓存（复习 03）。生产三个要点：

```python
# 要点 1：向量是二进制，用独立的 bytes 客户端（decode_responses=False）
#        —— 与文本客户端分开建，避免 decode 冲突（复习入门 3.2 陷阱）
emb_r = redis.Redis(host="...", decode_responses=False)

def emb_key(model: str, text: str) -> str:
    return f"emb:{model}:{hashlib.sha256(text.encode()).hexdigest()}"

def get_embeddings(model: str, texts: list[str]) -> list:
    keys = [emb_key(model, t) for t in texts]
    cached = emb_r.mget(keys)                          # 要点 2：MGET 批量读（复习 05）
    todo = [(i, t) for i, (c, t) in enumerate(zip(cached, texts)) if c is None]
    if todo:
        vectors = call_embedding_api(model, [t for _, t in todo])
        pipe = emb_r.pipeline(transaction=False)       # 要点 3：pipeline 批量回填
        for (i, _), vec in zip(todo, vectors):
            pipe.set(keys[i], np.asarray(vec, dtype=np.float32).tobytes())
        pipe.execute()
        for (i, _), vec in zip(todo, vectors):
            cached[i] = np.asarray(vec, dtype=np.float32).tobytes()
    return [np.frombuffer(c, dtype=np.float32) for c in cached]
```

- key 里带**模型名**：换 embedding 模型后向量空间不同，混用是隐性灾难。
- 选型两条路：手写如上；或直接用 LangChain 的封装（见 [[langchain/modules/langchain_embeddings/CacheBackedEmbeddings]]，其 Redis 后备存储就是本方案）。
- 策略配 `allkeys-lfu`（复习 07）：高频文档的向量常驻，长尾自然淘汰。

---

## 四、会话存储与 LangGraph Checkpointer

### 会话对象的拆分设计

**不要把整个会话塞进一个大 Hash/JSON**（多轮对话无界增长 → 大 key，复习 10）：

```
session:{sid}:meta   → Hash（用户 id、创建时间、agent 类型）        TTL 7 天，滑动续期
session:{sid}:msgs   → List（每轮对话 RPUSH 一条 JSON）             LTRIM 保留最近 200 条
session:{sid}:state  → Hash（Agent 工作状态：当前节点、中间结果）    TTL 7 天
```

```python
def append_message(sid: str, role: str, content: str):
    pipe = r.pipeline(transaction=False)
    pipe.rpush(f"session:{sid}:msgs",
               json.dumps({"role": role, "content": content}, ensure_ascii=False))
    pipe.ltrim(f"session:{sid}:msgs", -200, -1)          # 上界纪律（复习 10 陷阱 5）
    pipe.expire(f"session:{sid}:meta", 7 * 86400, gt=True)   # 滑动续期 7.0+
    pipe.execute()
```

**滑动续期**（`EXPIRE ... GT`，`7.0+`）：只在"当前 TTL 更长"时才设——活跃会话每次操作续 7 天，7 天不活跃自动清理，这正是会话想要的语义（复习 2.1 的条件过期）。

### LangGraph 的 Redis 检查点器

LangGraph 的状态持久化通过 Checkpointer 抽象，官方 Redis 实现装包即用（包名 `langgraph-checkpoint-redis`，迭代较快，装最新稳定版即可）：

```python
# pip install langgraph-checkpoint-redis
from langgraph.checkpoint.redis import RedisSaver

with RedisSaver.from_conn_string("redis://localhost:6379/0") as checkpointer:
    graph = builder.compile(checkpointer=checkpointer)   # 替换原 SqliteSaver（复习 langgraph 页）
    # thread_id 即会话 id；checkpoint 数据由包自动写 Redis 并管理 TTL
    graph.invoke({"messages": [...]}, config={"configurable": {"thread_id": "s1"}})
```

- 从 [[redis/01-入门阶段-基础教程/第00章-学前准备/0.1-Redis是什么]] 的定位看：checkpoint 属于**状态数据**（角色 ②）——`noeviction` + AOF 实例，给 key 配 TTL（如 30 天）做生命周期，但**不做缓存淘汰**。
- 与 PostgreSQL 存 checkpointer 的取舍：单机/小规模用 Redis（快、TTL 自动清理）；要求强持久审计用 PostgreSQL（你已有 PostgresSaver，复习 [[agent/memory/Agent_Memory]] 的选型框架）。

---

## 五、Agent 间任务分发：Stream 消费者组实战

### 先选型：List 还是 Stream（复习 02 后的决策表）

| 需求 | 用 |
|------|-----|
| 简单背压队列，丢一条可接受 | List（`LPUSH` + `BLPOP`） |
| **消息不许丢 + 多消费者 + 要重试/回溯** | Stream 消费者组（本节） |
| 纯广播通知（丢就丢） | Pub/Sub（复习 02 的局限） |

### 生产级生产者

```python
def submit_task(r, task: dict, stream="agent:tasks"):
    task_id = task["task_id"]                       # 调用方生成的唯一 id（UUID）
    # 幂等闸门：同 id 30 秒内拒绝重复投递（网络重试/用户双击防线）
    if not r.set(f"task:dedup:{task_id}", "1", nx=True, ex=30):
        raise ValueError(f"任务 {task_id} 提交过")
    payload = {k: str(v) for k, v in task.items()}
    r.xadd(stream, payload, maxlen=100_000, approximate=True)   # MAXLEN ~ 限长（复习 02）
```

**payload 纪律**：任务参数里的大对象（文档全文、图片）不放消息体——写对象存储，消息里只传引用（大 key 纪律，复习 10）。

### 生产级消费者（幂等 + 僵尸接管 + 死信）

```python
STREAM, GROUP = "agent:tasks", "workers"
DLQ = "agent:tasks:dlq"                     # 死信队列：重试超限的任务进这里
MAX_RETRY = 5

def run_worker(r, consumer: str):
    try:
        r.xgroup_create(STREAM, GROUP, id="0")
    except redis.ResponseError as e:
        if "BUSYGROUP" not in str(e):
            raise
    while True:
        # 1) 先收新消息（复习 02：> 只取未投递的）
        msgs = r.xreadgroup(GROUP, consumer, {STREAM: ">"}, count=1, block=10000)
        # 2) 再接管僵尸消息：空闲 > 60s 且未 ACK 的，抢过来重做
        #    Redis 8.4+ 可用 XREADGROUP ... CLAIM 一步完成（先升级再简化此处）
        claimed, _, _ = r.xautoclaim(STREAM, GROUP, consumer,
                                     min_idle_time=60_000, count=1)
        for source in (msgs, [None] if not claimed else [(STREAM, claimed)]):
            for _, entries in source or []:
                for msg_id, fields in entries:
                    handle_one(r, msg_id, fields)

def handle_one(r, msg_id, fields):
    task_id = fields["task_id"]
    retries = int(r.incr(f"task:retry:{task_id}"))       # 重试计数（随任务存）
    r.expire(f"task:retry:{task_id}", 86400)
    try:
        # 幂等执行：任务 ID 去重 + 结果写入即可重入安全
        execute_task(fields)
        r.xack(STREAM, GROUP, msg_id)                    # 成功才 ACK（at-least-once）
        r.delete(f"task:retry:{task_id}")
    except Exception as e:
        if retries >= MAX_RETRY:
            r.xadd(DLQ, {**fields, "error": str(e)})     # 死信 + 告警
            r.xack(STREAM, GROUP, msg_id)
        # 不 ACK → 留在 PEL，60s 后被其他消费者接管（复习 02 的 at-least-once）
```

**死信不是垃圾桶**：`DLQ` 流要接告警与人工处理流程——进死信的任务代表"自动重试已放弃"，通常是需要修代码或修数据的真问题。

### 与分布式锁的组合

同一时刻"一个用户只该有一个运行中的 Agent 会话"这类互斥，用 06 的三零件锁（`SET NX PX` + Lua 释放），锁 key 用 `lock:agent:{sid}`——锁保效率不保正确（复习 06 的定位），正确性靠任务系统的幂等。

---

## 六、限流：保护下游 LLM 配额

LLM API 有 RPM/TPM 配额，多实例共享配额必须用集中式限流——Redis 是标准答案。

### 固定窗口（简单场景）

```lua
-- 固定窗口计数：KEYS[1]=限流key ARGV[1]=窗口秒 ARGV[2]=上限
local current = redis.call('INCR', KEYS[1])
if current == 1 then
    redis.call('EXPIRE', KEYS[1], tonumber(ARGV[1]))
end
if current > tonumber(ARGV[2]) then
    return 0
end
return 1
```

```python
allow = limit_script(keys=[f"ratelimit:{api_key}:60s"],
                     args=[60, 100])          # 每分钟 100 次
if not allow:
    raise HTTPException(429, "LLM 配额限流中")
```

`INCR` + `EXPIRE` 必须用 Lua 绑原子（复习 04：两条命令间可被插队，非原子版在窗口边界会放行超额）。固定窗口的已知缺陷是**窗口边界双倍突刺**（前后两个窗口的边缘各放行满额），要求平滑用滑动窗口或令牌桶（`Redis 8.x` 开源版内置 `BF`/概率结构之外，限流也可用 `CL.THROTTLE` 类扩展，或自行实现令牌桶 Lua）。

### 限流 key 的层级实践

```
ratelimit:llm:{api_key}:60s      全局 API 配额
ratelimit:agent:{sid}:10s        单会话操作频控（防循环调用失控）
ratelimit:tool:{tool_name}:60s   单工具配额
```

限流 key 全部短 TTL 自动过期，属角色 ②④ 之间的轻量数据——放状态实例即可，无需持久化保障。

---

## 七、陷阱

1. **四个角色挤一个实例**：缓存淘汰把会话/任务挤没了（07 的事故场景）。至少把"可淘汰的缓存"与"不可丢的状态+队列"分开。
2. **缓存层故障拖垮主流程**：Redis 挂了，`chat_stream` 抛 `ConnectionError` 直接 500。缓存调用一律 try/except 降级——**缓存是加速器，不是依赖**：

```python
try:
    cached = cache_r.get(key)
except redis.RedisError:
    cached = None          # 降级：直接走 LLM，慢一点但能用
```

3. **temperature > 0 也进缓存**：缓存住"随机创意"输出，用户每次看到一样还以为坏了。规则 4 是硬边界。
4. **流式接口直接缓存中间块**：只缓存完整结果，命中端模拟流。缓存半截流是格式灾难。
5. **会话 List/Stream 无上界**：长会话跑一晚上，msgs 涨成大 key（复习 10）。所有会话集合 key 出生时就写好 `LTRIM`/`XTRIM`。
6. **消费者组用了 `noack`**：消息取走即确认，进程崩 = 任务丢（复习 02 的 noack 语义）。要 at-least-once 就显式 XACK。
7. **多进程共享一个 ConnectionPool**：连接池不是跨进程的，gunicorn/多进程子进程各自建池（且各自受 `max_connections` 约束，复习 05 的池规模估算）。
8. **幂等只靠 Redis 去重键**：`task:dedup` 有 TTL，TTL 过后同 id 仍可重复投递——幂等的最后防线在**执行端**（任务结果可覆盖写、状态机可重入），Redis 去重只是第一道闸。

---

## 小结

- 四角色架构：缓存（allkeys-lfu 可丢）/ 状态（noeviction+AOF）/ 队列（noeviction）/ 限流，至少缓存与状态分实例
- LLM 缓存四规则：全参数进 key、规范化序列化、版本前缀、temperature>0 不缓存；流式只缓存完整结果
- Embedding 缓存：bytes 客户端 + MGET + pipeline 回填，key 带模型名，收益最高
- 会话：meta/msgs/state 三 key 拆分 + `LTRIM` 上界 + `EXPIRE GT` 滑动续期；LangGraph 用 RedisSaver
- 任务分发：幂等闸门 + at-least-once（显式 XACK）+ XAUTOCLAIM 接管僵尸 + MAX_RETRY 死信告警；大 payload 外置
- 限流：INCR+EXPIRE 用 Lua 绑原子；分层 key（全局/会话/工具）；缓存调用全部可降级

## 练习

1. 按第一节把你的 Agent 项目的 Redis 使用清单填进"四角色"表——每个 key 标注角色、淘汰策略归属、丢失影响；填不进表的就是架构漏洞。
2. 实现 `chat_stream` 的缓存版，用 temperature=0 与 0.7 各跑三遍，验证：0 命中缓存（响应毫秒级）、0.7 三次输出不同。
3. 搭一个双消费者任务系统：故意让消费者 A 处理到一半 `kill -9`，观察消费者 B 通过 `XAUTOCLAIM` 接管消息重做；再造一个永远失败的任务，验证 5 次后进死信。
4. 给你的 LLM 调用加固定窗口限流，压测超过阈值的行为（429 响应占比），并确认 Lua 原子版与非原子两版在并发下的放行数差异。

## 相关专题

- 缓存模式与三大问题：[[redis/02-中段阶段-生产实践/03-缓存三大问题与缓存模式]]
- 一致性与删缓存失败的重试：[[redis/02-中段阶段-生产实践/09-缓存一致性-与数据库同步]]
- Stream 语义详解：[[redis/02-中段阶段-生产实践/02-发布订阅与Stream]]
- 大 key 纪律与无界集合：[[redis/02-中段阶段-生产实践/10-大Key与热Key治理]]
- 锁的正确性边界：[[redis/02-中段阶段-生产实践/06-分布式锁]]
- LangChain 侧的缓存封装：[[langchain/modules/langchain_embeddings/CacheBackedEmbeddings]]
- Agent 记忆体系选型：[[agent/memory/Agent_Memory]]
