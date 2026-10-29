---
tags: [interview, agent, 面试, 题库, 导航]
type: moc
created: 2026-10-18
updated: 2026-10-18
---

# Agent 面试题库

> **本目录的定位：Agent 通用能力层的面试题，不绑定任何编排框架。**
>
> 讲的是「Agent 怎么想、怎么用工具、怎么记忆、怎么协作、怎么评估、怎么安全上线」；
> **图怎么跑、状态怎么合并、checkpoint 怎么存，全部留给 [[langgraph/interview/README|LangGraph 面试题库]]。**

---

## 一、和另外两个目录的边界（⚠️ 先看这张表，避免重复背）

| 问题 | 去哪一篇 |
|------|---------|
| Agent 是什么、和 Workflow 怎么选、为什么失败 | **本目录 01** |
| ReAct 循环、停止条件、和 CoT/Plan-and-Execute 的区别 | **本目录 02** |
| 工具 schema、并行调用、MCP 协议 | **本目录 03** |
| 任务分解、反思、长任务目标漂移 | **本目录 04** |
| 记忆分层、上下文压缩、记忆冲突 | **本目录 05** |
| Skills 渐进加载、Skills vs MCP | **本目录 06** |
| 多 Agent 的协作拓扑、角色划分、上下文传递 | **本目录 07** |
| Agent 评估、轨迹评估、trace 归因 | **本目录 08** |
| Prompt 注入、沙箱、权限、成本控制 | **本目录 09** |
| `StateGraph` / `superstep` / reducer / Channel | [[langgraph/interview/01-核心概念与运行模型]] |
| `Send` / `Command` / 条件边 / 子图 | [[langgraph/interview/03-控制流与动态编排]] |
| Checkpointer / `thread_id` / 时间旅行 | [[langgraph/interview/04-持久化与Checkpointer]] |
| `interrupt()` / 审批恢复 / 幂等 | [[langgraph/interview/05-人机协同与中断恢复]] |
| `create_agent` / middleware / LCEL | `wiki/langchain/`（后续建 interview） |

⚠️ **一句话原则**：**本目录回答「为什么这么做」，LangGraph 目录回答「用图怎么实现」。**面试时先讲本目录的取舍，再落到 LangGraph 的机制，层次就出来了。

---

## 二、文件清单

| 文件 | 主题 | 题数 |
|------|------|------|
| [[agent/interview/01-Agent基础与能力边界]] | 定义、组成、自主性、适用边界、失败模式 | 9 |
| [[agent/interview/02-ReAct与推理循环]] | ReAct、CoT 对比、停止条件、死循环 | 9 |
| [[agent/interview/03-工具调用与MCP]] | Function Calling、schema、并行、MCP | 9 |
| [[agent/interview/04-规划反思与任务分解]] | Plan-and-Execute、Reflection、Reflexion | 8 |
| [[agent/interview/05-记忆与上下文工程]] | 长短记忆、压缩、写入/遗忘、多租户 | 9 |
| [[agent/interview/06-Skills与能力加载]] | Skills vs Prompt/MCP、渐进加载 | 8 |
| [[agent/interview/07-多智能体协作模型]] | 四种拓扑、角色、通信、A2A | 9 |
| [[agent/interview/08-Agent评估与可观测性]] | 指标、轨迹评估、LLM-as-judge、trace | 8 |
| [[agent/interview/09-安全与生产落地]] | 注入、沙箱、权限、成本、灰度 | 9 |

**合计 9 篇、78 题。**

---

## 三、使用方式

```
第一遍：只看问题，口述 60 秒版本
第二遍：读答案，把「追问」单独拎出来练
第三遍：按「01 → 02 → 03 → 05 → 07 → 09」串成一条项目叙事
```

**串讲顺序建议**：Agent 定位（01）→ 推理循环（02）→ 工具（03）→ 记忆（05）→ 多 Agent（07）→ 安全上线（09）。这条线正好对应一个 Agent 项目从设计到上线的顺序。

---

## 四、版本与口径

| 项 | 说明 |
|---|---|
| 框架绑定 | **不绑定**。题目本身不依赖 LangChain/LangGraph 版本 |
| 提到框架时 | 只作为「一种实现」，不作为标准答案 |
| 协议类 | MCP、A2A 仍在演进，答题时说「当前主流做法」，不要说死 |
| 模型能力 | 推理模型（内置 CoT）会改变 ReAct 的形态，答案里要体现这个变量 |

⚠️ **不要背某个框架的 API**。Agent 岗面试官更在意「你知道为什么需要这一层」，API 可以查文档。

---

## 五、和知识库其他部分的关系

本目录只给「答题版本」，机制细节回到：

| 主题 | 详解页 |
|------|--------|
| Agent 模式总览 | [[agent/patterns/Agent模式总览]] |
| ReAct | [[agent/patterns/ReAct]] |
| CoT / ToT / Reflection / Reflexion | [[agent/patterns/CoT]] · [[agent/patterns/ToT]] · [[agent/patterns/Reflection]] · [[agent/patterns/Reflexion]] |
| Plan-and-Execute | [[agent/patterns/PlanAndExecute]] |
| 多智能体 | [[agent/patterns/MultiAgent协作]] |
| 记忆 | [[agent/memory/Agent_Memory]] · [[agent/memory/短期记忆]] |
| Skills | [[agent/skills/Skills]] · [[agent/skills/Skills实现与集成]] |
| 上下文工程 | [[agent/prompt/上下文工程]] |
| Function Call | [[agent/harness/Function_Call]] |
| 沙箱与安全 | [[agent/harness/Agent沙箱与安全隔离]] |
| 六种架构 | [[agent/harness/Agent_六种架构]] |
| 项目话术 | [[interview/04-项目经验/项目一-终端多智能体系统]] |

---

## 六、答题总原则

| 原则 | 说明 |
|------|------|
| **先讲取舍，再讲实现** | 「为什么用 Agent 而不是 Workflow」比「我会写 ReAct」值钱 |
| **承认边界** | 能说清「什么场景不该用 Agent」的人，通常真踩过坑 |
| **带一个失败案例** | 死循环、工具误用、上下文爆炸，任选一个讲透 |
| **区分「模型能力」和「工程补偿」** | 模型变强会消掉一部分脚手架，但不会消掉权限、审计、恢复 |
| **不编指标** | 没有实测就说「我们关注的是任务成功率和单任务成本」，不要报假数字 |
