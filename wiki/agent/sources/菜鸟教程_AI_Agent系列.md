---
tags: [source, AI, Agent, 教程, 菜鸟教程]
created: 2026-06-23
updated: 2026-06-23
type: source
source: https://www.runoob.com/ai-agent/
---

# 菜鸟教程 AI Agent 完整系列

## 概述

菜鸟教程发布的 AI Agent 系列教程（共 14 篇），是一套面向初中级开发者的全链路学习资料。从 LLM 基础概念到 Agent 部署落地，系统覆盖了构建 AI Agent 所需的全部知识层次。

## 知识体系分层

### 基础层

- **LLM 原理**：Transformer 架构、自注意力机制、逐词生成原理，是理解一切上层应用的根基
- **API 调用**：OpenAI、DeepSeek、通义千问等主流模型的接口实践
- **[[agent/perception/多模态]]**：文本 + 图像 + 音频 + 视频的统一处理能力，扩展了 Agent 的感知边界

### 上下文层

- **[[agent/prompt/提示词工程]]**：清晰指令、角色设定、思维链（CoT）、少样本示例、XML 标签隔离——直接影响模型输出质量
- **[[agent/prompt/上下文工程]]**：上下文预算管理、系统提示工程化、工具描述优化、历史压缩策略、检索质量控制——决定 Agent 能"记住多少、理解多深"
- **[[agent/RAG/RAG]]**：离线索引 + 在线查询、文档切分策略、混合检索、重排序、GraphRAG——突破 LLM 知识时效性与容量限制
- **[[agent/RAG/向量数据库]]**：Embedding 原理、相似度计算（余弦/欧氏/点积）、HNSW/IVF 索引、主流选型对比

### Agent 核心层

- **Agent 定义**：Agent = LLM + Planning + Tool Use + Memory
- **核心组件**：大脑（LLM）、工具（[[agent/harness/Function_Call]]）、记忆（[[agent/memory/Agent_Memory]]：短期/长期/情节/语义）、规划
- **[[agent/harness/Agent_六种架构]]**：单 Agent 循环、规划+执行、多 Agent 协作、反思修正、RAG+Agent、工作流 DAG
- **推理规划**：CoT、ReAct、Plan-and-Execute、ToT、MCTS、Reflexion
- **术语大全**：10 大类别 100+ 术语详解，涵盖编程范式、工程角色、模型底层、Prompt 工程、Agent 架构、Harness 组件、Loop 工具、工具生态、评估安全、演进路线

### 应用层

- **Agent 类型**：单一任务、多模态、自主、模拟
- **多 Agent 协作**：角色分工、消息传递、[[agent/orchestration/AutoGen]] / CrewAI 等框架实践
- **部署落地**：Gradio/Streamlit 测试版、Docker + K8s 正式版
- **框架生态**：[[agent/orchestration/LangChain]]、[[agent/RAG/LlamaIndex]]、[[agent/orchestration/AutoGen]]、LangGraph、OpenAI Agents SDK、Semantic Kernel 等 20+ 框架

## 与其他概念的关系

- [[agent/harness/Agent_三层架构]] — 教程以三层架构（感知→思考→执行）为骨架组织知识
- [[agent/harness/Agent_六种架构]] — 核心层的重要组成，详解六种架构模式
- [[agent/harness/Function_Call]] — 工具调用的底层机制
- [[agent/memory/Agent_Memory]] — 记忆系统贯穿 Agent 全生命周期
- [[agent/prompt/上下文工程]] — 决定 Agent 实际表现的关键工程实践
- [[agent/prompt/提示词工程]] — 入门到进阶的第一课
- [[agent/RAG/RAG]] — 突破 LLM 知识限制的核心技术方案
- [[agent/RAG/向量数据库]] — RAG 的基础设施
- [[langchain/modules/langchain-mcp/MCP协议概述]] — 工具生态的新标准协议

## 相关实体

- [[agent/orchestration/LangChain]] — 最流行的 LLM 应用开发框架
- [[agent/RAG/LlamaIndex]] — 专注于数据连接和索引
- [[agent/orchestration/AutoGen]] — 微软推出的多 Agent 协作框架
- [[agent/RAG/RAG]] — 检索增强生成技术
- [[agent/RAG/向量数据库]] — 向量存储与检索基础设施
- [[langchain/modules/langchain-mcp/MCP协议概述]] — Model Context Protocol，工具连接协议

## 来源

- 菜鸟教程 AI Agent 系列：https://www.runoob.com/ai-agent/ （共 14 篇）
