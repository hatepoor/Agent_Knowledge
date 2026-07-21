---
title: "AI 底层架构 | 菜鸟教程"
source: "https://www.runoob.com/ai-agent/ai-architecture.html"
author:
published:
created: 2026-06-23
description: "AI 底层架构  下图展示了基础大模型到智能体应用的完整 AI 系统架构：                             层级             核心组件             核心作用             典型能力                                         基础层（模型）             LLM & Token、Transformer 架构、Token.."
tags:
  - "AI 底层架构"
---
## AI 底层架构

下图展示了基础大模型到智能体应用的完整 AI 系统架构：

![](https://www.runoob.com/wp-content/uploads/2026/05/1632bc97-9f34-43ec-8f6e-9abedf817470-ai-ar.png)

| 层级 | 核心组件 | 核心作用 | 典型能力 |
| --- | --- | --- | --- |
| **基础层（模型）** | LLM & Token、Transformer 架构、Token 化、注意力机制 | AI 的底层计算核心；负责文本理解、Token 预测、语言生成 | 自然语言理解、文本生成、上下文建模、概率预测 |
| **上下文层（记忆）** | Context Window、Prompt、Memory、RAG | 管理模型输入上下文；负责短期记忆、长期记忆、外部知识注入 | Prompt 控制、会话记忆、知识检索、上下文增强 |
| **能力扩展层（工具）** | MCP、Tool Calling、API、Database | 让 AI 不止聊天；通过工具调用扩展真实世界操作能力 | 联网搜索、代码执行、数据库查询、API 调用 |
| **智能体层（决策）** | Agent、Explore、Plan、Act | AI 自主决策大脑；负责目标理解、任务拆解、规划与闭环执行 | 任务规划、多步骤推理、自主决策、闭环执行 |
| **应用层（行动）** | Agent Skill、Workflow、Automation | 面向具体业务场景；将 Agent 能力封装为可落地产品 | 自动化工作流、行业AI助手、企业智能系统、AI SaaS应用 |

### 核心名词释义

- **Transformer** ：当前大模型核心架构，通过自注意力机制建立 Token 之间关联关系。
- **Token** ：模型处理文本最小单位，可为子词、单词、字符或符号。
- **Prompt** ：给到模型的输入指令，用来设定角色、控制行为、规范输出格式与目标。
- **RAG** ：检索增强生成，先检索外部专业知识库，再交由 LLM 整合生成精准答案。
- **MCP** ：模型上下文协议 ( Model Context protocol )，统一 AI 与工具、数据库、外部服务的通信标准。
- **Agent** ：在 LLM 基础上叠加 **目标+规划+执行** 能力的自主智能系统。
- **Tool Calling** ：模型主动识别需求、调用外部工具完成实际操作，不局限纯文本回复。
- **Memory** ：为 AI 提供短期会话记忆 + 长期持久记忆，解决上下文遗忘问题。

### AI Agent 完整运行流程

- **用户输入** ：用户下发指令 Prompt，进入上下文窗口 Context Window。
- **记忆增强** ：系统联动 Memory 会话记忆 + RAG 外部知识库，补充上下文与专业知识。
- **LLM 推理** ：通过 Transformer 架构对 Token 序列做注意力计算，进行语义理解与初步生成。
- **Agent 规划** ：智能体自主判断、拆解任务，决策是否需要多步骤执行或调用外部工具。
- **工具执行** ：基于 MCP 协议，调用 API、数据库、搜索引擎等外部能力完成实操。
- **输出结果** ：汇总工具返回数据与模型推理结果，整理后返回用户，同时写入记忆存档。