---
tags: [entity, 工具, AI, 框架, 多Agent]
created: 2026-06-23
updated: 2026-06-23
type: entity
---

# AutoGen

## 概述

AutoGen 是微软开源的多 Agent 协同框架，专注于让多个 AI Agent 之间进行**自动化对话和任务协作**。它是构建多 Agent 系统的主流方案之一。

## 核心能力

- **多 Agent 对话**：定义多个 Agent 角色，自动编排对话流程
- **角色分工**：按职责拆分 Agent（如研究员、写作者、审核者）
- **消息传递**：Agent 之间自动交换消息，协作完成复杂任务
- **人机协作**：支持人类作为 Agent 参与对话流程

## 在 Agent 开发中的角色

在 [[agent/orchestration/开发Agent实操指南]] 中，AutoGen 被推荐用于**进阶场景**——多 Agent 协同：
- 按角色分工：研究员 Agent（查资料）+ 写作 Agent（写内容）+ 审核 Agent（改格式）
- 与 LangChain Swarm 并列为多 Agent 协同的推荐方案

## 适用场景

- **复杂任务拆分**：需要多个专业角色协作的场景
- **代码生成+审核**：一个 Agent 写代码，另一个 Agent 审核
- **研究+写作**：一个 Agent 查资料，另一个 Agent 撰写报告
- **辩论/讨论**：多个 Agent 从不同角度分析问题

## 相关实体
- [[agent/orchestration/LangChain]] — 通用框架，也提供多 Agent 能力（LangChain Swarm）
- [[agent/RAG/LlamaIndex]] — 知识融合框架，可为多 Agent 系统提供数据支持

## 相关概念
- [[agent/harness/Agent_三层架构]] — 多 Agent 协同是三层架构的扩展，每个 Agent 各有独立的三层结构

## 来源
- [[raw/Clippings/如何开发一个Agent，来手把手教你]] — 原文"加多Agent协同"章节
