---
tags: [langchain, agents, react, loop, implementation, internals]
created: 2026-07-08
updated: 2026-07-08
type: concept
---

# Agent 循环实现原理

`create_agent` 创建的 ReAct Agent 底层是一个**编译好的 LangGraph 状态机图**。理解其四层架构有助于精确掌握 Agent 的行为。

---

## 第一层：`create_agent` 编译成 LangGraph 图

调用 `create_agent()` 时，它**不是**直接执行 ReAct 循环，而是编译一个 LangGraph 的 `StateGraph` 状态机：

```python
agent = create_agent(model="gpt-5.5", tools=[search], system_prompt="...")
# ↑ 内部编译为一个 StateGraph：
#   agent_node（调 LLM） → 判断 → tools_node（执行工具） → agent_node（再调 LLM）
#                              ↕
#                         checkpointer（持久化）
```

这个图在 `create_agent()` 调用时**一次性编译完成**，之后的 `agent.invoke()` 只是在这个图上运行。

---

## 第二层：ReAct 循环的核心逻辑

编译后的图驱动以下循环（伪代码）：

```python
def react_loop(state: AgentState) -> dict:
    """
    state = {
        "messages": [SystemMessage, HumanMessage, ...],  # 完整对话历史
    }
    """

    while True:
        # ============ 步骤 1：调 LLM ============
        # 把当前 messages 发给模型，模型返回响应
        # 响应可能包含 tool_calls（模型想调用工具）
        response = model.invoke(state["messages"])

        # ============ 步骤 2：判断是否有工具调用 ============
        if not response.tool_calls:
            # 模型没有要求调用工具 → 循环结束，返回最终答案
            state["messages"].append(response)
            return state

        # ============ 步骤 3：执行全部工具 ============
        # 模型可能在一次响应中要求调用多个工具（并行）
        for tool_call in response.tool_calls:
            tool_name = tool_call["name"]
            tool_args = tool_call["args"]

            tool_fn = find_tool(tool_name)
            result = tool_fn.invoke(tool_args)

            state["messages"].append(ToolMessage(
                content=str(result),
                tool_call_id=tool_call["id"],
            ))

        # ============ 步骤 4：持久化 ============
        if checkpointer:
            checkpointer.put(current_thread_id, state)

        # ============ 回到循环开头 ============
        # LLM 看到工具执行结果后，要么继续调工具，要么给出最终答案
```

---

## 第三层：中间件钩子位置

六个钩子精确嵌入了循环的各个阶段：

```
┌──────────────────────────────────────────────────────┐
│ before_agent                                          │
│     │                                                │
│     ▼                                                │
│ before_model                                          │  ← SummarizationMiddleware 在这压缩历史
│     │                                                │
│     ▼                                                │
│  LLM 调用 ← 返回响应（可能含 tool_calls）                │
│     │                                                │
│     ▼                                                │
│ after_model                                           │
│     │                                                │
│     ├── tool_calls 不为空 ──→ before_tool              │
│     │                           │                    │
│     │                           ▼                    │
│     │                      执行工具                    │
│     │                           │                    │
│     │                           ▼                    │
│     │                       after_tool                │
│     │                           │                    │
│     └── tool_calls 为空 ───── 跳出循环 → after_agent   │
│                                                    │
│  ←—— 回到 before_model（循环）                         │
└──────────────────────────────────────────────────────┘
```

各钩子的触发时机：

| 钩子 | 触发时机 | 典型用途 |
|------|---------|----------|
| `before_agent` | 整个 Agent 开始时 | 初始化 |
| `before_model` | 每次 LLM 调用前 | 压缩历史、注入上下文 |
| `after_model` | 每次 LLM 调用后 | 日志、修改响应 |
| `before_tool` | 每个工具执行前 | 审批、日志 |
| `after_tool` | 每个工具执行后 | 修改结果、审计 |
| `after_agent` | 整个 Agent 结束时 | 清理、汇总 |

---

## 第四层：Checkpointer 的持久化点

每次循环迭代结束时（执行完工具后），框架自动调用 `checkpointer.put()`：

```
第 1 轮：LLM → 搜索 → put()  → 持久化本轮状态
第 2 轮：LLM → 计算 → put()  → 持久化本轮状态
第 3 轮：LLM → 给出答案 → put() → 持久化最终状态
```

这就是为什么即使 Agent 崩溃在中间某一步，重启后也能从最新检查点恢复。

---

## 完整示例

用户问"北京和东京的天气怎么样"：

```python
# 第 1 次迭代
# state["messages"] = [System, Human("北京和东京的天气？")]

# before_model → LLM
# LLM 返回：tool_calls = [get_weather(北京), get_weather(东京)]  # 并行！
# after_model

# before_tool → 执行 get_weather("北京") → "22°C" → after_tool
# before_tool → 执行 get_weather("东京") → "18°C" → after_tool

# checkpointer.put()  ← 持久化

# 第 2 次迭代
# state["messages"] = [System, Human, AIMessage(tool_calls),
#                      ToolMessage("22°C"), ToolMessage("18°C")][[

# before_model → LLM（看到两个天气结果）
# LLM 返回：content = "北京22°C，东京18°C"（无 tool_calls → 结束）

# after_agent  ← 循环终止
```

## 相关页面

- [[langchain/agents/Agent代理]] — Agent 核心概念
- [[langchain/agents/系统提示词]] — System Prompt 编写指南
- [[langchain/middleware/中间件概述]] — 中间件系统
- [[langchain/memory/短期记忆]] — Checkpointer 核心概念
- [[langchain/memory/Checkpointer存储后端]] — 数据库存储原理
- [[agent/patterns/ReAct]] — ReAct 推理范式
