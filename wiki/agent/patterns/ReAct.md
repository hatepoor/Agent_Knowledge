---
tags: [agent, patterns, reasoning, react, 核心范式]
type: concept
created: 2026-06-23
updated: 2026-09-03
---

# ReAct（Reasoning + Acting）

> 本文档从原 `wiki/agent/planning/ReAct` 迁移本目录，作为 [[Agent模式总览]] 的成员。

## 定义
ReAct 是目前最主流的单智能体推理范式，全称 **Reasoning + Acting**。核心思想是把内部逻辑推理（Thought）与外部工具交互（Action）**交织在一起**，形成 Thought→Action→Observation 的动态闭环。让智能体「边想边做、眼睛睁开」。

## 核心循环
1. **Thought（思考）**：模型描述现在要做什么、为什么
2. **Action（行动）**：模型选择一个工具（搜索、计算、执行等）并调用
3. **Observation（观察）**：模型读取工具返回的结果
4. **Repeat（循环）**：重复以上，直到信息足够输出最终答案

```
Thought → Action → Observation → (Thought → Action → Observation → ...) → 最终答案
```

### 完整例子
用户问「帮我在北京找评分高于 4.5 的意大利餐厅」：
- Thought：需找到符合条件的餐厅 → Action：`search_web` → Observation：Bottega 意库等
- Thought：需地址和招牌菜 → Action：`get_restaurant_details` → Observation：获得详情
- Thought：信息足够 → 输出最终答案

## 与其他推理模式的关系
- **vs [[CoT]]**：CoT 只在模型内部推理、不调工具；ReAct 打开「眼睛」与环境交互。
- **vs [[PlanAndExecute]]**：ReAct 边想边做；Plan-and-Execute 先想完再做（长任务更优）。
- **vs [[ToT]]**：ReAct 单条线性路径；ToT 树状多路径并回溯。
- **vs [[Reflection]]**：ReAct 以「外部反馈（Observation）」推动下一步；Reflection 以「自我审查」推动改进。
- **vs [[Reflexion]]**：ReAct 无记忆复用；Reflexion 把失败经验写进记忆供重试参考。

## 优点 / 局限
| 优点 | 局限 |
|------|------|
| 与工具交互，能利用实时/外部信息 | 长链任务上下文易超载 |
| 决策可解释，逐步可回溯 | 思维与动作历史积压同一上下文 |
| 灵活性高，动态调整下一步 | 过长易死循环或遗忘初始目标 |
| 几乎所有框架原生支持 | 对「先规划再执行」类任务不如 Plan-and-Execute |

## 三种实现方式
1. **Prompt 驱动**：提示词要求模型用 Thought/Action/Observation 标签作答。
2. **框架支持**：LangChain Agent、LangGraph 的 ReAct agent、OpenAI Agents SDK 均内置。
3. **模型内置**：部分模型（o1、Claude 等）训练时已内化类 ReAct 能力。

## 最适合的场景
- 通用工具型 Agent：搜索、代码执行、数据库查询等需要实时反馈的任务
- 步骤不确定、需根据中间结果动态决策的任务（若要提前规划，选 [[PlanAndExecute]]）
- 单个边界清晰、建议 15 步以内的任务（超出考虑更结构化模式）

## 在架构中的角色
ReAct 是 [[agent/harness/Agent_六种架构]] 中**单 Agent 循环**的直接体现，几乎一切 Agent 框架都以其为推理底座。多智能体中的每个 worker 往往也是一个个 ReAct 智能体（见 [[MultiAgent协作]]）。

## 相关导航
- [[Agent模式总览]] — 模式全集与选型
- [[CoT]] — 无工具的推理底座
- [[PlanAndExecute]] / [[ToT]] / [[Reflection]] / [[Reflexion]] — 对照与组合
- [[agent/harness/Function_Call]] — Action 步骤的底层实现
- [[langgraph/tutorial/ReAct_Agent实现]] — LangGraph 落地代码
- [[agent/prompt/提示词工程]] — 提示词实现

## 来源与权威参考
- Yao et al.《ReAct: Synergizing Reasoning and Acting in Language Models》(2022)。
- Anthropic《Building Effective Agents》：将自主智能体视作「运行时决策循环」，ReAct 是其典型形态。
- [[agent/sources/菜鸟教程_AI_Agent系列]]：工作原理篇与推理规划篇。