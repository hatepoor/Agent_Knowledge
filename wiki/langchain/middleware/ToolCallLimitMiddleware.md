---
tags: [langchain, middleware, cost-control, tool-limit]
created: 2026-07-15
updated: 2026-07-15
type: source
source_url: https://docs.langchain.com/oss/python/langchain/middleware/built-in#tool-call-limit
langchain_version: ">=1.0"
---

# ToolCallLimitMiddleware（工具调用限制中间件）

## 一、定义与作用

`ToolCallLimitMiddleware` 限制 Agent 调用**特定工具**或**所有工具**的次数。与 `ModelCallLimitMiddleware` 互补：后者管模型调用，这个管工具调用。

**核心价值**：
- 防止 Agent 反复调用昂贵的工具（如付费搜索 API）
- 防止死循环（Agent 反复调同一个工具）
- 细粒度控制：可以为"查询数据库"设 50 次上限，但"发邮件"只能 1 次

## 二、使用场景

| 场景 | 案例 |
|------|------|
| 付费 API 调用 | Tavily 搜索每月限额，强制 Agent 最多调 N 次 |
| 防止死循环 | Agent 反复调 `web_search` 找不到答案，限制 5 次后强制总结 |
| 关键操作限流 | "发邮件"工具整个 thread 只能调 1 次 |
| 调试 | 让 Agent 最多用 3 个工具就出答案 |

## 三、参数详解

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `tool_name` | `string` | — | 要限制的**特定工具名**（不提供则全局生效） |
| `thread_limit` | `number` | — | 整个 thread 的最大工具调用数（需 checkpointer） |
| `run_limit` | `number` | — | 单次 run 的最大工具调用数（每次用户消息重置） |
| `exit_behavior` | `string` | `"continue"` | `'continue'` / `'error'` / `'end'` |

### exit_behavior 三种模式

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| `"continue"` | 屏蔽超额调用，Agent 继续运行 | 生产环境，最安全 |
| `"error"` | 抛 `ToolCallLimitExceededError` | 开发/测试，快速暴露问题 |
| `"end"` | 仅在**单工具**场景下生效，结束 Agent | 配合 thread_limit 强制结束 |

> **注意**：`thread_limit` 和 `run_limit` 至少需指定一个。

## 四、使用方法

### 限制特定工具（推荐）

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ToolCallLimitMiddleware

# 限制 web_search 工具
agent = create_agent(
    model="deepseek-v4-flash",
    tools=[web_search, send_email, calculator],
    middleware=[
        ToolCallLimitMiddleware(
            tool_name="web_search",
            thread_limit=10,         # 整个会话最多 10 次
            run_limit=3,             # 单次响应最多 3 次
            exit_behavior="continue",
        ),
    ],
)
```

### 限制所有工具（全局）

```python
ToolCallLimitMiddleware(
    # 不传 tool_name 表示限制所有工具
    thread_limit=20,
    run_limit=10,
    exit_behavior="continue",
)
```

### 多个工具分别限制

```python
middleware = [
    ToolCallLimitMiddleware(tool_name="web_search", thread_limit=10),
    ToolCallLimitMiddleware(tool_name="send_email", thread_limit=1, run_limit=1),
    ToolCallLimitMiddleware(tool_name="calculator", run_limit=50),  # 数学不限
]
```

## 五、注意事项

1. **`tool_name` 区分大小写**：必须与 `tools=[...]` 中工具的名称完全一致

2. **"continue" 行为细节**：超过限制后，模型仍决定要调该工具，但 `ToolMessage` 返回的是类似 `"Tool call limit reached"` 的错误消息。模型看到这个错误后会**自行决定**下一步

3. **"end" 仅限单工具**：如果指定了 `tool_name`，`end` 行为会立即停止 Agent；不指定 `tool_name` 时 `end` 行为无效

4. **需要 checkpointer**：`thread_limit` 需要持久化

5. **嵌套调用计数**：如果一个工具内部又调了另一个工具（MCP 工具链），会**累加**计数

6. **建议组合使用**：
   - 全局 `ToolCallLimitMiddleware`（防失控）
   - 关键工具（`send_email`、`delete_*`）单独的严格限制
   - 付费 API（`web_search`、`database_query`）按预算限制

## 六、相关页面

- [[langchain/middleware/ModelCallLimitMiddleware]] — 模型调用限制
- [[langchain/middleware/HumanInTheLoopMiddleware]] — 关键操作人工审批
- [[langchain/middleware/中间件概述]] — 总览
