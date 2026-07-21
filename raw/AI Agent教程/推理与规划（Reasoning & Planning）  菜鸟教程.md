---
title: "推理与规划（Reasoning & Planning） | 菜鸟教程"
source: "https://www.runoob.com/ai-agent/reasoning-planning.html"
author:
published:
created: 2026-06-23
description: "推理与规划（Reasoning & Planning）  在构建自主 AI Agent 的过程中，如果说大语言模型（LLM）是 Agent 的大脑，工具调用（Tool Use）是手脚，那么推理与规划（Reasoning & Planning）就是将其从简单的问答机升级为自主问题解决者的核心引擎。  复杂的现实任务往往无法通过一次生成（One-pass generation）完成。AI 需要具备拆解目标、逻辑推演、探索路径.."
tags:
  - "推理与规划（Reasoning & Planning）"
---
## 推理与规划（Reasoning & Planning）

在构建自主 AI Agent 的过程中，如果说大语言模型（LLM）是 Agent 的大脑，工具调用（Tool Use）是手脚，那么 **推理与规划（Reasoning & Planning）** 就是将其从简单的问答机升级为自主问题解决者的核心引擎。

复杂的现实任务往往无法通过一次生成（One-pass generation）完成。AI 需要具备拆解目标、逻辑推演、探索路径、自我修正以及调度工具的能力。

以下是目前业界最主流的推理与规划框架。

---

## 思维链（Chain of Thought, CoT）

### 逐步推理能力

传统 LLM 生成答案时往往是直觉式的一步到位。

思维链（CoT）的核心思想是： **强制要求模型在输出最终答案前，先显式地输出中间的推理步骤（Let's think step by step）** 。这种做法能显著激活模型在复杂数学、逻辑推理和常识问答中的潜力。

CoT 不仅让模型有了更多的计算时间（token 数量代表计算量），还让后续的生成能建立在前面正确的逻辑基础上。

## 实例：Few-shot CoT 提示词设计

\# 通过提供包含推理过程的示例，引导模型进行 CoT 推理  
prompt = """  
问题：罗杰有5个网球。他又买了2罐网球。每罐有3个网球。他现在共有多少个网球？  
解答：罗杰一开始有5个网球。2罐网球，每罐3个，共计 2 \* 3 = 6 个网球。5 + 6 = 11。答案是11。  
  
问题：食堂有23个苹果。如果他们用掉20个做午餐，又买了6个，现在有多少个苹果？  
解答：食堂本来有23个苹果。用掉20个后剩下 23 - 20 = 3 个。又买了6个，现在有 3 + 6 = 9 个。答案是9。  
  
问题：{user\_question}  
解答："""

---

## ReAct 框架（Reasoning + Acting）

### 推理 + 行动循环

如果说 CoT 只是在模型内部闭门造车，那么 **ReAct（Reason + Act）** 则是让模型睁开眼睛看世界。它将内部逻辑推理（Thought）与外部工具交互（Action）交织在一起，形成一个动态的闭环反馈系统。

在 ReAct 范式下，Agent 遵循 **Thought（思考） -> Action（行动） -> Observation（观察）** 的循环，直到得出最终结论。

<svg width="100%" viewBox="0 0 680 200" role="img" xmlns="http://www.w3.org/2000/svg"><title>ReAct 循环架构图</title> <desc>展示了 Thought, Action, Observation 之间的循环关系。</desc> <defs><marker id="arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M0 0L10 5L0 10" fill="#999"></path></marker></defs><g><rect x="100" y="70" width="120" height="60" rx="8" stroke-width="1.5"></rect><text x="160" y="94" text-anchor="middle" dominant-baseline="central">Thought</text> <text x="160" y="112" text-anchor="middle" dominant-baseline="central">分析现状与目标</text> </g><path d="M 230 100 Q 280 60 330 100" fill="none" stroke="#999" stroke-width="1.5" marker-end="url(#arrow)"></path><g><rect x="340" y="70" width="120" height="60" rx="8" stroke-width="1.5"></rect><text x="400" y="94" text-anchor="middle" dominant-baseline="central">Action</text> <text x="400" y="112" text-anchor="middle" dominant-baseline="central">调用外部工具/API</text> </g><path d="M 400 140 Q 400 180 340 180" fill="none" stroke="#999" stroke-width="1.5" marker-end="url(#arrow)"></path><g><rect x="190" y="150" width="140" height="40" rx="8" stroke-width="1.5"></rect><text x="260" y="170" text-anchor="middle" dominant-baseline="central">Observation (观察)</text> </g><path d="M 180 170 Q 140 170 140 140" fill="none" stroke="#999" stroke-width="1.5" marker-end="url(#arrow)"></path><g><rect x="20" y="20" width="90" height="30" rx="6" stroke-width="1.5"></rect><text x="65" y="35" text-anchor="middle" dominant-baseline="central">User Query</text></g><line x1="110" y1="35" x2="160" y2="60" stroke="#999" stroke-width="1.5" marker-end="url(#arrow)"></line></svg>

