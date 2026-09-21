---
tags: [agent, patterns, multi-agent, supervisor, network, hierarchical]
type: concept
created: 2026-09-03
updated: 2026-09-03
---

# 多智能体协作模式（Multi-Agent Collaboration）

多个智能体如何分工协调，是 Agent 模式的最后一大类。三种主流拓扑可类比组织形态：**主管（一个人指挥）、网络（点对点交接）、层级（多级管理）**。详细实现见 [[langgraph/modules/langgraph_multi_agent/概述|LangGraph 多智能体]]；本节讲「形态差异 + 使用场景」。

## 三种模式

### 1. Supervisor 主管模式（中央协调）
一个中央协调者（主管）把任务派发给专业 worker，worker 间**不直接通信**、结果回到主管汇总。
```
         ┌──────────┐
         │  主管     │
         └───┬──┬──┬┘
             │  │  │
        ┌────▼┐┌▼──┐┌▼────┐
        │Agent A││B││C│
        └──────┘└──┘└─────┘
```
- **适用**：分工明确、需中央控制、worker 无需互通的场景；规模 3~5 个最顺手。
- **官方实现**：Anthropic 的 **orchestrator-workers** 工作流、OpenAI 的 **Manager pattern（agents as tools）**，以及 LangGraph 的 `create_supervisor`。

### 2. Network 网络模式（去中心化交接）
智能体之间**直接交接**，没有中央协调者；谁接手专业做得更好就转给谁。
```
Agent A ◄────► Agent B
   ▲              │
   └──── Agent C ◄┘
```
- **适用**：任务动态变化、智能体需直接协作/并行转发、追求灵活性。
- **官方实现**：OpenAI 的 **Handoff（任务转移，agents as tools 之反）**、LangGraph 的 Network pattern。

### 3. Hierarchical 层级模式（多级管理）
多层级树：总监→经理→基层 worker，逐级监督，管理范围可扩展。
- **适用**：大型复杂系统、规模化到几十个 worker、天然的组织层级。

## 三种模式对比

| 维度 | Supervisor | Network | Hierarchical |
|------|-----------|---------|--------------|
| 控制方式 | 中央控制 | 去中心化 | 多层级 |
| 复杂度 | 低 | 中 | 高 |
| 灵活性 | 低 | 高 | 中 |
| 可扩展性 | 中 | 高 | 高 |
| 适用 | 明确分工 | 动态协作 | 大型系统 |
| 通信开销 | 低 | 高 | 中 |
| 调试难度 | 低 | 高 | 中 |

## 权威设计基准（Anthropic / OpenAI）
- **先问能否用「单个 Agent + workflow」解决**：Anthropic 明确建议，能用 prompt chaining、routing 等 workflow 解决的，就不要上多智能体 [$TRAE_REF](https://www.anthropic.com/engineering/building-effective-agents)。多智能体带来协调复杂度与更高 token 成本。
- **需要多智能体的真正理由**（OpenAI）[$TRAE_REF](https://openai.com/business-guides-and-resources/a-practical-guide-to-building-agents)：上下文管理（各 worker 各自上下文不超载）、分布式开发、并行化。
- **两种官方形态**：Manager（中央编排，≈Supervisor）vs Handoff（对等交接，≈Network）[$TRAE_REF](https://openai.github.io/openai-agents-python/)。
- **工具多到决策质量下降时**，是拆分多智能体的关键信号。

## 何时选 Supervisor（最常见）
- 任务分工清晰、每个 worker 有明确角色
- 需要统一决策与可审计路径
- worker 之间无需直接通信、规模适中
- 需要确定性的流程控制（或叠加规则路由）

## 何时避开多智能体
- worker 需紧密实时协作（考虑 Network）
- 规模很大（>10）且需分层（考虑 Hierarchical）
- 任务其实可被 workflow 拆掉 → 回到单 Agent

## 相关导航
- [[Agent模式总览]] — 模式全集与选型
- [[langgraph/modules/langgraph_multi_agent/概述|LangGraph 多智能体]] — 三种模式的详细实现与代码
- [[langgraph/modules/langgraph_multi_agent/主管模式|主管模式]]
- [[langchain/multi-agent/多Agent系统]] — LangChain 官方视角（子Agent/交接/技能/路由器）
- [[agent/harness/Agent_六种架构]] §3 多 Agent 协作

## 来源与权威参考
- Anthropic《Building Effective Agents》
- OpenAI《A practical guide to building agents》与 Agents SDK 多智能体设计模式
- [[Agent模式总览]] 中引用的整套权威框架