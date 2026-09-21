---
tags: [agent, patterns, index, overview, 设计模式]
type: index
created: 2026-09-03
updated: 2026-09-03
---

# Agent 模式总览

> **本目录（`wiki/agent/patterns/`）是 Agent 各种主流模式讲解的唯一集中入口。**
> 单智能体的执行/推理范式、以及多智能体协作模式，都在这里分册说明；每个模式页讲清「是什么、怎么工作、优缺点、最适合的场景、与其它模式的关系」。

## 一句话理解 Agent 模式

Agent 模式解决的是同一件事：**在「思考」与「行动」之间如何编排**。区别只在于三件事——
1. 思考是一次性做完还是边做边想（规划 vs 实时）；
2. 思考是单条线还是多条线（线性 vs 树状/并行）；
3. 行动者是单个还是多个（单智能体 vs 多智能体协作）。

## 目录：单智能体执行 / 推理范式

| 模式 | 一句话 | 核心场景 |
|------|--------|----------|
| [[agent/patterns/CoT\|CoT 思维链]] | 让模型分步推理再作答 | 数学、逻辑、常识推理；无需工具 |
| [[agent/patterns/ReAct\|ReAct]] | 边想边做，Thought→Action→Observation 闭环 | 需要调用外部工具的通用 Agent |
| [[agent/patterns/PlanAndExecute\|Plan-and-Execute]] | 先规划完整计划，再逐步执行 | 长线、多步、可预分解的复杂任务 |
| [[agent/patterns/Reflection\|Reflection 反思]] | 输出后自我审查、循环改进 | 代码生成、写作等质量敏感任务 |
| [[agent/patterns/ToT\|ToT 树状思维]] | 分叉多条思路并回溯求优 | 复杂的搜索/组合/规划类问题 |
| [[agent/patterns/Reflexion\|Reflexion]] | 把经验写进记忆，重试时教训复用 | 需要从失败中学习的多轮任务 |

## 目录：多智能体协作模式

[[agent/patterns/MultiAgent协作\|多智能体协作模式（Supervisor / Network / Hierarchical）]]
—— 多个智能体如何分工与协调，详细实现见 [[langgraph/modules/langgraph_multi_agent/概述|LangGraph 多智能体]]。

| 模式 | 一句话 | 核心场景 |
|------|--------|----------|
| Supervisor（主管） | 中央协调者派发任务给专业 worker | 分工明确、需中央控制 |
| Network（网络） | 智能体之间直接交接，去中心化 | 动态协作、灵活路由 |
| Hierarchical（层级） | 逐级管理、团队树 | 大型复杂系统、规模化 |

## 权威参考框架

业界对「模式」有两个著名的官方命名，可作为本目录的对照与选型骨架：

**Anthropic《Building Effective Agents》**把它们分两类 [$TRAE_REF](https://www.anthropic.com/engineering/building-effective-agents)：
- **Workflows（工作流）**：可预测路径的编排——链式调用（prompt chaining）、路由（routing）、并行化（parallelization）、编排器-工人（orchestrator-workers）、评估-优化器（evaluator-optimizer）。核心原则：**只要任务能拆成固定步骤，就用 workflow 而非 agent**。
- **Agents（自主智能体）**：运行时由 LLM 动态决策流程与工具调用，适用于开放式、不可预测的任务。

**OpenAI《A practical guide to building agents》**强调上下文工程 + 工具 + 护栏，并对多智能体给出两种主流形态 [$TRAE_REF](https://openai.com/business-guides-and-resources/a-practical-guide-to-building-agents)：
- **Manager pattern（agents as tools）**：中心管理者把专业智能体当工具调用，保留对话控制权（≈ Supervisor）。
- **Handoff（任务转移）**：对等智能体之间把控制权移交给更专业的智能体（≈ Network）。

## 模式对比总表

| 维度 | CoT | ReAct | Plan&Exec | Reflection | ToT | Reflexion | 多Agent(Supervisor) |
|------|-----|-------|-----------|------------|-----|-----------|----------------------|
| 调用工具 | ✗ | ✓ | ✓ | 可选 | 可选 | ✓ | ✓ |
| 思考路径 | 单线 | 单线 | 单线·先规划 | 单线·循环 | 多线·回溯 | 单线·重试 | 多主体 |
| 上下文压力 | 低 | 高(累积) | 低(隔离) | 中 | 高 | 高 | 中(隔离) |
| 延迟/成本 | 低 | 中 | 中(双阶段) | 高(多轮) | 高(分支多) | 高(重试) | 高(协调) |
| 状态保持 | 一次 | 循环 | 计划即状态 | 迭代 | 分支状态 | 记忆重放 | 共享状态 |
| 复杂度 | 低 | 低-中 | 中 | 中 | 高 | 高 | 高 |
| 最适用 | 单步推理 | 工具型Agent | 长任务 | 质量精修 | 组合搜索 | 试错学习 | 多方分工 |

## 如何选型（决策链）

1. **要不要调用外部工具/现实交互？**
   - 不需要 → 只要推理增强，用 [[agent/patterns/CoT|CoT]]。
   - 需要 → 进入第 2 步。
2. **任务是「一次想清楚再执行」还是「边想边做」？**
   - 任务可提前完整拆解、步骤固定 → [[agent/patterns/PlanAndExecute|Plan-and-Execute]]（先规划）。
   - 任务不确定、需要看到中间结果再决策 → [[agent/patterns/ReAct|ReAct]]（边想边做）。
3. **单条思考线够不够？**
   - 问题存在多个可行分支、需要取舍/回溯（搜索、组合规划）→ [[agent/patterns/ToT|ToT]]。
4. **输出质量是瓶颈吗？**
   - 需要反复打磨（代码/文案）→ 在任意范式上叠 [[agent/patterns/Reflection|Reflection]]。
   - 需要从失败中不断学习 → 叠 [[agent/patterns/Reflexion|Reflexion]]。
5. **单智能体搞不定了吗？**
   - 工具太多、专业分工、需要并行/多领域 → 升级为 [[agent/patterns/MultiAgent协作|多智能体协作]]（先问：能否用 workflow 拆掉）。

> 选型铁律：**模式复杂度要匹配任务复杂度**。能用单次提示或 workflow 解决的问题，不要上多智能体。详见 [[agent/harness/Agent_六种架构]] 的「常见误区」。

## 相关导航

- [[agent/harness/Agent_六种架构]] — 架构层视角：单循环 / 规划执行 / 多Agent / 反思 / RAG+Agent / DAG
- [[agent/patterns/ReAct|ReAct]]（迁移说明：原 planning 页已并入本目录）
- [[langgraph/modules/langgraph_multi_agent/概述|LangGraph 多智能体]] — Supervisor/Network/Hierarchical 的详细实现
- [[agent/sources/菜鸟教程_AI_Agent系列]] — 菜鸟教程推理规划篇：CoT、ReAct、Plan-and-Execute、ToT、MCTS、Reflexion

## 来源与权威参考

- Anthropic《Building Effective Agents》：https://www.anthropic.com/engineering/building-effective-agents
- OpenAI《A practical guide to building agents》：https://openai.com/business-guides-and-resources/a-practical-guide-to-building-agents
- OpenAI Agents SDK 多智能体设计模式：https://openai.github.io/openai-agents-python/