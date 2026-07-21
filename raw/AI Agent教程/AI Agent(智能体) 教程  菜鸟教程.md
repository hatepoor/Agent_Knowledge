---
title: "AI Agent(智能体) 教程 | 菜鸟教程"
source: "https://www.runoob.com/ai-agent/ai-agent-tutorial.html"
author:
published:
created: 2026-06-23
description: "AI Agent(智能体) 教程    AI Agent（Artificial Intelligence Agent） 称为智能体，本质是自动执行任务的程序，核心在于让模型不只回答问题，而是按步骤完成动作。  AI Agent（人工智能代理） 是一个能够感知环境、进行决策并执行行动，以达成特定目标的智能软件实体，它不仅仅是回答问题的聊天机器人，更是能够动手做事的智能执行者。  Agent = LLM (大脑) + Planning (规.."
tags:
  - "AI Agent(智能体) 教程"
---
## AI Agent(智能体) 教程

![](https://www.runoob.com/wp-content/uploads/2025/12/1745211719204.jpeg)

AI Agent（Artificial Intelligence Agent） 称为智能体，本质是自动执行任务的程序，核心在于让模型不只回答问题，而是按步骤完成动作。

**AI Agent（人工智能代理）** 是一个能够感知环境、进行决策并执行行动，以达成特定目标的智能软件实体，它不仅仅是回答问题的聊天机器人，更是能够动手做事的智能执行者。

Agent = LLM (大脑) + Planning (规划) + Tool use (执行) + Memory (记忆)。

快速体验 0 代码，一句话生成应用： [https://www.miaoda.cn/](https://www.miaoda.cn/?invitecode=user-93thly701s00) 。

---

## 谁适合阅读本教程？

1. 想使用 AI 自动化日常任务的人
2. 对编程不熟但想用 AI 做实际工作的新人
3. 已会基本电脑操作、但对 Agent/工作流 等概念零基础的人
4. 想把 AI 从聊天提升到真正干活的人

---

## 什么是 Agent？

Agent 就是一个能干活的智能助手。

Agent = LLM (大脑) + Planning (规划) + Tool use (执行) + Memory (记忆)。

学习 Agent 需要思维转变： 从对话框问答进化为目标驱动的任务执行。

![](https://www.runoob.com/wp-content/uploads/2025/12/0_0_ezapX2F_7BOysP.png)

传统的软件程序遵循固定的指令流程：输入 → 处理 → 输出，而 AI Agent 则更像一个有自主性的员工，它能够：

- **理解任务目标** ：明白你想要什么结果
- **制定计划** ：思考如何达成目标
- **使用工具** ：调用各种资源和 API
- **自我调整** ：根据反馈优化策略
- **持续执行** ：直到完成任务或遇到无法解决的问题

**类比理解：**

- 传统程序 = 自动售货机：投币 → 按按钮→ 出商品
- AI Agent = 私人助理：告诉需求 → 助理规划 → 完成任务并汇报

---

**核心结构：**

- **目标（Goal）：** 知道要完成什么
- **决策（Reasoning）：** 规划执行步骤
- **工具（Tools）：** 调用 API、代码或系统完成任务

**工作流程：**

```
输入 → 思考 → 调用工具 → 执行 → 返回结果 → 持续迭代
```

**与普通大模型区别：**

- 大模型：输出内容
- Agent：输出结果，并推动执行

比如我们在与 AI Agent 对话，输出：规划三天北京旅行，预算 5000，智能体就会完成以下任务：

- 拆解需求
- 查询机票、酒店、景点
- 生成行程方案
- 满足条件时继续完成预订

![](https://www.runoob.com/wp-content/uploads/2025/12/75e97117-0606-41cb-a044-bb38a4858735.jpg)

---

## 学习资源

现有平台及流行框架：

| 核心需求 | 推荐工具 | 关键优势 |
| --- | --- | --- |
| 秒哒，一句话生成应用 | [秒哒官网](https://www.miaoda.cn/?invitecode=user-93thly701s00) | 0 代码，通过一句话生成需求，并生成应用 |
| 搭子，桌面级 AI 智能体 | [搭子官网](https://www.dumate.cn/?track=aiwebsite_3) | 面向个人及团队的桌面级 AI 智能体，可以看见屏幕、操作软件、处理文件 |
| MonkeyCode，AI 应用开发平台 | [MonkeyCode 官网](https://monkeycode-ai.com/?ic=019d94af-c5d0-7207-a923-89d7ccf67d91) | 直接在平台里创建任务，让 AI 编码，在云端开发环境中使用终端、文件管理和预览 |
| [QoderWork](https://www.runoob.com/ai-agent/qoderwork.html) ，桌面级 AI Agent | [QoderWork](https://www.runoob.com/ai-agent/qoderwork.html) | 你说需求，它交付结果。 |
| 自动化触发与系统对接 | [n8n](https://github.com/n8n-io/n8n) | 集成面广，可自托管，常规内部系统都能打通 |
| 开发者可控的深度定制 | [Dify](https://github.com/langgenius/dify) / [LangChain](https://github.com/langchain-ai/langchain) | 前者提供完整开源方案；后者适合构建复杂推理链路 |
| 多角色协作与任务分解 | [AutoGen](https://github.com/microsoft/autogen) / [CrewAI](https://github.com/crewAIInc/crewAI) | 前者强调动态协作；后者以清晰角色体系驱动流程 |
| 自主任务执行 Agent | [AutoGPT](https://github.com/significant-gravitas/autogpt) | 早期现象级开源 Agent 项目，强调目标驱动、自主拆解任务、循环执行（Plan → Execute → Reflect） |

以下是其他流行的 AI Agent 开源框架，这些项目大多围绕工具调用（Tool Calling）、记忆（Memory）、工作流（Workflow）、多 Agent 协作（Multi-Agent）和长期任务执行能力展开。

| 项目 | 定位 | 特点 |
| --- | --- | --- |
| [OpenAI Agents SDK](https://github.com/openai/openai-agents-python) | 轻量级 Agent 开发框架 | 支持工具调用、交接（Handoff）、多 Agent 编排，结构简单，上手快 |
| [LangGraph](https://github.com/langchain-ai/langgraph) | 状态机式 Agent 编排 | 基于图结构控制复杂流程，支持长期状态和循环执行 |
| [LlamaIndex](https://github.com/run-llama/llama_index) | RAG + Agent 框架 | 擅长知识库、数据连接和检索增强场景 |
| [Semantic Kernel](https://github.com/microsoft/semantic-kernel) | 企业级 Agent 编排 | 微软开源，支持插件、流程、记忆与多模型协同 |
| [PydanticAI](https://github.com/pydantic/pydantic-ai) | 类型安全 Agent 开发 | 使用 Python 类型系统约束输出，适合工程化应用 |
| [Mastra](https://github.com/mastra-ai/mastra) | 现代全栈 Agent 框架 | 支持 Workflow、Memory、部署和可观测能力 |
| [Agno（原 Phidata）](https://github.com/agno-agi/agno) | 多 Agent 应用框架 | 强调 Agent + Knowledge + Tools 的组合能力 |
| [Camel-AI](https://github.com/camel-ai/camel) | 多智能体协作框架 | 通过角色扮演机制模拟团队协作与任务拆解 |
| [MetaGPT](https://github.com/FoundationAgents/MetaGPT) | 软件公司模拟框架 | 将产品、架构、开发、测试拆分成多个角色协同执行 |
| [Swarms](https://github.com/kyegomez/swarms) | Agent 集群编排 | 强调大规模 Agent 协同与任务调度 |
| [Haystack Agent](https://github.com/deepset-ai/haystack) | 搜索与知识增强 Agent | 适合企业搜索、文档问答和工具链组合 |
| [Atomic Agents](https://github.com/BrainBlend-AI/atomic-agents) | 可组合 Agent 架构 | 强调模块化设计与可测试性 |
| [DSPy](https://github.com/stanfordnlp/dspy) | 声明式 Prompt / Agent 框架 | 将 Prompt 与推理过程工程化、可优化 |