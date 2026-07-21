---
tags: [langchain, middleware, cost-control, limit]
created: 2026-07-15
updated: 2026-07-15
type: source
source_url: https://docs.langchain.com/oss/python/langchain/middleware/built-in#model-call-limit
langchain_version: ">=1.0"
---

# ModelCallLimitMiddleware（模型调用限制中间件）

## 一、定义与作用

`ModelCallLimitMiddleware` 限制 Agent 在一次会话中**调用模型的次数**，防止无限循环或成本失控。

它解决了一个真实的痛点：ReAct Agent 有时会陷入"调工具→失败→重试→再失败"的死循环，账单在不知情的情况下爆炸。这个中间件就是**保险丝**。

**核心价值**：给 Agent 加上"硬性熔断"，超过阈值立即停止。

## 二、使用场景

| 场景 | 案例 |
|------|------|
| 生产环境 ReAct Agent | 防止 Agent 陷入工具调用死循环 |
| 用户测试 | 限制试用版用户的模型调用次数 |
| 多租户 SaaS | 按线程/会话计费，防止单用户耗光配额 |
| 调试模式 | 强制 Agent 在 5 步内给出答案，加速调试 |

## 三、参数详解

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `thread_limit` | `number` | 无限制 | 整个 thread 内**所有 run** 的累计最大模型调用数 |
| `run_limit` | `number` | 无限制 | 单次**用户消息触发的 run** 内最大模型调用数 |
| `exit_behavior` | `string` | `"end"` | `'end'`（优雅终止）/ `'error'`（抛 `ModelCallLimitExceededError`） |

### thread_limit vs run_limit 的区别

```
用户消息1 → 模型调用1 → 工具调用 → 模型调用2 → 工具调用 → 模型调用3
                ↑─────────── run 1 (3 次) ───────────↑

用户消息2 → 模型调用4 → ...
                ↑─────────── run 2 (1 次) ───────────↑
                                                    ↑ thread 共 4 次
```

- `run_limit`：限制"一次用户消息"内的次数（每次新消息重置）
- `thread_limit`：限制"整个对话"内的次数（永不重置）

## 四、使用方法

### 基础用法

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ModelCallLimitMiddleware

agent = create_agent(
    model="deepseek-v4-flash",
    tools=[...],
    middleware=[
        ModelCallLimitMiddleware(
            thread_limit=20,        # 整个会话最多 20 次模型调用
            run_limit=10,           # 单次响应最多 10 次
            exit_behavior="end",    # 超过后优雅返回（不抛异常）
        ),
    ],
)
```

### 严格模式（抛异常）

```python
ModelCallLimitMiddleware(
    thread_limit=20,
    run_limit=10,
    exit_behavior="error",   # 超过后抛 ModelCallLimitExceededError
)
```

## 五、注意事项

1. **需要 checkpointer**：`thread_limit` 需要持久化来追踪跨 run 的调用次数

2. **exit_behavior="end" 行为**：Agent 在当前 step 结束**之后**停止，而不是立即停止。可能会多执行 1-2 个 tool 调用

3. **exit_behavior="error" 适用场景**：开发/测试环境，希望快速发现死循环；生产环境一般用 `"end"` 优雅终止

4. **run_limit 不需要 checkpointer**：仅基于当前 state 内的 messages 计算

5. **tool_call 不计入此限制**：只看模型调用次数。如要限制工具调用，用 `ToolCallLimitMiddleware`

6. **调参建议**：
   - 简单 RAG 任务：`run_limit=5`
   - ReAct + 多工具任务：`run_limit=15`
   - 复杂研究型 Agent：`run_limit=30`
   - `thread_limit` 通常是 `run_limit` 的 3-5 倍

## 六、相关页面

- [[langchain/middleware/ToolCallLimitMiddleware]] — 限制工具调用次数
- [[langchain/middleware/中间件概述]] — 总览
- [[langchain/agents/Agent循环实现]] — 理解 ReAct 循环如何计数
