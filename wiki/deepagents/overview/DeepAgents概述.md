---
tags: [deepagents, overview, agent-harness]
created: 2026-07-29
updated: 2026-07-29
type: concept
source_url: https://docs.langchain.com/oss/python/deepagents/overview
deepagents_version: ">=0.7"
---

# DeepAgents 概述

`deepagents` 是一个开源的 **Agent Harness（代理脚手架）**——基于 LangChain + LangGraph 构建的"开箱即用"长任务 Agent，内置规划、文件系统、子代理、上下文管理等能力。

> **一句话**：`create_agent` 是最小 Agent，`deepagents` 是内置了"规划+文件系统+子代理+摘要"全套战甲的完整 Agent。

## 一、与 create_agent 的关系

```
LangGraph 运行时
  +-- LangChain create_agent       <- 最小 Agent（ReAct 循环 + 中间件）
        +-- DeepAgents             <- 完整 Agent（create_agent + 默认中间件栈）
              ├── TodoListMiddleware      <- 任务规划
              ├── FilesystemMiddleware     <- 文件系统
              ├── SubAgentMiddleware       <- 子代理派发
              ├── SummarizationMiddleware  <- 上下文压缩
              ├── SkillsMiddleware         <- 技能按需加载
              ├── MemoryMiddleware         <- 跨线程记忆
              └── HumanInTheLoopMiddleware <- 关键操作审批
```

三者是**同一技术栈的不同封装层级**。deepagents 不替换 LangChain/LangGraph，而是预装配了最佳实践的中间件组合。

## 二、核心能力

| 能力 | 实现方式 | 解决的问题 |
|------|---------|-----------|
| **任务规划** | `write_todos` 工具（TodoListMiddleware） | 复杂任务拆解为子步骤，逐项跟踪完成状态 |
| **文件系统** | ls/read/write/edit/glob/grep（FilesystemMiddleware） | 大文件、长文本写入磁盘，不占上下文窗口 |
| **子代理** | `task` 工具（SubAgentMiddleware） | 大量搜索/分析结果隔离在子 Agent 中 |
| **上下文压缩** | 自动摘要（SummarizationMiddleware） | 长对话历史压缩，保持在 token 限制��� |
| **技能系统** | Markdown 技能（SkillsMiddleware） | 按需加载领域 SOP |
| **长期记忆** | Store backend（MemoryMiddleware） | 跨线程/跨会话持久化知识 |
| **人在环** | 工具调用前暂停（HumanInTheLoopMiddleware） | 敏感操作人工审批 |
| **沙箱执行** | Sandbox backend + `execute` 工具 | 代码安全隔离执行 |
| **权限控制** | `permissions` 参数 | 文件级读写控制 |

## 三、核心参数

```python
from deepagents import create_deep_agent

agent = create_deep_agent(
    model="claude-sonnet-4-6",            # 模型
    tools=[my_tool],                       # 自定义工具
    system_prompt="你是研究员...",          # 自定义指令
    backend=FilesystemBackend(...),         # 文件系统后端
    subagents=[CodeReviewSubagent],         # 自定义子代理
    skills=["code_review", "data_analysis"],# 技能列表
    memory=["AGENTS.md"],                   # 记忆文件
    permissions=[FilesystemPermission(...)],# 文件权限
    interrupt_on={"send_email": True},      # HITL 配置
    middleware=[CustomMiddleware()],         # 自定义中间件
)
```

## 四、与知识库其他模块的关系

| 知识库模块 | 关系 |
|-----------|------|
| [[langchain/middleware/中间件概述]] | DeepAgents 默认中间件栈的基础 |
| [[langchain/agents/Agent代理]] | `create_agent` → `create_deep_agent` 进化 |
| [[agent/skills/Skills实现与集成]] | DeepAgents 内置了 SkillsMiddleware |
| [[docker/沙箱/Docker沙箱概述]] | Sandbox backend 支持的沙箱类型 |
| [[fastapi/与LangChain集成]] | Deep Agent 部署为 FastAPI 服务 |

## 五、学习路径

```
第一轮：快速入门 → 理解默认行为
第二轮：中间件栈 → 理解内置能力来源
第三轮：内置工具 → 理解 filesystem/subagents/todos
第四轮：文件系统后端 → State/Filesystem/Store/Sandbox 区别
第五轮：Skills + 子代理 → 高级架构
第六轮：自定义与扩展 → 生产化定制
```

## 六、专题导航

| 目录 | 内容 |
|------|------|
| [[deepagents/overview/DeepAgents概述]] | 总览 |
| [[deepagents/overview/组件依赖全景]] | ⭐ 组件关联导航中枢：参数→中间件→工具→backend 完整链路 |
| [[deepagents/overview/快速入门]] | 第一个 Deep Agent |
| [[deepagents/overview/与LangChain_Agent对比]] | create_agent vs create_deep_agent |
| [[deepagents/middleware/中间件栈概述]] | 13 层中间件栈 |
| [[deepagents/middleware/FilesystemMiddleware详解]] | FilesystemMiddleware 生产配置 |
| [[deepagents/middleware/HumanInTheLoop详解]] | 人工审批：interrupt_on、四种决策、恢复执行 |
| [[deepagents/tools/内置工具概述]] | 内置工具完整说明 |
| [[deepagents/tools/MCP集成]] | MCP 工具加载与配置驱动接入 |
| [[deepagents/backend/后端概述]] | 6 种 Backend 选型 |
| [[deepagents/backend/Backend实战配置]] | 生产配置代码 |
| [[deepagents/backend/Sandbox后端详解]] | 沙箱隔离执行与安全 |
| [[deepagents/subagents/子代理概述]] | 子代理系统 |
| [[deepagents/context/上下文管理概述]] | 四层上下文管理 |
| [[deepagents/customization/自定义概述]] | HarnessProfile、权限、定制 |
| [[deepagents/skills/Skills系统]] | SkillsMiddleware 机制 |
| [[deepagents/production/源码架构]] | 源码架构与 graph.py |
| [[deepagents/production/生产部署]] | 生产完整骨架 |
| [[deepagents/production/Streaming与事件流]] | 流式输出与 WebSocket 推送 |
| [[deepagents/production/测试策略]] | 分层测试与 mock 模型 |
| [[deepagents/production/错误处理与重试]] | 超时、重试、错误兜底 |
