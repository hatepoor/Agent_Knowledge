---
tags: [langchain, middleware, subagent, deepagents, orchestration]
created: 2026-07-15
updated: 2026-07-15
type: source
source_url: https://docs.langchain.com/oss/python/langchain/middleware/built-in#subagent
langchain_version: ">=1.0"
deepagents_version: ">=0.6"
---

# SubAgentMiddleware（子代理中间件）

## 一、定义与作用

`SubAgentMiddleware` 来自 `deepagents` 包，它让主 Agent 可以通过 `task` 工具**派生子 Agent** 来执行子任务。子 Agent 有自己独立的上下文、工具和模型。

**核心价值**：
1. **上下文隔离**：子 Agent 的大量中间结果（搜索结果、文件读取）不会污染主 Agent 的上下文
2. **专业分工**：不同子 Agent 可以用不同模型/工具/系统提示
3. **任务并行**：多个子任务可以并发执行

这是 **Deep Agent 架构** 的核心——把复杂任务拆解给"专家团队"完成。

## 二、使用场景

| 场景 | 案例 |
|------|------|
| 研究任务 | 主 Agent 调度"搜索 Agent"+"总结 Agent"+"写报告 Agent" |
| 多步骤开发 | 主 Agent 调度"代码搜索 Agent"+"代码生成 Agent"+"测试 Agent" |
| 多模态任务 | 主 Agent 调度"图片理解 Agent"+"文本生成 Agent"+"格式转换 Agent" |
| 上下文爆炸场景 | 主 Agent 不能直接读 100 个文件，交给"读文件 Agent"汇总 |

## 三、参数详解

| 参数 | 说明 |
|------|------|
| `default_model` | 默认模型（子 Agent 共用） |
| `default_tools` | 默认工具（子 Agent 共用） |
| `subagents` | 子 Agent 列表（每个含 `name`、`description`、`system_prompt`、`tools`，可选 `model` 和 `middleware`） |

### 子 Agent 的字段

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | ✓ | 子 Agent 名称（用于 `task` 工具调用） |
| `description` | ✓ | 描述这个子 Agent 做什么（主 Agent 根据描述决定派发） |
| `system_prompt` | ✓ | 子 Agent 的系统提示 |
| `tools` | ✗ | 子 Agent 可用的工具（默认继承主 Agent 的工具） |
| `model` | ✗ | 子 Agent 的模型（默认用 `default_model`） |
| `middleware` | ✗ | 子 Agent 的中间件列表 |

### CompiledSubAgent

可以提供**预构建的 LangGraph 图**作为子 Agent：

```python
from deepagents import CompiledSubAgent

custom_graph = (
    StateGraph(...)
    .add_node(...)
    .add_edge(...)
    .compile()
)

SubAgentMiddleware(
    subagents=[
        CompiledSubAgent(
            name="custom_researcher",
            description="专门做研究的 LangGraph 图",
            runnable=custom_graph,
        ),
    ],
)
```

## 四、使用方法

### 基础用法

```python
# pip install deepagents
from langchain.agents import create_agent
from deepagents.middleware import SubAgentMiddleware

agent = create_agent(
    model="deepseek-v4-flash",
    tools=[],
    middleware=[
        SubAgentMiddleware(
            default_model="deepseek-v4-flash",
            subagents=[
                {
                    "name": "researcher",
                    "description": "专门做网络搜索和资料研究",
                    "system_prompt": "你是一个专业的研究员。使用搜索工具深入调研问题。",
                    "tools": [web_search, read_url],
                },
                {
                    "name": "writer",
                    "description": "专门写文章、报告",
                    "system_prompt": "你是一个专业写手。基于研究材料生成结构化文档。",
                    "tools": [write_file],
                },
            ],
        ),
    ],
)
```

### 配合 FilesystemMiddleware

```python
from deepagents.middleware import SubAgentMiddleware, FilesystemMiddleware

agent = create_agent(
    model="deepseek-v4-flash",
    middleware=[
        FilesystemMiddleware(),    # 主 Agent 有文件系统
        SubAgentMiddleware(
            subagents=[
                {
                    "name": "code_reviewer",
                    "description": "读代码、找 bug",
                    "system_prompt": "你是一个严格的代码审查员。",
                    "tools": [read_file, grep_search],  # 子 Agent 自己的工具
                },
            ],
        ),
    ],
)
```

## 五、注意事项

1. **来自 `deepagents` 包**：需要单独安装 `pip install deepagents`

2. **主 Agent 始终可访问 `general-purpose` 子 Agent**：这个特殊子 Agent 与主 Agent 指令相同，拥有所有工具，主要用于"上下文隔离"

3. **子 Agent 不共享 state**：子 Agent 的 todos、消息历史与主 Agent 独立。如果需要传递信息，用 `task` 工具的 `inputs` 参数

4. **子 Agent 可以再嵌套**：子 Agent 也可以有自己的 `SubAgentMiddleware`，形成多层架构

5. **description 决定派发质量**：主 Agent 根据 description 决定派给哪个子 Agent。description 写得好，调度就准

6. **token 成本**：派子 Agent 不省 token（多了系统提示+调度开销），但**省主 Agent 的上下文**

7. **状态管理**：子 Agent 的中间结果默认不返回主 Agent。如果子 Agent 找到关键信息，需要主动 `write_file` 到共享文件，主 Agent 再读取

## 六、相关页面

- [[langchain/middleware/中间件概述]] — 总览
- [[langchain/middleware/TodoListMiddleware]] — 任务规划
- [[langchain/middleware/FilesystemMiddleware]] — 配合文件系统实现子 Agent 间通信
- [[langchain/middleware/RubricMiddleware]] — 评分量规
- [[langchain/multi-agent/多Agent系统]] — 多 Agent 架构
