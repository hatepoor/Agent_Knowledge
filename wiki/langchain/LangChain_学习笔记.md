---
tags: [langchain, overview, agent, study]
created: 2026-07-07
updated: 2026-07-07
type: overview
---

# LangChain 学习笔记

本目录用于系统化学习 [[agent/orchestration/LangChain|LangChain]] 框架。已于 2026-07-07 完成官方核心文档的全量抓取与中文化翻译，按原文档目录结构组织。

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
│   └── Agent代理.md
├── tools/                         # 工具系统
│   └── 工具.md
├── models/                        # 模型系统
│   └── 模型.md
├── memory/                        # 记忆系统
│   └── 短期记忆.md
├── runtime/                       # 运行时
│   └── 运行时上下文.md
├── streaming/                     # 流式传输
│   └── 流式传输.md
├── multi-agent/                   # 多 Agent 系统
│   └── 多Agent系统.md
├── structured-output/             # 结构化输出
│   └── 结构化输出.md
├── middleware/                     # 中间件
│   └── 中间件概述.md
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
- [[langchain/tools/工具|工具（Tools）]] — 工具定义、上下文访问、动态选择、错误处理
- [[langchain/models/模型|模型（Models）]] — 初始化、调用、工具调用、结构化输出、动态选择

### 记忆与运行时
- [[langchain/memory/短期记忆|短期记忆（Short-term Memory）]] — Checkpointer、裁剪、删除、摘要化策略
- [[langchain/runtime/运行时上下文|运行时上下文（Runtime）]] — 依赖注入、Context、Store、Stream Writer

### 高级特性
- [[langchain/streaming/流式传输|流式传输（Streaming）]] — updates/messages/custom 模式、Reasoning Tokens、子 Agent 流
- [[langchain/multi-agent/多Agent系统|多 Agent 系统（Multi-agent）]] — 子 Agent、交接、技能、路由器模式及性能对比
- [[langchain/structured-output/结构化输出|结构化输出（Structured Output）]] — ProviderStrategy、ToolStrategy、错误处理
- [[langchain/middleware/中间件概述|中间件概述（Middleware）]] — 钩子系统、内置中间件、自定义中间件

### 实战教程
- [[langchain/tutorial/智能体核心技术教程/智能体核心技术教程|智能体核心技术教程]] — 4 大模块综合实战

---

## 交叉引用
- [[agent/orchestration/LangChain]] — 框架级实体介绍
- [[agent/RAG/RAG]] — 检索增强生成概念
- [[agent/planning/ReAct]] — Agent 推理范式
- [[agent/harness/Function_Call]] — 工具调用机制

---

## 学习路径建议

1. **基础入门** → [[langchain/overview/LangChain概述]] → [[langchain/install/安装LangChain]] → [[langchain/quickstart/快速入门]]
2. **核心组件** → [[langchain/agents/Agent代理]] → [[langchain/tools/工具]] → [[langchain/models/模型]]
3. **状态管理** → [[langchain/memory/短期记忆]] → [[langchain/runtime/运行时上下文]]
4. **高级特性** → [[langchain/streaming/流式传输]] → [[langchain/structured-output/结构化输出]] → [[langchain/middleware/中间件概述]]
5. **多 Agent** → [[langchain/multi-agent/多Agent系统]]
6. **综合实战** → [[langchain/tutorial/智能体核心技术教程/智能体核心技术教程]]

---

*来源：https://docs.langchain.com/oss/python/ 官方文档，2026 年 7 月版本。*
*新来源到达时请按 [SCHEMA](SCHEMA.md) 工作流归档。*
