---
tags: [deepagents, comparison, create_agent, langchain]
created: 2026-07-29
updated: 2026-07-29
type: comparison
deepagents_version: ">=0.7"
---

# DeepAgents vs LangChain Agent

`create_deep_agent` 与 `create_agent` 是同一技术栈的两个封装层级。选择哪一个取决于你需要的"开箱即用程度"。

## 一、定位对比

| | `create_agent` | `create_deep_agent` |
|---|---|---|
| 包 | `langchain` | `deepagents`（依赖 langchain） |
| 定位 | 最小 Agent 框架 | 预装配完整 Agent Harness |
| 默认中间件 | 0 个（全部手动配） | 13 层（默认栈自动装配） |
| 默认工具 | 0 个 | 8 个（write_todos + 6 个文件系统 + task；`execute` 仅沙箱后端出现） |
| 默认 backend | 无 | StateBackend（虚拟文件系统） |
| 子代理 | 需手动配 SubAgentMiddleware | 自动内置 |
| 上下文管理 | 需手动配 Summarization | 自动内置 + Prompt Caching |
| Skills | 需手动实现 | SkillsMiddleware 自动加载 |

## 二、代码对比

### create_agent（手动配置所有）

```python
from langchain.agents import create_agent
from langchain.agents.middleware import (
    TodoListMiddleware, FilesystemMiddleware,
    SubAgentMiddleware, SummarizationMiddleware,
)

agent = create_agent(
    model=llm,
    tools=[my_tool],
    middleware=[
        TodoListMiddleware(),
        FilesystemMiddleware(backend=...),
        SubAgentMiddleware(subagents=[...]),
        SummarizationMiddleware(model=summary_llm),
        # ... 还有 6 个没配
    ],
)
```

### create_deep_agent（开箱即用）

```python
from deepagents import create_deep_agent

agent = create_deep_agent(
    model=llm,
    tools=[my_tool],
    # 上面 create_agent 手动配的 4 个中间件 +
    # Skills / Memory / HITL / PatchToolCalls / PromptCaching
    # 全部自动装配
)
```

## 三、何时用哪个

| 场景 | 推荐 |
|------|------|
| 简单 RAG / 单轮问答 | `create_agent` |
| 需要 100% 控制中间件栈 | `create_agent` |
| 多步骤复杂任务（研究/编码/分析） | `create_deep_agent` |
| 需要文件系统 + 规划 + 子代理 | `create_deep_agent` |
| 要快速搭建生产 Agent 原型 | `create_deep_agent` |
| 已有自己的 LangGraph 图，只需要 ReAct 循环 | `create_agent` |
| 需要自定义每一层但不想从零开始 | `create_deep_agent` + 逐层覆盖 |

## 四、可以混合使用

```python
from deepagents import create_deep_agent, CompiledSubAgent
from langchain.agents import create_agent

# Deep Agent 作为主控
# LangChain Agent 作为子代理
simple_agent = create_agent(model=llm, tools=[calculator])

deep = create_deep_agent(
    model=llm,
    subagents=[
        CompiledSubAgent(
            name="calculator",
            description="纯数学计算",
            runnable=simple_agent,  # ← create_agent 的结果
        ),
    ],
)
```

**共同基础**：两者底层都是 LangGraph `CompiledStateGraph`，支持相同的 streaming、checkpointing、interrupt。

## 五、迁移指南

从 `create_agent` 迁移到 `create_deep_agent`：

```python
# Before: create_agent
agent = create_agent(
    model=llm,
    tools=[my_tool],
    system_prompt="你是助手",
    middleware=[Summary, HITL],
)

# After: create_deep_agent
agent = create_deep_agent(
    model=llm,
    tools=[my_tool],
    system_prompt="你是助手",
    middleware=[CustomSummary, CustomHITL],  # 同名匹配自动替换原位
    # 其他中间件自动装配
)
```

**需要注意的差异**：
- `create_deep_agent` 默认带了文件系统工具——如果不想让 Agent 写文件，用 `permissions` 限制或 `FilesystemMiddleware(tools=["read_file", "ls"])`
- system prompt 会与内置 prompt 合并，不能完全覆盖（用 HarnessProfile 实现完全替换）
- `interrupt_on` 参数直接传给 `HumanInTheLoopMiddleware`，无需手动配置

## 六、相关页面

- [[deepagents/overview/DeepAgents概述]] — 完整架构
- [[deepagents/overview/快速入门]] — 上手
- [[deepagents/middleware/中间件栈概述]] — 默认栈详解
- [[langchain/agents/Agent代理]] — create_agent 文档