> **局限性：** ReAct 在短期的、步骤清晰的任务中表现优异。但由于整个思维和动作历史都积压在同一个上下文窗口中，当任务链条过长时，极易陷入死循环或因为上下文超载而遗忘初始目标。

---

## Plan-and-Execute（规划先行执行模式）

为了解决 ReAct 在长线任务中的疲软，Plan-and-Execute 将思考和行动进行了解耦，采用了类似人类做大型项目的策略： **先出排期表，再挨个干活** 。

系统通常分为两个独立的角色：

1. **Planner（规划者）** ：负责接收大目标，生成详细的 Step-by-Step 子任务列表。
2. **Executor（执行者）** ：负责按顺序执行这些子任务。执行器通常就是一个小型的 ReAct Agent，每次只专注完成当前的一个小目标。
<svg width="100%" viewBox="0 0 680 200" role="img" xmlns="http://www.w3.org/2000/svg"><title>Plan-and-Execute 架构图</title> <defs><marker id="arr-pe" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M0 0L10 5L0 10" fill="#999"></path></marker></defs><g><rect x="20" y="80" width="80" height="40" rx="8" stroke-width="1.5"></rect><text x="60" y="100" text-anchor="middle" dominant-baseline="central">复杂目标</text> </g><line x1="100" y1="100" x2="130" y2="100" stroke="#999" stroke-width="1.5" marker-end="url(#arr-pe)"></line><g><rect x="140" y="50" width="140" height="100" rx="8" stroke-width="1.5"></rect><text x="210" y="70" text-anchor="middle" dominant-baseline="central">Planner (规划者)</text> <text x="210" y="95" text-anchor="middle" dominant-baseline="central">1. 抓取网页</text> <text x="210" y="115" text-anchor="middle" dominant-baseline="central">2. 提取数据</text> <text x="210" y="135" text-anchor="middle" dominant-baseline="central">3. 生成报告</text> </g><line x1="280" y1="100" x2="310" y2="100" stroke="#999" stroke-width="1.5" marker-end="url(#arr-pe)"></line><g><rect x="320" y="60" width="160" height="80" rx="8" stroke-width="1.5"></rect><text x="400" y="85" text-anchor="middle" dominant-baseline="central">Executor (执行者)</text> <text x="400" y="110" text-anchor="middle" dominant-baseline="central">逐个处理子任务</text> <text x="400" y="125" text-anchor="middle" dominant-baseline="central">(每次独立上下文)</text> </g><line x1="480" y1="100" x2="520" y2="100" stroke="#999" stroke-width="1.5" marker-end="url(#arr-pe)"></line><g><rect x="530" y="80" width="100" height="40" rx="8" stroke-width="1.5"></rect><text x="580" y="100" text-anchor="middle" dominant-baseline="central">最终交付</text></g></svg>

---

## Tree of Thoughts (ToT) 与树状多路径探索

### 树状多路径探索

无论是 CoT 还是 Plan-and-Execute，本质上都是 **线性** 的路径探索。但在写代码、解数学题或创意写作时，人类往往会设想多种方案，评估后选择最佳的，甚至在发现错误时回溯。

**ToT（思维树）** 将推理过程建模为一棵树：节点是当前的思维状态。模型会在每一个分支点生成多个候选 Thought，然后通过内部的 Evaluator（评估器）对这些节点进行打分（如：可行、可能有风险、不可行）。结合 **BFS（广度优先搜索）** 或 **DFS（深度优先搜索）** 算法，决定是继续深入还是回溯重试。

