---
tags: [concept, AI, Agent, 记忆, 向量数据库]
created: 2026-06-23
updated: 2026-06-23
type: concept
---

# Agent Memory（Agent 记忆系统）

## 定义

Agent Memory 是 AI Agent 维护和利用历史信息的能力系统，是 Agent 区别于普通 Chatbot 的核心能力之一。它让 Agent 能够记住用户偏好、历史交互、学习到的知识，从而提供个性化和连贯的交互体验。

## 核心组成

### 短期记忆（Short-term Memory）
- **实现方式**：框架自带的会话上下文管理
- **典型方案**：LangChain 的 `ConversationBufferMemory`、滑动窗口、摘要压缩
> ⚠️ ConversationBufferMemory 已弃用。新项目使用 create_agent(checkpointer=InMemorySaver()) 管理对话历史。
- **特点**：单次会话内有效，会话结束即丢失
- **用途**：维护对话连贯性、跟踪当前任务上下文

### 长期记忆（Long-term Memory）
- **实现方式**：对接向量数据库，将信息转为向量存储，通过相似性检索调取
- **典型技术栈**：
  - 向量数据库：Milvus、LanceDB、FAISS、Chroma
  - 嵌入模型：OpenAI Embedding、BGE、GTE
- **极简方案**：LlamaIndex 的 `SimpleVectorStore`，几行代码即可实现
- **用途**：用户偏好、历史知识、跨会话信息持久化

### 记忆的生命周期管理
- **写入**：交互中提取关键信息，向量化后存入记忆库
- **检索**：新交互时，根据语义相似性检索相关记忆
- **衰减/裁剪**：低频无用信息定期删除，避免检索效率下降
- **更新**：同一实体的新旧信息冲突时，需要覆盖或标注

## 关键设计考量

1. **不是越多越好** — 记忆要做裁剪和衰减，无限膨胀会导致检索精度下降
2. **向量检索精度** — 可加关键词过滤、重排序等策略提升召回质量
3. **隐私合规** — 用户记忆数据需要加密存储和访问权限控制
4. **成本控制** — 向量存储和检索都有成本，需要平衡记忆量和预算

## 相关概念
- [[agent/harness/Agent_三层架构]] — 记忆是思考层的核心子系统
- [[agent/RAG/RAG]] — 长期记忆的底层检索机制与 RAG 共享技术基础
- [[agent/harness/Function_Call]] — 记忆检索可作为一种工具被 Function Call 调用
- [[agent/memory/短期记忆]] — 短期记忆实战：Checkpointer 用法、自定义状态、裁剪/删除/摘要压缩
- [[langchain/memory/短期记忆]] — LangChain 短期记忆实现（Checkpointer）
- [[langchain/memory/上下文压缩]] — LangChain 上下文压缩方案

## 来源
- [[raw/Clippings/如何开发一个Agent，来手把手教你]] — 原文"叠加核心能力"中的记忆章节
