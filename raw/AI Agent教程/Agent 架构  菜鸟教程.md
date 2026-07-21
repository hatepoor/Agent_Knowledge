---
title: "Agent 架构 | 菜鸟教程"
source: "https://www.runoob.com/ai-agent/agent-architecture.html"
author:
published:
created: 2026-06-23
description:
tags:
  - "clippings"
---
## Agent 架构

Agent（智能体）是指能够自主感知环境、推理并采取行动以完成目标的 AI 系统。

本文从最简单的循环到多 Agent 协作，一文读懂六种主流架构的原理、图解与适用场景，帮助你在实际项目中做出合适的技术选型。

![](https://www.runoob.com/wp-content/uploads/2026/05/d274cacd-ee56-4e65-9330-0c792d33c1a2.webp)

---

## 什么是 Agent 架构

在 AI 应用开发中，Agent（智能体）是指能够感知环境、自主做出决策、并采取行动的 AI 系统。

与传统的"问一答一"式大模型调用不同，Agent 可以连续地执行多步操作，调用工具，甚至协调其他 Agent，来完成复杂任务。

Agent 架构是指 Agent 系统中各个组件的组织方式，决定了 Agent 的能力边界、可靠性、灵活性和适用场景。

Agent 的本质：感知 → 推理 → 行动

<svg width="100%" viewBox="0 0 680 120" role="img"><title>Agent 核心循环</title> <desc>展示 Agent 的三个核心环节：感知环境、推理决策、执行行动，形成一个循环</desc> <defs><marker id="arr0" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M2 1L8 5L2 9" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"></path></marker></defs><rect x="60" y="38" width="140" height="44" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="130" y="60" text-anchor="middle" dominant-baseline="central" fill="currentColor">感知环境</text> <rect x="270" y="38" width="140" height="44" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="340" y="60" text-anchor="middle" dominant-baseline="central" fill="currentColor">推理决策</text> <rect x="480" y="38" width="140" height="44" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="550" y="60" text-anchor="middle" dominant-baseline="central" fill="currentColor">执行行动</text> <line x1="200" y1="60" x2="270" y2="60" marker-end="url(#arr0)"></line><line x1="410" y1="60" x2="480" y2="60" marker-end="url(#arr0)"></line><path d="M550,82 L550,106 L130,106 L130,82" fill="none" stroke="currentColor" stroke-width="1.2" stroke-dasharray="4 3" marker-end="url(#arr0)"></path><text x="340" y="116" text-anchor="middle" fill="currentColor">观察结果，继续循环</text></svg>

Agent 的工作方式本质上是一个循环（Loop）——感知当前状态，推理下一步，执行行动，再次感知……直到任务完成。

不同架构的差异，就在于如何组织和扩展这个基本循环。

> 本文假设你已了解大语言模型（LLM）的基本概念，以及"工具调用"（Function Calling）的基本思路。如果还不熟悉，可以先了解这两个基础知识。

---

## 架构一：单 Agent 循环（Single Agent Loop）

入门首选 实现简单

最基础也最直观的架构，一个 Agent 从头到尾独立完成所有任务。

单 Agent 循环直接体现了 ReAct 模式（Reasoning + Acting）：每一步都是"先想再做"。LLM 充当大脑，工具调用是它的双手。

图 1 — 单 Agent 循环流程

<svg width="100%" viewBox="0 0 680 460" role="img"><title>单 Agent 循环架构图</title> <desc>展示单个 Agent 的完整工作循环：接收任务、感知环境、推理决策、调用工具，然后根据结果判断是否继续循环或任务完成退出</desc> <defs><marker id="arr1" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M2 1L8 5L2 9" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"></path></marker></defs><rect x="230" y="30" width="220" height="44" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="340" y="52" text-anchor="middle" dominant-baseline="central" fill="currentColor">用户任务输入</text> <line x1="340" y1="74" x2="340" y2="118" marker-end="url(#arr1)"></line><rect x="230" y="118" width="220" height="56" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="340" y="141" text-anchor="middle" dominant-baseline="central" fill="currentColor">感知 · 收集上下文</text> <text x="340" y="162" text-anchor="middle" dominant-baseline="central" fill="currentColor">读取文件、获取状态</text> <line x1="340" y1="174" x2="340" y2="220" marker-end="url(#arr1)"></line><rect x="230" y="220" width="220" height="56" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="340" y="243" text-anchor="middle" dominant-baseline="central" fill="currentColor">推理 · 决定下一步</text> <text x="340" y="264" text-anchor="middle" dominant-baseline="central" fill="currentColor">分析情况，选择工具</text> <line x1="340" y1="276" x2="340" y2="322" marker-end="url(#arr1)"></line><rect x="230" y="322" width="220" height="56" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="340" y="345" text-anchor="middle" dominant-baseline="central" fill="currentColor">行动 · 调用工具</text> <text x="340" y="366" text-anchor="middle" dominant-baseline="central" fill="currentColor">Bash / Read / Edit...</text> <line x1="450" y1="248" x2="504" y2="248" marker-end="url(#arr1)"></line><text x="476" y="238" text-anchor="middle" fill="currentColor">完成</text> <rect x="504" y="226" width="130" height="44" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="569" y="248" text-anchor="middle" dominant-baseline="central" fill="currentColor">输出结果</text> <path d="M340,378 L340,418 L168,418 L168,146 L230,146" fill="none" stroke="currentColor" stroke-width="1.4" stroke-dasharray="5 4" marker-end="url(#arr1)"></path><text x="160" y="285" text-anchor="end" fill="currentColor">继续循环</text> <rect x="20" y="110" width="4" height="272" rx="2" fill="currentColor" opacity="0.35"></rect><text x="30" y="246" fill="currentColor" transform="rotate(-90,30,246)" text-anchor="middle">Agent 循环</text></svg>

### 工作原理

感知：读取当前状态——文件内容、环境变量、之前步骤的输出结果，整合成当前上下文。

推理：LLM 根据上下文决定下一步动作——调用哪个工具、传入什么参数，或者判断任务是否已完成。

行动：执行工具调用，例如读写文件、搜索网络。工具的执行结果会被追加到上下文中，进入下一轮循环。

> 每次工具调用的结果都会回写到上下文（Context Window）。因此随着任务推进，上下文会不断增长，直到触达 LLM 的上下文窗口限制——这是单 Agent 循环最主要的瓶颈。

## 实例

\# 单 Agent 循环的简化实现 —— 展示 ReAct 模式的核心逻辑  
  
class SimpleAgent:  
"""单 Agent 循环的基本结构"""  
  
def \_\_init\_\_(self, model, tools, max\_turns=10):  
self.model = model # 大语言模型  
self.tools = tools # 可用工具列表  
self.max\_turns = max\_turns # 最大循环轮次，防止无限循环  
  
def run(self, task: str) -> str:  
"""执行任务的主循环"""  
context = f"用户任务：{task}"  
  
for turn in range(self.max\_turns):  
\# 第一步：思考 —— 让模型决定下一步  
response = self.model.think(context)  
  
\# 如果模型认为任务完成，返回最终答案  
if response.is\_final():  
return response.content  
  
\# 第二步：行动 —— 调用模型选择的工具  
tool\_name = response.tool\_name  
tool\_args = response.tool\_args  
tool\_result = self.tools\[tool\_name\](\*\*tool\_args)  
  
\# 第三步：将工具结果反馈给模型，进入下一轮  
context += f"\\n工具 {tool\_name} 返回：{tool\_result}"  
  
return "达到最大轮次，任务未完成"  
  
\# 使用示例  
agent = SimpleAgent(model=llm, tools={  
"read\_file": read\_file,  
"search\_code": search\_code,  
"run\_test": run\_test  
})  
result = agent.run("修复 runoob 项目中 user.py 的类型错误")

#### 优点

- 实现最简单，易于调试
- 适合任务边界清晰的场景
- 几乎所有 Agent 框架都支持

#### 缺点

- 上下文窗口容易撑满
- 复杂任务中容易"偏航"
- 无法并行处理多个子任务

最佳适用场景：修复一个 bug、编写一个函数、回答一个具体问题。任务明确，复杂度适中，不需要并行或多角色协作。

> 如果你的任务预计需要超过 15 轮工具调用，单 Agent 循环可能不是最佳选择。考虑使用多 Agent 协作或规划执行架构来分解复杂度。

---

## 架构二：规划 + 执行（Plan & Execute）

直觉友好 可审查

将"想清楚要做什么"和"实际去做"分离为两个独立阶段，提升任务的可预测性和可审计性。

规划 + 执行架构将 Agent 的工作拆分为两个明确阶段：先规划（Plan），再执行（Execute）。

在规划阶段，模型不执行任何操作，只生成一份详细的执行步骤列表。在执行阶段，系统依次完成每个步骤。这种分离让用户可以在执行前审查计划，就像 Claude Code 的 Plan Mode 一样。

图 2 — 规划 + 执行架构（含动态重规划）

<svg width="100%" viewBox="0 0 680 430" role="img"><title>规划+执行架构图</title> <desc>展示两阶段流程：规划阶段生成执行步骤列表，执行阶段依序完成每个步骤，并可根据结果动态重新规划</desc> <defs><marker id="arr2" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M2 1L8 5L2 9" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"></path></marker><marker id="arr2d" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M2 1L8 5L2 9" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"></path></marker></defs><rect x="20" y="16" width="76" height="24" rx="4" fill="currentColor" stroke="currentColor" stroke-width="0.5"></rect><text x="58" y="28" text-anchor="middle" dominant-baseline="central" fill="currentColor">规划阶段</text> <rect x="230" y="30" width="220" height="44" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="340" y="52" text-anchor="middle" dominant-baseline="central" fill="currentColor">用户任务</text> <line x1="340" y1="74" x2="340" y2="118" marker-end="url(#arr2)"></line><rect x="230" y="118" width="220" height="56" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="340" y="141" text-anchor="middle" dominant-baseline="central" fill="currentColor">规划器（Planner）</text> <text x="340" y="162" text-anchor="middle" dominant-baseline="central" fill="currentColor">生成分步执行计划</text> <line x1="340" y1="174" x2="340" y2="208" marker-end="url(#arr2)"></line><rect x="130" y="208" width="420" height="74" rx="8" fill="none" stroke="#ccc" stroke-width="1" stroke-dasharray="5 3"></rect><text x="148" y="222" fill="currentColor">执行计划</text> <rect x="150" y="224" width="106" height="44" rx="6" fill="currentColor" stroke="currentColor" stroke-width="0.7"></rect><text x="203" y="240" text-anchor="middle" dominant-baseline="central" fill="currentColor">步骤 1</text> <text x="203" y="258" text-anchor="middle" dominant-baseline="central" fill="currentColor">收集信息</text> <line x1="256" y1="246" x2="274" y2="246" marker-end="url(#arr2)"></line><rect x="274" y="224" width="106" height="44" rx="6" fill="currentColor" stroke="currentColor" stroke-width="0.7"></rect><text x="327" y="240" text-anchor="middle" dominant-baseline="central" fill="currentColor">步骤 2</text> <text x="327" y="258" text-anchor="middle" dominant-baseline="central" fill="currentColor">分析处理</text> <line x1="380" y1="246" x2="398" y2="246" marker-end="url(#arr2)"></line><rect x="398" y="224" width="106" height="44" rx="6" fill="currentColor" stroke="currentColor" stroke-width="0.7"></rect><text x="451" y="240" text-anchor="middle" dominant-baseline="central" fill="currentColor">步骤 3</text> <text x="451" y="258" text-anchor="middle" dominant-baseline="central" fill="currentColor">输出结果</text> <rect x="20" y="310" width="76" height="24" rx="4" fill="currentColor" stroke="currentColor" stroke-width="0.5"></rect><text x="58" y="322" text-anchor="middle" dominant-baseline="central" fill="currentColor">执行阶段</text> <line x1="340" y1="282" x2="340" y2="322" marker-end="url(#arr2)"></line><rect x="230" y="322" width="220" height="56" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="340" y="345" text-anchor="middle" dominant-baseline="central" fill="currentColor">执行器（Executor）</text> <text x="340" y="366" text-anchor="middle" dominant-baseline="central" fill="currentColor">依序完成每个步骤</text> <line x1="340" y1="378" x2="340" y2="408" marker-end="url(#arr2)"></line><text x="340" y="420" text-anchor="middle" fill="currentColor">完成任务</text> <path d="M450,350 L510,350 L510,246 L504,246" fill="none" stroke="currentColor" stroke-width="1.2" stroke-dasharray="5 3" marker-end="url(#arr2d)"></path><text x="520" y="300" text-anchor="start" fill="currentColor">动态重新规划</text> <text x="520" y="314" text-anchor="start" fill="currentColor">(可选)</text></svg>

### 两种变体

| 变体 | 行为 | 典型场景 |
| --- | --- | --- |
| 静态规划 | 计划一次性生成，按顺序线性执行，不中途调整 | 流程固定、步骤明确的任务，如数据迁移脚本 |
| 动态规划 | 每执行一步后重新评估，根据结果调整后续计划 | 结果不确定的任务，如调试、探索性数据分析 |

动态规划更健壮，但实现复杂度更高，且每步重新规划会消耗额外的 token。

> Claude Code 的 Plan Mode 就是这个架构的体现——点击"计划"后，AI 先输出一份详细计划供你审查，确认后才开始执行。这大幅提升了用户的掌控感。

## 实例

\# Plan & Execute 架构的简化实现  
  
class PlanExecuteAgent:  
"""先规划、后执行的 Agent"""  
  
def plan(self, task: str) -> list:  
"""阶段一：生成执行计划"""  
plan = self.model.generate(f"""  
请将以下任务拆解为可执行的步骤列表：  
任务：{task}  
返回 JSON 格式的步骤列表，每步包含：  
\- step\_id: 步骤编号  
\- description: 步骤描述  
\- tool: 需要调用的工具名  
""")  
return plan  
  
def execute(self, plan: list, dynamic: bool = False) -> str:  
"""阶段二：逐步执行计划"""  
results = \[\]  
remaining\_plan = plan.copy()  
  
while remaining\_plan:  
step = remaining\_plan.pop(0)  
output = self.tools\[step\["tool"\]\](step\["description"\])  
results.append({"step": step\["step\_id"\], "output": output})  
  
if dynamic and remaining\_plan:  
\# 动态规划：根据当前结果重新评估后续计划  
remaining\_plan = self.replan(remaining\_plan, results)  
  
return self.summarize(results)  
  
\# 使用示例  
agent = PlanExecuteAgent()  
plan = agent.plan("为 runoob 项目添加用户认证功能")  
\# 人类可以先审查 plan，确认合理后再执行  
result = agent.execute(plan, dynamic=True)

#### 优点

- 执行前可人工审查计划
- 推理和执行职责分离，清晰
- 对长任务友好

#### 缺点

- 初始计划可能不够准确
- 两阶段增加了延迟
- 静态版本难以应对意外情况

> Plan & Execute 的代价是增加了推理轮次，对于简单任务反而是一种浪费。如果一个任务可以在 3 步内完成，直接用单 Agent 循环更高效。

---

## 架构三：多 Agent 协作（Multi-Agent）

生产推荐 复杂任务

一个 Orchestrator（协调者）负责任务拆解和调度，多个 Subagent 各司其职，并行或串行地完成子任务，结果汇聚回 Orchestrator 做综合。

当单个 Agent 面临上下文窗口不足或任务过于复杂时，多 Agent 架构提供了一个优雅的解决方案：让多个专门化的子 Agent 并行工作，由一个 Orchestrator（协调者）统筹全局。

图 3 — 多 Agent 协作架构（Orchestrator + Subagents）

<svg width="100%" viewBox="0 0 680 376" role="img"><title>多 Agent 协作架构图</title> <desc>Orchestrator 协调者将任务分发给三个专门化的子 Agent，子 Agent 并行执行后将结果汇总给协调者</desc> <defs><marker id="arr3" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M2 1L8 5L2 9" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"></path></marker><marker id="arr3r" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M2 1L8 5L2 9" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"></path></marker></defs><rect x="220" y="20" width="240" height="56" rx="10" fill="currentColor" stroke="currentColor" stroke-width="1"></rect><text x="340" y="43" text-anchor="middle" dominant-baseline="central" fill="currentColor">协调者（Orchestrator）</text> <text x="340" y="63" text-anchor="middle" dominant-baseline="central" fill="currentColor">任务分解 · 调度 · 结果综合</text> <path d="M340,76 L340,130 L158,130 L158,176" fill="none" stroke="currentColor" stroke-width="1.4" marker-end="url(#arr3)"></path><line x1="340" y1="76" x2="340" y2="176" marker-end="url(#arr3)"></line><path d="M340,76 L340,130 L522,130 L522,176" fill="none" stroke="currentColor" stroke-width="1.4" marker-end="url(#arr3)"></path><text x="200" y="126" text-anchor="middle" fill="currentColor">分发任务</text> <rect x="236" y="148" width="208" height="20" rx="4" fill="currentColor" stroke="currentColor" stroke-width="0.5"></rect><text x="340" y="158" text-anchor="middle" dominant-baseline="central" fill="currentColor">并行执行</text> <rect x="68" y="176" width="180" height="56" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="158" y="199" text-anchor="middle" dominant-baseline="central" fill="currentColor">子 Agent A</text> <text x="158" y="219" text-anchor="middle" dominant-baseline="central" fill="currentColor">代码审查</text> <rect x="250" y="176" width="180" height="56" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="340" y="199" text-anchor="middle" dominant-baseline="central" fill="currentColor">子 Agent B</text> <text x="340" y="219" text-anchor="middle" dominant-baseline="central" fill="currentColor">安全检测</text> <rect x="432" y="176" width="180" height="56" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="522" y="199" text-anchor="middle" dominant-baseline="central" fill="currentColor">子 Agent C</text> <text x="522" y="219" text-anchor="middle" dominant-baseline="central" fill="currentColor">性能分析</text> <path d="M158,232 L158,280 L296,280 L296,306" fill="none" stroke="currentColor" stroke-width="1.2" stroke-dasharray="4 3" marker-end="url(#arr3r)"></path><line x1="340" y1="232" x2="340" y2="306" stroke="currentColor" stroke-width="1.2" stroke-dasharray="4 3" fill="none" marker-end="url(#arr3r)"></line><path d="M522,232 L522,280 L384,280 L384,306" fill="none" stroke="currentColor" stroke-width="1.2" stroke-dasharray="4 3" marker-end="url(#arr3r)"></path><text x="490" y="270" text-anchor="middle" fill="currentColor">返回结果</text> <rect x="220" y="306" width="240" height="50" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="340" y="327" text-anchor="middle" dominant-baseline="central" fill="currentColor">结果汇总</text> <text x="340" y="346" text-anchor="middle" dominant-baseline="central" fill="currentColor">综合各子 Agent 输出</text></svg>

### 独立上下文是核心优势

每个子 Agent 拥有独立的上下文窗口。代码审查 Agent 深度阅读 auth.py 不会影响性能分析 Agent 的判断；安全检测 Agent 产生的大量中间输出不会挤占其他 Agent 的空间。

> 子 Agent（Subagent）是短暂的、隔离的——完成一个任务后即销毁。Agent 团队（Agent Teams）则是多个独立 Agent 实例长时间协作、互相发消息，更像一个真实的团队。

## 实例

\# 多 Agent 协作的简化实现  
  
class Orchestrator:  
"""编排器：负责任务拆解、分发和结果汇总"""  
  
def \_\_init\_\_(self):  
self.subagents = {  
"code\_review": Subagent(  
name="代码审查",  
tools=\["read\_file", "static\_analysis"\],  
system\_prompt="你是代码审查专家..."  
),  
"security": Subagent(  
name="安全检测",  
tools=\["scan\_vulnerability", "check\_deps"\],  
system\_prompt="你是安全检测专家..."  
),  
"performance": Subagent(  
name="性能分析",  
tools=\["profile\_code", "analyze\_complexity"\],  
system\_prompt="你是性能分析专家..."  
)  
}  
  
def handle\_task(self, task: str) -> dict:  
\# 第一步：分析任务，决定需要哪些 Subagent  
needed = self.plan(task)  
  
\# 第二步：并行分发（各 Subagent 同时工作，独立上下文）  
results = {}  
for agent\_name in needed:  
sub\_task = self.decompose(task, agent\_name)  
results\[agent\_name\] = self.subagents\[agent\_name\].run(sub\_task)  
  
\# 第三步：汇总各 Subagent 的结果，综合输出  
return self.synthesize(task, results)  
  
\# 使用示例：一次运行，三个维度并行分析  
orch = Orchestrator()  
report = orch.handle\_task("审查 runoob 项目 PR #42")

#### 优点

- 天然支持并行，速度快
- 子 Agent 各自独立，上下文互不干扰
- 可以专门化每个子 Agent 的角色

#### 缺点

- 协调逻辑复杂，调试困难
- 多个 Agent 并行的 Token 成本更高
- Orchestrator 本身可能成为瓶颈

> 多 Agent 协作的主要成本是编排开销。如果子任务非常简单（每个只需 1-2 步），编排开销可能超过实际工作的开销，此时单 Agent 更合适。

---

## 架构四：反思与自我修正（Reflection）

高质量输出 易于集成

在 Agent 的输出环节加入质量评估，不满意则重新生成或修正，形成内部迭代循环。

反思架构为 Agent 增加了一个"质检环节"：每次生成输出后，都由一个评判者（Critic）来评估质量，如果不达标则要求修正，直到输出满足标准。

这就像开发者写完代码后自己跑一遍测试——在交付之前先自查一遍。

图 4 — 反思与自我修正架构

<svg width="100%" viewBox="0 0 680 410" role="img"><title>反思与自我修正架构图</title> <desc>Agent 执行任务后，Critic 评判者对输出质量打分，若通过则完成，否则进入修正环节</desc> <defs><marker id="arr4" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M2 1L8 5L2 9" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"></path></marker><marker id="arr4g" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M2 1L8 5L2 9" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"></path></marker><marker id="arr4r" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M2 1L8 5L2 9" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"></path></marker></defs><rect x="230" y="30" width="220" height="56" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="340" y="53" text-anchor="middle" dominant-baseline="central" fill="currentColor">执行器（Executor）</text> <text x="340" y="73" text-anchor="middle" dominant-baseline="central" fill="currentColor">生成初始输出</text> <line x1="340" y1="86" x2="340" y2="160" marker-end="url(#arr4)"></line><rect x="230" y="160" width="220" height="56" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="340" y="183" text-anchor="middle" dominant-baseline="central" fill="currentColor">评判者（Critic）</text> <text x="340" y="203" text-anchor="middle" dominant-baseline="central" fill="currentColor">评估质量与正确性</text> <line x1="450" y1="188" x2="504" y2="188" stroke="currentColor" stroke-width="1.4" fill="none" marker-end="url(#arr4g)"></line><text x="476" y="180" text-anchor="middle" fill="currentColor">通过</text> <rect x="504" y="166" width="130" height="44" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="569" y="188" text-anchor="middle" dominant-baseline="central" fill="currentColor">输出最终结果</text> <line x1="340" y1="216" x2="340" y2="288" stroke="currentColor" stroke-width="1.4" fill="none" marker-end="url(#arr4r)"></line><text x="356" y="255" text-anchor="start" fill="currentColor">未通过</text> <rect x="230" y="288" width="220" height="56" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="340" y="311" text-anchor="middle" dominant-baseline="central" fill="currentColor">修正器（Reviser）</text> <text x="340" y="331" text-anchor="middle" dominant-baseline="central" fill="currentColor">根据反馈改进输出</text> <path d="M340,344 L340,388 L168,388 L168,58 L230,58" fill="none" stroke="currentColor" stroke-width="1.3" stroke-dasharray="5 4" marker-end="url(#arr4)"></path><text x="158" y="225" text-anchor="end" fill="currentColor">重新生成</text> <text x="340" y="402" text-anchor="middle" fill="currentColor">(通常设置最大迭代次数限制)</text></svg>

### 两种实现方式

| 方式 | 机制 | 优点 | 缺点 |
| --- | --- | --- | --- |
| 自我反思 | 同一个模型先执行再评估自己的输出 | 实现简单，无额外模型成本 | 模型可能对自己的错误"视而不见" |
| Critic 模型 | 用独立的评判模型评估执行模型的输出 | 更客观，能发现执行模型盲区 | 增加模型调用成本和延迟 |

> "写单元测试 → 运行测试 → 观察失败 → 修复代码 → 再次运行"——这就是反思架构的经典应用。测试结果本身就是 Critic 的反馈信号。

## 实例

\# 反思架构的简化实现  
  
class ReflectiveAgent:  
"""带有自我反思能力的 Agent"""  
  
def \_\_init\_\_(self, model, tools, max\_reflections=3):  
self.model = model  
self.tools = tools  
self.max\_reflections = max\_reflections # 最多修正次数，防止死循环  
  
def run(self, task: str) -> str:  
\# 第一步：正常执行，产生初始输出  
output = self.model.generate(task)  
  
for i in range(self.max\_reflections):  
\# 第二步：反思 —— 评估输出质量  
critique = self.model.generate(f"""  
请严格评估以下输出：  
原始任务：{task}  
当前输出：{output}  
检查：事实错误？逻辑漏洞？遗漏信息？格式问题？  
如果输出完美无缺，请回复 "PASS"。  
""")  
  
if "PASS" in critique:  
break # 输出通过审查  
  
\# 第三步：修正 —— 根据批评意见改进  
output = self.model.generate(f"""  
原始任务：{task}  
上次输出：{output}  
问题反馈：{critique}  
请根据反馈修正输出。  
""")  
  
return output  
  
\# 使用示例  
agent = ReflectiveAgent(model=llm, tools={})  
code = agent.run("编写一个 Python 函数，实现 RUNOOB 字符串的 AES 加密")  
\# Agent 生成代码后自我检查加密实现、密钥处理，  
\# 发现漏洞后自动修正，确保输出安全可靠

#### 优点

- 显著提升输出质量
- 可以设置明确的质量标准
- 适合有客观评判标准的任务

#### 缺点

- 多次迭代增加延迟和成本
- 需要设置最大迭代次数防止死循环
- 评判标准难以形式化时效果有限

> 反思循环每次迭代都是一次额外的 LLM 调用，会明显增加延迟。此外需要设置反思次数上限，否则模型可能陷入"永远不满意"的死循环。

---

## 架构五：RAG + Agent（检索增强型智能体）

知识密集任务 大型知识库

在 Agent 的工具集里加入向量检索能力，让 Agent 在推理过程中动态查询外部知识库，克服上下文窗口的限制。

RAG（Retrieval-Augmented Generation，检索增强生成）本是一种让 LLM 查询外部知识库的技术。当它与 Agent 结合时，变得更加强大：Agent 可以主动决定何时检索、检索什么，而不是每次都被动地检索一次。

图 5 — RAG + Agent 检索增强架构

<svg width="100%" viewBox="0 0 680 400" role="img"><title>RAG + Agent 架构图</title> <desc>Agent 在推理决策过程中，主动向外部知识库发起检索，将检索到的相关内容融入上下文，再进行更准确的推理和行动</desc> <defs><marker id="arr5" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M2 1L8 5L2 9" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"></path></marker><marker id="arr5a" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M2 1L8 5L2 9" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"></path></marker></defs><rect x="230" y="20" width="220" height="44" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="340" y="42" text-anchor="middle" dominant-baseline="central" fill="currentColor">用户查询</text> <line x1="340" y1="64" x2="340" y2="110" marker-end="url(#arr5)"></line><rect x="230" y="110" width="220" height="56" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="340" y="133" text-anchor="middle" dominant-baseline="central" fill="currentColor">主动检索</text> <text x="340" y="153" text-anchor="middle" dominant-baseline="central" fill="currentColor">决定检索时机与内容</text> <rect x="480" y="100" width="165" height="76" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="562" y="128" text-anchor="middle" dominant-baseline="central" fill="currentColor">知识库</text> <text x="562" y="148" text-anchor="middle" dominant-baseline="central" fill="currentColor">文档 / 向量数据库</text> <text x="562" y="165" text-anchor="middle" dominant-baseline="central" fill="currentColor">代码库 / 记忆</text> <line x1="450" y1="131" x2="480" y2="131" fill="none" stroke="currentColor" stroke-width="1.4" marker-end="url(#arr5a)"></line><line x1="480" y1="148" x2="450" y2="148" fill="none" stroke="currentColor" stroke-width="1.4" marker-end="url(#arr5a)"></line><text x="465" y="124" text-anchor="middle" fill="currentColor">查询</text> <text x="465" y="162" text-anchor="middle" fill="currentColor">返回</text> <line x1="340" y1="166" x2="340" y2="220" marker-end="url(#arr5)"></line><text x="356" y="196" text-anchor="start" fill="currentColor">增强上下文</text> <rect x="230" y="220" width="220" height="56" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="340" y="243" text-anchor="middle" dominant-baseline="central" fill="currentColor">增强推理</text> <text x="340" y="263" text-anchor="middle" dominant-baseline="central" fill="currentColor">结合知识库内容推理</text> <path d="M450,248 L508,248 L508,138 L480,138" fill="none" stroke="currentColor" stroke-width="1.1" stroke-dasharray="4 3" marker-end="url(#arr5)"></path><text x="516" y="194" text-anchor="start" fill="currentColor">按需再检索</text> <line x1="340" y1="276" x2="340" y2="326" marker-end="url(#arr5)"></line><rect x="230" y="326" width="220" height="56" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="340" y="349" text-anchor="middle" dominant-baseline="central" fill="currentColor">执行行动</text> <text x="340" y="369" text-anchor="middle" dominant-baseline="central" fill="currentColor">基于知识的精准操作</text></svg>

### 与普通 RAG 的关键区别

普通 RAG 是"被动、一次性"的：用户提问时固定检索一次，将结果塞入 Prompt。

RAG + Agent 则不同：Agent 自主判断在推理的哪个环节需要补充知识、需要检索什么，并可以多次查询知识库，直到获得足够的信息来完成任务。

> 代码库问答 Agent 被问到"为什么这里用单例模式？"时，它会主动检索项目文档、设计决策记录、相关代码文件，而不是只凭 LLM 的训练记忆来猜测。

## 实例

\# RAG + Agent 的简化实现  
  
class RAGAgent:  
"""带有动态检索能力的 Agent"""  
  
def \_\_init\_\_(self, model, vector\_db, max\_retrievals=5):  
self.model = model  
self.vector\_db = vector\_db # 向量数据库  
self.max\_retrievals = max\_retrievals  
  
def should\_retrieve(self, context: str, question: str) -> bool:  
"""Agent 自己判断是否需要检索更多信息"""  
decision = self.model.generate(f"""  
当前已知信息：{context}  
当前问题：{question}  
现有信息是否足以回答问题？回答 YES 或 NO。  
""")  
return "NO" in decision  
  
def run(self, task: str) -> str:  
context = ""  
retrieval\_count = 0  
  
while retrieval\_count < self.max\_retrievals:  
\# Agent 自主判断是否需要检索  
if not self.should\_retrieve(context, task):  
break  
  
\# Agent 自主决定检索什么  
search\_query = self.model.generate(f"""  
任务：{task}  
已有信息：{context}  
为了完成任务，下一步应该检索什么信息？  
""")  
  
\# 执行检索，结果追加到上下文  
docs = self.vector\_db.search(search\_query)  
context += "\\n".join(docs)  
retrieval\_count += 1  
  
\# 综合所有信息生成最终答案  
return self.model.generate(f"任务：{task}\\n参考资料：{context}")  
  
\# 使用示例  
agent = RAGAgent(model=llm, vector\_db=runoob\_docs\_db)  
answer = agent.run("RUNOOB 框架中如何配置数据库连接池？")  
\# Agent 先检索"连接池配置"，发现提到"最大连接数"  
\# 如果不理解，会再次检索"最大连接数最佳实践"  
\# 最终综合多轮检索结果给出完整回答

#### 优点

- 突破上下文窗口限制
- 输出有据可查，减少幻觉
- 知识库可独立更新

#### 缺点

- 检索质量影响整体效果
- 向量数据库的维护成本
- 检索延迟增加响应时间

---

## 架构六：工作流编排（Workflow / DAG）

生产首选 高可靠性

把 Agent 行为固化为一张有向无环图（DAG），每个节点是一个 LLM 调用或工具调用，边表示数据依赖关系，由框架驱动执行。

这是最接近传统软件工程的一种 Agent 架构。与前面几种架构的最大区别在于：Agent 的自主决策空间被限制在单个节点内部，节点之间的流转是预先定义好的，不可更改。

图 6 — 工作流 DAG 架构（有向无环图）

<svg width="100%" viewBox="0 0 680 300" role="img"><title>工作流 DAG 架构图</title> <desc>有向无环图展示：数据输入后，任务 A 和任务 B 并行执行；两者完成后聚合节点处理；最后输出结果</desc> <defs><marker id="arr6" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M2 1L8 5L2 9" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"></path></marker></defs><rect x="30" y="124" width="100" height="44" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="80" y="146" text-anchor="middle" dominant-baseline="central" fill="currentColor">数据输入</text> <path d="M130,146 L152,146 L152,100 L176,100" fill="none" stroke="currentColor" stroke-width="1.4" marker-end="url(#arr6)"></path><path d="M130,146 L152,146 L152,192 L176,192" fill="none" stroke="currentColor" stroke-width="1.4" marker-end="url(#arr6)"></path><rect x="176" y="68" width="150" height="64" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="251" y="91" text-anchor="middle" dominant-baseline="central" fill="currentColor">任务 A</text> <text x="251" y="115" text-anchor="middle" dominant-baseline="central" fill="currentColor">独立并行执行</text> <rect x="176" y="160" width="150" height="64" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="251" y="183" text-anchor="middle" dominant-baseline="central" fill="currentColor">任务 B</text> <text x="251" y="207" text-anchor="middle" dominant-baseline="central" fill="currentColor">独立并行执行</text> <rect x="176" y="138" width="150" height="18" rx="4" fill="currentColor" stroke="currentColor" stroke-width="0.5"></rect><text x="251" y="147" text-anchor="middle" dominant-baseline="central" fill="currentColor">可并行</text> <path d="M326,100 L354,100 L354,146 L378,146" fill="none" stroke="currentColor" stroke-width="1.4" marker-end="url(#arr6)"></path><path d="M326,192 L354,192 L354,146 L378,146" fill="none" stroke="currentColor" stroke-width="1.4" marker-end="url(#arr6)"></path><text x="390" y="90" text-anchor="middle" fill="currentColor">A、B 均完成后才触发</text> <rect x="378" y="114" width="164" height="64" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="460" y="137" text-anchor="middle" dominant-baseline="central" fill="currentColor">聚合节点</text> <text x="460" y="161" text-anchor="middle" dominant-baseline="central" fill="currentColor">等待依赖，整合结果</text> <line x1="542" y1="146" x2="582" y2="146" marker-end="url(#arr6)"></line><rect x="582" y="124" width="84" height="44" rx="8" fill="currentColor" stroke="currentColor" stroke-width="0.8"></rect><text x="624" y="146" text-anchor="middle" dominant-baseline="central" fill="currentColor">输出</text></svg>

### 纯 Agent vs DAG 工作流

| 特性 | 纯 Agent | DAG 工作流 |
| --- | --- | --- |
| 流程控制 | 模型自主决定下一步 | 预定义的 DAG 图决定 |
| 可预测性 | 低，每次运行路径可能不同 | 高，运行路径确定 |
| 可调试性 | 难，依赖日志追踪 | 易，每个节点输入输出明确 |
| 容错性 | 依赖模型自行恢复 | 框架提供重试、断点续跑 |
| 灵活性 | 高，可应对意外情况 | 低，只能走预定义路径 |

DAG 的 "无环"（Acyclic）特性意味着工作流是确定性的——没有无限循环，执行路径可以完全预测，失败的节点可以单独重试。

> DAG 是"低自主性、高可预测性"的极端。你需要预先设计好整个流程。这不是缺点，而是刻意的设计取舍——生产环境中有时候确定性比灵活性更重要。

## 实例

\# DAG 工作流的简化定义（类似 LangGraph 风格）  
  
from langgraph import StateGraph  
  
\# 定义工作流状态 —— 节点间传递的数据对象  
class PipelineState:  
raw\_data: str = "" # 原始输入数据  
cleaned\_data: str = "" # 清洗后的数据  
analysis\_result: dict = {} # 分析结果  
final\_report: str = "" # 最终报告  
  
\# 定义 DAG 节点 —— 每个节点是独立的处理单元  
def extract\_data(state: PipelineState) -> PipelineState:  
"""节点1：从 runoob 数据库中提取原始数据"""  
state.raw\_data = query\_database("SELECT \* FROM logs")  
return state  
  
def clean\_data(state: PipelineState) -> PipelineState:  
"""节点2：清洗数据（去重、标准化格式）"""  
state.cleaned\_data = preprocess(state.raw\_data)  
return state  
  
def analyze\_data(state: PipelineState) -> PipelineState:  
"""节点3：统计分析"""  
state.analysis\_result = statistical\_analysis(state.cleaned\_data)  
return state  
  
def generate\_report(state: PipelineState) -> PipelineState:  
"""节点4：使用 LLM 生成报告"""  
state.final\_report = llm.generate(  
f"基于以下分析结果生成报告：{state.analysis\_result}"  
)  
return state  
  
\# 构建 DAG：定义节点和边（数据流向）  
workflow = StateGraph(PipelineState)  
workflow.add\_node("extract", extract\_data)  
workflow.add\_node("clean", clean\_data)  
workflow.add\_node("analyze", analyze\_data)  
workflow.add\_node("report", generate\_report)  
  
\# 定义边：extract → clean → analyze → report  
workflow.add\_edge("extract", "clean")  
workflow.add\_edge("clean", "analyze")  
workflow.add\_edge("analyze", "report")  
workflow.set\_entry\_point("extract")  
workflow.set\_finish\_point("report")  
  
\# 编译并运行  
app = workflow.compile()  
result = app.invoke(PipelineState())  
print(result.final\_report)

#### 优点

- 可预测、可审计、可重试
- 支持并行加速
- 工程化程度高，运维友好

#### 缺点

- 流程需要预先设计，灵活性低
- 难以应对未预期的情况
- 需要学习编排框架

---

## 横向对比与如何选择

从多个维度对比六种架构，帮助你快速定位适合的选项。

| 架构 | 自主性 | 可预测性 | 并行能力 | 适合任务复杂度 | 典型实现 |
| --- | --- | --- | --- | --- | --- |
| 单 Agent 循环 | 高 | 低 | 无 | 中低 | Claude Code 默认模式 |
| 规划 + 执行 | 中 | 中 | 部分 | 中高 | Claude Code Plan Mode |
| 多 Agent 协作 | 高 | 低 | 强 | 高 | AutoGen, CrewAI |
| 反思 / 自我修正 | 中 | 中 | 无 | 中 | Reflexion, Self-Refine |
| RAG + Agent | 高 | 中 | 无 | 中高 | LangChain RAG Agent |
| 工作流编排 | 低 | 高 | 强 | 高（固定流程） | LangGraph, Prefect |

### 常见组合模式

实际生产系统通常会组合使用多种架构，以下是几种成熟的组合模式：

工作流编排 + 多 Agent：用 DAG 定义主流程，每个节点内部是一个独立的 Agent。例如 CI/CD 流水线中，代码检查节点是 Code Review Agent，安全扫描节点是 Security Agent。

多 Agent + RAG：多个 Subagent 共享同一个向量知识库，各自根据子任务按需检索。例如客服系统中，订单查询 Agent 和退款处理 Agent 都查询同一个知识库但检索不同的内容。

规划执行 + 反思：先规划再执行，但每个步骤执行后加入反思环节确保质量。适合对质量要求极高的任务。

> 如果你不确定从哪里开始，从单 Agent 循环开始。它最容易实现和调试。当你发现上下文窗口不够用时，考虑多 Agent；当你需要质量保证时，加入反思层；当流程趋于稳定时，重构为 DAG 以提升可靠性。

### 常见误区

误区一：架构越复杂越好

多 Agent 协作看起来很强大，但如果你的任务用单 Agent 循环在 5 步内就能完成，引入编排开销反而降低效率。原则：用能满足需求的最简架构。

误区二：反思一定能提升质量

自我反思的有效性取决于模型的自我评估能力。如果质量要求极其严格，考虑使用 Critic 模型或引入外部验证（如代码自动测试）。

误区三：DAG 工作流不需要 Agent

DAG 定义了流程骨架，但每个节点内部仍然可以是 Agent 调用。工作流编排和 Agent 能力不是互斥的，而是互补的——DAG 提供可靠性，Agent 提供灵活性。

误区四：上下文窗口大了就不需要 RAG

即使模型支持 1M token 的上下文窗口，把所有文档塞进去仍然不是最优方案。RAG 的价值不仅在于"装得下"，更在于精准检索——减少噪音、降低推理成本、提高答案准确性。

---

## 总结

六种 Agent 架构覆盖了从高度自主到高度可控的完整光谱。

选择架构时，核心考量只有两个：你需要多大的灵活性来应对意外情况，以及你需要多大的确定性来保证结果可靠。

从简单开始，在确实需要时才增加复杂度，这是 Agent 架构选型的第一原则。