---

## 任务规划 & MCTS (蒙特卡洛树搜索)

### 复杂任务拆解与搜索

在涉及战略游戏或极高难度的推理任务（如前沿的数学验证、复杂的代码仓库重构）时，简单的 ToT 仍不够高效。业界开始将 LLM 与传统的强化学习搜索算法 **MCTS（Monte Carlo Tree Search）** 结合（类似 AlphaGo 的核心逻辑）。

- **LLM 作为策略网络（Policy Network）** ：提供启发式的下一步行动建议，减少无意义的分支扩展。
- **LLM/代码环境作为价值网络（Value Network）** ：通过仿真执行（Rollout）预判某个行动序列的最终胜率或成功率。
- **优势** ：在庞大的解空间中，能够找到最具有全局最优潜力的规划路径。

---

## Reflexion：自我反思与纠错

人类在执行任务时，如果第一次失败了，会总结经验教训并在下一次尝试中规避错误。

**Reflexion** 框架赋予了 Agent 类似的能力。

在 Reflexion 闭环中，当 Agent 的输出被判定为失败（例如：测试用例未通过、API 报错）时，会触发一个 **Reviewer 机制** 。

LLM 被要求根据历史动作和失败的反馈写一段口语化的反思（Reflection），例如： *"我刚才使用了错误的 API 参数格式，下一次我应该查阅文档后再传递 JSON"* 。这段反思会被存入情景记忆（Episodic Memory）中，作为下一次尝试的上下文提示，从而大幅提升 Agent 的自愈合能力。

## 实例：Reflexion 的反思提示词设计

reflection\_prompt = """  
你是一个正在尝试编写 Python 爬虫的 AI 助手。  
这是你刚才执行的代码：{previous\_code}  
这是运行环境返回的错误信息：{error\_traceback}  
  
请深刻反思：  
1\. 错误发生的根本原因是什么？  
2\. 在下一次尝试中，你具体的修改策略是什么？  
  
请将反思记录下来，以指导后续的行动。  
"""

---

## 任务分解策略与工程化实践

在实际生产级的 AI 智能体开发中，纯靠 LLM "零样本"进行复杂规划是不稳定的。常用的混合干预策略包括：

| 干预策略 | 核心做法 | 适用场景 |
| --- | --- | --- |
| **子任务模板化 (SOP)** | 不让 LLM 自由规划，而是预先定义好标准操作程序（SOP），让 LLM 在固定的状态机（State Machine）中流转。 | 客服系统、标准化的数据清洗流水线。 |
| **HITL (Human-in-the-Loop)** | 在 Planner 生成任务列表后，中断执行，要求人类用户进行确认、修改或审批（Approve），然后再交由 Executor 执行。 | 高风险操作：如删除数据库记录、发送群发邮件、大额资金转账。 |
| **RLHF 引导规划** | 利用强化学习和人类偏好反馈，专门微调大模型的规划能力，使其更倾向于生成安全、高效的步骤组合。 | 底层大语言模型基座的训练阶段（如 OpenAI 的 o1 模型训练）。 |

### 框架对比总结

| 模式 | 核心机制 | 优点 | 缺点 |
| --- | --- | --- | --- |
| **CoT** | Step-by-Step 线性推理 | 实现极简，显著提升基础推理准确度 | 无法调用外部工具，容易一条道走到黑 |
| **ReAct** | 交替思考与行动闭环 | 动态适应环境，能通过观测实时调整 | 上下文容易随步骤累积而爆炸，迷失初衷 |
| **Plan-and-Execute** | 先拆解为子任务，再隔离执行 | 极其适合长线复杂任务，上下文清晰 | 面对突发变化（规划本身出错时）不够灵活 |
| **ToT / MCTS** | 树状搜索，评估回溯 | 能解决最高难度的复杂逻辑问题 | 计算成本极其高昂，Token 消耗呈指数级 |
| **Reflexion** | 基于失败反馈生成反思记忆 | 具备自我纠错和持续进化的能力 | 依赖明确的反馈信号（如代码编译器报错） |