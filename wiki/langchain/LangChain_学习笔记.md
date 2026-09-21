---
tags: [langchain, overview, agent, study]
created: 2026-07-07
updated: 2026-09-10
type: overview
---

# LangChain 学习笔记

本目录用于系统化学习 [[agent/orchestration/LangChain|LangChain]] 框架。已于 2026-07-07 完成官方核心文档的全量抓取与中文化翻译，后续持续补充 modules 子目录和中间件详解。

---

## 总入口

📘 **章节化使用手册** → [[langchain/LangChain使用手册|LangChain 使用手册（章节结构总览）]] — 按章节结构组织的查询入口
🧩 **模块详解** → [[langchain/modules/LangChain模块详解|LangChain 模块详解]] — 16 个核心模块的用途、关键 API、完整示例

---

## 目录结构

```
wiki/langchain/
├── LangChain_学习笔记.md          # 本页（导航入口）
├── LangChain使用手册.md           # 章节化总览（推荐入口）
├── overview/                      # 概述
│   └── LangChain概述.md
├── install/                       # 安装
│   └── 安装LangChain.md
├── quickstart/                    # 快速入门
│   └── 快速入门.md
├── agents/                        # Agent 核心
│   ├── Agent代理.md
│   ├── Agent循环实现.md
│   └── 系统提示词.md
├── tools/                         # 工具系统
│   ├── 工具.md
│   ├── 工具来源与集成.md
│   ├── 工具注册与调度.md
│   └── Tavily搜索工具.md
├── models/                        # 模型系统
│   └── 模型.md
├── basics/                        # 基础对比
│   ├── LangChain核心方法对比.md
│   └── Agent_invoke_vs_ChatModel_invoke.md
├── memory/                        # 记忆系统（6 篇）
│   ├── 短期记忆.md
│   ├── 长期记忆.md
│   ├── 上下文压缩.md
│   ├── 记忆系统性能.md
│   ├── Checkpointer存储后端.md
│   └── 示例代码.md
├── runtime/                       # 运行时
│   └── 运行时上下文.md
├── streaming/                     # 流式传输
│   └── 流式传输.md
├── multi-agent/                   # 多 Agent 系统
│   └── 多Agent系统.md
├── structured-output/             # 结构化输出
│   └── 结构化输出.md
├── middleware/                    # 中间件（23 篇）
│   ├── 中间件概述.md
│   ├── 自定义中间件开发.md
│   ├── 最佳实践.md
│   ├── 与LangGraph关系.md
│   ├── 钩子机制详解.md
│   ├── ContextEditingMiddleware.md
│   ├── FilesystemMiddleware.md
│   ├── FilesystemFileSearchMiddleware.md
│   ├── HumanInTheLoopMiddleware.md
│   ├── LLMToolEmulator.md
│   ├── LLMToolSelectorMiddleware.md
│   ├── ModelCallLimitMiddleware.md
│   ├── ModelFallbackMiddleware.md
│   ├── ModelRetryMiddleware.md
│   ├── PIIMiddleware.md
│   ├── ProviderToolSearchMiddleware.md
│   ├── RubricMiddleware.md
│   ├── ShellToolMiddleware.md
│   ├── SubAgentMiddleware.md
│   ├── SummarizationMiddleware.md
│   ├── TodoListMiddleware.md
│   ├── ToolCallLimitMiddleware.md
│   └── ToolRetryMiddleware.md
├── modules/                       # 核心模块 API 详解（100+ 篇）
│   ├── LangChain模块详解.md          # 总览入口
│   ├── langchain_core/
│   ├── langchain_agents/          # create_agent 参数与用法
│   ├── langchain_callbacks/       # 回调与 LangSmith 追踪
│   ├── langchain_checkpoints/     # Checkpoint 与人机交互
│   ├── langchain_documents/       # 文档加载与分割
│   ├── langchain_embeddings/      # Embedding 模型
│   ├── langchain_messages/        # 消息类型与裁剪
│   ├── langchain_models/          # 模型输入输出与结构化
│   ├── langchain_output_parsers/  # 输出解析器
│   ├── langchain_prompts/         # 提示词模板
│   ├── langchain_retrievers/      # 检索器（7 种）
│   ├── langchain_runnables/       # Runnable 与 LCEL
│   ├── langchain_stores/          # Store 长期存储
│   ├── langchain_tools/           # 工具运行时
│   ├── langchain_vectorstores/    # 向量存储
│   └── langchain-mcp/            # MCP 协议集成
└── tutorial/                      # 综合实战
    └── 智能体核心技术教程/
        └── 智能体核心技术教程.md
```

---

## 文档索引

### 入门
- [[langchain/overview/LangChain概述|LangChain 概述]] — 框架定位、LangChain vs LangGraph vs Deep Agents
- [[langchain/install/安装LangChain|安装 LangChain]] — pip 安装、提供商包安装
- [[langchain/quickstart/快速入门|快速入门]] — 构建首个 Agent、从基础到真实世界应用

