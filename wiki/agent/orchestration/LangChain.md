---
tags: [entity, 工具, AI, 框架, LLM]
created: 2026-06-23
updated: 2026-06-23
type: entity
---

# LangChain

## 概述

LangChain 是当前生态最全面的 LLM 应用开发框架，支持 Python 和 JavaScript 两种语言。它提供了构建 Agent、RAG 管道、对话系统等 LLM 应用的完整工具链，是开发 AI Agent 最流行的开源框架。

## 核心能力

| 模块 | 功能 |
|------|------|
| Chains | 将多个 LLM 调用和工具串联成工作流 |
| Agents | 让 LLM 自主决策使用哪些工具完成任务 |
| Memory | 会话上下文管理（ConversationBufferMemory（已弃用）等） |
| Tools | 统一的工具接口，可挂载搜索、计算、文件读取等 |
| Retrievers | 文档检索组件，支持向量检索等多种策略 |
| PlanAndExecute | 任务规划与分步执行模块 |

## Agent 开发中的角色

在 [[agent/orchestration/开发Agent实操指南]] 中，LangChain 是**开源框架二次开发**的首选方案，覆盖 Agent 开发的全链路：
- 对接大模型 API（OpenAI/文心一言/通义千问）
- 配置 Prompt（定义 Agent 角色、任务、规则）
- 挂载工具（文件读取、联网搜索等）
- 实现「输入→思考→执行→输出」闭环

> 更详细的使用方法，参见 [[langchain/overview/LangChain概述]] 和 [[langchain/LangChain使用手册]]。

## 生态地位

- **最完整的工具链**：从原型到部署的全流程覆盖
- **最大的社区**：丰富的文档、教程和第三方集成
- **跨语言支持**：Python（LangChain）和 JS（LangChain.js）

## 相关实体
- [[agent/RAG/LlamaIndex]] — 侧重知识融合的替代框架
- [[agent/orchestration/AutoGen]] — 侧重多 Agent 协同的框架
- [[agent/RAG/RAG]] — LangChain 提供完整的 RAG 管道实现

## 相关概念
- [[agent/harness/Agent_三层架构]] — LangChain 的 Agent 模块实现了完整的三层架构
- [[agent/memory/Agent_Memory]] — LangChain 内置多种记忆管理方案
- [[agent/harness/Function_Call]] — LangChain 的 Tools 模块封装了 Function Call 机制

## 学习入口
- [[langchain/overview/LangChain概述]] — 官方文档概述，create_agent 入门
- [[langchain/LangChain使用手册]] — 12 章完整使用手册
- [[langchain/modules/LangChain模块详解]] — 16 个核心模块详解
- [[langchain/agents/Agent代理]] — Agent 核心概念

## 来源
- [[raw/Clippings/如何开发一个Agent，来手把手教你]] — 原文推荐的核心开发框架
