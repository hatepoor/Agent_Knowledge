---
tags: [entity, 协议, AI, Agent, 工具]
type: entity
created: 2026-06-23
updated: 2026-07-08
---

# MCP（Model Context Protocol）

## 概述
MCP（模型上下文协议，Model Context Protocol）是 Agent 连接外部工具、数据库、外部服务的**标准化通信协议**。它为 AI 模型与外部世界之间建立了统一的接口标准，类似于 USB 为各种设备提供的标准化连接方式。

## 核心作用
在 [[raw/AI Agent教程/AI 底层架构  菜鸟教程]] 的五层架构中，MCP 位于**能力扩展层（工具层）**，是 Agent 调用外部能力的标准化通道：
- 统一 AI 与工具、数据库、外部服务的通信标准
- 让 Agent 能以标准化方式连接任意外部系统
- 降低工具集成的开发成本

## 设计理念
MCP 的核心理念是**标准化**：
- 不同的工具提供商只需实现一次 MCP 接口
- 不同的 AI 模型/Agent 框架只需对接一次 MCP 协议
- 实现工具和模型的解耦，提升生态互操作性

## 在 Agent 架构中的位置
MCP 是 [[agent/harness/Function_Call]] 的标准化升级：
- Function Call 是模型级别的工具调用机制
- MCP 是协议级别的工具连接标准
- 两者配合：Agent 通过 Function Call 机制，经由 MCP 协议调用外部工具

## 与 A2A 的关系
MCP 与 [[a2a/README|A2A]] 是互补而非替代关系：
- **MCP** 用于**纵向**扩展：连接 API、数据库、外部工具（需要编码+服务器）
- **A2A** 用于**横向**协作：让不同团队/厂商的独立 Agent 发现彼此、委派任务、交换结果
- 一句话记法：**MCP 让智能体拿到工具，A2A 让智能体找到同行**
- ⚠️ 同一代码库内的多个 Agent **不需要** A2A，直接函数调用即可

## 与 Skills 的对比
MCP 与 [[agent/skills/Skills]] 是 Agent 生态中互补的两种能力扩展方式：
- **MCP** 用于**能力扩展**：连接 API、数据库、外部工具（需要编码+服务器）
- **Skills** 用于**知识复用**：封装 SOP、经验、工作流程（纯 Markdown，零门槛）

## 与相关概念的关系
- [[agent/harness/Function_Call]] — MCP 是 Function Call 的协议级标准化
- [[agent/harness/Agent_三层架构]] — MCP 连接思考层（决策）和执行层（工具）
- [[agent/prompt/上下文工程]] — MCP 管理的工具定义是上下文的重要组成部分
- [[agent/skills/Skills]] — MCP 用于能力扩展，Skills 用于知识复用，二者互补
- [[a2a/README]] — A2A 协议知识库（横向的 agent↔agent，与 MCP 互补）

## 来源
- [[agent/sources/菜鸟教程_AI_Agent系列]] — AI 底层架构篇
- [[agent/sources/菜鸟教程_AI_Agent系列]] — 术语篇中 Harness 组件部分
