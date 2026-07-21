---
tags: [langchain, middleware, planning, todo]
created: 2026-07-15
updated: 2026-07-15
type: source
source_url: https://docs.langchain.com/oss/python/langchain/middleware/built-in#todo-list
langchain_version: ">=1.0"
---

# TodoListMiddleware（待办列表中间件）

## 一、定义与作用

`TodoListMiddleware` 自动给 Agent 添加一个 `write_todos` 工具，让 Agent **可以维护和更新自己的待办列表**。

Agent 在处理复杂任务时，会自动把任务拆分成步骤（todos），并随着推进更新状态。这样：
- Agent 自己有"工作计划"，不容易遗漏步骤
- 用户能看到 Agent 当前在做什么、下一步做什么
- 复杂任务可中断/恢复——恢复时 Agent 知道进行到哪一步

**核心价值**：让 Agent 拥有"结构化任务管理"能力，从"走一步看一步"升级为"先规划再执行"。

## 二、使用场景

| 场景 | 案例 |
|------|------|
| 多步骤研究任务 | Agent 接到"调研 X 公司"后拆分成 5 步 |
| 复杂开发任务 | 拆分成"读代码→改代码→写测试→运行测试" |
| 长任务可观测 | 用户实时看到 Agent 进度 |
| 任务中断恢复 | 跨多个会话重新加载 todo 列表继续 |

**不适合**：
- 简单任务（直接回答"今天天气"）— 拆 todo 反而浪费 token
- 一次性工具调用任务

## 三、参数详解

| 参数 | 类型 | 说明 |
|------|------|------|
| `system_prompt` | `string` | 引导 todo 使用的自定义系统提示 |
| `tool_description` | `string` | `write_todos` 工具的自定义描述 |

## 四、使用方法

### 基础用法

```python
from langchain.agents import create_agent
from langchain.agents.middleware import TodoListMiddleware

agent = create_agent(
    model="deepseek-v4-flash",
    tools=[search_web, read_file, write_file],
    middleware=[
        TodoListMiddleware(),  # 自动添加 write_todos 工具
    ],
)

# 用户：调研 langchain 中间件
# Agent 自动调用 write_todos：
# [
#   {"content": "搜索中间件官方文档", "status": "in_progress", "active_form": "正在搜索中间件官方文档"},
#   {"content": "读取核心概念章节", "status": "pending", "active_form": "读取核心概念章节"},
#   {"content": "整理 18 个中间件清单", "status": "pending", "active_form": "整理 18 个中间件清单"},
#   {"content": "写总结报告", "status": "pending", "active_form": "写总结报告"},
# ]
```

### 自定义系统提示

```python
TodoListMiddleware(
    system_prompt="你是一个有任务规划能力的助手。复杂任务开始前必须先调用 write_todos 拆分步骤。每完成一步立即更新状态。",
    tool_description="更新你的待办列表。复杂任务必须用此工具拆分步骤。",
)
```

## 五、注意事项

1. **Agent 不会强制使用**：虽然提供了工具，但 Agent 可能选择不用（特别是任务很简单时）。在 `system_prompt` 中强化要求可以提高使用率

2. **状态不会自动持久化**：todo 存在 state 里，跨 thread 不保留。如需跨会话保留，用 `StoreBackend`

3. **状态字段名**：`todos`（不是 `todo`），自定义中间件访问时注意

4. **过度拆分的代价**：Agent 拆出 20 个 todo 反而浪费 token。建议在 prompt 中要求"5-10 步内完成"

5. **配合 SubAgentMiddleware**：父 Agent 拆任务 → 派子 Agent 执行，是更好的架构

6. **观察 todo 变化**：可以用 `stream_events` 监听 `write_todos` 工具的调用：

```python
for event in agent.stream_events(...):
    if event["event"] == "on_tool_end":
        if event["name"] == "write_todos":
            print(f"Todo 更新: {event['data'].tool_output}")
```

## 六、相关页面

- [[langchain/middleware/中间件概述]] — 总览
- [[langchain/middleware/SubAgentMiddleware]] — 派生子 Agent
- [[langchain/middleware/RubricMiddleware]] — 评分量规
- [[langchain/agents/Agent循环实现]] — 理解 Agent 如何决策