### 核心组件
- [[langchain/agents/Agent代理|Agent（代理）]] — Agent = Model + Harness，核心组件、调用、框架配置
- [[langchain/agents/Agent循环实现|Agent 循环实现]] — Agent 循环执行逻辑
- [[langchain/agents/系统提示词|系统提示词]] — 系统提示词编写
- [[langchain/tools/工具|工具（Tools）]] — 工具定义、上下文访问、动态选择、错误处理
- [[langchain/tools/工具来源与集成|工具来源与集成]] — MCP / 自定义 / 社区工具
- [[langchain/tools/工具注册与调度|工具注册与调度]] — 工具注册与动态调度
- [[langchain/tools/Tavily搜索工具|Tavily 搜索工具]] — 搜索工具集成
- [[langchain/models/模型|模型（Models）]] — 初始化、调用、工具调用、结构化输出、动态选择
- [[langchain/basics/LangChain核心方法对比|核心方法对比]] — invoke / stream / batch
- [[langchain/basics/Agent_invoke_vs_ChatModel_invoke|Agent vs ChatModel 调用]] — 调用差异

### 记忆与运行时
- [[langchain/memory/短期记忆|短期记忆（Short-term Memory）]] — Checkpointer、裁剪、删除、摘要化策略
- [[langchain/memory/长期记忆|长期记忆（Long-term Memory）]] — Store 长期记忆机制
- [[langchain/memory/上下文压缩|上下文压缩]] — 上下文压缩中间件
- [[langchain/memory/记忆系统性能|记忆系统性能]] — 性能分析与 OOM 防护
- [[langchain/memory/Checkpointer存储后端|Checkpointer 存储后端]] — 后端对比与选型
- [[langchain/runtime/运行时上下文|运行时上下文（Runtime）]] — 依赖注入、Context、Store、Stream Writer

### 高级特性
- [[langchain/streaming/流式传输|流式传输（Streaming）]] — updates/messages/custom 模式、Reasoning Tokens、子 Agent 流
- [[langchain/multi-agent/多Agent系统|多 Agent 系统（Multi-agent）]] — 子 Agent、交接、技能、路由器模式及性能对比
- [[langchain/structured-output/结构化输出|结构化输出（Structured Output）]] — ProviderStrategy、ToolStrategy、错误处理
- [[langchain/middleware/中间件概述|中间件概述（Middleware）]] — 钩子系统、内置中间件、自定义中间件
- [[langchain/middleware/最佳实践|中间件最佳实践]] — 选型与组合策略

### 核心模块 API 详解（modules/）
- [[langchain/modules/LangChain模块详解|模块总览]] — 16 个核心模块地图
- [[langchain/modules/langchain_core/概述|langchain_core]] — 基础类型与抽象
- [[langchain/modules/langchain_agents/概述|langchain_agents]] — Agent 创建与配置
- [[langchain/modules/langchain_callbacks/概述|langchain_callbacks]] — 回调系统与 LangSmith
- [[langchain/modules/langchain_checkpoints/概述|langchain_checkpoints]] — Checkpoint 与人机交互
- [[langchain/modules/langchain_documents/概述|langchain_documents]] — 文档加载与分割
- [[langchain/modules/langchain_embeddings/概述|langchain_embeddings]] — Embedding 模型
- [[langchain/modules/langchain_messages/概述|langchain_messages]] — 消息类型与裁剪
- [[langchain/modules/langchain_models/概述|langchain_models]] — 模型输入输出与结构化
- [[langchain/modules/langchain_output_parsers/概述|langchain_output_parsers]] — 输出解析器
- [[langchain/modules/langchain_prompts/概述|langchain_prompts]] — 提示词模板
- [[langchain/modules/langchain_retrievers/概述|langchain_retrievers]] — 检索器（7 种）
- [[langchain/modules/langchain_runnables/概述|langchain_runnables]] — Runnable 与 LCEL
- [[langchain/modules/langchain_stores/概述|langchain_stores]] — Store 长期存储
- [[langchain/modules/langchain_tools/概述|langchain_tools]] — 工具运行时
- [[langchain/modules/langchain_vectorstores/概述|langchain_vectorstores]] — 向量存储
- [[langchain/modules/langchain-mcp/概述|langchain-mcp]] — MCP 协议集成

### 实战教程
- [[langchain/tutorial/智能体核心技术教程/智能体核心技术教程|智能体核心技术教程]] — 4 大模块综合实战

---

## 交叉引用
- [[agent/orchestration/LangChain]] — 框架级实体介绍
- [[agent/RAG/RAG]] — 检索增强生成概念
- [[agent/patterns/ReAct]] — Agent 推理范式
- [[agent/harness/Function_Call]] — 工具调用机制
- [[rag/RAG_学习路径导航]] — RAG 14 步全流程学习入口
- [[langgraph/overview/LangGraph概述]] — LangGraph 框架
- [[deepagents/overview/DeepAgents概述]] — DeepAgents 框架

---

## 学习路径建议

1. **基础入门** → [[langchain/overview/LangChain概述]] → [[langchain/install/安装LangChain]] → [[langchain/quickstart/快速入门]]
2. **核心组件** → [[langchain/agents/Agent代理]] → [[langchain/tools/工具]] → [[langchain/models/模型]]
3. **状态管理** → [[langchain/memory/短期记忆]] → [[langchain/runtime/运行时上下文]]
4. **高级特性** → [[langchain/streaming/流式传输]] → [[langchain/structured-output/结构化输出]] → [[langchain/middleware/中间件概述]]
5. **多 Agent** → [[langchain/multi-agent/多Agent系统]]
6. **模块深挖** → [[langchain/modules/LangChain模块详解]] → 按需进入各子模块
7. **综合实战** → [[langchain/tutorial/智能体核心技术教程/智能体核心技术教程]]

---

*来源：https://docs.langchain.com/oss/python/ 官方文档，2026 年 7 月版本。*
*新来源到达时请按 [SCHEMA](../../SCHEMA.md) 工作流归档。*
