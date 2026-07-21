---
tags: [concept, AI, Agent, 推理, 核心范式]
type: concept
created: 2026-06-23
updated: 2026-07-08
---

# ReAct（Reasoning + Acting）

## 定义
ReAct 是目前最主流的 Agent 推理范式，全称 Reasoning + Acting。核心思想是将内部逻辑推理（Thought）与外部工具交互（Action）交织在一起，形成 Thought→Action→Observation 的动态闭环反馈系统。
  
一种将“思考”和“行动”紧密结合的范式，让智能体边想边做，动态调  
整。

## 核心循环
1. **Thought（思考）**：模型描述当前要做什么、为什么这么做
2. **Action（行动）**：模型选择一个工具（如搜索、计算）并调用
3. **Observation（观察）**：模型读取工具返回的结果
4. **Repeat（循环）**：重复上述步骤，直到得出最终答案

用一个完整例子：用户问"帮我在北京找评分高于4.5的意大利餐厅"
- Thought：需要搜索符合条件的餐厅 → Action：调用search_web → Observation：找到Bottega意库等
- Thought：需要地址和招牌菜 → Action：调用get_restaurant_details → Observation：获得详情
- Thought：信息足够 → 输出最终答案

## 与其他推理模式的关系
- **vs CoT（Chain-of-Thought）**：CoT 只在模型内部推理，不调用外部工具；ReAct 打开了"眼睛"，能与环境交互
- **vs Plan-and-Execute**：ReAct 是边想边做；Plan-and-Execute 是先想完再做
- **vs ToT（Tree-of-Thoughts）**：ReAct 是线性路径；ToT 是树状多路径探索

## 局限性
- 短期任务表现优异，但长链任务容易上下文超载
- 整个思维和动作历史积压在同一上下文窗口中
- 过长时可能死循环或遗忘初始目标

## 实现方式
- **Prompt 驱动**：在提示词中要求模型"先想再做"，用 Thought/Action/Observation 标签引导
- **框架支持**：LangChain 的 Agent 模块原生支持 ReAct 模式
- **模型内置**：部分模型（如 o1、Claude）在训练时就内化了 ReAct 推理能力

## 在 Agent 架构中的角色
ReAct 是 [[agent/harness/Agent_六种架构]] 中**单 Agent 循环**的直接体现。几乎所有 Agent 框架都以 ReAct 为基础推理范式。

## 相关概念
- [[agent/harness/Agent_六种架构]] — 单Agent循环直接体现ReAct
- [[raw/AI Agent教程/推理与规划（Reasoning & Planning）  菜鸟教程]] — ReAct 是推理规划框架的一种
- [[agent/harness/Function_Call]] — ReAct 的 Action 步骤通过 Function Call 实现
- [[agent/harness/Agent_三层架构]] — ReAct 驱动思考层和执行层的协作
- [[agent/prompt/提示词工程]] — ReAct 可通过 Prompt 工程实现

## 来源
- [[agent/sources/菜鸟教程_AI_Agent系列]] — 工作原理篇和推理规划篇
- [[raw/Clippings/如何开发一个Agent，来手把手教你]] — 隐含的 ReAct 推理模式
